# 06 · 模拟面试逐字稿：Design a Rate Limiter Service

> 一场真实节奏的 45 分钟 Senior 面试。**看的不是这个限流器怎么设计，是这 45 分钟怎么花。**
> 逐字稿里保留了停顿、自我修正、面试官的打断 —— 那些才是评分发生的地方。

---

## 0 · 本场设定

**岗位** Senior SWE（Infra/Platform），L5 ｜ **45 分钟共享白板** ｜ **面试官** 该平台网关团队现役工程师，题目来自他自己的系统 ｜ **结果** Strong Hire at L5，L6 未达（原因见文末）

**为什么用这道题当范本**：足够小，45 分钟能走完全流程；又足够深，自然带出**限流算法、分布式计数、一致性取舍、热点、多租户隔离、降级**六个考点。

阅读方式：**先遮住 `C:` 的所有台词，自己答一遍，再对照。** 直接顺读会产生"我也会"的幻觉。

---

## 1 · [00:00–06:00] 澄清：把一道开放题切成一个可设计的系统

**I:** So — design a rate limiter service for a multi-tenant API platform. Take it wherever you want.

**C:** Okay. Before I draw anything, I want to spend about five minutes pinning down the requirements, because "rate limiter" covers at least three different products and they don't share an architecture. Stop me if you'd rather I jump straight in.

**I:** Go ahead.

**C:** First one — where does this thing sit? Is it a library that every service imports, or a shared service that the API gateway calls on the request path?

**I:** It's at the gateway. Every external API call goes through it.

**C:** Good, so it's synchronous and on the critical path of 100% of traffic. That's the single most important constraint — my latency budget is tiny and my failure behavior matters more than my accuracy. Second — what are the dimensions? At minimum per-tenant, but is it also per-API-key and per-endpoint?

**I:** All three, actually. Tenants have multiple API keys. And we have expensive endpoints — search, bulk export — that need their own tighter limits.

**C:** So it's hierarchical — let me say it back: a request gets checked against a per-tenant global limit, a per-key limit, and a per-(tenant, endpoint-class) limit, and any one of them exhausted means reject. One thing that isn't obvious to me: does a rejection **consume** the other buckets? If the endpoint limit rejects, do I still decrement the tenant's global counter?

**I:** ...huh. Good question. What would you do?

**C:** I wouldn't consume. Otherwise a customer hammering one throttled endpoint silently burns the quota for their healthy traffic, and they'd see 429s on endpoints they never over-called. But it means I have to evaluate all buckets first and only commit if all pass — a two-phase check inside one atomic operation. I'll flag that as a design constraint.

> 💡 **旁注**：这一问是本场第一个明确加分点。它不是"功能澄清"，是**发现了题面里隐含的语义歧义**，而且这个歧义直接决定了算法必须支持"try-then-commit"而不是"边查边扣"。面试官反问"你会怎么做"是在测他有没有主见 —— 候选人给了立场 + 理由 + 后果三件套。
> 如果这里答 "Yeah either way is fine"，评分表上"取舍论证"这一栏就少了一次打勾机会。

**C:** Third — accuracy. How exact does the count have to be? If the limit is 1000 requests per minute and we let through 1030, is that a bug or is that fine?

**I:** Fine. What's not fine is if we let through 3000.

**C:** Okay — **over-admission tolerance ~5%, hard ceiling 2×**. And the other direction, false 429s on a customer who's under their limit? My instinct is that one is much worse: it's a visible broken promise and it generates support tickets.

**I:** Correct. False 429s are the thing that gets escalated to me.

**C:** *(writing)* Then that asymmetry drives a lot of what follows — when I'm uncertain, I lean toward allowing.

Fourth — on rejection, 429, or do we queue and shape?

**I:** 429. We don't queue.

**C:** Good, queueing at the gateway just converts a rate problem into a latency and memory problem. I'll return `429` plus `Retry-After` and the `RateLimit` / `RateLimit-Policy` headers so clients back off proactively instead of discovering the limit by hitting it.

Last one, and this one might change the whole design — is this **rate** limiting only, or do you also need **quota**? As in, "500,000 calls included in your plan per month."

**I:** *(pause)* Both. Why do you ask it that way?

**C:** Because they're different systems that people keep merging into one and then regret. Rate limiting protects **my** infrastructure — it has to be fast, it can be approximate, and if the counter is off by 3% nobody cares. Monthly quota is a **billing** artifact — it has to be auditable, it has to reconcile against the invoice, and it can tolerate 30 seconds of lag because nobody is defending anything with it.

If I put the monthly quota counter in the synchronous hot path with strong consistency, I've made my billing ledger a dependency of my API availability. I'd rather have the metering pipeline own quota asynchronously, and have the rate limiter only enforce a cheap cached "is this tenant hard-blocked" flag.

**I:** That's the answer I wanted. Assume metering exists as a separate pipeline — just tell me how the block flag gets to you.

⚠️ **常见错误版本（mid-level 在同一位置的回答）**
> **C:** "Okay, so we need to limit how many requests a user can make. I'll use Redis with an INCR and a TTL, and if the count is over the limit we return 429. Should I start drawing?"
>
> 这个回答不是"错"，是**信息量为零**：没区分 rate 和 quota、没问维度、没问精度、没问失败方向 —— 后面 40 分钟里每个设计决策都将没有依据，只能靠"一般都这么做"支撑。面试官反馈会写 *"jumped to a solution before establishing what problem he was solving"* —— 这是 L4 天花板的标准措辞。

**C:** Config plane pushes it. Let me put the summary on the board and move on —

```
┌─ REQUIREMENTS ────────────────────────────────────────────┐
│ 位置    gateway，同步，100% 流量                            │
│ 维度    tenant / api_key / (tenant × endpoint_class) 三层   │
│ 语义    all-or-nothing：全通过才扣，任一拒则不扣            │
│ 精度    over-admit ≤ 5%（硬顶 2×）；false-429 ≈ 0 ← 不对称  │
│ 拒绝    429 + Retry-After + RateLimit headers，不排队       │
│ 配额    月度 quota 走异步计量；限流器只消费一个 block 标志   │
│ 范围外  L3/L4 DDoS 防护、WAF、按 IP 的匿名限流              │
└───────────────────────────────────────────────────────────┘
```

I'm explicitly scoping out volumetric DDoS — that belongs at L4 in front of TLS termination and it's a different system. Tell me if you disagree.

**I:** Agreed, out of scope.

> 💡 **旁注**：`[05:40]` 收尾。**主动划出范围外**是评分表上"务实性"的直接信号，而且它买了保险 —— 后面如果时间不够，面试官不会因为他没讲 DDoS 而扣分。
> 另外注意最后那句 "tell me if you disagree"：它把一个单向声明变成了一次确认握手，成本 2 秒，收益是整场剩余时间的对齐。

---

## 2 · [06:00–10:00] 估算：口播算出 QPS、基数、内存

**C:** Numbers. What's the platform's traffic?

**I:** Call it 8.6 billion API calls a day. About 40,000 paying tenants.

**C:** Okay, let me do this out loud. 8.6 billion a day, a day is 86,400 seconds, so that's a clean **100,000 RPS average**. Peak-to-average for a business API with global customers... I'd guess 3× — US business-hours hump plus a few tenants running nightly batch. So **300k RPS peak**. Does that match?

**I:** Peak's closer to 280. Use 300.

**C:** Good. Now key cardinality, because that's what decides whether this fits in memory. 40,000 tenants; API keys per tenant — prod, staging, a couple per-service — call it **4**, so 160k keys; endpoint classes, **6** including a default. So: per-tenant global 40k, per-key 160k, per-(tenant × class) 240k → **~440,000 distinct counters**. That's small. Even 5× off it's small.

**I:** What if we grow to 400,000 tenants?

**C:** Then it's 4.4 million counters, and it still fits — let me show the memory math, because the memory answer depends entirely on which algorithm I pick, and I want that on the board before I choose.

```
① Token bucket / sliding-window counter  ── 每 key 存 2 个数
   payload  16 B (float tokens + int64 ts)
   Redis 实际开销 ≈ 100 B/key（key SDS + robj + dictEntry + 过期字典）
   440k keys  → ~44 MB
   4.4M keys  → ~440 MB          ← 一个 r6g.large 都用不满

② Sliding window LOG  ── 每个请求存一个时间戳
   窗口 60s × 300k RPS = 18,000,000 个在飞时间戳
   ZSET 每 entry ≈ 80–100 B（skiplist node + dict entry + member SDS）
   → 1.4 – 1.8 GB
   而且每请求 3 条命令：ZREMRANGEBYSCORE + ZADD + ZCARD
   → 900k Redis ops/s
```

**C:** So — and this is the point — sliding window log doesn't die on memory, 1.8 GB is nothing. It dies on **ops/s**. 900k operations per second means roughly 12 to 15 Redis shards doing nothing but bookkeeping, versus about 6 for a one-command-per-request counter — one Lua call per request at 300k RPS, and a script that size runs maybe 50k ops/s per shard. So it's 2 to 2.5× the infrastructure for accuracy I already established nobody needs.

Third number — latency budget. What's the platform's overall API p99?

**I:** We commit 150 milliseconds p99 for the read path.

**C:** Then I'll take **2 ms p99 and 5 ms p99.9** for the limiter, which is about 1.3% of the budget. And a **hard 5 ms timeout** — past that I stop waiting and fall back, because the one thing I refuse to build is a protection mechanism that becomes the outage.

> 💡 **旁注**：这段值钱的不是算术，是**每个数字后面跟了一句"所以呢"**。`900k ops/s → 12–15 个分片 → 4× 账单 → 换来一个没人要的精度`，这条链把算法选择变成了成本论证。面试官反馈里能引用的是这句，不是 "8.6B / 86400 = 100k"。**估算的价值在结论不在过程**，但过程必须口播，否则结论看起来像背的。

⚠️ **常见错误版本**
> **C:** "8.6 billion divided by 86400 is about 100k QPS, peak maybe 300k. Storage should be fine, it's just counters. Let me start on the design."

> 算对了，但**没有一个数字影响了后续决策**。这叫"仪式性估算"—— 面试官见得最多的一种。判定标准很简单：如果把估算整段删掉，后面的设计一个字都不用改，那这段估算就是 0 分。

---

## 3 · [10:00–18:00] 高层设计：先给单机，再扩到分布式

**C:** I'm going to build this in two steps — first a single-node version that's obviously correct, then I'll break it and fix it. It's faster than starting distributed and it makes the reasons for the complexity visible.

**Single node.** The gateway process holds an in-memory map from bucket key to a small struct:

```go
type Bucket struct {
    tokens float64   // 剩余令牌
    lastNs int64     // 上次补充的单调时钟
}
// check: 惰性补充，不用定时器
elapsed := now - b.lastNs
b.tokens = min(burst, b.tokens + elapsed*rate)
if b.tokens >= cost { b.tokens -= cost; allow } else { deny }
```

Two things worth saying. One, refill is **lazy** — computed on read instead of running a ticker over 440k buckets, so it's O(1) per request and zero when idle. Two, the whole check is about **100 nanoseconds**, an atomic on a striped lock — that's the number I'll compare everything else against. For the three-level hierarchy I evaluate all three in **dry-run** and only commit if all three pass, which is the all-or-nothing semantics we agreed on.

**I:** And this breaks when?

**C:** Immediately, at two nodes. Each node has its own counters, so with N gateway nodes a tenant gets N× their limit. At 200 gateway nodes that's not an approximation error, it's the limiter not existing.

So here's the distributed version.

```
                              ┌──────────────────────────┐
              policy stream   │ CONTROL PLANE  Postgres  │
              ~1s p99 收敛 ◀──│ 计划 / 租户覆盖 / block   │
                    │         └──────────────────────────┘
                    ▼
        ┌───────────────────────┐  EVALSHA   ┌────────────────────┐
 ──────▶│   API Gateway ×200    │  ~0.5 ms   │ Redis Cluster ×9   │
 client │  本地令牌桶 ~100 ns    │───────────▶│ hash tag {tenant}  │
        │  policy cache (30s)   │            │ AOF everysec       │
        └───────────┬───────────┘            └────────────────────┘
                    │ 用量事件，异步 10s 批
                    ▼
        ┌──────────────────────────────┐
        │ 计量 / 计费管道（月度 quota）  │
        └──────────────────────────────┘
```

Request path: gateway resolves tenant and key from the auth token — which it already did for authentication, so no extra cost — reads the policy from an in-memory cache, runs the check, then forwards or returns 429.

**Data model.** One Lua script, three keys, one round trip:

```
rl:{acme}:g            → HASH {tk, ts}   租户全局
rl:{acme}:k:ak_9f2c    → HASH {tk, ts}   单个 API key
rl:{acme}:e:search     → HASH {tk, ts}   端点类
TTL = 2 × 窗口         （空闲桶自动消失，省掉一整套 GC）
```

The `{acme}` hash tag is deliberate — it forces all three keys onto the same Redis Cluster slot so I can check them atomically in one `EVALSHA` instead of three round trips. **The cost of that decision is that a hot tenant concentrates on exactly one shard**, and I'll come back to it, because I think that's the most interesting failure in this system.

**I:** Why Lua and not a Redis module or three pipelined commands?

**C:** Pipelining doesn't give me atomicity across the dry-run and the commit — two gateways can both see "all three pass" and both commit. Lua gives me atomicity for free at the cost of ~30–50% throughput versus a raw `INCR`, which is why my ops budget assumed ~50k ops/s per shard rather than 150k. A module would be faster — that's what `redis-cell` does with GCRA — but it's an operational dependency on a custom build and I wouldn't take that in v1.

One detail: I pass `now` in from the gateway rather than calling `TIME` inside the script. Keeps the script deterministic, but makes me dependent on clock sync — NTP within ~50 ms, and I'd alert on skew, because a gateway 10 seconds in the future mints free tokens for everyone it serves.

> 💡 **旁注**：`[16:30]` 高层设计完成，用了 6 分半，符合预算。三个信号：**(a)** 先单机后分布式 —— "我知道复杂度是被逼出来的，不是默认的"；**(b)** 主动说出 hash tag 的代价并预告要回来讲，让面试官从"抓人"变成"一起解题"；**(c)** 时钟那段没人问，是他自己加的 —— **无人追问的细节是"做过 vs 读过"的最强区分器**。

**C:** That's the skeleton. I see three things worth digging into: **algorithm choice**, **how much of this can be done locally instead of hitting Redis**, and **what happens when Redis is unhappy**. I think the middle one is the hardest. Which do you want first?

**I:** Start with the algorithm, then we'll do the other two.

---

## 4 · [18:00–30:00] 深挖：算法、分布式计数、热点、Redis 故障

### 4.1 算法选择

**C:** Four candidates. Let me put them on the board with the number that decides each one.

| 算法 | 每 key 状态 | 每请求命令 | 边界行为 | 判决 |
|---|---|---|---|---|
| **Fixed window counter** | 1 int | 1 × INCR | **窗口交界可放 2× 限额** | 除非窗口 ≤ 1s，否则不用 |
| **Sliding window log** | N 个时间戳 | 3（ZREM/ZADD/ZCARD） | 精确 | 1.8 GB + 900k ops/s，为不需要的精度买单 |
| **Sliding window counter**（加权插值） | 2 int | 1 × Lua | 误差量级 0.003%（Cloudflare 公开数据，随流量形态变化） | 好，但不表达"突发额度" |
| **Token bucket** | 2 数（tokens, ts） | 1 × Lua | 突发是**显式参数** | ✅ 选它 |

**C:** I pick token bucket, and the reason is a product reason, not a technical one. On an API platform, burst allowance is something you **sell**. The contract you want to write on the pricing page is "1,000 requests per second sustained, bursts up to 5,000" — that's literally `rate` and `capacity`. With a sliding window counter I can only express one number, and then every customer running an hourly batch job files a ticket.

**I:** What's the fixed-window boundary problem, concretely? Say it in numbers.

**C:** Limit is 1000/min. Customer sends 1000 at 12:00:59 and another 1000 at 12:01:01. Both windows are legal, and my backend just took 2000 requests in two seconds — the intended rate is about 17 a second, so that's 60× over, and 2× the limit inside a single 60-second window. And it's worse than it sounds, because **every client that got a `Retry-After` pointing at the window reset arrives synchronized** — the limiter manufactures a thundering herd on its own boundary. Which is also why my `Retry-After` needs jitter: tell 5,000 clients "retry in 12 seconds" and you've scheduled an outage.

**I:** Nobody says that in an interview. Keep going.

**C:** One more I'll mention and not pick: **GCRA**. A leaky bucket expressed as a single float, the theoretical arrival time — `allow if TAT - τ ≤ now; TAT = max(TAT, now) + T·cost`. Half the memory of a token bucket and it smooths perfectly. I'm not picking it because that arithmetic is much harder for an on-call engineer to reason about at 3am and I'm not memory-bound at 440k keys anyway. Per-IP at 100 million keys, I'd switch.

> 💡 **旁注**："我不选 GCRA"的理由是**可运维性**而非技术劣势，且给了切换条件（1 亿 key）。**"说得出为什么不选 B" 比 "说得出为什么选 A" 值钱得多** —— 后者可能是背的，前者必须真的比较过。

⚠️ **常见错误版本**
> **C:** "I'd use sliding window log because it's the most accurate — fixed window has the boundary issue where you can get double the limit."

> 知识点全对，掉档原因是：**背出了教科书排序（log 最精确），却没把精确性和第 2 节自己算出的 900k ops/s 连起来**。面试官追问 "what does that cost you?" 就暴露了。教科书排序默认精度是目标，可在这道题里精度是**已经被明确放弃的东西**（over-admit ≤ 5%）。**自己在第 1 节澄清出来的约束，第 4 节自己不用，是最可惜的一种失分。**

### 4.2 分布式计数：集中式 vs 本地配额

**I:** Okay. Every request hitting Redis — 300k RPS. Talk me through whether that's actually fine.

**C:** Let me quantify both ends first.

**Centralized.** 300k Lua calls per second. A script this size runs maybe 50k ops/s per shard with headroom → 6 shards, and I'd run 9 across 3 AZs for AZ-loss tolerance. Latency: same-AZ Redis is p50 0.3 ms, p99 0.8 ms, p99.9 3–5 ms; cross-AZ adds 0.5–1 ms, so I need AZ-aware client routing or my p50 doubles for no reason. Cost is ~$1.3–1.5k a month, which is honestly irrelevant — nobody kills this design over $1.5k. The real cost is the other two: **every API call now has a hard synchronous dependency on Redis**, and at 300k RPS a p99.9 of 5 ms means **300 requests every second** pay 5 ms for a permission check.

**Local with periodic sync.** Central allocator hands each gateway node a **lease** of tokens every T milliseconds. Gateway spends its lease locally at 100 ns. Aggregate correctness is by construction — if the allocator only issues `limit × T` tokens per period, the total can't exceed the limit.

**I:** So the local one is strictly better?

**C:** No, and the failure isn't the one people expect. The aggregate is right; the **distribution** is wrong. Split a tenant's 1,000 rps evenly across 200 nodes and each gets 5 rps — but traffic isn't even, and one node behind a sticky LB might be seeing 200 rps of that tenant. It 429s at 5 rps while 199 nodes sit on unused tokens. So the local design's characteristic failure is **false 429s**, which is exactly the failure mode we established is unacceptable. That's not a detail; it inverts which design is safe.

**I:** How do you fix it?

**C:** Three things, in order of how much they buy:

1. **Demand-proportional leases.** Allocator distributes the next period's tokens by each node's observed share in the last period, not evenly. Handles steady skew, not sudden shifts.
2. **Out-of-band top-up.** If a node burns its lease in under 20% of the period, it asks for more immediately instead of waiting for the tick. Turns a 200 ms stall into a ~1 ms one.
3. **A hard rule about when local mode is allowed at all.** This is the one that matters:

```
本地计数只在   limit_rps × T / active_node_count ≥ 10 tokens/period   时才启用
T = 200 ms，200 个节点 → 需要 limit ≥ 10,000 rps
低于这个阈值 → 该租户强制走集中式
```

**C:** Below that threshold the per-node lease is a fraction of a token and quantization error dominates. A tenant with a 10 rps limit can never be counted locally across 200 nodes — the right answer for them is centralized, and centralized is cheap for them precisely *because* they're small. So it's **tiered**: big tenants local, small tenants central. In our numbers that's ~200 tenants out of 40,000 on the local path, carrying ~80% of the QPS. **I offload 80% of the load with 0.5% of the tenants.**

**I:** How do you know the local path's error is actually within tolerance in production?

**C:** I'd shadow it. Sample 1% of requests on the local path and *also* run them through the centralized counter, in observe-only mode, then export the disagreement rate as a first-class metric — `limiter_shadow_disagreement_ratio`, alarm at 1%. Approximation without a measurement of the approximation error is just hoping.

> 💡 **旁注**：整场最高分区，三个动作叠在一起：**(1)** 把"哪个更好"改写成"各自的特征失败是什么"，再用第 1 节澄清出的不对称性（false-429 更贵）来裁决 —— 前后呼应；**(2)** 给了一条**可执行判据**（`limit × T / nodes ≥ 10`）而不是"看情况"；**(3)** 给了一个**测量近似误差的机制**。第 3 点绝大多数候选人想不到 —— 只有被误差咬过的人才会先去建那个 metric。

### 4.3 热点租户

**I:** You flagged the hash tag earlier. Cash that check.

**C:** Right. Our biggest tenant is what fraction of platform traffic?

**I:** One tenant is about 35% at peak.

**C:** So 105k RPS, and because I hash-tagged all their buckets to `{acme}`, that's **105k ops/s landing on one shard** while the other eight are at 25k. That single shard is at or past its ceiling, and the blast radius is every tenant that shares it.

Two fixes. **Key splitting**: drop the hash tag for hot tenants, split each bucket into 16 sub-buckets `rl:acme:g:{0..15}`, each with limit/16, pick by `hash(request_id) % 16`. Spreads fine. Costs: three round trips instead of one, plus **up to ~5% of the limit is stranded** in whichever sub-buckets got unlucky, because you can't borrow across shards cheaply. So the customer's effective limit is quietly 95% of what they bought.

**Promote to local mode.** And here's the thing — the hot tenant is the *easy* case for local counting. Their limit is 150k rps; across 200 nodes at T=200 ms that's 150 tokens per node per period, so quantization error is negligible. **The tenant that breaks the centralized design is the one the local design handles best**, and that's not a coincidence — both properties come from "high volume." So hot tenants go local, and the hash-tag hotspot stops existing rather than getting mitigated.

**I:** How do you detect one? You can't reconfigure by hand.

**C:** Each gateway keeps a **Count-Min Sketch** over tenant→ops — fixed memory, a few hundred KB, no per-tenant allocation — and exports its top 100 every 10 seconds. Controller aggregates, and:

```
promote  : 持续 60 s > 5,000 rps
demote   : 持续 10 min < 2,000 rps      ← 迟滞，防止抖动
```

The gap between 5k and 2k is deliberate. Without hysteresis a tenant oscillating around one threshold flaps between modes, and every flip is a brief window where both paths are counting and you over-admit.

**I:** What's the cost of a flip?

**C:** During the transition both the local lease and the central bucket are live, so worst case you admit roughly 2× for one sync period — 200 ms. I'd accept that: it's inside the 2× hard ceiling we agreed, it's bounded in time, and it's rare because of the hysteresis. If it weren't acceptable I'd do a two-phase switch with a drain, but that's a lot of machinery for 200 ms.

### 4.4 Redis 挂了

**I:** Redis cluster goes down. Now what?

**C:** Depends which "down," and the distinction matters more than the answer.

**Clean failure** — connection refused, node gone. Easy: circuit breaker opens, I fall back. Detection in under a second.

**Latency failure** — Redis is up but p99 went from 0.8 ms to 80 ms because someone ran `KEYS` or a big key is being serialized. **This is the dangerous one**, because a naive client just waits, gateway worker threads pile up on the limiter, and the thing whose entire job is protecting the platform becomes the reason the platform is down. I've seen this pattern more than the clean-failure one.

So the client config is non-negotiable:

```
超时        5 ms 硬超时（不是 500 ms，不是"默认"）
并发上限    每网关 200 个在飞请求的信号量，满了直接走降级路径，不排队
熔断        10 s 窗口内错误率 > 20% 或超时率 > 5% → 打开
半开        每 2 s 放 1 个探针
```

**I:** And the fallback is fail open?

**C:** Not open. **Degraded.** "Fail open" on a rate limiter means during a Redis incident every tenant has an infinite limit, which is when you least want that — Redis being sick usually correlates with the platform being under load. Three layers instead:

```
L1  本地缓存的最后已知租户配额 × 1.5 安全系数
    → 总放行 ≈ 1.5 × limit，而不是 ∞
L2  每网关静态上限（如 2,000 rps/节点，与租户无关）
    → 后端受保护，与限流器正确性完全解耦
L3  预先约定的 shed 顺序：先砍 batch/低优先级流量，再砍交互式
```

L2 is the important one. It's a dumb static number in the gateway config with no dependencies at all — it's wrong for any individual tenant, but it guarantees the backend never sees more than `2,000 × 200 = 400k RPS` no matter what happens upstream. **A protection layer must have a mode where it depends on nothing.**

**I:** What about persistence — do you care if Redis loses the counters?

**C:** Barely. The state is at most one window old and it self-heals within a window. I'd run AOF `everysec` so a restarted node isn't blank, but I would absolutely **not** run `appendfsync always` — paying a durability cost per request for data with a 60-second half-life is a straight loss. And I'd turn `cluster-require-full-coverage` off, so losing one shard degrades the tenants hashed to it instead of taking the whole keyspace offline. For this workload availability beats consistency, and it's the one place in this design where I'd genuinely take AP.

> 💡 **旁注**：`[29:40]`。关键动作是**拒绝了"fail open"这个标准答案**并给出分层降级。"限流器 fail open" 是教程结论，说出来不加分；**"open 是无穷大，而 Redis 生病往往和平台过载同时发生"** 这句因果才加分。另外注意他把"延迟故障"和"干净故障"分开讲、并说前者更常见 —— 这是事故经验的直接指纹。

⚠️ **常见错误版本**
> **C:** "We fail open — better to let requests through than reject valid traffic. And we'd have Redis replicas with Sentinel for failover."

> 两句都不错，但它把问题当成了一道有标准答案的题。缺的是：**没区分"挂了"和"变慢了"**（后者才是杀手）、没有超时/并发/熔断的具体参数、没意识到 fail open 的上界是无穷。面试官想听的不是"你选 open 还是 closed"，是**"降级之后系统处于什么状态"**。

---

## 5 · [30:00–36:00] 被 challenge：部分坚持

**I:** I want to push on the tiered thing. You've got a local path, a central path, a promotion controller, hysteresis, a shadow comparison. That's a lot of machinery. Redis does 300k ops/s across nine shards — you said yourself it's $1,500 a month. Why isn't "just use Redis for everything" the right answer?

**C:** *(pause)* Let me split that, because I think you're right about part of it and I don't think you're right about all of it.

**Where you're right**: for v1, centralized-only is the correct thing to ship. It's simpler, it's exactly right rather than approximately right, and the ops cost is noise. If you told me this had to be in production in six weeks with three engineers, I'd ship centralized-only and I'd argue against my own tiered design in review. I should have led with that — I built the tiered version because you asked for the interesting problem, but the honest v1 is one Lua script and a Redis cluster.

**Where I'd hold**: the argument for local isn't throughput and isn't cost — both of those you can buy your way out of. It's two things you can't. First, **p99.9 latency at 300k RPS**: a 5 ms tail on 0.1% of requests is 300 requests a second paying a 5 ms tax for a permission check that costs 100 ns to compute. Fine today; at 1M RPS it's 1,000/sec and it shows up in customer-facing dashboards.

Second — the one I actually care about — **coupling**. Centralized-only means every API call is unavailable if Redis is unavailable. I can mitigate with degraded fallback, but the fallback path is by definition the one that's never exercised. With a local tier for the top tenants, the majority of traffic keeps a *correct* limiter through a Redis incident, and the fallback surface is much smaller.

**I:** But your local tier has its own failure modes. You just spent three minutes on lease distribution being wrong.

**C:** That's fair, and it's why I'd gate it behind the threshold rather than making it the default. Let me restate the position more narrowly than I did before, because I think I over-sold it:

> **Centralized is the default. Local mode is an escape hatch that turns on for a tenant only when (a) its limit exceeds 10,000 rps, and (b) it sustains more than 5,000 rps.** That's ~200 tenants. Everything else — 39,800 tenants — never touches the local path at all.

Framed that way it's not a second architecture, it's a per-tenant flag on one code path, and if the promotion controller has a bug the blast radius is 200 tenants whose limits are wrong by a few percent, not 40,000 tenants getting 429s.

**I:** And if I said no local tier at all, ever?

**C:** Then I'd take it, and the thing I'd want in exchange is the **static per-node ceiling** — the L2 layer. That one is thirty lines of code and it's what actually protects the backend when the limiter is wrong for any reason. If I only get one of my two ideas, I take that one. The local tier is an optimization; the static ceiling is a safety property.

> 💡 **旁注**：这六分钟是本场唯一一次真实压力测试，也是 L5 和 L4 分野最清楚的地方。候选人做了五个动作：
> **①** 先承认对方对的那一半，而且承认得非常具体（"六周三个人我会反对我自己的设计"）—— 消除"他在硬撑"的嫌疑；**②** **换了论据**：原论据是吞吐和成本，被驳倒后他没有重复，改成 p99.9 和耦合 —— 重复同一个被驳倒的论据是最伤的；**③** 主动收窄主张（"我刚才 over-sold 了"），把一个架构降级成一个 feature flag，并把爆炸半径量化成 200 个租户；**④** 在被彻底否定的假设下给出让步方案，并说明留下的那个是安全属性不是优化；**⑤** 全程没说过一次 "well, it depends."
>
> ⚠️ **两个失败版本**：**软骨型** "Yeah, you're right, let's just do that." → 记的是 *"论证不是他自己的，一推就倒"*；**追问 ≠ 你错了**，80% 的情况只是"展开讲"。**硬顶型** "No, at 300k RPS you really can't do it centrally." → 重复原论据、不给数字、不承认对方合理性，记的是 *"不听人话"* —— 这个在跨面试官 debrief 里比技术漏洞更致命。

---

## 6 · [36:00–40:00] 不知道的题

**I:** Different direction. We've been looking at doing the first layer of this in eBPF — XDP, drop at the NIC before the packet ever reaches userspace. Have you done that?

**C:** No. I've read about XDP and I've never written or operated an eBPF program in production, so I'll tell you how I'd reason about it rather than pretend.

The first thing I'd check is **whether the identity I need is even available at that layer**. At XDP I have Ethernet, IP, TCP — no TLS, no HTTP headers. Tenant identity in our system lives in the `Authorization` header, inside the TLS session. So an XDP program *structurally* cannot do per-tenant limiting; it can only do per-IP or per-connection. Which means it isn't a replacement for this system, it's a **different layer with a different job** — volumetric defense. Per-IP at line rate in front of TLS termination, per-tenant in userspace after auth. Those compose fine, and per-IP is exactly what I scoped out at the top as DDoS protection.

**I:** Suppose we terminate TLS and then want the limiter in kernel space anyway.

**C:** Then my concern shifts to the counter itself. My understanding is that eBPF maps are commonly per-CPU for performance, which means a naive counter is per-core — 64 cores, 64× the limit, exactly the problem I have with 200 gateway nodes, one level down. The fixes would be the same shape: a shared map with atomics and you pay contention, or per-core leases from a userspace allocator and you accept quantization error. So I'd expect the design to rhyme with what we've been discussing, with "core" substituted for "node." What I genuinely don't know is the **constant factor** — whether an atomic on a shared BPF hash map at 300k pps costs 50 ns or 5 µs, and what it does under contention. That number decides everything and I'd refuse to guess it.

**I:** So how would you find out?

**C:** Two-week spike, one engineer. Build the smallest possible XDP program that does a fixed-window counter on a shared LRU hash map, replay a captured hour of production traffic, and measure three things: packets per second per core, CPU per Mpps versus the userspace path, and the p99 of the map update under a hot key. Then compare against the 100 ns userspace number I already have. My prior — and it is a prior, not a conclusion — is that it wins decisively for per-IP volumetric work and doesn't pay for itself for per-tenant work, because the per-tenant path needs policy lookups and hierarchical evaluation that are awkward in a verified program with a bounded instruction count.

**I:** That's roughly where we landed too.

> 💡 **旁注**：这四分钟的分数可能比某些答对的段落还高。走的是框架四步 —— **承认边界 → 第一性原理推理 → 类比迁移 → 给验证方案**，一句编造都没有。最强的一步是**用已有约束反推可行性**："XDP 看不到 TLS 里的 header，所以它结构上不可能做 per-tenant" —— 这不是知识是推理，而且把一个"我不会"转成了一个分层结论。第二强的是明确指出**自己不知道的那个具体数字**（原子操作的常数因子）并说"我拒绝猜"。**能精确定位自己无知的边界，比模糊地知道更值钱。**

⚠️ **常见错误版本**
> **C:** "Yeah, eBPF is great for this — XDP runs before the kernel network stack so it's super fast, you'd put the counters in a BPF map and drop packets over the limit. Way faster than userspace."

> 每句都是从博客上来的正确废话，且**完全没回答 per-tenant 身份怎么拿到**。面试官自己在做这个方向，两个追问就戳穿。更致命的是二阶伤害：一旦判定你在这里编，他会回头重新怀疑前面 36 分钟里所有他没能力验证的断言 —— **编造一次，污染全场**。另一个极端同样差：`"I haven't done eBPF."` 然后沉默，直接扔掉 4 分钟评估样本。

---

## 7 · [40:00–43:00] 主动收敛

**C:** We've got about five minutes. I want to close this out myself rather than get cut off — let me do a 30-second recap and then the evolution path.

**Recap.** Token bucket, because burst is a product feature and it's two numbers per key. Hierarchical check — tenant, key, endpoint class — evaluated all-or-nothing in one Lua script, hash-tagged so it's one round trip. Centralized Redis by default, nine shards across three AZs, 5 ms hard timeout, bounded concurrency, circuit breaker. A local-lease escape hatch for the ~200 tenants above 5k rps, which absorbs most of the QPS and defuses the hot-shard problem. Degraded fallback in three layers, with a static per-node ceiling at the bottom that depends on nothing. Monthly quota is not in this system — it's a metering concern, and I only consume a cached block flag.

**Evolution.**

```
v0  进程内令牌桶 + 一致性哈希把租户粘到少数节点   2 周   撑到 ~20k RPS
v1  集中式 Redis + Lua，无本地层，静态上限兜底     6 周   撑到 ~500k RPS ← 只做一版就做这版
v2  本地租约层 + 热点自动升降级 + 1% 影子对比      +6 周  撑到 ~2M RPS
v3  多 region：按历史份额切分限额，或显式接受 N× 超发
```

**The one-way door** is the **policy schema** — what a limit is allowed to be shaped like. Rate plus burst plus a dimension tuple. Once customers have contracts written against it and SDKs parsing the `RateLimit` headers, changing it is a multi-quarter migration. Everything else here — algorithm, storage, local versus central — I can swap behind that interface in a sprint. So that's where I'd spend a disproportionate amount of design time up front.

**Where it hits the wall, and the signal:**

| 撞墙点 | 大约在 | 撞墙前的信号（监控上看得见的） |
|---|---|---|
| 单分片被热租户打满 | 一个租户 > 80k rps | 分片 CPU 分布的 p99/p50 > 3 |
| Lua 脚本 CPU | 集群 > 400k ops/s | Redis `used_cpu_sys` > 70%，`latency` 直方图右移 |
| 租约量化误差 | 节点数 > 500 或限额 < 10k rps | `shadow_disagreement_ratio` > 1% |
| 策略下发扇出 | 网关 > 2,000 个 | 配置收敛 p99 > 5 s |
| 跨 region 超发 | 上第 2 个 region 当天 | 全局用量 / 各 region 之和 > 1.1 |

**I:** Good. Questions for me?

⚠️ **常见错误版本**
> *(43:30)* **I:** "We're low on time — can you wrap up?" **C:** "Sure, um — so basically we use Redis with a token bucket, and we handle hot keys by splitting them..."

> 被动收尾和主动收尾在评分上不是量级差别，是有和无的差别。框架里 *"没有收敛的设计题，评分上等同于没做完"* 是字面意思。更常见的死法是根本没到这一步 —— 深挖讲嗨了，到点，面试官没听到任何 v0/v1、撞墙条件、单向门。技术深度满分，取舍论证和务实性两栏空白。

---

## 8 · [43:00–45:00] 反问

**C:** Three, stop me when we're out of time. One — you said false 429s are what gets escalated to you. When that happens, how long does it take to figure out *which* bucket rejected a specific request? I ask because in my design the answer has to be a response header or a trace attribute naming the bucket, and I'd rather learn that from you than from an incident.

**I:** *(laughs)* Header. We added it after the second escalation. `X-RateLimit-Policy-Matched`.

**C:** Two — is the limiter owned by the gateway team, or is it a separate service other teams call? The on-call shape is different: if it's shared, you inherit every team's misconfiguration at 3am.

**I:** Gateway team owns it. That's a fight we had.

**C:** Three, more of a curiosity — with the AI endpoints, do you limit by request count or by tokens? A request is a terrible unit when one call can cost 200× another, and I'd assume you need cost-weighted buckets with post-hoc reconciliation, since you don't know the output token count until after you've served it.

**I:** That's the project I'm on right now. We should talk about it if you come onsite.

> 💡 **旁注**：三个问题分别针对**可运维性**、**组织边界**、**技术前沿**，每个都暴露判断而不只是好奇心。第三个是本场最后一次加分：它把题面往前推了一步（**从"限流"到"按成本加权限流"**），顺带说出了那个非平凡约束 —— 输出 token 数事后才知道，所以必须先扣估算值再对账。这句话让面试官记住他，而不只是通过他。
> 反面：问"团队氛围""有没有 WLB"。不是不能问，是别用这 2 分钟问 —— **这 2 分钟仍然在被评分。**

---

## 这场面试的评分拆解

维度沿用 [`01-interview-framework.md`](01-interview-framework.md) 第 4 节的六项。

| 维度 | 得分时刻 | 候选人做了什么 | 如果没做 |
|---|---|---|---|
| **问题澄清** | `[02:10]` 追问"被拒的桶还扣不扣" | 发现了题面里的语义歧义，逼出 all-or-nothing 语义 | 算法要求会少一条约束，第 4 节的 Lua 原子性就没有理由，整条论证链断掉 |
| | `[04:50]` 分开 rate 与 quota | 把计费从可用性路径上摘掉 | 月度配额进热路径 → 计费库成为 API 的依赖 → 一个 L5 级的架构错误，且后面无法补救 |
| **结构化** | `[05:40]` 白板需求卡 + 显式范围外 | 后续每个决策都能指着它论证 | 深挖时论据只能靠"一般都这么做" |
| | `[17:50]` 提名三个深挖点让面试官选 | 交出选择权，同时展示他知道哪个最难 | 深挖方向由面试官决定，可能整场都在他不擅长的区 |
| | `[40:00]` 自己启动收敛 | 完整交付 v0/v1/v2 + 单向门 + 撞墙表 | 被动收尾 → "取舍与演进"整栏为空 → 直接压到 L4 |
| **技术深度** | `[08:30]` 900k ops/s 杀掉 sliding window log | 用自己算的数否决方案，而非背排序 | 追问 "what does that cost" 时暴露是背的 |
| | `[24:00]` 本地租约的特征失败是 false-429 | 指出误差方向而不只是误差大小 | 会得出"本地更好"的错误结论 |
| | `[28:30]` 区分 Redis 挂了 vs Redis 变慢 | 延迟故障 + 超时/并发/熔断具体参数 | 只答 fail open → 与教程无差别 → 无区分度 |
| **取舍论证** | `[22:40]` `limit × T / nodes ≥ 10` 判据 | 把"看情况"变成可执行阈值 | 面试官记 *"知道有取舍，但说不出在哪一侧"* |
| | `[26:30]` 热点：既给 key splitting 也给它 5% 的浪费 | 每个方案都带价签 | 方案变成菜单，不是决策 |
| | `[35:20]` 只保一个想法时选静态上限而非本地层 | 区分"优化"与"安全属性" | 这是本场最强的单句，缺了它 L5 仍成立但不亮眼 |
| **沟通** | `[31:00]` 承认对方对的那一半 | 消除硬撑嫌疑，换论据而非重复论据 | 软骨 → *"论证不是自己的"*；硬顶 → *"不听人话"* |
| | `[33:40]` "我刚才 over-sold 了" | 主动收窄主张 | 面试官必须自己去戳，协作感消失 |
| | `[36:20]` 承认没做过 eBPF 后仍推理 4 分钟 | 边界 + 推理 + 类比 + 验证方案 | 编 → 污染全场；沉默 → 丢掉 4 分钟样本 |
| **务实性** | `[05:30]` 主动划掉 DDoS/WAF | 收范围 | 澄清阶段膨胀到 10 分钟以上，深挖时间被吃掉 |
| | `[31:20]` "六周三个人我会反对我自己的设计" | 方案与团队规模挂钩 | 分层设计看起来像炫技 |
| | `[41:10]` "只做一版就做 v1" | 明确哪一版是真正要发的 | 演进路线变成许愿单 |

**为什么没到 L6**：技术深度和取舍论证已经到位，缺的三样全在"设计之外"——
- 全程没有提过**成本单位经济**（$/百万请求、这套东西占网关总成本多少）。只在 4.2 提了一句 $1.5k 就带过了。
- 没有提**迁移**。一个真实平台已经有某种限流了，从旧的切到新的是六步剧本（双写 → 影子对比 → 按 1%/10%/50% 切 → 回滚点），这是 L6 最稳的得分区，他一个字没说。
- 没有提**组织**。谁维护策略配置？租户自助改限额吗？超限了谁批？这些是平台题的隐藏一半。

---

## 如果这是 L6 / Staff 面试，还需要额外做什么

同一道题，L6 的交付物是**这个限流器怎么落进一个已经在跑的组织**。四条，每条给一句可直接说的话。

**a) 迁移剧本，不是新建设计**
> "这个平台肯定已经有某种限流 —— 可能是 nginx 里的 `limit_req`，可能是每个服务自己写的。所以真正的工作是切换：先把新限流器以 **observe-only** 模式部署，只记录它*会*拒绝哪些请求；跑两周，把新旧决策的差异率和差异样本拿出来给最大的 20 个租户逐个 review；差异率 < 0.1% 之后按租户灰度 1% → 10% → 50% → 100%，每一档卡 48 小时。回滚是一个配置开关，10 秒生效。**这里唯一不可回滚的是我们发出去的 `RateLimit` 响应头 —— 客户端一旦开始解析它，语义就锁死了**，所以那个 schema 我要在 observe 阶段之前定稿。"

**b) 单位经济**
> "这套东西的成本是 9 个 Redis 分片约 $1.5k/月，加上网关侧约 3% 的 CPU —— 按 200 个节点算大概 $4k/月。合计每百万请求约 $0.02。对比它防住的东西：一次热租户打爆后端的事故，光计算扩容就不止这个数。这个比值我会写进设计文档，因为**两年后有人想砍掉本地层省钱的时候，需要有人能算出砍掉之后会失去什么**。"

**c) 组织与自助**
> "限额值本身必须是**声明式且自助**的。如果每次客户要提限额都需要平台组改配置，那这不是平台，是工单队列。我要的形态是：计划里定义默认值，销售可以在 CRM 里打临时覆盖，覆盖带过期时间，全部走同一个策略流。第 10 个团队接入的成本必须低于第 1 个 —— 否则这个设计是失败的，不管它的 p99 多好看。"

**d) 消灭一整类问题**
> "限流不应该出现在任何业务代码里。网关统一做，业务服务连限流这个概念都不需要知道。同理，**每个新上线的服务默认就有租户级限流**，不需要接入动作 —— 默认安全比事后审计便宜一个数量级。这条原则的代价是网关变重、变成关键路径上的单点，所以网关本身的可用性目标要比它保护的任何服务高一位数。"

**e) 风险表**（L6 的风险必须四项齐全）

| 风险 | 概率 | 影响 | 缓解 | 缓解成本 |
|---|---|---|---|---|
| 策略 schema 定错 | 中 | 一个季度的客户侧迁移 | observe 阶段前定稿 + 版本化 | 2 周设计评审 |
| 本地层误差咬到大客户 | 低 | 单个大客户的 false-429 事故 | 1% 影子对比 + 自动降级回集中式 | 1 周 |
| 限流器成为网关故障源 | 中 | 全平台不可用 | 硬超时 + 并发上限 + 静态兜底 + 季度演练 | 持续，每季度 1 天 |
| 热点检测抖动 | 中 | 200 ms 超发窗口 | 迟滞 5k/2k | 已含 |

---

## 自测：把候选人的台词遮住，你能答出几段

每题限时 60 秒，出声答，答完对照原文。**能连贯说出来 ≠ 想得到**，必须出声。

| # | 时间点 | 题 | 及格线 |
|---|---|---|---|
| 1 | `[02:10]` | 三层限额里有一层拒了，另外两层扣不扣？给立场和后果 | 不扣 + 说出它逼出 try-then-commit |
| 2 | `[04:50]` | rate limiting 和 monthly quota 为什么不能是同一个系统？ | 说出"计费成为可用性依赖" |
| 3 | `[08:00]` | 300k RPS、60s 窗口，sliding window log 要多少内存、多少 ops/s？ | 1.8 GB / 900k ops/s，且指出**是 ops/s 杀死它**而非内存 |
| 4 | `[12:30]` | 单机令牌桶为什么用惰性补充而不是定时器？ | O(1)/请求、空闲零成本 |
| 5 | `[15:00]` | Redis Cluster 的 hash tag 给你什么、收你什么？ | 一次往返 vs 热租户全压一个分片 |
| 6 | `[19:00]` | fixed window 的边界问题用数字说一遍 | 1000/min → 两秒内 2000 → 瞬时 60× 于目标速率 |
| 7 | `[19:40]` | 为什么 `Retry-After` 必须加抖动？ | 同步重试 = 自制惊群 |
| 8 | `[21:30]` | 本地租约的特征失败是超发还是误拒？为什么？ | 误拒；总量由构造保证，错的是分配 |
| 9 | `[22:40]` | 什么时候**不能**用本地计数？给公式 | `limit_rps × T / nodes < ~10 tokens/period` |
| 10 | `[26:00]` | 为什么热点租户反而是本地计数最合适的对象？ | 两个性质都来自"量大" |
| 11 | `[27:20]` | 热点升降级阈值为什么不能是同一个数？翻转的代价多大？ | 迟滞防抖；一个同步周期约 2× |
| 12 | `[28:20]` | Redis 变慢比 Redis 挂了更危险，为什么？ | 线程堆积 → 保护机制变成故障源 |
| 13 | `[29:00]` | "fail open" 错在哪？你的三层降级是什么？ | open = ∞；最后一层必须零依赖 |
| 14 | `[31:00]` | 面试官说"直接全走 Redis 不就行了"，你怎么答？ | 先承认对的一半 + **换论据** + 收窄主张 |
| 15 | `[36:30]` | XDP 为什么结构上做不了 per-tenant 限流？ | 看不到 TLS 内的 header |
| 16 | `[41:30]` | 这个设计的单向门是什么？为什么？ | 策略 schema / 对外响应头契约 |
| 17 | `[42:00]` | 给三个撞墙点和它们在监控上的信号 | 见撞墙表任意三行 |

**判读**：
- **≤ 6 段** → 你在背组件，不在设计系统。回 [`01-building-blocks/`](../01-building-blocks/) 补内功，别急着刷题。
- **7–12 段** → L4/L5 边界。缺的通常是第 8、9、12、13 这类"失败方向"的题 —— 这些答案只能从事故里长出来，或从别人的事故记录里读出来。**13–15 段** → 稳 L5。
- **16–17 段且第 14 题答得像逐字稿里那样** → 你缺的不是设计能力，是文末 L6 那四条：迁移、成本、组织、默认值。

**最后一句**：这份逐字稿里技术含量最高的一段是 4.2，但真正决定档位的是 `[30:00–36:00]` 那六分钟。**被 challenge 时的反应，是整场面试里唯一无法准备台词的部分。**

---

**全书到此结束。** 剩下的部分只能在白板前长出来 —— 找个人，把这场逐字稿的题目讲一遍。

**回到目录** → [README.md](../README.md)
**进考场前 10 分钟** → [03-cheatsheet.md](03-cheatsheet.md) 这场里出现过的每一个数字都压缩在那一页｜**说不出口时** → [04-glossary.md](04-glossary.md) 术语、[05-english-phrasebook.md](05-english-phrasebook.md) 句式｜**换一道题再练** → [02-question-bank.md](02-question-bank.md)
