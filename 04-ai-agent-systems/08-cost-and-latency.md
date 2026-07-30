# 08 · 成本与延迟优化

> Agent 的成本对**轮次（turns）是二次的（quadratic）**，不是线性的。所以第一顺位的优化从来不是"换个便宜模型"，
> 而是让前缀字节稳定（byte-stable prefix，缓存命中）和让工具变粗（减少轮次）。
> 不先建模就开始优化，你会花两周砍掉只占 13% 的沙箱成本。

---

## 1. 先建模：成本与延迟的分解公式

```
C_run = C_infer + C_sandbox + C_tools + C_storage

C_infer = Σ_turns [ T_fresh × P_in                # 未缓存直读
                  + T_cw    × P_in × W            # W = 缓存写乘数（1.25× / 2×）
                  + T_cr    × P_in × 0.1          # 缓存读 ≈ 输入价 10%
                  + T_out   × P_out ]
C_sandbox = running_seconds × 单价                 # 是 running，不是 wall-clock
C_tools   = web_search 次数 × 单价 + 外部 API
C_storage = 向量库 + 对象存储 + Gemini 显式缓存持有费
```

**主导项是 `Σ_turns T_in`，而它随轮次二次增长** —— 第 n 轮必须重发前 n−1 轮的全部内容：

```
累计输入 token ≈ N·L₀ + Δ·N(N−1)/2
   N = 轮次   L₀ = 稳定前缀（系统提示 + 工具定义）   Δ = 每轮上下文增量
```

Δ 的真实量级：agentic KV 复用的生产实测给出每轮上下文增长中位约 **2,242 token**，输入:输出比 **131:1**（2026-05 口径）。**agent 的账单几乎全是输入 token，而输入 token 几乎全是重发的历史。** 这一条推出本篇 80% 的结论。

**延迟分解**：`L_e2e = Σ_turns [ Queue + TTFT + T_out × TPOT + Tool_time ] + Orchestration`，其中 `TTFT ≈ Queue + uncached_prefill_tokens / prefill_throughput + RTT`。

| 指标 | 交互式目标 | 依据 |
|---|---|---|
| TTFT | < 1 s，理想 < 600 ms，> 2 s 用户明确感知等待 | Nielsen 三档；Gemini 2.5 Flash、Claude Haiku 4.5 在中等 prompt 上稳定 < 600 ms |
| TPOT | ≤ 20 ms（≈ 50 tok/s） | GLM-5.2 生产 SLA：mean TTFT ≤ 2.5 s、mean TPOT ≤ 20 ms，实测 17 ms |
| 端到端 | 轮次 × 单轮 | 15 轮 × 3 s = 45 s，**用户体验的是 45 s，不是 TTFT** |

⚠️ **单指标 SLO 是陷阱**：同样满足 `TTFT ≤ 1.5s` 的两个配置，P95 TPOT 一个约 **50 ms**、另一个约 **494 ms**（10× 差异，2026-07 chatbot 负载）。SLO 必须是 `(TTFT, TPOT)` 联合的。

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

**推论：任何会被复用至少一次的前缀，无条件加 breakpoint。**

---

## 3. 优化手段 ROI 总表

按"先做哪个"排序。收益是相对该项作用域的量级，不是总账单。

| # | 手段 | 收益量级 | 实现成本 | 风险 / 撞墙条件 |
|---|---|---|---|---|
| 1 | **前缀字节稳定 + prompt caching** | 输入 −60~90%；TTFT −2~10× | 1–3 人日 | 静默 miss（silent cache miss，无报错，只有账单）；多租户共享有泄露面（§12） |
| 2 | **修失败运行** | 直接 −10~15% | 数天（做归因 attribution） | 生产遥测：**14.2% 的总花费花在失败的运行上**（639,381 执行步，2026 上半年） |
| 3 | **上下文瘦身（context trimming）** | 总 token −30~84% | 1–2 周 | 压缩会击穿缓存（cache bust）；信息丢失导致重做反而更贵 |
| 4 | **轮次削减（turn reduction）**（工具粒度 tool granularity 重设计） | 二次放大：轮次 −40% ⟹ 成本 −54% | 1–3 周（改工具契约） | 工具"大而全"后模型选择变难 |
| 5 | **非关键路径降档（tier down）**（子 agent 用 Sonnet/Haiku） | 该部分 −40~80% | 数天 | 无回归集则不知道降了什么 |
| 6 | **输出长度控制** | 输出 −20~50%；尾延迟（tail latency）明显改善 | 1–3 人日 | 截断导致重试，净成本上升 |
| 7 | **Batch API** | −50%（输入+输出），可与缓存叠加 | 数天 | 24h 窗口，用户在等的路径不适用 |
| 8 | **模型路由（model routing）/ 级联（cascade）** | −40~80%，保留 95~99% 质量 | 2–6 周 + 持续维护 | 路由器自身成本与错误率；**级联抬高 P99** |
| 9 | **语义缓存（semantic cache）** | 命中部分 100% 省 | 1–2 周 | 语义相似 ≠ 答案相同；**必须按租户分区** |
| 10 | **蒸馏（distillation）/ 微调（fine-tuning）** | 单位成本 −5~30× | $35k–120k + 数据 + 评测（⚠二手） | 上游迭代后不跟着变好，6–12 个月重做 |
| 11 | **自建 GPU** | 见 §9 盈亏平衡（break-even） | 数月 + 专职团队 $5k–15k/月 | 利用率 100%→20% 单位成本涨 5× |

> **面试金句**
> "我不会先去砍沙箱。官方算例里推理占 85–89%，沙箱只占 11–15%，存储近乎 0 —— 优化预算的 85% 必须投在推理侧。而推理侧的第一顺位不是换模型，是让前缀字节稳定：这是一周内能上线、不改变任何行为、不需要重跑 eval 的改造，能吃掉输入成本的 60–90%。换模型能省更多，但要重跑整个回归集，那是一个季度的事。"

---

## 4. Prompt caching：三家机制差异与工程约束

**以下数字均为 2026 年中量级，随时变动，上线前必须回查官方页。**

| | Anthropic | OpenAI | Gemini |
|---|---|---|---|
| 触发 | **显式** `cache_control` breakpoint | **全自动**；`prompt_cache_key` 提升路由命中，`prompt_cache_options.mode`=`explicit`/`implicit` | 隐式（默认开）+ 显式（须走 `generateContent`） |
| 最小前缀 | **非单调**：512（Opus 5 / Fable 5 / Mythos 5）、1024（Opus 4.8 / Sonnet 5 / 4.6 / 4.5）、2048（Opus 4.7 / Haiku 3.5）、**4096（Opus 4.6 / 4.5 / Haiku 4.5）** | 1024 tok | 隐式 4096（3.5 Flash / 3.1 Pro Preview）、2048（2.5 Flash / 2.5 Pro） |
| 写价 | 1.25×（5m）/ 2×（1h） | **GPT-5.6 起 1.25×**（此前免费） | — |
| 读价 | **0.1×** | **按代际不同**：GPT-5.x = 0.1×；gpt-4.1/o3/o4-mini = 0.25×；**gpt-4o = 0.5×** | — |
| TTL | 5m / 1h | GPT-5.6+ 至少保留 30 分钟；更早模型 5–10 分钟不活动淘汰，最长 1h | 隐式不可控 |
| 持有费（storage fee） | 无 | 无 | **显式缓存 $1.00 / 百万 token / 小时**（三家唯一） |
| 隔离域（isolation scope） | workspace（Bedrock/GCP 上按 organization） | 账户 | 项目 |

文档：[Anthropic Prompt Caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) · [OpenAI Prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching) · [Gemini Context caching](https://ai.google.dev/gemini-api/docs/caching)。⚠️ 广泛流传的"Gemini 显式缓存 75% 折扣、最小块 32K"在官方当前文档中**找不到对应表述**，视为未证实，不要写进容量模型。

**Anthropic 特有的三条硬约束（考点密集）**

1. **失效是级联的（cascading invalidation）**，渲染顺序 `tools → system → messages`：改 tool 定义 ⟹ 整条全废；改 system ⟹ system+messages 失效；改 `tool_choice` / 增删图片 ⟹ 仅 messages 失效。
2. **最多 4 个 breakpoint，每个只回看 20 个 content block。** Agent 单轮塞进 >20 个 `tool_use`/`tool_result` block 时下一轮**静默 miss** —— 没有异常、没有日志，只有账单。
3. **并发同前缀请求全部 miss**：缓存条目要等第一条响应开始流式输出后才可读。**N 路 fan-out 同时发出 = N 次全价写。** 正确做法：先发 1 条、等到首 token，再发其余。

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

**c) 检索量** —— 一阶段 top-100 → rerank 压到 30–50 → 实际塞 top-20（[Anthropic Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) 口径，top-20 优于 top-5/top-10）。⚠ 长上下文 vs RAG 经济性差距按二手估算约 **1,250×**（100K token 请求仅 input ≈ $0.20，同等 RAG query ≈ $0.00008），量级参考。

**服务端上下文编辑**（Anthropic，截至 2026-07 仍为 beta，header `context-management-2025-06-27`）：

```
clear_tool_uses_20250919:
  trigger 默认 100,000 input tokens │ keep 默认 3 个 tool_use/tool_result 对
  clear_at_least 最少清理多少 token —— 存在的唯一理由就是"防击穿 prompt cache"
  exclude_tools 如 web_search、memory 类 │ clear_tool_inputs 默认 false
clear_thinking_20251015: 组合使用时必须排在 edits 数组首位
预演：count_tokens 端点支持带 context_management，返回清理前后的 input_tokens
```

官方实测：100 轮 web search 场景 token **−84%**；memory tool + context editing 组合 **+39%** 性能，单独 **+29%**。

**这一节最重要的一句话：压缩和缓存是对立的。** 改写历史 = 前缀变了 = 整条缓存作废，下一轮按 1.25× 重写全部前缀。所以 `压缩净收益 = Δ_saved × 剩余轮次 × P_in × 0.1 − L_prefix × P_in × 1.25`。代入 §2 参数（L₀=12,000，Δ_saved=20,000）：重建代价 ≈ $0.075，每轮省 ≈ $0.010 ⟹ **剩余轮次 ≥ 8 才划算**。这就是 `clear_at_least` 必须设得足够大的原因 —— 小打小闹的清理是净亏损。

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
| **并行工具调用（parallel tool calls）** | 一轮内发多个 `tool_use`。⚠ 注意 20-block 窗口；**并行只用于"读"和"评"，"写"必须单线程** | 轮次 −30~50% |
| **确定性工作出模型** | 分页、排序、字段过滤、格式转换、单位换算用代码做 | 消灭"模型算错再重试"的整轮 |
| **明确终止条件（termination condition）** | MAST 分类里"不知道终止条件"占 **12.4%**、"步骤重复"占 **15.7%** | 消灭长尾会话 |

> **面试金句**
> "轮次削减是我在 agent 系统上做过 ROI 最高的一类改动，因为成本对轮次是二次的。我们把 12 个细粒度文件工具合并成 3 个带批量参数的工具，中位轮次从 15 降到 9 —— 轮次只降了 40%，累计输入 token 降了 54%，端到端延迟降了一半还多。代价是 schema 变复杂、模型偶尔传错参数，所以同时加了参数校验和一次性纠错提示，这部分开销远小于省下的。"

---

## 7. 模型路由与级联

```
路由 Router ：classify(q) → 一次调用
  E[C] = p_small·C_small + (1−p_small)·C_large   延迟：+ 路由器耗时，P99 不变
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

**三条工程纪律**：① 路由决策做成可一键下线的旁路（bypass，feature flag），出问题时 30 秒退回全走大模型；② **先埋点（instrumentation）后路由** —— 在全走大模型的状态下用影子流量（shadow traffic）跑小模型 + judge 对比，统计"多少比例小模型也能过"，低于 30% 就不值得做；③ **路由维度不只是模型**，还有 thinking budget、是否开工具、是否开检索。关掉 thinking 往往比换模型省得更多且质量损失更小 —— HAL 的 21,730 rollout 实验结论是**多数运行中提高 reasoning effort 反而降低准确率**。

---

## 8. 批处理档位与输出长度控制

| 档位 | 折扣 | 形态 | 适用 |
|---|---|---|---|
| 同步 | — | 阻塞 | 用户在等 |
| **OpenAI Flex** | 对齐 Batch 费率，**可叠加 caching** | **保持同步调用形态** | 能等几分钟的后台任务；更慢、更易超时、可能 429（**429 不计费**），SDK 默认 10 分钟超时需调大。beta、模型有限（[文档](https://developers.openai.com/api/docs/guides/flex-processing)） |
| **Batch** | **三家一律 50%**（输入+输出） | 提交 → 轮询 → 取结果 | 离线 eval、批量分类打标、embedding 回填、夜间报告 |
| **Fast mode**（反向档） | **2× 标准价**换 ≤**2.5×** 输出 tok/s | 仅第一方 API，**不能与 Batch 同用** | 只在感知延迟直接等于产品价值时成立（demo、实时语音） |

Batch 边界（2026 年中）：Anthropic 100,000 请求 / 256 MB、24h 过期、结果留 29 天；OpenAI 50,000 / 200 MB、`completion_window` **只能填 `24h`**、**独立速率池不占同步配额**；Gemini inline ≤20MB / 文件 ≤2GB、任务 48h 过期、结果留 6 周。

两个直接可用的结论：① **Batch 与 caching 可叠加** —— Opus 5 极值 `$5 × 0.1 × 0.5 = $0.25/M`，是同步未缓存价的 **1/20**，跑大规模 eval 时这是唯一正确姿势；② **batch 里别用 5 分钟 TTL** —— batch 常跑超 5 分钟，官方建议用 **1h TTL**，命中率是 best-effort（官方实测区间 **30%–98%**，波动极大），`max_tokens: 0` 的缓存预热（cache warming）技巧**在 batch 内不支持**。

**输出长度控制**。输出单价是输入的 **5×**（Opus 5：$25 vs $5），**不可缓存**，还直接决定 wall-clock —— 它同时是成本项和延迟项的乘数。

- **结构化输出 / 约束解码（constrained decoding）**：[XGrammar](https://arxiv.org/pdf/2411.15100) 已是 vLLM / SGLang / TensorRT-LLM 的默认后端（据称 2026-03 起），JSON Schema 场景 **< 40 µs/token**；合规率从 **61.1%–72%** 提到 **96%–100%**（[JSONSchemaBench](https://arxiv.org/pdf/2501.10868)，10K 真实 schema）。⚠ 但它不免费：一次性编译 **20–50 ms**，**每请求换新 schema 是最坏用法**；[SqueezeBits 实测](https://blog.squeezebits.com/guided-decoding-performance-vllm-sglang)显示 **vLLM 在 batch ≥ 8 时吞吐明显下降**（掩码串行生成成瓶颈），SGLang 把掩码生成与推理步重叠、损失极小。
- **`max_tokens` 按 tokenizer 重标定**：Claude 4.7+ 新 tokenizer 对同样文本产生约 **+30% token**。跨代升级时"单价不变 = 成本不变"是错的，`max_tokens` 与 compaction 阈值都要重标，否则会出现"同样任务突然被截断"。**跨代成本对比先跑 `count_tokens`。**
- **禁复述（no regurgitation）**：系统提示里写死"不要复述工具结果、不要重复用户输入、直接给结论"。Agent 最常见的浪费是把刚读到的 200 行文件在回复里重抄一遍。
- **thinking budget 是旋钮不是开关**：抑制过度思考后 Step Success Rate 从 **0.251 → 0.302**（2026-06）。thinking token 按输出价计费。

---

## 9. 自建 vs API 的盈亏平衡

```
自建单位成本 ($/M) = 节点日成本 × 隐性倍数 ÷ (吞吐 tok/s × 86,400 × 利用率 ÷ 1e6)
盈亏平衡日 token 量 T* 满足:  T* × P_api = 节点日成本 × 隐性倍数
```

**算例**：单 H100 跑 70B FP8 + vLLM 连续批处理（continuous batching）≈ **400 tok/s ≈ 34.5M tok/天**（⚠二手口径）。H100 现货 ~$2.00–3.00/hr，取 $2.50 ⟹ 裸租金 **$60/天**。

| 成本口径 | 日成本 | 对标 $25/M（Opus 5 输出） | 对标 $5/M | 对标 $1.20/M |
|---|---|---|---|---|
| 裸 GPU 租金 | $60 | 回本需 **2.4M tok/天** | 12.0M tok/天 | 50M > 上限 ⟹ **不可能** |
| ×3 隐性（工程/冗余/闲置） | $180 | 7.2M tok/天 | 36M > 上限 ⟹ **不可能** | 不可能 |
| ×5 隐性 | $300 | 12M tok/天 | 不可能 | 不可能 |

> 物理上限 34.5M tok/天（100% 利用率）。隐性倍数（hidden-cost multiplier）3–5× 来自二手汇总：专职工程 $5,000–15,000/月、冗余副本、闲置时段、可观测性栈。

**利用率（utilization）敏感性 —— 这张表就是自建决策的全部**（8×H100 跑 671B MoE，⚠二手，2026-06-17）：

| 利用率 | 100% | 50% | 20% | **5%** |
|---|---|---|---|---|
| 单位成本 | $10 / M output | $20 / M | $50 / M | **$200 / M** |

真实业务有日夜波峰谷，实际平均利用率很少超过 30%。

**这个话题存在明确争议**：一派算出 70B 单 H100 对标 $5/M 旗舰约 **12.17M tok/天**回本；另一派（2026-06-17）算出 8×H100 跑 671B MoE 满载 ≈ **$10/M output**，比 DeepSeek V4 Pro API 的 $0.87/M **贵 11 倍**。**根因不是谁算错了，是对标基线不同**（溢价旗舰 vs 廉价开源托管）。

> **面试金句**
> "自建 GPU 划不划算，取决于你对标哪个模型，而不是取决于 GPU 便不便宜。对标 $25/M 的旗舰输出，一台 H100 每天跑 2–12M token 就回本；对标 $1.20/M 的小模型，你把卡跑满也回不了本，因为裸租金口径的单位成本就已经是 $1.74/M。所以我的第一个问题不是'GPU 多少钱'，是'这个 workload 用得起最便宜的哪个模型'。"

**什么时候自建才对**：① 数据不能出网/出境（合规硬约束）；② 单一稳定高利用率 workload（批量 embedding、离线打标、固定分类）；③ API 上没有你需要的模型。**"省钱"单独不构成理由。**

**蒸馏 / 微调**同理。收益（⚠均为二手）：**5–30×** 单位成本下降；13B 微调在与前沿模型差 2 个准确率点内时，单 token 便宜 **12–40×**；单点案例给出投入 $35k–120k、回本约 **2.9 个月**。真实代价常被漏掉：**先得有评测体系**（LLM judge 起步就要 100+ 标注样本 + 每周维护）、几千到几万条训练对、**模型漂移（model drift）**（上游迭代后你的蒸馏模型不跟着变好，6–12 个月要重做）、一整套新 serving 栈（见 [01-llm-serving-infra.md](01-llm-serving-infra.md)）、一个长期 owner。**判据：任务窄 + 量大 + 稳定 ≥ 6 个月，三者同时成立才做。** Agent 主循环通常不满足（提示和工具每周都在改）；适合蒸馏的是**边缘子任务**：意图分类、字段抽取、重排（reranking）、query 改写、安全审查。

---

## 10. 延迟优化清单

**TTFT（按投入产出排序）**

| # | 手段 | 收益 | 备注 |
|---|---|---|---|
| 1 | **前缀缓存 / prompt caching** | 长输入下可降**两个数量级** | 先做这个 |
| 2 | **缓存感知路由（cache-aware routing）（会话粘性 session affinity）** | 极端对照 P90 TTFT **0.542 s**(precise) vs 31.083 s(approximate) vs **92.551 s**(random) ⟹ **57× / 170×** | 8 pod / 16×H100 / 150 租户 × 6k ctx。**这让 LLM 网关变成有状态路由（stateful routing）** |
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
| **$/成功任务** | 唯一和价值挂钩的。$/请求会奖励"失败得快" |
| $/活跃用户日 | 对齐订阅定价 |
| $/租户月 + 毛利 | 对齐商业模型 |
| **失败运行的花费占比** | 生产遥测 **14.2%** —— 这是"免费"的 14% |

**参考锚点**（[Claude Code 官方成本文档](https://code.claude.com/docs/en/costs)）：**$13 / 开发者 / 活跃日**，**$150–250 / 月**，**90% 用户 < $30 / 活跃日**；空闲期后台 token 通常 < $0.04/会话。⚠ 另有二手引用称 $6/日、P90 < $12/日，可能是不同时间点或人群，以官方现行文档为准。

**长尾（long tail）：P99 用户可以是 P50 的 20 倍。** 这是所有按人头定价的 AI 产品的核心风险，放大来自三处**相乘**：

```
会话时长 ×2–3（轮次更多，而成本对轮次是二次的） × 编排模式 ×7–15（agent teams(plan)
≈ 普通会话 7×；多 agent ≈ 15×） × 失败重试 ×1.2–1.5  ⟹  P99/P50 ≈ 20×
官方 P90/P50 才 30/13 ≈ 2.3× —— 尾巴在 P90 之后才真正抬起来
```

**成本分布是重尾的（heavy-tailed），均值毫无意义。** 定价用均值、容量规划用均值，两个都会翻车。行业背景：AI 产品平均毛利（gross margin）约 **52%**（ICONIQ 2026，n≈300）vs 传统成熟 SaaS **80–90%**；推理成本占总支出从 **20% 升到 23%**。

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

**什么时候不要优化**：① 月账单 < 一个工程师 1–2 周的成本（约 $5k–15k）时，优化是负 ROI，先去做产品；② 还在找 PMF、提示每周都在改时，不要蒸馏、不要自建、不要做复杂路由（这些的前提是接口稳定）；③ 质量还没到线时，任何降本都是在给一个没人用的产品省钱 —— 先把 eval 建起来（见 [06-evaluation-and-observability.md](06-evaluation-and-observability.md)）。

| # | 反模式 | 真相 |
|---|---|---|
| 1 | **优化沙箱** | 沙箱只占 **11–15%**，推理占 85–89%，存储 ~0。最常见的方向性错误 |
| 2 | 不监控 `cache_read_input_tokens` | 静默 miss 无异常无日志，只有账单。**必须是指标，不是排查手段** |
| 3 | 只看 `input_tokens` | 总输入 = 三项之和，只看第一项会低估一个数量级 |
| 4 | N 路 fan-out 同时发同前缀 | = N 次全价写。先发 1 条等首 token |
| 5 | agent 单轮 > 20 个 content block | 触发 20-block 回看限制，**账单暴涨但没有任何报错** |
| 6 | batch 里用默认 5 分钟 TTL | Batch 常超 5 分钟，基本无效。改 1h TTL |
| 7 | 会话中途增删工具 / 切模型 | 工具定义渲染在位置 0，任何改动使整条缓存失效 |
| 8 | 跨代升级假设"单价不变 = 成本不变" | Claude 4.7+ 同文本多约 **30% token** |
| 9 | 把长上下文当免费 | Anthropic 1M 无溢价，但 **Gemini 3.1 Pro 超 200k 后输入 ×2、输出 ×1.5** |
| 10 | **前缀缓存无条件收益** | 存在争议：一派实测吞吐 **+30–50%**，另一派实测 **−36.7%**、TPOT +25%。差别在前缀重叠率（prefix overlap rate）与并发数 —— **上线前必须先量出重叠率** |
| 11 | 语义缓存不分租户 | 语义相似 ≠ 答案相同；跨租户复用是数据泄露 |
| 12 | 用流式冒充延迟优化 | 不降 wall-clock、不降成本 |
| 13 | 把路由的 85% 当均值 | 那是偏易基准的上限；生产 40–80%，且**级联抬高 P99** |
| 14 | 把"会话空闲后第一条消息"当普通请求 | 缓存 TTL 过期后**全量重算**，长会话里可能是最大单笔支出。对策：`max_tokens: 0` 预热、1h TTL、或从摘要重建 |

**必须显式写出的一个张力：缓存是最大杠杆，也是最大泄露面。**
杠杆侧 —— agentic KV 复用把命中率从 1.7% 拉到 92.2%，P50 TTFT **↓46×**、端到端 **↓8.6×**、吞吐 **↑3.8×**（GB200×60，2026-05；⚠ 代码未合入主干）。泄露侧 —— PROMPTPEEK（NDSS 2025）实证共享 prefix cache 可被**逐 token 重建他人 prompt**，无背景知识成功率 **95%**。

结论只能是 **同租户内共享、跨租户默认关闭** —— 而这会直接削掉多租户场景里最大的性能杠杆。这不是可以两全的取舍，是必须在设计评审上明说的成本。详见 [07-agent-security.md](07-agent-security.md)。

---

## 面试官会追问

1. 你的 agent 一次运行成本 $2，让你降到 $0.6，按什么顺序动手？为什么不先换便宜模型？
2. 为什么 agent 的成本对轮次是二次的？写出累计输入 token 的公式。
3. 缓存写乘数 1.25×（5m）/ 2×（1h），读 0.1×。回本点各是读几次？推导给我看。
4. 你上线了上下文压缩，账单反而涨了 10%。可能是什么原因？
5. 团队要做级联（Haiku 先做、Opus 兜底）。什么条件下这会比直接用 Opus 更贵？给临界公式。
6. 缓存命中率从 85% 掉到 5%，没有任何报错。列出你会依次检查的 5 件事。
7. 老板问"我们自建 GPU 能省多少"。你需要先问清楚哪三个数才能回答？
8. P50 用户成本 $13/日，P99 $260/日。定价怎么设计才不会被重度用户拖垮毛利？
9. 前缀缓存既是最大性能杠杆又是跨租户泄露面。多租户平台上你怎么处理这个取舍？

---

**下一篇** → [01-slo-and-error-budget.md](../05-reliability/01-slo-and-error-budget.md)
