# 07 · 经典题速解集：10 道最高频题的压缩打法

> 前 6 个案例是深度题，这一篇是**广度题**：10 道最经典的通用系统设计题，每题只写面试里真正拿分的那 20%。
> 而这 10 题其实只有 **4 种母题**。认出母题比背题重要 —— 认出来了，没见过的题也能现推；背题的人换个题面就崩。

---

## 0. 经典题的通用骨架

| 4 种母题 | 它守的不变量（invariant） | 标准武器 | 典型撞墙条件 |
|---|---|---|---|
| **A · ID 与唯一性**<br>ID & uniqueness | "这个东西全局只有一个" | 号段分配 / Snowflake / UUIDv7；**数据库唯一索引作为唯一裁决者**；业务派生的幂等键 | 单行/单索引热点；发号器单点；时钟回拨 |
| **B · 扇出与聚合**<br>fan-out & aggregation | "一次写变 N 次写，或一次读要归并 N 个源" | 混合推拉 + 用成本交点算阈值；节点预存 Top-K；预聚合与不可变镜像 | 长尾（大 V、热门前缀）把一条路径打爆 |
| **C · 并发与库存**<br>concurrency & inventory | "总数有限的资源不能被超发" | 条件更新 `WHERE n > 0` / 唯一约束；带 TTL 的持有 + **惰性过期判定**；准入层削峰 | 单行 TPS 上限（~1k）；持锁进程 GC 暂停 |
| **D · 长连接与状态**<br>connection & state | "客户端在线、有状态、且状态会跨故障" | 接入层与逻辑层分离 + session 注册表；**推送管延迟、拉取管正确性**；每个状态都有超时兜底 | 重连风暴；心跳成本超过业务本身；僵尸状态堆积 |

### 识别母题的 4 个提问（30 秒内做完）

```
① 有没有一个必须"全局唯一"的东西？→ A     ③ 有没有"总量有限"的资源被多人抢？→ C
② 有没有一次写变 N 次写 / 一次读归并 N 源？→ B  ④ 客户端会不会长在线且需被推送？→ D
一道题命中 2 个母题很正常（打车 = C+D）。命中的那两个，就是你该提名的深挖点。
```

### 题目 → 母题 → 本书章节

| # | 题目 | 母题 | 决定评分的那一个子问题 | 对应章节 |
|---|---|---|---|---|
| 1 | URL shortener | A | 自定义短码与自动短码的冲突裁决 | [01-storage-engines](../01-building-blocks/01-storage-engines.md)、[02-caching](../01-building-blocks/02-caching.md) |
| 2 | Rate limiter | A+C | 限流器自己挂了怎么办 | [03-resilience-patterns](../05-reliability/03-resilience-patterns.md) |
| 3 | News feed | B | 大 V 阈值怎么算出来 | [03-messaging-and-streams](../01-building-blocks/03-messaging-and-streams.md) |
| 4 | Chat system | D | 推送不可靠时正确性靠什么 | [04-networking-and-edge](../01-building-blocks/04-networking-and-edge.md)、[06-notification-platform](06-notification-platform.md) |
| 5 | Distributed cache | B | 缓存挂了 DB 怎么活 | [02-caching](../01-building-blocks/02-caching.md) |
| 6 | Ride-hailing | C+D | 司机独占性与超时收敛 | [05-consensus-and-coordination](../01-building-blocks/05-consensus-and-coordination.md) |
| 7 | Ticket booking | C | 不超卖、不锁死、能自动释放 | [01-storage-engines](../01-building-blocks/01-storage-engines.md) |
| 8 | Payment system | A+C | 三方状态不一致怎么收敛 | [02-event-driven-and-cqrs](../02-architecture-patterns/02-event-driven-and-cqrs.md)、[04-usage-based-billing](04-usage-based-billing.md) |
| 9 | Autocomplete | B | 热度更新与个性化的成本 | [02-context-engineering-and-rag](../04-ai-agent-systems/02-context-engineering-and-rag.md) |
| 10 | Object storage | — | 字节不流经应用层 | [04-networking-and-edge](../01-building-blocks/04-networking-and-edge.md) |

**用法**：每题 45 分钟档的时间分配固定为 `澄清 5 / 估算 4 / 高层 10 / 深挖 18 / 收敛 5`，节奏见 [`07-interview/01-interview-framework.md`](../07-interview/01-interview-framework.md)。

---

## 1. 十题速解

### 1. 设计短链接服务（Design a URL Shortener）

**一句话本质**：不考"怎么把长 URL 变短"，考**发号器的冲突处理**和 100:1 读放大下的缓存分层。跳转本身是一次 KV 查询，没有难度。

**必问的 3 个澄清问题**
1. 短码要**不可枚举**吗（防竞品爬全量）？→ 决定用顺序号段还是加置换。
2. 支持**自定义短码（custom alias）**吗？→ 决定有没有唯一性冲突这条路径，这是胜负手。
3. 链接会过期/删除吗？点击统计要实时吗？→ 决定 TTL 回收和第二条异步管道。

**关键估算**
```
1 亿跳转/天 = 1,157 QPS 均值 × 3（峰谷比）≈ 3,500 QPS 读；写 100 万/天 = 12 QPS ⇒ 读写比 100:1
10 年存量 36.5 亿条 × 500 B ≈ 1.8 TB（含索引 ×3 ≈ 5 TB）
Base62：62⁶ = 568 亿，62⁷ = 3.5 万亿 ⇒ 7 位永远够用
Top 1% 链接占约 50% 流量 ⇒ 1 GB 缓存可覆盖 90%+ 请求
```

**核心设计**
```
写: Client ─▶ API ─▶ 号段分配器（DB 一行: max_id, step=100k，实例一次领 10 万个）
                     └─▶ id → Feistel 置换 → Base62 → code
                     └─▶ INSERT(code PK, url, owner, expire_at)   ← 唯一索引 = 唯一裁决者
读: Client ─▶ CDN ─▶ LB ─▶ Redirect Svc ─▶ L1 LRU(1万) ─▶ Redis ─▶ DB
                                │ 302
                                └─▶ Kafka: click 事件（异步，不阻塞跳转）
```
- 号段（segment）分配：本地自增，**无锁无冲突**，发号对 DB 的写压力 ≈ 0。代价是实例重启丢一段号 —— 62⁷ 的空间里每天丢 10 万也要 9 万年才耗尽。Feistel 网络（或任意 64-bit 块置换）再把自增 ID 映射成不可预测整数：**双射保证零冲突，置换保证不可枚举**，比"哈希 + 撞了重试"强一个数量级。
- 跳转返回 **302 而不是 301**：301 会被浏览器永久缓存，你永远收不到统计、也无法改目标或封禁恶意链接 —— 这是不可逆的。

**面试的胜负手**
> 短码绝不用"哈希 + 撞了重试"生成 —— 重试让写路径尾延迟不可控，且会和用户占用的自定义码空间互相踩。我用号段分配 + Feistel 置换：双射零冲突、置换不可枚举。**自定义短码和自动短码放同一张表的同一个唯一索引下，由数据库唯一约束做唯一裁决者，应用层绝不做"先查再插"。**
> I'd never generate short codes by hashing and retrying on collision — retries make write-path tail latency unbounded, and they fight with the custom-alias namespace for the same keys. I'd use segment-based ID allocation plus a Feistel permutation: the bijection gives me zero collisions, and the permutation makes the codes unguessable, so nobody can crawl the whole namespace. Custom aliases and generated codes live in the same table under one unique index, so the database constraint is the only arbiter — the application never does check-then-insert.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 自定义码和自动码会撞吗 | 同表同唯一索引，`INSERT ... ON CONFLICT DO NOTHING`；应用层"先 SELECT 再 INSERT"在并发下必错 |
| 缓存穿透（拿随机码扫） | 布隆过滤器 + 空值缓存 60 s + 按 IP 限流；短码不可枚举本身已把扫描成本抬到不可行 |
| 热点链接（一条被转爆） | L1 进程内缓存 1–5 s，直接削掉 Redis 单分片 90% QPS；再热就推 CDN |
| 怎么过期与回收 | `expire_at` 索引 + 每日批量清理；**短码永不回收** —— 回收会让旧链接指向新目标，是安全事故 |
| 统计要精确吗 | 不要。HyperLogLog / 批量 flush，0.8% 误差换 100× 成本 |

**失败模式**
- **随机码在 B-Tree 上是随机插入**，页分裂严重；写超过 5k TPS 时按 code 前 2 位分片。
- **号段服务单点**：号段表所在 DB 挂 = 新短链创建全停（跳转不受影响）。每实例预取两段可撑过 10 分钟故障。

**常见错误答法**
- ❌「用 MD5 取前 7 位」—— 36 亿条下生日碰撞必然发生，你被迫处理冲突，退化成"查-重试"循环；面试官下一句必问重试上限，你答不出。
- ❌「自增 ID 直接 Base62」—— 可枚举，竞品能遍历你全部链接并从 ID 推出你的业务量。能跑，但安全 review 必被打回。

---

### 2. 设计限流器（Design a Rate Limiter）

**一句话本质**：算法只占 20% 的分。真正考的是**在 200 个实例上做分布式计数时，你愿意为精确性付多少延迟**，以及限流器自己挂了怎么办。

**必问的 3 个澄清问题**
1. 限流的**维度与基数**：per-tenant，还是 per-tenant × endpoint × model？基数 1 万还是 1 亿？→ 决定能不能集中式。
2. **超发（over-admission）**容忍度：超 1% 可接受，还是必须精确？→ 决定本地桶还是 Redis。
3. 限流器挂了 **fail-open 还是 fail-closed**？→ 计费限额必须 fail-closed，防滥用可以 fail-open。

**关键估算**
```
200 实例 × 5,000 QPS = 100 万 QPS 总入口
集中式：每请求 1 次 Redis = 100 万 Redis QPS ⇒ 8–10 分片，且每请求 +0.5 ms
本地令牌桶 + 1 s 周期同步：Redis 侧 200 QPS（降 5,000×），代价是最坏超发 ≈ 一个同步周期的配额量
限流键基数 100 万 × 100 B = 100 MB ⇒ 单实例装得下，基数才是选型的支配变量
```

**核心设计**
```
               ┌── 每 1 s 上报实际用量 / 领取下一秒配额 ──┐
 请求 ─▶ 网关实例 i                                      ▼
           └─ 本地令牌桶(rate_i, burst_i)  ──▶  Quota Coordinator (Redis + Lua)
                   ├─ 有令牌 → 放行                        │ 按各实例上一秒用量份额
                   └─ 耗尽  → 429 + Retry-After            └─ 重算 rate_i（不是均分！）
 降级：Coordinator 不可达 → 退回静态 rate/N 继续本地判定，绝不整体拒绝
```
- 选**令牌桶**：天然支持 burst，且天然支持"一次取 N 个令牌"（TPM/token 限流必需）。滑动窗口日志精确但内存 O(请求数)，只用于低基数高价值场景。
- 配额按**上一秒实际用量份额**再分配，不是均分 —— 流量倾斜（80% 打到 3 个实例）时均分会误杀 60% 合法请求。429 必须带 `Retry-After` 与 `X-RateLimit-Remaining/Reset`，否则客户端立刻重试，限流器变成放大器。

**面试的胜负手**
> 分布式限流的正确答案不是选算法，是**把"精确"降级成"有界误差"，换掉每请求一次的网络往返**。误差上界是一个同步周期的量，可以写进 SLA；而集中式方案让限流器成为整个网关可用性的天花板 —— **限流器的可用性必须严格高于它保护的服务**，这一条比精确性重要得多。
> The right answer to distributed rate limiting isn't picking an algorithm — it's trading exactness for a bounded error so you can drop one network round trip per request. The error is bounded by one sync interval, and that's a number you can actually put in an SLA. A centralized counter caps the whole gateway's availability at the limiter's own, and the limiter has to be strictly more available than the thing it protects. That matters far more than precision.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 时钟漂移 | 令牌按**单调时钟**补充，不用墙钟；跨实例只交换用量计数，不交换时间戳 |
| 突发流量跨窗口边界 | 固定窗口可被 2× 速率穿过；滑动窗口计数器（前窗加权）修掉 90%，内存仍 O(1) |
| 多维限流 | 分层判定，**从最便宜且最可能拒绝的维度开始**，任一维度拒绝即短路 |
| Redis 挂了 | 退回静态配额 `rate/N`，并把这件事做成**告警**而不是事后排查手段 |
| 限流 ≠ 配额 | 限流是速率（可恢复），配额是余额（不可恢复）。**并发额度不能用令牌桶** —— 进程崩溃会让计数永久泄漏，必须用带 TTL 的租约 |

**失败模式**
- 头部租户的键集中在单个 Redis 分片 → 热 key；p99 从 0.5 ms 涨到 5 ms 就是该切本地桶的信号。
- fail-open 在计费场景 = 免费送算力；fail-closed 在防滥用场景 = 自伤。**必须分维度配置**，一刀切必错。

**常见错误答法**
- ❌「每请求 `INCR` 一次 Redis，超了就拒」—— 能跑，但它精确地给出"没做过 100 万 QPS"的信号。
- ❌ 只讲四种算法对比表、不讲分布式部分 —— 这是**背过八股但没做过**的最强信号。

**完整版** → [`06-mock-interview.md`](../07-interview/06-mock-interview.md)（同一道题的 45 分钟英文逐字稿：算法选型、本地租约判据、Redis 降级三层，本节的每个结论在那里都有推导过程）｜[`02-question-bank.md` Q02](../07-interview/02-question-bank.md)｜[`02-llm-gateway.md` 限流与配额](02-llm-gateway.md)｜[`03-resilience-patterns.md`](../05-reliability/03-resilience-patterns.md)

---

### 3. 设计信息流 / 时间线（Design a News Feed / Timeline）

**一句话本质**：不考"推还是拉"，考**你能不能把阈值算出来**，以及大 V 这个长尾特例怎么做到不污染主链路。

**必问的 3 个澄清问题**
1. 时间线是**严格时序**还是**排序/推荐**？→ 排序会让"预计算时间线"失去意义，退化成候选召回 + 在线打分。
2. 粉丝数分布的**头部有多重**？最大 V 多少粉？→ 阈值就是从这张直方图算出来的。
3. 用户读**最新 N 条**还是要往回翻很深？→ 决定时间线存多长（800 条覆盖 99% 滚动深度）。

**关键估算**
```
2 亿 DAU × 读 10 次 = 20 亿读/天 = 23k QPS × 3 = 7 万 QPS 读；发帖 5,000 万/天 = 580 QPS 均值 × 3 ≈ 1,700 QPS 写
纯写扩散：平均 200 粉 → 34 万 写/s（可承受）；但 1 亿粉的大 V 一帖 = 1 亿次写，
         即使 10 万写/s 也要 1,000 秒才扩散完 ⇒ 主链路被一个账号堵死
时间线只存 ID+score：800 × 16 B = 13 KB/人 × 2 亿 = 2.6 TB（Redis 集群量级）
```

**核心设计**
```
发帖 ─▶ Post Svc ─▶ Posts(分片 by post_id) ─▶ Kafka: post_created
                                                │
                        ┌───────────────────────┴────────────────────────┐
              粉丝×发帖频率 低（普通用户）                    高（大 V）
              Fanout Worker：写 N 份 timeline:uid            不扩散，只写 celebrity 表
              （Redis ZSET，截断 800，仅近 7 天活跃粉丝）
读 ─▶ Feed Svc ─┬─▶ ZREVRANGE timeline:uid 0 50        ← 已扩散部分 O(1)
                ├─▶ 关注的大 V（通常 < 50 个）各取最新 20 条 → 并发归并
                └─▶ 合并 + 去重 + 权限过滤 + 排序 → 返回
```
- 阈值不是拍脑袋：`扩散写成本 × 粉丝数` 与 `读时归并成本 × 该账号粉丝的日活读次数` 的交点。**真正的判据是 `粉丝数 × 发帖频率`** —— 粉丝多但半年发一条的账号应该走扩散，粉丝少但每分钟发帖的机器人应该走拉。
- 时间线只存 ID + score，内容单独缓存后 MGET 取回（否则一条帖子被编辑/删除/改隐私要改 1 亿份拷贝）；扇出**只对近 7 天活跃粉丝做**（约占 20%），其余读时补 —— 1 亿次写变 2,000 万次，用户感知延迟不变。

**面试的胜负手**
> 大 V 不是"一个需要优化的边缘 case"，它是**必须从主链路物理分离出去的第二条路径**。混合方案的阈值我用 `粉丝数 × 发帖频率` 的成本交点算，而不是拍一个粉丝数整数。而且扇出只写给近期活跃粉丝 —— 给一个三年没登录的人写时间线是纯粹的浪费。
> A celebrity isn't an edge case you optimize — it's a second code path that has to be physically separated from the main one. I'd get the hybrid threshold from the cost crossover on `follower_count × posting_rate`, not from a hand-picked follower number. And fan-out only targets followers who've been active in the last week — writing a timeline for someone who hasn't logged in for three years is pure waste.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 用户关注了 100 个大 V | 归并退化。对这类重度用户做**预归并缓存**（TTL 30 s），把 N 次归并摊薄成一次 |
| 新关注一个人要回填吗 | 不回填（回填 = 又一次扇出）。新帖才扩散，历史靠读时归并 |
| 删帖 / 改隐私 | 时间线只存 ID，读时按当前权限过滤。**绝不去 1 亿份时间线里删** |
| 时间线 Redis 挂了 | 降级到纯读扩散：延迟 50 ms → 500 ms 但可用。这条降级路径必须常态演练 |
| 排序不是时序怎么办 | 时间线降级为"候选池"，在线打分层单独扩容；**排序特征服务的 p99 会直接支配 feed 的 p99** |

**失败模式**
- **扇出队列被大 V 堵死**：普通用户的时间线跟着延迟。必须按扇出量分优先级队列，小扇出走快车道。
- **归并的尾延迟放大**：读时拉 50 个源，p99 = 最慢那个。必须并发 + 超时 + 允许返回部分结果。ZSET 不截断则单 key 涨到几十万成员，`ZREVRANGE` 变慢且内存爆。

**常见错误答法**
- ❌「全部写扩散，简单」—— 面试官只需回一句"1 亿粉丝的账号发一条呢"，这题就结束了。
- ❌「全部读扩散，省存储」—— 7 万 QPS × 平均 200 个源 = 1,400 万次后端查询/秒。给不出这个数字说明根本没估算。

---

### 4. 设计聊天系统（Design a Chat / Messaging System）

**一句话本质**：考**有状态服务的路由**。消息存储是简单的；难的是"A 的连接在机器 7、B 的在机器 91，怎么找到 91"，以及离线时这条消息怎么不丢。

**必问的 3 个澄清问题**
1. 单聊还是群聊？群上限多少人？→ 10 人群和 10 万人群是两个系统（后者是广播，不能写扩散）。
2. 消息**必达**吗？允许乱序吗？→ 决定要不要会话内序号和客户端 ack。
3. 要**已读回执 / 正在输入 / 在线状态**吗？→ 这三个的写量常常是消息本身的 10 倍以上，是隐藏的容量炸弹。

**关键估算**
```
5,000 万 DAU × 40 条/天 = 20 亿条/天 = 23k msg/s × 3 = 7 万 msg/s 峰值
长连接：5,000 万并发 ÷ 单机 10 万连接 = 500 台接入机（每连接 30–50 KB 内存）
心跳 30 s × 5,000 万 = 170 万 心跳/s ⇒ 光心跳就是消息量的 24 倍
已读回执若逐条上报：100 人群 = 100× 写放大 ⇒ 必须按会话合并成"读到 seq=N"
```

**核心设计**
```
 App ══WSS══▶ [接入层 Gateway ×500]   只做：连接、鉴权、心跳、编解码（无业务逻辑）
                   │ 注册 uid→gw_id 到 Session Registry(Redis, TTL 60 s，心跳续期)
                   ▼
              Logic Svc ─▶ 分配 conv_seq（会话内单调递增，由单个逻辑分片发放）
                   ├─▶ Message Store（分片 by conv_id，PK = (conv_id, seq)）
                   ├─▶ 查 Registry：在线 → 直投目标 gw → 推送
                   └─▶ 离线 → Offline Queue + APNs/FCM
 客户端: GET /messages?conv_id&since_seq=N   ← 断线重连靠这条补齐，不靠推送可靠性
```
- **有序性只在会话内保证**（`(conv_id, seq)`）。跨会话全局有序既不需要也做不到。大群（> 500 人）不写扩散：只存一份会话消息，成员按 `last_read_seq` 拉，只对 @提及 的人推送。
- 三层状态 `sent → delivered → read` 各有独立 ack 通道；**已读回执必须按会话合并上报"读到 seq=1042"**，而不是每条一个回执 —— 这一条能把回执写量降两个数量级。

**面试的胜负手**
> 消息系统的可靠性不能建立在推送上。我把它拆成两条独立的路：**推送负责延迟，拉取负责正确性** —— 服务端落库并分配会话内单调 seq，客户端存 `last_seq`，重连时用 `since_seq` 补齐。这样接入层完全无状态可随时重启，而弱网、切网、后台冻结、App 崩溃这四种场景全部收敛到同一个机制上。
> I'd never build message reliability on push. Push is for latency, pull is for correctness: the server persists every message with a per-conversation monotonic seq, the client keeps its last_seq, and re-syncs with since_seq on reconnect. That keeps the connection tier stateless and restartable, and it collapses a flaky network, a WiFi-to-cellular handoff, the app getting suspended, and an outright crash into one mechanism.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 一台接入机挂了 1,000 万连接 | 客户端**指数退避 + 抖动**重连（否则同时重连打死剩余机器）；服务端接入限速；Registry 靠 TTL 自愈 |
| 怎么找到接收方的连接 | Session Registry：`uid → {gw_id, device_id}` 集合，TTL 60 s 心跳续期；多端登录是集合不是单值 |
| 多端同步 | seq 是会话级的，每设备各自维护游标，服务端存 per-device `last_read_seq` |
| 端到端加密后服务端能做什么 | 不能服务端搜索、不能内容审核、不能服务端聚合已读。**E2EE 是产品决策，不是一个加密选项** |
| 撤回 / 编辑 | 写 tombstone 事件并占一个新 seq，客户端按序应用。**不物理删除**，否则已同步的客户端无法收敛 |

**失败模式**
- **重连风暴**：一台接入机重启 → 10 万客户端同时重连 → 打死 Registry 和鉴权链路。
- **心跳成本被低估**：170 万心跳/s 的 CPU 与带宽常常超过消息本身，心跳要调到 60–180 s + TCP keepalive。**在线状态扇出爆炸**同理：一人上线通知 500 好友 × 5,000 万人，presence 必须**只在对方打开会话时按需查询**，不做主动广播。

**常见错误答法**
- ❌「用 Kafka 做投递，每个用户一个 topic」—— 5,000 万 topic 会让 Kafka 的分区元数据直接爆炸。Kafka 是管道，不是用户级信箱。
- ❌「已读回执每条一条记录」—— 群聊下 100× 写放大且零产品价值差异。给不出这个倍数，就是没算过账。

---

### 5. 设计分布式缓存（Design a Distributed Cache）

**一句话本质**：考两件事 —— 扩容时怎么不让全部 key 失效，以及**缓存挂了数据库怎么活下来**。后者才是评分点，前者是入场券。

**必问的 3 个澄清问题**
1. 缓存挂了业务能降级吗？DB 平时承担多少比例流量？→ **这决定整道题的答案**。
2. 需要 read-your-writes 吗，还是可容忍秒级陈旧？→ 决定失效策略。
3. value 大小分布？有没有 > 1 MB 的大 value？→ 大 key 是分片倾斜与 p99 毛刺的主因。

**关键估算**
```
读 100 万 QPS，命中率 95% ⇒ 未命中 5 万 QPS 打向 DB
DB 上限约 2 万 QPS ⇒ 缓存全挂时 DB 要接 100 万 QPS = 上限的 50 倍，秒级崩溃；
      且崩溃后无法预热 ⇒ 这是一个**无法自愈**的故障
热数据 500 GB ÷ 单节点 64 GB = 8 分片（+副本 16 节点）；单节点 8–15 万 QPS，也刚好要 8 分片
每物理节点 150–200 个虚拟节点，才能把负载方差压到 ±5%（无 vnode 时可达 ±40%）
```

**核心设计**
```
 Client SDK（一致性哈希环 + 本地 L1 LRU + 单飞 + 熔断）
     │ hash(key) → ring → node
     ▼
 ┌──────── 一致性哈希环（每物理节点 200 vnode）────────┐
 │ N1' N3' N2' N1'' N4' ...   移除 N2 → 仅 1/N 的 key 重映射 │
 └────────────────────────────────────────────────┘
     │ miss
     ▼
 回源信号量（semaphore，容量 = DB 可承受并发，如 200）─▶ DB
     └─ 拿不到 → 返回陈旧值 / 降级值 / 503（load shedding）
```
- 环放在**客户端**而不是代理层：少一跳（0.3 ms）；代价是拓扑要靠配置中心 + 版本号分发，切换期允许短暂双查。
- **回源路径上的信号量是这题的灵魂**：它把"缓存挂了"从"DB 崩溃"变成"部分请求降级"。写路径则是先写 DB、再**删**缓存（不是更新），彻底方案是订阅 binlog/CDC 作为唯一失效来源，见 [`02-caching`](../01-building-blocks/02-caching.md) §2。

**面试的胜负手**
> 分布式缓存最重要的设计不在缓存里，在**未命中路径上**。100 万 QPS、95% 命中率的系统，缓存全挂时 DB 要接 50 倍于上限的流量，秒级死亡，而且死后缓存永远填不满 —— 这是一个无法自愈的故障。所以回源必须有并发上限。**宁可拒绝 90% 的请求，也不能让数据库死掉：活着的数据库能把缓存重新喂满，死掉的不能。**
> The most important part of a distributed cache isn't in the cache — it's on the miss path. At a million QPS with a 95% hit rate, losing the cache entirely sends the database fifty times its capacity. It dies in seconds, and once it's dead the cache can never refill, so that failure doesn't self-heal. That's why the origin path has to be bounded by a semaphore sized to what the database can actually take. I'd rather shed ninety percent of requests than let the database die — a live database can refill the cache, a dead one can't.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 扩容时怎么不全失效 | 一致性哈希 + 200 vnode/节点，加一个节点只迁 1/N；迁移期读旧写新，几分钟窗口 |
| 热 key（单 key 50 万 QPS） | 客户端 L1 缓存 1–5 s（最有效）或 key 打散 `k#0..k#9`；**单分片上限就是 15 万 QPS，加分片解决不了单 key** |
| 大 key（10 MB value） | 阻塞单线程 → 全分片 p99 毛刺。硬限 value ≤ 100 KB，超了放对象存储、只缓存指针 |
| 缓存击穿与淘汰 | 单飞（single-flight）+ 概率提前过期（XFetch）；有稳定热点用 LFU / W-TinyLFU，LRU 会被一次全表扫描污染。见 [`02-caching`](../01-building-blocks/02-caching.md) §3、§5 |

**失败模式**
- **雪崩**：批量预热导致 TTL 同时到期。TTL 必须加抖动 `base + rand(0, 0.1×base)`。
- **缓存和锁/会话混在一个实例**：淘汰策略无法同时满足"缓存可淘汰"和"锁不能丢"，必须分实例。**迁移期拓扑不一致**同理致命：一半客户端用新环、一半用旧环 → 命中率腰斩 + 脏读，必须有拓扑版本号与双查窗口。

**常见错误答法**
- ❌「用取模分片，扩容时 rehash」—— 8 台加到 9 台，`hash % 9` 让 **89% 的 key 失效**，等价于主动制造一次缓存雪崩。最经典的送命答案。
- ❌ 花 15 分钟讲一致性哈希的实现细节 —— 那是本科知识，不是这题的区分度；面试官在等"缓存挂了怎么办"。

---

### 6. 设计打车 / 附近司机（Design a Ride-Hailing Service）

**一句话本质**：地理索引只是入场券。真正考的是**一个司机同时被两个订单匹配到怎么办**，以及派单超时后的状态收敛 —— 这是一道伪装成 LBS 题的分布式一致性题。

**必问的 3 个澄清问题**
1. **立即派单**还是**批量撮合**（攒 3–5 s 一起算）？→ 后者匹配质量高 10–20%，但延迟高、复杂度高一个量级。
2. 司机位置上报频率？→ 3 s 与 15 s 差 5 倍写量，直接决定位置存储选型。
3. 需要**真实路网 ETA** 还是直线距离够？→ 前者会让 ETA 服务支配整条链路的 p99。

**关键估算**
```
100 万在线司机 × 每 4 s 上报 = 25 万 写/s ← 全系统最大的写流量，且只需保留最新值
   ⇒ 只放内存、不落盘（为它扛 25 万 IOPS 毫无价值）；位置总量 100 万 × 100 B = 100 MB
叫车 500 万单/天 = 58 QPS × 5（早晚高峰）= 300 QPS 匹配 ← **匹配本身根本不是容量问题**
高峰市中心单格可能 5,000 司机 ⇒ 候选必须截断（Top 50 → ETA 过滤 → Top 20）
派单 15 s × 最多 3 轮 = 乘客最长等待 45 s ← 产品与架构的共同参数
```

**核心设计**
```
 司机 App ──位置(4s)──▶ Location Ingest（无状态，按 driver_id 分片）
                            └─▶ S2 cell(level 13 ≈ 1 km) → 内存 driver_index（仅最新值）
 乘客 App ──叫车──▶ Dispatch Svc
      ① 候选：中心格 + 8 邻格 → Top 50 → 并发 ETA（超时 200 ms，降级直线距离）→ Top 20
      ② 排序：ETA + 接单率 + 评分
      ③ 逐个派单：SETNX driver_lock:{did} = order_id EX 15    ← 全系统唯一的强一致点
             ├─ 抢到 → 推送，等 15 s 应答；接受 → offered→accepted，锁转为订单持有
             └─ 拒绝/超时 → DEL 锁 → 下一个候选
 状态机: created→matching→offered→accepted→arrived→in_trip→completed
         **每一个状态都有 TTL**，超时由 Scheduler 强制回退到 matching 或 cancelled
```
- 用 **S2 / H3 而不是裸 GeoHash**：GeoHash 相邻格前缀可能完全不同（边界不连续），必查 9 格；S2 是希尔伯特曲线上的区间，可做范围查询且支持多 level 自适应密度。位置可以陈旧、ETA 可以近似、排序可以不最优 —— **只有"一个司机不能同时接两单"不能近似**。

**面试的胜负手**
> 这题看起来是地理索引题，实际是**在一个最终一致的世界里守住一个强一致的不变量：任一时刻一个司机只能被一个订单持有**。我用带 TTL 的锁 + 订单状态机做这件事，且**每个状态都有超时和自动回退** —— 因为司机 App 会闪退、会没网、会就是不点。没有超时兜底的状态机会持续累积卡在 `offered` 的僵尸订单，最后只能人工清理。
> This looks like a geo-index problem, but it's really about holding one strongly consistent invariant inside an eventually consistent world: at any instant, a driver can be held by at most one order. I'd enforce that with a TTL'd lock plus an order state machine where every state has a timeout and an automatic fallback — because driver apps crash, they lose signal, and sometimes the driver just never taps. A state machine without timeouts piles up zombie orders stuck in 'offered' until someone clears them out by hand.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| GeoHash / S2 / H3 怎么选 | GeoHash 边界不连续（查 9 格）；S2 可范围查、多 level 自适应；H3 六边形邻居等距，适合需求热力与调度 |
| 市中心一格 5,000 司机 | 按密度自适应分裂 level；查询先用细 level，候选不足再放大，永远不返回全量 |
| 司机接单瞬间断网 | 锁 TTL 到期自动释放，订单回退 matching；司机重连后一律以服务端状态为准 |
| 批量撮合值不值得 | 3–5 s 窗口 + 二分图匹配可降 10–20% 空驶，但延迟 +5 s、复杂度高一档。v1 用立即派单，v2 再上 |

**失败模式**
- **僵尸订单**：某个状态漏了超时 → 订单永久卡住、司机被永久占用。每状态 TTL + 兜底扫描器，两者都要有。
- **热点城市分片倾斜**：一个超大城市占 30% 流量，分片键要用 `city_id + hash(order_id)`。**ETA 服务成为 p99 支配项**：串行调 20 次 ETA 任一慢就拖垮匹配，必须并发 + 超时 + 降级直线距离。

**常见错误答法**
- ❌「用 SQL 的 `WHERE lat BETWEEN … AND lng BETWEEN …`」—— 二维范围无法被复合 B-Tree 索引有效裁剪，100 万行也要大范围扫描。
- ❌「匹配就是找最近的司机」—— 最近 ≠ 最优（在河对岸、正在下高速），而且**完全没提独占性** —— 那才是这题唯一的评分点。

---

### 7. 设计票务系统（Design a Ticket Booking System）

**一句话本质**：库存扣减的并发正确性。三个约束互相打架 —— **不超卖、不锁死、保留能自动释放**。只解决其中一个的方案在追问下必崩。

**必问的 3 个澄清问题**
1. **座位有身份吗**（选座 vs 只买张数）？→ 前者是 N 个独立唯一资源，后者是一个计数器，并发方案完全不同。
2. **超卖能容忍吗**？→ 航空可以（overbooking 是商业模式），演唱会绝对不行。
3. 有**秒杀级峰值**吗（10 万人抢 1 万张）？→ 有的话必须在扣减之前加准入层，否则任何库存方案都会被打穿。

**关键估算**
```
热门演出开票瞬间：10 万人抢 1 万张，峰值 ≈ 10 万 × 3 次重试 / 5 s ≈ 6 万 QPS
  30 万次请求里真正需要扣减的只有 1 万次 ⇒ **约 97% 的请求注定失败**
  ⇒ 核心结论：让这 97% 在到达库存层之前就被拒绝
单行库存的并发写上限：行锁下约 500–1,000 TPS（硬物理上限，优化不了）；1 万个座位
  分散在 1 万行 → 无热点，但"总余量"若是一行，它就是死点
```

**核心设计**
```
 用户 ─▶ 准入层 Admission
          ├─ 排队令牌：全局只放 库存×3 的人进下单页（签名防伪，按进入时间 FIFO）
          ├─ "还剩多少张"走 CDN，允许陈旧 3 s；其余进虚拟等待室并给出位次与预计时间
              │ 约 3 万人（而不是 10 万）
              ▼
 ─▶ 座位保留 Hold: INSERT INTO seat_hold(seat_id PK, user_id, expire_at)
                   ← 主键唯一约束 = 唯一裁决者；冲突即失败，不重试、不加锁
              │ 成功 → hold_id，倒计时 10 min
              ▼
 ─▶ 支付（幂等键 = hold_id）→ 成功: 同一事务内 hold → booking；超时/失败: 释放
 释放的两条路（必须都有）:
   ① 惰性：读到 expire_at < now 的 hold 一律视为无效、可被覆盖   ← 正确性靠这条
   ② 主动：Scheduler 每 30 s 扫描清理                          ← 体验靠这条
```

**面试的胜负手**
> 不超卖、不锁死、能自动释放，这三件事必须用**三个不同的机制**。不超卖用数据库唯一约束或条件更新 `WHERE n > 0` —— 单条语句原子，不需要分布式锁，也就没有"持锁者崩溃"这一整类问题；不锁死用 hold 的 `expire_at` **惰性判定**，正确性只依赖读时比时间，后台扫描器挂了也不会永久锁死库存；秒杀峰值用**准入层**解决，而不是让 6 万 QPS 全部撞到库存行上。用一把分布式锁同时解决这三件事的方案，会在"持锁进程 GC 暂停 30 秒"这个追问上崩掉。
> Not overselling, not deadlocking, and releasing holds automatically are three separate problems, and they need three separate mechanisms. Not overselling comes from a unique constraint or a conditional update — `WHERE n > 0` — atomic in a single statement, no distributed lock, so the whole "what if the lock holder crashed" class of bugs never exists. Not deadlocking comes from evaluating the hold's expire_at lazily: correctness only depends on comparing timestamps at read time, so a dead sweeper can't lock inventory up forever. And the flash-sale peak gets handled at the admission layer, instead of letting 60k QPS land on one inventory row. Any design that leans on a single distributed lock for all three falls apart the moment you ask what happens when the holder GC-pauses for thirty seconds.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 为什么不用分布式锁 | 锁要正确必须配 fencing token（GC 暂停会让锁过期而持有者不知道）；唯一约束天然原子，成本还低一个量级 |
| 只买张数（不选座）怎么扣 | 单行 = 热点，~1,000 TPS 封顶。拆 100 个分桶（每桶 100 张），随机选桶扣，扣空再借桶 |
| Redis 预扣 + 异步落库行吗 | 行，但它把正确性从"DB 事务"降级成"Redis 不丢数据"，必须持久化 + 主从 + 对账，且要**显式说出这个代价** |
| 支付成功但座位已被释放 | 支付回调幂等且能"复活" hold：先查座位是否已被占，占了就自动退款 + 通知。**这条路径必须设计，不能假装不会发生**；并且每小时对 `seat / hold / 支付流水` 三方对账 —— 不对账的库存系统必然长期漂移 |

**失败模式**
- **只靠扫描器释放 hold**：扫描器 OOM 或积压 → 1 万座位卡死 → 显示售罄但实际有空座。惰性判定是唯一兜底。
- **单行库存热点**：行锁等待会占满连接池，把整个数据库拖垮，而不只是这一行慢；叠加客户端自动重试 3 次，峰值还会再翻 3 倍。

**常见错误答法**
- ❌「用 Redis 分布式锁锁座位」—— 面试官必问"持锁进程 GC 暂停 30 秒后醒来继续写呢"。没有 fencing token 就接不下去，而唯一约束根本不需要面对这个问题。
- ❌「先查余量，够就扣」—— check-then-act，并发下必然超卖。这是判断候选人有没有真写过并发代码的一道单选题。

---

### 8. 设计支付系统（Design a Payment System）

**一句话本质**：钱不能丢、不能重复扣。但真正的难点不在你的系统内部，在**你和三方渠道状态不一致时怎么收敛** —— 因为你不能回滚对方。

**必问的 3 个澄清问题**
1. 你做的是**支付网关**（接三方渠道）还是**账本系统**（内部记账）？→ 两者难点完全不同，必须先切一刀。
2. 多币种 / 结算周期？→ 决定要不要汇率快照与 T+N 结算。
3. 退款、部分退款、争议（chargeback）在范围内吗？→ 这三个是状态机上最脏的分支。

**关键估算**
```
1,000 万笔/天 = 116 TPS 均值 × 5 = 600 TPS 峰值 ⇒ **这题不是容量题**
每笔至少 5 次持久化写（请求 + 幂等记录 + 复式记账 2 行 + 状态事件）= 3,000 写/s
  ⇒ 单主库完全撑得住，不要主动引入分片和分布式事务
三方渠道 p99 = 3–30 s，超时率 0.1–1% ⇒ **每天 1 万–10 万笔状态未知**
  ⇒ 对账不是可选项，是每天要消化 10 万条的常规业务流；合规保留 7 年 ≈ 25 TB
```

**核心设计**
```
 商户 ─▶ Payment API   Idempotency-Key: 业务派生（merchant_id + merchant_order_id）
          │ ① 幂等表 INSERT（唯一索引）：已存在 → 直接返回原响应快照
          │ ② 落 payment_intent，状态 = created
          ▼
   状态机 created → processing → { succeeded → refunded/disputed | failed | UNKNOWN }
          ▼
   Channel Adapter ─▶ 三方渠道 ← 不可回滚的边界；超时/网络错误 ⇒ UNKNOWN，绝不猜测
   ┌──────┴─ 主动查询：退避 5s/30s/2m/10m/1h，最长 24 h
   ├───────── 被动回调：验签 + 事件去重 + 只允许状态机**单调前进**
   └───────── 兜底对账：T+1 拉渠道清算文件逐笔比对
                  └─▶ 差异分类 → 自动补偿 / 人工工单（必须有这条出口）
 账本：复式记账 append-only，不变量 SUM(debit) = SUM(credit)；余额是派生视图不是真相
```

**面试的胜负手**
> 支付系统里最贵的一个词是"超时"。网络超时的语义是**状态未知**，不是失败 —— 对方可能已经扣了钱。把 timeout 直接写成 failed 并让用户重试，是重复扣款的第一大来源。所以我的状态机里有一个显式的 `UNKNOWN` 状态，它只能被主动查询、渠道回调、或 T+1 对账这三条路径之一推进，**绝不能由超时本身推进**。另外分布式系统里没有 exactly-once，只有 at-least-once 投递 + 幂等处理，所以幂等键必须由业务语义派生，不能让客户端每次重试生成新 UUID。
> The most expensive word in a payment system is 'timeout'. A network timeout means unknown, not failed — the counterparty may already have moved the money. Mapping timeout to failed and letting the user retry is the number one source of double charges. So my state machine has an explicit UNKNOWN state, and it can only be advanced by an active status query, a channel webhook, or T+1 reconciliation — never by the timeout itself. And since there's no such thing as exactly-once delivery, only at-least-once plus idempotent processing, the idempotency key has to be derived from business semantics, never a fresh UUID per retry.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 幂等键存哪、存多久 | 独立幂等表 + 唯一索引，存请求指纹与响应快照，保留 ≥ 争议窗口（常见 24 个月）。**相同 key 不同 body 必须返回 409，不能返回旧结果** |
| 回调乱序 / 重放 | 验签 + 事件 ID 去重 + 状态机单调前进；回调必须可被无限重放而最终状态不变 |
| 渠道一直不给结果 | 24 h 后转人工工单 + 对账兜底；用户侧显示"处理中"而不是"失败"。退款用条件更新 `WHERE refunded + ? <= amount` 防超退，且退款本身也要独立幂等键 |
| 2PC 还是 Saga | 三方渠道不可能加入你的事务协调器 ⇒ 只能 Saga + 补偿；而**补偿本身也会失败**，所以对账层是终局收敛手段 |

**失败模式**
- **重复扣款**：客户端超时重试 + 服务端未幂等。修复代价包括退款、客诉、监管报告 —— 这类系统里最贵的 bug。
- **对账差异积压**：每天 1 万条未平差异没人处理，三个月变 90 万条且无法追溯。必须有**差异老化告警**（T+3 未平条数）。
- **状态机回退**：乱序回调把 `succeeded` 改回 `processing`，下游据此重复发货。状态转移必须白名单化。

**常见错误答法**
- ❌「用 2PC 保证一致」—— 三方渠道不会加入你的事务协调器。说这句话等于宣布没做过真实支付。
- ❌「超时就当失败，让用户重试」—— 直接制造重复扣款。本题最致命的单点错误。

---

### 9. 设计搜索自动补全（Design Search Autocomplete / Typeahead）

**一句话本质**：不是数据结构题（Trie 是本科作业），是**在 p99 < 50 ms 的预算里，怎么塞下"热度更新"和"个性化"这两个昂贵的东西**。

**必问的 3 个澄清问题**
1. 建议来自**固定词库**还是**实时查询日志**？→ 后者要一整条流式管道，是完全不同的系统。
2. 热度更新可接受的延迟：分钟级还是小时级？→ 决定流式还是批量。
3. 要**个性化**吗（按用户历史 / 地域）？→ 这一个需求会让"全局预计算"整体失效，是最大的架构分水岭。

**关键估算**
```
1 亿次搜索/天 × 每次约 4 个 typeahead 请求 = 4 亿/天 = 4,600 QPS × 3 = 1.4 万 QPS
  p99 必须 < 50 ms（人打字间隔约 200 ms）；客户端防抖 100–150 ms + 本地缓存
  ⇒ 直接砍掉 50–70% 请求，这是全题最便宜的容量优化
词库 1,000 万 query：Trie 节点带 Top-10 ≈ 1,000 万 × 10 × 8 B ≈ 800 MB
  ⇒ 全部放内存、单机可承载；**分片只为可用性和 QPS，不为容量**
全量重建 1,000 万词的 Trie 约 2–5 分钟 ⇒ 5 分钟级重建完全可行
```

**核心设计**
```
 在线（读路径，纯内存，零 IO，零网络依赖）:
   "sys" ─▶ 客户端缓存/CDN ─▶ Typeahead Svc（本地不可变 Trie 副本）
              ① 沿 Trie 走到 "sys" 节点  ② 直接读该节点预存的 Top-10（不遍历子树！）
              ③ 与个性化候选轻量融合（本地历史置顶 2–3 条）   ⇒ p99 < 5 ms
 离线（写路径，与读完全解耦）:
   查询日志 ─▶ Kafka ─▶ 5 min 窗口聚合 + 时间衰减(λ≈0.95/天)
            ─▶ 过滤（敏感词 / 低频尾巴 / 拼写噪声）
            ─▶ 构建带 Top-K 的 Trie ─▶ 打包成不可变镜像
            ─▶ 灰度分发（1 台 → 10% → 全量）→ 原子切指针，旧镜像保留可回滚
```
- **每个节点预存 Top-10**：查询是 O(前缀长度)，不是"遍历子树再排序"。多花 8 倍内存换掉 100 倍延迟 —— 这是本题唯一的算法要点。实时热点（突发事件）主镜像跟不上，单独做一个**小的实时层**（最近 5 分钟 Top-1000 词）与主结果融合；两层结构，不要试图让主 Trie 实时。

**面试的胜负手**
> 自动补全的正确架构是**把变化的东西和不变的东西彻底分开**：Trie 是不可变镜像，在线服务只读、无状态、可秒级回滚；热度更新走完全独立的离线管道，5–60 分钟的陈旧是我主动买下的代价。个性化绝不能进 Trie —— 按用户分片的 Trie 意味着上亿份索引，成本上不可能。个性化只能在**返回之后融合**：客户端本地历史置顶几条，服务端最多做到地域级（几十份镜像，不是几亿份）。
> The right architecture for typeahead is a hard split between what changes and what doesn't. The trie is an immutable snapshot, so the serving tier is read-only, stateless, and can be rolled back in seconds, while popularity flows through a completely separate offline pipeline that runs 5 to 60 minutes behind — that's staleness I'm buying on purpose. Personalization never goes inside the trie: a per-user trie means hundreds of millions of indexes, which is economically impossible. It gets merged in after retrieval — a few entries from local history pinned on top, and at most region-level snapshots on the server side, so dozens of copies, not billions.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 为什么不遍历子树排序 | 前缀 "a" 的子树有几百万节点，遍历是秒级。节点预存 Top-K，空间 ×8、延迟 ÷100 |
| 内存放不下 / 拼写纠错 | 按首 1–2 字符分片，热前缀放内存 Trie、长前缀走 KV（**长前缀天然低频**）；编辑距离 1 的候选另建索引（SymSpell / BK-tree）后融合，**不要在 Trie 上做模糊匹配** |
| 中文 / 多语言 | 必须建拼音 + 汉字双索引；CJK 无空格，"前缀"的定义本身要重新想 |
| 敏感词与合规 | 构建期和返回期**各过滤一遍**：构建期漏的靠返回期黑名单兜底，黑名单能秒级生效 |

**失败模式**
- **全量同时切镜像**：所有节点同一秒重建 → 内存翻倍 + p99 尖刺。必须滚动切换、预留 2× 内存、保留上一版可回滚。
- **热度反馈回路**：展示 → 点击 → 排更前 → 展示更多，会把一个偶然热词固化成永久第一。需时间衰减 + 探索性打散。

**常见错误答法**
- ❌「查询时从节点往下遍历收集所有词再排序」—— 前缀 "a" 就是几百万词，p99 秒级。这题全部技术含量就在"预存 Top-K"这一步。
- ❌「用 SQL 的 `LIKE 'sys%'`」—— 能走索引，但 1,000 万行 + 排序 + 1.4 万 QPS 会直接打死 DB，且完全不解决热度排序。

---

### 10. 设计对象存储 / 文件上传（Design an Object Storage / File Upload Service）

**一句话本质**：唯一的考点是**别让文件字节流经你的应用服务器**。分片、断点续传、去重，全都是这一条的推论。

**必问的 3 个澄清问题**
1. 最大文件多大？→ 5 MB 头像和 50 GB 视频是两个系统（后者必须分片 + 断点续传）。
2. 需要**服务端处理**吗（转码 / 扫毒 / 缩略图）？→ 决定要不要上传后的事件驱动管道。
3. 访问控制粒度：公开 / 私有 / 分享链接？→ 决定用预签名 URL 还是 CDN 签名。

**关键估算**
```
100 万次上传/天 × 平均 10 MB = 10 TB/天 入流量
  若流经应用层：10 TB ÷ 86,400 s = 116 MB/s ≈ 0.93 Gbps 均值，按 3× 峰值 ≈ 350 MB/s，
  而单台 4 vCPU 应用机的实际转发上限只有 100–200 MB/s ⇒ 需 3–5 台机器纯做"字节搬运"
  （还没算冗余），成本与故障面全白给
  ⇒ 预签名直传后，应用层每次上传只传 < 1 KB 的 URL，流量降到 ≈ 0
存储 10 TB/天 × 365 = 3.6 PB/年；热 5% / 温 20% / 冷 75% 分层可省 60–70%
分片：50 GB ÷ 8 MB = 6,400 片；单片失败只重传 8 MB 而不是 50 GB
```

**核心设计**
```
 ① Client ─▶ App: POST /uploads {filename,size,sha256}
      App: 校验配额与权限 → 生成 object_key → 写 metadata(status=pending)
           → 返回预签名 URL（15 min 有效，绑定 size/content-type 条件）
 ② Client ═════════ 字节直传 ═════════▶ 对象存储（S3/GCS/OSS）
           应用服务器完全不在这条路径上；分片并发 4–8，失败仅重传该片
 ③ Client ─▶ App: POST /uploads/{id}/complete {parts[etag]} → 校验 sha256 → ready
           → 发 object_created 事件 ─▶ Kafka ─▶ 转码 / 扫毒 / 缩略图 / 索引
 ④ 下载: Client ─▶ App 换取 CDN 签名 URL（短 TTL）─▶ CDN ─▶ 回源
 ── 元数据在数据库，字节在对象存储，两者靠 status 字段 + lifecycle 规则收敛 ──
```
- 元数据与数据分离必然带来不一致（客户端传完了但没调 complete）。解法是"状态 + 双兜底"：`pending` 超 24 h 由清理任务删除，**同时**在对象存储上配 lifecycle 规则自动清理未完成的 multipart。
- 秒传（去重）：客户端先算 sha256，命中就建引用、零字节上传。代价是引用计数 + 一个隐私面（能探测某文件是否已被他人上传过）。

**面试的胜负手**
> 这题只有一个真正的评分点：**文件字节永远不经过应用服务器**。10 TB/天流经应用层意味着 3–5 台机器纯做字节搬运、上传时长受 HTTP 与 LB 超时约束、且单个大文件就能打爆一台机器的内存；预签名直传把应用层流量降到每次上传约 1 KB。剩下的一切 —— 分片、断点续传、去重、转码 —— 都是这个决策的推论。**唯一的代价是元数据与数据会短暂不一致，我用 pending 状态 + 存储侧 lifecycle 规则两条路兜底。**
> There's exactly one thing being graded here: file bytes must never pass through your application servers. Pushing 10 TB a day through the app tier means three to five machines doing nothing but copying bytes, upload times bounded by HTTP and load-balancer timeouts, and one large file able to exhaust a server's memory. Presigned direct upload cuts app-tier traffic to roughly 1 KB per upload. Everything else — multipart, resumable uploads, dedup, transcoding — falls out of that one decision. The only cost is that metadata and bytes diverge for a while, and I cover that with a pending state plus a storage-side lifecycle rule.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 分片大小怎么选 | 8–16 MB。太小 → 请求数爆炸（50 GB ÷ 1 MB = 5 万请求）；太大 → 重传代价与内存占用高。S3 的 10,000 片上限是硬约束 |
| 断点续传怎么做 | 客户端记已完成分片的 ETag，重连后调 ListParts 取服务端视角只补缺片。**服务端不需要额外记状态** |
| 预签名 URL 泄露 / 超额上传 | 短 TTL（15 min）+ 绑定 content-length / content-type 条件；配额在**发券时**校验并把 size 上限写进预签名条件，存储侧直接拒超限 PUT，不要传完再校验 |
| 冷热分层 | 按 last_access 自动降级（标准 → IA → 归档）；归档层**取回延迟是分钟到小时级**，产品必须显式暴露这个等待 |

**失败模式**
- **未完成的 multipart 永久计费**：不配 lifecycle 规则，几个月后账单里会出现一大块在控制台看不到的碎片。
- **元数据与对象漂移**：定期双向对账 —— DB 有记录但对象不在（标记损坏）；对象在但 DB 无记录（孤儿对象，按 lifecycle 清）。**热点对象**：爆款文件打穿单个存储前缀（部分对象存储对单前缀有 QPS 上限），修法是 CDN + 多副本 key `obj#0..obj#9`。

**常见错误答法**
- ❌「客户端传到应用服务器，服务器再转存 S3」—— 双倍带宽、双倍延迟、应用层成为容量瓶颈。本题唯一的"立即出局"错误。
- ❌「大文件一次 PUT 传完」—— 网络抖一次就要从头再来，且 LB 的空闲超时（常见 60 s）会直接杀掉连接。

---

## 2. 把 10 题压成 4 条肌肉记忆

| 母题 | 一句话规则 | 反例（说了就掉分） | 覆盖题号 |
|---|---|---|---|
| **A · ID 与唯一性** | 唯一性由**存储层的唯一约束**裁决，应用层永不 check-then-act；幂等键必须从业务语义派生 | "先查再插"、"哈希撞了就重试"、"客户端每次重试生成新 UUID" | 1, 2, 8 |
| **B · 扇出与聚合** | 长尾（大 V / 热前缀 / 热 key）必须走**另一条物理路径**；阈值用成本交点算，不拍数字 | "全部写扩散"、"全部读扩散"、"遍历子树再排序" | 3, 5, 9 |
| **C · 并发与库存** | 削峰在**准入层**，正确性在**条件更新**，释放在**惰性过期**；三件事三个机制 | "用分布式锁保护库存"、"先查余量再扣" | 2, 6, 7, 8 |
| **D · 长连接与状态** | 推送管延迟、拉取管正确性；**每个状态都必须有超时和自动回退** | "推送保证送达"、"状态由客户端上报决定" | 4, 6 |

**跨 4 个母题共用的三条**：
- **"总量有限"的东西** → 谁是唯一裁决者？（几乎总是数据库的一个约束，不是一把锁）
- **"N 倍放大"的东西** → N 的分布长什么样？（头部长尾决定架构，均值毫无意义）
- **"依赖外部/网络"的东西** → 超时了状态是什么？（unknown ≠ failed，这条能救半数题）

---

## 3. 这 10 题共用的 5 句话（中英对照，可直接说）

| 时机 | 中文 | English |
|---|---|---|
| **第 3 分钟**<br>收范围 | "我把范围收到 X 和 Y，Z 我明确不做。如果时间允许再回来。" | "I'm scoping this to X and Y, explicitly excluding Z. We can come back if time allows." |
| **第 8 分钟**<br>数字后跟推论 | "峰值 6 万 QPS、总共 30 万次请求，但真正能成功的只有 1 万次 —— 所以问题不是库存层扛不扛得住，是怎么让那 97% 根本到不了库存层。" | "Peak is 60k QPS — 300k requests in all — and only 10k of them can ever succeed. So the problem isn't inventory throughput, it's keeping the other 97% from ever reaching the inventory layer." |
| **第 20 分钟**<br>提名深挖点 | "主链路讲完了。我想深挖 A 和 B。我认为 B 是这系统里最难的部分，我从它开始 —— 除非你想先看别的。" | "The main path is done. I'd like to go deep on A and B. I think B is the hardest part, so I'll start there — unless you'd rather see something else." |
| **每个决策后**<br>说出代价 | "代价是最坏一个同步周期的超发，约 1–3%。我接受它，因为它换掉了每请求一次的网络往返。" | "The cost is up to one sync interval of over-admission, roughly 1–3%. I accept that because it removes a network round-trip per request." |
| **最后 5 分钟**<br>撞墙条件 | "第一个撑不住的是 C，大概在 M QPS。撞墙前的信号是 [指标] 超过 [值]，我会设这条告警。" | "The first thing to break is C, at around M QPS. The leading signal is [metric] crossing [value], and that's the alert I'd set." |

**自测方法**：遮住每题的"胜负手"和"深挖点"，给自己 6 分钟讲一遍（澄清 1 分 + 估算 1 分 + 设计 2 分 + 深挖 2 分）。讲不出数字的题，就是你的缺口。10 题全过一遍约 60 分钟。

---

## 面试官会追问

1. 这道题属于哪个母题？如果我把它换成另一个题面，你的方案哪一部分不变？
2. 短链的自定义短码和自动短码撞了，谁来裁决？为什么不是应用层？
3. 缓存全挂了，你的数据库会怎么样？它能自己恢复吗？
4. 一个司机同时被两个订单派到，你怎么保证只有一个成功？持锁进程 GC 暂停 30 秒呢？
5. 库存的 hold 到期了但后台扫描器挂了，库存会永久锁死吗？
6. 支付调用三方渠道超时了，你把状态置成什么？为什么不是 failed？
7. 10 TB/天的上传流量如果流经应用层，你需要多少台机器？这些机器在做什么有价值的事？
8. 这 10 题里，哪一道你会在流量只有 1/100 时用完全不同的方案？

---

**下一篇** → [../07-interview/01-interview-framework.md](../07-interview/01-interview-framework.md)：45 分钟的时间预算与评分信号。
