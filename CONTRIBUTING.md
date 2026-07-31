# 贡献指南

欢迎补充、纠错、更新。这本手册的价值取决于内容的**准确性**和**时效性** —— 尤其是
`04-ai-agent-systems/` 这一章，里面的事实每几个月就会过期。

## 什么样的 PR 最受欢迎

1. **纠错**：数字错了、机制描述错了、协议版本过时了。请附一手来源链接。
2. **更新前沿章节**：新的推理优化、协议版本、定价变化、被证伪的做法。
3. **补充失败模式**：你在生产里真实踩过的坑，比任何理论都值钱。
4. **补充反例**：某个"最佳实践"在什么情况下是错的。

## 什么样的 PR 会被拒

- 把内容改得更"入门"（本书明确面向 Senior/Staff，不做科普）
- 只有结论没有数字和代价的段落
- 从别处大段复制的内容
- 无来源的性能声称

## 写作规范

每一篇的结构是固定的：

```markdown
# NN · 标题
> 一句话的核心断言

---

## 1. ...
## 2. ...

---

## 面试官会追问
1. ...（6–8 条）

---

**下一篇** → [文件名](相对路径)
```

内容要求：

| 要求 | 说明 |
|---|---|
| 给数字 | "很快"是无效表述，"p99 约 2ms（同 AZ）"才是 |
| 给代价 | 每个方案必须写出它的坏处 |
| 给撞墙条件 | "到 X 规模会失效，信号是 Y" |
| 有反模式 | 每篇必须有一节讲"什么时候不要这么做" |
| 可执行 | SQL / 配置 / 伪代码 / 公式，而不是抽象描述 |
| 前沿内容带链接 | 2025 年之后的事实必须有一手来源 |

### 图的约定

**默认用 ASCII 画图。** 理由是四条，且它们叠加起来足以压倒"渲染出来更好看"：

| 理由 | 说明 |
|---|---|
| 白板可复现 | 面试时你要在白板上重画它。画不出来的图，对本书的读者没有价值 |
| 终端可读 | `cat` / `less` / `git diff` / GitHub 的 raw view 里都能直接看 |
| grep 友好 | 图里的状态名、字段名能被 `grep` 搜到，mermaid 里的也能，但 ASCII 的上下文对齐关系搜出来仍然可读 |
| 任何渲染器都能看 | 不依赖 mermaid 版本、不依赖渲染器是否开启该功能、离线可读 |

**只有两类图允许用 mermaid：时序图（`sequenceDiagram`）和状态机（`stateDiagram-v2`）。**
理由是这两类各有一件 ASCII 表达不了的事：

- **时序图 → 时间交错。** 多个参与者的消息在时间轴上谁压着谁，尤其是"A 还没写回去、B 已经删完了"这种竞态窗口，以及 `par` / `alt` 这种同一时刻的分叉。ASCII 只能画出"先后"，画不出"重叠"。
- **状态机 → 状态可达性。** 哪条边不存在、哪个状态没有出边、哪个状态看起来是终态其实不是。**画得出的边和画不出的边同样是信息**，ASCII 的箭头一多就会互相穿插到无法辨认。

**禁止用 mermaid 画 flowchart、架构图、拓扑图、部署图、类图、ER 图。** 这些用 ASCII 方框画，本书全部现有章节都是这么画的。

#### 正例

架构与拓扑 → ASCII：

```
   Client ──► LB ──► App ──► Cache ──► DB
                      │                 ▲
                      └── Outbox ──► Relay ──► Kafka
```

时间交错 → mermaid 时序图（这件事 ASCII 画不清）：

````markdown
```mermaid
sequenceDiagram
    autonumber
    participant L as Leader
    participant F1 as Follower1
    participant F2 as Follower2
    L->>F1: AppendEntries
    L->>F2: AppendEntries
    F1-->>L: ack
    Note over L,F1: 多数派已达成，此刻就 committed
    F2-->>L: ack arrives late
```
````

状态可达性 → mermaid 状态机（重点在"没有哪条边"）：

````markdown
```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Open: error rate over threshold
    Open --> HalfOpen: cooldown elapsed
    HalfOpen --> Closed: probes pass
    HalfOpen --> Open: one probe fails
```
````

#### 反例

❌ 用 mermaid 画流程/架构（应该用 ASCII 方框）：

````markdown
```mermaid
flowchart LR
    A[Client] --> B[LB] --> C[App] --> D[(DB)]
```
````

❌ 把旁边已有的 ASCII 图换个画法再画一遍。**宁缺毋滥** —— 如果这张 mermaid 图和上面的 ASCII 图传递的是同一份信息，删掉 mermaid 那张。

❌ 空洞的引导句和"读图要点"：

```markdown
下面这张图展示了整个流程，如下图所示：
> 读图要点：这张图很重要，请仔细阅读。
```

#### 硬性要求

1. **每张 mermaid 图前必须有一句引导句**，说清楚"这张图要让你看见什么，而这件事为什么上面的 ASCII / 表格 / 代码说不出来"。
2. **每张图后必须有一条以 `> 📖` 开头的"读图要点"引用块**，指向图里**具体的**边或步骤（"第 7 步和第 9 步的先后"、"`Shipped` 没有指回补偿态的边"），而不是复述图的内容。
3. **术语用英文，且必须与 [`07-interview/04-glossary.md`](07-interview/04-glossary.md) 第 2 列一致**（`origin fetch` 不是 `back to source`，`fencing token` 不是 `isolation token`）。图里的状态名 / 参与者名要和同一节正文、ASCII 图里的写法**逐字一致**，不要另起一套。
4. **复杂度上限**：时序图 `participant ≤ 7`，状态机状态数 `≤ 10`。超了就拆图或砍掉次要分支。
5. **图里的数字不能和正文/表格打架**（比如状态机写"连续 3 次成功"，而下面参数表写 `permittedNumberOfCalls = 5–10`）。
6. **提交前跑一遍解析**，语法错的图在 GitHub 上会渲染成一段红色报错：
   ```bash
   npx -y @mermaid-js/mermaid-cli -i 你的文件.md -o /tmp/out.md
   ```

## 事实的时效性标注

涉及价格、性能、版本号的内容，请标注时间和不确定性：

```markdown
（2026 年中量级，随时变动）
```

不确定的数字用"约 / 量级"限定。**宁可写"未找到公开数据"，也不要编造。**

## 提交

- 一个 PR 只做一件事
- commit message 用中文或英文都可以，说清"改了什么、为什么"
- 大改动请先开 issue 讨论
