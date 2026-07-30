# 05 · 发布工程（release engineering）

> 发布工程的全部内容可以压缩成一句话：**任何时刻，生产上必须能同时跑新旧两个版本的代码，且数据库对两者都是合法的。**
> 这一条推导出后面所有规则。做不到这一条的团队，回滚（rollback）按钮是假的。

---

## 1. 部署 ≠ 发布

```
构建 (build)  →  部署 (deploy)  →  发布 (release)  →  下线旧版 (retire)
   代码变成      新代码进生产      用户开始看到       老代码删除
   制品          （但被 flag 关着）  新行为
```

**解耦（decoupling）这两件事是现代发布工程的地基。** 部署由工程按 CI 节奏做，发布由产品/运营按业务节奏做。它们解耦之后：

- 回滚不再需要重新部署（翻 flag，秒级）
- 高风险变更可以先在生产环境静默运行数天再打开
- 发布窗口不再需要"全公司周四晚上"

**度量**（DORA 四指标，用来判断你处在哪一档）：

| 指标 | 精英档 | 低效档 | 说明 |
|---|---|---|---|
| 部署频率（deployment frequency） | 按需，每天多次 | 每月~每半年 | |
| 变更前置时间（lead time for changes） | < 1 天 | > 1 个月 | commit → 生产 |
| **变更失败率（change failure rate）** | 5–15% | 40–60% | **最重要的一个**，它决定你敢不敢加速 |
| 恢复时间（time to restore / MTTR） | < 1 小时 | > 1 周 | 自动回滚能把它压到分钟级 |

> 如果变更失败率 > 30%，先别谈提高部署频率 —— 你只会更快地制造事故。先把金丝雀门禁（canary gate）和 schema 纪律做好。

---

## 2. Feature Flag：分类、债务、一致性

### 2.1 四类 flag，生命周期完全不同

| 类型 | 目的 | 生命周期 | 求值频率 | 谁改 | 删除纪律 |
|---|---|---|---|---|---|
| **Release toggle** | 隐藏未完成功能 / 渐进放量（progressive rollout） | **天~周** | 部署或会话级 | 工程 | **发布完成后 2 个迭代内必须删** |
| **Experiment toggle** | A/B 实验 | 天~周 | 每请求（按用户稳定分桶 bucketing） | 数据/产品 | 实验结束即删 |
| **Ops toggle / kill switch** | 降级（degradation）、熔断（circuit breaking）、关掉重功能 | **月~永久** | 运行时立即生效 | 值班工程师 | 长期保留，但要定期演练 |
| **Permission toggle / entitlement** | 按套餐/租户开功能 | **永久** | 每请求 | 产品/销售 | 不是 flag，是**授权模型的一部分** |

**最重要的一条**：把 entitlement（谁买了什么）从 flag 系统里搬出去，放进授权/计费模型。否则你的 flag 系统会变成一个没有 schema、没有审计、没有一致性保证的第二套授权系统。

### 2.2 Flag 债务是真实的成本

```
n 个可独立开关的 flag  ⇒  2^n 条代码路径  ⇒  你实际测过的：2 条
```

**强制纪律**（用 CI 检查，不是靠自觉）：

1. 创建 flag 必须填 `owner` + `expires_at` + `type`。
2. 过期的 release toggle 自动开工单，超期两周后 CI **阻断合并**。
3. 仓库内 release toggle 总数设上限（比如 20），超了就得先还债。
4. 死代码（dead code）检测：flag 已 100% 打开且稳定 30 天 → 自动生成"删除分支"的 PR。

### 2.3 求值（evaluation）的一致性与性能

**分桶必须稳定且去相关（decorrelated）**：

```python
def is_enabled(flag_key, entity_id, rollout_bps):        # bps = 万分比
    h = xxhash64(f"{flag_key}:{entity_id}")              # ⚠️ flag_key 必须参与哈希
    return (h % 10_000) < rollout_bps
```

⚠️ **不把 `flag_key` 放进哈希是高频 bug**：那样所有 flag 的"前 10%"是同一批用户，实验之间产生相关性，且这批倒霉用户会同时踩到所有新功能。

**一次请求内必须求值一次**：在入口（网关/BFF）求值，把结果放进请求上下文往下传。让每个下游服务各自求值，会因为规则集推送延迟（propagation delay，通常 1–30 秒）导致**同一请求内前后不一致** —— 服务 A 认为开了、服务 B 认为关了，产生的数据状态是任何一个版本的代码都没预期过的。

**性能与故障模式**：
- SDK 必须**本地求值（local evaluation）**：规则集推送到进程内，求值 < 1 µs。**绝不在请求路径上做网络调用**。
- flag 服务不可用时：SDK 用最后一次缓存的规则集；再不行用**硬编码的代码内默认值**。
- **默认值必须等于"当前生产行为"**，不是"新行为"。这条规则让 flag 服务的故障变成 no-op 而不是一次意外发布。

### 2.4 Kill switch 的三个硬要求

1. **通路独立**：不能依赖正在故障的那个系统（典型翻车：kill switch 的配置存在你正要关掉的服务背后的数据库里）。
2. **生效时间 < 30 秒**：推送（SSE/长轮询 long polling）而非 5 分钟轮询。事故中的 5 分钟是很长的时间。
3. **定期演练（game day）**：每季度真的拉一次。没演练过的 kill switch 在事故当天有很高概率是坏的。

---

## 3. 渐进式交付（progressive delivery）

| 手段 | 回滚速度 | 资源成本 | 能测出什么 | **测不出什么** |
|---|---|---|---|---|
| **金丝雀（canary）** | 秒~分钟 | +5% | 真实流量下的错误率、延迟、资源 | 低频路径、长尾（long-tail）租户、需要数小时才显现的泄漏 |
| **蓝绿（blue-green）** | 秒（切流） | **2×** | 全量切换的兼容性 | 渐进信号（要么全好要么全坏） |
| **影子流量（shadow traffic）** | N/A（不影响用户） | +1× 计算 | 性能、响应差异 | **写路径**（有副作用，除非做完整的副作用隔离） |
| **Ring / 按租户放量** | 分钟~天 | 低 | 租户维度的爆炸半径（blast radius）、企业客户的定制路径 | 需要租户路由能力 |
| **按 cell 部署** | 分钟 | 低 | 单元级隔离，最坏影响 ≤ 1/N 流量 | 需要 cell 架构（见 [01-control-plane.md](01-control-plane.md)） |

**默认组合**：金丝雀（自动门禁）→ ring 放量（内部 → 小租户 → 大租户）→ 全量。蓝绿只在"必须原子切换"（比如换了存储引擎）时用，因为它的 2× 资源成本和"没有中间信号"是真实缺点。

### 3.1 金丝雀的自动回滚判据清单

**四个前置条件**（不满足就不要开自动回滚，会误伤）：

- [ ] 变更是**无状态可回滚**的：schema 处于 expand 之后、contract 之前
- [ ] 有**并发运行的基线组（concurrent baseline group）**（同时段的老版本流量），**不是**"和昨天同一时刻比"
- [ ] 样本量门槛：canary 累计 ≥ **1,000 个请求**且 ≥ **5 分钟**，两者取后到者；不满足就不判定
- [ ] 回滚动作本身经过演练，且回滚后**自动冻结流水线（freeze the pipeline）**（否则下一个人会把同样的变更再推一遍）

**门禁信号**（任一触发即回滚）：

| # | 信号 | 阈值示例 | 判定窗口 | 说明 |
|---|---|---|---|---|
| 1 | HTTP 5xx / gRPC 非 OK 率 | canary > baseline × **1.5** 且绝对值 > 0.1% | 5 min | 双条件避免低基数误报 |
| 2 | 关键路径 p99 延迟 | canary > baseline × **1.2** | 5 min | 用 p99 不用均值 |
| 3 | **新错误指纹（error fingerprint）** | 出现 baseline 中不存在的 exception 堆栈指纹 | **立即** | 最灵敏的信号 |
| 4 | 下游依赖错误率 | > baseline × **2** | 5 min | 抓"新版本把下游打挂了" |
| 5 | 饱和度（saturation） | CPU / 内存 / 连接池 / 队列深度（queue depth）> **85%** | 5 min | 抓资源泄漏 |
| 6 | **业务 KPI** | 关键转化率 < baseline × **0.95** | 15 min | 抓"技术指标全绿但功能坏了" |
| 7 | SLO 燃尽率（burn rate） | 1 小时窗口 burn rate > **14.4** | 立即 | 见 [05-reliability/01](../05-reliability/01-slo-and-error-budget.md) |

**分阶段与 bake time**：`1% (10min) → 5% (30min) → 25% (1h) → 50% (2h) → 100%`。
低频路径（月结、导出、Webhook 重试）在 1% 阶段可能一次都没被执行 —— 对这类变更，bake time 必须按**业务周期**而不是时钟设定。

**三个反模式**：
- **用 CPU/内存做主门禁**：滞后且与用户体验不相关。门禁必须是 SLI。
- **用"昨天同期"做基线**：周期性、外部事件、其他团队的发布全都会污染它。**必须是并发基线组。**
- **回滚后不冻结**：事故会在 40 分钟后原样重演。

---

## 4. 数据库 Schema 迁移六步法（expand–migrate–contract）

**这是发布工程里最容易出人命的一节。** 根本原因：滚动发布（rolling deploy）期间新旧代码同时在跑，数据库必须同时对两者合法。

### 4.1 六步表

| 步 | 名称 | 操作 | 读 / 写状态 | 可回滚性 | 危险信号 | 典型时长 |
|---|---|---|---|---|---|---|
| **1** | **Expand** | 加**可空**列 / 新表 / 新索引；**不加约束、不加默认值（除非是常量）** | 老代码完全不知道新结构 | ✅ 完全可回滚 | DDL 卡在锁队列 | 分钟 |
| **2** | **Dual write** | 新代码同时写新旧两处 | **写双，读老** | ✅ 关 flag 即回滚 | 双写不一致率 > 0 | 1 次发布 |
| **3** | **Backfill** | 分批回填历史数据（**幂等 + 可重跑 + 限速 throttling**） | 写双，读老 | ✅ 新列是纯冗余 | 复制延迟（replication lag）上升、锁等待、批次超时 | 小时~天 |
| **4** | **Verify** | 影子读（shadow read）+ 逐行 diff（抽样或全量） | 写双，读老，**旁路比对新** | ✅ 完全可回滚 | diff 率不收敛到 0 | 1~7 天 |
| **5** | **Switch read** | flag 控制读新 | 写双，**读新** | ✅ **秒级回滚**（翻 flag） | 切读后错误率/空值率 | 1~7 天 bake |
| **6** | **Contract** | 停写老、加约束、删列/删表 | 只用新 | ❌ **不可逆** | —— | 在第 5 步之后再等 ≥1 个完整回滚窗口 |

**四条纪律**：
1. **每一步是一次独立发布**，不能合并。合并第 1、2 步是最常见的翻车方式（老实例还在跑时新列已被写入且被加了约束）。
2. **第 6 步与第 5 步之间至少隔 1–2 周**，覆盖"所有老版本实例都已下线"+"一个完整的回滚窗口"。
3. **Backfill 必须幂等**：用 `WHERE new_col IS NULL LIMIT 1000` 分批，记录进度，随时可中断可重跑。**限速**到复制延迟 < 1 秒。
4. **删列前先撤权限或改名观察**：`ALTER TABLE ... RENAME COLUMN x TO x_deprecated_20260801`，跑一周没人报错再删。

### 4.2 在线 DDL（online DDL）实操（Postgres）

```sql
-- ⚠️ 这两行是强制项，不是建议
SET lock_timeout = '2s';        -- DDL 拿不到锁就快速失败，外层重试
SET statement_timeout = '0';    -- 但 DDL 本身不要被 statement_timeout 砍断

-- ① 加列
ALTER TABLE orders ADD COLUMN region text;                        -- 瞬时（只改 catalog）
ALTER TABLE orders ADD COLUMN tier text DEFAULT 'free';           -- PG11+ 瞬时
ALTER TABLE orders ADD COLUMN uid uuid DEFAULT gen_random_uuid(); -- ❌ volatile 默认值 → 全表重写

-- ② 加索引：必须 CONCURRENTLY（不阻塞写，代价是耗时 2–3×，且失败会残留 INVALID 索引）
CREATE INDEX CONCURRENTLY idx_orders_region ON orders (region);
-- 失败后清理：SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
--             DROP INDEX CONCURRENTLY <name>;

-- ③ 加 NOT NULL：直接 SET NOT NULL 会全表扫描并持 ACCESS EXCLUSIVE
ALTER TABLE orders ADD CONSTRAINT orders_region_nn
      CHECK (region IS NOT NULL) NOT VALID;                       -- 瞬时，只对新写入生效
ALTER TABLE orders VALIDATE CONSTRAINT orders_region_nn;          -- 只取 SHARE UPDATE EXCLUSIVE
ALTER TABLE orders ALTER COLUMN region SET NOT NULL;              -- PG12+ 复用已验证的 CHECK，跳过全表扫

-- ④ 加外键：同样 NOT VALID → VALIDATE
ALTER TABLE orders ADD CONSTRAINT fk_tenant
      FOREIGN KEY (tenant_id) REFERENCES tenants(id) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT fk_tenant;

-- ⑤ 改类型：ALTER COLUMN TYPE 重写整表 + 持 ACCESS EXCLUSIVE 全程
--    ⇒ 大表上等同于一次计划外停机。正确做法就是走六步法（新列 → 双写 → 回填 → 切读 → 删老列）
```

> **Staff 级的那个细节：锁队列（lock queue）。**
> `ALTER TABLE` 需要 ACCESS EXCLUSIVE 锁。它在**等锁**的时候，所有后到的查询（包括 `SELECT`）都会排在它后面 —— Postgres 的锁队列是 FIFO。所以一个"瞬时"的 DDL 撞上一个跑了 30 秒的分析查询，会造成 **30 秒的全表不可用**，且监控上看起来像"数据库突然挂了"。
> `lock_timeout = '2s'` + 外层指数退避（exponential backoff）重试是唯一正确的做法。这一条比本节其他所有内容加起来更常救命。

**MySQL 侧**：`ALGORITHM=INSTANT` 覆盖加列等一部分操作；其余用 [gh-ost](https://github.com/github/gh-ost)（基于 binlog，无触发器，**可暂停、可限流、可随时中止**）或 `pt-online-schema-change`（基于触发器，写放大 write amplification 更大）。选 gh-ost。

### 4.3 回滚策略的分层现实

| 回滚对象 | 可行性 | 手段 | 时间 |
|---|---|---|---|
| **代码** | 总是可行 | 镜像回退 / flag | 秒~分钟 |
| **Schema（expand 阶段）** | 可行 | 删掉新增的可空列/索引 | 分钟 |
| **Schema（contract 之后）** | ❌ 不可行 | 只能**前滚（roll forward）** | —— |
| **数据（已被新代码写坏）** | ❌ 几乎不可行 | 从备份 PITR（会丢其他数据） | 小时 |

⇒ **所有的发布纪律本质上都是在保护"代码回滚"这一条永远可用。** 一旦某次发布让数据进入了老代码无法解释的状态，你就失去了回滚能力。

---

## 5. 多版本共存与向后兼容（backward compatibility）契约

滚动发布 = 新旧版本同时在线。所以**每次发布必须与上一版兼容（N-1 兼容）**。

| 变更 | 兼容性 | 说明 |
|---|---|---|
| 加可选字段 | ✅ 兼容 | 前提：读方是 Tolerant Reader（忽略未知字段） |
| 加必填字段 | ❌ breaking | 老写方不会填 |
| 删字段 | ❌ breaking | 先标 deprecated，观察调用方，再删 |
| 改字段类型 | ❌ breaking | 走新字段 + 六步法 |
| **改字段语义（semantic change）**（单位从"分"变"元"） | ☠️ **最危险** | 编译期、schema 校验、契约测试**全都发现不了**。改语义必须改名字。 |
| 收紧校验（原来接受空串，现在拒绝） | ❌ breaking | 属于行为变更，要走 flag |
| 放宽校验 | ✅ 兼容 | |

**三条落地手段**：
- **Protobuf**：删字段必须 `reserved` 字段号和名字，**永不复用 tag**。
- **事件 schema**：事件是不可变的，历史事件永远是老 schema ⇒ 消费者必须能处理所有历史版本。用 schema registry 强制 `BACKWARD` 兼容性检查（新 schema 能读旧数据）。
- **契约测试（contract testing）**：消费者驱动的契约（consumer-driven contract，Pact 类）放进 CI，让"我改了一个字段"在合并前就被下游的契约挡住。

API 版本化的完整讨论见 [02-architecture-patterns/04-api-design-and-versioning.md](../02-architecture-patterns/04-api-design-and-versioning.md)。

---

## 6. 发布的可观测性

**没有部署标记的监控系统，事故排查会浪费掉 80% 的时间。**

必须做的三件事：

1. **部署标记（deployment marker）**：每次部署往指标系统打一条带 `service / version / commit_sha / actor` 的注解事件。所有 dashboard 自动画竖线。
2. **`version` 作为一等指标维度（first-class metric dimension）**：所有 RED 指标（Rate/Error/Duration）带 `version` 标签，这样金丝雀对比是一次查询而不是一个项目。
3. **告警自动关联最近变更**：告警触发时，自动附上"过去 60 分钟内该服务及其直接依赖的所有部署与 flag 变更"。这一条能把 MTTR 砍掉一半。

**自动化的变更前后报告**：每次部署完成后 30 分钟自动生成 diff 报告（错误率、p50/p99、成本、关键 KPI），推到部署 PR 的评论里。它的价值不在于告警（金丝雀已经做了），而在于**让缓慢劣化（gradual degradation）被看见**。

---

## 7. AI 系统的发布：五个可版本化对象

传统发布只有"代码"一个可变对象。AI 系统有五个，其中两个经常被忘掉：

```yaml
# release manifest —— 这五项缺一不可，且必须一起版本化、一起回滚
release: agent-v37
model:      claude-opus-5-20260514        # ⚠️ 固定 snapshot，禁止用 latest 别名
prompt:     sys/agent@v12                 # git sha 可解析
tools:      tools/manifest@sha256:9f3a…   # 哈希 pin（同时防 MCP 侧 rug pull）
retriever:                                # ← 最常被忘的一项
  index: corpus-2026-07-15
  embed: voyage-context-4                 # 换 embedding 模型 = 全量重建索引
sampling:   {temperature: 0.2, max_tokens: 8192}   # ← 第二常被忘的一项
eval_gate:  regression-suite@v9           # 通过率不得低于生产基线 2pp
```

### 7.1 工具定义变更是最贵的一次发布

Anthropic 的 prompt cache **失效级联（invalidation cascade）顺序是 `tools → system → messages`**：

| 你改了什么 | 失效范围 | 后果 |
|---|---|---|
| **tool 定义** | **全部失效** | 全量 cache miss |
| system prompt | system + messages | 大部分失效 |
| `tool_choice` / 增删图片 | 仅 messages | 影响小 |

缓存读价格约为标准输入价的 **10%**（三家一致，2026 年中量级，随时变动）⇒ **一次工具定义变更会让输入成本在缓存重建期间瞬间涨到 10 倍**，同时 TTFT 显著恶化。

**工程结论**：
- 工具定义变更要**错峰发布（off-peak rollout）**（低谷期），并盯住 cache hit rate 指标直到恢复。
- 把工具定义拆成"稳定核心 + 可变扩展"没用 —— 缓存前缀是**逐 token 精确匹配**的，任何一个字节的变化都全废。
- 所以：**把工具定义的发布节奏和提示词的发布节奏分开**，前者按周甚至按月，后者可以按天。

### 7.2 "提示词是代码还是配置"——工程结论

**提示词是代码。** 它决定行为、需要 review、需要版本、需要回归测试、需要回滚。SOC 2 CC8.1（变更管理）也不会接受"在后台文本框里直接改生产行为且无审批"。

但它需要配置的**分发速度**（不重建镜像就能改）。

```
唯一真相源 = git 仓库
分发通道   = 配置系统 / 对象存储（GitOps：合并即分发）
运行时开关 = 只允许在 git 里预先定义好的枚举值之间切换
```

**允许的运行时动作**：切到某个已发布版本、关掉某个工具、切到保守提示词（kill switch）。
**禁止的运行时动作**：在生产后台自由编辑提示词文本。这是无 review 的生产代码变更。

### 7.3 回归评测门禁（regression eval gate）

**门禁的形态不是"必须 ≥ X 分"，而是"不得比当前生产基线退化超过 2pp，且不得在任何一个已知失败类别上出现新增失败"。** 绝对分数会诱导你去优化基准，相对退化才反映真实风险。

| 要求 | 数值 | 来源/理由 |
|---|---|---|
| CI eval 数据集规模 | **100+** 精选样本 | 来自**生产轨迹（production traces）**，不是公开 benchmark |
| 初次错误分析 | **20–50 条** trace；后续每轮 **≥100 条** | 饱和判据：连续约 20 条无新失败类别 |
| 错误分析节奏 | 活跃期每 **2–4 周**一轮；间隔期每周看 10–20 条 | |
| 失败模式分类法（failure taxonomy）规模 | **5–10 类**；手工 open-code 30–50 条后再让 LLM 辅助聚类 | |
| LLM-as-judge 上线门槛 | 与人工的 **Cohen's kappa ≥ 0.6 可上线，≥ 0.8 强**（人-人基线 0.5–0.8） | |

⚠️ **公开 benchmark 不能作为验收门禁**。Berkeley RDI 在 2026-04 的工作里把 **8 个主流 Agent 基准中的 7 个刷到 ~100%**：10 行 `conftest.py` 解决 SWE-bench Verified 全部实例；一个假的 `curl` wrapper 拿下 Terminal-Bench 全部 89 个任务。**基准分数是营销素材，回归集才是门禁。**

⚠️ **judge 有系统性偏差（systematic bias），且换 judge = 换基线**。已观测到的量级：**风格偏差（style bias）0.10–0.76，位置偏差（position bias）≤0.04**（风格偏差是位置偏差的十倍以上）；冗长偏好方向**因模型而异**（部分模型偏好长回答 +0.24~+0.44，Claude 系偏好简洁 −0.12）。⇒ judge 模型也要写进 release manifest，换 judge 必须重新校准（recalibrate），**不同 judge 的分数不可横向比较**。

### 7.4 模型下线与迁移

**换模型不是换一个字符串。** 一次模型迁移至少要重算四件事：

1. **Tokenizer 变了** —— Claude 4.7 及以后（Opus 4.7/4.8/5、Sonnet 5、Fable 5）同样文本约多产生 **+30% token**。你的上下文预算（context budget）、截断阈值（truncation threshold）、成本模型、限流阈值全部要重算。
2. **最小可缓存前缀长度按模型不同且非单调** —— **512**（Opus 5 / Fable 5 / Mythos 5）、**1024**（Opus 4.8 / Sonnet 5 / 4.6 / 4.5）、**2048**（Opus 4.7 / Haiku 3.5）、**4096**（Opus 4.6 / 4.5 / Haiku 4.5）（2026 年中量级，随时变动）。换模型可能让原本命中的缓存全部失效。
3. **提示词不可移植** —— 为 A 模型调优的提示词在 B 模型上通常退化。迁移必须重跑完整回归集，且往往要重写系统提示。
4. **定价切换也是发布事件** —— 例如 Claude Sonnet 5 的促销价有明确切换日 **2026-09-01**（$2/$10 → $3/$15，2026 年中量级，随时变动）。把它当作一次有日期的容量/成本变更来排期。

**纪律**：`model` 字段一律写完整 snapshot ID。用 `latest` 别名等于把"发布"这个动作外包给了供应商 —— 你会在某个周三凌晨经历一次你没做的发布。把每个模型的 EOL 日期当作**有到期日的技术债（technical debt）**跟踪，和证书过期一样对待。

### 7.5 A/B 与在线评测

- **分流单位（unit of assignment）是会话/任务，不是请求。** 同一会话必须 sticky，否则：(a) 前缀缓存全废（成本与 TTFT 双恶化），(b) 用户看到人格分裂的行为，(c) 指标被污染。
- **LLM 的方差（variance）比传统服务大**：同一输入多次采样结果不同 ⇒ 需要的样本量显著更大，1% 流量的小实验通常得不出结论。先算功效（power），再开实验。
- **影子流量对 LLM 是成本翻倍**：必须抽样（sampling，1–5%），且只影子**读路径**（有工具副作用的路径不能影子）。
- **在线指标优先于离线分数**：任务完成率、人工干预率（human intervention rate）、重试率、会话轮数中位数、**每任务成本**。离线分数用来挡住明显退化，在线指标用来决定是否全量。

> **面试金句**：
> "AI 系统的发布单元不是'代码',是 **(模型 snapshot, 提示词版本, 工具定义哈希, 检索索引版本, 采样参数)** 这个五元组 —— 它们必须一起版本化、一起灰度、一起回滚。我见过最贵的一类事故是只回滚了代码没回滚提示词，结果新提示词配老工具定义，模型开始调用不存在的工具，重试逻辑把成本打上去 8 倍。所以我的 release manifest 是一个文件、一次 PR、一个 git sha —— 五个东西没有任何一个可以单独在生产上被改。"

---

## 8. 什么时候不要这么做（反模式 anti-pattern）

| 反模式 | 后果 | 正确做法 |
|---|---|---|
| **把六步法压缩成两步**（加列 + 加约束一起上） | 滚动发布期间老实例写入违反约束 → 大面积 5xx | 每步一次独立发布 |
| **DDL 不设 `lock_timeout`** | "瞬时"的 DDL 撞上长查询 → 全表锁队列堆积 → 看起来像数据库宕机 | `lock_timeout = '2s'` + 指数退避重试 |
| **金丝雀用"昨天同期"当基线** | 周期性和外部事件污染判定，误报+漏报 | 并发运行的基线组 |
| **金丝雀门禁用 CPU** | 滞后且与用户无关 | 用 SLI：错误率、p99、业务 KPI |
| **flag 默认值 = 新行为** | flag 服务故障 = 一次意外的全量发布 | 默认值恒等于当前生产行为 |
| **每个服务各自求值 flag** | 同一请求内前后不一致，产生没人预期过的数据状态 | 入口求值一次，随上下文传递 |
| **用 flag 做 entitlement** | 变成没有 schema/审计/一致性的第二套授权系统 | 放进授权与计费模型 |
| **flag 只加不删** | 2^n 路径，没人敢改任何一处 | owner + 到期日 + CI 阻断 |
| **模型用 `latest` 别名** | 供应商替你发布 | 固定 snapshot ID |
| **只回滚代码不回滚提示词/工具定义** | 版本错配，行为不可预测，成本失控 | 五元组一起版本化、一起回滚 |
| **用公开 benchmark 做上线门禁** | 8 个基准中 7 个可被刷到 ~100% | 生产轨迹回归集 + 相对退化门槛 |
| **换 judge 模型后直接比分数** | 风格偏差量级 0.10–0.76，基线已经变了 | judge 写进 manifest，换 judge 必须重新校准 |
| **回滚后不冻结流水线** | 40 分钟后同样的事故重演 | 自动冻结 + 强制事故记录 |
| **变更冻结窗口（change freeze）滥用**（"季度末不许发布"） | 冻结期结束时积压一大批变更同时上线 —— 风险不是消失了，是被压缩了 | 只冻结高风险类别，且冻结期照常做小批量发布 |

---

## 面试官会追问

1. 你要给一张 5 亿行的表加一个 `NOT NULL` 的列并建索引。完整说出你的步骤和每一步的锁级别。
2. 为什么 `ALTER TABLE` 即使是"瞬时"的也可能造成 30 秒全站不可用？怎么防？
3. 六步法里，哪一步之后就不能回滚了？你怎么决定这一步什么时候做？
4. 金丝雀 5 分钟没发现问题就全量了，结果两小时后爆了。哪里错了？
5. 自动回滚的判据你会设哪几条？为什么不用 CPU？基线从哪来？
6. Feature flag 服务挂了，你的系统会怎样？如果它返回的是"全部开启"呢？
7. 一次发布只改了工具定义的一个字段描述，成本涨了 10 倍。为什么？
8. 提示词应该放 git 还是放配置后台？运营要求不发版就能改，你怎么设计？
9. 你怎么给一次提示词变更设置上线门禁？为什么不用公开 benchmark 的分数？
10. 供应商宣布你在用的模型 6 个月后下线。列出迁移必须重算的所有东西。

---

**下一章** → [`04-ai-agent-systems/`](../04-ai-agent-systems/)：LLM 与 Agent 基础设施。
