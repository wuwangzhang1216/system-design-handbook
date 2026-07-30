# 04 · 中英术语对照表

> 概念你已经懂了。这张表解决的是另一件事：**英文面试时，那个词能不能在半秒内出口**。
> 三类词会让你当场卡住 —— **中文直译会错的**（限流不是 limit current，链路不是 link，雪崩不是 avalanche）、**有固定行话必须整块背的**（cache stampede / fencing token / blast radius / lethal trifecta）、**发音或重音错到对方听不懂的**（idempotent、quorum、canary、choreography）。
> 全表 445 条，14 组主题，每条给一句**面试里真会说出口的整句**，不是词典释义。

---

## 0. 怎么用这张表

**三种用法，各自的时间成本不同：**

| 场景 | 怎么用 | 耗时 |
|---|---|---|
| **面试前一天** | **只扫第 3 列**（常见误译 / 注意）。第 1、2 列你多半已经会了，风险全在第 3 列 | 15–20 分钟 |
| **准备时卡壳** | 按主题跳到对应组。想不起来"分片再平衡"怎么说，去第 11 组 | 30 秒 |
| **日常练习** | **朗读第 4 列的例句**。默读无效 —— 阅读理解和口腔肌肉记忆是两套系统，前者不会自动变成后者 | 每组 5 分钟 |

**标注约定：**

```
⚠  中文直译会错，或英语里根本没有对应说法 —— 硬翻会让对方一脸茫然
◆  固定行话，必须整个短语一起用，换一个词对方就听不懂
♪  发音或重音容易错，详见文末「发音与重音提醒」
```

**关于第 4 列的例句：**每句都是**可以直接嵌进回答里的完整句子**，多数句子里带一个数字或一句代价 —— 这是全书的一贯要求，也是面试官在反馈里能直接引用的形态。含糊的正确 < 具体的可辩护。

**这张表不做什么**：不解释概念（去对应正文章节），不给国际音标（给音节），不收只在论文里出现、面试里说不出口的词。

---

## 1. 性能与延迟

这一组是**开口密度最高**的一组 —— 几乎每一轮回答都要用到三四个。重灾区是 latency / delay 混用、percentile 不会读、backpressure 写成两个词。

| 中文 | English | 常见误译 / 注意 | 面试例句 |
|---|---|---|---|
| 延迟 | latency ♪ | ⚠ 不是 delay。delay 指"被推迟了"，latency 指固有耗时 | "Our latency budget is 3 seconds end to end; the model gets 1.5 of it." |
| 吞吐 | throughput ♪ | ⚠ 不是 flux / flow。读 THROO-put，一个词 | "Past a point, adding nodes actually lowers throughput." |
| 尾延迟 | tail latency ◆ | ⚠ 不是 tail delay。指 p95/p99 那一段的统称 | "One straggler backend can dominate your tail latency." |
| 分位数 | percentile ♪ | p99 读 "P ninety-nine"，正式场合说 "the ninety-ninth percentile" | "Percentiles don't aggregate across dimensions — you have to sum the buckets first." |
| 尾延迟放大 | tail latency amplification ◆ | 扇出场景专用 | "With a fan-out of 100, tail latency amplification means 63% of requests hit at least one slow backend." |
| 排队延迟 | queueing delay | queueing / queuing 两种拼法都对 | "Most of the p99 here is queueing delay, not service time." |
| 延迟放大 | latency amplification | | "At 90% utilization you get 10x latency amplification — that's why we target 60 to 70%." |
| 利用率 | utilization | ⚠ 不是 usage rate | "Steady-state target utilization for online services is 60 to 70%." |
| 余量 / 头部空间 | headroom ◆ | ⚠ 不说 margin（那是利润或页边距） | "I leave twenty to thirty percent headroom for traffic jitter." |
| 峰谷比 | peak-to-average ratio / peak-to-trough ratio | ⚠ 不是 peak valley ratio | "I'm assuming a 3x peak-to-average ratio — tell me if that's wrong." |
| 秒杀 | flash sale | ⚠ 绝对不要说 second kill | "A flash sale can be 100x the average, so we pre-scale instead of autoscaling." |
| 队头阻塞 | head-of-line blocking ◆ | 连字符不能省，缩写 HOL blocking | "One slow message causes head-of-line blocking for the whole partition." |
| 扇出 | fan-out (n.) / fan out (v.) | 名词带连字符，动词分开写 | "A celebrity post is a hundred-million fan-out write, so celebrities go pull." |
| 慢节点 | straggler ◆♪ | ⚠ 不说 slow node。读 STRAG-ler | "One straggler backend can dominate your tail latency." |
| 抖动 | jitter | ⚠ 不是 shake / fluctuation | "Add jitter to the TTL so keys don't all expire at the same instant." |
| 毛刺 | latency spike | ⚠ 不是 burr。glitch 太口语，别用在指标语境 | "Refresh-ahead removes the cold-start spike." |
| 背压 | backpressure ◆ | ⚠ 一个词，不是 back pressure | "Without backpressure the edge just buffers until it OOMs." |
| 准入控制 | admission control | ⚠ 不是 access control（那是权限） | "Admission control rejects when queue delay crosses the threshold." |
| 负载卸载 | load shedding ◆ | ⚠ 不是 unload。shed = 主动抛掉 | "Load shedding kicks in when queue wait exceeds 100 ms — protect the requests already in flight." |
| 在途请求 | in-flight requests ◆ | ⚠ 不说 on-the-way | "By Little's Law, in-flight requests equal QPS times latency." |
| 并发限制 | concurrency limiting | 与 rate limiting 区分：一个限"同时"，一个限"每秒" | "We do concurrency limiting rather than QPS limiting — it tracks real resources." |
| 有界 / 无界队列 | bounded / unbounded queue | | "An unbounded queue just defers the OOM to the worst possible moment." |
| 排队论 | queueing theory | | "Two bits of queueing theory carry most of capacity planning: Little's Law and M/M/1." |
| 利特尔法则 | Little's Law | 大写 L，撇号别丢 | "Size the connection pool with Little's Law, then leave 2x headroom." |
| 对冲请求 | hedged request ◆ | 行话说 hedged；backup request 也有人用，但 hedged 更标准 | "Hedged requests cost about 5% extra traffic and require idempotency." |
| 感知延迟 | perceived latency | | "Streaming only changes perceived latency; it doesn't change wall-clock or cost." |
| 墙钟时间 | wall-clock time ◆ | ⚠ 不是 physical time | "There's a wall-clock deadline on top of the turn and token caps." |
| 超时预算 | timeout budget | | "Each hop only subtracts from the timeout budget — a downstream timeout is never larger than upstream's." |
| Deadline 传播 | deadline propagation | | "Deadline propagation means downstream never outlives upstream." |
| 撞墙 | hit a scaling limit / hit a wall ◆ | ⚠ 不是 hit the ceiling | "I'd only move off Postgres when I can say which metric hit a scaling limit and that I measured it." |

---

## 2. 一致性与事务

这一组的杀伤力在**形容词形式**：strong consistency 的形容词是 **strongly** consistent，不是 strong consistent。说错一次面试官不会纠正你，但会记下来。

| 中文 | English | 常见误译 / 注意 | 面试例句 |
|---|---|---|---|
| 线性一致性 | linearizability ♪ | 形容词 linearizable。⚠ 不等于 serializability | "R plus W greater than N doesn't give you linearizability — for that you need consensus." |
| 强一致性 | strong consistency | ⚠ 形容词说 strongly consistent | "Strong consistency costs you an extra round trip on every read." |
| 最终一致性 | eventual consistency | ⚠ 形容词说 eventually consistent | "Default reads are eventually consistent; you opt into strong reads." |
| 读己之写 | read-your-writes ◆ | 固定写法，别改成 read-my-write | "The user saves and the list doesn't show it — that's a read-your-writes violation." |
| 陈旧读 | stale read | | "Which path can tolerate a stale read, and which absolutely can't?" |
| 陈旧度 | staleness | ⚠ 不是 old degree | "Every cache key gets a maximum acceptable staleness." |
| 隔离级别 | isolation level | | "What isolation level are we running — read committed or serializable?" |
| 快照隔离 | snapshot isolation (SI) | | "Readers pin a metadata file, so snapshot isolation is basically free." |
| 可串行化 | serializable ♪ | ⚠ 与 serialization（序列化）同根异义，看上下文 | "Serializable costs you throughput; we run read committed and fix the two anomalies by hand." |
| 写偏斜 | write skew ◆ | ⚠ 不是 write deviation。SI 下的经典异常 | "We fix write skew by materializing the conflict with SELECT FOR UPDATE." |
| 物化冲突 | materializing conflicts | | "Materializing the conflict turns an invisible anomaly into a normal lock wait." |
| 脏读 | dirty read | | "Sagas have no isolation, so you get dirty reads of intermediate states." |
| 丢失更新 | lost update | | "The compensation overwrote a concurrent write — classic lost update." |
| 幂等 | idempotent ♪⚠ | **全场最高频的发音坑**，见文末 | "Every side-effecting endpoint has to be idempotent." |
| 幂等性 | idempotency ♪ | idempotency 比 idempotence 更常用；两个都能听懂 | "At-least-once delivery means idempotency is mandatory, not optional." |
| 幂等键 | idempotency key | ⚠ 不是 idempotent key | "The idempotency key has to be derived from business semantics, not a fresh UUID per retry." |
| 乐观并发控制 | optimistic concurrency control (OCC) | | "We use optimistic concurrency — update where version equals N." |
| 乐观锁 | optimistic locking | ⚠ 它不是锁；别把 optimistic lock 当名词用 | "409 from optimistic locking means re-read and retry; 409 from an idempotency key means stop." |
| 条件更新 | conditional update / compare-and-swap (CAS) | CAS 读 "cass" 或 "C-A-S" | "A conditional update gives you the same mutual exclusion without a lock." |
| 唯一约束 | unique constraint | | "I'd push mutual exclusion down to a unique constraint in the database." |
| 两阶段提交 | two-phase commit (2PC) | 读 "two-P-C" | "Two-phase commit across nodes caps you around 20 nodes." |
| 分布式事务 | distributed transaction | | "Provisioning is a distributed transaction across five external systems." |
| Saga | saga ♪ | 读 SAH-guh，不翻译 | "A saga splits a long-running transaction into local transactions with compensations." |
| 补偿事务 | compensating transaction ◆ | ⚠ 不是 rollback transaction | "Compensation isn't a rollback — it's a new forward business operation." |
| 法定人数 | quorum ♪ | 读 KWOR-um，两个音节 | "R plus W greater than N gives you quorum overlap, not linearizability." |
| 双写问题 | dual-write problem ◆ | | "Writing to the DB and publishing to Kafka is the classic dual-write problem." |
| 精确一次 | exactly-once ⚠ | 传输层不存在。诚实说法是 effectively-once | "Exactly-once billing doesn't exist in transport — it's at-least-once delivery plus idempotent apply." |
| 至少一次 / 至多一次 | at-least-once / at-most-once | 做定语时连字符不能丢 | "Delivery is at-least-once, so every compensation has to be idempotent." |
| 超卖 | overselling | ⚠ 名词用 -ing 形式 | "We tolerate slight overselling and compensate afterwards." |
| 一致性边界 | consistency boundary | | "The consistency boundary is the account balance; everything else can be eventual." |
| 全局不变量 | invariant ♪ | in-VAIR-ee-unt | "CRDTs can't express a cross-replica invariant like 'these must sum to 100'." |

---

## 3. 存储引擎与索引

高危词是 **compaction**（不是 compression）和 **回表**（英语里没有这个词，只能描述）。

| 中文 | English | 常见误译 / 注意 | 面试例句 |
|---|---|---|---|
| 写放大 | write amplification ◆ | | "Leveled compaction gives you 30x write amplification, which burns through SSD endurance." |
| 读放大 | read amplification | | "Read amplification in an LSM is roughly the number of levels you have to probe." |
| 空间放大 | space amplification | | "Size-tiered compaction can double your space amplification in the worst case." |
| 压实 / 合并 | compaction ⚠ | 中文常译"压缩"，会和 compression 混死。compaction 重整文件，compression 压字节 | "Compaction causes latency jitter — you see p99 spikes every few minutes." |
| 压缩 | compression | | "Columnar storage plus zstd gets us 3 to 10x compression." |
| 原地更新 | in-place update | | "B-trees do in-place updates, so they need page-level latching." |
| 页分裂 | page split | | "Random UUIDs cause page splits and destroy insert throughput." |
| 表膨胀 | table bloat ◆ | ⚠ 不是 table expansion。特指死元组占空间 | "You have to prune the outbox table or you get bloat." |
| 回表 | go back to the heap / extra heap fetch ⚠ | **英语里没有"回表"这个词** —— 只能描述 | "It's a covering index, so we never touch the heap." |
| 覆盖索引 | covering index | | "A covering index avoids the extra table lookup entirely." |
| 联合索引 | composite index / multi-column index | ⚠ 不是 union index | "I'd add a composite index on tenant_id, status, created_at." |
| 最左前缀 | leftmost prefix rule | | "Because of the leftmost prefix rule, that index can't serve a query on column C alone." |
| 选择性 | selectivity | | "Put the high-selectivity equality columns first, then the range and sort columns." |
| 部分索引 | partial index | Postgres 叫 partial，SQL Server 叫 filtered | "A partial index on WHERE deleted_at IS NULL cut the index size in half." |
| 表达式索引 | expression index / functional index | | "Add an expression index on lower(email) so the query can use it." |
| 倒排索引 | inverted index | | "GIN is an inverted index — great for full text, slow to write." |
| 二级索引 | secondary index | | "Every secondary index makes writes 5 to 15 percent slower." |
| 分区裁剪 | partition pruning | | "Read EXPLAIN and count the partitions still in the plan — otherwise pruning silently stopped working." |
| 谓词下推 | predicate pushdown ♪ | PRED-i-kut（名词） | "A non-LEAKPROOF function blocks predicate pushdown and blows up the scan." |
| 列裁剪 | column pruning | | "Ban SELECT star in CI — column pruning is 3 to 10x on wide tables." |
| 行存 / 列存 | row store / columnar store | column store 也对 | "A row store reads the whole tuple even if you only need one column." |
| 向量化执行 | vectorized execution | | "Vectorized execution runs a column in batches of about 1,024 values, so per-row interpreter overhead amortizes away." |
| 点查 | point query / point lookup | ⚠ 不是 single query | "OLTP is mostly point queries; OLAP scans a few columns over many rows." |
| 全表扫描 | full table scan / sequential scan | Postgres 口语说 "a seq scan" | "Text-to-SQL gets you wrong joins and a full table scan on the bill." |
| 布隆过滤器 | Bloom filter | B 大写（人名 Burton Bloom） | "A Bloom filter is only trustworthy in one direction — 'not present' is exact." |
| 误判率 | false positive rate | | "At a one percent false positive rate, 100 million keys need about 120 megabytes." |
| 墓碑 | tombstone ◆ | | "Above 20% tombstones we rebuild the segment." |
| 软删除 / 硬删除 | soft delete / hard delete | | "Soft delete at T plus zero, hard delete at T plus 30 days." |
| 不可变段 | immutable segment | | "Segments are immutable: writes append, deletes tombstone, updates are delete-plus-insert." |
| 热分区 | hot partition ◆ | ⚠ 不是 hot area | "A celebrity user gives you a hot partition no matter how good the hash is." |
| 首字节延迟 | first-byte latency / TTFB | | "Object storage has 20 to 100 milliseconds of first-byte latency, so keep it off the hot path." |
| 存算分离 | storage-compute separation | 也说 disaggregated storage | "Storage-compute separation lets us scale the query layer independently." |
| 召回率 | recall ⚠ | ⚠ 不是 recall rate，更不是 callback。Recall@10 读 "recall at ten" | "We target Recall@10 above 0.90 and tune the knobs backwards from there." |

---

## 4. 缓存

**中文的"穿透 / 击穿 / 雪崩"三分法在英语里不存在。**硬翻 penetration 对方会想到渗透测试，硬翻 avalanche 对方完全听不懂。正确做法是用现象名 + 一句描述。

| 中文 | English | 常见误译 / 注意 | 面试例句 |
|---|---|---|---|
| 缓存命中率 | cache hit rate / hit ratio | | "A 95 percent hit rate sounds fine until you look at the absolute miss QPS." |
| 未命中 | cache miss | ⚠ 不说 not hit | "On a miss we do an origin fetch, and that path has to be rate-limited." |
| 回源 | origin fetch / fall through to the origin ⚠ | ⚠ 没有 "back to source" 这个说法 | "Put a semaphore on the origin fetch path so a cold cache doesn't stampede the backend." |
| 缓存击穿 | cache stampede ◆⚠ | 热 key 过期瞬间的并发回源，固定叫 stampede | "A hot key expiring at 50k QPS is a cache stampede — I'd use probabilistic early expiration." |
| 缓存穿透 | lookups for keys that don't exist ⚠ | ⚠ 别说 penetration。直接描述现象 | "Random UUID lookups never hit the cache, so we do negative caching plus a Bloom filter." |
| 缓存雪崩 | mass expiration → cascading failure ⚠ | ⚠ **avalanche 不是行话** | "Everything got the same TTL at warm-up, so it all expired at once and the database fell over." |
| 惊群 | thundering herd ◆ | | "Always add jitter, or your retries synchronize into a thundering herd." |
| 缓存空值 | negative caching ◆ | | "Negative caching with a short TTL kills the scan for keys that don't exist." |
| 单飞 / 请求合并 | single-flight / request coalescing ♪ | koh-uh-LESS-ing | "Single-flight only dedupes in-process; across processes you need a distributed lock." |
| 概率提前过期 | probabilistic early expiration | 论文名 XFetch | "XFetch is probabilistic early expiration — it refreshes before everyone hits the wall." |
| 返回旧值 | serve stale ◆ | ⚠ 不是 return old value | "We serve stale and refresh in the background, so there's always a value to return." |
| TTL 抖动 | TTL jitter | | "Add jitter to the TTL so keys don't all expire at the same instant." |
| 淘汰 | eviction ♪ | ⚠ 不是 elimination。读 ih-VIK-shun | "Evictions above zero mean memory is short and the hit rate will keep degrading." |
| 淘汰策略 | eviction policy | | "The eviction policy is allkeys-lru for pure caches." |
| 准入策略 | admission policy | | "The admission policy is what makes W-TinyLFU scan-resistant." |
| 旁路缓存 | cache-aside ◆ | ⚠ 不是 bypass cache | "Cache-aside with delete-on-write covers ninety percent of cases." |
| 写穿 / 写回 | write-through / write-behind | write-back 同义 | "Write-behind is fast and loses data when the cache dies — only for view counts." |
| 预刷新 | refresh-ahead | | "Refresh-ahead removes the cold-start spike at the cost of refreshing keys nobody reads." |
| 延迟双删 | invalidate, wait, invalidate again ⚠ | ⚠ 中文圈行话，英语里必须展开说 | "We do a delayed double delete: invalidate, wait 500 milliseconds, invalidate again." |
| 缓存失效 | cache invalidation | ⚠ 不是 cache failure（那是缓存挂了） | "There are only two hard problems: cache invalidation and naming things." |
| 失效级联 | invalidation cascade | | "Memorize the invalidation cascade: tools, then system, then messages." |
| 缓存键 | cache key | | "Cache key design is the whole game with a CDN." |
| 查询串规范化 | query-string normalization | | "Query-string normalization — strip the utm params, sort the rest." |
| 共享缓存 | shared cache | s-maxage 管的那层 | "s-maxage controls the shared cache; max-age controls the browser." |
| 版本化 URL | cache busting ◆ | | "We use content-hashed filenames for cache busting, plus immutable." |
| 缓存标签 | surrogate key / cache tag ◆ | | "Surrogate keys let us purge everything touching article-42 in one call." |
| 预热 | cache warming / warm-up | | "Without cache warming, restarts make the overload worse." |
| 本地兜底 | local fallback cache | fallback 是一个词 | "The local L1 cache is our fallback when Redis is completely gone." |
| 语义缓存 | semantic cache | | "A semantic cache without tenant partitioning is a data leak, not an optimization." |

---

## 5. 消息与流

高危词：消息堆积（consumer lag，不是 pile up）、水位线（watermark，不是 water level）、死信队列（DLQ，别说 dead queue）。

| 中文 | English | 常见误译 / 注意 | 面试例句 |
|---|---|---|---|
| 消息队列 | message queue ♪ | queue 读作字母 Q，一个音节 | "Don't use Kafka as a task queue — you can't ack individual messages." |
| 竞争消费者 | competing consumers ◆ | | "SQS is a competing-consumers model — each message goes to exactly one worker." |
| 消费组 | consumer group | | "Each consumer group keeps its own offsets and reads the full stream." |
| 消费位点 | offset | ⚠ 不是 position | "With a log you can replay from any offset; with a queue you can't." |
| 消息堆积 | consumer lag / backlog ⚠ | ⚠ 不是 message pile。Kafka 语境固定说 consumer lag | "We had 100 million messages of consumer lag — how do you drain that?" |
| 分区 | partition | | "Partition count is your ceiling on consumer parallelism." |
| 顺序保证 | ordering guarantee | | "Adding partitions remaps keys and breaks the ordering guarantee." |
| 全局有序 | total ordering | | "Total ordering means a single partition, which caps your throughput." |
| 去重 | deduplication / dedupe ♪ | 动词口语说 dedupe，读 "dee-DOOP" | "Global deduplication means unbounded state — always window it." |
| 去重表 | dedup table | | "We write to a dedup table keyed by message_id in the same transaction as the business write." |
| 去重窗口 | deduplication window | | "A 32-day dedup window is really a storage bill in disguise." |
| Outbox 模式 | outbox pattern ◆ | 不翻译，直接说 outbox | "Use the outbox pattern so the event and the row commit together." |
| 变更数据捕获 | change data capture (CDC) | 读字母 C-D-C | "CDC gives you a row-change stream, not a business event stream." |
| 领域事件 | domain event | | "If the name has 'Updated' in it, it's a row change, not a domain event." |
| 事件通知 vs 携带状态 | event notification vs event-carried state transfer ◆ | | "Thin events force every consumer into a read-back, which gives you an N-plus-one." |
| 死信队列 | dead-letter queue (DLQ) ◆ | ⚠ 不是 dead queue。口语直接说 "the DLQ" | "After the retry budget is exhausted it goes to a dead-letter queue for manual replay." |
| 毒丸消息 | poison pill ◆ | | "A poison pill will crash-loop the consumer and wedge the partition forever." |
| 可见性超时 | visibility timeout | SQS 术语 | "Set the visibility timeout longer than the slowest task, and heartbeat." |
| 心跳续约 | heartbeat renewal / lease extension | | "We extend the lease with heartbeat renewal while the agent is still running." |
| 重放 | replay | | "Aggregation must be upsert-based so it's replayable without doubling." |
| 断点重放 | replay from checkpoint | | "CDC needs replay from checkpoint on day one, not as a follow-up." |
| 水位线 | watermark ◆ | ⚠ 不是 water level。流处理专用 | "Watermarks are how you handle out-of-order events." |
| 迟到数据 | late-arriving data / late events | | "Late-arriving events after period close go into a true-up." |
| 乱序 | out-of-order events | | "Version-guarded upserts make the projection immune to duplicates and out-of-order events." |
| 窗口聚合 | windowed aggregation | | "Windowed aggregation emits a preliminary result at the watermark." |
| 事务性入队 | transactional enqueue | | "Transactional enqueue solves the dual-write problem for free." |
| 数据库即队列 | database as a queue | | "Use the database as a queue with SKIP LOCKED — it holds up to a few thousand TPS." |
| 削峰 | smooth traffic spikes / buffer the burst ⚠ | ⚠ 见第 14 组，绝不要直译 | "If you're only smoothing a 2x spike, just add machines instead." |
| 饥饿 | starvation ⚠ | ⚠ 不是 hunger | "Interactive tasks must not starve behind batch — separate queues or weighted round-robin." |
| 公平队列 | fair queuing | | "Round-robin is wrong when request cost varies — you want deficit-based fair queuing." |
| 两将军问题 | Two Generals Problem | 首字母大写 | "End-to-end exactly-once delivery is impossible — that's the Two Generals Problem." |

---

## 6. 网络与负载均衡

高危词：round robin（负载均衡）和 polling（反复查询）中文都叫"轮询"；packet loss 不是 package loss。

| 中文 | English | 常见误译 / 注意 | 面试例句 |
|---|---|---|---|
| 四层 / 七层负载均衡 | L4 / L7 load balancing | 读 "L four" / "layer four" 均可 | "We do L4 load balancing for throughput and L7 for smart routing." |
| 一致性哈希 | consistent hashing ◆ | ⚠ 不是 consistency hashing | "Consistent hashing with virtual nodes keeps rebalancing to one over N of the keyspace." |
| 虚拟节点 | virtual node (vnode) | | "Each physical node owns a few hundred virtual nodes so the load stays even." |
| 轮询（负载均衡） | round robin ⚠ | ⚠ 与"轮询接口"的 polling 是两个词 | "Stateless round-robin dilutes the prefix cache and makes latency worse." |
| 轮询（反复查询） | polling | | "Replace the polling with SSE or a webhook." |
| 长轮询 | long polling | | "We pull with an ETag and long polling — 304s are nearly free." |
| 最少连接 | least connections | | "We swap least-connections for consistent hashing with an overflow threshold." |
| 延迟感知路由 | latency-aware routing | Peak EWMA 是具体算法 | "Peak EWMA is latency-aware: it weights backends by observed p90." |
| 负载倾斜 | load skew ⚠ | ⚠ 不是 tilt | "If the LB balances per connection, long-lived HTTP/2 connections cause severe load skew." |
| 会话粘性 | session affinity / sticky session ◆ | sticky routing 也通用 | "We need session affinity so the request lands on the instance holding the KV cache." |
| 连接池 | connection pool | | "Instance count times pool size has to stay under max_connections." |
| 空闲超时 | idle timeout | | "Client idle timeout has to be lower than the server's, or you get random 502s." |
| 多路复用 | multiplexing ♪ | MUL-ti-plex-ing | "HTTP/2 gives you multiplexing over a single TCP connection." |
| 连接迁移 | connection migration | QUIC 特性 | "QUIC supports connection migration, so switching from WiFi to 4G doesn't drop the stream." |
| 丢包 | packet loss ⚠ | ⚠ 不是 package loss（package 是包裹） | "On a lossy network, HTTP/2 can actually be slower than HTTP/1.1." |
| 载荷 | payload ⚠ | ⚠ 不是 load | "Protobuf payloads are 3 to 10x smaller than JSON." |
| 摘流量 | drain (traffic) ◆ | ⚠ 不是 remove traffic | "We drain the node instead of killing it outright." |
| 浅 / 深健康检查 | shallow / deep health check | | "The shallow check says the process is alive; that's all it says." |
| 就绪 / 存活探针 | readiness probe / liveness probe | K8s 术语 | "Cache warming happens in the readiness probe, not the liveness one." |
| 灰色故障 | gray failure ◆ | | "A shallow health check misses gray failures — the process is alive but serving garbage." |
| 熔断器 | circuit breaker ⚠ | ⚠ 不是 fuse。动词说 trip the breaker | "The circuit breaker belongs on the caller side, not the callee." |
| 半开 | half-open | | "In half-open we only allow a couple of low-cost idempotent probes." |
| 调用方 / 被调方 | caller / callee ♪ | callee 读 kal-EE | "Circuit breaking protects the caller from a slow dependency." |
| 出网 / 入网流量 | egress / ingress traffic ♪ | EE-gress / IN-gress | "Egress traffic to the internet costs 5 to 9 cents a gig — that drives the design." |
| 拓扑感知路由 | topology-aware routing | | "Topology-aware routing keeps traffic inside the AZ and kills the cross-AZ bill." |
| 东西向 / 南北向 | east-west / north-south traffic ◆ | | "gRPC for internal east-west traffic, REST for anything public." |
| 级联故障 | cascading failure ◆ | ⚠ 不是 avalanche | "No timeout means leaked threads, which means cascading failure." |
| 流式空闲超时 | stream idle timeout | 也说 no-progress timeout | "Use a stream idle timeout — 30 seconds with no new token — not a total-duration timeout." |
| 优雅关闭 | graceful shutdown | | "Killing one instance showed we never implemented graceful shutdown." |
| N+1 查询 | N-plus-one query problem | 读 "N plus one" | "You have to batch with DataLoader or you'll hit the N-plus-one query problem." |

---

## 7. 共识与协调

**fencing token** 是这一组的关键词 —— 中文没有统一译名，所以中国工程师往往整个概念说不出口。分布式锁题几乎必问。

| 中文 | English | 常见误译 / 注意 | 面试例句 |
|---|---|---|---|
| 共识 | consensus ♪ | kun-SEN-sus，不可数 | "Every level up the consistency ladder costs a consensus round." |
| Leader 选举 | leader election ◆ | ⚠ 现在统一用 leader/follower，master/slave 已被弃用 | "Leader election costs an election timeout plus one round trip — call it one to three seconds." |
| 任期 / 纪元 | term (Raft) / epoch ♪ | Raft 叫 term，多数系统叫 epoch，读 EP-uk | "Routing carries a monotonically increasing epoch; stale writes get a 409." |
| 租约 | lease ♪ | 读 leess，清音结尾 | "The lock is really a lease, and a lease can expire while you're paused in GC." |
| 隔离令牌 | fencing token ◆⚠ | **中文无统一译名，英文必须说 fencing token** | "Without a fencing token the lock isn't safe — a 40-second GC pause is all it takes." |
| 脑裂 | split-brain ◆ | ⚠ 不是 brain split，连字符不能丢 | "I'd rather be unwritable than split-brain." |
| 单调性检查 | monotonicity check | | "Safety comes from the storage-side monotonicity check, not from the lock service." |
| 复制日志 | replicated log | | "Consensus gives you an ordered replicated log." |
| 状态机复制 | state machine replication | | "Replicated log plus state machine replication equals a strongly consistent database." |
| 落盘 | fsync / durable write ⚠ | ⚠ 不说 drop to disk。fsync 读 "F-sync" | "Every write goes through the leader and has to fsync, so throughput caps out." |
| FLP 不可能性 | FLP impossibility | 读字母 | "FLP impossibility says no deterministic algorithm guarantees consensus in a fully async system." |
| 部分同步 | partial synchrony ♪ | SIN-kruh-nee | "Real algorithms assume partial synchrony — that's why they need timeouts." |
| 故障检测 | failure detection | | "Failure detection is timeout-based, which is why jitter triggers spurious elections." |
| 网络分区 | network partition ⚠ | ⚠ 不是 network split | "Under a network partition, neither half of a two-node cluster has a majority." |
| 拜占庭故障 | Byzantine fault ♪ | BIZ-un-teen，大写 B | "Byzantine faults only matter across trust boundaries." |
| 时钟漂移 | clock drift | 速率偏差 | "Quartz drift is about plus or minus 50 ppm — roughly four seconds a day." |
| 时钟偏移 | clock skew | 瞬时差值 | "Allow at most 60 seconds of clock skew on exp and nbf." |
| 时钟跳变 | clock step / clock jump | | "NTP can step the clock backwards, so t2 minus t1 goes negative." |
| 时钟回拨 | clock rollback | | "Clock rollback is Snowflake's only correctness risk — you block or switch worker IDs, never keep issuing." |
| 单调时钟 | monotonic clock ⚠ | ⚠ 不是 monotonous | "Always measure durations with a monotonic clock, never wall clock." |
| 逻辑时钟 | logical clock | | "Logical clocks give you causality without trusting the wall clock." |
| 向量时钟 | vector clock | | "Vector clocks can actually detect concurrent updates; Lamport clocks can't." |
| 混合逻辑时钟 | hybrid logical clock (HLC) | | "If you need cross-node causal order, HLC is the default answer." |
| 因果先于 | happens-before ◆ | 固定写法带连字符 | "Lamport clocks guarantee happens-before implies a smaller timestamp — but not the converse." |
| 时钟不确定性 | clock uncertainty | TrueTime 的 epsilon | "TrueTime turns clock uncertainty into a bounded interval, usually under 7 milliseconds." |
| 分布式锁 | distributed lock | | "Most distributed lock implementations aren't actually safe." |
| 无协调 | coordination-free ◆ | | "ULIDs are coordination-free and still roughly sortable." |
| 租约配额 | leased quota / quota pre-allocation | | "Each instance leases a slice of the global quota and decrements locally." |
| 单点故障 | single point of failure (SPOF) | 读 "spoff" 或字母 | "DNS is the most underrated single point of failure." |

---

## 8. 架构与领域

这一组最容易读错：**choreography**（编舞）重音在第三音节，**strangler fig**（绞杀榕）不能只说 strangler。

| 中文 | English | 常见误译 / 注意 | 面试例句 |
|---|---|---|---|
| 限界上下文 | bounded context ◆ | ⚠ 不是 boundary context | "The same word meaning different things is the signal for a bounded context." |
| 防腐层 | anti-corruption layer ◆ | ⚠ 缩写 ACL 会和访问控制列表撞车，口语说全称 | "Put an anti-corruption layer in front of the legacy system." |
| 绞杀者模式 | strangler fig pattern ◆♪ | ⚠ 是 strangler **fig**（绞杀榕），Fowler 后来改的名 | "We migrated with a strangler fig: facade first, then shift traffic one percent at a time." |
| 编排 | orchestration ♪ | or-kess-TRAY-shun | "Use orchestration when there's a real business process with compensation." |
| 编舞 | choreography ♪⚠ | kor-ee-OG-ruh-fee，**重音第三音节** | "Choreography stops being maintainable past three hops." |
| 模块化单体 | modular monolith ◆ | | "I'd start with a modular monolith and split it when the team hits twenty people." |
| 分布式单体 | distributed monolith ◆ | | "If you must deploy them in order, you've built a distributed monolith." |
| 服务边界 | service boundary | | "Getting the service boundaries right matters more than the deployment topology." |
| 高内聚 / 低耦合 | high cohesion / loose coupling ⚠ | ⚠ 是 **loose** coupling，不是 low coupling | "High cohesion means one reason to change." |
| 同步耦合 | synchronous coupling | | "Shared code is synchronous coupling in disguise — prefer duplication." |
| 数据所有权 | data ownership | | "Every piece of data needs exactly one service that owns it." |
| 康威定律 | Conway's Law | 撇号别丢 | "By Conway's Law, five services and eight engineers means nothing has a real owner." |
| 逆康威策略 | Inverse Conway Maneuver ◆♪ | maneuver 读 muh-NOO-ver | "We used the Inverse Conway Maneuver: design the architecture, then reorg to match." |
| 认知负荷 | cognitive load | | "One team owning 15 services means shallow understanding of all 15 — that's cognitive load." |
| 平台团队 / 赋能团队 | platform team / enabling team | Team Topologies 术语 | "The platform team's job is to lower the cognitive load on stream-aligned teams." |
| 独立部署 | independent deployability | | "If your deploys aren't blocked on another team, you don't need independent deployability." |
| 事件驱动 | event-driven | | "We went event-driven to decouple the write path from the consumers." |
| 事件溯源 | event sourcing | | "Event sourcing without an audit requirement is complexity for its own sake." |
| 聚合（DDD） | aggregate ♪ | 名词读 AG-ri-gut，动词读 AG-ri-gate | "The consistency boundary in event sourcing is a single aggregate." |
| 投影 | projection | | "A projection is a pure function of the event stream, so replaying it is safe." |
| 读模型 | read model | | "A read model you can't rebuild is a read model you can't fix." |
| 物化视图 | materialized view | | "Start with a materialized view before you build a separate read store." |
| 真相源 | source of truth ◆ | 常说 single source of truth (SSOT) | "The raw event lake is the single source of truth; aggregates are derived." |
| 预留 | reservation | | "We put a reservation with a TTL in front of the irreversible step." |
| 反模式 | anti-pattern | 连字符 | "Full-system event sourcing is the classic anti-pattern here." |
| 单向门 / 双向门 | one-way door / two-way door ◆ | Amazon 行话，面试官普遍认 | "That's a two-way door — decide fast, don't hold a meeting." |
| 可逆性 | reversibility ♪ | ri-VER-suh-BIL-i-tee | "I rank decisions by reversibility and spend my time on the irreversible ones." |
| 架构决策记录 | Architecture Decision Record (ADR) | 读字母 | "Every one-way door gets an ADR with the rejected alternatives written down." |
| 演进式架构 | evolutionary architecture | | "This is evolutionary architecture: design for 10x, mark where it breaks." |
| 复杂度预算 | complexity budget ◆ | | "That's over our complexity budget for a team of four with no ops experience." |
| 多语言持久化 | polyglot persistence ♪ | POL-ee-glot | "Polyglot persistence really costs you N backup, upgrade, and on-call playbooks." |
| 自建 vs 采购 | build vs buy | ⚠ 不是 self-build | "My build-versus-buy line is: build metering, buy rating, tax, and invoicing." |

---

## 9. 多租户与 SaaS

**blast radius / noisy neighbor / bulkhead / control plane** 这四个词几乎是多租户题的通行证。说不出来，面试官会默认你没做过 SaaS。

| 中文 | English | 常见误译 / 注意 | 面试例句 |
|---|---|---|---|
| 多租户 | multi-tenancy / multi-tenant ♪ | tenant 读 TEN-unt，重音第一 | "We do multi-tenancy with a pooled default and silos for the whales." |
| 租户 | tenant | ⚠ 不等于 customer。tenant 是隔离单位，customer 是商务概念 | "A single noisy tenant shouldn't be able to burn the whole monthly budget." |
| 噪音邻居 | noisy neighbor ◆ | | "Per-tenant bulkheads keep one noisy neighbor from starving everyone." |
| 爆炸半径 | blast radius ◆ | ⚠ 不是 explosion radius | "Cells don't raise your SLA, they shrink the blast radius." |
| 单元化 / Cell 架构 | cell-based architecture ◆ | ⚠ 不说 unitization | "A cell-based architecture caps the blast radius at one over N." |
| 控制面 / 数据面 | control plane / data plane ◆ | ⚠ plane 不是 plain | "The control plane is never on the data plane's critical path." |
| 协调循环 | reconcile loop ♪ | REK-un-sile | "Every reconcile step has to be idempotent — check first, then create." |
| 漂移 | drift | | "A spike in drift means someone is bypassing the control plane." |
| 开通 | provisioning ♪ | pruh-VIZH-un-ing | "Tenant provisioning is a six-step saga, not a single API call." |
| 计量 | metering ⚠ | ⚠ 不是 measurement。计费用量固定说 metering | "Metering is decoupled from the business write path." |
| 用量计费 | usage-based pricing (UBP) | | "UBP doesn't fix margin; it just moves cost variance onto the customer." |
| 超额计费 | overage billing ♪ | OH-ver-ij | "Subscription plus overage billing plus a hard cap is the only pricing that survives the P99 user." |
| 对账 | reconciliation ♪ | rek-un-sil-ee-AY-shun | "We run three-way reconciliation daily, with a 0.5 percent tolerance." |
| 收入漏损 | revenue leakage ◆ | | "A 0.5 percent event loss is silent revenue leakage of fifty thousand dollars a year." |
| 配额 | quota ♪ | 读 KWOH-tuh | "Set the quota in dollars, because one request can be a tenth of a cent or five dollars." |
| 软限额 / 硬限额 | soft limit / hard limit | | "The soft limit degrades and alerts; the hard limit rejects." |
| 令牌桶 | token bucket | 允许突发 | "Per-tenant token bucket with burst set at ten times steady state." |
| 漏桶 | leaky bucket | 平滑输出 | "A leaky bucket smooths output rate; a token bucket allows a burst." |
| 隔板 | bulkhead ◆⚠ | 船舱隔壁。⚠ 不是 partition / isolation board | "We bulkhead the connection pool so no tenant can take more than 20 percent." |
| 行级安全 | Row-Level Security (RLS) | | "RLS is my last line of defense, not my only one." |
| 池化 vs 独立栈 | pooled vs silo ◆ | AWS SaaS 术语：pool / silo / bridge | "Pooled tenants cost cents a month; a silo stack is hundreds of dollars." |
| 大客户 | whale ◆ | 口语直接说 "a whale tenant" | "Moving the whale to a dedicated shard is the right answer ninety percent of the time." |
| 升舱 | tier promotion | | "Write the tier-promotion thresholds into the runbook, don't decide them live." |
| 逃生舱 | escape hatch ◆ | | "Leave the sub-key column in the schema on day one as an escape hatch for whales." |
| 数据驻留 | data residency ⚠ | ⚠ 不是 data resident。与 data sovereignty 不同 | "EU data residency is a sales blocker, so we run a dedicated EU cell." |
| 跨租户越权 | cross-tenant access | | "Swap the tenant_id in the URL — that's the cross-tenant access test." |
| 删除权 | right to erasure | 也说 right to be forgotten | "The right to erasure means dropping the tenant's segments and destroying the DEK, not queuing deletes." |
| 加密粉碎 | crypto-shredding ◆ | | "One DEK per tenant is the sweet spot for crypto-shredding." |
| 最小权限 | least privilege ⚠ | ⚠ 不是 minimum permission | "CC6.1 is least privilege — you show them the authz decision log." |
| 默认拒绝 | default deny | | "The policy engine's fallback branch is default deny." |
| 关系图授权 | ReBAC / relationship graph | Zanzibar 模型 | "The moment you need hierarchy, inheritance and sharing, go ReBAC." |

---

## 10. 可靠性与运维

**page / on-call / toil / postmortem** 是四个中文母语者几乎必错的词。尤其 page —— 半夜被叫醒是 "get paged"，不是 "be called"。

| 中文 | English | 常见误译 / 注意 | 面试例句 |
|---|---|---|---|
| SLI / SLO / SLA | service level indicator / objective / agreement | 读字母 | "We define SLOs on user journeys, not on individual services." |
| 错误预算 | error budget ◆ | | "We burned 80 percent of our monthly error budget in one incident." |
| 燃尽率 | burn rate ◆ | ⚠ 不是 burnout rate | "A 14.4x burn rate exhausts the monthly budget in about 50 hours." |
| 多窗口多燃尽率告警 | multi-window multi-burn-rate alerting ◆ | | "We page on multi-window multi-burn-rate, not on a raw error-rate threshold." |
| 高基数 | high cardinality ♪ | KAR-di-NAL-i-tee | "That label is high cardinality — it belongs in traces, not metrics." |
| 症状告警 | symptom-based alerting | | "Page on symptoms, put causes in the runbook." |
| 告警疲劳 | alert fatigue ♪ | fuh-TEEG | "More than two pages per shift and you get alert fatigue." |
| 寻呼 / 半夜叫醒 | page (v.) ⚠ | ⚠ **不是 be called**。"I got paged at 3 a.m." | "Nothing below Sev-2 is allowed to page at night." |
| 值班 | on-call ⚠ | ⚠ 不是 duty / on duty | "Rotations need at least six people to stay sustainable." |
| 运维负担 | toil ◆⚠ | SRE 专有词，读 toyl。⚠ 不是 operation burden | "Half the on-call time has to go into removing toil." |
| 运行手册 | runbook / playbook ⚠ | ⚠ 不是 plan | "A runbook is only correct if it's been through a drill." |
| 止血 | stop the bleeding ◆ | 正式场合说 mitigate | "First we stop the bleeding, then we look for the root cause." |
| 缓解 | mitigation | | "Ninety percent of MTTR improvement comes from mitigation, not detection." |
| 复盘 | postmortem ◆⚠ | ⚠ 不是 review。无责复盘 = blameless postmortem | "Sev-1 requires a postmortem within five working days." |
| 无责的 | blameless ◆ | | "Escalating early is always blameless." |
| 根因 | root cause | 动词形式 root-cause it | "Root cause analysis happens in the postmortem, not during the incident." |
| 事故等级 | severity level ♪ | Sev-1 读 "sev one" | "Severity levels are defined by user impact, not technical difficulty." |
| 宣告事故 | declare an incident ◆ | | "Anyone can declare an incident — the cost has to be near zero." |
| 升级（找人） | escalation ⚠ | ⚠ 与版本 upgrade 完全不同 | "No progress for three turns triggers escalation to a human." |
| 混沌工程 | chaos engineering ♪ | KAY-oss | "Chaos engineering is hypothesis-driven, not random breakage." |
| 演练 | game day / drill / fire drill | | "A region-level game day needs exec sign-off." |
| 重试放大 | retry amplification ◆ | | "Three layers of three retries is 27x retry amplification." |
| 重试预算 | retry budget | | "The retry budget caps retries at 10 percent of total traffic." |
| 全抖动 | full jitter ◆ | | "Full jitter: sleep equals rand of zero to min of cap and base times two to the n." |
| 自适应限流 | adaptive throttling | | "Client-side adaptive throttling kicks in as soon as rejections rise." |
| 线程池 / 信号量隔离 | thread pool / semaphore isolation ♪ | SEM-uh-for | "Thread pool isolation is the only way to actually enforce a timeout." |
| 优雅降级 | graceful degradation ⚠ | ⚠ **不是 downgrade**（那是版本降级） | "Graceful degradation must be defined, not best-effort." |
| 静态稳定性 | static stability ◆ | AWS 行话 | "Static stability means zero control-plane dependency during a failure." |
| 失败域 | failure domain | | "List everything the replicas share — that's your real failure domain." |
| 灾难恢复 | disaster recovery (DR) | RTO / RPO 读字母 | "An untested RTO is a wish, not a disaster recovery capability." |
| 恢复演练 | restore drill / recovery drill | | "Backups you've never restored aren't backups — do the restore drill." |
| 巴士因子 | bus factor ◆ | | "We run a bus-factor drill twice a year." |
| 状态页 | status page | | "The status page has to be hosted outside your own infrastructure." |

---

## 11. 扩展与迁移

中文一个"扩容"对应英文两个词：**scale up**（换大机器）和 **scale out**（加机器）。说错方向，整段论证就废了。

| 中文 | English | 常见误译 / 注意 | 面试例句 |
|---|---|---|---|
| 分片 | sharding ⚠ | ⚠ 见第 14 组。名词 shard，动作 shard / re-shard | "Sharding is a quarter of work and a one-way door." |
| 分片键 | shard key / partition key | | "The shard key is the one-way door in this design, so I'll spend extra time on it." |
| 逻辑分片 | logical shard | | "Fix 1024 logical shards up front; only the physical database count changes." |
| 再平衡 | rebalancing | | "Rebalancing moves whole logical shards, one at a time." |
| 热点 / 热分片 / 热 key | hotspot / hot shard / hot key ◆ | | "One hot tenant turns into a hot shard and pins one node at 100 percent." |
| 打散 | salt the key / spread ◆⚠ | 加随机后缀固定叫 salt the key | "For the whales we salt the key to spread them across shards." |
| 数据倾斜 | data skew ⚠ | ⚠ 不是 tilt / deviation | "With tenant sharding the data skew is decided by your customers, not by you." |
| 单调递增键 | monotonically increasing key | | "A monotonically increasing shard key means only the last shard ever writes." |
| 垂直扩容 | scale up / vertical scaling ⚠ | **换大机器** | "Vertical scaling is the only option that adds no architectural complexity." |
| 水平扩展 | scale out / horizontal scaling ⚠ | **加机器**。中文都叫"扩容"，英文必须分 | "Three hard signals tell you it's time to scale out." |
| 按域垂直拆库 | functional partitioning | | "Functional partitioning buys 2 to 5x and is far simpler than sharding." |
| 读扩展 / 写扩展 | read scaling / write scaling | | "Read scaling is easy; write scaling is where the real choices are." |
| 只读副本 | read replica ⚠ | ⚠ 主库现在叫 primary，不叫 master | "Read replicas scale reads linearly but break read-your-writes." |
| 复制延迟 | replication lag ⚠ | ⚠ 不是 replication delay | "A 404 right after a write is usually just replication lag." |
| 故障转移 | failover (n.) / fail over (v.) | 名词一个词，动词两个词 | "Failing traffic over is easy; failing the data over is the hard part." |
| 双写 | dual write | | "During migration we dual-write, and the old store stays the source of truth." |
| 回填 | backfill | 名词动词同形 | "Throttle the backfill so the old database CPU rises less than 10 percent." |
| 影子读 | shadow read | | "Shadow reads have to run clean for three to seven days before we cut over." |
| 影子流量 | shadow traffic / traffic mirroring | | "Shadow traffic must bypass every cache or it poisons the eval." |
| 切流 | cut over (v.) / cutover (n.) ⚠ | ⚠ 不是 switch traffic（太含糊） | "We take a short read-only window to avoid split-brain at cutover." |
| 灰度切读 | progressive rollout of reads | | "Gradual rollout of reads by tenant: one, ten, fifty, a hundred percent." |
| 扩展-收缩法 | expand and contract ◆ | 也叫 parallel change | "We use expand-and-contract: add column, dual-write, backfill, switch, drop." |
| 前滚 | roll forward | | "After the contract step you can't roll back, only roll forward." |
| 回滚条件 | rollback criteria ♪ | kry-TEER-ee-uh（复数）；单数 criterion | "Every step of the migration has explicit rollback criteria written down beforehand." |
| 观察期 | bake time ◆ | ⚠ 不是 observation period | "Give each step a bake time of days to weeks before moving on." |
| 冻结窗口 | freeze window | | "Target a freeze window under thirty seconds." |
| 幽灵数据 | ghost record / resurrected row | | "The backfill resurrected deleted rows — classic ghost records." |
| 限速 | throttling | | "Throttle background jobs so they can't compete with user traffic." |
| 冷热分层 | hot/cold tiering | | "Hot/cold tiering cuts storage cost by an order of magnitude." |
| 在线 DDL | online schema change / online DDL | | "Online DDL still needs a lock_timeout." |
| 锁队列 | lock queue | | "Postgres lock queues are FIFO, so a waiting DDL blocks every later select." |

---

## 12. AI 推理

这一组的词几乎全是 2023 年后才固化的，**没有中文标准译名**，所以直接背英文更省事。**goodput** 是一个词，不是 good throughput。

| 中文 | English | 常见误译 / 注意 | 面试例句 |
|---|---|---|---|
| 首 token 延迟 | time to first token (TTFT) ◆ | 读 "T-T-F-T" 或全称 | "TTFT needs bucketing by input length or the curve is meaningless." |
| 每 token 输出时延 | time per output token (TPOT) ◆ | 同义词 inter-token latency (ITL) | "Two configs both meeting TTFT under 1.5 seconds can differ 10x on P95 TPOT." |
| 有效吞吐 | goodput ◆⚠ | ⚠ **一个词**，不是 good throughput | "Capacity planning only works on goodput — requests per second meeting both TTFT and TPOT." |
| 预填充 | prefill ◆ | 不翻译 | "Prefill is compute-bound; decode is memory-bandwidth-bound." |
| 解码 | decode | | "Decoding is serial, so a 1,000-token answer takes 10 to 30 seconds — it has to stream." |
| PD 分离 | prefill-decode disaggregation ◆♪ | dis-uh-gruh-GAY-shun。口语说 "P-D disagg" | "Prefill-decode disaggregation isn't a default win; without RDMA it's slower than aggregated." |
| 聚合部署 | aggregated deployment | | "For short prompts and low concurrency, aggregated deployment is simpler and faster." |
| 自回归推理 | autoregressive inference ♪ | inference 读 IN-fer-unce，**重音第一** | "Autoregressive inference splits into two phases with opposite hardware profiles." |
| 算术强度 | arithmetic intensity ♪ | 形容词读 uh-RITH-MET-ik | "In fp16, decode has an arithmetic intensity of roughly the batch size — that's why batching is the only lever." |
| KV 缓存 | KV cache ◆ | 读 "K-V cache" | "A shared KV cache is the biggest cross-tenant leakage surface we have." |
| 前缀缓存 | prefix caching ◆ | | "Prefix caching is the dominant cost lever — cache reads are about 10 percent of input price." |
| 前缀重叠率 | prefix overlap ratio | | "Measure your prefix overlap ratio before you turn prefix caching on." |
| 字节稳定前缀 | byte-stable prefix ◆ | | "Step one is a byte-stable prefix — no timestamps, no UUIDs in the system prompt." |
| 静默 cache miss | silent cache miss ◆ | | "A silent cache miss has no error and no log; the only symptom is the bill." |
| 最小可缓存前缀 | minimum cacheable prefix | | "Below the minimum cacheable prefix nothing is cached, and you get no error at all." |
| 缓存感知路由 | cache-aware routing ◆ | 也说 prefix-aware routing | "Cache-aware routing turned P90 TTFT from ninety seconds into half a second." |
| 有状态路由 | stateful routing | | "An LLM gateway has to be a stateful router, not a stateless L7 load balancer." |
| 连续批处理 | continuous batching ◆ | 也叫 in-flight batching | "Continuous batching is what gets you about 400 tokens per second aggregate on one card." |
| 分页注意力 | PagedAttention | 产品名，一个词，两个大写 | "PagedAttention keeps a block table that maps logical to physical KV blocks." |
| 抢占 | preemption ♪ | pree-EMP-shun；动词 preempt | "Rising preemption counts mean the scheduler is evicting and recomputing live requests." |
| 重算 | recompute | | "When we preempt a sequence we have to recompute its KV from scratch." |
| 投机解码 | speculative decoding ◆ | | "Speculative decoding gives about 1.7x at low concurrency and roughly nothing at batch 48." |
| 量化 | quantization ♪ | kwon-ti-ZAY-shun。int8 读 "int eight" | "int8 scalar quantization gets us another 4x on memory." |
| 张量 / 流水 / 专家并行 | tensor / pipeline / expert parallelism | | "Tensor parallelism must stay inside the NVLink domain." |
| 显存 | GPU memory / VRAM ⚠ | ⚠ 不说 video memory。带宽受限说 "we're HBM-bound" | "Concurrency is capped by GPU memory, not CPU." |
| 约束解码 | constrained decoding | 也说 structured output | "Constrained decoding pushes schema compliance from about seventy percent to near a hundred." |
| 分词器 | tokenizer ♪ | TOH-kun-eye-zer | "Token counts aren't convertible across providers — each tokenizer is different." |
| 思考 token | reasoning tokens | | "Reasoning tokens are billed as output whether or not you expose them." |
| 模型分级 / 降档 | model tiering / tier down ◆ | | "The savings come from context length, cache hit rate, and model tiering for sub-agents." |
| 级联路由 | cascade routing | | "A cascade is cheaper only if the escalation rate stays below one minus the price ratio." |
| 蒸馏 | distillation ♪ | dis-tuh-LAY-shun | "Model drift kills distillation — your model doesn't improve when the frontier does." |
| 盈亏平衡 | break-even ⚠ | ⚠ 不是 balance point | "Self-hosted GPU break-even is brutally sensitive to utilization." |
| 单位经济模型 | unit economics ◆ | | "The only unit-economics metric that matters is dollars per successful task." |

---

## 13. AI Agent 与检索

**context rot / lethal trifecta / context firewall / runaway fan-out** —— 这几个词 2026 年的面试官全都在用。说得出来，等于证明你在跟进。

| 中文 | English | 常见误译 / 注意 | 面试例句 |
|---|---|---|---|
| 上下文工程 | context engineering ◆ | | "Context engineering covers system prompt, tools, retrieval, history and memory." |
| 上下文窗口 | context window | | "The context window is a budget, not a container." |
| 上下文腐化 | context rot ◆⚠ | ⚠ 不是 context decay | "Context rot means performance degrades before the window is even full." |
| 上下文污染 | context pollution | 恶意的那种叫 context poisoning | "Tool results are the biggest source of context pollution, not the chat history." |
| 上下文压缩 | compaction ⚠ | ⚠ Agent 语境说 compaction，不是 compression | "Start compaction once the working set passes half the window." |
| 外部化状态 | externalize state ◆ | | "Long tasks must externalize state to files, not rely on session survival." |
| 只追加 | append-only | | "Structure the context as a byte-stable prefix plus an append-only tail." |
| 工具调用 | tool calling / function calling ⚠ | ⚠ 不是 tool invoke。两个说法都通用 | "Every provider has a different tool calling format — that's why you need one abstraction." |
| 工具定义 | tool definition | | "Changing a single tool definition invalidates the entire cached prefix." |
| 并行工具调用 | parallel tool calls | | "Parallel tool calls are for reads and reviews; writes stay single-threaded." |
| 轮次 | turn ⚠ | ⚠ 不是 round。上限固定说 max turns | "Agent cost is quadratic in turns, so cutting turns beats swapping models." |
| 终止条件 | termination criteria / stop condition | | "Termination is an OR over four independent gates." |
| 轨迹 | trajectory ♪ | truh-JEK-tur-ee | "Outcome-only eval is blind to multi-agent failures; you need trajectory-level eval." |
| 子代理 | sub-agent / subagent | | "The sub-agent burns fifty thousand tokens and returns a one-thousand-token summary." |
| 上下文防火墙 | context firewall ◆ | | "Subagents act as a context firewall — they hand back a summary, not the transcript." |
| 扇出配额 | fan-out quota | | "My fan-out quota is two to four subagents for a comparison, ten-plus for deep research." |
| 失控扇出 | runaway fan-out ◆ | | "Runaway fan-out is the most expensive multi-agent incident shape." |
| 过早终止 | premature termination ♪ | 美式读 pree-muh-CHOOR | "Premature termination and missing verification are the top production failure class." |
| 任务跑偏 | task derailment ◆ | | "Task derailment shows up as an agent quietly solving a different problem." |
| 检索增强生成 | retrieval-augmented generation (RAG) | 读 "rag" | "RAG isn't dead — it's the thing that keeps the context small enough to work." |
| 混合检索 | hybrid retrieval / hybrid search | | "Hybrid retrieval is the default — dense embeddings fail systematically on error codes and SKUs." |
| 词法检索 | lexical search ♪ | LEK-si-kul | "Error codes, SKUs and controlled vocabulary all need BM25." |
| 重排 | reranking ⚠ | ⚠ 不是 re-sort | "Retrieve top-100, rerank 30 to 50, return top-20." |
| 分块 | chunking | | "Chunking granularity decides your incremental update granularity." |
| 重建索引 | reindex | | "Changing the embedding model forces a full reindex." |
| 近似最近邻 | approximate nearest neighbor (ANN) | 读 "A-N-N" | "ANN trades recall for speed; we tune ef_search to hit 0.95 recall." |
| 召回-延迟取舍 | recall-latency tradeoff | | "HNSW is fundamentally a recall-latency tradeoff tuned by efSearch." |
| 幻觉 | hallucination ♪ | huh-loo-suh-NAY-shun | "A hallucinated answer looks perfectly healthy to traditional SLIs." |
| 弃答 | abstain / abstention ♪ | ab-STAIN | "Claude models tend to abstain when uncertain; others hallucinate more." |
| 提示注入 | prompt injection ◆ | ⚠ 不是 prompt attack | "Prompt injection is an architecture problem, not a bug you patch with a classifier." |
| 间接注入 | indirect injection | | "Indirect injection comes through web pages, emails, and tool return values." |
| 致命三要素 | lethal trifecta ◆♪ | try-FEK-tuh | "Once all three of the lethal trifecta hold, data exfiltration is inevitable." |
| 护栏 | guardrail ⚠ | ⚠ 概率性防御不能当边界用 | "The guardrail is defense in depth; egress control is the actual boundary." |
| 出口管控 | egress control ◆ | | "Egress control is capability granting, not destination filtering." |
| 人在回路 | human-in-the-loop (HITL) ◆ | 连字符全带 | "Irreversible actions require a human-in-the-loop confirmation, defined statically at design time." |
| 审批疲劳 | approval fatigue ◆ | | "A 93 percent auto-approve rate is approval fatigue, not security." |
| 记忆投毒 | memory poisoning ◆ | | "Memory poisoning outlasts ordinary prompt injection because it persists." |
| 评测集 / 评测框架 | eval set / eval harness ⚠ | 口语说 "our evals"（复数当名词） | "The eval set comes from production traces, not a public benchmark." |
| LLM 评审 | LLM-as-judge ◆ | | "Swapping the judge model means swapping the standard — you have to redo the calibration." |
| 回归集 | regression set | | "We keep a versioned regression set of production trajectories." |
| 发布门禁 | release gate | | "Public benchmarks can't be the release gate; the gate is our own regression set." |
| 成本熔断 | cost circuit breaker ◆ | | "The cost circuit breaker is an availability control, not just a finance one." |

---

## 14. 最容易说错的 30 个

**这一节是本文件的核心。**上面 13 组是"你不知道的词"，这一节是"**你以为你知道、但一说出口就露馅的词**"。

### 14.1 名词类（15 个）

| 中文 | ❌ 中式英文 | ✅ 地道说法 | 面试例句 |
|---|---|---|---|
| 分库分表 | split database / divide table | **sharding** / horizontal partitioning | "A single Postgres primary won't hold this, so we either batch writes or start sharding." |
| 埋点 / 打点 | hit point / bury point | **instrumentation** / telemetry | "Put the instrumentation as close as possible to the undeniable fact." |
| 压测 | pressure test | **load test**（目标负载） / **stress test**（压到崩） | "We load-tested to 3x peak and the connection pool saturated first." |
| 灰度 | gray release | **canary** / progressive rollout | "Config changes need the same progressive rollout as code — most outages are config, not code." |
| 兜底 | bottom guarantee | **fallback** / safety net | "Every breaker needs a defined fallback — otherwise it just converts slow errors into fast ones." |
| 链路 | link | **request path** / call chain / trace | "A classic request trace fans out three to five hops." |
| 雪崩 | avalanche | **cascading failure**（缓存场景说 mass expiration） | "Without backpressure one slow dependency turns into a cascading failure." |
| 限流 | limit current ⚠ | **rate limiting** / throttling | "Rate limiting is about fairness; load shedding is about survival." |
| 熔断 | fuse / melt break | **circuit breaker** | "The circuit breaker opens above a 50 percent error rate with at least 20 samples." |
| 预案 | plan | **runbook** / playbook | "For a shard-key change I'd follow a six-step migration playbook, not a single cutover." |
| 值班 | duty / on duty | **on-call** | "What's the on-call load on this team, roughly?" |
| 复盘 | review / summary | **postmortem**（无责的叫 blameless postmortem） | "Sev-1 requires a blameless postmortem within five working days." |
| 主备 / 主从 | master-slave ⚠ | **primary–replica** / leader–follower / active–standby | "Every write goes through the leader and has to fsync, so throughput caps out." |
| 双活 | double live | **active–active** | "The hard part of active-active isn't routing, it's write conflicts." |
| 毛刺 | burr / glitch | **latency spike** / p99 spike | "Compaction causes latency jitter — you see p99 spikes every few minutes." |

### 14.2 动作与搭配类（15 个）

| 中文 | ❌ 中式英文 | ✅ 地道说法 | 面试例句 |
|---|---|---|---|
| 上线 | go online / online the feature | **ship** / **roll out** / go live / deploy to production | "We ship behind a flag, then roll out one percent, ten, fifty, a hundred." |
| 下线（服务） | offline it | **decommission** / sunset / take it out of service | "We sunset the old endpoint after 180 days of brownouts." |
| 回滚 | roll back the version | **roll back**（动词） / **rollback**（名词） / revert | "One-click rollback is the highest-ROI mitigation we have." |
| 削峰填谷 | cut peak fill valley | **smooth traffic spikes** / flatten the peak | "Going async buys us the right to smooth traffic spikes instead of provisioning for them." |
| 扛住流量 | resist the traffic | **handle** / **absorb** / sustain X QPS | "A single primary sustains about 20k TPS; past that we shard." |
| 摘节点 | pick off the node | **drain** the node / take it out of rotation | "We drain the node instead of killing it outright." |
| 打散 key | break up the key | **salt the key** / spread it across suffixes | "For the whales we salt the key to spread them across shards." |
| 降级 | downgrade ⚠ | **degrade gracefully** / fall back to | "At 100 percent of budget we degrade gracefully to a cheaper model instead of erroring out." |
| 定位问题 | locate the problem | **narrow it down** / isolate the cause / root-cause it | "Knowing which change caused it is enough to mitigate — we root-cause it in the postmortem." |
| 排查 | check the problem | **troubleshoot** / debug / triage | "Debugging an agent is trace replay plus error analysis, not a stack trace." |
| 报警 | alarm / report to police | **alert**（告警） / **page**（叫醒人） | "If the alert isn't actionable, delete it — don't retune it." |
| 兼容旧版本 | compatible with old version | **backward-compatible** | "Every release has to keep N-minus-one backward compatibility." |
| 联调 | joint debugging | **integration testing** / wire it up end to end | "We wire it up end to end behind a flag before anyone sees it." |
| 性能很好 | the performance is very good ⚠ | 给数字：**p99 is under 50 ms** / it sustains 20k QPS | "p99 is under 50 milliseconds at 20k QPS, with 30 percent headroom." |
| 数据量很大 | the data is very big ⚠ | 给数字：**we're at 2 TB, growing 5x a year** | "We're at 2 terabytes growing 5x a year, so the shard-key decision can't wait." |

> **两条通用规则**，比这 30 个词更重要：
> 1. **凡是能用数字说的，都不要用形容词。** "very good / very big / very fast" 在系统设计面试里等于没说。这是全书反复强调的评分点，在英文面试里更致命 —— 形容词是母语者的强项，数字是你的。
> 2. **拿不准就描述现象，不要硬翻。** "缓存穿透"翻不出来没关系，说 "lookups for keys that don't exist never hit the cache" —— 对方立刻懂，而且显得你在描述真实系统而不是背名词。

---

## 发音与重音提醒

**标注方式**：音节用连字符分开，**大写的那个音节是重音**。不用国际音标 —— 面试时你需要的是能立刻模仿的东西。

### A. 全场最高频的十个坑

| 词 | 读法 | 说明 |
|---|---|---|
| **idempotent** | eye-dem-**POH**-tent | 重音在第三音节。名词 idempotency = eye-dem-**POH**-ten-see。⚠ 不是 "ai-de-mo-tent"，也不要读成四个平音节 |
| **quorum** | **KWOR**-um | **两个音节**，不是三个。⚠ 不要读成 "qu-o-rum" |
| **cache** | 就读作 **cash** | 一个音节，和"现金"完全同音。⚠ 不是 "ca-che"、不是 "cah-SHAY"（那是 cachet，另一个词） |
| **queue** | 就读作字母 **Q** | 一个音节。queued = "cued"，queueing = "**CUE**-ing"。⚠ 五个字母只发一个音 |
| **latency** | **LAY**-ten-see | 重音第一音节。⚠ 不是 "la-TEN-cy" |
| **schema** | **SKEE**-muh | ⚠ 不是 "she-ma"。复数 schemas 或 schemata (skee-**MAH**-tuh) |
| **canary** | kuh-**NAIR**-ee | **重音第二音节**。⚠ 中国工程师普遍读成 "CAN-a-ry"，错得很明显 |
| **choreography** | kor-ee-**OG**-ruh-fee | 重音第三音节。形容词 choreographed = **KOR**-ee-uh-graft |
| **inference** | **IN**-fer-unce | 重音第一。⚠ 不是 "in-FER-ence"（动词 infer 才是 in-**FUR**） |
| **percentile** | per-**SEN**-tile | p99 口语读 "P ninety-nine"，正式说 "the ninety-ninth percentile" |

### B. 产品与技术名

| 词 | 读法 | 说明 |
|---|---|---|
| **Kubernetes** | koo-ber-**NET**-eez | 四个音节。缩写 K8s 读 "kates" 或 "K-eights" |
| **nginx** | "**engine**-X" | ⚠ 官方读法。不是 "en-jinx"、不是 "N-G-I-N-X" |
| **Kafka** | **KAHF**-kuh | 同作家卡夫卡 |
| **Redis** | **RED**-iss | 重音第一，⚠ 不是 "ree-DIS" |
| **Postgres** | **POST**-gres | 口语几乎不说全称。PostgreSQL = "post-gres-Q-L" |
| **etcd** | "et-see-dee" | 来自 `/etc` + distributed，逐字母读后半段 |
| **Envoy / Istio** | **EN**-voy / **ISS**-tee-oh | |
| **Prometheus / Grafana** | pro-**MEE**-thee-us / gruh-**FAH**-nuh | |
| **Paxos / Raft** | **PACK**-sohs / raft | Raft 就读"木筏" |
| **CUDA / NVMe** | **COO**-duh / "N-V-M-E" | |
| **YAML / JSON** | **YAM**-ul（押 camel）/ **JAY**-sun | |
| **JWT** | "**JOT**" 或字母 J-W-T | RFC 建议读 jot，两种都常见 |
| **OAuth** | "**OH**-auth" | 两音节，oh + awth |
| **gRPC / CDN / QPS** | 逐字母读 | "gee-R-P-C"、"C-D-N"、"Q-P-S" |
| **SQL** | "S-Q-L" 或 "**SEE**-quel" | 两种都可以，但同一场面试里保持一致 |

### C. 容易读错重音的长词

| 词 | 读法 |
|---|---|
| **asynchronous** | ay-**SINK**-ruh-nus（重音第二） |
| **deterministic** | dee-ter-**MIN**-is-tik |
| **ephemeral** | ih-**FEM**-er-ul（重音第二） |
| **cardinality** | kar-di-**NAL**-i-tee |
| **granularity** | gran-yuh-**LAIR**-i-tee |
| **observability** | ob-zerv-uh-**BIL**-i-tee |
| **linearizability** | lin-ee-uh-rize-uh-**BIL**-i-tee（慢慢读，没人会催你） |
| **orchestration** | or-kess-**TRAY**-shun |
| **provisioning** | pruh-**VIZH**-un-ing |
| **quantization** | kwon-ti-**ZAY**-shun |
| **hallucination** | huh-loo-suh-**NAY**-shun |
| **exfiltration** | eks-fil-**TRAY**-shun |
| **hierarchy** | **HIGH**-uh-rar-kee |
| **taxonomy** | tak-**SON**-uh-mee |
| **heuristic** | hyoo-**RISS**-tik（h 要发音） |
| **vulnerability** | vul-ner-uh-**BIL**-i-tee |
| **egress / ingress** | **EE**-gress / **IN**-gress |
| **tuple** | **TOO**-pul 或 **TUP**-ul（两种都能听懂，别纠结） |

### D. 两条比发音更重要的口语规则

1. **读错单词不致命，读错重音才致命。** 母语者能从上下文补全一个音发错的词，但重音落错位置会让整个词的节奏变形，对方需要停下来解码 —— 那 0.5 秒的迟疑会打断你的论证流。**所以只背 A 组那十个就够了。**

2. **不确定发音时，换一个你有把握的词。** 说不准 linearizability，就说 "the strongest consistency model, where every operation appears to happen at a single instant"。**描述永远比术语安全** —— 而且往往更能证明你真的理解它。反过来，为了显得专业硬塞一个读不准的词，是纯粹的负收益。

---

**下一篇** → [05-english-phrasebook.md](05-english-phrasebook.md)：词有了，接下来是把词组装成有分量的整句。
**配合使用** → [03-cheatsheet.md](03-cheatsheet.md) 背数字与公式｜[02-question-bank.md](02-question-bank.md) 54 题自测｜[01-interview-framework.md](01-interview-framework.md) 控 45 分钟节奏
