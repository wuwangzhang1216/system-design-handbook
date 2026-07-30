# 05 · 英文话术库：把想清楚的东西说得像 Senior

> 这本手册前 41 篇教你**怎么想**。这一篇只教**怎么说**。
> 面试官不给你脑子里的东西打分，只给**出口的那部分**打分 —— 而非母语者平均要在这一步损失掉 30%。
> 前提：技术你已经懂了，卡的是把一个正确的判断说成一句有分量的话。

---

**怎么用**：每节 = 场景 → 可直接背诵的句式 → 中文注解（这句在传递什么信号）→ ❌ 反面例句及原因。
英文全部按**口语**写：缩写、破折号、口头连接词都是故意的，书面语说出口反而扣分。

**先看这把尺子。** 同一个技术判断，说法不同，评级差两档：

```
说服力（conviction）刻度   weak ─────────────────────────────▶ strong

L3  "I think maybe we could possibly use a cache here?"      ← 三重 hedge + 升调
L4  "We could use a cache."                                  ← 有想法，没有主人
L5  "I'd put a cache in front of this."                      ← 有主人
L5+ "I'd put a cache here, and the cost is stale reads for 60 seconds."
L6  "I'd put a cache here — 60s TTL with jitter — and it stops working
     when a tenant's working set outgrows one node. The signal is
     eviction rate going non-zero."
```

**唯一的规律**：往右走靠的是**加约束、加数字、加边界**，不是加形容词。说不出 "and the cost is…"，加多少个 `very` 都到不了 L5。

---

## 1 开场与需求澄清（clarifying requirements）

**场景**：题面刚落地。5–8 分钟，问出 6–8 个**能改变设计**的问题，且不能让人觉得你在拖。

**① 礼貌地拒绝立刻开画**

> "Before I start drawing, I'd like to pin down a few things — otherwise I'll design the wrong system very efficiently."

- "Give me five minutes on requirements. I'd rather spend them here than build the wrong thing for forty."

中文注解：`pin down` / `nail down` 是工程师之间的日常词，比 `confirm some requirements` 自然一个量级。第二句显式给出**时间预算**（five minutes），等于告诉面试官"我知道总共只有 45 分钟"—— 这是评分表「结构化」维度的直接信号。

❌ **不要说**："Can I ask you some questions?" —— 你在请求许可。他必须回一句 "Sure"，白白浪费十秒，并把你放在下级位置。澄清是你的职责，不是恩赐。

**② 七类必问，每句都要带单位、带后果**（中文版清单见 [`01-interview-framework.md`](01-interview-framework.md) 阶段 1）

| 问什么 | 怎么问（可直接说） | 这个答案会改变什么 |
|---|---|---|
| 规模 | "Roughly what scale are we at? And is that today's traffic, or where we expect to be in eighteen months?" | 要不要现在就分片 |
| 形状 | "What's the read-to-write ratio — ballpark? 10-to-1, or the other way around? I just need to know which side to optimize." | 副本 / CQRS / 缓存层级 |
| 峰值 | "How peaky is it? A smooth diurnal curve, or flash-crowd spikes like a ticket drop?" | 队列、预留容量、限流 |
| 延迟 | "What's the latency budget on the read path? p99 under 100 milliseconds, or is a second fine?" | 同步还是异步、能不能跨 region |
| 可用性 | "If this is down for ten minutes, what actually breaks for the business? Can we degrade to read-only instead of going dark?" | 多 AZ / 多 region / 降级路径 |
| 合规 | "Is there PII in here? Any data-residency requirement? That changes where I can put the cache, not just the database." | 存储选型、加密、cell 划分 |
| 成本 | "Is there a cost target — dollars per request, or per tenant per month? I'd rather know now than design something we can't afford." | 冷热分层、自建 vs 托管 |

中文注解：`ballpark`（大致量级）是这场面试里最值钱的一个词 —— 它明确宣告"我不要精确数字，我要数量级"，对方就敢给你一个粗数。`diurnal curve`、`flash crowd`、`latency budget` 是内行词，说出来就把你和背题的人分开。**"今天的量还是 18 个月后的量" 是 L5 才会问的问题。** 注意合规那一行的后半句 —— 问完立刻说出**这个答案会改变什么**：不带后果的提问是背清单，带后果的提问是设计。

❌ **不要说**："How many users do we have?" —— 太泛，只能换回一句 "What do you think?"，问题被弹回给你。
❌ **不要说**："So it needs to handle massive traffic, right?" —— `massive` 不是数字，这句话里没有一个能被设计使用的信息。

**③ 把假设说出来并让对方确认**

- "I'll assume we're read-heavy, around 20-to-1, unless you'd rather I design for write-heavy."
- "I'm going to assume a single region to start, and I'll call out where multi-region would change the design. Stop me if that's the interesting part."
- "Two assumptions, both cheap to reverse: eventual consistency on the feed, and no hard delete. Flag either one if it's wrong."

中文注解：`unless you'd rather I…` 是本节核心句型 —— 它把假设变成一个**可否决的提案**，而不是一个赌注。`Stop me if that's the interesting part` 主动邀请对方把面试导向他真正想考的地方，几乎总能换到有用信息。

**④ 澄清收尾（务必说出来，这是一个明确的评分点）**

> "Let me play that back to you. We're building a read-heavy system — call it 20-to-1 — peaking around 50k QPS, p99 under 200 milliseconds, three nines, with PII that has to stay in-region. **Out of scope**: offline analytics and the permissions model. I'll run with that — interrupt me if I drift."

中文注解：`play that back to you` 是母语者复述确认的标准说法。`Out of scope` 必须显式说 —— 主动砍范围是加分项，不是承认无能。结尾的 `interrupt me if I drift` 建立协作契约，后面被打断时就不会显得你被抓包。

---

## 2 提出估算（back-of-the-envelope）

**场景**：5 分钟，三个数（峰值 QPS、稳态数据量、月成本量级）。难点不是算，是**边算边说**且不显得心虚。

**① 开口就把过程交出去**

> "Let me do this out loud so you can stop me if an assumption looks off."

中文注解：把估算变成**双人活动**。面试官纠正你的假设不算失分（假设本来就该协商）；算错了没人拦、你自己也没发现，那才是失分。

**② 口播模板（照这个节奏说）**

- "Call it a hundred million daily actives. Ten reads each a day is a billion reads. There are about a hundred thousand seconds in a day, so that's roughly ten thousand QPS average. I'll take a 3x peak factor — call it 30k at peak."
- "Each record is about a kilobyte. A billion a day is a terabyte a day raw, times three for indexes and replicas — call it a petabyte a year. That's the number that decides whether we can keep everything hot."

中文注解：`Call it X` 是估算里最有用的两个词 —— 它宣告"这是我选的一个方便数字，不是我查到的事实"，既诚实又果断。`There are about a hundred thousand seconds in a day`（实际 86,400）请背下来，它把所有除法变成移小数点。

**③ 把误差范围说出来而不显得心虚**

- "I'm rounding aggressively — I care about the exponent, not the digits."
- "That's within a factor of two either way, and it doesn't matter: at 15k or at 60k the conclusion is the same — one primary won't do it."
- "If I'm off by 10x, this whole approach changes. Let me sanity-check that assumption before I build on it."

中文注解：**主动量化误差**（`within a factor of two`），再说明**误差不影响结论**（`the conclusion is the same`），这是把不确定性变成论证力量的标准手法。第三句更强：它区分了"哪些误差无所谓"和"哪些误差会推翻结论"，这是 L6 的表述。

**④ 算错了怎么改口**

> "Hold on — I dropped a zero. It's 300k, not 30k. That actually changes my answer: at 300k we can't do this in one region."

中文注解：干脆纠正，然后**立刻说这个纠正带来的后果**。这比一次算对还加分 —— 它证明你的数字是推出来的，不是背出来的。

**⑤ 每个数字后面跟一句 "so what"**

- "45k writes a second means a single Postgres primary is out — those top out around 20k. So sharding isn't a v2 thing, it's day one."
- "That's about eight thousand dollars a month in egress alone, more than the compute. So the interesting optimization isn't CPU, it's where the bytes cross the network."

中文注解：**算了不解释等于白算。** 每个数字必须落到一个架构判断上，`That means…` / `So…` 是强制自己落地的开关。

❌ **不要说**："I'm not sure if this is correct, but maybe it's around... some large number."
❌ **不要说**："Sorry, I'm not very good at math." —— 纯自伤。你在给面试官提供一个写进反馈的负面标签，而他本来不会自己想到。**算错了就改，不要道歉。**

---

## 3 提出方案（proposing a design）

**场景**：10–12 分钟画出主链路。全部难度在**分层**：先给骨架，再给细节，中间不能乱。

**① 宣布层次（先说你要说几件事）**

> "At a high level there are three pieces: an ingest path, a fan-out path, and a read path. Let me get those on the board first, then zoom into whichever one you care about."

中文注解：`At a high level` + 数量（three pieces）= 给听众一个栈帧。面试官在记笔记，你给他标题他就跟得上；跟不上的部分他不会给你分。

**② 缩放的连接词（这几组请背熟）**

| 意图 | 说法 |
|---|---|
| 往上拉 | "Stepping back for a second — the whole point of this layer is to absorb bursts." |
| 往下钻 | "Zooming into the fan-out service —" / "Let me open up that box." |
| 横向平移 | "That's the write path. The read path is much simpler — thirty seconds." |
| 暂存 | "Let me park that on the right side of the board and come back to it." |
| 回主线 | "OK, back to the main path — we were at the fan-out." |

中文注解：`park it` / `come back to it` 是**合法的延迟回答**。它不是回避 —— 回避是"这个我不确定"然后转移话题；`park` 是显式承诺、写在白板上、稍后兑现。**没兑现的 park 会被记住，所以只 park 你真的会回来的东西。**

**③ 口述走一遍主链路（画完必须做，很多人跳过）**

> "Let me trace one write end to end. Client posts with an idempotency key, gateway authenticates, the write service checks the key against Redis, writes to the primary, drops an event in the outbox table in the same transaction, returns 201. A relay tails the outbox and publishes to Kafka. **Nothing downstream is in the user's latency path.**"

中文注解：最后一句才是这段话的价值 —— 它把一串组件变成一个**关于延迟的论断**。图是死的，链路是活的。

**④ 主动认领你知道的问题 / 把选择权交出去**

- "I'm drawing a synchronous double-write here, and I know that's wrong — it multiplies the failure rates. I'm doing it to get the main path across; I'll replace it with an outbox in the deep dive."
- "That's the skeleton. The two hardest parts are hot partitions and cache coherence. I think hot partitions is the real problem — I'd start there, unless you'd rather see something else."

中文注解：**主动暴露 > 被抓到** —— 面试官不必花力气"抓"你，你们变成同一边的人，协作感是评分表上的隐藏项。第二句里 `I think hot partitions is the real problem` 是排序判断：你提名的深挖点必须是这题最难的部分，不是你最熟的部分，**选错了他会知道你不知道难在哪**。

❌ **不要说**："And then we add Kafka here." —— 没有理由的组件是白送的攻击面。改成："I'd put a queue here to decouple write latency from fan-out — the cost is that reads can lag by a second or two."
❌ **不要说**："This is the standard architecture." —— `standard` / `everyone does this` / `best practice` 是无论证断言，评分表上有专门一栏记这个。

---

## 4 讲取舍（the money section）

**场景**：整场面试唯一真正区分 Mid 和 Senior 的地方。别处可以平庸，这一节不行。

**四段式英文版（背下来，能套 80% 的决策）**

```
choice   →  "I'd go with A."
reason   →  "Under [constraint], A gives us [quantified win] over B."
cost     →  "The cost is [specific downside]."
boundary →  "I'm fine paying it because [X]. If [condition] flipped, I'd switch to B."
```

> "I'd go with an eventually consistent read replica for the feed. We said 20-to-1 reads and a couple seconds of staleness is fine, so replicas buy us roughly 20x the read capacity for the price of one extra node type. The cost is that a user can post and not see their own post — so I'd add read-your-writes by pinning that session to the primary for five seconds. If staleness turned out to be unacceptable somewhere, I'd move **that specific path** to the primary, not the whole system."

中文注解：注意最后一句 —— **回退方案限定在一条链路上，而不是整个系统**。能把让步收窄到最小范围，是 Senior 和 Mid 最明显的语言差别。

**"I'd rather take X and pay Y" 家族（10 句，直接背）**

1. "I'd rather take the extra write amplification and pay for it in disk — disk is the cheapest thing in this budget."
2. "I'll trade strict consistency for availability here, and I'll tell you exactly where the seam is: it's the counter, and it can be off by a few for up to ten seconds."
3. "This buys us about 40% on cost and costs us a second of freshness. At our numbers that's a good trade — and it's the first thing I'd give back if product pushed on freshness."
4. "Both options are bad in different ways. Let me tell you which kind of bad I'd rather own: I'd rather be slow than wrong here, because this is billing."
5. "The failure mode of B is worse than A's. A pages someone; B silently serves stale data for a week and nobody notices until an audit."
6. "That's a one-way door — the shard key. I want two more minutes on it. Everything else here is reversible in a sprint, so I'd rather ship and revisit."
7. "I'm optimizing for p99, not median, and I'm explicitly deprioritizing throughput. If this were a batch system I'd invert both."
8. "At today's numbers this would be over-engineering. I'd start with one Postgres, and here's the exact trigger to move: writes past 60% of a primary at peak, or one tenant over 20% of the table."
9. "If I'm wrong about the access pattern, the blast radius is a backfill — two weeks of engineering, not a rewrite. That's what makes me comfortable committing now instead of studying it for a month."
10. "There's no free lunch here. Anything that makes this faster makes recovery slower, because it's the same buffer."

中文注解：十句共享一个结构 —— **主动说出你放弃了什么**。面试官从不指望你的方案没有缺点，他在看你**知不知道**缺点在哪。第 4 句和第 10 句尤其值钱：承认"两个都不好"之后仍然做出选择，这是判断力，不是含糊。

**怎么说"这个方案在某个规模会失效"**

- "This holds until roughly 50k QPS. The signal we're getting close is replication lag creeping past 500 milliseconds at peak — that's the alert I'd set, and it fires weeks before anything breaks."
- "The scaling limit isn't CPU, it's open connections. We hit it around 8,000 concurrent, and the symptom is a latency cliff, not a gradual slowdown. Cliffs are the dangerous kind — the average gives you no warning."
- "This is fine up to about a terabyte per tenant. Past that, rebalancing takes longer than the growth rate and you can never catch up."

中文注解：`the signal is…` 是这本手册那句话的英文形。**说得出规模上限但说不出信号，只值半分** —— 面试官在等的是一句能写进 runbook 的话。

**怎么承认代价而不显得方案很差**

- "The weakest part of this design is the rebalancing story. It works, but it's manual and takes an afternoon. I'd ship it that way and automate it once we're doing it more than once a month."（承认 + 给升级触发条件）
- "Yes, this adds a network hop — about a millisecond. Against a 200 millisecond budget I'll spend it, and I get tenant isolation for free."（量化代价并对比预算）
- "It's more moving parts than I'd like. What makes it acceptable is that each part fails independently and the degraded mode is still useful."（承认复杂度 + 说明被什么抵消）

中文注解：三个模式的共同点 —— 代价后面永远跟着一句**让它变得可接受**的话，而不是一句道歉。

❌ "Redis is faster, so I'd use Redis." —— 快是属性不是论证，缺了约束（为什么这里需要快）和代价（丢数据、内存、又一个要运维的东西）。
❌ 单独一句 "It depends." —— 说了等于没说。永远补上轴："It depends on how much staleness product can take — and we said two seconds, so I'd go with the replica."
❌ "There's no real downside to this." —— 这一句能把你从 L5 打到 L4。说没有代价，意思是你没想过。
❌ "We can always add it later." —— 除非紧接着说 later 有多贵："…it's a config change, not a migration."

---

## 5 被 challenge 时（handling pushback）

**场景**：面试官说 "But what if…"。这是全场信息量最大的时刻 —— 他刚刚告诉了你他关心什么。

**先分清他在干什么：**

```
"Are you sure?"        ──▶ (a) 想听更多细节  (b) 你确实漏了一个 case  (c) 压力测试
                           三种意图，正确的第一句永远一样：先展开，不要先改。
"Why not just use X?"  ──▶ 他想知道你有没有想过 X（"I did consider X. The reason I didn't…"）
"What if X fails?"     ──▶ 他在考失败模式，不是在质疑方案本身
"Hmm." / 沉默           ──▶ 你说太快了，或者他不同意但还没想好怎么说
```

**A · 同意并修正（你真的漏了）**

> "You're right, I missed that. If two writers hit the same key in the same millisecond, my version check doesn't help — it's read-modify-write without a fence. The smallest fix is a conditional update with a version column; the bigger hammer is a lease. I'd take the conditional update, because it doesn't add a new failure mode."

中文注解：承认要**具体**（说出机制为什么坏），然后给**两个修法并选一个**。空口 "You're right, let me change it" 拿不到分 —— 他无法确认你真的理解了刚才那个 case。

**B · 部分同意（最常见的正确答案）**

- "Partly. That's true on the write path — there we do need the lock. On the read path we're covered, because reads go through a snapshot and never observe a partial batch. So I'd change the write path only, not the whole design."
- "The scenario is real, but rarer than it sounds — it needs a leader change *and* an in-flight batch *and* a client that doesn't retry. I'd handle it with idempotency keys rather than making the whole path synchronous. That's a big price for a narrow case."

中文注解：`Partly` 开头是最有用的一个词：既不投降也不对抗，并且**逼你去界定范围** —— 而界定范围正是这场面试要考的东西。第二句展示了另一个高阶动作：**估算这个 case 的概率**，再按概率决定投入多少复杂度。

**C · 有依据地坚持（最难，也最加分）**

- "That's fair — though I'd still lean toward the async path here, because the failure mode of synchronous is that one slow downstream takes the whole write path down with it. We said earlier that a 30-second delay is acceptable. If that's not actually true, I'd flip to sync-with-job-id plus SSE for progress."
- "I hear you, and I think we're optimizing for different things. If the priority is simplicity for a three-person team, you're right, the single service wins — I was optimizing for blast radius. Which one matters more here?"

中文注解：坚持的公式是 **承认对方合理（That's fair / I hear you）→ 给机制层面的理由（the failure mode of Y is…）→ 引用之前确认过的约束（we said earlier…）→ 给前提变化后的方案（If that's not true, I'd…）**。四段齐全，你就不是固执，是在辩护一个立场。第二句更进一步：把分歧归因到**优化目标不同**再抛回去，几乎总能问出他真正的评分点。

**买时间的两句**

- "Let me make sure I've got the scenario right — you're saying the leader dies after it acks but before it replicates. Is that it?"
- "Can I take thirty seconds on that? I'd rather give you a real answer than a reflex."

中文注解：复述对方的场景有三个作用 —— 确认理解、给自己十秒思考、并证明你在听。**全场性价比最高的一句话。**

❌ **绝对不要**：说 "You're right"（如果你其实不觉得）然后立刻推翻整个设计。
> 这是评分表上有专门条目的死法："论证不是自己的，是背的"。面试官的追问有一半是故意的压力测试 —— **秒改答案，你就把前面二十分钟的论证一起作废了。** 修正应该是**局部的**：改一个组件、改一个参数，不是重画。

❌ "No, that can't happen." —— 绝对化的防御。改成 "It can happen — here's how often, and here's what it costs when it does."
❌ "As I mentioned earlier…" —— 在英语里带着"你没在听"的味道。改成 "Right, and that ties back to the constraint we set —"

---

## 6 不知道的时候（the hardest one）

**场景**：被问到你没做过的东西。这里的分差比任何技术问题都大 —— 面试官几乎都在自己的专业领域出题，编造会被当场识破，并**污染你前面所有的可信度**。

**三段式：承认 → 推理路径 → 验证方法**

> "I haven't worked with HNSW directly, so let me be upfront about that. My mental model is that it's a layered skip list over the vector space, and `efSearch` is the recall-versus-latency knob — bigger means better recall and more hops. **I'd validate that by** running a hundred-million-vector benchmark and measuring Recall@10 against p99 before committing to self-hosting."

- "I don't know that off the top of my head, but I can reason about it. The problem it has to solve is [A], under constraint [B], so mechanically it almost has to do something like [C]. Whether it uses a lease or a fence, I'm not sure."
- "That's the same shape as [D], which I have done. There the thing that mattered was [E], and I'd expect the same tension here — that's where I'd look first."
- "I'd rather not guess at a number I don't know. I can give you the order of magnitude and how I'd confirm it: single-digit milliseconds, and I'd check with a one-hour load test before designing around it."

中文注解：**关键是 `I don't know` 后面不能有句号。** 单独的 "I don't know" 把这道题的评估样本削成零 —— 面试官什么也没看到。三段式让他看到：诚信 + 第一性原理推理 + 工程验证习惯，这三项**都是可迁移信号**，比一个背出来的正确答案更值钱。

**区分"我读过"和"我做过"（诚信项，权重高于技术项）**

| 你的真实情况 | 怎么说 |
|---|---|
| 生产上跑过、on-call 过 | "I've run this in production — we got paged for it twice." |
| 用过但没运维过 | "I've used it, but I've never operated it at scale." |
| 只读过 / 只看过 talk | "I've read about it but haven't touched it. What I understand is…" |
| 完全不知道 | "That's new to me. Can you tell me what problem it solves? Then I can reason about it." |

中文注解：最后一行是被低估的动作 —— **反问定义，然后现场推理**。它把"不知道"变成一段可评估的思考过程，很多面试官会直接给线索并陪你推下去，那一段通常是全场得分最高的部分。

❌ "I don't know."（然后停住）
❌ "I think it uses a B-tree internally."（当你其实不知道）—— 编造的成本不是这一题扣分，是**他开始怀疑你前面说的每一句话**。
❌ "I have a lot of experience with Kafka."（如果你只读过文档）—— 一次深挖就会暴露。诚信风险是一票否决项，和技术分不在一个量级。

---

## 7 主动驱动与收敛（driving and converging）

**场景**：默认失败模式是"面试变成问答" —— 他问一句你答一句，45 分钟结束，他只看到了他自己想到的那几个点。

**① 自己看时间、自己宣布**

> "We're about twenty minutes in. I'd like to spend the rest going deep on the sharding strategy — unless you'd prefer another area."

- "I've got maybe ten minutes left, and I want to make sure we get to tradeoffs. Let me close this thread out and move on."
- "Before I go deeper — is this the level of detail you want, or should I go one layer down?"

中文注解：**时间由你报，不由面试官报。** 一旦是他说出 "We have about ten minutes left"，那已经是一个记下的失分事件。第三句是校准问句，防止你在他不关心的层次上讲五分钟。

**② 每段结束挂一句下一步**

- "That's fan-out done. Three things left that matter: hot tenants, the rebalance path, cross-region writes. I'd rank them in that order — pick one and I'll go deep."

**③ 公开承认时间管理失误（比假装没事强）**

- "I've spent too long on requirements — that's on me. Let me speed up: main path in five minutes, then we'll talk about where it breaks."

中文注解：承认 + 立刻给补救计划。评分表「结构化」这一栏看的是**你有没有在管理时间**，不是你有没有完美地管理时间。

**④ 收敛信号（剩 8 分钟时无条件说）**

> "Let me stop the deep dive here and spend the last few minutes on tradeoffs and how this evolves — I think that's more valuable than finishing this thread."

中文注解：**没有收敛的设计题，评分上等同于没做完。** 主动放弃一个没讲完的深挖去换收敛，本身就是在展示取舍能力。

❌ "Do you want me to continue?" —— 太被动，方向盘交出去了。改成 "I'll keep going on the read path unless you want to switch."
❌ "I still have a lot to cover."（然后加速念）—— 加速不是收敛。收敛是**砍掉东西并说明为什么砍**。

---

## 8 深挖时的表达（going deep）

**场景**：15–20 分钟，2–3 个组件，唯一的加分区。语言上的分水岭：只说 what，还是把 **why-this-number** 说出来。

**① 参数：数字后面必须跟理由**

- "I'd set the TTL to sixty seconds with ten percent jitter. Sixty because that's the staleness product signed off on; jitter because otherwise everything we warmed up at boot expires in the same second and we get a synchronized stampede."
- "Batch size of 500, flushed every 200 milliseconds, whichever comes first. 500 keeps us under the 1 MB message limit with headroom; 200 milliseconds is what's left of the latency budget after the network."

中文注解：`whichever comes first` / `with headroom` / `what's left of the budget` —— 这三个短语传达的是**你真的配过这个东西**。背来的方案给不出第二个数字的来源。

**② 失败模式：怎么坏、谁先知道、变成什么**

- "Let me talk about how this breaks. The first thing to go is the relay lagging — the outbox table starts growing. **We find out from queue depth, not from user complaints**, and that's the point. If it grows past ten minutes' worth, we stop accepting non-critical writes rather than let the table eat the disk."
- "There's no silent failure here by design. If enrichment is down we fail closed and drop the request into a retry queue — we never write a half-enriched record, because **a half-enriched record looks correct forever**."

中文注解：`We find out from X, not from user complaints` 把可观测性和失败模式绑在了一起。`a half-enriched record looks correct forever` 这类具体后果，是面试官写反馈时会直接引用的原话 —— **可引用性本身就是分数。**

**③ 监控：区分 page 和 dashboard**

> "The one thing I'd page on is error-budget burn rate — multi-window, fast and slow. Everything else is a dashboard. If you page on raw error count you'll wake someone at 3am for something that self-heals in ninety seconds, and within a month nobody reads the pages."

中文注解：`Everything else is a dashboard, not a page` 省事又高分 —— 它证明你想过**告警疲劳**，而告警疲劳是每个 on-call 过的人的共同创伤，面试官会立刻认出同类。

**④ 迁移剧本（Staff 信号）**

> "The rollout is dual-write, then a rate-limited backfill watching primary IOPS, then shadow reads — serve the old path, compare against the new. Once the diff rate is under 0.01% for a week, shift reads 1, 10, 50, 100. Old path stays warm for two weeks. **Every step rolls back in minutes, not hours** — that's the property I care about, more than how fast we finish."

中文注解：最后一句把六个步骤压缩成一个**你在优化什么**的陈述。步骤谁都能背，优化目标才是判断力。

❌ "We'd add monitoring and alerting." —— 等于没说，任何人都会说。必须给**指标名 + 阈值 + 谁被叫醒**。

---

## 9 收尾与提问（wrap-up）

**30 秒总结模板（照填，五个槽位一个不落）**

> "Let me wrap up. We're building [one line]. The shape is [three pieces]. The main trade I made was [X over Y], and it costs us [Z]. The one decision I'd want to be right about before anyone writes code is [one-way door], because changing it later is a quarter-long migration. And the first thing that breaks at 10x is [component] — the signal is [metric] crossing [value]."

中文注解：**这段话是面试官写反馈时的素材。** 说得越结构化，他写得越容易，你的分越高。含糊的正确 < 具体的可辩护。

**问面试官的 3 个好问题**

- "What was the last serious incident on this system, and what changed afterwards?"
- "What's the on-call load like for this team — pages per week, roughly?"
- "Where do you think this architecture will need to be rewritten in two years? What's already hurting?"
- （AI / 基础设施岗加一个）"Are you self-hosting inference or going through an API? And roughly what fraction of revenue does that cost eat?"

中文注解：共同点是**只有真正在这里工作的人才答得出来**，所以它们能换到真信息，同时证明你在评估这份工作而不是在讨好。

❌ "Do you have any feedback for me?" / "How did I do?" —— 多数公司不允许当场反馈。你把他推进一个要么违规、要么尴尬的位置。
❌ 福利、假期、work-life balance —— 留给 recruiter。技术官的三分钟只能问工程现实。

---

## 10 高频连接词与缓冲语（buying yourself time）

**场景**：你需要 5–10 秒想事情。沉默超过 20 秒会被记成"不知道他在干什么"，但**有标记的沉默完全合法**。

**沉默预算**：单次 ≤ 10 秒，全场 ≤ 3 次，每次前面必须有一句标记语。

| 用途 | 说这个 |
|---|---|
| 明确要时间 | "Let me think about that for a second."（然后真的想，不要边想边发声） |
| 承认问题难 | "Good question — the reason it's tricky is that those two requirements fight each other." |
| 拆分问题 | "There are two things there. Let me take them separately." |
| 区分概念 | "I want to separate two issues: whether it's correct, and whether it's fast enough." |
| 从抽象落地 | "Let me put a number on that." / "Concretely —" |
| 重新框定 | "The way I'd frame it is —" / "Let me back up and say what I think the real constraint is." |
| 定义术语 | "To be precise about what I mean by 'eventually consistent' here —" |
| 承认没想过 | "That's a case I hadn't considered. Give me a moment." |
| 换个说法 | "Let me put that differently." |

中文注解：`Let me think about that for a second` 之后的八秒沉默，效果**优于**立刻答一个中庸答案 —— 它显示你在真的思考，而不是在检索背过的模板。前提是你真的用上那八秒。

❌ **不要用**：`umm` / `you know` / `like` / `so… so… so…` / `how to say` / "in Chinese we have a word…"
> 它们不传递信息，且会累积成"紧张、不熟练"的整体印象。**把填充音换成标记语，是这一节唯一要做的事。**

---

## 11 中式英语高频陷阱

**核心断言**：面试官不给语法打分。冠词、时态、复数错了没人扣你分。**扣分的是 hedging（对冲）—— 它直接读作"缺乏 conviction"，而 conviction 是评级维度。**

| 中式表达 | 地道说法 | 为什么换 |
|---|---|---|
| "I think maybe we can use Redis." | "I'd go with Redis." | 三重 hedge 叠加（think + maybe + can），听起来你自己都不信 |
| "I will design a system that can handle…" | "Let's start with the write path." | 未来时 + 宣告式 = 作文腔。面试是协作，用 `I'd` / `Let's` |
| "It is very very important." | "This is the part that matters." | 英文靠**词汇强度**表达强调，不靠重复副词 |
| "It's a little bit slow." | "It's about 200 ms — that blows our budget." | `a little bit` 是道歉性缓冲，换成数字 |
| "Actually, actually…" | （删掉） | `actually` 暗示"与你所想相反"，滥用像在纠正对方 |
| "I have some experience about Kafka." | "I've run Kafka in production." | `about` → `with`；且必须区分 read / used / operated |
| "We must use a queue." | "We'd need a queue here." | `must` 在英文里过强，像下命令 |
| "Do you think it's OK?" | "Does that match how you'd approach it?" | 前者求批准，后者邀请对齐 |
| "I'm not sure, but…"（每句开头） | 全场用一次就够 | 反复自我否定是非母语者最大的隐性扣分项 |
| **"Sorry, my English is not good."** | **（绝对不说）** | 你把注意力从设计转移到你的英语上 —— 他本来可能根本没注意 |
| "Open the database." | "Hit the database." / "Query it." | |
| "The QPS is very high, maybe ten thousand." | "Call it 10k QPS at peak." | |
| "I want to say that…" | "My point is —" | |
| "Let me introduce my design." | "Here's the shape of it." | `introduce` 是中式演讲腔 |
| "In my opinion I think…" | "I'd argue…" / "My read is…" | 双重表态叠加 |
| "It has many advantages." | "It buys us two things: X and Y." | 具体 > 抽象，且顺手给了数量 |
| "We should consider using cache." | "I'd cache this." | 动词化，短一半，强一倍 |
| "Because of the reason that…" | "Because…" | |
| "How to say…" | "Let me put that differently." | |

**三个语音层面的陷阱（比语法重要）**

```
① 陈述句尾音上扬  → 你的结论听起来像在征求同意。
   "I'd shard by tenant ID?"  ✘        "I'd shard by tenant ID."  ✔

② 紧张时语速飙升  → 面试官在记笔记，跟不上的部分不给分。
   每讲完一个组件停 1 秒。停顿是给对方的，不是给你的。

③ 音量随不确定性下降 → 你自己给答案打了个低分。
   不确定的内容用「标记」处理（"I'm less sure about this part, but—"），
   不要用音量处理。
```

中文注解：非母语者最容易犯的错，是**用语言的弱化来表达技术上的不确定**。正确做法是把不确定**说出来并限定范围**（"I'm confident about the write path; the rebalance story is where I'd want to prototype"），而不是让整段话都变弱。

---

## 12 一页速记卡

打印这一页。考前五分钟看一遍，别的都别看。

```
━━ 开场 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 1  "Before I start drawing, I'd like to pin down a few things."
 2  "Ballpark — is it 10-to-1 reads, or the other way around?"
 3  "I'll assume X unless you'd rather I go the other way."
 4  "Let me play that back to you… Out of scope: Y. Interrupt me if I drift."

━━ 估算 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 5  "Let me do this out loud so you can stop me if an assumption looks off."
 6  "Call it 100M DAU… about 100,000 seconds in a day… so ~30k at peak."
 7  "That's within a factor of two, and the conclusion doesn't change either way."
 8  "45k writes a second means one primary is out. So sharding is day one."

━━ 方案 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 9  "At a high level there are three pieces… then I'll zoom in."
10  "Let me park that and come back once the main path is up."
11  "I know this is wrong — I'll replace it with an outbox in the deep dive."

━━ 取舍（最重要）━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
12  "I'd go with A. Under [constraint] it buys [X]. The cost is [Y].
     If [condition] flipped, I'd switch to B."
13  "I'd rather be slow than wrong here, because this is billing."
14  "The failure mode of B is worse: A pages someone, B silently serves
     stale data for a week."
15  "This holds until ~50k QPS. The signal is replication lag past 500 ms."
16  "That's a one-way door. Everything else is reversible in a sprint."

━━ 被质疑 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
17  "Let me make sure I've got the scenario right — you're saying… Is that it?"
18  "Partly. That's true on the write path; the read path is covered because…"
19  "That's fair — though I'd still lean toward X, because the failure mode
     of Y is [specific]. If [constraint] isn't real, I'd flip."

━━ 不会 / 收尾 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
20  "I haven't worked with X directly. My mental model is [Y] — I'd validate
     that by [Z]."
21  "We're about twenty minutes in — I'd like to spend the rest on sharding,
     unless you'd prefer another area."
22  "The one decision I'd want to be right about before anyone writes code
     is [one-way door]. The first thing that breaks at 10x is [component]."

━━ 绝不出口 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 ✘ "Sorry, my English is not good."      ✘ "There's no real downside."
 ✘ "I think maybe we can…"                ✘ "It depends."（后面没有轴）
 ✘ "You're right."（然后推翻整个设计）     ✘ "I don't know."（然后沉默）
 ✘ "This is the standard architecture."   ✘ "We'd add monitoring and alerting."
```

---

## 面试官会追问

这一节的自检方式不同 —— **不要在脑子里过，要出声说。** 录音、回放，听三件事：hedge 词的数量、有没有升调、每个方案后面有没有跟一句 "the cost is"。

1. 用英文说一遍你上一个项目的核心取舍，30 秒，必须包含 choice / reason / cost / boundary 四段。
2. 面试官说 "Are you sure about that?" —— 你的第一句话是什么？（不能是 "You're right"）
3. 被问到一个完全没做过的组件，说出三段式，全程不超过 40 秒。
4. 你的方案在什么规模失效？用英文说，必须带一个具体指标名和一个具体阈值。
5. 数一下你刚才那段话里有几个 `I think` / `maybe` / `a little bit`。超过一个就重说。
6. 30 秒收尾模板，五个槽位一个不落地说一遍。

---

**下一篇** → [06-mock-interview.md](06-mock-interview.md)：45 分钟英文逐字稿，看这些句式在真实节奏里怎么落地。
**回看框架** → [01-interview-framework.md](01-interview-framework.md)｜**题库自测** → [02-question-bank.md](02-question-bank.md)｜**数字速查** → [03-cheatsheet.md](03-cheatsheet.md)｜**术语补漏** → [04-glossary.md](04-glossary.md)
