# 08 · ML Systems / Inference Engineer —— 考点 → 章节映射

> 这一页不讲内容，它只回答一个问题：**JD 上那份 syllabus 的每一项，在这本书的哪一节，以及面试会怎么考它。**
> 九篇正文在下面的表里被逐项拆开引用；想按顺序读，直接从 [01-ml-system-overview.md](01-ml-system-overview.md) 开始。

**这一章是给谁的**：面 **ML Systems Engineer / Inference Engineer / ML Platform / 推荐排序基础设施** 的人。
判据不是关键词本身，而是职责是否包含模型产物、在线推理容量、特征数据、实验平台或模型发布。JD 只写一句“与 ML 团队合作”，但工作仍是普通 Web API 时，不必因此走完整专章。

**如果你是普通 Full Stack Developer**：默认不需要读完整 08；调用 LLM/Agent API 走 [04 的应用路径](../04-ai-agent-systems/README.md)。如果你负责的是分类、排序等预测接口，或要把团队训练出的模型接进生产，可先读 [01 · 总览](01-ml-system-overview.md)，再选 [04 · 在线推理](04-online-inference.md) 的 §1–§3、§9–§11 和 [09 · 部署](09-model-deployment.md) 的双层门禁。开始负责训练产物、GPU 容量、特征或实验平台后，再继续深读其余章节。

> ⚠️ **08 和 [04-ai-agent-systems](../04-ai-agent-systems/) 关注点不同，但边界会重叠：**
>
> | | [04 · AI / Agent 系统](../04-ai-agent-systems/) | **08 · ML Systems（本章）** |
> |---|---|---|
> | 对象 | 以 LLM/Agent 应用层为主，也涉及自建 LLM serving | 以模型生命周期、特征、在线推理和实验为主；模型既可能自训，也可能来自外部 |
> | 典型问题 | prompt 怎么组、KV cache 怎么省、Agent 循环怎么终止、提示注入怎么防 | 特征怎么对齐、批处理怎么组、金丝雀判据怎么写、漂移怎么检出 |
> | 延迟量级 | 生成式交互常见首 token 数百 ms、全程数秒；embedding/分类也可为毫秒级 | 在线排序常见毫秒级；批量、生成式模型和复杂视觉模型也可能是秒级 |
> | 质量怎么判 | 可用确定性断言、人工标注、业务反馈与校准后的 LLM-as-judge | 标签常延迟或缺失；离线指标只能筛选，最终结合在线实验、护栏和人工审查 |
> | 谁重叠 | 批处理、GPU 显存、冷启动、成本模型这四块两章都讲，各自的侧重不同 | 同左 |
>
> 同时服务 LLM 与传统模型的推理平台岗位通常两章都要读；其他岗位先按实际职责选主线，再补交叉部分 —— 04 走 [路径 C](../START-HERE.md#5-路径-cai-agent--llm-应用岗的额外-2-周)，08 走 [路径 F](../START-HERE.md#6-路径-fml-systems--inference-engineer6-周)。

---

## 1. 这一章的九篇

| 篇 | 内容 | 它在 syllabus 里覆盖哪一组 |
|---|---|---|
| [01 · ML 系统与普通系统的根本差别](01-ml-system-overview.md) | 行为写在数据里、Hidden Technical Debt、ML 特有失败模式、双指标 SLO | 全章的词典，不对应单项 |
| [02 · 模型生命周期](02-model-lifecycle.md) | Registry schema、版本四元组、Checkpoint 大小与频率、保留成本、产物分发 | 模型生命周期（前 3 项） |
| [03 · 加载、预热与冷启动](03-model-loading-and-warmup.md) | 冷启动九项分解、预热收敛判据、预热池容量公式、双缓冲切换、显存超售 | 模型生命周期（Model loading / Warmup） |
| [04 · 在线推理](04-online-inference.md) | 在线/近线/离线、同步/异步、streaming 的两个意思、动态批处理、准入控制、队列纪律 | **在线推理组（6 项全覆盖）** |
| [05 · 推理性能优化](05-inference-optimization.md) | 定位 180 ms 去哪了、Roofline、量化 ROI、算子融合、CPU/GPU 盈亏平衡、GPU 利用率的谎言 | 本书补充项（见 §4） |
| [06 · 特征平台与训练服务一致性](06-feature-and-data.md) | 偏斜五种成因、时点正确性、as-of join、在线取数 10 ms 预算、新鲜度分级 | 本书补充项（见 §4） |
| [07 · 模型质量：离线评测与在线实验](07-model-quality-and-experimentation.md) | 时序划分、离线-在线鸿沟六因、分流、样本量、CUPED、交错实验、影子部署、反馈采集 | **模型质量组（6 项全覆盖）** |
| [08 · 漂移检测与 ML 监控](08-drift-and-monitoring.md) | 静默降级、数据漂移 vs 概念漂移、PSI 的正确用法、无标签期监控什么、何时重训 | 模型质量（Drift detection）+ 生产运行 |
| [09 · 模型部署](09-model-deployment.md) | 服务金丝雀判据为何失效、双层门禁、模型回滚 ≠ 代码回滚、多模型路由、级联模型 | 模型生命周期（Canary / Rollback） |

配套的四道完整设计题：[06/18 模型服务平台](../06-case-studies/18-model-serving-platform.md) ｜ [06/19 特征平台](../06-case-studies/19-feature-store.md) ｜ [06/20 排序服务](../06-case-studies/20-ranking-service.md) ｜ [06/21 A/B 实验平台](../06-case-studies/21-ab-experiment-platform.md)

---

## 2. 通用 System Design 考点 → 章节（4 组 27 项）

**这一半不是背景知识。** 不同公司的题目占比会变，但队列、超时、分片、容量与发布决定模型系统在过载和故障时会怎样；准备时不能只看模型算法。

### 2.1 流量层（5 项）

| 考点 | 本书章节 | 面试会怎么考 |
|---|---|---|
| Load balancer | [01/04 §1](../01-building-blocks/04-networking-and-edge.md#1-负载均衡load-balancingl4-vs-l7) L4 vs L7 ｜ [08/09 §5.3](09-model-deployment.md#53-模型路由必须有一层-pointer) 模型 pointer 路由 | 「推理实例之间怎么均衡？」先量请求成本差异、队列、in-flight token/显存与批状态；成本差异大时，纯 round-robin 可能失效，可评估 least-outstanding-work 或模型感知路由 |
| API gateway | [02/04 §1、§6](../02-architecture-patterns/04-api-design-and-versioning.md#1-协议选型三分钟决策) ｜ [06/18 §3.2](../06-case-studies/18-model-serving-platform.md#32-目标架构300-个模型) | 「模型服务前面那一层做什么？」考的是**认证、配额、版本路由、请求改写**四件事各归谁 |
| Rate limiting | [06/09 限流器](../06-case-studies/09-rate-limiter.md) 全篇 ｜ [02/04 §6](../02-architecture-patterns/04-api-design-and-versioning.md#6-限流rate-limiting响应头与客户端契约) 响应头契约 | 高频独立题。ML 场景的变体：**限流单位不是请求数，是 GPU-second 或 token 数**（[06/18 §4.6](../06-case-studies/18-model-serving-platform.md#46-多租户隔离与成本归因gpu-second-是唯一诚实的计量单位)） |
| Authentication | [03/03 §2、§3](../03-saas-platform/03-identity-and-authz.md#2-oidc唯一你应该用的登录协议) OIDC / JWT 校验清单 ｜ [§9](../03-saas-platform/03-identity-and-authz.md#9-机器与机器api-key--oauth--mtls-怎么选) M2M | 至少能完整讲 JWT 验证、撤销/会话版本与服务间身份；若岗位涉及多租户或高风险模型，还要讲授权与配额边界 |
| Request routing | [01/04 §5](../01-building-blocks/04-networking-and-edge.md#5-服务发现service-discovery与流量治理traffic-management) 服务发现与流量治理 ｜ [08/09 §5.1–5.3](09-model-deployment.md#51-切分维度决定了你能测什么) | ML 变体是主场：「同一个模型有 v3 和 v4 两个版本，流量怎么切？」—— 按请求 / 按用户 / 按租户三种切法**决定了你能测什么** |

### 2.2 数据与状态（7 项）

| 考点 | 本书章节 | 面试会怎么考 |
|---|---|---|
| Cache | [01/02 §2、§3](../01-building-blocks/02-caching.md#2-缓存模式) 模式与三大故障 ｜ [06/12 分布式缓存](../06-case-studies/12-distributed-cache.md) | ML 变体：**embedding 该不该缓存**。先算热集合大小、更新频率、复用率和一致性预算；能装进进程内存只是一个候选方案，案例数值不能外推（[06/20 §4.5](../06-case-studies/20-ranking-service.md#45-在线特征取数把-10001-的读放大消灭掉)） |
| Database | [01/01 §1、§2、§8](../01-building-blocks/01-storage-engines.md#1-rum-猜想所有存储的根本约束) RUM / B+Tree vs LSM / 选型总表 | 「在线特征存什么？」先写点查/批读、QPS、新鲜度、耐久、更新和团队能力，再比较 KV、关系库与缓存；不能由“高 QPS”直接推出唯一产品 |
| Object storage | [01/01 §7](../01-building-blocks/01-storage-engines.md#7-对象存储被低估的数据库) ｜ [06/17](../06-case-studies/17-object-storage.md) ｜ [08/02 §7](02-model-lifecycle.md#7-产物的存储与分发不要把权重烤进镜像) | 常见设计是把权威模型产物放对象存储，再用本地缓存、分层分发与预热避免每个请求或所有 Pod 同时冷拉；容量与带宽用目标产物大小重算 |
| Queue | [01/03 §1](../01-building-blocks/03-messaging-and-streams.md#1-队列-vs-日志两种根本不同的东西) 队列 vs 日志 ｜ [08/04 §10](04-online-inference.md#10-队列管理深度纪律丢弃) 队列管理 | 见下面「在线推理」组的 Queue management —— 这一项在 ML 面试里是**深挖区**，不是背景知识 |
| Replication | [01/06 §1–§4](../01-building-blocks/06-replication.md#1-复制在解决四件不同的事而它们要的方案不一样) ｜ **[01/06 §10](../01-building-blocks/06-replication.md#10-专项选读ml-场景的复制) ML 场景的复制** | 特征读取与模型产物常因新鲜度、耐久和回滚需求采用不同策略，但不是必须相反；分别写 RPO、读一致性和发布原子性，再选复制方式 |
| Sharding | [05/05 §5、§6](../05-reliability/05-scaling-playbook.md#5-分片深度剖析) 分片与在线迁移 ｜ [00/00 §5](../00-foundations/00-concepts.md#5-副本分片分区--三个被混用的词) | 「特征表怎么分片？」分片键选 entity_id，然后被追问**热点用户**怎么办 —— 这题和普通分片题同构 |
| Consistency | [00/01 §4](../00-foundations/01-fundamentals.md#4-两条正交轴副本一致性与事务隔离) 一致性全谱 ｜ [01/06 §4](../01-building-blocks/06-replication.md#4-会话保证三件套读己之写单调读一致前缀读) 会话保证 | ML 变体：「用户刚点了一下，多久能影响推荐？」答案是**分档**而不是一个数（[08/06 §6](06-feature-and-data.md#6-新鲜度分级只给真正影响决策的特征付实时的钱) 新鲜度分级） |

### 2.3 分布式系统（8 项）

| 考点 | 本书章节 | 面试会怎么考 |
|---|---|---|
| Leader election | [01/05 §2](../01-building-blocks/05-consensus-and-coordination.md#2-raft够用的心智模型) Raft 心智模型 ｜ [§8](../01-building-blocks/05-consensus-and-coordination.md#8-脑裂split-brain与防护) 脑裂 | 一般只问到"谁来决定哪个模型版本是 primary"。**不要主动展开 Raft 细节**，那不是这个岗位的加分区 |
| Service discovery | [01/04 §5](../01-building-blocks/04-networking-and-edge.md#5-服务发现service-discovery与流量治理traffic-management) | 「新起的推理实例什么时候开始收流量？」—— 钩子是**就绪探针必须包含预热**（[08/03 §3](03-model-loading-and-warmup.md#3-预热跑什么跑多少次怎么算跑完了)），这是本项的 ML 落点 |
| Retry | [05/03 §3](../05-reliability/03-resilience-patterns.md#3-重试先算放大倍数再决定要不要重试) 先算放大倍数 ｜ [00/01 §9](../00-foundations/01-fundamentals.md#9-幂等--重试--超时三件套必须一起设计) 三件套 | 过载与 deadline 不足时不重试；对幂等、瞬时失败且仍有预算的请求，可做一次有退避/重试预算的重试。对冲只适合有冗余容量且能取消 loser 的尾延迟场景 |
| Timeout | [05/03 §2](../05-reliability/03-resilience-patterns.md#2-超时预算timeout-budget与-deadline-传播) 超时预算与 deadline 传播 ｜ [08/04 §11](04-online-inference.md#11-延迟预算分解一个具体算例) 延迟预算分解 | 「你的 150 ms 预算怎么分给召回/粗排/精排？」这是排序题的骨架（[06/20 §2.2](../06-case-studies/20-ranking-service.md#22-延迟预算--这题的骨架)），**每层只减不加** |
| Idempotency | [00/01 §5](../00-foundations/01-fundamentals.md#5-幂等idempotency分布式系统的第一公民) 幂等键 ｜ [01/05 §6](../01-building-blocks/05-consensus-and-coordination.md#6-幂等与去重协调视角) ｜ [02/04 §5](../02-architecture-patterns/04-api-design-and-versioning.md#5-幂等契约idempotency-contract) | 批量推理任务提交、特征物化作业重跑 —— 都要说出**去重键怎么选** |
| Circuit breaker | [05/03 §4](../05-reliability/03-resilience-patterns.md#4-熔断器circuit-breaker阈值粒度半开half-open) 阈值 / 粒度 / 半开 | 「特征存储挂了，排序服务怎么办？」考的是熔断**之后降级到什么**，不是熔断本身 |
| Backpressure | [00/01 §6](../00-foundations/01-fundamentals.md#6-背压backpressure没有它系统就会雪崩cascading-failure) ｜ [05/03 §6](../05-reliability/03-resilience-patterns.md#6-负载卸载load-shedding与准入控制) 负载卸载 ｜ [08/04 §9](04-online-inference.md#9-准入控制过载时该拒绝谁) | 这一项和下面的 Admission control 是**同一个考点的两个名字**，一起准备 |
| Fault tolerance | [05/03 §1、§9、§10](../05-reliability/03-resilience-patterns.md#1-第一原理你控制不了故障只能控制传播) ｜ [05/01 §4](../05-reliability/01-slo-and-error-budget.md#4-依赖链的可用性乘法) 依赖乘法 ｜ [08/01 §4](01-ml-system-overview.md#4-ml-系统特有的失败模式清单) | ML 特有的那条：**ML 系统坏掉时返回 200 OK**。普通的 fault tolerance 对它完全失明 —— 这句话本身就是加分点 |

### 2.4 生产运行（7 项）

| 考点 | 本书章节 | 面试会怎么考 |
|---|---|---|
| Metrics | [05/02 §1、§2](../05-reliability/02-observability.md#1-四种遥测的分工) 四种遥测 / 基数 ｜ [08/04 §10](04-online-inference.md#10-队列管理深度纪律丢弃) 三个必上指标 | 「扩缩容挂什么指标？」**不要答 GPU 利用率** —— `nvidia-smi` 的利用率会骗人（[08/05 §9](05-inference-optimization.md#9-gpu-利用率nvidia-smi-为什么会骗你)），要挂队列等待时间 |
| Logging | [05/02 §3](../05-reliability/02-observability.md#3-日志最容易失控的一支) ｜ [08/07 §10](07-model-quality-and-experimentation.md#10-反馈采集曝光日志的完整性与随机流量的必要性) 曝光日志 ｜ [08/06 §7](06-feature-and-data.md#7-版本化监控下线) 特征日志 | ML 变体是本岗位主场：**特征日志的采样策略是架构决策，不是运维参数**（[06/19 §2.4](../06-case-studies/19-feature-store.md#24-特征日志采样策略是架构决策不是运维参数)） |
| Tracing | [05/02 §4](../05-reliability/02-observability.md#4-追踪采样是唯一真正的设计决策) 传播、属性与采样 ｜ [08/04 §11](04-online-inference.md#11-延迟预算分解一个具体算例) | 「一条推理请求，时间去哪了？」要保证跨异步/批处理阶段的上下文关联、控制属性基数，并用采样预算保留慢与错请求；[08/05 §1](05-inference-optimization.md#1-先定位这-180-ms-到底在哪) 给出分解方法 |
| Alerting | [05/02 §8](../05-reliability/02-observability.md#8-告警的设计) ｜ [05/01 §6](../05-reliability/01-slo-and-error-budget.md#6-多窗口多燃尽率告警multi-window-multi-burn-rate-alerting) 多窗口多燃尽率 ｜ **[08/08 §6](08-drift-and-monitoring.md#6-告警怎么定漂移告警天生高误报)** | ML 变体是必答点：**漂移告警天生高误报**，直接照搬 SLO 告警那套会把值班炸掉 |
| Deployment | [03/05 §1、§3](../03-saas-platform/05-release-engineering.md#1-部署--发布) 部署≠发布 / 渐进式交付 ｜ **[08/09 §1–§3](09-model-deployment.md#1-为什么服务的金丝雀判据搬到模型上会失效)** | 见下面「模型生命周期」组的 Canary release |
| Rollback | [03/05 §3](../03-saas-platform/05-release-engineering.md#3-渐进式交付progressive-delivery) ｜ **[08/09 §4](09-model-deployment.md#4-模型回滚--代码回滚) 模型回滚 ≠ 代码回滚** | 高频。答"回滚权重"只拿一半分，要说出**版本是四元组**（权重 + 特征管道 + 依赖锁 + 配置） |
| Capacity planning | [00/02 §1–§3](../00-foundations/02-capacity-estimation.md#1-估算的黄金流程) 排队论 ｜ [05/05](../05-reliability/05-scaling-playbook.md) ｜ [08/03 §5](03-model-loading-and-warmup.md#5-预热池要多大公式与算例) 预热池公式 ｜ [06/18 §2](../06-case-studies/18-model-serving-platform.md#2-估算) | 「300 个模型要几张卡？」考的是**负载单位不是请求数**（[06/18 §2.2](../06-case-studies/18-model-serving-platform.md#22-负载单位请求数不是负载)），以及那个没人愿意说的真实利用率 |

---

## 3. ML System Design 考点 → 章节（3 组 19 项）

### 3.1 模型生命周期（7 项）

| 考点 | 本书章节 | 面试会怎么考 |
|---|---|---|
| Model registry | [08/02 §2](02-model-lifecycle.md#2-registry-该记什么一张-schema-表重点在最后一列) Registry schema 表 ｜ [06/18 §4.1](../06-case-studies/18-model-serving-platform.md#41-模型注册与产物分发版本的单位不是权重) | 「Registry 里存什么？」不要列 20 个字段，要说出**最少的那几个 + 为什么少一个就回滚不了** |
| Checkpoint storage | [08/02 §4](02-model-lifecycle.md#4-checkpoint先算大小再谈频率) 先算大小再谈频率 ｜ [§5](02-model-lifecycle.md#5-保留策略与成本一个必须自己算一遍的算例) 保留策略与成本算例 | 「checkpoint 多久存一次？」这是**成本题不是可靠性题** —— 先算单个多大、一天写多少、留多久 |
| Versioning | [08/02 §3](02-model-lifecycle.md#3-版本化与血缘semver-为什么不适用) 为什么 SemVer 不适用 ｜ [§8](02-model-lifecycle.md#8-不可变性与提升dev--staging--prod) dev→staging→prod 提升 | 必答的一句：**模型版本是不可分割四元组**。SemVer 表达不了"权重没变但特征管道变了" |
| Model loading | [08/03 §1、§2](03-model-loading-and-warmup.md#1-为什么第一次推理和第一百次不是一回事) 冷启动九项分解 ｜ [§4](03-model-loading-and-warmup.md#4-缩短冷启动的手段收益代价撞墙条件) 缩短手段与撞墙条件 | 「冷启动 4 分钟，怎么优化？」考的是**顺序**：镜像 > 权重 I/O 并行 > 编译缓存。换更快的 GPU 几乎救不了（CPU-bound） |
| Warmup | [08/03 §3](03-model-loading-and-warmup.md#3-预热跑什么跑多少次怎么算跑完了) 跑什么形状 / 收敛判据 / 就绪探针 | 高频且答不好。**不要定次数，定收敛判据**；预热必须是就绪探针的一部分，否则实例一起来就收满流量 |
| Canary release | [08/09 §1–§3](09-model-deployment.md#1-为什么服务的金丝雀判据搬到模型上会失效) 服务判据为何失效 / 双层门禁 ｜ [06/18 §4.5](../06-case-studies/18-model-serving-platform.md#45-多版本金丝雀与影子模型的发布和服务的发布不是一回事) | **本组分差最大的一项**：服务的"5% 起步、10 分钟看错误率"搬到模型上直接失效 —— 模型的质量信号要几天才攒够样本 |
| Rollback | [08/09 §4、§4.1–4.3](09-model-deployment.md#4-模型回滚--代码回滚) ｜ [08/02 §5](02-model-lifecycle.md#5-保留策略与成本一个必须自己算一遍的算例) 旧版本留多久 | 追问链是固定的：回滚什么 → 回滚窗口里的数据怎么办 → 旧模型保留多久 → 这笔存储多少钱 |

### 3.2 在线推理（6 项）

| 考点 | 本书章节 | 面试会怎么考 |
|---|---|---|
| Online vs batch | [08/04 §1](04-online-inference.md#1-在线近线还是离线先回答这个) 在线 / 近线 / 离线 | 开局先按新鲜度、候选组合与请求依赖拆出可预计算部分；节省比例要用本系统调用占比与账单验证，不能先写“砍半” |
| Sync vs async | [08/04 §2](04-online-inference.md#2-同步异步回调异步轮询) 同步 / 异步回调 / 异步轮询 | 「一次推理 30 秒的模型接口怎么设计？」→ 202 + 状态资源（契约在 [02/04 §8](../02-architecture-patterns/04-api-design-and-versioning.md#8-长任务long-running-operation202--状态资源--通知)） |
| Streaming | [08/04 §3](04-online-inference.md#3-streaming-的两个意思别混) **streaming 的两个意思别混** | 陷阱题。面试官说 streaming 时可能指"逐 token 流式返回"，也可能指"流式特征管道"，**先问清楚**再答 |
| Dynamic batching | [08/04 §4–§8](04-online-inference.md#4-动态批处理原理与两个旋钮) 两个旋钮 / 延迟-吞吐曲线 / 显存天花板 / 变长分桶 / 与 continuous batching 的本质区别 | **本章最长的一节，也是这个岗位的核心区。** 必须能说出两个旋钮（max_batch_size、max_queue_delay）各自往哪边推曲线 |
| Admission control | [08/04 §9](04-online-inference.md#9-准入控制过载时该拒绝谁) 入队前做容量/配额准入，出队前再按剩余 deadline 淘汰过期请求 | 「QPS 突然翻三倍怎么办？」答"扩容"不够（扩容 90 s–8 min，尖峰 30 s）。要答准入控制 + 预热池 |
| Queue management | [08/04 §10](04-online-inference.md#10-队列管理深度纪律丢弃) 深度用 Little's Law 反推 / FIFO vs LIFO / 三个监控指标 | **过载时 FIFO 和 LIFO 的结论会反转** —— 说得出这一条基本就锁定了这一项的分 |

### 3.3 模型质量（6 项）

| 考点 | 本书章节 | 面试会怎么考 |
|---|---|---|
| Offline eval | [08/07 §1](07-model-quality-and-experimentation.md#1-离线评测划分错了后面全白做) 时序划分 / 指标只是筛子 | 当部署目标包含未来时间段或数据会随时间变化时，用时间切分并做 point-in-time 特征，避免未来信息泄漏；真正 IID 的任务可以采用其他切分，但要证明假设（[08/06 §3.3](06-feature-and-data.md#33-为什么这个错误在离线指标上表现为指标变好本章最重要的一段)） |
| Online eval | [08/07 §2](07-model-quality-and-experimentation.md#2-离线-在线鸿沟六类成因逐条给应对) 离线-在线鸿沟六类成因 ｜ [§7](07-model-quality-and-experimentation.md#7-护栏指标与效应的时间形状) 护栏与效应的时间形状 | 「离线指标上涨，上线没效果，为什么？」先验证实验分流/样本与指标管道，再查特征、代码、流量和反馈差异；SRM 是随机实验分配异常的检查，不是所有离线差异的通用第 0 步 |
| A/B testing | [08/07 §3–§6](07-model-quality-and-experimentation.md#3-分流为什么排序实验必须按用户分) 分流 / 样本量 / CUPED / 多重检验 ｜ [06/21](../06-case-studies/21-ab-experiment-platform.md) 全篇 | **排序实验必须按用户分**（§3）；MDE 缩 5 倍样本量涨 25 倍（平方关系），这个数直接决定实验周期是 2–3 周而不是几天 |
| Drift detection | [08/08 §2、§3](08-drift-and-monitoring.md#2-数据漂移-vs-概念漂移这条界必须画清楚) 数据漂移 vs 概念漂移 / PSI 的正确用法 ｜ [§4](08-drift-and-monitoring.md#4-没有标签的那段时间里你监控什么) 无标签期 | PSI 桶边界来自已版本化 reference，不能随当前数据自动漂移；可同时保留固定训练/发布 reference 捕捉累计偏移，以及季节性/近期基线捕捉短期异常 |
| Shadow deployment | [08/07 §9](07-model-quality-and-experimentation.md#9-影子部署能证明什么不能证明什么) 能证明什么、不能证明什么 ｜ [08/09 §3](09-model-deployment.md#3-双层门禁技术护栏立即判--质量指标延迟判) | 陷阱题：「影子能不能替代金丝雀？」**不能** —— 预测不返回给用户就没有反馈闭环，CTR / 转化 / 留存一律测不到 |
| Feedback collection | [08/07 §10](07-model-quality-and-experimentation.md#10-反馈采集曝光日志的完整性与随机流量的必要性) 曝光日志完整性与随机流量 ｜ [08/08 §9](08-drift-and-monitoring.md#9-反馈回路你的训练数据是模型自己写的) 反馈回路 | **训练数据是模型自己写的** —— 说得出这条退化回路，以及随机曝光流量要付多少 CTR，是这一项的分水岭 |

---

## 4. 本书额外补充、且在生产型岗位常被追问的

上面的 syllabus 漏了三块。它们不在任何一份公开考纲里，但**每一场 ML Systems 面试都会撞上**：

| 补充项 | 在哪 | 为什么值得准备 |
|---|---|---|
| **Feature store 与训练/服务偏斜** | [08/06](06-feature-and-data.md) 全篇（§2 五种成因、§3 时点正确性与 as-of join、§5 在线取数 10 ms 预算）｜[06/19](../06-case-studies/19-feature-store.md) | syllabus 把它归进"数据"一笔带过，但它是**ML 系统 bug 的最大来源**。而且它有一个别的考点都没有的性质：**错了会让离线指标变好**，所以监控和评测同时失效。追问链固定为「怎么发现 → 怎么定位 → 第 0 步为什么是 SRM 检查」 |
| **Roofline 与 GPU 利用率** | [08/05 §2](05-inference-optimization.md#2-roofline判断你撞的是哪堵墙) Roofline ｜ [§8](05-inference-optimization.md#8-cpu-vs-gpu-的盈亏平衡公式与算例) CPU/GPU 盈亏平衡 ｜ **[§9](05-inference-optimization.md#9-gpu-利用率nvidia-smi-为什么会骗你) `nvidia-smi` 为什么会骗你** | 「你怎么知道该优化什么？」没有 Roofline 就只能靠试。而 §9 那条 —— **利用率 90% 可能只是"有 kernel 在跑"，不是"算力用满了"** —— 是区分"用过 GPU"和"调过 GPU"的分水岭。优化手段的 ROI 总表在 §3 |
| **多阶段漏斗的延迟预算** | [06/20 §2.2、§4.1、§4.2](../06-case-studies/20-ranking-service.md#22-延迟预算--这题的骨架) ｜ [08/04 §11](04-online-inference.md#11-延迟预算分解一个具体算例) | 排序岗的**骨架题**。召回/粗排/精排的候选数与单条算力互为倒数，预算怎么切决定了整个架构。而且它有一个反直觉的必答点：**超时即放弃，不是超时即重试**，以及超时了到底返回什么（[06/20 §4.7](../06-case-studies/20-ranking-service.md#47-降级超时了到底返回什么)） |

---

## 5. 怎么读这一章

| 你的情况 | 怎么走 |
|---|---|
| 有 6 周，系统准备 | **[START-HERE 路径 F](../START-HERE.md#6-路径-fml-systems--inference-engineer6-周)** —— Week 1–2 通用地基、Week 3–4 本章九篇、Week 5 案例、Week 6 面试装置，逐日排好 |
| 通用系统设计已经很熟，只补 ML | 本章九篇按 01 → 09 顺序读（01 是词典，别跳），然后做 [06/18](../06-case-studies/18-model-serving-platform.md) 和 [06/20](../06-case-studies/20-ranking-service.md) 两道计时题 |
| 一周内面试 | 只读四节：[08/04 §4、§9、§10](04-online-inference.md#4-动态批处理原理与两个旋钮)（批处理与准入）、[08/09 §1–§4](09-model-deployment.md#1-为什么服务的金丝雀判据搬到模型上会失效)（金丝雀与回滚）、[08/06 §2、§3](06-feature-and-data.md#2-训练服务偏斜五种成因各自的检测手段)（偏斜）、[08/08 §2、§3](08-drift-and-monitoring.md#2-数据漂移-vs-概念漂移这条界必须画清楚)（漂移）。**它覆盖 ML 组 19 项里的 7 项** —— 在线推理 3（Dynamic batching / Admission control / Queue management）+ 模型生命周期 2（Canary / Rollback）+ 模型质量 2（Shadow deployment / Drift detection）—— 外加 §4 的两个补充考点（偏斜、延迟预算）。**这 7 项是分差最大的 7 项，不是覆盖面最广的读法**：要覆盖面就得读满九篇 |
| 卡在某个词 | 回 [00-foundations/README.md 知识地图](../00-foundations/README.md) §4 查它首次定义在哪一节 |

**覆盖率**：通用 4 组 27 项 **全部**映射到具体小节；ML 3 组 19 项 **全部**映射到具体小节；
另有 3 项本书补充考点（§4）。合计 **46 + 3 = 49 项**。

---

**开始读** → [01-ml-system-overview.md](01-ml-system-overview.md)：先把"ML 系统坏掉时会返回 200 OK"这件事讲清楚，
后面八篇的每一个设计决策都是从这一句长出来的。
