# Senior System Design Handbook

> **通用 full-stack / 后端 senior 系统设计面试的完整备考材料。**
> 不讲"什么是负载均衡"，讲的是：在什么约束下选什么、代价是什么、失败时会怎么塌、怎么用数字论证你的选择。

## 先选一条：你为什么打开这个仓库

| 你的情况 | 去哪 |
|---|---|
| 🎯 **我是来准备面试的** | → **[START-HERE.md](START-HERE.md)** —— 10 分钟自测 → 按分数选路径 → 四周逐日计划。**别从这个 README 往下读**，那是查阅顺序不是学习顺序 |
| 🔍 **我是来查一个具体问题的** | → 下面的 [目录](#目录)。按主题跳，不要通读 |
| 🎤 **知识够了，卡在开口和时间管理** | → **[PRACTICE.md](PRACTICE.md)** —— 训练流程、录音自查清单、自评 rubric、30 天排期 |

**默认读者定位**：有 3–8 年经验的 **full-stack / 后端工程师**，目标是 **senior 后端 / 全栈 / 平台岗**的系统设计面试。
不是 AI 岗专用材料，也不是研究生课程 —— 全书的选材标准只有一条：**它会不会决定你面试里的分数。**

> ⚠️ **[04-ai-agent-systems](04-ai-agent-systems/) 是可选章节，仅 AI 岗需要。**
> 那 8 篇约占全书 1/6 篇幅，但对电商、支付、SaaS、基础设施、平台组的面试**一个考点都不会出现**。
> **通用面试路径（START-HERE 路径 A / B）整章跳过它。** 只有当 JD 里出现「LLM / Agent / 推理 / RAG / GPU」中任意一个词时，
> 它才从"可跳过"变成"面试官的主场"（走路径 C）。判断依据看**团队做什么**，不看公司做什么。

**中英双语**：正文里关键概念首次出现时都跟上英文原词（平均每篇 40–70 处），
`07/04` 把它们汇成一张 445 条对照表，`07/05` 是配套英文话术库，`07/06` 是一场英文模拟面试全逐字稿。
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

### 04 · AI / Agent 系统 [`04-ai-agent-systems/`](04-ai-agent-systems/) — 🔶 可选 · 仅 AI 岗需要

> **通用 full-stack / 后端面试路径会整章跳过这里。** 见上文的读者定位说明。
> 面 AI 产品 / 推理平台 / Agent 基础设施 → 这 8 篇是你的主场，走 [START-HERE 路径 C](START-HERE.md#2-选你的路径)。

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

### 06 · 案例研究 [`06-case-studies/`](06-case-studies/) — 17 篇
每个案例按 **需求澄清 → 估算 → 高层设计 → 深挖 → 失败模式 → 演进** 的完整流程走，末尾都有「面试官会追问」和自测题。

> **下表按面试重要性排序，不按文件名排序**（物理文件名是历史遗留，08–17 才是最高频的那批）。
> 通用面试：**先做 A 组的 10 篇**，B 组按你的岗位方向挑 2–3 篇，C 组留到面试前一天。

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

## 学习路径

### 主入口：三条完整路径（在 [START-HERE.md](START-HERE.md) 里逐日排好）

**不要自己拼路径。** [START-HERE.md](START-HERE.md) 里有一个 10 分钟自测，答完按分数进对应路径，
每一天读什么、练什么、练多久都排好了，还有周检查点和进度勾选表。

| 路径 | 适合谁 | 时长 | 04 章 |
|---|---|---|---|
| **[A · 通用 full-stack / 后端](START-HERE.md#3-路径-a四周逐日计划)**（默认） | 大多数人。面 senior 后端 / 全栈 / 平台岗 | 4 周（每天 60–90 min） | ✕ 整章跳过 |
| **[B · 10 天冲刺](START-HERE.md#4-路径-b10-天冲刺)** | 面试在 10 天内，或自测答对 10+ | 10 天（每天 90–120 min） | ✕ 整章跳过 |
| **[C · AI / 基础设施岗](START-HERE.md#5-路径-cai--基础设施岗的额外-2-周)** | 目标是 AI 产品 / 推理平台 / Agent 基础设施 | 6 周 = 路径 A + 额外 2 周 | ✓ 主体 |

配套的练习方法（录音、自评 rubric、缺口记录、30 天排期）在 **[PRACTICE.md](PRACTICE.md)** —— 和路径一起用，不是二选一。

### 补充：按主题的补路径

下面五条**不是完整备考计划**，是「我只缺这一块」时的定点补课清单。
已经在走上面三条路径的人不需要它们 —— 那三条里已经包含了这些内容并排好了顺序。

**主题 A：补齐分布式内功**（2–3 周）
`00 全部` → `01 全部` → `05-reliability/03` → `06/12 分布式缓存` → `06/10 信息流`

**主题 B：转向 AI 基础设施**（1–2 周，假设已有分布式基础）
`00-foundations/01` → `04 全部` → `06/01` → `06/02` → `06/03`

**主题 C：SaaS 平台 / 创业技术负责人**（2 周）
`02/03 多租户` → `03 全部` → `05/01` → `06/04`

**主题 D：面试冲刺**（5 天，时间比路径 B 还紧时的最小版）
`07/03 速查` → `07/01 框架` → `06/07 经典题速解（一天过完 10 题）` → `06 的 A 组挑 3 篇精读` → `07/06 逐字稿对照自己的节奏` → `07/02 自测`

**主题 E：英文面试冲刺**（3 天，中文技术已过关，卡在开口）
`07/04 术语表：只扫第 3 列「常见误译」` → `07/05 话术库：出声念，录音回放` → `07/06 逐字稿：先遮住候选人台词自己答，再对照` → `06/07 挑 3 题用英文各讲 6 分钟`
最后一天只做一件事：**把 07/05 的「被 challenge」和「30 秒收尾」两节练到不用想。**

---

## 怎么用这本手册

1. **不要通读**。带着一个具体问题来查，看完立刻在自己系统上找对应物。
2. **每篇末尾的「面试官会追问」**是自检清单，答不上来的地方就是你的缺口。
3. **数字要背**。`07/03` 里的那张表，是所有估算的地基。
4. **反例比正例值钱**。每篇都有「什么时候不要这么做」，那部分才是 Senior 和 Mid 的分水岭。
5. **术语表和话术库要出声用，不要读**。`07/04` 不是词典，别通读 —— 面试前一天只扫「常见误译」那一列，准备时卡住了再按主题跳回去查；`07/05` 的句式必须念出来并录音回放，听 hedge 词的数量和有没有升调。**默读能通过的检查，开口时一条都过不了。**
6. **读不等于会**。读的产出是知识，练的产出是表现，面试只买后者。方法在 [PRACTICE.md](PRACTICE.md)：一天一道题，出声讲，录音，对照清单打勾。**不录音就不算练过。**

---

## 三个文件的分工

| 文件 | 回答什么问题 | 什么时候打开 |
|---|---|---|
| **[START-HERE.md](START-HERE.md)** | 我今天该干什么？ | 第一次来，以及之后每一天 |
| **[PRACTICE.md](PRACTICE.md)** | 我该怎么练？ | 配好流程（10 分钟），之后每次练习前 |
| **README.md**（本文件） | 书里有什么？某个主题在哪一篇？ | 查东西的时候 |

---

## 贡献与许可

内容基于公开的工程实践与论文整理，欢迎 issue / PR 补充与纠错。
License: [MIT](LICENSE)
