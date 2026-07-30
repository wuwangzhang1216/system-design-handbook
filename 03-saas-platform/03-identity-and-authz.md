# 03 · 身份与授权

> 认证是"你是谁"，授权是"你能干什么"。前者已经被标准化到几乎没有设计空间，后者是每个 SaaS 都会自己重造一遍、并且重造错的那一部分。
> 到了 Agent 时代，还多了第三个问题：**"你代表谁"**。

---

## 1. 三个必须分清的词

| 概念 | 回答的问题 | 谁负责 | 出错的后果 |
|---|---|---|---|
| **Authentication（认证 / AuthN）** | 你是谁 | IdP（Okta/Entra/Auth0/自建） | 账号被冒用 |
| **Authorization（授权 / AuthZ）** | 你能对这个资源做什么 | **你自己的授权服务** | 越权访问 = 数据泄露 |
| **Delegation（委派）** | 谁在代表谁行事 | Token 交换层 | Agent 拿着用户全部权限乱跑 |

**第一条纪律**：认证外包，授权自建。
认证是标准协议（OIDC），自研没有任何收益且极易出错；授权是业务语义，没有任何现成产品知道你的 `project` 和 `workspace` 是什么关系。

**第二条纪律**：**授权决策必须发生在服务端，且发生在数据访问路径上。**
前端隐藏按钮不是授权，网关按 URL 前缀放行不是授权（`/api/projects/{id}` 的 `{id}` 是用户可控的）。

---

## 2. OIDC：唯一你应该用的登录协议

[OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html) 是 OAuth 2.0 之上的认证层。2026 年的正确姿势是 **Authorization Code + PKCE**，其他 flow 基本都被 [RFC 9700（OAuth 2.0 Security BCP，2025-01）](https://www.rfc-editor.org/rfc/rfc9700.html) 判了死刑：implicit flow **MUST NOT** 用，password grant（ROPC）**MUST NOT** 用，PKCE 对所有客户端（含机密客户端）都要求。

```
浏览器                你的后端 (RP)                    IdP (OP)
  │  GET /login          │                              │
  ├─────────────────────>│ 生成 state + nonce + PKCE     │
  │                      │ verifier，存进 HttpOnly cookie │
  │ 302 → /authorize?client_id=…&redirect_uri=…&scope=openid profile
  │        &state=S&nonce=N&code_challenge=S256(verifier)&code_challenge_method=S256
  │<─────────────────────┤                              │
  ├────────────────────────────────────────────────────>│ 用户认证 + 同意
  │ 302 → /callback?code=C&state=S                      │
  │<────────────────────────────────────────────────────┤
  ├─────────────────────>│ ① state 必须匹配（防 CSRF）    │
  │                      ├── POST /token ──────────────>│ code=C + code_verifier
  │                      │   客户端认证用 private_key_jwt │ （不要用 client_secret_post）
  │                      │<── id_token + access_token + refresh_token
  │                      │ ② 校验 id_token：签名/iss/aud/exp/nonce
  │ Set-Cookie: sid=…    │ ③ 建立**你自己的会话**，不要把 id_token 当会话
  │<─────────────────────┤    HttpOnly; Secure; SameSite=Lax; __Host- 前缀
```

**三个高频错误**：
1. **把 `id_token` 存进 localStorage 当会话用**。ID token 是给 RP 消费的一次性认证凭证，不是 API 凭证，更不该被 JS 读到。后端 session cookie 才是对的。
2. **`state` 不校验或全局共享**。`state` 必须与浏览器会话绑定且一次性。
3. **`nonce` 不校验**。`nonce` 是防 ID token 重放的唯一手段。

---

## 3. JWT 的校验清单与真实漏洞

JWT 的问题不是格式，是**几乎每个库的默认配置都不安全**。

### 必查清单（缺一项就是漏洞）

| # | 校验项 | 不做的后果 |
|---|---|---|
| 1 | **算法白名单硬编码**（如只接受 `RS256`），**不信 header 里的 `alg`** | `alg: none` 绕过；RS256→HS256 混淆：拿公钥当 HMAC 密钥伪造 |
| 2 | `kid` 只作为**查表键**，白名单化 | `kid` 路径遍历（`../../dev/null`）/ SQL 注入 |
| 3 | **禁用 `jku` / `x5u`**（或严格 host 白名单） | 攻击者托管自己的 JWKS，签什么都过 |
| 4 | `aud` 必须等于本服务标识 | A 服务的 token 拿去打 B 服务 —— **confused deputy 的标准入口** |
| 5 | `iss` 必须匹配预期 IdP | 多租户 IdP 下别的租户的 token 被接受 |
| 6 | `exp` / `nbf`，时钟偏移 ≤ 60s | 过期 token 长期可用 |
| 7 | `typ` 校验：access token 应为 `at+jwt`（[RFC 9068](https://www.rfc-editor.org/rfc/rfc9068.html)） | ID token 被当 access token 使用 |
| 8 | JWKS 缓存 TTL 5–10 min；遇到未知 `kid` 触发**限流的**刷新 | 不刷新 → 轮换密钥时全站 401；不限流 → 伪造 `kid` 打爆 IdP |
| 9 | payload 不放敏感数据 | JWT 是 **base64 不是加密**，任何人可读 |

> ⚠️ 不要自己写 JWT 校验。用 IdP 官方 SDK 或成熟库，并**显式传入允许的算法列表**——库的"自动检测 alg"就是漏洞本身。

### 令牌生命周期与撤销

这是 JWT 最难的部分：**自包含 token 天然不可撤销**。

```
access token TTL = 你能容忍的"权限已撤销但仍可用"的窗口
```

| 手段 | 撤销延迟 | 代价 |
|---|---|---|
| **短 TTL（5–15 min）** | ≤ TTL | 刷新频率上升；对多数场景**这就是答案** |
| **jti 黑名单**（Redis，只存到 `exp`） | 秒级 | 每次校验多一次 Redis 读（~0.5ms）；存储 = 撤销量 × TTL |
| **不透明 token + introspection**（[RFC 7662](https://www.rfc-editor.org/rfc/rfc7662.html)） | 立即 | 每次校验一次网络调用；IdP 变成关键路径 |
| **会话版本号**：JWT 带 `sv`，与用户表的 `session_version` 比对 | 立即 | 需要一次用户表读（可缓存），撤销 = `session_version++` |

**Refresh token 必须做轮换 + 重放检测**：每次刷新签发新 refresh token 并作废旧的；如果旧 token 被再次使用 → 判定为泄露 → **撤销整个 token 家族**并强制重新登录。没有重放检测的轮换等于没做。

**合规口径**：SOC 2 审计里常见的要求是"离职/权限变更后 ≤15 分钟失效"。把这句话直接翻译成 access token TTL ≤ 15 min，比写一堆策略文档有用。

---

## 4. 企业里绕不开的两件事：SAML 与 SCIM

**SAML 2.0（2005 年的规范）在 2026 年仍然活着**，因为大型企业的 IdP 集成、采购流程和安全审计都是围绕它建的。现实结论：

- **新建 SaaS 只做 OIDC，但企业版必须补 SAML**，否则丢单。
- SAML 的实现风险显著高于 OIDC：**XML Signature Wrapping（XSW）** 攻击、`AudienceRestriction` 不校验、注释截断（`admin@a.com<!---->.evil.com`）等。**用成熟库，禁止手写 XML 解析。**
- 定价现实：SSO 通常被放进企业档（"SSO tax"）。这是商业决策不是技术决策，但它决定了你要不要在 v1 就做多 IdP 抽象层。

**SCIM（[RFC 7644](https://www.rfc-editor.org/rfc/rfc7644.html)）才是真正影响架构的那个。** 它是用户生命周期的推送通道：

```
IdP (Okta/Entra) ──SCIM POST /Users──> 你的 SCIM 端点 ──> 用户表
                 ──SCIM PATCH /Groups─>                 ──> 组织成员关系
                 ──SCIM DELETE ───────>                 ──> 停用（不是删除！）
```

**四个必须做对的点**：
1. **`DELETE` 语义是"停用"不是"删除"**。真删会破坏审计链和历史归属。用 `active: false`。
2. **`externalId` 是 IdP 侧主键，必须建唯一索引**。用 email 做匹配键会在改邮箱时炸。
3. **幂等**：IdP 会重放。`PUT /Users/{id}` 必须幂等，`POST` 要用 `externalId` 去重。
4. **Group → 你的组织模型的映射要显式配置**，不要猜。企业的 AD group 结构和你的 workspace 结构不同构，这是集成时最大的扯皮点。

**去配置（deprovisioning）的 SLA 是审计重点**：SCIM 收到停用 → 用户所有活跃 session/token 必须在 N 分钟内失效。如果你的 access token TTL 是 1 小时，你的实际去配置 SLA 就是 1 小时，不管你的文档怎么写。

---

## 5. 授权模型演进：RBAC → ABAC → ReBAC

| 模型 | 决策依据 | 表达力 | 撞墙点 |
|---|---|---|---|
| **RBAC** | 用户的角色 | 低 | 角色爆炸：`project_42_editor`、`org_7_billing_admin`……角色数 = O(资源数 × 动作数) |
| **ABAC** | 主体/资源/环境的属性 + 策略引擎（如 [Cedar](https://www.cedarpolicy.com/)、OPA/Rego） | 高 | **"列出我能访问的所有项目"无法回答** —— 只能遍历全部资源逐个求值 |
| **ReBAC**（Zanzibar 系） | 主体与资源之间的**关系图** | 高 | 需要独立的、有一致性语义的存储服务；建模有学习曲线 |

**判据**：
- 资源之间有**层级/继承/共享**语义（文件夹、组织→项目→文档、"分享给某个团队"）→ **ReBAC**。
- 决策依赖**运行时属性**（IP、时间、设备合规状态、数据分级）→ **ABAC**，通常和 ReBAC 并存：ReBAC 回答"关系上允许吗"，ABAC 回答"当前环境允许吗"。
- 只有几个固定角色、资源不共享 → **RBAC 够用，别过度设计**。

> **别急着上 Zanzibar。** 如果你的权限模型能在一张表里用 `(user_id, resource_id, role)` 表达，并且没有"分享"和"继承"，那 ReBAC 只会给你增加一个必须运维的、在关键路径上的新服务。

---

## 6. Zanzibar：关系元组与一次 check 的推导

[Google Zanzibar（USENIX ATC 2019）](https://www.usenix.org/conference/atc19/presentation/pang) 的量级：**> 2 万亿关系元组、> 1000 万 QPS、p95 < 10 ms、三年可用性 99.999%**。它证明了"把授权做成一个独立的、全球分布的图数据库"是可行的。

### 6.1 关系元组（relation tuple）

基本形式：

```
⟨object⟩#⟨relation⟩@⟨subject⟩
```

`subject` 可以是一个用户，也可以是**另一个对象的某个关系**（`team:platform#member`），这就是 ReBAC 的表达力来源。

### 6.2 命名空间配置（用 SpiceDB schema 语法）

```zed
definition user {}

definition organization {
    relation admin:  user
    relation member: user
    permission administrate = admin
}

definition team {
    relation member: user
}

definition project {
    relation org:    organization
    relation owner:  user
    relation editor: user | team#member
    relation viewer: user | team#member

    // "+" = 并集；"->" = 沿关系穿透到另一个对象的 permission
    permission edit = owner + editor + org->administrate
    permission view = viewer + edit
}
```

### 6.3 元组实例

```
organization:acme#admin@user:alice
team:platform#member@user:bob
project:atlas#org@organization:acme
project:atlas#editor@team:platform#member       ← 整个团队是编辑者
project:atlas#viewer@user:carol
```

### 6.4 一次 check 的完整推导：`check(user:bob, view, project:atlas)`

```
view(project:atlas, bob)
├─ viewer 直接命中？
│   查 project:atlas#viewer@user:bob            → MISS
│   查 project:atlas#viewer@<any>#...           → 只有 user:carol，MISS
└─ edit(project:atlas, bob)                     ← view = viewer + edit
    ├─ owner？ 查 project:atlas#owner@user:bob   → MISS
    ├─ editor？查 project:atlas#editor@*
    │   → 命中元组 project:atlas#editor@team:platform#member
    │   → 展开子问题：member(team:platform, bob)?
    │       查 team:platform#member@user:bob     → ✅ HIT
    │   ⇒ editor = TRUE
    └─ 短路返回

结果：ALLOW    展开深度 3    元组查询 4 次（其中 1 次命中）
```

对比 `check(user:alice, view, project:atlas)` —— alice 不在任何 team，走的是**穿透路径**：

```
edit → org->administrate
     → 查 project:atlas#org@organization:acme        HIT
     → 子问题 administrate(organization:acme, alice)
     → 查 organization:acme#admin@user:alice          ✅ HIT
```

**这就是 ReBAC 的价值**：alice 的 admin 权限自动覆盖了 acme 下所有项目，**没有任何一条元组把 alice 和 atlas 直接关联**。用 RBAC 表达同样的语义，你要么给 alice 写 N 条项目级角色（写放大 = 项目数），要么在应用里手写 join（那正是 Zanzibar 要消灭的东西）。

### 6.5 zookie / 一致性：新旧敌人问题

Zanzibar 的核心难点不是图遍历，是**一致性**。两个失效场景：

- **新敌人问题（new enemy problem）变体 A（顺序倒置）**：你先把 Bob 从文档移出，再把敏感内容写进文档。如果授权服务用了旧快照，Bob 仍能读到新内容。
- **变体 B（旧 ACL 用于新内容）**：ACL 更新与内容更新之间没有因果序。

解法是**一致性令牌**：每次权限写入返回一个不透明 token（Zanzibar 叫 zookie，SpiceDB 叫 **ZedToken**，OpenFGA 用 `consistency` 参数），调用方在后续 check 里带上它，授权服务保证读到**不早于**该时间点的快照。

| 模式 | 语义 | 延迟 | 用在哪 |
|---|---|---|---|
| `minimize_latency` / 无 token | 读任意较新副本，可能陈旧数百 ms | **最低** | 高频只读列表 |
| `at_least_as_fresh(ZedToken)` | 不早于该写入 | 中 | **默认应该用这个**：改完权限后的读 |
| `at_exact_snapshot` | 精确快照 | 中 | 分页遍历时保持一致视图 |
| `fully_consistent` | 强一致，绕过缓存 | **最高（可 10× 以上）** | 授权变更页面、安全敏感操作 |

**工程结论**：把 ZedToken 存进你的资源行（`resource.authz_token`），读路径带上它。这样"改完权限立刻刷新页面"是正确的，其余请求走低延迟路径。**不要全局开 `fully_consistent`** —— 那等于关掉了 Zanzibar 的全部缓存设计。

### 6.6 2026 年的实现现状

| | [SpiceDB](https://github.com/authzed/spicedb) | [OpenFGA](https://openfga.dev/) |
|---|---|---|
| 治理 | Authzed（开源 Apache-2.0） | **CNCF Incubating**（2025-10-28 升级） |
| 一致性 | ZedToken，四档 | `consistency` 参数 |
| 2026 动向 | 查询计划器（v1.53 query plan dispatch、集合论优化器）、composable schema（v1.51）、Postgres FDW | v1.10.0 引入基于运行时统计的 query planner，部分场景 `/check` **最快 10×** |
| ⚠️ 安全 | **v1.51.1（2026-04-14）修 CVE-2026-40091，低于此版本必须升级** | — |
| 幂等语义 | — | `/write` 新增参数控制"写重复 tuple / 删不存在 tuple"的行为 |

---

## 7. 授权的性能：check 是在你的 p99 预算里

一个典型 API 请求会打 **1–5 次** check。授权服务的 p99 直接叠进业务 p99。

### 延迟预算

```
业务 API p99 预算 200ms
  ├─ 授权 check（1 次，缓存命中）      0.5–2 ms
  ├─ 授权 check（1 次，未命中，1 跳）   3–8 ms
  ├─ 授权 check（未命中，4 跳穿透）     10–25 ms   ← 深层嵌套是延迟杀手
  └─ 业务逻辑 + DB                    …
```

**限制展开深度**（SpiceDB 有 `--dispatch-max-depth`，默认 50）。超过 5 跳的模型基本都是建模错误。

### 四个必须做的优化

1. **批量 check**：列表页面用 `CheckBulkPermissions`（SpiceDB）/ `BatchCheck`（OpenFGA），把 50 次 RTT 变成 1 次。**N+1 授权是最常见的性能事故。**
2. **不要用 `LookupResources` 做主查询**。"列出我能看的所有项目"在资源量大时会退化。正确做法是**先按业务过滤（分页 + tenant_id）拿 100 条候选，再批量 check 过滤**；只有在权限极稀疏时才反过来。
3. **反规范化 / Leopard 式索引**：Zanzibar 用一个专门的索引服务预计算深层组成员关系。自建的等价物是"把稳定的、扇出巨大的关系（org 成员、team 成员）物化成扁平表"，并接受**秒级**的更新延迟。
4. **两级缓存**：进程内 LRU（TTL 1–5s，按 ZedToken 分桶）+ 授权服务自身的分布式缓存。注意**缓存键必须包含一致性 token**，否则你会缓存出越权。

> **面试金句**：
> "授权服务的 SLO 必须比它保护的服务更严 —— 因为每个业务请求要打 1–5 次 check，它的可用性是乘进去的，不是加进去的。所以我给它单独的读副本和 10ms 的 p95 预算，并且**明确 fail-closed**。有人会说超时就放行以保可用性，那是把一个可用性问题换成了一个数据泄露事件 —— 这两者的赔付曲线完全不同。真正的解法是降级到本地缓存的上一次决策，而不是降级到 allow。"

### 正确性的三条硬规则

1. **默认拒绝（default deny）**。策略引擎的兜底分支必须是 deny，新增资源类型在没有规则时必须不可访问。
2. **授权决策在服务端，且在数据层之前**。查完数据再过滤 = 已经把数据从 DB 读出来了，任何日志/异常/时序都可能泄露。
3. **PEP 只有一个**。授权执行点（Policy Enforcement Point）散落在 20 个 handler 里，就一定有一个忘了写。用中间件/repository 层强制注入 tenant + 权限过滤，让"忘记检查"在编译期或框架层就不可能。

---

## 8. 多租户下的层级建模

```
organization        ← 计费、合同、数据驻留、SSO 配置的边界
  ├── workspace/team ← 协作与成员管理的边界
  │     └── project  ← 资源与权限的主要边界
  │           └── resource (document / dataset / agent / api_key)
  └── service_account / agent   ← 非人类主体，直接挂在 org 下
```

**四个必须提前决定的问题**（改起来都是数据迁移级别）：

| 问题 | 选 A 的后果 | 选 B 的后果 |
|---|---|---|
| 一个 user 能否属于多个 org？ | **能**（B2B SaaS 必须）：登录后需要"当前 org"上下文，所有 API 要带 org 作用域 | 不能：账号体系简单，但企业用户会用同一邮箱在两家公司 → 炸 |
| 跨 org 共享资源？ | 支持：ReBAC 天然能表达，但**计费归属、数据驻留、删除语义全部变复杂** | 不支持：v1 的正确选择 |
| 角色定义是全局还是每 org 自定义？ | 每 org 自定义：企业客户会要，schema 要支持 per-tenant relation | 全局：简单，但大客户会拿它当阻塞项 |
| tenant 上下文从哪来？ | **服务端会话 / 路由**：安全 | 客户端传参：只要有一个 handler 忘了校验成员关系就越权 |

**关于 JWT 里的 `tenant_id`**：可以放，因为它被你签名了。但**切换 org 必须重新签发 token**，并在签发时校验成员关系。绝不能让客户端在请求头里自由指定 tenant。

---

## 9. 机器与机器：API key / OAuth / mTLS 怎么选

| 方案 | 适用 | 优点 | 代价与陷阱 |
|---|---|---|---|
| **API key** | 客户集成、CLI、Webhook 验签 | 上手成本最低 | 长期有效 = 泄露即长期沦陷。**必须**：前缀可识别（`sk_live_`）便于扫描器检出、只存哈希、**支持 scope 与到期**、支持轮换期双 key 并存、泄露检测（GitHub secret scanning 伙伴计划） |
| **OAuth 2.0 client credentials** | 第三方应用、合作伙伴 | 短期 token、scope 化、可撤销 | 需要 AS；client_secret 仍是长期凭证 → 用 **private_key_jwt** 或 mTLS 客户端认证 |
| **mTLS / SPIFFE** | 服务间、零信任内网 | 无长期共享秘密，身份绑定在工作负载上 | 证书生命周期管理（SPIFFE SVID 通常小时级）；出问题时排查成本高 |
| **OIDC 联合身份（workload identity federation）** | CI/CD → 云资源 | **完全无静态凭证**：GitHub Actions 的 OIDC token 换云上短期凭证 | 信任策略写错（`sub` 通配太宽）= 任何仓库都能拿你的云权限 |

**默认建议**：对外用 API key（带 scope + 到期 + 轮换），对内用 mTLS/SPIFFE，CI/CD 一律用 OIDC 联合身份。**任何地方都不要长期存明文 secret。**

---

## 10. Agent 的授权：2026 年真正的新问题

### 10.1 问题的形状

传统模型是 `(user) × (resource)`。Agent 引入了第三方：

```
        委派                    执行
user ──────────> agent ──────────────> tool / MCP server ──> 上游 API
  │                │                        │
  └── 谁承担责任    └── 独立身份？            └── 用谁的 token？
```

**核心反模式**：让 Agent 直接持有用户的 token。后果是——
- 下游日志里的主体是"用户"，无法追责、无法单独封禁这个 Agent；
- Agent 被提示注入劫持时，攻击者拿到的是**用户的完整权限**；
- 一个 MCP server 拿到的 token 可以去打另一个受众的 API（confused deputy）。

### 10.2 [MCP 授权规范（rev 2025-11-25）](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization) 的硬性约束

这是目前 Agent 授权领域**约束力最强的一手规范**，逐条都是可直接落地的：

- MCP Server 是 **OAuth 2.1 Resource Server**；**MUST** 实现 [RFC 9728](https://www.rfc-editor.org/rfc/rfc9728.html)（Protected Resource Metadata），客户端 **MUST** 用它发现 AS。
- 客户端 **MUST** 实现 [RFC 8707 Resource Indicators](https://www.rfc-editor.org/rfc/rfc8707.html)：`resource` 参数在**授权请求和 token 请求中都必须带**，且不管 AS 是否支持都要发。
- Server **MUST** 校验 token 受众是自己；**MUST NOT** 接受或转发其他 token。
- **"MCP server MUST NOT pass through the token it received from the MCP client"** —— 需要调上游 API 时，MCP Server 自己作为 OAuth client 换取**另一个** token。
- PKCE `S256` 必须；若 AS metadata 缺 `code_challenge_methods_supported`，客户端 **MUST refuse to proceed**。
- 权限不足返回 **403 + `WWW-Authenticate: error="insufficient_scope", scope=…`**，触发**增量提权（step-up）**，而不是一次性索要全量 scope。
- **CIMD 优先，DCR（RFC 7591）降为向后兼容 fallback**。

> ⚠️ 版本注意：MCP 规范线为 `2026-07-28` ← `2025-11-25` ← `2025-06-18`。**2026-07-28 版把协议改成无状态**（删 `initialize` 握手、删 `Mcp-Session-Id`、Roots/Sampling/Logging 弃用、反向请求改 MRTR、Tasks 移出核心变扩展），弃用窗口 ≥ 12 个月。授权部分的 MUST 语义延续。

### 10.3 令牌传递 vs 令牌交换

```
❌ token passthrough                    ✅ token exchange (RFC 8693)
user_token ──> agent ──> MCP ──> API    user_token ──> agent
                                              │
                                              ├─ POST /token
                                              │   grant_type=token-exchange
                                              │   subject_token=<user_token>
                                              │   actor_token=<agent_identity>
                                              │   resource=https://api.crm.example
                                              │   scope=contacts.read
                                              ▼
                                        新 token：
                                          sub  = user:bob        （代表谁）
                                          act  = agent:a7        （谁在执行）
                                          aud  = api.crm         （受众唯一）
                                          scope= contacts.read   （最小）
                                          exp  = now + 300s      （5 分钟）
```

`act`（actor）声明是 [RFC 8693](https://www.rfc-editor.org/rfc/rfc8693.html) 定义的**委派链**表示法 —— 下游服务据此在审计日志里同时记录"代表谁"和"谁在执行"。这一条几乎是所有 Agent 合规要求的地基。

**跨应用委派**：企业场景下的新路径是 **ID-JAG / [Cross-App Access (XAA)](https://oauth.net/cross-app-access/)** —— 应用把 OIDC ID token 在企业 IdP 处按 RFC 8693 换成短生命周期的 JWT Authorization Grant，再换成目标资源应用的 access token，**无需每次用户交互同意**。ID-JAG 已被 IETF OAuth WG 采纳为工作组文档，Okta 以 XAA 品牌落地（有公开 sandbox）。它正好补上 MCP 规范"禁止透传但没规定怎么取上游 token"的空白。
⚠️ 目前主要由单一厂商推动，**跨 IdP 互操作性尚未被广泛验证**，写进架构前先做 POC。

### 10.4 Agent 需要自己的身份

**Agent 必须是一等身份主体，且挂一个人类 sponsor。** Microsoft [Entra Agent ID](https://learn.microsoft.com/entra/) 于 2026-04 GA，把这件事产品化了：Agent 的 sign-in 与 audit 日志与人类用户同一套；access packages 定义权限范围；**sponsor lifecycle workflow 强制每个 Agent 身份挂一个负责人**；封禁 Agent、改权限、改 sponsor 全部记入审计。

在 ReBAC 里建模就是加一个 definition：

```zed
definition agent {
    relation sponsor:      user          // 人类负责人，可追责
    relation delegated_by: user          // 当前代表谁
    relation grant:        capability    // 该 Agent 被授予的能力上限
}
```

**有效权限 = 用户权限 ∩ Agent 授权上限 ∩ 本次任务 scope**。三者取交集，缺一不可：

```
allow(agent:a7 on behalf of user:bob, edit, project:atlas) =
      check(user:bob,  edit, project:atlas)      // 用户本来就有吗
  AND check(agent:a7,  edit, project:atlas)      // Agent 被允许对它做吗
  AND task_scope_contains("project:atlas", "edit")  // 本次任务批准的范围内吗
```

**"Agent 拿到了用户的全部权限"的解法就是这个交集**，加上四个约束：

1. **每任务派生的短命凭证**：任务开始时按任务声明的资源清单换一批 5–15 分钟的窄 scope token，任务结束即作废。
2. **凭证不进沙箱**：长期 token 由沙箱外的代理持有，只向沙箱内下发派生凭证（MCP 规范的 MUST NOT + 主流产品实现双向印证）。
3. **每个 MCP server / 工具当作独立的不可信安全域**：独立凭证、独立 scope、**绝不共享 token**、tool 定义做哈希 pin 防 rug pull。
4. **高影响动作走人工审批**，且**清单必须在设计期静态定义，不得运行时交给 Agent 自判**（[Five Eyes 联合指南《Careful Adoption of Agentic AI Services》，2026-05](https://www.cisa.gov/)）。典型清单：转账、权限/身份变更、对外发送消息、发布公开内容、不可逆删除、生产写操作。

⚠️ 审批不是银弹：实测 Claude Code 场景下人类的**自动批准率约 93%**。审批点必须**少而重**，并且弹窗要展示完整参数（不能截断），否则审批疲劳会把它变成一个昂贵的点击动作。

**术语升级**：Agent 场景下更准确的原则不是最小权限（least privilege，限制"能访问什么"），而是**最小代理权（least agency）** —— 进一步限制"能做什么、什么时候做、以什么频率、在什么上下文、是否需要升级审批"。

---

## 11. 什么时候不要这么做（反模式）

| 反模式 | 为什么错 | 正确做法 |
|---|---|---|
| **把 ReBAC check 塞进应用 ORM 做本地 join** | 那正是 Zanzibar 要解决的问题：权限图跨服务、跨库，join 不出来且没有一致性语义 | 独立的、可水平扩展的、有一致性 token 的授权服务 |
| **超时就放行（fail-open）** | 把可用性问题换成数据泄露 | fail-closed + 降级到缓存的上次决策 |
| **每个 handler 手写权限检查** | 一定会漏 | 中间件/repository 层强制注入，让漏检不可能 |
| **用 `LookupResources` 当主查询** | 资源多时退化 | 业务分页 + 批量 check 过滤 |
| **全局 `fully_consistent`** | 关掉了整个缓存层，p99 涨 10× | 只在权限变更后的读带 ZedToken |
| **Agent 复用人类凭证 / token 透传** | MCP 规范 MUST NOT；下游日志主体错误、无法单独封禁 | token exchange + `act` 声明 |
| **omnibus scope（`*` / `all` / `full-access`）** | 一旦泄露就是全量 | 渐进式最小权限 + 403 step-up |
| **在 JWT 里放 PII** | base64 不是加密 | 只放标识符 |
| **一开始就上 ReBAC** | 多一个关键路径上的有状态服务要运维 | 没有共享/继承语义时，RBAC 表就够 |
| **SCIM `DELETE` 真删用户** | 破坏审计链与历史归属 | `active: false` |

---

## 面试官会追问

1. 你的 access token TTL 设多久？这个数字和"离职后多久失效"的合规要求是什么关系？如果要求是"立即"，你怎么做？
2. 一个 JWT 校验库默认接受 header 里的 `alg`，会发生什么攻击？给出完整利用链。
3. 画出 `check(user:X, view, project:Y)` 的展开过程，说明哪些步骤会打 DB、哪些能命中缓存。
4. 用户刚被移出项目，他立刻刷新页面还能看到内容 —— 这是 bug 还是设计？你怎么控制？
5. 一个列表页要渲染 200 条资源，每条都要鉴权。你怎么做到不打 200 次 check？如果权限极其稀疏（用户只能看到 3 条）呢？
6. 授权服务挂了，你的系统是全站 503 还是全站放行？为什么？中间状态有什么？
7. Agent 代表用户去调一个第三方 CRM，token 链条是怎样的？为什么不能直接把用户的 token 传下去？
8. 你怎么防止一个被提示注入劫持的 Agent 拿着用户全部权限做破坏？说出至少三层控制。
9. RBAC / ABAC / ReBAC，你在什么信号出现时会从前一个迁移到后一个？迁移过程怎么做到不停机？

---

**下一篇** → [04-isolation-and-compliance.md](04-isolation-and-compliance.md)
