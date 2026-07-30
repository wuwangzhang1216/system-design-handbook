# 05 · 数据平台：Lakehouse、契约与指标层

> 数据平台的技术难点在 2026 年基本被解决了（表格式收敛、目录标准化、引擎可换）。
> 还没解决的是社会问题：**谁对这张表的正确性负责**。数据契约（data contract）就是把这个社会问题变成一个会挡住合并按钮的机器人。

---

## 1. OLTP → 分析：三条路径

| 路径 | 新鲜度 | 对 OLTP 的压力 | 复杂度 | 撞墙条件与信号 |
|---|---|---|---|---|
| **直读只读副本（read replica）** | 秒级 | 中（大扫描吃 IO buffer，副本 lag 上升） | 最低 | 数据 > ~500 GB 或分析查询 > 几十 QPS；**信号：副本 lag 从毫秒涨到秒，业务开始抱怨"读不到刚写的"** |
| **批量导出（batch export）**（每日/每小时快照） | 1–24 h | 低（低峰跑） | 低 | 需要小时内新鲜度；或单表 > 1 TB 时全量 dump 跑不完窗口；**信号：导出任务开始重叠** |
| **CDC → 流 → 湖** | 秒到分钟 | 最低（读 WAL/binlog，不走查询路径） | **高** | 从第一天就复杂：schema 变更、迟到事件（late-arriving events）、断点重放（replay from checkpoint）、快照+增量的接缝全要处理 |

**演进顺序是固定的**：只读副本 → 批量导出 → CDC。**跳级是最常见的浪费**。一个 ARR 不到 1000 万的公司直接上 Debezium + Flink + Iceberg，通常会在 6 个月后发现 80% 的查询其实是"昨天的数据就够"。

CDC 的实现细节（Outbox、快照与增量的一致接缝、幂等消费）见 [01-building-blocks/03-messaging-and-streams.md](../01-building-blocks/03-messaging-and-streams.md)。这里只强调一条：**CDC 给你的是"行变更流（row-change stream）"，不是"业务事件流（business event stream）"**。`UPDATE orders SET status='shipped'` 这条 CDC 记录不告诉你"为什么发货"。把 CDC 当业务事件用，会让下游被迫从列的变化里反推语义，而这个反推逻辑会在源系统重构时静默失效。

---

## 2. Lakehouse 表格式：核心机制

三种格式（[Apache Iceberg](https://iceberg.apache.org/) / [Delta Lake](https://delta.io/) / [Apache Hudi](https://hudi.apache.org/)）解决的是同一件事：**在只有"整对象覆盖"语义的对象存储上，做出 ACID 表**。

### Iceberg 的元数据分层（三种格式思路相通）

```
Catalog（唯一的可变指针）
   │  原子操作：CAS 把 table → v7.metadata.json 换成 v8.metadata.json
   ▼
metadata.json          表 schema、分区规格、快照列表、属性
   ▼
manifest list          一个快照 = 一份 manifest 清单（含分区统计，用于剪枝）
   ▼
manifest file          一批数据文件的清单 + 每列的 min/max/null_count
   ▼
data files (Parquet)   实际数据 + delete files / deletion vectors
```

**关键推论：**

1. **快照隔离（snapshot isolation）是"免费"的**：读取者拿到一个 metadata.json 就锁定了一致的文件列表，写入者往后追加新文件不影响它。代价是**旧快照的文件不能立刻删**（这在合规删除上会咬你，见 §10）。
2. **写入冲突靠乐观并发控制（optimistic concurrency control）**：两个写入者同时想把指针从 v7 换到 v8，只有一个 CAS 成功，另一个重读元数据、重放自己的变更、再试（Iceberg `commit.retry.num-retries` 默认 4）。**这意味着高频小事务会退化成互相打架** —— 每分钟 commit 一次是上限量级，每秒 commit 一次不行。
3. **查询规划（query planning）的成本在元数据层**，不在数据层。manifest 里的列统计让引擎能在读任何 Parquet 之前就剪掉（prune）95%+ 的文件。**manifest 太多（小文件的副作用）会让规划本身变成几十秒的瓶颈。**
4. **列由 ID 标识，不由名字标识**。所以在 Iceberg 里重命名列是纯元数据操作，不需要重写数据 —— 这是它相对 Hive 表最实用的一个差异。⚠️ 但**表格式安全 ≠ 契约安全**：重命名对下游的 SQL 仍然是破坏性变更（见 §4）。

### 三家的真实差异

| | Iceberg | Delta Lake | Hudi |
|---|---|---|---|
| 元数据 | 分层清单（manifest list / manifest） | 顺序事务日志 `_delta_log/*.json` + 每 10 次 commit 一个 checkpoint | LSM 结构时间线（1.0 起） |
| 强项 | **引擎中立（engine-neutral）**，规范与实现分离，生态最广 | Databricks 一等公民，Liquid Clustering 等运行时优化最成熟 | **索引化 upsert**（record-level index），增量查询原生 |
| 更新语义 | CoW + MoR（v2 位置/等值删除文件，v3 起用二进制 deletion vector） | CoW + MoR（deletion vector） | CoW / MoR 可按表选择 |
| 最适合 | 多引擎、多云、长期资产 | 已在 Databricks 上 | 高频 upsert 的准实时表（CDC 落地） |

**2025–2026 的生态收敛是这一节最重要的部分：**

- **Iceberg REST Catalog** 把"目录"从库变成了 HTTP 协议（OpenAPI 定义）。后果：引擎不再需要为每种目录写适配器，目录实现可以自由替换（[Apache Polaris](https://polaris.apache.org/)、Unity Catalog OSS、Lakekeeper、Nessie、Glue 的 Iceberg REST 端点等）。**选型建议：只要你的引擎支持 REST catalog，就用它，不要再用 Hadoop/Hive catalog。**
- **S3 条件写入（conditional write）**（2024 年起支持 `If-None-Match`，随后支持 `If-Match`）给了对象存储原生的 compare-and-swap。这直接消掉了"在 S3 上做 Iceberg 必须外挂一个锁服务"的历史包袱，也让轻量目录实现成为可能。
- **Iceberg 表规范 v3**（2025 年定稿，引擎侧支持在 2025–2026 陆续到位）加入了二进制 deletion vector、行级血缘（row-level lineage，`_row_id` / `_last_updated_sequence_number`）、`variant` 半结构化类型、地理类型、纳秒时间戳与列默认值。**行级血缘对增量物化和合规删除追溯是实质性升级。**
- **Delta 通过 UniForm 同时输出 Iceberg 元数据**，Databricks 收购 Tabular 之后两个阵营在做互操作（interoperability）而不是互斥；[Apache XTable](https://xtable.apache.org/)（孵化中）做三格式间的元数据翻译。
- **新方向**：DuckLake（2025）把元数据放回 SQL 数据库而不是对象存储上的文件 —— 因为"用文件模拟事务日志"本来就是在重新发明数据库。这条路线在中小规模上简单得多。

> **面试金句**：
> "选表格式在 2026 年已经不是技术决策了，三家的核心机制是同构的。真正的决策是**目录**：目录是唯一的可变状态，是所有引擎的会合点，也是你唯一换不掉的东西。我会先选 REST catalog 协议，再选实现 —— 这样表格式和引擎都还能换。"

### 小文件（small files problem）与 compaction：唯一真正要调的旋钮

```
流式写入，每分钟 commit，20 个分区
  → 1,440 次 commit/天 × 20 文件 = 28,800 个文件/天
  → 每个文件 2 MB（远低于目标），查询要开 28,800 次 GET
  → 元数据侧：1,440 个快照 + 上万个 manifest → 规划耗时从 200ms 涨到 30s+
```

**目标文件大小 128 MB – 1 GB**（Iceberg `write.target-file-size-bytes` 默认 512 MB）。必须常态化跑四类维护作业：

```sql
-- 1. 数据文件 bin-pack 合并 + 按聚簇键排序（最重要）
CALL catalog.system.rewrite_data_files(
  table => 'silver.orders', strategy => 'sort', sort_order => 'tenant_id, occurred_at',
  options => map('target-file-size-bytes','536870912', 'min-input-files','5',
                 'partial-progress.enabled','true'));   -- 分批提交，降低提交冲突

-- 2. 清单重写（元数据侧的 compaction；规划慢时先做这个）
CALL catalog.system.rewrite_manifests('silver.orders');

-- 3. 快照过期（真正释放存储 + 合规删除的前提）
CALL catalog.system.expire_snapshots(table => 'silver.orders',
  older_than => TIMESTAMP '2026-06-30 00:00:00', retain_last => 5);

-- 4. 孤儿文件清理（作业失败留下的、不被任何快照引用的文件）
CALL catalog.system.remove_orphan_files(table => 'silver.orders',
  older_than => TIMESTAMP '2026-07-27 00:00:00');
```

⚠️ **`remove_orphan_files` 的时间阈值必须大于你最长的运行中作业时长**，否则会删掉正在写入的文件 —— 这是 lakehouse 最经典的自伤方式。⚠️ **compaction 与写入会抢同一个 CAS**（提交冲突 commit contention）：开 `partial-progress.enabled` 分批提交，并把 compaction 排在写入低峰。

---

## 3. 分层（layering）：Medallion 与批流一体的现实

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 源   OLTP(Postgres/MySQL)    SaaS API      事件流(Kafka)    文档/对象     │
└────┬───────────────┬──────────────┬───────────────┬──────────────────────┘
     │ CDC           │ 增量拉取      │ 直接落        │ 对象存储事件通知
     ▼               ▼              ▼               ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ BRONZE 原始层   append-only，与源同构，不清洗、不改类型                   │
│   分区 = ingest_date   保留 30–90 天   唯一职责：可重放                   │
│   ⚠ 不对外开放查询。它不是数据集市，它是磁带。                            │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ 去重 / 类型规约 / 迟到窗口 / 【契约校验】
                                │ 校验不过的行 → quarantine 表（不丢）
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ SILVER 明细层   建模后的实体与事件，一行一事实，SCD2，有主键有约束        │
│   Iceberg 表   分区 = event_date   聚簇 = tenant_id   有数据契约           │
│   ★ 数据契约的边界画在这里：Bronze 无契约，Silver 起有契约                │
└───────────────────────────────┬──────────────────────────────────────────┘
                    ┌───────────┴───────────────┐
                    ▼                           ▼
┌───────────────────────────────┐  ┌────────────────────────────────────┐
│ GOLD 消费层                    │  │ 语义层 / 指标层                    │
│ 宽表、汇总、特征、导出快照      │  │ 指标定义即代码，一处定义多处消费    │
└──────┬────────────────────────┘  └───────────────┬────────────────────┘
       │                                           │
       ▼                                           ▼
   BI/报表    反向 ETL→SaaS    RAG 索引    ML 特征    Agent 的 SQL 查询
```

**"批流一体（unified batch and streaming）"在 2026 年的真实含义不是"用一套代码跑两种模式"**，而是：**同一套表**（Iceberg 表同时支持流写与批读）+ **同一套契约与质量规则**，但**仍然是两套执行** —— Flink/Spark Streaming 负责秒-分钟级的 Silver，Spark/Trino 批作业负责小时-天级的 Gold。

**增量物化（incremental materialization）**是把成本降一个数量级的关键：Gold 层不要每天全量重算，用快照区间做增量。

```sql
-- Iceberg 增量读：只读 (snapshot_a, snapshot_b] 之间新增的行（或用 incremental scan API）
SELECT * FROM silver.orders FOR SYSTEM_VERSION AS OF 8172635
-- dbt 侧
{{ config(materialized='incremental', unique_key='order_id', incremental_strategy='merge') }}
WHERE occurred_at > (SELECT max(occurred_at) FROM {{ this }})
```

⚠️ **增量物化的第一个坑是迟到数据（late-arriving data）**：`WHERE occurred_at > max(occurred_at)` 会永久漏掉所有迟到事件。正确做法是用**处理时间（processing time）**（`_ingested_at`）做增量水位（watermark），用**事件时间（event time）**做业务分区，并定期跑一个覆盖最近 N 天的全量回填（backfill）（典型 N = 3–7 天，取决于你的迟到分布 p99.9）。

---

## 4. 数据契约：把口头约定变成 CI 门禁（CI gate）

**数据契约不是文档，是一个可执行的、版本化的、有 owner 的接口定义。** 参考规范：[Open Data Contract Standard (ODCS)](https://bitol-io.github.io/open-data-contract-standard/)（Bitol / LF AI & Data）与 [datacontract.com](https://datacontract.com/) 的规范。

```yaml
apiVersion: v3.0.2                    # Open Data Contract Standard
kind: DataContract
id: acme.commerce.orders
name: orders
version: 2.1.0                        # 语义化版本：破坏性变更 = major
status: active
domain: commerce

description:
  purpose: 已结算订单事实表，收入口径的唯一来源
  usage: 财务对账、留存分析、订单问答 RAG 索引
  limitations: 不含未支付订单；退款以负行表示，不做原地更新

servers:
  - server: prod
    type: iceberg
    catalog: rest_prod
    schema: silver
    location: s3://acme-lake/silver/orders/

schema:
  - name: orders
    physicalType: iceberg
    properties:
      - {name: order_id,  logicalType: string, required: true, unique: true, primaryKey: true}
      - {name: tenant_id, logicalType: string, required: true,
         criticalDataElement: true}         # 分区/隔离键，变更需安全评审
      - name: customer_email
        logicalType: string
        required: true
        classification: pii                 # ← 触发脱敏 + 删除传播 + 访问审计
        transformSourceObjects: [crm.customers.email]
      - name: amount_minor
        logicalType: integer
        required: true
        description: 最小货币单位整数；禁止浮点
        quality: [{rule: valueRange, mustBeBetween: [-100000000, 100000000]}]
      - name: currency
        logicalType: string
        required: true
        quality: [{rule: enumeration, values: [USD, EUR, JPY, CNY]}]
      - name: status
        logicalType: string
        required: true
        quality:                            # unknown = canary 值，见下文
          - {rule: enumeration, values: [paid, refunded, chargeback, unknown]}
      - {name: occurred_at, logicalType: timestamp, required: true, partitionKeyPosition: 1}

slaProperties:
  - {property: frequency,    value: 15, unit: minute}            # 写入频率
  - {property: latency,      value: 30, unit: minute, element: orders.occurred_at}
  - {property: completeness, value: 0.999}                       # 相对源的行数偏差上限
  - {property: retention,    value: 7,  unit: year}

team:                                       # 必须是有名有姓的人，不是"数据组"
  - {username: liwei, role: owner}
  - {username: data-platform, role: steward}
support: [{channel: "#data-commerce", tool: slack}]

customProperties:
  - property: breakingChangePolicy
    value: "破坏性变更需 90 天弃用期 + 全部下游 owner approve"
  - property: downstreamConsumers           # 由血缘工具自动回填，不手写
    value: ["finance.revenue_daily", "growth.retention", "rag.orders_index"]
```

### CI 门禁（没有这一步，契约就是过期的文档）

```
PR 修改契约 YAML
   ├─ ① 静态：schema 语法、必填字段、PII 分类是否缺失
   ├─ ② 兼容性：与上一版做 diff，判定 BACKWARD / FORWARD / FULL
   │        破坏性变更 → 要求 major 版本号 + 下游 owner 的 CODEOWNERS approve
   ├─ ③ 契约 ↔ 实现一致性：拿契约生成断言，跑在 staging 表的样本上
   ├─ ④ 生成物：Avro/Protobuf schema、dbt source yml、质量测试、文档
   └─ ⑤ 影响面：查血缘图，把受影响的下游模型与 dashboard 贴到 PR 评论里
```

**兼容性判定表**（Avro / Confluent Schema Registry 语义）：

| 变更 | BACKWARD（新代码读旧数据） | FORWARD（旧代码读新数据） | FULL |
|---|---|---|---|
| 加带默认值的字段 | ✅ | ✅ | ✅ |
| 加不带默认值的必填字段 | ❌ | ✅ | ❌ |
| 删带默认值的字段 | ✅ | ✅ | ✅ |
| 删必填字段 | ✅ | ❌ | ❌ |
| 类型放宽（int → long） | ✅ | ❌ | ❌ |
| 类型收紧（long → int） | ❌ | ✅ | ❌ |
| 重命名字段 | ❌（= 删 + 加） | ❌ | ❌ |
| **新增枚举值（enum value）** | ✅ | ❌ | ❌ |

**默认选 BACKWARD**（消费者先升级），因为分析场景的消费者数量远大于生产者，且你无法协调所有 BI 用户同时升级。

**新增枚举值是数据侧和 API 侧同构的炸弹**（见 [04-api-design-and-versioning.md](04-api-design-and-versioning.md) §7）：契约里必须写死"消费者遇到未知枚举归入 `unknown`"，并在 v1 就注入一个真实但罕见的 canary 值让不合规的消费者提前暴露。

> **面试金句**：
> "数据契约的价值不在 YAML，在 CI 门禁。没有门禁的契约就是文档，而文档一定会过期。我会让契约变更走 PR、兼容性检查是必过 check、破坏性变更需要下游 owner approve —— 把'数据是产品'这句口号变成一个会挡住合并按钮的机器人。"

---

## 5. 数据质量与新鲜度 SLO

数据的 SLO 和服务的 SLO 是同一套方法（见 [05-reliability/01-slo-and-error-budget.md](../05-reliability/01-slo-and-error-budget.md)），只是 SLI 不同。

| SLI | 定义 | 典型 SLO |
|---|---|---|
| **新鲜度（freshness）** | `now() - max(event_time)` | 工作时段 p99 ≤ 30 min |
| **完整性（completeness）** | `count(target) / count(source)` | ≥ 99.9%，按天 |
| **有效性（validity）** | 通过全部规则的行占比 | ≥ 99.99% |
| **唯一性（uniqueness）** | 主键重复行数 | = 0（硬门禁） |
| **分布漂移（distribution drift）** | 关键列的 null 率 / 基数 / 分位数相对 7 日基线的偏移 | 偏移 > 3σ 告警 |
| **任务准时率** | 在 SLA 时刻前完成的天数比例 | ≥ 99%（月） |

**阻断（blocking）vs 观测（observing），这是要显式做的决策：**

```
Bronze → Silver ：观测式 + quarantine
   坏行不阻断管道，写入 orders_rejected 表（带 rule_id、raw payload、检测时间）
   理由：一条脏数据不该卡死整条链路；但也绝不能丢，否则你永远查不清

Silver → Gold ：阻断式（circuit breaker）
   质量门禁不过 → 不发布新分区，保留上一版可用
   理由：一个错误的收入指标，比一个延迟 2 小时的收入指标贵得多
```

**quarantine 表的行数本身就是一个 SLI**，且它是唯一能提前发现"源系统悄悄改了格式"的信号。没有 quarantine，你的第一个信号会是下游的告警，那时候已经晚了几小时。

---

## 6. 指标层（metrics layer）/ 语义层（semantic layer）

**问题**：`revenue` 这个指标在 BI 里有 4 个定义（含不含退款？按下单时间还是结算时间？含不含税？多币种怎么换算？），在 4 个 dashboard 里给出 4 个数字，然后每个季度末都要开一次"到底哪个对"的会。

**指标层（[dbt Semantic Layer / MetricFlow](https://docs.getdbt.com/docs/build/about-metricflow)、[Cube](https://cube.dev/)、LookML 等）的价值**：把指标定义变成一份版本化的、可测试的代码，BI、Notebook、API、Agent 都从同一个语义端点查询。

```yaml
semantic_models:
  - name: orders
    model: ref('fct_orders')
    defaults: {agg_time_dimension: occurred_at}
    entities: [{name: order_id, type: primary}, {name: tenant, type: foreign, expr: tenant_id}]
    measures:
      - {name: gross_amount,  agg: sum, expr: amount_minor}
      - {name: refund_amount, agg: sum,
         expr: "case when status='refunded' then -amount_minor else 0 end"}
    dimensions:
      - {name: occurred_at, type: time, type_params: {time_granularity: day}}
      - {name: currency, type: categorical}
metrics:
  - name: net_revenue
    label: 净收入（已结算，含退款冲销，按结算时间）
    type: derived
    type_params: {expr: "(gross_amount + refund_amount) / 100.0",
                  metrics: [gross_amount, refund_amount]}
```

**指标层在 LLM 时代的价值被显著放大了。** 让 Agent 直接写 SQL 打你的仓库，你会同时得到：错误的 join、错误的口径（metric definition）、以及一次全表扫描（full table scan）的账单。让它调用语义层，它只能选**已定义的指标 × 已定义的维度**，口径正确性和成本上界都被约束住了。

> 把 Text-to-SQL 换成 Text-to-Metric，是把一个开放性问题换成了一个封闭性问题。前者是研究课题，后者是工程问题。

**代价（要说清楚）**：语义层是一个新的查询规划层，它有自己的性能特性（复杂 metric 的 SQL 生成可能很丑）、自己的缓存问题、以及一个必须有人维护的 DSL。**只有当"同一个指标有多个消费者"时才值** —— 单一 dashboard 的团队上语义层是纯负担。

---

## 7. 反向 ETL（reverse ETL）：把仓库的结论推回业务系统

```
Gold 层的 customer_health_score / churn_risk / usage_tier
        │
        ▼  Hightouch / Census / 自建
   Salesforce 字段  |  Intercom 属性  |  广告受众  |  应用内 feature flag
```

**四条工程要求：**

1. **必须幂等（idempotent）+ 增量**：按行做 diff（存上次同步的哈希），只推变化的行。全量推送会在第一天就打爆目标 SaaS 的 API 配额（多数 SaaS 的写 API 在每秒几十到几百次量级）。
2. **必须限速并处理 429**：目标系统的限流是你的硬约束，见 [04-api-design-and-versioning.md](04-api-design-and-versioning.md) §6。
3. **PII 扩散（PII sprawl）是主要风险**：反向 ETL 是把仓库里的 PII 复制到 N 个第三方系统的最快途径。契约里的 `classification: pii` 必须能阻止字段被同步到未授权的目的地。
4. **绝不放进关键路径（critical path）**：仓库有小时级延迟、有批处理失败、有回填。用它驱动"销售线索评分"可以，用它驱动"能不能下单"不行。

---

## 8. 成本治理：真实的旋钮与量级

**扫描量计费的量级锚点（2026 年中，随时变动）**：BigQuery 按需约 **$6.25 / TiB 扫描**，Athena 约 **$5 / TB 扫描**，Snowflake 按 warehouse 秒计信用点（$2–4/credit 区间，随版本与地区）。

| 旋钮 | 典型收益（量级） | 代价 / 陷阱 |
|---|---|---|
| **分区剪枝（partition pruning）**（按查询最常见的时间过滤列分区） | **10–100×** 扫描量下降 | 分区过细 → 小文件与元数据爆炸。经验值：**单分区 ≥ 1 GB**，分区总数 < 10 万 |
| **聚簇 / 排序（clustering / sorting）**（按 tenant_id、user_id 等高选择性列） | **2–10×** | 需要定期 re-cluster，是持续成本 |
| **文件大小治理**（8 MB → 512 MB） | 规划耗时降一个数量级，扫描吞吐显著提升 | compaction 作业本身要算钱 |
| **列裁剪（column pruning）**（禁止 `SELECT *`） | 宽表上常见 **3–10×** | 需要在 lint/CI 里挡，靠人自觉无效 |
| **物化/增量而非全量重算** | 大表日更常见 **5–20×** | 迟到数据回填逻辑（§3） |
| **结果缓存（result cache）+ BI 只查 Gold** | 消除重复扫描 | BI 工具默认会绕过物化层直查明细 |
| **冷热分层（hot/cold tiering）+ 保留策略（retention policy）** | 存储费线性下降 | 保留期是合规问题，不是成本问题，要一起决策 |
| **压缩用 zstd 而非 snappy** | 存储与扫描 **-20~30%** | 解压 CPU 略高，对 CPU 计费的引擎要实测 |

**治理机制比技术手段更重要：**

- **每个查询强制打标签**（`team` / `dashboard_id` / `job_id`），按扫描量出账（chargeback）到具体团队。没有归因（attribution）就没有节约动机。
- **给按需引擎设硬上限**（BigQuery 的 `maximum_bytes_billed`），防止一次手滑的 `SELECT *` 烧掉一个月预算。
- **测量"每个 dashboard 的 30 天成本 / 30 天独立查看人数"**。这个比值会暴露出你最大的一笔浪费：**没人看但每小时刷新的 dashboard**。在多数公司这一项能占分析总成本的 20–40%。

---

## 9. AI 时代的数据平台

这是 2024–2026 新增的一整层职责，也是最容易做错的地方。

### a) RAG 文档管道 = 一条要求增量与可删除的 ETL

```
源文档（Confluence / Git / S3 / 工单系统）
   │  ① 变更检测：源侧 webhook 或轮询 (etag/mtime)
   ▼
解析 → 规范化（doc_id, doc_version, content_hash, acl, tenant_id）
   │  ② 分块：数百 token/块（典型 512–800），保留结构边界
   ▼
chunks 表（Iceberg）：chunk_id, doc_id, doc_version, chunk_hash, text, position, acl
   │  ③ 只对 chunk_hash 变化的块重新计算
   ▼
上下文化（可选）→ embedding → 向量索引 + BM25 索引
```

**增量更新的关键在 chunk 级哈希**（分块 chunking 的粒度决定了增量粒度）：一次文档编辑通常只改动 1–3 个 chunk，按文档整体重嵌（re-embedding）是 10–50× 的浪费。成本锚点（2026 年中量级，随时变动）：embedding 约 **$0.12–0.15 / 百万 token**；Anthropic 的 contextual retrieval 做法给每个 chunk 生成 50–100 token 的上下文，一次性成本约 **$1.02 / 百万文档 token**（2024 年官方口径），配合 prompt caching 可再降。

### b) 删除传播（deletion propagation）：这是最容易漏的一条

一次"删除这个用户的数据"要传播到**六个地方**：

```
□ 源系统的行
□ Lakehouse 的 Silver/Gold 表        DELETE FROM ... （只写了一个删除标记）
□ Lakehouse 的【历史快照】           ← 必须 expire_snapshots + remove_orphan_files
□ chunks 表 + 向量索引               ← HNSW 只做墓碑标记，真正回收要重建
□ 派生物：embedding 缓存、语义缓存、特征表、导出快照
□ 下游第三方（反向 ETL 推过去的）     ← 需要一份"推送到哪儿了"的台账
```

⚠️ **快照过期（snapshot expiration）是最常被审计抓到的一条**：`DELETE FROM orders WHERE user_id = ?` 在 Iceberg/Delta 里只是写了一个 deletion vector，**旧快照仍然可以读到原值**。如果你的 `history.expire.max-snapshot-age-ms` 是 30 天，而合规删除 SLA 是 30 天，你在数学上就不可能达标。规则：**快照保留期必须显著小于删除 SLA**（比如保留 7 天 / SLA 30 天）。

**Crypto-shredding 是更省事的方案**：每个用户/租户一把数据密钥，PII 列加密后落盘，删除时只删密钥。代价是加密列无法直接过滤/聚合（要么留明文的哈希列做等值查询，要么接受全扫）。与 [03-saas-platform/04-isolation-and-compliance.md](../03-saas-platform/04-isolation-and-compliance.md) 的 BYOK 是同一套机制。

### c) Embedding 版本管理：不能原地升级（in-place upgrade）

**索引的身份必须包含全部影响向量语义的东西**，换模型的正确剧本是新建索引 + 双写（dual write）+ 影子查询（shadow query）+ 灰度切读（gradual rollout），而不是改旧索引：

```
index_key = (model_id, model_version, dim, chunk_strategy_version, normalize?, prefix_template_hash)

1. 新建一个 index_key 不同的索引（不要碰旧的）
2. 双写：新文档同时进新旧索引；存量做后台回填
3. 影子查询：线上流量复制到新索引，离线比对 Recall@10 / MRR + 人工抽检
4. 按租户灰度切读（1% → 10% → 50% → 100%）
5. 旧索引留至少一个回滚窗口（7 天）再删
```

⚠️ **绝不能"边换模型边查询"**：同一个索引里混着两个模型的向量，相似度计算在数学上就是无意义的，而且**不会报错**，只会让召回（recall）悄悄变差 —— 这是 RAG 系统最隐蔽的一类事故。（Matryoshka 表示可以把 3072 维截到 256 维，通常只损 2–3% 精度、存储降 4×；但截断维度同样是 `index_key` 的一部分。）

### d) 血缘（data lineage）要能反查到人

```
用户 U → 源记录 R → Silver 行 → chunk C_17 → 向量 V → 被检索 → 进 Agent 上下文 → 回答 A
```

这条链在两个场景是硬需求：**合规删除**（正向传播）与**事故追责/审计**（反向追溯"这个错误回答基于哪份文档的哪个版本"）。要求 chunk 表里存 `doc_id + doc_version + source_record_id`，Agent 轨迹日志里存被检索的 `chunk_id` 列表 —— 而不只是拼好的 prompt 文本。

---

## 10. 什么时候不要建数据平台（反模式 anti-pattern）

| 反模式 | 为什么错 |
|---|---|
| **先建平台，后找用例** | 最常见也最贵。判据：**先有三个明确的、有 owner 的分析问题，再建管道**。一个只读副本 + 几个物化视图能撑到相当大的规模 |
| **让业务方直接查 Bronze** | Bronze 没有契约、没有清洗、schema 跟着源系统抖。一旦有人建了依赖它的 dashboard，你就再也改不动 ingestion 了 |
| **每分钟 commit 追求"实时"** | 小文件 + 元数据爆炸 + 提交冲突。先问清楚业务能不能接受 15 分钟。**"实时"的需求 90% 是没被追问过的** |
| **用 lakehouse 做点查（point lookup）** | 单行按 key 查在 lakehouse 上是 100 ms–数秒。那是 OLTP 或 KV 的活 |
| **契约只有 YAML 没有门禁** | 见 §4。没有 CI 检查的契约在 3 个月内必然与实现脱节 |
| **指标定义散落在 BI 工具里** | 同一个 `revenue` 有 4 个值，每季度末开一次会 |
| **Data Mesh 教条化** | "域自治"可行且正确；"每个域自建平台"= N 份重复基础设施 + N 套不兼容的质量标准。**可行的部分是：域拥有数据产品（data product）+ 中央提供自助平台（self-serve platform）+ 联邦治理（federated governance）定标准**，三者缺一不可 |
| **反向 ETL 进关键路径** | 仓库的可用性是"小时级、会失败、会回填"，不是在线服务级 |
| **忘了快照过期** | 合规删除在审计时被发现"数据还在" |
| **原地升级 embedding 模型** | 混合向量空间，静默召回退化 |
| **把 embedding 存在 lakehouse 里靠全表扫做检索** | 百万级向量的暴力扫描（brute-force scan）是秒级；向量索引是毫秒级。存可以在湖里，查必须进索引 |
| **用 Text-to-SQL 直连生产仓库给 Agent** | 口径错误 + 无成本上界 + 权限难约束。走语义层 |

**最后一条判据**：

> 数据平台的成熟度不看你用了什么表格式，看两个数字：
> **① 从"源系统改了一列"到"下游 owner 收到告警"要多久**（好的答案：CI 阶段就挡住，根本进不了 prod）；
> **② 一次"删除某用户全部数据"的请求，需要几个人手工介入**（好的答案：0）。
> 这两个数字答不上来的平台，不管栈多现代，都还在"数据沼泽（data swamp）"阶段。

---

## 面试官会追问

1. 从 OLTP 到分析的三条路径，你会按什么顺序演进？每条路径的撞墙信号分别是什么？
2. Iceberg 的快照隔离是怎么实现的？两个写入者同时提交会发生什么？这对提交频率有什么约束？
3. 流式写入每分钟 commit 一次，一个月后查询慢了 50 倍，为什么？怎么诊断、怎么修？`remove_orphan_files` 的时间阈值设小了又会发生什么？
4. 数据契约里加一个新的枚举值，是破坏性变更吗？对谁破坏？怎么提前暴露？
5. BACKWARD 和 FORWARD 兼容你会默认选哪个？为什么分析场景和消息队列场景的选择可能不同？
6. 用户要求删除全部数据，你需要动几个地方？为什么"DELETE FROM 跑成功了"不等于删干净了？
7. 要换 embedding 模型，给我一个不会让线上召回退化的剧本。为什么不能原地升级？
8. 你的分析成本一年涨了 3 倍，给我三个最可能的原因和对应的排查手段。为什么要有语义层？只有一个 dashboard 还需要吗？

---

**下一篇** → [../03-saas-platform/01-control-plane.md](../03-saas-platform/01-control-plane.md)
