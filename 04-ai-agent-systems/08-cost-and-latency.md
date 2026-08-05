# 08 · 成本与延迟优化

> 当 Agent 每轮都重发一份**只追加、不压缩、未命中缓存**的完整历史时，累计输入里会出现随轮次近似二次增长的部分。
> 有前缀缓存、滑动窗口、摘要或服务端状态后，曲线会改变。因此优化的第一步不是背“缓存优先”或“换模型优先”，而是先按一次成功任务拆账单和延迟，再动最大的可控项。

---

## 读这一章之前

**你在工作中遇到过这个**

下面是一个**说明归因方法的假想案例**，不是行业成本结构：你们的 coding agent 上个月账单 $40k，老板让你砍一半。
你花两周把沙箱换成更便宜的机型、把闲置容器回收从 10 分钟压到 2 分钟、给对象存储加了生命周期策略 ——
账单只降了 **1.4%**。这份样本账单里，沙箱约占 12%，推理约占 85%；推理输入中，大部分又来自每轮重发的历史。
你两周的工作从一开始就动错了地方，而这件事在你写第一行代码之前用一张纸就能算出来。

**需要先懂的概念**

| 概念 | 一句话 | 详见 |
|---|---|---|
| 延迟 / 吞吐 / 并发 | 单次耗时 / 单位时间处理量 / 此刻在途的请求数 | [00/00 §2](../00-foundations/00-concepts.md#2-延迟吞吐并发--三个最常被混淆的词) |
| 分位数与长尾 | p50/p99 描述分布形状；少数极端样本能主导总量 | [00/00 §3](../00-foundations/00-concepts.md#3-为什么平均值是骗人的p50--p90--p99) |
| Prefill / Decode 与 TTFT / TPOT | 处理输入和逐 token 生成是两台特性相反的机器，各有各的延迟指标 | [01 §1](01-llm-serving-infra.md#1-两个阶段本质上是两台不同的机器) |
| 前缀缓存 | 请求开头满足供应商/引擎复用条件时，可以复用先前计算；命中条件、折扣与 TTL 因模型和平台而异 | [01 §6](01-llm-serving-infra.md#6-前缀缓存与缓存感知路由网关设计的真正约束) |
| Compaction | 会话太长时把历史压成摘要，腾出窗口 | [04 §7](04-agent-memory-and-state.md#7-compaction触发实现以及它和前缀缓存prefix-caching的正面冲突) |
| 多 Agent 的成本放大 | 额外角色会重复上下文、规划与汇总；倍数由拓扑、轮次、共享状态和缓存决定，必须实测 | [05 §1](05-multi-agent-orchestration.md#1-先算账多-agent-贵在哪) |

**这一章要回答的问题**

1. 哪些上下文策略会让累计输入出现**二次项**，哪些策略会把它压回近线性或分段增长？
2. 一次运行的钱到底花在哪几项上？哪一项值得动，哪一项动了也白动？
3. 某些供应商/TTL 的当前示例里，缓存写价是输入价的 1.25×–2×、读价约 0.1×；代入你的实际价目后要复用几次才回本？
4. 我上线了上下文压缩，账单为什么反而涨了 10%？
5. 自建 GPU 到底什么时候划算？决定这件事的是 GPU 价格还是别的东西？

**本章新引入的术语**

| 术语 | English | 一句话定义 |
|---|---|---|
| 轮次 | turn | 模型一次"读完整个上下文 → 输出 → 调工具 → 拿到结果"的完整往返；一次运行由 N 个轮次串成 |
| 二次增长 | quadratic growth | 在“历史只追加且每轮完整重发”的模型里，第 n 轮包含前 n−1 轮增量，累计输入因而出现平方项；不是所有 Agent 的固有规律 |
| 字节稳定前缀 | byte-stable prefix | 每一轮请求开头那一段内容**逐字节完全相同**，一个字符都不差 —— 只有这样缓存才命中 |
| 缓存写 / 缓存读 | cache write / cache read | 第一次保存前缀与后续命中的计费项；1.25×–2× 写价和约 0.1× 读价是本文供应商快照，不是统一行业规则，计算前要查目标模型、地区与 TTL |
| 缓存断点 | cache breakpoint | 在 prompt 里显式标出"到这里为止的内容请缓存下来"的标记 |
| 静默缓存未命中 | silent cache miss | 缓存没命中通常不会报业务错误；应从 API usage 中的缓存读/写 token、命中率、TTFT 和账单变化发现，而不是等用户报错 |
| 级联失效 | cascading invalidation | 改了靠前的一段内容，它**后面**的全部缓存断点一起作废 |
| 模型路由 / 级联 | model routing / cascade | 先判断难度再挑模型（并行一次决策）/ 先用便宜模型跑、判断不行再升级给贵模型（串行两次） |
| 升级率 | escalation rate | 级联里被判定为"便宜模型搞不定、必须交给贵模型"的请求比例 |
| 隐性倍数 | hidden-cost multiplier | 自建推理的真实成本相对**裸 GPU 租金**的倍数：专职工程、冗余副本、闲置时段、可观测性栈 |
| 利用率 | utilization | GPU 真正在算的时间占你持有它的时间的比例；单位成本 ≈ 固定成本 ÷ 利用率 |
| 重尾分布 | heavy-tailed | 少数极端样本贡献了大部分总量，因此均值既不代表典型用户，也不代表最坏用户 |

---

## 1. 先建模：成本与延迟的分解公式

```
C_run = C_infer + C_sandbox + C_tools + C_storage

C_infer = Σ_turns [ T_fresh × P_in                # 未缓存直读
                  + T_cw    × P_in × W            # W = 目标平台的缓存写乘数
                  + T_cr    × P_in × R            # R = 目标平台的缓存读乘数
                  + T_out   × P_out ]
C_sandbox = billable_units × price_per_unit        # 单位可能是 running time、vCPU/内存秒或 session
C_tools   = web_search 次数 × 单价 + 外部 API
C_storage = 向量库 + 对象存储 + 平台可能收取的缓存持有费
```

若会话历史只追加且每轮完整重发，`Σ_turns T_in` 中会出现二次项：

```
累计输入 token ≈ N·L₀ + Δ·N(N−1)/2
   N = 轮次   L₀ = 稳定前缀（系统提示 + 工具定义）   Δ = 每轮上下文增量
```

这个公式是**诊断模型，不是事实声明**：`Δ`、输入/输出比和缓存重叠率都要从你自己的 trace / usage 统计。工具结果很大、轮次很多时输入可能主导；短问答、长生成或强压缩会得到完全不同的成本结构。若历史被裁剪、压缩或服务端保存，按每个阶段分别建模，不能继续套同一个平方项。

**延迟分解**：若 TTFT 按“客户端发出请求到首 token”定义，它已经包含模型侧排队，则 `L_e2e = Σ_turns [TTFT + (T_out−1) × TPOT + Tool_time] + Orchestration`。若能单独观测排队，也可写成 `TTFT_service ≈ prefill + RTT`，再用 `Queue + TTFT_service`；两种口径不要重复相加。

（**TTFT** = time to first token，从发出请求到收到第一个 token 的时间；**TPOT** = time per output token，第一个 token 之后每多吐一个 token 的间隔。**prefill** = 把整段输入一次性算完那个阶段，它算力受限；**decode** = 之后逐个 token 生成，它访存受限。两者是两台特性相反的机器，见 [01 §1](01-llm-serving-infra.md#1-两个阶段本质上是两台不同的机器)。）

| 指标 | 交互式目标 | 依据 |
|---|---|---|
| TTFT | 可从 <1 s（理想 <600 ms）做交互式起点 | 目标随 UI 是否流式、prompt 长度、地区和模型变化，应由用户研究与基线校准 |
| TPOT | 可从 ≤20 ms（≈50 tok/s）做流式文本起点 | 代码、语音和后台任务需求不同；同时报告 mean 与高分位 |
| 端到端 | 轮次 × 单轮 | 15 轮 × 3 s = 45 s，**用户体验的是 45 s，不是 TTFT** |

⚠️ **只看一个延迟统计会漏掉体验**：TTFT 快但 TPOT 慢，首字很快、整段仍会卡。诊断时同时看 `(TTFT, TPOT, 端到端完成时间)`；可燃尽的延迟 SLO 应写成事件比例，例如“有效交互中至少 99% 同时满足 TTFT ≤ X、TPOT ≤ Y、总耗时 ≤ Z”，阈值由产品体验和基线确定。

---

## 2. 完整算例：把账单拆到每一项

**场景**：coding agent 会话，Claude Opus 5（$5 输入 / $0.50 缓存读 / $25 输出；5m 缓存写 $6.25 —— [定价页](https://platform.claude.com/docs/en/about-claude/pricing)，2026 年中量级，随时变动）。

```
L₀ = 12,000 tok（系统提示 3k + 20 个工具定义 9k）    N = 15 轮
Δ  =  2,200 tok/轮（工具结果 1,500 + 模型输出 700）   输出合计 10,500 tok
沙箱 running 25 分钟 @ $0.08 / session-hour（Anthropic Managed Agents 计价）
─────────────────────────────────────────────────────────────────────
A. 不开缓存
   累计输入 = 15×12,000 + 2,200×(14×15/2) = 411,000 tok
   输入 411,000 × $5/M = $2.055 │ 输出 10,500 × $25/M = $0.263 │ 沙箱 $0.033
   合计 ≈ $2.35        沙箱占 1.4%，推理占 98.6%

B. 开 5 分钟 prompt caching
   cache_read  = Σ_{n=2..15}[12,000 + 2,200(n−2)] = 368,200 tok × $0.50/M = $0.184
   cache_write = 12,000 + 14×2,200 = 42,800 tok   × $6.25/M              = $0.268
   输出 $0.263 │ 沙箱 $0.033
   合计 ≈ $0.75        整单 −68%，输入项 −78%
```

**对照官方口径**：Anthropic 给的 1 小时 Opus 5 算例（50k 输入 / 15k 输出）是 $0.705 → $0.525，**整单只降 25.5%，输入项降 72%**。差别在轮次 —— 轮次越多、前缀占比越高，缓存杠杆越大。**不要拿别人的降幅当自己的预期，代入自己的 (N, L₀, Δ)。**

**缓存写值不值？** 一次写 + k 次读 vs 全价 k+1 次：

```
5m TTL:  1.25 + 0.1k < 1 + k  ⟹  k > 0.28  ⟹  读 1 次回本
1h TTL:  2.00 + 0.1k < 1 + k  ⟹  k > 1.11  ⟹  读 2 次回本
```

**推论：先按供应商写入/读取价格、TTL、复用次数和实测重叠率算回本点，再加 breakpoint。** 上面的 5m 价格示例里一次复用就回本，但长 TTL、低重叠、高并发或自建引擎的调度开销可能改变结论。

---

## 3. 优化手段 ROI 总表

下面是常见候选项，不是跨产品通用排序。表中比例来自本文算例或特定研究/产品快照，只能帮助你设计实验，**不能直接当降本承诺**；实际顺序由 `$/成功任务` 的归因结果决定。

| # | 手段 | 收益量级 | 实现成本 | 风险 / 撞墙条件 |
|---|---|---|---|---|
| 1 | **前缀字节稳定 + prompt caching** | 取决于可复用 token 占比、命中率、TTL 与平台价格 | 1–3 人日 | miss 通常无业务报错，要看 usage 指标；多租户共享有泄露面（§12） |
| 2 | **修失败运行** | 上限接近“失败且不产生价值的成本占比” | 数天（做归因 attribution） | 先定义成功/部分成功，别把用户取消或有价值的探索一概算浪费 |
| 3 | **上下文瘦身（context trimming）** | 总 token −30~84% | 1–2 周 | 压缩会击穿缓存（cache bust）；信息丢失导致重做反而更贵 |
| 4 | **轮次削减（turn reduction）**（工具粒度 tool granularity 重设计） | 本文“完整历史逐轮重发、无压缩且缓存未命中”算例中：轮次 −40% ⟹ 成本 −54%；其他上下文策略要重算 | 1–3 周（改工具契约） | 工具"大而全"后模型选择变难 |
| 5 | **非关键路径降档（tier down）**（子 agent 用 Sonnet/Haiku） | 该部分 −40~80% | 数天 | 无回归集则不知道降了什么 |
| 6 | **输出长度控制** | 输出 −20~50%；尾延迟（tail latency）明显改善 | 1–3 人日 | 截断导致重试，净成本上升 |
| 7 | **Batch API** | −50%（输入+输出），可与缓存叠加 | 数天 | 24h 窗口，用户在等的路径不适用 |
| 8 | **模型路由（model routing）/ 级联（cascade）** | −40~80%，保留 95~99% 质量 | 2–6 周 + 持续维护 | 路由器自身成本与错误率；**级联抬高 P99** |
| 9 | **语义缓存（semantic cache）** | 命中部分 100% 省 | 1–2 周 | 语义相似 ≠ 答案相同；**必须按租户分区** |
| 10 | **蒸馏（distillation）/ 微调（fine-tuning）** | 单位成本 −5~30× | $35k–120k + 数据 + 评测（⚠二手） | 上游迭代后不跟着变好，6–12 个月重做 |
| 11 | **自建 GPU** | 见 §9 盈亏平衡（break-even） | 数月 + 专职团队；人力成本按地区、资历与 on-call 范围估算 | 利用率 100%→20% 时，纯固定成本/有效 token 约涨 5×；实际还受吞吐和弹性成本影响 |

> **面试金句**
> "我会先按自己的成本归因找最大项。若可复用的推理输入占大头，第一批实验可以是稳定前缀并启用缓存；即使理论上不改语义，也要跑缓存命中、输出一致性、延迟与跨租户隔离回归。收益取决于重叠率、TTL 和目标平台价格，不能照搬别人的百分比。"

---

## 4. Prompt caching：三家机制差异与工程约束

**以下是 2026 年中官方页面的供应商/模型快照，只用于展示“为什么不能写一个通用缓存公式”。模型名、最小前缀、计费和 TTL 都可能独立变化；上线前必须按目标账户与地区回查，并把快照日期写进容量表。**

| | Anthropic | OpenAI | Gemini |
|---|---|---|---|
| 触发 | **显式** `cache_control` breakpoint | **全自动**；`prompt_cache_key` 提升路由命中，`prompt_cache_options.mode`=`explicit`/`implicit` | 隐式（默认开）+ 显式（须走 `generateContent`） |
| 最小前缀 | **非单调**：512（Opus 5 / Fable 5 / Mythos 5）、1024（Opus 4.8 / Sonnet 5 / 4.6 / 4.5）、2048（Opus 4.7 / Haiku 3.5）、**4096（Opus 4.6 / 4.5 / Haiku 4.5）** | 1024 tok | 隐式 4096（3.5 Flash / 3.1 Pro Preview）、2048（2.5 Flash / 2.5 Pro） |
| 写价 | 1.25×（5m）/ 2×（1h） | **GPT-5.6 起 1.25×**（此前免费） | — |
| 读价 | **0.1×** | **按代际不同**：GPT-5.x = 0.1×；gpt-4.1/o3/o4-mini = 0.25×；**gpt-4o = 0.5×** | — |
| TTL（time to live，缓存条目的存活时间，过期即失效） | 5m / 1h | GPT-5.6+ 至少保留 30 分钟；更早模型 5–10 分钟不活动淘汰，最长 1h | 隐式不可控 |
| 持有费（storage fee） | 无 | 无 | **显式缓存 $1.00 / 百万 token / 小时**（三家唯一） |
| 隔离域（isolation scope） | workspace（Bedrock/GCP 上按 organization） | 账户 | 项目 |

文档：[Anthropic Prompt Caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) · [OpenAI Prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching) · [Gemini Context caching](https://ai.google.dev/gemini-api/docs/caching)。⚠️ 广泛流传的"Gemini 显式缓存 75% 折扣、最小块 32K"在官方当前文档中**找不到对应表述**，视为未证实，不要写进容量模型。

**Anthropic 特有的三条硬约束（考点密集）**

1. **失效是级联的（cascading invalidation）**，渲染顺序 `tools → system → messages`：改 tool 定义 ⟹ 整条全废；改 system ⟹ system+messages 失效；改 `tool_choice` / 增删图片 ⟹ 仅 messages 失效。
2. **最多 4 个 breakpoint，每个只回看 20 个 content block**（content block：一条消息里的一个内容单元 —— 一段文本、一张图、一次 `tool_use`、一个 `tool_result` 各算一个）。Agent 单轮塞进 >20 个 `tool_use`/`tool_result` block 时，下一轮可能在没有业务错误的情况下 miss；用响应 usage 里的缓存创建/读取 token 和 TTFT 发现。
3. **并发同前缀请求全部 miss**：缓存条目要等第一条响应开始流式输出后才可读。**N 路 fan-out（把同一份工作同时分发给 N 个执行者，见 [00/01 §7](../00-foundations/01-fundamentals.md#7-尾延迟放大tail-latency-amplification)）同时发出 = N 次全价写。** 正确做法：先发 1 条、等到首 token，再发其余。

**工程做法：prompt 装配写成"字节稳定前缀 + 易变尾巴"两段**

```python
# ❌ 后面所有 breakpoint 全废
system = f"You are an agent. Time: {now()}. User: {user_id}. Tools: {json.dumps(tools)}"

# ✅ 位置 0 起字节级恒定，变量全部下沉到 messages 尾部
BLOCKS = [{"type": "text", "text": TOOLS_DOC,  "cache_control": {"type": "ephemeral"}},
          {"type": "text", "text": POLICY_DOC, "cache_control": {"type": "ephemeral"}}]
messages = history + [{"role": "user", "content": f"[ctx] {now()} {user_id}\n{user_msg}"}]
```

规则（每条对应一个真实的账单事故）：system prompt 里禁止 `now()`/UUID/`user_id`/未排序的 `json.dumps`；**会话中途不要增删工具、不要切模型**（缓存按模型隔离，模式切换改用 mid-conversation system message）；检索到的文档放 messages 尾部而不是拼进 system；上下文只追加。

**必须做成监控指标，而不是排查手段**

```sql
SELECT date_trunc('hour', ts) AS h, model, agent_role,
       sum(cache_read_input_tokens)::float / nullif(sum(cache_read_input_tokens
         + cache_creation_input_tokens + input_tokens), 0) AS cache_read_ratio
FROM llm_calls WHERE ts > now() - interval '24 hours' GROUP BY 1,2,3 ORDER BY 1 DESC;
-- 告警：同一 (model, agent_role) 的 cache_read_ratio 环比跌 > 20 个百分点
```

⚠️ **`input_tokens` ≠ prompt 总长**。真实总输入 = `input_tokens + cache_creation_input_tokens + cache_read_input_tokens`。只看第一项会把"上下文压缩"的收益算反。参考量级：Claude Code 官方 session 示例是 **940k cache read : 50k cache write : 1.2k 直读输入** —— 长会话下缓存读占输入 token 的 95%+。

---

## 5. 上下文瘦身：最大的杠杆，也是最危险的一个

瘦身同时打击 `L₀` 和 `Δ`，且对 Δ 的削减被二次项放大。三个下手点，按 ROI 排序：

**a) 工具输出（Δ 的主体，也是最大污染源）** —— 返回值走 schema 白名单，只回传真正需要的字段（`list_issues` 完整 JSON 8k token vs `[{id,title,state}]` 400 token）；文件读取强制分页（`offset`/`limit`）；大结果落盘只回句柄（handle）：`saved to /tmp/r42.json (1,240 rows, cols: ...)`。

**b) 工具定义（L₀ 的主体）** —— 30 个工具 × 400 token = 12,000 token，**每轮全量重发**。做法：按阶段/角色做工具的命名空间加载（namespaced tool loading），别把所有 MCP server 的工具一次全挂上。

**c) 检索量** —— 一阶段多召回 → rerank（重排：用更准的模型对候选重新排序并砍掉尾部，见 [02 §5.4](02-context-engineering-and-rag.md#54-rerank收益最确定的一环)）→ 只塞达到质量门槛的片段。`top-100 → 30–50 → 20` 是 [Anthropic Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) 特定实验的候选配置，不是通用最优值。若仅按示例价格 `$0.20 / $0.00008`，比例是 **2,500×**；但这没有计入索引构建、embedding、rerank、存储、质量差异与缓存，只能说明“值得建完整单位经济模型”，不能直接证明 RAG 便宜多少倍。

**服务端上下文编辑**（Anthropic，截至 2026-07 仍为 beta，header `context-management-2025-06-27`）：

```
clear_tool_uses_20250919:
  trigger 默认 100,000 input tokens │ keep 默认 3 个 tool_use/tool_result 对
  clear_at_least 最少清理多少 token —— 用来避免为很小的释放量频繁改写上下文，并降低缓存重建得不偿失的概率
  exclude_tools 如 web_search、memory 类 │ clear_tool_inputs 默认 false
clear_thinking_20251015: 组合使用时必须排在 edits 数组首位
预演：count_tokens 端点支持带 context_management，返回清理前后的 input_tokens
```

官方实测：100 轮 web search 场景 token **−84%**；memory tool + context editing 组合 **+39%** 性能，单独 **+29%**。

**这一节最重要的一句话：压缩和缓存是对立的。** 改写历史（compaction，把长历史压成摘要，见 [04 §7](04-agent-memory-and-state.md#7-compaction触发实现以及它和前缀缓存prefix-caching的正面冲突)）= 前缀变了 = 整条缓存作废，下一轮按 1.25× 重写全部前缀。所以 `压缩净收益 = Δ_saved × 剩余轮次 × P_in × 0.1 − L_prefix × P_in × 1.25`。代入 §2 参数（L₀=12,000，Δ_saved=20,000）：重建代价 ≈ $0.075，每轮省 ≈ $0.010 ⟹ **剩余轮次 ≥ 8 才划算**。这就是 `clear_at_least` 必须设得足够大的原因 —— 小打小闹的清理是净亏损。

---

## 6. 轮次削减：被严重低估的一档

```
N = 15 → 累计输入 411,000 tok
N =  9 → 累计输入 187,200 tok        轮次 −40%  ⟹  成本 −54%
```

| 做法 | 说明 | 典型收益 |
|---|---|---|
| **批量参数（batch parameters）** | `read_file(path)` → `read_files(paths[])`；`get_issue(id)` → `get_issues(ids[])` | 5 次调用变 1 次 |
| **一次给够** | 搜索工具返回 snippet 而非仅 ID，避免必然发生的第二次 fetch | 砍掉一整类往返（round trip） |
| **并行工具调用（parallel tool calls）** | 一轮内发多个 `tool_use`。注意 20-block 窗口；纯读默认可并行，写仅在冲突键不相交且幂等时并行 | 轮次 −30~50%（需实测） |
| **确定性工作出模型** | 分页、排序、字段过滤、格式转换、单位换算用代码做 | 消灭"模型算错再重试"的整轮 |
| **明确终止条件（termination condition）** | MAST 分类里"不知道终止条件"占 **12.4%**、"步骤重复"占 **15.7%** | 消灭长尾会话 |

> **面试金句**
> "在我们那条‘历史只追加、每轮完整重发’的链路里，轮次削减 ROI 很高。我们把 12 个细粒度文件工具合并成 3 个带批量参数的工具；代入本节算例，中位轮次从 15 降到 9，累计输入 token 从 411k 降到 187k。生产收益还要从 usage 与 eval 复测。代价是 schema 变复杂、模型可能传错参数，所以同时加参数校验和一次性纠错提示。"

---

## 7. 模型路由与级联

```
路由 Router ：classify(q) → 一次调用
  E[C] = p_small·C_small + (1−p_small)·C_large   延迟：每次都增加路由器耗时，P99 也可能上升
级联 Cascade：small → 置信度低则 escalate → large
  E[C] = C_small + p_escalate·C_large            延迟：P99 = 小+大串行 ⟹ 尾延迟恶化
级联反而更贵的条件： C_small + p·C_large > C_large ⟹ p > 1 − C_small/C_large
```

Haiku 4.5 : Opus 5 的价格比约 1:5 ⟹ **升级率（escalation rate）超过 80% 时，级联比直接用大模型还贵，而且更慢。** 上线前必须先离线量出 `p_escalate`；量不出来就别上。

**路由器本身不是免费的：**

| 实现 | 每次成本量级 | 延迟 | 备注 |
|---|---|---|---|
| Embedding + 逻辑回归 / kNN | ~$0.0001（embedding $0.12–0.15/M） | 10–30 ms | 首选：可离线训练、可解释、可回放 |
| 小模型分类（Haiku / Flash-Lite） | ~$0.0005–0.002 | 200–500 ms | 中庸 |
| 大模型当路由器 | $0.005+ | 500 ms+ | **会吃掉 20–30% 的节省** —— 反模式 |
| 规则（长度 / 是否带代码 / 工具数） | ~0 | ~0 | v0 就该有，能覆盖 40–60% 的明显案例 |

**收益的真实区间**：实测 **40–85% 降本、保留 95–99% 质量**。[arXiv 2606.27457《Cluster, Route, Escalate》](https://arxiv.org/abs/2606.27457)（2026-06-25）的两阶段级联 —— 先聚类分配最省模型，再用 quality estimation 升级低置信输出 —— **保留最强模型 97–99% 的准确率**并降低 TPOT。⚠ RouteLLM 系被反复引用的"降本 >85%、仅 14% 查询走强模型"是**二手转述且是上限不是均值**（MT-Bench 类偏易基准）；生产混合流量的现实区间是 **40–80%**。

可抄的编排层对照（2026-03，10,000 份 SEC 文件 / 25 类字段 / 5 模型）：Reflexive 自纠正 F1 0.943 / 成本 **2.3×**；Hierarchical supervisor-worker F1 0.921 / **1.4×**；**Hybrid（语义缓存 + 模型路由 + 自适应重试）成本 1.15× 就拿回 reflexive 89% 的收益**。

**三条工程纪律**：① 路由决策做成可快速下线的旁路（bypass，feature flag），并演练退回单一路径；② **先埋点（instrumentation）后路由** —— 在全走基线模型的状态下用去副作用的影子流量（shadow traffic：把线上真实请求复制一份喂给候选方案，两边都打分，但候选输出不返回给用户，见 [06 §8](06-evaluation-and-observability.md#8-上线路径离线--影子shadow-金丝雀canary-全量)）测小模型通过率，再用“可路由比例 × 单次节省 − 路由/维护成本”判断是否值得做，不设跨业务的 30% 门槛；③ **路由维度不只是模型**，还有 reasoning budget、是否开工具、是否开检索。任何降档都按任务切片跑质量和延迟回归，不把某个 benchmark 的结论直接搬到生产。

---

## 8. 批处理档位与输出长度控制

| 档位 | 折扣 | 形态 | 适用 |
|---|---|---|---|
| 同步 | — | 阻塞 | 用户在等 |
| **OpenAI Flex** | 对齐 Batch 费率，**可叠加 caching** | **保持同步调用形态** | 能等几分钟的后台任务；更慢、更易超时、可能 429（HTTP 状态码 Too Many Requests，即被限流；**429 不计费**），SDK 默认 10 分钟超时需调大。beta、模型有限（[文档](https://developers.openai.com/api/docs/guides/flex-processing)） |
| **Batch** | 本文三家价目快照均约为标准输入/输出价的 50%；不是永久统一标准 | 提交 → 轮询 → 取结果 | 离线 eval、批量分类打标、embedding 回填、夜间报告 |
| **Fast mode**（反向档） | **2× 标准价**换 ≤**2.5×** 输出 tok/s | 仅第一方 API，**不能与 Batch 同用** | 只在感知延迟直接等于产品价值时成立（demo、实时语音） |

Batch 边界（2026 年中）：Anthropic 100,000 请求 / 256 MB、24h 过期、结果留 29 天；OpenAI 50,000 / 200 MB、`completion_window` **只能填 `24h`**、**独立速率池不占同步配额**；Gemini inline ≤20MB / 文件 ≤2GB、任务 48h 过期、结果留 6 周。

两个可验证的结论：① 若目标供应商明确允许 Batch 与 caching 叠加，Opus 5 这个价目快照的理论输入价是 `$5 × 0.1 × 0.5 = $0.25/M`；这是理想命中上界，不代表整单或所有模型都省 20×。大规模 eval 还要比较失败重跑、完成时限与缓存命中。② Batch 任务可能超过 5 分钟，TTL 应覆盖任务排队与执行窗口；官方材料里的 **30%–98%** 是特定批任务观测区间，不是 SLA。是否支持 1h TTL、缓存预热和叠加计费都按当前 API 验证。

**输出长度控制**。在本篇 Opus 5 快照里输出单价是输入的 **5×**（$25 vs $5），且输出不能作为本次请求的缓存读；其它模型比例不同。输出 token 还直接影响 wall-clock，因此要按实际模型分别核算成本与延迟。

- **结构化输出 / 约束解码（constrained decoding）**：XGrammar 已集成到若干 serving 框架，但是否为“默认后端”取决于框架版本、构建和配置，部署时要查 release/config 而不是据此假设。论文与 JSONSchemaBench 报告的 `<40 µs/token`、合规率 `61.1%–72% → 96%–100%`，以及第三方测得的 `20–50 ms` 编译、batch≥8 吞吐变化，都是各自硬件/schema/版本下的快照；你的 schema 缓存命中率、batch 与模型必须单独压测。
- **`max_tokens` 按 tokenizer 重标定**：Claude 4.7+ 新 tokenizer 对同样文本产生约 **+30% token**。跨代升级时"单价不变 = 成本不变"是错的，`max_tokens` 与 compaction 阈值都要重标，否则会出现"同样任务突然被截断"。**跨代成本对比先跑 `count_tokens`。**
- **禁复述（no regurgitation）**：系统提示里写死"不要复述工具结果、不要重复用户输入、直接给结论"。Agent 最常见的浪费是把刚读到的 200 行文件在回复里重抄一遍。
- **thinking budget 是旋钮不是开关**：抑制过度思考后 Step Success Rate 从 **0.251 → 0.302**（2026-06）。thinking token 按输出价计费。

---

## 9. 自建 vs API 的盈亏平衡

```
自建单位成本 ($/M) = 节点日成本 × 隐性倍数 ÷ (吞吐 tok/s × 86,400 × 利用率 ÷ 1e6)
盈亏平衡日 token 量 T* 满足:  T* × P_api = 节点日成本 × 隐性倍数
```

**算例**：单 H100 跑 70B FP8 + vLLM 连续批处理（continuous batching：一个序列生成完就立刻把空位让给排队中的新请求，不必等整批跑完，见 [01 §3](01-llm-serving-infra.md#3-分页连续批处理chunked-prefill)）≈ **400 tok/s ≈ 34.5M tok/天**（⚠二手口径）。H100 现货 ~$2.00–3.00/hr，取 $2.50 ⟹ 裸租金 **$60/天**。

| 成本口径 | 日成本 | 对标 $25/M（Opus 5 输出） | 对标 $5/M | 对标 $1.20/M |
|---|---|---|---|---|
| 裸 GPU 租金 | $60 | 回本需 **2.4M tok/天** | 12.0M tok/天 | 50M > 上限 ⟹ **不可能** |
| ×3 隐性（工程/冗余/闲置） | $180 | 7.2M tok/天 | 36M > 上限 ⟹ **不可能** | 不可能 |
| ×5 隐性 | $300 | 12M tok/天 | 不可能 | 不可能 |

> 物理上限 34.5M tok/天（100% 利用率）。这里的隐性倍数 3–5× 只是二手场景假设，混合了工程人力、冗余副本、闲置时段与可观测性；其中 `$5,000–15,000/月` 不能跨地区或团队直接复用。正式决策应逐项列本地 fully-loaded labor、on-call、冗余和利用率。

**利用率（utilization）敏感性 —— 这张表就是自建决策的全部**（8×H100 跑 671B MoE（mixture of experts：模型总参数很大，但每个 token 只激活其中一小部分专家，见 [01 §10](01-llm-serving-infra.md#10-并行策略与-moe-服务化)），⚠二手，2026-06-17）：

| 利用率 | 100% | 50% | 20% | **5%** |
|---|---|---|---|---|
| 单位成本 | $10 / M output | $20 / M | $50 / M | **$200 / M** |

真实业务有日夜峰值，平均利用率可能远低于峰值利用率；是否低于 30% 取决于能否混部、批处理填谷、弹性释放和负载稳定性，必须从调度指标计算。

**这个话题存在明确争议**：一派算出 70B 单 H100 对标 $5/M 旗舰约 **12.17M tok/天**回本；另一派（2026-06-17）算出 8×H100 跑 671B MoE 满载 ≈ **$10/M output**，比 DeepSeek V4 Pro API 的 $0.87/M **贵 11 倍**。除了价格基线不同，模型质量、上下文长度、输出速度、可用性和安全控制也不同。只有候选模型在同一任务 eval 和 SLO 上达到可接受等价后，单位 token 成本才可直接比较。

> **面试金句**
> "自建 GPU 划不划算，取决于你对标哪个模型，而不是取决于 GPU 便不便宜。对标 $25/M 的旗舰输出，一台 H100 每天跑 2–12M token 就回本；对标 $1.20/M 的小模型，你把卡跑满也回不了本，因为裸租金口径的单位成本就已经是 $1.74/M。所以我的第一个问题不是'GPU 多少钱'，是'这个 workload 用得起最便宜的哪个模型'。"

**什么时候自建才对**：① 数据不能出网/出境（合规硬约束）；② 单一稳定高利用率 workload（批量 embedding、离线打标、固定分类）；③ API 上没有你需要的模型。**"省钱"单独不构成理由。**

**蒸馏（distillation：拿大模型在你这类任务上的输出当训练数据，去训一个小模型，让小模型在这一类任务上逼近大模型）/ 微调**同理。收益（⚠均为二手）：**5–30×** 单位成本下降；13B 微调在与前沿模型差 2 个准确率点内时，单 token 便宜 **12–40×**；单点案例给出投入 $35k–120k、回本约 **2.9 个月**。真实代价常被漏掉：**先得有评测体系**（LLM judge 起步就要 100+ 标注样本 + 每周维护）、几千到几万条训练对、**模型漂移（model drift）**（上游迭代后你的蒸馏模型不跟着变好，6–12 个月要重做）、一整套新 serving 栈（见 [01-llm-serving-infra.md](01-llm-serving-infra.md)）、一个长期 owner。**判据：任务窄 + 量大 + 稳定 ≥ 6 个月，三者同时成立才做。** Agent 主循环通常不满足（提示和工具每周都在改）；适合蒸馏的是**边缘子任务**：意图分类、字段抽取、重排（reranking）、query 改写、安全审查。

---

## 10. 延迟优化清单

**TTFT（按投入产出排序）**

| # | 手段 | 收益 | 备注 |
|---|---|---|---|
| 1 | **前缀缓存 / prompt caching** | 长输入下可降**两个数量级** | 先做这个 |
| 2 | **缓存感知路由（cache-aware routing）（会话粘性 session affinity：把同一会话的后续请求固定送回上一次那台实例，因为它上面还留着这个会话算好的 KV cache）** | 极端对照 P90 TTFT **0.542 s**(precise) vs 31.083 s(approximate) vs **92.551 s**(random) ⟹ **57× / 170×** | 8 pod / 16×H100 / 150 租户 × 6k ctx。**这让 LLM 网关变成有状态路由（stateful routing）** |
| 3 | **KV cache 复用 / 融合** | [CacheBlend](https://arxiv.org/pdf/2405.16444) 相比纯前缀缓存再降 **2.2–3.3× TTFT**、吞吐 +2.8–5× | RAG 的多段非前缀复用 |
| 4 | 分层 KV 卸载 | LayerKV 超长 prompt 最高 **69×**；vLLM CPU offload 单请求 TTFT **2–22×** | 需自建 serving |
| 5 | 上下文压缩 / 裁剪 | 与 §5 同一件事 | 注意击穿缓存 |
| 6 | 会话级调度 | [ConServe](https://arxiv.org/abs/2606.01839)：p95 time-to-first-effective-token **−51.08%** | 需自建 serving |

**Decode 与端到端**

- **投机解码（speculative decoding）只降 decode 感知延迟，不降 TTFT**，且**并发下急剧衰减**：1.69×(c=1) → **1.05×(c=64)**；Qwen3-8B bs=2 的 1.93× → **bs=48 的 0.99×（反而变慢）**。高并发生产环境别指望它。
- **流式输出不改 wall-clock、不降成本，只改感知延迟（perceived latency）。** 汇报成"延迟优化"会误导容量规划（capacity planning）。
- **并行工具调用**（只读）、**预取（prefetch）**（模型还在生成时就对高置信下一步发起检索，命中白赚一整轮）、**投机执行（speculative execution）**（小模型先跑候选、大模型确认后采用；**有副作用的动作绝不能投机**）、**把确定性工作挪出模型**（一个 200ms 的 SQL vs 一整轮 3s 的模型往返）。

**一份可抄的延迟预算（交互式 agent）**

```
网关+鉴权+限流 ≤30ms │ 检索(hybrid+rerank) ≤250ms │ 排队 ≤200ms │ TTFT ≤800ms
输出 500 tok @ TPOT 20ms = 10s │ 工具往返(并行后取 max) ≤1.5s
────────────────────────────────────────────────────────────────────────
单轮 ≈ 12.8 s  ×  中位 9 轮  ≈  115 s 端到端   ← 汇报这个，不是 800 ms 的 TTFT
```

---

## 11. 单位经济模型（unit economics）与产品护栏（guardrail）

| 指标 | 为什么 |
|---|---|
| **$/成功任务** | 比 $/请求 更接近价值；但要同时定义部分成功、用户取消和人工接管，避免把有价值的中间产出算成失败 |
| $/活跃用户日 | 对齐订阅定价 |
| $/租户月 + 毛利 | 对齐商业模型 |
| **失败运行的花费占比** | 按失败原因与是否产生部分价值分层；它给出修失败路径的理论节省上限 |

**参考锚点**（[Claude Code 官方成本文档](https://code.claude.com/docs/en/costs)）：**$13 / 开发者 / 活跃日**，**$150–250 / 月**，**90% 用户 < $30 / 活跃日**；空闲期后台 token 通常 < $0.04/会话。⚠ 另有二手引用称 $6/日、P90 < $12/日，可能是不同时间点或人群，以官方现行文档为准。

**长尾（long tail）：P99 可能远高于 P50，但倍数必须从自己的分布测。** 下面只是一个用于找变量的压力测试，不是行业观测值：

```
P99/P50 ≈ 会话输入倍数 × 角色/分支倍数 × 失败重做倍数 × 单价组合倍数
例如：2.0 × 2.5 × 1.4 × 1.0 = 7×
若历史完整重发，轮次还会通过二次项放大输入；有缓存或压缩时要换成实测曲线
```

**成本分布常呈重尾（heavy-tailed），只看均值会隐藏典型用户与尾部风险。** 均值仍用于总成本、预算和期望容量；同时报告中位数、P90/P99、按租户/任务类型分层的并发峰值与尾部贡献，定价再配额度和超额机制。毛利与推理占比受产品形态、定价、客户结构和会计口径影响，应从自己的损益表计算，不拿行业均值代替。

**成本归因表**（不做这个，上面所有指标都算不出来）：

```sql
CREATE TABLE llm_call_costs (
  ts timestamptz NOT NULL, tenant_id uuid NOT NULL, user_id uuid,
  session_id uuid NOT NULL, run_id uuid NOT NULL, turn_idx int NOT NULL,
  agent_role text,                 -- lead / subagent:search / judge / router
  model text NOT NULL, input_tokens bigint,          -- input_tokens = 未缓存直读
  cache_creation_input_tokens bigint, cache_read_input_tokens bigint, output_tokens bigint,
  cost_usd numeric(12,6) NOT NULL, -- ⚠ 写入时按当时价目表固化
  outcome text);                   -- success / failed / aborted
CREATE INDEX ON llm_call_costs (tenant_id, ts DESC);
```

两条纪律：**`cost_usd` 写入时固化**（价格会变 —— Sonnet 5 就有明确的 2026-09-01 促销切换日；事后 join 价格表算不出历史真实成本）；**`agent_role` 必须有**（否则你只有一个总数，无法回答"是哪个子 agent 在烧钱"）。

**产品侧护栏**

| 护栏 | 做法 | 为什么 |
|---|---|---|
| **配额（quota）按 $ 不按请求数** | `budget_usd_per_day` per tenant | 一个请求可以是 $0.001 也可以是 $5 |
| **per-run 硬上限** | `max_cost_usd` + `max_turns`，超了终止并返回已完成部分 | 防单次运行失控 |
| **降级阶梯（graceful degradation）** | Opus → Sonnet → Haiku → 排队 → 拒绝，**且对用户可见** | 静默降级（silent degradation）会变成信任事故 |
| **定价对齐** | 订阅 + 超额按量（overage billing）+ 硬上限 | 纯订阅（行业 58%）不封顶重度用户；纯消费型（35%）用户不可预测 |
| 结果型定价（outcome-based pricing）参考 | Intercom Fin **$0.99/outcome**（一次会话只计一个，月最低 50）；Salesforce Agentforce Flex Credits **$0.005/credit**，三动作工单 ≈ $0.30 | 结果型占 18%，正在增长 |

计量管道的正确性（幂等 idempotency、去重、对账 reconciliation）见 [03-saas-platform/02-billing-and-metering.md](../03-saas-platform/02-billing-and-metering.md)。

---

## 12. 什么时候不要优化 / 反模式（anti-patterns）

**什么时候不要优化**：① 先用 `可节省月成本 × 置信度` 对比本团队的 fully-loaded 实现与维护成本，不使用跨地区的 `$5k–15k` 硬阈值；② 还在找 PMF、提示每周都在改时，蒸馏、自建和复杂路由的返工风险很高；③ 质量还没到线时先把 eval 建起来（见 [06-evaluation-and-observability.md](06-evaluation-and-observability.md)），否则无法判断降本是否破坏产品。

| # | 反模式 | 真相 |
|---|---|---|
| 1 | **先优化没归因的小项** | 沙箱、推理、工具和存储占比随产品变化；先按 `$/成功任务` 拆账，再选择最大且可控的项 |
| 2 | 不监控缓存 usage | miss 往往没有业务错误，但通常能从缓存读/写 token、命中率、TTFT 与账单变化发现。**必须是指标，不是排查猜测** |
| 3 | 只看 `input_tokens` | 总输入 = 三项之和，只看第一项会低估一个数量级 |
| 4 | N 路 fan-out 同时发同前缀 | = N 次全价写。先发 1 条等首 token |
| 5 | agent 单轮 > 20 个 content block | 触发 20-block 回看限制，**账单暴涨但没有任何报错** |
| 6 | batch 里用默认 5 分钟 TTL | Batch 常超 5 分钟，基本无效。改 1h TTL |
| 7 | 会话中途增删工具 / 切模型 | 工具定义渲染在位置 0，任何改动使整条缓存失效 |
| 8 | 跨代升级假设"单价不变 = 成本不变" | Claude 4.7+ 同文本多约 **30% token** |
| 9 | 把长上下文当免费 | Anthropic 1M 无溢价，但 **Gemini 3.1 Pro 超 200k 后输入 ×2、输出 ×1.5** |
| 10 | **把前缀缓存当无条件收益** | 不同负载实测可能增益也可能退化。先量前缀重叠率、并发、写放大、命中后的 TTFT/TPOT 与跨租户隔离 |
| 11 | 语义缓存不分租户 | 语义相似 ≠ 答案相同；跨租户复用是数据泄露 |
| 12 | 用流式冒充延迟优化 | 不降 wall-clock、不降成本 |
| 13 | 把路由的 85% 当均值 | 那是偏易基准的上限；生产 40–80%，且**级联抬高 P99** |
| 14 | 把"会话空闲后第一条消息"当普通请求 | 缓存 TTL 过期后**全量重算**，长会话里可能是最大单笔支出。对策：`max_tokens: 0` 预热、1h TTL、或从摘要重建 |

**必须显式写出的一个张力：缓存是最大杠杆，也是最大泄露面。**
杠杆侧 —— agentic KV 复用把命中率从 1.7% 拉到 92.2%，P50 TTFT **↓46×**、端到端 **↓8.6×**、吞吐 **↑3.8×**（GB200×60，2026-05；⚠ 代码未合入主干）。泄露侧 —— PROMPTPEEK（NDSS 2025）实证共享 prefix cache 可被**逐 token 重建他人 prompt**，无背景知识成功率 **95%**。

结论只能是 **同租户内共享、跨租户默认关闭** —— 而这会直接削掉多租户场景里最大的性能杠杆。这不是可以两全的取舍，是必须在设计评审上明说的成本。详见 [07-agent-security.md](07-agent-security.md)。

---

## 这一章的三句话

1. **先从 trace 和账单还原成本曲线。** 只有在“历史只追加、每轮完整重发”时累计输入才含明显二次项；缓存、裁剪和服务端状态会改变曲线。若可复用输入是最大项，就试前缀稳定与缓存；若失败重跑、输出、工具或沙箱更大，就先处理它们。所有改动都要重跑 eval 与隔离回归。
2. **压缩收益要和缓存重建成本一起算。** 对会把历史改写进缓存前缀的平台，压缩可能让下一轮重建缓存；本篇价目与会话算例的回本点是剩余约 8 轮。换模型、TTL、压缩位置或缓存机制后都要重算，不能把 8 轮当规则。
3. **自建划不划算取决于等质量、等 SLO 下对标哪个模型，以及能维持多少利用率。** 纯固定成本模型里，利用率从 100% 掉到 20%，每个有效 token 分摊的固定成本约涨 5 倍；真实利用率必须从自己的负载、混部和弹性能力计算。

---

## 面试官会追问

1. 你的 agent 一次运行成本 $2，让你降到 $0.6，按什么顺序动手？为什么不先换便宜模型？
2. 在什么上下文策略下，agent 累计输入会出现轮次的二次项？写出公式；再说出两种会改变这条曲线的机制。
3. 在本篇供应商快照里，缓存写乘数 1.25×（5m）/ 2×（1h），读 0.1×。回本点各是读几次？若读价变成 `R`，公式怎么写？
4. 你上线了上下文压缩，账单反而涨了 10%。可能是什么原因？
5. 团队要做级联（Haiku 先做、Opus 兜底）。什么条件下这会比直接用 Opus 更贵？给临界公式。
6. 缓存命中率从 85% 掉到 5%，没有任何报错。列出你会依次检查的 5 件事。
7. 老板问"我们自建 GPU 能省多少"。你需要先问清楚哪三个数才能回答？
8. P50 用户成本 $13/日，P99 $260/日。定价怎么设计才不会被重度用户拖垮毛利？
9. 前缀缓存既是最大性能杠杆又是跨租户泄露面。多租户平台上你怎么处理这个取舍？

---

**按训练路径阅读** → 回 [START-HERE](../START-HERE.md) 按所选路径继续；页尾链接只表示本目录或专章的顺读顺序。

**AI 专章顺读下一篇** → [01-slo-and-error-budget.md](../05-reliability/01-slo-and-error-budget.md)
