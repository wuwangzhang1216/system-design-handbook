# 04 · API 设计与演进

> API 是你唯一无法回滚（roll back）的东西。代码可以重写，数据库可以迁移，但已经发出去的响应形状永远在别人的生产代码里。
> 所以 API 设计的核心不是"怎么设计得优雅"，而是"怎么设计得可以在不打断任何人的前提下改变"。

---

## 1. 协议选型：三分钟决策

| 维度 | REST / JSON | gRPC | GraphQL |
|---|---|---|---|
| 典型序列化开销（serialization overhead） | 0.1–1 ms | **0.02–0.2 ms**（protobuf + HTTP/2） | 与 REST 同级 + 规划开销 |
| 演进机制 | 靠约定（加字段安全） | **字段号 + reserved**，机制内建 | schema deprecation + 字段级用量统计 |
| 可缓存性（cacheability）/ 调试 | **HTTP 缓存全套可用** / curl | 基本无（都是 POST）/ grpcurl | 差 / playground |
| 适用 | 公开 API、合作伙伴 | **内部东西向（east-west traffic）**、移动端长连接 | BFF、字段需求高度可变的前端 |

**默认选择**：公开/合作伙伴 API → REST + OpenAPI；内部东西向 → gRPC；前端聚合 → BFF 或 GraphQL；LLM 推理 → REST + SSE（§10）；异步见 [messaging](../01-building-blocks/03-messaging-and-streams.md)。

**反直觉的一条**：GraphQL 不是"更好的 REST"，它是**把查询规划的责任从服务端转移给了客户端**。代价是 N+1 跑到 resolver 层（必须上 DataLoader）、限流单位从"请求数"变成"查询复杂度（query complexity）"（你要先写复杂度计算器，典型上限深度 ≤ 10 / 复杂度分 ≤ 1000）、HTTP 缓存基本失效。只有当「前端形态高度可变 + 后端数据源多 + 前端团队比后端团队大」三条同时成立时才值。

---

## 2. 资源建模与 URL：实际规则

```
✅ GET  /v1/workspaces/ws_42/api_keys?status=active&limit=20
✅ POST /v1/exports/exp_01J8.../cancel      ← 动作作为子资源，不要硬套 CRUD
❌ POST /v1/getUserById                      ← RPC 伪装成 REST
❌ GET  /v1/users/42/orders/7/items/3/tags   ← 超过两层嵌套
❌ GET  /v1/users?id=42                      ← 单资源用查询串
```

**七条硬规则**：

1. **嵌套不超过两层**。再深就用顶层资源 + 过滤：`/items?order_id=7`。
2. **ID 用带前缀的不透明字符串（opaque ID）**：`msg_01J8XQ...`（ULID/UUIDv7 + 类型前缀）。自增整数（auto-increment integer）会泄露业务量（竞对注册两个账号就能算出你的日增），也让分库分表（sharding）变难；前缀让误传的参数在校验层就被拒。
3. **动作用子资源 POST**，不要发明 `PUT /orders/7?action=cancel`。
4. **标量约定**：时间一律 RFC 3339 UTC 带毫秒（`2026-07-30T11:02:03.412Z`）；金额用最小单位整数（minor units）+ 币种（`{"amount":1250,"currency":"USD"}`）或字符串 decimal，**永远不要用浮点表示钱**。
5. **枚举（enum）全小写下划线**，并在契约（contract）里写死"客户端 MUST 把未知枚举当 `unknown` 处理"（见 §7 枚举炸弹）。
6. **列表响应必须是对象不是数组**：`{"data":[...],"next_cursor":"..."}`。裸数组让你以后加不了任何元数据。
7. **PATCH 语义必须选一个写进文档**：JSON Merge Patch（[RFC 7386](https://www.rfc-editor.org/rfc/rfc7386)，简单但无法操作数组元素）、JSON Patch（[RFC 6902](https://www.rfc-editor.org/rfc/rfc6902)，表达力强但客户端难写）、field_mask（Google AIP-134）。**选不出来就用 field_mask** —— 它是唯一能同时表达"设为 null"和"不要动"的方案。

---

## 3. 错误契约：RFC 9457 + 可重试性（retryability）标注

[RFC 9457 Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)（2023，废弃 RFC 7807）定义了 `application/problem+json`。核心成员：`type`（问题类型 URI）、`title`、`status`、`detail`、`instance`。

**但直接用 RFC 9457 是不够的。** 生产上必须加两个扩展成员：

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
| 404 不存在 | ❌ | 例外：**读己之写（read-your-writes）场景可重试 1 次**（复制延迟 replication lag） |
| 409 冲突 | ⚠️ 分情况 | 乐观锁（optimistic locking）版本冲突 → 重读后重试；幂等键（idempotency key）冲突 → **绝不重试** |
| 408 / 499 超时或断开 | ✅ | **必须带幂等键**，否则可能重复扣款 |
| 429 超配额 | ✅ | 必须遵守 `Retry-After` |
| 500 内部错误 | ✅ | 必须带幂等键 |
| 502 / 503 / 504 上游或容量 | ✅ | 退避 + 抖动（backoff + jitter） |
| 529 服务过载（Anthropic 用此码） | ✅ | 与 429 的区别：**不是你的错，是我们容量不够** |

> **面试金句**：
> "429 和 503 必须分开：429 是'你超了你自己的配额'，客户端应该降速并遵守 Retry-After；503/529 是'我们容量不够'，客户端应该退避但不需要改自己的速率上限。把两者混成一个码，会导致大客户在我们扩容之后仍然长期自我限速。"

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
| **offset/limit** | ~2 ms | **~800 ms+**（要扫描并丢弃 200,000 行） | ❌ 漏行 + 重行 | ✅ | 后台管理页、总量 < 1 万 |
| **keyset（seek）** | ~2 ms | **~2 ms** | ✅ 稳定 | ❌ | 内部 API、已知排序 |
| **opaque cursor** | ~2 ms | ~2 ms | ✅ 稳定 | ❌ | **公开 API 的唯一正确答案** |

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

⚠️ **常见错误**：把行值比较手写成 `created_at < $3 OR (created_at = $3 AND id < $4)`。逻辑等价，但很多优化器无法把它转成单次索引 seek，会退化成全索引扫描。用元组比较语法。

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

**不要返回 `total_count`。** 千万行表上带同样 WHERE 的 `COUNT(*)` 是一次全索引扫描（full index scan，几百 ms 到几秒），而 99% 的客户端只是把它渲染成一个没人看的数字。真要给就给 `total_count_estimate`（`pg_class.reltuples` 或采样）并标注是估算。

---

## 5. 幂等契约（idempotency contract）

`Idempotency-Key` 目前是 IETF httpapi 工作组的 [Internet-Draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)，**截至 2026 年中仍未成为 RFC**。但 Stripe / Adyen / Square 等的实现已经收敛成同一套语义，可以直接照做。

### 服务端状态机

```
收到 POST + Idempotency-Key: ik_7f3a
fingerprint = SHA256(method ‖ path ‖ tenant_id ‖ canonical(body))
在 (tenant_id, key) 上做条件插入：
  ├─ 插入成功 → in_flight，执行业务逻辑
  │     ├─ 成功    → 【同一个事务里】写业务数据 + 更新记录为 done(status, headers, body)
  │     └─ 失败5xx → 删除记录（让客户端能重试）；4xx 则记 done 并原样重放
  ├─ 已存在 done & fingerprint 相同 → 重放原响应 + `Idempotent-Replayed: true` + 原 request-id
  ├─ 已存在 done & fingerprint 不同 → 422 `idempotency.key_reused_with_different_payload`
  └─ 已存在 in_flight              → 409 `idempotency.request_in_progress` + Retry-After: 1
```

**四条容易踩的坑：**

1. **幂等记录必须与业务写在同一个事务里提交。** 分两步的话，在"业务已提交、幂等记录未提交"的窗口崩溃 → 重试会重复执行。业务在另一个库就必须用 Outbox（见 [messaging](../01-building-blocks/03-messaging-and-streams.md)）。
2. **保留窗口（retention window）必须 ≥ 客户端最长重试地平线（retry horizon）。** Stripe 是 24 小时。若客户端有"第二天人工补跑"的流程，24 小时就不够 —— 补跑会产生第二笔。窗口是你的**存储成本项**（每条含响应体 1–10 KB；100 万次/天 × 7 天 ≈ 7–70 GB）。
3. **最致命的失效模式（failure mode）：客户端每次重试都生成新 UUID。** 幂等完全不生效，且没有任何报错。规则：**幂等键必须由业务语义派生并在第一次尝试之前持久化**（`ik_{tenant}_{order_id}_{attempt_group}`），而不是在重试循环里 `uuid4()`。这件事应该由 SDK 替客户端做。
4. **幂等 ≠ exactly-once。** 它保证"同一个 key 不被执行两次"，不保证"这个操作只被请求了一次"。

**适用范围**：GET/HEAD/PUT/DELETE 天然幂等（PUT 需要 `If-Match` 做乐观锁；DELETE 第二次返回 404 还是 204 要写死）；**POST 与 PATCH 必须支持 `Idempotency-Key`**。

---

## 6. 限流（rate limiting）：响应头与客户端契约

IETF 的 [RateLimit header fields 草案](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/) 用 [RFC 9651 结构化字段](https://www.rfc-editor.org/rfc/rfc9651.html)定义了两个头（`q`=配额、`w`=窗口秒、`r`=剩余、`t`=距重置秒）：

```http
RateLimit-Policy: "requests";q=1000;w=60, "tokens";q=400000;w=60
RateLimit:        "requests";r=873;t=37,  "tokens";r=0;t=42
```

**该草案截至 2026 年中仍未成为 RFC**，所以现实做法是**两套都发**：草案头给未来，事实标准（de facto standard）的 `X-RateLimit-Limit / -Remaining / -Reset` 给存量客户端。

**多维限额是 LLM API 的常态**（请求数、输入 token、输出 token、并发数四个桶各自独立）。三条要求：所有维度都要在响应头里暴露（否则客户端只能靠撞墙学习）；`Retry-After` 只反映当前绑定的那个维度；**problem 的 `code` 必须指出是哪个桶满了** —— `rate_limit.output_tokens_per_minute` 的应对是减小 `max_tokens`，`rate_limit.requests_per_minute` 的应对是减小并发，完全不同。

**`Retry-After` 用秒数不用 HTTP-date**（日期形式要求两边时钟一致，而客户端时钟偏移几十秒是常态）。

**客户端侧契约（写进 SDK，不要指望用户自己写）**：

```python
delay = min(cap, base * 2**attempt) * random.uniform(0.5, 1.0)   # 全抖动
delay = max(delay, retry_after_from_header)                       # 服务端说了算
# 总重试预算：retries / requests < 10%，超了就熔断（见 05-reliability/03）
```

---

## 7. 版本化（versioning）：策略与迁移剧本

### 五种策略

| 策略 | 优点 | 代价 |
|---|---|---|
| **URL 路径** `/v1/` | 直观，路由/缓存友好 | 粒度太粗；v1→v2 是全量迁移，**结果是你永远卡在 v1** |
| **媒体类型** `Accept: application/vnd.acme.v2+json` | 资源级粒度 | 工具链差；CDN 要 `Vary: Accept`，缓存碎片化（cache fragmentation） |
| **日期版本头** `anthropic-version: 2023-06-01`、`Stripe-Version: 2025-03-31.basil` | **细粒度、可按账号 pin、能同时活几十个版本** | 服务端要维护版本转换链 |
| **不破坏式演进（non-breaking evolution）**（Google AIP / protobuf） | 零迁移成本 | 依赖客户端"忽略未知字段"；枚举扩展是隐形炸弹 |
| **beta 头** `anthropic-beta: context-management-2025-06-27` | 新能力不污染稳定版，可随时撤 | 客户端会把 beta 用到生产然后向你要 SLA |

**我的主张**：URL 里放 `/v1` 作为"大结构版本"（几乎永不变），真正的演进走**日期版本头**。Stripe 的做法值得抄（见 [Stripe 的 API 版本化工程文章](https://stripe.com/blog/api-versioning)）：服务端只有**一份**当前的内部表示，每个历史版本是一个小的双向转换器（version changer），响应从内部表示出发按时间倒序逐个应用转换器直到目标版本。于是加一个新版本 = 写几十行转换器 + 它的测试，内部代码只关心最新形状，版本数可以长到几十个而不失控。代价是**每个转换器都是永久的技术债**，且转换器的组合必须有测试覆盖（N 个版本 = N 条链路）。

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

没有遥测，剧本就是猜。需要的粒度不是"/v1 的 QPS"，而是 `(client_id, api_version, endpoint, 实际读到的字段, 最近使用时间)`。字段级用量对 GraphQL 和 protobuf 是免费的（选择集 / presence bit）；JSON REST 要在序列化层打点，只对**被渲染进响应且非默认值**的字段计数。

**判据不是流量，是独立客户端数。** 90% 的 v1 流量可能来自某个已迁移完的客户遗留的健康检查脚本。关门条件是"过去 30 天使用过的 `client_id` 数 ≤ N，且这 N 个都已单独沟通"。

```
T-180d  公告 + 迁移指南 + 新版本 GA（新版本必须先 GA，不能"边迁边发"）+ 旧版本响应加弃用头
T-150d  遥测拆到 (client_id, endpoint, field) 粒度，建看板
T-120d  逐客户外呼；Top-20 客户配专人；其余走邮件 + 控制台横幅
T-60d   brownout #1：每天 UTC 14:00–14:05（5 分钟）返回 410
T-30d   brownout #2：每小时的前 5 分钟返回 410
T-14d   最后通牒；只对已申请且带到期日的白名单豁免
T-0     关闭，返回 410 Gone + problem type 指向迁移文档
T+90d   移除代码与转换器
```

**brownout 是整个剧本里唯一真正有效的一步。** 邮件会被忽略，弃用头会被忽略，但"每天有 5 分钟报错"会直接进对方的告警系统。选低峰时段、公告精确到分钟、brownout 期间的 problem body 里必须有迁移文档链接。

弃用头用 [RFC 9745](https://www.rfc-editor.org/rfc/rfc9745.html)（Deprecation，2025）+ [RFC 8594](https://www.rfc-editor.org/rfc/rfc8594.html)（Sunset）：

```http
Deprecation: @1767225600
Sunset: Sat, 31 Jan 2026 23:59:59 GMT
Link: <https://docs.acme.dev/migrate/v1-to-v2>; rel="deprecation"; type="text/html"
```

---

## 8. 长任务（long-running operation）：202 + 状态资源 + 通知

任何 p99 超过 ~10 秒的操作都不该是同步请求。HTTP 中间件（ALB 默认空闲超时 idle timeout 60 s、Cloudflare ~100 s）会在你毫无察觉的情况下切断连接。

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

1. **状态资源是一等资源（first-class resource）**：有自己的 URL、能被列举、能被鉴权、有明确保留期（终态 24 h / 7 d，写进文档）。不要暴露内部队列 ID。
2. **任务失败时 HTTP 状态仍是 200**，problem+json 放在 body 的 `error` 字段里 —— 因为"查询状态"这个操作本身成功了。**这是这类 API 最常见的设计错误**：用 500 表达"任务失败"会让所有客户端的重试逻辑误触发。
3. **`Retry-After` 用来建议轮询间隔**，负载高时服务端可以调大。这是你唯一能控制轮询风暴（polling storm / thundering herd）的旋钮。
4. **三种通知方式全都要有**：webhook 是主路径，轮询是兜底（webhook 端点挂了也能拿到结果），SSE 给 UI。只做一种都会被投诉。
5. **`/cancel` 必须幂等**，且要区分 `cancelling` 与 `cancelled`。取消是异步的。

**注意**：MCP 在 [2026-07-28 规范](https://modelcontextprotocol.io/specification/2026-07-28/changelog)里把长任务从阻塞式 `tasks/result` 改成轮询 `tasks/get`，并把 Tasks 移出核心变成扩展。方向和上面一致：**长耗时操作不要押在长连接上**。

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

**四条：** ① **在 raw bytes 上验签，在 JSON 解析之前**（先 parse 再 re-serialize 会因 key 顺序/空白/Unicode 转义不同而永远验不过）；② 时间戳容差 **±5 分钟**，超出直接拒（防重放 replay protection）；③ **签名头允许多个值**（空格分隔）—— 这就是密钥轮换（key rotation）方案，新旧密钥各签一份、重叠 24–72 小时，没有它轮换密钥就等于制造一次事故；④ 用恒定时间比较（constant-time comparison，`hmac.compare_digest`）。

### 重试、乱序与载荷形态

典型退避表（backoff schedule，Standard Webhooks / Svix 口径）：`5s → 5min → 30min → 2h → 5h → 10h → 10h`，总窗口约 24 小时；连续失败 N 天后禁用端点并告警。

**顺序永远不保证** —— 重试会让 `updated` 排在 `created` 之前。消费者必须按 `webhook-id` 幂等（去重表 dedup table 保留期 ≥ 48 h），且载荷里要有单调递增（monotonic）的 `resource_version` 让消费者丢弃旧事件。

**瘦载荷（thin payload）vs 胖载荷（fat payload）是这里唯一真正的架构取舍：**

| | 胖载荷（带完整状态） | 瘦载荷（只带 ID，消费者回读） |
|---|---|---|
| 消费者延迟 / 陈旧风险（staleness） | 低 / **高**（乱序时旧状态覆盖新状态） | 高（多一次回读）/ 无 |
| 你的读流量 / 泄露面 | 无额外 / 大（端点被劫持即泄露数据） | **+1 次/事件** / 小（回读要鉴权） |

金融/合规选瘦载荷，其余选胖载荷 + 版本号。**两个必做项**：
- **SSRF 防护**：用户注册的端点 = 允许用户让你的服务器向任意地址发 POST。解析后校验 IP，拒绝 RFC 1918 / 169.254.169.254 / ::1（防 DNS rebinding：解析一次后直连该 IP）；禁止跟随重定向；走独立出口代理，超时 3–5 秒。
- **投递日志（delivery log）是一等资源**：`GET /v1/webhook_deliveries?status=failed` + `POST /v1/webhook_deliveries/{id}/replay`。没有它，每次客户说"我没收到"都要你去查日志。这是投入产出比最高的一个端点。

---

## 10. AI API 的特殊约定

这一节是 2024–2026 才成型的东西，也是现在最容易被问到的部分。

### a) 流式（streaming）：SSE 事件序列（含工具调用）

```http
POST /v1/messages
{"model":"claude-opus-5","stream":true,"tools":[...],"messages":[...]}

HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-store
X-Accel-Buffering: no          ← 关掉 nginx 响应缓冲，否则整段流被攒到最后一次性吐出
request-id: req_01J8XQ7YB3F0KJ2M

event: message_start
data: {"type":"message_start","message":{"id":"msg_01J8...","model":"claude-opus-5","role":"assistant","content":[],"usage":{"input_tokens":21,"cache_creation_input_tokens":0,"cache_read_input_tokens":13052,"output_tokens":1}}}

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

: keepalive                     ← SSE 注释行，每 15 s 一次，防中间层按空闲断连

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

1. **内容块按 `index` 寻址，不按到达顺序。** 客户端维护 `blocks[index]` 而不是数组尾插 —— 这让服务端未来可以并行发多个块。
2. **工具参数是分片的 JSON 字符串**（`input_json_delta.partial_json`），只有 `content_block_stop` 之后才是合法 JSON。**不要对半截 JSON 做增量解析然后触发副作用** —— 你会渲染出 `{"tenant`，更糟的是可能用错误参数提前触发工具执行。
3. **用量分两处回传**：`message_start` 给输入侧（含缓存读/写），`message_delta` 给输出侧 —— 因为开始时还不知道会输出多少。
4. **HTTP 200 一旦刷出就不能再改状态码。** 中途失败只能发带内错误（in-band error）事件（`event: error` + `{"type":"error","error":{...}}`）。推论：**限流、鉴权、参数校验必须在开流之前完成**；开流后才发现超配额，你只能发一个客户端多半没处理的事件。
5. **`: keepalive` 注释行是必须的，不是可选优化。** ALB/Cloudflare/企业代理的空闲超时在 60–100 s，而长 prefill 有几秒到几十秒完全无输出。
6. **断流即丢。** 除非实现了 `Last-Event-ID` 恢复，否则连接断了整个请求作废。MCP 在 2026-07-28 规范里**主动删除了** SSE 断线续传（stream resumption），要求客户端用新 request id 重发 —— 这是明确的"不要在流协议里做可靠性"信号。可靠性属于长任务 API（§8），不属于流。

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

**部分失败（partial failure）是这里的核心设计点。** 两个并行工具调用一个成功一个失败时：不要返回 HTTP 错误；不要省略失败那个的 `tool_result`（缺一个就是格式非法，直接 400）；用 `is_error: true` 把失败**作为数据**回传给模型。

推广到所有 Agent 相关 API：**在 Agent 循环里，错误是模型的输入，不是传输层的状态。** 你把工具错误变成 5xx，模型就失去了自我修正的机会（它本可以换个参数重试）。

### c) 取消（`POST /v1/responses/{id}/cancel`）

三件必须做的事：① **客户端断开必须真的传播到调度器** —— HTTP 层的 `context.Done()` 要一路传到推理引擎的 request slot，否则你在为没人读的 token 付 GPU 钱（中等规模网关上"已断开但仍在生成"的请求能占几个百分点的 decode 容量）；② **契约里写死"取消按已生成的 token 计费"**，算力已经花了，不写清楚就变成客服问题；③ **取消是异步的** —— 返回 202 + 状态转 `cancelling`，别假装同步。

### d) 用量回传与可复现（reproducibility）

```json
"usage": {"input_tokens":21, "cache_creation_input_tokens":0,
          "cache_read_input_tokens":13052, "output_tokens":87}
```

**四个字段缺一不可**：缓存读价约为标准输入价的 **10%**，缓存写约 **1.25×**（2026 年中量级，随时变动），不拆开就没法算成本。

**关键工程要求**：用量必须在**服务端**记账（metering），且在客户端中途断开时也要记。只依赖流里最后一个 `message_delta` 计费，会在断流时系统性少收钱 —— 这个方向永远对供应商不利。见 [billing-and-metering](../03-saas-platform/02-billing-and-metering.md)。

**`request-id` 必须出现在所有响应上**（包括错误与流式响应，在 header 里、body 之前）。可复现的最小记录集：`request_id, model_version, prompt_template_version, tool_schema_hash, temperature, top_p, seed, 上下文 token 数, 检索到的文档 ID 列表`。

⚠️ **不要承诺"相同输入 → 相同输出"**（浮点非结合性 + 批处理组成变化 + MoE 路由，即使 temperature=0 也做不到）。承诺"**相同 request_id → 相同的已记录响应**"，即从日志回放（replay）而非重新推理。这个区别在合规场景要写进合同。

### e) 把性能旋钮做进契约的代价

Anthropic 的 `cache_control: {"type":"ephemeral"}` 把"在哪里切缓存断点"变成了 API 字段。收益是客户端能精确控制成本；代价是"**最多 4 个 breakpoint、每个只回看 20 个 content block**"这类实现细节成了公开契约，以后想换实现就是破坏性变更。

MCP 2026-07-28 规范把同一类东西做成了服务端声明：`tools/list` 等结果必须带 `ttlMs` 与 `cacheScope`（`public`/`private`），POST 必须带 `Mcp-Method` / `Mcp-Name` 头**让网关不解 body 就能路由**，且建议 `tools/list` 返回确定性顺序（顺序一变，下游的 prompt cache 全废）。三条可迁移的原则：

> **① 路由信息放头里，不放 body 里** —— 否则每一层代理都要解析并信任 body。
> **② 缓存性由服务端声明，不由客户端猜** —— 客户端猜错的代价是脏数据。
> **③ 列表的顺序是契约的一部分** —— 只要下游有任何缓存，非确定性排序就是持续的缓存击穿。

---

## 11. 什么时候不要这么做（反模式 anti-pattern 清单）

| 反模式 | 为什么错 / 正确做法 |
|---|---|
| **HTTP 200 + `{"error": ...}`** | 所有中间件（LB 健康检查、APM、重试库）都靠状态码判断成败。**例外只有两个**：流式（头已刷出）与任务状态查询（§8） |
| **一上来就设计 v2** | v2 永远不会完成，你会同时维护两套。用日期版本 + 不破坏式演进 |
| **把 DB 列名直接映射成 API 字段** | schema 迁移变成 API 破坏性变更。中间加 DTO 显式解耦 |
| **返回 `total_count` / 让客户端拼分页 URL** | 前者是全表扫描且没人看；后者让你以后改不了参数名 |
| **只提供 OpenAPI，不提供 SDK** | 重试退避、幂等键生成、分页迭代、流解析，每个客户端都会各写错一遍。**SDK 是契约的一部分** |
| **在同步请求里做超过 10 s 的事** | 会被 LB 静默切断，客户端无法区分"超时"和"失败"。用 202 + 状态资源 |
| **webhook 端点不验签** | 任何人都能伪造事件。HMAC + 时间戳容差 + 恒定时间比较 |
| **枚举穷举 switch** | 服务端加一个值就崩。未知值走 `unknown` 分支 + canary 枚举提前暴露 |
| **公开 GraphQL 不限复杂度** | 一个嵌套查询打爆你。深度/复杂度上限 + persisted query 白名单（allowlist） |
| **对 LLM API 承诺 deterministic output** | 物理上做不到。承诺"可回放"，不承诺"可复算" |

**最后一条，也是最重要的一条**：

> 不要为"未来可能的需求"设计通用性。一个能表达任意查询的 `POST /v1/query` 端点在设计评审上很好看，在生产上会变成：无法限流（不知道代价）、无法缓存（不知道键）、无法弃用（不知道谁在用什么）、无法演进（任何行为改变都可能破坏某个组合）。**API 的表达力上限，应该等于你能承诺 SLO 的上限。**

---

## 面试官会追问

1. offset 分页在什么规模下开始撞墙？给我具体数字和两种失效模式（性能与正确性各一）。
2. cursor 里你会编码什么？为什么要签名 —— 是为了保密吗？
3. keyset 分页要求排序是全序，排序列有重复值会发生什么？SQL 怎么写才能吃到索引？
4. 客户端带着同一个 `Idempotency-Key` 但请求体不一样，你返回什么？为什么不直接重放上次结果？幂等记录和业务数据不在一个库时怎么办？
5. 429 和 503 对客户端的行为要求有什么不同？为什么要分开？
6. 流式响应已经发出 HTTP 200，中途模型服务过载了，你怎么告诉客户端？这对限流的位置有什么约束？
7. 你要下线 v1，遥测要拆到什么粒度？关门判据是流量还是别的？brownout 怎么排？
8. Webhook 载荷里放完整状态还是只放 ID？LLM API 的 usage 只在流的最后一个事件回传、客户端中途断了，你的账单会怎么样？

---

**下一篇** → [05-data-platform.md](05-data-platform.md)
