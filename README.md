# Senior System Design Handbook

> 面向 **Senior / Staff 级别** 的系统设计手册。不讲"什么是负载均衡"，讲的是：
> 在什么约束下选什么、代价是什么、失败时会怎么塌、怎么用数字论证你的选择。
>
> 覆盖经典分布式系统 + 现代 SaaS 平台工程 + **LLM / Agent 基础设施**（2025–2026 最前沿的一批工程实践）。
>
> **全书术语中英对照**：正文里关键概念首次出现时都跟上英文原词（平均每篇 40–70 处），
> 章末 `07/04` 把它们汇成一张 445 条的对照表，`07/05` 是配套的英文话术库。
> 换句话说，中文面试和英文面试用的是同一套材料 —— 你不需要为了英文面试再读一遍别的书。

---

## 这本手册和别的有什么不一样

大多数 system design 资料停在 "Junior→Mid" 的层次：画三个框（LB / App / DB），加个 Redis，加个 Kafka，结束。

Senior 级别被考察的东西完全不同：

| 维度 | Mid 的回答 | Senior 的回答 |
|---|---|---|
| 需求 | 直接开画 | 先把 SLO、QPS、数据量、成本上限、合规边界问清楚，写在白板角上 |
| 存储选型 | "用 MySQL / 用 Mongo" | 按访问模式推导：写放大、读放大、空间放大三选二；这个 workload 的 p99 在哪一层被吃掉 |
| 一致性 | "用事务" | 明确到每条链路：这里可以 read-your-writes，那里必须 linearizable，代价是多一次 RTT |
| 扩展 | "加机器 / 分库分表" | 分片键选择 → 热点分析 → 再平衡方案 → 迁移期双写与回滚剧本 |
| 失败 | "加重试" | 重试预算、抖动、幂等键、熔断半开、隔板、优雅降级的**具体降级到什么** |
| 成本 | 不提 | $/请求、$/租户、单位经济模型，以及哪个旋钮能省 40% |
| 演进 | 一次画完 | v0 能上线，v1 加什么，什么时候会撞墙，撞墙前的信号是什么 |

**核心断言**：Senior 的设计能力 = 在不确定性下做可辩护的取舍，并且能把取舍量化。

---

## 目录

### 00 · 基础内功 [`00-foundations/`](00-foundations/)
| 文件 | 内容 |
|---|---|
| [01-fundamentals.md](00-foundations/01-fundamentals.md) | 延迟数字、CAP/PACELC、一致性模型全谱、幂等、背压、尾延迟放大 |
| [02-capacity-estimation.md](00-foundations/02-capacity-estimation.md) | 餐巾纸估算方法论、排队论（Little's Law / USL）、成本建模 |
| [03-tradeoff-framework.md](00-foundations/03-tradeoff-framework.md) | 需求澄清清单、决策记录（ADR）、非功能需求量化、演进式架构 |

### 01 · 构件层 [`01-building-blocks/`](01-building-blocks/)
| 文件 | 内容 |
|---|---|
| [01-storage-engines.md](01-building-blocks/01-storage-engines.md) | B-Tree vs LSM、RUM 猜想、OLTP/OLAP/HTAP、列存、向量索引内核 |
| [02-caching.md](01-building-blocks/02-caching.md) | 多级缓存、失效策略、穿透/击穿/雪崩、stampede、CDN 与边缘 |
| [03-messaging-and-streams.md](01-building-blocks/03-messaging-and-streams.md) | Kafka/Pulsar 内核、exactly-once 真相、Outbox、CDC、流处理语义 |
| [04-networking-and-edge.md](01-building-blocks/04-networking-and-edge.md) | L4/L7 负载均衡、连接管理、gRPC/HTTP3、Anycast、边缘计算 |
| [05-consensus-and-coordination.md](01-building-blocks/05-consensus-and-coordination.md) | Raft/Paxos、租约、分布式锁的正确做法、时钟与 HLC、TrueTime |

### 02 · 架构模式 [`02-architecture-patterns/`](02-architecture-patterns/)
| 文件 | 内容 |
|---|---|
| [01-microservices-vs-modular-monolith.md](02-architecture-patterns/01-microservices-vs-modular-monolith.md) | 服务边界的正确切法、分布式单体的病征、康威定律的工程后果 |
| [02-event-driven-and-cqrs.md](02-architecture-patterns/02-event-driven-and-cqrs.md) | 事件驱动、Saga、事件溯源、CQRS 的真实成本 |
| [03-multi-tenancy.md](02-architecture-patterns/03-multi-tenancy.md) | 多租户四种隔离级别、噪音邻居、分片路由、租户迁移 |
| [04-api-design-and-versioning.md](02-architecture-patterns/04-api-design-and-versioning.md) | REST/gRPC/GraphQL 选型、演进式 API、幂等与分页契约 |
| [05-data-platform.md](02-architecture-patterns/05-data-platform.md) | Lakehouse、数据契约、Data Mesh 的可行部分、指标层 |

### 03 · SaaS 平台工程 [`03-saas-platform/`](03-saas-platform/)
| 文件 | 内容 |
|---|---|
| [01-control-plane.md](03-saas-platform/01-control-plane.md) | 控制面/数据面分离、声明式协调循环、Cell 架构、单元化爆炸半径 |
| [02-billing-and-metering.md](03-saas-platform/02-billing-and-metering.md) | 用量计量管道、幂等出账、配额与限额、AI 产品的 token 计费 |
| [03-identity-and-authz.md](03-saas-platform/03-identity-and-authz.md) | OIDC/SAML/SCIM、RBAC→ABAC→ReBAC（Zanzibar 模型）、Agent 的授权 |
| [04-isolation-and-compliance.md](03-saas-platform/04-isolation-and-compliance.md) | 数据驻留、BYOK/租户密钥、SOC2/GDPR 的架构影响、审计日志 |
| [05-release-engineering.md](03-saas-platform/05-release-engineering.md) | Feature flag、渐进式交付、Schema 迁移六步法、多版本共存 |

### 04 · AI / Agent 系统 [`04-ai-agent-systems/`](04-ai-agent-systems/) ⭐ 最前沿
| 文件 | 内容 |
|---|---|
| [01-llm-serving-infra.md](04-ai-agent-systems/01-llm-serving-infra.md) | Prefill/Decode 分离、KV Cache、PagedAttention、连续批处理、GPU 调度 |
| [02-context-engineering-and-rag.md](04-ai-agent-systems/02-context-engineering-and-rag.md) | 上下文工程范式、混合检索、重排、GraphRAG、上下文腐化与压缩 |
| [03-agent-runtime.md](04-ai-agent-systems/03-agent-runtime.md) | Agent 循环、工具调用、MCP 协议、沙箱、长任务与可恢复执行 |
| [04-agent-memory-and-state.md](04-ai-agent-systems/04-agent-memory-and-state.md) | 记忆分层、状态机、checkpoint/resume、会话压缩 |
| [05-multi-agent-orchestration.md](04-ai-agent-systems/05-multi-agent-orchestration.md) | 编排 vs 编舞、子代理、扇出/扇入、何时**不要**用多代理 |
| [06-evaluation-and-observability.md](04-ai-agent-systems/06-evaluation-and-observability.md) | Eval 体系、LLM-as-judge 的偏差、轨迹追踪、在线回归 |
| [07-agent-security.md](04-ai-agent-systems/07-agent-security.md) | 提示注入、致命三要素、权限边界、出口管控、沙箱逃逸 |
| [08-cost-and-latency.md](04-ai-agent-systems/08-cost-and-latency.md) | 缓存分层、模型路由、投机解码、蒸馏、单位经济模型 |

### 05 · 可靠性 [`05-reliability/`](05-reliability/)
| 文件 | 内容 |
|---|---|
| [01-slo-and-error-budget.md](05-reliability/01-slo-and-error-budget.md) | SLI 定义、多窗口多燃尽率告警、错误预算政策 |
| [02-observability.md](05-reliability/02-observability.md) | 三支柱 + Profiling、OTel、高基数、采样策略、成本控制 |
| [03-resilience-patterns.md](05-reliability/03-resilience-patterns.md) | 超时预算、重试放大、熔断、隔板、负载卸载、优雅降级 |
| [04-incident-and-chaos.md](05-reliability/04-incident-and-chaos.md) | 事件指挥、无责复盘、混沌工程、演练 |
| [05-scaling-playbook.md](05-reliability/05-scaling-playbook.md) | 从 1k→1M QPS 的分阶段剧本、热点、分片再平衡、迁移 |

### 06 · 案例研究 [`06-case-studies/`](06-case-studies/)
每个案例按 **需求澄清 → 估算 → 高层设计 → 深挖 → 失败模式 → 演进** 的完整流程走。

| 文件 | 案例 |
|---|---|
| [01-ai-coding-agent-platform.md](06-case-studies/01-ai-coding-agent-platform.md) | 设计一个云端 AI 编码 Agent 平台（沙箱 / 并发 / 计费 / 安全） |
| [02-llm-gateway.md](06-case-studies/02-llm-gateway.md) | 设计一个企业级 LLM 网关（路由 / 限额 / 缓存 / 审计） |
| [03-multi-tenant-vector-search.md](06-case-studies/03-multi-tenant-vector-search.md) | 设计多租户向量检索服务（10 万租户 / 隔离 / 冷热分层） |
| [04-usage-based-billing.md](06-case-studies/04-usage-based-billing.md) | 设计用量计费系统（准确性 / 幂等 / 对账） |
| [05-realtime-collaboration.md](06-case-studies/05-realtime-collaboration.md) | 设计实时协作编辑（CRDT/OT / 在线状态 / 冲突） |
| [06-notification-platform.md](06-case-studies/06-notification-platform.md) | 设计通知平台（扇出 / 去重 / 优先级 / 退避） |
| [07-classic-canon.md](06-case-studies/07-classic-canon.md) | 经典题速解集：短链 / 限流 / feed / 聊天 / 打车 / 订票 / 支付 / 补全 / 对象存储等 10 题，归约到 4 种母题（唯一性、扇出、并发争抢、长连接状态），每题只写决定评分的那个子问题 |

### 07 · 面试 [`07-interview/`](07-interview/)
| 文件 | 内容 |
|---|---|
| [01-interview-framework.md](07-interview/01-interview-framework.md) | 45 分钟节奏控制、评分信号、常见失分点 |
| [02-question-bank.md](07-interview/02-question-bank.md) | 54 道题 + 每题的关键考点与"面试官在等的那句话" |
| [03-cheatsheet.md](07-interview/03-cheatsheet.md) | 速查：数字、公式、选型决策树、进考场前的 10 条自检 |
| [04-glossary.md](07-interview/04-glossary.md) | 445 条中英术语对照，14 组主题；每条给一句能直接嵌进回答的整句，并标出直译会错的词、必须整块背的行话、重音容易念错的词 |
| [05-english-phrasebook.md](07-interview/05-english-phrasebook.md) | 英文话术库：澄清、提方案、认代价、被 challenge、承认不会、30 秒收尾；每个句式配"这句在传递什么信号"与反面例句 |
| [06-mock-interview.md](07-interview/06-mock-interview.md) | 45 分钟英文模拟面试全逐字稿（Design a Rate Limiter），保留停顿、自我修正与面试官打断；附 17 条评分锚点与档位判读 |

---

## 推荐学习路径

**路径 A：补齐分布式内功**（2–3 周）
`00 全部` → `01 全部` → `05-reliability/03` → `06/05` → `06/06`

**路径 B：转向 AI 基础设施**（1–2 周，假设已有分布式基础）
`00-foundations/01` → `04 全部` → `06/01` → `06/02` → `06/03`

**路径 C：SaaS 平台 / 创业技术负责人**（2 周）
`02/03 多租户` → `03 全部` → `05/01` → `06/04`

**路径 D：面试冲刺**（5 天）
`07/03 速查` → `07/01 框架` → `06/07 经典题速解（一天过完 10 题）` → `06 挑 3 个案例精读` → `07/06 逐字稿对照自己的节奏` → `07/02 自测`

**路径 E：英文面试冲刺**（3 天，中文技术已过关，卡在开口）
`07/04 术语表：只扫第 3 列「常见误译」` → `07/05 话术库：出声念，录音回放` → `07/06 逐字稿：先遮住候选人台词自己答，再对照` → `06/07 挑 3 题用英文各讲 6 分钟`
最后一天只做一件事：**把 07/05 的「被 challenge」和「30 秒收尾」两节练到不用想。**

---

## 怎么用这本手册

1. **不要通读**。带着一个具体问题来查，看完立刻在自己系统上找对应物。
2. **每篇末尾的「面试官会追问」**是自检清单，答不上来的地方就是你的缺口。
3. **数字要背**。`07/03` 里的那张表，是所有估算的地基。
4. **反例比正例值钱**。每篇都有「什么时候不要这么做」，那部分才是 Senior 和 Mid 的分水岭。
5. **术语表和话术库要出声用，不要读**。`07/04` 不是词典，别通读 —— 面试前一天只扫「常见误译」那一列，准备时卡住了再按主题跳回去查；`07/05` 的句式必须念出来并录音回放，听 hedge 词的数量和有没有升调。**默读能通过的检查，开口时一条都过不了。**

---

## 贡献与许可

内容基于公开的工程实践与论文整理，欢迎 issue / PR 补充与纠错。
License: [MIT](LICENSE)
