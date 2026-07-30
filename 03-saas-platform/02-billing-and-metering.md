# 02 · 计量（metering）与计费（billing）

> 计量系统的误差是**有方向的**：丢事件永远是你少收钱，重复事件永远是客户投诉。
> 所以架构目标不是"准确"，是"**在两个方向上都有可审计的收敛机制**"。

---

## 1. 为什么 2026 年这件事变难了

传统 SaaS 按席位收费（per-seat pricing）：计量 = `SELECT count(*) FROM users`，一个月跑一次，错了也没人发现。AI 产品把三件事同时打破了：

| 变化 | 后果 |
|---|---|
| **推理成本进了 COGS** | AI 产品平均毛利（gross margin）约 **52%**，传统成熟 SaaS 是 **80–90%**（[ICONIQ 2026 State of AI](https://www.iconiq.com/growth/reports/2026-state-of-ai-bi-annual-snapshot)，n≈300）。毛利薄 ⇒ 计量误差直接吃掉利润 |
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
| 0 | 事件产生 | 进程崩溃、事件还在内存 | **少收钱** | **先落本地 WAL 再返回用户**；异步刷入总线 |
| 0 | 事件产生 | 客户端重试生成新 UUID | **多收钱** | 幂等键（idempotency key）必须由**业务语义派生**，见 §4 |
| 1 | 投递 | 总线不可用 | 少收钱 | 本地磁盘缓冲 + 有界队列（bounded queue）+ **反压（backpressure）到业务路径的开关必须存在但默认关** |
| 2 | 去重（deduplication） | 重复超出去重窗口 | 多收钱 | 去重窗口 = 你能容忍的客户端重放延迟上限 |
| 3 | 聚合 | 迟到事件（late-arriving events）晚于窗口关闭 | 少收钱 | 允许迟到期 + true-up，见 §5 |
| 4 | 富化（enrichment） | 用了**出账时**的价格而不是**事件时**的价格 | 双向 | 价格表按 `effective_from` 版本化，评级时按事件时间取版本 |
| 5 | 评级 | 阶梯（tiered pricing）/包量计算浮点误差累积 | 双向 | 全程整数（最小计费单位），**绝不用 float 存金额** |
| 6 | 出账 | 重复出账 | 多收钱 | `(tenant, period, invoice_version)` 唯一约束（unique constraint） |
| 7 | 对账 | 与上游 provider 账单对不上 | 少收钱 | 三方对账，见 §9 |

**两条结构性原则：** ① **计量必须与业务写路径解耦** —— 绝不在同步请求路径里做去重查询或调用计费 SaaS，那等于把计费系统的可用性绑进产品可用性；API 层只负责可靠投递到 log，去重与聚合放流处理层。② **原始事件湖是唯一真相源（source of truth），聚合结果是可重算的派生物** —— 任何口径变更、价格追溯、客户争议（dispute）都靠 replay 解决。**没有事件湖的计费系统无法处理争议**，最后只能靠客服送额度。

---

## 3. AI 用量事件的 Schema

这是 AI 计费和传统计量最大的差别：**一次调用产生的不是一个数，是一个向量**。

| 字段 | 类型 | 说明 / 为什么必须有 |
|---|---|---|
| `id` | uuid | 幂等键。与 `source` 组成去重主键（[CloudEvents](https://cloudevents.io/) 惯例） |
| `source` | string | 产生方，如 `gateway-cell3-pod7`。**去重是 `(source, id)` 而不是单 `id`** |
| `type` | enum | `llm.completion` / `tool.call` / `sandbox.session` / `search.query` / `storage.gb_hour` |
| `time` | ts(µs) | **事件时间（event time）**，用于窗口聚合（windowed aggregation）与价格版本选取 |
| `ingest_time` | ts | 到达时间。`ingest_time - time` = 迟到程度，必须监控其 p99 |
| `tenant_id` | string | 分区键（partition key） |
| `workspace_id` / `user_id` | string | 二级归因（attribution）维度 |
| `feature` | string | **业务功能**（`code_review` / `chat` / `batch_index`）。没有这个维度就无法回答"哪个功能在烧钱" |
| `session_id` / `run_id` / `agent_id` | string | Agent 场景的成本归因与**单会话硬上限**依据 |
| `provider` / `model` | string | `anthropic` / `claude-opus-5` |
| `model_version` | string | 快照版本。价格和 tokenizer 都跟版本走 |
| `tokenizer_version` | string | ⚠ **必须有**。Claude 4.7+ 对同样文本约多产生 **+30% token**（Opus 4.7/4.8/5、Sonnet 5、Fable 5；Sonnet 4.6 及更早为旧 tokenizer）。没记版本，历史重算一定错 |
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
| `upstream_cost_micros` | int64 | 你付给 provider 的成本（微分单位整数）。毛利分析的地基 |
| `schema_version` | int | 事件格式演进 |

> 单价数据来自各厂商定价页 2026-07-30 抓取（[Anthropic](https://docs.claude.com/en/docs/about-claude/pricing) / [OpenAI prompt caching](https://platform.openai.com/docs/guides/prompt-caching) / [Gemini caching](https://ai.google.dev/gemini-api/docs/caching)），**2026 年中量级，随时变动**。

**三个非显然的设计点：** ① `upstream_cost_micros` 必须和用量在同一条事件里 —— 分开算意味着你永远无法在秒级知道毛利，只能月末发现亏了。② 绝不用自己的 tokenizer 估算来计费，用 provider 返回的 `usage` 字段；自估在多模态输入、工具定义、思考 token 上会系统性偏差。③ **成本口径与计费口径要分离**：`upstream_cost_micros` 是成本，`billable_units` 是你卖给客户的单位，两者可以完全不同（你按 outcome 卖但成本按 token 算）。混在一起的系统改一次定价要动整条管道。

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

幂等只保证**同一个 key 不重复入账**，它**不保证客户端在重试时生成了同一个 key**。真实的失效模式是：客户端超时重试，每次 `uuid4()` 一个新 ID ⇒ 三次重试 = 三条事件 = 三倍收费，而且**你的幂等层完全正常工作**。

**正确做法：幂等键必须由业务语义派生，且在重试路径上稳定。**

```python
# ❌ 错
event_id = uuid4()

# ✅ 对：从不会因重试而变化的业务标识派生
event_id = uuidv5(NS, f"{tenant_id}:{request_id}:{attempt_semantic_step}")
#                              ↑ 由最外层入口生成一次，全链路透传（含重试）
```

**这条规则对上游 provider 也成立**：调用 LLM API 时透传你自己的幂等键，否则你重试一次就真的被收两次钱。

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

**水位线（watermark）策略**：`watermark = max(event_time) - allowed_lateness`。水位线越过窗口末端时**发出初步结果**，之后到达的迟到事件产生**增量修正（retraction + 新值）**，直到账期关闭。**账期关闭后的迟到事件只有三种处理方式，必须三选一并写进产品条款：**

| 策略 | 优点 | 代价 | 适用 |
|---|---|---|---|
| **丢弃** | 简单，账单不可变 | 少收钱，且金额不可控 | 事件量小、迟到率 < 0.01% |
| **计入下期（true-up）** | 不动历史账单，会计上干净 | 客户看到"上月的用量出现在这个月" | **推荐默认** |
| **重开账期（rebill）** | 最准确 | 需要作废+重开发票，税务/审计成本高 | 单笔金额超过阈值（如 $500）时触发 |

⚠ **公开资料里找不到主流计费厂商对"账期关闭后迟到事件如何处理"的官方 SLA。** 选型时必须实测，不要相信文档没写的东西。

**要监控的两个指标**：`late_event_ratio`（迟到事件占比，健康 < 0.1%；突增 = 某个数据面 cell 的投递链路堵了）、`revenue_adjustment_ratio`（true-up 金额 / 总收入，> 0.5% 说明 allowed lateness 设短了）。

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

**"恰好一次计费"（exactly-once billing）的真相**：它不存在于传输层，只存在于**应用层的幂等收敛** —— 工程等价物是 `至少一次投递 + 幂等应用 + 可对账的最终收敛`。面试时说"我们做到了 exactly-once 计费"会被追问到崩溃。**支付环节**必须带幂等键（就是 `invoices.idem_key`），且**先落"支付意图"再调用**，绝不先调用后记账 —— 否则网络超时时你不知道钱扣没扣。

---

## 7. 配额与限额

### 四种额度，四种算法（别混用）

| 类型 | 语义 | 算法 | 典型 |
|---|---|---|---|
| **速率**（rate） | 单位时间的量 | 令牌桶（token bucket）/ 滑窗（sliding window） | RPM、TPM |
| **累计量**（counter） | 周期内总量 | 分布式计数 + 租约 | 月 token 额度、预算 $ |
| **并发**（gauge） | 同时在跑的数量 | **信号量（semaphore）+ 租约（lease）+ TTL 心跳（heartbeat）** | 并发沙箱数、并发 Agent 数 |
| **绝对上限**（cap） | 单个实体的硬顶 | 本地即可判定 | 单会话最大 token、单请求最大输出 |

⚠ **并发配额不能用令牌桶。** 并发是 gauge 不是 counter：进程崩溃会让计数永久泄漏。必须用带 TTL 的租约 + 心跳续期，崩溃后租约自动过期归还。这是配额系统里最常见的实现错误。

### 分布式近似计数（approximate distributed counting）：本地令牌桶 + 周期同步（伪代码）

核心思想：**每个实例向中心"租"一段额度，本地扣减，周期归还与续租。** 用有界的过量发放（over-issuance）换掉每请求一次的远程调用。

```python
class LeasedQuota:
    """全局配额 Q，M 个实例。过量发放上界 = M × lease_size —— 这个数你必须能算出来。"""

    def __init__(self, tenant, meter, mode):
        self.local_balance = 0          # 本地持有的额度
        self.lease_size    = MIN_LEASE  # 自适应，见 refresh()
        self.burn_ewma     = 0.0        # 燃烧率（单位/秒）的指数滑动平均
        self.next_refresh  = 0
        self.mode          = mode       # SOFT | HARD
        self.stale_since   = None

    def try_consume(self, n) -> Decision:
        if self.local_balance >= n:
            self.local_balance -= n
            return ALLOW

        # 本地额度不足 —— 只有这时才走远程，热路径上是纯本地操作
        ok = self.refresh(need=n)
        if ok and self.local_balance >= n:
            self.local_balance -= n
            return ALLOW

        if not ok:                       # 配额服务不可达
            # 这一行是整个系统最重要的策略决策，不是实现细节：
            #   SOFT（软限额/降级用）→ fail-open：可用性优先，事后靠对账追回
            #   HARD（防欺诈/防烧钱）→ fail-closed：正确性优先，宁可拒绝
            return ALLOW if self.mode == SOFT else DENY_UNAVAILABLE

        return DENY_QUOTA_EXCEEDED

    def refresh(self, need) -> bool:
        # 自适应租约：够用一个刷新周期，且不至于把全局额度囤死在冷实例里
        target = max(need, self.burn_ewma * REFRESH_INTERVAL * 1.2)
        self.lease_size = clamp(target, MIN_LEASE, min(MAX_LEASE, GLOBAL_Q * 0.05))
        try:
            granted = quota_service.acquire(          # 中心侧：Redis/Postgres 上的原子扣减
                tenant=self.tenant, meter=self.meter,
                want=self.lease_size, hold_ttl=LEASE_TTL)   # TTL 让崩溃实例的额度自动归还
        except Unavailable:
            self.stale_since = self.stale_since or now()
            return False
        self.local_balance += granted
        self.next_refresh = now() + REFRESH_INTERVAL
        return granted > 0

    def on_shutdown(self):
        quota_service.release(self.tenant, self.meter, self.local_balance)  # 优雅归还
```

**参数怎么定（可直接抄）：**

```
REFRESH_INTERVAL   1–10 s        越短越准，越长越省
LEASE_TTL          3 × REFRESH_INTERVAL
MIN_LEASE          够 1 个典型请求用（LLM 场景 ≈ 单请求 p95 token 数）
MAX_LEASE          GLOBAL_Q × 5%   ← 防止一个实例囤走大部分额度
误差上界           M × lease_size / Q，目标 < 2%
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
    → 告警租户 + 自动降级：Opus → Sonnet/Haiku、关闭多 Agent 扇出、
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
2. **"幂等键 = 不会重复计费"。** 真正的失效模式是客户端每次重试生成新 UUID。键必须由业务语义派生。
3. **"计量最终一致，差一点点无所谓"。** 误差方向是单向的，全是你的损失。必须有对账层。
4. **用 float 存金额。** 阶梯定价 + 大量小额累加 = 必然对不上。
5. **没有原始事件湖。** 客户第一次争议账单时，你除了道歉送额度没有别的选择。
6. **单层 token 预算。** 见 §8c。
7. **没有 `tokenizer_version` 和 `model_version`。** 换模型后所有历史重算全错，而且你不会立刻发现。
8. **成本口径和计费口径混在一张表里。** 改一次定价要动整条管道。
9. **把 tokenizer 自估当计费依据。** 用 provider 返回的 `usage`。
10. **不记 `feature` 维度。** 三个月后 CEO 问"哪个功能毛利为负"，你只能说"不知道"。

---

## 面试官会追问

1. 计量事件丢了 0.5% 会怎样？重复了 0.5% 会怎样？两者哪个更可怕、为什么？
2. 客户端超时重试，你怎么保证不重复计费？幂等键从哪来？
3. 账期已经关闭了，一条 5 天前的用量事件到了，怎么办？给三种策略和各自的代价。
4. 你说你做到了 exactly-once 计费 —— 具体是怎么做的？（→ 正确答案是承认它不存在于传输层）
5. 一次 LLM 调用有几个计量项？缓存写和缓存读的价差是多少？Batch 能不能和 Fast 叠加？
6. 100 个网关实例共享一个租户的 TPM 配额，怎么实现？误差上界是多少？
7. 配额服务挂了，你放行还是拒绝？（→ 必须区分软限额和硬限额）
8. 流式响应生成到一半客户端断了，provider 收你钱了，你收客户钱吗？
9. 一个 Agent 陷入死循环，你的哪一层防线会先拦住它？
10. 怎么和上游 provider 的账单对账？误差目标是多少？对不上时你先查什么？

---

**完整设计题** → [`06-case-studies/04-usage-based-billing.md`](../06-case-studies/04-usage-based-billing.md)：本章的机制在 10 万租户 / 1.5 亿事件每天的规模下怎么落地、怎么估算、怎么答。

**下一篇** → [03-identity-and-authz.md](03-identity-and-authz.md)
