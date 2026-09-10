> 调研对象：AMD **Ryzen AI Max 400 系列**，旗舰 **Ryzen AI Max+ 495**，代号 **Gorgon Halo**（前代 **Strix Halo** 旗舰为 **Ryzen AI Max+ 395**）。
> 调研时间：2026-09-10。重点：本地大模型推理平台，容量 / 带宽 / 性价比 / 未来路线。

---

## 一、核心结论（TL;DR）

- **495 是 Gorgon Halo（Ryzen AI Max 400）的旗舰**，是 Strix Halo（395）的**换代刷新款**，不是架构跃迁：CPU 仍是 **Zen 5**、GPU 仍是 **RDNA 3.5**，NPU 升到 **XDNA 2（55 TOPS）**。
- **最大的升级是内存**：从 Strix Halo 的 **128GB** 统一内存拉到 **192GB**（其中 **160GB** 可分配给 iGPU 当显存），成为**首个能在 x86 客户端单芯片跑 300B+ 大模型**的平台。
- **命门在带宽，不在容量**：理论 **~256 GB/s**（实测 ~215 GB/s）。这个带宽决定了**稠密模型（dense）解码很慢、MoE 模型飞快**。
- **对跑大模型的人**：这是"用最低成本把 70B/120B/300B 模型装进一台小盒子"的路线，**容量党 / MoE 党**的最优选之一；**追求单模型解码速度**的人，带宽更高的 **Apple M4 Max（~546 GB/s）/ M3 Ultra（819 GB/s）** 或独显更合适。

---

## 二、它是什么：平台定位

AMD 的赌注是——**对本地 AI 而言，一大池"统一内存"比"快但小"的独显更值钱**。

传统做法是 CPU + 独立显卡（各自独立显存）；Strix Halo / Gorgon Halo 把 CPU、GPU、NPU 和内存**全塞进一个封装**，CPU 和 GPU **共享同一池统一内存**，并且允许把大部分内存直接划给 GPU 当显存用。

结果：一台 ~$2,000–$4,000 的迷你主机 / 小盒子，就能加载 **24GB/32GB 独显根本塞不下的 70B 级模型**，成本远低于堆多张卡。

- **CPU**：16 核 / 32 线程 **Zen 5**（495 睿频 **5.2 GHz**）
- **GPU**：**RDNA 3.5**，40 个计算单元（395 为 **Radeon 8060S**，495 为 **Radeon 8065S**），接近 **RTX 4070 Laptop** 档位
- **NPU**：**XDNA 2**，395 = 50 TOPS，495 = **55 TOPS**，负责低功耗端侧 AI
- **内存**：**LPDDR5X-8000 统一内存**，256-bit 总线，理论 **256 GB/s**
- **制程**：台积电 **N4P（4nm）**
- **TDP**：约 45–120W（整机按 120W 设计）

---

## 三、规格对比：395（在售 Strix Halo）vs 495（Gorgon Halo）

| 项目 | **Ryzen AI Max+ 395**（Strix Halo，已上市） | **Ryzen AI Max+ Pro 495**（Gorgon Halo，Q3 2026 上市） |
| --- | --- | --- |
| CPU | 16C/32T Zen 5，睿频 ~5.1 GHz | 16C/32T Zen 5，睿频 **5.2 GHz**（+100MHz） |
| GPU | **Radeon 8060S**，RDNA 3.5，40 CU | **Radeon 8065S**，RDNA 3.5，40 CU |
| NPU | XDNA 2，**50 TOPS** | XDNA 2，**55 TOPS** |
| 缓存 | 80 MB | 80 MB |
| 统一内存 | 最高 **128GB**（**96GB** 可作显存） | 最高 **192GB**（**160GB** 可作显存，留 32GB 给系统） |
| 内存带宽 | 理论 256 GB/s（实测 ~215 GB/s） | 同级（同为 LPDDR5X-8000） |
| 定位 | 消费 / 商用（PRO 版在 HP 等） | 商用 PRO 版先行，消费版待定 |
| 状态 | 已上市（Framework / GMKtec / Beelink / Minisforum / HP / AMD 自家 Halo） | **已官方发布，系统"coming soon"，2026 Q3 陆续出** |

> 495 本质是"刷个频 + 上更大内存"的小改款，CPU/GPU 微架构不变，**性能比 395 高约 10%**（主要来自频率和 NPU）。真正拉开差距的是 **192GB 容量**。

---

## 四、性能实测（以在售 395 为基准，495 约快 10%）

### 4.1 决定速度的唯一数字：内存带宽

生成 token 时，每产出一个 token 都要把用到的权重从内存里读一遍，所以：

> **解码速度 ≈ 内存带宽 ÷ 每 token 读取的字节数**

Strix Halo 理论 256 GB/s、实测 ~215 GB/s——**只有独显（800–1000 GB/s）的 1/4**，但这正是"本地 AI 慢"口碑的来源，**只对稠密模型成立**。

带宽横向对比：

| 平台 | 内存带宽 | 容量 | 软件栈 |
| --- | --- | --- | --- |
| **Strix Halo（395/495）** | 256 GB/s（~215 实测） | 128 / 192 GB | ROCm / Vulkan / llama.cpp |
| **NVIDIA DGX Spark（GB10）** | 273 GB/s | 128 GB | CUDA |
| **Apple Mac Studio M4 Max** | **最高 546 GB/s** | 最高 128 GB | MLX / llama.cpp |
| **Apple M3 Ultra** | 819 GB/s | — | MLX |
| **独立显卡（RTX 级）** | 800–1000 GB/s | 24–32 GB（小） | CUDA |

### 4.2 稠密 vs MoE：同一块芯片，15 倍差距（4-bit 量化）

**这是买 Strix Halo 前最该懂的一点**——速度取决于**模型架构**远大于模型大小：

| 模型（4-bit） | 类型 | 每 token 激活参数 | Strix Halo 解码速度 | 结论 |
| --- | --- | --- | --- | --- |
| **7–13B** | 稠密 | 7–13B | ~30–45 tok/s | 轻快 |
| **Qwen3-30B-A3B / Qwen3-Coder 30B** | **MoE** | ~3B | **~70–100 tok/s** | 甜点区，比阅读还快 |
| **GPT-OSS 120B** | **MoE** | ~5.1B | **~31 tok/s**（@120W） | 舒适可用 |
| **稠密 70B（如 Llama 70B）** | 稠密 | ~70B | **~5 tok/s** | 能读，不轻快 |

**逻辑**：稠密模型每个 token 都要读全部权重 → 带宽吃紧 → 单 token 个位数 tok/s；**MoE** 整模型驻留内存但每 token 只激活几个专家，读取量小得多 → **同一台机器 15 倍速**。这正是 MoE 浪潮（Qwen3 / GPT-OSS / DeepSeek）对"带宽受限的统一内存盒子"最大的利好。

### 4.3 第三方实测细节（Beelink GTR9 Pro，395，128GB）

来自 AGmind Systems Lab 的公开实测（llama.cpp，Vulkan/ROCm）：

- **Qwen3.6-35B-A3B Q4_K_M**：Vulkan 解码 **15.9 ms/token（≈63 tok/s）**，ROCm 18.7 ms/token；首 token（TTFT）~210ms
- **并发**：并发 4 时中位 30 ms/token，并发 8 时 TTFT 涨到 ~865ms
- **3 小时持续满载**：并发 4 下速度漂移仅 **1.6%**（散热好，不降频明显）
- **长上下文（32k token 文档）**：无缓存 TTFT ~34 秒；**开启 prompt cache 后第二问只要 ~860ms**（缓存价值巨大）
- **prefill（提示词处理）是算力瓶颈**：长上下文 / RAG / 整文件代码场景，首 token 延迟会明显大于独显——**短聊天无感，长检索要吃亏**

> 后端经验：**Vulkan 通常比 ROCm 解码更快**；**Q8_0 与 Q4_K_M 质量门几乎无差别，Q4 解码更快**——日常建议直接用 Q4。

---

## 五、生态与软件

- 运行栈是 **ROCm / Vulkan / llama.cpp / LM Studio**，**不是 CUDA**——推理之外更重的 GPU 计算比 NVIDIA 生态粗糙。
- **支持 Windows + Linux**（AMD 自家盒子主打 Linux）；对比 **NVIDIA DGX Spark 只支持 Linux**。
- AMD 官方口径：Ryzen AI Halo 在 **GLM 4.7 Flash 30B** 上比 DGX Spark 高 **~14% tok/s**，**Qwen 3.6 35B** 上高 ~4%（官方数据，参考即可）。

---

## 六、产品与价格

同一颗芯片各厂商一致，差别在散热 / IO / 价格：

| 机器 | 定位 / 亮点 | 参考价 |
| --- | --- | --- |
| **Framework Desktop** | 最便宜可上 128GB，mini-ITX，可玩性最高 | ~$3,449（128GB 配） |
| **GMKtec EVO-X2** | 旗舰、安静、双 M.2，常开设备 | $1,999–$3,649 |
| **Beelink GTR9 Pro** | 双 10GbE + 双 USB4，散热强，适合 NAS/集群 | ~$4,349 |
| **Minisforum MS-S1 Max** | IO 最全：双 10GbE / USB4 v2 / PCIe x16 / 320W | ~$3,719 |
| **HP Z2 Mini G1a** | 商用 PRO：vPro、ECC、3 年保 | $3,300–$3,734 |
| **AMD Ryzen AI Halo（395）** | AMD 官方盒子，128GB + 2TB，Wi-Fi7 + 10GbE | **$3,999 起**（6 月开订） |
| **NVIDIA DGX Spark（对手）** | GB10，128GB + 4TB，只支持 Linux | $4,700 |

> 注意：内存是**焊死的**，没有升级路径——**要 128/192GB 就一步到位买对应 SKU**。

---

## 七、未来发展

### 7.1 已定：Gorgon Halo（400 系列）Q3 2026 落地

- **已官方发布**，系统"coming soon"，OEM 机器 **2026 Q3 起**陆续公布（Framework / GMKtec / Beelink / Minisforum 等大概率复刻 Strix Halo 的保守铺货节奏）。
- 首发确认机型是 **Ryzen AI Halo（495 版）**，定价与 395 版（$3,999 起）接近。
- 商用 **PRO** 版先行（企业级安全 / 可管理性 / 可靠性），消费版"still up in the air"。

### 7.2 竞争格局

- **NVIDIA DGX Spark**：带宽略高（273 GB/s）+ CUDA + 集群软件，单盒裸速不如 AMD 略胜在软件。
- **Apple**：M4 Max（546 GB/s）/ M3 Ultra（819 GB/s）带宽碾压，**追求 128GB 下最快解码是 Apple 的场**；但 Apple 走 ARM + 闭源。
- AMD 的差异化叙事是 **"token 经济学"**：官方称一台 Ryzen AI Halo 相比云端可**每月省 ~$750**，按每天 600 万 token 计算 **6 个月回本**——瞄准 AI Agent 的高 token 消耗场景。

### 7.3 风险 / 不确定

1. **DRAM 全球缺货**正在推高所有内存价格——**192GB 版本能否稳定量产是个问号**（同期 Apple 已砍掉 Mac Studio 的 512GB 甚至 128GB 选项）。
2. **192GB 只是"能跑 300B"，不代表快**：容量上去了，带宽仍是 ~256 GB/s 那档，稠密 300B 解码依旧个位数 tok/s。
3. **消费版 495 时间表未定**，目前只确认商用 PRO 与 AMD 官方盒子。
4. 生态仍是 **ROCm 非 CUDA**，重度 GPU 计算 / 训练类场景不如 NVIDIA 成熟。

---

## 八、对达叔的落地建议（结合你现有的 M4 Max 36GB）

- 你手上的 **M4 Max 36GB 带宽 ~546 GB/s**，**解码速度比 Strix Halo 快 ~2 倍多**——**36GB 内能装下的模型，你的 Mac 更快**。
- Strix Halo / Gorgon Halo 的真正价值是**容量**：128–192GB 能装下你 Mac 装不下的 **70B / 120B MoE / 300B** 模型。
- **所以别指望它"更快"，而是"更大"**：想跑 MoE 大模型（Qwen3-30B-A3B、gpt-oss-120b 这类）本地常驻，这是一台很划算的小盒子；**追求单模型快解码，继续用你的 M4 Max 或上独显**。
- 若买，**优先 MoE 工作负载 + 一步到位买 128/192GB SKU**，并选散热好的盒子（Beelink GTR9 Pro / Minisforum MS-S1 Max）以保持续满载不降频。
- **等 Gorgon Halo 495 还是现在买 395？** 495 只比 395 快 ~10% 但内存大 50%——**如果预算够、不急着用，等 Q3 出 495 更值**；急着用，395 现货已很成熟。

---

## 来源

- Tom's Hardware：Gorgon Halo / Ryzen AI Max 400 规格表、Ryzen AI Halo 定价与 DGX Spark 对比、DRAM 缺货风险 — tomshardware.com/pc-components/cpus/amd-ryzen-ai-max-400-gorgon-halo-packs-up-to-192gb-of-unified-memory-refreshed-apu-uses-zen-5-and-rdna-3-5-and-can-clock-up-to-5-2-ghz
- datahardware.ai：Strix Halo 芯片解析（395 规格 / 96GB 显存 / 带宽对比）— datahardware.ai/blog/amd-ai-max-395-explained
- datahardware.ai：Strix Halo tokens-per-second 2026（稠密 vs MoE 实测表、prefill vs decode、机型对比）— datahardware.ai/blog/strix-halo-tokens-per-second-2026
- AGmind Systems Lab（botAGI）：Strix Halo LLM 实测数据（Beelink GTR9 Pro，Qwen3.6-35B-A3B，Vulkan/ROCm，3 小时持续负载、prompt cache）— github.com/botAGI/strix-halo-llm-benchmarks
- wccftech / partofstyle / anintent：Ryzen AI Max+ 495 Gorgon Halo 泄露与 192GB / 300B LLM 报道（2026-05）
- Wikipedia：Zen 5 / Ryzen 微架构背景
