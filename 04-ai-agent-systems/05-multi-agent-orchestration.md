# 05 · 多 Agent 编排（multi-agent orchestration）

> 2025 年业界为"要不要多 Agent"吵了一年，2026 年收敛成一条可执行的结论：
> **并行只用于"读"和"评"，"写"必须单线程。**
> 另外两个数字你必须记住：多 Agent 的 token 消耗约是单 Agent 的 **3.7 倍**（相对纯聊天是 **15 倍**），
> 而实测失效责任 **67.7% 在 orchestrator 自己身上**，不在 worker。

---

## 1. 先算账：多 Agent 贵在哪

任何多 Agent 讨论必须从这张表开始，否则就是在做没有成本约束的架构幻想。

| 形态 | 相对纯 chat 的 token 倍数 | 来源 |
|---|---|---|
| 单次 chat 补全 | 1× | 基线 |
| **单 Agent（带工具循环）** | ≈ **4×** | Anthropic，2025-06-13 [一手] |
| **多 Agent 系统** | ≈ **15×** | 同上 |
| Agent teams（多会话协作） | ≈ **7×** 相对单会话 | 官方文档，2026 |

即：**多 Agent ≈ 单 Agent 的 3.7 倍**。按 2026 年中的量级换算一下（价格随时变动）：

```
一次中等复杂度的研究任务，单 Agent 约 200K 输入 + 20K 输出（Opus 5 档，$5/M in、$25/M out）
  单 Agent：  200K × $5/M  + 20K × $25/M  = $1.00 + $0.50 = $1.50
  多 Agent：  ×3.7                                        ≈ $5.55
  开缓存后（缓存读 = 输入价 10%，命中 80%）：
  单 Agent 输入降到 200K × (0.2×$5 + 0.8×$0.5)/M = $0.28  → 总 $0.78
  多 Agent 的缓存命中率通常更低（每个 worker 是全新上下文，前缀不共享）
```

**注意最后一行**：多 Agent 不只是 token 多，它还**系统性地降低 prefix cache 命中率**——每个 worker 的上下文是全新构造的，父的前缀在子那里一点也用不上。所以真实成本倍数常常高于 3.7×。

**同时，token 用量也是收益的主要解释变量**：Anthropic 报告 orchestrator-worker（Opus 4 lead + Sonnet 4 subagents）相对单 Agent Opus 4 提升 **+90.2%**（内部研究评测），而在 BrowseComp 上，**仅 token 用量一项就解释了 80% 的性能方差**。

> **这两句话必须一起说**：多 Agent 的收益很大程度上来自"花了更多 token"，而不是来自"多个 Agent 这个结构本身"。所以在你决定上多 Agent 之前，先问：**同样的预算，让单 Agent 多跑几轮、多采样几次，是不是就够了？**

---

## 2. 编排模式全景

| 模式 | 拓扑 | 相对单 Agent 的成本倍数 | 适用判据 | 撞墙条件 / 失效信号 |
|---|---|---|---|---|
| **Orchestrator-Worker** | 1 lead → N worker（**同步**执行），worker 回传摘要 | ~2–4× | **广度优先（breadth-first）的只读探索（read-only exploration）**：多来源检索、竞品调研、代码库全局理解 | lead 上下文 ~6 步后熵饱和（entropy saturation）→ 决策退化；一个慢 worker 阻塞全局；worker 重复劳动 |
| **Map-Reduce 扇出（fan-out）** | 同构任务 N 份并行 → 单点 reduce | ~N× 计算，但墙钟时间（wall-clock time） −最多 90% | 任务**完全同构且互不依赖**：批量文档抽取、批量文件分类 | reduce 阶段上下文爆炸；分片边界（shard boundary）跨越了语义边界 |
| **Planner-Executor** | 先出计划（固化）→ 按计划执行 | ~1.3–1.6× | 步骤可预先枚举、需要人审计划、需要抗注入（计划固化后工具调用序列不可被改写） | 计划过时（环境在执行中变了）；**参数仍可被注入（prompt injection）影响**，只有序列不可变 |
| **Handoff / Swarm** | Agent A 把控制权连同上下文转交 Agent B | ~1.2–2× | 明确的**角色切换**：分诊（triage）→ 专科；意图识别 → 领域 Agent | 无结构的 swarm 会形成环；转交时上下文丢失导致"重新问一遍" |
| **反思（self-reflection）/ 批评者（critic）**（Reflexion、Code-Review-Loop） | 生成 Agent ⟷ 批评 Agent，多轮 | **~2.3×**（F1 0.943，相对 sequential pipeline 基线） | 输出质量比成本重要；**有客观验收信号**（测试、编译、schema） | **无外部反馈的纯自反思会退化** —— 模型常把对的改错（degeneration-of-thought） |
| **辩论（debate）/ 投票** | N 个 Agent 互相说服，或独立采样后多数投票（majority voting） | ~N× | 只在**没有确定性验收器（deterministic verifier）**时考虑 | 同等预算下，辩论**不可靠地优于简单 self-consistency 多数投票**。有验收器时应直接用验收器 |
| **分层（hierarchical）supervisor-worker** | 多层 orchestrator | **~1.4×**（F1 0.921） | 任务域天然分层（部门 → 团队 → 个人） | 层数 > 3 时归因和调试成本爆炸；深度默认上限就是 3 |
| **Hybrid（路由 + 缓存 + 自适应重试）** | 单 Agent 主干 + 按需升级 | **~1.15×，拿回 reflexive 收益的 89%** | **成本敏感的生产系统的默认选择** | 路由器（router）本身成为质量瓶颈 |
| **编排脚本化（orchestration as code）**（Dynamic Workflows 形态） | 模型**写一段编排脚本**，运行时在会话外执行 | 取决于扇出，但**中间结果不进上下文** | 扇出大、中间结果体积大 | 运行中不能插入用户输入；恢复语义有坑（见 §8） |

⚠ 成本倍数的基线不统一：**2.3× / 1.4× / 1.15×** 三个数来自同一项研究（[arXiv:2603.22651](https://arxiv.org/abs/2603.22651)，10,000 份 SEC 文件 / 25 类字段 / 5 个模型，2026-03-24），基线是 **sequential pipeline**；**4× / 15× / 7×** 来自 Anthropic 口径，基线是**纯 chat**。**不可混用，引用时必须带基线。**

---

## 3. 拓扑与上下文流

```
═══ A · Orchestrator-Worker（只读探索）═══════════════════════════════

  用户 query
      │
      ▼
 ┌──────────────────────────────────────┐  ctx：完整会话历史
 │  Lead / Orchestrator                  │  ⚠ ~6 步后熵饱和，决策质量掉档
 │  强模型，但**低 reasoning effort**     │  ⚠ 中间结果绝不能落在这个窗口里
 └──┬───────────┬───────────┬───────────┘
    │           │           │   父→子唯一通道 = 一段自包含的 prompt 字符串
    │           │           │   （父的历史、工具结果、system prompt 都不传）
    ▼           ▼           ▼
 ┌───────┐  ┌───────┐  ┌───────┐   每个 worker：全新隔离上下文
 │Worker1│  │Worker2│  │Worker3│   10–15 次工具调用，**只读工具**
 │ ctx∅  │  │ ctx∅  │  │ ctx∅  │   maxTurns 必填，完成判据写死在 prompt 里
 └───┬───┘  └───┬───┘  └───┬───┘
     │1–2K      │1–2K      │1–2K token 摘要（有损；父无法复审原始证据）
     │          │          │  ⇒ worker 必须把原始证据写外部存储，摘要里带引用路径
     ▼          ▼          ▼
 ┌──────────────────────────────────────┐
 │  Lead 综合 → 单写者落盘（single-writer）│  ← 写永远单线程
 └──────────────────────────────────────┘

═══ B · 写路径的正确形态（Cognition 2026 收敛结论）═══════════════════

  ┌───────────┐   建议/评审（只输出意见，不动手）   ┌───────────────┐
  │ Critic /  │ ─────────────────────────────────► │  唯一 Writer   │
  │ Smart     │ ◄───────────────────────────────── │  （单线程）    │
  │ Friend    │        代码 / diff / 上下文         └───────┬───────┘
  └───────────┘                                            │
     ⚠ 双方都必须在前沿能力档位，                            ▼
       能力不对称时弱模型不知道何时该求助               确定性验收器
                                                    （测试 / 编译 / schema）
   ⚠ Code-Review-Loop 的实测：审查 Agent 与编码 Agent
     **上下文完全隔离**时效果最好（≈ 每 PR 抓 2 个 bug，其中 ~58% 严重）

═══ C · Map-Reduce（同构扇出）═══════════════════════════════════════

   分片器（确定性代码，不是 LLM）
        │  shard_1 … shard_N，边界必须对齐语义单元（文件/文档/租户）
   ┌────┼────┬────┬────┐
   ▼    ▼    ▼    ▼    ▼
  [A1] [A2] [A3] …  [An]      并发上限 16；总量上限 1000
   └────┴────┴────┴────┘
        │  结构化输出（强制 JSON schema，不是自然语言）
        ▼
   Reduce（**优先用代码**；只有需要判断时才用 LLM）
```

---

## 4. 上下文如何跨 Agent 传递

这是多 Agent 里唯一真正的技术难点。三种传递方式，信息损失（information loss）和成本正相关：

| 传递方式 | 信息保真（fidelity） | 成本 | 一手契约参考 | 适用角色 |
|---|---|---|---|---|
| **完整 trace 继承（trace inheritance）** | 高 | 父上下文 × N，且 worker 无法享受自己的前缀缓存 | Cognition 2025-06 原则①：共享**完整 agent trace 而非单条消息** | **继承派**：延续同一条决策线的角色（接力执行、深挖某个分支） |
| **自包含 prompt 字符串（self-contained prompt）** | 中（取决于你写得多全） | 低 | Claude Agent SDK subagent：子 Agent 上下文**全新隔离**，父→子唯一通道是 Agent tool 的 prompt 字符串；**父只拿到子的最后一条消息** | 多数场景的默认 |
| **结构化产出交换（structured output exchange）** | 低但**可验证** | 最低 | 强制 JSON schema，字段级校验（field-level validation） | **新生派**：需要独立判断的角色（评审、验证、投票） |

**最重要的一条设计规则，且它是有争议的**：

- Cognition 2025-06-12 主张共享完整 trace；同一作者 2026-04-22 又报告 Code-Review-Loop 里审查 Agent 与编码 Agent **完全干净、互不共享上下文**时效果最好。**两篇没有互相调和。**
- 我的调和判据：**执行链共享上下文，验证链隔离上下文。** 一个角色如果要**延续**前面的决策，它需要继承；一个角色如果要**独立判断**前面的决策对不对，继承就是污染（context pollution）——它会被前面的推理说服。

**信息损失是设计意图，不是缺陷**：子 Agent 可能消耗数万 token 探索，只回传 **1,000–2,000 token** 的浓缩摘要（condensed summary），完整探索上下文被丢弃、不污染父上下文（[Anthropic 一手口径](https://www.anthropic.com/engineering/multi-agent-research-system)）。代价明确：**父 Agent 无法复审原始证据**。

对策不是"回传更多"，而是：**子 Agent 把原始证据写入外部存储（对象存储 / 文件 / DB），摘要里只带引用路径与 hash**。父需要复审时按路径去取。这同时解决了摘要失真和上下文膨胀。

⚠ **子 Agent 的输出是不可信输入（untrusted input）。** 生产 harness 已经在做这件事：扫描子 Agent 最终消息中的"指令形状"——伪造 `<system-reminder>` 控制标签就地转义、伪造 `Human:` / `Assistant:` 轮次标记加反斜杠、提及权限配置时加标记行（**只标记不删改**）。**自建编排必须自己补这一层**：子 Agent 返回值不能直接拼进父的 system prompt 区域，跨 Agent 消息不能构成权限授予（privilege grant）。

---

## 5. 什么时候不要用多 Agent

### 两派原始论据（不要选边，它们讨论的是不同象限）

| | Anthropic（2025-06-13） | Cognition（2025-06-12《Don't Build Multi-Agents》） |
|---|---|---|
| 结论 | orchestrator-worker **+90.2%** | 别建多 Agent |
| 象限 | **广度优先的只读研究** | **有依赖的编码写入** |
| 自承的反例 | "**大多数编码任务没有足够的可并行工作量**"；需要共享同一上下文 / 高依赖协调的场景不适合 | Flappy Bird 例子：两个 subagent 分别造出 Mario 风格背景和不匹配的小鸟 |
| 核心原则 | 并行探索 + 单点综合 | ①共享上下文 ②**行动隐含决策，冲突的决策产生糟糕结果** |

[LangChain 的调和点](https://www.langchain.com/blog/how-and-when-to-build-multi-agent-systems)（2025-06-16）最简洁：**读操作天然比写操作可并行**；Anthropic 的系统正是"多 Agent 做读、单 Agent 一次调用做最终综合"。Cognition 十个月后的修正（2026-04-22）把结论改成 **"only one writer, only intelligence is bundled"** —— 额外的 Agent 贡献智能而不是动作。

### 我的四问判据（面试里直接用这四个问题回应"要不要多 Agent"）

1. **子任务之间有隐含的决策依赖（decision dependency）吗？** 两个子任务的产出需要"风格一致""接口一致""假设一致"→ **有依赖 → 不要并行**。这类一致性无法靠 prompt 约定实现，只能靠共享上下文，而共享上下文正是多 Agent 想省的东西。
2. **是读还是写？** 写 → 单线程。这是 2026 年的共识，没有例外条款。
3. **子任务边界能用一段自包含的 prompt 完整描述吗？** 必须包含：文件路径、错误原文、已经定下的决策、**完成判定标准（completion criteria）**、**输出 schema**、**最大轮数（max turns）**。写不出来 → 说明边界还没切清楚，先别拆。MAST 里"步骤重复"占 **15.7%**，根因就是任务边界模糊导致 subagent 执行完全相同的搜索。
4. **值不值？** 3.7× 的 token × 你的单价 vs 预期收益。如果同样的预算给单 Agent 多采样几次 + 一个确定性验收器就能达到，**就不要上多 Agent**。

### 明确不要用的四种情况

| 情况 | 为什么 |
|---|---|
| **强依赖的顺序任务** | 每一步的输入是上一步的输出。并行度为 1 时，多 Agent 只增加了摘要损失和协调开销 |
| **需要共享细粒度上下文** | 子 Agent 拿不到父的历史。你会看到"重新问一遍"和"假设不一致" |
| **子任务边界不清晰** | 见判据 3。边界模糊 → 重复劳动 + 冲突写入 |
| **你还没有轨迹级（trajectory-level）eval** | 多 Agent 的失效是隐蔽的（轨迹脏但答案对、答案对但花了 10 倍钱）。**outcome-only eval 对多 Agent 是全盲的** |

> **面试金句**：
> "我不会先问'要不要多 Agent'，我会先问'哪些子任务是只读的'。只读的部分可以扇出，因为读之间没有冲突；写的部分必须收敛到单一 writer，因为**行动隐含决策，两个并行的写者会做出互相冲突的决策，而这种冲突在合并时才暴露，那时已经晚了**。所以我的默认架构是：N 个只读 worker 并行探索 → 一个 lead 综合 → 一个 writer 单线程落盘 → 一个确定性验收器（跑测试）判定。额外的 Agent 只贡献判断，不贡献动作。"

---

## 6. 失效模式分类

### 学术分类法（failure taxonomy）：MAST

[MAST（arXiv:2503.13657）](https://arxiv.org/abs/2503.13657)，1600+ 条标注 trace、7 个框架、标注者一致性（inter-annotator agreement）κ=0.88，把多 Agent 失效分成三大类：

| 大类 | 占比 | 细分 top 项 |
|---|---|---|
| **① 系统设计 / 规格不清** | **43.8%** | 步骤重复（step repetition） 15.7%、违反任务规格（spec violation） 11.8%、不知道终止条件（termination condition） 12.4%、不主动澄清 6.8% |
| **② Agent 间错位** | **32.25%** | 推理-行动不一致（reasoning-action mismatch） 13.2%、任务跑偏（task derailment） 7.4% |
| **③ 任务验证缺失** | **23.5%** | 错误验证 9.10%、无/不完整验证 8.20%、过早终止（premature termination） 6.20% |

同一研究的干预实验给出了性价比排序：**只调 workflow（加一个审批环节）→ +9.4% 成功率；加高层验证步骤 → +15.6%**。也就是说，**在改模型、改 prompt 之前，先加验证步骤**。

### ⚠ 但不要直接照搬到你的系统

生产遥测（639,381 执行步 / 23,624 次运行 / 5 个月）显示：**Agent 间协调类失效在 single-agent-per-cycle 系统里"几乎不存在"**，而在 MAST benchmark 里占 32%。生产中最常见的是「任务未完成」和「验证缺失」。**先确认你的系统真的是多 Agent，再套 MAST。**

同一份遥测的两个数字值得记：失败率从 ~14%（2–3 月）降到 **0.4%**（5 月）而运行量同期涨 ~4×；**14.2% 的总花费花在失败的运行上**——这是一个可以直接拿去和财务谈的优化目标。

### 责任在 orchestrator，不在 worker

[ICML 2026 的实证（arXiv:2606.01351）](https://arxiv.org/abs/2606.01351)：

- **orchestrator 承担 67.7% 的失败责任**（executor 32.3%）；Agentic RAG 场景 orchestrator 高至 **79.0%**，Web Browser 场景 executor 才升到 36.2%。
- **orchestrator 的"认知视界"约 6 步后熵饱和**；实测某模型 Step1 → Step2 的成功率从 **92% 掉到 22%**；不同模型的任务成功率区间 **7.43% ~ 44.14%**。
- **反直觉发现**：抑制过度思考后 Step Success Rate 从 **0.251 → 0.302**。

三条直接可执行的推论：

1. **orchestrator 的上下文必须主动裁剪**，中间结果不能落在它的窗口里（这正是"编排脚本化"把中间结果放在脚本变量里的动机）。
2. **orchestrator 不要用重推理模型 / 高 reasoning effort**，把深推理预算留给 worker。这和大多数人的直觉相反。
3. **超过 ~6 步的编排，切成分层或改成脚本化**，不要让一个 orchestrator 线性走 20 步。

---

## 7. 并行写冲突与隔离

**实证基线**：AI 编码 Agent 的 PR 合并冲突率（merge conflict rate）**27.67%**（[AgenticFlict, arXiv:2604.03551](https://arxiv.org/abs/2604.03551)，107,000+ 次完成的合并模拟中 29,000+ 冲突，142,000+ Agent PR / 59,000+ 仓库）。这不是理论风险。

四种隔离手段，按强度递增：

| 手段 | 做法 | 解决什么 | **不解决什么** |
|---|---|---|---|
| **分文件所有权** | 拆分工作，每个 Agent 拥有不同的文件集 | 直接覆盖 | 语义冲突（semantic conflict，改了同一个接口的两端） |
| **文件锁（file lock）/ 共享任务列表** | 共享任务清单 + 锁，防竞态（race condition） | 并发认领同一任务 | 长时持锁导致的串行化（serialization） |
| **git worktree 隔离** | 每个 Agent 一个独立工作区与分支（`.claude/worktrees/<name>/`，分支 `worktree-<name>`，运行期自动 `git worktree lock`） | **工作区级文件隔离** | **任务分解、依赖跟踪、语义冲突、合并选择** —— 冲突被推迟到 merge time 由 git 检出，好处是不再静默发生 |
| **单写者 + 合并裁决 Agent（merge arbiter）** | 并行只产出 patch/建议，由唯一 writer 或裁决 Agent 决定合入 | 语义冲突 | 吞吐（这是有意的） |

**注意 worktree 的隐藏坑**：`.env` 这类被 gitignore 的文件默认不会进新工作区，需要显式声明（如 `.worktreeinclude`）复制进去，否则 Agent 在新 worktree 里跑不起来，而失败信息看起来像业务 bug。

**结论**：worktree 是必要不充分条件。**真正解决冲突的是"只有一个 writer"这条架构约束**，worktree 只是让并行探索不互相踩脚。

---

## 8. 预算、并发与深度控制

失控扇出（runaway fan-out）是多 Agent 最贵的事故形态——Anthropic 观察到简单 query 被拉起 **50+ 个 subagent**。产品化的对策是三层同时设限，**缺任何一层都会被穿透**：

```
第 1 层 · 结构上限（防止拓扑爆炸）
  ├ 并发上限        16（Dynamic Workflows 硬上限，低核机器更少）
  ├ 单次运行总量    1,000 个 Agent
  ├ 嵌套深度        默认 3 层（`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`，设 1 关闭）
  └ workflow 规模   small <5 / medium <15（默认）/ large <50 / unrestricted

第 2 层 · 预警阈值（防止悄悄变贵）
  └ >25 个 Agent  或  预计 token >150 万  → 提示（不拦截）

第 3 层 · 组织级硬闸（防止跑飞的 workflow 吃掉团队配额）
  ├ 组织 spend limit / workspace rate limit
  ├ 单会话 / 单任务的**绝对 token 上限**（Agent 循环的方差比人类会话大一个数量级）
  └ 每个 subagent 的 `maxTurns` —— **必填，不是可选**
```

**扇出配额（fan-out quota，可直接抄的一手口径）**：

| 任务类型 | 配额 |
|---|---|
| 简单事实查找 | **1 个 Agent / 3–10 次工具调用** |
| 直接对比 | **2–4 个 subagent / 每个 10–15 次调用** |
| 复杂研究 | **10+ subagent** |
| 协作型 team | **3–5 个 teammate，每人 5–6 个任务**；teammate 用便宜模型（Sonnet 档）而非 lead 的模型 |

并行化对复杂 query 最多能减少 **90%** 的研究耗时——但注意 **lead 是同步执行 subagent 的，一个慢 subagent 会阻塞整个系统**。异步执行能进一步并行，代价是协调复杂度。

### 恢复语义（resume semantics）：一个真实的坑

Dynamic Workflows 的恢复是**按 Agent 启动顺序回放（replay），缓存命中止于第一个未完成的 Agent，此后启动的全部重跑，哪怕它们已经完成**。A/B/C/D 依次启动、停在 B 运行中 → 恢复时只有 A 命中，**B/C/D 全部重跑**。另外恢复**仅限同一会话**，退出 CLI 后下次会话从零开始。

**工程推论有两条**：
1. **把工作切成很多小 Agent 比一个长 Agent 保住更多进度。**
2. 并行度越高，"停在中间"的重跑代价越大 —— 这是"并行 = 更快"的一个反例。

同时注意运行时约束：脚本本身**无文件系统 / shell 访问**（只有 Agent 有）；运行中**不能插入用户输入**（只有权限提示能暂停）；生成的 subagent 一律以 `acceptEdits` 模式运行并继承你的工具 allowlist ⇒ **长跑前必须把需要的命令预先加入 allowlist**，否则会在半路卡死。

---

## 9. 结果聚合与冲突消解

**第一原则：能用确定性裁决（deterministic arbitration）就不要让 Agent 互相说服。**

| 裁决方式 | 何时用 | 证据 |
|---|---|---|
| **确定性验收器**（测试、编译、schema 校验、linter） | **只要能构造出来，就永远优先** | MAST 干预实验：加高层验证步骤 +15.6% |
| **多路采样 + 多数投票**（self-consistency） | 无验收器、答案可比较 | 同等响应预算下，多 Agent 辩论**不可靠地优于**简单多数投票（[arXiv:2508.17536](https://arxiv.org/abs/2508.17536)、[arXiv:2601.19921](https://arxiv.org/abs/2601.19921)） |
| **单一裁决 Agent** | 需要语义判断的合并（多个 patch 合入） | — |
| **Agent 辩论** | 最后选择 | 成本 N×，收益不稳定 |
| **纯自反思（无外部反馈）** | **不要用** | 会退化：模型常把对的改错（[arXiv:2310.01798](https://arxiv.org/abs/2310.01798)） |

**聚合（aggregation）的三条工程规则：**

1. **子 Agent 的输出必须是结构化的**（JSON schema 强制），不是自然语言。自然语言聚合会把 reduce 阶段变成第二次推理，成本和错误都翻倍。
2. **reduce 优先用代码**。只有确实需要语义判断的部分才交给 LLM，且要限定它只做判断不做重写。
3. **冲突要显式表达而不是静默取胜（silent overwrite）**。聚合结果里保留 `conflicts: [{field, candidates, resolved_by}]`，让人能审。多 Agent 系统里最贵的 bug 是"两个 worker 给了矛盾结论，lead 随便选了一个，没人知道"。

---

## 10. 可观测（observability）：父子 span 与归因（attribution）

### span 结构

```
workflow  (gen_ai.workflow.duration)
└── invoke_agent  lead            (gen_ai.invoke_agent.duration
    │                              / .inference_calls / .tool_calls)
    ├── chat  <model>             (gen_ai.client.token.usage：
    │                               input / output / cache_read / cache_creation
    │                               / reasoning.output_tokens)
    ├── invoke_agent  worker_1
    │   ├── execute_tool  <name>  (gen_ai.execute_tool.duration)
    │   └── chat  <model>
    ├── invoke_agent  worker_2 …
    └── chat  <model>  (综合)
```

span 命名规则（OTel GenAI 约定）：推理/嵌入 `{gen_ai.operation.name} {gen_ai.request.model}`；工具 `execute_tool {gen_ai.tool.name}`。

⚠ **状态警告**：OTel GenAI 语义约定（semantic conventions）**全部条目仍是 Development，0 项 GA**（2026-07-30 一手核实），且已在 2026-06 迁到独立仓库、**尚无 release/tag ⇒ 拿不到可 pin 的版本化 schema URL**。主 semconv 仓库 v1.42.0 弃用全部 `gen_ai.*`、v1.43.0 完全移除。**要接就用兼容开关 `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` 双发，别把属性名硬编码进告警规则。**

另外两个必须知道的空白：**官方命名空间里没有 cost 属性，也没有 tenant / user 属性**。多 Agent 的成本归因（cost attribution）是你自己要补的扩展，跨平台不保证被识别。

### 归因：要回答的四个问题

| 问题 | 靠什么回答 |
|---|---|
| 这次运行花了多少钱，花在哪个 Agent 上？ | 按 `invoke_agent` span 聚合 token（含 `cache_read` / `cache_creation` 分拆）。**当 provider 同时报 used 与 billable token 时，必须上报 billable** |
| 失败责任在 orchestrator 还是 worker？ | 轨迹级判定：worker 都成功但最终答案错 → orchestrator。参考基线：orchestrator 67.7% |
| 有没有重复劳动？ | 确定性 evaluator：跨 worker 的工具调用参数去重率 |
| 有没有死循环（infinite loop）/ 过早终止？ | step count 对预算、必经步骤是否出现、工具调用序列匹配 |

**关键做法：可判定的轨迹指标用代码 evaluator，不要用 LLM judge。** 布尔轨迹分数在每条采样 trace 上计算几乎是免费的，适合直接做告警信号；只有语义层（"这一步选这个工具合不合理"）才交给 judge。轨迹匹配可以做成四档：`strict`（完全一致同序）/ `unordered`（同一组工具，顺序不限，适合并行取数）/ `subset`（不许多调）/ `superset`（至少覆盖参考轨迹）。

⚠ **最后一条，也是最容易被忽略的**：生产遥测（production telemetry）里一个基础设施 bug 曾在两周窗口内把失败率**虚高约 2.5×**，同时影响 15 个 Agent，伪装成"终止挂起"。**先信数据管道，再信数据。**

---

## 11. 反模式清单（anti-patterns）

1. **"Anthropic 说好、Cognition 说不好，二选一"** —— 两边讨论的是不同象限（只读研究 vs 有依赖的写入），2026 已收敛为"读可并行、写必单线程"。
2. **"共享完整 trace 永远更好"** —— 执行链共享、验证链隔离。一刀切两边都错。
3. **"并行 = 更快"** —— lead 同步执行 subagent，一个慢 worker 阻塞全局；恢复语义还会让并行度放大重跑代价。
4. **"多 Agent 辩论能提升准确率"** —— 同等预算下不可靠地优于多数投票；纯自反思会退化。**有验收器就用验收器。**
5. **"worktree 解决了并行冲突"** —— 只解决工作区级文件隔离，不解决语义冲突。
6. **orchestrator 用最强模型 + 最高 reasoning effort** —— 实证反了。抑制过度思考反而提升 step 成功率；深推理预算应该给 worker。
7. **subagent 的 prompt 不写完成判据 / 输出 schema / `maxTurns`** —— MAST 里"不知道终止条件 + 过早终止 + 无验证"合计约 36%，是生产环境的第一梯队失效。
8. **只设并发上限，不设组织级 spend limit** —— 一次跑飞的 workflow 能吃掉团队当天的 TPM 配额。三层预算缺一不可。
9. **把子 Agent 的返回值直接拼进父的 system prompt** —— 子 Agent 输出是不可信输入。伪造 `<system-reminder>`、伪造轮次标记、声称"已获授权"都是已知形态。
10. **把学术失效分布直接当自己的分布** —— 先确认你真的是多 Agent 系统。
11. **拿 pre-production 案例当生产能力证据** —— 旗舰案例（如 ~750,000 行 / 11 天 / 测试 99.8% 通过的大规模重写）**官方明确标注 pre-production**。
12. **把实验特性当 GA 用** —— 例如 Agent teams 默认关闭、官方标 experimental，已知限制包括不支持 `/resume` `/rewind`、一会话仅一个 team、不支持嵌套、lead 不可转移、权限在 spawn 时固定。**上生产前先查当前版本的能力矩阵（capability matrix）。**
13. **没有轨迹级 eval 就上多 Agent** —— outcome-only eval 看不见错工具、参数畸形、死循环、10 倍成本。

---

## 面试官会追问

1. 我要做一个"深度研究"功能，你会用单 Agent 还是多 Agent？你的判断依据是什么，成本差多少？
2. 多 Agent 比单 Agent 好在哪？有没有可能它的收益只是因为花了更多 token？
3. 父 Agent 怎么把上下文给子 Agent？三种方式各丢了什么信息？你怎么选？
4. 三个 worker 并行改同一个仓库，你怎么防冲突？worktree 够吗？
5. 一个 workflow 跑飞了，拉起 50 个 subagent 把配额吃光。你的系统有几道闸？分别在哪一层？
6. 多 Agent 系统失败了，你怎么判断是 orchestrator 的问题还是 worker 的问题？
7. 长跑的 workflow 中途崩了，恢复时哪些 Agent 会重跑？这对你怎么切分任务有什么影响？
8. 你怎么给多 Agent 系统做可观测？哪些指标用代码算，哪些用 LLM judge？

---

**下一篇** → [06-evaluation-and-observability.md](06-evaluation-and-observability.md)
