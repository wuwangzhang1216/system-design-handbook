# Senior System Design Handbook

> **通用 full-stack / 后端 senior 系统设计面试的完整备考材料。**
> 主体不讲"什么是负载均衡"，讲的是：在什么约束下选什么、代价是什么、失败时会怎么塌、怎么用数字论证你的选择。
> **唯一的例外是第一篇** [00-concepts.md](00-foundations/00-concepts.md) —— 它假设你什么都不知道，
> 把地基上的每个词真正讲清楚（包括负载均衡在那条链路上到底在干什么），因为剩下 68 篇全都站在它上面。

**这本书有两种用法，它们互不冲突：**

| | 当**教材**顺读 | 当**手册**查阅 |
|---|---|---|
| 入口 | [00-foundations/00-concepts.md](00-foundations/00-concepts.md) → [知识地图](00-foundations/README.md) 的最短路径 → [START-HERE.md](START-HERE.md) 的四周计划 | 下面的 [目录](#目录) ｜ [07/03 速查表](07-interview/03-cheatsheet.md) ｜ [07/04 术语表](07-interview/04-glossary.md) |
| 顺序 | 有严格的概念依赖顺序，每一篇开头的「读这一章之前」写明了它的前置 | 没有顺序，按主题跳；每篇自带前置清单，缺什么当场补什么 |
| 适合 | 第一次系统学、或自测发现地基有洞 | 已经有工程经验、临时查一个决策或一个数字 |

**唯一一条硬规则**：无论哪种用法，遇到读不下去的段落，先回 [00-foundations/README.md 知识地图](00-foundations/README.md) 查这个词首次定义在哪一节 —— 卡住通常不是这一篇难，是前面漏了一站。

## 先选一条：你为什么打开这个仓库

| 你的情况 | 去哪 |
|---|---|
| 😵 **看不懂 / 术语一头雾水** | → **[00-foundations/00-concepts.md](00-foundations/00-concepts.md)** —— 全书唯一一篇假设你什么都不知道的。p99、分片、最终一致、幂等这些词在这里被真正讲清楚，之后 40 多篇不再解释。**先花 2–3 天读完它再谈路径** |
| 🎯 **我是来准备面试的** | → **[START-HERE.md](START-HERE.md)** —— 10 分钟自测 → 按分数选路径 → 四周逐日计划。**别从这个 README 往下读**，那是查阅顺序不是学习顺序 |
| 🔍 **我是来查一个具体问题的** | → 下面的 [目录](#目录)。按主题跳，不要通读 |
| 🗺️ **我想知道概念之间谁依赖谁** | → **[00-foundations/README.md](00-foundations/README.md)** —— 概念依赖图 + 最短阅读路径 + 「这个词首次定义在哪一节」对照表 |
| 🎤 **知识够了，卡在开口和时间管理** | → **[PRACTICE.md](PRACTICE.md)** —— 训练流程、录音自查清单、自评 rubric、30 天排期 |

**默认读者定位**：有 3–8 年经验的 **full-stack / 后端工程师**，目标是 **senior 后端 / 全栈 / 平台岗**的系统设计面试。
不是 AI 岗专用材料，也不是研究生课程 —— 全书的选材标准只有一条：**它会不会决定你面试里的分数。**

**本书现在覆盖三个方向**，共用同一套 00–03 / 05 / 06 / 07 的通用地基，只在专章上分岔：

| 方向 | 专章 | 路径 | 判据（看 JD 里团队做什么，不看公司做什么） |
|---|---|---|---|
| **通用后端 / 全栈**（默认） | — | [A](START-HERE.md#3-路径-a四周逐日计划) / [B](START-HERE.md#4-路径-b10-天冲刺) | 电商、支付、SaaS、基础设施、平台组 |
| **AI Agent / LLM 应用** | [04](04-ai-agent-systems/)（8 篇） | [C](START-HERE.md#5-路径-cai--基础设施岗的额外-2-周) | JD 里有「LLM / Agent / RAG / prompt / GPU 推理」 |
| **ML Systems / 推理** | [08](08-ml-systems/)（9 篇）+ [06/18–21](06-case-studies/) | [F](START-HERE.md#6-路径-fml-systems--inference-engineer6-周) | JD 里有「模型服务 / 推理 / 特征平台 / 排序 / 实验平台 / 模型上线」 |

> ⚠️ **04 和 08 都是可选专章，而且它们不是一回事 —— 别当成同一块内容。**
>
> | | **04 · AI / Agent 系统** | **08 · ML Systems** |
> |---|---|---|
> | 对象 | **LLM 与 Agent 应用**：别人训好的大模型，你在它上面搭产品 | **通用 ML 模型的训练产物**：你团队自己训出来的 XGBoost / DNN / 双塔，怎么上线与服务 |
> | 典型问题 | prompt 怎么组、KV cache 怎么省、Agent 循环怎么终止、提示注入怎么防 | 特征怎么对齐、批处理怎么组、金丝雀判据怎么写、漂移怎么检出 |
> | 延迟量级 | 秒级 | **毫秒级**（整条排序链路 100–200 ms） |
> | 质量怎么判 | eval 集 + LLM-as-judge，没有标签 | **有标签**：AUC / NDCG 离线判，A/B 在线判 |
>
> **两章都读的只有一种人**：面「推理平台」且这个平台同时服务 LLM 和传统模型。其余情况按 JD 选一章。
> **通用面试路径（路径 A / B）两章都整章跳过** —— 加起来约占全书 1/4 篇幅，对电商、支付、SaaS、平台组的面试一个考点都不会出现。

**中英双语**：正文里关键概念首次出现时都跟上英文原词（平均每篇 40–70 处），
`07/04` 把它们汇成一张 545 条对照表，`07/05` 是配套英文话术库，`07/06` 是一场英文模拟面试全逐字稿。
中文面试和英文面试用的是同一套材料 —— 你不需要为了英文面试再读一遍别的书。

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

> 这是**查阅顺序**，不是学习顺序。想按学习顺序走，去 [START-HERE.md](START-HERE.md)。

### 00 · 基础内功 [`00-foundations/`](00-foundations/)
| 文件 | 内容 |
|---|---|
| [README.md](00-foundations/README.md) | **知识地图**：概念依赖图（谁是谁的前提）、零基础的最短阅读路径、「这个词首次定义在哪一节」对照表 |
| [00-concepts.md](00-foundations/00-concepts.md) | **全书唯一一篇假设你什么都不知道的。** 请求的一生、延迟/吞吐/并发、p99、扩展、副本与分片、一致性、事务与隔离级别、同步异步、有无状态、可用性、缓存/队列/索引 —— 每个词都给定义，且定义里不含未定义的词。**这一篇必须按顺序读**，后面 40 多篇默认你已经站在它上面 |
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
| [06-replication.md](01-building-blocks/06-replication.md) | 同步/异步/半同步的 RPO 定价、复制延迟与读己之写、failover 与脑裂、多主与 Quorum、ML 场景的复制 |

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

### 04 · AI / Agent 系统 [`04-ai-agent-systems/`](04-ai-agent-systems/) — 🔶 可选 · 仅 **LLM / Agent 应用**岗需要

> **通用 full-stack / 后端面试路径会整章跳过这里。** 见上文的读者定位说明。
> 面 AI 产品 / LLM 网关 / Agent 基础设施 → 这 8 篇是你的主场，走 [START-HERE 路径 C](START-HERE.md#2-选你的路径)。
> **不要和 [08 章](08-ml-systems/)混淆**：04 讲的是「别人训好的 LLM，你在它上面搭应用」，08 讲的是「你自己训的模型怎么上线与服务」。

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

### 06 · 案例研究 [`06-case-studies/`](06-case-studies/) — 21 篇
每个案例按 **需求澄清 → 估算 → 高层设计 → 深挖 → 失败模式 → 演进** 的完整流程走，末尾都有「面试官会追问」和自测题。

> **下表按面试重要性排序，不按文件名排序**（物理文件名是历史遗留，08–17 才是最高频的那批）。
> 通用面试：**先做 A 组的 10 篇**，B 组按你的岗位方向挑 2–3 篇，C 组留到面试前一天；D 组仅 ML Systems 岗需要。

#### A 组 · 经典高频题（08–17）— 通用面试的主体，优先做完

这 10 题覆盖了绝大多数公司系统设计轮的实际出题范围，归约到 **4 种母题**：
**A 唯一性 · B 扇出与聚合 · C 并发与库存 · D 长连接与状态**。认出母题比背题重要。

| 文件 | 案例 | 母题 |
|---|---|---|
| [08-url-shortener.md](06-case-studies/08-url-shortener.md) | 短链接服务 —— 零协调发号器、100:1 读放大的缓存层、301 为什么是不可撤销的决定 | A |
| [09-rate-limiter.md](06-case-studies/09-rate-limiter.md) | 限流服务 —— 200 进程共享一个计数付多少延迟、限流器自己挂了系统处于什么状态 | A+C |
| [10-news-feed.md](06-case-studies/10-news-feed.md) | 信息流 / 时间线 —— 推拉阈值怎么算出来、收件箱里到底存什么、大 V 长尾怎么物理隔离 | B |
| [11-chat-messaging.md](06-case-studies/11-chat-messaging.md) | 聊天 / 消息系统 —— 有状态服务的路由、两级租约、"推送管延迟、拉取管正确性" | D |
| [12-distributed-cache.md](06-case-studies/12-distributed-cache.md) | 分布式缓存 —— 一致性哈希只是入场券，回源路径上那个能说出数字的并发上限才是分数线 | B |
| [13-ride-hailing.md](06-case-studies/13-ride-hailing.md) | 打车 / 附近的人 —— 地理索引、司机独占性、epoch 与超时收敛；99.7% 的写可以丢 | C+D |
| [14-ticket-booking.md](06-case-studies/14-ticket-booking.md) | 票务系统 —— 不超卖、不锁死、能自动释放三个约束同时成立，需要三个互不依赖的机制 | C |
| [15-payment-system.md](06-case-studies/15-payment-system.md) | 支付系统 —— 幂等键 + 状态机 + 对账；胜负手是和三方渠道状态不一致时靠什么收敛 | A+C |
| [16-search-autocomplete.md](06-case-studies/16-search-autocomplete.md) | 搜索自动补全 —— 一个必须每小时变一次的东西，怎么做成永远不变的东西来服务 | B |
| [17-object-storage.md](06-case-studies/17-object-storage.md) | 对象存储 / 文件上传 —— 元数据与字节分离，以及"字节永远不经过应用服务器"的全部推论 | — |

#### B 组 · 现代 / 平台方向案例（01–06）— 按岗位方向挑

| 文件 | 案例 | 方向 |
|---|---|---|
| [05-realtime-collaboration.md](06-case-studies/05-realtime-collaboration.md) | 实时协作编辑（CRDT/OT、在线状态、权限撤销、离线合并） | 通用 · 协作/文档类产品 |
| [06-notification-platform.md](06-case-studies/06-notification-platform.md) | 通知平台（扇出、去重、优先级、退避、静默时段） | 通用 · 与 10/11 同构，性价比高 |
| [04-usage-based-billing.md](06-case-studies/04-usage-based-billing.md) | 用量计费系统（准确性、幂等、对账、实时配额） | SaaS / 平台 / 计费组 |
| [03-multi-tenant-vector-search.md](06-case-studies/03-multi-tenant-vector-search.md) | 多租户向量检索（10 万租户、隔离、冷热分层） | 🔶 AI / 检索基础设施 |
| [02-llm-gateway.md](06-case-studies/02-llm-gateway.md) | 企业级 LLM 网关（路由、限额、缓存、审计） | 🔶 AI 基础设施 |
| [01-ai-coding-agent-platform.md](06-case-studies/01-ai-coding-agent-platform.md) | 云端 AI 编码 Agent 平台（沙箱、并发、计费、安全） | 🔶 AI 岗 |

#### C 组 · 复习速查

| 文件 | 用途 |
|---|---|
| [07-classic-canon.md](06-case-studies/07-classic-canon.md) | **A 组 10 题的压缩速查版**（每题只写 20%，不能替代展开版）。归约到 4 种母题。两个用法：① 开局用一天扫完摸底，标出你说不出子问题的那几题再去读展开版；② 面试前一天连着扫一遍，30 分钟，检查母题识别还在不在 |

#### D 组 · ML Systems 案例（18–21）— 🔷 仅 ML Systems / Inference 岗

> 这 4 篇是 [08 章](08-ml-systems/)的合成题，**通用面试路径整组跳过**。走 [路径 F](START-HERE.md#6-路径-fml-systems--inference-engineer6-周) 的人在 Week 5 做完这四道。

| 文件 | 案例 | 这题的枢纽 |
|---|---|---|
| [18-model-serving-platform.md](06-case-studies/18-model-serving-platform.md) | 模型服务平台 —— 300 个模型共享 GPU 池 | GPU 上唯一能超售的资源是**时间**，显存不行；负载单位不是请求数 |
| [19-feature-store.md](06-case-studies/19-feature-store.md) | 特征平台 —— 训练和线上用同一份定义，为什么算出来不一样 | 在线延迟由 **feature view 个数**驱动（= 串行往返数），不是特征个数 |
| [20-ranking-service.md](06-case-studies/20-ranking-service.md) | 推荐/排序的在线推理 —— 多阶段漏斗的延迟预算 | 每请求 **1,000:1** 的特征读放大 ⇒ item 侧必须进程内驻留 |
| [21-ab-experiment-platform.md](06-case-studies/21-ab-experiment-platform.md) | A/B 实验平台 —— 分层分流、曝光口径、序贯检验 | MDE 缩 5 倍样本量涨 **25 倍**（平方关系）⇒ 决策指标不需要实时 |

### 07 · 面试 [`07-interview/`](07-interview/)
| 文件 | 内容 |
|---|---|
| [01-interview-framework.md](07-interview/01-interview-framework.md) | 45 分钟节奏控制、评分信号、常见失分点 |
| [02-question-bank.md](07-interview/02-question-bank.md) | 54 道题 + 每题的关键考点与"面试官在等的那句话" |
| [03-cheatsheet.md](07-interview/03-cheatsheet.md) | 速查：数字、公式、选型决策树、进考场前的 10 条自检 |
| [04-glossary.md](07-interview/04-glossary.md) | 545 条中英术语对照，15 组主题（含复制模式与 ML 系统/实验两组）；每条给一句能直接嵌进回答的整句，并标出直译会错的词、必须整块背的行话、重音容易念错的词 |
| [05-english-phrasebook.md](07-interview/05-english-phrasebook.md) | 英文话术库：澄清、提方案、认代价、被 challenge、承认不会、30 秒收尾；每个句式配"这句在传递什么信号"与反面例句 |
| [06-mock-interview.md](07-interview/06-mock-interview.md) | 45 分钟英文模拟面试全逐字稿（Design a Rate Limiter），保留停顿、自我修正与面试官打断；附 17 条评分锚点与档位判读 |

### 08 · ML Systems / 推理 [`08-ml-systems/`](08-ml-systems/) — 🔷 可选 · 仅 **ML Systems / Inference** 岗需要

> **入口是 [08-ml-systems/README.md](08-ml-systems/README.md)** —— 一张《ML Systems / Inference Engineer 考点 → 章节》映射表，
> 把 syllabus 的 46 项逐条指到具体文件的具体小节，并给出「这一项面试会怎么考」。**先看那一页再决定读哪几篇。**
>
> 通用面试路径（A / B）整章跳过。和 [04 章](04-ai-agent-systems/)的分工见上文读者定位里的对照表。

| 文件 | 内容 |
|---|---|
| [README.md](08-ml-systems/README.md) | **考点映射表**：通用 4 组 27 项 + ML 3 组 19 项 → 具体小节；以及本书补充的三块必考内容 |
| [01-ml-system-overview.md](08-ml-systems/01-ml-system-overview.md) | ML 系统的行为写在数据里、Hidden Technical Debt、ML 特有失败模式（坏掉时返回 200 OK）、延迟+质量双指标 SLO |
| [02-model-lifecycle.md](08-ml-systems/02-model-lifecycle.md) | Registry schema、模型版本四元组（SemVer 为什么不适用）、Checkpoint 大小与频率、保留成本算例、产物分发 |
| [03-model-loading-and-warmup.md](08-ml-systems/03-model-loading-and-warmup.md) | 冷启动九项分解（每项多少秒）、预热的收敛判据、预热池容量公式、双缓冲切换、显存超售与 LRU 卸载 |
| [04-online-inference.md](08-ml-systems/04-online-inference.md) | 在线/近线/离线、同步 vs 异步、streaming 的两个意思、动态批处理两个旋钮、准入控制、队列纪律（FIFO vs LIFO 会反转） |
| [05-inference-optimization.md](08-ml-systems/05-inference-optimization.md) | 180 ms 去哪了、Roofline、量化的 ROI、算子融合、前后处理经常比推理还慢、CPU/GPU 盈亏平衡、`nvidia-smi` 为什么会骗你 |
| [06-feature-and-data.md](08-ml-systems/06-feature-and-data.md) | 训练服务偏斜五种成因、时点正确性与 as-of join、为什么穿越会让离线指标"变好"、在线取数 10 ms 预算、新鲜度分级 |
| [07-model-quality-and-experimentation.md](08-ml-systems/07-model-quality-and-experimentation.md) | 时序划分、离线-在线鸿沟六因、分流、样本量与功效、CUPED、多重检验、交错实验、影子部署的边界、反馈采集 |
| [08-drift-and-monitoring.md](08-ml-systems/08-drift-and-monitoring.md) | 静默降级、数据漂移 vs 概念漂移、PSI 的两个错误用法、没有标签时监控什么、监控四层、何时重训、自动重训的风险 |
| [09-model-deployment.md](08-ml-systems/09-model-deployment.md) | 服务金丝雀判据搬到模型上为何失效、双层门禁、模型回滚 ≠ 代码回滚、多版本显存账与 pointer 路由、级联模型、端侧回滚不了 |

配套案例：[06/18 模型服务平台](06-case-studies/18-model-serving-platform.md) ｜ [06/19 特征平台](06-case-studies/19-feature-store.md) ｜ [06/20 排序服务](06-case-studies/20-ranking-service.md) ｜ [06/21 A/B 实验平台](06-case-studies/21-ab-experiment-platform.md)

---

## 学习路径

### 主入口：四条完整路径（在 [START-HERE.md](START-HERE.md) 里逐日排好）

**不要自己拼路径。** [START-HERE.md](START-HERE.md) 里有一个 10 分钟自测，答完按分数进对应路径，
每一天读什么、练什么、练多久都排好了，还有周检查点和进度勾选表。

| 路径 | 适合谁 | 时长 | 04 章 | 08 章 |
|---|---|---|---|---|
| **[A · 通用 full-stack / 后端](START-HERE.md#3-路径-a四周逐日计划)**（默认） | 大多数人。面 senior 后端 / 全栈 / 平台岗 | 4 周 / 33 个学习日（每天 60–90 min） | ✕ 跳过 | ✕ 跳过 |
| **[B · 10 天冲刺](START-HERE.md#4-路径-b10-天冲刺)** | 面试在 10 天内，或自测答对 10+ | 10 天（每天 90–120 min） | ✕ 跳过 | ✕ 跳过 |
| **[C · AI Agent / LLM 应用岗](START-HERE.md#5-路径-cai--基础设施岗的额外-2-周)** | 目标是 AI 产品 / LLM 网关 / Agent 基础设施 | 6 周 = 路径 A + 额外 2 周 | ✓ 主体 | ✕ 跳过 |
| **[F · ML Systems / Inference Engineer](START-HERE.md#6-路径-fml-systems--inference-engineer6-周)** | 目标是模型服务 / 推理 / 特征平台 / 排序 / 实验平台 | 6 周 / 37 个学习日（每天 60–115 min） | ✕ 跳过 | ✓ 主体 |

> **F 不是 A 的续集，是一条独立的 6 周路径**：它把路径 A 四周里 ML 岗用不上的部分（多租户、SaaS 平台、协作编辑）砍掉，
> 换成 syllabus 明确点名的通用四组（流量层 / 数据与状态 / 分布式 / 生产运行），压进 Week 1–2。
> 逐项映射见 [08-ml-systems/README.md](08-ml-systems/README.md)。

配套的练习方法（录音、自评 rubric、缺口记录、30 天排期）在 **[PRACTICE.md](PRACTICE.md)** —— 和路径一起用，不是二选一。

### 补充：按主题的补路径

下面六条**不是完整备考计划**，是「我只缺这一块」时的定点补课清单。
已经在走上面四条路径的人不需要它们 —— 那四条里已经包含了这些内容并排好了顺序。

**主题 A：补齐分布式内功**（2–3 周）
`00 全部` → `01 全部` → `05-reliability/03` → `06/12 分布式缓存` → `06/10 信息流`

**主题 B：转向 AI 基础设施**（1–2 周，假设已有分布式基础）
`00-foundations/01` → `04 全部` → `06/01` → `06/02` → `06/03`

**主题 C：SaaS 平台 / 创业技术负责人**（2 周）
`02/03 多租户` → `03 全部` → `05/01` → `06/04`

**主题 D：面试冲刺**（5 天，时间比路径 B 还紧时的最小版）
`07/03 速查` → `07/01 框架` → `06/07 经典题速解（一天过完 10 题）` → `06 的 A 组挑 3 篇精读` → `07/06 逐字稿对照自己的节奏` → `07/02 自测`

**主题 F：ML Systems 定点补课**（1 周，通用系统设计已过关，只缺 ML 那一半）
`08/README 映射表（先看，决定读哪几篇）` → `08/04 在线推理` → `08/09 模型部署` → `08/06 特征与偏斜` → `08/08 漂移监控` → `06/18` → `06/20`
时间只够四节的话，读 [08-ml-systems/README.md §5](08-ml-systems/README.md) 那一行「一周内面试」—— 那四节覆盖 ML 组 19 项里的 13 项。

**主题 E：英文面试冲刺**（3 天，中文技术已过关，卡在开口）
`07/04 术语表：只扫第 3 列「常见误译」` → `07/05 话术库：出声念，录音回放` → `07/06 逐字稿：先遮住候选人台词自己答，再对照` → `06/07 挑 3 题用英文各讲 6 分钟`
最后一天只做一件事：**把 07/05 的「被 challenge」和「30 秒收尾」两节练到不用想。**

---

## 怎么用这本手册

1. **当手册用时不要通读**，带着一个具体问题来查，看完立刻在自己系统上找对应物；**当教材用时不要乱序**，按 [知识地图](00-foundations/README.md) 的最短路径走，每一步都不会遇到未定义的词。**唯一必须按顺序读完的是 [00/00 基础概念](00-foundations/00-concepts.md)**，从 00/01 开始"读不懂就跳过、第二遍再回来"才成立。
2. **每篇末尾的「面试官会追问」**是自检清单，答不上来的地方就是你的缺口。
3. **数字要背**。`07/03` 里的那张表，是所有估算的地基。
4. **反例比正例值钱**。每篇都有「什么时候不要这么做」，那部分才是 Senior 和 Mid 的分水岭。
5. **术语表和话术库要出声用，不要读**。`07/04` 不是词典，别通读 —— 面试前一天只扫「常见误译」那一列，准备时卡住了再按主题跳回去查；`07/05` 的句式必须念出来并录音回放，听 hedge 词的数量和有没有升调。**默读能通过的检查，开口时一条都过不了。**
6. **读不等于会**。读的产出是知识，练的产出是表现，面试只买后者。方法在 [PRACTICE.md](PRACTICE.md)：一天一道题，出声讲，录音，对照清单打勾。**不录音就不算练过。**

---

## 五个入口文件的分工

| 文件 | 回答什么问题 | 什么时候打开 |
|---|---|---|
| **[00-foundations/00-concepts.md](00-foundations/00-concepts.md)** | 这些词到底是什么意思？ | 术语一头雾水时，以及自测答对 < 3 题时。**读完再谈路径** |
| **[00-foundations/README.md](00-foundations/README.md)** | 这个词的前提是什么？它首次定义在哪一节？ | 读某一篇卡住时（先查词，再决定要不要跳过） |
| **[START-HERE.md](START-HERE.md)** | 我今天该干什么？ | 第一次来，以及之后每一天 |
| **[PRACTICE.md](PRACTICE.md)** | 我该怎么练？ | 配好流程（10 分钟），之后每次练习前 |
| **README.md**（本文件） | 书里有什么？某个主题在哪一篇？ | 查东西的时候 |

---

## 贡献与许可

内容基于公开的工程实践与论文整理，欢迎 issue / PR 补充与纠错。
License: [MIT](LICENSE)
