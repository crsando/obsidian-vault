
> 检索日期：2026-08-31（周一）
> 承接：`Qwen3.8-27B-M4-Max-36GB-可行性调研-2026-08-31`（36GB 版结论：可行但 tight fit）

---

## 一、结论速览

**64GB 解决了"内存焦虑"，但暴露了另一个更棘手的问题——Qwen3.8 新架构在 Mac 上还没优化好。**

- **内存完全不是瓶颈**：64GB 扣掉系统后可用约 **56–58GB**，27B Q4 只占 16–18GB，余量充足，上下文能到 32K–64K，甚至能装下 70B Q4（40GB）。
- **但速度被 Qwen3.8 拖累**：M4 Max 64GB 实测 Qwen3.8 27B 4bit **只有 ~15 tok/s**（真实用户 @tomgreenwald 实测），比 Qwen3.5/3.6 的 23–35 tok/s **慢 40–50%**。原因是 hybrid attention 新架构的 Metal kernel 还没优化。
- **⚠️ 必查项：64GB 有 14 核 / 16 核两种版本**，带宽 410 vs 546GB/s，买错差 25%。
- **真正的速度钥匙是 MTP 和 MoE**：MTP 能翻倍；而 Qwen3.6 35B-A3B（MoE）在同样机器上 50–57 tok/s，比 27B dense 快 3 倍。

---

## 二、关键发现一：64GB 也有 14核/16核之分（410 vs 546GB/s）

| 版本 | CPU/GPU | 内存带宽 | 对 27B decode 影响 |
|---|---:|---:|---|
| 14 核（binned） | 14C + 32核 GPU | **410 GB/s** | 慢约 25% |
| 16 核（完整） | 16C + 40核 GPU | **546 GB/s** | 基准 |

- **M4 Max 64GB 两种芯片都能配**（Apple 官方规格 + Google AI 汇总一致确认）。
- 16 核完整版跑 27B 比 14 核 binned 版**快约 25%**（modelfit / Google 一致口径）。
- **购买时必须看 CPU 核数，不能只看内存**。Mac Studio / MacBook Pro 的 64GB 档，默认可能是 14 核版，要明确选 16 核才拿到 546GB/s。
- ⚠️ 时效提示：Mac Studio 已于 2026-08-25 迭代到 M5 Max 世代，M4 Max 64GB 新机同样转二手/渠道库存。

---

## 三、关键发现二：64GB 的内存账（内存不再是瓶颈）

modelfit 实测口径（MacBook Pro M4 Max 64GB）：

| 项目 | 占用 |
|---|---:|
| macOS 内核 + 服务 | ~4GB |
| 常驻应用（浏览器/编辑器） | ~2–4GB |
| **可用给 LLM** | **~56–58GB** |

- Q4_K_M 量化成本约 **0.6GB / 10 亿参数**：27B ≈ 16GB、35B MoE ≈ 21–22GB、70B ≈ 40GB。
- 对比 36GB（可用 25.9GB）→ 64GB（可用 56–58GB），**可用内存翻倍以上**，这直接解决 36GB 版"余量 4GB、上下文 24K"的两大痛点。
- 上下文能力：64GB 跑 27B Q4，**32K–64K 上下文处理大型代码库/多文档 RAG 没问题**（中文社区 2026-07 反馈）；但 Qwen3.8 的 prefill 慢，长 prompt 的 TTFT 会显著拉长（见下）。

---

## 四、关键发现三：Qwen3.8 新架构拖慢 Mac（真实实测）

**这是本次调研最重要的真相。** vramcalculator 汇总了社区真实实测（均注明测试者），Qwen3.8 27B 在 Mac 上：

| 硬件 | 量化/路径 | decode | 来源 |
|---|---:|---|
| **MacBook M4 Max 64GB** | 4-bit | **~15 tok/s**（32s prefill） | @tomgreenwald |
| M3 Ultra | MLX，无 MTP | 13 tok/s（188 prefill） | @CFC3DNC |
| Mac mini M4 Pro 24GB | thinking 模式 | 15.1 tok/s | @mertcobanov |
| MacBook Pro M5 Max | MTPLX（MTP） | 73 tok/s peak | @Youssofal_ |

**对比前任 Qwen3.5/3.6（同是 27B dense，架构不同）：**

| 模型 | M4 Max 64GB decode | 来源 |
|---|---:|---|
| Qwen3.5 27B | **35 tok/s**（81K ctx） | willitrunai |
| Qwen3.6 27B | 13.4 tok/s（262K ctx） | willitrunai |
| Qwen3.8 27B | **~15 tok/s** | vramcalculator |

**规律：Qwen3.6 之后的 hybrid attention 新架构，在 Mac 的 Metal/MLX kernel 上尚未优化，比 Qwen3.5 慢 40–60%。** 这跟之前 terminalbytes 在 M3 Ultra 上测到的"3.8 比 3.6 慢一半（14 vs 28.6）"完全吻合。**结论：Qwen3.8 27B 在 Mac 上的真实裸跑速度就是 13–15 tok/s 这个量级，短期内不会因为换 64GB 而变快——瓶颈是 kernel，不是内存。**

---

## 五、速度实测总表（国内外交叉验证）

### 5.1 Mac 阵营（Qwen3.8 27B，除非注明）

| 硬件 | 路径 | decode | 说明 |
|---|---:|---|
| M4 Max 64GB (546) | 4bit 裸跑 | ~15 tok/s | vramcalculator 实测 |
| M3 Ultra (819) | MLX 裸跑 | 13–14 tok/s | 两家一致 |
| M5 Max | MTPLX MTP | 73 tok/s | MTP 全开峰值 |

### 5.2 显卡阵营（同模型，MTP 开关对照，@sudoingX 配对 A/B 实测）

| 显卡 | MTP off | MTP on | 增幅 |
|---|---:|---:|---:|
| RTX 5090 32GB | 66 | **144** | +118% |
| RTX 4090 24GB | 36 | **75** | +107% |
| RTX 3090 24GB | 41 | **63** | +54% |
| RX 7900 XTX 24GB | 31 | 44 | +42% |
| Ryzen AI Max+ 395 | 11.5 | 23.7 | 翻倍 |

**MTP 是决定 Qwen3.8 速度的头号变量，比换显卡更关键。** 一张 RTX 5090 靠一个 flag 从 66 飙到 144 tok/s。

---

## 六、真实用户反馈汇总（分来源）

### 1. vramcalculator（Qwen3.8 27B 社区实测总表，5 位测试者 6 组数据）
- **量化梯队真相**：短 prompt 下 Q8_0 到 2-bit 全梯队只差 2.9 分（几乎无损）；**但长上下文是照妖镜**——Q6_K/Q4_K_XL/Q4_K_M 在 16K/32K/64K 深度检索全部 45/45 满分，而 IQ3_XXS 掉到 33/45。**结论：4bit 是长上下文的安全线，再往下有暗伤。**
- **reasoning effort 的巨大影响**：thinking 开关值约 30 分（GPQA-diamond 47→80），比量化选择（全梯队仅差 11 分）值 **3 倍**。但 xhigh 会把模型逼进"想到死循环"，low 与 xhigh 在 agentic 任务上得分一样、token 却差 7–11 倍。
- **4bit ≈ 8bit 基准**：superalesha 用 4×3090 跑 67 小时 4800 任务，AWQ INT4 / NVFP4 / GGUF Q4_K_M 得分 ≥ FP8 基线（统计上并列）。**4bit 没有质量损失。**

### 2. kyu.co（M4 Max 128GB，Qwen3.6 27B dense 全路径实测，2026-05）
- **低功耗模式**：没有路径能到 15 tok/s（27B dense）。
- **高功耗模式**：OptiQ serve 和 Ollama coding NVFP4 到 **18–22 tok/s**（短/中轮）。
- **长上下文是灾难**：11K token 的 wall 速度掉到 **2–10 tok/s**（prefill 主导耗时），Ollama 内部 eval 显示 16 tok/s 但用户体感只有 2.8。
- **MoE 完胜**：Qwen3.6 35B-A3B 在同样机器 **50–57 tok/s**，是 27B dense 的 3 倍。
- 结论："长上下文 agent 会话，27B dense 没有赢家，2–10 tok/s。"

### 3. modelfit（Best LLM for M4 Max 64GB 专项，2026）
- M4 Max 64GB 是**"能跑 70B 的笔记本"**——这是 64GB 相对 48GB 的唯一质变点（48GB 跑 27B 也够，但装不下 70B + 上下文）。
- 速度估算表（Ollama Q4_K_M）：Qwen3.6 27B 20–30、35B-A3B 45–60、Llama 3.3 70B Q4 12–18 tok/s。
- 主动散热不降频：有风扇，跑 70B 几小时速度稳定（对比无风扇 Air 20–30 分钟后降频）。
- **14核 vs 16核**：binned 版慢 25%。

### 4. willitrunai（M4 Max 64GB 规格页）
- 546GB/s，MSRP $3,999，171 模型全速、364 兼容。
- 最佳匹配 **Qwen3.6 35B-A3B（44 tok/s）**，而非 27B dense。
- Qwen3.5 27B 35 tok/s vs Qwen3.6 27B 13.4 tok/s——再次印证新架构变慢。

### 5. Ollama 0.19 MLX 引擎（2026-03 发布，对 Mac 的重大利好）
- Ollama 0.19 起在 Apple Silicon 上**原生走 MLX backend**（preview），35B-A3B 的 decode 从 58 → **112 tok/s**、prefill 从 1154 → **1851 tok/s**。
- 只对 MLX/safetensors 格式加速，GGUF 不享受；门槛 32GB+ 内存。
- **含义：Mac 上跑模型的速度，正在被 MLX 引擎 + MTP 双重加速，旧数字会快速过时。**

### 6. 中文社区（百度聚合）
- M4 Max 64GB 跑 Qwen3.8-27B（MLX 4bit）：**23–28 tok/s**（这个数字偏乐观，可能是开了 MTP 或短上下文下的峰值；vramcalculator 裸跑实测是 15）。
- 上下文分水岭：<2K 20–30+、8K 15–20、16K–32K <14、>32K 几乎不可用。
- "64GB 开 32K 上下文勉强能扛，同时跑浏览器+IDE 会吃紧。"

---

## 七、坑点与关键结论

1. **Qwen3.8 在 Mac 上慢是架构问题，不是内存问题**：裸跑 13–15 tok/s，换 64GB 不会变快，只能等 kernel 优化或用 MTP/MoE。
2. **长上下文是 27B dense 的死穴**：11K 以上 wall 速度掉到个位数（prefill 慢）。要做长文档/RAG/agent，27B dense 在 Mac 上体验很差。
3. **64GB 必须买 16 核版**（546GB/s），14 核版（410GB/s）慢 25%，等于白多花内存钱。
4. **MTP 是速度开关**：翻倍；但 Mac 上 Qwen3.8 的 MTP 实测还少（M5 Max MTPLX 73 tok/s 是参考上限）。
5. **量化选 4bit 是安全线**：Q4_K_M / Q4_K_XL 长上下文零损失，再往下（IQ3）有暗伤。
6. **reasoning 默认档即可**：xhigh 会死循环，low 与 xhigh 在 agentic 任务得分一样。

---

## 八、针对达叔场景的建议（M4 Max 64GB vs 36GB）

- **64GB 相对 36GB 的真价值**：不是让 27B 跑更快，而是**解决内存焦虑**——可用内存翻倍、上下文从 24K 提到 32K–64K、还能顺带跑 70B Q4 或 35B-A3B MoE。
- **但 64GB 解决不了 Qwen3.8 的根本问题**：裸跑 13–15 tok/s 是新架构 kernel 未优化导致的，换什么 Mac 都一个样（M3 Ultra 819GB/s 也才 13–14）。
- **如果你坚持跑 Qwen3.8 27B**：
  - 短问答/聊天：15 tok/s 勉强够用，开 MTP 能到 25–30，体验尚可。
  - 长文档/RAG/agent：**不推荐 Mac**，27B dense 长上下文 wall 速度个位数，等 kernel 优化或上显卡。
- **更聪明的替代**：**Qwen3.6 35B-A3B（MoE）** 在同样 M4 Max 64GB 上 45–57 tok/s，质量接近甚至超过 27B dense，是 Mac 上"又快又好"的真正甜点；或等 Qwen3.8 的 MoE 变体。
- **一句话结论**：M4 Max 64GB（务必 16 核版）是"内存从容、但被 Qwen3.8 架构拖累"的档位——**买它解决的是容量，不是 Qwen3.8 的速度**。要速度，要么上 RTX 5090（MTP 后 144 tok/s），要么改用 MoE 模型。

---

## 九、来源清单

1. vramcalculator（Qwen3.8 27B 真实硬件实测总表）：https://vramcalculator.com/qwen3-8-27b-speed/
2. kyu.co（M4 Max 128GB Qwen3.6 27B 全路径实测）：https://kyu.co/posts/qwen36-27b-benchmark-2026-05-11/
3. modelfit（Best LLM for MacBook Pro M4 Max 64GB）：https://modelfit.io/blog/best-llm-macbook-pro-m4-max-64gb/
4. willitrunai（M4 Max 64GB 规格页）：https://willitrunai.com/macs/m4-max-64gb
5. Google AI 汇总（M4 Max 带宽 + 27B 速度）：https://www.google.com/search?q=M4+Max+64GB+memory+bandwidth+410+546+binned
6. Ollama 0.19 MLX 引擎：https://ollama.com/blog/mlx 及 Level Up Coding / Medium 转述
7. Apple 官方 M4 Max 新闻稿（546GB/s）：https://www.apple.com/newsroom/2024/10/apple-introduces-m4-pro-and-m4-max/
8. 中文社区（百度聚合）：M4 Max 64GB 跑 27B 体验、上下文分水岭
9. 前序笔记：`Qwen3.8-27B-M4-Max-36GB-可行性调研-2026-08-31`、`Qwen3.8-27B-本地部署高性价比方案对比与结论 2026-08-31`

> 备注：vramcalculator 的 "M4 Max 64GB ~15 tok/s" 是 Qwen3.8 裸跑实测，willitrunai 的 "Qwen3.5 27B 35 tok/s" 是旧架构数据，中文社区 "23–28 tok/s" 疑似含 MTP 或短上下文峰值。三者口径不同，报告已分别标注，勿直接横向对比。
