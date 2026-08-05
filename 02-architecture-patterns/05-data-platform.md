# 05 · 数据平台：Lakehouse、契约与指标层

> 表格式、目录和查询引擎已经提供了许多成熟积木，但数据平台的技术难点并没有消失：提交冲突、小文件、成本、删除与跨引擎兼容仍需工程治理。
> 更容易被忽略的是责任问题：**谁对这张表的正确性负责**。数据契约（data contract）把口头约定变成可评审、可测试、必要时会挡住合并按钮的规则。

---

## 读这一章之前

**你在工作中遇到过这个**

上游后端把 `orders.status` 的取值从 `paid` 改成了大写 `PAID`，一行代码的事，没人想起来通知数据组。
当天的收入看板依然是绿的 —— 口径写死了 `status = 'paid'`，匹配不到就是 0 行，画出来是一条完美的平线，没有任何告警。
三天后 CFO 在经营会上问"周三收入为什么是 0"，你才开始查。
同一周，那个每分钟提交一次的流式作业跑满了一个月，同样的查询从 200 ms 变成 40 秒，而数据量只涨了 3 倍。

**需要先懂的概念**

| 概念 | 一句话 | 详见 |
|---|---|---|
| 副本 / 分片 | 同一份数据存多份 vs. 把数据切成互不重叠的块分开放 | [00-concepts §5](../00-foundations/00-concepts.md#5-副本分片分区--三个被混用的词) |
| 快照隔离 / 乐观并发 | 读的人锁定一个瞬间的一致视图；写的人靠版本号抢，抢输了重来 | [00-concepts §7](../00-foundations/00-concepts.md#7-事务与隔离级别) |
| 最终一致 | 分析链路通常滞后于业务库；若业务需要“多久内可见”，还要另外定义新鲜度 SLO | [00-concepts §6](../00-foundations/00-concepts.md#6-什么是一致性--一个词两种完全不同的意思) |
| 幂等 | 同一批数据被重复处理一次，结果不能变 | [00/01 §5](../00-foundations/01-fundamentals.md#5-幂等idempotency分布式系统的第一公民) |
| 消息、流与 Outbox | 变更怎么可靠地从业务库流到下游 | [01/03](../01-building-blocks/03-messaging-and-streams.md) |
| SLO 与错误预算 | 用可测量的指标承诺质量，超支就停下来修 | [05/01](../05-reliability/01-slo-and-error-budget.md) |

**这一章要回答的问题**

1. 业务库里的数据怎么变成能被分析的数据？只读副本、批量导出、CDC 三条路，什么时候该走下一步？
2. 对象存储不能像数据库那样原地更新一行，Iceberg 这类表格式如何用不可变文件和原子元数据提交提供表级事务？提交频率受哪些因素约束？
3. 上游偷偷改了一列，怎么在下游看板算错之前就把它挡住？
4. 用户要求"删除我的全部数据"，为什么 `DELETE` 跑成功了还不等于删干净了？

**本章新引入的术语**

| 术语 | English | 一句话定义 |
|---|---|---|
| 湖仓 | lakehouse | 在对象存储的一堆文件之上加一层事务元数据，让"文件的集合"具备数据库式的表语义 |
| 表格式 | table format | 一套约定，规定哪些文件属于这张表、表现在的结构是什么、以及如何原子地把一批文件换成另一批 |
| 目录 | catalog | 保存"表名 → 当前元数据文件"这个唯一可变指针的服务，是所有查询引擎的会合点 |
| 快照 | snapshot | 表在某一时刻的完整文件清单；读的人一旦拿到它，之后别人怎么写都不影响这次读 |
| 小文件问题 | small files problem | 高频写入产生海量几 MB 的碎文件，让查询在读到数据之前就先被"清点文件"拖垮 |
| 合并压实 | compaction | 后台把大量小文件重写成少量大文件的维护作业 |
| CoW / MoR | copy-on-write / merge-on-read | 改一行就重写整个数据文件（写慢读快）/ 只追加一条"这行已删/已改"的记录，读的时候再合并（写快读慢） |
| 数据契约 | data contract | 一份版本化、有署名 owner、且能在 CI 里被机器检查的表结构与质量约定 |
| 分层（青铜/白银/黄金） | medallion architecture | 原始层（只追加、可重放）→ 明细层（清洗建模、有契约）→ 消费层（面向具体用途的汇总） |
| 隔离区表 | quarantine table | 专门接收"校验没通过但绝不能丢"的坏行的表，它的行数本身就是一个告警信号 |
| 指标层 / 语义层 | metrics layer / semantic layer | 把每个指标的口径写成一份版本化代码，看板、笔记本、Agent 都只能从这里取数 |
| 反向 ETL | reverse ETL | 把仓库里算出来的结论（如流失风险分）写回业务 SaaS（CRM、客服系统）的同步作业 |

---

## 1. OLTP → 分析：三条路径

**OLTP**（online transaction processing）就是你的业务库：海量小事务、按主键读写、要求毫秒级返回。
**OLAP**（online analytical processing）是分析侧：查询很少但每条都要扫过成百上千万行做聚合。
两者的访问模式差异很大（前者常按少量行读写，后者常扫描少量列并聚合），所以分析数据通常要复制到更合适的布局。下面是三条常见路径，不是穷举：查询联邦、业务事件直写和托管连接器也可能参与。
第三条里的 **CDC（change data capture，变更数据捕获）**通常读取数据库日志（MySQL binlog、PostgreSQL WAL 等），也有基于触发器或时间戳查询的实现。日志型 CDC 输出的是在日志位置上可排序的行级变更；跨分片、跨事务源并不天然存在一个全局发生顺序。

| 路径 | 新鲜度 | 对 OLTP 的压力 | 复杂度 | 撞墙条件与信号 |
|---|---|---|---|---|
| **直读只读副本（read replica）** | 通常秒级到分钟级 | 中（大扫描会争用副本 IO/CPU，并可能放大 lag） | 最低 | **信号**：分析查询排队、复制或回放延迟超出 SLO、查询影响故障转移容量；阈值由实例与查询实测 |
| **批量/增量导出（batch export）** | 取决于调度周期 | 通常较低，但快照和全量扫描仍会耗源库资源 | 低到中 | **信号**：作业重叠、窗口内跑不完、回补困难，或业务要求的新鲜度小于调度周期 |
| **CDC → 流 → 湖** | 秒到分钟 | 通常低于分析扫描，但解码、复制槽和 WAL 保留仍有成本 | **高** | 从第一天就要处理 schema 变更、迟到事件、断点重放、快照+增量接缝，以及消费者停滞导致的日志堆积 |

**常见演进顺序**是只读副本 → 批量/增量导出 → CDC，因为每一步都增加运维面；但它不是固定阶梯。审计日志、秒级风控或源库不允许分析扫描时，第一天就上 CDC 可能合理。判断依据是新鲜度、可重放、源库压力和团队维护能力，不是 ARR 或公司人数。

CDC 的实现细节（Outbox、快照与增量的一致接缝、幂等消费）见 [01-building-blocks/03-messaging-and-streams.md](../01-building-blocks/03-messaging-and-streams.md)。这里只强调一条：**CDC 给你的是"行变更流（row-change stream）"，不是"业务事件流（business event stream）"**。`UPDATE orders SET status='shipped'` 这条 CDC 记录不告诉你"为什么发货"。把 CDC 当业务事件用，会让下游被迫从列的变化里反推语义，而这个反推逻辑会在源系统重构时静默失效。

---

## 2. Lakehouse 表格式：核心机制

三种格式（[Apache Iceberg](https://iceberg.apache.org/) / [Delta Lake](https://delta.io/) / [Apache Hudi](https://hudi.apache.org/)）都用不可变数据文件、版本化元数据和原子提交，在对象存储之上提供数据库式的表语义。具体支持哪些事务、并发写和隔离能力，要以所选格式、catalog 与引擎组合为准；“支持 ACID”不等于跨表事务或任意更新都和 OLTP 数据库一样。

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

1. **快照隔离来自不可变文件 + 固定快照引用**（见 [00-concepts §7](../00-foundations/00-concepts.md#7-事务与隔离级别)）：读取者先解析一个快照，在这次查询里使用同一文件集合。它不是“免费”的——元数据读取、旧快照保留、垃圾回收和长查询保护都要成本。
2. **写入冲突通常靠乐观并发控制**：两个写入者都基于 v7 生成新元数据时，只有一个原子比较并更新当前指针成功；另一个要重读、验证自己的文件是否仍可提交，再重试或失败。高频小提交会增加元数据和冲突，但不存在通用的“每分钟上限”：可接受频率取决于 catalog、写入者数量、分区重叠、文件数和重试预算，要压测。
3. **查询在读数据前先付元数据规划成本。** manifest 的分区/列统计可剪掉无关文件；剪枝比例取决于数据布局和谓词。manifest 与小文件过多时，列文件清单、打开对象和调度任务本身就可能成为主要延迟。
4. **列由 ID 标识，不由名字标识**。所以在 Iceberg 里重命名列是纯元数据操作，不需要重写数据 —— 这是它相对 Hive 表最实用的一个差异。⚠️ 但**表格式安全 ≠ 契约安全**：重命名对下游的 SQL 仍然是破坏性变更（见 §4）。

### 三家怎么选：先验证组合，不背功能表

Iceberg、Delta Lake、Hudi 都在快速演进，同一个功能也可能只在部分引擎或版本中可读、不可写。选型时做一个与你实际版本绑定的兼容矩阵：

| 要验证的问题 | 为什么重要 |
|---|---|
| 生产查询/写入引擎能否读、写、升级目标表格式版本 | “规范支持”不等于你当前引擎实现完整 |
| 并发 append、overwrite、merge、schema evolution 的冲突行为 | 决定失败后能否安全重试，以及会不会丢更新 |
| CoW / MoR、行级 delete/upsert 与 compaction 的读写代价 | CDC 表和只追加事实表的最佳选择可能相反 |
| catalog 的原子提交、权限、审计、分支/标签与容灾 | catalog 是提交协调点，也是控制面依赖 |
| 另一种引擎能否独立读回同一张表 | 用实际文件和故障演练验证“引擎中立”，不要只看 logo |
| 快照过期、孤儿清理、升级和回滚工具 | 日常维护与删除 SLA 往往比建表更难 |

三个词先解释清楚：**upsert** = 这行在就更新、不在就插入。**CoW（copy-on-write）**通常在写入时重写受影响的数据文件，写放大更高、读取路径更简单。**MoR（merge-on-read）**把变更写入增量文件或删除结构，读取或 compaction 时再合并，降低前台写成本但增加读取与维护复杂度。不同格式的 delete file / deletion vector 表示法并不完全相同，是否更小、更快要以数据分布和引擎实现为准。

几个值得关注、但必须按当前版本核对的方向：REST catalog 把 catalog 交互协议化；对象存储的条件写可用于某些原子创建/比较更新流程；表格式持续加入行级删除、半结构化类型和血缘能力；UniForm、[Apache XTable](https://xtable.apache.org/) 等尝试生成兼容元数据。它们降低迁移成本，但不会自动统一权限语义、所有数据类型、删除行为或写入冲突。

> **面试金句**：
> “表格式和 catalog 要一起选。我的第一步不是比较功能清单，而是拿真实表做并发写、schema 演进、删除、故障恢复和跨引擎读写测试；再把缺失能力和迁移路径写进 ADR。”

### 小文件（small files problem）与 compaction：最常见的维护旋钮

```
流式写入，每分钟 commit，20 个分区
  → 1,440 次 commit/天 × 20 文件 = 28,800 个文件/天
  → 每个文件 2 MB（远低于目标），查询要开 28,800 次 GET
  → 元数据侧：1,440 个快照 + 上万个 manifest → 规划、列举与任务调度成本持续增长
```

目标文件大小常落在数百 MB 的量级，但没有跨引擎通用值：对象存储请求成本、查询并行度、行宽和写入延迟都会改变最优点。下面是 **Iceberg + 某些 Spark procedure 的示意**；名字、默认值和参数随引擎/版本变化，上线前查对应文档。常见维护包括：

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

⚠️ `remove_orphan_files` 的安全窗口至少要覆盖最长运行作业、重试/暂停时间、时钟误差和对象可见性假设，并先在测试表或 dry-run 清单验证；否则可能把尚未提交但仍在使用的文件当孤儿。compaction 和前台写入可能争用提交点；可通过限制重叠分区、分批提交、调度低峰和冲突重试降低影响，而不是默认打开某一个参数就结束。

---

## 3. 分层（layering）：Medallion 与批流一体的现实

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 源   OLTP(Postgres/MySQL)    SaaS API      事件流(Kafka)    文档/对象     │
└────┬───────────────┬──────────────┬───────────────┬──────────────────────┘
     │ CDC           │ 增量拉取      │ 直接落        │ 对象存储事件通知
     ▼               ▼              ▼               ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ BRONZE 原始层   尽量保留源形状与摄取元数据，通常只追加                    │
│   常按 ingest_date 分区   保留期按可重放/合规/成本决定                    │
│   至少有 envelope 契约；消费权限受控，不作为稳定业务接口                  │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ 去重 / 类型规约 / 迟到窗口 / 【契约校验】
                                │ 校验不过的行 → quarantine 表（不丢）
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ SILVER 明细层   清洗后的实体与事件；事实表一行一事实，维表按需用 SCD2     │
│   分区/聚簇由查询与数据分布决定；有消费级数据契约                          │
│   ★ Bronze 管摄取 envelope；Silver 起承诺业务字段与质量语义                │
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

图里最下面那行出现了 **RAG**（retrieval-augmented generation，检索增强生成：模型回答之前，先按问题从外部知识库里检索出若干片段拼进 prompt，让它基于这些材料作答，而不是只靠训练时记住的东西 —— 完整讲法见 [`04-ai-agent-systems/02`](../04-ai-agent-systems/02-context-engineering-and-rag.md)）；
对数据平台来说，它只是 Gold 层的又一个消费者，且是要求"能增量更新、能按行删除"的那一类（§9 展开）。

图里 SILVER 层提到的 **SCD2**（slowly changing dimension type 2，缓慢变化维第 2 型）用于需要追溯历史属性的**维表**：实体属性变化时不覆盖旧行，而是新增版本并记录生效区间。于是能回答“这个客户在去年 3 月属于哪个套餐”。不是所有 Silver 表都该用 SCD2；事件/事实表通常追加事实，当前状态表也可能只保留最新版。

**“批流一体（unified batch and streaming）”有多个层次**：可以共享表、schema、质量规则和计算 API，也可能仍由不同执行引擎或作业负责。不要因为营销名称就假定“同一份代码、同一语义、同一运维方式”；要逐项验证事件时间、重放、更新和 checkpoint 语义是否一致。

**增量物化（incremental materialization）**常能显著降低 Gold 层成本，但会增加去重、迟到数据、更新/删除传播和回填复杂度。只有在这些语义可证明正确时，才用它替代全量重算。

```sql
-- “读某个快照的完整状态”和“读两个快照之间的变化”不是一回事。
-- 前者可用 time travel；后者要用目标引擎明确提供的 incremental/changelog API。
-- 不要把 FOR SYSTEM_VERSION AS OF 误当成 snapshot diff。
-- dbt 侧
{{ config(materialized='incremental', unique_key='order_id', incremental_strategy='merge') }}
WHERE _ingested_at > (SELECT coalesce(max(_ingested_at), '1970-01-01') FROM {{ this }})
```

⚠️ **增量物化的第一个坑是迟到数据（late-arriving data）**：`WHERE occurred_at > max(occurred_at)` 会漏掉事件时间较旧、但刚到达的数据。一种批处理做法是用单调的摄取位置或 `_ingested_at` 推进增量读取，用**事件时间（event time）**做业务分区，再按实测迟到分布回填最近一段窗口；若更新和删除也要传播，仅有时间戳还不够，还要保存源日志位置/版本并做幂等 merge。

（**事件时间水位线 watermark**：流处理器对“事件时间大致已推进到哪里”的估计，用来触发窗口计算和清理状态；水位线之前再来的事件叫迟到事件。它不必然“永远不计算”：引擎可配置 allowed lateness，并选择更新旧窗口、写侧输出/quarantine 或丢弃。**回填 backfill**则把历史区间重算并以可审计方式替换结果。策略必须与业务允许修正多久以前的数据相匹配。）

---

## 4. 数据契约：把口头约定变成 CI 门禁（CI gate）

**数据契约不是文档，是一个可执行的、版本化的、有 owner 的接口定义。** 参考规范：[Open Data Contract Standard (ODCS)](https://bitol-io.github.io/open-data-contract-standard/)（Bitol / LF AI & Data）与 [datacontract.com](https://datacontract.com/) 的规范。

```yaml
# 结构示意，不保证可被某个工具原样执行；采用标准时固定并验证具体 schema 版本
apiVersion: <pinned-contract-schema-version>
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

（**血缘 lineage**：一张"哪张表、哪个字段是由哪些上游算出来的"的依赖图，通常由解析 SQL 自动生成，§9d 会展开它的另一半用途。）

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

**常见起点是 BACKWARD**（先让新消费者能读已有数据），但默认值取决于部署顺序和保留数据：如果新旧生产者/消费者会长期并存，可能需要 FULL；表 schema、Avro 消息和 API JSON 的兼容规则也不能直接共用一张表。

**新增枚举值可能是数据侧和 API 侧同构的炸弹**（见 [04-api-design-and-versioning.md](04-api-design-and-versioning.md) §7）：消费者应保留未知原值并走 `unknown` 分支。可以在契约测试/沙箱数据中注入未来值验证兼容性；不要为了测试而向真实生产统计随意注入会污染口径的 canary 行。

> **面试金句**：
> "数据契约的价值不在 YAML，在 CI 门禁。没有门禁的契约就是文档，而文档一定会过期。我会让契约变更走 PR、兼容性检查是必过 check、破坏性变更需要下游 owner approve —— 把'数据是产品'这句口号变成一个会挡住合并按钮的机器人。"

---

## 5. 数据质量与新鲜度 SLO

数据的 SLO 和服务的 SLO 是同一套方法（见 [05-reliability/01-slo-and-error-budget.md](../05-reliability/01-slo-and-error-budget.md)），只是 **SLI** 不同 ——
**SLI**（service level indicator，服务等级指标）是实际测量值，**SLO** 是团队为它设定的内部目标；只有写进合同的对外承诺才是 **SLA**。
要计算错误预算与燃尽率，优先把 SLI 写成 `好事件 / 有效事件`：例如“工作时段内，每个 5 分钟观测点的水位滞后 ≤ 30 分钟”是好事件，SLO 可以是滚动 28 天好事件比例 ≥ 99.9%。`p99 水位滞后 ≤ 30 min` 仍适合做分布诊断或压测目标，但不能直接提供坏事件计数。服务侧通常数“成功且延迟 < X 的请求 / 有效请求”，数据侧则换成下面这些质量维度。

| SLI | 定义注意事项 | 示例目标（需按业务校准） |
|---|---|---|
| **新鲜度（freshness）** | 目标已处理的源日志位置/水位距当前源位置多远；`now()-max(event_time)` 会被无新事件或未来时间戳误导 | 工作时段 p99 ≤ 30 min |
| **完整性（completeness）** | 对同一边界比较源/目标主键、控制总数或金额；简单 count 比会被重复行抵消 | ≥ 99.9%，按天 |
| **有效性（validity）** | 通过全部规则的行占比 | ≥ 99.99% |
| **唯一性（uniqueness）** | 主键重复行数 | = 0（硬门禁） |
| **分布漂移（distribution drift）** | 关键列的 null 率 / 基数 / 分位数相对季节性基线偏移 | 超出按历史误报率校准的阈值告警 |
| **任务准时率** | 在约定截止时刻前完成的天数比例；若该时刻写进客户合同，才称 SLA 时刻 | ≥ 99%（月） |

**阻断（blocking）vs 观测（observing），这是要显式做的决策：**

```
Bronze → Silver ：观测式 + quarantine
   坏行不阻断管道，写入 orders_rejected 表（带 rule_id、raw payload、检测时间）
   理由：一条脏数据不该卡死整条链路；但也绝不能丢，否则你永远查不清

Silver → Gold ：阻断式（circuit breaker）
   质量门禁不过 → 不发布新分区，保留上一版可用
   理由：一个错误的收入指标，比一个延迟 2 小时的收入指标贵得多
```

**quarantine 表的行数/比例本身就是一个 SLI**。schema registry、契约 CI、源端变更事件和运行时解析错误也能提前发现格式变化；quarantine 的独特价值是保住坏行，方便修复后重放，而不是让管道在第一条异常上停死。

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

1. **必须幂等 + 尽量增量**：按目标主键 upsert，并保存同步水位/哈希，只推变化的行。全量同步是否可接受取决于数据量、目标 API 配额和变更比例，不能背一个通用 QPS。
2. **必须限速并处理 429**：目标系统的限流是你的硬约束，见 [04-api-design-and-versioning.md](04-api-design-and-versioning.md) §6。
3. **PII 扩散（PII sprawl）是主要风险**：反向 ETL 是把仓库里的 PII 复制到 N 个第三方系统的最快途径。契约里的 `classification: pii` 必须能阻止字段被同步到未授权的目的地。
4. **绝不放进关键路径（critical path：用户必须等它跑完才能拿到结果的那条链，见 [00-concepts §1](../00-foundations/00-concepts.md#1-一个请求到底经历了什么)）**：仓库有小时级延迟、有批处理失败、有回填。用它驱动"销售线索评分"可以，用它驱动"能不能下单"不行。

---

## 8. 成本治理：真实的旋钮与测量方法

云数仓可能按扫描字节、计算时间/slot、warehouse 大小、存储和请求分别计费，价格还会随地区、合约与版本变化。设计时从当前账单导出 `$/query`、`$/dashboard`、`$/pipeline run`，不要把某个公开标价写成长期容量常数。

| 旋钮 | 典型收益（量级） | 代价 / 陷阱 |
|---|---|---|
| **分区剪枝（partition pruning）**（按常用、低到中基数过滤列分区） | 只读匹配分区，收益取决于谓词选择性 | 分区过细 → 小文件与元数据爆炸；用每分区大小/文件数和规划耗时找阈值 |
| **聚簇 / 排序（clustering / sorting）** | 让列统计更容易剪掉文件；用扫描字节验证收益 | 需要定期重写，是持续成本 |
| **文件大小治理** | 减少对象打开与任务调度；收益由查询引擎实测 | compaction 作业本身要算钱，过大又会降低并行度 |
| **列裁剪（column pruning）** | 宽表上避免读取无关列 | 可在 lint/CI 提醒，但探索查询是否允许 `SELECT *` 要分环境处理 |
| **物化/增量而非全量重算** | 只处理变化数据 | 迟到、更新、删除和回填逻辑（§3） |
| **结果缓存（result cache）+ BI 只查 Gold** | 消除重复扫描 | BI 工具默认会绕过物化层直查明细 |
| **冷热分层（hot/cold tiering）+ 保留策略（retention policy）** | 存储费线性下降 | 保留期是合规问题，不是成本问题，要一起决策 |
| **压缩编码选择** | 更高压缩率可能降低 IO/存储 | zstd、snappy 等在压缩率、CPU 和引擎兼容性间不同，拿真实列分布压测 |

**治理机制比技术手段更重要：**

- **每个查询强制打标签**（`team` / `dashboard_id` / `job_id`），按扫描量出账（**chargeback**：把一笔公共开销按实际用量分摊回发起方团队自己的预算上）到具体团队。没有归因（attribution）就没有节约动机。
- **给按需引擎设硬上限**（BigQuery 的 `maximum_bytes_billed`），防止一次手滑的 `SELECT *` 烧掉一个月预算。
- **测量“每个 dashboard 的周期成本 / 独立查看人数”**。它能暴露没人看却高频刷新的报表；占比必须从自己的账单算，不要套行业百分比。

---

## 9. 专项选读：RAG / AI 的数据管道

普通分析平台读者可以跳到 §10；只有平台还要服务 RAG / Agent 时再读本节。

### a) RAG 文档管道 = 一条要求增量与可删除的 ETL

```
源文档（Confluence / Git / S3 / 工单系统）
   │  ① 变更检测：源侧 webhook 或轮询 (etag/mtime)
   ▼
解析 → 规范化（doc_id, doc_version, content_hash, acl, tenant_id）
   │  ② 分块：大小由检索评测决定，优先保留标题/段落等结构边界
   ▼
chunks 表（Iceberg）：chunk_id, doc_id, doc_version, chunk_hash, text, position, acl
   │  ③ 只对 chunk_hash 变化的块重新计算
   ▼
上下文化（可选）→ embedding → 向量索引 + BM25 索引
```

（**embedding**：把一段文本映射成一个几百到几千维的浮点向量，语义相近的文本得到的向量也相近，于是"找相似的文档"变成了"找距离近的向量"。**chunk**：把长文档切成的一小段，几百 token，是检索和嵌入的最小单位。）

**chunk 级哈希能避免重算内容完全相同的块**，但分块算法的边界一变，后续所有 chunk ID 都可能漂移。应把解析器、分块器、上下文模板与 embedding 模型版本都纳入键，并从当前供应商账单测量“每次文档变更重算多少 token/块”；价格和典型修改块数都不是系统常数。

### b) 删除传播（deletion propagation）：这是最容易漏的一条

一次“删除这个用户的数据”至少要沿真实血缘枚举这些地方（数量不是固定六个）：

```
□ 源系统的行
□ Lakehouse 的 Silver/Gold 表        DELETE/MERGE 产生新表快照
□ Lakehouse 的历史快照与未引用文件   按保留/法律保全策略过期并回收
□ chunks 表 + 向量索引               按产品的逻辑删除与物理回收语义处理
□ 派生物：embedding 缓存、语义缓存、特征表、导出快照
□ 下游第三方（反向 ETL 推过去的）     ← 需要一份"推送到哪儿了"的台账
□ 备份、日志、DLQ 与临时文件           ← 各自有保留期、恢复后再删除流程
```

（**HNSW** 是常见向量索引：把向量按邻近关系连成多层图，查询时逐层逼近。删除行为取决于具体实现：有的先写墓碑、后台重建才回收空间，有的支持在线物理删除但仍可能留在快照/备份。必须分别验证“查询不再返回”“主存储不可读”“备份到期”三个时刻。）

⚠️ `DELETE FROM` 提交的是一个**新快照**；底层可能用 delete file / deletion vector，也可能重写数据文件。无论哪种，仍被旧快照引用的原文件通常不会立刻消失。删除 SLA 必须给快照过期、物理清理、下游传播、失败重试和验证留出总预算，同时处理 legal hold 与备份例外；仅把快照保留期设成等于 SLA，通常没有执行余量。

**Crypto-shredding 可以缩短不可恢复时间，但不是免费捷径**：只有当每份副本都只含密文、所有可解密密钥副本（含备份/缓存）都按同一策略销毁，而且密钥粒度与删除主体一致时才成立。每用户一把 key 会显著增加 KMS、轮换和恢复复杂度；确定性哈希辅助查询仍可能是个人数据。它也不等同于 BYOK，详见 [隔离与合规](../03-saas-platform/04-isolation-and-compliance.md)。

### c) Embedding 版本管理：不同向量空间不能混用

**索引的逻辑身份必须包含全部影响向量语义的东西**。底层可以复用同一个服务或 collection，但新旧向量必须可区分、不可在一次近邻搜索里混算。常见迁移剧本是新逻辑索引 + 双写 + 影子查询 + 灰度切读：

```
index_key = (model_id, model_version, dim, chunk_strategy_version, normalize?, prefix_template_hash)

1. 新建一个 index_key 不同的索引（不要碰旧的）
2. 双写：新文档同时进新旧索引；存量做后台回填
3. 影子查询：线上流量复制到新索引，离线比对 Recall@10 / MRR + 人工抽检
4. 按租户灰度切读（1% → 10% → 50% → 100%）
5. 旧索引保留一个覆盖观察期与回滚操作耗时的窗口，再按删除策略清理
```

⚠️ 不要把**不兼容模型/版本**生成的向量放进同一次相似度比较；数值维度碰巧相同也不代表坐标空间兼容，系统通常不会报错，只会静默降低召回率。某些明确支持 Matryoshka 表示的模型允许截断维度，但质量损失、存储和查询收益都依模型与索引实现而变，必须离线评测；截断维度仍是 `index_key` 的一部分。

### d) 血缘（data lineage）要能反查到人

```
用户 U → 源记录 R → Silver 行 → chunk C_17 → 向量 V → 被检索 → 进 Agent 上下文 → 回答 A
```

这条链在两个场景是硬需求：**合规删除**（正向传播）与**事故追责/审计**（反向追溯"这个错误回答基于哪份文档的哪个版本"）。要求 chunk 表里存 `doc_id + doc_version + source_record_id`，Agent 轨迹日志里存被检索的 `chunk_id` 列表 —— 而不只是拼好的 prompt 文本。

---

## 10. 什么时候不要建数据平台（反模式 anti-pattern）

| 反模式 | 为什么错 |
|---|---|
| **先建平台，后找用例** | 先有明确、有 owner、有新鲜度/质量要求的消费场景，再决定是否超出只读副本、物化视图或托管仓库的能力；用例数量不是固定门槛 |
| **让业务方直接查 Bronze** | Bronze 没有契约、没有清洗、schema 跟着源系统抖。一旦有人建了依赖它的 dashboard，你就再也改不动 ingestion 了 |
| **为了“实时”盲目提高 commit 频率** | 小文件、元数据和冲突都会增长。先定义新鲜度 SLO，再压测 micro-batch 大小与维护成本 |
| **未经验证用 lakehouse 做在线点查** | 文件表通常不为高并发低尾延迟点查优化；若新索引/引擎能满足 SLO 可以用，否则交给 OLTP/KV |
| **契约只有 YAML 没有门禁** | 见 §4。没有自动校验与 owner 评审，契约很容易与实现漂移 |
| **指标定义散落在 BI 工具里** | 同一个 `revenue` 有 4 个值，每季度末开一次会 |
| **Data Mesh 教条化** | "域自治"可行且正确；"每个域自建平台"= N 份重复基础设施 + N 套不兼容的质量标准。**可行的部分是：域拥有数据产品（data product）+ 中央提供自助平台（self-serve platform）+ 联邦治理（federated governance）定标准**，三者缺一不可 |
| **反向 ETL 进关键路径** | 仓库的可用性是"小时级、会失败、会回填"，不是在线服务级 |
| **忘了快照过期** | 合规删除在审计时被发现"数据还在" |
| **原地升级 embedding 模型** | 混合向量空间，静默召回退化 |
| **不做容量验证就靠全表扫向量** | 小数据或离线批任务可以暴力扫描；在线规模增大后通常需要 ANN/专用索引。以召回率、p99、成本和更新延迟决定切换点 |
| **用 Text-to-SQL 直连生产仓库给 Agent** | 口径错误 + 无成本上界 + 权限难约束。走语义层 |

**最后一条判据**：

> 数据平台的成熟度不看你用了什么表格式，看两个数字：
> **① 从"源系统改了一列"到"下游 owner 收到告警"要多久**（好的答案：CI 阶段就挡住，根本进不了 prod）；
> **② 一次"删除某用户全部数据"的请求，需要几个人手工介入**（好的答案：0）。
> 这两个数字答不上来的平台，不管栈多现代，都还在"数据沼泽（data swamp）"阶段。

---

## 这一章的三句话

1. **引擎选型和数据责任缺一不可。** 格式/catalog 决定技术边界，owner、契约和血缘决定变更时谁被通知、谁负责修。
2. **契约的价值来自持续执行。** YAML 只是载体；CI 兼容检查、运行时质量监控、owner 审批与下游迁移流程共同防止它和实现漂移。
3. **`DELETE` 提交成功不等于所有副本都已不可读。** 新快照、历史快照、物理文件、索引、缓存、备份和第三方下游各有自己的删除与验证时钟。

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

**按训练路径阅读** → 回 [START-HERE](../START-HERE.md) 按所选路径继续；页尾链接只表示本目录或专章的顺读顺序。

**目录顺读下一篇** → [../03-saas-platform/01-control-plane.md](../03-saas-platform/01-control-plane.md)
