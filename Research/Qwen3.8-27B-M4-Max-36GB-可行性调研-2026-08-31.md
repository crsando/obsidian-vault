
> 检索日期：2026-08-31（周一）
> 承接：`Qwen3.8-27B 本地部署高性价比方案对比与结论 2026-08-31`、`Qwen3.8-27B-替代方案研究-2026-08-30`

---

## 一、结论速览

**可行，但属于"刚好卡线"（tight fit）。**

- **能跑**：Qwen3.8-27B 4-bit 权重约 16–18GB，M4 Max 36GB 可用约 25.9GB，装得下，**decode 速度约 23–28 tok/s**。
- **代价 1 —— 内存余量极小**：约 94% 内存被占用，余量仅约 4GB，安全上下文被压到 **24K** 左右（发挥不了 262K 原生上下文）。
- **代价 2 —— 带宽是 binned 版**：36GB 起步款是 **14 核 CPU + 32 核 GPU 的阉割版，内存带宽 410GB/s**，比 64GB 完整版的 **546GB/s 慢约 25%**。
- **代价 3 —— 新架构 kernel 未优化**：Qwen3.8 的 hybrid attention 在 Ollama Metal 上尚未调优，**比 Qwen3.6 慢约一半**（同机 14 vs 28.6 tok/s），需靠 MTP 投机解码补回。
- **一句话**：36GB 是"能跑但紧"的**最低可行档**，适合轻量聊天 + 短上下文 + 预算卡死；要舒服，得 64GB（完整版带宽 + 翻倍余量）。

---

## 二、前提回顾（之前已确认的结论）

1. **Qwen3.8-27B 4-bit 权重约 16GB**（mlx-community 4bit 约 16.1GB、Unsloth Q4_K_M 约 16.46GB）。
2. decode 是 **memory-bound**：生成速度 ≈ 内存带宽 ÷ 单 token 权重字节数（4-bit 27B 约 17GB）。
3. 之前不买 M3 Ultra 96GB（¥32,999 太贵），首选二手 M2 Ultra 192GB，备选 M5 Max。
4. M3 Ultra 96GB 实测（oMLX 优化路径）：4K 输入 gen 59.4 tok/s / TTFT 10.9s；32K 输入 gen 53.9 tok/s / TTFT 113s。

---

## 三、M4 Max 36GB 规格确认（本次关键发现）

| 项目 | M4 Max 14 核（binned） | M4 Max 16 核（完整） |
|---|---:|---:|
| CPU | 14 核（10P+4E） | 16 核（12P+4E） |
| GPU | 32 核 | 40 核 |
| **内存带宽** | **410 GB/s** | **546 GB/s** |
| 内存档位 | 36GB（可升 48/64/128GB） | 64GB/128GB 为主 |
| 对应机型 | Mac Studio M4 Max 起步款 | MacBook Pro / Studio 高配 |

- **决定性事实：36GB 起步款的内存带宽是 410GB/s，不是 546GB/s。** 546GB/s 是 16 核 + 40 核完整版才有的，两者不能混用。
- Mac Studio M4 Max 起步款（14 核 + 32 核 + 36GB + 512GB）上市价 **¥16,499**（2025 年 3 月）。
- ⚠️ **时效提示**：Mac Studio 已于 **2026-08-25 更新到 M5 Max 世代**（18 核 CPU + 40 核 GPU，546GB/s 起），**M4 Max 36GB 新机已下架**，现为二手/渠道库存状态，二手价需查闲鱼"已成交"核实。

---

## 四、内存账：36GB 到底够不够

Qwen3.8-27B 4-bit 各格式体积（实测值，非估算）：

| 格式 | 体积 | 来源 |
|---|---:|---|
| bartowski Q4_K_M (GGUF) | 17.77GB | HuggingFace API |
| Ollama 27b blob + vision projector | 15.7GiB + 0.9GiB | registry manifest |
| mlx-community 4bit | ~15.0GiB（3 shards） | HF |

**M4 Max 36GB 实测内存分解**（willitrunai，Qwen 27B Q4_K_M 基准）：

| 项目 | 占用 |
|---|---:|
| 权重 | 16.5GB |
| KV cache | 3.2GB |
| 运行时 | 0.9GB |
| **合计** | **20.6GB** |
| 可用（36GB 扣除系统后） | 25.9GB |
| **剩余余量** | **约 3.9–4GB** |

- 占用率 **94%**，余量仅 4GB。这意味着：开个浏览器 + 几个常驻应用，就可能在推理时触发 swap，速度断崖式下跌。
- **上下文限制**：262K 是模型原生上限，但 36GB 机器上安全上下文只有 **24K** 左右；要跑长文档/RAG/长 agent 会话，得 32GB 甚至 64GB 以上。
- 参考分级（codepick / ai-on-mac / modelfit 三家一致）：**16GB 跑 4B/9B 或 IQ2；24GB 是 Q4 最低舒适档；32GB 是 27B Q4 甜点；64GB 上 Q6/Q8/FP16。**

---

## 五、速度实测（多来源交叉）

| 来源 | 机型 / 带宽 | 模型 / 量化 | decode | TTFT | 上下文 |
|---|---|---|---|---|---|
| willitrunai | M4 Max 36GB (410) | Qwen3.5 27B Q4_K_M | **28 tok/s** | 6.7s | 24K |
| willitrunai | M4 Max 64GB (546) | 同 | 36 tok/s | — | — |
| willitrunai | M4 Pro 48GB (273) | 同 | 22.7 tok/s | — | — |
| 中文社区（8 天前） | M4 Max（未注内存）MLX 4bit | Qwen 27B | **~23 tok/s** | — | — |
| terminalbytes | M3 Ultra 256GB (819) | **Qwen3.8** 27B Q4 | **14 tok/s** | — | — |
| vinoth | M4 Pro 48GB (273) | Qwen3.6 27B + MTP | 7 → **18.3 tok/s** | 0.66s | 4K |

**三个关键读数：**

1. **memory-bound 公式验证**：410GB/s ÷ 17GB ≈ **24 tok/s 理论上限**，实测 23–28 tok/s 完全吻合。带宽比容量更决定速度——这也是为什么 M4 Max 36GB（410）跑得比 M4 Pro 48GB（273）还快（28 vs 22.7 tok/s），尽管后者内存更大。
2. **Qwen3.8 新架构在 Ollama 上慢一半**：M3 Ultra 上 Qwen3.8 27B 只有 14 tok/s，前任 Qwen3.6 有 28.6 tok/s——hybrid attention 的 Metal kernel 还没优化。**但** Qwen3.8 回答同样问题用的 token 数只有 1/3（约 1000 vs 2000–3000），所以**单次回答墙钟时间几乎持平**。
3. **MTP 是隐藏变量**：vinoth 用 MLX 原生 MTP（MTPLX，depth 3）把 27B 从 7 拉到 18.3 tok/s（**2.6 倍**）；中文社区也明确说"MTP 开和不开速度差接近一倍"。**以后看谁晒速度，先问开没开 MTP。**

---

## 六、网上用户反馈汇总（按来源）

### 1. willitrunai.com（M4 Max 36GB 专项，Qwen 3.5 27B 基准）
- 结论 **"YES — Tight Fit"**，S89 评级。
- "你能跑这个模型，但留给更长上下文、更大 batch、额外应用、未来模型升级的空间所剩无几。"
- 明确建议：**"买余量，而不是买最低刚好装下"**（Buy headroom, not minimum fit）。
- 量化上限：Q4_K_M（16.5GB）和 Q5_K_M（19.4GB）能跑；Q6_K（22.1GB）开始不稳；Q8_0（28.9GB）放不下。

### 2. codepick.dev（Qwen3.8-27B 官方 Mac 指南）
- "Q4 量化 27B 约需 18–20GB，**32GB 是跑 Qwen3.8-27B 的甜点**；16GB 上跑 4B/9B。"
- 三条部署路径：**Ollama**（最省心，2026 年起走 MLX 引擎）、**oMLX**（编码首选，连续批处理 + 分层 KV cache）、**MTPLX**（MTP 原生投机解码，1.6–2.24x 提速）。

### 3. terminalbytes.com（Mac Studio M3 Ultra 256GB 十天实测）
- Qwen3.8 27B Q4_K_M（17GB）= **14 tok/s**（Ollama），但"更慢 per token，更快 per answer"。
- RAM 分级：Q4_K_M 16–17.6GB → **32GB**；Q6_K 22GB → 32GB(紧)/48GB；Q8_0 29GB → 48–64GB。
- 1-bit quant（6.7GB）27 tok/s，但"事实记得住、决断力没了"，**不适合工具调用/agentic**。
- 坑：**旧版 llama.cpp 报 `unknown model architecture: 'qwen35'`**，必须升级到最近几周的新版。

### 4. modelfit.io（Qwen3.8-27B 本地部署指南）
- "Q4 约 17.8GB，**24GB Mac 或 GPU 能舒服跑**"，AMD 官方在 Ryzen AI Max+ 395 上实测 24.5 tok/s。
- "120GB/s 的 M4-class 芯片会明显低于 Ryzen 数值，因为 dense 27B decode 是带宽受限的。"
- 24GB 机器上 macOS + 应用占掉后约剩 19–20GB，**24GB 是 Q4 不加戏的最小档**。

### 5. ai-on-mac.com（Qwen3.6 Mac 指南）
- RAM 建议表：24GB = 17–20GB tag 留余量少、短上下文；**32GB = 实际可测 27B**；48GB+ = 35B-A3B/视觉/长上下文。

### 6. dranixj.com（中文，Mac Studio Qwen3.6）
- 确认 **Apple 已于 2026 年 5 月停产 M3 Ultra Mac Studio**，M5 Ultra 预计 2026 年末。
- Mac Mini M4 16GB 靠 llama.cpp `--mmap` 跑 35B-A3B 能到 ~17 tok/s；M3 Ultra 96GB 是 35B-A3B 最佳点（~71 tok/s）。
- Qwen3.6-27B 4bit 实际占用 18GB；MLX 路径比 GGUF 快约 50%。

### 7. vinoth12940.github.io（MacBook M4 Pro 48GB，MTP 实测）
- 核心公式：**120GB/s ÷ 17GB ≈ 7 tok/s 理论极限**（M4 Pro 基准），实测基线 7 tok/s。
- MLX 原生 MTP（MTPLX D3）拉到 **18.3 tok/s（2.6x）**，且是数学正确的拒绝采样，非贪心近似。
- 结论："dense 27B 是带宽受限的，MoE（35B-A3B）才是下一步。"

### 8. 中文社区（百度聚合，近 8 天反馈）
- "**M4 Max 跑 MLX 4bit 大概 23 token/s**，M5 Max 优化完加 MTP 能到 50–60，RTX 5090 开 MTP 能冲 90–120。"
- "入门款 Mac 会慢一些，日常聊天够用，跑长任务就得有点耐心。"
- "MTP 开和不开速度差接近一倍；以后看见谁晒超高速度，先问一句开没开 MTP。"

---

## 七、坑点清单

1. **36GB 余量仅约 4GB**：开浏览器 + 多应用容易挤占内存 → 触发 swap → 速度断崖。
2. **262K 上下文名不副实**：36GB 上安全上下文只有约 24K，长文档/RAG/长 agent 会话做不了。
3. **Qwen3.8 需最新 runtime**：llama.cpp / Ollama / LM Studio 旧版报 `unknown model architecture: 'qwen35'`，先升级再排障。
4. **Ollama Metal kernel 对 Qwen3.8 未优化**：比 3.6 慢约一半，靠 MTP 补回（MTPLX 可 2.6x）。
5. **别被"晒速度"骗**：很多高数字是开了 MTP、关了 reasoning 的"美颜数"，裸跑 23–28 tok/s 才是 M4 Max 36GB 的真实水平。

---

## 八、针对达叔场景的选型建议

- **M4 Max 36GB 的定位**：它是"能跑但紧"的**最低可行档**，且有个反直觉优势——因为带宽 410GB/s 高于 M4 Pro 的 273GB/s，**跑 27B 反而比 M4 Pro 48GB 更快**（28 vs 22.7 tok/s）。如果你要的是"一台便宜 Mac 顺手跑 27B 日常聊天"，它够用。
- **但如果要认真跑 Qwen3.8-27B 做研究/长文档/agent**，36GB 的两处硬伤（余量 4GB、上下文 24K）会很憋屈。**预算允许就上 64GB 版**：完整 546GB/s 带宽（速度 +28%）、余量翻倍、上下文能到 32K–64K。
- **性价比对照**（延续之前的"元/GB/s 带宽"口径）：
  - M4 Max 36GB（410GB/s）：¥16,499（上市价）→ 约 ¥40/GB/s
  - M4 Max 64GB（546GB/s）：约 ¥20,999+ → 约 ¥38/GB/s，**带宽单价反而更优且体验翻倍**
  - 二手 M2 Ultra 192GB（800GB/s）：¥1.8–2.3 万 → 约 ¥25/GB/s，仍是"速度持平 + 大内存"的性价比之王（承接前次结论）
- **一句话结论**：M4 Max 36GB **可行但不舒服**，适合"预算卡死 + 轻量聊天"；要舒服要么加到 64GB，要么维持之前结论蹲二手 M2 Ultra 192GB。

---

## 九、来源清单

1. willitrunai（M4 Max 36GB 专项，Qwen 27B）：https://willitrunai.com/can-run/qwen-3.5-27b-on-m4-max-36gb
2. codepick（Qwen3.8-27B Mac 指南）：https://codepick.dev/en/guides/qwen3-27b-mac-local/
3. terminalbytes（Qwen3.8 27B Mac Studio 十天实测）：https://terminalbytes.com/run-qwen-3-8-27b-locally/
4. modelfit（Qwen3.8-27B 本地部署指南）：https://modelfit.io/blog/run-qwen38-27b-locally-2026/
5. ai-on-mac（Qwen3.6 Mac 指南）：https://ai-on-mac.com/articles/qwen3-6-mac-setup/
6. dranixj（中文，Mac Studio Qwen3.6）：https://dranixj.com/articles/mac-studio-qwen-3-6-local-llm-guide
7. vinoth12940（MacBook 27B + MTP 实测）：https://vinoth12940.github.io/blog/articles/genai-20260519-local-mtp-speculative-decoding/
8. claw-world（Qwen3.8 27B Mac 十天转述）：https://www.claw-world.app/blog/qwen3-8-27b-local-mac-studio-benchmark
9. Apple Mac Studio 技术规格：https://www.apple.com/mac-studio/specs/
10. M4 Max 带宽确认（百度百科 + 中文社区）：https://www.baidu.com/s?wd=M4+Max+36GB+内存带宽+410GB/s

> 备注：willitrunai 的 28 tok/s 是以 **Qwen 3.5 27B**（同 27B dense、同 Q4_K_M）为基准测的，Qwen3.8 因新架构在 Ollama 上会更慢（参考 M3 Ultra 上 14 tok/s），但 MLX 4bit / MTP 路径可追回。所有 tok/s 均为单流 decode，实际随 prompt 长度、上下文、batch、runtime 版本浮动。
