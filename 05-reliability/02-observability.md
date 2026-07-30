# 02 · 可观测性

> 可观测性不是"装了 Prometheus + ELK + Jaeger"，是**你能不能在不发新代码的前提下回答一个你从没预想过的问题**。
> 它同时也是你云账单上通常排第二的那一项——仅次于计算。绝大多数团队为自己从来不查的数据付了 70% 的钱。

---

## 1. 四种遥测的分工

| | **Metrics** | **Logs** | **Traces** | **Profiles** |
|---|---|---|---|---|
| 数据形态 | 预聚合的数值时序 | 离散的结构化事件 | 因果关联的 span 树 | 栈采样的火焰图 |
| 单位成本 | **最低**（按 series 计） | **最高**（按 GB 计） | 中（按 span 计） | 低（按 profile 计） |
| 基数承受力 | **极差**（笛卡尔积爆炸） | 好（全文索引） | 好（每条 trace 独立） | 中 |
| 回答什么 | "有问题吗？多严重？" | "这一次具体发生了什么？" | "慢在哪一跳？谁调了谁？" | "哪行代码/哪块内存？" |
| 天然缺陷 | 看不到个体 | 关联全靠自己拼 | 采样后无法做统计 | 无请求上下文 |
| 保留期 | 13 个月（降采样后） | 7–30 天 | 7–15 天 | 30 天 |

心智模型：`Metrics 告诉你「有事」→（exemplar 跳转）→ Traces 告诉你「在哪一跳」→（trace_id 关联）→ Logs 告诉你「为什么」→ Profiles 告诉你「哪行代码」`。

### 问题类型 → 该看哪种遥测（必背表）

| 问题 | 主 | 辅 | 用错那个会怎样 |
|---|---|---|---|
| 整体错误率上升 | Metrics | Traces | 只翻日志 = 在 2 TB 里大海捞针 |
| p99 变慢但 p50 正常 | **Traces（尾部采样）** | Profiles | Metrics 只说"慢了"，不说慢在哪跳 |
| 所有请求都慢了 20% | **Profiles（差分火焰图）** | Metrics | Traces 显示每一跳都均匀变慢 = 无信息 |
| 单个租户报错，其他人正常 | **Logs（按 tenant 过滤）** | Traces | 给 metric 打 tenant 标签 = 基数爆炸账单 |
| 内存缓慢增长 / OOM | **Profiles（heap）** | Metrics | 日志和 trace 完全看不见 |
| 偶发 5xx（万分之一） | **尾部采样 Traces** | Logs | 头部采样 1% 会把它采没 |
| 一次部署后回归 | Metrics（版本维度） | 差分 Profiles | — |
| 跨服务的因果（谁触发了谁） | **Traces** | — | 日志时间戳对不齐，永远拼不出来 |
| 队列堆积 / 背压 | Metrics（队列深度 + 队首等待） | Traces | 只看利用率会漏掉排队 |
| 死锁 / goroutine 泄漏 | **Profiles（goroutine/mutex）** | — | 其他三支柱完全无能为力 |
| 成本异常（某租户烧钱） | 低基数 Metrics + **离线明细表** | — | 做成高基数 metric = 拿 TSDB 当数据仓库 |

**最后一行是 Staff 与 Mid 的分水岭**：`成本 / 用量 / 计费` 这类需要精确、可审计、可按任意维度回溯的数据**属于数据仓库，不属于可观测性系统**。塞进 TSDB 你会同时得到高账单和不准的数字。

---

## 2. 指标：基数是唯一的成本变量

### 基数就是笛卡尔积

一个活跃时间序列 = `指标名 + 一组标签值的唯一组合`。

```
http_request_duration_seconds{endpoint(200), method(4), status(8), pod(30), region(3)}
  = 200 × 4 × 8 × 30 × 3                          = 576,000 series
  × 14（12-bucket 直方图展开成 12+sum+count）      = 8,064,000 series   ← 一个指标
  × 5,000（加一个 tenant_id 标签）                 = 403 亿 series      ← 破产
```

**每增加一个基数为 N 的标签，成本乘以 N。不是加 N，是乘 N。** 这是唯一要记的成本公式。

按数量级锚点 **$1 / 1,000 活跃 series / 月**（2026 年中量级，各家差异很大且有阶梯折扣，**只用于数量级估算，不要当报价**）：上面的基线是 **$8,000/月**，加了 `tenant_id` 是 **$4,000 万/月**。加 `user_id` / `trace_id` 则是每个请求一条新 series——那不是指标，那是拿 TSDB 当日志用。

**高基数标签的正确归宿：**

| 想按什么维度下钻 | 放哪里 |
|---|---|
| tenant / customer（数千+） | **Trace 属性 + 日志字段**；metric 侧只留 top-N 大客户白名单 |
| user_id / request_id / trace_id | **只放 exemplar 和日志**，绝不做 metric 标签 |
| region / az / version / endpoint 组 | metric 标签（低基数，是分维度下钻的主力） |
| 错误码（上百个） | 归并成 5–10 个类别做标签，原始码进日志 |

**治理手段（按 ROI 排序）：**

1. **Allowlist，不是 denylist。** 在 Collector 里做标签白名单，新标签默认丢弃并计入 `dropped_labels_total`。denylist 永远追不上开发者的创造力。
2. **查询审计驱动删除。** 把 90 天内所有 dashboard / 告警 / 手工查询引用的指标名与实际写入的求差集，直接 drop 差集。典型结果：**30–60% 的 series 从未被任何人查过。**
3. **Exemplar 代替标签。** 在直方图 bucket 上挂几个 `trace_id` 样本，就获得了"从 p99 那根柱子一键跳到真实慢请求"的能力，**而基数不变**。这是过去五年最被低估的特性。
4. **拆指标而不是加标签。** 要按租户看错误率？做 `tenant_slo_breach_total`（只在该租户 SLI 跌破阈值时 +1），基数 = 租户数 × 1，而不是租户数 × 所有维度。

### RED / USE：不要发明自己的指标体系

- **RED**（请求驱动的服务）：**R**ate、**E**rrors、**D**uration。
- **USE**（资源：CPU / 磁盘 / 队列 / 连接池）：**U**tilization、**S**aturation、**E**rrors。

**USE 里最被忽略的是 Saturation。** 利用率 80% 和"利用率 80% + 队列长度 500"是完全不同的两个系统，后者已在雪崩边缘。**几乎所有"CPU 才 70% 为什么这么慢"的答案都在 Saturation 那一栏**（连接池排队、线程池队列、GC 停顿、磁盘 IO 深度）。

### 分位数聚合的正确性陷阱（必考）

**分位数不能取平均，也不能取最大。**

```
服务 A：1000 请求，p99 = 100 ms      服务 B：10 请求，p99 = 5000 ms
❌ avg(p99) = 2550 ms   ← 不是任何真实用户的体验
❌ max(p99) = 5000 ms   ← 系统性高估，让你去优化不存在的问题
✅ 真实整体 p99 ≈ 100 ms（B 只占 1% 流量，其慢请求落在 p99.9 之后）
```

这个错误在生产中无处不在，因为几乎所有 dashboard 工具都让你毫不费力地写出 `avg(p99_latency) by (service)`。**正确做法只有一个：聚合 bucket 计数，再从聚合后的分布算分位数。**

```promql
✅ histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
❌ avg(http_request_duration_p99) by (service)
```

三个衍生要点：

- **"合成端到端 p99"不存在。** A 的 p99 + B 的 p99 ≠ 端到端 p99——慢请求不一定在两跳都慢（也可能正相反）。端到端延迟必须在端到端那一层直接测。
- **经典直方图的精度受 bucket 边界限制。** `histogram_quantile` 在 bucket 内线性插值：500ms 和 1s 之间没有边界时，算出的 "p99 = 743ms" 是虚构的。原生直方图（native histogram，Prometheus 3.x 仍在演进）/ DDSketch / t-digest 用相对误差保证（典型 1–2%）解决它，代价是后端支持面窄。
- **SLI 别用分位数算。** 直接打两个 counter（见 [01-slo-and-error-budget.md](01-slo-and-error-budget.md) §1），既准确又便宜，还天然可聚合。

---

## 3. 日志：最容易失控的一支

### 成本模型

```
一行结构化 JSON 日志 ≈ 400–800 B（带 trace_id、tenant、service、level 等标准字段后）

10,000 rps × 每请求 5 行 × 600 B = 30 MB/s = 2.6 TB/天 = 78 TB/月
```

托管日志平台的摄入单价量级 **$0.5–3 / GB**（2026 年中量级，随时变动；存储与查询另计）。78 TB/月 × $1/GB ≈ **$78,000/月**。

**这就是为什么"每个请求打 5 行 info 日志"是一个需要架构评审的决定。**

### 该记什么 / 不该记什么

| ✅ 必须记 | ❌ 不要记 |
|---|---|
| **决策点**："选了降级路径，因为 circuit_breaker=open" | 热路径上的 `entering function X` |
| **外部调用**：`(target, status, latency_ms, attempt, idempotency_key)` | 完整的 request/response body |
| **状态转移**：`order 123: pending → paid` | 能从 metric 得到的计数（"处理了 1 条消息"） |
| **异常的完整上下文**：stack + 触发参数 + trace_id | 每次循环迭代 |
| **人工/自动干预**：谁在什么时候改了什么 flag | 心跳 / 探针 |
| **不可逆动作的审计**：付款、删除、外发 | 你打算"以后可能有用"的一切 |

**判据**：写这行日志之前问"如果它触发了，我会因为它做什么？"答不上来就不写。

### 采样与 PII

```
ERROR / WARN      → 100% 保留，永不采样
带 trace 的 INFO  → 与 trace 采样决策一致（同一条 trace 要么全留要么全丢）
高频热路径 INFO   → 1–5%（按 trace_id 哈希，保证一致性）
DEBUG             → 生产默认关闭，按 tenant / 请求头动态开启，带 15 分钟自动过期
```

**"按 tenant 动态开 DEBUG + 自动过期"是 ROI 最高的可观测性功能之一**——它堵死了"为排查一个客户而全局开 DEBUG"这个最大的成本事故源头。

PII 四条：**字段级 allowlist 而非 denylist**（未声明的字段默认 redact）；**在 Collector 侧再脱敏一遍**（第三方库会绕过你的 logger）；保留期按类别分（审计 1–7 年 / 应用 7–30 天 / 带内容的调试日志 ≤ 24 小时）；LLM 系统里**最大的 PII 面是 prompt 与 response 全文**（见 §10）。

---

## 4. 追踪：采样是唯一真正的设计决策

### 上下文传播

标准是 [W3C Trace Context](https://www.w3.org/TR/trace-context/)：`traceparent`（trace-id + span-id + flags）与 `tracestate`（厂商特定）。另有 `baggage`（[W3C Baggage](https://www.w3.org/TR/baggage/)）携带跨服务的业务标签（`tenant.id`、`experiment.id`）。⚠ **baggage 会被塞进每一跳的 HTTP 头**：放 5 个字段是工具，放 20 个字段就是每请求多几 KB 网络开销加一个信息泄露面（它会一路传给你的第三方依赖）。

**跨异步边界的传播才是难点：**

```
HTTP → HTTP           传 header，SDK 自动搞定
HTTP → Kafka          把 traceparent 写进 message header（不是 body！）
Kafka → Consumer      从 header 恢复，但用 SpanLink 而非 parent-child
定时任务 / 批处理       每次运行新建 root span，用 link 指回触发它的那条 trace
Agent 的工具调用       工具执行的 span 必须挂在这一步推理的 span 下
重试                  同一个 trace，每次尝试一个独立 span（带 attempt 属性）
```

**为什么消费侧要用 `link` 而不是 `child`：** 一条消息可能被 3 个 consumer group 消费、可能在 6 小时后才被处理、一个批次可能一次处理 500 条消息（500 个 parent）。强行做成 parent-child 会产生一条永远不结束、跨越 6 小时、有 500 个分支的 trace——查询和存储都会崩。**Link 表达"由此触发"，parent 表达"是其一部分"，这两件事不一样。**

### 采样策略对比表（必背）

| 策略 | 决策时机 | 能抓到罕见错误 | 成本可预测 | 实现复杂度 | 适用 |
|---|---|---|---|---|---|
| **头部固定比例** | 请求入口 | ❌ 1% 采样漏掉 99% 的错误 | ✅ 完全可预测 | 极低 | 流量大、错误率高、只看统计分布 |
| **头部按属性** | 请求入口 | 部分（可对特定 endpoint/tenant 提高比例） | ✅ | 低 | **绝大多数团队的正确起点** |
| **尾部采样** | trace 结束后 | ✅ **可以规则化保留所有错误/慢请求** | ⚠ 取决于规则 | **高** | 错误稀疏、必须抓全 |
| **基于错误/延迟的头部提升** | 入口猜 + 下游可提升 | 部分 | 中 | 中 | 折中方案 |
| **自适应采样** | 动态调比例保持恒定 span 速率 | 部分 | ✅ **最可预测** | 中 | 流量剧烈波动 |
| **调试触发** | 请求头 `x-debug-trace: 1` | 定向 | ✅ | 极低 | 排障，**必须限流+鉴权** |

**尾部采样的真实代价（这是面试考点）：**

```
决策前必须把整条 trace 的所有 span 缓存在内存里：
内存 ≈ span 速率 × 平均 span 大小 × 决策窗口 × 安全系数
     = 50,000 spans/s × 600 B × 30 s × 2.5 = 2.25 GB   ← 单个 Collector 实例
```

更麻烦的是：**同一条 trace 的所有 span 必须落到同一个 Collector 实例上**，否则实例 A 看不到实例 B 手里的 error span，做不出正确决策。这需要一层按 `trace_id` 哈希的负载均衡（OTel Collector 的 `loadbalancing` exporter）。**"尾部采样"实际上是"一个有状态、需要一致性哈希的分布式系统"**——很多团队在预算会上答应了它，在实施时才发现这一点。

**推荐的落地顺序（前四条覆盖 90% 场景，不够时才上尾部采样）**：① 头部按属性采样，默认 1%；② 应用侧发现 error 时直接把 trace flag 置为 sampled，错误 100% 留存；③ Exemplar 打通 metric → trace；④ 带鉴权与限流的 `x-debug-trace` 头做定向排查。

---

## 5. 持续 Profiling：第四支柱，也是最便宜的一支

**它解决的是其他三支柱结构性看不见的问题：** 所有请求均匀变慢 20%（trace 显示每一跳都慢一点 = 零信息）；时间花在没有 instrument 的库里（那里没有 span）；内存缓慢增长到 OOM（只有 heap profile 能归因到分配点）；锁竞争 / goroutine 泄漏（完全不可见）；发布后 CPU 涨 30%（metric 只说涨了，不说哪个函数）。

**开销量级**：CPU profile 采样 100 Hz 约 **1–2% CPU**；eBPF 方案（无需改代码、无需重启）通常 **< 1%**。开源实现：[Grafana Pyroscope](https://grafana.com/oss/pyroscope/)、[Parca](https://www.parca.dev/)。

**杀手级用法是差分火焰图**：`profile(部署后) − profile(部署前)` 直接高亮新增的栈帧。一次"部署后 p99 涨了 200ms"的排查，差分火焰图通常 **5 分钟**定位到函数，trace + 日志要**半天**。**把它接进金丝雀流程**——金丝雀实例与基线实例自动做 profile diff，作为发布门禁的一项。

---

## 6. OpenTelemetry：现状、迁移与陷阱

### 为什么值得迁

**Collector 是关键，不是 SDK。** 真正的收益是拿到一个**厂商无关的解耦点**：

```
应用 ──OTLP──▶ OTel Collector ──▶ 后端 A（metrics）/ 后端 B（traces）/ S3（冷归档）
                    └─ 采样 · PII 脱敏 · 标签 allowlist · 属性规范化 · 按 tenant 路由
```

换后端从"改 2,000 个服务"变成"改一个 Collector 配置"。**这一条本身就值得迁。**

### 成熟度：traces / metrics 可用，logs 演进中，GenAI **不能当稳定契约**

**GenAI 语义约定的真实状态（2026-07-30）**——这是必须写进架构决策的事实：

- **全部 GenAI 专有的 span / metric / event / attribute 至今仍是 `Development`，没有任何一项 GA。** 只有跨域共享属性（`error.type`、`server.address`）是 Stable。
- **仓库已在 2026-06 拆分**到 [`open-telemetry/semantic-conventions-genai`](https://github.com/open-telemetry/semantic-conventions-genai)。主 semconv 仓库 v1.42.0（2026-06-12）**弃用全部 `gen_ai.*`**，v1.43.0 **完全移除**。新仓库**尚无 release / 无 tag / 无 schema URL**（README 那一节写着 TODO）——你**拿不到一个可 pin 的版本化 schema**。
- **破坏性改名历史**（这才是你会被咬的地方）：

| 版本 / 日期 | 变更 |
|---|---|
| v1.27.0 / 2024-08 | `prompt_tokens` → `input_tokens` |
| v1.37.0 / 2025-08 | `gen_ai.system` → `gen_ai.provider.name` |
| v1.38.0 / 2025-10 | 新增 `gen_ai.evaluation.result` |
| v1.40.0 / 2026-02 | 新增 retrieval span + cache token 属性 |
| v1.41.0 / 2026-04 | agent span 拆分 + reasoning token |
| v1.42.0 / 2026-06-12 | 弃用全部 `gen_ai.*`（迁往独立仓库） |

**两年内至少 5 次破坏性改名。跨版本查询不做 `coalesce` 会静默丢数据——不是报错，是曲线上出现一段莫名其妙的空白。**

### 四条必须写进设计文档的对策

**① 在采集与查询之间加一层内部规范化 schema**，把上游的动荡挡在自己的命名空间之外：

```yaml
# otel-collector.yaml
processors:
  transform:
    metric_statements:
      - context: datapoint
        statements:      # 新旧属性 coalesce 到自有命名空间
          - set(attributes["ai.provider"], attributes["gen_ai.provider.name"])
              where attributes["gen_ai.provider.name"] != nil
          - set(attributes["ai.provider"], attributes["gen_ai.system"])
              where attributes["ai.provider"] == nil
          - set(attributes["ai.schema_version"], "internal-v3")   # 查询侧据此分支
  attributes/tenant:
    actions:             # OTel 里没有 tenant 概念，必须自己加
      - {key: app.tenant.id, action: insert, from_context: auth.tenant_id}
```

**② 官方命名空间里没有 cost，也没有 tenant/user。** 新仓库的属性 registry 原文直说 `No dedicated cost attributes exist`、`User/Tenant Attributes — Notably absent`。很多 2026 年的文章直接用 `gen_ai.usage.cost_usd`——**那是厂商扩展，不是标准，跨平台不保证被识别**。

> **面试金句**
> "计价必须放在后端：采集侧只上报 token 数（`input` / `output` / `cache_read` / `cache_creation` / `reasoning` 五个分开），后端拿 `(model_id, 时间点)` 查单价表算钱。把价格烧进 SDK 的后果是——供应商 9 月 1 日调价，你所有历史数据的口径就断了，而且没法回算。**遥测记录事实，不记录估值。**"

**③ 多租户拆分用的自定义属性必须第一天就打全**（`app.tenant.id`、`app.feature`、`app.request.id`）。**这类属性事后不可回溯补齐**，历史数据永远缺失——这是最常见也最贵的可观测性架构失误。

**④ 内容属性是 Opt-In，且是最大的 PII 面。** `gen_ai.input.messages` / `output.messages` / `system_instructions` / `tool.definitions` 默认不采集。打开前必须有：按环境/租户的开关 + 采样 + 脱敏管道 + 独立的短保留期。"评测需要全文"应被限制在采样子集上，而不是全量。

---

## 7. 成本治理：可观测性经常是你的第二大账单

### 削减手段（按 ROI 从高到低）

| 手段 | 典型削减幅度 | 代价 |
|---|---|---|
| **删除 90 天无人查询的 series** | **30–60%** | 几乎为零（需要查询审计能力） |
| 标签 allowlist + 高基数标签下沉到 trace/日志 | 20–50% | 失去 metric 侧的细粒度下钻 |
| 日志分层保留（热 7d / 温 30d 降采样 / 冷 13m 归档到对象存储） | 40–70% 存储费 | 查冷数据慢（分钟级） |
| INFO 日志采样到 1–5% | 30–50% 摄入费 | 需要保证同 trace 一致性 |
| 减少直方图 bucket 数 | 20–30%（指标侧） | SLO 阈值必须仍是边界 |
| 换后端 / 自建 | 50–80% | 2–4 个 FTE 的长期运维投入 |

**第一条是关键，也是最少被做的。** 实现方式：解析所有 dashboard JSON、告警规则和后端查询日志，提取被引用的指标名，与实际写入的指标名求差集。绝大多数团队第一次跑这个脚本时会震惊。

**分层保留的具体切法：**

```
指标：  原始精度 15 天 → 5 分钟降采样 90 天 → 1 小时降采样 13 个月（只留 SLI 与容量指标）
日志：  热索引 7 天 → 对象存储 90 天（可按 trace_id 检索）；审计日志单独 1–7 年
Trace： 采样后 15 天；错误 trace 30 天
Profile：30 天；每次发布的基线 profile 永久保留（用于差分）
```

**⚠ 两条不能碰的红线**：① 不要为省钱关掉 SLI 指标——它低基数低成本（一条旅程 4 个 counter），却是整套错误预算体系的地基。② 不要把成本压力转嫁成"出事时没有数据"——正确做法是**故障期间自动提高采样率**（Collector 监听告警状态，错误率上升时把 trace 采样从 1% 拉到 100%），平时便宜、出事全量。

---

## 8. 告警的设计

### 三条硬规则

**① 基于症状，不基于原因。**

```
❌ "DB 主库 CPU > 90%"          → CPU 高但用户无感，你半夜起来干什么？
❌ "Pod 重启了 3 次"             → 那是 Kubernetes 在正常工作
✅ "工作区加载 SLO 燃尽率 14.4×"  → 用户正在受影响，且 50 小时会用完月度预算
```

原因型指标应该出现在 **dashboard 和 runbook 里**，作为排查的下钻维度，**而不是作为 page 的触发条件**。

**② 每条 page 级告警必须能回答三个问题**，答不上来就降级成 ticket 或直接删除：**谁受影响**（哪条旅程、多少租户、什么比例）、**我现在能做什么**（runbook 链接存在且有具体的第一步）、**不处理会怎样**（多久用完错误预算 / 多久触发 SLA 赔付）。

**③ 告警疲劳是可以量化的。** 经验阈值：**一个班次（12 小时）超过 2 条 page 级告警，人就开始批量确认而不阅读。** 这不是纪律问题，是注意力的物理上限。一个可类比的公开数据点：在高频权限确认场景中观测到约 **93% 的自动批准率**——**当确认变成条件反射，它就不再是控制**。告警也一样。

```
可执行率 = 导致实际处置动作的告警 / 全部 page 告警      目标 > 70%
噪音率   = 5 分钟内自动恢复的告警 / 全部 page 告警       目标 < 10%
```

每季度审计一次，把可执行率低于 30% 的告警**直接删除**——不是调阈值，调阈值是拖延。

### 从阈值告警迁移到 SLO 告警

| | 阈值告警 | SLO 燃尽率告警 |
|---|---|---|
| 触发条件 | "错误率 > 1%" | "以当前速度 50 小时耗尽月度预算" |
| 严重度分级 | 靠拍脑袋 | **由消耗的预算比例天然决定**（2% / 5% / 10%） |
| 低流量服务 | 极度嘈杂 | 需要加最小事件数门槛，但框架统一 |
| 与业务对话 | 无法翻译 | 直接可翻译成"客户会不会投诉/退款" |

参数表见 [01-slo-and-error-budget.md](01-slo-and-error-budget.md) §6。

---

## 9. 调试分布式问题的方法论

**固定路径，每一步都在缩小搜索空间。**

```
① SLI 异常          "工作区加载 SLI 从 99.95% 掉到 98.2%"
      │  先问三个问题（80% 的事故在这里就结案）：
      │    有新部署吗（按 version 切）· 有配置/flag 变更吗 · 流量形状变了吗
      ▼
② 分维度下钻        按 region / az / version / endpoint / tenant 逐层切
      │             ├─ 单 AZ 坏    → 基础设施，直接切流量
      │             ├─ 单版本坏    → 回滚，先止血再定位
      │             └─ 均匀全坏    → 共享依赖或容量问题
      ▼
③ Exemplar → Trace  从 p99 那根柱子跳进一条真实慢请求，看 span 瀑布
      │             ├─ 某跳 self time 高      → 去 ⑤
      │             ├─ 某跳等很久才开始       → 排队/背压，看 USE 的 Saturation
      │             └─ 每跳都均匀慢          → 跳过 ④，直接去 ⑤
      ▼
④ Logs             用 trace_id 精确过滤（不是时间范围！），拿参数/异常栈/重试次数
      ▼
⑤ Profile（差分）   慢进程 vs 基线，或部署后 vs 部署前，定位到函数级
```

**两个高频误区：** ①**跳过第②步直接看 trace**——你会看到一条慢 trace 然后花两小时优化它，而它可能只是正常长尾。先确定"是不是所有人都坏了"，再看个体。②**在第④步用时间范围搜日志**——高流量下同一秒有几万条日志。**没有 trace_id 关联的日志系统在分布式排障里价值接近于零**，所以"日志必须带 trace_id"是硬要求，不是最佳实践。

---

## 10. AI 系统的可观测：基础设施视角

评测、judge、轨迹分析见 [`04-ai-agent-systems/06-evaluation-and-observability.md`](../04-ai-agent-systems/06-evaluation-and-observability.md)。这里只讲**基础设施层你必须有的东西**。

### 必备指标表

| 类别 | 指标 | 为什么它不能省 |
|---|---|---|
| **GPU** | SM 利用率、显存占用、SM occupancy、功耗、温度、ECC 错误（DCGM 采集） | **SM 利用率高 ≠ 在做有用功**：显存带宽打满或 all-to-all 阻塞时利用率同样很高 |
| **KV cache** | KV 块使用率、被抢占（preempted）请求数、重算次数 | KV 满 → 请求被抢占 → 重新 prefill → TTFT 尖刺。**这是延迟毛刺最常见的根因，且从外部完全看不出来** |
| **队列** | running seqs / waiting seqs / 队首等待时长 | waiting 持续 > 0 = 已过拐点，加机器或限流，别再调参 |
| **吞吐** | prefill tok/s 与 decode tok/s **分开** | 两者的瓶颈完全不同（算力 vs 显存带宽），合并成一个数字后不可用于容量规划 |
| **前缀缓存** | 命中率，**必须按租户/场景切分** | 见下方 ⚠ |
| **延迟** | TTFT / TPOT 直方图，**按输入长度分桶** | 不分桶时一个长上下文请求就把曲线拉飞 |
| **goodput** | 每秒同时满足 TTFT 与 TPOT SLO 的请求数 | 唯一能做容量规划的指标 |
| **Token** | `input` / `output` / `cache_read` / `cache_creation` / `reasoning` **五个分开** | 单价差 10–20 倍，合并计数就无法归因成本 |
| **Agent** | 每会话步数、工具调用次数、重复调用次数、上下文占用比例 | 循环和步数爆炸是 Agent 系统独有的高频失效 |
| **成本** | 每租户/每功能/每会话的 token 成本（**在后端由单价表算**） | 尾部会话可以烧掉中位数的 20–50 倍 |

### ⚠ 前缀缓存命中率是**每租户**指标，不是全局指标

这是 LLM 基础设施可观测性里最重要的一条实践：

> 同一套 vLLM 部署，一个租户 TTFT 从 480ms 降到 110ms（命中率 70–90%），**另一个租户命中率 0.3%，完全没有改善**——因为它动态拼 prompt，每次前缀都不同。

如果你只看全局平均命中率（比如 65%），你会认为"缓存工作良好"，而那个 0.3% 的租户正在付 10 倍的钱、等 4 倍的时间，并且**在你的 dashboard 上完全隐形**。

同理，agentic 多轮场景里工具输出插在上下文中间会打碎共享前缀，**stock prefix cache 命中率可跌破 20%**。所以：

```
必须有的维度：prefix_cache_hit_ratio by (tenant, route, prompt_template_version)
必须有的告警：某租户命中率 < 20% 且请求量 > 阈值 → ticket（不是 page，但必须有人看）
```

### Trace 的形状变了

传统请求 trace 是"扇出 3–5 跳、总时长 200 ms"。Agent 会话是"**串行 40 步、总时长 8 分钟、嵌套 3 层子代理**"。三个后果：

- **一条 trace 可能有数千个 span、几 MB 大小。** 默认 span 数上限（很多 SDK 是 1,000）会**静默截断**——你会得到一条"看起来正常结束了"的假 trace。必须显式配置并监控 `dropped_spans`。
- **尾部采样的决策窗口要按会话时长设**（分钟级而非秒级），这会让 §4 的内存算例乘以 10–20。多数团队的正确选择是：**Agent 轨迹不走通用 trace 管道，走专门的轨迹存储**（Langfuse / LangSmith / Braintrust / 自建），通用管道只留一个汇总 span。
- 子代理的探索上下文（数万 token）**只回传 1,000–2,000 token 的摘要**。必须保留原始探索轨迹的引用路径，否则事后无法复审父代理为什么做了那个决定。

### 内容采集的边界

`prompt` / `response` 全文是**最有用**也是**最危险**的遥测：默认不采集（OTel 的内容属性本就是 Opt-In）；1–5% 采样子集经 PII 脱敏后采集、保留期 ≤ 7 天；特定租户/环境可全量，但需合同条款 + 显式配置 + 到期自动关闭；**永不**把 prompt 全文放进 metric 标签或 span name。

⚠ 额外一条：即使启用了零数据保留（ZDR），**推理侧的 prompt cache tensor 仍可能被加密保留最长 24 小时**（某主流厂商 2026 年的公开条款）。在合规文档里写"我们不保留任何 prompt"之前，先核对这一条。

---

## 11. 什么时候不要这么做 / 反模式

| 反模式 | 后果 | 正确做法 |
|---|---|---|
| 给 metric 加 `user_id` / `request_id` / `trace_id` | series 无限增长，后端 OOM，账单失控 | 高基数进 trace/日志，metric 侧只留 exemplar |
| `avg(p99)` / `max(p99)` 跨服务聚合 | 数字既不真实也不可用，会引导你优化错的东西 | 聚合 bucket 再算分位数 |
| 把成本/计费数据塞进 TSDB | 高账单 + 不准确 + 无法审计回溯 | 明细进数据仓库，TSDB 只留低基数汇总 |
| "先全量采集，以后再优化" | 三个月后账单 10 倍，且没人敢删 | 第一天就上 allowlist + 查询审计 |
| 尾部采样没做 trace_id 亲和路由 | 决策基于不完整的 trace，静默漏掉错误 | `loadbalancing` exporter 按 trace_id 哈希 |
| 日志里没有 trace_id | 分布式排障退化成时间范围大海捞针 | trace_id 是结构化日志的必填字段 |
| 告警基于原因（CPU/重启/磁盘） | 告警疲劳，真问题被淹没 | 症状告警（SLO 燃尽率）+ 原因指标进 runbook |
| 直接采用 `gen_ai.*` 属性名做长期查询 | 两年 5 次破坏性改名，曲线静默断档 | 内部规范化 schema + 查询侧 coalesce |
| LLM 成本用 `gen_ai.usage.cost_usd` 上报 | 非官方属性；调价后历史数据口径断裂 | 只报 token 数，后端用单价表算钱 |
| 只看全局前缀缓存命中率 | 掩盖"某租户 0.3% 命中率"的严重浪费 | 按 tenant / route / prompt 版本切分 |
| 事后再补 `tenant_id` 属性 | **历史数据永远缺失，不可回溯** | 第一天就打全自定义属性 |
| 为省钱关掉 SLI 指标 | 错误预算体系失效 | 省别的；出事时自动提高采样率 |
| 追求"完整的可观测性覆盖" | 花光预算，仍然答不上"用户现在爽不爽" | 先把 3–7 条旅程的 SLI 做对，其余按需补 |

**最后一条展开说：** 可观测性投资的正确顺序是 ① 3–7 条用户旅程的 SLI（低基数 counter，几乎免费）→ ② 症状告警（SLO 燃尽率）+ runbook → ③ Trace + exemplar → ④ 带 trace_id 的结构化日志 → ⑤ 持续 profiling。**这五项覆盖 90% 的排障需求，成本占比不到 30%。** 尾部采样、全量日志、dashboard 农场、Agent 轨迹平台都排在它们后面。

**倒过来做的团队（先买一个大而全的平台，再想怎么用）会花 3 倍的钱，拿到一半的能力。**

---

## 面试官会追问

1. 给 metric 加一个 `tenant_id` 标签（5,000 租户），成本怎么变？那想按租户看错误率该怎么办？
2. 10 个实例各自 p99 = 100ms，整体 p99 是多少？为什么不能求平均？SLO 阈值 500ms 但 bucket 是 `[0.1, 0.25, 1, 2.5]`，SLI 准吗？
3. 头部采样和尾部采样怎么选？尾部采样的实现成本具体在哪？内存怎么估？
4. Trace 跨 Kafka 怎么传播？为什么消费侧要用 link 而不是 parent-child？
5. 所有请求都均匀慢了 20%，trace 看不出问题。下一步查什么？
6. 可观测性账单一年涨了 4 倍，给我三个削减手段和各自的代价。
7. 为什么不能对 "DB CPU > 90%" 发 page 级告警？那这个指标放哪？
8. OTel 的 GenAI 语义约定现在能直接用吗？架构上怎么防它继续改名？LLM token 成本你会怎么上报？
9. 全局前缀缓存命中率 65%，看起来不错。你还会看什么？一个 Agent 会话产生 3,000 个 span，你的 trace 管道会发生什么？

---

**下一篇** → [03-resilience-patterns.md](03-resilience-patterns.md)
