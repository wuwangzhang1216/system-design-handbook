# 04 · AI Agent 系统：从 AI 应用 Full Stack 到生产级 Agent

这一章默认你会写普通 Web 服务，也调用过一次 LLM API；**不默认你会 CUDA、分布式训练或 GPU 调度**。
如果你的目标是把 Agent 产品做对、做稳，先走应用主线。只有准备自托管模型或面试 Inference / ML Systems 岗位时，才需要把 01 当作必修。

---

## 先选阅读路径

### 路径 A：负责 AI / Agent 应用的 Full Stack（推荐）

这里的 “Full Stack” 特指**职责里包含 LLM / RAG / Agent 的应用工程师**，不是根目录路径 A 的普通 Web 全栈。若你的岗位不负责 AI 功能，通用系统设计主线会跳过本章。

1. [02 · Context Engineering 与 RAG](02-context-engineering-and-rag.md)：先理解一次请求里模型实际能看到什么。
2. [07 · Agent 安全](07-agent-security.md) §1–§5：在接入工具和外部数据之前，先建立“数据不等于指令、权限不能继承”的安全边界。
3. [03 · Agent Runtime](03-agent-runtime.md)：把模型调用变成有状态机、超时、幂等和恢复能力的服务。
4. [06 · 评测与可观测](06-evaluation-and-observability.md)：知道怎样证明它工作，而不是只看几个成功 demo。
5. [08 · 成本与延迟](08-cost-and-latency.md)：先用业务 SLO、质量门槛和单位经济决定哪些优化值得做。
6. [04 · 记忆与状态](04-agent-memory-and-state.md)：在已经有写入门禁和评测之后，再加入长期记忆。
7. [05 · Multi-Agent](05-multi-agent-orchestration.md)：只有单 Agent 已经撞到并行或上下文隔离瓶颈时再拆。
8. [01 · LLM Serving Infra](01-llm-serving-infra.md)：**自托管进阶选修**；只调用模型 API 的读者可以跳过。

### 路径 B：自托管 LLM / Inference Platform

先读 [01 · LLM Serving Infra](01-llm-serving-infra.md)，再读 [08 · 成本与延迟](08-cost-and-latency.md)，然后进入
[08 · ML Systems](../08-ml-systems/) 的 03–05 与 09。应用层仍建议补读 02、03、06、07，避免只优化 GPU 而漏掉质量与安全。

---

## 本章的安全不变量

后文无论讨论 RAG、工具、记忆还是多 Agent，都默认下面四条成立：

1. **外部内容只能提供数据，不能自动获得更高指令优先级。** 网页、邮件、检索片段和工具返回值都可能带有恶意指令。
2. **模型建议不等于授权。** 写生产、发消息、花钱、读取跨租户数据等能力必须由确定性代码校验权限与参数。
3. **读和写分开建模。** 并行读通常安全；写操作要声明冲突键、幂等键、审批策略和补偿方式，不能仅凭工具名字判断。
4. **影子与评测流量必须去副作用。** 不返回给用户不代表风险为零；影子执行也可能写外部系统、消耗配额或复制敏感数据。

完整威胁模型见 [07 · Agent 安全](07-agent-security.md)。

---

## 怎样理解文中的数字

本章同时包含三类信息，阅读时不要混在一起：

| 类型 | 含义 | 应该怎么用 |
|---|---|---|
| **稳定概念** | 幂等、deadline、信任边界、Little's Law、评测分层等不依赖厂商的原理 | 可以迁移到自己的系统 |
| **场景假设** | 某个模型、流量、SLO 下的算例 | 只用于学习推导；替换成自己的输入重算 |
| **日期快照** | 截至某日的模型、GPU、价格、框架版本或厂商能力 | 选型前重新核对，不当作长期事实 |

文中的阈值和比例若没有统计推导，默认都是**起始假设而不是行业标准**。生产决策应以自己的 workload、质量基线和端到端压测为准。

---

## 八篇分别解决什么

| 篇 | 核心问题 | AI 应用路径 |
|---|---|---|
| [01](01-llm-serving-infra.md) | 自托管模型怎样做显存、批处理、路由与 GPU 容量规划 | 选修 |
| [02](02-context-engineering-and-rag.md) | 怎样把正确上下文交给模型，并知道检索何时失败 | 是 |
| [03](03-agent-runtime.md) | 怎样让工具调用可终止、可恢复、可审计 | 是 |
| [04](04-agent-memory-and-state.md) | 哪些状态该保存、何时写入、怎样遗忘与防投毒 | 按需 |
| [05](05-multi-agent-orchestration.md) | 什么时候拆 Agent，怎样传上下文和汇总结果 | 进阶 |
| [06](06-evaluation-and-observability.md) | 怎样离线评测、线上追踪并发现静默退化 | 是 |
| [07](07-agent-security.md) | 怎样隔离不可信输入、权限、数据与执行环境 | 是，且前置 |
| [08](08-cost-and-latency.md) | 怎样拆成本/延迟并按 ROI 优化 | 是 |
