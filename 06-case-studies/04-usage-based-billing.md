# 04 · 案例：设计用量计费系统（准确性 / 幂等 / 对账）

> 计费系统的正确性目标是**不对称**的：漏计（under-billing）主要造成收入与财务核算偏差，重计（over-billing）还直接造成客诉、退款和法律风险。
> 用同一套机制去保证这两件事，是这道题最常见的失分点。

---

## 读这道题之前

> **阅读层级与合规口径**：full-stack 读者先沿“事实产生 → 可靠投递 → 去重 → 聚合 → 评级 → 发票 → 对账”读主干；状态存储与流处理优化是进阶。价格、厂商限额、税务与发票编号规则会随时间和辖区变化，本文数字是截至 **2026-08** 的教学算例，不替代财务、税务或法律意见。

**如果你是直接翻到这道题的**：正文里的 outbox、幂等键、watermark、租约都是当已知用的 —— 案例篇不再解释构件。第 2 题答不出，§3 那句"用两种完全不同的机制"你只会读成一句口号。

**先确认你能回答这三个问题**

1. 双写问题（dual-write）是什么？"先写业务库，再往 Kafka 发一条计量事件"，进程恰好死在两步之间会怎样？Outbox 是怎么把它变成一次写的？
   答不出 → 先读 [`03-messaging-and-streams.md`](../01-building-blocks/03-messaging-and-streams.md) §4
2. at-least-once 投递保证"不丢"，幂等键保证"不重"。那么漏计和重计分别该由这两件里的哪一件负责？为什么一个 exactly-once 同时解决两个，最后两个都保证不了？
   答不出 → 先读 [01-fundamentals §5 幂等](../00-foundations/01-fundamentals.md#5-幂等idempotency分布式系统的第一公民)、[`03-messaging-and-streams.md`](../01-building-blocks/03-messaging-and-streams.md) §3
3. 事件时间（event time）和处理时间（processing time）差在哪？watermark 判定的到底是什么，它一旦推过去，迟到的事件会怎么样？
   答不出 → 先读 [`03-messaging-and-streams.md`](../01-building-blocks/03-messaging-and-streams.md) §6

**这道题会用到的构件**

| 构件 | 用在哪 | 详见 |
|---|---|---|
| Outbox、at-least-once、DLQ、背压 | §5 为什么日志不能当计费依据、Kafka 写不进去时返回 429 | [`03-messaging-and-streams.md`](../01-building-blocks/03-messaging-and-streams.md) §4、§5 |
| 流处理窗口、watermark、迟到事件 | §6 迟到事件三选一、§7 聚合必须幂等可重跑 | [`03-messaging-and-streams.md`](../01-building-blocks/03-messaging-and-streams.md) §6 |
| 幂等键的来源、派生规则与去重窗口 | §6 一次业务操作如何在首次尝试前获得稳定 ID、32 天窗口那笔 154 GB 的账 | [01-fundamentals §5](../00-foundations/01-fundamentals.md#5-幂等idempotency分布式系统的第一公民)、[`02-billing-and-metering.md`](../03-saas-platform/02-billing-and-metering.md) §4 |
| 租约（lease）与本地令牌 | §10 实时配额：超支上界 = 实例数 × MAX_LEASE | [`05-consensus-and-coordination.md`](../01-building-blocks/05-consensus-and-coordination.md) §4 |
| 不可变原始层与重算 | §2 在明确保留期内保存 canonical facts、§9 出账重算的输入 | [`05-data-platform.md`](../02-architecture-patterns/05-data-platform.md) §2、§3 |

**这道题的一句话本质**

> **计费的正确性目标是不对称的，所以它必须由两套互不依赖的机制分别保证。**
> 漏计靠 outbox + at-least-once 保证"一定送到"，重计靠**首次尝试前持久化、所有重试复用**的业务操作 ID 派生幂等键，保证"不会算两次"。带着这句话往下读 —— 每见到一个组件就问一次："它在守漏计，还是在守重计？"两个都不守的，就是这道题的装饰品。

---

## 1. 需求澄清（前 5 分钟）

| 问题 | 为什么决定架构 |
|---|---|
| 计费维度：token / 请求 / 席位 / 结果（outcome）？ | 决定事件密度，直接决定管道量级 |
| 误差预算（error budget）是多少？漏计和重计**分别**能容忍多少？ | **这是整道题的地基**，见 §3 |
| 账期（billing period）按自然月还是订阅锚点日（anchor day）？租户时区？ | 决定聚合边界与月末峰值 |
| 需要实时硬限额（hard limit）吗？延迟预算多少？ | 决定要不要做本地令牌 + 周期同步 |
| 账期关闭后的迟到事件（late-arriving event）怎么处理？ | **必须写进产品条款，三选一**，见 §6 |
| 争议窗口（dispute window）12 个月还是 24 个月？ | 决定原始事件保留期与审计粒度 |
| 自建到哪一层？评级和发票也自建吗？ | 见 §11 —— 大部分人在这里画错线 |

**本文假设**：LLM 平台型 SaaS，10 万租户，按 token 计费（input / output / cache_read 三种 meter），需要实时硬限额，多币种，账期按订阅锚点日。

---

## 2. 规模估算

```
5,000 万 LLM 请求/天 × 3 个 meter/请求 = 1.5 亿 计量事件/天
  = 1,736 events/s 平均 × 5（日内峰均比） = 约 8,700 events/s 峰值

事件 ~400 B → 60 GB/天 → 1.8 TB/月
  → Parquet + zstd 约 5× → 360 GB/月 → S3 约 $8/月（2026 年中量级，随时变动）
```

**推论 1**：按 2026-08 的示例限额，`8,700 events/s` 可能超过普通单事件入口或逼近单流边界；实际还取决于 batch/stream 产品、账户配额与事件能否一次携带多个 meter。
⇒ 本题选择在自己这一侧保留 canonical usage facts，并按外部计费系统支持的粒度预聚合（pre-aggregation）；上线前重新查官方 limits 与合同配额。

**推论 2**：仅按压缩后对象字节估算约 **$8/月**，但这没有包含请求、元数据、复制、查询、归档与合规成本。
⇒ 原始计量事实应在争议、审计和重算所需的**明确保留期**内不可变并可追溯，之后按合同、税务和隐私最小化要求归档或删除。它是本方 canonical fact layer，但不是唯一证据；provider 账单与业务结果是独立对账来源。

---

## 3. 误差预算：先定义"对"，再谈架构

```
月度计费误差目标（按金额）：
  ├─ 漏计 under-billing    < 0.05%    → 直接的收入漏损
  ├─ 重计 over-billing     < 0.005%   → 客诉/退款/合规，容忍度低一个数量级
  └─ 跨账期错配            < 0.04%    → 非净损失，但制造对账噪声
```

**为什么不对称**：丢事件 = 少收钱，方向**永远对供应商不利** —— $5,000 万 ARR 上 0.5% 的事件丢失 = **$25 万/年**。而重复计费（double billing）是客户看到账单翻倍 → 工单 → 退款，在受监管市场可能是合规事件。

> **面试金句**
> "我会给漏计和重计设两个差一个数量级的预算，并且用**两种完全不同的机制**保证：漏计靠 outbox + at-least-once 保证'一定送到'，重计靠首次尝试前持久化、重试复用的业务操作 ID 派生幂等键（idempotency key），再配去重窗口（deduplication window）保证'不会算两次'。试图用一个 exactly-once 同时解决两个问题的系统，最后两个都保证不了。"

---

## 4. 高层架构

```
 ┌── 产生 ─────┐  ┌── 传输 ──┐  ┌── 处理 ──────────┐  ┌── 商业逻辑 ─┐  ┌── 收款 ──┐
 │ LLM Gateway │→ │  Outbox  │→ │ 去重（幂等键）    │→ │  Rating     │→ │ 发票     │
 │ (provider   │  │    ↓     │  │      ↓           │  │ (价格计划   │  │   ↓      │
 │  usage 对象 │  │  Kafka   │  │ 预聚合            │  │  版本化)    │  │ 支付网关 │
 │  的产生点)  │  │ (tenant  │  │ tenant×meter     │  │      ↓      │  │   ↓      │
 │             │  │  分区)   │  │   ×1min 桶        │  │  账期快照   │  │ 催收     │
 └─────────────┘  └────┬─────┘  └────┬─────────────┘  └─────────────┘  └──────────┘
                       ↓             ├──→ 实时配额服务（本地令牌 + 周期同步）
                  原始事件湖         └──→ 客户看板（新鲜度承诺 ≤ 15 min）
                  Iceberg on S3            ↑
                  按保留策略归档 ──────────┴── 对账作业（三方比对，每日）
```

**四条不可动摇的边界**：

1. **计量处理与业务写路径解耦，但事实产生不能双写**。有业务数据库时用同事务 outbox；无数据库的网关用可恢复 WAL/spool。不要在同步路径查询远端去重库。
2. **原始事件在保留期内追加且版本可追溯**。聚合、评级和发票可从 canonical facts 重算；更正用反向/修正事件，不原地篡改旧事实。
3. **已出具发票不原地改金额**。按适用会计/税务规则用贷记单、借记单或替代发票调整，并保留关联链路。
4. **看板与账单共享同一套 canonical facts、口径和聚合代码**。看板读未关闭的实时聚合，账单引用关账快照；两者可因 watermark/迟到而暂时不同，但差异必须可解释。

---

## 5. 深挖一：事件的产生与传输

### 在哪里埋点（instrumentation）

| 位置 | 可信？ | 有业务语义？ | 结论 |
|---|---|---|---|
| 客户端 SDK | ❌ | ✅ | **绝不**。用户能改的东西不能用来收钱 |
| LB / 访问日志 | ✅ | ❌ 不知道 token 数 | 只能做兜底交叉验证 |
| **网关收到 provider usage / 账单事实时** | ✅ | ✅ | 本题主来源；中断流可能没有最终 usage，须先估算并在 provider export 到达后对账 |
| 业务服务内部 | ✅ | ✅ | 可以，但每个服务都要重实现一遍可靠投递 |

**原则：在最接近"不可否认事实"的位置埋点。** 对 LLM 产品，优先使用 provider 返回的 usage，并保留 provider request id；客户端中断或中途错误拿不到终态时，用本地观测生成 `estimated` 事实，后续由 provider usage export 修正。

⚠️ **不要把本地 tokenizer 估算当最终收费事实。** tokenizer 会随模型版本变化，不同 provider 口径也不同；本地估算适合准入、流中断时的临时用量与异常检测，最终金额应使用合同约定的 provider/自建推理权威计数并对账。若产品明确按自定义计数器收费，则该计数器本身必须版本化、可复现且写入合同。

### 为什么不能只靠应用日志

1. **尽力投递（best-effort delivery）**：采样、缓冲区满即丢、磁盘满、agent OOM，没有一环承诺不丢。
2. **schema 无契约**：有人改了字段名，计费静默错一个月。
3. **SLO 不匹配**：普通日志允许采样或有限丢失；计量事实通常允许延迟，但丢失预算要单独定义并显著更严。
4. **保留期不匹配**：日志留 30 天，争议窗口 12–24 个月。
5. **无法区分"没打日志"和"没发生"** —— 这一条是致命的，对账时你无法证明任何事。

⇒ **计量事件是一等业务事件，走 Outbox → Kafka**（见 `../01-building-blocks/03-messaging-and-streams.md`）。

### 丢失概率量化，然后选机制

| 方案 | 丢失量 | 代价 | 适用 |
|---|---|---|---|
| 进程内缓冲，1 s flush | 每次崩溃 ~1,700 条；日均 5 次重启 → 8,700 条/天 = **0.006%** | 0 | 单事件 < $0.0001 |
| **Outbox**（与业务结果同事务写 outbox + CDC） | 消除事务边界内的双写窗口；仍需备份、监控与对账 | 多一次本地写，延迟实测 | 有业务数据库且事件与结果同事务 |
| 持久 WAL/spool + 后台投递 | 取决于磁盘持久性与复制；节点永久丢失仍可能丢 | 要处理磁盘满、损坏、恢复和重放 | 无业务 DB 的网关层 |

LLM 场景一次请求可能值 $0.05 ⇒ **用 Outbox 或 spool，不要用裸缓冲**。

### 背压（backpressure）：宁可拒绝，不要静默丢

Kafka 不可用时，producer 调用会阻塞、超时或在异步 callback/future 中返回失败；客户端库不会替应用保证业务事实已持久化。必须检查发送结果并显式兜底：

```properties
acks=all
enable.idempotence=true
max.block.ms=30000
delivery.timeout.ms=120000
```
应用层：若没有事务 outbox，先把事件持久化到可恢复 spool，再异步发 Kafka；spool 空间/恢复能力接近边界时，按产品风险策略拒绝会产生新收费事实的请求或切到明确的免费/降级模式。背压策略必须可见、可告警，不能静默继续并假装会计量。

---

## 6. 深挖二：准确性保证

### 幂等键标识业务操作，不标识某一次网络尝试

```python
# 错：每次 HTTP 重试都生成新的 operation_id → 重试 3 次就是 3 个业务操作
operation_id = str(uuid.uuid4())

# 对：发起方在第一次尝试前生成并持久化 operation_id，之后所有超时重试都复用它。
# operation_id 可以是随机 UUID；关键不是“随机还是确定性”，而是“一次生成、全程复用”。
operation_id = operation_store.load_or_create(client_action_ref)
# 同一次业务操作里的不同收费步骤，再从稳定 operation_id + semantic_step 确定性派生事件 ID。
event_id = uuid.uuid5(NAMESPACE_METERING,
                      f"{tenant_id}:{operation_id}:{meter_name}:{semantic_step}")
```

> **"有幂等键 = 不会重复计费"是错的。** 幂等只保证同 key 不重复入账，不保证重试复用了同一个业务操作 ID。ID 可以由客户端、SDK 或服务端生成；**权威生成方必须能在首次尝试前持久化它，并让同一次业务操作的所有重试复用它**。若客户端协议没有稳定操作引用，服务端无法凭两次长得相同的 HTTP 请求安全判断它们是重试还是两次真实购买；在每个请求入口新建 ID 只能去重该请求内部的下游重试，不能去重 HTTP 级重试。

采用 CloudEvents 时，去重身份通常取稳定逻辑 `source` + `id`；`source` 不应包含 pod/region 等部署实例，否则迁移或重放会改变身份。不同计费产品的窗口、迟到与回填语义不一，选型时用重试、越窗与 backfill 用例实测。详见 [`03-saas-platform/02-billing-and-metering.md §4`](../03-saas-platform/02-billing-and-metering.md#4-去重与幂等最容易讲错的一节)。

### 去重窗口：它的长度就是一笔存储账

本题的规模把窗口长度直接变成了一张账单。取行业默认的 **32 天**窗口（[OpenMeter](https://openmeter.io/blog/usage-deduplication)）代入本题的 1.5 亿事件/天：

```
窗口 = 你能容忍的客户端重放延迟上限；成本 = 流处理状态存储

1.5 亿/天 × 32 天 = 48 亿键
  精确存储（RocksDB，32 B/键）→ 154 GB 状态
  分层：Bloom filter（48 亿键 @ 1% FP ≈ 5.7 GB）前置
        BF 说"没见过" → 一定不在旧集合：写入精确存储后放行（多数流量省一次读）
        BF 说"可能见过" → 查精确存储（1% 流量 + 真重复）
```

**更省的做法**：按事件发生日/小时分桶，整桶到期删除；若把窗口压到 7 天，必须同时把“超过 7 天的重放如何处理”写进 API/合同，并用发票快照、provider 对账或长期业务操作索引防止越窗重计。越界事件进入**人工审查/异常队列**，不要把它和处理失败的普通 DLQ 混为一谈，也不要静默丢弃或接受。

### 分区与热点（hot partition）

Kafka 按 `tenant_id` 分区时，同租户有序且去重状态局部，但巨型租户会打爆单分区。可对明确识别的巨型租户做二级分片：
```
partition_key = (tenant_id, hash(operation_id) % tenant_shard_count)
```
同一 `operation_id` 必须稳定落在同一子分区，才能保留该 key 的局部去重；代价是丢失租户全局顺序、聚合要 fan-in 多个分片、扩缩 `tenant_shard_count` 还要版本化路由。因此它不是“免费午餐”，只在热点证据出现后启用。

### 迟到事件：三选一，写进合同

用 `event_time`（业务发生）与 `ingest_time`（进入管道）双时间戳 + watermark：

| 情形 | 处理 | 客户可见？ |
|---|---|---|
| 账期关闭前到达 | 正常计入 | 否 |
| 账期关闭后、宽限期（如 3 天）内 | **计入下期**，发票单列 `prior period adjustment` 行 | ✅ 必须 |
| 超过宽限期 | 按合同选择下期调整、贷/借记单或人工争议流程；原始事实进入异常队列，不静默消失 | ✅ 条款写明 |

**未找到任何主流厂商对"账期关闭后迟到事件"的公开 SLA。** 你必须自己定义并写进条款，否则第一次出现就变成一次谈判。

**账期关闭（period close）必须有明确的"关闭时刻"和"关闭后不可变"语义**：finalize 时把该账期聚合结果快照进不可变表（`usage_snapshots`），发票行引用快照 ID。之后无论上游怎么重算，这张发票的依据不变。

**上面那张表讲的是"怎么处理"，但真正决定处理方式的是事件落在关账时刻的哪一侧。同一条事件、同一个账期归属，只因为到达时间差了几分钟，走的就是两条完全不同的链路：**

```mermaid
sequenceDiagram
    autonumber
    participant Svc as Service
    participant MP as MeteringPipeline
    participant Agg as Aggregator
    participant Led as Ledger
    participant Inv as Invoice

    Note over Svc,Inv: Period P is still open
    Svc->>MP: emit event with event_time in P and idem_key
    MP->>MP: dedup inside the 32d window
    MP->>Agg: deduped event
    Agg->>Agg: upsert 1min bucket then roll up to hourly
    Agg->>Led: post usage into period P
    Note over Agg,Led: watermark is behind close so the ledger of P is still mutable
    Led->>Inv: finalize P and freeze usage_snapshot
    Inv-->>Svc: invoice of P issued with a locked amount
    Note over Led,Inv: PERIOD CLOSE. snapshot and invoice amount are immutable from here
    Svc->>MP: emit event with event_time in P arriving after close
    MP->>MP: dedup says this key was never seen
    MP->>Agg: deduped late event
    Agg->>Led: this event belongs to the already closed period P
    Note over Led: contract says invoice P is immutable; do not rewrite it in place
    alt inside the grace window of 3 days
        Led->>Inv: prior period adjustment line on the invoice of P plus 1
        Inv-->>Svc: adjustment is customer visible and invoice of P stays untouched
    else past the grace window
        Led->>Inv: route to contract-defined adjustment or manual review
    end
```

> 📖 **读图要点**：关账后不原地改 P 期发票；迟到事实走可审计的调整对象或人工流程，并关联原账期。具体是下期调整、贷/借记单还是替代发票，由合同与辖区决定，系统要表达这项策略而不是把一种做法写死成“唯一合法”。

### 对账：三方比对（three-way reconciliation），每天跑

```sql
WITH provider AS (   -- 三个来源先归一成相同维度；真实系统还会带 account/region
  SELECT usage_date AS d, provider, model, token_type AS meter, SUM(tokens) AS tok
  FROM provider_usage_export WHERE usage_date = :d GROUP BY 1,2,3,4),
metered AS (
  SELECT event_date AS d, provider, model, meter, SUM(qty) AS tok
  FROM metering_events_daily WHERE event_date = :d GROUP BY 1,2,3,4),
app AS (
  SELECT created_at::date AS d, provider, model, token_type AS meter, SUM(tokens) AS tok
  FROM llm_calls WHERE created_at >= :d AND created_at < :d + INTERVAL '1 day'
  GROUP BY 1,2,3,4),
dims AS (
  SELECT d, provider, model, meter FROM provider
  UNION SELECT d, provider, model, meter FROM metered
  UNION SELECT d, provider, model, meter FROM app)
SELECT k.*, p.tok AS provider_tok, m.tok AS metered_tok, a.tok AS app_tok,
       (m.tok - p.tok)::numeric / NULLIF(p.tok, 0) AS drift_vs_provider,
       (m.tok - a.tok)::numeric / NULLIF(a.tok, 0) AS drift_vs_app
FROM dims k
LEFT JOIN provider p USING (d, provider, model, meter)
LEFT JOIN metered  m USING (d, provider, model, meter)
LEFT JOIN app      a USING (d, provider, model, meter);
```
查询层再把“任一来源缺失”视为单独告警，并对非零分母比较 drift；不要用 `COALESCE(..., 1)` 把缺失和 100% 漂移混成一个不可解释的数。

**允许的漂移（drift）**：与 provider < 0.5%（存在跨时区归属、舍入、失败请求是否计费的口径差异）；与应用侧 < 0.05%（同一份事实的两条路径，漂移大就是有 bug）。

**必须显式决策：失败请求是否计入用量与预算。** 退款体验好但制造滥用面，计入更公平但对瞬时故障显得苛刻。**这个决定必须写进合同并对租户可见。**

---

## 7. 深挖三：聚合

```
raw events (Iceberg on S3, 永久)
   ↓ 流处理，窗口 1 min，watermark 容忍乱序 5 min
1min 桶 (tenant, meter, model, bucket_start) → qty    ← 看板读这一层
   ↓ 每小时
1h 桶                                                  ← 配额同步读这一层
   ↓ 每日 + 账期关闭时
账期聚合 (tenant, meter, period_id) → qty              ← 评级读这一层
```

**预聚合 vs 查询时聚合**：直接扫原始事件灵活但看板昂贵；预聚合快，但未保留的维度只能在原始事实仍处于保留期时回填。不要为了“以后也许有用”随意多留高基数或敏感维度；用已知定价/对账需求决定 schema，并给历史重算保留明确窗口。

### 聚合必须幂等且可重跑（replayable）

```sql
-- 错：append（重跑就翻倍）           INSERT INTO usage_1min SELECT ...;
-- 对：每次从 canonical deduped facts 重算完整的半开桶，再覆盖该桶
INSERT INTO usage_1min (tenant_id, meter, model, bucket_start, qty, event_count, updated_at)
SELECT tenant_id, meter, model, date_trunc('minute', event_time), SUM(qty), COUNT(*), now()
FROM metering_events_deduped
WHERE event_time >= :bucket_from AND event_time < :bucket_to
GROUP BY 1,2,3,4
ON CONFLICT (tenant_id, meter, model, bucket_start)
DO UPDATE SET qty = EXCLUDED.qty,          -- 覆盖而非累加
              event_count = EXCLUDED.event_count, updated_at = now();
```
调用方必须传完整桶边界；若只拿一个任意子区间覆盖整桶，会把区间外已有用量抹掉。并发回填还要用 generation/version 防止旧重算覆盖新重算：接管租约时原子递增 generation，写入必须带 `WHERE stored_generation < :generation`（或等价 CAS），由存储层拒绝恢复运行的旧 writer。只有 TTL 的“单写者租约”没有 fencing，挡不住旧进程暂停后醒来继续覆盖。计费管道一定会重跑（上游修 bug、迟到回填、灾难恢复），不可重跑的聚合等于不可修复。

### 时区与账期边界

**账期按租户的 billing timezone 计算，不是 UTC。** 事件统一存 UTC，转换在聚合时做。

```
账期 = [anchor_day 00:00:00 @tenant_tz, 下一个 anchor_day 00:00:00 @tenant_tz)
必须显式定义的边界（否则每年出两次事）：
  · anchor = 31 日 → 2 月落到 28/29；4/6/9/11 月落到 30
  · 夏令时切换日有 23 或 25 小时 → 用绝对时刻算，不要"加 24 小时"
  · anchor 分布决定月末峰值：全用自然月 = 1 号凌晨 10 万份发票同时生成，DB 必挂
    ⇒ 按 anchor day 打散是免费的削峰
```

---

## 8. 深挖四：评级（Rating）与定价规则

| 原语 | 语义 | 陷阱 |
|---|---|---|
| **阶梯 tiered** | 分档单价 | **graduated（累进）vs volume（一口价）差异巨大** |
| 包含额度 included | 先扣免费额度 | 是否跨期结转（rollover）必须定义 |
| 承诺用量 commitment | 保底消费 | 未用完是否退、是否结转 |
| 折扣 discount | 百分比 / 固定 / 覆盖单价 | 多个折扣**叠加顺序**影响结果 |
| 多币种（multi-currency） | 按租户结算币种 | **汇率必须在出账时刻快照写进发票行，永不重算** |

```
价目：0–1M token $10/M，1M–10M $8/M，>10M $5/M；实际用量 12M
  graduated（累进，每档各算）：1×10 + 9×8 + 2×5 = $92
  volume（一口价，全按最高档）：12×5              = $60
  差 53%。合同里写的是哪个？
```
这是真实的争议来源。**评级引擎必须显式支持两种模式并在价格计划里标注**，不能靠"我们默认是累进"这种口头约定。

### 规则引擎：数据驱动，不要图灵完备

嵌入式脚本（Lua/JS）表达定价 = 不可审计、不可静态校验、不可确定性重放。用声明式（declarative）价格计划（pricing plan）：

```json
{ "plan_id": "pro-2026-07", "version": 3, "immutable": true, "currency": "USD",
  "components": [
    { "meter": "input_tokens", "model": "tiered_graduated", "unit": 1000000, "included": 0.5,
      "tiers": [ {"up_to": 1, "unit_amount": "10.00"}, {"up_to": 10, "unit_amount": "8.00"},
                 {"up_to": null, "unit_amount": "5.00"} ] },
    { "meter": "output_tokens", "model": "flat", "unit": 1000000, "unit_amount": "30.00" } ],
  "commitment": { "amount": "500.00", "period": "month", "drawdown": true } }
```

**三条铁律**：
1. **价格计划版本化且不可变**。订阅指向 `(plan_id, version)`。改价 = 发新版本 + 迁移订阅，不是 `UPDATE`。
2. **评级是纯函数（pure function）**：`rate(usage_snapshot, plan_version, fx_snapshot) → line_items`。同输入永远同输出，可离线重放。
3. **金额用定点数（fixed-point）**（`NUMERIC(20,10)` 或整数最小货币单位），**绝不用 float**；舍入（rounding）规则（方向、在哪一步舍）写死并测试。

---

## 9. 深挖五：出账（invoicing）幂等与重算

```sql
BEGIN;
  -- 先有对应的 partial unique index；应用检查 RETURNING 结果
  INSERT INTO invoices (id, subscription_id, period_start, period_end,
                        status, currency, fx_snapshot_id)
  VALUES (:id, :sub, :ps, :pe, 'draft', :cur, :fx)
  ON CONFLICT (subscription_id, period_start, period_end)
    WHERE status <> 'voided'
  DO NOTHING RETURNING id;

  -- 只有本事务刚创建了 invoice 时才执行下面两步；若无返回，查询并返回既有 id，
  -- 不能继续拿本次随机 :id 插 line。
  INSERT INTO usage_snapshots(...) SELECT ...;
  INSERT INTO invoice_lines  (...) SELECT ...;
COMMIT;
```
上面是 SQL 形状，不是完整存储过程；实际实现应把“创建者分支”封装成一个事务并加并发测试。**发票编号规则因辖区而异**：有的要求连续、唯一或能解释缺口。编号通常在 finalize 时由专用序列/账簿分配，作废也保留审计记录；不要未经税务确认就宣称全球都必须 gapless，也不要用可回滚的普通序列假装能保证无缺口。

```
draft ──finalize──► open ──payment_succeeded──► paid
  │                  ├──payment_failed × N────► uncollectible
  └──void──► voided  └──void（仅未付时）──────► voided

finalized 之后金额不可变。任何调整只能是 credit_note（贷记单）。
```

### 价格改了要不要追溯（retroactive）？

**默认不追溯。** 已 finalize 的发票是历史事实，改价只影响未来账期。必须调整时：

```
1. dry-run 重算（新价格 + 原 usage_snapshot）
2. diff 报告：受影响发票数、总金额差、按租户分布
3. 人工审批（超阈值需财务 + 法务）
4. 生成 credit note（多收）或 additional invoice（少收），关联原发票
5. 原发票保持不变
```
**"直接 UPDATE 已发出的发票金额"是这道题最严重的错误答案** —— 它破坏会计不可变性，且客户手上的 PDF 和你库里的数字对不上。

### 出账前 sanity check（防定价配置错误）

```
· 本期金额 vs 上期偏差 > ±50%        → 挂起转人工
· 单张发票 > $100,000                → 挂起
· 本轮出账总额 vs 上轮偏差 > ±20%    → 停止整批
（$/M 被写成 $/K 会造成 1000× 跳变，上面三条能兜住）
```
**批量出账必须有"停止整批"开关。** 发出 10 万张错误发票和发出 1 张，成本差三个数量级。

---

## 10. 深挖六：实时配额（quota）检查

配额检查在请求关键路径上：要 **< 5 ms**，又**不能超支**。这两件事在分布式系统里天然冲突。

### 本地令牌 + 周期同步（租约模型）

```
中心配额服务持有租户剩余额度 R
网关实例租借：lease = clamp(R / (2 × active_replicas), MIN_LEASE, MAX_LEASE)
中心发租约时先冻结该片额度；实例本地原子扣减，周期上报单调累计 consumed
归还：只归还中心确认未消费的余额；崩溃/过期租约先封存为 unknown，等待对账

若业务选择“超时后释放 unknown 额度”以换可用性，则新增超支暴露上界约为：
同时失联且会被释放的租约数 × MAX_LEASE = 20 × 10,000 = 200,000 token
          按 $5/M 计 ≈ $1.00      ← 可接受

自适应收敛：R=1,000 万 → lease 25 万（少调中心）
            R=10 万    → lease 2,500
            R=1 万     → lease = MIN_LEASE = 500（接近精确）
```

> **面试金句**
> "我不假装本地租约同时拥有零延迟和零误差。中心预先冻结额度时，正常消费不会超发；真正的取舍发生在实例失联后：保留 unknown 额度会暂时少卖，超时释放会产生可计算的超支上界。业务选择策略，并用租约大小、期限和对账速度调这个旋钮。"

### 双层限额（单层一定会出事）

```
软限额（80% / 100%）：告警 + 邮件 + 看板横幅 + 可选降级到小模型
硬限额（120% 或合同上限）：拒绝新请求，返回 402 / 429 + 明确 error code
Agent 场景额外加：单会话 / 单任务的绝对 token 上限
```
单层硬限额会让大租户在月中直接停服（商业灾难）；单层软限额挡不住失控的 Agent 循环 —— **Agent 循环的 token 方差比人类会话大一个数量级**，绝对上限不是可选项。

### 配额服务不可用时

| 档位 | 策略 | 理由 |
|---|---|---|
| 免费 / 试用 | **fail-close** | 滥用面大，拒绝的商业损失接近 0 |
| 付费、非合同硬上限 | 可按显式风险策略有限 fail-open + 事后补记 | 设时间/金额上界并告警；不能无限放行 |
| 客户要求的合同硬上限 | **fail-close 或只用尚有效的本地租约** | 超支可能违反合同；不能用“付费客户”一刀切 |
| 已欠费 / 冻结 | **fail-close** | 状态缓存在网关本地，不依赖配额服务 |

**这就是计量和配额必须是两条独立管道的原因**：配额挂了，计量还在记，事后能补齐。

---

## 11. 支付、催收、看板与自建边界

**支付网关**：付款请求的幂等键由稳定的 `invoice_id + payment_attempt_no` 派生；同一次网络重试复用 key，新一次经批准的扣款尝试用新 attempt。webhook 是可重放、可能乱序的通知，不是单独的绝对真相：验签后按 provider event id 幂等处理，并用 provider API/结算报表与本地不可变 payment ledger 定期对账。

**催收（dunning）**：`D+0` 通知 → `D+3` 重试 → `D+7` 重试 + 降级（只读/限速）→ `D+14` 重试 + 暂停服务（保留数据）→ `D+30` 标记 uncollectible。
全程叠加"卡即将过期"webhook 提前 14 天提醒 —— **这一条 ROI 最高**，大部分失败扣款其实是卡过期。

**客户看板**：UI 上展示数据新鲜度、水位时间与“账单以关账快照为准”。例如承诺最多延迟 15 分钟，但具体值由窗口、watermark、调度与缓存的实测决定。看板与账单应共享事件身份、计量口径和聚合实现；账单冻结 snapshot，实时看板继续接收迟到事实，所以关账前后可以有可解释的差异。不要维护两条独立计数管道。

### 自建到哪一层（build vs buy）

```
┌─────────────────────────────────────────────────────────────┐
│ 计量（埋点/投递/去重/预聚合）        → 自建                  │
│   与产品语义强耦合；8,700 eps 超过多数计费 SaaS 的入口限额；  │
│   canonical facts 在明确保留期内留在自己手里（争议/重算/审计）│
├─────────────────────────────────────────────────────────────┤
│ 评级 / 发票 / 税 / 多币种 / 催收     → 外购                  │
│   税务合规不是你的核心竞争力，且随辖区持续变化               │
└─────────────────────────────────────────────────────────────┘
```

**2026 年的市场现实**：一年内三起并购 —— [Stripe 完成收购 Metronome](https://stripe.com/newsroom/news/stripe-completes-metronome-acquisition)（2026-01-14）、[Adyen 完成收购 Orb](https://www.adyen.com/knowledge-hub/talon-one-orb-acquisitions)（2026-07-01，$335M 全现金）、[Kong 收购 OpenMeter](https://konghq.com/blog/news/kong-acquires-openmeter)（2025-09）。
⇒ **独立的计量/计费 SaaS 正被支付与网关巨头收编。** 把"被收购后产品方向变化"当成真实的选型风险；剩下的主要独立开源选项是 [Lago](https://getlago.com/)（自建 ingestion，支持 REST/batch/SDK/Kafka/Kinesis/S3 多路接入，去重逻辑可自定义）。

**别指望用量计费（usage-based billing）能修复毛利（gross margin）。** AI 产品平均毛利约 **52%**（传统成熟 SaaS 基准 80–90%，ICONIQ 2026 口径），而这个数字是在大量公司**已经转向用量计费之后**测出来的。真正拉毛利的是模型路由、缓存、蒸馏和规模效应（见 `../04-ai-agent-systems/08-cost-and-latency.md`）；定价只是把成本方差转移给客户。

---

## 12. 什么时候不要这么做

| 情况 | 该做什么 |
|---|---|
| < 100 万事件/天，< 1,000 租户 | Postgres 一张 `usage_events` + 唯一索引去重 + 每日 cron 聚合。**别上 Kafka** |
| 纯席位制订阅，无用量维度 | Stripe Billing 直接接，不需要计量管道 |
| < 1,000 eps 且能接受厂商去重语义 | 直发计费 SaaS 的 meter event 接口。**但仍要在自己这边留一份原始事件** |
| 内部成本归因（不对外收钱） | 误差预算放宽到 5%，采样 + 日志即可，别按计费标准建 |

**十条反模式（anti-pattern）**：

1. **用应用日志计费**。见 §5 的五条理由。
2. **把本地 tokenizer 估算当最终收费事实**。它只能做准入/临时估算；最终按合同计数口径与 provider/自建权威用量对账。
3. **每次重试都新建随机 UUID**。随机 UUID 本身没问题；没有在首次尝试前持久化并复用，才会把一次操作伪装成多次。
4. **在同步请求路径做去重查询**。把计费的可用性绑进产品可用性。
5. **没查当前 batch/stream 配额就把所有原始事件逐条直发计费 SaaS**。超过合同入口能力时再预聚合或扩配额，别把 1k 写成永久通则。
6. **聚合作业用 append 而不是 upsert**。重跑即翻倍。
7. **`UPDATE` 已 finalize 的发票**。破坏会计不可变性，且和客户手上的 PDF 对不上。
8. **单层配额**。月中停服 或 挡不住 Agent 循环，二选一。
9. **看板和账单两套数据源**。必然有一天对不上，且你无法解释。
10. **"最终一致，差一点点无所谓"**。0.5% 丢失在 $5,000 万 ARR 上是 $25 万/年，**且方向永远对你不利**。

---

## 13. 失败模式（failure mode）与演进

| 失败模式 | 信号 | 应对 |
|---|---|---|
| **漏计**：Kafka 不可用，生产者静默丢 | 事件数 / 请求数比值下降 | `acks=all` + spool 兜底；**这个比值是最好的漏计探针** |
| **重计**：消费者 rebalance 重放 | 对账 drift 转正、客诉 | 幂等消费 + 去重窗口吸收 rebalance 期重复 |
| 去重状态膨胀 → 流处理 OOM | 状态大小增长曲线变陡 | 分桶去重 + BF 前置 + 缩短窗口；别靠加内存拖 |
| 定价配置错误（$/M 写成 $/K） | 本期总额跳变 1000× | 出账前 sanity check + 批量停止开关 |
| 月末出账把 DB 打挂 | 1 号凌晨 p99 飙升 | anchor day 打散 + 出账队列限流 |
| 巨型租户单分区热点 | 单分区 lag 持续增长 | 二级分区（见 §6） |
| 时钟漂移（clock drift）导致事件落错桶 | 跨桶漂移在午夜集中 | `event_time` 由服务端赋值（不信客户端时间）+ NTP 监控 |
| 配额与计量不一致 | 客诉"没超却被拒" | 对账租约/事实；人工临时提额需审批、上限、过期和审计，合同硬上限不能被随手绕过 |
| 迟到事件撑破已关闭账期 | `discarded_events` 增长 | 宽限期策略 + 异常队列有人值守 |

```
v0  < 100 万事件/天
    Postgres：usage_events（唯一索引去重）+ 物化视图 + 每日 cron；Stripe Billing 出账。
    撞墙信号：聚合查询 > 30 s；唯一索引写入争用。

v1  < 1,000 eps
    Kafka + 流去重 + 预聚合 1min 桶；原始事件落 S3。评级/发票仍外购。
    撞墙信号：去重状态 > 单机内存；对账 drift > 0.5%。

v2  1,000–10,000 eps   ← 本文
    分层预聚合 + Iceberg 湖 + 独立配额服务 + 三方对账 + 自建评级预览（出账仍外购）。
    撞墙信号：计费 SaaS 入口限流；多币种/多法人实体需求。

v3  多法人实体 / 收入确认（ASC 606）/ 渠道分成
    计费成为独立财务系统，需要专职团队与外部审计。
    信号：审计师开始问"你怎么证明这张发票的数字是对的" —— 答案必须是
          "原始事件 → 快照 → 发票的可重放链路"，而不是"我们的系统很可靠"。
```

---

## 面试官会追问

1. 漏计和重计哪个更严重？你的误差预算分别是多少？用什么机制分别保证？
2. 幂等键由谁生成、何时持久化？为什么“客户端传一个首次生成后始终复用的 UUID”可以是对的，而“每次重试传新 UUID”一定是错的？
3. 去重窗口设多长？这个长度的成本是什么？48 亿个键怎么存？
4. 账期已经关闭，一条 3 天前的事件到了。你怎么处理？这件事需要客户知道吗？
5. 怎么和 provider 的账单对账？允许多大漂移？漂移超了先怀疑谁？
6. 实时配额要 5 ms 内返回又不能超支，怎么做？你的超支上界是多少，怎么算的？
7. 定价改了，上个月的发票要不要重算？如果法务说必须调，你怎么做？
8. 客户看板显示 100 万 token，账单显示 103 万。为什么会发生？怎么从架构上杜绝？
9. 用量计费系统你会自建哪一层、外购哪一层？分界线画在哪，为什么？

---

**按训练路径阅读** → 回 [START-HERE](../START-HERE.md) 按所选路径继续；页尾链接只表示本目录或专章的顺读顺序。

**案例顺读下一篇** → [05-realtime-collaboration.md](05-realtime-collaboration.md)
