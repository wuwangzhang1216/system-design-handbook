# 02 · 计量（metering）与计费（billing）

> 计量系统的误差是**有方向的**：丢事件永远是你少收钱，重复事件永远是客户投诉。
> 所以架构目标不是"准确"，是"**在两个方向上都有可审计的收敛机制**"。
>
> **阅读分层**：§2–§7 是 Core，AI 定价矩阵与结果型定价是进阶。带厂商名、价格、吞吐与法规的数字是 **2026-08 时间快照/示例**，实现或签约前必须复核现行官方条款。

---

## 读这一章之前

**你在工作中遇到过这个**

一个客户的 Agent 在凌晨 2 点陷进了工具调用死循环。你的预算告警是按天聚合跑的，第二天早上 9 点才响 —— 那时候这个租户已经烧掉 $4,200，是他整月套餐费的 14 倍。
你想打开账单让他看清楚钱花在哪一次调用上，才发现系统里只留了"按天 × 按租户"的聚合数，原始调用记录 7 天前就被清掉了。
这笔钱最后你没敢收，送额度了事。这一章讲的就是为什么护栏、归因、原始事件这三件事必须在你上线第一版计费之前就摆好。

**需要先懂的概念**

| 概念 | 一句话 | 详见 |
|---|---|---|
| 幂等 / 幂等键 | 同一操作做多次和做一次效果相同；幂等键就是用来识别"这是同一次操作"的那个标识 | [00-concepts §12](../00-foundations/00-concepts.md#12-本章术语速查)、[00/01 §5](../00-foundations/01-fundamentals.md#5-幂等idempotency分布式系统的第一公民) |
| 最终一致 | 停止写入后所有副本最终收敛，中间可能读到旧值，且没有时间上界 | [00-concepts §6](../00-foundations/00-concepts.md#6-什么是一致性--一个词两种完全不同的意思) |
| 背压 | 处理不过来时向上游反向施压（拒绝或阻塞），而不是无限排队 | [00/01 §6](../00-foundations/01-fundamentals.md#6-背压backpressure没有它系统就会雪崩cascading-failure) |
| 日志型总线与分区键 | Kafka 这类只追加、可重放的消息总线；分区键决定一条消息进哪个分区，同分区内有序 | [01/03 §1–2](../01-building-blocks/03-messaging-and-streams.md#1-队列-vs-日志两种根本不同的东西) |
| 水位线与迟到事件 | 水位线是流处理对“事件时间已经推进到哪里”的单调估计；一条事件到达时，如果它的事件时间已落在当前水位线之前，就属于迟到。水位线不等于 allowed lateness | [01/03 §6](../01-building-blocks/03-messaging-and-streams.md#6-流处理语义与状态) |
| 租户 / tenant_id | 一套系统同时服务多个互不可见的客户组织，tenant_id 是贯穿所有数据的隔离维度 | [02/03](../02-architecture-patterns/03-multi-tenancy.md) |

**这一章要回答的问题**

1. 客户端超时重试了三次，为什么"我们用了幂等键"这句话救不了你？
2. 一条 5 天前的用量事件在账期关闭之后才到，这笔钱算谁的？有哪几种合法处理方式？
3. 100 个网关实例共享同一个租户的 token 配额，怎么做到不是每个请求都去问中心？误差上界怎么算？
4. 一次 LLM 调用到底有几个不同单价的计量项？为什么"我们按 token 收费"这句话本身就不完整？

**本章新引入的术语**

| 术语 | English | 一句话定义 |
|---|---|---|
| 计量 | metering | 把"谁在什么时候用了多少"记录成可计价的事件与数字的过程，发生在算钱之前 |
| 评级 | rating | 拿一段时间的用量数字去套价格表、算出金额的那一步纯计算 |
| 出账 | invoicing | 把算出的金额组装成一张有编号、有状态、可以拿去收款的账单 |
| 用量计费 | usage-based pricing (UBP) | 按客户实际用了多少收钱，而不是按人头或固定月费 |
| 阶梯定价 | tiered pricing | 用量落在不同区间时适用不同单价（前 100 万个单位 $5，之后 $3） |
| 去重窗口 | deduplication window | 系统还记得"这个事件 ID 我见过"的那段时长；超出它的重复事件无法被识别 |
| 允许迟到期 | allowed lateness | 水位线首次越过窗口末端后，窗口状态还保留多久、迟到事件还能否修正结果的策略；它不参与计算水位线 |
| 补收 | true-up | 把本期算漏的用量放进下一期账单补上，而不去改已经出过的账单 |
| 对账 | reconciliation | 拿两份本应相等的数字逐笔比对，并要求每一条差异都能被解释 |
| 成本护栏 | cost guardrail | 在钱花出去之前就能拦住请求的硬性上限，与"事后告警"相对 |
| 预扣与结算 | reservation / settlement | 先按估计值扣掉一笔额度，等真实用量出来再退回差额或补扣 |
| 结果型定价 | outcome-based pricing | 按"办成了几件事"收钱，而不是按消耗了多少资源 |

---

## 1. 为什么 2026 年这件事变难了

传统 SaaS 按席位收费（per-seat pricing）：计量 = `SELECT count(*) FROM users`，一个月跑一次，错了也没人发现。AI 产品把三件事同时打破了：

| 变化 | 后果 |
|---|---|
| **推理成本进了 COGS**（cost of goods sold，营收成本：为交付服务直接花掉的钱，营收减去它就是毛利） | AI 产品平均毛利（gross margin）约 **52%**，传统成熟 SaaS 是 **80–90%**（[ICONIQ 2026 State of AI](https://www.iconiq.com/growth/reports/2026-state-of-ai-bi-annual-snapshot)，n≈300）。毛利薄 ⇒ 计量误差直接吃掉利润 |
| **单位从"人"变成"事件"** | 事件量从 10³/月 变成 10⁹/月，计量管道从财务后台变成**实时数据基础设施** |
| **成本方差（cost variance）极大** | 同一个"用户提问"，成本可以差 1,000 倍（一次 Haiku 单轮 vs 一次 Opus 多 Agent 长任务） |

行业用并购给这件事投了票：一年内 **Stripe 收 Metronome（2026-01-14 完成）**、**Adyen 收 Orb（$335M，2026-07-01 完成）**、**[Kong 收 OpenMeter](https://konghq.com/blog/news/kong-acquires-openmeter)（2025-09）**。剩下的主要独立开源选项是 [Lago](https://getlago.com/)。定价模式分布（可多选，ICONIQ 2026）：订阅/平台费 **58%**、消费型（consumption-based）**35%**、结果型 **18%**；**37%** 的公司计划在未来 12 个月改定价。

⚠ **别把"改成用量计费（usage-based pricing, UBP）"当成毛利解药。** 52% 这个数字是在大量公司**已经转 UBP 之后**测出来的。真正拉毛利的是模型路由（model routing）、缓存、蒸馏（distillation）和规模效应（economies of scale）；定价只是把成本方差转移给客户。

---

## 2. 端到端管道

```
  数据面（产生）        总线（可靠投递）       流处理（去重+聚合）        计费域（评级+出账）        财务
┌──────────────┐      ┌───────────────┐     ┌─────────────────┐     ┌────────────────┐   ┌──────────┐
│ LLM 网关      │──┐   │               │     │ ① 去重          │     │ ⑤ Rating       │   │ ⑦ 对账    │
│ 沙箱 / 工具层 │──┼─▶ │ Kafka/Kinesis │ ──▶ │   (source,id)   │ ──▶ │   套价格表      │─▶ │ ⑧ 争议    │
│ API / 存储    │──┘   │ 分区键=tenant │     │ ② 富化(plan/价) │     │ ⑥ Invoicing    │   │ ⑨ 退款    │
└───────┬──────┘      └───────┬───────┘     │ ③ 窗口聚合      │     │   幂等出账      │   └──────────┘
        │ 本地 WAL 落盘        │             │ ④ 水位线/迟到    │     │   → 支付网关    │
        │ （断网也不丢）        │             └────────┬────────┘     └────────────────┘
        │                     │                      │
        │                     ▼                      ▼
        │            ┌────────────────────────────────────────────────┐
        └─── 反压 ◀──│  原始事件湖（S3 + Iceberg，保留 13–25 个月）      │
                     │  ← 争议举证、重算（replay）、审计的唯一真相源     │
                     └────────────────────────────────────────────────┘
```

**每一步的失败模式（这张表是这一篇的核心）：**

| # | 步骤 | 失败模式 | 方向 | 对策 |
|---|---|---|---|---|
| 0 | 事件产生 | 进程崩溃、事件还在内存 | **少收钱** | 在返回“业务已完成”前，把 usage 与业务结果原子写入 outbox；无业务库的网关才使用可恢复、受监控的 WAL spool。流式请求需另行定义中断与估算语义 |
| 0 | 事件产生 | 客户端在每次重试时重新生成 UUID | **多收钱** | 首次发送前生成并持久化稳定 operation key，所有重试复用；事件 ID 再由它派生，见 §4 |
| 1 | 投递 | 总线不可用 | 少收钱 | 本地磁盘缓冲 + 有界队列（bounded queue）+ **反压（backpressure）到业务路径的开关必须存在但默认关** |
| 2 | 去重（deduplication） | 重复超出去重窗口 | 多收钱 | 去重窗口 = 你能容忍的客户端重放延迟上限 |
| 3 | 聚合 | 迟到事件（late-arriving events）晚于窗口关闭 | 少收钱 | 允许迟到期 + true-up，见 §5 |
| 4 | 富化（enrichment） | 用了**出账时**的价格而不是**事件时**的价格 | 双向 | 价格表按 `effective_from` 版本化，评级时按事件时间取版本 |
| 5 | 评级 | 阶梯（tiered pricing）/包量计算浮点误差累积 | 双向 | 全程整数（最小计费单位），**绝不用 float 存金额** |
| 6 | 出账 | 重复出账 | 多收钱 | `(tenant, period, invoice_version)` 唯一约束（unique constraint） |
| 7 | 对账 | 与上游 provider 账单对不上 | 少收钱 | 三方对账，见 §9 |

**两条结构性原则：** ① **计量处理与业务请求解耦，但事件产生不能与业务事实脱节**：优先在业务提交事务内写 outbox，再异步投递；不要在同步路径调用计费 SaaS。② **不可变原始计量事件是计费侧的规范事实层，聚合结果可重算**；它仍要与业务记录和供应商 usage/账单对账，不能把事件湖称作整个业务的唯一真相。口径变更、价格追溯和争议靠带版本的 replay + 对账共同解决。

---

## 3. AI 用量事件的 Schema

这是 AI 计费和传统计量最大的差别：**一次调用产生的不是一个数，是一个向量**。

| 字段 | 类型 | 说明 / 为什么必须有 |
|---|---|---|
| `id` | uuid | 幂等键。与 `source` 组成去重主键（[CloudEvents](https://cloudevents.io/) 惯例） |
| `source` | string | **稳定的逻辑生产者命名空间**，如 `urn:acme:llm-gateway`；不得含 pod / 进程 / 重试目标。去重键是 `(source, id)`，部署位置变化不能改变它 |
| `type` | enum | `llm.completion` / `tool.call` / `sandbox.session` / `search.query` / `storage.gb_hour` |
| `time` | ts(µs) | **事件时间（event time）**，用于窗口聚合（windowed aggregation）与价格版本选取 |
| `ingest_time` | ts | 到达时间。`ingest_time - time` = 迟到程度，必须监控其 p99 |
| `tenant_id` | string | 分区键（partition key） |
| `workspace_id` / `user_id` | string | 二级归因（attribution）维度 |
| `feature` | string | **业务功能**（`code_review` / `chat` / `batch_index`）。没有这个维度就无法回答"哪个功能在烧钱" |
| `session_id` / `run_id` / `agent_id` | string | Agent 场景的成本归因与**单会话硬上限**依据 |
| `provider` / `model` | string | `anthropic` / `claude-opus-5` |
| `model_version` | string | 快照版本。价格和 tokenizer 都跟版本走 |
| `tokenizer_version` | string | 本地估算或预算使用了 tokenizer 时必须记录。模型迁移后同样文本的 token 数可能显著变化；“约 +30%”只可能是特定语料/版本的观测，不能外推。最终计费仍优先使用 provider 返回的 usage 与计量版本 |
| `input_tokens` | int64 | **未命中缓存**的输入 |
| `cache_read_tokens` | int64 | 缓存读（cache read），单价约为输入的 **10%** |
| `cache_write_5m_tokens` | int64 | 5 分钟 TTL 缓存写，**1.25×** 基础输入价 |
| `cache_write_1h_tokens` | int64 | 1 小时 TTL 缓存写，**2×** 基础输入价 |
| `output_tokens` | int64 | 输出 |
| `reasoning_tokens` | int64 | 思考 token（reasoning tokens）。通常按输出计价，但**产品上是否向客户暴露是独立决策** |
| `service_tier` | enum | `standard` / `batch`（**50% off**）/ `fast`（Opus 5 为 **2× 标准价**，换 ≤2.5× 输出 tok/s，**不能与 batch 同用**）/ `priority` |
| `inference_geo` | string | 数据驻留（data residency）。Anthropic `inference_geo:"us"` 为 **1.1×** 全项；OpenAI 区域化处理 **+10%** |
| `outcome` | enum | `success` / `client_error` / `provider_error` / `client_disconnect` |
| `billable` | bool | 由 §8 的策略计算得出，**必须显式存**，不要在出账时临时判断 |
| `provider_usage_ref` | string? | 可选：供应商返回的 usage / request 引用，用于三方对账；原始事实事件不烧入会随时间变化的价格 |
| `schema_version` | int | 事件格式演进 |

> 单价示例来自各厂商定价页的历史时间快照（[Anthropic](https://docs.claude.com/en/docs/about-claude/pricing) / [OpenAI prompt caching](https://platform.openai.com/docs/guides/prompt-caching) / [Gemini caching](https://ai.google.dev/gemini-api/docs/caching)）。它们只用于展示 schema 为什么要拆维度；**不得直接当作现行报价或合同口径**。

**三个非显然的设计点：** ① **遥测记录事实，不记录估值**：事件保存分项 usage、实际 provider/model、发生时间与可对账引用；成本由后端用 `(provider, model, service_tier, event_time, price_version)` 查不可变价目表派生，既能近实时算毛利，也能在合同价修正后重算。② 对转售托管 API 的正式账单，以 provider 返回的 `usage` 为准；本地 tokenizer 只能在流中断时作为带 `usage_source=estimated` 的暂估，不能冒充供应商事实。③ **成本口径与客户计费口径要分离**：provider usage 是成本输入，`billable_units` 是卖给客户的单位，两者可以完全不同。混在一起的系统改一次定价就要动整条管道。

---

## 4. 去重与幂等：最容易讲错的一节

**事实标准**：去重键 = CloudEvents 的 `source` + `id`。[OpenMeter 默认去重窗口 **32 天**](https://openmeter.io/blog/usage-deduplication)，架构是 **API Server 无脑写 Kafka（不做唯一性检查），去重放在流处理阶段，最终一致** —— 即短时间内可能存在重复。

**工程含义：去重窗口 = 你能容忍的客户端重放延迟上限。** 32 天不是免费的，它是流处理状态存储的成本：

```
去重状态大小 ≈ 事件速率 × 窗口 × 键大小(≈40B) / 压缩比
10,000 events/s × 32 天 × 40 B ≈ 1.1 TB 原始键空间
⇒ 实践：两级去重
   L1  Redis / RocksDB Bloom + 精确集，窗口 24–72 h，抓 99.9% 的重复
   L2  批处理（每日）在事件湖上做全窗口精确去重，产出修正
```

### 那个致命误解

> ❌ **"用了幂等键就不会重复计费"**

幂等只保证**同一个 key 不重复入账**，它**不保证客户端在重试时复用了同一个 key**。真实的失效模式是：客户端超时重试，每次都重新调用 `uuid4()` ⇒ 三次重试 = 三条事件 = 三倍收费，而且**你的幂等层完全正常工作**。随机 UUID 本身完全可以用；错的是把“每一次网络尝试”都铸成一个新业务操作。

**正确做法：发起方在首次发送前生成并持久化 operation key，且在同一次业务操作的所有重试中复用；计量事件 ID 再由这个稳定身份与收费步骤派生。**

```python
# ❌ 错：这个函数在每次 retry attempt 里重新运行
client_operation_key = uuid4()

# ✅ 对：调用方/SDK 在首次发送前生成一次并持久化；随机 UUID 可以用
client_operation_key = operation_store.load_or_create(client_action_ref)
# 之后每次 HTTP/provider 重试都读取同一个 key；semantic_step 区分同一操作里的不同收费事实
event_id = uuidv5(NS, f"{tenant_id}:{client_operation_key}:{meter}:{semantic_step}")
```

如果调用方没有稳定 operation key，网关每次入口生成一个新 `request_id` **无法识别 HTTP 级重试**；它只对该次请求内部的 provider 重试稳定。支持幂等键的上游应透传同一个键；不支持的上游在“超时但可能已接受”时存在歧义，必须查状态/对账，不能宣称 exactly-once。

**事件顺序**：不要依赖它。分区键选 `tenant_id` 保证同租户有序即可，跨租户无序无所谓。真正需要顺序的是**同一 `session_id` 内的预扣/结算配对**，把它们放同一分区。

---

## 5. 聚合、水位线与迟到事件

```
事件时间轴 ──────────────────────────────────────────────────────▶
   [窗口 W: 2026-07-01 00:00 – 2026-08-01 00:00]
                                        │
   正常事件 ●●●●●●●●●●●●●●●●●●●●●●●●●●●●│
   迟到事件         ○(+2h)      ○(+30h) │        ○(+5d)      ○(+40d)
                                        │                        │
                              窗口结束 ──┤                        │
                     allowed lateness ──┼──▶ (24–72 h) ──┤        │
                              账期关闭 ──────────────────┤ T+3d   │
                                                        │        │
                       计入本期 ─────────────────────────┘        │
                       计入下期（true-up）────────────────────────┘
                       超出去重窗口(32d) ⇒ 无法安全计入，只能丢弃 + 告警
```

**先把两个经常被混写的参数分开**：watermark 是“事件时间进度”的估计；allowed lateness 是“首次出结果后还愿意保留状态、接受修正多久”的产品与存储策略。不能写成 `watermark = max(event_time) - allowed_lateness`。本题可用下面的起点，再按生产延迟分布调参：

```text
每个活跃分区 p：
  wm_p = monotonic(max_valid_event_time_seen_p - out_of_order_bound)
全局窗口水位线：
  wm_global = min(wm_p for p in active_partitions)
```

`out_of_order_bound` 来自同一数据面链路的乱序/投递延迟观测（例如 p99.9 再留安全带），不是 allowed lateness。取所有**活跃**分区的最小值，是因为任意一个慢分区都可能还有更早事件；用全局最大值会让快分区替慢分区提前关窗。分区扩缩或 rebalance 时要随 checkpoint 恢复各自单调水位，不能从当前第一条消息重新起算。

**空闲分区与未来时间戳是两个必须显式处理的坑**：

- 一个长期无流量的分区会把 `min(...)` 永久钉住。只有在消费位点已追平、连续超过配置的 idle timeout 且来源健康时，才把它标成 idle、暂时排除；恢复流量后，其旧事件按迟到策略处理并告警，不能偷偷改写已经关闭的账期。
- 单条未来时间戳会把 `max_event_time` 推到未来。计量事实应优先由受信服务端赋 `event_time`，并校验 `event_time <= ingest_time + max_future_skew`；越界事件隔离/修正并告警，绝不能参与水位推进。历史 backfill 走单独标记的回填通道，也不能冒充实时分区进度。

当 `wm_global` 首次越过窗口末端时**发出初步结果**；此后在 allowed lateness（例如 24–72 h）内到达的事件产生**增量修正（retraction + 新值）**。allowed lateness 到期可以清理流处理状态，但财务上的最终截止仍是账期关闭；两者可以不同。若窗口状态已清而账期尚未关闭，后来的事件要走从 canonical facts 重算该桶/账期的批量修正路径，不能假装流式状态还在。**账期关闭后的迟到事件只有三种处理方式，必须三选一并写进产品条款：**

| 策略 | 优点 | 代价 | 适用 |
|---|---|---|---|
| **丢弃** | 简单，账单不可变 | 少收钱，且金额不可控 | 事件量小、迟到率 < 0.01% |
| **计入下期（true-up）** | 不动历史账单，会计上干净 | 客户看到"上月的用量出现在这个月" | **推荐默认** |
| **重开账期（rebill）** | 最准确 | 需要作废+重开发票，税务/审计成本高 | 单笔金额超过阈值（如 $500）时触发 |

⚠ **公开资料里找不到主流计费厂商对"账期关闭后迟到事件如何处理"的官方 SLA。** 选型时必须实测，不要相信文档没写的东西。

**至少监控这几类指标**：`watermark_lag`（按分区看，避免全局最小值掩盖坏分区）、`idle_partition_count/transitions`、`future_timestamp_quarantined`、`late_event_ratio`（突增通常表示某个数据面 cell 的投递链路堵了）和 `revenue_adjustment_ratio`（true-up 金额 / 总收入；持续偏高才说明 allowed lateness 或关账策略需要重估）。健康阈值必须从自己的历史分布与误差预算推导，不把示例百分比当通则。

---

## 6. 评级（Rating）与出账（Invoicing）的幂等

**评级**：把 `(tenant, meter, period, quantity)` 套价格表算出金额。三条铁律：

1. **金额全程用整数最小单位**（micros，10⁻⁶）。float 在阶梯定价上累积误差，且不同语言的舍入行为不一致。
2. **价格表按 `effective_from` 版本化**，评级时按**事件时间**取版本。租户升级套餐时用**按比例分摊（proration）**，不要按出账时的套餐重算整个月。
3. **评级必须是纯函数（pure function）**：`rate(usage_snapshot, price_version) → line_items`。同样输入永远同样输出，才能在争议时复现三个月前的账单。

**出账的幂等模型：**

```sql
CREATE TABLE invoices (
  tenant_id     text NOT NULL,
  period        daterange NOT NULL,
  version       int NOT NULL DEFAULT 1,     -- 每次 rebill +1，旧版本作废不删除
  idem_key      text NOT NULL,              -- = sha256(tenant|period|version|usage_snapshot_hash)
  usage_hash    text NOT NULL,              -- 用量快照哈希，用于判断"要不要重开"
  total_micros  bigint NOT NULL,
  status        text NOT NULL,              -- draft|finalized|paid|void
  PRIMARY KEY (tenant_id, period, version),
  UNIQUE (idem_key)
);
```

**"恰好一次计费"（exactly-once billing）的真相**：消息系统可在自己的事务作用域内提供 exactly-once processing，但跨计量、账本与支付渠道后，常见业务保证是 `至少一次投递 + 明确窗口内的幂等应用 + 可对账收敛`。所以必须先声明边界，不能只说一句“端到端 exactly-once”。**支付环节**要用稳定业务操作 ID，并先持久化支付意图再调用渠道；否则网络超时时无法判断钱是否已扣。

---

## 7. 配额与限额

### 四种额度，四种算法（别混用）

| 类型 | 语义 | 算法 | 典型 |
|---|---|---|---|
| **速率**（rate） | 单位时间的量 | 令牌桶（token bucket：以固定速率往桶里放令牌，每个请求取走一个，桶空就拒绝）/ 滑窗（sliding window） | RPM、TPM |
| **累计量**（counter） | 周期内总量 | 分布式计数 + 租约 | 月 token 额度、预算 $ |
| **并发**（gauge） | 同时在跑的数量 | **信号量（semaphore：一个有上限的计数器，占用时 +1、释放时 −1，到顶就拒绝）+ 租约（lease：带过期时间的占用凭证，持有者必须靠心跳续期，不续就自动作废归还，见 [01/05 §4](../01-building-blocks/05-consensus-and-coordination.md#4-租约lease比锁更好的抽象)）+ TTL 心跳（heartbeat）** | 并发沙箱数、并发 Agent 数 |
| **绝对上限**（cap） | 单个实体的硬顶 | 本地即可判定 | 单会话最大 token、单请求最大输出 |

⚠ **并发配额不能用令牌桶。** 并发是 gauge（当前值，随请求开始和结束上下浮动）不是 counter（只增不减的累计值）：进程崩溃会让计数永久泄漏。必须用带 TTL 的租约 + 心跳续期，崩溃后租约自动过期归还。这是配额系统里最常见的实现错误。

### 分布式近似计数（approximate distributed counting）：本地额度租片 + 中心结算

核心思想：**每个实例向中心预留一段可消费额度，本地扣减，周期上报累计消费。** 注意它和“并发槽租约”不同：实例死掉意味着并发槽结束，可以自动释放；已经发出的 token / 金额是消耗品，TTL 到期**不能把整段额度自动复活**。

```python
# 概念伪代码：省略了本地原子锁、RPC 重试、持久化 schema 与鉴权，不能直接粘贴运行。
class QuotaSlice:
    def refresh(self, need):
        # 中心在同一事务中：检查 remaining、扣除 grant、创建 (slice_id, epoch, grant)。
        # grant 一经发出就已从全局可用额度中预留，多个实例不能重复拿到。
        self.slice = quota.acquire_slice(
            tenant=self.tenant, meter=self.meter,
            want=self.target_size(need), ttl=LEASE_TTL)
        self.consumed = 0
        self.reported = 0

    def try_consume(self, n):
        if self.slice.remaining() < n:
            self.refresh(n)
        # 生产实现必须是进程内原子操作。
        self.consumed += n
        return ALLOW

    def heartbeat(self):
        # 中心只接受同 slice_id + epoch 的单调累计值，重复 RPC 不会重复结算。
        quota.report_cumulative(
            self.slice.id, self.slice.epoch, consumed_total=self.consumed)
        self.reported = self.consumed

    def close(self):
        # 只有正常关闭且最终累计值已确认时，才归还明确未消费的部分。
        quota.close_slice(
            self.slice.id, self.slice.epoch, consumed_total=self.consumed)
```

中心侧在 slice 过期时将它 **seal**，拒绝旧 epoch 的迟到上报。对 hard cap，崩溃后无法证明未消费的部分继续保持预留（或按全额消费处理），再由独立计量事件对账；不能因 TTL 到期就全部归还。对 soft cap 才可以选择延迟归还并接受可计算的超发风险。这样换来的误差是“短时少放行/额度暂时被占”，而不是已消费额度凭空复活。

**参数示例（必须按业务成本与容忍误差重算）：**

```
REFRESH_INTERVAL   1–10 s        越短越准，越长越省
LEASE_TTL          3 × REFRESH_INTERVAL
MIN_LEASE          够 1 个典型请求用（LLM 场景 ≈ 单请求 p95 token 数）
MAX_LEASE          GLOBAL_Q × 5%   ← 防止一个实例囤走大部分额度
软限额超发/硬限额暂占上界  ≤ 同时未结算 slice 数 × MAX_LEASE
长尾租户           QPS < 1 的租户直接走远程强一致，不租约（它们不值得优化）
```

### LLM 专属：预扣与结算（reservation）

LLM 的输出长度**在请求发出时未知**，所以配额扣减必须分两步：

```
1. 预扣  reserve = input_tokens + min(max_tokens, p95_output_tokens_for_this_route)
2. 执行  ...
3. 结算  settle(reserve, actual) → 退回差额（或补扣，若 actual > reserve）
4. 兜底  流式请求断开 / 进程崩溃 ⇒ 预扣记录带 TTL，超时自动按已观测到的 token 结算
```

⚠ **按 `max_tokens` 全额预扣是常见的过度保守**：实际输出通常只用到 `max_tokens` 的 20–40%，全额预扣会让租户的配额在真正用完之前就显示耗尽。用 p95 分位预扣 + 允许小幅超发，比精确但过紧好得多。

---

## 8. AI 产品计费的特殊难题

### a) 单价矩阵，不是单价

一次 LLM 调用最多有 **7 个不同单价的计量项**。以 Claude Opus 5 为例（2026 年中量级，随时变动）：

| 项 | 相对基础输入价 | Opus 5 实价 |
|---|---|---|
| 未缓存输入 | 1× | $5 / M |
| 缓存读 | **0.1×** | $0.50 / M |
| 缓存写（5m TTL） | 1.25× | $6.25 / M |
| 缓存写（1h TTL） | 2× | $10 / M |
| 输出（含 reasoning） | 5× | $25 / M |
| Batch 模式 | **全项 ×0.5** | 输入 $2.50 / 输出 $12.50 |
| Fast 模式 | **全项 ×2**，不可与 Batch 叠加 | 输入 $10 / 输出 $50 |

**叠加极值（stacked discounts）**：Batch + 缓存读 = `$5 × 0.1 × 0.5 = $0.25/M`，是同步未缓存价的 **1/20**。你的计费系统必须能表达这个组合，否则你会把成本优化的收益全部漏算。

三个跨厂商的坑：**① 缓存读折扣按代际不同** —— OpenAI GPT-5.x 是输入价的 10%，gpt-4.1/o3/o4-mini 是 25%，gpt-4o 是 50%；价格表必须按 `(provider, model, generation)` 三元组索引。**② 缓存写从免费变成收费** —— OpenAI GPT-5.6+ 的缓存写是 1.25× 未缓存输入价（此前免费），老代码里"缓存写不计费"的假设会静默漏计。**③ Gemini 有持有成本** —— $1.00 / 百万 token / 小时的缓存存储费是三家里唯一按时间收的，你的成本模型里因此多了一个 `∫ cached_tokens dt` 项而不是一个事件。

### b) 成本归因：从租户到功能到会话

```
每条 LLM 事件必须携带的归因维度（缺一个就少一个可回答的问题）：
  tenant_id   → "哪个客户在亏钱"                 → 单位经济模型
  feature     → "哪个功能在烧钱"                 → 产品决策
  session_id  → "哪一次会话失控了"               → 熔断依据
  agent_id    → "哪个子 Agent 在空转"            → 多 Agent 系统必需
  model       → "路由策略有没有生效"             → 降本验证
  cache_*     → "缓存命中率是多少"               → 最大的单点优化杠杆
```

**Agent 场景的方差（variance）是关键**：单 Agent 的 token 用量约为 chat 的 **4×**，多 Agent 系统约 **15×**（Anthropic，2025-06-13）。参考量级：Claude Code 官方口径 **$13 / 开发者 / 活跃日**、**$150–250 / 月**，P90 < $30/活跃日（⚠ 另有二手来源称 $6/日，口径不同，以官方现行文档为准）。**这个方差决定了你不能只做租户级预算** —— 一个失控的 Agent 循环可以在 20 分钟内烧掉一个租户一个月的额度，而月度预算只会在事后报警。

### c) 成本护栏（cost guardrail）：三层，缺一不可

```
L1  单请求/单会话绝对上限（本地即可判定，零延迟）
    max_tokens_per_request、max_tokens_per_session、max_tool_calls_per_run、max_wall_clock
    ⇒ 这一层挡住 Agent 死循环。**没有这一层的 Agent 平台早晚出一张天价账单。**

L2  软预算（达到 70% / 90% 触发）
    → 告警租户 + 自动降级：Opus → Sonnet/Haiku、关闭多 Agent 扇出
      （fan-out：一个请求向下派生出 N 个并行子请求，见 00/01 §7）、
      降低 reasoning effort、强制走 Batch 通道（50% off，但延迟到小时级）
    ⚠ 未找到主流厂商对"软限额降级到小模型"的公开量化效果（成本降 % / 质量降 %），
      只有定性描述。**上线前必须自己在生产流量上做 A/B。**

L3  硬预算（100%）
    → 拒绝新请求（429 + 明确的 error code 和恢复条件），已在跑的任务允许跑完
    ⇒ 硬熔断必须是 fail-closed 的（见 §7 的 mode=HARD）
```

**单层预算是反模式**：单层硬限额（hard limit）会让大租户在月中直接停服（销售会来找你）；单层软限额（soft limit）挡不住失控 Agent 循环（财务会来找你）。**必须双层 + L1 的绝对上限。**

### d) 必须显式决策的一条策略：失败请求是否计费

| outcome | provider 是否向你收费 | 你是否向客户收费 | 说明 |
|---|---|---|---|
| `success` | 是 | 是 | — |
| `client_error`（4xx，如 prompt 超长） | 通常否 | 否 | — |
| `provider_error`（5xx / 超时） | 通常否 | **否**（推荐） | 但你的重试可能已经产生了成本 |
| `client_disconnect`（流式中途断开） | **是**（已生成的 token） | **必须显式决策** | 这是最容易漏的一项 |
| 内容安全拦截 | 是 | 通常是（已消耗输入） | 需在条款里写明 |

> **面试金句**：
> "失败请求是否计费，我会把它当成**产品条款而不是实现细节**来设计。不计费对租户体验好，但制造了滥用面 —— 攻击者可以用必然失败的请求把我的上游成本打满而自己零成本；计费更公平，但一次我方 5xx 就会变成客诉。我的做法是按 outcome 分档：客户端错误和我方错误不计费，**流式中途断开按已生成 token 计费**，因为 provider 确实向我收了这笔钱。关键是这三条必须写进文档并对租户可见 —— 计费系统里所有'我们内部决定一下'的策略，最后都会变成争议工单。"

### e) 结果型定价（outcome-based）的计量难题

行业实例（2026 年中量级）：[Intercom Fin](https://www.intercom.com/learning-center/ai-customer-service-agent-pricing-comparison) **$0.99 / outcome**，一次会话只计一个 outcome，每月最低 50 个；Salesforce Agentforce Flex Credits **$500 / 100k credits = $0.005/credit**，一个三动作工单约 **$0.30**。

它把计量难度提高了一个量级，因为要回答三个**主观**问题：**什么算一个 outcome**（用户没再回复算解决吗）、**谁来判定**（规则 / LLM-as-judge / 人工抽检 —— judge 的偏差会直接变成收入偏差，见 [06-evaluation-and-observability.md](../04-ai-agent-systems/06-evaluation-and-observability.md)）、**判错了怎么争议**。

**工程结论**：结果型定价的账单必须**可解释到单条 outcome**，每条能点开看到触发它的完整轨迹（trace）。做不到就不要卖结果型定价 —— 第一次大客户争议会让你手工重算整个月。

---

## 9. 对账（Reconciliation）

**三方比对（three-way reconciliation），缺一不可：**

```
        ① Provider 侧账单/用量 API（真金白银的口径）
                     ▲
                     │  日度比对，差异 = 你漏记的调用 / 重试放大
                     ▼
        ② Gateway 侧访问日志（每次上游调用的 usage 字段）
                     ▲
                     │  实时比对，差异 = 事件投递丢失
                     ▼
        ③ 应用侧计量事件（进了计费管道的）
                     ▲
                     │  月度比对
                     ▼
        ④ 已开发票金额
```

| 比对对 | 频率 | 差异含义 | 目标误差率 |
|---|---|---|---|
| ① vs ② | 日（T+1） | 你的网关漏记了上游调用；或有绕过网关的直连 | < 0.1% |
| ② vs ③ | 实时（分钟级） | 事件投递链路丢事件 | < 0.05% |
| ③ vs ④ | 月度 + 账期关闭时 | 评级/出账逻辑 bug | **0**（差异必须能逐笔解释） |

**为什么 0.5% 的丢失率不可接受**：token 计费下，0.5% 的事件丢失在 $10M ARR 上就是 **$50k/年**的无声漏损（revenue leakage），**且方向永远对你不利**（丢事件 = 少收钱；重复事件会被客户投诉纠正 —— 系统性偏差（systematic bias）是单向的）。**对账的产物必须是差异明细而不是一个百分比**，超阈值时自动开工单并附样本事件 ID；只报总数的对账等于没有对账。

---

## 10. 退款、信用额度与争议

| 机制 | 会计语义 | 实现要点 |
|---|---|---|
| **信用额度（credit）** | 预付余额（prepaid balance），先于计费扣减 | 有优先级（促销额度先扣、有到期日）；扣减顺序必须确定且可解释 |
| **调整（adjustment）** | 对已出账单的**追加行项** | 用于 true-up。**不改原发票**，加一行 |
| **贷记单（credit memo）** | 冲销已开发票的一部分 | 税务上是独立单据，不能靠"改发票金额"实现 |
| **退款（refund）** | 真实资金流出 | 必须幂等（带 `refund_idem_key`），且与支付网关（payment gateway）对账 |

**两条不能省的规则**：① 发票一旦 `finalized` 就**不可变（immutable）**，所有修正走新单据（调整/贷记单/新版本发票）—— 可变发票的系统在审计时会被直接判死。② 信用额度扣减必须记录在事件流里、和用量事件一样可 replay；手工改余额必须走 break-glass 通道并留审计。

---

## 11. 什么时候不要这么做

| 情况 | 别做 | 做什么 |
|---|---|---|
| 事件 < 100/s，只做订阅制 | 不要自建流式计量管道 | 直接用计费 SaaS 的 API，每天批量推送 |
| **> ~1k events/s** | 不要直发计费 SaaS | 自己这一侧做**预聚合（pre-aggregation）**（按 `tenant × meter × 时间桶`），原始事件留在自己的数据湖。参考量级：[Stripe Billing](https://docs.stripe.com/api/billing/meter-event) Meter Event 端点 live mode **1,000 calls/s**，Meter Event Stream API 单流 **10k events/s**、单商户可到 **100k events/s** |
| 计费口径还在每月改 | 不要固化聚合层 | 只固化**原始事件 schema**，聚合和评级保持可重算。口径没稳定前，聚合层是负债 |
| 只有 3 个大客户 | 不要做实时配额系统 | 月度对账 + 人工沟通。配额系统的复杂度只在客户数 > 100 时回本 |
| 内部平台，成本内部分摊 | 不要做发票和支付 | 只做计量 + 归因报表（chargeback / showback） |

**反模式（anti-pattern）速查：**

1. **在同步请求路径里查去重表。** 把计费系统的可用性绑进了产品可用性。
2. **"幂等键 = 不会重复计费"。** 真正的失效模式是客户端每次重试生成新 UUID。随机 UUID 可以作 operation key，但必须在首次发送前持久化，并由同一业务操作的所有重试复用；事件键再从它与语义步骤派生。
3. **"计量最终一致，差一点点无所谓"。** 误差方向是单向的，全是你的损失。必须有对账层。
4. **用 float 存金额。** 阶梯定价 + 大量小额累加 = 必然对不上。
5. **没有原始事件湖。** 客户第一次争议账单时，你除了道歉送额度没有别的选择。
6. **单层 token 预算。** 见 §8c。
7. **没有 `tokenizer_version` 和 `model_version`。** 换模型后所有历史重算全错，而且你不会立刻发现。
8. **成本口径和计费口径混在一张表里。** 改一次定价要动整条管道。
9. **把 tokenizer 自估当计费依据。** 用 provider 返回的 `usage`。
10. **不记 `feature` 维度。** 三个月后 CEO 问"哪个功能毛利为负"，你只能说"不知道"。

---

## 这一章的三句话

1. **计量系统的误差是单向的：丢事件永远是你少收钱，重复事件永远是客户投诉。** 所以"平均误差接近 0"这个目标毫无意义 —— 你要的是两个方向上各自有独立的收敛机制（对账追回少收的，幂等挡住多收的）。
2. **幂等键只保证“同一个键不重复入账”，管不住每次重试都生成新键。** 最好由调用方提供并复用稳定的业务 operation key；若平台首次接单时生成，则必须把它与业务对象持久绑定并在后续重试取回。单次 HTTP 入口临时 `uuid4()` 只能关联该请求内部的下游重试。
3. **不可变原始计量事件是计费侧的规范事实层，聚合只是可重算派生物。** 它还必须与业务记录、供应商 usage 和资金流水对账；没有这三方证据，争议时既无法证明，也无法安全重算。

---

## 面试官会追问

1. 计量事件丢了 0.5% 会怎样？重复了 0.5% 会怎样？两者哪个更可怕、为什么？
2. 客户端超时重试，你怎么保证不重复计费？幂等键从哪来？
3. 账期已经关闭了，一条 5 天前的用量事件到了，怎么办？给三种策略和各自的代价。
4. 你说你做到了 exactly-once 计费 —— 保证的作用域、去重窗口和外部支付未知结果分别怎么处理？
5. 一次 LLM 调用有几个计量项？缓存写和缓存读的价差是多少？Batch 能不能和 Fast 叠加？
6. 100 个网关实例共享一个租户的 TPM 配额，怎么实现？误差上界是多少？
7. 配额服务挂了，你放行还是拒绝？（→ 必须区分软限额和硬限额）
8. 流式响应生成到一半客户端断了，provider 收你钱了，你收客户钱吗？
9. 一个 Agent 陷入死循环，你的哪一层防线会先拦住它？
10. 怎么和上游 provider 的账单对账？误差目标是多少？对不上时你先查什么？

---

**完整设计题** → [`06-case-studies/04-usage-based-billing.md`](../06-case-studies/04-usage-based-billing.md)：本章的机制在 10 万租户 / 1.5 亿事件每天的规模下怎么落地、怎么估算、怎么答。

**按训练路径阅读** → 回 [START-HERE](../START-HERE.md) 按所选路径继续；页尾链接只表示本目录或专章的顺读顺序。

**目录顺读下一篇** → [03-identity-and-authz.md](03-identity-and-authz.md)
