# 02 · 题库：54 道题与"面试官在等的那句话"

> 每道题的价值不在题面，在**那一句最能证明你做过真东西的断言**。
> 背题面没用；能不能在 40 分钟里自然地说出那句话，才是区分度。

---

## 怎么用这份题库

**每道题四行**：题面 → 关键考点 → 面试官在等的那句话 → 本手册对应章节。

- `〔L5〕` = Senior 档常见题；`〔L6〕` = Staff 档常见题（多了组织约束、迁移、平台化）。
- **"等这句"不是标准答案**，是这道题的**深度锚点**：把它说出来，面试官立刻知道你的理解不是从博客上抄的。
- 自测方法：遮住"等这句"，自己讲 5 分钟，然后对比。**答不出来的地方就是你的缺口** —— 去读"相关"里的章节。
- 别按顺序刷。**从你最不熟的那一组开始**，那一组才有信息量。

---

## 1. 经典分布式（10 题）

**Q01 · 短链接服务** 〔L5〕— 设计一个日均 1 亿次跳转、峰均比（peak-to-average ratio）暂按 3× 估算的短链服务。
- **考点**：短码生成（哈希 vs 号段）｜读写比 100:1 的缓存分层（cache tiering）｜自定义短码的唯一性｜过期回收｜防扫描枚举
- **等这句**：「短码不用哈希取模生成 —— 哈希会撞，撞了要重试，重试在高并发下是**不可控的尾延迟（tail latency）**。用预分配号段（每个实例一次领 10 万个）+ Base62，无锁、无冲突、可离线生成。代价是号段丢失会留空洞，我接受，62⁷ 的空间浪费一点无所谓。」
- **相关**：[01-storage-engines](../01-building-blocks/01-storage-engines.md)、[02-caching](../01-building-blocks/02-caching.md)

**Q02 · 分布式限流器** 〔L5〕— 给一个 200 实例的 API 网关做租户级 QPS 限流。
- **考点**：固定窗口/滑动窗口/令牌桶（token bucket）/漏桶（leaky bucket）的差异｜本地 vs 集中（Redis）的一致性-延迟取舍｜时钟漂移｜限流器自身的可用性
- **等这句**：「每个请求打一次 Redis 会把限流器放进关键路径。我先按业务后果选边界：普通读请求可用**本地令牌桶 + 周期性配额再分配**，中心每秒按用量发租约，失联后只花完已租配额再退到有硬上限的静态配额；登录、支付、管理写操作则 fail-closed 或进入更严格的本地上限。这里不能把 fail-open 写成全系统默认，安全和可用性的取舍要按端点分级。」
- **相关**：[03-resilience-patterns](../05-reliability/03-resilience-patterns.md)、[02-billing-and-metering](../03-saas-platform/02-billing-and-metering.md)

**Q03 · 全局唯一 ID 生成器** 〔L5〕— 每秒 100 万个、单调递增、可排序的 ID。
- **考点**：Snowflake 的位分配与时钟回拨｜UUIDv7 / ULID｜机器 ID 分配｜单调性 vs 可猜测性｜作为主键对 B-Tree 的影响
- **等这句**：「Snowflake 的主要正确性风险有时钟回拨、worker ID 重复和同毫秒序号耗尽；回拨时要阻塞、切换到受控的新 workerId，或使用逻辑时间，不能带着旧状态继续发号。如果不需要跨机器严格单调，我会优先评估 UUIDv7：它大体按时间聚集、通常比随机 UUID 更友好，但同一毫秒内仍不保证严格顺序，也仍要接受更宽的主键。」
- **相关**：[01-storage-engines](../01-building-blocks/01-storage-engines.md)、[05-consensus-and-coordination](../01-building-blocks/05-consensus-and-coordination.md)

**Q04 · 分布式锁服务** 〔L6〕— 给一个跨 50 个服务的批处理系统提供互斥。
- **考点**：Redlock 的争议｜租约（lease）与 fencing token｜GC 暂停导致的锁失效｜锁粒度｜"锁"其实常常是错的抽象
- **等这句**：「**光有锁不够，必须有 fencing token。** 持锁进程可能被 GC 暂停 30 秒，锁过期后它醒来仍然认为自己持锁并写入。正确做法是锁服务返回单调递增的 token，存储层拒绝比已见过的 token 更小的写。做不到这一点的话，我会把问题改造成**幂等 + 乐观并发（optimistic concurrency）**，根本不用锁。」
- **相关**：[05-consensus-and-coordination](../01-building-blocks/05-consensus-and-coordination.md)

**Q05 · 新闻 Feed / 时间线** 〔L5〕— 5 亿用户，含 1 亿粉丝的大 V。
- **考点**：推 vs 拉（push vs pull）的扇出（fan-out）取舍｜大 V 特例｜时间线的存储与裁剪｜排序与去重｜冷启动
- **等这句**：「纯推在大 V 发帖时是 1 亿次写，纯拉在普通用户读时要归并几百个源。常见起点是混合：普通作者推，大 V 拉，读时归并；但 10 万粉丝只是案例参数。真正阈值来自作者发帖频率、活跃粉丝比例、收件箱写成本和读时归并成本的交点，用生产分布回放后再定。」
- **相关**：[03-tradeoff-framework](../00-foundations/03-tradeoff-framework.md)、[03-messaging-and-streams](../01-building-blocks/03-messaging-and-streams.md)

**Q06 · 网页爬虫** 〔L5〕— 每天抓 10 亿页，遵守 robots，不重复。
- **考点**：URL 前沿队列（frontier queue）的优先级与礼貌性（per-host 速率）｜去重（布隆过滤器的误判代价）｜内容去重（SimHash）｜陷阱与死循环｜断点续爬
- **等这句**：「礼貌性约束决定了队列结构：**必须按 host 分桶**，每个 host 一个独立的令牌桶，而不是全局一个大队列。全局队列会让一个热门站点的几十万 URL 挤在一起，要么打死对方，要么被迫串行导致整体吞吐塌掉。」
- **相关**：[03-messaging-and-streams](../01-building-blocks/03-messaging-and-streams.md)

**Q07 · 附近的人 / 地理检索** 〔L5〕— 千万级在线，查询半径 5 公里内的对象。
- **考点**：GeoHash / S2 / H3 的取舍｜边界问题（相邻格子）｜热点区域（市中心）｜位置更新的写放大（write amplification）｜精确距离二次过滤
- **等这句**：「GeoHash 的边界会把物理相邻点放进不同前缀，所以不能只查中心 cell。小半径、固定精度时常从中心格加相邻格起步；半径跨越更多格、靠近极区或 cell 尺寸不合适时，需要覆盖查询圆的可变 cell 集合，再做精确距离过滤。真正的工程重点还有热点：高密区域要换更细层级、二次分桶或按实体 hash 拆写，不能假设永远正好查 9 格。」
- **相关**：[01-storage-engines](../01-building-blocks/01-storage-engines.md)

**Q08 · 转账与对账** 〔L6〕— 跨两个账户系统的资金转移，不允许丢也不允许重。
- **考点**：exactly-once 的真相｜Saga vs 2PC｜幂等键（idempotency key）从业务语义派生｜对账层（reconciliation layer）｜补偿的可观测
- **等这句**：「先画 exactly-once 的边界：流处理系统可以在自己的事务范围内做到 exactly-once processing，但转账跨账户系统、消息和外部副作用后，常见落地仍是 **at-least-once + 幂等状态转换 + 对账 = effectively-once**。幂等键应代表同一次业务操作（如稳定的 `transfer_request_id`）；随机 UUID 本身没错，错的是每次重试都生成一个新的。另外必须有独立对账层，因为补偿本身也会失败。」
- **相关**：[02-event-driven-and-cqrs](../02-architecture-patterns/02-event-driven-and-cqrs.md)、[04-usage-based-billing](../06-case-studies/04-usage-based-billing.md)

**Q09 · 分布式 KV 存储** 〔L6〕— 自己实现一个类 Dynamo 的 KV。
- **考点**：一致性哈希（consistent hashing）与虚拟节点（virtual node）｜quorum（R+W>N）的真实语义｜向量时钟 vs LWW｜反熵与 Merkle 树｜hinted handoff
- **等这句**：「**R+W>N 并不给你线性一致（linearizability）**，它只保证读集合与写集合有交集 —— 但没有'读修复前的顺序'保证，也不防并发写的丢失更新。想要线性一致必须上共识（Raft/Paxos）。所以第一个问题永远是：这个数据到底需要哪级一致性？大多数场景 quorum + LWW 就够，用共识是多付一个 RTT 和一整套 leader 选举的运维负担。」
- **相关**：[05-consensus-and-coordination](../01-building-blocks/05-consensus-and-coordination.md)、[01-fundamentals](../00-foundations/01-fundamentals.md)

**Q10 · 配置中心与服务发现** 〔L5〕— 5000 个实例订阅配置变更，秒级生效。
- **考点**：长轮询 vs watch 流｜惊群（5000 个实例同时重连）｜配置的灰度与回滚｜客户端本地快照兜底｜配置变更也是发布
- **等这句**：「配置中心挂掉时，客户端应继续使用最后一份已验证快照，并明确快照过旧后的策略；首次启动没有快照时则 fail-closed 或用内置安全默认值。配置变更和代码一样能触发全局故障，所以也要 schema 校验、审查、分批放量、自动回滚与版本审计。事故占比因组织而异，不需要虚构一个‘配置一定比代码更常出错’的统计。」
- **相关**：[05-consensus-and-coordination](../01-building-blocks/05-consensus-and-coordination.md)、[05-release-engineering](../03-saas-platform/05-release-engineering.md)

---

## 2. 数据密集（7 题）

**Q11 · 指标监控系统** 〔L6〕— 1000 万条时间线，1 秒采样，保留 13 个月。
- **考点**：时序压缩（Gorilla delta-of-delta + XOR）｜高基数（high cardinality）爆炸｜降采样与 rollup｜查询的时间范围下推｜写入是纯 append
- **等这句**：「容量同时由样本量、活跃时间线基数、标签索引和查询范围决定；高基数往往最容易被漏算。`user_id` 这类无界标签要在写入侧做预算、白名单或重写，超限时按策略拒绝/降级并告警。Gorilla 类压缩对平滑时间戳和数值很有效，但每点字节数取决于分布、标签和块元数据，要用真实样本测，不能把 1.4 B 当承诺。」
- **相关**：[02-observability](../05-reliability/02-observability.md)、[01-storage-engines](../01-building-blocks/01-storage-engines.md)

**Q12 · 日志检索平台** 〔L5〕— 每天 50 TB 日志，支持关键词与结构化过滤。
- **考点**：倒排索引（inverted index） vs 列存 vs 无索引扫描｜冷热分层（hot/cold tiering）与对象存储｜采样与降级｜查询的资源隔离｜成本才是主约束
- **等这句**：「50 TB/天不能无差别给每个字段建倒排索引。先按查询工作量分层：常用过滤字段建索引或排序键，原文压缩放廉价存储，冷数据按需并行扫描；安全调查中需要任意关键词检索的热窗口则值得付索引成本。判据是字段选择性、查询频率、扫描字节和等待 SLO，不是‘每天不到一次就一定不建’。」
- **相关**：[02-observability](../05-reliability/02-observability.md)、[05-data-platform](../02-architecture-patterns/05-data-platform.md)

**Q13 · 数据仓库摄取管道** 〔L6〕— 从 200 个业务库同步到 Lakehouse。
- **考点**：CDC vs 批量快照｜schema 演进与数据契约｜迟到数据与水位线（watermark）｜幂等写入与 exactly-once 语义｜回填（backfill）不能压垮源库
- **等这句**：「200 个源库由许多团队独立演进，靠人工通知很难可靠协调。为关键数据集定义 owner、兼容规则和数据契约，在 CI / 发布流程中拦截破坏性变更；同时保留未知字段和版本并存的迁移期。回填要有源库限速、可暂停游标与独立资源预算；能否走只读副本取决于复制延迟和 CDC 位点要求，不能机械把所有回填都扔给副本。」
- **相关**：[05-data-platform](../02-architecture-patterns/05-data-platform.md)、[03-messaging-and-streams](../01-building-blocks/03-messaging-and-streams.md)

**Q14 · CDC 到搜索索引** 〔L5〕— 数据库变更实时同步到搜索引擎，允许秒级延迟。
- **考点**：Outbox 模式 vs 直读 binlog｜顺序保证与分区键（partition key）｜删除与 tombstone｜全量重建的能力｜双写（dual write）为什么是错的
- **等这句**：「请求线程直接写 DB 再写搜索引擎没有共同事务，失败会留下无法立即判断的差异；不要把它当正确性边界。保留 DB 作为权威源，用同事务 outbox 或 binlog CDC 产生带版本的索引变更，消费者幂等应用；再用对账与全量重建修复管道 bug。迁移期若要新旧索引双投，也从同一份日志投递，不做两次独立业务写。」
- **相关**：[03-messaging-and-streams](../01-building-blocks/03-messaging-and-streams.md)、[02-event-driven-and-cqrs](../02-architecture-patterns/02-event-driven-and-cqrs.md)

**Q15 · A/B 实验平台** 〔L6〕— 同时跑 500 个实验，支持分层与互斥。
- **考点**：分流哈希（bucketing hash）与稳定性｜分层设计（正交层）｜曝光日志与指标计算｜实验间干扰｜统计功效与最小样本量
- **等这句**：「先按可能相互干扰的产品面划实验层：需要互斥的实验共享层，能同时运行的实验用独立 salt 做稳定分流；‘跨层正交’是设计目标，还要用样本相关性和 A/A 检验验证。真正的工程难点是曝光口径：记录用户真正触发了变体，而不是只记录被分桶，否则效应会被稀释。」
- **相关**：[05-release-engineering](../03-saas-platform/05-release-engineering.md)、[05-data-platform](../02-architecture-patterns/05-data-platform.md)

**Q16 · 推荐系统的召回与排序** 〔L6〕— 亿级物料，100ms 内返回 20 条。
- **考点**：多路召回与融合｜粗排/精排的漏斗与算力分配｜特征时效性与在线-离线一致性｜冷启动｜多样性与去重
- **等这句**：「先把 100ms 拆给召回、特征读取、粗排、精排和网络，再用基准决定每层保留多少候选；1000→100→20 只是可压测的起点。在线—离线特征偏斜是关键风险：要复用同一特征定义与变换代码，保存时点信息，并持续比较线上样本与训练样本；同一产品品牌的 feature store 也不会自动消除语义和时间偏斜。」
- **相关**：[06-feature-and-data](../08-ml-systems/06-feature-and-data.md)、[07-model-quality-and-experimentation](../08-ml-systems/07-model-quality-and-experimentation.md)、[20-ranking-service](../06-case-studies/20-ranking-service.md)

**Q17 · 十亿级去重** 〔L5〕— 判断一个事件/URL 是否已经处理过。
- **考点**：布隆过滤器（Bloom filter）的空间-误判率公式｜误判方向的业务后果｜不支持删除的问题｜分片与滚动窗口｜精确去重的兜底
- **等这句**：「布隆过滤器只有**单向可信**：说'不存在'是 100% 准的，说'存在'有误判。所以它只能用在'误判导致多做一次工作'的场景，不能用在'误判导致漏掉一笔钱'的场景。10 亿 key、1% 误判约需 1.2 GB。真要精确，就用**滚动时间窗 + 精确去重表**，窗口长度等于你能容忍的客户端重放延迟上限。」
- **相关**：[02-caching](../01-building-blocks/02-caching.md)、[02-billing-and-metering](../03-saas-platform/02-billing-and-metering.md)

---

## 3. 实时与协作（6 题）

**Q18 · 聊天系统（IM）** 〔L5〕— 1 亿 DAU，支持群聊 5000 人，消息不丢不乱序。
- **考点**：长连接网关与会话路由｜消息 ID 的单调性与已读位点｜离线消息与多端同步｜群消息的扇出｜端到端加密对服务端能力的限制
- **等这句**：「顺序保证不能靠时间戳，要靠**会话内单调递增的 seq**，客户端用 seq 检测空洞并触发拉取。5000 人群不做写扩散（fan-out on write） —— 一条消息扇出 5000 次写是灾难，改成群维度存一份 + 每个成员只存一个已读位点，把扇出从写移到读。」
- **相关**：[04-networking-and-edge](../01-building-blocks/04-networking-and-edge.md)、[06-notification-platform](../06-case-studies/06-notification-platform.md)

**Q19 · 实时协作编辑** 〔L6〕— 100 人同时编辑一个文档，离线可编辑后合并。
- **考点**：OT vs CRDT 的取舍｜CRDT 的墓碑膨胀与压缩｜会话亲和路由（session affinity）与内存驻留｜历史与快照｜光标与在线状态是另一条链路
- **等这句**：「序列 CRDT 的主要代价之一是操作 ID、墓碑或位置元数据膨胀，具体倍率取决于算法和编辑历史。生产上要做快照、增量日志截断和安全垃圾回收；回收条件通常是服务器确认相关副本已越过因果稳定点，或让长期离线客户端丢弃旧历史、从新快照重同步，不现实地等待‘所有客户端永远都在线一次’。」
- **相关**：[05-realtime-collaboration](../06-case-studies/05-realtime-collaboration.md)

**Q20 · 在线状态（Presence）** 〔L5〕— 显示千万用户的在线/离线/正在输入。
- **考点**：心跳频率与状态过期｜写放大（每次状态变更通知所有好友）｜订阅关系的规模｜最终一致可接受｜成本远高于直觉
- **等这句**：「Presence 的成本由订阅图的边数和状态变更频率决定，不只是用户数。一个用户有 1000 个活跃订阅者时，一次状态变化最多就是 1000 次逻辑通知；因此按当前可见会话/活跃关系订阅、合并抖动，并给大 fan-out 用户单独路径。正在输入的节流从 3–5 秒起压测，但产品体验和连接规模会改变这个值。」
- **相关**：[05-realtime-collaboration](../06-case-studies/05-realtime-collaboration.md)、[04-networking-and-edge](../01-building-blocks/04-networking-and-edge.md)

**Q21 · 直播弹幕 / 大扇出广播** 〔L5〕— 单场 1000 万观众，弹幕秒级到达。
- **考点**：分层扇出树｜降级采样（不是所有弹幕都要送达每个人）｜边缘节点聚合｜背压（backpressure）｜峰值是平时的 100×
- **等这句**：「1000 万观众 × 每秒 100 条弹幕 = 10 亿次推送/秒，这个数字本身就说明**不能全量送达**。正确设计是承认弹幕是**有损体验**：边缘节点按秒聚合、采样、限流后再广播给本地连接。这不是妥协，这是产品语义 —— 没有人能读完每秒 100 条弹幕。」
- **相关**：[04-networking-and-edge](../01-building-blocks/04-networking-and-edge.md)、[03-resilience-patterns](../05-reliability/03-resilience-patterns.md)

**Q22 · 网约车派单** 〔L6〕— 实时匹配司机与乘客，秒级决策。
- **考点**：地理索引与候选集生成｜批量撮合 vs 逐单贪心｜派单的公平性与全局最优｜状态机与超时回收｜司机位置更新的写压力
- **等这句**：「逐单最近匹配会忽略即将到达的供需，批量撮合可能改善全局目标，但窗口越大用户等待越久。先用历史订单回放比较 0、1、2、5 秒窗口在接驾时间、取消率、公平性和算力上的曲线，再由产品选点；不能把 10–20% 收益写成所有城市和时段都成立。」
- **相关**：[02-event-driven-and-cqrs](../02-architecture-patterns/02-event-driven-and-cqrs.md)

**Q23 · 实时竞价（RTB）** 〔L6〕— 100ms 硬超时内完成竞价，日均百亿请求。
- **考点**：硬延迟预算（latency budget）的分配｜超时即放弃（不重试）｜特征的本地化与预加载｜降级到默认出价｜单请求成本必须 < 收益
- **等这句**：「竞价截止时间是单请求硬 deadline，服务 SLO 则描述有多少请求能在它之前完成，两者都要写。每一跳传递剩余预算；只有在剩余预算、额外流量和幂等性都允许时才做一次早期重试/对冲，临近截止直接放弃并用本地默认出价。远程特征可以存在，但必须有严格超时和本地/陈旧值降级，不能让任何依赖活得比竞价请求更久。」
- **相关**：[01-fundamentals](../00-foundations/01-fundamentals.md)、[03-resilience-patterns](../05-reliability/03-resilience-patterns.md)

---

## 4. SaaS 平台（8 题）

**Q24 · 多租户控制面** 〔L6〕— 5 万租户，租户级配置、开通、迁移。
- **考点**：控制面/数据面（control plane / data plane）分离｜声明式协调循环（reconcile）｜租户到 cell 的映射与路由形态｜爆炸半径（blast radius）｜控制面不能进数据面关键路径
- **等这句**：「租户→cell 的映射**必须能被数据面缓存并在控制面挂掉时继续用**，否则控制面就成了全站单点。另外 AWS 自己的 cell 架构文档**没有承诺可用性 SLA 提升**，它承诺的是降低爆炸半径和缩短恢复时间 —— 这两件事经常被混为一谈，不要在 SLA 承诺里写'因为 cell 化所以 99.99%'。」
- **相关**：[01-control-plane](../03-saas-platform/01-control-plane.md)、[03-multi-tenancy](../02-architecture-patterns/03-multi-tenancy.md)

**Q25 · 用量计量与出账** 〔L6〕— 每秒 10 万条用量事件，月底出账不能错。
- **考点**：事件去重（CloudEvents `source`+`id`）与去重窗口成本｜预聚合 vs 原始事件保留｜迟到事件的处理策略｜三方对账｜幂等键从业务语义派生
- **等这句**：「聚合和出账从业务响应路径解耦，但用量事实必须在业务提交边界内可靠落下——同事务 outbox、持久日志或可对账的源记录，不能只是一次可能失败的异步 publish。事件丢失会少收，重复会多收，方向并不总对供应商不利；所以要去重、保留原始事件，并让供应商账单、网关/产品事实与计量结果三方对账。迟到事件如何跨账期修正要写进产品条款。」
- **相关**：[02-billing-and-metering](../03-saas-platform/02-billing-and-metering.md)、[04-usage-based-billing](../06-case-studies/04-usage-based-billing.md)

**Q26 · 权限系统（ReBAC）** 〔L6〕— 支持嵌套组、资源继承、每次请求鉴权 < 10ms。
- **考点**：RBAC→ABAC→ReBAC 的演进动机｜Zanzibar 的关系元组与图遍历｜一致性 token（改权限后立刻读）｜缓存与失效｜反向查询（列出我能看的所有资源）
- **等这句**：「简单单体里的 RBAC 可以先用数据库约束和本地策略；出现跨服务关系图、嵌套组和统一审计后，再把授权抽成独立服务。ReBAC 服务要为‘写关系后立刻 check’提供版本/一致性 token，并把正向 check 与 list-objects 两种访问模式分别建索引和压测；反向查询通常更重，但不是固定贵十倍。」
- **相关**：[03-identity-and-authz](../03-saas-platform/03-identity-and-authz.md)

**Q27 · 企业身份接入（SSO / SCIM）** 〔L5〕— 支持 500 个企业客户各自的 IdP。
- **考点**：OIDC vs SAML 的现实差异｜SCIM 用户同步与去激活时延｜多 IdP 的租户发现（home realm discovery）｜JIT provisioning｜离职员工的会话撤销
- **等这句**：「安全审计真正会问的是**去激活的端到端时延**：员工在 IdP 被停用后，多久他手上的 access token 会失效？如果你的 token 有效期 1 小时且没有实时撤销通道，答案就是 1 小时 —— 这在很多合规框架下不可接受。所以要么缩短 token 生命周期（增加 IdP 压力），要么接 SCIM/CAEP 的实时事件。」
- **相关**：[03-identity-and-authz](../03-saas-platform/03-identity-and-authz.md)

**Q28 · Feature flag 与渐进式发布** 〔L6〕— 支持按租户/用户/百分比灰度，秒级生效与回滚。
- **考点**：flag 求值放客户端还是服务端｜配置分发的惊群（thundering herd）与本地兜底｜flag 债务（清理机制）｜与 schema 迁移的配合｜回滚必须比发布快
- **等这句**：「Flag 系统的真正难点不是分发，是**清理**。没有过期机制的 flag 会在两年内变成几千个，代码里全是死分支，而且没人敢删。必须在创建时就强制填写'预期下线日期'和 owner，过期自动提 issue。另外**回滚路径必须比发布路径更快更简单** —— 发布可以走 CI，回滚必须一个开关。」
- **相关**：[05-release-engineering](../03-saas-platform/05-release-engineering.md)

**Q29 · 审计日志与合规导出** 〔L6〕— 不可篡改、可按租户导出、保留 7 年。
- **考点**：写入路径的可靠性（审计写失败要不要阻塞业务）｜不可篡改（哈希链/WORM 存储）｜按租户分区与导出｜冷存储成本｜查询模式与保留分层
- **等这句**：「第一个问题是审计记录与业务动作的原子边界：高风险操作可以同事务写 append-only outbox，失败则不提交；低风险遥测可异步，但要定义丢失预算。AI 日志记录哪些模型、prompt、检索来源和授权依据要按用途、PII 与风险分类决定。不要误引法律：EU AI Act 高风险系统的日志/记录条款主要在 Article 12 与 19；[Article 50](https://digital-strategy.ec.europa.eu/en/policies/guidelines-transparency-ai-generated-content)讲某些 AI 交互和生成内容的透明度。SOC 2 也应映射到具体控制，不能说二者‘指向同一份日志’。」
- **相关**：[04-isolation-and-compliance](../03-saas-platform/04-isolation-and-compliance.md)

**Q30 · Webhook 投递平台** 〔L5〕— 给 10 万客户投递事件，对方端点经常挂。
- **考点**：至少一次投递与幂等要求｜指数退避（exponential backoff）与重试预算｜慢消费者隔离（一个客户挂了不能影响别人）｜签名与重放防护｜死信（dead-letter queue）与人工重投
- **等这句**：「最常见的翻车是**队头阻塞（head-of-line blocking）**：一个客户的端点超时 30 秒，把共享的 worker 池占满，所有客户都受影响。必须做**按客户分区的隔板（bulkhead）**（每个客户独立的并发配额），并对持续失败的端点做熔断（比如连续失败 24 小时就暂停投递并发邮件）。签名要带时间戳并校验窗口，否则签名可被无限重放。」
- **相关**：[06-notification-platform](../06-case-studies/06-notification-platform.md)、[03-resilience-patterns](../05-reliability/03-resilience-patterns.md)

**Q31 · 数据驻留（data residency）与租户密钥（BYOK）** 〔L6〕— 欧盟客户要求数据不出境、密钥自持。
- **考点**：存储位置 / 处理位置 / 缓存位置是三个独立开关｜密钥层级（KEK/DEK）与撤销语义｜跨境的元数据泄露｜BYOK 撤销后的可恢复性｜合规溢价的成本建模
- **等这句**：「数据驻留至少拆成存储、处理、缓存/日志、备份和运维访问五条数据流；逐项对照合同与目标厂商当期 region 文档，不背会过期的‘10 个/3 个’。BYOK 还要定义撤销语义：哪些派生缓存、索引和备份由客户密钥保护，撤销后多久不可读、能否恢复、谁能重新授权。做不到全链路就明确范围，不把部分加密包装成全系统 crypto-erasure。」
- **相关**：[04-isolation-and-compliance](../03-saas-platform/04-isolation-and-compliance.md)

---

## 5. AI 与 Agent（16 题）

**Q32 · 企业级 LLM 网关** 〔L6〕— 统一接入多家模型，做限额、路由、缓存、审计。
- **考点**：虚拟 key 与 TPM/RPM/预算三元组｜供应商故障的 failover 与语义差异｜缓存分层（前缀/语义/工具结果）｜流式响应的中途失败｜失败请求是否计入预算
- **等这句**：「网关必须显式决策一条策略：**失败请求是否计入租户预算**。退款体验好但制造滥用面，计入更公平但对瞬时故障苛刻 —— 这个决定必须写进合同并对租户可见。另外流式响应的失败最难处理：token 已经吐出去一半，计费怎么算、重试怎么做，这是所有 LLM 网关的共同坑。」
- **相关**：[02-llm-gateway](../06-case-studies/02-llm-gateway.md)、[08-cost-and-latency](../04-ai-agent-systems/08-cost-and-latency.md)

**Q33 · 企业知识库 RAG** 〔L6〕— 1000 万文档，多租户，回答要可溯源。
- **考点**：混合检索（BM25 + dense + RRF）｜分块与上下文化｜重排（reranking）的候选数与延迟｜权限过滤必须在检索层｜评测集与召回目标
- **等这句**：「权限必须在文档进入模型上下文前过滤；生成后再删已经太晚。对错误码、SKU、函数名等精确词法和语义问法并存的企业语料，BM25 + dense 是合理候选，但是否融合、用 RRF 还是加权分数要由自有查询集评测。先做几十到几百条分层标注用于方向判断，再根据置信区间补样本；+1.3% 和 40 条都不是通用结论。」
- **相关**：[02-context-engineering-and-rag](../04-ai-agent-systems/02-context-engineering-and-rag.md)

**Q34 · 多租户向量检索** 〔L6〕— 10 万租户，冷热差异极大，单租户 1 万–1 亿向量。
- **考点**：namespace / 独立索引 / 过滤三种隔离范式的硬边界｜冷租户的内存驻留成本｜索引构建与在线更新｜召回率与延迟的可调参数｜namespace 数量爆炸
- **等这句**：「先画每租户向量数和查询频率分布；若确实长尾明显，再把热租户常驻、冷租户放磁盘/共享索引或可加载快照。对象存储只是冷层候选，首次查询延迟取决于索引格式、下载量和缓存，不能先承诺 1–3 秒。namespace、payload filter 和独立索引都有厂商硬边界，上线前查 limits，并压测最大租户与大量空闲租户两个极端。」
- **相关**：[03-multi-tenant-vector-search](../06-case-studies/03-multi-tenant-vector-search.md)、[01-storage-engines](../01-building-blocks/01-storage-engines.md)

**Q35 · 云端编码 Agent 平台** 〔L6〕— 1 万开发者，每人每天几个会话，要跑代码。
- **考点**：沙箱隔离（microVM vs gVisor vs 容器）｜会话时长上限与状态外置｜成本结构｜并发配额与背压｜代码执行的出口管控（egress control）
- **等这句**：「先按会话把推理、沙箱、存储和出网成本归因；某个厂商的单一算例不能证明推理永远占 85–89%。代码编译、浏览器和长空闲会话可能让沙箱成为主项。隔离也按威胁模型分级：普通容器共享宿主内核，不足以单独承载高风险不可信代码；可选 gVisor、Kata/microVM 或专用 VM，并配 seccomp、只读文件系统、资源上限、出口代理和短时凭证。不是只说‘必须 microVM’就安全了。」
- **相关**：[01-ai-coding-agent-platform](../06-case-studies/01-ai-coding-agent-platform.md)、[03-agent-runtime](../04-ai-agent-systems/03-agent-runtime.md)

**Q36 · Agent 运行时与可恢复执行** 〔L6〕— 支持运行数小时的长任务，进程崩了要能续。
- **考点**：checkpoint vs durable execution 的定义之争｜外部副作用的幂等键 `(workflow_id, step_id)`｜状态外置（progress 文件 + git commit）｜沙箱会话硬上限｜重放的确定性
- **等这句**：「Checkpoint 只回答‘从哪份状态继续’，durable execution 还要回答步骤是否重跑、外部副作用如何去重、代码版本变化后怎么恢复。纯计算步骤可从 checkpoint 重算；付款、发信、开单等副作用用稳定的 `(workflow_id, step_id)`、状态日志与查询/对账。长任务不要押在某个进程或沙箱会一直存活，具体会话时限查所选平台并设计租约续期与状态外置。」
- **相关**：[03-agent-runtime](../04-ai-agent-systems/03-agent-runtime.md)、[04-agent-memory-and-state](../04-ai-agent-systems/04-agent-memory-and-state.md)

**Q37 · Agent 记忆系统** 〔L6〕— 让 Agent 跨会话记住用户偏好与项目状态。
- **考点**：记忆分层（工作记忆/情景/语义/程序）｜写入时机与冲突消解｜遗忘策略｜检索记忆本身会污染上下文｜记忆的租户隔离
- **等这句**：「记忆系统最危险的失败之一是错误事实被反复召回。因此每条记忆带来源、时间、作用域和可撤销/覆盖关系；用户陈述与系统推断分开。检索到的每条记忆都会占上下文并可能成为干扰项，所以用自有任务测 top-k、冲突率和过期命中，优先少而相关，不背‘1 个或 4 个 distractor’的通用拐点。」
- **相关**：[04-agent-memory-and-state](../04-ai-agent-systems/04-agent-memory-and-state.md)、[02-context-engineering-and-rag](../04-ai-agent-systems/02-context-engineering-and-rag.md)

**Q38 · 多 Agent 编排** 〔L6〕— 一个研究型任务分给多个子 Agent 并行做。
- **考点**：编排 vs 编舞（orchestration vs choreography）｜子 Agent 的上下文防火墙与有损摘要｜共享写入所有权｜扇出配额｜orchestrator 的认知视界
- **等这句**：「并行最适合独立的检索、评审和互不重叠的产物；多个 Agent 写同一工作区、同一记录或同一外部系统时，必须先划分所有权，或让单一协调者串行合并。不是所有'写'都必须单线程，而是**共享可变状态不能没有冲突协议**。子 Agent 回传摘要时还要附原始证据或产物路径，让父 Agent 能抽查，而不是把有损摘要当成不可复审的真相。」
- **相关**：[05-multi-agent-orchestration](../04-ai-agent-systems/05-multi-agent-orchestration.md)

**Q39 · 模型评测平台（Eval）** 〔L6〕— 给 50 个 AI 功能提供上线门禁（release gate）。
- **考点**：离线 eval / 在线 A/B / 影子流量（shadow traffic）的分工｜LLM-as-judge 的偏差与校准｜回归集从生产轨迹构建｜公开 benchmark 不能当门禁｜评测本身的成本
- **等这句**：「公开 benchmark 可以做外部参照，不能单独充当上线验收。[Berkeley RDI 的对抗审计](https://rdi.berkeley.edu/blog/trustworthy-benchmarks-cont/)展示了 8 个 agent benchmark 的评分基础设施都可被不同程度利用，其中 SWE-bench Verified 能在未修 bug 的情况下被打到 100%；这证明你同时在测 agent 和 evaluator。门禁要补自有生产轨迹回归集、污染检查与人工抽查。LLM judge 先与盲标人工样本校准，报告分类型混淆矩阵和人与人基线；Cohen's kappa 没有脱离任务风险的统一上线阈值。」
- **相关**：[06-evaluation-and-observability](../04-ai-agent-systems/06-evaluation-and-observability.md)

**Q40 · LLM 可观测性与轨迹追踪** 〔L6〕— 记录每次 Agent 运行的完整轨迹并可查。
- **考点**：trace 结构（span 层级、工具调用、token 计数）｜高基数与采样策略｜PII 脱敏｜成本归因（cost attribution）到租户/功能｜OTel GenAI 语义约定的现状
- **等这句**：「我会全量保留低成本的结构化元数据（模型、token、延迟、工具名、结果状态），但 prompt、tool 参数与输出正文按 PII、租户策略、错误/慢请求和成本预算分层采样、脱敏、加密与限期保留；'全部原文永远存'既可能很贵，也可能违规。OpenTelemetry 的 GenAI 约定仍在演进，并已迁到独立仓库；自己的稳定 schema 与外部 `gen_ai.*` 字段之间要留版本化映射，见 [OTel 说明](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-exceptions/)。」
- **相关**：[06-evaluation-and-observability](../04-ai-agent-systems/06-evaluation-and-observability.md)、[02-observability](../05-reliability/02-observability.md)

**Q41 · GPU 推理集群调度** 〔L6〕— 200 张卡，多模型多租户，要满足延迟 SLO。
- **考点**：TTFT/TPOT/goodput 的定义｜PD 分离的前置条件｜KV cache 感知路由｜MIG/MPS/time-slicing 的隔离性差异｜模型加载与冷启动
- **等这句**：「容量规划看 **goodput**（每秒同时满足 TTFT 与 TPOT SLO 的请求数），不能只报 TTFT。需要强故障与资源隔离的跨租户负载优先 MIG 或整卡；MPS / time-slicing 更适合互信或低风险共享，隔离强度与硬件代际有关，必须分别压测显存争用、长 kernel、OOM 和故障传播，不能只看平均吞吐。」
- **相关**：[01-llm-serving-infra](../04-ai-agent-systems/01-llm-serving-infra.md)

**Q42 · 提示词与上下文管理平台** 〔L6〕— 给 30 个团队管理 prompt 的版本、灰度与回滚。
- **考点**：prompt 作为代码制品（版本、审查、回滚）｜与前缀缓存的耦合｜A/B 与影子评测｜模板变量的注入风险｜跨模型迁移
- **等这句**：「前缀缓存按前缀字节或 provider 定义的缓存块复用；改动发生在已缓存前缀里时，后续部分会 miss，改在稳定前缀之后则未必清空全部收益。Prompt 平台因此要显示每个版本的稳定前缀边界，灰度前估算 `cache_read` 变化，并同时看质量、延迟和账单。具体失效粒度、TTL 与折扣按模型供应商实测，不能背成'改一个字符必然全量 miss、成本固定涨十倍'。」
- **相关**：[08-cost-and-latency](../04-ai-agent-systems/08-cost-and-latency.md)、[05-release-engineering](../03-saas-platform/05-release-engineering.md)

**Q43 · AI 计费与 token 配额** 〔L6〕— 防止单个租户或失控 Agent 烧穿月度预算。
- **考点**：软/硬双层限额｜单会话与单任务的绝对上限｜实时用量聚合的延迟｜降级到小模型的策略｜成本归因到功能
- **等这句**：「预算要按时间与作用域分层：请求/任务硬上限阻止单次失控，会话和租户窗口做告警/降级，月度合同额度决定何时拒绝或人工放行。软限额与硬限额的组合通常比单个阈值更可运营，但层数和动作要由产品合同决定。多 Agent 扇出会把 token 成本乘上分支数和返工轮次，因此创建子 Agent 前做预算预留，运行中按实际用量回收，倍率用自己的 trace 分布测，不背一个通用的 15×。」
- **相关**：[02-billing-and-metering](../03-saas-platform/02-billing-and-metering.md)、[08-cost-and-latency](../04-ai-agent-systems/08-cost-and-latency.md)

**Q44 · Agent 安全网关** 〔L6〕— 让 Agent 能用工具，但不能被提示注入（prompt injection）劫持。
- **考点**：致命三要素（私有数据 + 不可信内容 + 外发能力）｜确定性隔离 vs 概率性护栏｜出口管控与域名白名单｜工具描述本身就是攻击面｜人在回路的静态清单
- **等这句**：「提示注入要按架构风险处理：分类器和模型自检只能做纵深，真正边界是能力最小化、参数校验、出口管控，以及对高影响不可逆动作的确定性策略和人工确认。MCP 的风险在工具调用前就开始了——工具描述会进入模型上下文；调用后的返回内容同样是不可信输入。发现、选择、执行、输出四段都要有信任边界，不能把风险只定位在 tool description 或只定位在调用本身。」
- **相关**：[07-agent-security](../04-ai-agent-systems/07-agent-security.md)

**Q45 · 多模态处理管道** 〔L6〕— 每天 10 万个视频/音频/PDF，抽取结构化信息。
- **考点**：分片与并行（长视频切段）｜多阶段漏斗（便宜模型先筛）｜幂等与断点续处理｜大文件的存储与传输成本｜结果的可溯源（时间戳/页码）
- **等这句**：「多模态管道的成本几乎完全由**漏斗设计（funnel design）**决定：先用便宜的确定性手段（关键帧抽取、VAD、PDF 文本层）把输入量降一到两个数量级，再让模型处理剩下的。直接把整个视频喂给多模态模型是最贵也最不准的做法。另外每个抽取结果必须带**定位信息**（时间戳、页码、bbox），否则下游无法核验，整个管道的输出就不可信。」
- **相关**：[02-context-engineering-and-rag](../04-ai-agent-systems/02-context-engineering-and-rag.md)、[08-cost-and-latency](../04-ai-agent-systems/08-cost-and-latency.md)

**Q46 · 语义缓存（semantic cache）服务** 〔L5〕— 用向量相似度复用历史回答。
- **考点**：相似 ≠ 答案相同｜阈值与租户分区｜时效性数据不能缓存｜命中率与误命中的代价不对称｜和前缀缓存的 ROI 对比
- **等这句**：「语义相似不等于答案可复用：地点、时间、身份或权限差一个字段，答案就可能完全不同。因为误命中通常比未命中更贵，我会先做确定性的前缀/精确缓存，再只在无个性化、低时效且可验证的 FAQ 上试语义缓存；阈值由正负样本 ROC、业务损失和租户隔离共同决定，不从 0.97 或'十倍 ROI'这类通用数字起誓。」
- **相关**：[02-caching](../01-building-blocks/02-caching.md)、[08-cost-and-latency](../04-ai-agent-systems/08-cost-and-latency.md)

**Q47 · 模型路由与级联** 〔L6〕— 用小模型接住大部分请求，只把难的升级到大模型。
- **考点**：路由信号（分类器/置信度/规则）｜级联抬高尾延迟｜质量回归的监控｜路由决策必须可旁路｜实测收益区间
- **等这句**：「级联的收益完全取决于请求难度分布和路由器误判成本，先用影子流量测'小模型可独立通过的比例、漏升率、单位质量成本'，再给结论。串行升级会把小模型和大模型延迟相加并抬高尾部，所以路由要能一键旁路；对明显困难的请求可直接进大模型，避免先付一次注定无用的小模型延迟。」
- **相关**：[08-cost-and-latency](../04-ai-agent-systems/08-cost-and-latency.md)、[02-llm-gateway](../06-case-studies/02-llm-gateway.md)

---

## 6. 偏运维与可靠性（7 题）

**Q48 · SLO 与错误预算平台** 〔L6〕— 给 200 个服务统一定义 SLI/SLO 并告警。
- **考点**：SLI 必须从用户视角定义｜多窗口多燃尽率（burn rate）告警｜错误预算（error budget）政策的组织约束｜依赖服务的 SLO 组合｜99.99% 的真实代价
- **等这句**：「**多窗口多燃尽率**是一个经过验证的默认起点：短窗口抓快烧、长窗口过滤瞬时噪声；窗口和阈值要按 SLO 周期、值班响应时间与流量形状校准，不能把 1 小时 / 14.4× 和 6 小时 / 6× 当成所有系统的唯一参数。另外错误预算真正的价值在**组织政策**：预算耗尽后是冻结发布、提高审批还是只限制高风险变更，必须预先约定。」
- **相关**：[01-slo-and-error-budget](../05-reliability/01-slo-and-error-budget.md)

**Q49 · 分布式追踪采样系统** 〔L6〕— 每天 100 亿 span，要能查到慢请求和错误。
- **考点**：头部采样（head-based sampling） vs 尾部采样（tail-based sampling）的取舍｜尾部采样的内存与状态成本｜采样决策的传播（一致性）｜高基数属性｜成本上限是硬约束
- **等这句**：「头部采样便宜，却可能在知道结果前丢掉稀有错误和慢请求；尾部采样能按结果保留，代价是要把 trace 暂存在内存或磁盘直到决策完成。组合策略通常是：给错误、慢请求和关键租户更高保留率，正常请求按成本预算采样，并设置每租户与全局上限，避免故障时错误突增把采样器本身打垮。采样状态还要随 trace context 传播，或在后端明确接受部分 trace。」
- **相关**：[02-observability](../05-reliability/02-observability.md)

**Q50 · CI/CD 与制品分发** 〔L5〕— 500 个服务，每天 2000 次部署。
- **考点**：制品（artifact）不可变与内容寻址（content-addressed）｜依赖图与增量构建｜部署编排与并发限制｜回滚是一等公民｜构建缓存的命中率
- **等这句**：「**制品必须不可变且内容寻址** —— 同一个 tag 指向不同内容是所有'在我机器上是好的'的根源。部署编排的重点是**并发限制与依赖顺序**：2000 次部署/天意味着任何时刻都有几十个在跑，不限并发会让下游依赖同时经历多个上游变更，故障归因变成不可能。」
- **相关**：[05-release-engineering](../03-saas-platform/05-release-engineering.md)

**Q51 · 混沌工程平台** 〔L6〕— 在生产环境注入故障且不伤害用户。
- **考点**：稳态假设（steady-state hypothesis）与自动中止（abort）条件｜爆炸半径控制｜实验的可观测前置条件｜从演练到常态化｜组织接受度
- **等这句**：「混沌实验的前提不是勇气是**可观测性**：如果你没有能在 60 秒内判断'系统是否偏离稳态'的指标，你就不能注入故障，因为你不知道什么时候该停。所以正确顺序永远是**先补齐监控和自动中止，再做实验**，而且第一批实验要在预发或单个 cell 里跑 —— 爆炸半径是设计出来的，不是祈祷出来的。」
- **相关**：[04-incident-and-chaos](../05-reliability/04-incident-and-chaos.md)

**Q52 · 分片再平衡系统** 〔L6〕— 在线把 8 个分片扩到 32 个，不停服。
- **考点**：逻辑分片（logical shard）与物理库的 1:N 映射｜带位点的快照 + CDC｜影子读（shadow read）｜迁移期写入 fencing / 版本校验｜限速与源库保护｜回滚点
- **等这句**：「逻辑分片能让多数扩容变成‘移动若干虚拟分片’，但数量要按增长、目录大小和迁移粒度测算，1024 只是例子。迁移时保留一个权威写入源：先记录 CDC 位点并做限速快照，再持续应用带版本的变更；追平后做影子读比对，按租户或逻辑分片灰度切读，最后用 epoch/fencing 阻止旧路由继续写，再停止旧副本。若业务被迫双写，也要由同一份 outbox/变更日志幂等地投到两边，不能让请求线程做无事务的两次写。每一步都预写回滚条件，但切断旧写入之后要明确新的恢复点。」
- **相关**：[05-scaling-playbook](../05-reliability/05-scaling-playbook.md)、[03-multi-tenancy](../02-architecture-patterns/03-multi-tenancy.md)

**Q53 · 全球流量调度与故障转移（failover）** 〔L6〕— 3 个 region，单 region 故障要 5 分钟内切走。
- **考点**：DNS TTL 的真实生效时间｜Anycast vs GSLB｜健康检查的灰色故障（gray failure）问题｜切走后的容量（另外两个 region 扛得住吗）｜数据层能不能跟着切
- **等这句**：「切流量容易，**切数据难**。真正要回答的是：其他 region 有没有满足 RPO 的副本、复制窗口内的写怎么裁决，以及失去一个 region 后剩余容量能否承接故障流量。容量余量要从故障模型和流量分布算，不能背 1.5×。DNS TTL 也只是缓存期限的一部分；客户端、连接复用、健康检查和路由层都会影响实际切换尾巴，所以要实测并保留非 DNS 的隔离/切流手段。」
- **相关**：[04-networking-and-edge](../01-building-blocks/04-networking-and-edge.md)、[05-scaling-playbook](../05-reliability/05-scaling-playbook.md)

**Q54 · 密钥管理与轮换（key rotation）** 〔L6〕— 500 个服务的数据库口令与 API key 自动轮换。
- **考点**：静态密钥 vs 短时凭据（workload identity）｜轮换期的双密钥并存｜轮换失败的爆炸半径｜密钥泄露的检测与撤销｜Agent 场景的凭据下发
- **等这句**：「轮换的正确形态是**双密钥并存窗口**：先发新密钥、等所有消费者都在用新的、再撤旧的 —— 单点切换的轮换一定会在某个凌晨炸掉一个你忘了的消费者。更根本的方向是**消灭静态密钥**，用 workload identity 换短时凭据。Agent 场景还有一条硬规矩：**凭据不进沙箱** —— token 由沙箱外的代理持有，只向内下发短时、窄 scope 的派生凭据，下游一律 token exchange 不透传。」
- **相关**：[03-identity-and-authz](../03-saas-platform/03-identity-and-authz.md)、[07-agent-security](../04-ai-agent-systems/07-agent-security.md)

---

## 面试官会追问

不管你抽到哪道题，下面 8 条会以某种形式出现。**先把这 8 条练熟，比多刷 20 道题有用。**

1. 这个方案什么时候会失效？失效前我在监控上能看到什么信号？
2. 你这个组件挂了会怎样？谁先感知到？降级成什么具体行为？
3. 这个数字（QPS / 容量 / 成本）你怎么算出来的？假设是什么？
4. 流量涨 10×，第一个撑不住的是什么？
5. 这里面哪个决策是单向门？改回去要多久、要多少人？
6. 如果预算砍一半 / 团队只有 3 个人，你砍掉什么？
7. 一个租户的异常流量会不会影响其他租户？隔离边界在哪？
8. （AI 题必问）一个重度用户一个月消耗多少钱？你的定价覆盖得住吗？

---

**按训练路径阅读** → 回 [START-HERE](../START-HERE.md) 按所选路径继续；页尾链接只表示本目录或专章的顺读顺序。

**面试专章顺读下一篇** → [03-cheatsheet.md](03-cheatsheet.md)：数字、公式、决策树、口袋话术。
