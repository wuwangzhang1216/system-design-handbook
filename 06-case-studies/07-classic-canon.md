# 07 · 经典题速解集：10 道最高频题的压缩打法

> ⚠️ **这一篇是压缩速查版，不是学习材料。**
> 这 10 题各自都有一篇**完整展开版**，就在 [`08-url-shortener.md`](08-url-shortener.md) 到 [`17-object-storage.md`](17-object-storage.md)
> （编号一一对应：本篇第 N 题 = 文件 `0(7+N)`，例如第 3 题 News feed → [`10-news-feed.md`](10-news-feed.md)）。
>
> **本篇有两个用途**：
> ① **诊断性摸底** —— 只扫 §0 和十题题面，不背结论；标出认不出母题的题，然后从对应展开版学习；
> ② **临场复习** —— 深挖读完之后，面试前一天连着扫一遍，30 分钟，检查母题识别还在不在。
>
> **错误用法是把它当成学这 10 题的唯一材料。** 它每题只写 20%，剩下 80%（估算、深挖、失败模式、数字推导）
> 全在展开版里，而面试深挖阶段的 18 分钟考的正是那 80%。

> 前 6 个案例是深度题，这一篇是**广度题**：10 道最经典的通用系统设计题，每题只写面试里真正拿分的那 20%。
> 前 9 题可以复用 **4 种高频母题**；第 10 题对象存储是刻意保留的边界案例。分类的价值是复用推理，不是把所有题硬塞进四个格子。

---

## 怎么用这一篇

**正式复习时，这一篇假设你已经读过展开版。** 诊断性摸底是例外：只用它定位缺口，不把压缩结论当成已经学会。留下的 20% 是母题、必问问题和深挖入口；被砍掉的估算、推导、失败模式与演进条件都在 [`08-url-shortener.md`](08-url-shortener.md) 到 [`17-object-storage.md`](17-object-storage.md) 里。

**先对号入座，再决定从哪读**

| 你现在的状态 | 该怎么用这一篇 |
|---|---|
| **第一次见这些题** | 可先用 §0 做 10 分钟诊断，但**正式学习从 08–17 开始**。不要背这里的“胜负手” |
| 展开版读过一遍 | 用一天扫完 10 题，标出你复述不出"决定评分的那个子问题"的那几篇，回去重读它们的展开版 |
| 面试前一天 | 连着扫一遍，30 分钟，只检查一件事：母题还认不认得出来 |
| 明天就面、来不及读展开版 | 那就只读 §0 和每题的"一句话本质"+"胜负手"，其余跳过 —— 但要知道你是在赌面试官不深挖 |

**这一篇独有的主要价值是 §0 的 4 种高频母题分类。** 对象存储题是刻意保留的边界案例：它的核心是控制面、元数据面与字节数据面分离，不强塞进四类。

其余每一段，在展开版里都有更完整的版本；唯独"这 10 题其实只有 A/B/C/D 四类"这个横向视角，08–17 里没有 —— 每篇展开版只在自己那一题的范围内说话，看不到共性。而面试里真正救命的恰恰是它：**遇到一道没见过的题，你要么在 30 秒内认出它是哪一类母题、直接调用那一类的标准武器和撞墙条件，要么从零开始现想。** 背题的人换个题面就崩，认母题的人不会。

所以正确读法是**竖着读 §0 的表，横着扫 §1 的十题**：先理解 4 类母题各守什么不变量、用什么武器、在哪撞墙，再用前 9 题验证；最后用对象存储检验自己能否识别分类边界。§1 的标注是推理结果，不是要死背的标签。

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

**「题目」一列直接链到展开版**（08–17），本表因此也是这 10 篇的目录。

| # | 题目（→ 展开版） | 母题 | 决定评分的那一个子问题 | 对应章节 |
|---|---|---|---|---|
| 1 | [URL shortener](08-url-shortener.md) | A | 自定义短码与自动短码的冲突裁决 | [01-storage-engines](../01-building-blocks/01-storage-engines.md)、[02-caching](../01-building-blocks/02-caching.md) |
| 2 | [Rate limiter](09-rate-limiter.md) | A+C | 限流器自己挂了怎么办 | [03-resilience-patterns](../05-reliability/03-resilience-patterns.md) |
| 3 | [News feed](10-news-feed.md) | B | 大 V 阈值怎么算出来 | [03-messaging-and-streams](../01-building-blocks/03-messaging-and-streams.md) |
| 4 | [Chat system](11-chat-messaging.md) | D | 推送不可靠时正确性靠什么 | [04-networking-and-edge](../01-building-blocks/04-networking-and-edge.md)、[06-notification-platform](06-notification-platform.md) |
| 5 | [Distributed cache](12-distributed-cache.md) | B | 缓存挂了 DB 怎么活 | [02-caching](../01-building-blocks/02-caching.md) |
| 6 | [Ride-hailing](13-ride-hailing.md) | C+D | 司机独占性与超时收敛 | [05-consensus-and-coordination](../01-building-blocks/05-consensus-and-coordination.md) |
| 7 | [Ticket booking](14-ticket-booking.md) | C | 不超卖、不锁死、能自动释放 | [01-storage-engines](../01-building-blocks/01-storage-engines.md) |
| 8 | [Payment system](15-payment-system.md) | A+C | 三方状态不一致怎么收敛 | [02-event-driven-and-cqrs](../02-architecture-patterns/02-event-driven-and-cqrs.md)、[04-usage-based-billing](04-usage-based-billing.md) |
| 9 | [Autocomplete](16-search-autocomplete.md) | B | 热度更新与个性化的成本 | [01-storage-engines](../01-building-blocks/01-storage-engines.md)、[02-caching](../01-building-blocks/02-caching.md) |
| 10 | [Object storage](17-object-storage.md) | **边界案例** | 字节不流经应用层 | [04-networking-and-edge](../01-building-blocks/04-networking-and-edge.md) |

**用法**：45 分钟可先按 `澄清 5 / 估算 4 / 高层 10 / 深挖 18 / 收敛 5 / 转场与追问缓冲 3` 分配，再按面试官互动调整，节奏见 [`07-interview/01-interview-framework.md`](../07-interview/01-interview-framework.md)。

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
1 亿跳转/天 = 1,157 QPS 均值 × 3（峰均比）≈ 3,500 QPS 读；写 100 万/天 = 12 QPS ⇒ 读写比 100:1
10 年存量 36.5 亿条 × 500 B ≈ 1.8 TB（含索引 ×3 ≈ 5 TB）
Base62：62⁶ = 568 亿，62⁷ = 3.5 万亿 ⇒ 在本题 10 年规模与不回收策略下，7 位有充足余量
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
- 号段（segment）分配：本地自增，生成器之间无锁且不重复，发号对 DB 的写压力很低。Feistel 网络再把自增 ID 映射成不可预测整数：双射保证**自动生成码彼此不冲突**，置换降低可枚举性。若自动码与自定义 alias 共用语法空间，最终仍必须由唯一索引裁决；想承诺生成路径零冲突，就让两个命名空间语法不相交（展开版 §4.2）。
- 跳转返回 **302 而不是 301**：301 会被浏览器永久缓存，你永远收不到统计、也无法改目标或封禁恶意链接 —— 这是不可逆的。

**面试的胜负手**
> 自动短码可以用随机/哈希候选 + 唯一约束 + 有界重试；空间足够稀疏时，它简单而可靠。若写入规模让碰撞尾延迟或可枚举性成为约束，再考虑号段分配 + Feistel 置换。无论哪种，自定义码和自动码都由同一唯一约束裁决，应用层不做有竞态的“先查再插”；重试必须复用同一个创建幂等键。
> I'd never generate short codes by hashing and retrying on collision — retries make write-path tail latency unbounded, and they fight with the custom-alias namespace for the same keys. I'd use segment-based ID allocation plus a Feistel permutation: the bijection gives me zero collisions, and the permutation makes the codes unguessable, so nobody can crawl the whole namespace. Custom aliases and generated codes live in the same table under one unique index, so the database constraint is the only arbiter — the application never does check-then-insert.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 自定义码和自动码会撞吗 | 同表同唯一索引，`INSERT ... ON CONFLICT DO NOTHING`；应用层"先 SELECT 再 INSERT"在并发下必错 |
| 缓存穿透（拿随机码扫） | **这题不用布隆过滤器** —— 36.5 亿键 @1% 误判 = 4.4 GB，塞不进边缘节点。不可枚举性本身就是防线：单次命中率 0.104%，平均 962 次请求才撞到一个有效码，叠加 100 req/min/IP 限流后，采集 1% 存量要单 IP 约 667 年。再加 60 s 空值缓存就够了 |
| 热点链接（一条被转爆） | L1 进程内缓存 1–5 s，直接削掉 Redis 单分片 90% QPS；再热就推 CDN |
| 怎么过期与回收 | `expire_at` 索引 + 每日批量清理；**短码永不回收** —— 回收会让旧链接指向新目标，是安全事故 |
| 统计要精确吗 | 总点击量用 counter/聚合；独立访客数才适合 HyperLogLog。两者都应异步批量写 |

**失败模式**
- **随机码在 B-Tree 上是随机插入**，页分裂严重；写超过 5k TPS 时按 code 前 2 位分片。
- **号段服务单点**：号段表所在 DB 挂 = 新短链创建全停（跳转不受影响）。每实例预取两段（20 万个 ID）＝ 按单实例 10 万/天的份额可撑约 **2 天**；耗尽后创建接口返回 503，**绝不降级成"随便发一个码"**。

**常见错误答法**
- ❌「用 MD5 取前 7 位」—— 36 亿条下生日碰撞必然发生，你被迫处理冲突，退化成"查-重试"循环；面试官下一句必问重试上限，你答不出。
- ❌「自增 ID 直接 Base62」—— 可枚举，竞品能遍历你全部链接并从 ID 推出你的业务量。能跑，但安全 review 必被打回。

---

### 2. 设计限流器（Design a Rate Limiter）

**一句话本质**：算法只占 20% 的分。真正考的是**在 200 个实例上做分布式计数时，你愿意为精确性付多少延迟**，以及限流器自己挂了怎么办。

**必问的 3 个澄清问题**
1. 限流的**维度与基数**：per-tenant，还是 per-tenant × endpoint × model？基数 1 万还是 1 亿？→ 决定能不能集中式。
2. **超发（over-admission）**容忍度：超 1% 可接受，还是必须精确？→ 决定本地桶还是 Redis。
3. 限流器挂了 **fail-open 还是 fail-closed**？→ **正确答案是分维度，而且没有一档是真正的 fail-open**：计费/额度硬顶 fail-closed，防滥用的 per-IP 也 fail-closed 到保守静态值（攻击者是唯一受益方），保护后端的 per-tenant 降级到"缓存策略 × 1.5"。

**关键估算**
```
200 实例 × 5,000 QPS = 100 万 QPS 总入口
集中式：每请求 1 次 Redis = 100 万 Redis QPS，且每请求 +0.8 ms（p99）
  **分片数由脚本复杂度决定，不由数据量决定**：裸 `INCR` ≈ 15 万 ops/s ⇒ 7 分片；
  三层层级判定的 Lua ≈ 5 万 ops/s ⇒ **20 分片**（同样 100 MB 状态，写法差 3 倍账单）
本地令牌桶 + 1 s 周期同步：Redis 侧 200 QPS（降 5,000×），代价是最坏超发 ≈ 一个同步周期的配额量
限流键基数 100 万 × 100 B = 100 MB ⇒ **内存从来不是这题的约束**，ops/s 才是
  （结构性开销 80 B/key 是 payload 16 B 的 5 倍 —— 换算法只能改那 16 B）
```

**核心设计**
```
               ┌── 每 1 s 上报实际用量 / 领取下一秒配额 ──┐
 请求 ─▶ 网关实例 i                                      ▼
           └─ 本地令牌桶(rate_i, burst_i)  ──▶  Quota Coordinator (Redis + Lua)
                   ├─ 有令牌 → 放行                        │ 按各实例上一秒用量份额
                   └─ 耗尽  → 429 + Retry-After            └─ 重算 rate_i（不是均分！）
 降级：Coordinator 不可达 → 普通读接口可退到有硬上限的本地配额；登录、支付和管理写按风险 fail-closed
```
- 选**令牌桶**：天然支持 burst，且天然支持"一次取 N 个令牌"（TPM/token 限流必需）。滑动窗口日志精确但内存 O(请求数)，只用于低基数高价值场景。
- 配额按**上一秒实际用量份额**再分配，不是均分 —— 流量倾斜（80% 打到 3 个实例）时均分会误杀 60% 合法请求。429 必须带 `Retry-After` 与 `X-RateLimit-Remaining/Reset`，否则客户端立刻重试，限流器变成放大器。

**面试的胜负手**
> 分布式限流的正确答案不是选算法，是**把"精确"降级成"有界误差"，换掉每请求一次的网络往返**。而"本地还是集中"不是架构偏好，是一个公式：`q = limit × T / N`，即每个节点每个同步周期分到多少令牌。q < 1 必然误拒；q < 10 时光泊松抖动就有 32%；q > 100 时本地和集中已经分不出来。200 个节点、200 ms 周期，分界线正好是 10,000 rps —— 线以上走本地，线以下留在集中式（它们天然量小，集中式对它们根本不构成问题）。而集中式方案让限流器成为整个网关可用性的天花板 —— **限流器的可用性必须严格高于它保护的服务**。
> The right answer to distributed rate limiting isn't picking an algorithm — it's trading exactness for a bounded error so you can drop one network round trip per request. And the centralized-versus-local call collapses into one number: q equals limit times the sync period divided by node count — how many tokens each node gets per period. Below one, you're guaranteed to reject valid traffic; below ten, Poisson jitter alone is thirty-two percent; above a hundred, local and centralized are indistinguishable. With two hundred nodes and a two-hundred-millisecond period the line sits at ten thousand requests per second: tenants above it go local, tenants below it stay central, and they're cheap to serve centrally precisely because they're small. A centralized counter also caps the whole gateway's availability at the limiter's own, and the limiter has to be strictly more available than the thing it protects.

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
- fail-open 在计费场景 = 免费送算力且不可回收；per-IP 防滥用若 fail-open，唯一受益方就是攻击者。**必须分维度配置**，一刀切必错。**`fail-open` 的正确问法不是"开还是关"，是"降级之后的上界是多少"** —— open 的上界是 ∞，任何有限上界的降级都比它好。

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
3. 用户读**最新 N 条**还是要往回翻很深？→ 决定时间线存多长（p99 滚动深度约 200 条）。

**关键估算**
```
2 亿 DAU × 读 10 次 = 20 亿读/天 = 23k QPS × 3 = 7 万 QPS 读；发帖 5,000 万/天 = 580 QPS 均值 × 3 ≈ 1,700 QPS 写
纯写扩散：去掉大 V 后 34 万 写/s（可承受）；但 1.5 亿粉的大 V 一帖 = 1.5 亿次写，
         即使 10 万写/s 也要 1,500 秒才扩散完 ⇒ 主链路被一个账号堵 25 分钟
时间线只存 (post_id, author_id, score) 24 B：200 条 = 4.8 KB/人 × 2 亿 = 960 GB
  ⚠ 这个 24 B 有前提：必须把 zset-max-listpack-entries 从默认 128 调到 256（否则存 200 条
    默认就已经是 skiplist），且裁剪必须是确定性的（概率裁剪没有硬上界：单周期约 1/3 越界，
    而 listpack→skiplist 单向不可逆 ⇒ 概率沿时间累积，几周内几乎必然漂移过去）
    ⇒ 漂移后 80 B/member = 3.2 TB / ~$47k
  ⚠ 也别用"800 条 × 16 B = 2.6 TB"这个流传很广的算法 —— 同样的编码切换，真实值是 12.8 TB（失真 5 倍）
  被追问"200 条为什么还是 24 B"时，答案就是上面第一条。展开推导见 10-news-feed §4.2
```

**核心设计**
```
发帖 ─▶ Post Svc ─▶ Posts(分片 by post_id) ─▶ Kafka: post_created
                                                │
                        ┌───────────────────────┴────────────────────────┐
              粉丝×发帖频率 低（普通用户）                    高（大 V）
              Fanout Worker：写 N 份 timeline:uid            不扩散，只写 celebrity 表
              （Redis ZSET，确定性裁剪到 200，仅近 7 天活跃粉丝）
读 ─▶ Feed Svc ─┬─▶ ZREVRANGE timeline:uid 0 50        ← 已扩散部分 O(1)
                ├─▶ 关注的大 V（通常 < 50 个）各取最新 20 条 → 并发归并
                └─▶ 合并 + 去重 + 权限过滤 + 排序 → 返回
```
- 阈值不是拍脑袋，而且要**两个互相独立的闸门**。① **成本闸**：`push = P × αF × C_w`、`pull = αF × R × C_r`，令两边相等，**粉丝数 αF 从两边同时消掉** ⇒ `P* = R × (C_r/C_w)`，取 `C_r/C_w ≈ 6`、`R = 10` ⇒ **发帖 > 60 条/天走 pull**，与粉丝数无关（抓的是资讯机器人）。② **毒性闸**：单帖扇出不得占用超过 20% 的扇出吞吐超过 5 s ⇒ `30 万写/s × 20% × 5 s = 30 万活跃粉丝 ÷ α(20%) = 150 万总粉丝`。**粉丝数管的不是成本，是队列公平性** —— 这就是一个判据不够的原因。
- 时间线只存 `(post_id, author_id, score)`，内容单独缓存后 MGET 取回（否则一条帖子被编辑/删除/改隐私要改上亿份拷贝）；多存的那 8 字节 `author_id` 让取关/拉黑/静音全部能在**读时按当前关系过滤**，零写放大。扇出**只对近 7 天活跃粉丝做**（约占 20%），其余读时补 —— 1.5 亿次写变 3,000 万次，用户感知延迟不变。

**面试的胜负手**
> 大 V 不是"一个需要优化的边缘 case"，它是**必须从主链路物理分离出去的第二条路径**。阈值我不拍数字，我从两个互相独立的闸门算：成本闸把 push 和 pull 的成本写成等式，**粉丝数会从两边同时消掉** —— 交点只取决于发帖频率与读频率之比，落在 60 帖/天；毒性闸管的是队列公平性，单帖扇出不能占用超过 20% 的吞吐超过 5 秒，反推是 30 万活跃粉丝、按 20% 活跃率就是 150 万总粉丝。而且扇出只写给近期活跃粉丝 —— 给一个三年没登录的人写时间线是纯粹的浪费。
> A celebrity isn't an edge case you optimize — it's a second code path that has to be physically separated from the main one. I don't hand-pick the threshold; I derive it from two independent gates. The cost gate: write push cost equals pull cost and follower count cancels from both sides, so the crossover only depends on the ratio of posting rate to read rate — about sixty posts a day. The toxicity gate is about queue fairness: no single post's fan-out may hold more than twenty percent of throughput for more than five seconds, which works back to three hundred thousand active followers, or one and a half million total at a twenty percent active rate. And fan-out only targets followers active in the last week — writing a timeline for someone who hasn't logged in for three years is pure waste.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 用户关注了 100 个大 V | 归并退化。对这类重度用户做**预归并缓存**（TTL 30 s），把 N 次归并摊薄成一次 |
| 新关注一个人要回填吗 | **回填最近 30 条**（关注只有约 600 QPS，30 次 ZADD ≈ 0.2 ms，成本是噪声）。闸门在批量：一次导入通讯录关注 500 人 → 转异步 + 每人只回填 5 条 |
| 删帖 / 改隐私 | 时间线只存引用，读时按当前权限过滤。**绝不去三千万份时间线里删** —— 3,000 万次 ZREM 按 30 万写/s 要 100 秒，而合规删除的时限是秒级 |
| 时间线 Redis 挂了 | 降级到纯读扩散：延迟 50 ms → 500 ms 但可用。这条降级路径必须常态演练 |
| 排序不是时序怎么办 | 时间线降级为"候选池"，在线打分层单独扩容；**排序特征服务的 p99 会直接支配 feed 的 p99** |

**失败模式**
- **扇出队列被大 V 堵死**：普通用户的时间线跟着延迟。必须按扇出量分优先级队列，小扇出走快车道。
- **归并的尾延迟放大**：读时拉 50 个源，p99 = 最慢那个。必须并发 + 超时 + 允许返回部分结果。ZSET 不截断则单 key 涨到几十万成员，`ZREVRANGE` 变慢且内存爆。

**常见错误答法**
- ❌「全部写扩散，简单」—— 面试官只需回一句"1.5 亿粉丝的账号发一条呢"，这题就结束了。
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
                   │ 两级租约：gw:7 → alive EX 30（每台每 10 s 续一次，全局 50 次/s）
                   │           uid:bob → {gw:7, dev, conn_epoch}  无 TTL，连断时写/删
                   │ ❌ 每连接一个 TTL + 每次心跳 SETEX = 166.7 万 Redis 写/s（17 个分片纯续期）
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
| 一台接入机挂了 10 万连接 | 瓶颈是 **TLS 握手 CPU**，不是内存：8 核约 2 万握手/s/台，全局 5,000 万重连的理论下限是 5 s。客户端**指数退避 + 全抖动**；服务端入连接限速（如 2,000/s/台）；鉴权结果本地缓存 60 s |
| 怎么找到接收方的连接 | Session Registry `uid → {gw_id, device_id, conn_epoch}` 集合（多端是集合不是单值）+ **两级租约**：读时先看 `gw:7` 是否存活（本地缓存 1 s）。续期量从 166.7 万/s 降到 50/s，代价是网关被 SIGKILL 时 uid 条目最长残留 30 s —— 期间直投失败即降级为推送 + 拉取，**不影响正确性** |
| 多端同步 | seq 是会话级的，每设备各自维护游标，服务端存 per-device `last_read_seq` |
| 端到端加密后服务端能做什么 | 不能服务端搜索、不能内容审核、不能服务端聚合已读。**E2EE 是产品决策，不是一个加密选项** |
| 撤回 / 编辑 | 写 tombstone 事件并占一个新 seq，客户端按序应用。**不物理删除**，否则已同步的客户端无法收敛 |

**失败模式**
- **重连风暴**：一台接入机重启 → 10 万客户端同时重连 → 打死 Registry 和鉴权链路。
- **心跳成本被低估，但贵的不是心跳本身**：170 万心跳/s 摊到 500 台只有 3,333 次/s/台、2.1 Gbps，便宜；真正贵的是它带动的 **Registry 续期**（每次心跳一次 SETEX = 166.7 万 Redis 写/s）。解法是两级租约，不是把心跳间隔拉长。**在线状态扇出爆炸**同理：一人上线通知 500 好友 × 5,000 万人，presence 必须**只在对方打开会话时按需查询**，不做主动广播。

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
读 100 万 QPS。**目标命中率不是拍的，是从 DB 容量倒推的**：
      DB 点查硬上限 2 万 QPS × 60% 安全利用率 = 1.2 万 miss 预算
      ⇒ 命中率 ≥ 1 − 1.2万/100万 = 98.8% ⇒ 定 **99%**（稳态 miss 1 万 QPS）
      对照："95% 听起来不错"其实 miss 5 万 QPS = 硬上限的 2.5 倍，**稳态就把 DB 压在红线外**
缓存全挂时 DB 要接 100 万 QPS = 上限的 50 倍，秒级崩溃；
      且崩溃后无法预热 ⇒ 这是一个**无法自愈**的故障
热数据 500 GB ÷ 单节点 64 GB = 8 分片（+副本 16 节点）；单节点 8–15 万 QPS，也刚好要 8 分片
每物理节点 200 个虚拟节点，把负载标准差压到 ≈ 1/√200 ≈ 7%（无 vnode 时可达 ±40–50%）
      **判据是"最重的节点还在不在容量内"，不是标准差本身**
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
> 分布式缓存最重要的设计不在缓存里，在**未命中路径上**。100 万 QPS、99% 命中率的系统，如果数据库安全上限是 2 万 QPS，缓存全挂会送来约 50 倍于上限的回源流量，而且数据库倒下后缓存也无法回填。回源必须有一个可校准的全局并发上限：例如压测得到数据库安全吞吐 1.2 万 QPS、连接平均占用 15 ms，则 Little's Law 给出平均在途约 180；可先把共享数据库代理/准入层上限设为 150，保留波动余量，再用排队等待、拒绝率和 DB p99 调整。不要把 p99 延迟直接代进 Little's Law，也不要让 600 个实例各自误以为自己拥有 150 个许可。**宁可有界拒绝，也不要让真相源进入无法自愈的过载。**
> The most important part of a distributed cache is the miss path. At one million QPS and a 99% hit rate, losing the cache can send roughly fifty times the database's safe load to origin. Suppose load tests show 12,000 safe QPS and a mean connection-hold time of 15 ms: Little's Law gives about 180 requests in flight. I might start a shared database proxy or admission layer at 150, leaving headroom, then tune it from queue time, rejection rate, and database p99. I would not substitute p99 for the mean in Little's Law, and I would not give each of 600 app instances its own allowance of 150. Bounded shedding keeps the source of truth alive so the cache can recover.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 扩容时怎么不全失效 | 一致性哈希 + 200 vnode/节点，加一个节点只重映射 `1/N`。**但重映射比例 ≠ 命中率跌落**：18→19 台只重映射 5.3%，命中率却从 99% 掉到 93.8%、回源从 1 万涨到 6.2 万 QPS = 预算的 5.2 倍 —— 一次"平滑扩容"就能打死 DB。解法是迁移期**双读**（新节点 miss 时回查旧属主，额外 miss 全挡在缓存层内）；改不了 miss 路径时才分批迁 slot（每批 ≤ 0.2%、27 批、2.3 小时） |
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
2. 司机位置上报频率？→ 4 s 与 15 s 差 3.75 倍写量，直接决定位置存储选型。
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
                            └─▶ H3 res 8(边长 461 m) → 内存 driver_index（仅最新值）
 乘客 App ──叫车──▶ Dispatch Svc
      ① 候选：中心格 + k-ring(3) = 37 格 → Top 50 → 并发 ETA（超时 200 ms，降级直线距离）→ Top 20
      ② 排序：ETA + 接单率 + 评分
      ③ 逐个派单：driver_assignment 一行上的条件更新     ← 全系统唯一的强一致点
             UPDATE driver_assignment SET order_id=?, expires_at=now()+15s, epoch=epoch+1
              WHERE driver_id=? AND (order_id IS NULL OR expires_at < now());
             ├─ 影响 1 行 → 推送 offer（带 epoch），等 15 s 应答
             └─ 影响 0 行 → 没抢到。**不重试、不等待**，直接换下一个候选
 状态机: created→matching→offered→accepted→arrived→in_trip→completed
         **每一个状态都有 TTL**，超时由 Scheduler 强制回退到 matching 或 cancelled
```
- 用 **S2 / H3 而不是裸 GeoHash**：GeoHash 相邻格前缀可能完全不同（边界不连续），且格子长宽比在奇偶位间反转，6 位覆盖 2 km 半径要 40 格；S2 是希尔伯特曲线上的区间，可做范围查询；H3 邻居等距，适合热力聚合。**"中心格 + 8 邻格"只在格边长 ≈ 搜索半径时才成立** —— H3 res 8 边长 461 m，覆盖 2 km 要 k-ring(3) = 37 格。位置可以陈旧、ETA 可以近似、排序可以不最优 —— **只有"一个司机不能同时接两单"不能近似**。

**面试的胜负手**
> 这题看起来是地理索引题，实际是**在一个最终一致的世界里守住一个强一致的不变量：任一时刻一个司机只能被一个订单持有**。我用 `driver_assignment` 一行上的条件更新守它 —— 单条语句原子、数据库是唯一裁决者，所以"持锁进程 GC 暂停 30 秒"这一整类问题根本不存在；过期用 `expires_at < now()` **惰性判定**，清理器挂了也不会永久占住运力。再加一个单调递增的 `epoch` 当 fencing token：迟到 10 秒的"我接受了"会被 epoch 不匹配直接拒掉，否则它会把一个已重派的订单改回来，两个司机同时开向同一个乘客。
> This looks like a geo-index problem, but it's really about holding one strongly consistent invariant inside an eventually consistent world: at any instant, a driver can be held by at most one order. I enforce that with a conditional update on a single driver_assignment row — one atomic statement, the database is the only arbiter, so the whole "what if the lock holder GC-pauses for thirty seconds" class of bugs never exists. Expiry is evaluated lazily against expires_at, so a dead sweeper can't strand supply. And a monotonic epoch acts as a fencing token: an "I accept" that shows up ten seconds late fails the epoch check, because otherwise it would rewrite an order that's already been reassigned and send two drivers to the same rider.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| GeoHash / S2 / H3 怎么选 | GeoHash 边界不连续且长宽比反转；S2 可范围查（一个圆 → 8–12 个整数区间），是索引落在只支持范围扫描的存储里时的唯一选择；H3 六边形邻居等距，适合需求热力与调度。**覆盖半径 `R(k) ≈ 0.866 × (2k+1) × e − e`，格子数 `3k²+3k+1`** |
| 市中心一格 5,000 司机 | 按密度自适应：**先细后粗** —— 先用 res 9 查 k-ring(2)，候选不足 50 再退到 res 8。先粗后细等于取回 5,000 个对象再截断 |
| 司机接单瞬间断网 | `expires_at` 到期后被下一次条件更新顺手回收，订单回退 matching；司机重连后一律以服务端状态为准，迟到的接受靠 epoch 拒掉 |
| 批量撮合值不值得 | 3–5 s 窗口 + 二分图匹配可降 10–20% 空驶，但延迟 +5 s、复杂度高一档。v1 用立即派单，v2 再上 |

**失败模式**
- **僵尸订单**：某个状态漏了超时 → 订单永久卡住、司机被永久占用。每状态 TTL + 兜底扫描器，两者都要有。
- **热点城市分片倾斜**：一个超大城市占 30% 流量。位置索引的分片键从 `city_id` 换成 `h3_res5_cell`（天然按地理打散，超大城市再切到 res 6）；订单侧用 `hash(order_id)`。**ETA 服务成为 p99 支配项**：串行调 20 次 ETA 任一慢就拖垮匹配，必须并发 + 超时 + 降级直线距离。

**常见错误答法**
- ❌「用 SQL 的 `WHERE lat BETWEEN … AND lng BETWEEN …`」—— 二维范围无法被复合 B-Tree 索引有效裁剪，100 万行也要大范围扫描。
- ❌「用 Redis 分布式锁锁住司机」—— 面试官必问"持锁进程 GC 暂停 30 秒醒来继续写呢"、"Redis 主从异步复制、主挂了锁丢了呢"。没有 fencing token 就接不下去；条件更新根本不需要面对这一类问题。
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
 ─▶ 座位保留 Hold（**一张 seat 表，不要拆两张 —— 拆了就要跨表原子性**）:
      UPDATE seat SET hold_id=?, hold_expire_at=now()+interval '15 min'
       WHERE event_id=? AND seat_id=? AND booking_id IS NULL
         AND COALESCE(payment_state, 'none') NOT IN ('processing', 'unknown')
         AND (hold_expire_at IS NULL OR hold_expire_at < now());
      ← 单条条件更新 = 唯一裁决者；影响 0 行即抢失败，不重试、不加锁
              │ 成功 → hold_id。**用户看到 10 min 倒计时，库里存 15 min**
              │   （hold 必须 > 支付 deadline + 安全带，见下面的"支付成功但座位已释放"）
              ▼
 ─▶ 支付（幂等键 = hold_id）
      ├─ 明确成功: 同一事务内 hold → booking
      ├─ 明确拒付/取消: 标记 failed 后释放
      └─ 超时/网络错误: payment_state=UNKNOWN，冻结该 hold，不得释放或转卖
             └─ 主动查单 + webhook + 对账收敛；确认失败才释放，确认成功才出票
 释放的两条路（必须都有，但都跳过 processing / UNKNOWN）:
   ① 惰性：读到 expire_at < now 且没有在途/未知支付的 hold，才视为无效、可被覆盖 ← 正确性靠这条
   ② 主动：Scheduler 每 30 s 扫描同样的安全集合                         ← 体验靠这条
```

**面试的胜负手**
> 不超卖、不锁死、能自动释放，这三件事必须用**三个不同的机制**。不超卖用数据库唯一约束或条件更新 `WHERE n > 0` —— 单条语句原子，不需要分布式锁，也就没有"持锁者崩溃"这一整类问题；不锁死用 hold 的 `expire_at` **惰性判定**，正确性只依赖读时比时间，后台扫描器挂了也不会永久锁死库存；秒杀峰值用**准入层**解决，而不是让 6 万 QPS 全部撞到库存行上。唯一例外是支付状态未知：网络超时只说明“不知道渠道是否已扣款”，此时不能把 hold 当普通过期项释放，必须先查单/回调/对账收敛。用一把分布式锁同时解决这三件事的方案，会在"持锁进程 GC 暂停 30 秒"这个追问上崩掉。
> Not overselling, not deadlocking, and releasing holds automatically are three separate problems, and they need three separate mechanisms. Not overselling comes from a unique constraint or a conditional update — `WHERE n > 0` — atomic in a single statement, no distributed lock, so the whole "what if the lock holder crashed" class of bugs never exists. Not deadlocking comes from evaluating the hold's expire_at lazily: correctness only depends on comparing timestamps at read time, so a dead sweeper can't lock inventory up forever. The exception is an unknown payment: a network timeout does not prove the channel declined, so that hold stays fenced until a status query, webhook, or reconciliation resolves it. And the flash-sale peak gets handled at the admission layer, instead of letting 60k QPS land on one inventory row. Any design that leans on a single distributed lock for all three falls apart the moment you ask what happens when the holder GC-pauses for thirty seconds.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 为什么不用分布式锁 | 锁要正确必须配 fencing token（GC 暂停会让锁过期而持有者不知道）；唯一约束天然原子，成本还低一个量级 |
| 只买张数（不选座）怎么扣 | 单行 = 热点，~1,000 TPS 封顶。**桶数 = ceil(准入后峰值 TPS / 单行 TPS)** —— 削峰做对后是 `3,000 ÷ 1,000 = 3`，取 8 留余量；削峰没做才要 120 个。**桶越多碎片化越重**：1 万张剩 200 张时，8 桶每桶 25 张（买 4 张一次成功），100 桶每桶 2 张（几乎必然要借桶）。借桶必须按 `bucket_id` 升序遍历，随机会活锁 |
| Redis 预扣 + 异步落库行吗 | 行，但它把正确性从"DB 事务"降级成"Redis 不丢数据"，必须持久化 + 主从 + 对账，且要**显式说出这个代价** |
| 支付调用超时怎么办 | 写入显式 `UNKNOWN`，保持座位 fenced，主动查单 + webhook + 对账；**超时不能走失败释放分支**。只有渠道确认失败/取消才能释放，确认成功才能出票 |
| 支付成功但座位已被释放 | 主路径先靠 `processing/UNKNOWN` 禁止过期复用；仍遇到竞态时，支付回调要幂等并尝试在同一事务内完成 booking。若座位已经卖给别人，只能自动退款 + 通知。**这条路径必须设计，不能假装不会发生**；并且每小时对 `seat / hold / 支付流水` 三方对账 —— 不对账的库存系统必然长期漂移 |

**失败模式**
- **只靠扫描器释放 hold**：扫描器 OOM 或积压 → 1 万座位卡死 → 显示售罄但实际有空座。惰性判定是唯一兜底。
- **支付 timeout 直接释放**：渠道其实已扣款，座位又卖给第二个人 → 两人付一张票。timeout 必须进入 `UNKNOWN` 并冻结库存，不能映射成 `failed`。
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
每笔约 10 行持久化（幂等 1 + payment 1 + 状态事件 2 + 复式分录 6）= 6,000 写/s；大促 20× ⇒ 23,000 写/s
  ⇒ 单主库在 6,000 是舒适区，23,000 要提前扩容或削峰；**不要主动引入分片和分布式事务**
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
> 支付系统里最贵的一个词是"超时"。网络超时的语义是**状态未知**，不是失败 —— 对方可能已经扣了钱。把 timeout 直接写成 failed 并让用户重试，会制造重复扣款风险。所以状态机里要有显式的 `UNKNOWN`，由主动查询、渠道回调或约定周期的对账推进，不能由超时本身猜测终态。消息/数据库在各自事务边界内可以提供更强保证，但跨支付渠道的业务效果通常仍靠 at-least-once、稳定业务操作 ID、幂等状态转换与对账形成 effectively-once；随机 UUID 可以作 ID，前提是同一次业务操作的所有重试都复用它。
> The most expensive word in a payment system is 'timeout'. A network timeout means unknown, not failed — the counterparty may already have moved the money. Mapping timeout to failed and letting the user retry is the number one source of double charges. So my state machine has an explicit UNKNOWN state, and it can only be advanced by an active status query, a channel webhook, or T+1 reconciliation — never by the timeout itself. Some internal platforms provide exactly-once processing inside their transaction boundary; across a payment channel, I still use a stable operation ID, an idempotent state transition, and reconciliation. A fresh UUID per retry would represent a new operation and defeat that protection.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 幂等键存哪、存多久 | 独立幂等表 + 唯一索引，存请求指纹与响应快照，保留 ≥ 争议窗口（Visa 极端值 540 天；热层 7 天在 Postgres、冷层 540 天归档）。**相同 key 不同 body 必须返回 409，不能返回旧结果** |
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

**一句话本质**：不是数据结构题（Trie 是本科作业），是**一个必须每小时变一次的东西，怎么做成一个永远不变的东西来服务** —— 以及在 p99 < 100 ms 的预算里怎么塞下"热度更新"和"个性化"这两个昂贵的东西。

**必问的 3 个澄清问题**
1. 建议来自**固定词库**还是**实时查询日志**？→ 后者要一整条流式管道，是完全不同的系统。
2. 热度更新可接受的延迟：分钟级还是小时级？→ 决定流式还是批量。
3. 要**个性化**吗（按用户历史 / 地域）？→ 这一个需求会让"全局预计算"整体失效，是最大的架构分水岭。

**关键估算**
```
1 亿次搜索/天 × 每次约 4 个 typeahead 请求 = 4 亿/天 = 4,600 QPS × 3 = 1.4 万 QPS
  p99 < 100 ms（打字场景的感知阈值）；客户端防抖 150 ms + 本地 LRU
  ⇒ 12 请求/搜索砍到 4，**直接砍掉 2/3 的全站请求**，这是全题 ROI 最高的一次优化（还不在服务端：
     边缘请求费从 ~$27k/月 降到 ~$9k/月，省下的比整个服务端集群还多）
**延迟预算里 Trie 只占 0.05%**：RTT 20–40 ms 是支配项，Trie 前缀查询只有 50 µs
  ⇒ 优化方向永远是"把不可变镜像放得离用户更近 + 提高边缘命中率"，不是把 Trie 写得更快
词库 1,000 万 query ≈ 1,500 万 radix 节点：节点结构 60 B + 子树 ≥ 20 的节点存 top-15（120 B）
  + 字符串表 240 MB ⇒ 镜像 **2.3 GB**（全存 top-k 则 4 GB）
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
   查询日志 ─▶ Kafka ─▶ 5 min 窗口聚合 + 时间衰减(λ≈0.97/天，半衰期约 23 天)
            ─▶ 过滤（敏感词 / 低频尾巴 / 拼写噪声）
            ─▶ 构建带 Top-K 的 Trie ─▶ 打包成不可变镜像
            ─▶ 灰度分发（1 台 → 10% → 全量）→ 原子切指针，旧镜像保留可回滚
```
- **每个（子树词条数 ≥ 20 的）节点预存 Top-15**：查询是 O(前缀长度) 的 50 µs，不是"遍历子树再排序"的秒级。多花 360 MB 内存换掉 **4 个数量级**的延迟 —— 这是本题唯一的算法要点。存 15 而不是 10，是给返回期敏感词过滤和去重留冗余，否则过滤掉 3 条就只能返回 7 条。实时热点（突发事件）主镜像跟不上，单独做一个**小的实时层**（最近 5 分钟 Top-1000 词）与主结果融合；两层结构，不要试图让主 Trie 实时。

**面试的胜负手**
> 自动补全的正确架构是**把变化的东西和不变的东西彻底分开**：Trie 是不可变镜像，在线服务只读、无状态、可秒级回滚；热度更新走完全独立的离线管道，5–60 分钟的陈旧是我主动买下的代价。个性化绝不能进 Trie —— 按用户分片的 Trie 意味着上亿份索引，成本上不可能。个性化只能在**返回之后融合**：客户端本地历史置顶几条，服务端最多做到地域级（几十份镜像，不是几亿份）。
> The right architecture for typeahead is a hard split between what changes and what doesn't. The trie is an immutable snapshot, so the serving tier is read-only, stateless, and can be rolled back in seconds, while popularity flows through a completely separate offline pipeline that runs 5 to 60 minutes behind — that's staleness I'm buying on purpose. Personalization never goes inside the trie: a per-user trie means hundreds of millions of indexes, which is economically impossible. It gets merged in after retrieval — a few entries from local history pinned on top, and at most region-level snapshots on the server side, so dozens of copies, not billions.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 为什么不遍历子树排序 | 前缀 "a" 的子树有 300 万词条，遍历是秒级。节点预存 Top-K：多 360 MB，延迟 ÷10,000 |
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

**一句话本质**：对大文件上传，默认把**控制面**留在应用层、让**字节数据面**直达对象存储；小文件、需要同步检查或客户端受限时可以选择代理上传。

**必问的 3 个澄清问题**
1. 最大文件多大？→ 5 MB 头像和 50 GB 视频是两个系统（后者必须分片 + 断点续传）。
2. 需要**服务端处理**吗（转码 / 扫毒 / 缩略图）？→ 决定要不要上传后的事件驱动管道。
3. 访问控制粒度：公开 / 私有 / 分享链接？→ 决定用预签名 URL 还是 CDN 签名。

**关键估算**
```
100 万次上传/天 × 平均 10 MB = 10 TB/天 入流量；读写比 10:1 ⇒ 100 TB/天 出流量
  **进出都要算**：入 347 MB/s 峰值 + 出 3,470 MB/s 峰值 = 3,817 MB/s
  单台 4 vCPU 应用机的转发上限只有 100–200 MB/s（取 150）⇒ 26 台跑满
    ÷ 60% 目标利用率 = 43 台 → × 1.5（跨 3 AZ 且要能挂一个）= **65 台纯做字节搬运**
  ⇒ 预签名直传后应用层每次上传只过约 1 KB 控制消息 ≈ 2 GB/天 ⇒ **3 台**
  ⚠ 不要假设直传一定改变出网单价；费用取决于云厂商、地域、LB/传输处理和 CDN 路径。
     稳定收益是减少应用层带宽、连接与发布耦合，具体账单实施时重算
存储 10 TB/天 × 365 = 3.6 PB/年；热 5% / 温 20% / 冷 75% 分层可省 60–70%
分片：50 GB ÷ 8 MB = 6,400 片；单片失败只重传 8 MB 而不是 50 GB
```

**核心设计**
```
 ① Client ─▶ App: POST /uploads {filename,size,sha256}
      App: 校验配额与权限 → 生成 object_key → 写 metadata(status=pending)
           → 返回预签名 URL（15 min；绑定 key/content-type/checksum，
              大小按厂商能力签入或完成后校验）
 ② Client ═════════ 字节直传 ═════════▶ 对象存储（S3/GCS/OSS）
           应用服务器完全不在这条路径上；分片并发 4–8，失败仅重传该片
 ③ Client ─▶ App: POST /uploads/{id}/complete {parts[etag]}
           → 把 ETag 当不透明 part 标识；比对存储侧全对象 checksum
           → verifying ─▶ 扫毒通过 ─▶ ready，再发可消费事件
 ④ 下载: Client ─▶ App 换取 CDN 签名 URL（短 TTL）─▶ CDN ─▶ 回源
 ── 元数据在数据库，字节在对象存储，两者靠 status 字段 + lifecycle 规则收敛 ──
```
- 元数据与数据分离必然带来不一致（客户端传完了但没调 complete）。解法是"状态 + 双兜底"：`pending` 超 24 h 由清理任务删除，**同时**在对象存储上配 lifecycle 规则自动清理未完成的 multipart。
- 秒传（去重）：客户端 sha256 只能当候选键，不能当可信完整性证明。只有服务端已验证该对象、调用者通过权限/所有权证明时才建引用；否则仍上传并由存储侧 checksum 或受信后台验证。代价是引用计数，以及“能否探测他人是否上传过某文件”的隐私面。

**面试的胜负手**
> 这题的主轴是：**大文件字节默认不经过应用服务器**。理由不是简单的“省出网单价”，而是解除应用机器数、部署和负载均衡超时与上传成功率的耦合。在这组假设下，转发需要约 **65 台**，直传控制面约 **3 台**；这组数必须跟假设一起说，不能当通用常数。代价也不只元数据短暂不一致：还包括签名 URL 泄漏面、两套状态的对账和厂商协议差异，因此要用短 TTL、`pending/verifying/ready`、服务端校验和与 lifecycle 兜底。
> The main design axis is that large-file bytes should normally bypass the application tier. The reason is not simply egress price; it is to decouple upload success from application fleet size, load-balancer timeouts, and rolling deployments. Under these assumptions the estimate is roughly sixty-five proxy nodes versus three control-plane nodes, but the pair must travel with its assumptions. The costs include metadata-byte reconciliation, signed-URL exposure, and provider-specific upload semantics, so I use short TTLs, explicit pending/verifying/ready states, trusted checksums, and storage lifecycle cleanup.

**必答的深挖点**

| 面试官会挖 | 你要说 |
|---|---|
| 分片大小怎么选 | 在这里取 8–16 MB。太小 → 请求数爆炸（50 GB ÷ 1 MB = 5 万请求）；太大 → 重传代价与内存占用高。AWS S3 的 10,000 片上限是本案例约束；其他厂商要查各自文档 |
| 断点续传怎么做 | 客户端持久保存每片的 `(part_number, ETag/part_identifier)`，跨设备时把 manifest 存进应用元数据；重连后用分页 `ListParts` 核验存储侧已收到的片并补缺。`upload_id` 和 manifest 各解决一半问题，不能把 listing 当成唯一完成清单 |
| 预签名 URL 泄露 / 超额上传 | 短 TTL（15 min）+ 精确 key/content-type/checksum。若厂商允许把 `Content-Length` 签入 PUT 就使用；否则完成后 HEAD 校验并隔离/删除超限对象。`content-length-range` 是 POST policy 条件，不是通用 PUT 参数 |
| 冷热分层 | 按 last_access 自动降级（标准 → IA → 归档）；归档层**取回延迟是分钟到小时级**，产品必须显式暴露这个等待 |

**失败模式**
- **未完成的 multipart 永久计费**：不配 lifecycle 规则，几个月后账单里会出现一大块在控制台看不到的碎片。
- **元数据与对象漂移**：定期双向对账 —— DB 有记录但对象不在（标记损坏）；对象在但 DB 无记录（孤儿对象，按 lifecycle 清）。**热点对象**：爆款文件打穿单个存储前缀（部分对象存储对单前缀有 QPS 上限），修法是 CDN + 多副本 key `obj#0..obj#9`。

**常见错误答法**
- ❌「不看文件大小和处理要求，一律经应用层代理」—— 大文件会把应用扩展维度变成带宽和长连接；若确实代理，必须证明流式转发、连接排空、超时、重试和容量都能承受。
- ❌「大文件一次 PUT 传完」—— 网络抖动后通常要重传整对象，也缺少可靠的分片级恢复；代理链上的 idle/request timeout 还要逐层核对。持续传输不会仅因为 60 秒 idle timeout 自动断开，不能把 idle timeout 错算成固定文件大小上限。

---

## 2. 把前 9 题归约成 4 条肌肉记忆，再保留 1 个边界案例

| 母题 | 一句话规则 | 反例（说了就掉分） | 覆盖题号 |
|---|---|---|---|
| **A · ID 与唯一性** | 唯一性由**存储层的唯一约束**裁决，应用层永不 check-then-act；幂等键必须稳定标识一次业务操作（可确定性派生，也可首次生成后持久复用） | "先查再插"、"哈希撞了就重试"、"客户端每次重试生成新 UUID" | 1, 2, 8 |
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

**按训练路径阅读** → 回 [START-HERE](../START-HERE.md) 按所选路径继续；页尾链接只表示本目录或专章的顺读顺序。

**案例顺读下一篇** → [08-url-shortener.md](08-url-shortener.md)：本篇第 1 题的展开版 —— 发号器、100:1 读放大、以及 301 这个不可撤销的决定。

> 从 08 开始是这 10 题的**逐题展开版**（08 → 17，顺序与本篇 1 → 10 一一对应）。
> 复习时回到本篇，第一次学时去读展开版。
