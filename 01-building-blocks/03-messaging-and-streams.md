# 03 · 消息与流：解耦的代价

> 引入队列不是"加个组件"，是把你的系统从"同步的、可调试的"变成"异步的、最终一致的（eventually consistent）、需要幂等（idempotency）和死信处理的"。这个代价必须自觉支付。

---

## 读这一章之前

**你在工作中遇到过这个**

你把"发确认邮件"从下单主流程里挪了出去，扔进 Kafka，下单 p99 从 650 ms 掉到 60 ms，皆大欢喜。
三周后客服转来一张工单：某个客户收到了 7 封一模一样的确认邮件。你还没查完，又出事了 ——
上游改了字段名，一条 JSON 让消费者反序列化时 panic、重启、重新读到同一条、再 panic，
那个分区后面的 41 万条消息卡了 6 个小时。这期间监控面板全绿，因为消费者进程一直"活着"。

**需要先懂的概念**

| 概念 | 一句话 | 详见 |
|---|---|---|
| 同步 / 异步 | 调用方是原地等结果，还是发出去就走、结果稍后另行送达 | [00-concepts §8](../00-foundations/00-concepts.md#8-同步--异步--阻塞--非阻塞) |
| 关键路径 | 用户必须等它做完的那条链；"移出关键路径"就是这一章的全部动机 | [00-concepts §1](../00-foundations/00-concepts.md#1-一个请求到底经历了什么) |
| 最终一致 | 用户看到"成功"时事情可能还没做完，只承诺最终收敛，不承诺多久 | [00-concepts §6](../00-foundations/00-concepts.md#6-什么是一致性--一个词两种完全不同的意思) |
| partition 的两个意思 | 数据切分出来的"块"，和网络故障的"分区"，英文是同一个词 | [00-concepts §5](../00-foundations/00-concepts.md#5-副本分片分区--三个被混用的词) |
| 幂等 / 幂等键 | 同一操作做多次效果等于做一次；靠请求自带的去重标识实现 | [01-fundamentals §5](../00-foundations/01-fundamentals.md#5-幂等idempotency分布式系统的第一公民) |
| 背压 | 处理不过来时向上游反向施压（拒绝/阻塞），而不是无限排队 | [01-fundamentals §6](../00-foundations/01-fundamentals.md#6-背压backpressure没有它系统就会雪崩cascading-failure) |

**这一章要回答的问题**

1. 队列（SQS）和日志（Kafka）到底是不是同一类东西？选错了会以什么形态炸出来？
2. 号称 exactly-once 的系统，实际做到了什么、做不到什么？我该怎么表述才不算吹牛？
3. "写数据库"和"发消息"凑不成一个原子操作，那下游怎么才能既不漏也不多算？
4. 一条处理不了的消息，凭什么能把一整个分区卡死六个小时？

**本章新引入的术语**

| 术语 | English | 一句话定义 |
|---|---|---|
| 提交日志 | commit log | 只追加、消息被读走之后也不删除的存储；每个读者自己记住"我读到第几条" |
| 位点 | offset | 一条消息在分区里的序号；提交位点 = 声明"这个序号及之前的我都处理完了" |
| 消费组 | consumer group | 一组共同分担同一批分区的消费者；不同组各读各的位点，互不影响 |
| 队头阻塞 | head-of-line blocking | 排在最前面的那一条卡住，后面本来能立刻处理的全部跟着等 |
| effectively-once | effectively-once | 投递允许重复，但处理方对重复无感，所以最终效果等于只执行了一次 |
| 双写问题 | dual-write problem | 一次业务动作要改两个独立系统（数据库 + 消息系统），而这两个写无法一起成功、一起失败 |
| 事务发件箱 | transactional outbox | 把"待发送的消息"当成一行数据，和业务数据写进同一个本地事务，再由独立进程读出来投递 |
| 变更数据捕获 | CDC / change data capture | 读数据库自己的重做日志（WAL / binlog），把每一行的变更变成一条事件 |
| 死信队列 | DLQ / dead letter queue | 专门存放"重试多次仍失败"的消息的独立队列，等人排查或修好后重放 |
| 可见性超时 | visibility timeout | 消息被取走后对其他消费者暂时隐藏的时长；到期还没确认就重新投递给别人 |
| 水位线 | watermark | 流处理里的一个时间标记，含义是"我认为不会再收到比它更早发生的事件了" |
| 消费积压 | consumer lag | 分区里最新一条消息的序号，减去消费者已经处理到的序号 |

---

## 1. 队列 vs 日志：两种根本不同的东西

| | **消息队列**（SQS, RabbitMQ） | **提交日志**（Kafka, Pulsar, Redpanda） |
|---|---|---|
| 模型 | 消费成功后通常从可见集合移除；保留与删除由产品配置 | 按保留策略保存追加记录，消费者维护 offset |
| 顺序 | 通常无保证（FIFO 队列除外） | 分区内严格有序 |
| 重放（replay） | 通常不提供任意历史回放；部分产品有归档/重驱 | 可回到仍在保留期内的 offset；已过期或压实掉的记录不能保证存在 |
| 多消费者 | 竞争消费（competing consumers，每条消息给一个） | 每个消费组（consumer group）独立读全量 |
| 扩展消费者 | 受队列吞吐、in-flight 配额和下游容量限制 | 同一消费组的并行度受分区数限制 |
| 单条 ACK | 支持（可乱序 ack） | 不支持（位点本身是一条水位线：提交它 = 声明"它之前的全部处理完了"，没法只确认中间某一条。⚠️ 这个"水位线"和 §6 流处理的 watermark 不是一回事） |
| 延迟消息（delayed message） | 原生支持 | 需要额外设计 |
| 吞吐 | 取决于消息大小、确认、批量、分区与产品配额 | 同样必须按目标配置压测；日志通常擅长批量顺序 I/O |

**选择判据：**
- 需要**重放历史 / 多个独立消费者 / 严格顺序 / 极高吞吐** → 日志（Kafka）
- 需要**任务分发 / 每条独立重试 / 延迟队列 / 简单运维** → 队列（SQS）

**需要谨慎**：用 Kafka 做长短差异很大的独立任务队列。
> 一个慢任务可能阻塞分区位点（head-of-line blocking），而 Kafka 原生确认以分区 offset 为中心，不提供传统队列那样的单条 visibility lease。若任务短、可批处理且分区内顺序重要，Kafka 仍可胜任；否则独立 ack/nack 的任务队列通常更贴合。

**反方向的边界**：SQS 配合 SNS/EventBridge 等扇出可以给多个下游各建队列，适合实时事件分发；但它不是可任意回溯的共享提交日志。若需求包含长时间历史回放、新消费者从旧数据重建或统一分区顺序，日志模型通常更合适。

---

## 2. Kafka 内核：你必须理解的几件事

```
Topic: orders
├─ Partition 0: [msg][msg][msg][msg]...  ← 追加写，严格有序
├─ Partition 1: [msg][msg][msg]...          Leader 在 broker-1
└─ Partition 2: [msg][msg][msg][msg][msg]   Follower 在 broker-2,3

消费组 A: consumer-1 → P0,P1    consumer-2 → P2
消费组 B: consumer-x → P0,P1,P2   （独立的 offset）
```

### 关键设计点

**a) 分区数 = 并行度（parallelism）上限**
```
消费者数 > 分区数  →  多余的消费者空闲（浪费）
分区数太多         →  每个分区一组文件句柄 + 元数据，broker 压力大，
                      rebalance 变慢，端到端延迟上升
```
起算公式是 `max(目标吞吐/压测得到的单分区吞吐, 所需并行消费者数)`，再为热点、故障和增长留余量。不存在通用的 12–100；分区上限要结合 broker、控制器、客户端与运维能力验证。
**分区数只能增不能减**，且增加会破坏 key → 分区的映射（已有数据不迁移）→ **顺序保证（ordering guarantee）被打破**。所以初始规划要留余量。

**b) 顺序保证的边界**
> Kafka 只保证**同一分区内**有序。同一个 key 通过 `hash(key) % partitions` 落到同一分区 → **同一实体的事件有序**。
> 跨分区无序。所以"全局有序"（total ordering）要么单分区（吞吐 = 单分区上限），要么在消费端重排（reorder）（需要版本号/时间戳）。

**c) 复制与持久性（durability）**
```
acks=0     不等确认         → 可能丢
acks=1     等 leader 接受并追加到本地日志；不等 follower，且不等于每条都 fsync
acks=all   等当前 ISR 达到确认条件；配合 min.insync.replicas 限制最少同步副本数

ISR = in-sync replicas：与 leader 保持同步、没落后太多的那批副本。
      落后太久的副本会被踢出 ISR —— 所以"所有 ISR 确认"在最坏情况下
      可能只剩 leader 自己，这正是必须同时设 min.insync.replicas 的原因。
```
一种常见耐单 broker 故障配置是 `replication.factor=3, min.insync.replicas=2, acks=all`，但还要同时配置 producer idempotence、unclean leader election、持久化与跨故障域布局，并做故障演练。它主要在写入可用性与已确认记录的耐久风险之间取舍，不能仅凭这三个参数宣称 CAP 的线性一致性。

**d) 消费者 rebalance 会造成延迟抖动**
消费者加入/退出/超时会触发 rebalance。经典 eager 协议可能让整个消费组暂时停顿；cooperative/incremental 协议只撤销需要移动的分区，影响范围更小，但仍要处理重复、暂停和状态迁移。
- 用 **Cooperative Sticky** 分配策略（增量 rebalance，只动需要动的分区）
- `max.poll.interval.ms` 要大于最慢的一次处理时间，否则消费者被误判死亡 → 无限 rebalance 循环
- 静态成员（`group.instance.id`）避免滚动重启（rolling restart）触发 rebalance

---

## 3. "Exactly-Once" 的真相

“Exactly-once”必须先写清**边界与可观察效果**。某个事务系统内部可以把输入位点、状态和输出绑定成一次原子提交；一旦副作用跨到不参与同一事务的数据库、HTTP API 或现实世界，就不能把消息系统内部保证直接外推为端到端只执行一次。

常见工程目标是：at-least-once 传递 + 在明确作用域与保留窗口内去重/幂等 + 原子提交业务效果，得到 **effectively-once** 的业务结果。窗口外重复、乱序和外部未知结果仍要单独设计。

### 三种做法

**a) Kafka 的事务（EOS = exactly-once semantics：把"写输出 topic"和"提交输入位点"绑成一个原子操作）**
```java
producer.initTransactions();
producer.beginTransaction();
producer.send(outputRecord);              // 写输出 topic
producer.sendOffsetsToTransaction(offsets, groupMetadata);  // 提交 offset
producer.commitTransaction();             // 原子：要么都成功要么都失败
```
✅ 有效范围：**Kafka → Kafka**（read-process-write）。
❌ **无效范围**：Kafka → 外部数据库/HTTP API。因为 Kafka 事务管不到外部系统。

**b) 消费端幂等（最通用，生产主力）**
```sql
-- 处理消息时，用消息的唯一 ID 做去重
INSERT INTO processed_messages (message_id, partition, offset, processed_at)
VALUES ($1, $2, $3, now())
ON CONFLICT (message_id) DO NOTHING;
-- 如果没插入成功 → 已处理过 → 跳过业务逻辑
-- 业务写入必须和这条 INSERT 在同一个事务里
```

**c) 幂等的业务操作**
`SET status='shipped'` 重复执行的最终值相同，但乱序消息仍可能把 `delivered` 回退成 `shipped`；应配合法状态转换或单调版本条件。`balance += 100` 不幂等，可改成 `记账条目 + 业务操作唯一约束`。

**面试标准答案**：
> "我会明确边界：传递是 at-least-once；在去重记录的保留窗口内，消费端用稳定的业务操作 ID 去重，并把去重记录与业务写入放在同一数据库事务里，所以这部分数据库效果 effectively-once。外部 API、副作用和窗口外重放另有幂等键与对账策略。"

---

## 4. 双写问题（dual-write problem）与 Outbox 模式

### 问题

```python
def create_order(order):
    db.insert(order)              # ① 成功
    kafka.send("order_created")   # ② 失败/超时 → 数据库有订单，下游永远不知道
```
这两个操作**无法原子完成**（不同的系统，2PC 不实用 —— 2PC 即两阶段提交：协调者先让所有参与者"预备"，都答应了再统一发"提交"；协调者卡在两阶段之间时，参与者只能一直锁着资源干等，见 [`05 §9`](05-consensus-and-coordination.md#9-现代系统里的协调尽量避免它)）。

调换顺序也不行：先发消息后写库，可能消息发出去了但库写失败 → 下游看到不存在的订单。

### Transactional Outbox —— 标准解法

```sql
BEGIN;
  INSERT INTO orders (id, ...) VALUES (...);
  INSERT INTO outbox (id, aggregate_id, event_type, payload, created_at)
  VALUES (gen_ulid(), $order_id, 'OrderCreated', $json, now());
COMMIT;   -- 原子：要么都写，要么都不写
```

然后由一个独立的 **Relay** 把 outbox 表的记录发到消息系统：

```
┌──────────┐    同一事务    ┌──────────┐
│  orders  │◄─────────────►│  outbox  │
└──────────┘                └────┬─────┘
                                 │ ① 轮询（SELECT ... FOR UPDATE SKIP LOCKED
                                 │    ——锁住选中行、跳过别人已锁住的行，§9 展开）
                                 │ ② 或 CDC（读 WAL/binlog）← 推荐
                                 ▼
                            ┌─────────┐
                            │  Kafka  │
                            └─────────┘
```

（WAL / binlog：数据库在真正修改数据页之前、先按顺序写下的那份"我接下来要改什么"的日志。它本来是给崩溃恢复用的，CDC 只是搭了个便车。）

**两种 Relay 实现：**

| | 轮询（polling） outbox 表 | CDC（Debezium 等） |
|---|---|---|
| 延迟 | 100ms–1s | 10–100ms |
| DB 负载 | 持续查询 + 删除 | 只读 WAL，几乎无影响 |
| 运维 | 简单，自己写 | 需要运维 Debezium/Kafka Connect |
| 表膨胀（table bloat） | 需要定期清理 outbox | 同样需要 |

表中的延迟和负载是常见形态而非保证；轮询间隔、批量、索引、WAL 量与连接器配置会改变结果。

**关键点**：Relay 是 at-least-once 的（发了但没标记已发 → 重发）。所以**下游必须幂等**。轮询 Relay 也不能拿着数据库事务/行锁跨越网络发送：先在短事务中原子 claim 一批记录，写入 `lease_owner`、数据库时间计算的 `lease_until` 和递增 `lease_epoch` 后提交；事务外发送；最后仅当 owner 与 epoch 仍匹配时标记完成。租约过期可接管，旧 worker 的迟到结果会被 fencing 条件拒绝。若消息系统按稳定 `outbox.id` 去重，也要把这个 ID 跨重试复用。

**上面的 ASCII 图画的是拓扑；下面这张画的是时间。重点看第 8 步之后：Kafka 已经 ack 了，但 Relay 在持久化"已发送"位点前挂掉——这一瞬间的裂缝，就是所有重复消息的来源。**（图中的 LSN = log sequence number，WAL 里的位置编号；Relay 靠持久化它来记住"我读到哪了"。）

```mermaid
sequenceDiagram
    autonumber
    participant S as Service
    participant DB as OrderDB_Outbox
    participant R as CDC_Relay
    participant K as Kafka
    participant C as Consumer
    S->>DB: BEGIN insert order row
    S->>DB: insert outbox row msg_id=M1
    S->>DB: COMMIT
    Note over S,DB: atomic. one local transaction. no 2PC
    DB-->>S: 200 order created
    R->>DB: tail WAL from stored LSN
    DB-->>R: OrderCreated msg_id=M1
    R->>K: produce M1
    K-->>R: ack persisted to partition
    R--xDB: commit new LSN FAILS then crash
    Note over R,DB: send succeeded but progress did not
    K->>C: deliver M1
    C->>C: insert M1 into processed_ids
    C->>C: apply business effect once
    Note over R,DB: Relay restarts from the OLD stored LSN
    R->>DB: tail WAL from stored LSN
    DB-->>R: OrderCreated msg_id=M1 again
    R->>K: produce M1 again
    K->>C: deliver M1 again
    alt M1 already in processed_ids
        C->>C: skip. no business effect
    else first time seen
        C->>C: apply and record M1
    end
    Note over K,C: duplicates are designed in. idempotency is the contract
```

> 📖 **读图要点**：盯住第 9 步那个 `--x` 和它后面的第 10–12 步——位点没存住这件事，**完全不妨碍**消息被投递、被消费、在消费者侧产生真实的业务效果。第 8 步的 ack 和第 9 步的位点持久化之间不存在任何原子性，所以"发一次且只发一次"在这条链路上物理不可能；能做的只有把去重责任推到最右边那个 `processed_ids` 上。

### CDC（Change Data Capture）的两种用法

**1. Outbox 的 Relay**（推荐）：只捕获 outbox 表，事件是**有意设计的领域事件（domain event）**。

**2. 直接捕获业务表**（谨慎）：把数据库表变更直接变成事件流。
> ❌ 问题：这把**内部数据库 schema 变成了公开契约（public contract）**。改个列名就破坏所有下游。
> ✅ 适用：数据同步到分析库/搜索引擎（下游是你自己的，且是投影（projection：把源表的行原样换个存储再放一份，不承载额外的业务语义）而非业务事件）。

**Senior 的表述**：
> "我用 Outbox + CDC。CDC 只读 outbox 表，不直接读业务表 —— 因为我不想让数据库 schema 成为跨团队的 API 契约。outbox 里的事件是我明确设计的、带版本的领域事件。"

---

## 5. 消息投递的失败处理

### 重试与死信队列（DLQ）

```
消息 → 消费 → 失败 → 重试(退避) → 失败 N 次 → DLQ
                                              ↓
                                        告警 + 人工/自动处理
```

**分类处理错误**：

| 错误类型 | 处理 |
|---|---|
| **瞬时错误**（网络、下游 503、锁冲突） | 重试，指数退避（exponential backoff） + 抖动（jitter） |
| **不可重试的技术错误**（无法解析、schema 不兼容） | 隔离到 DLQ / quarantine，修复后再决定重放 |
| **预期业务拒绝**（余额不足、订单已取消） | 记录为明确业务结果或 rejected 状态，通常不应伪装成 DLQ 故障 |
| **毒丸消息**（poison pill，导致消费者崩溃） | 必须有重试上限，否则消费者无限重启 |

⚠️ **毒丸是队列系统最常见的生产事故**：一条格式异常的消息让消费者 panic → 重启 → 再消费同一条 → 再 panic → **整个分区永久卡住**。
消费循环要把可重试、不可重试和进程级故障分开，记录尝试次数，并把超限消息隔离；不要用一个宽泛的 `catch` 把 OOM/进程损坏后继续运行。

**DLQ 不是垃圾桶**：
- 必须有按 SLO 设置的**告警**（有些业务允许少量待人工处理，不能把任何一条都固定当事故）
- 必须有**重放工具**（修好 bug 后把 DLQ 消息放回主队列）
- DLQ 消息要保留原始上下文（原 topic、offset、失败原因、重试次数、trace_id）

**把上面那条线性的"重试 → DLQ"展开成状态机，你会发现它其实有环、而且有一条谁都不画的边：**

```mermaid
stateDiagram-v2
    [*] --> Queued
    Queued --> InFlight: consumer polls and lease starts
    InFlight --> Acked: handler succeeds
    Acked --> [*]
    InFlight --> RetryBackoff: transient error such as 503 or lock conflict
    RetryBackoff --> InFlight: backoff expires and attempts plus 1
    InFlight --> DeadLetter: permanent error such as schema or business reject
    RetryBackoff --> DeadLetter: attempts exceed max
    DeadLetter --> Queued: replayed after the bug is fixed
    DeadLetter --> [*]: discarded after human review
    InFlight --> Queued: visibility timeout no heartbeat renewal DUPLICATE DELIVERY
```

> 📖 **读图要点**：两条边最值得盯。一是 `DeadLetter --> Queued`——DLQ 是个**可回到起点的中转站**，没有这条边的系统等于把消息扔进黑洞。二是 `InFlight --> Queued` 那条可见性超时边：消费者可能**正在成功处理**，只是慢过了**租约**（lease：有时间限制、可续期的独占权 —— 这里就是"这条消息在接下来 N 秒归你处理"，到期不续约就自动失效、消息重新对别人可见，见 [`05 §4`](05-consensus-and-coordination.md#4-租约lease比锁更好的抽象)），消息已被重新投递；这条边和"永久错误"完全无关，却是线上重复消费最常见的真实来源，也是为什么慢处理必须**续约心跳**而不是简单调大超时。

### 延迟队列 / 重试队列的实现

```
主队列 → 失败 → retry-5s 队列 → 5s 后回主队列 → 失败 → retry-30s → ... → DLQ
```
- SQS：原生 `DelaySeconds`（最长 15 分钟）
- RabbitMQ：TTL + Dead Letter Exchange
- Kafka：需要自建（多个 retry topic，或用 timer wheel 服务）
- Redis：ZSET，score = 到期时间戳，轮询 `ZRANGEBYSCORE`

---

## 6. 流处理：语义与状态

### 时间语义

```
事件时间（Event Time）：事件实际发生的时间     ← 按业务发生时间统计时使用
摄取时间（Ingestion Time）：进入系统的时间
处理时间（Processing Time）：被处理的时间      ← 适合只关心当前处理速率/系统行为的场景
```

当窗口语义描述“业务何时发生”且需要可重放结果时，应使用事件时间。只关心摄取或处理行为的监控、运维窗口可以合理使用 ingestion/processing time；先定义语义再选时钟。

### 水位线（Watermark）—— 处理乱序（out-of-order events）

```
数据流: [t=10][t=12][t=9][t=15][t=11][t=20]...   乱序到达

一种简单策略：Watermark(W) = 观察到的最大事件时间 − 最大乱序估计 Δ_ooo
含义："系统估计 t < W 的事件大体已到齐，可以先产出结果"；这是进度估计，不是不再迟到的保证
→ 当 W 越过窗口结束时间，触发窗口计算
```

这里的 `Δ_ooo` 是生成 watermark 的 **bounded out-of-orderness / watermark delay**；它与下面的 **allowed lateness** 是两个参数。后者从 watermark 首次越过窗口末端后开始算，决定窗口状态还保留多久、迟到事件能否修正已产出的结果。

**迟到数据（late-arriving data）的三种策略**：
1. **丢弃**（最简单，但会漏数据）
2. **侧输出流**（single output，业界通用写法是 **side output**）单独处理 —— 把迟到的数据分流到另一条流，不混进主结果
3. **允许延迟（allowed lateness）+ 更新结果**（要求下游能处理修正，即 upsert 语义：同一个主键再来一次就覆盖旧值，而不是插入第二行）

**取舍**：`Δ_ooo` 设大 → 首次结果更慢、被判为迟到的事件更少；设小 → 首次结果更快、后续修正更多。是否丢数据还取决于 allowed lateness、状态保留期以及迟到事件选择丢弃、侧输出还是 upsert，不能由 watermark 大小单独推出。这是流处理最核心的取舍，**没有正确答案，只有业务选择**。

### 窗口类型

| 类型 | 说明 | 用途 |
|---|---|---|
| 滚动（Tumbling） | 固定不重叠 `[0,5)[5,10)` | 每分钟统计 |
| 滑动（Sliding） | 固定重叠 `[0,5)[1,6)` | 移动平均；注意每条数据属于多个窗口，状态开销大 |
| 会话（Session） | 按不活跃间隔切分 | 用户会话分析、**Agent 对话轮次分组** |
| 全局 | 无窗口，需自定义触发 | |

### 状态管理

有状态流处理（去重（deduplication）、JOIN、聚合）需要状态后端（state backend）：
- **本地 RocksDB + Checkpoint 到 S3**（Flink 的做法。checkpoint：周期性把算子当前的全部状态打一个快照传到远端存储，进程崩溃后从最近一个快照 + 之后的消息重放恢复，而不是从头重算）
- 状态大小是主要成本。去重状态尤其危险：**"全局去重"意味着无限增长的状态** → 必须加时间窗口（"24 小时内去重"）。

---

## 7. 消息系统的设计规范

### Schema 与版本演进

跨团队或长期保留的事件必须有版本化 schema、兼容性检查和 owner；集中式 Schema Registry（Avro / Protobuf / JSON Schema）是常见实现，但小系统也可在代码仓库与 CI 中执行同样规则。

**兼容性规则：**
| 类型 | 允许的变更 | 谁先升级 |
|---|---|---|
| Backward | 删字段、加带默认值的字段 | 消费者先 |
| Forward | 加字段、删有默认值的字段 | 生产者先 |
| **Full** | 只加/删带默认值的字段 | 任意顺序 ✅ 推荐 |

**永远不要**：改字段类型、改字段含义、复用已删除的字段编号（Protobuf 用 `reserved`）。

### 事件设计：三种事件类型

```json
// 1. 事件通知（Event Notification）—— 只有 ID，消费者自己回查
{ "type": "OrderCreated", "orderId": "ord_123" }
// ✅ 载荷小、耦合低    ❌ 消费者要回查（N+1 压力、可能查到更新后的状态）

// 2. 事件携带状态转移（Event-Carried State Transfer）—— 携带完整快照
{ "type": "OrderCreated", "orderId": "ord_123", "items": [...], "total": 99.9, "customer": {...} }
// ✅ 消费者自治、无需回查    ❌ 载荷大、schema 耦合强

// 3. 领域事件（Domain Event）—— 携带发生了什么（增量）
{ "type": "OrderItemAdded", "orderId": "ord_123", "item": {...}, "version": 5 }
// ✅ 语义丰富、可做事件溯源    ❌ 消费者要自己重建状态
```

三种类型没有统一默认。事件通知载荷小但造成回查耦合；携带状态让消费者自治，却扩大 PII、schema、删除传播与载荷成本；领域事件表达业务变化但要求消费者重建。根据消费者需要、新鲜度、隐私与演进成本选择，必要时组合使用。

### 事件信封（Envelope）标准

每个事件都应该有：
```json
{
  "event_id": "01HQ...",           // ULID（时间戳前缀 + 随机后缀的 128 位 ID，本地生成即近似有序，见 05 §9），用于去重
  "event_type": "order.created",
  "event_version": "v2",
  "occurred_at": "2026-07-30T10:00:00Z",   // 事件时间
  "producer": "order-service@1.4.2",
  "trace_id": "...",               // 分布式追踪
  "tenant_id": "acme",             // 多租户路由/隔离
  "partition_key": "ord_123",      // 顺序保证的依据
  "data": { ... }
}
```
**`event_id` 和 `trace_id` 是事后排查唯一的抓手，不要省。**

---

## 8. 专项选读：队列在 AI/Agent 系统中的角色

通用 Full Stack 路径可以先跳到 [§9](#9-何时不要引入消息队列)。

Agent 系统天然是异步的、长时的，队列是核心构件：

```
用户请求 → API（立即返回 run_id）
              ↓
           [任务队列]  ← 优先级分层：交互式 / 批量 / 后台
              ↓
        Agent Worker（可能运行 30 秒 – 30 分钟）
              ├→ [工具调用队列]（并发受限）
              ├→ 中间状态 checkpoint 写入
              └→ [事件流] → SSE/WebSocket 推给用户（流式 token）
                          → 计费计量管道
                          → 轨迹存储（可观测）
```

（SSE / WebSocket：服务端持续往客户端推数据的两种长连接方式，选型见 [04 §4](04-networking-and-edge.md#4-流式传输streamingsse-vs-websocket)。checkpoint 见上面 §6。）

**Agent 场景的特殊要求：**

| 需求 | 做法 |
|---|---|
| **任务可能运行几十分钟** | 消息可见性超时（visibility timeout）要长，且要**心跳续约**（heartbeat renewal，否则消息被重新投递 → 任务重复执行） |
| **需要中途取消** | 把取消意图持久化并带版本/epoch；pub/sub 只做低延迟通知，worker 还要定期读取持久状态，避免离线时漏掉取消 |
| **成本高，重复执行很贵** | 强幂等 + checkpoint 恢复，而不是从头重跑 |
| **优先级** | 交互式任务不能被批量任务饿死（starvation） → 分队列 + 加权轮询（weighted round-robin），或独立 worker 池 |
| **流式输出** | 队列送任务，**结果走单独的流通道**（不要把 token 流塞进 Kafka） |
| **公平性（fairness）** | 单租户不能占满 worker → 按租户分片队列 或 并发配额（concurrency quota） |

**一个真实的坑**：
> Agent 任务运行 10 分钟，SQS 默认可见性超时 30 秒 → 消息在任务还在跑时被重新投递给另一个 worker → **同一个任务被执行了 20 次**，账单爆炸。
> 修复：`ChangeMessageVisibility` 心跳续约 + 幂等键（idempotency key，run_id）。

---

## 9. 何时不要引入消息队列

| 情况 | 说明 |
|---|---|
| 调用方需要立即知道结果 | 异步只会让你在客户端实现轮询，得不偿失 |
| 只有一个消费者且逻辑简单 | 直接同步调用，用重试 + 熔断（circuit breaker：某个依赖的失败率超过阈值就暂时不再调它、直接快速失败，冷却后放几个探测请求试探，见 [`05-reliability/03 §4`](../05-reliability/03-resilience-patterns.md#4-熔断器circuit-breaker阈值粒度半开half-open)） |
| 团队没有 DLQ / 幂等 / 监控的准备 | 队列会把 bug 变成"消息神秘消失" |
| 只是为了"削峰"（smoothing traffic spikes：把瞬时尖峰暂存进队列，让下游按自己的速度匀速消费，见 [`00-concepts §11`](../00-foundations/00-concepts.md#11-三个最常见的优化手段各在优化什么)） 但峰值只有 2× | 加机器更便宜也更简单 |

**替代方案：数据库即队列（database as a queue）**
```sql
-- 以下是 claim 阶段的骨架：必须放在一个短事务里并立即提交。
-- 生产实现应一次 claim 小批量，并使用数据库时钟计算 lease_until。
WITH candidate AS (
  SELECT id
  FROM jobs
  WHERE status IN ('pending', 'running')
    AND run_at <= now()
    AND (status = 'pending' OR lease_until < now())
  ORDER BY priority DESC, run_at
  FOR UPDATE SKIP LOCKED
  LIMIT 10
)
UPDATE jobs j
SET status = 'running',
    lease_owner = $worker_id,
    lease_until = now() + $lease_interval,
    lease_epoch = lease_epoch + 1
FROM candidate c
WHERE j.id = c.id
RETURNING j.*;
```

claim 提交后才在事务外执行工作；心跳续租和最终完成都要带 `WHERE lease_owner = ? AND lease_epoch = ?`，使过期旧 worker 无法覆盖接管者。外部副作用还需要稳定幂等键或下游 fencing；超时且结果未知时先对账。

✅ 无新组件、事务性入队（transactional enqueue，自动解决同库业务写与入队的双写问题）、易查询和调试。
❌ 需要自己实现 lease/fencing、重试、DLQ、清理、公平性与监控；吞吐上限取决于索引、争用、批量和数据库余量，必须压测。

当任务量适中、与同库事务强相关且团队愿意实现上述协议时，这是很好的起点；需要长期回放、多消费组或数据库已是瓶颈时，应直接评估专用消息系统。

---

## 这一章的三句话

1. **引入队列不是加了一个组件，是把一次函数调用换成了一份"允许重复、允许乱序、允许迟到几小时"的契约。** 所以消费端的幂等不是优化项，是入场券 —— 没有它，队列带给你的可用性收益会被数据正确性损失全部吃掉。
2. **说 exactly-once 时必须同时说清边界。** 单一事务域内可以原子提交位点、状态和输出；跨事务域通常用 at-least-once + 有作用域与保留窗口的幂等/去重得到 effectively-once，并为外部未知结果设计对账。
3. **队列最贵的代价不是多运维一个系统，是让故障从"报错"变成了"静默"。** 积压、毒丸卡死分区、DLQ 里躺着两万条没人认领 —— 这些都不会让任何一个用户请求返回 500。**在你有 DLQ 告警和消费积压告警之前，别上队列。**

---

## 面试官会追问

1. Kafka 保证顺序吗？在什么范围内？
2. 你说 exactly-once，具体怎么实现的？Kafka 事务能覆盖写数据库吗？
3. 写数据库 + 发消息怎么保证原子性？
4. 一条毒丸消息会怎么样？怎么防？
5. 消费者处理很慢，消息堆积（consumer lag / backlog）了 1 亿条，怎么办？（→ 加分区+消费者 / 跳过历史 / 并行处理 / 先分流再处理）
6. Agent 任务跑 20 分钟，队列的可见性超时怎么设？
7. 什么时候你会选择不用 Kafka？

---

**按训练路径阅读** → 回 [START-HERE](../START-HERE.md) 按所选路径继续；页尾链接只表示本目录或专章的顺读顺序。

**目录顺读下一篇** → [04-networking-and-edge.md](04-networking-and-edge.md)
