# 01 · LLM 推理基础设施

> 推理优化的杠杆顺序是：**不重算 > 换拓扑 > 换精度 > 换卡**。2026 年同一批 GB300 上，纯软件在半年内做出 2.7× 吞吐（[MLPerf Inference v6.0](https://mlcommons.org/2026/04/mlperf-inference-v6-0-results/)）——选框架和拓扑的收益已经大于选硬件。
> 还有一条：**只写"P95 TTFT < 2s"而不写 TPOT 的 SLO，等于没有 SLO。**

---

## 1. 两个阶段，本质上是两台不同的机器

Transformer 自回归推理分成两段，它们的硬件特征相反：

| | **Prefill**（处理输入） | **Decode**（逐 token 生成） |
|---|---|---|
| 一次前向处理的 token 数 | 整个 prompt（数千） | 每个序列 1 个 |
| 瓶颈 | **算力**（compute-bound） | **访存带宽**（memory-bound） |
| 算术强度（FLOP/byte） | ≈ 2 × prompt 长度 → 数千 | ≈ 2 × batch size |
| 典型 MFU | 35–55% | **1–10%**（小批时） |
| 延迟指标 | TTFT | TPOT / ITL |
| 加卡的收益 | 近线性 | 需要**加批**才有收益 |

**为什么 decode 这么惨。** 每生成一个 token，都要把**整个模型权重**从 HBM 读一遍。H100 SXM 的算力/带宽比（ridge point）约 **300 FLOP/byte**（BF16 口径，990 TFLOPS ÷ 3.35 TB/s，量级）。decode 的算术强度 ≈ 2×batch，所以：

```
batch = 1    → 强度 2      → 用掉不到 1% 的算力，GPU 在等内存
batch = 32   → 强度 64     → 仍在访存墙下
batch ≈ 150  → 强度 300    → 才刚够到 roofline 拐点
```

> **面试金句**：
> "decode 阶段的 GPU 不是在算，是在搬。所以 decode 的唯一优化方向是**把 batch 做大**——把权重那一次读摊给更多序列。而 batch 做大又要求 KV cache 装得下，于是 decode 的实际上限是**显存**，不是算力。这就是为什么 KV cache 是整个推理系统的中心。"

**结论 1**：同一张卡不可能同时对两个阶段最优。prefill 想要小批、快进快出；decode 想要极大批。这是 PD 分离（§5）的全部动机。

**结论 2**：**prefill-only 的 tok/s 和混合口径的 tok/s 差一个数量级**。同一份 DeepSeek-R1 在 GB300 上 prefill 报 22,476 TGS，混合负载（ISL=2k/OSL=1k）报约 3,072 TGS——差 7×。看到没有标注 ISL/OSL/并发/量化/版本的 tok/s，直接判为不可用。

---

## 2. KV cache：显存预算的主项（公式 + 算例）

### 公式

```
每 token 每层 KV 字节数 = 2 (K和V) × n_kv_heads × head_dim × bytes_per_elem
每 token 总字节数       = 上式 × n_layers

单请求 KV 占用 = 每 token 字节数 × (ISL + OSL)

KV 显存预算 = N_gpu × HBM_per_gpu − 权重 − 激活/CUDA graph/框架开销
最大并发    = KV 显存预算 ÷ 单请求 KV 占用
```

框架开销留 8–15 GB/卡是安全值（vLLM 的 `--gpu-memory-utilization` 默认 0.9 就是在做这件事）。

### 算例：Llama-3.1-70B，FP8 权重 + FP8 KV，2× H100

- 结构：`n_layers=80, n_kv_heads=8 (GQA), head_dim=128`
- 每 token 每层 = 2 × 8 × 128 × 1 byte = **2 KiB**
- 每 token 总计 = 2 KiB × 80 = **160 KiB/token**（BF16 KV 则为 320 KiB）
- 单请求（ISL 4,000 + OSL 500 = 4,500 token）= 4,500 × 160 KiB ≈ **0.70 GiB**

显存账：

```
2 × 80 GB HBM              = 160 GB
− 权重 (70B × 1 byte)      =  70 GB
− 激活/graph/框架 (~6/卡)  =  12 GB
────────────────────────────────────
KV 预算                    ≈  78 GB
最大并发 = 78 / 0.70       ≈  111 个请求
```

**同一台机器，KV 精度从 FP8 换成 BF16，并发直接砍半到 55。** 这是 FP8 KV cache 在 2026 被 vLLM 定调为"长上下文部署默认起点"的原因（[FP8 KV-Cache 现状](https://vllm.ai/blog/2026-04-22-fp8-kvcache)：长上下文 AUC 恢复 94–98%，输出吞吐 +5–15%）。

### 注意力结构决定一切

| 结构 | 代表 | 每 token KV（量级，BF16） | 相对 MHA |
|---|---|---|---|
| MHA | Llama-2-70B（64 KV heads） | ~2.5 MB | 1× |
| **GQA** | Llama-3.1-70B（8 KV heads） | ~320 KB | **1/8** |
| MQA | 早期 PaLM（1 KV head） | ~40 KB | 1/64 |
| **MLA**（低秩潜变量） | DeepSeek-V3/R1（671B） | **~70 KB** | 671B 的模型比 70B 的还省 |

那条流传的"KV cache 约 1 MB/token"经验值（[PROMPTPEEK, NDSS 2025](https://www.ndss-symposium.org/wp-content/uploads/2025-1772-paper.pdf) 口径）反映的是 MHA 时代。**2026 年请自己按上面的公式算，模型之间能差 30 倍。**

⚠️ **撞墙信号**：`vllm:num_preemptions_total` 持续 > 0，或 `gpu_cache_usage_perc` 长期贴 100%。这意味着调度器在抢占并**重算**已生成的请求——吞吐会阶跃式塌陷，而 GPU 利用率图上看起来还很"忙"。

---

## 3. 分页、连续批处理、chunked prefill

**PagedAttention**（[arXiv 2309.06180](https://arxiv.org/abs/2309.06180)）：把 KV cache 按固定块（典型 16 token）管理，用块表做逻辑→物理映射，像 OS 的虚拟内存。收益是消灭外部碎片，内部碎片 < 4%（对比预分配 max_len 的 60–80% 浪费），并让 beam/并行采样零拷贝共享前缀。

⚠️ **不要误读 release note**：[vLLM v0.25.0](https://github.com/vllm-project/vllm/releases/tag/v0.25.0) 写了 "PagedAttention has been removed"——删掉的是 **V0 时代的 legacy attention kernel 路径**，分页块管理本身仍是 V1/MRv2 的地基。这是 2026 年最常见的一条读错。

**连续批处理（continuous batching）**：请求一完成就把槽位让出来，新请求立刻插入正在跑的 batch，而不是等整批跑完。这是 2–10× 吞吐提升的来源，今天是所有引擎的默认行为。

**Chunked prefill**：把长 prompt 的 prefill 切成小块，塞进 decode 的 batch 里一起跑，避免一个 8k prompt 把 decode 池"卡住"数百毫秒。vLLM V1 默认开启。

```bash
vllm serve meta-llama/Llama-3.1-70B-Instruct \
  --tensor-parallel-size 2 \
  --quantization fp8 --kv-cache-dtype fp8_e4m3 \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.92 \
  --max-num-batched-tokens 8192 \   # chunked prefill 的每步 token 预算
  --max-num-seqs 256 \
  --enable-prefix-caching
```

**代价必须说清楚**：chunked prefill 用**同批 decode 请求的 TPOT** 换整体吞吐。`--max-num-batched-tokens` 调大 → TTFT 好、TPOT 差；调小 → 反之。**当 TTFT 和 TPOT 都是硬 SLO 时，调参解决不了，得上 PD 分离。**

---

## 4. Goodput：唯一能用来做容量规划的指标

三个定义（[CNCF: Why goodput matters more than throughput](https://www.cncf.io/blog/2026/07/20/why-goodput-matters-more-than-throughput-for-llm-serving/)）：

- **TTFT** = 首 token 到达时间（含网络 + 排队 + prefill）
- **TPOT / ITL** = 首 token 之后的平均产出间隔
- **goodput** = 每秒完成、**且同时满足 TTFT 与 TPOT SLO** 的请求数

throughput 会一路涨到 GPU 打满，goodput 会在某个批大小之后**崩到 0**：

```
  聚合吞吐 tok/s                        单请求延迟
  高│              ╭──────────── 饱和   高│                      ╱ TTFT（排队主导）
    │           ╭──╯                      │                    ╱
    │        ╭──╯                         │                 ╱
    │      ╭─╯                            │        ╱────╱  TPOT（带宽争抢）
    │    ╭─╯                              │  ╱────╯
  低│  ╭─╯                              低│──
    └──┴────┴────┴────┴────┴──→          └──┴────┴────┴────┴────┴──→
       1   16   64  256 1024  batch          1   16   64  256 1024
                                                  ├── goodput 窗口 ──┤
                                                  两条 SLO 同时成立的区间

  goodput = f(batch)：      ╭───╮
                          ╭─╯   ╰──╮
                      ╭───╯         ╰────────  → 0（吞吐仍在涨，但全部超时）
```

**这条曲线的实操含义**：容量规划必须在 goodput 峰值的**左侧**取工作点，留出 20–30% 余量应对流量抖动。跑在峰值右侧的系统，一次小流量尖峰就会让全部请求同时违约。

**真实反例**：同样满足 TTFT ≤ 1.5s 的两个配置，P95 TPOT 一个约 50 ms、另一个约 494 ms——**10× 差异，单指标 SLO 完全看不出来**。

测量方法论抄 CNCF 那条 **windowing rule**：要求**连续 8 个稳定采样点**才计分，避免用瞬时峰值定容量。

---

## 5. PD 分离（Prefill/Decode Disaggregation）

把 prefill 和 decode 放到**不同的 GPU 池**，中间通过 RDMA 传 KV cache。

```
                    ┌──────────────────────────────────┐
   请求 ──► 路由器 ─┤ Prefill 池                        │
              │     │  TP2 × N 副本                     │
              │     │  max-num-seqs: 1  ← 故意不做大批   │
              │     │  目标：最小化 TTFT                 │
              │     └──────────────┬───────────────────┘
              │                    │ KV cache 传输
              │                    │ (UCX/NIXL over IB/RoCE)
              │                    ▼
              │     ┌──────────────────────────────────┐
              └────►│ Decode 池                         │
                    │  TP4 × M 副本                     │
                    │  max-num-seqs: 1024 ← 极大批       │
                    │  目标：最小化 $/token，稳住 TPOT    │
                    └──────────────────────────────────┘
```

（形状参考 [NVIDIA Dynamo 分离式服务文档](https://docs.nvidia.com/dynamo/user-guides/disaggregated-serving) 的官方示例。）

### 硬性前置条件：RDMA

Dynamo 走 UCX/NIXL，要求 InfiniBand 或 RoCE，并且需要：

```yaml
env:
  - name: UCX_RNDV_SCHEME
    value: get_zcopy          # 零拷贝 RDMA
securityContext:
  capabilities: { add: ["IPC_LOCK"] }
resources:
  limits:
    rdma/ib: 1                # 按 TP size 申请
```

**判据（背下来）**：

```
KV_bytes / RDMA_带宽   vs   该请求的 decode 时长
        前者更大  ⇒  不要分离
```

量级参考：70B FP8 / 8k token ≈ **1.3 GB/请求**；671B / 100k 上下文 ≈ 数十 GB。1.3 GB 走 400 Gb/s RDMA 约 26 ms，而 decode 500 token × 25 ms = 12.5 s——比值 0.2%，分离显然划算。同样 1.3 GB 回落到 10 GbE TCP 则是 **1 秒以上**，直接吃掉全部 TTFT 预算。

### 什么时候不要 PD 分离

NVIDIA 官方文档罕见地明确反驳了自家方案："**It is not automatically better for every workload. For small models, short prompts, low concurrency, or clusters without fast KV transfer, an aggregated deployment may be simpler and faster.**"

| 不要分的信号 | 原因 |
|---|---|
| 没有 IB/RoCE | KV 传输时间盖过 decode 时间 |
| 模型 < 30B | 权重读得快，decode 本来就不惨 |
| prompt < 512 token | prefill 根本不是瓶颈 |
| 并发 < 50 | decode 池凑不出大批，分离白付传输成本 |
| 只有 1–2 台机器 | P:D 比例无法调，退化成静态浪费 |

另有一派主张统一内存路线（Tenstorrent 声称片内 SRAM 带宽足够大就完全不需要 KV 传输）——**这是 2026 年一个真实的未决争论**，不要当成已有定论。

### 生产模板（可直接抄的 SLA 形状）

[vLLM 上 GLM-5.2 / 24× B300 的 PD 分离部署](https://vllm.ai/blog/2026-07-23-glm-5.2-nvfp4-b300-pd)：744B MoE（40B 激活）NVFP4，**4 个 prefill 节点（TP1 DP4 EP）+ 1 个 decode 节点（TP1 DP8 EP）**，SLA 定为 mean TTFT ≤ 2.5 s、mean TPOT ≤ 20 ms，实测 TPOT 17 ms（基线约 40 ms）。注意 P:D = 4:1 是**从这份负载的 ISL/OSL 推出来的，不是通用常数**。

分离的维度在 2026 年还在扩展：Encoder 分离（EPD）、Hybrid SSM 分离、[Attention/FFN 分离（AFD，实验性）](https://vllm.ai/blog/2026-07-23-vllm-afd-plugin)。判据永远是同一条：**这两段的算力/带宽/显存 profile 是否显著不同 + 中间态传输是否便宜**。

---

## 6. 前缀缓存与缓存感知路由（网关设计的真正约束）

### RadixAttention

[SGLang](https://arxiv.org/abs/2312.07104) 把所有请求的 KV 块组织成一棵**基数树（radix tree）**，新请求沿树匹配最长公共前缀，命中部分直接跳过 prefill。淘汰按 LRU 在树上做。vLLM 的 APC（Automatic Prefix Caching）是同类机制，V1 默认开启。

**收益（一手实测区间）**：共享系统提示场景命中率 70–90%，聚合吞吐 +30–50%，某租户 TTFT 480 ms → 110 ms。

### 但命中率是 workload 属性，不是框架能力

同一套 vLLM，另一个租户命中率 **0.3%**，完全没有改善——因为它每次动态拼 prompt。更要命的是 **agentic 多轮场景**：工具输出和中间推理插在上下文中间，把共享前缀打碎，stock prefix cache 命中率可跌破 20%。

另有实测报告开启 prefix caching 后吞吐 **−36.7%**、TPOT **+25.0%**（前缀重叠率低时，哈希与查表纯属开销）。**这两组结论可能都对，取决于前缀重叠率。上线前必须先量出重叠率，不要按框架 feature 承诺收益。**

### 缓存感知路由：比缓存本身更值钱

有了缓存还不够——请求必须被路由到**已经持有那份 KV 的实例**上。[llm-d 的对照实验](https://llm-d.ai/blog/kvcache-wins-you-can-see)（8 pod / 16× H100，Qwen-32B，150 租户 × 6k 上下文，KV 需求占集群 73%）：

| 路由策略 | P90 TTFT | 输出吞吐 |
|---|---|---|
| **precise**（精确 KV 感知） | **0.542 s** | 8,730 tok/s |
| approximate（近似前缀哈希） | 31.083 s | 6,944 tok/s |
| random（轮询） | **92.551 s** | 4,428 tok/s |

**57× / 170×。这是本篇最大的单个数字。**

K8s 侧的事实标准入口是已 GA 的 [Gateway API Inference Extension (GAIE)](https://github.com/kubernetes-sigs/gateway-api-inference-extension)，由 EPP（external processing pod）对候选 pod 打分。

**这对网关架构的约束是硬的**：

1. LLM 网关**必须是有状态路由器**，不能是无状态 L7 LB。它要维护"哪个实例持有哪些前缀"的视图。
2. 缓存亲和性与负载均衡**直接冲突**。纯亲和会让热租户压垮单个 pod；纯均衡会让命中率归零。生产做法是**打分函数**：`score = w1 × prefix_match_len − w2 × queue_depth − w3 × kv_usage`，`w1` 权重最大但不是无穷。
3. 实例扩缩容会让缓存视图失效，**扩容后短期内 TTFT 会变差**——这与"扩容改善延迟"的直觉相反，必须写进 runbook。

### 与安全的正面冲突（必须显式写出）

同一份共享 KV cache 是**最大的跨租户泄露面**。PROMPTPEEK（NDSS 2025）利用 KV 共享导致的服务顺序/时序侧信道，逐 token 重建他人 prompt：已知模板时成功率 **99%**，反推模板本身 98%，**无任何背景知识 95%**；已知模板时约 **60 次请求**就能套出受害者的性别/年龄/体重/身高。攻击面覆盖 vLLM、SGLang、LightLLM、DeepSpeed，ByteDance 已确认该侧信道。

> **面试金句**：
> "前缀缓存是我手里收益最大的一个旋钮，也是唯一一个开了就有跨租户泄露风险的旋钮。我的默认策略是**同租户内共享、跨租户默认关闭**——代价是多租户场景下最大的性能杠杆被削掉一半。要拿回来，只能在**共享的那部分是我自己写死的系统提示、不含任何租户数据**这个前提下开白名单。密钥和 PII 绝不进可缓存前缀。"

---

## 7. KV cache 分层卸载

链路（[LMCache](https://github.com/lmcache/lmcache) 的形态）：

```
GPU HBM  →  CPU DRAM  →  本地 NVMe SSD  →  远端（Redis / S3 / Mooncake）
 ~3 TB/s     ~83 GB/s       ~5 GB/s          ~1–10 GB/s
             (PCIe DMA)
```

**收益**（[vLLM KV offloading connector](https://vllm.ai/blog/2026-01-08-kv-offloading-connector)，v0.12.0）：单请求 TTFT 改善 2×–22×，并发吞吐最高 9×（在 0–80% CPU 命中率区间）。关键工程细节是把 KV block 粒度从 KB 级放大到 **0.5–2 MB**——小块的 DMA 发起开销会吃掉全部收益。

**判据**：

```
从 CPU 取回的时间  =  KV_bytes / 83 GB/s
重算的时间        =  prefix_tokens / prefill_吞吐(tok/s)
                     取小的那个
```

70B 模型 8k 前缀：取回 ≈ 1.3 GB / 83 GB/s ≈ **16 ms**；重算 ≈ 8,000 / 5,600 ≈ **1.4 s**。取回赢 90×。反过来，**几百 token 的短前缀直接重算更快**——所以卸载要设最小块阈值。

⚠️ **成熟度警告**：vLLM 原生 KV offload 从 v0.11.0 实验引入 → v0.12.0 重写 → GA 目标 v0.14.0 → v0.26.0 才写 "matured"，且一手博客只覆盖 CPU DRAM 层，NVMe/远端为后续。跨实例复用的 Mooncake 集成在 2026-05 时点**仍是未合入主干的 PR**，其 [1.7% → 92.2% 命中率、P50 TTFT ↓46×](https://vllm.ai/blog/2026-05-06-mooncake-store) 的数字是原型口径。**上生产前逐项核对 release note，别信汇总博客。**

---

## 8. 投机解码：一个会随并发反转符号的优化

用小 draft 模型（或 MTP 头）一次猜 K 个 token，target 模型一次前向并行验证，接受前缀。

```
理论上限加速比 ≈ AL / (1 + K × C_draft/C_target)
AL (accept length) = 每个 target 前向步平均接受的 token 数
```

**AL 是强领域依赖的**（EAGLE-3 实测 11 个领域）：均值 2.77，coding 3.16 / math 3.12 / RAG 3.11 / 多语言 3.07，**开放式写作只有 2.0–2.9**。

**最反直觉的一条——加速比随并发衰减**：

| batch / 并发 | 加速比 |
|---|---|
| 1 | 1.69× |
| 8 | ~1.4× |
| 48 | **0.99×（变慢）** |
| 64 | 1.05× |
| 128（EAGLE on Llama-3-70B） | 1.21× |

原因是结构性的：**高并发下 GPU 已经 compute-bound，验证的额外算力不再"免费"**。而绝大部分公开评测在 batch=1 下做，与真实服务（decode 池上千 seq）完全不是一个 regime。

**工程结论**：
- **PD 分离后 decode 池天然是大批 ⇒ decode 池默认不开投机解码。** GLM-5.2 提 TPOT 用的是 speculative padding（40→22 ms），不是纯 spec decode。
- 正确形态是**按队列深度动态开关 + 动态调 draft 树深度**。2026 SOTA 是置信度调度（SGLang DSpark：DeepSeek-V4-Pro / B300 / TP8 单用户 383.7 tok/s，AL ≈ 5）。
- 低并发的**内部/单租户/交互式**场景（IDE 补全、单人 agent）是投机解码的正确战场。

---

## 9. 量化：权重和 KV cache 要分开决策

| 维度 | 结论 | 数字 |
|---|---|---|
| **格式** | NVFP4 = group 16 + 完整 FP8 E4M3 scale；MXFP4 = group 32 + E8M0 scale ⇒ NVFP4 精度更好是结构性的 | — |
| **权重量化（大模型）** | 安全 | DeepSeek-R1 671B FP8→NVFP4：MMLU 90.8 → 90.7（**−0.1 pt**） |
| **权重量化（小模型）** | 危险 | Qwen3-8B BF16→NVFP4：MMLU **−2.2%**、GSM8K −1.9% |
| **吞吐收益** | NVFP4 prefill 相对 FP8 **1.8×**（DeepSeek-V3.2 / GB300 / TP2） | |
| **KV cache 量化** | FP8 e4m3 已是长上下文默认起点 | 长上下文 AUC 恢复 94–98%；输出吞吐 +5–15%；decode 延迟斜率降到 BF16 的 54% |
| **硬件对应** | Ampere/Ada（A100/L40S/4090）→ AWQ INT4；Hopper → FP8；Blackwell → NVFP4 | AWQ 相对 GPTQ 约 +1–2 分且 kernel 更快 |

**唯一需要背的规则**：**参数量越小，每个权重承载的信息越密，量化损失越大。≤10B 的模型不要上 FP4。** 把 671B 的 −0.1 pt 和 8B 的 −2.2 pt 放在同一句话里做论据，是常见的错误引用。

---

## 10. 并行策略与 MoE 服务化

| 并行方式 | 切什么 | 通信 | 何时选 |
|---|---|---|---|
| **TP**（张量并行） | 每层的矩阵按列/行切 | 每层 2 次 all-reduce，**极重** | 单卡放不下；**必须在 NVLink 域内**，跨节点 TP 是灾难 |
| **PP**（流水并行） | 按层切到不同节点 | 层边界 P2P，**轻** | 跨节点扩展、模型极大；代价是流水气泡，**decode 阶段气泡尤其难填** |
| **EP**（专家并行） | MoE 的专家切到不同卡 | all-to-all dispatch/combine | MoE 模型的默认；专家数 ≥ 卡数时才划算 |
| **SP**（序列并行） | 序列维切 | ring/all-gather | 超长上下文 prefill；配合 TP 省激活显存 |
| **DP**（数据并行/多副本） | 整模型多份 | 无 | 放得下就优先——**没有通信就是最好的通信** |

**决策顺序**：能 DP 就 DP → 放不下就在单节点内 TP → 还放不下才跨节点 PP 或 EP。

### MoE 的头号瓶颈是 all-to-all

[DeepEP](https://github.com/deepseek-ai/DeepEP) 的带宽表说明了物理上限：

```
机内 NVLink (SM100, 64 SM)：  dispatch 726 GB/s / combine 740 GB/s
跨节点 RDMA (SM90+CX7, 12 SM)： dispatch  90 GB/s / combine  81 GB/s
                                        └── 差约 8× ──┘
```

**wide-EP 的天花板就在这里。** 跨节点扩专家数是"用 8× 慢的链路换更大的模型容量"，必须靠计算/通信重叠（Two-Batch Overlap，prefill +27–35%）和专家负载均衡撑住。

**EPLB（专家负载均衡）不是锦上添花**：单独贡献 prefill **1.49×** / decode **2.54×**（[LMSYS 96× H100 DeepSeek 实测](https://www.lmsys.org/blog/2025-05-05-large-scale-ep/)）。专家负载天然倾斜——没有均衡的 wide-EP 会被最热的那几个专家拖死，表现为"大部分卡在等，少数卡 100%"。

⚠️ 弹性专家并行（vLLM Elastic EP）2026 年仍是 **beta，且只支持 `tensor_parallel_size=1` + 单 API server + Ray DP backend**，官方博客没给任何扩缩容耗时数字。别写进架构图当既成事实。

---

## 11. GPU 多租户与调度

| 共享方式 | 显存隔离 | 故障隔离 | 适用 |
|---|---|---|---|
| **整卡独占** | ✅ | ✅ | 跨租户默认答案 |
| **MIG** | ✅ 硬件级 | ✅ | 跨租户；但仅限支持的卡，**profile 必须静态规划**，碎片浪费 10–30% |
| **MPS** | ❌ | ❌ 单 client 崩溃影响同卡其他 client | **仅同租户内**的可信 CUDA 负载 |
| **time-slicing** | ❌ | ❌ | 开发/测试环境；**绝不跨租户** |

**跨租户只有 MIG 或整卡。** 把 MPS 当隔离手段是 2026 年仍然常见的严重错误。

K8s 侧：**DRA（Dynamic Resource Allocation）核心已在 [Kubernetes v1.34（2025-09）GA 并默认开启](https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/)**，正在取代 device-plugin；NVIDIA 已把 DRA Driver for GPUs 捐给 CNCF（它负责把通用 DRA 请求翻译成 MIG 分区、NVLink 拓扑、time-slicing、MPS 和 GB200/GB300 的 ComputeDomains）。注意 GA 的只有核心 API，`DRAConsumableCapacity` 等子特性仍在 beta/alpha —— 落地前按你要用的具体子特性查当版 release notes。

**公平性的正确单位是 token 而不是请求。** 一个 100k 上下文的请求消耗的资源是 500-token 请求的 200 倍。限流器必须按 `input_tokens + α × output_tokens` 扣配额（α 取 3–10，反映 decode 更贵）。按 QPS 限流的多租户网关一定会被长上下文租户打穿。

⚠️ **未找到公开数据**：vLLM / SGLang / Dynamo 在 2026 年**均未提供请求级抢占的公开 SLO 保障指标**。想做"高优请求插队"，目前只能在网关层做队列，不能指望引擎。实用形态是**分池**：给高优流量单独的实例池，用容量而非调度器保证优先级。

---

## 12. 容量规划：从 QPS 与 token 分布推 GPU 数

**输入**（这些必须先问出来，问不出来就没法算）：

```
峰值 QPS        = 10
ISL (p50/p95)   = 4,000 / 12,000 token
OSL (p50/p95)   = 500 / 2,000 token
SLO             : TTFT p95 ≤ 2 s, TPOT p95 ≤ 30 ms
模型            : 70B dense, FP8 权重 + FP8 KV, TP2 on H100
```

**第一步：decode 侧并发（Little's Law）**

```
单请求驻留时间 = TTFT + OSL × TPOT = 2 s + 500 × 0.03 s = 17 s
稳态并发 L = λ × W = 10 × 17 = 170 个请求
```

**第二步：decode 侧受什么约束**

```
KV 约束：  每副本 KV 预算 78 GB ÷ 0.70 GB/请求 = 111 并发/副本
           170 / 111 → 2 个 decode 副本（TP2）= 4 张 H100

TPOT 校验：每步每卡访存 = 35 GB(权重/TP2) + 85 × 0.35 GB(KV/TP2) ≈ 65 GB
           65 GB / 3.35 TB/s ≈ 19.4 ms  +  kernel/all-reduce 开销 ≈ 25 ms  ✅ < 30 ms
```

**第三步：prefill 侧受算力约束**

```
需要的 prefill 速率 = 10 QPS × 4,000 token = 40,000 tok/s
每卡 prefill 吞吐  = H100 FP8 峰值 1,979 TFLOPS × 40% MFU ÷ (2 × 70e9 FLOP/token)
                   ≈ 5,600 tok/s
每 TP2 副本（85% 扩展效率） ≈ 9,600 tok/s
40,000 / 9,600 → 5 个 prefill 副本 = 10 张 H100
```

**第四步：合账**

```
prefill 10 卡 + decode 4 卡 = 14 卡
+ 冗余/滚动升级/长尾（+20~30%）→ 16–18 卡（2 个 8 卡节点）
P:D 副本比 ≈ 5:2
```

**这里出现了本篇最重要的推论**：P:D 比是 **5:2**。如果不做 PD 分离而是聚合部署，你必须按**两个瓶颈的最大值**配卡——prefill 需要 10 卡的算力、decode 需要 4 卡的显存，聚合后两边互相干扰，实际要 20 卡以上才能同时满足两条 SLO。**PD 分离省下的不是算力，是"为了满足另一个指标而多买的卡"。**

### 扩缩容的现实：GPU 不是弹性资源

| 环节 | 耗时（量级） |
|---|---|
| 云厂商分配 GPU 实例 | 30 s – 数分钟（H100 常态缺货，可能失败） |
| 拉容器镜像（含 CUDA/引擎，10–30 GB） | 1–5 分钟 |
| 加载模型权重到 HBM（70B FP8 = 70 GB） | 30 s – 3 分钟 |
| CUDA graph 捕获 / kernel 自动调优 | 30 s – 2 分钟 |
| **前缀缓存预热到稳态命中率** | **数分钟到数十分钟** |
| **合计冷启动** | **5–15 分钟** |

**HPA 那套"CPU > 70% 就扩容"在 GPU 上是不成立的。** 正确形态：

1. **预热池（warm pool）**：常驻 15–25% 的空闲容量，接受这笔成本。
2. **排队而非弹性**：过载时进入准入队列 + 明确的 429 与 `Retry-After`，而不是指望扩容救场。
3. **按预测扩容**：用 T-15min 的流量趋势和日历（工作日 09:00 尖峰、批处理窗口）提前拉起。
4. **缩容要慢**：缩容窗口 ≥ 30 分钟，否则缓存反复冷启动，你会看到"缩容后延迟变差"。
5. 溢出到托管 API 作为兜底通道——这是自建 + API 混合部署的主要理由之一。

---

## 13. 自建 vs API：盈亏平衡怎么算

```
API_$/hr   = QPS × 3600 × (ISL × P_in + OSL × P_out) / 1e6
自建_$/hr  = N_gpu × P_gpu × k_hidden        # k_hidden = 3–5

盈亏平衡 QPS = N_gpu × P_gpu × k_hidden ÷ [ 3600 × (ISL×P_in + OSL×P_out)/1e6 ]
```

`k_hidden` 是关键：**总成本约为裸 GPU 租金的 3–5×**（网络、存储、K8s 控制面、监控、以及 $5,000–15,000/月的专职工程人力）。忽略它是自建成本模型最常见的谎言。

**算例**（沿用 §12 的 16 卡配置，2026 年中量级，随时变动）：

```
自建：16 × $2.50/hr(H100 现货) × 3 = $120/hr
API （对标 Claude Haiku 4.5：$1/M 输入、$5/M 输出）：
     每请求 = (4,000×1 + 500×5)/1e6 = $0.0065
     每 QPS-小时 = 0.0065 × 3600 = $23.4

盈亏平衡 = 120 / 23.4 ≈ 5.1 QPS，7×24 稳定负载
```

**读法**：需要**全天候** 5 QPS（≈ 44 万请求/天）才打平。如果流量是白天 10 QPS、夜里 0（日均利用率 40%），实际需要峰值约 13 QPS 才打平。**利用率从 100% 掉到 20%，单位成本涨 5 倍**——这才是自建的真实风险。

⚠️ **这个结论有真实的公开分歧（写进你的答案里）**：一派算出 70B 单 H100 约 400 tok/s、对标 $5/M 的旗舰模型约 12M token/天回本；另一派实测 8×H100 跑 671B MoE 满载约 **$10/M 输出**，比廉价开源托管 API（约 $0.87/M）**贵 11 倍**。**两边都没算错，是对标基线不同**（溢价旗舰 vs 廉价开源托管）。引用任何自建 ROI 之前，先钉死你对标的是哪个模型的哪个档位。

**自建的正当理由通常不是省钱**：数据驻留/合规、延迟确定性（不受厂商限流影响）、模型定制（微调/LoRA 热插）、供应商中断的兜底。**如果理由只有"省钱"，先把利用率曲线画出来再说。**

---

## 14. 优化手段 → 收益 → 代价 总表

| 手段 | 典型收益 | 代价 / 撞墙条件 |
|---|---|---|
| 连续批处理 | 吞吐 2–10× | 默认必开；批内超长请求拖尾 |
| 分页 KV（PagedAttention） | 碎片 <4%，并发 2–4× | 块表索引开销、kernel 复杂度 |
| Chunked prefill | 混合负载吞吐提升，TTFT 毛刺消失 | **拉长同批 decode 的 TPOT**；TPOT 是硬 SLO 时需限 chunk 或改分离 |
| 前缀缓存 | 命中 70–90% 时 TTFT 降数倍；极端场景 46× | 命中率是 workload 属性；低重叠下净吞吐可 **−36.7%**；**跨租户共享 = 泄露面** |
| 缓存感知路由 | P90 TTFT **57–170×** | 路由器变有状态；与负载均衡冲突；扩容后短期劣化 |
| KV 卸载到 CPU | TTFT 2–22×，吞吐 ≤9× | PCIe 83 GB/s 是上限；短前缀不如重算；GA 成熟度存疑 |
| PD 分离 | **同时**满足 TTFT + TPOT；省下"为另一指标多买的卡" | **必须 RDMA**；小模型/短 prompt/低并发下更慢 |
| FP8 权重 | 显存 ÷2，吞吐显著提升 | 需要 Hopper+ |
| NVFP4 权重 | prefill 1.8× vs FP8 | 需要 Blackwell；**≤10B 模型掉约 2 分** |
| FP8 KV cache | KV 显存 ÷2（并发 ×2），输出吞吐 +5–15% | 长上下文 AUC 恢复 94–98%，推理任务最多掉 1–2 分 |
| 投机解码 | bs=1 时 1.7–2× | **bs=48 时 0.99×（变慢）**；decode 池默认不开 |
| wide-EP + EPLB | prefill 1.49× / decode 2.54× | 跨节点 RDMA 比 NVLink 慢 8×；专家倾斜会拖死集群 |
| MIG | 硬件级隔离 | profile 静态规划，碎片浪费 10–30% |
| 预热池 | 消除 5–15 分钟冷启动暴露 | 常驻 15–25% 闲置成本 |

---

## 15. 什么时候不要做这些（反模式）

1. **QPS < 1 就自建推理栈**。你付的是工程人力，不是 GPU 钱。托管 API + prompt caching + Batch（50% off）的组合在这个规模上不可战胜。
2. **没有 RDMA 就上 PD 分离**。KV 传输回落到 TCP 时会主导端到端延迟——这是最常见的失效模式。
3. **把 prefix caching 当"开了就有"的能力**。先量前缀重叠率。agentic 负载不做 prompt 布局治理（静态部分严格前置、逐字节稳定）就开缓存，命中率可能是 0.3%。**系统提示改一个字符 = 全量 miss。**
4. **把投机解码当免费加速**。公开评测大多在 batch=1 下做，与生产 regime 无关。
5. **≤10B 的模型上 FP4**。省下的显存救不了掉的 2 分。
6. **用 MPS/time-slicing 做跨租户隔离**。MPS 无故障隔离，time-slicing 连显存都不隔离。
7. **只写一个 SLO 指标**。TTFT 和 TPOT 必须同时约束，否则 P95 TPOT 能在你不知情的情况下漂 10×。
8. **横向比较单卡 tok/s**。同一批 288 卡半年内软件涨 2.7×；prefill-only 口径与混合口径差 7×。没有 ISL/OSL/并发/量化/版本标注的数字一律作废。
9. **跟 latest 版本**。vLLM 两周一个 minor 且 v0.25.0 直接删了 legacy attention 路径；SGLang 一个月三个版本。**生产必须锁版本 + 有回归基线**。
10. **信"博客里的 GA"**。2026 年大量能力仍是实验态：AFD 明确 experimental、Mooncake 集成未合入主干、Elastic EP 只支持 TP=1、多层 MTP 未稳定。逐项核对 release note。
11. **用 CPU/GPU 利用率驱动 HPA**。decode 阶段 GPU 利用率读数很高而 MFU 只有 5%——这个指标对 LLM 服务几乎没有信息量。用 goodput、队列深度、`gpu_cache_usage_perc` 和抢占计数。

---

## 面试官会追问

1. prefill 和 decode 的瓶颈分别是什么？为什么 decode 的 batch 要做到 100 以上才有意义？
2. 给你 2 张 H100 和一个 70B 模型，算一下能支撑多少并发、上下文多长。公式写出来。
3. 什么情况下 PD 分离会让系统**变慢**？判据是什么？
4. 前缀缓存的命中率取决于什么？为什么 agent 场景的命中率会跌破 20%？你怎么把它救回来？
5. 缓存感知路由和负载均衡冲突时你怎么办？扩容之后延迟为什么会先变差？
6. 你的 SLO 只写了 P95 TTFT < 2s，我给你一个满足它但用户体验很差的配置——问题出在哪？
7. 投机解码在你的生产系统里为什么可能是负收益？什么场景该开？
8. 跨租户共享 KV cache 有什么风险？你的默认策略是什么，代价是什么？
9. 从 QPS = 10、ISL = 4k、OSL = 500 推到要几张卡，把过程讲一遍。
10. 流量突增 3×，GPU 冷启动要 10 分钟——你的降级剧本是什么？

---

**下一篇** → [02-context-engineering-and-rag.md](02-context-engineering-and-rag.md)
