# 02 · 案例：设计一个企业级 LLM 网关

> LLM 网关看起来像 API 网关，其实不是：后端不一定是同构副本（换模型就换了输出分布），
> 缓存位置也不一定由你控制（自建推理的 KV cache 在具体实例；托管 API 的缓存作用域由供应商定义），
> 所以你需要**感知缓存状态的路由**，但不等于把持久状态塞进网关进程；
> 计费单位不是请求而是 token（所以限流必须**预扣（reservation）+ 结算（settlement）**），
> 而且响应是一条可能持续 10 分钟、中途才会失败的流。

---

## 读这道题之前

> **阅读层级与时间口径**：主干是需求 → 路由 → 配额 → 流式 → 计量，full-stack 开发者先读这些即可；具体供应商型号、价格、缓存阈值和合规能力是截至 **2026-08** 的示例快照，上线前必须重新查合同与官方文档。

🔶 **这道题属于 AI 岗方向**：通用 full-stack / 后端面试路径（[README 路径 A / B](../README.md#学习路径)）可以整题跳过 —— 判断依据和 [`04-ai-agent-systems/`](../04-ai-agent-systems/) 一致：JD 里出现「LLM / Agent / 推理 / RAG / GPU」中任意一个词，它才从"可跳过"变成"面试官的主场"（路径 C）。

**如果你是直接翻到这道题的**：这题的每一个难点都是"API 网关的某条基本假设在 LLM 上反掉了"。第 3 题答不出，你会把限流设计成按请求数计数 —— 而正文从头到尾在论证那条路为什么必然超配额。

**先确认你能回答这三个问题**

1. 有状态（stateful）和无状态（stateless）的判据是什么？一个"只做转发"的网关，为什么一旦要吃到 provider 的前缀缓存，就不能再随便挑一个后端了？
   答不出 → 先读 [00-concepts §9 有状态 vs 无状态](../00-foundations/00-concepts.md#9-有状态-vs-无状态)、[04-networking-and-edge §2 连接管理](../01-building-blocks/04-networking-and-edge.md#2-连接管理)
2. 前缀缓存命中的前提是"前缀逐字节相同"。在 system prompt 里放一个 `now()`、或一段未排序的 `json.dumps`，命中率和账单会怎么变？
   答不出 → 先读 [02-caching §7 LLM 时代的新缓存层](../01-building-blocks/02-caching.md#7-专项选读llm-缓存层)、[08-cost-and-latency §4 Prompt caching](../04-ai-agent-systems/08-cost-and-latency.md#4-prompt-caching三家机制差异与工程约束)
3. 一次调用要用掉多少 token，请求发出前你知道吗？不知道的话令牌桶怎么扣？20 个 pod 各扣各的本地桶，全局配额会错成什么样？
   答不出 → 先读 [02-billing-and-metering §7 配额与限额](../03-saas-platform/02-billing-and-metering.md#7-配额与限额)、[03-resilience-patterns §6 负载卸载与准入控制](../05-reliability/03-resilience-patterns.md#6-负载卸载load-shedding与准入控制)

**这道题会用到的构件**

| 构件 | 用在哪 | 详见 |
|---|---|---|
| 有状态 vs 无状态、长连接、SSE vs WebSocket | §7 缓存亲和路由把网关变成有状态的、§8 流式代理 | [00-concepts §9](../00-foundations/00-concepts.md#9-有状态-vs-无状态)、[`04-networking-and-edge.md`](../01-building-blocks/04-networking-and-edge.md) §2、§4 |
| 超时预算、重试放大、熔断、负载卸载 | §5 路由与故障转移、§12 失败模式清单 | [`03-resilience-patterns.md`](../05-reliability/03-resilience-patterns.md) §2、§3、§4、§6 |
| 前缀缓存 / 精确缓存 / 语义缓存 | §7 三种缓存各自的命中前提与泄露面 | [`02-caching.md`](../01-building-blocks/02-caching.md) §7、[`08-cost-and-latency.md`](../04-ai-agent-systems/08-cost-and-latency.md) §4 |
| 计量管道：事件 schema、去重幂等、对账 | §9 四类 token 必须分开计、三方对账 | [`02-billing-and-metering.md`](../03-saas-platform/02-billing-and-metering.md) §3、§4、§9 |
| 数据驻留、审计留痕、PII 边界 | §9 PII 与留痕、§10 安全 | [`04-isolation-and-compliance.md`](../03-saas-platform/04-isolation-and-compliance.md) §2、§6 |

**这道题的一句话本质**

> **LLM 网关看起来像 API 网关，其实四条基本假设全反了：后端不是同构副本、缓存不在你手里、计费单位是 token 不是请求、响应是一条可能十分钟后才失败的流。**
> 整篇正文都可以读成对这四条的逐个补救。每读到一个组件就问一次："它在补哪一条反掉的假设？"补不上的那一条，就是面试官会追问你的地方。

---

## 1. 需求澄清（前 8 分钟，不要动笔画图）

面试官说"设计一个 LLM 网关"时，至少有五种完全不同的东西叫这个名字。先把它钉死。

| 问题 | 为什么决定架构 |
|---|---|
| **谁是使用方？** 内部团队 / 产品服务 / 外部客户 | 内部 = 信任边界宽、可要求改客户端；外部 = 必须做租户隔离（tenant isolation）与滥用防护（abuse prevention） |
| **多少团队、多少 key？** | 10 个团队用配置文件就够；2,000 个 key 需要真正的控制面 |
| **托管 API / 自建推理 / 都有？** | 只接托管 API → 数据面进程通常仍可无状态；接自建 vLLM 池 → 路由器必须读取 KV cache/队列状态，但持久状态仍应放在外部系统 |
| **强制走网关吗？有 break-glass 吗？** | 强制 = 网关是 SPOF，可用性目标要比业务高一档 |
| **合规边界？** 数据驻留（data residency）/ ZDR / 审计留痕（audit trail） | 决定是否按租户绑定 provider 区域端点，以及日志能不能存 prompt 原文 |
| **SLO 与成本归属？** | 网关自身预算应 ≤ 上游延迟的 5%；决定预算是"告警"还是"拒绝"，以及**失败请求是否计费** |

**本文锁定的假设**（面试里显式写在白板角上）：

```
2,000 名工程师、日活 1,200、30 个产品团队 + 若干 agent 服务 → 240,000 请求/日
形态：70% 流式对话/agent（多轮长前缀），30% 同步短请求（分类/抽取/embedding）
Provider：Anthropic + OpenAI + Gemini 托管；自建 vLLM 池 v2 才接
合规：EU 子公司流量必须区域内处理；默认不落 prompt 原文，按租户白名单开
内部目标：网关自身附加延迟的 p99 候选阈值为 25 ms；可燃尽 SLO 写成“至少 99.9% 的有效网关调用附加延迟 ≤ 25 ms”，可用性目标 99.95%；强制走网关但保留 break-glass
```

---

## 2. 估算：先算清楚这是个多大的东西

```
QPS      240,000 请求/日 ÷ 86,400 ≈ 2.8 RPS 平均；8h 工作时段 + 3× 尖峰 ⇒ 峰值 25–30 RPS
         agent 扇出（一个 workflow 30–80 次调用）⇒ 瞬时 200+ RPS
并发流   单请求 ≈ TTFT 1 s + 800 tok @ 60 tok/s ≈ 14 s
         峰值 = 30 × 14 ≈ 420 条 SSE；加 agent 突发 ⇒ 按 1,500 条设计
```
**这不是一个高 QPS 系统。** 网关的难点从来不是 QPS，是**并发的长连接（long-lived connection）**和**每请求的字节数**。单个 Go/Rust 进程扛 2,000+ 条 mostly-idle 流很轻松 ⇒ **3 个 pod 够跑，6 个用于 HA 与滚动发布**。CPU 不花在转发上，花在**逐 token 做内容过滤/计数**上 —— 那是容量模型里唯一需要压测的部分。

**Token 与成本**（教学算例，不是报价；下列型号、单价和折扣会随合同与日期变化）
```
输入 12,000 tok/请求（含稳定前缀）、输出 800 tok/请求
⇒ 输入 2,880 M tok/日   输出 192 M tok/日
全 Opus 5 零缓存：2,880×$5 + 192×$25 = $19,200/日 ≈ $576k/月
                 （= $16/活跃工程师/日，与 Anthropic 公布的 $13/活跃日 同量级）
开 80% 缓存读：输入均价 0.2×$5 + 0.8×$0.5 = $1.40/M ⇒ $8,832/日 ≈ $265k/月  ← 降 54%
再把 50% 流量路由到 Sonnet 5 / Haiku 4.5 档 ⇒ ~$120–150k/月
```
> **这两个数字就是网关的商业理由**：不是"统一 SDK"，是**缓存命中率（cache hit ratio）+ 路由**这两个旋钮，一年 500 万美元。任何让缓存前缀不稳定的设计决策，都要按这个价签评审。

**留痕**：全文留痕（full-payload logging）240k × ~50 KB ≈ **12 GB/日 → 4.4 TB/年**；仅元数据 ≈ 480 MB/日。⇒ **全文默认关，按租户 + 采样开**。这不只是成本，是最大的 PII 面。

---

## 3. 高层架构

```
                       ┌─────────────────────── 控制面 (Control Plane) ────────────────────────┐
                       │  虚拟 Key 服务 │ 策略/路由配置 │ 价目表 │ 预算账本 │ 模型注册表(能力位) │
                       └───────┬───────────────────────────────────────────────┬───────────────┘
                               │  推送 + 本地缓存（TTL 30s，控制面挂掉照常服务）  │
                               ▼                                               ▼
┌────────────┐   ┌──────────────────────── 数据面 Gateway Pod ─────────────────────────┐
│ 应用 / SDK  │   │ ① 入口 TLS/H2/SSE                                                   │
│ agent 服务  ├──▶│ ② 认证：虚拟 Key → (tenant, team, actor, on_behalf_of, scopes)       │
│ OpenAI 兼容 │   │ ③ 策略闸：模型 allowlist / 数据驻留 / 内容策略 / 全文留痕开关          │
└────────────┘   │ ④ 准入：多维限流 + token 预扣（Redis Lua，见 §6）                     │
                 │ ⑤ 归一化：→ 内部 IR（统一请求对象）                                   │
                 │ ⑥ 缓存层：精确缓存查（语义缓存默认关，见 §7）                          │
                 │ ⑦ 路由器：别名 → 模型池 → 实例（成本/能力/健康/**前缀亲和**）           │
                 │ ⑧ 出站：provider adapter + 密钥注入 + cache_control 生成               │
                 │ ⑨ 流式代理：事件归一 / 空闲超时 / 取消传播 / 中途错误                   │
                 │ ⑩ 结算：usage → 计量事件（幂等键）+ 预扣回补                           │
                 │ ⑪ 留痕：审计日志（脱敏后）+ OTel span                                  │
                 └───┬──────────────┬──────────────┬──────────────┬─────────────────────┘
                     ▼              ▼              ▼              ▼
              Anthropic API   OpenAI API     Gemini API     自建 vLLM 池（v2）
              (第一方/Bedrock/  (含 Azure)                    ← EPP/KV 感知路由
               Vertex 三个接入点)                                （GAIE）

  旁路（异步，不在关键路径）：计量事件 → Kafka → 去重/聚合 → 计费 & FinOps
                            审计日志 → 对象存储（WORM）；影子流量 → 评测管线
```

**三条不可违背的分层原则：**
1. **控制面不进每次请求的同步关键路径（critical path）。** 策略/价目/预算以推送 + 本地缓存驻留数据面（data plane）；控制面故障时使用带版本、签名和有效期的最后已知良好快照。若从未拿到有效快照，认证、租户隔离、数据驻留和硬预算等边界应拒绝请求；只有明确批准的低风险策略才可退化。
2. **计量是旁路。** 写 Kafka 失败不能让用户请求失败；但**必须先落本地 WAL 再回响应**——丢事件的方向永远对供应商不利。
3. **网关不做 agent 循环。** 网关是"每请求一次决策"的策略执行点（policy enforcement point）。谁把 agent loop 写进网关，谁就会在半年后为了加一个 checkpoint 重写整个数据面。

---

## 4. 深挖一：Provider 抽象层——抽到什么粒度

**推荐边界：归一化（normalization）到"请求语义与计量口径（metering semantics）"，不要假装能统一"生成行为"。** 下表字段名以各家当前文档为准，说明的是**差异形态**而非稳定 API 契约。

| 维度 | Anthropic Messages | OpenAI | Gemini | 抽象策略 |
|---|---|---|---|---|
| system prompt | 顶层 `system`，可为 block 数组 | `instructions` / `role:"system"` 消息 | 顶层 `systemInstruction` | **归一**（无损） |
| 工具定义 | `tools[].input_schema` | `tools[].function.parameters` | `functionDeclarations[].parameters` | **归一到 JSON Schema 交集**；超出交集的关键字（`oneOf`/条件）按 provider 能力位（capability flag）降级或拒绝 |
| 工具调用与回填 | `tool_use` / `tool_result` block；天然并行 | `tool_calls[]` + `role:"tool"`；并行有开关 | `functionCall` / `functionResponse` part | **归一** + 能力位 `supports_parallel_tools` |
| 思考 / reasoning | `thinking` block + token 预算 | reasoning effort + 加密 reasoning item | thinking 配置 | **不归一。** 语义、可见性、计费口径三者都不同，强行统一必然错计费 |
| 缓存控制 | **显式** `cache_control` breakpoint（≤4 个，回看 20 个 block） | **自动** + `prompt_cache_key` / `prompt_cache_options.mode` | 隐式（阈值 2048/4096）+ 显式 CachedContent（**有 $1.00/M tok/小时 存储费**） | **不归一。** 由网关按 provider 自动生成，用户只声明"哪一段是稳定前缀" |
| 停止原因 | `stop_reason` | `finish_reason` | `finishReason` | **归一为内部枚举 + 原值透传（pass-through）**（`raw_finish_reason`） |
| 用量字段 | `input_tokens` / `cache_creation_input_tokens` / `cache_read_input_tokens` / `output_tokens` | `usage.*` + `prompt_tokens_details.cached_tokens` | `usageMetadata.*` | **归一为统一计量模型**（见 §9），四类 token 必须分开 |
| 流式事件 | `message_start` / `content_block_delta` / `message_delta` / `message_stop` | chunk 的 `delta` / typed events | 增量 `parts` | **归一为内部事件流**（见 §8） |
| 错误 | `error.type`: `overloaded_error` / `rate_limit_error` … | `error.code`: `rate_limit_exceeded` / `context_length_exceeded` … | HTTP status + `status` | **归一为错误分类**（见下表） |
| 长上下文计价 | 1M 上下文按标准价（4.6+，无溢价） | — | 3.1 Pro >200k：输入 ×2、输出 ×1.5 | **价目表按 provider × 上下文档位**，不能一个单价打天下 |
| 分词器（tokenizer） | Claude 4.7+ 新 tokenizer，同文本 **约 +30% token** | 不同 | 不同 | **token 数跨 provider 不可换算。** 预算与限流必须按 provider 分别标定 |

### 四层抽象（照抄这个分层）

```
L0 IR（内部表示）  : messages / tools / stop / max_output / stream / stable_prefix_hint
L1 Adapter         : IR ↔ provider wire format（双向，含流式事件归一）
L2 Capability Gate : 模型注册表声明能力位；不支持就 **拒绝**，不模拟
L3 Escape Hatch    : provider_options.{anthropic|openai|gemini} 透传 + /v1/raw/{provider} 直通
```
**L2 的纪律是本节核心：不要模拟不存在的能力。** 用 prompt hack 给不支持 tool calling 的模型"模拟"工具调用，会在三处炸：解析脆弱、缓存前缀被偷偷改写、评测不可复现。能力位为假 → 返回 `422 capability_not_supported`，把选择权还给调用方。

**L3 的存在理由**：任何抽象层都会落后 provider 新特性 6–12 周。没有逃生舱（escape hatch），团队就绕过网关直连——那才是真正的失控。逃生舱要有代价：直通端点**不享受缓存优化，但仍然计量、限流、留痕**。

### 错误归一（error normalization，可用性的地基）

| 内部类 | 触发 | 可重试 | 可换实例 | 可换模型 |
|---|---|---|---|---|
| `RATE_LIMITED` | 429 + `retry-after` | ✅ 按 `retry-after` | ✅ 换接入点 | ⚠ 仅显式允许时 |
| `OVERLOADED` | 529 / 503 | ✅ 指数退避（exponential backoff）+ 抖动（jitter） | ✅ | ⚠ |
| `UPSTREAM_TIMEOUT` | 无 token 超过空闲阈值 | ⚠ **仅当请求幂等且未产出 token** | ✅ | ⚠ |
| `CONTEXT_TOO_LONG` | 上下文超限 | ❌ | ❌ | ⚠ 可选更大窗口模型，也可由调用方截断、摘要、检索或缩小输出预算；网关不要擅自改 prompt |
| `CONTENT_FILTERED` | 安全策略拒绝 | ❌ | ❌ | ❌ **绝不换 provider 重试**——那是在做"合规套利（compliance arbitrage）"，审计时会要命 |
| `INVALID_REQUEST` | 4xx 参数错 | ❌ | ❌ | ❌ |
| `AUTH_FAILED` | 密钥问题 | ❌ | ⚠ 只可切换同一合同账户内已知健康的轮换凭证；撤销/权限失败应停止并告警 | ❌ |

---

## 5. 深挖二：路由与故障转移（failover）

```yaml
routes:                                  # 客户只认别名，网关拥有实现
  - alias: chat-default
    strategy: cost_aware_cascade         # 先小后大
    targets: [ {model: claude-sonnet-5, weight: 90, role: primary},
               {model: claude-opus-5,   weight: 10, role: escalate} ]
    endpoints: [anthropic_first_party, bedrock_us_east_1, vertex_us_central1]  # 候选接入点；需逐一通过兼容性评测
    allow_model_downgrade: false         # 默认关！见下文
    retry_budget: { max_attempts: 3, total_ms: 20000, retry_ratio: 0.1 }
    idle_timeout_ms: 30000
    residency: any
  - alias: chat-eu
    residency: eu                        # 只允许"区域内处理"的接入点
    endpoints: [anthropic_eu, azure_openai_swedencentral]
```

| 维度 | 做法 | 代价 |
|---|---|---|
| **按别名** | 客户只写 `chat-default`，网关决定实际模型 | 需要在响应里回 `x-llm-served-model`，否则调试地狱 |
| **按能力** | 需要 `tools` + `json_schema` + 128k 上下文 → 过滤模型池 | 能力位要维护，模型发版时会漂 |
| **按成本（级联 cascade）** | 小模型先跑，置信度低再升级（[Cluster, Route, Escalate](https://arxiv.org/abs/2606.27457)：保留最强模型 97–99% 准确率） | **抬高尾延迟（tail latency）**：p99 = 小模型 + 大模型串行。实测降本区间 **40–85%**，但 85% 是偏易基准的上限，生产混合流量按 40–70% 规划 |
| **按可用性** | 健康度打分 + 熔断 + 接入点轮换 | 见下 |

### 故障转移：把两类转移严格分开

> **面试金句：**
> "LLM 网关的 failover 和 HTTP 网关的 failover 不是一回事。HTTP 后端是同构副本（homogeneous replica），转移是无损的；模型不是同构副本，转移一定改变输出分布。所以我把它分成两类：
> **(a) 同一模型工件的多接入点**——第一方 API、Bedrock、Vertex 即使名字相同，也可能在版本、参数、过滤器、tokenizer 和发布节奏上不同。只有注册表确认工件版本并通过同一套契约/质量评测后，才把某组接入点视为可自动转移；
> **(b) 跨模型降级（cross-model downgrade）**——Opus → Sonnet，输出会变，必须显式：响应头回 `x-llm-served-model` 和 `x-llm-degraded: true`，计量按实际模型计价，且只对声明了 `allow_model_downgrade: true` 的路由生效。
> 默认关闭跨模型自动降级。因为一个下游做严格 JSON schema 解析的批处理任务，会在你'成功兜底'的那一刻开始**静默产出错误数据**——那比返回 503 糟糕得多。"

**跨模型降级前必须回答的三个问题**（写进 ADR）：①调用方能否检测到降级（→ 响应头 + 计量事件记 `served_model`）；②下游是否严格解析输出（→ 是则禁止降级，或要求降级目标通过同一套 schema 回归集）；③降级目标的 tokenizer 是否不同（→ Claude 4.7+ 同文本多约 30% token，会撞 `max_tokens`，"同样任务突然被截断"就是这么来的）。

**重试的三条硬纪律：** ①给重试单独设预算（如额外流量不超过 10%，具体值由容量与 SLO 验证），上游过载时无限重试会继续放大故障；②**流已产出 token 后不做透明重试**（重复内容 + 两次生成都计费）；③遵守 `Retry-After`，并基于错误作用域选择健康接入点。只有确认是单凭证配额问题时才轮换同账户凭证，不能靠换 key 绕过组织级限额或撤销。

---

## 6. 深挖三：限流（rate limiting）与配额——为什么必须按 token 而不是请求

**按请求数限流对 LLM 是失效的**：一个 200k 输入的请求和一个 200 token 的请求，对上游的压力和对账单的影响都差 1000 倍。上游 provider 自己就是按 **TPM（tokens per minute）+ RPM** 双维度限的；网关只限 RPM，唯一效果是把上游的 429 原样转给用户。

```
配额层级 = tenant → team → virtual_key/actor，并另按 model_family、priority_class 分类
每个需要约束的层级可设 RPM / TPM_input / TPM_output；不要为所有维度做笛卡尔积 key
外加两条绝对上限：单请求 max_input_tokens、单会话/单任务 total_tokens
```
**最后那条"单任务绝对上限"是 agent 时代新增的**。人类会话的 token 方差可预测，agent 循环的方差大一个数量级——一个跑飞的 workflow 能在 20 分钟内吃掉团队当天全部 TPM。三层同时设：并发上限 / 单任务 token 上限 / 组织级 spend limit。

### 预扣与结算（本节考点）

流式请求在**开始时**你不知道会产出多少 token。所以：

```python
# ── 准入（预扣）──────────────────────────────────────────────
est_input  = count_tokens(req, tokenizer_of(model))
est_output = min(req.max_tokens, route.max_output_cap)
reserve    = WeightedUsage(est_input, est_output)
ok, reservation_id = ratelimit.acquire_hierarchy(keys, reserve,
                                                   idempotency_key=req.id)
if not ok: return 429 with Retry-After
# ── 结算（回补）──────────────────────────────────────────────
observed = Usage(input=est_input, output=0)              # 先初始化，异常前也有值
outcome = "unknown"
try:
    outcome = proxy_and_stream(req, on_usage=observed.merge_monotonic)
finally:
    ratelimit.settle_once(reservation_id, observed, outcome)
```

**四个必踩的坑：** ①**不给 `max_tokens` 设 cap** —— 客户端填成模型上限，预扣会抽干桶；②先进入 `try` 才初始化 `actual` —— 代理在赋值前抛错时，`finally` 自己也会报错；③只靠进程内 `finally` —— pod 崩溃时它不会执行，所以 reservation 必须是持久状态，并由对账任务收口；④失败请求是否计入配额没有行业统一答案。策略必须写进文档并对租户可见。

reservation 过期也**不能无条件全额退款**：进程失联时，上游可能已经生成并计费。恢复器应把它标成 `unknown`，先保守占用已预扣上限或已观测用量，再用 provider usage/账单和本地流式日志对账；只有确认未消费的部分才归还。这样既不会永久泄漏，也不会把未知消费变成无限超发窗口。

### 分布式限流算法选型

| 算法 | 精度 | 热路径 Redis 往返 | 突发 | 适用 |
|---|---|---|---|---|
| 滑动窗口日志（sliding window log，ZSET） | 精确 | 1 + O(n) 内存 | 好 | 低 QPS 高价值维度（如"每租户每分钟 10 次昂贵模型"） |
| 滑动窗口加权计数 | 近似（误差 <1%） | 1 | 好 | RPM 维度的通用默认 |
| **令牌桶（Lua）** | 精确，可配 burst | 1 | 可配 | **TPM 维度首选**，因为天然支持"取 N 个令牌" |
| GCRA / 漏桶（leaky bucket） | 精确、无突发 | 1 | 整形 | **对上游 provider 出站整形（egress traffic shaping）**（让自己永远不撞 429） |
| **本地租约（local lease）+ 中心结算** | 近似（超发 ≤ N×slack） | 每租约 1 次 | 好 | 高 QPS 时把 Redis RTT 从每请求 1 次降到每秒 1 次 |

```lua
-- token_bucket.lua：单个 bucket 原子地取 N 个令牌；只接受 need > 0
-- KEYS[1] = "rl:{tenant:acme}:m:claude-sonnet-5:tpm_in"  ← hash tag 保证同租户落同 slot
-- ARGV = capacity, refill_per_ms, requested, ttl_ms
local st  = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local t = redis.call('TIME')                              -- 不信客户端时钟
local now = tonumber(t[1]) * 1000 + math.floor(tonumber(t[2]) / 1000)
local cap, rate, need = tonumber(ARGV[1]), tonumber(ARGV[2]), tonumber(ARGV[3])
if not cap or not rate or not need or cap <= 0 or rate <= 0 or need <= 0 then
  return redis.error_reply('invalid token bucket arguments')
end
local tok = math.min(cap, (tonumber(st[1]) or cap) + (now - (tonumber(st[2]) or now)) * rate)
if tok < need then
  return {0, math.ceil((need - tok) / rate)}   -- 回传还需等多久 → 生成 Retry-After，比盲目退避强
end
redis.call('HMSET', KEYS[1], 'tokens', tok - need, 'ts', now)
redis.call('PEXPIRE', KEYS[1], ARGV[4])
return {1, 0}
```

不要把结算实现成“同一脚本传负数”。重放一次负数请求就能反复加额度。结算必须按 `reservation_id` **幂等**：存储 `reserved`、单调递增的 `observed` 与终态，只应用尚未结算的差额，并把退款封顶在已预扣量。多个层级若要一次原子扣减，Redis Cluster 下所有 key 必须在同一 hash slot；做不到时就用中心配额服务/分层租约，并明确最大超发边界，不能假装多次单 key 调用是原子的。

**撞墙条件**（数字须由自己的压测校准）：限流键基数上涨、头部租户形成热 key，Redis p99 和超时率持续恶化。届时可切**本地租约模式**：每个 pod 从中心预领一小片额度，本地令牌桶消费并周期性报告单调累计值。到期只归还中心确认未消费的余额；失联租约先封存到对账完成。代价是最多超发“同时有效租约数 × 每租约 slack”，用可量化误差换掉大部分中心往返。

---

## 7. 深挖四：缓存——为什么路由必须感知状态

### 前缀缓存的路由亲和性（cache affinity，最重要的一节）

前缀缓存（KV cache reuse）把输入价打到 **10%**，把 TTFT 降一个数量级。架构后果是：**缓存住在某个具体的地方，请求必须被送到"见过这个前缀的地方"。**

- **自建推理**：KV 在某个 pod 的显存/本地 CPU 层。[llm-d 的对照实验](https://llm-d.ai/blog/kvcache-wins-you-can-see)（8 pod / 16×H100 / Qwen-32B / 150 租户 × 6k 上下文）的极端差距：P90 TTFT **0.542 s（精确缓存感知路由） vs 92.551 s（随机路由）**，约 **170×**。这不是优化，是"能不能用"。
- **托管 API**：缓存的隔离与路由范围由供应商、接入方式和合同决定；你通常看不到内部缓存节点。网关能做的是在文档保证的作用域内固定 provider endpoint、账户/workspace 与缓存相关参数。跨接入点可能丢失缓存收益，但不要凭猜测把“同会话固定到内部实例”写成保证。

```
hash_key = (tenant_id, session_id 或 stable_prefix_hash) → 一致性哈希 → 目标 pod / 接入点
溢出规则：目标负载 > 阈值时才放弃亲和（宁可牺牲少量缓存，不要排队）
```

> **面试金句：**
> "网关进程仍可以无状态，但**路由决策必须感知外部状态**。自建推理时，可从'最少连接'升级为'按前缀/会话的一致性哈希 + KV 命中分数 + 负载上限溢出'；托管 API 只能利用供应商公开保证的缓存作用域。我用 TTFT 和单位请求成本验证收益，并在目标队列过深时放弃亲和——缓存命中是优化，排队超出 SLO 就应溢出。"

### 让缓存真的命中：前缀的字节稳定性（byte stability）

网关必须强制这条布局，因为应用团队一定会写坏它：

```
[ 工具定义 ][ 系统提示 ]  ← 逐字节稳定，网关在这里打 cache breakpoint
[ 只追加的历史 ]          ← append-only，不能在中间插入或改写
[ 本轮变化的输入 ]        ← 检索结果、时间戳、用户输入都放这里
```
三家参数（2026 年中量级，随时变动，以官方文档为准）：

| | Anthropic | OpenAI | Gemini |
|---|---|---|---|
| 触发 | 显式 `cache_control` breakpoint | 自动，`prompt_cache_key` 提升路由命中 | 隐式自动 + 显式 CachedContent |
| 最小前缀 | **非单调**：Opus 5 = 512；Sonnet 5 / Opus 4.8 = 1024；Opus 4.7 = 2048；**Haiku 4.5 = 4096** | 1024 | 4096（3.5 Flash / 3.1 Pro）、2048（2.5 系） |
| 写/读价 | 写 1.25×（5m）/ 2×（1h），读 **0.1×** | 写 1.25×（GPT-5.6+），读**按代际**：GPT-5.x 10%、gpt-4.1/o3 25%、gpt-4o 50% | 隐式无存储费；显式**另收 $1.00/M tok/小时** |
| 失效 | 级联 `tools → system → messages`；改 tool 定义 = 全失效；每 breakpoint 只回看 **20 个 content block** | 前缀变即失效 | 前缀变即失效 |

**四个必须做成监控指标而不是排查手段的失效模式：**
1. **`cache_read_input_tokens` 恒为 0** —— 静默 miss（silent cache miss）没有异常、没有日志，**只有账单**。最贵的失效模式。
2. **system prompt 里出现 `now()` / UUID / 未排序的 `json.dumps`** —— 网关在 IR 层做**前缀指纹（prefix fingerprint）**，每分钟变化率超阈值就告警到该团队。
3. **单轮 > 20 个 content block** —— 触发 Anthropic 回看窗口限制，症状是账单暴涨且零报错。装配时插入中间 breakpoint。
4. **N 路 fan-out 同时发同前缀请求** —— 缓存条目要等第一条响应开始流式输出后才可读，N 条并发全是全价写。**正确做法：同前缀 single-flight 预热**——先放 1 条、等到首 token，再放其余。agent 扇出场景能直接省几十个百分点。

### 精确缓存（exact-match cache） vs 语义缓存（semantic cache）

| | 精确缓存 | 语义缓存 |
|---|---|---|
| 键 / 命中率 | `hash(tenant, authz_version, model_artifact, normalized_input, tools, all_behavioral_params, policy_version)`；命中率取决于业务 | query embedding 相似度 > 经评测确定的阈值；命中率取决于语料 |
| 风险 | 中：即便键相同，模型也可能非确定；还要处理时效、撤权、模型/策略升级和敏感内容落盘 | **高**：语义近不等于答案可复用，尤其是实体、时间和权限不同的查询 |
| 适用 | 调用方显式声明可缓存、允许复用且有 TTL 的分类/抽取/embedding 等任务；`temperature=0` 也不是确定性保证 | 无个性化、低时效且有离线误命中评测的 FAQ |
| 网关默认 | **关或仅对明确 allowlist 开**；按租户/权限分区、加密、TTL、撤权失效 | **关**，opt-in；阈值由数据集与误命中成本决定，强制按 (tenant, 权限指纹) 分区 |

**语义缓存的隐藏杀伤力是污染评测**：一次 A/B 的差异可能全部来自缓存命中率而非模型。影子流量（shadow traffic）/评测流量必须**强制绕过所有缓存**。

### 跨租户（cross-tenant）缓存：性能杠杆与最大泄露面的直接冲突

[PROMPTPEEK（NDSS 2025）](https://www.ndss-symposium.org/wp-content/uploads/2025-1772-paper.pdf) 实证：共享 prefix cache 的时序侧信道（timing side channel）可**逐 token 重建他人的 prompt**——已知模板时成功率 99%，**无任何背景知识时 95%**；攻击面覆盖 vLLM / SGLang / LightLLM / DeepSpeed（完整数据与缓解矩阵见 [`03-saas-platform/04-isolation-and-compliance.md §8.4`](../03-saas-platform/04-isolation-and-compliance.md#84-kv-cache最大的性能杠杆也是最大的跨租户泄露面)）。这是一个真张力：

```
跨租户共享 prefix cache = 最大性能杠杆，同时 = 已被实证的跨租户 prompt 泄露面
⇒ 默认：同租户内共享，跨租户关闭 —— 你主动放弃了多租户场景的最大杠杆
⇒ 补偿：系统提示做成"每租户一份"而非"全局一份"，租户内仍能拿到 80%+ 收益
⇒ 硬约束：密钥、PII 绝不放进会被缓存的前缀
```
用托管 API 也要写进 DPIA：OpenAI 的 ZDR 仍会保留**加密的 prompt cache tensors 最长 24 小时**（[data controls](https://developers.openai.com/api/docs/guides/your-data)）。

---

## 8. 深挖五：流式代理（streaming proxy）——细节全在这里

### 内部事件流与超时

不要"攒完再回"。归一化成内部事件流，边转边发，输出侧按客户端要的协议（OpenAI 兼容 SSE / Anthropic 事件 / gRPC）重新编码。**成本是每事件一次解析 + 一次序列化**——网关唯一真实的 CPU 开销，压测按 token/s 而不是 RPS 建模。

```
IR Event = { type, index, delta?, tool_call?, usage?, finish?, error? }
  type ∈ { start, text_delta, thinking_delta, tool_call_delta, usage, finish, error }

TTFT_TIMEOUT = 60 s     # 首 token。200k 输入的 prefill 合法地慢
IDLE_TIMEOUT = 30 s     # 两个 token 之间的最大间隔 ← 真正的健康判据
HARD_TIMEOUT = 900 s    # 绝对上限，防无限生成/失控循环
```
> **超时判据是 token 之间的间隔，不是墙钟（wall clock）。** 用总时长做超时等于告诉用户"你不许生成长文档"：一个 8k token 的输出在 40 tok/s 下合法地跑 200 秒，而一个卡死的连接在 5 秒内就该被杀掉。

### 中途错误：HTTP 200 已经发出去了

流式第一个字节发出后状态码和普通响应头就锁死了。上游在第 3,000 个 token 挂掉，你**不能**改回 500，也不能在最后补一个普通 header。必须：①流内发带 `partial: true` 的终止 `error` 事件；②明确结束流，并要求 SDK 只有看到 `finish` 终止事件才算成功；③记录部分用量并按公开策略计费/计配额。若客户端和传输层显式协商支持 HTTP trailers，可以把 `x-llm-stream-outcome` 放在 trailer；否则终态只放在流内事件与服务端计量记录。**静默截断（silent truncation）**会让下游把不完整内容当成功结果，是这里的 P0 风险。

### 取消传播（cancellation propagation）

客户端断开 → 立刻 abort 上游。不做的三个后果：上游继续生成（继续计费）、并发额度被占、agent 重试时并发翻倍。**关键细节：usage 通常只在流的最后一个事件里给**，客户端中断时你永远拿不到它。所以网关必须**在流上自行累计输出 token**（按 delta 计数或本地 tokenizer），并在计量事件上打 `usage_source: estimated`，让对账（reconciliation）层知道这条不精确。

### 流式中的内容过滤：一个没有好答案的问题

输出侧 PII/敏感词过滤面对三件事：敏感串跨 token 边界（`"1234-5678-9012-3456"` 会被切成 6 个 token）；要检出必须**缓冲**，缓冲直接抬高感知延迟；检出时前面的内容已经发出去了，**收不回来**。三个姿势，明码标价：

| 姿势 | 延迟代价 | 有效性 |
|---|---|---|
| 滑动窗口缓冲（保留尾部 N 字符，如 N=64） | 每次 flush 延后 N 字符，TTFT 基本不变 | 能抓住跨边界的模式匹配（卡号/邮箱/密钥），抓不住语义级 |
| 分句缓冲（按句号/换行 flush） | 感知延迟明显变差（+0.5–2 s） | 可以跑轻量分类器 |
| 不在流上过滤，改在**入口**和**副作用点**拦 | 0 | **推荐**：出口管控（谁能收到）比内容过滤（说了什么）可靠得多 |

> 网关能可靠做的是**入站策略（inbound policy）**（拒绝含密钥的 prompt、按租户禁模型）与**审计留痕**。指望在 200 tok/s 的流上做可靠的输出侧内容安全，是在用概率性手段守一个确定性边界。

---

## 9. 深挖六：可观测与计量

### 统一计量模型（unified metering model，四类 token 必须分开）

```sql
-- ClickHouse：网关侧计量事实表 —— 你自己的"真相"，用于对账
CREATE TABLE llm_usage_events (
  event_id UUID,               -- 稳定幂等键：可由 request_id 派生，或首次生成后持久化并跨重试复用
  ts DateTime64(3),
  tenant_id LowCardinality(String), team_id LowCardinality(String),
  actor_id String,             -- 人类用户 或 agent 的独立身份
  on_behalf_of String,         -- agent 的人类 sponsor（空 = 人类直接调用）
  virtual_key_id String,
  route_alias LowCardinality(String),        -- 客户请求的别名
  served_provider LowCardinality(String),
  served_model LowCardinality(String),       -- **实际服务的模型工件/版本**，不是别名
  price_tier LowCardinality(String),          -- batch/standard/priority 等合同档位
  input_tokens UInt32, cache_read_tokens UInt32,   -- 0.1× 计价
  cache_write_tokens UInt32,                       -- 1.25× / 2× 计价
  output_tokens UInt32, reasoning_tokens UInt32,   -- reasoning 口径各家不同，单独归因
  ttft_ms UInt32, total_ms UInt32,
  attempt_no UInt8,            -- 重试第几次 —— 重试的成本必须可见
  stream_outcome Enum8('complete'=1,'client_abort'=2,'upstream_error'=3,'idle_timeout'=4),
  usage_source Enum8('provider'=1,'estimated'=2), degraded UInt8,
  record_version UInt64                      -- 同一 event 的单调更新版本
) ENGINE = ReplacingMergeTree(record_version) ORDER BY (tenant_id, event_id);

-- 成本计算放后端，不要把单价烧进 SDK；价目表必须带生效区间
SELECT tenant_id, served_model,
       sum(input_tokens)/1e6*p.input_price + sum(cache_read_tokens)/1e6*p.cache_read_price
     + sum(cache_write_tokens)/1e6*p.cache_write_price + sum(output_tokens)/1e6*p.output_price AS usd
FROM llm_usage_events FINAL AS e JOIN model_prices AS p
  ON p.provider = e.served_provider
 AND p.model = e.served_model
 AND p.price_tier = e.price_tier
 AND e.ts >= p.valid_from AND e.ts < p.valid_to
WHERE ts >= now() - INTERVAL 1 DAY GROUP BY 1,2 ORDER BY usd DESC;
```
价目表**带半开生效区间 `[valid_from, valid_to)`**是硬要求，并在写入时拒绝同一 provider/model/tier 的重叠区间；否则切价瞬间会 join 两次。历史账单不能被新价改写。`ReplacingMergeTree` 的去重是异步的，示例查询用 `FINAL` 只为说明正确性；生产大表通常用上游幂等写入/物化聚合避免昂贵的 `FINAL`。若 reasoning token 单独计价，也要把它加入价目与公式。

### 三方对账（必须有）

`provider 侧账单 ←→ 网关侧 usage_events ←→ 应用侧业务事件`，每日跑差异，容忍阈值 **0.5%**，超阈值告警。token 计费下 0.5% 的事件丢失在千万美元 ARR 上就是可观漏损，**且方向永远对供应商不利**（丢事件 = 少收钱）。三方不一致的常见根因：客户端中断的 `estimated` 用量、重试的重复计费、流式中途错误的部分输出。

### OTel：不要以为它已经是标准

[GenAI 语义约定](https://github.com/open-telemetry/semantic-conventions-genai)（2026-07 状态）**全部条目仍是 Development，0 项 GA**，自 2024-08 起至少 5 次破坏性改名（`prompt_tokens`→`input_tokens`、`gen_ai.system`→`gen_ai.provider.name`）；主 semconv 已在 v1.42.0 弃用全部 `gen_ai.*`、v1.43.0 移除。**三条对策都要做**：①采集与查询之间加一层**内部规范化 schema**（自有属性名 + `schema_version`）；②查询侧对新旧属性名 `coalesce`，否则跨版本静默丢数据；③**registry 里没有 cost 属性、没有 tenant 属性** —— `app.tenant.id` / `app.team.id` / `app.route.alias` 必须自定义，且**一开始就打全，事后不可回溯**。

### PII 与留痕

**prompt/response 内容属性在 OTel 里是 opt-in、默认不采——保持这个默认。** 网关提供三档：`none`（默认）/ `hashed`（内容哈希，用于去重与缓存分析）/ `full`（按租户白名单 + 采样率、目的与保留期）。租户等高基数字段放 trace/log 或离线聚合，不直接做无界 metrics label。审计日志应含模型/模板版本、工具调用与授权决策等必要元数据；参数、结果和最终输出默认存哈希或脱敏摘要，只有合法依据、最小权限与明确保留期同时满足时才留原文。具体法规与审计证据要求由法务/审计按适用范围确认。

---

## 10. 深挖七：安全

| 面 | 做法 | 反模式 |
|---|---|---|
| **密钥集中托管** | provider 真实密钥只存在网关的 KMS/Vault 里；用户只拿**虚拟 key**（可撤销、有 scope、有预算、有 TTL） | 把真实 API key 发给团队 —— 泄露后无法定位来源、无法单独撤销 |
| **Agent 身份** | Agent 有**独立可追溯身份 + 人类 sponsor**（`actor_id` + `on_behalf_of`）；禁止复用人类凭证 | token 透传。下游日志主体错误 → 无法追责、无法单独封禁 |
| **越权（confused deputy）** | 虚拟 key 的 scope 里显式声明可用模型、可用路由、数据驻留域；策略在**网关侧**判定，不信客户端传的 `tenant_id` | 用请求体里的 tenant 字段做隔离 |
| **日志泄露** | 全文留痕默认关；日志管道内做 PII 检测 + 结构化脱敏（redaction）；**日志系统的访问权限单独审批** | 把 prompt 原文写进普通应用日志，然后被全公司的日志检索工具索引 |
| **数据驻留** | 存储位置 / 推理位置 / 缓存位置是**三个独立开关**。OpenAI 存储可选 10 区但"区域内处理"仅 US/EU/UAE 三区；Anthropic `inference_geo: "us"` 约 1.1× 溢价 | "我们选了 EU 存储所以合规" —— 推理仍可能在境外 |
| **内容策略** | 入站拒绝（含密钥的 prompt、被禁模型）+ 出口管控 + 审计。`CONTENT_FILTERED` **绝不换 provider 重试** | 用换 provider 绕过安全过滤 = 合规套利，审计时是重大发现 |
| **供应链（supply chain）** | 若网关同时代理 MCP，把每个 MCP server 当**独立的不可信安全域**；注意 [MCP 2026-07-28 规范](https://modelcontextprotocol.io/specification/2026-07-28/changelog) 已把 `Mcp-Method` / `Mcp-Name` 头做成必填以支持网关无需解 body 路由 | 复用同一个 token 给多个 MCP server |

**买还是自建（buy vs build）**：[LiteLLM](https://github.com/BerriAI/litellm)、Portkey、Kong AI Gateway 在 2026 已收敛成同一形态——虚拟 key + `(key, user, team) × (TPM, RPM, budget)` 三元组 + 路由 + 重试 + 缓存 + 可观测（LiteLLM 开源版需 Postgres/Redis 承载 key 与预算状态）。**先用现成的**，除非你有前缀亲和路由 / 自建推理池 / 特殊合规这三类需求之一。但要把网关当**关键路径上的安全组件**来运维：`CVE-2026-59822`（LiteLLM MCP 认证绕过，CVSS 8.8）就是提醒——网关持有全公司的模型凭证，它被攻破的爆炸半径（blast radius）是全部。

---

## 11. 多 Provider 的一致性问题（最容易被忽略的一节）

同一个 prompt 在不同 provider 上：输出不同、token 数不同、拒答倾向不同（Claude 系倾向不确定时弃答 abstain，GPT 系幻觉率 hallucination rate 最高）、工具调用格式不同、finish reason 语义不同。

**三个连锁后果：** ①**缓存被打碎** —— 缓存键必须含 `served_model`；别名级缓存（`chat-default`）是错的，今天路由到 Sonnet、明天到 Opus，同一个键返回两种答案。②**评测不可比** —— 换 provider 后指标变化可能来自弃答/幻觉倾向而非能力差异，**评测必须按 `served_model` 分层**，不能按别名聚合。③**回归不可控** —— 调用方针对 A 调好的下游解析器在 B 上会碎。

**网关必须提供的三件事：**
```
① served_model 出现在响应头 + 计量事件 + trace 属性（三处，缺一处就查不动线上问题）
② 影子流量 mirror：复制线上请求给候选模型，两边打分，候选输出**永不返回用户**
③ 固定路由 pinning：调用方可把某个别名钉死在具体模型+版本上，直到自己解锁
```
影子模式 + 同一个 correlation ID 关联结果与打分，是在**真实流量分布**上比较候选模型的常见手段；脱敏生产回放、离线 eval 和人工抽查仍各自覆盖不同风险。常见上线链是离线 eval → 去副作用影子 → 金丝雀（canary）→ 扩量，尽量复用同一套 scorer。回滚阈值要由 rubric 基线方差、样本量、业务风险和标签延迟决定，不能跨产品固定成 2–3 个百分点或 15–60 分钟。

---

## 12. 失败模式清单

| 失败模式 | 信号 | 对策 |
|---|---|---|
| **静默 cache miss** | `cache_read_tokens` 骤降为 0，账单涨 3–5× 而流量不变 | 做成一等监控指标 + 前缀指纹变化率告警 |
| **failover 后缓存不回迁** | 一次故障后成本永久抬高 | 亲和路由要有"回归主接入点"的收敛逻辑，不能只有溢出没有回落 |
| **预扣状态悬空** | 租户被限流但已确认用量不高 | 正常路径幂等 settle；崩溃路径标 `unknown` 并对账，只退还已确认未消费部分，不能 TTL 到期盲目全退 |
| **重试放大（retry amplification）** | 上游 429 时网关自身流量翻倍 | 重试预算（retry budget）≤10%、指数退避 + 抖动、熔断半开 |
| **静默截断** | 下游 JSON 解析错误率上升，网关无告警 | 流内终止 error + `partial: true`；仅在协商支持时用 HTTP trailer，SDK 未见 finish 一律失败 |
| **控制面故障拖垮数据面** | 网关 5xx 与配置服务同步下降 | 使用带有效期的最后已知良好快照；无有效快照时，认证/隔离/驻留/硬预算 fail closed，只有批准过的低风险功能可退化 |
| **限流 Redis 热点** | Redis p99 0.5 ms → 5 ms | 切本地租约模式，接受 1–3% 超发 |
| **模型下线/改名** | 上游 404 或行为突变 | 模型注册表带生命周期状态；provider 弃用公告接入告警；别名层是你唯一的缓冲带 |
| **tokenizer 换代** | 同样任务突然被截断；预算莫名超支 | 跨代升级先跑 `count_tokens` 重新标定 `max_tokens` 与限流阈值（Claude 4.7+ 约 +30% token） |
| **单任务跑飞** | 一个 workflow 吃掉团队当天 TPM | 单会话/单任务绝对 token 上限 + 组织级 spend limit |

---

## 13. 什么时候不要建 LLM 网关 / 反模式

**不要建的情况：** 团队 < 3 个、单一 provider、日调用 < 1 万次 —— 一个共享客户端库 + 一份配置就够，网关此时是净负债（多一跳延迟、多一个 SPOF、多一套要值班的东西）；只是为了"统一 SDK" —— 那是库的活，网关的正当理由只有三个：**集中密钥与策略**、**集中计量与预算**、**集中缓存与路由**；延迟极度敏感且只有一个模型 —— 直连省掉 5–25 ms 和一次 TLS。

**明确的反模式：**

1. **网关偷偷改写用户 prompt**（注入系统提示、加安全前缀、做"prompt 优化"）。三重伤害：破坏前缀缓存、评测不可复现、客户不知道自己在跑什么。要加就**显式**：做成可声明的 `prompt_template_id`，并出现在计量事件与审计日志里。
2. **在网关里做 agent 循环 / 重排 / 长任务编排。** 有状态的东西放应用层或专门的 runtime。
3. **默认开语义缓存**（污染评测、跨权限边界、时效性问题返回过期答案）；**跨模型自动降级默认开**（见 §5）；**总时长超时**（见 §8）。
4. **同步依赖计量系统** —— 计费系统的可用性会变成产品的可用性。
5. **没有 break-glass。** 网关是强制路径时应准备经审批、有留痕、限时且最小权限的备用数据面或预配置直连凭证；它仍须保留租户/驻留等硬边界，不能把“应急”变成无治理绕行。
6. **把网关可用性目标定得和业务一样。** 它在所有业务的关键路径上，目标要高一档；同时数据面必须能在控制面全挂时继续跑。

---

## 14. 演进路线

| 阶段 | 形态 | 撞墙条件与信号 |
|---|---|---|
| **v0** | 共享客户端库 + 中心配置；密钥仍在各服务 | 信号：密钥泄露事故、没人说得清上个月钱花在哪 |
| **v1** | 集中网关：虚拟 key、多维限流、计量事件、审计留痕、OpenAI 兼容入口 | 信号：账单里 80% 是"未知"归属；某团队一次跑飞打爆全公司配额 |
| **v2** | 缓存治理（前缀指纹 + breakpoint 自动生成 + single-flight 预热）、多接入点 failover、影子评测、成本路由 | 信号：缓存命中率上不去（前缀被应用层写坏）；换模型没人敢批 |
| **v3** | 接入自建推理池：KV cache 感知路由（[GAIE](https://github.com/kubernetes-sigs/gateway-api-inference-extension) 已 GA，EPP 打分）、按租户的 cell 化、区域化数据面 | 信号：托管 API 成本压不下去且流量可预测；合规要求推理不出境 |

**每一步都要能独立回滚。** v2 的亲和路由要能一键退回"最少连接"，v3 的自建池要能一键把流量全部切回托管 API —— 因为自建推理的盈亏平衡对利用率极度敏感（利用率从 100% 掉到 20%，单位成本涨 5 倍）。

---

## 面试官会追问

1. 为什么按 token 限流而不是按请求数？流式请求在开始时不知道输出长度，你怎么预扣和结算？漏掉结算会怎样？
2. 网关进程可以无状态，为什么路由仍要感知缓存和队列状态？自建推理与托管 API 分别能控制到哪一层？
3. Anthropic 挂了，你自动切到 OpenAI。列出这个决定会造成的三个下游问题。你会怎么把它变成安全的默认值？
4. 流式响应已经发出了 3,000 个 token，上游断了。HTTP 状态码是 200，你怎么办？这条请求要计费吗？
5. 客户端中途断开连接，你从哪里拿到 usage？拿不到怎么办？对账时怎么标记？
6. 用户抱怨"这个月账单翻了 3 倍但流量没变"。排查顺序是什么？第一个要看的指标是哪个？
7. 跨租户共享 prefix cache 能显著提升命中率，但你不做。为什么？放弃了多少收益，用什么补回来？
8. 三个团队各自跑评测，结论互相矛盾。作为网关 owner，你怀疑哪三件事？

---

**按训练路径阅读** → 回 [START-HERE](../START-HERE.md) 按所选路径继续；页尾链接只表示本目录或专章的顺读顺序。

**案例顺读下一篇** → [03-multi-tenant-vector-search.md](03-multi-tenant-vector-search.md)
