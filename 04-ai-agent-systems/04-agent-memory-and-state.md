# 04 · Agent 记忆与状态

> "给 Agent 加记忆"这个需求，90% 的情况下被错误地实现成"把对话塞进向量库"。
> 真正的问题是：**四类完全不同的状态被同一个词绑架了**，而它们的一致性要求、生命周期、丢失后果、合规义务全都不一样。

---

## 1. 先把"记忆"这个词拆掉

Agent 系统里被叫做"记忆"的东西至少有四类。**它们不该用同一套存储。**

| # | 类别 | 具体例子 | 生命周期 | 典型体量 | 一致性要求 | 存储选型 | 丢了会怎样 |
|---|---|---|---|---|---|---|---|
| ① | **会话上下文**<br>conversation context | messages 数组、tool_use / tool_result、thinking 块 | 单会话，分钟–小时 | 10K–1M token（40KB–4MB） | **持久 append 即可**，不需要事务 | 对象存储 append-only JSONL（一会话一对象）+ 元数据行在 Postgres；活跃会话在 Redis 做热副本 | 会话续不上，用户从头再来 |
| ② | **任务状态**<br>task / execution state | 当前 step、待办图、重试计数、已发生的副作用与幂等键（idempotency key）、预算余额 | 单任务，秒–天 | **KB 级，硬上限 64KB** | **强一致（strong consistency） + CAS**，这是唯一必须事务的一类 | 关系库单行（Postgres `FOR UPDATE` 或 `version` 乐观锁 optimistic locking）/ DynamoDB 条件写；或交给 durable execution 引擎 | 恢复退化成重跑，**副作用重复执行**（重复扣款、重复发信） |
| ③ | **长期记忆**<br>long-term memory | 用户偏好、项目约定、踩过的坑、"上次我们决定用 pnpm" | 跨会话，月–年 | 每用户 KB–低 MB | **最终一致（eventual consistency）就够** | Postgres 结构化事实表（可枚举、可审计、**可删除**）+ 可选向量索引作二级召回；代码类场景直接用文件系统 | 体验退化（重复提问），**不影响正确性** |
| ④ | **外部真相源**<br>source of truth | 代码仓、工单系统、CRM、业务数据库 | 与业务同寿 | GB–TB | 由业务系统定义 | **不归你管**，Agent 只读或受控写 | 业务事故 |

**两个最常见的架构错误：**

1. **把 ③ 塞进 ①** —— 把记忆直接拼进 system prompt 并永久累积。结果：上下文单调膨胀、prompt cache 每次改记忆就作废、用户要求"忘掉这件事"时你只能重写历史。
2. **把 ④ 复制进 ③** —— 把 CRM 里的客户信息、代码库里的函数签名"记住"。结果：记忆过期而真相源已变，Agent 拿着三个月前的快照做决策，而且你无从知道它错在哪。**④ 永远重新查，不要记。**

> **判据（decision rule）**：一条信息如果能在 200ms 内从真相源查到，就不要写进记忆。记忆存的是**真相源里没有的东西**——用户的偏好、你和用户之间的约定、失败过的路径。

---

## 2. 记忆分层：认知类比与工程对应物

认知科学的四层分法在这里不是修辞，它直接决定了**存在哪、怎么注入**。

| 认知层 | 内容 | 工程对应物 | 存活期 | 注入方式 | 是否进稳定前缀 |
|---|---|---|---|---|---|
| **工作记忆**<br>working | 当前正在处理的东西 | 上下文窗口本身 | 单轮–单会话 | 天然在场 | — |
| **情节记忆**<br>episodic | "上次我们做了什么、为什么" | 会话摘要 + 事件日志 + trace 存档 | 会话级，保留数十天 | 检索注入（recency × 相似度） | ❌ 每次不同，放末尾 |
| **语义记忆**<br>semantic | "用户偏好 4 空格缩进"、"这个仓库用 pnpm" | 结构化事实表：`(subject, predicate, object, confidence, valid_from, valid_until)` | 长期 | **会话开始一次性注入，会话内冻结** | ✅ |
| **程序性记忆**<br>procedural | "跑测试前要先 `pnpm build`" | Skill / `CLAUDE.md` / 工具定义 / few-shot 示例 | 长期，**版本化** | 进 system 段 | ✅ |

**最重要的一条结论：程序性记忆的正确载体不是向量库，是可执行物。**

一条"记得先 build 再 test"的自然语言记忆，命中率（hit rate）取决于检索；一个 `make test` 里写死 `build && test` 的 Makefile，命中率是 100%。**能变成工具、脚本、lint 规则、CI 检查的记忆，就不要留在记忆系统里。** 这是把非确定性（non-determinism）换成确定性的免费午餐。

---

## 3. 写入判据：什么值得记，谁来决定

无节制写入是记忆系统的头号死法。1000 轮对话如果每轮提取 2 条事实，一年后你有一个装满互相矛盾垃圾的库，检索质量崩塌。

### 三档写入权限（按风险分级）

| 档位 | 谁触发 | 适用内容 | 是否需确认 |
|---|---|---|---|
| **A. 用户显式** | 用户说"记住…" | 任何内容 | 否，但要回显"我记住了 X"，给撤销入口 |
| **B. Agent 提议 + 用户裁决** | Agent 检测到候选 | 偏好、约定、决策 | **是**，批量在会话结束时确认 |
| **C. 自动抽取** | 后台流水线 | **仅限低风险、可证伪（falsifiable）、可自动失效的类别**（如"该仓库使用 pnpm"—— 下次读 `package.json` 就能验证） | 否，但必须标 `confidence` 且能被自动推翻 |

**C 档的边界要卡死**：涉及人（同事姓名、职级判断）、涉及承诺（"客户同意了 X"）、涉及金额与权限的，一律不允许自动抽取。这不是保守，这是因为**错误的长期记忆会长期生效**，而它的错误在被使用时不产生任何信号。

### 写入门函数（gating function，可直接抄）

```python
def should_persist(candidate: Fact, ctx: Session) -> Decision:
    # 1. 硬闸门：真相源可查的不记
    if candidate.derivable_from_source_of_truth:
        return REJECT("queryable")

    # 2. 稳定性：预计有效期 < 7 天的不进语义记忆，进情节记忆
    if candidate.expected_valid_days < 7:
        return DEMOTE_TO_EPISODIC

    # 3. 复用性：估计未来 30 天被用到的概率
    if candidate.p_reuse < 0.2:
        return REJECT("low_reuse")

    # 4. 可证伪性：说不出"什么情况下这条是错的"就别记
    if candidate.falsification_condition is None:
        return REJECT("unfalsifiable")

    # 5. 污点：来源含不可信内容（网页/邮件/第三方工具返回）→ 必须人工确认
    if candidate.provenance.tainted:
        return REQUIRE_HUMAN_APPROVAL

    # 6. 敏感：PII / 凭据 / 健康 / 支付 → 拒绝，或走独立加密域
    if candidate.pii_class != NONE:
        return REJECT("pii") if not ctx.pii_memory_enabled else ROUTE_TO_ENCRYPTED_STORE

    return ACCEPT
```

### 每条记忆的必备元数据

少一个字段，后面的更新、遗忘（forgetting）、合规删除就都做不了：

```json
{
  "memory_id": "mem_01J...",          // 可寻址 → 用户可编辑/可删除的前提
  "scope": "user:u_123",              // user / project / tenant / global，决定隔离边界
  "kind": "semantic",                 // semantic | episodic | procedural
  "content": "该仓库用 pnpm，不要用 npm install",
  "confidence": 0.9,
  "source": "conversation",           // conversation | extraction | user_explicit | tool_output
  "provenance": {                      // 出了问题能追到哪一轮
    "session_id": "s_88", "turn": 42, "tainted": false
  },
  "valid_from": "2026-03-01T00:00:00Z",
  "valid_until": null,                 // 显式时效，null = 直到被推翻
  "supersedes": ["mem_01H..."],        // 矛盾消解链，不覆盖只追加
  "version": 3,                        // 多设备并发写的 CAS 依据
  "last_used_at": "2026-07-20T...",   // 遗忘策略的输入
  "use_count": 17
}
```

---

## 4. 检索：何时注入、注入多少

### 三种注入时机的取舍

| 策略 | 做法 | prefix cache | 相关性（relevance） | 适用 |
|---|---|---|---|---|
| **会话开始一次性注入** | 拉该用户全部语义记忆（截断到预算），放进 system 段后的固定块，**会话内不再变** | ✅ **全程命中** | 中（有无关记忆占位） | 记忆总量 < 5K token 时的默认选择 |
| **每轮检索注入** | 每轮按当前 query 检索 top-k 记忆拼进 prompt | ❌ **每轮击穿**（记忆块在历史前面，其后全部作废） | 高 | 记忆库很大且必须精准时，且**只能放在消息末尾** |
| **工具化按需拉取（on-demand fetch）** | 给 Agent 一个 `search_memory(query)` 工具，它自己决定要不要查 | ✅ 命中（追加式） | 高，但多一次往返 | 记忆库大 + 会话长的默认选择 |

**这是本篇和 [02-caching.md](../01-building-blocks/02-caching.md) §7 直接对接的地方**：Anthropic 的缓存失效级联（invalidation cascade）顺序是 **tools → system → messages**（2026-07 口径）——改工具定义全废，改 system 则 system+messages 废，只改 messages 只废 messages。所以：

> **面试金句**：
> "记忆注入的位置决定了它的成本。我会把语义记忆和程序性记忆放进 system 段，并在会话开始时冻结成快照——它们在整个会话里逐字节不变，所以能进 prompt cache 的稳定前缀，缓存读只要输入价的 10%。情节记忆和检索结果我一律放在 messages 末尾追加，绝不插到中间。如果产品要求'记忆实时更新到当前会话'，我会明确告诉产品这个需求的成本是每轮重建整段缓存——按 Opus 5 档位算，把 30K 前缀重写一次是 $0.19，而缓存读只要 $0.015，差 12 倍。"

### 注入多少：把记忆当预算而不是当福利

- **记忆预算 ≤ 上下文窗口（context window）的 2–5%**。200K 窗口 → 4K–10K token。
- 依据来自 [Chroma 的 context rot 实证](https://www.trychroma.com/research/context-rot)（2025-07-14，18 个前沿模型）：性能随输入长度**单调下降，远在窗口填满之前就开始**；**单个 distractor 就能降低准确率，4 个复合放大**；LongMemEval 上 **~300 token 的聚焦 prompt 显著优于 113,000 token 的完整 prompt**。
- 推论：**多注入一条不相关的记忆是净负收益**，不是"反正也不贵"。记忆检索的目标函数是精确率（precision），不是召回率（recall）——这跟 RAG 正好相反。

---

## 5. 更新与遗忘：这一节是合规硬要求

### 矛盾消解（conflict resolution）

新旧记忆冲突时的四种策略，按可辩护性排序：

| 策略 | 做法 | 问题 |
|---|---|---|
| **显式 supersede 链**（推荐） | 不覆盖，写新条目并在 `supersedes` 里指向旧条目；旧条目标 `superseded_at` 但保留 | 存储增长；需要检索时过滤 |
| 置信度加权 | 高置信覆盖低置信 | 置信度本身是模型估的，不可靠 |
| Last-write-wins | 新的赢 | 一次误抽取就永久污染 |
| 交给用户裁决 | 冲突时弹给用户 | 打扰成本高，只用于高影响记忆 |

**为什么必须是 supersede 而不是覆盖**：覆盖之后你无法回答"这条记忆是什么时候、因为哪一轮对话变成现在这样的"。记忆投毒（memory poisoning）事件的排查完全依赖这条链（见 §10）。

### 时效与级联失效

两套时间戳，别混：

- `valid_from / valid_until` —— **事实在现实世界中的有效期**（"用户 2026 年 Q3 在做 X 项目"）
- `created_at / superseded_at` —— **记录在你系统里的存在期**

这就是 bitemporal 建模。没有它，"用户三个月前说他讨厌 dark mode，上个月改主意了"这种情况你无法正确回答"他现在偏好什么"和"上个月我为什么给他推了 dark mode"这两个问题。

**级联失效（cascading invalidation）**：如果记忆 M 是从文档 D 推出的，D 被删/被改，M 必须失效。做法是给每条派生记忆存 `derived_from: [doc_id@version]`，在文档更新事件上跑失效扫描。**这是缓存失效（cache invalidation）问题的翻版，同样的坑。**

### 遗忘策略

不设遗忘的记忆库会在 6–12 个月内检索质量崩塌。三条可执行规则：

1. **访问衰减（access decay）**：`score = confidence × exp(-λ · days_since_last_used)`，λ 取 1/90（半衰期 half-life 约 62 天）。score < 阈值的降级到冷存储，不参与检索。
2. **容量上限**：每个 scope 硬性上限（如每用户 500 条语义记忆）。超限时按 score 淘汰（eviction），**淘汰要通知用户**而不是静默。
3. **强制复核**：`confidence < 0.7` 且 `use_count == 0` 且存活 > 90 天 → 删除。没被用过的记忆没有价值。

### 用户可编辑与可删除（GDPR/CCPA 的硬约束）

删除权（right to erasure，GDPR Art.17）落到 Agent 系统上，**要删的不止那一行记录**。上线前必须能回答"删干净了吗"的清单：

| 载体 | 是否含用户数据 | 删除方案 |
|---|---|---|
| 记忆表主记录 | 是 | 硬删（hard delete，不是 soft delete） |
| supersede 链上的历史版本 | 是 | 同批删 |
| **向量索引里的 embedding** | **是**（embedding 是个人数据的派生物） | 必须支持按 `memory_id` 定点删除 → **这直接排除掉不支持删除的索引结构** |
| 会话上下文归档（JSONL） | 是 | 要么按用户分区能整体删，要么设短保留期 |
| compaction 生成的摘要 | 是 | 摘要是派生物，同样要删——**很多实现漏掉这个** |
| 训练/评测数据集里的样本 | 是 | 需要 `(user_id → sample_id)` 反查索引 |
| **provider 侧 prompt cache** | 是 | **你删不掉**。OpenAI 即便在 ZDR 下仍存储加密的 prompt cache tensor **最长 24 小时**（2026 口径）；这必须写进你的 DPIA 和数据流图，并向用户披露"最长 24 小时的技术性驻留（data residency）" |
| 可观测平台的 trace | 是 | OTel GenAI 的 `gen_ai.input.messages` / `gen_ai.output.messages` 是 **Opt-In，默认不采集**——保持默认关闭是最省事的合规姿势 |

⚠ **架构后果**：删除权要求"记忆可枚举、可归因到自然人、可物理删除"。这三条同时排除了一个很流行的做法：**把记忆混进 system prompt 的一大段自由文本里**。那段文本不可寻址（addressable），你只能整体重写。**从第一天就用结构化条目存记忆，不要用自由文本。**

---

## 6. 文件系统作为记忆：2026 的主流实践

把记忆写成文件、让 Agent 用 `read`/`write`/`grep` 自己管理，是编码类 Agent 的事实标准。

```
repo/
├── CLAUDE.md                 # 程序性记忆：约定、命令、禁忌。进 system 段，稳定前缀
├── .agent/
│   ├── progress.md           # 情节记忆：当前目标、已完成、卡在哪、下一步
│   ├── features.json         # 任务状态：[{id, desc, status: pass|fail|todo, test_cmd}]
│   ├── decisions/            # 语义记忆：一决策一文件，带日期，append-only
│   │   ├── 2026-03-11-use-pnpm.md
│   │   └── 2026-05-02-drop-redis.md
│   └── scratch/              # 工作区，随时可删，不进 git
└── (git history)             # 天然的情节记忆 + 回滚点：描述性 commit message
```

**为什么这套在实践中赢了向量库：**

- **可被人读写、可 code review、可进 git**。记忆错了直接改文件，不用查数据库。
- **天然版本化**。`git log .agent/decisions/` 就是记忆的审计日志（audit log）。
- **和真相源同处一地**，不存在同步问题。
- [Anthropic 的官方建议是"外部化状态（externalize state）"而非"更聪明的压缩"](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)：结构化 JSON feature list（带 pass/fail）、progress 文件、描述性 git commit 做回滚点、init script 做环境重建；**一次只做一个 feature**；固定启动序列为「读 progress 与 git 历史 → 先跑端到端测试 → 先修已有 bug 再做新活」。

**边界在哪：** [《Is Grep All You Need?》（arXiv:2605.15184）](https://arxiv.org/abs/2605.15184) 在 LongMemEval 子集上报告 grep 总体准确率高于向量检索，但同一篇明确强调结果**强依赖 harness 与 tool-calling 风格**；[LlamaIndex 2026-01-13 的对照](https://www.llamaindex.ai/blog/did-filesystem-tools-kill-vector-search) 显示扩到 100/1000 篇文档后 RAG 在速度上大幅胜出。**这是有争议的，不要选边。** 可用的规模分流判据：

| 记忆语料形态 | 选择 |
|---|---|
| 文本文件、**数百到数千个**、在一个文件系统里 | 文件 + grep + 目录约定 |
| 结构化事实、需要按 scope/时效/置信度过滤 | 关系库（先过滤再排序，别上向量） |
| 跨会话的自然语言片段、**数万条以上**、跨用户 | 向量索引 + 强制 `scope` 过滤 |

---

## 7. Compaction：触发、实现，以及它和前缀缓存（prefix caching）的正面冲突

### 触发

Anthropic 的服务端 context editing（beta header `context-management-2025-06-27`，截至 2026-07 **仍是 beta**）给出了可抄的参数。**完整参数表与三个坑见 [`02-context-engineering-and-rag.md §10`](02-context-engineering-and-rag.md)**，这里只补记忆视角特有的两条：

| 事项 | 说明 |
|---|---|
| `exclude_tools` **必须排除 memory 类工具** | 记忆读写的结果被清掉 ⇒ Agent 反复重读同一条记忆，token 不降反升，且在轨迹（trajectory）里表现为"步骤重复" |
| 上线前用 `count_tokens` 干跑（dry run） | 该端点接受 `context_management` 参数，同时返回 `original_input_tokens` 与清理后的 `input_tokens` —— 这是唯一能在不花推理钱的前提下调 `trigger` / `clear_at_least` 的方式 |

⚠ Claude Code 侧的具体阈值（约 95% 容量触发、预留 buffer ~33,000 token、环境变量 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` / `CLAUDE_CODE_AUTO_COMPACT_WINDOW`、社区建议调到 40–70%）为**社区逆向，官方未文档化，各来源互相矛盾**——知道有这回事就行，不要当规格用。

**多水位线（watermark）的心智模型**（⚠ 同为社区逆向）：`Normal → Warning → 微压缩 → Error → AutoCompact（起一个 compact 子代理做摘要）→ BlockingLimit（强行阻断）`。值得抄的是**分级**这件事本身：不要只有一个"满了就压"的开关，先做便宜的（清工具结果），再做贵的（摘要重写）。

### 一次 compaction 的前后对比

```
── compaction 前：190K / 200K，逼近上限 ──────────────────────────
┌──────────────────────────────────────────┬───────┬──────────────┐
│ system prompt                            │  2.0K │ ┐            │
│ 工具定义（12 个 tool）                    │  3.5K │ ├ 稳定前缀   │
│ CLAUDE.md（程序性记忆）+ 语义记忆快照      │  1.5K │ ┘ cache 命中 │
├──────────────────────────────────────────┼───────┼──────────────┤
│ 第 1–40 轮对话文本                        │  28K  │ ┐            │
│ 工具结果（文件内容、搜索结果、日志）        │ 142K  │ ├ 可压缩区   │← 大头在这
│ 最近 3 轮                                 │  13K  │ ┘            │
└──────────────────────────────────────────┴───────┴──────────────┘
                                             190K

── compaction 后：36K / 200K ──────────────────────────────────────
┌──────────────────────────────────────────┬───────┬──────────────┐
│ system prompt                            │  2.0K │ ┐ 未改动     │
│ 工具定义                                  │  3.5K │ ├ 若 cache   │
│ CLAUDE.md + 语义记忆快照                  │  1.5K │ ┘ breakpoint │
│                                          │       │   打在这里， │
│                                          │       │   仍然命中   │
├──────────────────────────────────────────┼───────┼──────────────┤
│ ▣ COMPACTED SUMMARY                       │  4.5K │              │
│   · 目标与验收标准（逐字保留，不许摘要）    │       │              │
│   · 已做出的决策 + 理由                    │       │              │
│   · 已修改的文件清单 + git SHA             │       │              │
│   · 失败过的尝试（防止重复踩坑）           │       │  ← messages  │
│   · 未决问题                               │       │     段被     │
│ ▣ 外部化产物指针                          │  0.4K │     整体重写 │
│   .agent/progress.md、features.json、SHA  │       │   → 缓存作废 │
│ ▣ keep=3：最近 3 组 tool_use/result       │  4.0K │              │
│ ▣ 最近 3 轮原文（逐字）                   │  13K  │              │
└──────────────────────────────────────────┴───────┴──────────────┘
                                              28.9K（messages 段 −85%）
```

**摘要里必须逐字（verbatim）保留、不许模型改写的四样东西**：用户的原始目标、验收标准、明确的禁止项、以及所有外部标识符（文件路径、git SHA、工单号、API 资源 ID）。模型在摘要时最容易做的坏事就是"顺手优化"目标描述——这是 MAST 分类里"违反任务规格"的主要制造机制。

### 和前缀缓存的账：compaction 不是无条件省钱

Compaction 改写 messages 段 → 该段 prefix cache 全部作废 → 下一次调用要重新写缓存。按 Opus 5 档位（输入 $5/M、缓存读 $0.50/M、5 分钟缓存写 1.25× = $6.25/M，2026 年中量级，随时变动）：

```
压缩前，每轮读 150K 缓存：  150,000 × $0.50/M   = $0.0750 / 轮
压缩后，首轮重写 30K 缓存：  30,000 × $6.25/M   = $0.1875 （一次性）
压缩后，每轮读 30K 缓存：    30,000 × $0.50/M   = $0.0150 / 轮

回本轮数 = 0.1875 / (0.0750 − 0.0150) ≈ 3.1 轮
```

**结论：如果 compaction 之后会话在 3 轮内结束，你亏了。** 这就是 `clear_at_least` 存在的理由——不要为了清 2000 个 token 去击穿 150K 的缓存。上线前必须做的是**缓存命中率与 token 数的联合核算**，只看 token 数会得出完全错误的结论。

Anthropic 官方给出的 context editing 收益是：100 轮 web search 场景 token 消耗 **−84%**；context editing 单独 **+29%** 性能，配 memory tool **+39%**（单一 benchmark，一手）。这些数字成立的前提正是长会话——短会话上 compaction 是纯亏。

---

## 8. 状态机与 checkpoint

### 存什么

```python
Checkpoint = {
  "workflow_id": "wf_...", "step_id": "step_017", "version": 17,   # CAS 依据
  "goal": "...",                                # 不变量，逐字保留
  "plan": [{"id":"t1","status":"done","deps":[]}, ...],
  "effects": {                                  # 已发生的副作用 → 恢复时跳过
      "step_012": {"idem_key":"wf_x:step_012", "at":"...", "result_ref":"s3://..."}
  },
  "cursors": {"repo_sha":"a3f9b2", "api_page_token":"...", "file_versions":{...}},
  "budget": {"tokens_used": 412_000, "tokens_cap": 1_500_000, "deadline":"..."},
  "context_ref": "s3://sess/s_88.jsonl#bytes=0-4210331",   # 指针，不是内容
}
```

**不存**：完整 messages（存指针）、大工具输出（存 `hash + 指针`）、模型 thinking（可重新生成）。

**大小目标 < 64KB。** 超过就说明你在 checkpoint 里存内容而不是存状态，恢复会变慢、写放大（write amplification）会失控。

### 恢复的正确性：三条硬规则

1. **checkpoint ≠ durable execution。** 这一点存在公开争议：LangChain 的立场是配了 checkpointer 即为 durable execution，可任意点 pause/resume；[Diagrid 的反驳](https://www.diagrid.io/blog/checkpoints-are-not-durable-execution-why-langgraph-crewai-google-adk-and-others-fall-short-for-production-agent-workflows)是 checkpoint 模型缺确定性重放（deterministic replay）、跨服务编排与精确一次（exactly-once）副作用。**根因是对 "durable" 的定义不同：状态快照恢复 vs 执行历史重放。** 无论站哪边，工程结论是一致的：**有外部副作用（付款、发信、开单、合并 PR）的步骤，必须 journal + 幂等键，不能只靠状态快照。**

2. **幂等键必须从 `(workflow_id, step_id)` 派生**，不能用随机 UUID。理由和计费系统一样：客户端重试时如果生成新 key，幂等就退化成"重新执行一次"。

3. **checkpoint 的粒度决定你能保住多少进度。** [LangGraph 的持久化层只在节点之间存 state，不存节点内部](https://docs.langchain.com/oss/python/langgraph/durable-execution)——节点内一个跑了 8 分钟的工具调用崩了，整节点重跑。`durability` 三档：`"exit"`（只在图退出时落盘，**无中途恢复**）/ `"async"`（异步落盘，**崩溃时有小概率 checkpoint 未写入**）/ `"sync"`（每步同步落盘，有性能开销）。**生产选 `sync`，除非你已经量过它的开销并能接受重跑。**

**同类的恢复语义陷阱**：Claude Code Dynamic Workflows 的恢复是**按 Agent 启动顺序回放，缓存命中止于第一个未完成的 Agent，此后启动的全部重跑**——A/B/C/D 依次启动、停在 B 运行中，恢复时 B/C/D 全部重跑，哪怕 C/D 已经完成。**工程推论：把工作切成很多小 Agent 比一个长 Agent 保住更多进度。**

### 环境不是状态的一部分，但恢复需要它

沙箱会消失（E2B 会话硬上限 Hobby **1 小时** / Pro **24 小时**，2026-07 一手）。所以 checkpoint 里存的是**重建环境的配方**（git SHA + init script + 依赖 lock 文件），不是环境快照。**"恢复"= 重建沙箱 + 检出 SHA + 跑 init + 从 step_id 继续**，不是"把容器唤醒"。

---

## 9. 多设备 / 多会话一致性

| 对象 | 写者模型 | 机制 |
|---|---|---|
| **会话上下文** | **单写者（single-writer）** | 一个 session 同时只允许一个活跃 runner。用**租约**（lease，TTL 30–60s，心跳续期）而不是锁——runner 崩了租约自然过期。第二个客户端接入时只读，或抢租约后接管 |
| **任务状态** | **单写者** | 同上，且写走 CAS（`WHERE version = ?`） |
| **长期记忆** | **多写者** | 手机上说"记住 A"、电脑上说"记住 not A"会真实发生。每条记忆带 `version`，写用 CAS；冲突不覆盖，走 supersede 链，把裁决留到读时（或推给用户） |

**read-your-writes 的做法**：写记忆返回一个 version token（形态类似 SpiceDB 的 ZedToken），客户端下次读带上它，读路径保证不返回比该 token 旧的视图。没有这一条，用户会遇到"我刚说了记住，换个设备它就不知道"——这是最伤信任的一类 bug，因为它让用户觉得记忆系统整体不可信。

---

## 10. 多租户（multi-tenancy）隔离与记忆投毒

### 隔离

记忆是**跨会话生效的持久写入**，所以它的隔离等级要按"数据库"而不是按"缓存"来设计：

- **`scope` 是强制查询条件，不是过滤器**。向量检索尤其危险——先做向量近邻再按 tenant 过滤，会在候选集里泄露信息（延迟差、结果数量差都是侧信道 side channel），且 top-k 会被别的租户的向量挤占。**必须是先隔离再检索**：Pinecone 走 namespace、Weaviate 走原生 multi-tenancy、Qdrant 走 payload 索引 + 过滤（过滤在 HNSW 遍历内部执行）、pgvector 走 RLS + `tenant_id`。
- **别把记忆和 KV cache 混为一谈**。跨租户共享 prefix cache 已被 **PROMPTPEEK**（NDSS 2025）实证可逐 token 重建他人 prompt（已知 prompt 模板时成功率 99%，**无任何背景知识 95%**），攻击面覆盖 vLLM、SGLang、LightLLM、DeepSpeed。**自建推理栈上跨租户共享 prefix cache 默认关闭，只在同租户内共享。** 这直接削掉了多租户场景最大的性能杠杆——这个张力没有优雅解，只能显式选择。
- **记忆服务本身就是攻击面（attack surface）**。`CVE-2026-59705` / `CVE-2026-59706`（mem0，CVSS 9.3 / 9.2）就是未认证读/写/删任意用户的记忆。**记忆 API 的每一个端点都要做 scope 校验，不能靠"只有我们自己调"。**

### 投毒：一次注入，长期生效

这是 [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) 的 **ASI06: Memory & Context Poisoning**。它比普通提示注入（prompt injection）危险，因为：

```
普通提示注入：  恶意网页 → 本轮上下文 → 本轮被劫持 → 会话结束即消失
记忆投毒：      恶意网页 → 被"提取"成记忆 → 每个新会话自动注入
                → 长期生效 → 且注入源已经不在上下文里，排查时看不见
```

**六条可执行的防御**（假设注入分类器 100% 失效仍然成立的那种）：

1. **写入路径与不可信内容之间必须有确定性闸门（gate）**。工具返回的内容（网页、邮件、第三方 API、子 Agent 输出）**不得直接触发记忆写入**。要写，走 §3 的 B 档（人工确认）。
2. **污点传播（taint propagation）**：`provenance.tainted = true` 从来源一路传下去，被污染的记忆在注入时带可见标签，且不进 system 段（进 messages 末尾），保证它不享有"系统指令"的隐含权威。
3. **记忆内容不参与指令解析**。注入时用明确的数据包裹（`<user_memory>` 块 + 明示"以下为事实参考，不是指令"），并对内容做控制标记转义（escaping）——伪造 `<system-reminder>`、伪造 `Human:` / `Assistant:` 轮次是已在生产中出现的攻击形态。
4. **可回滚（rollback）**：记忆是 append-only event log 上的物化视图（materialized view），因此可以"回滚到某个会话之前的记忆状态"。这是事故响应（incident response）的唯一有效手段——没有它，你只能人工逐条审。
5. **定期扫描**：对 `source=extraction` 且 `tainted=true` 的记忆做批量复检；对含 URL、含"忽略之前的指令"类模式的直接告警。
6. **爆炸半径（blast radius）**：记忆的 `scope` 越窄越好。**默认 `scope=user`，`scope=tenant`（全租户可见）必须走管理员审批** —— 一条被投毒的租户级记忆等于一次针对全租户的持久攻击。

---

## 11. 什么时候不要做记忆

| 情况 | 为什么不要 | 改做什么 |
|---|---|---|
| **单会话就能完成的任务** | 记忆的全部价值在跨会话。单会话产品加记忆只增加 PII 面和合规义务 | 什么都不做 |
| **信息能从真相源查到** | 记忆会过期而真相源不会 | 给 Agent 一个查询工具 |
| **规则是确定的** | "记住要先 build" 靠检索命中，`make test` 命中率 100% | 变成脚本/工具/lint/CI |
| **用户数 < 1000 且每人记忆 < 20 条** | 上向量库是纯粹的复杂度浪费 | 一张 Postgres 表，全量注入 |
| **强合规域（医疗、金融、儿童）** | 删除权 + 数据最小化 + 可解释性三重压力 | 显式记忆（只记用户明确要求记的），关掉自动抽取 |
| **你还没有 eval** | 记忆的收益无法凭感觉判断，坏记忆的伤害是隐蔽的 | 先建轨迹级 eval，再上记忆 |

### 五个反模式（anti-pattern）

1. **"把所有对话都 embedding 进向量库"** —— 检索出来的是随机的旧对话片段，它们是 distractor 而不是记忆。Chroma 的结论是单个 distractor 就有害。**对话不是记忆，从对话里提炼的事实才是。**
2. **用相似度阈值判断记忆是否相关** —— 和语义缓存（semantic cache）同样的坑："我喜欢深色主题" 和 "我不喜欢深色主题" 的 embedding 相似度极高。**记忆检索必须先按 scope/kind/时效硬过滤，再排序。**
3. **把记忆当缓存** —— 缓存失效了回源（origin fetch），记忆失效了没人知道。**任何有 TTL 语义的东西都不该进记忆系统。**
4. **compaction 之后不外部化** —— 压缩掉的信息如果没有落到 `progress.md` / git commit / feature list 里，它就是**永久丢失**。Anthropic 的官方姿态是"外部化状态"优先于"更聪明的压缩"，理由正在于此。
5. **记忆写入没有 eval 门禁** —— 上线三个月后你会有一个装满矛盾事实的库，而且没有任何指标下降来告诉你。**最低限度要监控：记忆条数增长率、`use_count == 0` 的占比（健康值 < 30%）、supersede 率（突增 = 抽取器坏了）。**

---

## 12. 可观测（observability）

OTel GenAI 语义约定已经给记忆操作留了位置（⚠ 但**全部条目仍是 Development，0 项 GA**，2026-07-30 一手核实）：`gen_ai.operation.name` 的取值枚举里包含 `create_memory` / `update_memory` / `upsert_memory` / `search_memory` / `delete_memory` / `create_memory_store` / `delete_memory_store`。

务必注意两点：
- **官方 `gen_ai.*` 命名空间里没有 cost 属性，也没有 tenant / user 属性**（新仓库 registry 原文明确提到这两类"notably absent"）。你要按租户归因（attribution）记忆成本，得自己加属性，跨平台不保证被识别。
- **内容类属性默认不采集**（`gen_ai.input.messages` 等是 Opt-In）。保持默认——一旦开了，你的 trace 平台就变成了一个需要执行删除权的个人数据副本。

必须上的四个指标：

| 指标 | 健康值 | 异常含义 |
|---|---|---|
| 记忆注入 token / 总输入 token | 2–5% | > 10% = 注入策略失控，正在制造 distractor |
| `use_count == 0` 的记忆占比 | < 30% | 高 = 写入判据太松 |
| supersede 率（新增中带 supersedes 的比例） | 稳定 | 突增 = 抽取器退化 或 正在被投毒 |
| compaction 后的 cache 重建成本 / 节省 | > 1 | < 1 = 压得太频繁，见 §7 的账 |

---

## 面试官会追问

1. "给 Agent 加记忆"这个需求，你会先问哪几个问题？四类状态你分别放在哪？
2. 用户在手机上说"记住 A"，在电脑上说"记住 not A"，你的系统怎么处理？
3. 记忆放在 system prompt 里和放在 messages 末尾，成本差多少？为什么？
4. 用户行使删除权，你要删哪些东西？有哪些你删不掉，要怎么向用户交代？
5. Compaction 什么时候会让你更贵而不是更便宜？给我一个具体的算式。
6. 一个 Agent 读了一封含恶意指令的邮件，把它"记住"了。三个月后这条记忆还在生效。你的系统在哪一层应该拦住它？拦不住的话怎么发现和回滚？
7. checkpoint 里你会存什么、不存什么？恢复时怎么保证不重复扣款？
8. 什么情况下你会建议这个产品**不要**做记忆功能？

---

**下一篇** → [05-multi-agent-orchestration.md](05-multi-agent-orchestration.md)
