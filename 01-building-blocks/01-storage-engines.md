# 01 · 存储引擎：从 B-Tree 到向量索引

> 存储选型不是"选 MySQL 还是 Mongo"，是"我的访问模式在哪种数据结构上代价最低"。

---

## 1. RUM 猜想：所有存储的根本约束

**RUM Conjecture**：任何存储结构，在 **Read（读放大）、Update（写放大）、Memory（空间放大）** 三者中，最多优化两个。

```
        Read 优化
           /\
          /  \
         /    \
        /      \
  Update ────── Memory
   优化           优化
```

| 结构 | R | U | M | 典型 |
|---|---|---|---|---|
| B+Tree | ✅ | ❌ | ✅ | Postgres, MySQL InnoDB |
| LSM Tree | ❌ | ✅ | ✅ | RocksDB, Cassandra, ClickHouse* |
| 全内存哈希 | ✅ | ✅ | ❌ | Redis |
| 列存 + 编码 | ✅(分析) | ❌ | ✅✅ | Parquet, ClickHouse |

**记住这个猜想，你就能回答任何"为什么这个数据库读快写慢"的问题。**

---

## 2. B+Tree vs LSM Tree：真正的差别

### B+Tree

```
                [ 50 | 100 ]              ← 内部节点（只有键）
               /     |      \
        [10|30]  [60|80]  [120|150]
        /  |  \    ...       ...
     叶子节点（双向链表相连，存实际数据）
```

**写路径**：
1. 找到目标叶子页（可能多次随机 IO，但内部节点通常在 buffer pool 里）
2. **原地修改（in-place update）** 该页
3. 写 WAL（顺序）+ 脏页（dirty page）异步刷盘（随机）
4. 页满则分裂（page split），可能级联向上

**关键特征**：
- 写放大（write amplification） = 一次逻辑写 → 至少一个完整页（8 KB / 16 KB）落盘。改 100 字节的行 = 写 8 KB → **写放大 80×**
- 读放大（read amplification）低：树高 3–4 层，热数据在内存，通常 **1 次磁盘 IO**
- 空间放大（space amplification）低：页填充率（fill factor） ~70%
- **范围扫描（range scan）快**：叶子页链表相连
- 有 in-place update → **需要锁/latch**，高并发写会有页级竞争

### LSM Tree

```
写入 → MemTable（内存跳表/红黑树）
         ↓ 满了，冻结
       Immutable MemTable
         ↓ flush（顺序写）
       L0: [SST][SST][SST]     ← 键范围重叠
         ↓ compaction
       L1: [SST][SST]...       ← 键范围不重叠，总大小 = L0 × 10
         ↓
       L2: ...                 ← × 10
```

**写路径**：内存写 + WAL 顺序写。**写入极快，且是顺序 IO**。

**读路径**：MemTable → L0（所有文件）→ L1（二分找一个文件）→ L2 → ...
- 每层要查布隆过滤器（Bloom filter） + 可能一次 IO
- **读放大 = 层数**（典型 5–7 层）

**Compaction 策略是 LSM 的灵魂：**

| 策略 | 写放大 | 读放大 | 空间放大 | 适用 |
|---|---|---|---|---|
| **Leveled** | 高（~10× 每层，总 30–40×） | 低 | 低（~1.1×） | 读多写少，Cassandra/RocksDB 默认 |
| **Tiered / Size-tiered** | 低（~5×） | 高 | 高（最坏 2×） | 写密集，时序数据 |
| **FIFO / TTL** | ~1× | 中 | 低 | 纯时序，过期即删 |
| **Universal** | 中 | 中 | 中 | 折中 |

### 选型决策

| 你的场景 | 选 |
|---|---|
| 读写均衡，需要事务和 JOIN | **B+Tree**（Postgres / MySQL） |
| 写入量 >> 读，key-value | **LSM**（RocksDB / Cassandra） |
| 时序/日志，写入巨大且几乎不更新 | **LSM + Tiered/TTL** |
| 数据能全放内存 | **哈希/跳表**（Redis） |
| 分析查询，扫大量行取少数列 | **列存**（ClickHouse / DuckDB / Parquet） |

**面试常见追问：为什么 Cassandra 写这么快？**
> 因为写只落 MemTable + 顺序 WAL，没有随机 IO、没有读-改-写（read-modify-write）、没有 in-place 锁。代价是读要查多层 + compaction 持续消耗 IO 带宽和 CPU，且**空间放大和读延迟抖动（latency jitter）**（compaction 期间 p99 会毛刺）。

---

## 3. 三种放大，及它们在生产中的表现

```
写放大 WA = 实际写入设备的字节 / 应用逻辑写入的字节
读放大 RA = 实际读取的字节 / 应用需要的字节
空间放大 SA = 实际占用空间 / 逻辑数据大小
```

**为什么重要**：SSD 有写入寿命（TBW）。写放大 30× 意味着你的 SSD 寿命只有标称的 1/30。

**生产中的信号：**

| 症状 | 可能原因 |
|---|---|
| 写入 QPS 上不去，磁盘 util 100% 但吞吐低 | 随机写 / 写放大高 → 考虑批量、调页大小 |
| p99 周期性毛刺（每几十分钟一次） | LSM compaction / B-Tree checkpoint |
| 磁盘用量是数据量的 2 倍且持续增长 | LSM 空间放大 / 未回收的 MVCC 旧版本（PG 的 bloat） |
| 读延迟随数据量线性增长 | 布隆过滤器失效 / L0 文件堆积 |

**Postgres 特有的坑**：MVCC 旧版本靠 VACUUM 回收。高更新表如果 autovacuum 跟不上 → **表膨胀（bloat）**，可能 100 GB 的表实际占 400 GB，且索引扫描变慢。监控 `n_dead_tup` 和 `pg_stat_progress_vacuum`。

---

## 4. OLTP / OLAP / HTAP

| | OLTP | OLAP |
|---|---|---|
| 查询 | 点查（point query）、小范围，涉及少数行的多个列 | 扫描大量行的少数列 |
| 写入 | 高频小事务 | 批量导入 / 流式追加 |
| 存储 | **行存（row store）** | **列存（columnar store）** |
| 索引 | B+Tree 二级索引（secondary index） | 稀疏索引（sparse index） + 分区裁剪（partition pruning） + Zone Map |
| 并发 | 高（万级） | 低（十级），但每个查询很重 |
| 代表 | Postgres, MySQL, DynamoDB | ClickHouse, Snowflake, BigQuery, DuckDB |

### 列存为什么快 100×

```
行存：  [id=1, name="A", score=90][id=2, name="B", score=85]...
        查 AVG(score) 要读全部字段

列存：  id:    [1,2,3,4,...]
        name:  ["A","B","C",...]
        score: [90,85,72,...]     ← 只读这一列
```

三个叠加效应：
1. **IO 减少**：只读需要的列（100 列表只查 2 列 → IO 减少 50×）
2. **压缩率高**：同列同类型数据，字典编码（dictionary encoding）/RLE/Delta 编码 → 压缩 5–20×
3. **向量化执行（vectorized execution）**：SIMD 一次处理 1024 个值，且分支预测友好

**再加上分区裁剪和 Zone Map（每个数据块记 min/max），大部分数据块根本不用读。**

### HTAP：混合负载的三种做法

| 做法 | 说明 | 代表 |
|---|---|---|
| **单引擎双存储** | 行存主副本 + 列存副本，Raft 同步 | TiDB (TiFlash) |
| **CDC 到分析库** | OLTP → Debezium/CDC → 列存 | 最常见的生产做法 |
| **Lakehouse** | 统一存 Parquet/Iceberg，多引擎读 | Databricks, Snowflake |

**Senior 的判断**：
> 真正需要 HTAP 引擎的场景很少。大部分"实时分析"需求，用 CDC 到 ClickHouse 延迟 5–30 秒就够了，且运维简单得多、成本低得多。只有当业务需要「事务里读分析结果」（如实时风控评分参与事务决策）才需要真 HTAP。

---

## 5. 索引：超越"加个索引"

### 索引类型速查

| 类型 | 用途 | 注意 |
|---|---|---|
| B+Tree | 等值 + 范围 + 排序 | 默认选择 |
| Hash | 只等值 | PG 中很少用（不支持范围，且以前不写 WAL） |
| **联合索引（composite index）** | 多列过滤 | **最左前缀原则（leftmost prefix rule）**，列顺序 = 高选择性（selectivity）/等值在前 |
| **覆盖索引（covering index）** | 索引里就有所有需要的列 | 避免回表（table lookup），Postgres 的 `INCLUDE` |
| **部分索引（partial index）** | `WHERE deleted_at IS NULL` | 索引体积大幅下降 |
| **表达式索引（expression index）** | `ON (lower(email))` | 让函数查询能走索引 |
| GIN / 倒排（inverted index） | 全文、数组、JSONB | 写入慢 |
| GiST / R-Tree | 地理、范围类型 | |
| BRIN | 超大表 + 物理有序（时序） | 索引极小，适合按时间追加的表 |
| **HNSW / IVF** | 向量相似度 | 见下节 |

### 联合索引的列序规则

```sql
-- 查询：WHERE tenant_id = ? AND status = ? AND created_at > ? ORDER BY created_at
CREATE INDEX ON orders (tenant_id, status, created_at);
--                       ↑等值      ↑等值    ↑范围+排序
```
规则：**等值列（equality column）在前，范围/排序列在后**。范围列之后的列无法用于索引过滤。

### 索引的隐藏成本

- 每个索引让写入慢 5–15%
- 索引占空间（有时比表还大）
- **索引会阻碍 HOT update**（Postgres：更新了被索引的列就无法 HOT，产生更多膨胀）
- 太多索引让优化器选错计划

**经验值**：一张表超过 5–6 个索引就该审视了。用 `pg_stat_user_indexes.idx_scan = 0` 找出从没被用过的索引。

---

## 6. 向量索引：AI 时代的新构件

### 问题定义

给定 d 维向量 q，从 N 个向量中找最近的 k 个。精确搜索是 O(N·d)，1000 万 × 1536 维 = 每次查询 150 亿次浮点运算 → **不可行**。所以用 ANN（近似最近邻），用召回率（recall）换速度。

### 三大索引家族

**1. HNSW（分层可导航小世界图）** —— 目前的默认选择

```
Layer 2:   ●──────────────●            稀疏，长跳
           │              │
Layer 1:   ●────●────●────●            
           │    │    │    │
Layer 0:   ●─●─●─●─●─●─●─●─●─●─●      全部节点，密集连接
```
查询从顶层开始贪心走，逐层下降。

| | |
|---|---|
| 查询延迟 | **最低**（1M 向量约 1–5 ms） |
| 召回率 | 高（0.95–0.99 容易达到） |
| 内存 | **高**：每向量 ~(d×4 + M×2×4) 字节，M=16 时 1536 维约 6.2 KB → **1000 万向量 = 62 GB** |
| 构建 | 慢 |
| 增量插入 | 支持 |
| 删除 | **软删除（soft delete），需要重建才能真正回收** ← 生产大坑 |

关键参数：`M`（每节点连接数，16–64）、`ef_construction`（构建时搜索宽度，100–500）、`ef_search`（查询时宽度，**唯一的运行时召回/延迟旋钮**）。

**2. IVF（倒排文件）+ PQ（乘积量化）**

```
1. 用 k-means 把向量空间分成 nlist 个簇
2. 查询时只搜最近的 nprobe 个簇   ← 召回/速度旋钮
3. PQ：把 1536 维切成 96 段，每段 16 维用 256 个中心点表示 → 1 字节
   压缩：1536×4 = 6144 字节 → 96 字节，压缩 64×
```

| | |
|---|---|
| 内存 | **极低**（PQ 后 ~100 字节/向量 → 1 亿向量 10 GB） |
| 召回 | 中（PQ 有量化损失，需要 rerank 补救） |
| 适用 | **超大规模、内存受限** |

**标准做法**：IVF-PQ 粗筛（coarse search）top-200 → 用原始向量精确重排 → top-10。

**3. DiskANN / SPANN** —— 磁盘上的图索引，内存放压缩向量 + 图，全精度向量在 SSD。适合 10 亿级、成本敏感。

### 选型决策树

```
向量数 < 100 万，且已有 Postgres
  → pgvector (HNSW)。别引入新组件。

100 万 – 5000 万，延迟敏感，预算够
  → HNSW（Qdrant / Milvus / 托管服务）

> 1 亿，或内存成本是主要约束
  → IVF-PQ 或 DiskANN

需要强过滤（多租户 + 元数据条件）
  → 必须选支持"过滤感知搜索"的引擎，见下
```

### ⚠️ 生产中最容易踩的坑：过滤 + 向量搜索

```sql
-- 你想要：某个租户的、最近 30 天的、最相似的 10 篇文档
SELECT * FROM docs
WHERE tenant_id = 'acme' AND created_at > now() - interval '30 days'
ORDER BY embedding <=> $1 LIMIT 10;
```

三种执行方式，性能天差地别：

| 方式 | 做法 | 问题 |
|---|---|---|
| **Pre-filter** | 先过滤出候选集，再暴力算距离 | 候选集大时退化成全扫描 |
| **Post-filter** | 先 ANN 取 top-1000，再过滤 | **可能一个都不剩**（该租户的文档不在全局 top-1000 里） |
| **过滤感知搜索（filter-aware search）** | 在图遍历时就应用过滤 | 正确做法，但需要引擎支持 |

**多租户（multi-tenant）向量搜索的正解**：**按租户分区/分集合**，让每个租户有自己的索引段。这样过滤退化成"选哪个索引"，不影响 ANN 质量。代价是小租户很多时，索引碎片化（index fragmentation） —— 见 [`06-case-studies/03-multi-tenant-vector-search.md`](../06-case-studies/03-multi-tenant-vector-search.md)。

### 混合检索（Hybrid Search）—— 现在的事实标准

纯向量检索在**精确匹配（exact match）**上很差（产品型号 "X-4200"、人名、错误码）。生产系统都是：

```
        ┌─ BM25 / 全文检索 ──→ top-50 ─┐
Query ──┤                              ├─→ RRF 融合 ─→ 重排模型 ─→ top-10
        └─ 向量 ANN ────────→ top-50 ─┘
```

**RRF（Reciprocal Rank Fusion）**：`score(d) = Σ 1/(k + rank_i(d))`，k 通常取 60。
优点：不需要归一化（normalize）两种分数（它们量纲完全不同），只用排名，鲁棒。

详见 [`04-ai-agent-systems/02-context-engineering-and-rag.md`](../04-ai-agent-systems/02-context-engineering-and-rag.md)。

---

## 7. 对象存储：被低估的"数据库"

S3 / GCS / R2 现在的能力已经改变了架构：

| 特性 | 影响 |
|---|---|
| **强一致性（strong consistency）**（S3 2020 后） | 可以直接作为元数据存储的基础（Iceberg/Delta） |
| **条件写入（conditional write）**（If-None-Match，2024） | **可以在 S3 上实现无外部锁的原子提交（atomic commit）** → 湖仓表格式不再需要独立的 catalog 锁 |
| 单对象最大 5 TB | |
| 首字节延迟（first-byte latency） 20–100 ms | 不适合点查热路径（hot path），适合批量 |
| 成本 $0.023/GB/月 | 比块存储便宜 5–10× |
| 分层（IA / Glacier） | $0.004 / $0.001 per GB |

**现代模式：存算分离（storage-compute separation）**
```
计算层（无状态，可弹性伸缩） ──→ 本地 SSD 缓存 ──→ 对象存储（唯一真相源）
```
代表：Snowflake、ClickHouse Cloud、Neon、WarpStream（Kafka 协议直接写 S3）。

优势：存储成本降 10×、计算可缩到 0（scale to zero）、副本由 S3 保证。
代价：延迟高（靠缓存救）、S3 API 调用费（PUT $5/百万次 —— 小文件写会很贵，必须批量）。

---

## 8. 数据存储选型总表

| 需求 | 选择 | 理由 |
|---|---|---|
| 通用事务型主存储 | **Postgres** | 事务、JSONB、pgvector、扩展生态、你几乎不会后悔 |
| 超高写入 KV，可最终一致（eventual consistency） | Cassandra / ScyllaDB | LSM + 无主（leaderless） |
| 会话/缓存/排行榜/限流（rate limiting）计数 | Redis / Valkey | 内存 + 丰富数据结构 |
| 分析查询 | ClickHouse | 列存 + 向量化，性价比极高 |
| 全文检索（full-text search） | OpenSearch / Postgres FTS | 小规模用 PG 就够 |
| 向量检索 | pgvector → Qdrant/Milvus | 按规模升级 |
| 时序指标 | Prometheus / VictoriaMetrics / TimescaleDB | |
| 事件流 | Kafka / Redpanda / WarpStream | |
| 大对象 / 数据湖 | S3 + Iceberg | |
| 图关系（如权限） | Postgres 递归 CTE → 专用（Neo4j/SpiceDB） | 大多数"图"需求 PG 能搞定 |

**默认建议**：**从 Postgres 开始，直到它明确扛不住某个具体维度**。它现在能做 KV、JSON 文档、全文、向量、时序（TimescaleDB）、队列（SKIP LOCKED）、地理。一个数据库的运维成本远低于五个。

---

## 面试官会追问

1. 为什么 Cassandra 写快读慢？compaction 会带来什么生产问题？
2. Postgres 表膨胀是怎么回事？怎么发现和处理？
3. 联合索引 (a,b,c)，查询 `WHERE a=? AND c=?` 能用上多少？
4. 列存为什么比行存快，具体是哪三个机制？
5. HNSW 1000 万条 1536 维要多少内存？删除怎么办？
6. 向量搜索加上 `WHERE tenant_id=?` 过滤，为什么可能返回不了结果？
7. 什么时候你会放弃 Postgres？具体的临界指标是什么？

---

**下一篇** → [02-caching.md](02-caching.md)
