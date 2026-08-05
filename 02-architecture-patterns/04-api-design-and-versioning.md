# 04 · API 设计与演进

> API 是你唯一无法回滚（roll back）的东西。代码可以重写，数据库可以迁移，但已经发出去的响应形状永远在别人的生产代码里。
> 所以 API 设计的核心不是"怎么设计得优雅"，而是"怎么设计得可以在不打断任何人的前提下改变"。

---

## 读这一章之前

**你在工作中遇到过这个**

你给订单状态加了一个新的枚举值 `chargeback`，评审会上所有人都同意"只是加个取值，不算破坏性变更"。
灰度 20 分钟后，某个大客户的告警群炸了：他们的 SDK 用穷举 `switch` 匹配状态、没有兜底分支，收到未知值直接抛异常，整条对账链路停摆。
你回滚了，可那 20 分钟里没被处理的回调得你写脚本一条条补回去 —— 更糟的是，你现在不知道还有哪几个客户是同样的写法。

**需要先懂的概念**

| 概念 | 一句话 | 详见 |
|---|---|---|
| 幂等 | 同一个操作执行多次，效果和执行一次相同 | [00-concepts §12](../00-foundations/00-concepts.md#12-本章术语速查)、[00/01 §5](../00-foundations/01-fundamentals.md#5-幂等idempotency分布式系统的第一公民) |
| 超时 / 重试 / 退避 | 网络超时后你分不清"请求丢了"还是"响应丢了"，所以一定会重试 | [00/01 §9](../00-foundations/01-fundamentals.md#9-幂等--重试--超时三件套必须一起设计) |
| 复制延迟与最终一致 | 刚写进主库的数据，从副本上可能还读不到 | [00-concepts §6](../00-foundations/00-concepts.md#6-什么是一致性--一个词两种完全不同的意思) |
| 分位数 p99 | 排序后第 99% 位置的那个耗时，决定"最倒霉那批用户"的体验 | [00-concepts §3](../00-foundations/00-concepts.md#3-为什么平均值是骗人的p50--p90--p99) |
| 背压与限流 | 处理不过来时要往上游反向施压，而不是无限排队 | [00/01 §6](../00-foundations/01-fundamentals.md#6-背压backpressure没有它系统就会雪崩cascading-failure) |

**这一章要回答的问题**

1. 已经有几百个客户在用的 API，怎么在不打断任何一个人的前提下改它的形状？
2. 客户端超时后重发了同一笔支付，我凭什么保证不会扣两次钱？
3. 分页翻到第 10,000 页为什么会又慢又漏行？公开 API 该给客户端什么形式的"下一页"？
4. 流式响应已经把 HTTP 200 发出去了，中途模型服务过载，我要怎么告诉客户端？

**本章新引入的术语**

| 术语 | English | 一句话定义 |
|---|---|---|
| 破坏性变更 | breaking change | 一次服务端改动，让不改代码的已有客户端从"能工作"变成"报错或行为不对" |
| 不透明游标 | opaque cursor | 服务端生成、客户端只能原样回传的"下一页位置"字符串，内部结构不对外承诺 |
| keyset 分页 | keyset / seek pagination | 用"上一页最后一行的排序键值"作为下一页起点，而不是"跳过前 N 行" |
| 全序 | total order | 一种排序方式，任意两行都能分出先后，不存在两行完全并列 |
| Problem Details | problem+json (RFC 9457) | HTTP 错误响应的标准 JSON 结构，固定含 type / title / status / detail 四个成员 |
| 可重试性标注 | retryability | 服务端在错误响应里显式告诉客户端"这个错误再试一次有没有意义" |
| 枚举炸弹 | enum bomb | 服务端给枚举加了一个新取值，把所有做了穷举分支的客户端打崩 |
| 日期版本头 | dated version header | 用 `2023-06-01` 这样的日期做 API 版本号、放在请求头里，可按账号单独固定 |
| brownout | brownout | 下线一个旧版本之前，每天故意让它在固定的几分钟里返回错误，逼没迁移的客户暴露出来 |
| 长任务 | long-running operation | 立刻返回一个任务 ID、让客户端随后自己查状态的接口形态，用于耗时几十秒以上的操作 |
| 带内错误 | in-band error | HTTP 状态码和响应头已经发出去之后，只能把错误当作流里的一条数据再发给客户端 |
| 瘦载荷 / 胖载荷 | thin / fat payload | webhook 里只带资源 ID 让消费者回读 / 直接带上资源的完整状态 |

---

## 1. 协议选型：三分钟决策

| 维度 | REST / JSON | gRPC | GraphQL |
|---|---|---|---|
| 序列化与传输 | 文本体积较大，工具生态成熟 | protobuf 通常更紧凑；收益要以真实载荷压测 | JSON 传输之外还有解析与 resolver 规划成本 |
| 演进机制 | 靠约定（加字段安全） | **字段号 + reserved**，机制内建 | schema deprecation + 字段级用量统计 |
| 可缓存性（cacheability）/ 调试 | **HTTP 缓存全套可用** / curl | 基本无（都是 POST）/ grpcurl | 差 / playground |
| 适用 | 公开 API、合作伙伴 | **内部东西向（east-west traffic）**、移动端长连接 | BFF、字段需求高度可变的前端 |

**常见起点**：公开/合作伙伴 API → REST + OpenAPI；内部东西向（**east-west** = 服务与服务之间的内部调用，区别于用户从外面打进来的**南北向 north-south**）→ REST 或 gRPC；前端聚合 → **BFF**（backend for frontend：为某一种前端形态专门做的聚合层）或 GraphQL；需要服务端单向流式输出时可考虑 **SSE**（Server-Sent Events：服务端在一条长连接上持续推事件，§10 展开）；异步处理见 [messaging](../01-building-blocks/03-messaging-and-streams.md)。协议要根据客户端生态、浏览器支持、调试方式、延迟和演进成本选择，不是按“内部/外部”机械套公式。

**反直觉的一条**：GraphQL 不是“更好的 REST”，它让客户端声明所需字段、由服务端执行查询计划。代价是 N+1（取回 N 条记录后又为每条各查一次）可能转移到 resolver 层，需要批量加载；限流也常要从“请求数”升级为深度、字段权重或预计成本。HTTP/CDN 缓存并非不能做，但通常要依赖 persisted query、GET 或应用层缓存。当前端数据组合变化快、聚合源多，且团队愿意治理查询成本时，它才更划算；深度和复杂度上限应由压测与成本预算决定。

---

## 2. 资源建模与 URL：实际规则

```
✅ GET  /v1/workspaces/ws_42/api_keys?status=active&limit=20
✅ POST /v1/exports/exp_01J8.../cancel      ← 动作作为子资源，不要硬套 CRUD
❌ POST /v1/getUserById                      ← RPC 伪装成 REST
❌ GET  /v1/users/42/orders/7/items/3/tags   ← 超过两层嵌套
❌ GET  /v1/users?id=42                      ← 单资源用查询串
```

**七条实用起点（有反例时把例外写进契约）**：

1. **避免过深嵌套**。当子资源能独立寻址或嵌套让 URL 难以演进时，改用顶层资源 + 过滤：`/items?order_id=7`；“两层”是可读性经验，不是协议限制。
2. **对外 ID 优先用带类型前缀的不透明字符串**：`msg_01J8XQ...`。ULID / UUIDv7 在高位编码时间，便于大致按生成时间排序，但唯一性仍依赖生成算法，连续键也可能形成写热点，是否直接做主键要压测。自增整数实现最简单，却可能泄露业务量并增加跨库 ID 分配成本；`msg_` 这类类型前缀还能在校验层发现把订单 ID 传到消息参数的错误。
3. **动作用子资源 POST**，不要发明 `PUT /orders/7?action=cancel`。
4. **标量约定**：时间一律 RFC 3339 UTC 带毫秒（`2026-07-30T11:02:03.412Z`）；金额用最小单位整数（minor units）+ 币种（`{"amount":1250,"currency":"USD"}`）或字符串 decimal，**永远不要用浮点表示钱**。
5. **枚举（enum）全小写下划线**，并在契约（contract）里写死"客户端 MUST 把未知枚举当 `unknown` 处理"（见 §7 枚举炸弹）。
6. **需要分页或元数据的列表用对象包裹**：`{"data":[...],"next_cursor":"..."}`。简单且确定永不扩展的内部接口可以返回数组，但公开 API 往往很快会需要游标、告警或链接。
7. **PATCH 语义必须选一个写进文档**：JSON Merge Patch（[RFC 7386](https://www.rfc-editor.org/rfc/rfc7386)，简单，但 `null` 通常表示删除字段）、JSON Patch（[RFC 6902](https://www.rfc-editor.org/rfc/rfc6902)，表达力强，客户端更复杂）、field mask（如 Google AIP-134，需要额外定义空值语义）。没有“唯一正确”选择；按客户端能力、数组操作和 `null` 语义决定。

---

## 3. 错误契约：RFC 9457 + 可重试性（retryability）标注

[RFC 9457 Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)（2023，废弃 RFC 7807）定义了 `application/problem+json`。核心成员：`type`（问题类型 URI）、`title`、`status`、`detail`、`instance`。

RFC 9457 解决通用错误外形，业务 API 通常还需要稳定机器码；如果 SDK 要自动重试，也可以加明确的可重试性扩展：

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/problem+json
RateLimit-Policy: "tokens";q=400000;w=60
RateLimit: "tokens";r=0;t=42
Retry-After: 42
request-id: req_01J8XQ7YB3F0KJ2M

{ "type":   "https://api.acme.dev/problems/rate-limit-exceeded",
  "title":  "Rate limit exceeded",
  "status": 429,
  "detail": "Token budget for workspace ws_42 exhausted (400,000 tok/min).",
  "instance": "/v1/messages",
  "code":      "rate_limit.tokens_per_minute",  ← 稳定机器键，独立于 type URI
  "retryable": true,                            ← 显式标注，不让客户端从状态码猜
  "retry_after_ms": 42000,
  "request_id": "req_01J8XQ7YB3F0KJ2M",
  "docs": "https://docs.acme.dev/errors/rate_limit" }
```

**`code` 必须独立于 `type`**：`type` 是 URI，你迟早会重构文档站的路径结构；一旦客户端把 `type` 当成 switch 的 case，你的文档 URL 就成了 API 契约的一部分。给一个短的、永不改的 `code`，让 `type` 只承担"人去点开看"的职责。**`retryable` 必须显式给**，因为状态码到可重试性的映射不是全序的：

| 状态 | 可重试 | 前提 / 例外 |
|---|---|---|
| 400 / 422 参数错 | ❌ | 重试只浪费配额 |
| 401 / 403 认证授权 | ❌ | 例外：token 过期（用 `code` 区分 `auth.token_expired`，可重试一次） |
| 404 不存在 | ❌ | 例外：**读己之写（read-your-writes：自己刚写完的东西，自己再读一定读得到）场景可重试 1 次** —— 写落在主库、读打到了从副本，从副本还没追上，这段差距叫**复制延迟（replication lag）**，见 [00-concepts §6](../00-foundations/00-concepts.md#6-什么是一致性--一个词两种完全不同的意思) |
| 409 冲突 | ⚠️ 分情况 | 乐观锁（optimistic locking：不加锁，提交时用版本号检查有没有被别人改过，见 [00-concepts §7](../00-foundations/00-concepts.md#7-事务与隔离级别)）版本冲突 → 重读后重试；**幂等键**（idempotency key：客户端生成、服务端拿它给重试去重的请求标识，见 [00/01 §5](../00-foundations/01-fundamentals.md#5-幂等idempotency分布式系统的第一公民)，本篇 §5 展开）冲突 → **绝不重试** |
| 408 / 连接超时或断开 | ⚠️ 分情况 | 结果可能是“请求没到”或“响应丢了”；只有读取，或带稳定幂等键/可查询操作状态的写入，才可自动重试。`499` 是部分代理使用的非标准状态码 |
| 429 超配额 | ✅ | 必须遵守 `Retry-After` |
| 500 内部错误 | ⚠️ 分情况 | 只有操作可安全重试，并且仍在超时/重试预算内时才重试；5xx 不证明业务写入没有提交 |
| 502 / 503 / 504 上游或容量 | ⚠️ 通常可重试 | 前提仍是操作安全；遵守 `Retry-After`，使用退避 + 抖动并限制次数（见 [00/01 §9](../00-foundations/01-fundamentals.md#9-幂等--重试--超时三件套必须一起设计)） |
| 其他厂商自定义码 | 看契约 | 不要把某家厂商的非标准状态码当成通用 HTTP 语义；优先读取稳定的 `code` / `retryable` |

> **面试金句**：
> “429 通常表示当前客户端触发了限额，503 表示服务暂时无法处理；两者都可能带 `Retry-After`，但长期应对不同。客户端仍要以 API 契约里的稳定错误码为准，不能只看状态码猜。”

**字段级校验错误**用 `errors` 扩展数组（RFC 9457 未标准化，但已是事实约定），`pointer` 用 [RFC 6901 JSON Pointer](https://www.rfc-editor.org/rfc/rfc6901)，前端才能把错误自动挂到对应表单控件上：

```json
"errors": [
  {"pointer": "/messages/0/content", "code": "required",     "detail": "must not be empty"},
  {"pointer": "/max_tokens",         "code": "out_of_range", "detail": "must be 1..64000"}
]
```

---

## 4. 分页（pagination）：offset 是个陷阱

| 方式 | 第 1 页 | 第 10,000 页 | 并发写入下 | 可跳页 | 适用 |
|---|---|---|---|---|---|
| **offset/limit** | 通常快 | 随 offset 增大而变慢 | 活跃数据集可能漏行/重行 | ✅ | 小数据集、后台页、需要跳页 |
| **keyset（seek）** | 通常快 | 延迟通常更稳定 | 在固定排序契约下稳定前进 | ❌ | 已知排序、连续浏览 |
| **opaque cursor** | 取决于内部实现 | 可封装 keyset 或快照状态 | 取决于游标是否绑定排序/过滤/快照 | ❌ | 不想暴露实现细节的公开 API |

**offset 的两个独立灾难**：

```
灾难 1（性能）  SELECT ... ORDER BY created_at DESC LIMIT 20 OFFSET 200000
               数据库必须真的取出 200,020 行再丢掉前 200,000 行。O(N)。
灾难 2（正确性）第 1 页读了 [100..81]，此时插入 5 行；
               第 2 页 OFFSET 20 读到 [85..66] → 81..85 被重复返回。删除同理会漏行。
               这个 bug 在导出/对账场景是致命的，而且不会报错。
```

### keyset 的正确实现（排序必须是全序 total order）

**核心要求：排序键必须唯一。** `ORDER BY created_at DESC` 在 created_at 有重复时不是全序，翻页会乱。必须补一个唯一列做 tie-break。

```sql
-- 必需索引：列顺序必须与 ORDER BY 完全一致
CREATE INDEX idx_events_page ON events (tenant_id, type, created_at DESC, id DESC);

-- 第一页：多取 1 行判断 has_more，不要用 COUNT
SELECT id, created_at, payload FROM events
WHERE tenant_id = $1 AND type = $2
ORDER BY created_at DESC, id DESC LIMIT 21;

-- 后续页：行值比较（row-value comparison），能吃到复合索引
SELECT id, created_at, payload FROM events
WHERE tenant_id = $1 AND type = $2
  AND (created_at, id) < ($3::timestamptz, $4::text)   -- ← 不要拆成 OR 条件
ORDER BY created_at DESC, id DESC LIMIT 21;
```

元组比较更短，也更不容易把升降序和边界写错。展开成 `created_at < $3 OR (created_at = $3 AND id < $4)` 在逻辑上等价，能否同样使用索引取决于数据库和执行计划；不要靠猜，使用目标数据库的 `EXPLAIN` 验证。

### cursor 的编码方案（可直接抄）

```
cursor = base64url(payload) + "." + base64url(HMAC-SHA256(key, payload)[:16])
payload = {
  "v": 1,                                        // 游标格式版本，允许你换实现
  "s": "created_at:desc,id:desc",                // 排序契约全文
  "k": ["2026-07-30T11:02:03.412Z", "01J8XQ7Y"], // 上一页最后一行的排序键值
  "f": "9f2a1c4e",                               // 规范化后过滤条件的哈希
  "t": "ws_42",                                  // 租户 ID
  "exp": 1785400000 }                            // 过期时间（7 天）
```

**每个字段都在防一种事故**：`v` 防你换了排序实现后旧游标返回错数据；`s` 防客户端换 `sort=` 却复用旧游标（直接 400，而不是静默返回乱序）；`f` 防换过滤条件后复用（**最常被忽略的一条**）；`t` 防跨租户重放（replay attack）；`exp` 让你有权在任意时刻废掉全部游标。
**HMAC 不是为了保密，是为了防止客户端把 cursor 当成任意 `WHERE` 条件的注入点。**

响应里给 `next` 的**完整 URL**（不是让客户端自己拼参数），这样你以后换分页参数名不用改客户端：
`{"data":[...], "has_more":true, "next":"https://api.acme.dev/v1/events?...&cursor=eyJ2..."}`

**不要默认返回精确 `total_count`。** 对大范围过滤，精确计数可能需要扫描大量索引或数据；如果产品确实需要页数或进度，可以异步计算、缓存，或返回明确标注的 `total_count_estimate`。是否昂贵取决于过滤条件、索引、数据库统计信息和并发，应该用执行计划与线上指标判断。

---

## 5. 幂等契约（idempotency contract）

`Idempotency-Key` 是客户端和服务端共同约定的重试标识。具体头名、保留期和冲突状态码要写进自己的 API 契约；如果参考 [IETF httpapi 工作组文档](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)，上线前还要核对当时版本，不能把草案文字默认为稳定标准。

先定义作用域，避免两个端点碰巧用了同一个 key：

```
scope       = (tenant_id, operation, idempotency_key)
operation   = normalized(method, route_template)       # POST /v1/payments，而不是含具体 ID 的原始 URL
fingerprint = SHA256(canonical(relevant_headers, body))
```

同一作用域下，fingerprint 不同说明客户端错误地复用了 key，应返回稳定的 4xx 错误；相同才允许重放。服务端有两种实现，关键区别是业务操作能不能放进一个短事务。

### 模式 A：业务写和幂等记录在同一个数据库

```
BEGIN
  INSERT idempotency(scope, fingerprint, status, response)
    VALUES (..., 'running', NULL)       # scope 有 UNIQUE 约束
  执行短小的业务写
  UPDATE idempotency
    SET status='done', response=要重放的状态码/必要响应头/body
COMMIT
```

插入、业务写和终态响应**一起提交**。并发重复请求会在唯一键上等待；第一个事务提交后，重复请求读取并重放 `done`，第一个事务回滚后则可以重新竞争。这里不需要把一个可见的 `running` 先单独提交，否则反而制造“记录存在，但业务到底做没做”的中间状态。响应体过大时可以保存资源 ID 和足以重建相同语义响应的结果，而不是无限复制 body。

### 模式 B：包含长任务、外部支付或跨库副作用

此时无法用一个数据库事务包住网络调用。需要一个**持久化意图状态机**：

```
accepted → running(owner, lease_until, lease_epoch)
         → succeeded(response)
         → known_failed(error)
         → unknown(needs_reconciliation)
```

Worker 用短事务领取任务并递增 `lease_epoch`，网络调用放在事务外；对下游始终复用同一个稳定幂等键。完成时用 `(owner, lease_epoch)` 做 fenced update，过期 worker 的迟到结果不能覆盖新 owner。遇到超时或 5xx 时，**不能直接删幂等记录**：业务副作用可能已经提交、只是响应丢了。应先重试同一个下游 key、查询下游操作状态，或进入 `unknown` 交给对账/人工处理。只有明确知道“尚未执行”的失败，才可标成可重试。

**四条容易踩的坑：**

1. **保留窗口必须覆盖完整重放地平线。** 不只算 SDK 自动重试，还要算队列延迟、离线客户端和人工补跑；超过窗口后再用旧 key，系统只能按新请求处理。窗口越长，存储和隐私删除成本越高，应由业务流程与数据量共同决定。
2. **随机 key 完全可以正确。** 错误不是 `uuid4()` 本身，而是每次重试都重新生成。客户端或 SDK 要在第一次发送前生成一次、持久化，并在同一操作组的所有重试中复用。业务派生 key 也可以，但要避免可猜测信息泄露和不同操作误碰撞。
3. **不要把所有响应头原样永久重放。** `Date`、临时签名 URL、跟踪头等可能已过期；契约要明确保存哪些字段、哪些字段重建，以及重复请求是否沿用原 `request_id`。
4. **幂等不等于全局 exactly-once。** 它只约束给定作用域和保留窗口内的同一个 key；换 key、窗口过期或作用域之外的下游副作用仍要独立去重和对账。

**HTTP 方法不要混为一谈**：GET、HEAD、PUT、DELETE 在 HTTP 语义上定义为幂等，表示重复相同请求的预期效果相同，并不保证每次状态码或响应体相同。`If-Match` 是防止丢失更新的乐观并发条件，不是让 PUT 变幂等的原因。PATCH 不天然幂等；POST/PATCH 只有在产品承诺“副作用操作可安全重试”时才需要 `Idempotency-Key`，普通查询或天然可重复赋值不必机械添加。

---

## 6. 限流（rate limiting）：响应头与客户端契约

限流响应头有标准化工作，也有大量历史上的 `X-RateLimit-*` 约定。下面用 [IETF httpapi RateLimit 文档](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)的结构化字段形式说明（`q`=配额、`w`=窗口秒、`r`=剩余、`t`=距重置秒）；实施前请核对当时规范状态与字段语义：

```http
RateLimit-Policy: "requests";q=1000;w=60, "tokens";q=400000;w=60
RateLimit:        "requests";r=873;t=37,  "tokens";r=0;t=42
```

不要无条件同时发两套含义可能不同的头。新 API 选定一种契约并在 SDK 中实现；已有客户端依赖 `X-RateLimit-*` 时，可在迁移期同时发送，并用契约测试确保两套值表达同一个窗口和重置时刻。

**多维限额是 LLM API 的常态**（请求数、输入 token、输出 token、并发数四个桶各自独立）。三条要求：所有维度都要在响应头里暴露（否则客户端只能靠撞墙学习）；`Retry-After` 只反映当前绑定的那个维度；**problem 的 `code` 必须指出是哪个桶满了** —— `rate_limit.output_tokens_per_minute` 的应对是减小 `max_tokens`，`rate_limit.requests_per_minute` 的应对是减小并发，完全不同。

`Retry-After` 可以是延迟秒数或 HTTP 日期。秒数通常更容易正确实现；无论选哪种，SDK 都应做上限裁剪并容忍缺失或格式错误。

**客户端侧契约（写进 SDK，不要指望用户自己写）**：

```python
delay = random.uniform(0, min(cap, base * 2**attempt))  # full jitter
delay = max(delay, retry_after_from_header or 0)
# 同时受最大次数、总耗时 deadline 与 retry budget 约束；阈值由 SLO 和压测确定
```

（**熔断 circuit breaker**：错误率超过阈值后的一段时间内，客户端**直接快速失败、一个请求都不发给下游**，冷却期过了再试探性放几个过去。它和限流的区别是：限流说的是"我只让你发这么多"，熔断说的是"我现在一个都不发"。）

---

## 7. 版本化（versioning）：策略与迁移剧本

### 五种策略

| 策略 | 优点 | 代价 |
|---|---|---|
| **URL 路径** `/v1/` | 直观，路由/缓存友好 | 粒度较粗；跨大版本迁移需要同时维护两套路由 |
| **媒体类型** `Accept: application/vnd.acme.v2+json` | 资源级粒度 | 工具链差；CDN 要 `Vary: Accept`，缓存碎片化（cache fragmentation） |
| **日期版本头** `api-version: 2025-03-31` | 可按客户端固定版本，便于渐进迁移 | 缓存要正确设置 `Vary`；服务端要维护版本转换与测试链 |
| **不破坏式演进（non-breaking evolution）**（Google AIP / protobuf） | 零迁移成本 | 依赖客户端"忽略未知字段"；枚举扩展是隐形炸弹 |
| **预览功能头** `api-preview: feature-name` | 新能力不污染稳定版 | 一旦客户用于生产就很难撤；必须明确支持级别和退出日期 |

**一个可行组合**：URL 里的 `/v1` 表示很少变化的大结构，兼容演进优先通过加字段和弃用完成；确实需要长期并存的行为差异时，再用日期版本头按客户端固定。也可以只使用 URL 大版本，关键是团队能否长期维护和测试，而不是版本号放在哪里。版本转换器模式可让内部代码只面对最新表示，但转换未必可逆，也不是“几十行就完成”：必须覆盖请求与响应、错误、分页、webhook 和副作用语义，并为每条受支持链路保留契约测试。

### 什么算破坏性变更（breaking change）

| 安全（可随时上线） | 破坏性（必须走版本） |
|---|---|
| 新增端点；新增可选请求头 | 删除/重命名字段；改字段类型（含 `int → string`） |
| 新增**可选**请求字段（默认值保持旧行为） | 收紧校验；让可选字段变必填 |
| 新增响应字段；放宽校验 | 改默认值（含分页默认 size）、改默认排序 |
| 新增错误 `code`（在已有 `type` 下） | 改 HTTP 状态码；缩短幂等键保留窗口 |
| | **新增响应枚举值** ← 见下 |

**枚举炸弹**：新增枚举值在规范上"只是加东西"，现实中会打爆所有写了穷举 `switch`（exhaustive switch）的客户端（很多语言 match 不全直接抛异常）。两个缓解：① 契约第一天就写死"客户端 MUST 把未知枚举当 `unknown` 并保留原始字符串"；② **在 v1 发布时就注入一个 canary 枚举值**（一个真实但罕见的取值），让不合规的客户端在低流量时暴露，而不是在你上新功能的那天。

### 弃用剧本（deprecation playbook，用量遥测 telemetry 驱动）

没有遥测，剧本就是猜。最低粒度应到 `(client_id, api_version, endpoint, 最近使用时间)`。GraphQL 能看到客户端选择了哪些字段；普通 JSON 和 protobuf 服务端通常只能知道自己**发送了**什么，不能知道客户端实际读取了什么。字段级弃用因此还要结合 SDK/代码搜索、客户确认或显式 capability，而不能把“序列化过”当成“仍被使用”。

**流量和独立客户端都要看。** 高流量可能只是健康检查，低流量却可能是月末结算任务。关闭前应确认观察窗口覆盖所有业务周期，并把剩余 `client_id` 映射到 owner；“还有 N 个”本身不是安全阈值。

```
T-公告   新版本稳定可用；发布迁移指南、支持期限和弃用响应头
T-迁移   遥测到 client_id + endpoint；逐客户联系并跟踪 owner/状态
T-演练   如风险允许，公告后对测试环境或明确范围做 brownout，验证告警与回退
T-关停   只有剩余调用者、白名单、回滚条件和支持值班都清楚时才关闭
T+清理   观察无回退需求后再移除旧代码、转换器和监控
```

**brownout 是高风险但有效的演练手段，不是必选动作。** 它能暴露仍在使用旧版本的客户端，也会主动制造故障；支付、医疗等关键链路可能不适合。若采用，先限定客户/环境或很小流量，公告精确窗口、提供迁移链接与紧急回退，并把“客户是否收到告警”作为演练结果。

弃用头用 [RFC 9745](https://www.rfc-editor.org/rfc/rfc9745.html)（Deprecation，2025）+ [RFC 8594](https://www.rfc-editor.org/rfc/rfc8594.html)（Sunset）：

```http
Deprecation: @1767225600
Sunset: Sat, 31 Jan 2026 23:59:59 GMT
Link: <https://docs.acme.dev/migrate/v1-to-v2>; rel="deprecation"; type="text/html"
```

---

## 8. 长任务（long-running operation）：202 + 状态资源 + 通知

当操作耗时可能接近客户端、网关或负载均衡器的超时，或需要跨断线继续执行时，应改成异步长任务。不要用固定的“10 秒”判断：先列出端到端 deadline、各中间层当前配置、任务耗时分布以及客户端是否愿意等待。

```
 客户端                      API                     Worker            通知边
   │ POST /v1/exports          │                        │                │
   │ Idempotency-Key: ik_..    │ 写 task 行 + 入队       │                │
   ├──────────────────────────►│───────────────────────►│ 领取，写心跳    │
   │◄──────────────────────────┤ 202 Accepted                            │
   │   Location: /v1/exports/exp_01J8...  Retry-After: 5                 │
   │   {"id":"exp_01J8...","status":"queued"}                            │
   │ GET /v1/exports/exp_01J8… │                        │                │
   ├──────────────────────────►│                        │                │
   │◄──────────────────────────┤ 200 {"status":"running","progress":0.42}│
   │                           │◄───────────────────────┤ 完成 → 写终态   │
   │                           │                        ├───────────────►│ POST webhook
   │◄──────────────────────────┤ 200 {"status":"succeeded",              │
   │                           │      "result_url":"<预签名 15min>"}     │
```

**五条规则**：

1. **状态资源是一等资源（first-class resource）**：有自己的 URL、鉴权规则和明确保留期；是否允许列举取决于产品需求。不要暴露可被伪造或跨租户猜测的内部队列 ID。
2. **任务失败时 HTTP 状态仍是 200**，problem+json 放在 body 的 `error` 字段里 —— 因为"查询状态"这个操作本身成功了。**这是这类 API 最常见的设计错误**：用 500 表达"任务失败"会让所有客户端的重试逻辑误触发。
3. **`Retry-After` 用来建议轮询间隔**，客户端还应加抖动。缓存状态响应、限制并发轮询和推送通知也能抑制轮询风暴，它不是唯一旋钮。
4. **通知方式按消费者选择**：最小契约通常是可轮询状态资源；跨系统集成可加 webhook，交互式 UI 可加 SSE/WebSocket。每增加一种都意味着新的鉴权、重试和测试面。
5. **`/cancel` 应可安全重试**，并明确哪些状态还能取消。取消常是异步的，可使用 `cancelling` → `cancelled`；若工作已完成，要返回稳定、可解释的结果。

（图里那个 `result_url` 用的是**预签名 URL**（presigned URL）：一条带签名和过期时间的临时下载链接，拿到它的人在有效期内不需要再带任何凭证就能下载 —— 所以有效期要短，且链接本身要当敏感信息对待。）

> 这里讨论的是通用 HTTP 形态。具体协议或 SDK 的任务接口会演进，实施时应核对当前规范；可迁移的原则是：任务状态要持久化，客户端断线不应让结果变成“未知”。

---

## 9. Webhook：签名、重试、幂等、乱序（out-of-order）

### 签名（照抄 [Standard Webhooks](https://www.standardwebhooks.com/)）

```http
POST /hooks/acme HTTP/1.1
webhook-id: msg_2gY8kQ1nZ
webhook-timestamp: 1785398400
webhook-signature: v1,g0hM9SsE+OTPJTGt/tmIKtSyZlE3uFJELVlNIOLJ1OE= v1,K5oZfzN95Z9UVu1Esf...

{"type":"export.succeeded","data":{"id":"exp_01J8..."}}

signed_payload = "{webhook-id}.{webhook-timestamp}.{raw_body}"
signature      = base64(HMAC-SHA256(secret, signed_payload))
```

**四条：** ① **在 raw bytes 上验签，在 JSON 解析之前**（先 parse 再序列化会改变 key 顺序、空白或 Unicode 表示）；② 校验时间戳并设置与你的投递延迟和时钟误差相符的容差，过旧请求再结合事件 ID 去重；③ 签名格式要支持密钥版本或多个签名，让新旧密钥在可观测的轮换窗口内重叠；④ 用恒定时间比较（constant-time comparison，例如 `hmac.compare_digest`）。时间容差和重叠时长是安全策略，不是协议常数。

### 重试、乱序与载荷形态

一个**示例**退避表可以是 `5s → 5min → 30min → 2h → 5h → 10h`，但真正的次数、抖动、总投递窗口、禁用条件和人工 replay 窗口都要写进发送方契约，并由接收方容量与事件时效决定。

**除非契约明确提供按资源有序投递，否则消费者应按乱序设计。** 重试可能让 `updated` 先于 `created` 到达。消费者按 `webhook-id` 去重，去重记录的保留期至少覆盖发送方自动重试、人工 replay 和队列积压的最长窗口；资源更新最好带可比较的 `resource_version`。版本必须来自资源的单调序列或可比较提交位置，不能拿普通时间戳假装全序。

**瘦载荷（thin payload）vs 胖载荷（fat payload）是这里唯一真正的架构取舍：**

| | 胖载荷（带完整状态） | 瘦载荷（只带 ID，消费者回读） |
|---|---|---|
| 消费者延迟 / 陈旧风险（staleness） | 低 / 乱序时旧状态可能覆盖新状态 | 多一次回读 / 通常读到较新状态，但可能跳过中间状态或受复制延迟影响 |
| 你的读流量 / 泄露面 | 无额外 / 大（端点被劫持即泄露数据） | **+1 次/事件** / 小（回读要鉴权） |

选择取决于消费者是否需要每个中间事件、回读 API 的可用性/成本、数据敏感度和事件体大小，不能只按行业套结论。**两个常见必做项**：
- **SSRF 防护**（server-side request forgery：诱使**你的服务器**访问内网或云元数据地址）：用户注册端点等于让用户影响你的出站目标。端点注册和每次连接都要校验解析结果，阻止 loopback、私网、link-local、保留地址和不允许的端口；重定向后的目标也要重新校验。更稳妥的是走限制目的地的独立出口代理，并设置基于实测的连接/响应 deadline。只在注册时解析一次仍挡不住 DNS rebinding。
- **投递日志（delivery log）是一等资源**：`GET /v1/webhook_deliveries?status=failed` + `POST /v1/webhook_deliveries/{id}/replay`。没有它，每次客户说"我没收到"都要你去查日志。这是投入产出比最高的一个端点。

---

## 10. 专项选读：AI API 的特殊约定

普通 Full Stack API 读到 §9 已经完整；只有在设计 LLM / Agent 接口时再读这一节。下面给的是可迁移的设计原则，不代表任何一家供应商的当前字段或价格，接入时仍要核对目标 API 的最新契约。

### a) 流式（streaming）：SSE 事件序列（含工具调用）

```http
POST /v1/messages
{"model":"model-x","stream":true,"tools":[...],"messages":[...]}

HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-store
X-Accel-Buffering: no          ← 关掉 nginx 响应缓冲，否则整段流被攒到最后一次性吐出
request-id: req_01J8XQ7YB3F0KJ2M

event: message_start
data: {"type":"message_start","message":{"id":"msg_01J8...","model":"model-x","role":"assistant","content":[],"usage":{"input_tokens":21,"output_tokens":0}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"我先查一下这个租户的订单。"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: content_block_start
data: {"type":"content_block_start","index":1,"content_block":{"type":"tool_use","id":"toolu_01A9...","name":"search_orders","input":{}}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"{\"tenant"}}

: keepalive                     ← SSE 注释行；间隔要短于链路中最小空闲超时

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"_id\":\"ws_42\",\"limit\":20}"}}

event: content_block_stop
data: {"type":"content_block_stop","index":1}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"tool_use"},"usage":{"output_tokens":87}}

event: message_stop
data: {"type":"message_stop"}
```

**这个序列里藏着六条设计原则：**

1. **每个事件带稳定类型和定位字段。** 示例用 `index` 定位内容块；如果协议承诺同一连接严格有序，数组尾插也能工作，但显式 ID/index 更利于并行块、断点恢复和校验。
2. **工具参数可能是分片 JSON。** 只有收到明确完成事件并通过完整 schema 校验后才能触发副作用；半截字符串不是合法业务输入。
3. **用量可以增量展示，权威值在服务端结算。** 开始时能知道输入侧，结束后才知道输出侧；字段按你的计费维度定义，不必照搬某家供应商。
4. **HTTP 200 一旦刷出就不能再改状态码。** 中途失败只能发带内错误（in-band error）事件（`event: error` + `{"type":"error","error":{...}}`）。推论：**限流、鉴权、参数校验必须在开流之前完成**；开流后才发现超配额，你只能发一个客户端多半没处理的事件。
5. **长静默阶段要有心跳，但间隔不是常数。** 先确认客户端、代理、网关和服务端中最短的空闲超时，再把心跳设得更短，并关闭会吞掉流的响应缓冲。
6. **断线语义必须显式选择。** 简单实现可声明流不可恢复，客户端用同一个操作幂等键重新发起；高成本任务可持久化事件序号并支持恢复，或直接采用 §8 的任务资源。不能既不恢复又让客户端不知道重发是否会重复计费/执行工具。

### b) 工具调用的形状对称性

```jsonc
// 助手轮：可能同时发出多个 tool_use
{"role":"assistant","content":[
  {"type":"tool_use","id":"toolu_A","name":"search_orders","input":{...}},
  {"type":"tool_use","id":"toolu_B","name":"get_customer","input":{...}}]}
// 下一轮：必须为【每一个】 tool_use 回一个 tool_result，用 tool_use_id 对应
{"role":"user","content":[
  {"type":"tool_result","tool_use_id":"toolu_A","content":"[{\"id\":\"ord_1\"}]"},
  {"type":"tool_result","tool_use_id":"toolu_B","is_error":true,"content":"upstream timeout"}]}
```

**部分失败（partial failure）是这里的核心设计点。** 在示例契约中，每个 `tool_use` 都必须有对应的 `tool_result`；一个失败时仍返回带 `is_error: true` 的结果，而不是让整轮丢失另一个成功结果。不同供应商的角色和字段会不同，但“每个调用 ID 都有终态”应保持。

要区分两层错误：**工具正常返回的业务失败**可以成为模型输入，让它改参数或换工具；鉴权失败、协议损坏、网关不可用仍是传输/控制层错误。不要把所有失败都包装成 200，也不要用一个 5xx 抹掉已经完成的并行结果。

### c) 取消（`POST /v1/responses/{id}/cancel`）

三件事要写清：① 客户端断开是否自动取消，HTTP 的 cancellation signal 如何传播到队列/调度器；② 取消前已消耗的资源如何计量和收费；③ 取消是同步确认还是 `202 + cancelling`，以及“请求取消到达时任务已经完成”返回什么。对需要断线继续的批任务，客户端断开反而不应自动取消。

### d) 用量回传与可复现（reproducibility）

```json
"usage": {"input_tokens":21, "cached_input_tokens":0, "output_tokens":87}
```

用量结构要能区分**价格或配额不同**的维度，例如普通输入、缓存输入、输出、工具或媒体处理；具体字段和费率属于你的版本化计费契约。若这些维度价格相同，也不必为了模仿供应商而制造字段。

**关键工程要求**：用量必须在**服务端**记账（metering），且在客户端中途断开时也要记。只依赖流里最后一个 `message_delta` 计费，会在断流时系统性少收钱 —— 这个方向永远对供应商不利。见 [billing-and-metering](../03-saas-platform/02-billing-and-metering.md)。

**`request-id` 应出现在所有响应上**（包括错误与流式响应）。为了调查和回放，通常还要记录 `request_id, 实际模型版本, prompt 模板版本, tool schema hash, 采样参数, 上下文版本/哈希, 检索文档 ID`；敏感 prompt 和工具结果要遵守脱敏、访问控制与保留策略。

⚠️ **不要在无法控制模型版本和运行时的情况下承诺“相同输入一定产生逐字相同输出”。** 即使采样参数固定，模型升级、浮点计算、批处理与路由也可能改变结果。若业务需要审计，应保存被批准的原始响应或不可篡改哈希并提供受控回放；这和“重新推理得到近似结果”是两种能力。

### e) 把性能旋钮做进契约的代价

把缓存断点、TTL、缓存作用域或路由提示做成公开字段，能给客户端更多控制，也会把当前实现细节变成长期契约。上线前先问：如果以后换模型、缓存算法或网关，这个字段还能保持同样语义吗？下面三条原则通常可迁移：

> **① 网关需要而且可信的路由元数据，应放在可验证的头或路径中**，避免每层都解析大 body；敏感值仍要鉴权，不能因为在 header 就信任。
> **② 缓存作用域和失效规则由服务端契约声明**，并把租户、权限、模型/模板版本纳入 key。
> **③ 参与内容寻址或前缀缓存的列表要确定性序列化**；若顺序没有语义，先规范化再哈希，而不是把偶然顺序永久变成业务契约。

---

## 11. 什么时候不要这么做（反模式 anti-pattern 清单）

| 反模式 | 为什么错 / 正确做法 |
|---|---|
| **HTTP 200 + `{"error": ...}`** | 所有中间件（LB 健康检查、APM、重试库）都靠状态码判断成败。**例外只有两个**：流式（头已刷出）与任务状态查询（§8） |
| **没有兼容策略就复制一套 v2** | 会长期维护两套路由且行为漂移。先判断能否不破坏式演进；确需大版本时给出迁移、遥测与关闭计划 |
| **把 DB 列名直接映射成 API 字段** | schema 迁移变成 API 破坏性变更。中间加 DTO 显式解耦 |
| **每个列表都同步算精确 `total_count`** | 大范围计数可能比取一页贵得多；按产品需求选择估算、缓存或异步计数。分页链接应由服务端生成，或把拼接规则写成稳定契约 |
| **复杂公开 API 只给 schema、不给可执行示例/SDK** | 重试、幂等、分页和流解析容易被各客户端重复写错；至少提供契约测试和参考实现，主流语言再按使用量提供 SDK |
| **让可能超过端到端 deadline 的工作留在同步请求** | 任一中间层都可能先断开，客户端无法判断结果。改用 202 + 状态资源，或明确延长并协调全链路 deadline |
| **webhook 端点不验签** | 任何人都能伪造事件。HMAC + 时间戳容差 + 恒定时间比较 |
| **枚举穷举 switch** | 服务端加一个值就崩。未知值走 `unknown` 分支 + canary 枚举提前暴露 |
| **公开 GraphQL 不限复杂度** | 一个嵌套查询打爆你。深度/复杂度上限 + persisted query 白名单（allowlist；persisted query = 客户端只能提交服务端事先登记过的查询、用哈希来引用，于是你重新知道了"每个查询要花多少钱"） |
| **对 LLM API 承诺 deterministic output** | 物理上做不到。承诺"可回放"，不承诺"可复算" |

**最后一条，也是最重要的一条**：

> 不要为"未来可能的需求"设计通用性。一个能表达任意查询的 `POST /v1/query` 端点在设计评审上很好看，在生产上会变成：无法限流（不知道代价）、无法缓存（不知道键）、无法弃用（不知道谁在用什么）、无法演进（任何行为改变都可能破坏某个组合）。**API 的表达力上限，应该等于你能承诺 SLO 的上限。**

---

## 这一章的三句话

1. **API 的兼容性不由规范定义，由最粗心的那个客户端定义。** "新增一个枚举值"在规范上是纯加法，在对方的穷举 `switch` 里是一次崩溃 —— 所以判断破坏性变更时，问的不是"我删东西了吗"，而是"有没有一种合理的客户端写法会因此挂掉"。
2. **幂等是两端共同完成的状态机。** 客户端必须在同一操作的重试中复用 key；服务端必须绑定作用域和请求指纹，并把“已完成、明确失败、结果未知”分开处理。随机 UUID 不是问题，每次重试换 UUID 才是。
3. **弃用靠遥测、沟通、演练和明确退出条件共同完成。** 看独立客户端与业务周期，而不只看 QPS；brownout 可以暴露隐藏调用者，但它会制造真实故障，只能在风险可控且可回退时使用。

---

## 面试官会追问

1. offset 分页什么时候开始撞墙？不要背固定行数：说出执行计划、索引、offset 深度和并发写入如何分别影响性能与正确性。
2. cursor 里你会编码什么？为什么要签名 —— 是为了保密吗？
3. keyset 分页要求排序是全序，排序列有重复值会发生什么？SQL 怎么写才能吃到索引？
4. 客户端带着同一个 `Idempotency-Key` 但请求体不一样，你返回什么？为什么不直接重放上次结果？幂等记录和业务数据不在一个库时怎么办？
5. 429 和 503 对客户端的行为要求有什么不同？为什么要分开？
6. 流式响应已经发出 HTTP 200，中途模型服务过载了，你怎么告诉客户端？这对限流的位置有什么约束？
7. 你要下线 v1，遥测要拆到什么粒度？关门判据是流量还是别的？brownout 怎么排？
8. Webhook 载荷里放完整状态还是只放 ID？LLM API 的 usage 只在流的最后一个事件回传、客户端中途断了，你的账单会怎么样？

---

**按训练路径阅读** → 回 [START-HERE](../START-HERE.md) 按所选路径继续；页尾链接只表示本目录或专章的顺读顺序。

**目录顺读下一篇** → [05-data-platform.md](05-data-platform.md)
