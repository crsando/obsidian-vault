
> 调研日期：2026-09-05（周六）
> 承接笔记：`Qwen3.8-27B-M4-Max-36GB-可行性调研-2026-08-31`、`Qwen3.8-27B-M4-Max-64GB-可行性调研-2026-08-31`、`Qwen3.8-27B-Mac部署方案成熟度与最优选-2026-08-31`
> 本文专门补齐 48GB 档位（36GB / 64GB 两篇均未单独覆盖），其中可参考 64GB 部分结论。

---

## 一、结论速览

**48GB 是 M4 Max 里「最容易被买错」的档位——它同时存在 14 核（410GB/s）和 16 核（546GB/s）两种芯片，买错差 25% 带宽。**

- **内存不是瓶颈**：48GB 扣掉系统后可用约 **41–43GB**，27B Q4 只占 16–18GB，余量约 **24–26GB**，上下文能从容开到 **128K**（hybrid 架构 KV cache 很小）。
- **速度依旧被 Qwen3.8 架构拖累**：裸跑（无 MTP）只有 **13–15 tok/s**，与 36GB/64GB 一个样——瓶颈是 Metal kernel 未优化，不是内存也不是带宽。
- **48GB 的定位**：比 36GB 多出的价值是「余量翻倍 + 长上下文」，比 64GB 差的唯一硬伤是「**跑不动 70B Q4**」。对 27B 而言，48GB 已经到顶，再加 16GB 内存（上 64GB）对 27B 速度零提升。
- **⚠️ 必查项**：买「M4 Max 48GB」必须确认是 **16 核（546GB/s）**，14 核（410GB/s）版虽然内存一样是 48GB，但带宽阉割，等于多花内存钱没拿到满血带宽。

---

## 二、最关键发现：48GB 是「分水岭」配置（14 核 / 16 核都有）

这是本调研最重要的结论，也是 36GB / 64GB 两篇笔记没单独点破的盲区。

Apple 的 M4 Max 有两个芯片变体，**内存档位在两档芯片之间是交叉的**：

| 芯片变体 | CPU / GPU | 内存带宽 | 默认内存 | 可选升级内存 |
|---|---:|---:|---|---|
| **binned 版（阉割）** | 14 核 + 32 核 GPU | **410 GB/s**（384-bit） | 36GB | **48GB** |
| **完整版（满血）** | 16 核 + 40 核 GPU | **546 GB/s**（512-bit） | **48GB** | 64GB / 128GB |

**关键点：48GB 是唯一一个两档芯片都能配的内存容量。**

- 14 核 binned 版：默认 36GB，**可加钱升到 48GB**（这是「阉割芯片 + 拉满内存」的组合）。
- 16 核完整版：**默认就是 48GB**，可再升 64GB / 128GB。

因此，二手市场 / 渠道里写「M4 Max 48GB」的机器，**既可能是 14 核 410GB/s，也可能是 16 核 546GB/s**，两者 decode 速度差约 25%，但内存数字完全一样。**买之前必须核对 CPU 核数（16 核 = 满血，14 核 = 阉割），不能只看内存。**

> 注：这比 64GB 档的陷阱更隐蔽。64GB 只有 16 核版才有（14 核最高只到 48GB），所以「64GB」天然等于满血；而「48GB」是模糊的，必须看核数。

来源：Wikipedia Apple M4（Max 内存 36/48/64/128GB，CPU 14/16 核，GPU 32/40 核）、EveryMac MacBook Pro M4 Max 14 核 / 16 核规格页、Apple Newsroom M4 Pro/M4 Max 新闻稿。

---

## 三、48GB 的内存账（内存不再是瓶颈）

M4 Max 48GB（16 核版）实测口径（modelfit / 社区）：

| 项目 | 占用 |
|---|---:|
| 物理总内存 | 48.0 GB |
| macOS 内核 + 服务 | ~4–5.5 GB |
| **可用给 LLM**（调 `sysctl iogpu.wired_mem_limit` 可到 90–92%） | **~41.5–43 GB** |
| Qwen3.8-27B 4-bit 权重 | ~16.5–17.8 GB |
| KV cache（hybrid 架构，32K 上下文 BF16） | ~1–2 GB |
| **剩余余量** | **~24–26 GB** |

**量化口径（约 0.6GB / 10 亿参数）：**

- Q4_K_M / MLX 4bit：27B ≈ **16.5–17.8GB**
- Q5_K_M ≈ 19.4GB、Q6_K ≈ 22.1GB、Q8_0 ≈ 28.9GB
- 35B-A3B MoE（Qwen3.6）≈ **22–26GB**（48GB 轻松装下）

**能跑什么 / 不能跑什么：**

| 模型 | 能否在 48GB 跑 | 说明 |
|---|---|---|
| Qwen3.8-27B 4bit | ✅ 从容 | 余量 24GB+，上下文 128K 无压力 |
| Qwen3.8-27B Q8_0 | ✅ 可以 | 权重 29GB，余量约 12GB，上下文中等 |
| Qwen3.6 35B-A3B（MoE） | ✅ 完美 | 22–26GB，是 48GB 上「又快又好」的甜点 |
| Qwen 70B Q4（约 40GB） | ❌ 不实用 | 权重 39–41GB 逼近 48GB 物理极限，上下文只能 2–4K，易触发 swap 速度雪崩 |

**核心结论：48GB 对 27B 已经「容量到顶」**——余量从 36GB 的 4GB 提到 24GB+，上下文从 24K 提到 128K，这是它相对 36GB 的真正价值；但它装不下 70B Q4，这是它相对 64GB 的唯一硬伤。

---

## 四、速度：两种芯片 × 裸跑/MTP 四种情况

先记住两条铁律（来自前序笔记）：

1. **Qwen3.8 裸跑慢是架构问题**：hybrid attention 的 Metal/MLX kernel 尚未优化，M4 Max 全系裸跑只有 **13–15 tok/s**，换什么 Mac 都一个样（M3 Ultra 819GB/s 也才 13–14）。
2. **MTP 是速度开关**：开 MTP 能到 **38–53 tok/s**（oMLX），峰值参考 M5 Max MTPLX 73 tok/s。

### 4.1 48GB（16 核，546GB/s）——与 64GB 速度完全一致

| 路径 | decode | 说明 |
|---|---:|---|
| Ollama / llama.cpp 裸跑 | **~15 tok/s** | vramcalculator @tomgreenwald 实测 64GB 同款 |
| oMLX 原生 MTP | **48–65 tok/s** | Mac Studio M4 Max 实测（成熟度笔记） |
| MTPLX（GUI） | **~2 倍速** | 无损投机解码 |
| 内存带宽理论上限 | 546÷17 ≈ 32 tok/s | 仅当 kernel 优化后（旧架构 Qwen3.5 实测 35 吻合） |

### 4.2 48GB（14 核，410GB/s）——与 36GB 速度一致

| 路径 | decode | 说明 |
|---|---:|---|
| Ollama / llama.cpp 裸跑 | **~13–15 tok/s** | kernel 瓶颈，410 vs 546 对裸跑影响被掩盖 |
| oMLX 原生 MTP | **~38–50 tok/s** | 带宽开始成为约束，比 16 核低约 20–25% |

### 4.3 关键 nuance：410 vs 546 对 Qwen3.8 的影响被「kernel 瓶颈」稀释

- **裸跑阶段**：Qwen3.8 的 kernel 未优化，410 和 546 都卡在 13–15 tok/s，**带宽差异几乎看不出**。
- **MTP / 未来优化后**：带宽才真正兑现，546 比 410 快 25%；且对**旧架构模型（Qwen3.5/3.6）和 MoE（35B-A3B）**，这个 25% 从第一天就体现（36GB/410 跑 Qwen3.5 是 28 tok/s，64GB/546 是 35–36 tok/s）。

**含义**：如果你「只跑 Qwen3.8 27B、且只用裸跑」，14 核和 16 核体验几乎一样；但只要开 MTP、或跑 MoE、或等未来 kernel 优化，16 核的 546GB/s 就值回票价。**所以哪怕 48GB，也建议买 16 核版。**

---

## 五、M4 Max 三档内存横向对比

| 维度 | 36GB（14核，410GB/s） | **48GB（14核，410GB/s）** | **48GB（16核，546GB/s）** | 64GB（16核，546GB/s） |
|---|---:|---:|---:|---:|
| 可用内存 | ~26GB | ~41–43GB | ~41–43GB | ~55–58GB |
| 27B Q4 余量 | ~4GB | ~24GB | ~24GB | ~35GB+ |
| 安全上下文 | ~24K | **128K** | **128K** | 128K+ |
| 27B 裸跑 | 13–15 tok/s | 13–15 tok/s | ~15 tok/s | ~15 tok/s |
| 27B + MTP | 38–42 | 38–50 | **48–65** | 48–65 |
| 35B-A3B MoE | 吃紧，易 OOM | ✅ 支持 | ✅ 完美（45–57 tok/s） | ✅ 完美 |
| 70B Q4 | ❌ 跑不了 | ❌ 跑不了 | ❌ 不实用 | ✅ 可跑（16–32K） |
| 一句话定位 | 能跑但紧 | 内存够、带宽阉割 | **27B 的容量甜点** | 唯一能上 70B 的档 |

---

## 六、部署方案（48GB 同样适用，与 64GB 一致）

按成熟度排序（承接《成熟度与最优选》笔记）：

1. **oMLX 原生 MTP（最优选，有完整实测）**
   - Mac Studio M4 Max 实测 **48–65 tok/s**，含 A/B + benchmark JSON
   - 配方：https://github.com/Weschera/Qwen3.8-27B-oMLX-MTP-Mac
   - oMLX 本体：https://github.com/jundot/omlx（连续批处理 + 分层 KV cache，OpenAI 兼容 API，适合接 agent/编码）

2. **MTPLX（省心 GUI，2 倍速无损）**
   - 官网：https://mtplx.com/ ；GitHub：https://github.com/youssofal/MTPLX
   - 模型：https://huggingface.co/Youssofal/Qwen3.8-27B-MTPLX-Optimized-Speed
   - macOS 14+ / Apple Silicon

3. **Ollama 0.32.13+（最省心，MTP 刚落地）**
   - 模型：`qwen3.8:27b`（GGUF）、`qwen3.8:27b-mlx`（MLX/nvfp4）
   - 裸跑 13–15 tok/s，MTP 支持刚发布、速度潜力未完全释放，可再等一两周

4. **mlx-dspark（极致速度，3–4 倍无损）**
   - https://github.com/ARahim3/mlx-dspark
   - DeepSeek DSpark + DFlash 投机解码的 MLX 移植

5. **官方量化基线**：mlx-community 4bit（https://huggingface.co/mlx-community/Qwen3.8-27B-4bit）、LM Studio（MLX 后端）

**坑点提醒**：旧版 llama.cpp / Ollama / LM Studio 会报 `unknown model architecture: 'qwen35'`，必须先升级到最新 runtime。

---

## 七、购买建议（2026-09 行情，M4 Max 已随 M5 世代上市而停产转二手/渠道）

| 机型 | 二手/渠道参考价 | 备注 |
|---|---:|---|
| MacBook Pro 14" M4 Max（16核+40核 / 48GB / 1TB） | **¥17,500–19,800** | CTO 版 |
| MacBook Pro 16" M4 Max（16核+40核 / 48GB / 1TB） | **¥19,500–22,500** | 16 寸散热更好 |
| Mac Studio M4 Max（16核+40核 / 48GB / 512GB–1TB） | **¥12,500–14,000** | 性价比最高，桌面静音 |

**针对达叔场景的建议：**

- **如果坚持跑 Qwen3.8-27B**：M4 Max 48GB（务必 16 核 546GB/s）是「27B 的容量甜点」——内存够、上下文 128K、MTP 后 48–65 tok/s，比 36GB 舒服太多，又不用为「跑不动 70B」多花 64GB 的钱（对 27B 零收益）。
- **短问答/聊天**：开 MTP 后 40–50 tok/s，体验流畅。
- **长文档/RAG/agent**：48GB 内存够开 128K 上下文，但 Qwen3.8 的 prefill 慢、TTFT 长，27B dense 长上下文 wall 速度会掉——这是 Mac 通病，不是 48GB 的问题。
- **更聪明的替代**：**Qwen3.6 35B-A3B（MoE）** 在 48GB 上 45–57 tok/s，质量接近 27B dense，是「又快又好」的真甜点；或等 Qwen3.8 的 MoE 变体。
- **一句话结论**：M4 Max 48GB（16 核版）是「27B 部署的性价比最优档」——**买它图的是容量和带宽的均衡，别指望它跑 70B**。要 70B 上 64GB；要极致速度上 RTX 5090（MTP 后 144 tok/s）或改用 MoE。

---

## 八、来源清单

1. Wikipedia — Apple M4 芯片规格（Max：36/48/64/128GB，14/16 核，32/40 核 GPU）：https://en.wikipedia.org/wiki/Apple_M4
2. EveryMac — MacBook Pro 14" M4 Max 14核/32核规格页（默认 36GB，可升 48GB）：https://everymac.com/systems/apple/macbook_pro/specs/macbook-pro-m4-max-14-core-cpu-32-core-gpu-14-2024-specs.html
3. EveryMac — MacBook Pro 16" M4 Max 16核/40核规格页（默认 48GB，可升 64/128GB）：https://everymac.com/systems/apple/macbook_pro/specs/macbook-pro-m4-max-16-core-cpu-40-core-gpu-16-2024-specs.html
4. Apple Newsroom — M4 Pro / M4 Max 发布新闻稿（带宽 up to 546GB/s，16核/40核满血版）：https://www.apple.com/newsroom/2024/10/apple-introduces-m4-pro-and-m4-max/
5. willitrunai — MacBook Pro M4 Max 48GB 规格页（546GB/s，$2,499 MSRP，170 模型全速）：https://willitrunai.com/macs/m4-max-48gb
6. vramcalculator — Qwen3.8 27B 真实硬件实测总表（M4 Max 64GB ~15 tok/s）：https://vramcalculator.com/qwen3-8-27b-speed/
7. modelfit — Best LLM for MacBook Pro M4 Max 64GB（14核/16核差 25%）：https://modelfit.io/blog/best-llm-macbook-pro-m4-max-64gb/
8. 前序 Obsidian 笔记：`Qwen3.8-27B-M4-Max-36GB-可行性调研-2026-08-31`、`Qwen3.8-27B-M4-Max-64GB-可行性调研-2026-08-31`、`Qwen3.8-27B-Mac部署方案成熟度与最优选-2026-08-31`

> 备注：所有 tok/s 均为单流 decode，随 prompt 长度、上下文、batch、runtime 版本浮动。价格中「二手/渠道」为 2026-09 行情估算，需查闲鱼「已成交」核实。速度口径中，Qwen3.8 裸跑 13–15 tok/s 是真实实测，中文社区「23–28 tok/s」疑似含 MTP 或短上下文峰值，勿直接横向对比。
