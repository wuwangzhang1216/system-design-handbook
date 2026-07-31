# 03 · 多租户：隔离级别、噪音邻居与租户迁移

> 多租户（multi-tenancy）的本质是把"隔离强度（isolation level）"和"单位成本（unit cost）"焊在同一根旋钮上。
> 你不是在选一个架构，你是在选这根旋钮的刻度 —— 而且**每个租户的刻度可以不一样**。

---

## 读这一章之前

**你在工作中遇到过这个**

周一早上 9 点，你们最大的那个客户点了"批量导入"，8 分钟里往同一张 `documents` 表写了 400 万行。
同一个 Postgres 实例上另外 3 万个小客户的接口 p99 从 80 ms 涨到 6 秒，值班群炸了。
你打开监控想回答"到底是谁在写"—— 所有面板都是全局聚合的，没有一张图能按客户拆开，你只能挂在 `pg_stat_activity` 上一条条看 SQL 猜是谁。
三个月后同一个客户提了另一个要求："把我们的数据恢复到上周二下午三点。"而那 400 万行，和另外 3 万个客户的行，躺在同一张表里。

**需要先懂的概念**

| 概念 | 一句话 | 详见 |
|---|---|---|
| 分片 / 分片键 | 把数据切成互不重叠的块分放到多台机器；分片键决定一行落到哪块 | [00-concepts §5](../00-foundations/00-concepts.md) |
| 副本 | 同一份数据的多个完整拷贝，解决可用性和读扩展，**不解决容量** | [00-concepts §5](../00-foundations/00-concepts.md) |
| 可用性与依赖相乘 | 串行依赖的可用性是乘法：0.999³ = 99.7% | [00-concepts §10](../00-foundations/00-concepts.md) |
| 背压与限流 | 处理不过来时主动拒绝，而不是让队列无限增长 | [01-fundamentals §6](../00-foundations/01-fundamentals.md) |
| 缓存与命中率 | 把算过的结果暂存复用；命中率决定它挂掉时后端要承受几倍流量 | [01-building-blocks/02](../01-building-blocks/02-caching.md) |
| Outbox / CDC | 把数据库变更可靠地变成事件流 —— 租户迁移的"增量追平"靠它 | [01-building-blocks/03](../01-building-blocks/03-messaging-and-streams.md) |

**这一章要回答的问题**

1. 一千个客户的数据，是放在同一张表里靠 `tenant_id` 区分，还是一个客户一个库？两端的单位成本差多少倍？
2. 一个客户的批量导入拖慢了所有人，我怎么先把它量化出来，再限制住它？
3. 客户要求"恢复到昨天下午三点"或"彻底删除我们的全部数据"，在共享表模式下怎么做？要多久？
4. 一个租户长成了占 40% 负载的巨兽，我该改架构，还是把它搬走？

**本章新引入的术语**

| 术语 | English | 一句话定义 |
|---|---|---|
| 多租户 / 租户 | multi-tenancy / tenant | 同一套代码和基础设施同时服务多个互不信任的客户组织，每个组织就是一个租户 |
| Pool / Silo | pool / silo | 所有租户共用同一份表和同一个实例（pool）；每个租户独占自己的库、甚至独占整套栈（silo） |
| 爆炸半径 | blast radius | 一次故障最多波及多少个租户 —— 隔离设计真正在优化的是这个数字，不是可用性百分比 |
| 控制面 / 数据面 | control plane / data plane | 控制面存"哪个租户在哪、配额多少"这类元数据，每天只写几十次；数据面处理用户的每一次真实业务请求 |
| Cell | cell | 一整套能自成一体运行的服务 + 存储的副本；一批租户被整体放进一个 cell，cell 之间不共享故障 |
| 虚拟桶 | virtual bucket | 在"数据"和"物理机器"之间插一层固定数量（如 4096 个）的桶，扩容时只搬桶，不改哈希函数 |
| 噪音邻居 | noisy neighbor | 共享资源的场景下，一个租户的用量把其他租户的性能拖了下去 |
| 计量 | metering | 把每一份资源消耗（CPU 时间、SQL 耗时、存储、token）按租户逐笔记账，是配额与计费的前提 |
| 行级安全 | Row-Level Security (RLS) | 数据库层的一条规则，给这张表上的每条查询自动附加过滤条件；应用漏写 `WHERE tenant_id=...` 时也读不到别人的行 |
| fail-static / fail-closed | fail-static / fail-closed | 依赖挂掉时继续用最后一份已知正确的数据（fail-static）；判断条件缺失时默认拒绝而不是放行（fail-closed） |
| 升舱 | tier promotion | 按预先写死的信号，把某一个租户从共享模式搬到隔离更强的模式 |
| 前缀缓存 | prefix caching | 把多个请求共享的那段开头内容的推理中间结果缓存下来，后续请求跳过对这一段的重算 |

---

## 1. 四种隔离级别

```
 共享程度高 / 单位成本低 ──────────────────────────► 隔离强度高 / 单位成本高

 ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────┐
 │ L1 Pool    │  │ L2 Schema  │  │ L3 Silo-DB │  │ L4 Silo-Stack│
 │ 共享表     │  │ 共享实例   │  │ 独立库 /   │  │ 整栈单租户   │
 │ + tenant_id│  │ 每租户一个 │  │ 独立实例   │  │ (cell / VPC) │
 │ 行级过滤   │  │ schema     │  │            │  │              │
 └────────────┘  └────────────┘  └────────────┘  └──────────────┘
```

| 维度 | L1 Pool | L2 Schema | L3 Silo-DB | L4 Silo-Stack |
|---|---|---|---|---|
| 单租户边际成本（marginal cost）/月 | **$0.01–1** | $1–20 | $50–500 | **$500–5,000** |
| 单集群/实例租户上限 | 10⁶+ | **500–2,000**（见下） | 100–500 | 1 |
| 隔离强度 | 应用/RLS 逻辑隔离（logical isolation） | 逻辑隔离 + 独立对象 | 进程与存储隔离 | 网络+计算+存储全隔离 |
| 单租户备份/恢复 | ❌ 极难（要恢复到影子库再回填） | ⚠️ 可 `pg_dump -n` | ✅ 原生 | ✅ 原生 |
| 单租户硬删除（hard delete） | ⚠️ 大表 DELETE + vacuum | ✅ `DROP SCHEMA CASCADE` | ✅ `DROP DATABASE` | ✅ 销毁栈 |
| 单租户 PITR（point-in-time recovery，把数据恢复到过去任意时刻） | ❌ | ❌ | ✅ | ✅ |
| 爆炸半径（blast radius） | 全部租户 | 全部租户（共享实例资源） | 该实例上的租户 | 1 个租户 |
| per-tenant schema 定制 | ❌ | ✅（也是诅咒，见下） | ✅ | ✅ |
| 版本/发布节奏（release cadence） | 全体同步 | 全体同步 | 可分批 | **可完全独立** |
| 合规适配（数据驻留 data residency/BYOK） | 难 | 难 | 中 | ✅ 天然满足 |
| 跨租户分析查询 | ✅ 一条 SQL | ⚠️ 要 UNION N 个 schema | ❌ 需要数据平台 | ❌ |
| 迁移到下一级的难度 | — | 中 | 中 | 高（回不去） |

（表里两个合规词：**数据驻留（data residency）** = 合同要求这个客户的数据只能存放在指定国家或区域境内；**BYOK（bring your own key）** = 加密密钥由客户自己保管，客户随时可以吊销它，从而让你无法再解密他的数据。这两条一旦写进合同，就直接排除了共享实例的可能。）

### L2 的撞墙点（scaling wall）：schema 数量

`schema-per-tenant` 在 Postgres 上有硬性上限，且**撞墙时是断崖式的**：

```
每租户 50 张表 + 每表 3 个索引 = 每租户 ~200 个 pg_class 行
2,000 租户 → 40 万个 pg_class 行

后果（按出现顺序）：
  1. autovacuum 工作队列变长，系统表本身开始膨胀
  2. 每个连接的 relcache / catcache 膨胀 → 单连接常驻内存从 ~5 MB 涨到 100–500 MB
  3. pg_dump / 逻辑复制槽初始化从分钟变小时
  4. DDL 迁移：一次加列 = 2,000 次 ALTER TABLE，且需要 2,000 把 ACCESS EXCLUSIVE 锁
```

（`pg_class` 是 Postgres 的系统表，每张表、每个索引在里面各占一行；`ACCESS EXCLUSIVE` 是最强的表锁，持有期间连读都被挡住 —— 所以"2000 次 ALTER TABLE"不是慢，是 2000 段小停机。）

**实践上限：500–2,000 个 schema。** 早期信号是"每次发版的 migration 时间线性增长"和"连接数没变但内存涨了"。

⚠️ **L2 允许 per-tenant schema 定制 —— 这是它最大的卖点，也是最大的陷阱。** 一旦有 5 个租户的表结构不同，你的迁移脚本就不再是一段代码，而是一个需要人工判断的流程。**判据：如果产品不打算卖"定制字段"，就不要用 L2，直接 L1。**

### 混合（Bridge）才是常态

真实的 SaaS 几乎都是混合：

```
     ┌──────────────────────────────────────────────────┐
     │  路由层（tenant_id → placement）                  │
     └───┬──────────────┬───────────────┬───────────────┘
         ▼              ▼               ▼
    ┌─────────┐   ┌───────────┐   ┌──────────────┐
    │ Pool A  │   │ Pool B    │   │ Silo: acme   │  ← 大客户/高合规
    │ 40k 小租户│  │ 40k 小租户 │   │ Silo: globex │
    └─────────┘   └───────────┘   └──────────────┘
      99% 租户                       1% 租户 / 60% 收入
```

**升舱（tier promotion）判据（写进 runbook，不要临时判断）：**

| 信号 | 阈值（起步值，按自己系统标定） | 动作 |
|---|---|---|
| 单租户占本 pool 存储 | > 20% | 迁到独立 pool |
| 单租户占本 pool QPS/CPU | > 25% | 迁到独立 pool |
| 单租户行数占单表 | > 30% | 拆分片（shard split）或独立 DB |
| 合同要求数据驻留 / BYOK / 私有网络 | 任一 | 直升 L4 |
| 客户要求独立发布窗口 | 任一 | 直升 L4 |

---

## 2. 选择判据决策树

```
       有合规硬约束（数据驻留 / BYOK / 客户自有 VPC）？── 是 ─► L4 Silo-Stack（没得选）
                              │否
                       租户数量级？
        ┌─────────────────────┼─────────────────────┐
     < 50 个              50–2,000               > 2,000
        │                     │                      │
        ▼            需要 per-tenant 定制？           ▼
  L3 Silo-DB 可行     ┌───是──┴──否───┐          L1 Pool（唯一可行）
  （运维尚可接受）      ▼              ▼                │
                 L2 Schema        L1 Pool ◄────────────┘
                （接受迁移税）        + 大租户按信号升舱 → L3 / L4
```

**默认答案是 L1 + 升舱通道。** 从 L1 升到 L3 是一次数据迁移；从 L3 降到 L1 是一次架构重写。**先选便宜的那一端，把升舱通道建好。**

---

## 3. 租户路由与分片

### tenant_id 作为分片键（shard key）

**它几乎总是正确的分片键**，因为绝大多数查询天然带 `tenant_id`，不会产生跨分片扇出（fan-out：一个请求被拆成 N 个下游请求并等它们全部返回；N 越大，整体 p99 越接近"最慢的那一个"，见 [`00-foundations/01 §7`](../00-foundations/01-fundamentals.md)）。

代价：**分片内数据倾斜（data skew：各分片的数据量或负载严重不均，少数分片扛了大部分量）完全由租户大小决定**，你控制不了。所以分片键要预留一层间接：

```
❌ shard = hash(tenant_id) % N            -- 扩容要 rehash 全部数据
✅ shard = shard_map[hash(tenant_id) % 4096]   -- 虚拟桶：4096 个桶映射到 N 个物理分片
                                                -- 扩容 = 搬若干个桶，不动 hash
```

**虚拟桶（virtual bucket）数取 1024–8192**，一次定死。桶数是唯一不能在线改的参数。

⚠️ **超大租户必须能突破单桶**：给 whale 租户（巨型租户：一个租户的数据量或流量占到整个池的两位数百分比，下面 §8 会展开）启用二级键 `(tenant_id, sub_key)`，`sub_key` 取业务上天然的第二维度（project_id / workspace_id / 日期）。**这个逃生舱（escape hatch）必须在第一天就在 schema 里留好位置**（哪怕暂时恒为 `'0'`），事后加等于全量迁移。

### 路由表放哪

```
 控制面 Control Plane
   tenant_placement：tenant_id → {shard, region, tier, state}
   写入频率：每天几十次（新租户 / 迁移）
        │ 变更事件（版本号单调递增）
        ▼
 数据面 Data Plane 本地缓存
   · 进程内全量副本（10 万租户 × 100 B ≈ 10 MB，装得下）
   · TTL 60 s + 变更事件主动失效
   · ★ 控制面挂掉时继续用旧副本（fail-static，不是 fail-open）
```

⚠️ **反模式（anti-pattern）：控制面进入数据面关键路径（critical path）。** 每次请求去查控制面的路由表，等于把控制面的可用性乘进产品可用性。路由表必须能被数据面完整缓存并在控制面不可用时继续服务。

### Router 的三种形态（[AWS Guidance for Cell-Based Architecture](https://github.com/aws-solutions-library-samples/guidance-for-cell-based-architecture-on-aws) 口径）

| 形态 | 做法 | 优点 | 代价 |
|---|---|---|---|
| **负载均衡型** | 一个共享 LB 按 tenant 转发到 cell | 对客户端透明 | **router 在关键路径，自己变成新的单点** |
| **转发型** | 首次连接返回目标 cell 地址，之后客户端直连 | 延迟低，router 不在稳态路径 | 客户端需要逻辑；重定向缓存要能失效 |
| **DNS 型** | `acme.api.example.com` → cell 的 CNAME | 最简单最可靠 | DNS TTL 决定迁移生效时间（≥ 60 s） |

**注意 AWS 官方文档并不承诺 cell 化提升可用性 SLA**，它承诺的是**降低爆炸半径与缩短恢复时间**。不要在 SLA 承诺里写"因为做了 cell 化所以 99.99%"。

Cell 内部的组成、协调循环、跨 cell 迁移与 cell 级金丝雀（canary）见 [`03-saas-platform/01-control-plane.md`](../03-saas-platform/01-control-plane.md)；本节只讲**租户落到哪个 cell** 这一层。

---

## 4. 租户迁移剧本

这是多租户系统最常执行也最容易翻车的操作。**九步，每步有回滚点。**

```
时间轴 ────────────────────────────────────────────────────────────►

 ①标记      ②快照      ③增量追平        ④冻结   ⑤切换  ⑥观察     ⑦清理
 MIGRATING  批量拷贝    CDC 持续同步     写入    路由   双读校验   删源
    │          │            │            │       │      │         │
    └──可回滚──┴────可回滚───┴───可回滚────┤       ├─可回滚┤         │
                                          └─冻结窗口─┘（目标 < 30 s）
```

1. **标记状态**：`tenant_placement.state = 'MIGRATING'`，数据面开始对该租户拒绝**结构性变更**（DDL、批量导入），普通读写照常。
2. **快照拷贝**：源库一致性快照（Postgres `pg_export_snapshot` / 逻辑复制槽）批量导入目标库。**限速（throttling）**：不超过源库 IOPS 的 30%，否则会拖垮同 pool 的其他租户 —— 迁移本身就是最大的噪音邻居（noisy neighbor）。
3. **增量追平（catch-up）**：CDC（change data capture，变更数据捕获：订阅数据库自己的变更日志，把每一行的增删改实时变成一条事件流，见 [`01-building-blocks/03`](../01-building-blocks/03-messaging-and-streams.md)）把快照点之后的变更持续应用到目标。等到 **replica lag（目标库比源库落后的时间）< 1 s** 才进下一步。
4. **冻结写入**：把该租户的写请求排队（不是拒绝），返回 `Retry-After`，或前端显示"同步中"。**冻结窗口（freeze window）= 剩余 lag + 校验时间 + 路由生效时间**，目标 < 30 s。
5. **最终校验**：行数 + 关键表的 checksum（`sum(hashtext(row::text))`）比对。**不一致就回滚**，不要"先切了再说"。
6. **切换路由**：控制面更新 placement + 版本号，广播失效。DNS 型 router 这里要等 TTL。
7. **解冻**：排队的写请求放行到新位置。
8. **双读校验期（dual-read verification）**（1–7 天）：读走新库，同时异步采样读旧库比对，差异率上报。这是唯一能发现"漏迁了某张表"的手段。
9. **清理源数据**：确认无回滚需求后删除。**这一步经常被跳过**，结果半年后有人从旧库读到了幽灵数据。

**必须提前准备的两个东西：**
- **回滚脚本（rollback script）**（把 placement 改回去 + 反向 CDC），且演练过。
- **迁移期间的写入语义**：租户是否接受 30 s 只读？如果不接受，就需要双写（dual write）+ 冲突解决（conflict resolution），复杂度翻三倍。**先去问产品，再写代码。**

---

## 5. 噪音邻居：先量化，再治理

### 没有 per-tenant 计量（metering）就没有治理

在你能回答"acme 昨天消耗了多少 CPU-ms / IOPS / token"之前，所有配额都是拍脑袋。**最小可用计量集：**

| 资源 | 采集点 | 归因（attribution）方法 |
|---|---|---|
| API QPS / 并发 | 网关 | 直接按 tenant_id 打标 |
| CPU-ms | 应用中间件 | 请求开始/结束时取线程 CPU 时间 |
| DB 时间 | 连接池包装层 | 每条 SQL 的耗时归到当前 tenant；`pg_stat_statements` **跨租户聚合，无法直接归因** |
| 存储 | 定时扫描 | 按 tenant_id 聚合表/对象大小 |
| LLM token | 网关 | 输入/输出/缓存读 分别计量（三者单价差 10–50×） |

⚠️ Postgres 没有 per-tenant 资源控制。归因只能靠 [sqlcommenter](https://google.github.io/sqlcommenter/) 风格的注释标签（`/*tenant='acme',route='/v1/search'*/`）+ 应用侧计时。

### 治理手段

| 噪音源 | 检测信号 | 治理手段 |
|---|---|---|
| 某租户突发流量 | 单租户 QPS 占比 > 25% | **令牌桶限流（token bucket rate limiting）**（per-tenant）—— 桶里按固定速率放令牌，每个请求取走一个，取不到就拒绝；桶容量决定能容忍多大的瞬时突发。突发容量（burst capacity）= 稳态速率 × 10 |
| 某租户慢查询 | 单租户 DB 时间占比 > 30% | `statement_timeout` per-tenant（按 tier 设 1 s / 5 s / 30 s） |
| 某租户占满连接池 | 池等待时间 p99 上升 | **隔板（bulkhead）**：把池按 tier 切分，任一 tenant 上限 = 池容量 × 20% |
| 大批量导入 | 写 QPS 尖峰 | 批量作业走独立队列 + 独立连接池，与在线路径物理隔离 |
| 长任务饿死（starvation）短任务 | p99/p50 比值 > 20 | **公平队列（fair queuing）**（下） |
| 存储无限增长 | 单租户存储超配额 | 软限额（soft limit）告警 → 硬限额（hard limit）拒写（保留读） |

### 公平队列：DRR 是这里的正确答案

轮询（round-robin）在**请求代价方差大**时是错的 —— 一个租户全是 10 ms 的请求，另一个全是 5 s 的请求，轮询给了后者 500 倍的资源。

**Deficit Round Robin (DRR)** 按**代价**而不是**次数**分配：

```
每个租户一个队列 + 一个 deficit 计数器
每轮：deficit[t] += quantum[t]
     while deficit[t] >= cost(队首): deficit[t] -= cost(队首); 发出请求
     队列空时 deficit[t] = 0            ← 防止攒额度打突刺
参数：quantum[t] = 基础配额 × tier 权重（free=1, pro=4, enterprise=16）
     cost(req)  = HTTP 用历史 p50 耗时；LLM 用 (input_tok + 4 × output_tok)
```

**这里有一个 AI 场景特有的坑**：LLM 请求的 `cost` 在**执行前不可知**（输出长度未知）。做法是"先按 `max_tokens` 预扣（pre-deduct / reserve upfront），完成后按实际用量退还差额"—— 与第 2 篇的 TCC 预留模式同构。不做预扣的话，一个开着 `max_tokens=32000` 的租户能把整个队列拖死。

---

## 6. Postgres 行级安全（Row-Level Security, RLS）：正确用法与两个坑

### 正确的写法

```sql
-- ① 应用角色不能是表 owner，且不能有 BYPASSRLS
CREATE ROLE app_rw NOLOGIN;
-- ② 开启 RLS，并 FORCE —— 默认情况下表 owner 会绕过 RLS！
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents FORCE  ROW LEVEL SECURITY;

-- ③ 策略：读与写都受约束。WITH CHECK 缺失 = 可以写入别人的 tenant_id
CREATE POLICY tenant_isolation ON documents
  USING      (tenant_id = current_setting('app.tenant_id', true)::uuid)
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);

-- ④ 索引必须以 tenant_id 打头，否则 RLS 谓词变成全表过滤
CREATE INDEX ON documents (tenant_id, updated_at DESC);
CREATE INDEX ON documents (tenant_id, status) WHERE deleted_at IS NULL;
```

`current_setting('app.tenant_id', true)` 的第二个参数 `missing_ok=true`：未设置时返回 NULL，比较结果为 NULL → **零行**。这是正确的 fail-closed 行为。
⚠️ **绝对不要**为了"方便本地调试"写成 `COALESCE(current_setting(...), tenant_id::text)` —— 那一行代码把整个隔离层变成装饰品。

### 坑 1：连接池 + session 变量（最常见的生产事故）

> **transaction pooling（事务级连接池）**：PgBouncer 的一种模式 —— 一条数据库连接只在**一个事务期间**属于你，事务一提交就被还回池子给别的请求用。它能用几十条真实连接扛住几千个应用连接，代价就是下面这个坑：任何"会话级"的设置都会跟着连接漏给下一个租户。

```sql
-- ❌ 在 PgBouncer transaction pooling 下这是灾难
SET app.tenant_id = 'acme-uuid';       -- 会话级：连接归还池后仍然生效
SELECT * FROM documents;               -- 下一个租户拿到这条连接 → 读到 acme 的数据

-- ✅ 正确：is_local = true，事务结束自动还原
BEGIN;
  SELECT set_config('app.tenant_id', $1, true);   -- 第三个参数 true = 事务局部
  SELECT * FROM documents WHERE ...;
COMMIT;
```

**三条必须同时满足的规则：**

1. 用 `set_config(..., true)` 或 `SET LOCAL`，**永远不用 `SET`**。
2. **必须在显式事务里**。transaction pooling 下，autocommit 的单条语句各自是独立事务，`set_config` 会在下一条语句执行前就失效。
3. 用 `SET ROLE` 做隔离的方案同理会泄露，且 `server_reset_query = DISCARD ALL` **只在 session pooling 下生效**。

**防御性检查**（值得写进中间件）：每次借出连接后执行一次
`SELECT current_setting('app.tenant_id', true)` ，非空即为泄露 → 立刻丢弃该连接并告警。

### 坑 2：RLS 的性能陷阱

| 陷阱 | 机制 | 对策 |
|---|---|---|
| 索引失效 | RLS 谓词是 filter，若索引不以 tenant_id 打头，先扫全表再过滤 | 所有索引前缀加 tenant_id |
| **非 LEAKPROOF 函数阻止谓词下推（predicate pushdown）** | 规划器不允许把用户提供的、可能泄露信息的函数推到 RLS 过滤之下 → 扫描量放大 | 避免在 WHERE 里用自定义函数；必要时 `ALTER FUNCTION ... LEAKPROOF`（需 superuser，且你要为正确性负责） |
| 策略里调用函数 | 每行调用一次 | 策略里只用 STABLE 函数（`current_setting` 是 STABLE，安全） |
| 分区剪枝（partition pruning）失效 | 按 tenant hash 分区时，RLS 谓词形式不匹配分区键表达式 | 分区键与策略表达式写成同一形式；验证 `EXPLAIN` 里出现 `Partitions removed` |

**开销量级**：谓词能走索引时 **5–15%**；击穿索引时可以是 **10–100×**。所以上线 RLS 必须配一次全量 `EXPLAIN (ANALYZE, BUFFERS)` 回归。

> **面试金句**
> "RLS 我会用，但我不把它当**唯一**的隔离层。它是**纵深防御（defense in depth）的最后一道**：应用层照样在每个 query 里显式带 `tenant_id`，RLS 是防止我漏写的兜底。理由很实际——RLS 只在数据库连接上生效，而我的系统里还有缓存、搜索索引、对象存储、向量库、消息队列，它们没有 RLS。**隔离必须在每一层各自实现一遍**，指望一个数据库特性覆盖全栈是错的。"

---

## 7. 租户级备份 / 恢复 / 删除 / 导出

这四件事在合同里是义务，在架构上是四道题。**L1 pool 模式下它们全都很难 —— 这是 pool 模式的真实成本，通常在签第一个企业合同时才被发现。**

| 操作 | L1 Pool | L2 Schema | L3/L4 Silo |
|---|---|---|---|
| 备份 | 全库备份（无法单租户） | `pg_dump -n tenant_x` | 原生 |
| **恢复单租户到某时间点** | **PITR 到影子实例（shadow instance）→ 导出该租户 → 回填（backfill）**（小时级） | 同左但导出快 | 原生 PITR，分钟级 |
| 硬删除 | 分批 `DELETE` + vacuum，大表需数小时 | `DROP SCHEMA CASCADE` 秒级 | `DROP DATABASE` |
| 导出（数据可携带权 data portability） | 按 tenant_id 全表扫，要限速 | 快 | 快 |

**L1 的单租户恢复剧本**（唯一可行路径）：

```
1. 从快照 PITR 恢复一个影子实例到 T-1h        （15–60 min，取决于 WAL 量）
2. 从影子实例导出该租户全部表的行             （限速，避免影响生产恢复窗口）
3. 生产库上把该租户当前数据移到 *_quarantine   （不要直接删，留证据）
4. 导入影子数据；校验行数与外键
5. 失效该租户的所有缓存 / 搜索索引 / 读模型     ← 最容易漏
6. 销毁影子实例
```

**必须提前定的两个数字**：单租户恢复的 **RTO**（recovery time objective：从决定恢复到服务重新可用，允许花的最长时间 —— 现实值 2–6 小时，别承诺 15 分钟）和 **RPO**（recovery point objective：允许丢掉最近多长时间的数据 —— 这里等于 WAL 归档间隔；WAL = write-ahead log，预写日志，数据库先把每一次改动追加进这个顺序日志，再去改真正的数据文件）。

**删除的正确做法（软删除 soft delete → 硬删除 hard delete，呼应上一篇的 crypto-shredding）：**

```
T+0    软删除：state='DELETED'，立刻停止服务与计费，数据保留
T+30d  硬删除：DELETE / DROP + 删除该租户的 KEK
       同步擦除清单（缺一不可）：
         □ 主库          □ 只读副本      □ 备份（或备份保留期 ≤ 30d）
         □ 对象存储      □ 搜索索引      □ 向量索引（embedding 是 PII 派生物）
         □ 缓存          □ 日志/APM      □ 数据仓库/湖    □ 下游 SaaS
```

**per-tenant KEK 是 L1 pool 下唯一实用的删除加速器**（KEK = key encryption key，密钥加密密钥：用来加密其他密钥的那把主钥匙 —— 删掉它，被它保护的所有数据密钥就都解不开了，等于该租户的密文全部同时变成乱码）：租户数据按其 KEK 加密，删 KEK 让所有副本（包括备份里的）同时失效，剩下的物理删除可以慢慢做。代价见上一篇：KEK 存储绝不能开 PITR。

---

## 8. 巨型租户（Whale Tenant）的逃生舱

**信号**（在第 1 节升舱判据之外再加两条，任一触发就进逃生流程，不要等到出事）：单租户导致的慢查询占比 **> 40%**；单租户 token 消耗占本网关池 **> 20%**。

**三级逃生舱（成本递增）：**

1. **分片内二级分裂**：启用 `(tenant_id, sub_key)`，把该租户的数据打散（spread / shard the key）到同一分片的多个分区。最便宜，但只解决单表热点，不解决资源竞争。
2. **专用分片（dedicated shard）**：迁到只有它一个租户的分片（走第 4 节剧本）。解决资源竞争，保留同一套代码和发布节奏。**这是 90% 的 whale 的正确答案。**
3. **专用 cell / 单租户栈**：整栈隔离，独立发布窗口。只在客户为此付费（或合规强制）时做。

**反模式：为一个 whale 修改全局架构。**
> 一个租户占了 40% 的负载，正确的动作是把它搬到单独的分片，而不是给所有 5 万个租户引入一套复杂的分片内二级路由。**先隔离，再决定要不要泛化。**

---

## 9. AI 场景的多租户新问题

这是 2025–2026 才出现的一整类问题，且**大部分传统多租户经验在这里不成立**。

### a) 向量库的租户分区

| 系统 | 隔离原语 | 硬边界 |
|---|---|---|
| **Pinecone** | namespace | 单次查询**只能命中一个 namespace**（天然强制隔离，但也无法跨租户检索） |
| **Weaviate** | 原生 multi-tenancy | **必须先显式创建 tenant 才能写入**（fail-closed，很好） |
| **Qdrant** | payload 字段索引 + 过滤 | 过滤在 HNSW 遍历内部执行，性能好；但隔离靠查询正确性保证 |
| **pgvector** | Postgres RLS + tenant_id | 复用本篇第 6 节全部结论；适合几十到几百租户量级 |

（**HNSW** 是向量库最常用的近似最近邻索引结构：把向量组织成多层图，查询时从稀疏层往稠密层逐层逼近，用一点召回率换几个数量级的速度。**namespace / collection** 是各家给"一组互不相干的向量"取的名字，可以粗略地当成向量库里的一张表。）

⚠️ **namespace/collection 数量爆炸是多租户向量库最常见的翻车点。** 各家的 namespace 上限数字在 2026 年的对比文章里普遍**未经一手核实且厂商在改**（量级参考：Pinecone 侧有"10 万 namespace / 20 index"的二手说法，Cloudflare Vectorize 侧有"5 万 namespace / 每 index 500 万向量"的二手说法 —— **上线前必须查官方 limits 页并压测**）。

**通用对策**：小租户共用一个 collection + 强制 filter，大租户独立 collection。切换阈值按"单租户向量数 > 100 万"或"单租户查询占比 > 20%"。

### b) 前缀缓存（prefix caching）的跨租户泄露（已被实证，不是理论风险）

这是本手册里最需要单独强调的一条：**KV / prefix cache 既是 2026 最大的性能杠杆，也是最大的跨租户泄露面。**

> 三个词先说清楚：**KV cache** = 模型生成时缓存下来的注意力中间结果，让生成后一个 token 时不用把前面所有内容重算一遍；**prefix cache（前缀缓存）** = 把这份中间结果按"请求开头那段一模一样的内容"跨请求复用，谁的开头和它相同谁就能跳过这段计算；**TTFT** = time to first token，从请求发出到模型吐出第一个字的时间。泄露就发生在"跨请求复用"这五个字上 —— 命中与否会体现在耗时上，而耗时是攻击者能测量的。

- [PROMPTPEEK（NDSS 2025）](https://www.ndss-symposium.org/wp-content/uploads/2025-1772-paper.pdf) 利用 KV cache 共享造成的**服务顺序/时序侧信道（timing side channel）**逐 token 重建他人的 prompt。实测成功率：**已知 prompt 模板 99%、反推模板本身 98%、无任何背景知识 95%**；已知模板时约 **60 次请求**即可套出受害者的性别/年龄/体重/身高。攻击面覆盖 vLLM、SGLang、LightLLM、DeepSpeed；已向厂商披露，ByteDance 已确认该侧信道。
- 另一面：前缀缓存能把 TTFT 降低数倍、命中率（hit rate）在共享 system prompt 场景可达 70–90%、聚合吞吐（aggregate throughput）+30–50%（收益幅度存在争议，也有实测报告在特定负载下吞吐**下降** 36% 的相反结论 —— 取决于前缀重叠率与并发数）。

**结论（这是一个必须显式写出来的取舍，不是可以两全的）：**

```
prefix cache 共享策略：
  ✅ 同租户内共享      ← 保留大部分收益（同租户的 system prompt/工具定义本来就一样）
  ❌ 跨租户默认关闭    ← 放弃"多租户共享 system prompt"这个最大的单点杠杆
  ⚠️ 缓存键必须包含 tenant_id，且这是安全边界，不是性能优化
  ⚠️ 密钥、PII 绝不放进会被缓存的系统提示前缀
```

**用托管 API 时同样要处理**：OpenAI 的 ZDR（零数据保留）**不消除 prompt cache 驻留** —— ZDR 排除 abuse monitoring 中的客户内容并强制 `store=false`，但**仍会存储加密的 prompt cache tensors，最长 24 小时**。这一点必须写进 DPIA（data protection impact assessment，数据保护影响评估：GDPR 要求的一份文档，说明这套系统怎么处理个人数据、风险在哪、怎么缓解）与数据流图，不能因为"我们开了 ZDR"就跳过。

### c) 微调 / 适配器隔离

多 LoRA 共享底座是主流成本方案（一个 base model + N 个租户适配器）—— LoRA 是一种低成本微调方法：冻结底座模型的全部权重，只训练并加载一小组增量权重（几十 MB 而不是几十 GB），所以一个底座能同时挂载很多个租户的适配器。但 **vLLM 本身不提供租户隔离**：一个 Pod 内的 batch 共享同一个 KV cache 与 scheduler。

```
❌ 假设：不同 LoRA = 不同租户 = 天然隔离
✅ 现实：隔离必须由上层路由/策略层实现
   · 路由层保证同一 batch 内不混租户（或显式接受侧信道风险）
   · 适配器权重的访问控制在权重仓库侧做，不能靠推理引擎
   · 适配器可能泄露训练数据 → 微调数据走与生产数据同级的合规流程
```

学术方向可关注 [InfiniLoRA](https://arxiv.org/html/2604.07173v1)（解耦 LoRA 与基座执行）、[ForkKV](https://arxiv.org/pdf/2604.06370)（copy-on-write KV cache）、[Selective KV-Cache Sharing](https://arxiv.org/pdf/2508.08438)（缓解时序侧信道），但截至 2026 年中它们都还不是生产默认。

### d) Token 配额的公平性

**token 不是请求。** 传统的 RPM 限流在 LLM 场景基本无效：一个 200k 上下文的请求可以是一个 500 token 请求成本的 400 倍。

```
必须的三元组（LiteLLM / Portkey / Kong AI Gateway 已收敛为同一套）：
   per-key / per-user / per-team  ×  { TPM, RPM, 预算($) }
必须的双层预算：
   软限额 → 降级（切小模型 / 关思考预算 / 提高缓存命中要求）+ 告警
   硬限额 → 拒绝
   ★ Agent 场景再加：单会话 / 单任务的绝对 token 上限
     （Agent 循环方差比人类会话大一个数量级：单 Agent ≈ 4× chat token，
       多 Agent 系统 ≈ 15×）
```

**一条必须显式决策并写进合同的策略：失败请求是否计入租户预算。** 退款给租户体验好但制造滥用面；计入更公平但对瞬时故障显得苛刻。**没有正确答案，但必须选一个并对租户可见。**

### e) 平台级隔离不是免费的

[AWS Lambda Tenant Isolation Mode](https://aws.amazon.com/about-aws/whats-new/2025/11/aws-lambda-tenant-isolation-mode)（GA 2025-11-19）：调用时传租户标识，Lambda 保证执行环境（Firecracker microVM）**永不跨租户复用**，同租户内仍复用 —— 所以可以安全地缓存租户级配置与凭据。

⚠️ 代价：**冷启动（cold start：没有现成的执行环境可以复用时，要新建一个、加载运行时和你的代码再执行，这一次请求因此额外多花几百毫秒到几秒）次数从 O(并发) 变成 O(租户 × 并发)。** 隔离是免费的，冷启动不是。公告未披露最大租户数、冷启动放大倍数与定价细节 —— **三项都必须自测**。对策仍然是 bridge：小租户走 pool 模式，大租户/高合规租户走 silo 模式。

---

## 10. 什么时候**不要**做多租户

| 情况 | 理由 |
|---|---|
| 客户 < 20 个且都是大企业 | 每个客户独立部署更简单，且他们本来就要求这个 |
| 每个客户的数据模型差异巨大 | 你在做定制软件，不是 SaaS。共享 schema 只会让你两头不讨好 |
| 强合规域（医疗/金融/政府）且客户要求物理隔离 | 合同直接排除 pool 模式 |
| 团队 < 5 人且还没有 PMF | 多租户的复杂度（路由、配额、迁移、RLS、计量）会吃掉全部工程带宽 |

**四个反模式：**

1. **只在应用层过滤 tenant_id，没有第二道防线。** 一次 `WHERE` 漏写就是全量数据泄露。**必须有 RLS 或等价物做纵深防御**，且要有自动化测试专门验证"跨租户读取返回零行"。
2. **把 tenant_id 放在 URL 而不是从会话/token 推导。** `GET /api/tenants/{tenant_id}/docs` 看起来 RESTful，实际是把授权决策交给了客户端。tenant_id 必须来自已验证的凭据。
3. **事后改分区键（partition key）。** 等于全量数据迁移。若业务天然存在跨租户协作（共享工作区、跨组织审批），**要么把协作对象本身作为分区单元，要么承认该子系统不适用分片模型**。
4. **用"我们以后会做多租户"来推迟计量。** 计量（per-tenant 的资源账本）是所有配额、计费、噪音治理、成本归因的前提，且**它必须在第一行业务代码里就带上 tenant 标签** —— 事后补标签是全链路改造。

---

## 这一章的三句话

1. **隔离级别不是一个全局架构选择，而是每个租户各自一档的旋钮。** 默认把所有人放进最便宜的共享池，同时把"升舱"的信号（存储 > 20%、QPS > 25%、合规硬要求）和搬迁剧本提前写进 runbook —— 从 L1 升到 L3 是一次数据迁移，从 L3 降回 L1 是一次架构重写。
2. **没有 per-tenant 计量，就没有任何配额、限流和成本归因可言。** 而计量必须在第一行业务代码里就带上租户标签，事后补是全链路改造 —— 这是多租户系统里唯一一个"晚一天做，贵一个数量级"的决定。
3. **隔离必须在每一层各自实现一遍。** RLS 只保护数据库连接，你的缓存、搜索索引、对象存储、向量库、消息队列和 LLM 前缀缓存全都没有它 —— 指望一个数据库特性覆盖全栈，是多租户设计里最贵的一个错觉。

---

## 面试官会追问

1. 四种隔离级别怎么选？给我一个具体的判据，不要说"看情况"。
2. schema-per-tenant 到多少租户会撞墙？信号是什么？
3. 分片键选 tenant_id 有什么问题？超大租户怎么办？
4. 租户路由表放哪？控制面挂了数据面还能工作吗？
5. 把一个租户从 A 分片迁到 B 分片，完整步骤是什么？冻结窗口多长？怎么回滚？
6. Postgres RLS + PgBouncer transaction pooling，有什么坑？正确写法是什么？
7. 只有 RLS 够不够？（→ 缓存、搜索、对象存储、向量库、队列都没有 RLS）
8. Pool 模式下，一个客户要求恢复到昨天 3 点的数据，你怎么做？RTO 多少？
9. 一个租户占了 40% 的集群负载，你的处理顺序是什么？
10. 多租户 LLM 服务里，共享 prefix cache 有什么风险？你会怎么设计缓存键？
11. 为什么 RPM 限流在 LLM 场景不够？失败的请求算不算租户的预算？

---

**下一篇** → [04-api-design-and-versioning.md](04-api-design-and-versioning.md)
