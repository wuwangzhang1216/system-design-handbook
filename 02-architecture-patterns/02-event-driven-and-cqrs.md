# 02 · 事件驱动、Saga 与 CQRS

> 事件驱动买来的是解耦，付出的是"没有任何一个地方能看到全局状态"。
> 事件溯源和 CQRS 是两个**独立**的决策，大多数团队正确的选择是两个都不选。

---

## 1. 事件的三种类型：延伸与选择判据

三种基本形态在 [`01-building-blocks/03-messaging-and-streams.md`](../01-building-blocks/03-messaging-and-streams.md) 里已经定义过（事件通知 / 携带状态转移 ECST / 领域事件）。这里补上真正决定选型的四个维度：

| 维度 | 事件通知（只有 ID） | ECST（携带完整快照） | 领域事件（增量） |
|---|---|---|---|
| 载荷 | 100–300 B | 1–50 KB（压缩后常见 2–8 KB） | 200 B–2 KB |
| 消费者是否要回查 | **必须**，产生 N+1 与时序错乱 | 否 | 否，但要自己重建状态 |
| 上游 schema 耦合 | 最低 | **最高**（下游依赖你的全部字段） | 中（依赖事件语义） |
| **PII 扩散** | 无 | **把 PII 复制到每个下游 + 每个 topic 的保留期** | 视字段而定 |
| 可否做事件溯源 | 否 | 否（快照不是事实） | **是**，这是唯一能重建历史的形态 |
| 顺序敏感度 | 低 | 低（后到的快照覆盖先到的即可，天然幂等） | **高**（漏一条就永久错） |

**判据（按优先级）：**

1. 消费者需要**历史**（"这个订单为什么变成现在这样"）→ 领域事件，且你已经在做事件溯源。
2. 消费者需要**当前状态**且不想回查 → ECST。**这是默认选择。**
3. 载荷超过消息系统上限（Kafka `message.max.bytes` 默认 1 MB）→ **Claim-Check 模式**：事件里放 S3/对象存储的指针 + 内容哈希，正文单独存。别去调大 `message.max.bytes`，那会拖垮整个 broker 的复制与页缓存。
4. 事件里有 PII 且系统要能删除 → **不要用 ECST**，或者用下面第 4 节的 crypto-shredding。

> **面试金句**
> "我默认用 event-carried state transfer，因为它消除了下游对上游的同步依赖 —— 这才是解耦的真正价值。但它有一个我必须提前处理的代价：**我把 PII 复制到了 N 个下游和 N 个 topic 的保留窗口里**。所以在有 GDPR 义务的域，我要么把 PII 字段换成引用 ID，要么在事件里对 PII 字段做 per-subject 加密。这个决定必须在第一个消费者接入之前做，之后就是全网回填。"

---

## 2. Saga：跨服务事务的现实解法

### 为什么不是 2PC

- **协调者故障时参与者持锁阻塞**：数据库行锁持有从毫秒变成分钟，连锁放大。
- **锁持有时长 = 整条链路时长**：3 个服务 × p99 200 ms → 锁 600 ms+，吞吐塌陷。
- **跨组织 / 跨异构存储不可用**：你无法让 Stripe 参与你的 2PC；Postgres + Kafka + S3 也没有共同的 XA。

**Saga 的定义**（[Garcia-Molina & Salem, 1987](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf)）：把一个长事务拆成 N 个本地事务 T1…Tn，每个 Ti 配一个补偿 Ci，失败时按 **逆序** 执行 Ci…C1。

**Saga 提供的是 ACD，不是 ACID —— 缺的是 I（隔离）。** 这不是实现瑕疵，是定义本身。中间状态对外可见。

### 编排 vs 编舞

| | 编排（Orchestration） | 编舞（Choreography） |
|---|---|---|
| 流程定义在哪 | 一个显式状态机，可读、可画、可测 | 散在 N 个服务的事件订阅里 |
| 加一个新步骤 | 改编排器一处 | 改 2 个服务 + 新事件契约 |
| 调试"卡在哪一步" | 查一张表 | 拼 N 个服务的日志 |
| 循环依赖 | 不可能 | **可能且难发现**（A→B→C→A 的事件环） |
| 耦合点 | 编排器知道所有参与者 | 每个服务只知道自己的上下游 |
| 可维护跳数 | 10+ | **> 3 跳后不可维护** |

**判据**：有补偿逻辑、有明确终态、需要 SLA 的流程 → 编排。纯通知（发邮件、写审计、更新搜索索引）→ 编舞。

⚠️ 编排器**不是**单点：它是无状态的执行器 + 一张状态表。真正的单点是那张表，和你的主库同级。

### 完整订单 Saga 状态机（含补偿路径）

```
  正向路径 (forward)                                 补偿路径 (compensation)
 ═══════════════════                               ══════════════════════════

      [CREATED]
          │  T1  ReserveInventory        ← 可补偿 (compensatable)
          ├──────────────失败──────────────────────────────────► [FAILED_CLEAN]
          ▼                                                       (无副作用，直接失败)
   [INV_RESERVED]
          │  T2  AuthorizePayment        ← 可补偿（只授权，不扣款）
          ├──────────────失败───────────► [COMPENSATING] ─ C1 ReleaseInventory ─┐
          ▼                                     ▲                               │
  [PAY_AUTHORIZED]                              │                               ▼
          │                                     │                        [COMPENSATED]
   ═══════╪═══════ PIVOT 线：越过此线不再回滚 ═══╪══════════════════════════════════
          │                                     │
          │  T3  CapturePayment          ← PIVOT：不可补偿*（退款 ≠ 撤销）
          ├──────────────失败───────────────────┘   （失败时仍可回滚，因为钱没动）
          ▼
       [PAID]
          │  T4  CreateShipment          ← retriable：只能重试到成功，禁止回滚
          │      失败 → 指数退避重试 → 超 N 次 → [NEEDS_HUMAN] + 告警
          ▼
     [SHIPPED]
          │  T5  SendConfirmationEmail   ← retriable 且不可撤销（邮件发出去了）
          ▼
    [COMPLETED]
```

**三段式结构是 Saga 设计的核心，不是装饰：**

| 段 | 步骤性质 | 规则 |
|---|---|---|
| **Compensatable** | 有对应的语义补偿 | 排在最前。失败 → 逆序补偿 |
| **Pivot** | 提交后 saga 必然向前完成 | **有且只有一个**。它是"不可回头点" |
| **Retriable** | 无补偿，但保证最终成功 | 排在 pivot 之后，只允许重试 |

**设计规则：把不可补偿的操作尽量往后排，并让 pivot 尽可能靠后。** 如果你的流程里有两个互相独立的不可补偿操作（扣款 + 调用一次 $2 的 LLM 批处理），你就有两个 pivot —— 这时必须引入**预留**（下面）把其中一个变成可补偿的。

### 补偿事务的正确写法

补偿**不是回滚**，是一笔新的正向业务操作：

```
❌ UPDATE inventory SET qty = qty + 10          -- 盲目加回，并发下会超卖
✅ UPDATE inventory_reservation
      SET state = 'RELEASED', released_at = now()
    WHERE reservation_id = $1 AND state = 'HELD'   -- 条件更新 = 幂等
```

**四条硬规则：**

1. **补偿必须幂等**。补偿消息一定会重复投递（at-least-once），用 `WHERE state = 'HELD'` 这种条件更新，而不是相对增量。
2. **补偿必须能重试到成功，不允许"补偿失败"这个终态**。补偿失败只能进 `NEEDS_HUMAN` 队列 + 告警，绝不能静默丢弃 —— 那是钱漏出去的地方。
3. **补偿要能处理"还没执行就来补偿"**（正向请求超时但实际成功/失败未知）。用同一个幂等键写一条 tombstone，让后到的正向请求被拒绝。
4. **补偿要留痕**：补偿本身是一个领域事件（`InventoryReleased`），不是一次 UPDATE。

### 不可补偿操作怎么办：语义锁与预留（TCC）

真正不可补偿的东西只有三类：**已发出的通知、已交付的物理动作、已花掉的外部成本**（第三方 API 计费、LLM token）。

**TCC（Try-Confirm-Cancel）** 把它们变成可补偿的：`Try` 预留资源且不产生外部效果（带 TTL），`Confirm` 兑现（此时才不可逆），`Cancel` 释放预留（幂等）。

| 场景 | Try | Confirm | Cancel | 关键参数 |
|---|---|---|---|---|
| 库存 | 写 `reservation(HELD, expires_at)` | `HELD → CONSUMED` | `HELD → RELEASED` | TTL = saga p99 × 3，典型 5–15 分钟 |
| 支付 | Authorization（信用卡授权） | Capture | Void | 授权有效期典型 7 天，逾期自动失效 |
| 邮件 | 写入 `outbox_email(scheduled_at = now+120s)` | 到点发送 | 删除记录 | **120 秒延迟就能把"发邮件"变成可补偿** |
| LLM 批任务 | 只做 `count_tokens` 预估 + 扣预算配额 | 提交 batch job | 退还配额 | Batch 半价，但 24h 周转，几乎必须预留 |

⚠️ **预留必须有超时清理器**，否则预留泄漏会把库存"锁死"。清理器本身是一个扫表任务：`WHERE state='HELD' AND expires_at < now()`，每 30 秒跑一次。

### 隔离性缺失的三种异常与对策

| 异常 | 场景 | 对策 |
|---|---|---|
| **脏读** | Saga 中途，另一个事务读到 `PAY_AUTHORIZED` 的订单并当作已完成 | **语义锁**：在记录上打 `saga_pending = true` 标志，读方必须显式处理"处理中"状态 |
| **丢失更新** | Saga 补偿时覆盖了期间的其他写入 | **交换更新**（只做可交换的增量）或 **重读值**（补偿前校验值未变，变了就升级为人工） |
| **模糊读** | 同一 saga 内两次读同一数据得到不同结果 | **悲观视图**：重排步骤，把读放在写之前；或把关键数据快照进 saga payload |

---

## 3. Outbox + Saga 编排器：可抄的实现

Outbox 本身在 [`01-building-blocks/03`](../01-building-blocks/03-messaging-and-streams.md) 讲过。这里是它和 Saga 状态机拼在一起的样子 —— **注意：saga 状态推进和事件发出必须在同一个本地事务里**，否则你只是把双写问题从业务层挪到了编排层。

```sql
-- 编排器的唯一真相源
CREATE TABLE saga_instance (
  saga_id       uuid PRIMARY KEY,
  saga_type     text        NOT NULL,       -- 'order_fulfillment@v3'
  business_key  text        NOT NULL,       -- 订单号 → 业务级幂等
  state         text        NOT NULL,       -- RUNNING | COMPENSATING | DONE | NEEDS_HUMAN
  step_cursor   int         NOT NULL DEFAULT 0,
  ctx           jsonb       NOT NULL,       -- 各步骤产出的上下文
  attempt       int         NOT NULL DEFAULT 0,
  next_run_at   timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now(),
  UNIQUE (saga_type, business_key)          -- 同一订单只会有一个 saga
);

CREATE TABLE saga_step (                    -- step_id 如 'T2.AuthorizePayment'
  saga_id uuid, step_id text, status text,  -- SUCCEEDED | FAILED | COMPENSATED
  idem_key text NOT NULL, result jsonb,     -- idem_key = 下发给下游的幂等键
  PRIMARY KEY (saga_id, step_id));

CREATE INDEX ON saga_instance (next_run_at) -- 调度索引：只扫待执行的
  WHERE state IN ('RUNNING','COMPENSATING');
```

```python
IDEM = lambda saga_id, step_id: sha256(f"{saga_id}:{step_id}").hexdigest()
#      ↑ 幂等键必须由 (saga_id, step_id) 派生，不能用 uuid4()。
#        "客户端每次重试生成新 UUID" 是幂等失效的头号原因。

def tick(saga_id):
    # 1) 取行级锁，防止多个 worker 同时推进同一个 saga
    with db.tx() as tx:
        s = tx.query("SELECT * FROM saga_instance WHERE saga_id=%s FOR UPDATE SKIP LOCKED", saga_id)
        if not s or s.state not in ("RUNNING", "COMPENSATING"):
            return
        step = PLAN[s.saga_type][s.step_cursor]

    # 2) 副作用在事务之外执行（绝不在持有 DB 事务时做网络调用）
    direction = "forward" if s.state == "RUNNING" else "compensate"
    try:
        result = step.invoke(direction, s.ctx, idem_key=IDEM(saga_id, step.id))
        ok = True
    except TransientError:
        ok, result = None, None            # 稍后重试
    except BusinessError as e:
        ok, result = False, e.payload      # 业务性失败 → 触发补偿

    # 3) 状态推进 + 事件发出：同一事务，原子
    with db.tx() as tx:
        if ok is None:                                        # 瞬时失败：退避
            backoff = min(2 ** s.attempt, 300) * jitter()     # 上限 300s
            tx.exec("UPDATE saga_instance SET attempt=attempt+1, "
                    "next_run_at=now()+%s*interval '1s', updated_at=now() "
                    "WHERE saga_id=%s", backoff, saga_id)
            if s.attempt + 1 >= step.max_attempts:
                escalate(tx, saga_id, step)                   # RUNNING→COMPENSATING
                                                              # 或 →NEEDS_HUMAN（retriable 段）
            return

        tx.exec("INSERT INTO saga_step(saga_id,step_id,status,idem_key,result) "
                "VALUES (%s,%s,%s,%s,%s) ON CONFLICT DO NOTHING",
                saga_id, step.id, "SUCCEEDED" if ok else "FAILED",
                IDEM(saga_id, step.id), result)

        forward = (s.state == "RUNNING" and ok)
        nxt, st = (s.step_cursor + 1, "RUNNING") if forward \
             else (s.step_cursor - 1, "COMPENSATING")         # 失败或补偿中 → 逆序
        if nxt < 0: st = "COMPENSATED"
        if nxt >= len(PLAN[s.saga_type]): st = "DONE"

        tx.exec("UPDATE saga_instance SET step_cursor=%s, state=%s, attempt=0, "
                "ctx=ctx||%s, next_run_at=now(), updated_at=now() WHERE saga_id=%s",
                nxt, st, json(result or {}), saga_id)

        # ★ 关键：领域事件写进 outbox，与状态推进同一事务
        tx.exec("INSERT INTO outbox(id,aggregate_id,event_type,payload) VALUES (%s,%s,%s,%s)",
                ulid(), s.business_key, f"saga.{step.id}.{st}", json(result or {}))
```

**卡住的 Saga 检测（必须有，不是可选）：**

```sql
SELECT saga_id, saga_type, state, step_cursor, now() - updated_at AS stuck_for
FROM saga_instance
WHERE state IN ('RUNNING','COMPENSATING')
  AND updated_at < now() - interval '15 minutes'
ORDER BY updated_at;
-- 告警阈值：任何 saga 卡超过 p99 时长的 10 倍
```

**撞墙条件**：单表轮询编排器在 **~2,000 saga/s** 或 saga_instance 表超过千万活跃行时开始退化（`next_run_at` 索引膨胀、`FOR UPDATE SKIP LOCKED` 竞争）。信号是 tick 延迟 p99 上升而 CPU 不高。到这里换分区表 + 按 `hash(saga_id)` 分片 worker，或换成 durable execution 引擎（[Temporal](https://temporal.io/)、[Restate](https://restate.dev/)、DBOS）。

---

## 4. 事件溯源的真实成本

**事件溯源（Event Sourcing）= 只存事件，当前状态是事件的折叠结果。** 它不是"加个审计表"，是把真相源换掉。

### 成本清单（这是决策的全部依据）

| 成本项 | 量级 | 说明 |
|---|---|---|
| **快照** | 每 100–500 个事件一次 | 目标：单聚合重建 < 50 ms。没有快照的 ES 系统在聚合活跃度上升后会突然变慢 |
| **重放时长** | 单线程投影 1–5 万 events/s | 5 亿事件 ÷ 3 万 eps ≈ **4.6 小时**；按聚合 ID 哈希并行 16 路 ≈ **20 分钟** |
| **存储** | 事件量是状态量的 10–100× | 且**永不删除**。1 KB/事件 × 5 亿 = 500 GB，还要算索引与副本 |
| **Schema 演进** | 需要 upcaster 链 | 老事件永远在，代码里永远要能解析 v1。写 v5 时你还在维护 v1→v2→…→v5 的转换 |
| **删除（GDPR）** | 与"事件不可变"直接冲突 | 见下 |
| **人员** | 团队学习曲线 3–6 个月 | 且新人 onboard 成本长期高于 CRUD |

**重放时长必须被当作 SLO 管理**：如果全量重放要 6 小时，那你的"投影 bug 修复"最快也要 6 小时才能上线。这个数字决定了你能不能在事故中重建读模型。

**必须提前设计的两件事：**
1. **按聚合并行重放**（事件在同一聚合内有序，跨聚合无序）——事后加并行需要重写投影器。
2. **重放和实时消费共用同一段投影代码**，否则会出现"重放结果 ≠ 实时结果"这种最难查的 bug。

### GDPR 删除 vs 事件不可变：crypto-shredding

冲突是真实的：GDPR Art. 17 要求"被遗忘权"，事件日志的核心性质是 append-only 不可变。三种解法：

| 方案 | 做法 | 代价 |
|---|---|---|
| 重写日志 | 停机，过滤重写全部分区 | 破坏 offset、破坏下游 checkpoint，实践上不可行 |
| PII 外置 | 事件里只存 `subject_id`，PII 存可删的独立表 | 事件失去自包含性，重放要 join 外部表（且历史值已丢） |
| **Crypto-shredding** | **每个 data subject 一把 DEK，事件中的 PII 字段用它加密；删除 = 删 DEK** | key store 成为新的关键路径与单点 |

```json
{ "event_type": "CustomerRegistered", "subject_id": "cus_9f2a",
  "enc":  { "alg": "AES-256-GCM", "key_id": "dek:cus_9f2a:v1" },
  "data": { "country": "DE",              // 非 PII，明文，投影可直接用
            "email": "b64:8Kx2...",       // 密文
            "name":  "b64:9Qz1..." } }
```

**六个必须一起处理的地方**（漏掉任何一个，删除就是假的）：

```
删 DEK 之后仍持有明文的地方：
  □ 读模型 / 投影表        → 显式 UPDATE 成 tombstone
  □ 聚合快照               → 作废并重建（重建后 PII 字段解密失败 → 占位符）
  □ 下游服务的本地副本     → 发 SubjectErased 事件，各下游自行擦除（要求确认回执）
  □ 备份 / DR 副本         → 备份保留期必须 ≤ 删除 SLA，或备份也按 DEK 加密
  □ 日志 / APM / trace     → 结构化日志里的 PII 字段是最常被漏的
  □ 搜索索引 / 向量索引     → embedding 可被反演，向量本身即 PII 派生物
```

⚠️ **最容易踩的坑：key store 自己开了 PITR**。你删了 DEK，但 KMS 的时间点恢复能把它找回来 —— 那删除就没发生。要么 key store 不开 PITR，要么 PITR 窗口严格短于删除 SLA（行业实践 30 天）。

⚠️ 第二个坑：**crypto-shredding 不能用于需要按该字段聚合的场景**。密文无法 GROUP BY、无法建索引、无法做范围查询。所以"哪些字段进 DEK"是一个产品决策，不是加密决策。

---

## 5. CQRS：读模型与它的滞后

### CQRS 的四个层级（大部分人只需要前两级）

| 级别 | 形态 | 复杂度 | 何时用 |
|---|---|---|---|
| L0 | 同一个模型读写 | 0 | 默认 |
| L1 | 同库，读写用不同的对象/查询（读走只读副本） | 低 | **80% 的"我们要 CQRS"其实只需要这个** |
| L2 | 读模型在另一个存储（ES / ClickHouse / 物化视图），异步投影 | 中 | 读写负载特征差异 10× 以上；读需要完全不同的索引形态 |
| L3 | L2 + 事件溯源 | 高 | 需要时间旅行、需要重建任意历史读模型 |

**CQRS ≠ 事件溯源。** 可以只做 CQRS 不做 ES（从 CDC 建读模型），也可以只做 ES 不做 CQRS。混为一谈是这个话题最常见的误解。

### 读模型的构建

```
  写侧（真相源）        outbox/CDC      ┌─ order_list_view   ← 为 UI 的一张屏定制
 ┌───────────────┐                     │
 │ orders+outbox │──────► Projector ───┼─ order_search_idx  ← Elasticsearch
 └───────────────┘        (checkpoint) │
                                       └─ order_metrics     ← ClickHouse
```

**投影器的四个必备件：**
1. **checkpoint**（消费到哪个 offset/LSN），且 checkpoint 与投影写入必须原子（同库事务，或投影表里带 offset 列）。
2. **幂等 upsert**：`INSERT ... ON CONFLICT DO UPDATE WHERE view.version < excluded.version`，用事件里的聚合版本号做守卫，天然抗重复与乱序。
3. **重建脚本**：一条命令重放到一张新表，然后原子切换（改视图指向 / 改别名）。**没有重建能力的读模型是不可修复的。**
4. **滞后指标**：`projection_lag_seconds`（事件 `occurred_at` 到投影写入的时间差）。

### 读模型滞后带来的 UX 问题

用户点了"保存"，跳回列表页，**看不到自己刚保存的东西** —— 这是 CQRS 最真实的成本，且是产品级的，不是技术细节。

| 解法 | 做法 | 代价 | 适用 |
|---|---|---|---|
| **乐观 UI** | 前端本地先渲染写入结果 | 前端要维护"待确认"状态与回滚 | 单条记录的创建/编辑，**首选** |
| **命令返回结果** | POST 直接返回投影后的完整对象，前端拿它填充 | 命令侧要能算出读侧形状 | 单对象场景，与乐观 UI 组合最好 |
| **粘性读主** | 写后 N 秒内，该用户的读走写侧/主库 | 主库承担读流量；需要会话粘性 | N = 投影 p99 × 3，典型 2–5 秒 |
| **版本水位（consistency token）** | 写返回一个 token，读带上，投影未追上就等 | 最正确，但把异步变回半同步 | 强需求场景（金额、权限变更） |
| **同步投影关键路径** | 关键的那 1 个读模型在写事务内同步更新 | 放弃了 CQRS 的一半好处 | 只对"必须立刻可见"的那一张视图用 |

**版本水位的具体实现：**

```http
POST /orders                       →  201 Created
                                      X-Consistency-Token: orders_view:918273

GET /orders?after=orders_view:918273
   投影器当前 offset ≥ 918273  → 正常返回
   否则                        → 最多等待 200 ms（轮询/notify）
                                 超时仍未追上 → 200 + X-Stale: true，前端显示"同步中"
```

**参数**：等待上限 **200–500 ms**。超过这个数，你只是把异步系统改造成了一个更慢的同步系统 —— 那不如一开始就别做 L2。

**目标线**：`projection_lag` p99 < 500 ms；> 2 s 时用户可感知；> 30 s 应该触发页面级降级提示，而不是让用户看到不存在的数据。

---

## 6. 什么时候**不要**用事件溯源 / CQRS

| 情况 | 为什么不要 |
|---|---|
| 只是想要审计日志 | **写一张 append-only 审计表就够了**，成本是 ES 的 1/50。ES 的价值是"从事件重建任意状态"，审计不需要这个 |
| 只是想要读写分离的性能 | 只读副本（L1）能解决 90% 的问题，且零一致性成本 |
| 领域本身是 CRUD | 用户资料、配置项、字典表。没有有意义的"状态变迁史"，事件流只是 UPDATE 的复述 |
| 团队没做过 | ES 的错误无法回滚 —— 事件建模错了（粒度太粗/太细）之后，历史事件永远错着 |
| 有强 GDPR 删除义务且没做过 crypto-shredding | 上线后再补，是全量重加密 |
| 需要跨聚合的强一致约束 | ES 的一致性边界是单聚合。"全局唯一邮箱"在 ES 里需要额外的唯一性预留服务 |

**三个具体反模式：**

1. **全系统事件溯源。** ES 是**局部**技术，用在那 1–2 个真正有复杂状态机的聚合上（订单、账户余额、工单）。给"用户偏好设置"做 ES 是纯亏损。
2. **为每个 CRUD 操作发领域事件。** `UserProfileFieldUpdated` 不是领域事件，是 UPDATE 的转述。事件应该对应**业务上有名字的事情**（`OrderCancelled`、`SubscriptionDowngraded`）。如果你的事件名里有 "Updated"/"Changed"，先停下来想想。
3. **CQRS 但没有重建能力。** 投影器上线三个月后必然有 bug 导致读模型脏。没有一键重建，你只能手写修数据 SQL —— 而写侧还在持续产生新事件。

**CQRS 税的量化**：每个额外的读模型 ≈ 一个消费者进程 + checkpoint 管理 + 重建工具 + 滞后监控 + schema 版本 ≈ **0.1–0.2 个工程师/年**的持续成本（对照 [`01-microservices`](01-microservices-vs-modular-monolith.md) 里"每个服务 0.2–0.5 人年"）。5 个读模型 = 一个人的一半时间。

---

## 7. 事件驱动系统的调试与可观测

### 三个 ID，含义各不相同

```json
{ "event_id":       "01J8...",     // 这条事件本身（去重用）
  "correlation_id": "req_a1b2",    // 整条业务流程（一次点击产生的所有事件共享）
  "causation_id":   "01J7...",     // 直接导致本事件的那条事件的 event_id
  "trace_id":       "4bf92f..." }  // W3C traceparent，与 APM 打通
```

**`causation_id` 是唯一能重建因果树的东西**，绝大多数团队只做了 `correlation_id`，结果只能看到"这次请求产生了 47 条事件"，看不到"谁引发了谁"。

```
cmd:PlaceOrder (corr=req_a1b2)                         ← 重建出来的因果树
  └─ OrderCreated              (caus=cmd)
      ├─ InventoryReserved     (caus=OrderCreated)
      │   └─ PaymentAuthorized (caus=InventoryReserved)
      └─ SearchIndexUpdated    (caus=OrderCreated)     ← 编舞分支，与主流程无关
```

发现事件环（A→B→C→A）就是在这棵树上找重复的 `event_type` 路径。生产上把"同一 correlation_id 内事件数 > 100"设成告警，那基本就是环。

### 重放工具：三种重放，风险完全不同

| 重放类型 | 目标 | 副作用风险 | 必须的护栏 |
|---|---|---|---|
| **投影重放** | 重建读模型 | 无（投影是纯函数） | 写到影子表再切换；限速避免打爆 DB |
| **处理器重放** | 修复业务逻辑 bug 后重跑 | **高**：会重新发邮件、重新扣款 | 副作用适配器必须有 `dry_run` 模式；幂等键必须已存在于下游 |
| **环境重放** | 把生产事件放到 staging | 中：可能打到生产的第三方 | 出口全部走 mock；PII 必须脱敏或用假 DEK |

> **面试金句**
> "重放能力必须在第一天就设计，但**处理器重放默认应该是禁用的**。我会把所有有外部副作用的步骤放在一个显式的 effects 层后面，重放时这层切成 no-op 或 dry-run。区分'重算派生数据'和'重做事情'—— 前者永远安全，后者永远需要人审批。这也是我不会给 Agent 系统开放自动重放的原因。"

### 指标（缺一不可）

| 指标 | 健康值 | 异常含义 |
|---|---|---|
| `saga_completion_rate` | > 99.5% | 下降 = 某个下游在批量失败 |
| `saga_compensation_rate` | < 1%（业务相关） | 突增 = 上游校验失效或下游容量不足 |
| `saga_duration` p99 | 按流程定 SLO | 上升先于失败出现，是最好的先行指标 |
| `saga_stuck_count` | 0 | > 0 必须有人看，这里是钱漏出去的地方 |
| `projection_lag_seconds` p99 | < 0.5 s | > 2 s 用户可感 |
| `dlq_depth` | 0 | 见 [`01-building-blocks/03`](../01-building-blocks/03-messaging-and-streams.md) |
| `event_ordering_violations` | 0 | 同聚合版本号回退 = 分区键错了 |

### Agent 系统：每个 Agent 循环都是一个 Saga

这是 2025–2026 最有价值的迁移：**Agent 的多步工具调用在结构上就是 saga，而且它的补偿问题更糟**——

- 工具调用的副作用（写文件、发 PR、调 API、花掉 token）**大多不可补偿**，而且 Agent 会**自己决定**调用顺序，你无法像订单流程那样静态地把 pivot 排到最后。
- **对策**：不可逆动作的清单必须**设计期静态定义**，运行时不能交给模型自判；这类动作统一走人在回路审批。按 Five Eyes 五国联合指南 [《Careful Adoption of Agentic AI Services》](https://www.cisa.gov/resources-tools/resources/careful-adoption-agentic-ai-services)（2026-05）的口径，这是**必需控制项**而非可选增强。
- **幂等键规范化为 `(workflow_id, step_id)`**：与上面 Saga 的 `IDEM(saga_id, step_id)` 完全同构。Agent 重试/恢复时不会重复扣款。
- ⚠️ **checkpoint ≠ durable execution**（这一点存在明确争议）：一派认为配了 checkpointer 就能任意点 pause/resume 即为 durable；另一派（如 Diagrid）认为状态快照恢复缺少确定性重放与精确一次副作用，不足以支撑生产工作流。**分歧的根源是对 "durable" 的定义不同**：状态快照恢复 vs 执行历史重放。工程判据很简单：**你的恢复路径会不会重复执行外部副作用？** 会 → 你需要的是 journal + replay 或幂等键，不是 checkpoint。

---

## 面试官会追问

1. Saga 和 2PC 的区别是什么？Saga 牺牲了 ACID 的哪一个字母，会产生哪三种异常？
2. 你的订单流程里"发货"这一步没法补偿，怎么设计步骤顺序？pivot 是什么？
3. 补偿事务失败了怎么办？（→ 不允许"补偿失败"这个终态，只能进人工队列）
4. 幂等键从哪来？为什么不能用 `uuid4()`？
5. 事件溯源系统里，用户要求删除个人数据，怎么做？crypto-shredding 之后还有哪些地方留着明文？
6. 5 亿条事件的读模型要重建，需要多久？这个数字影响你的什么决策？
7. 用户保存后跳转，看不到自己刚写的数据，你有几种解法？各自代价是什么？
8. 什么时候你会拒绝上事件溯源？（→ 只想要审计、领域是 CRUD、团队没做过、GDPR 未设计）
9. 编排和编舞怎么选？编舞在什么规模下不可维护？
10. `correlation_id` 和 `causation_id` 有什么区别？没有后者会失去什么能力？

---

**下一篇** → [03-multi-tenancy.md](03-multi-tenancy.md)
