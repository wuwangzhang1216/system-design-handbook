# 06 · 特征平台与训练服务一致性

> 模型上线之后掉点，八成不是模型的问题，是同一个特征名在两条管道里算出了两个值。
> 而最贵的那一类错误——训练时看到了未来——在离线指标上表现为**指标变好**。

---

## 读这一章之前

**你在工作中遇到过这个**

你的排序模型离线 AUC 从 0.712 涨到 0.751，灰度 5% 上线，CTR 反而跌了 3.1%。回滚、重跑特征、重训，三轮下来结论一样：
离线永远好，线上永远差。第八天有人把线上真正喂给模型的那份特征向量 dump 下来，按 `request_id` 和训练集里同一行逐列比对 ——
`user_ctr_7d` 这一列，92% 的样本对不上，训练侧平均高 0.04。根因是离线那份 7 日点击率是当天跑批时用**全量日志**算的，
包含了这次曝光**之后**发生的点击；线上只能拿到请求那一刻之前的数据。模型训练时学会了"这个用户接下来会点"，
上线后那一列的信息量凭空消失，剩下的权重全是噪声。

**需要先懂的概念**

| 概念 | 一句话 | 详见 |
|---|---|---|
| p50 / p99 / 尾延迟 | 排序后第 50% / 99% 位置的那个耗时；扇出到 N 个后端时慢的那个决定整体 | [00-concepts §3](../00-foundations/00-concepts.md)、[00/01 §7](../00-foundations/01-fundamentals.md) |
| 最终一致与复制延迟 | 写完之后别人不一定立刻读到；中间那段窗口是合法的、不报警的 | [00-concepts §6](../00-foundations/00-concepts.md) |
| 缓存的回源与热 key | 命中率 90% 的缓存挂掉 = 下游瞬间 10 倍流量；单个 key 太热要单独一条路径 | [01/02](../01-building-blocks/02-caching.md) |
| 事件时间 vs 处理时间、有状态流处理 | 事件"发生"的时刻和你"算到"它的时刻是两回事，窗口聚合按哪个算结果完全不同 | [01/03 §6](../01-building-blocks/03-messaging-and-streams.md) |
| Lakehouse 表格式与数据契约 | Iceberg/Delta 的快照与时间旅行；schema 变更要过 CI 门禁而不是靠口头约定 | [02/05 §2 §4](../02-architecture-patterns/05-data-platform.md) |
| ML 系统坏掉时会返回 200 OK | 没有异常、没有报错，只有指标慢慢变差 —— 普通系统的监控对它是失明的 | [08/01 §4](01-ml-system-overview.md) |

**这一章要回答的问题**

1. 离线涨、线上跌，我怎么在一周之内定位到"是哪一列特征坏了"，而不是继续猜模型结构？
2. "训练时看到了未来"具体是怎么发生的？为什么它在离线评估里表现为指标**变好**，因而几乎不可能被离线评估自己抓到？
3. 一次排序请求要取几百个特征，怎么在 10 ms 内取完？限制我的到底是特征个数，还是别的东西？
4. 2026 年了，我还要不要单独立一个 feature store 项目？判据是什么，不立的代价是什么？

**本章新引入的术语**

| 术语 | English | 一句话定义 |
|---|---|---|
| 训练/服务偏斜 | training-serving skew | 同一个特征名，训练集里那个值和线上真正喂给模型的那个值不一样 |
| 特征穿越 / 数据泄漏 | data leakage | 训练样本里混进了"标签发生那一刻还不存在"的信息 |
| 时间点正确性 | point-in-time correctness | 每条训练样本只允许看到标签时间戳之前已经存在的特征值，一秒都不能多 |
| as-of join | as-of join | 按实体主键匹配、并只取时间戳 ≤ 标签时间戳的那条最新特征值的拼接方式 |
| 实体 | entity | 特征挂靠的主键对象（用户 / 商品 / 商户 / 设备），它决定了在线取数按什么 key 查 |
| 特征视图 | feature view / feature group | 共享同一个实体主键和同一份计算逻辑的一组特征，是物化和取数的最小单位 |
| 在线存储 / 离线存储 | online store / offline store | 前者按主键返回"当前值"、毫秒级；后者存全部带时间戳的历史值，供拼训练集 |
| 物化 | materialization | 把离线算好的特征值批量写进在线存储的那个动作；它的滞后决定在线值有多旧 |
| 特征新鲜度 | feature freshness | 从"产生这个特征的事件真的发生"到"线上能取到它"之间的时间 |
| 服务端特征日志 | feature logging | 线上把实际喂给模型的那份特征向量连同预测结果一起落盘 |
| 回填 | backfill | 用今天的代码，重新计算历史某一段时间的特征值 |
| 请求时变换 | on-demand transformation | 只有拿到请求参数才能算的特征（当前距离、距上次点击多少秒），在读路径上现算 |

---

## 1. 一个特征名，两条命

先看清楚 skew 是从哪里长出来的：**同一个特征名，在你的系统里有两份完全独立的实现，跑在两套数据源上，用两个不同的时钟。**

```
                        ┌── 离线侧：训练 ──────────────────────────────┐
用户行为   ┌─► Kafka ──►│ 数仓落表(T+1) ─► Spark 批作业 ─► 离线特征表  │
  事件 ────┤            │  (Iceberg 全历史)      (entity, event_time)  │
           │            │ 标签表(request_id, label_time, y) ─┐         │
           │            └────────────────────────────────────┴ as-of join ─► 训练集 ─► 训练
           │            ┌── 在线侧：服务 ──────────────────────────────┐
           └─► Flink ──►│ 有状态窗口聚合 ─► 物化 ─► 在线 KV ─┐         │
                        │ 排序请求 ─► 特征服务 ──────────────┘         │
                        └──────────────────┬───────────────────────────┘
                                           └─► 特征向量 ─► 模型

两条路径上任何一处不同，都会变成一个数值差：
   线上值 = f_serve(源_online,  语义_online,  t_请求)
   训练值 = f_train(源_offline, 语义_offline, t_标签)
   skew   = 训练值 − 线上值      ← 你要监控的是这个差，不是模型指标
```

**这条式子里有三个自变量，而绝大多数团队只对齐了第一个（源）。**
下一节的五种成因，正好是这三个自变量的排列组合。

---

## 2. 训练服务偏斜：五种成因，各自的检测手段

Google 把 skew 拆成三个来源：训练与服务**管道实现差异**、训练之后**数据本身漂移**、模型与自身产出之间的**反馈回路**
（[Google Cloud](https://cloud.google.com/blog/topics/developers-practitioners/monitor-models-training-serving-skew-vertex-ai)）。这一节只谈第一类 —— 它是唯一**上线第一天就 100% 存在、且完全由你造成**的那一类。

| # | 成因 | 具体长什么样 | 症状 | 怎么检测 |
|---|---|---|---|---|
| 1 | **两套实现** | 离线是 Spark SQL，在线是 Java 服务里手写的循环。"30 天购买数"被实现成 15 天 | 单列均值系统性偏移；模型该列重要性上线后骤降 | 逐列**精确匹配率**：用 `request_id` 把服务端日志和训练集 join，算 `value_train == value_serve` 的比例 |
| 2 | **数据源不同** | 离线读数仓 T+1 快照，在线读第三方实时 API；或过滤条件漏了"仅已结算" | 匹配率长期卡在某个固定水位（如 87%），差值分布双峰 | 差值的**均值 + P99 分位**。均值≈0 但 P99 很大 = 少数样本严重不一致，通常是源不同 |
| 3 | **时间窗口语义不同** | `last_24h` 离线被理解成"当天 00:00 至今"，线上是"滚动 24 小时"；回填用 24h 窗口而线上推理用 5min 窗口 | 匹配率随一天中的小时数呈周期波动 —— 这是最好认的指纹 | 把匹配率**按 hour-of-day 分组**画出来。平的 = 没问题，凌晨高白天低 = 窗口口径不同 |
| 4 | **缺失值处理不同** | 训练侧新用户是 `NULL`（XGBoost 会为 NULL 学一个默认分支方向），线上服务查不到时填 `0` | 新用户 / 长尾实体上的预测系统性偏高或偏低 | 分开监控 **null rate**：训练侧 null 率 8%、服务侧 0%，这一对数字本身就是结论 |
| 5 | **特征上线延迟 / 物化滞后** | 特征定义改了，离线重跑立刻生效，在线要等下一次物化；或新特征训练里有、线上还没铺到全部分片 | 上线当天匹配率好，第 3 天开始下滑；或按机房/分片分组时只有一部分差 | **物化滞后监控**：`now − max(已物化 event_time)`，超过 2× 调度周期告警；再按 `serving_pod` 分组看匹配率 |

**检测三件套（Nubank 的口径，可以直接抄）**：每日**精确匹配率** / **差值均值** / **差值 P99 分位**，
按 `instance_id`（一次预测的唯一 id）把训练侧和服务侧的特征 join 起来逐列比
（[Nubank 工程博客, 2023-06-27](https://building.nubank.com/dealing-with-train-serve-skew-in-real-time-ml-models-a-short-guide/)）。

**唯一能把 skew 收敛到零、而不只是检测它的做法：服务端特征日志。**
线上把实际喂进模型的那份向量连同 `request_id`、`prediction`、模型版本一起落盘，下一轮训练**直接用这份日志当训练集** ——
"训练值"和"线上值"在定义上就是同一份数据。代价两条：① 存储，500 列 × 8 B × 10 亿次预测/天 ≈ 4 TB/天原始，
要按列存 +（只做监控时）1–5% 采样，做训练集则必须全量或分层采样；② 你只能学到**已上线的特征**，
新特征必须先 **shadow（影子：新特征照常计算并落盘，但不参与线上打分）**记录几周才能进训练集。
**"新特征先 dark launch"的真正理由是攒训练数据，不是稳定性。**

**Chronon 的闭环值得单独说**：它把线上 fetch 请求日志（key + timestamp）当作 join 的**左表**，
用离线代码重新回填一遍，再和线上实际取到的值逐条比对，产出一个一致性指标
（[Airbnb 工程博客, 2024-04-08](https://medium.com/airbnb-engineering/chronon-airbnb-s-ml-feature-platform-is-now-open-source-d9c4dba859e8)）。
这是公开资料里最干净的 skew 检测闭环 —— 关键在于它比对的不是分布，而是**逐条的值**。

**分布距离只是兜底**：Vertex AI 对数值特征用 JS divergence、类别特征用 L∞ 距离，默认告警阈值 **0.3**，默认频率 24 h、最小粒度 1 h
（[Vertex AI ModelMonitoringObjectiveConfig](https://docs.cloud.google.com/vertex-ai/docs/reference/rest/v1beta1/ModelMonitoringObjectiveConfig)）。
但分布相同 ≠ 逐条相同：把两列的值整体打乱，分布距离是 0，匹配率是 0。**能逐条比就别比分布。**

⚠️ **skew 不是 drift。** 两者基线不同：**skew 是拿线上值和训练集比，drift 是拿今天和上一个时间窗比。**
只做 drift 监控会完全漏掉"部署第一天就存在"的管道实现差异 —— 它每天都一样地错，drift 曲线看起来非常平稳。
修好它值多少钱：Google Play 修复 training-serving skew 后，主落地页安装率提升 **2%**（同上 Google Cloud 博客）——
在成熟业务里，2% 是一个模型结构改动很难拿到的量级。

---

## 3. 时间点正确性：穿越是怎么发生的

### 3.1 一条时间轴看清它

标签是"这次曝光有没有被点击"，特征是 `user_ctr_7d`。

```
真实世界时间轴（用户 u=42）
  t₀        t₁        t₂     ★ t_label      t₃        t₄        今夜 23:00
  │         │         │         │           │         │            │
 点击      点击      点击    曝光·未点击   点击      点击        跑日批
──┴─────────┴─────────┴─────────┴───────────┴─────────┴────────────┴──►

  ✅ 正确的 user_ctr_7d ：只数 [t_label−7d, t_label) 区间   → 3 次点击 / 40 次曝光 = 0.075
  ├───────────────────────────────────┤
                                      ╳ 到此为止，右边一律不可见

  ❌ 错误的 user_ctr_7d ：日批在 23:00 用"当天全量日志"算    → 5 次点击 / 44 次曝光 = 0.114
  ├──────────────────────────────────────────────────────┤
                                      └── t₃、t₄ 就是"未来" ──┘

  上线后模型能拿到的只有 ✅ 那条。它训练时用的是 ❌ 那条。
```

关键在于 **`dt` 粒度的 join 天然就是穿越**：日批产出的是"这一天结束时的值"，而这一天里发生的每一次曝光 `t_label` 都早于 23:00 ——
**按 `dt` join，等于让每条样本看到了它当天之后的所有行为。**

### 3.2 正确的拼接：as-of join

```sql
-- ❌ 穿越版：按天 join 特征快照（99% 的 skew 事故从这一行开始）
SELECT l.request_id, l.y, f.user_ctr_7d
FROM   labels l
JOIN   feat_daily f
  ON   f.user_id = l.user_id AND f.dt = l.dt;

-- ✅ as-of join：只取 event_time ≤ label_time 的最新一条
--    有 ASOF 语法的引擎（DuckDB / ClickHouse / Databricks）直接写
SELECT l.request_id, l.y, f.user_ctr_7d
FROM   labels l
ASOF JOIN feat_versioned f
  ON   f.user_id = l.user_id AND f.event_time <= l.label_time;

-- ✅ 没有 ASOF 语法时的等价写法（Spark / Hive 通用）
SELECT request_id, y, user_ctr_7d FROM (
  SELECT l.request_id, l.y, f.user_ctr_7d,
         ROW_NUMBER() OVER (PARTITION BY l.request_id ORDER BY f.event_time DESC) AS rn
  FROM   labels l JOIN feat_versioned f
    ON   f.user_id = l.user_id
   AND   f.event_time <= l.label_time
   AND   f.event_time >  l.label_time - INTERVAL 7 DAY   -- 必须限定下界，否则全表扫
) WHERE rn = 1;
```

代价要说清楚：**as-of join 比按天 join 贵一个量级。** 它是不等值 join，Spark 上会退化成 sort-merge
（先按 `user_id, event_time` 全局排序）；1 亿条标签 × 每用户日均 20 个特征版本 = 20 亿行右表，典型是几百 core-hour 一次。
那个 `event_time > label_time − 7d` 的下界不是可选项，它把右表扫描量从"全历史"压到"7 天"。
Databricks 的语义定义可以直接照抄：时序特征表必须声明 timestamp key（`TimestampType` / `DateType`），训练集构造用 AS OF join ——
主键精确匹配 + 取 ≤ 标签时间戳的最新值，**无历史值则返回 null**
（[Databricks: Point-in-time feature joins](https://docs.databricks.com/aws/en/machine-learning/feature-store/time-series)）。

### 3.3 为什么这个错误在离线指标上表现为"指标变好"（本章最重要的一段）

```
普通的 bug：训练集里某列算错了 → 那列信息量下降 → 离线 AUC 下降 → 你当场发现
穿越     ：训练集里某列混进了未来 → 那列信息量上升 → 离线 AUC 上升 → 你当场庆祝
                                    ↑
                          验证集也来自同一份穿越数据，
                          所以交叉验证、holdout、早停 —— 全都通过
```

**离线评估无法证伪穿越，因为验证集和训练集共享同一个穿越管道。** 你唯一能拿到的信号方向是"变好"，
而变好正是你在追求的东西 —— 这就是它极难被发现的全部原因。四个能真正抓到它的手段：

| 手段 | 做法 | 能抓到什么 | 抓不到什么 |
|---|---|---|---|
| **单列 AUC 体检** | 每列单独当作打分器算 AUC | 任何单列 AUC > 0.75 都当嫌疑处理 | 多列共同泄漏、弱泄漏 |
| **严格时间外推验证** | 验证集时间**整体晚于**训练集最后一条，中间留 gap ≥ 一个特征窗口 | 按天 join 型穿越 | 特征表本身就是穿越版时无效 |
| **在线-离线回放比对** | 用线上 fetch 日志的 (key, timestamp) 重新回填，逐条 diff（Chronon 模式） | **几乎全部** | 需要先有线上流量 |
| **服务端特征日志当训练集** | 直接消灭"两份值" | **全部**（定义上不可能穿越） | 新特征必须先 shadow 攒数据 |

⚠️ **关于"穿越让离线 AUC 虚高多少"**：流传的"5–20%"来自二手教程，**没有一手实验数据支撑**，
不要拿它去承诺收益或做容量估算 —— 本书调研也**没有找到可引用的一手实验值**，所以这里不给数字，只给量级：
**它足以让一个线上不可用的模型在离线看起来是当前最优**（本章开头那个例子里，AUC 从 0.712 涨到 0.751 而线上 CTR 跌 3.1%）。
在面试里正确的说法就是这句量级判断，而不是报一个具体百分比 —— 报了会被追问出处。

### 3.4 三个变体，比主线更常踩

1. **回填窗口 ≠ 线上窗口。** backfill 用 24 h 聚合窗口而线上推理用 5 min，模型会完全跑偏。这不是 drift，是纯粹的时间戳逻辑错误，
   而且分布监控看不出来 —— 两个窗口各自的分布都很"正常"。
2. **`lookback_window` 只在离线生效。** Databricks 明确：**在线推理永远取最新值**，`lookback_window` 只作用于训练与批量推理（同上文档）。
   "多久算过期"必须**手动**在在线侧配一个 TTL 去对齐，这是一个天然的语义不对称点，没有任何平台会替你对齐。
3. **在线发布模式选错。** snapshot 只存最新值、仅支持按主键查；window 保留全部带时间戳的值 + TTL 过期。
   做"用户过去 N 天行为序列"类特征时选了 snapshot，线上就永远拿不到窗口内的历史（同上）。

> **面试金句**
> "Point-in-time correctness isn't a nice-to-have — it's the difference between a model that works and a model that
> *looks* like it works. And here's the nasty part: leakage only ever moves your offline metric **up**. So your
> validation set can't save you, your holdout can't save you — they're both downstream of the same broken join.
> The only thing that actually catches it is replay: take the exact keys and timestamps we fetched in production,
> re-derive the features offline, and diff them row by row."
> 中文："时间点正确性不是加分项，它决定了模型是真的能用还是**看起来**能用。恶心的地方在于：穿越只会让离线指标**变好**。
> 所以验证集救不了你，holdout 也救不了你 —— 它们和训练集在同一条坏掉的 join 下游。
> 唯一真能抓到的是回放：拿线上真实取数的 key 和时间戳，用离线代码重算一遍，逐行 diff。"

---

## 4. Feature store 到底解决了什么、没解决什么

它其实只有三个部件：

```
        ┌───────────── 统一定义（一份代码 / 一份 YAML）─────────────┐
        │  entity: user   ttl: 7d   window: 24h   source: clicks    │
        └────────┬────────────── 编译成两条执行 ───────────┬────────┘
        ┌────────▼─────────────┐               ┌───────────▼────────┐
        │ 离线存储 offline     │ ─── 物化 ──►  │ 在线存储 online    │
        │ Iceberg/Delta 全历史 │               │ Redis/DynamoDB/PG  │
        │ 带 event_time        │               │ 只存当前值         │
        │ → as-of join 拼训练集│               │ → 按主键毫秒级取   │
        └──────────────────────┘               └────────────────────┘
             ▲ 训练/回填走这边                      ▲ 排序/风控请求走这边
```

| 它真的解决 | 它不解决 |
|---|---|
| **一次定义、两处执行**，消灭"两套实现"这一类 skew（成因 #1） | **执行语义仍然是两套**：同一份 SQL 在批引擎和流引擎下的窗口边界、乱序容忍、水位线都不同（成因 #3 原封不动） |
| as-of join 的正确实现（自己写十有八九漏掉下界或 tie-break） | 数据漂移、标签质量、样本选择偏差 —— 这些不在它的职责里 |
| 物化管道 + 在线 KV 的运维（重试、幂等、批量大小、TTL） | **新鲜度上限**：预物化的在线值按定义就是上一次管道跑完的东西 |
| 特征发现与复用（Uber Palette 托管 **20,000+ 特征**、**5000+ 生产模型**、峰值 **1000 万次实时预测/秒**，[Uber, 2024-05-02](https://www.uber.com/us/en/blog/from-predictive-to-generative-ai/)） | 元数据变更的爆炸半径 —— 见下 |
| 版本化与血缘的挂载点 | 缺失值语义（成因 #4）：`NULL` 还是 `0` 是你在两侧各自写的代码决定的 |

**最贵的教训来自 Uber**：2021 年一次**错误的元数据变更被推上线，直接打挂了 Tier-1 场景的 OnlineServing**。根因是校验放在客户端
而不是服务端、元数据仓库碎片化、更新走全量刷新。整改后元数据更新延迟从 **>1 小时降到 15 分钟**，Cassandra 迁移耗时 **−90%**，
接入部署时间 **−95%**（[Uber: Palette Meta Store Journey](https://www.uber.com/us/en/blog/palette-meta-store-journey/)）。
**结论：特征平台的高危变更面是元数据，不是特征值。** 元数据变更必须服务端校验 + 增量刷新 + 灰度，按生产变更管。

### 生态现状（2026 年中，随时变动）

| 方案 | 状态与关键数字 | 判断 |
|---|---|---|
| **Tecton** | **2025-08-22 被 Databricks 收购**（[公告](https://databricks.com/blog/tecton-joining-databricks-power-real-time-data-personalized-ai-agents)），在线服务重构到 Lakebase Postgres；容量档 CU_1/2/4/8、最多 3 个只读副本；**旧版 online tables 已 "no longer supported"** | 独立厂商赛道基本出清。已在用的要注意"publish 到新在线库后，任何 endpoint 变更（含扩缩容）会自动切源"这个隐式行为 |
| **Feast** | LF AI & Data，约 7.2k stars；2026 上半年月更一个 minor（v0.60→v0.65）。**但成熟度分层要看清**：online/offline store = GA，**On-demand Transformations = Beta，Streaming Transformations / Vector Search = Alpha，Lineage Explorer 未实现**（[Roadmap](https://docs.feast.dev/roadmap)） | 能用，但不要按官网首页的能力清单排期。原生数据质量监控到 v0.64.0（2026-06-13）才有 |
| **Chronon** | Airbnb + Stripe，Apache 2.0；主打 point-in-time 正确性 + **可度量**的在线离线一致性；宣称 **sub-10 ms p99**（Vert.x）。PMC 席位只分给 Airbnb(8) 与 Stripe(5) | 一致性闭环是全场最强的，但治理集中在两家公司，自建能力弱的团队慎入 |
| **SageMaker Feature Store** | 没有被弃用，还在补 API（2026-07 加 `BatchWriteRecord` / `ListRecords`）。⚠️ 别搞混：SageMaker 的 Model Monitor / Clarify **自 2026-07-30 起不向新客户开放**，Feature Store 不在该列表 | 已在 AWS 上的默认选项 |
| **Feathr** | 1.9k stars，**未找到 2025–2026 的公开 release** | 按维护模式对待，不要用于新项目 |
| **自建（Netflix 路线）** | **存 fact 不存 feature**：Axion 只存原始观测事实，特征在需要时用**和线上共享的同一份 encoder 代码**重算，从源头消灭 skew。Iceberg + EVCache 混合后特定访问模式快 **3×–50×**；数据质量体系提前发现 **>95%** 的数据问题（[Netflix TechBlog, 2022-04-26](https://netflixtechblog.com/evolution-of-ml-fact-store-5941d3231762)） | 上限最高、门槛最高。它把"两套实现"换成了"一套代码两处调用"，代价是在线要有算力重算 |

---

## 5. 在线特征的延迟预算：几百个特征怎么在 10 ms 内取完

### 5.1 先立预算

一次排序请求：1 个用户实体 + 200 个候选商品实体，端到端 p99 预算 100 ms。

```
100 ms  端到端 p99 预算
 −20 ms 召回（向量检索 + 规则）
 −30 ms 模型推理（200 候选一批，CPU/GPU）
 −10 ms 业务逻辑、多样性重排、序列化
 −25 ms 余量（GC、网络抖动、尾延迟放大 —— 见 00/01 §7）
─────────
 ≈15 ms 留给特征检索
```

### 5.2 决定延迟的不是特征个数

**本节唯一需要背下来的判断：延迟的驱动量是 feature view 的个数，不是特征的个数。**
10 个特征分散在 5 个 feature view 里 = **5 次串行网络往返**；分散在 10 个里 = 10 次。
**唯一的例外是 Redis：同一个 entity 的所有 feature view 共享一个 hash key，往返永远是 1 次**
（[Feast: Online Server Performance Tuning](https://docs.feast.dev/how-to-guides/online-server-performance-tuning)）。
单个在线存储的分位数（Tecton 官方对比，是公开资料里最可信的一组）：

| 在线存储 | p50 | p99 | p999 | 备注 |
|---|---|---|---|---|
| **Redis** | 600–700 µs | 2.5–3.0 ms | 9.0–12.0 ms | 同 entity 恒 1 次往返；单分片 cache.m5.2xlarge ≈ 18,000 QPS 或 18 GB |
| **DynamoDB** | 3–4 ms | 20–25 ms | 60–120 ms | 最终一致读 = 0.5 RRU/查询；**单 entity key 3000 QPS 硬上限**，响应 2 MiB |
| **PostgreSQL** | 3–10 ms | 未见公开分位 | — | Feast 口径；连接池是主要瓶颈 |

（来源：[Tecton: Select Your Online Store](https://docs.tecton.ai/docs/beta/setting-up-tecton/connecting-an-online-store/connecting-an-online-store-aws/selecting-your-online-store)、Feast 调优文档。）

### 5.3 算例：从 150 ms 砍到 9 ms

| 步骤 | 朴素做法 | p99 | 收敛后的做法 | p99 |
|---|---|---|---|---|
| 用户侧 180 列（6 个 feature view） | 6 次串行 DynamoDB GetItem | 6 × 22 ≈ **132 ms** | 合并成同一个 entity 的 1 个 Redis hash，1 次 HMGET | **2.5–3.0 ms** |
| 商品侧 200 候选 × 40 列 | 逐候选查 200 次 | 不可行 | 1 次 pipeline / MGET，200 key 一批（客户端按 slot 分组，每分片 1 次往返、分片间并行） | **3–5 ms** |
| 用户 embedding（256 维 fp32 = 1 KB） | 每请求取一次 | +2 ms | 会话开始时预取，进程内缓存 TTL 2 h（Meta Mosaic 用的就是 2 h TTL） | **≈0**（命中时） |
| 请求时变换 20 列 | pandas UDF | 8–15 ms | 原生 Python；重变换搬到写入侧预物化 | **1–2 ms** |
| 反序列化 + 组装 | protobuf → dict → DataFrame | 5 ms | 直接填 numpy buffer | **1 ms** |
| **合计** | | **≈150 ms+** | | **≈9 ms** |

四条动作，按 ROI 排序：

1. **收敛 feature view**：把同一个 entity 的特征合进同一个 view / 同一个 hash key。这一条通常就吃掉 80% 的收益。
2. **批量取数**：200 个候选是一次 MGET，不是 200 次 GET。Feast 的 DynamoDB 批量固定 100（API 上限），要调的是 `max_read_workers`。
3. **重变换搬到写入侧**（`write_to_online_store=True`），读路径只做拼装。Feast 官方实测请求时变换用原生 Python 比 pandas
   快 **3–10×**（[Feast blog, 2025-01-14](https://feast.dev/blog/feature-transformation-latency/)）—— 请求时用 pandas
   等于把一个向量化框架的固定开销压到单行请求上。
4. **embedding 走独立通道**：TTL + 多级缓存，不要塞进行式 KV 的通用路径。Meta Mosaic 的用户 embedding 在下游排序模型的
   **20k+ 输入特征中排进前 2%**，服务侧 AOTI + 模型切分降 **56%** GPU 用量、再叠 memcache 累计降 **79%**
   （[arXiv:2607.24015, 2026-07-27](https://arxiv.org/html/2607.24015v1)）。

### 5.4 撞墙条件（这些是硬上限，不是调参能绕过的）

| 约束 | 数值 | 撞墙的信号 |
|---|---|---|
| 单请求从在线存储取回的字节 | ≤ **2 MB**（Tecton SLO 资格线，[文档](https://docs.tecton.ai/docs/monitoring/online-serving)） | 算一遍：200 候选 × 40 列 × 8 B = 64 KB，很安全；但**把用户 1000 长度的行为序列（8 KB）跟着每个候选重复带一遍 = 1.6 MB**，当场撞线 |
| Redis 查询耗时 | ≤ **25 ms**（超过不算 SLO-eligible）；后端超时 **2 s** | p999 超过 12 ms 就该看大 key 和分片倾斜了 |
| DynamoDB 单 entity key | **3000 QPS** 硬上限 | 爆款商品 / 头部商户会稳定撞上，必须像热 key 一样单独处理（[01/02 §4](../01-building-blocks/02-caching.md)） |
| 成本拐点 | 100K 读写 QPS：Redis ≈ **$76,404/年** vs DynamoDB ≈ **$788,476/年**；500 GB 数据集：DynamoDB ≈ **$9,329/年** vs Redis ≈ **$305,619/年**（Tecton 估算） | **高 QPS 选 Redis，大数据量选 DynamoDB。** 两者都要的场景就分层：热实体 Redis、长尾 DynamoDB |

规模上限的参照物：DoorDash 每秒从 Redis 读取数千万个 key/value，只读负载下单次查找约 **1.9 µs**，把扁平 KV 改成 Redis Hash 显著降 CPU
（[DoorDash, 2020-11-19](https://careersatdoordash.com/blog/building-a-gigascale-ml-feature-store-with-redis/)；
后续"客户端缓存再提 70%"的说法来自标题与检索摘要，**未能直接核实原文**）。
⚠️ 这一整节讲的是**行式 KV 取数**；LLM 服务侧那套"缓存"（KV cache 复用、continuous batching —— 复用的是模型算出的中间张量，
不是特征值）是完全不同的问题，见 [04/01 §2 §3](../04-ai-agent-systems/01-llm-serving-infra.md)。

---

## 6. 新鲜度分级：只给真正影响决策的特征付实时的钱

| 档 | 新鲜度 | 典型实现 | 相对单位成本 | 该放什么特征 | 撞墙条件 |
|---|---|---|---|---|---|
| **日级批** | 6–24 h | Spark 日批 → 物化到 KV | **1×** | 人口属性、长期偏好、商品静态属性、90 天窗口聚合 | 促销首日、冷启动实体上完全失效 |
| **分钟级流** | 1–10 min | 微批 Structured Streaming | 5–15× | 近 1 h 的曝光/点击计数、会话级统计 | **微批延迟本身就在制造在线/离线不一致** |
| **秒级 / 亚秒** | < 1 s | Flink；或 Spark Real-Time Mode（**GA 2026-03-19**，端到端最低 5 ms，内测 p99 数毫秒~300 ms，[公告](https://www.databricks.com/blog/announcing-general-availability-real-time-mode-apache-spark-structured-streaming-databricks)） | 20–50× | 反欺诈的会话内行为、库存、风控速率限制 | 状态大小与 checkpoint 成本；长窗口不可能纯流式重放 |
| **请求时计算** | 0（= 源延迟） | on-demand transformation | 读路径 CPU | 只有请求参数才能算的：当前距离、距上次点击秒数、query × item 交叉 | p99 直接进关键路径，且这段代码在训练侧必须一模一样地跑一遍 |

**Coinbase 的实测把第二档的问题说透了**：用 Spark Real-Time Mode 把特征计算端到端延迟从 **800–900 ms 降到 100–250 ms**
（无状态 150 ms / 有状态聚合 250 ms），**p99 < 100 ms**，统一引擎计算 **250+ 个特征**，
**在线/离线特征一致性从明显受损提升到 99%**，年度计算成本估算 **−51%**
（[Databricks Coinbase 案例](https://www.databricks.com/customers/coinbase/lakeflow)，客户站台口径）。
**因果关系要读对：不是"实时了所以准"，而是微批的那 800 ms 延迟本身就是不一致的来源** ——
线上在 t 时刻取到的是 t−800ms 的聚合，离线回填算的是 t 时刻的精确聚合，两者天然对不上。

**长窗口只能 lambda**：批任务每日算好上传 KV **播种初值**，流式只在批次之间做增量 ——
90 天窗口不可能重放全部原始事件（Chronon 的做法，同上 Airbnb 博客）。Pinterest 的实时用户信号服务同理：摄入 **1.2M events/s**，
端到端 **p99 ≈ 10 秒**，事件存储只留约 7 天、靠增量计算支撑 90 天窗口；**有状态聚合把服务端 p99 从 ~100 ms 降到 < 35 ms**，
单 signal **50k QPS**（[Pinterest, 2019-12-05](https://medium.com/pinterest-engineering/real-time-user-signal-serving-for-feature-engineering-ead9a01e5b)）。

**分级判据（一条就够）**：这个特征的值在决策窗口内**会不会变到改变决策**？用户的 30 天购买力不会（日批）；
这个设备 60 秒内试了 12 张卡会（秒级）。把所有特征都升到秒级，你付 20–50× 的成本，换来的是多出来的几位有效数字。

---

## 7. 版本化、监控、下线

**版本化的对象是"定义 + 执行语义"，不是只有定义。**
只统一定义、不统一执行时机与窗口语义，仍然会 skew ——
> "Feature definitions match. Execution semantics do not. That gap is where skew lives."
> （[dev.to, 2025-01-21](https://dev.to/synapcores/why-feature-stores-didnt-fix-training-serving-skew-fad)）

落地三条：① 特征引用必须带版本 `user_ctr_7d@v3`；② 训练产物必须记录它用到的 feature spec 版本集合，否则模型无法复现；
③ 改窗口、改缺失值填充、改上游源 = **新版本**，不是原地改 —— 原地改会让所有历史训练集失去可解释性。
参考现实：Feast 到 **v0.62.0（2026-04-08）** 才有 FeatureView 版本追踪与血缘图版本标记
（[Releases](https://github.com/feast-dev/feast/releases)）—— 这类能力不是"买了平台就有"。

**监控的最小指标集**（跨来源共识）：

| 指标 | 口径 | 动作线 |
|---|---|---|
| 物化滞后 freshness lag | `now − max(已物化 event_time)` | > 2× 调度周期告警。这是唯一能提前发现"在线值不再更新"的指标 |
| 缺失率 null rate | 按 feature × 天，**训练侧和服务侧分开算** | 两侧数字不同 = 成因 #4 命中 |
| 在线/离线精确匹配率 | 按 `request_id` join 逐列比 | **< 99% 就当故障处理**（Coinbase 达标线是 99%） |
| 分布距离 | 数值 JS divergence / 类别 L∞，默认阈值 0.3 | 只作兜底；能逐条比就别比分布 |
| 在线服务 p50/p99/错误率 | 按 feature view 分组 | 定位是哪个 view 拖慢了整体 |

注意基数：按 `feature × 天 × 模型版本 × 机房` 打标签会让指标基数指数膨胀，而基数是可观测性的唯一成本变量
（[05/02 §2](../05-reliability/02-observability.md)）。实践上按 feature view 聚合，逐列比对走离线批任务。

**下线（这是最容易出事的一步）**：

```
标记 deprecated → CI 门禁阻止新引用 → 观察 N 周零引用 → 停止物化 → 在线数据留 M 天（可回滚）→ 删表
                                                        ▲
   ⚠️ 危险全在"停止物化"这一步：停了之后，在线 KV 里的旧值**仍然可读**，模型不报错，
      只会安静地吃一份永远不变的陈旧特征。停物化必须同时让在线读**返回 NULL 或直接报错**。
```

Feast 的 Lineage Explorer 至今仍是"计划中"（同上 Roadmap），所以"谁在用这个特征"多数团队要自己从训练代码里扫。
没有这个能力就不要下线特征 —— 留着一列不用的特征成本是每天几块钱的物化，删错一列成本是一次线上事故。

---

## 8. 2026 年还需不需要一个独立的 feature store

**反方（不需要）**：Xebia《You Still Don't Need A Feature Store》（更新于 **2026-01-28**）主张多数场景用模型内预处理 +
共享 transform 函数 + 普通 KV + 纯代码定义的 feature catalog 就够了，作者花了 **6 个月**重构一个过度工程化的自建 Feast 实现
（[Xebia](https://xebia.com/blog/you-still-don-t-need-a-feature-store/)）；Chalk 的**新鲜度天花板论**指出预物化的在线值
"按定义就是陈旧的"，无论怎么调参都受批次调度约束（[Chalk](https://chalk.ai/blog/why-feature-stores-have-freshness-ceiling)）；
再加上 §7 那句"skew 活在执行语义里" —— 统一定义本身买不到你想要的东西。

**正方（仍需要）**：连 Xebia 自己也让步 —— **低延迟在线服务 + 流式更新（风控、动态定价、推荐）、多团队共享受治理的定义、
skew 来自有状态实时特征**，这三类场景下离线+在线 feature store 站得住。规模本身也是论据：Uber Palette 的
20,000+ 特征 / 5000+ 生产模型 / 1000 万次预测每秒，以及 Chronon 的用户名单（Airbnb、Stripe、Netflix、OpenAI、Uber、Monzo），
说明独立特征平台在高价值场景仍在扩散。

**我的判断**：**"独立采购的 feature store 产品"在收缩，"feature store 的能力"在扩散。**
2026 年更常见的形态是 lakehouse（Iceberg/Delta）+ 托管在线 KV/Postgres + 流引擎 + 一套时点拼接与一致性校验的**代码约定**。

```
立项判据（三条必须全中，缺一条就大概率是净负债）
  (a) 在线路径要在 < 50 ms 内拿到跨多个实体的特征
  (b) 同一批特征被 ≥ 3 个团队 / 模型复用
  (c) 存在有状态的时间窗口聚合（不只是维表 join）
```

三条不全中，正确答案是 **Iceberg 表 + 一份共享的 transform 函数 + Redis + 一个逐条比对的定时任务** ——
这四样一周能搭出来，而一个自建 feature store 的中位数交付周期是季度级。

---

## 9. 什么时候不要这么做（反模式）

| 反模式 | 为什么错 | 正确做法 |
|---|---|---|
| 按 `dt` join 特征快照造训练集 | 每条样本都看到了当天之后的行为，这是穿越的头号来源 | as-of join，且必须带下界（§3.2） |
| 数据集**随机**切分 train/valid | 未来的样本进了训练集，等价于全局穿越 | 按时间切，验证集整体晚于训练集，中间留 ≥ 一个特征窗口的 gap |
| 离线特征生成里出现 `current_date()` / wall-clock | 回填出来的值和当时真实可得的值不同，回填一次结果就变一次 | 一律用事件时间；回填必须可重放且幂等 |
| 只统一定义、不统一执行语义 | 同一份 SQL 在批引擎和流引擎下窗口边界不同，skew 原封不动 | 定义 + 执行语义一起版本化，并上逐条比对流水线 |
| 所有特征一刀切上实时 | 20–50× 成本换绝大多数特征多出来的几位有效数字 | 按"值会不会变到改变决策"分三档（§6） |
| 只做 drift 监控 | drift 拿今天和上个窗口比，对"每天都一样地错"完全无感 | skew 必须单独监控，基线是训练集 |
| 主键是 DATE/TIMESTAMP 却不声明为 timeseries 列 | Databricks 上 `create_training_set()` / `publish_table()` 直接失败 | 主键转 STRING，时间列的角色必须唯一 |
| 把 embedding 塞进通用行式 KV 路径 | 1 KB × 200 候选 = 200 KB/请求，且它的正确管理方式是 TTL 不是流式 | 独立通道 + 多级缓存 + TTL（Meta 用 2 h） |
| 迁移在线存储时直接切流 | 新旧管道"看起来一样"和"逐条一样"是两回事 | 双读比对：event-level + sequence-level 都过了才切（Pinterest 模式） |
| 元数据变更当配置改 | Uber 的 Tier-1 事故就是这么来的 | 服务端校验 + 增量刷新 + 灰度，按生产变更管 |
| 团队 < 10 人、1 个模型就自建 feature store | Xebia 记录的真实结局是原作者弃坑、6 个月重构 | 先用共享 transform 函数 + Redis，撞到 §8 三条判据再说 |

⚠️ 还有一条不在表里但更根本的：**不要在没有服务端特征日志的情况下讨论"要不要上 feature store"** —— 没有日志，你连自己现在有没有 skew 都不知道，这个决策没有输入。

---

## 这一章的三句话

1. **离线指标能证伪几乎所有错误，唯独证伪不了穿越 —— 因为穿越只会让它变好。** 所以任何"离线大涨"的实验都应该先被当作嫌疑
   而不是成果；能证伪它的只有一件事，就是拿线上真实的 key 和时间戳把特征重算一遍逐条 diff。
2. **feature store 统一的是定义，skew 活在执行语义里。** 同一份 SQL 在两个引擎、两个时钟下跑出来的东西可以完全不同，
   所以"上了平台就没有 skew 了"这句话在 Uber 那次 Tier-1 事故里就已经被证伪 —— 那次的根因甚至不是特征值，是元数据。
3. **在线特征的延迟预算按 feature view 的个数算，不按特征个数算。**
   200 个特征收在一个 Redis hash 里是 1 次往返 ≈3 ms（Redis p99 2.5–3.0 ms）；
   同样这 200 个特征分散在 10 个 view 里，走 DynamoDB 就是 10 次串行往返 ≈200 ms（p99 20 ms/次），
   走 Redis 也要 ≈30 ms —— 无论哪个存储，**决定你能不能进 10 ms 的从来不是你取了多少列，是你往返了几次**。

---

## 面试官会追问

1. 离线 AUC 涨了 0.04，上线 CTR 跌 3%。给我一个**一周内能定位到具体哪一列**的排查顺序。
2. 什么是 point-in-time correctness？给我写出正确的 as-of join SQL，并说清那个时间下界为什么不能省。
3. 为什么穿越在离线评估里表现为指标变好？既然如此，交叉验证和 holdout 为什么都救不了你？
4. 训练服务偏斜有哪几类成因？对每一类，给我一个**能报警的指标**，而不是"人工排查"。
5. skew 和 drift 有什么区别？只做 drift 监控会漏掉什么？
6. 一次排序请求要取 500 个特征、200 个候选，端到端 p99 预算 100 ms。你的特征检索预算是多少，怎么花？为什么"特征个数"不是驱动量？
7. 你要下线一个特征。完整流程是什么？停止物化那一步最危险的地方在哪？
8. 什么情况下你会立项做独立 feature store，什么情况下你会明确反对？给三条判据。

---

**下一篇** → [07-model-quality-and-experimentation.md](07-model-quality-and-experimentation.md)：特征对齐之后，轮到"离线指标到底能不能信"——离线评测、在线实验与影子部署。
