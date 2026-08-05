# 02 · 上下文工程与检索

> 上下文窗口（context window）不是容器，是**预算**。它有限、会腐化、且每多放一个 token 都在稀释其他 token 的注意力。
> RAG 没有死，它降格成了上下文工程里的一个子操作：`select`。

---

## 读这一章之前

**你在工作中遇到过这个**

你给客服 Agent 接了知识库，内部测试问什么答什么，准确率 92%。上线后有个用户连着聊了 20 轮，
第 21 轮问了一个第 3 轮就答对过的问题，这次答错了 —— 而正确答案一字未改地躺在上下文的第 4 万个 token 上，
你甚至能在日志里把它高亮出来。你的第一反应是"给的材料不够"，于是把检索的 top-k 从 5 调到 20，
结果准确率又掉了 4 个点。

**需要先懂的概念**

| 概念 | 一句话 | 详见 |
|---|---|---|
| 缓存的三条代价 | 数据会旧、多一个会挂的组件、失效逻辑是新的 bug 源 | [00-concepts §11](../00-foundations/00-concepts.md#11-三个最常见的优化手段各在优化什么) |
| 索引在优化什么 | 用额外维护的结构，把"全部翻一遍"变成"直接定位" | [00-concepts §11](../00-foundations/00-concepts.md#11-三个最常见的优化手段各在优化什么) |
| p99 / 尾延迟 | 排序后第 99% 位置的耗时；串成多跳之后会被放大 | [00-concepts §3](../00-foundations/00-concepts.md#3-为什么平均值是骗人的p50--p90--p99) |
| TTFT 与 prefill | 到第一个字的时间，主要由"读完你的输入"这一段决定 | [01 §1](01-llm-serving-infra.md#1-两个阶段本质上是两台不同的机器) |
| 前缀缓存 | 请求开头完全相同的那段只算一次，后来者直接复用 | [01 §6](01-llm-serving-infra.md#6-前缀缓存与缓存感知路由网关设计的真正约束) |
| 多租户隔离 | 同一套系统服务多个客户时，数据绝不能串 | [02/03](../02-architecture-patterns/03-multi-tenancy.md) |

**这一章要回答的问题**

1. 窗口是 200K，我实际能用多少？为什么塞满会让准确率**下降**而不是持平？
2. 系统提示、工具定义、检索结果、对话历史各该放在提示词的哪一段？放错位置成本差多少倍？
3. 1M 上下文的模型已经有了，RAG 还有必要吗？用成本和质量两条线分别怎么论证？
4. 检索指标全绿、答案还是错的，怎么判断问题出在检索侧还是生成侧？

**本章新引入的术语**

| 术语 | English | 一句话定义 |
|---|---|---|
| 上下文工程 | context engineering | 决定每一次推理时往窗口里放哪些内容、什么时候把哪部分挪出去的整套策略 |
| 上下文腐化 | context rot | 同一道题，输入越长模型答得越差，而且远在窗口填满之前就开始 |
| 干扰片段 | distractor | 看起来切题、实际不含答案的片段；它出现在窗口里会拉低准确率 |
| 分块 | chunking | 把长文档切成若干小段，分别建索引、分别召回 |
| 稠密检索 / 稀疏检索 | dense / sparse retrieval | 把文字变成一串数字后找数值相近的 / 按关键词字面命中来打分（BM25 是代表） |
| 混合检索 | hybrid retrieval | 上面两种同时跑，再把两份排名合并成一份 |
| 重排 | rerank | 用一个更贵但更准的模型，对第一轮召回的几十条候选重新排序 |
| 召回率 / 精确率 | recall / precision | 该找到的里有多少被找到了 / 找回来的里有多少是真的相关 |
| 压缩 | compaction | 会话太长时把中间过程摘要掉、腾出窗口空间的那个动作 |
| 稳定前缀 | stable prefix | 提示词开头逐字节不变的那一段；只有它能被前缀缓存复用 |
| 失效级联 | invalidation cascade | 改动靠前的内容，会让它后面所有内容的缓存一起作废 |
| 上下文污染 | context pollution | 与后续决策无关的内容（大头是工具输出）挤占窗口、稀释注意力 |

---

## 1. 范式位移：从"怎么写提示词"到"往窗口里放什么"

[Anthropic 的官方分界（2025-09-29）](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)值得逐字记住：

| | 定义 | 时间尺度 |
|---|---|---|
| **Prompt engineering** | 编写和组织 LLM 指令的方法 | 单次调用 |
| **Context engineering** | **在 LLM 推理期间策划并维护最优 token 集合的策略集合** | 跨多轮、跨工具、跨会话 |

差别不是措辞。Prompt engineering 的对象是"一段文本"，context engineering 的对象是**系统提示（system prompt） + 工具定义（tool definition） + 外部数据 + 消息历史 + 记忆（memory）**这五者在每一步推理时的**组合与取舍**。

经典 full-attention Transformer 的注意力计算随序列长度近似按 n² 增长：任意 token 都要与其它位置计算相关度，长度翻倍时这部分计算约变四倍。
稀疏注意力、分块注意力和厂商优化会改变实际曲线，但不会消除另一个产品事实：**模型从更长、更杂的上下文里找到并正确使用关键信息会更难**。
因此“多一个 token 就机械地分走固定注意力份额”只是直觉比喻，不是注意力机制的精确工作方式；真正要用后面的任务评测衡量 context rot。

学术侧的确认：[《A Survey of Context Engineering for LLMs》(arXiv:2507.13334)](https://arxiv.org/abs/2507.13334)（166 页，综述 1400+ 篇）把它拆成 Context Retrieval and Generation / Context Processing / Context Management 三个基础组件，并指出一个核心缺口：模型**理解**复杂上下文的能力远强于**生成**同等复杂度长文本的能力。

> **面试金句**：
> "我不会说我们'做了 RAG'。RAG 是上下文工程里的 select 操作。完整的问题是：这一步推理的 token 预算是多少，其中多少给工具定义、多少给检索结果、多少给历史，以及什么时候把哪部分挪出窗口。只谈检索不谈预算，等于只谈缓存不谈淘汰策略（eviction policy）。"

---

## 2. 上下文会腐化（context rot）——这一节的实证决定了后面所有设计

[Chroma 的 Context Rot 研究（2025-07-14）](https://www.trychroma.com/research/context-rot)在 **18 个前沿模型**（GPT-4.1 / Claude 4 / Gemini 2.5 / Qwen3 各系列）上做了控制变量实验：只改输入长度，任务难度不变。

（实验范式叫 **needle-in-a-haystack**：把一句关键信息 —— needle —— 藏进一大堆无关文本 —— haystack —— 里，看模型能不能把它捞出来。下面的 needle / haystack / distractor 都指这套设定里的角色。）

结论（这些是你要背的）：

1. **性能随输入长度单调下降，而且远在窗口填满之前就开始。** 不存在"窗口 200K，塞到 190K 都没事"这回事。
2. **needle 与 question 的语义相似度（semantic similarity）越低，长度带来的退化越剧烈。** 精确匹配的检索题掩盖了真实退化。
3. **单个 distractor（似是而非的干扰片段）就能降低准确率，4 个 distractor 复合放大**，且不同 distractor 的伤害不均匀 —— 某些会稳定诱发幻觉（hallucination：模型用同样自信的语气编出一个上下文里根本没有、现实中也不成立的说法）。
4. **LongMemEval 上，~300 token 的聚焦 prompt 准确率显著高于 113,000 token 的完整 prompt**，所有模型家族一致。
5. **模型行为差异**：Claude 系倾向不确定时弃答（abstain），GPT 系幻觉率最高。
6. 重复词复制这类"零推理"任务在 **500–1000 词**之后开始明显崩坏。
7. **反直觉**：把 haystack 打乱，反而比保持逻辑连贯的 haystack 表现**更好**，18 个模型一致 —— 论文未给出解释。

⚠️ 网上广泛转述的"18 个模型准确率下降 30–50%"这个数字**在原文中并不以单一数字形式存在**，不要引用。

**第 7 条容易被误读。** 它不是说"随便乱放"，而是说：**连贯的干扰文本比零散的干扰文本更伤注意力**（模型更容易被一段读起来合理的错误上下文带走）。把"上下文排布顺序"当作**可 A/B 的变量**，而不是当常识去优化。**第 5 条更阴险**：同一套检索管线（retrieval pipeline）换个模型、指标变了，你可能会去调检索 —— 但真实原因是弃答/幻觉倾向不同。**换模型后必须重跑检索基线**。

---

## 3. 上下文预算表：把窗口当一等资源核算

反模式（anti-pattern）：把 200K 窗口当 200K 可用空间。正确做法是给每个段落**分配硬预算**，并在超预算时有明确的降级（graceful degradation）动作。

下面是一个 200K 窗口、工具型 Agent 的**起始模板**。比例不是行业标准；请用自己的任务分布、输出长度和 context-rot 评测重新标定：

| 段落 | 占比 | 200K 下的绝对值 | 缓存属性 | 超预算时怎么办 |
|---|---|---|---|---|
| 系统提示 / 角色 / 输出契约 | 2–4% | 4k–8k | **稳定前缀**（stable prefix），逐 token 不变 | 不能动；动了全窗口缓存失效 |
| 工具定义（JSON Schema） | 5–15% | 10k–30k | **稳定前缀**，排在 system 之前 | 按任务阶段做工具子集（tool subsetting，见 §4 isolate） |
| 长期记忆 / 项目规范 | 2–5% | 4k–10k | 准稳定，追加式 | 摘要化、外部化（externalize）到文件 |
| **检索结果** | 10–25% | 20k–50k | **每轮变化，绝不能放前缀** | 降 top-k、加 rerank、提高阈值 |
| 对话历史 + **工具结果** | 30–50% | 60k–100k | 只追加 | **优先清理工具结果，不是对话** |
| 当前用户输入 | <1% | 0.5k–2k | 变化 | — |
| 思考/输出预留 buffer | 15–20% | 30k–40k | — | 这是硬预留，不许侵占 |

三条规则：

**规则 1：不要等窗口快满才压缩。** 本算例在有效工作区（历史 + 检索）超过窗口 50% 时开始 compaction；40–70% 可作为首轮实验区间，而不是通用阈值。社区对 Claude Code 的逆向普遍认为 auto-compact 在容量 **~95%** 触发（⚠️ 官方未文档化，社区逆向值互相矛盾），这只能说明产品的兜底触发点，不代表你的质量最优点。

**规则 2：工具结果是最大的污染源（context pollution），不是对话历史。** 一次 `ls -R`、一次全文件读取、一次 API 返回的 JSON，动辄几千 token，且 90% 与后续决策无关。压缩的第一目标永远是工具结果。（社区对 Claude Code 的逆向给出"工具响应占 agent 总 token **67.6%**、系统提示 3.4%"—— ⚠️ 非官方、未核实，只当量级参考。）

**规则 3：预算要按步骤设，不是按会话设。** 一个 30 步的 Agent，如果每步只加 3k token，第 30 步就是 90k。把"单步最大新增 token"写进运行时约束。

---

## 4. 上下文的四种操作：Write / Select / Compress / Isolate

[LangChain 系统化的四操作](https://www.langchain.com/blog/context-engineering-for-agents)是目前最好用的心智模型。**工程顺序是固定的：先 select 再 compress**（先选对，再压缩；反过来是在压缩垃圾）。

| 操作 | 做什么 | 具体技术 | 触发阈值 |
|---|---|---|---|
| **Write** | 把知识写到窗口**之外**并持久化 | scratchpad 文件、`progress.md`、memory tool、结构化 feature list、git commit | 任何"下一步还要用但当前不需要"的信息 |
| **Select** | 决定哪些源进入窗口 | 检索（RAG）、工具子集、记忆召回、文件读取 | 每一步推理前 |
| **Compress** | 在关键事实结构化之后再压缩 | 工具结果清理、分层摘要（hierarchical summarization）、compaction | 工作区 > 窗口 50%，或 tool result 累计 > 3 万 token |
| **Isolate** | 按域/子代理切分上下文可见性 | 子代理（sub-agent）、沙箱（sandbox）、工具命名空间（tool namespace）、独立 MCP server | 上下文互相污染，或需要独立判断 |

**Write 的关键判据：长任务必须外部化状态。** 不要押在"单个会话/单个沙箱存活"上 —— 写 progress 文件 + git commit + 结构化 feature list。E2B 的会话硬上限是 Hobby **1 小时** / Pro **24 小时**（2026 年中量级，随时变动），你的任务可能比会话活得久。

### Isolate 的两个方向（互相矛盾，是真实张力）

- **需要延续决策线的角色**（继续写同一个模块的子代理）→ 应该继承上下文。
- **需要独立判断的角色**（代码审查、事实核查）→ 上下文**完全隔离**效果最好。Cognition 报告的 Code-Review-Loop 在"审查 Agent 与编码 Agent 上下文完全隔离"时，平均每个 PR 抓出 2 个 bug，其中约 58% 为严重问题。

同一个团队在 2025-06 和 2026-04 分别给出了"共享完整 trace"和"完全干净互不共享"两条相反建议 —— **它们没有被调和，这是按角色类型分流的问题，不是谁对谁错**。详见 [05-multi-agent-orchestration.md](05-multi-agent-orchestration.md)。

⚠️ **Isolate 的固有代价**：子代理回传是**有损摘要**（lossy summary），规模约 **1,000–2,000 token**。这是设计意图不是缺陷，但代价是父代理**无法复审原始证据**。当你需要审计链路（audit trail）时，isolate 是错的选择。

---

## 5. 检索流水线全景

```
                        ┌──────────── 离线：索引构建 ────────────┐
  原始文档 ──► 解析/清洗 ──► 分块 ──► [上下文化] ──► Embedding ──► 向量索引
   (PDF/HTML/         (表格、代码   │  50–100 tok    │  1024–3072d   │  HNSW/IVF-PQ
    Markdown/代码)     块要特判)     │  每 chunk      │               │
                                    │                └──► BM25 倒排索引
                                    └──► late chunking：先整篇 embed 再切
                        └───────────────────────────────────────┘

                        ┌──────────── 在线：查询 ────────────────┐
  用户 query ──┬──► [查询改写/HyDE]  ──┬──► BM25 ────► top-100 ──┐
               │                       │                          ├─► RRF 融合
               └──► Embedding ─────────┴──► ANN  ─────► top-100 ──┘   k=60
                                                                       │
                            ┌── 权限/租户过滤（必须在这之前或之内）──┘
                            ▼
                        候选 30–50  ──► Cross-encoder Rerank ──► top-10~20
                                          (100–200 ms)               │
                                                                     ▼
                        [组装：稳定前缀 | 历史 | ★检索结果放这里★ | 当前问题]
                                                                     │
                                                                     ▼
                                                                  LLM 推理
                        └───────────────────────────────────────────┘
```

图里出现的这些词，先各给一句话（后面几节会分别展开）：

| 词 | 一句话 |
|---|---|
| **Embedding** | 把一段文字映射成一串固定长度的数字（如 1024 个浮点数），语义相近的文字数值也相近 |
| **BM25** | 经典关键词打分算法：一个词在这篇文档里出现得越多、在整个语料里越罕见，得分越高；字面不出现就不可能命中 |
| **ANN**（近似最近邻） | 在几千万个向量里快速找出最接近的那几个，用一点准确率换几个数量级的速度；**HNSW**（分层图）和 **IVF-PQ**（先聚类再压缩）是两种主流索引结构 |
| **倒排索引** | 从"词 → 包含它的文档列表"的映射表，BM25 就跑在它上面 |
| **HyDE** | 先让模型对着问题**编**一个假答案，再拿这个假答案去做向量检索 —— 因为答案和答案更像，问题和答案不一定像 |
| **Cross-encoder** | 把 query 和候选文档**拼在一起**过一遍模型算相关度，比"各自算向量再点积"准得多，也贵得多，所以只用在最后几十条上 |
| **top-k** | 取排名最靠前的 k 条；`top-100 → top-20` 就是"先粗筛 100 条，再精排留 20 条" |

### 5.1 分块（chunking）：三条路线，成本差 8×

| 路线 | 做法 | 收益 | 代价 |
|---|---|---|---|
| 朴素切分 | 递归 512 token + overlap 64 | 基线 | 代词/跨引用丢失，chunk 脱离语境 |
| **Contextual Retrieval** | 用 LLM 给每个 chunk 生成 50–100 token 的文档级上下文，拼在 chunk 前 | 检索失败率 **−35%**（5.7%→3.7%）；+ contextual BM25 **−49%**；+ rerank **−67%**（→1.9%） | **$1.02 / 百万文档 token** 一次性成本；多一个 LLM 环节 |
| **Late chunking** | 先整篇过长上下文 embedding 模型拿 token 级表示，**再**切 chunk 做 mean pooling | 保住代词/跨引用；BEIR 增益**随文档长度增长而增长** | 需要长上下文 embedding 模型 |
| **Contextualized chunk embeddings** | 模型内置（voyage-context-4 自带 auto-chunking，示例 target 512 / overlap 64） | 省掉 LLM 环节，**$0.12/1M**（比 contextual retrieval 低约 8.5×） | **厂商锁定**（vendor lock-in）；基准全是厂商自评 |

[Anthropic Contextual Retrieval（2024-09-19）](https://www.anthropic.com/engineering/contextual-retrieval)仍是被引最多的基线，它的参数值得直接抄：chunk 数百 token（示例 800 token chunk / 8k token 文档），reranker **top-150 → 截 top-20**（top-20 优于 top-5/top-10）。上下文化成本可用 prompt caching 再降最多 90%。

⚠️ **不要把"top-20 优于 top-5"推广到无 rerank 的场景。** 那是 rerank 之后的结论。没有 rerank 时，多召回（recall）= 多 distractor = §2 的第 3 条。

### 5.2 Embedding 选型（2026 年中量级，随时变动）

| 模型 | 价格 / 1M token | 维度 | 备注 |
|---|---|---|---|
| [voyage-context-4](https://blog.voyageai.com/2026/06/29/voyage-context-4/) | $0.12 | 2048 / 1024 / 512 / 256（Matryoshka） | 内置 auto-chunking，>32K 文档自动切分 |
| Cohere Embed v4 | $0.12 | — | **128K token 输入窗口**，统一多模态 |
| Gemini Embedding 001 | $0.15 | 3072 → MRL 截 1536/768 | 价格与维度无关；英文 MTEB 均分 68.32 |
| OpenAI text-embedding-3-small | **$0.02** | 1536 | MTEB 62.26 —— 便宜 6.5× 的默认起点 |
| OpenAI text-embedding-3-large | $0.13 | 3072 | 比 small 贵 6.5×，MTEB 只高约 2.34 分 |
| Qwen3-Embedding-8B（自托管） | GPU 成本 | 4096 | MTEB 70.6，但**过拟合风险高** |

三条选型纪律：

1. **别用 MTEB 分数直接决定选型。**（MTEB / BEIR 是两套公开的检索与 embedding 评测基准合集，会给出一个跨任务的平均分。）榜单过拟合（overfitting：在这些公开题目上调得很好，换一份没见过的语料就掉下来）严重，多语言/长文档/领域词表现与英文均分脱节。
2. **Matryoshka 截断几乎是免费的**（Matryoshka / MRL：一种训练方式，让向量的前 N 维自己就能用，所以可以直接砍掉后面的维度而不必重新训练）：截到 **256 维**通常只损 **2–3%** 精度，存储降 **4×**。1 亿向量从 1536 维（约 600 GB）降到 256 维（约 100 GB），这是真金白银。
3. **embedding 缓存键必须包含 `model_id`** —— 换模型后旧向量全部作废，且**必须全量重建索引（reindex）**。把重建时间写进容量模型：1 亿 chunk × $0.02/1M token × 平均 300 token ≈ **$600** 的 API 费用 + 若干小时的索引构建。

### 5.3 混合检索（hybrid retrieval）与 RRF

**为什么必须混合**：dense 在**词法（lexical）精确**查询上系统性失效 —— 错误码、SKU、函数名/符号名、罕见实体、受控词表（controlled vocabulary）。embedding 会把 `ERR_4471` 映射到"错误码语义邻域"，召回一堆别的错误码。

**RRF（Reciprocal Rank Fusion）**之所以在生产胜出，是因为它 **scale-agnostic** —— 只吃排名不吃分数，绕开 cosine(0–1) 与 BM25(无上界) 的归一化不稳定问题：

```
score(d) = Σ_i  1 / (k + rank_i(d))        k = 60（行业惯例）
```

已在 Elasticsearch（`rrf` retriever）、[OpenSearch](https://opensearch.org/blog/introducing-reciprocal-rank-fusion-hybrid-search/)、Weaviate（默认融合）、Qdrant（`Fusion.RRF`）原生可用。

⚠️ **收益幅度存在争议，别对外承诺大数字**：部分 2026 文章称 hybrid 相对 dense-only NDCG **+26–31%**（未见可核对的原始实验）；而第三方实测给出的口径是**裸 RRF 相对 BM25 只 +1.3% NDCG，调优后才 +7.4~7.5%**。专利检索这种极窄领域，RRF(K=30) 的 NDCG@100 绝对增益只有 **+0.0094**（0.3475 vs 0.3381）。

**结论：领域越窄，hybrid 的绝对增益越小。** 收益基本来自 alpha/权重调参与领域适配（domain adaptation），而调 alpha 只需要 **~40 条**标注的 query-relevance 对 —— 这是整条流水线里 ROI 最高的 40 条标注。

convex 融合的 alpha 经验值（alpha = dense 权重）：**技术文档 0.3、混合语料 0.6、对话式 0.7–0.8**。

WANDS 基准上的三方对照：hybrid **0.7497** / BM25 0.6983 / 纯向量 0.6953。

### 5.4 Rerank：收益最确定的一环

一阶段召回（first-stage retrieval）top-100 → 压到 **30–50** 候选 → cross-encoder 重排 → 截 top-10~20。端到端 **100–200 ms**。

| Reranker（第三方榜，2026-02 量级） | ELO | 延迟 | 单价 |
|---|---|---|---|
| Zerank 2 | 1638 | 265 ms | $0.025 |
| Cohere Rerank 4 Pro | 1629 | 614 ms | $0.050 |
| Zerank 1 | 1573 | 266 ms | — |
| Voyage Rerank 2.5 | 1544 | 613 ms | $0.050 |
| Jina Reranker v3 | — | **188 ms**（Hit@1 81.33%） | — |

⚠️ 该榜的 nDCG@10 列（0.079–0.110）与 ELO 排序不一致，疑为相对增量口径 —— **不要引用它的绝对 nDCG**。延迟是唯一可以横向比的列，而且它很重要：600ms 的 reranker 在多跳（multi-hop：一个问题要连着检索好几轮才能答出来，比如"A 的供应商的竞争对手是谁"）Agent 里会被放大 10 倍。

### 5.5 Late interaction / ColBERT：先算存储再谈质量

**late interaction（延迟交互）** 指的是：不把整篇文档压成一个向量，而是**每个 token 各存一个向量**，检索时逐 token 比对再取最大值累加 —— 精度接近 cross-encoder，而重活可以离线做完。代价直接写在存储上。ColBERT 是它的代表实现，PLAID / MUVERA 是把它变得算得动的两代工程优化。

1M 文档规模的存储账：dense 单向量 ≈ **3 GB**；late interaction 无量化 ≈ **102 GB（34×）**；2-bit 量化 ≈ 6 GB；配 PLAID 后总体为 dense 的 **2–4×**（PQ / product quantization，乘积量化：把一个长向量切成若干小段，每段用一本码本里最接近的条目编号代替，存储降一到两个数量级）。

PLAID 相对 vanilla ColBERTv2 提速 GPU **2.5–7×** / CPU **9–45×**；[MUVERA（arXiv:2405.19504）](https://arxiv.org/html/2405.19504)相对 PLAID recall **+10%**、latency **−90%**，FDE 经 PQ 再压 32×。Weaviate 1.31 已提供 `Encoding.muvera(...)`，Qdrant/Vespa 亦有落地 —— **它已经从研究变成了配置开关**。

但默认答案仍然是：**先把 hybrid + rerank 调好，再考虑多向量（multi-vector）**。为了"看起来先进"上 ColBERT，代价是存储涨 34×。

---

## 6. Agentic retrieval：让 Agent 自己 grep，还是预先向量检索

这是 2025–2026 最重要的转向，也是最容易被过度概括的一个。

**争议的两边（都有一手证据，别选边）：**

- [《Is Grep All You Need?》(arXiv:2605.15184, 2026-05-14)](https://arxiv.org/abs/2605.15184)：LongMemEval 116 题子集，**grep 总体准确率高于 vector retrieval**。但论文自己强调：结果**强烈依赖使用哪个 harness 和何种 tool-calling 风格** —— 检索方法本身不决定结果。
- [LlamaIndex 基准（2026-01-13）](https://www.llamaindex.ai/blog/did-filesystem-tools-kill-vector-search)：小规模（5 篇 arXiv 论文，22–52 页）下文件系统 Agent correctness **8.4/10**、relevance 9.6/10、平均 **11.17 s**；传统 hybrid RAG correctness **6.4/10**、relevance 8.0/10、平均 **7.36 s**。**扩到 100 / 1000 篇后：RAG 在速度上大幅胜出、correctness 略胜、relevance 持平。**

**这不是矛盾，是规模分流。** 判据（decision criteria）表：

| 维度 | 选 Agentic（grep/ls/read） | 选预先检索索引 |
|---|---|---|
| 语料规模 | **数百 ~ 数千文档** | 数万 ~ 数亿 |
| 语料形态 | 文件系统上的文本/代码 | 异构、跨系统、有 ACL |
| 语料位置 | Agent 已有沙箱访问权 | 需要跨服务查询 |
| 延迟预算 | 宽松（多轮工具调用，10s+） | 紧（<1s 单跳） |
| 成本模型 | 多轮工具调用 token 成本高且不可预测 | 单次检索成本固定 |
| 新鲜度（freshness） | **实时**（读的就是当前状态） | 索引延迟（indexing lag，秒~小时） |
| 权限 | 靠文件系统权限兜底 | 必须在检索层做过滤 |

**grep 路线的两条硬假设，破了就崩：**
1. 语料以文本文件形式存在于文件系统上；
2. 规模在数百到数千文档量级，Agent 能自己挑文件而不被低信噪比（signal-to-noise ratio）淹没。

[Doug Turnbull 的补充](https://softwaredoug.com/blog/2026/04/06/agentic-search-is-having-a-grep-moment)很关键：grep 能用的真正原因**不是检索工具本身，而是 harness（承载 Agent 的那层运行时脚手架：怎么组装提示词、怎么调工具、怎么校验模型输出、什么时候停 —— 见 [03-agent-runtime.md](03-agent-runtime.md)）用 hook / 结构化输出（structured output）做验收校验在兜底（fallback）**。naive 的多轮工具调用最终会在 token 成本上翻车。

**工程结论**：单 repo / 单项目 → 文件系统工具；企业级异构语料 → 检索索引。**混合形态是最实用的**：向量检索定位到"哪几个文件/目录相关"，然后让 Agent 用 grep + read 在这个小范围里精确挖 —— 检索负责收窄搜索空间（search space），Agent 负责精确性。

---

## 7. 长上下文模型出现之后，RAG 的定位

**"1M 上下文出来了，RAG 可以扔了"是错的，有三个独立理由：**

1. **质量**：context rot（§2）在窗口填满之前就发生。
2. **成本**：一次 100K token 请求仅 input 约 **$0.20**，1M token 约 **$2/次**；同一个问题走 RAG 的检索 query 约 **$0.00008** —— 以 100K 那档相除是约 **2,500×**（⚠️ 二手估算，只当量级）。10 万 query/天：100K 长上下文约 **$20,000/天**（若每次 1M 则约 $200,000/天）vs RAG 检索 **约 $8/天**；生成、重排、存储与索引成本需另计。
3. **延迟**：1M token 的 prefill 即便有 8× 的 prefill 加速，TTFT 也在秒级。

**2026 的默认工程形态是混合**：

```
检索出 5 万 – 20 万 token 的相关内容  ──►  让长上下文模型在其上推理
       └─ 用检索解决"从 10 亿里挑 10 万"     └─ 用长上下文解决"跨文档整体推理"
```

纯 RAG 丢掉单文档的整体推理能力（chunk 化本身就是信息损失），纯长上下文会 rot。两个失败模式互补，所以混合。

**唯一的例外**：**小而稳定**的知识库（比如 3 万 token 的产品手册）+ prompt caching，全量塞入比建检索基础设施更快更便宜。但这个例外有硬边界 —— **一旦知识库频繁更新，或每个用户的文档集不同，缓存立即失效**，经济性瞬间翻转。

顺带一提定价：Claude 4.6 及以后的 **1M 上下文按标准价，无长上下文溢价**；而 Gemini 3.1 Pro Preview 是一线厂商里唯一有明确长上下文阶梯溢价的（≤200k **$2.00** / >200k **$4.00** 每百万输入，2026 年中量级）。这会直接改变你的"塞进去还是检索"的临界点（crossover point）。

---

## 8. GraphRAG：边界比宣传窄得多

**GraphRAG** = 离线时用 LLM 把语料里的实体和它们之间的关系抽出来，建成一张图（"公司 A ──供应商──▶ 公司 B"），检索时在图上走边而不是在向量空间里找邻居；再对图的社区做分层摘要，用来回答"总结一下全部……"这类问题。相对地，**flat RAG** 指前面几节那套"切块 → embedding → ANN"的平铺式检索。

**成本**：索引约为 flat RAG 的 **6–8×**，运行约 **3×**。GPT-4 档索引 **$20–50 / 百万 token**（换廉价模型可降约 10×，但抽取质量下降）。

**不要用 token 数设硬阈值。** 语料不到 100 万 token 时，完整 GraphRAG 往往需要更强的收益证据，但小语料若关系密、全局综合/多跳问题占比高，也可能值得；反之，大语料若主要是点查，flat RAG 仍可能更好。是否采用要用真实 query 集比较质量、索引/刷新成本与查询延迟，`100 万 token` 只能作首轮试验分桶，不能作 go/no-go 线。

**重要反证**（这是面试里能加分的一条）：[《RAG vs. GraphRAG: A Systematic Evaluation》(arXiv:2502.11371)](https://arxiv.org/html/2502.11371v3) 显示 GraphRAG 在 Natural Questions 上准确率比 vanilla RAG **低 13.4%**，时效性问题下降约 **16.6%**。

多跳场景它确实赢：DualGraphRAG 在 HotpotQA 上 F1 比 GraphRAG/NaiveRAG 高 **31.79% / 5.99%**。

**唯一站得住的模式是按 query 类型路由**：

```
query ──► 分类器 ──┬── 点查 / 局部事实 / 时效性 ──► vector + BM25 hybrid
                   ├── 全局综合（"总结所有客户的共性抱怨"）──► graph
                   └── 多跳关系（"A 的供应商的竞争对手"）──► graph
```

上 GraphRAG 之前先评估 **LazyGraphRAG**：索引成本约为完整 GraphRAG 的 **0.1%**（与普通 vector RAG 持平），global query 最多便宜 **700×**。

---

## 9. 上下文组装顺序与前缀缓存的硬约束

这一节把 [01-building-blocks/02-caching.md](../01-building-blocks/02-caching.md) §7 的结论落到具体参数上。**前缀缓存（prefix caching）是 2026 最大的单点性能杠杆**（不重算 > 算得更快），而它对上下文组装（context assembly）施加的是**硬约束，不是建议**。

**正确的组装顺序（从前到后）：**

```
[ 工具定义 ] [ 系统提示 ] [ 长期记忆/规范 ] │ [ 只追加的历史 ] │ [ 检索结果 ] [ 当前问题 ]
└────────── 稳定前缀，命中缓存 ──────────┘   └─ 追加式 ─┘   └── 每轮变化 ──┘
```

**厂商参数（2026 年中量级，随时变动）：**

| 项 | Anthropic | OpenAI | Gemini |
|---|---|---|---|
| 缓存读折扣 | **0.1×** 输入价 | GPT-5.x **10%**；gpt-4.1/o3 25%；gpt-4o 50% | — |
| 缓存写费 | 5m **1.25×** / 1h **2×** | GPT-5.6+ **1.25×**（此前免费） | — |
| 回本点（break-even point） | 5m 读 **1** 次；1h 读 **2** 次 | — | — |
| 最小可缓存前缀（cacheable prefix） | 2026 年中快照：512（Opus 5 / Fable 5）、1024（Sonnet 5 / Opus 4.8）、2048（Opus 4.7）、4096（Haiku 4.5） | 2026 年中快照：1024 tokens | 2026 年中隐式缓存快照：4096（3.5 Flash / 3.1 Pro）、2048（2.5 系） |
| 存活时间（TTL） | 5m / 1h | GPT-5.6+ 至少保留 30 分钟；更早模型 5–10 分钟不活动淘汰 | — |
| 存储费 | 无 | 无 | **$1.00 / 百万 token / 小时**（唯一有持有成本的） |
| 控制参数 | 最多 **4 个** breakpoint，每个回看 **20 个** content block | `prompt_cache_key`、`prompt_cache_options.mode` = `explicit`/`implicit` | — |

**Anthropic 的失效级联（invalidation cascade）顺序必须背下来**：`tools → system → messages`。

- 改一个 **tool 定义** → **全部失效**
- 改 **system** → system + messages 失效
- 改 `tool_choice` / 增删图片 → 仅 messages 失效

**这给两个常见设计加了约束：** ①工具子集仍然可以减少 token 与误调用，但不要每一步抖动；在**会话或任务阶段边界**选择一次工具集，并在该阶段保持稳定。②不要把每轮变化的检索结果插进已经准备缓存的稳定前缀或历史中间；它应追加到**现有历史之后、当前问题附近**。所有 message 形式上都会出现在 system 之后，真正影响缓存的是“变化内容是否插到了可复用前缀里面”。

**再叠一层成本**：缓存和 Batch 可叠加 —— Opus 5 的 batch + cache read = $5 × 0.1 × 0.5 = **$0.25/M**，是同步未缓存价的 **1/20**。⚠️ 另外注意 **Claude 4.7+ 的 tokenizer 对同样文本约多产生 +30% token**（Opus 4.7/4.8/5、Sonnet 5、Fable 5；Sonnet 4.6 及更早为旧 tokenizer），换模型时上下文预算表要按这个比例重算。

---

## 10. 压缩与摘要：什么时候触发、保留什么、丢什么

Anthropic 的 context editing 提供了可直接抄的参数（beta header `context-management-2025-06-27`，截至 2026-07 **仍是 beta**，[文档](https://platform.claude.com/docs/en/build-with-claude/context-editing)）：

| 策略 | 参数 | 默认值 | 建议值 |
|---|---|---|---|
| `clear_tool_uses_20250919` | `trigger` | 100,000 input tokens | **30,000–100,000** |
| | `keep` | 3 个 tool use/result 对 | 3 |
| | `clear_at_least` | — | **5000**（关键，见下） |
| | `exclude_tools` | — | `web_search`、memory 类 |
| | `clear_tool_inputs` | `false` | `false`（保留调用参数便于溯源 traceability） |
| `clear_thinking_20251015` | — | Opus 4.5+/Sonnet 4.6+ 默认保留全部；更早模型只留最后一轮 | 组合使用时必须排在 edits 数组**首位** |
| 服务端 compaction | `compact_20260112` | — | SDK 客户端 compaction（`compaction_control`）**已废弃** |

**实测收益**：100 轮 web search 场景 token 消耗 **−84%**；context editing 单独 **+29%** 性能，配合 memory tool **+39%**（⚠️ 单一 benchmark）。

**三个必须知道的坑：**

**坑 1：压缩会击穿 prompt cache 前缀，可能净亏钱。** 清理 tool result 改写了 messages 的中段 → 后面全部重算。cache read 便宜 90%，你省下的 token 数可能远抵不上失去的缓存折扣。**`clear_at_least` 就是为这个存在的**：一次至少清理够多的 token，让"重建缓存"这笔投资摊得开。必须做 **token 数与 cache 命中率（hit rate）的联合核算**，不是只看 token 数。

**坑 2：忘掉 `exclude_tools`。** memory、`web_search` 这类结果被清掉，Agent 会**反复重查** —— 成本不降反升，而且会在轨迹（trajectory）里表现为"步骤重复"（MAST 分类法里占 15.7% 的头号失败模式）。

**坑 3：压缩是不可逆的信息丢失。** 摘要掉的东西，Agent 之后**不知道自己不知道**。

**保留清单（摘要时必须逐字保留 verbatim，不许改写）：**

| 类别 | 例子 | 理由 |
|---|---|---|
| 标识符 | 文件路径、PR 号、commit SHA、订单 ID | 改写 = 后续操作打到错误对象 |
| 未完成的承诺 | "还需要更新 3 个调用点" | 丢了就是任务不完整 |
| 约束与否决 | "用户明确说不要改 schema" | 丢了会重犯 |
| 失败记录 | "方案 A 试过，报错 X" | 丢了会无限重试 |
| 用户原话中的需求 | 验收标准（acceptance criteria） | 摘要最容易偷偷改变语义的地方 |

**可以安全丢弃的**：中间工具输出的原始体、被否决方案的推导过程、已完成且已验证步骤的细节。

**预演（dry run）工具**：`count_tokens` 端点支持带 `context_management` 参数，返回 `original_input_tokens` 与清理后的 `input_tokens` —— 上线前用它把压缩策略跑一遍，别在生产上试。

---

## 11. 多租户（multi-tenancy）与权限过滤：检索层最容易出的合规事故

**三种做法没有脱离索引实现的唯一答案：**

| 做法 | 适用边界与代价 |
|---|---|
| **Post-filter**（先 ANN 取 top-k，再过滤租户） | 不能作为安全边界；top-k 可能全被过滤，召回质量也不稳定。若中间结果进入日志、缓存或调试面，还会扩大跨租户暴露面 |
| **Pre-filter / filtered ANN**（遍历时带 tenant/ACL filter） | 支持 payload filter、bitmap 或 filtered-HNSW 的引擎可以安全使用；过滤集合极小时要压测连通性、over-fetch 与 recall |
| **按租户分区/分集合**（partitioning） | 隔离最直观，适合高敏感或大租户；小租户很多时会产生索引碎片、运维与冷启动成本 |

常见默认是：共享索引内做**引擎级 pre-filter**，生成前再做一次权威授权检查；高风险或超大租户升级为独立分区。无论哪种拓扑，不能让 LLM 自己承担授权。

详见 [01-building-blocks/01-storage-engines.md](../01-building-blocks/01-storage-engines.md) §6 与案例 [06-case-studies/03-multi-tenant-vector-search.md](../06-case-studies/03-multi-tenant-vector-search.md)。

**另外四条纪律：**

1. **权限过滤（permission filtering）必须在检索层，不能交给 LLM 判断。** "在 prompt 里告诉模型只能用 A 部门的文档"不是访问控制（access control），是许愿。
2. **ACL（access control list，访问控制列表：记录"谁能访问哪个对象"的那张表）会变，检索结果会陈旧（staleness：你手上这份数据在权威源那边已经被改过了，而你还不知道）。** 检索时带上 ACL 版本号/快照时间，**生成前做一次二次授权检查**。用户 10:00 被移出项目组，10:01 的检索不能还命中该项目文档。
3. **跨租户共享 prefix cache 是已实证的泄露面（leakage surface）。** PROMPTPEEK（NDSS 2025）表明共享 prefix cache 可被逐 token 重建他人 prompt：已知模板时成功率 **99%**，**无任何背景知识也有 95%**，约 **60 次请求**即可套出用户属性。攻击面覆盖 vLLM、SGLang、LightLLM、DeepSpeed。
4. **由此产生本章最大的张力**：prefix cache 是最大的性能杠杆（§9），也是最大的跨租户泄露面。

> **面试金句**：
> "缓存策略只有一个安全的默认值：**同租户内共享 prefix cache，跨租户默认关闭**。我知道这会削掉多租户场景下最大的性能杠杆 —— 系统提示和工具定义那 3 万 token 本来是所有租户都一样的。我的处理是把'共享池'限制在**确实无租户数据的那一段**（工具定义 + 系统提示），并让它成为一个显式的、可审计的边界，而不是让整条前缀隐式共享。"

---

## 12. 检索评测：分层归因，否则你在调错东西

**第一步永远是分层归因（layered attribution）**，而不是看端到端答案质量：

```
答案错了
  ├─ gold chunk 在不在最终 context 里？
  │    ├─ 不在 → 检索问题：查 Recall@k / 分块 / 融合权重 / rerank
  │    └─ 在   → 生成问题：查 context rot / distractor / 提示词 / 模型选择
```

| 层 | 指标 | 本章示例起始线（必须按语料重标定） |
|---|---|---|
| 一阶段召回 | Recall@100 | > 95%（rerank 的天花板 upper bound 由它决定） |
| 融合后 | **Recall@10** | **85–91%** |
| | **MRR** | **> 0.80** |
| | Hit Rate@10 | **> 90%** |
| 排序质量 | nDCG@10 | 只做**同语料纵向比较**，绝不跨语料横向比 |
| 端到端 | faithfulness（有没有编）、answer relevance | 先用双人标注得到人—人基线，再按 prevalence、误放/误杀成本、样本量和置信区间为该 rubric 选择 judge 门槛；κ ≥ 0.6 只能作为某次首轮实验的候选起点 |

表里五个指标各一句话（它们都要求你先有一批标好"哪几条是对的"的题）：

- **Recall@k**：正确答案所在的那条，出现在前 k 条结果里的题目占比。**它是上限** —— 一阶段没召回来，后面 rerank 再准也救不回来。
- **Hit Rate@k**：和 Recall@k 常被混用；本书口径是"至少命中一条正确文档"的题目占比，Recall@k 则看正确文档被找回的比例。
- **MRR**（mean reciprocal rank，平均倒数排名）：第一条正确结果排在第 1 位记 1 分、第 2 位记 1/2、第 3 位记 1/3……全部题目取平均。**它只关心第一条对的排多靠前。**
- **nDCG@k**（normalized discounted cumulative gain）：既看相关度高低、又按位置打折（排得越靠后加权越少），再除以理想排序的分数归一化到 0–1。**绝对值只在同一份语料内部有可比性。**
- **LLM judge**：用另一个模型给答案打分。**κ**（Cohen's kappa）是它和人工标注的一致性系数，扣掉了“按边际分布碰巧一致”的部分；它会受类别比例和 rubric 影响，不存在跨任务的 `κ ≥ 0.6` 可信线。要同时报告混淆矩阵、样本量/置信区间，并和人—人一致性比较。

**数据集怎么选：**

- **公开集**（BEIR / MTEB / WANDS）只用来**排除明显烂的选项**，不能作为验收标准。榜单过拟合严重。
- **唯一可靠的做法是在自己语料上跑 A/B。** 厂商自评基准互相打架，且无法调和：Voyage 称 context-3 超 contextual retrieval 20.54%、超 Jina late chunking 23.66%（厂商自评）；另有第三方称 ContextualRankFusion 在 NFCorpus 上优于 late chunking；还有 2026-02 的厂商基准把最朴素的"recursive 512-token 切分"排在七种策略第一。
- **标注量的现实答案**：调 hybrid alpha 只需 **~40 条** query-relevance 对；建一个能上线的 LLM judge 需要 **100+ 标注样本 + 每周维护**。先做前者。

详见 [06-evaluation-and-observability.md](06-evaluation-and-observability.md)。

---

## 13. 什么时候不要用 RAG

| 情况 | 为什么 | 该用什么 |
|---|---|---|
| 知识库 **< 5 万 token 且稳定** | 全量塞入 + prompt caching 更快更便宜，还没有召回率损失 | 静态前缀 + 缓存 |
| 语料是**单 repo 的文本/代码**，规模数百~数千文件 | Agent 用 grep/ls/read 的 correctness 更高，且**实时**（§6） | 文件系统工具 |
| 问题需要**整篇文档的连贯推理**（合同审查、长报告总结） | chunk 化本身就是信息损失，检索永远拼不回全文逻辑 | 长上下文直读 |
| 需要**精确、完整**的结构化答案（"列出所有超期订单"） | 检索是 top-k 近似，天然**不保证完整性** | SQL / API，让 Agent 调工具 |
| 数据**高频变化**（分钟级） | 索引延迟 + embedding 重算成本；且缓存全废 | 实时查询工具 |
| 仅因为语料达到某个 token 数就上 GraphRAG | token 数不能代表关系密度或全局/多跳 query 占比，索引与刷新开销未必换来质量收益 | 用真实 query 对 flat RAG / GraphRAG 做质量、延迟与总成本对照；小而稳定的点查语料通常先用 flat RAG |
| 只是为了"看起来有 AI 架构" | 你会得到一套需要维护的索引管道、一个新的一致性问题、一个新的权限泄露面 | 别做 |

**最大的反模式：把 RAG 当成"记忆"。** 检索是无状态的 select 操作，它不知道 Agent 上一步做了什么。用向量库存对话历史然后指望它变成记忆，会得到一个既检索不准、又没有时序、还无法遗忘的系统。Agent 的记忆需要显式的 write 操作和状态机（state machine） —— 见 [04-agent-memory-and-state.md](04-agent-memory-and-state.md)。

**第二大反模式：先建检索基础设施，再想清楚 query 长什么样。** 分块策略、embedding 维度、融合权重全部依赖 query 分布。**没有 40 条真实 query 之前，你调的每一个参数都是猜的。**

---

## 这一章的三句话

1. **窗口是预算不是容器，多放一条不相关的内容是净负收益。** context rot 的实证说明退化在窗口填满之前很久就开始，而单个 distractor 就能拉低准确率 —— 所以"反正还塞得下，多给点材料"这个直觉是本章要杀掉的头号错误。
2. **上下文的排列顺序决定了它的价格，而不只是它的效果。** 稳定的东西（工具定义、系统提示、程序性记忆）放在可复用前缀且逐字节不变；每轮变化的检索结果和工具输出追加在历史之后，不要插进稳定前缀或历史中间。
3. **压缩之前先选对，检索之前先有 query。** 先 select 再 compress，否则你只是在精心压缩垃圾；而分块大小、embedding 维度、融合权重全都由 query 分布决定 —— 没有 40 条真实 query 之前，你调的每个参数都是猜的，而这 40 条标注是整条流水线里 ROI 最高的投入。

---

## 面试官会追问

1. 上下文窗口是 200K，你会用到多少？为什么不是 190K？你的预算表怎么分？
2. context rot 的实证结论有哪几条？"多召回一点没坏处"错在哪里？
3. Write / Select / Compress / Isolate 四个操作，为什么顺序必须是先 select 再 compress？
4. 压缩上下文为什么可能**反而更贵**？`clear_at_least` 解决的是什么问题？
5. 检索结果应该放在上下文的哪个位置？如果放在系统提示后面会发生什么？
6. 你的工具定义要不要按任务动态变化？说出代价的量级。
7. 什么规模下应该让 Agent 用 grep 自己找，什么规模下必须用向量索引？给判据。
8. 1M 上下文模型出来了，你还做 RAG 吗？用成本和质量两条线论证。
9. hybrid 检索能带来多少提升？你从哪里拿到这个数字？
10. 多租户下向量检索的权限过滤怎么做？post-filter 为什么是错的？
11. prefix cache 既是最大杠杆又是最大泄露面，你怎么取舍？
12. 检索指标好但答案还是错，你怎么定位是检索问题还是生成问题？

---

**按训练路径阅读** → 回 [START-HERE](../START-HERE.md) 按所选路径继续；页尾链接只表示本目录或专章的顺读顺序。

**AI 专章顺读下一篇** → [03-agent-runtime.md](03-agent-runtime.md)
