
> 检索日期：2026-08-31（周一）
> 承接：36GB / 64GB 两轮可行性调研

---

## 一、成熟度结论

**分两条线：裸跑尚未成熟，MTP 优化方案已相当成熟。**

- Qwen3.8 发布于 2026-08-14，至 8 月底两周半，Mac 生态快速演进。
- 转折点：**Ollama 0.32.13（本周）才正式支持 Qwen3.8 的 MTP**。
- 裸跑（Ollama 默认 GGUF / llama.cpp 无 MTP）：hybrid attention 的 Metal kernel 未优化，仅 **13–15 tok/s**。
- 开 MTP 的专用 runtime 已全面支持 Qwen3.8，速度可到 **48–73 tok/s**，是无损投机解码。

---

## 二、方案排序与原始出处

### 最成熟稳定：Ollama（0.32.13+）
- 模型：`qwen3.8:27b`（GGUF）、`qwen3.8:27b-mlx`（MLX/nvfp4）
- 出处：https://ollama.com/library/qwen3.8
- MTP 支持刚落地，速度潜力尚未完全释放。

### 最优选（有完整实测）：oMLX 原生 MTP
- **Mac Studio M4 Max 实测 48–65 tok/s**，含同日 A/B + benchmark JSON
- 配方仓库：https://github.com/Weschera/Qwen3.8-27B-oMLX-MTP-Mac
- oMLX 本体：https://github.com/jundot/omlx（20k stars，连续批处理 + 分层 KV cache）

### 省心 GUI：MTPLX（v2.10.1，2 倍速无损）
- 官网：https://mtplx.com/
- GitHub：https://github.com/youssofal/MTPLX
- 模型：https://huggingface.co/Youssofal/Qwen3.8-27B-MTPLX-Optimized-Speed
- 实测参考：M5 Max 上 MTPLX 73 tok/s peak
- 明确支持 Qwen 3.8/3.6/3.5/Gemma 4 MTP builds；macOS 14+ / Apple Silicon

### 极致速度：mlx-dspark（3–4 倍无损）
- GitHub：https://github.com/ARahim3/mlx-dspark
- 中文解读：https://www.jdon.com/94010-qwen3-8-27b-mac-3-mlx-dspark.html
- DeepSeek DSpark + z-lab DFlash 投机解码的 MLX 移植

### 基础/官方量化
- mlx-community 官方 4bit：https://huggingface.co/mlx-community/Qwen3.8-27B-4bit
- LM Studio（MLX 后端，GUI，MTP 参数已支持）

---

## 三、关键关系与推荐

- **oMLX 与 MTPLX 同一内核**：MTPLX 作者 Youssof Altoukhi 的 verify-shape Metal kernel，正是 oMLX Lightning MTP 的底层。oMLX = 服务端（OpenAI 兼容 API / 编码 agent），MTPLX = GUI app。
- **推荐**：跑 Qwen3.8 27B 别用 Ollama 默认裸跑（13–15 tok/s）。要接 agent/编码 → oMLX 原生 MTP（48–65 tok/s）；图省心 → MTPLX（2 倍速一键）；追极致 → mlx-dspark（3–4 倍）。Ollama 等 MTP 再熟一两周可回。
