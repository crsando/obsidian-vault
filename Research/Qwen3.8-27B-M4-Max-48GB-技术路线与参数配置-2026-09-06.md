
> 调研日期：2026-09-06（周日凌晨）
> 承接：`Qwen3.8-27B-M4-Max-48GB-部署调研-2026-09-05`（规格/内存/速度结论见那份）
> 本文聚焦：**该走哪条技术路线、每条路线的安装命令、参数设置、针对 48GB 的具体配置**

---

## 一、结论速览（先说答案）

**首选：oMLX + ANE prefill + 原生 MTP。** 这是目前 M4 Max 上 Qwen3.8-27B 唯一有「完整实测 + 现成配方」的路线，实测 **53.3 tok/s（prose）/ 72.1 tok/s（code）**，且是 OpenAI 兼容 API，能直接喂给 Claude Code / OpenCode 这类编码 agent。

**备选梯队：**
- **图省心 + 要 GUI** → MTPLX（和 oMLX 同一个 MTP 内核，一键自动调 depth，2x 左右提速）
- **图生态最省心** → Ollama（0.19+ MLX 引擎 + 0.32.13 MTP，但 MTP 刚落地、速度潜力没 oMLX 释放得彻底）
- **追求极限** → mlx-dspark（DSpark/DFlash 投机解码，math/code 场景 3–4x 无损）
- **要自己写代码/要视觉** → mlx-lm / mlx-vlm（基础，可控性最强）

**一句话**：认真跑研究/编码 agent → oMLX；日常聊天图省事 → MTPLX 或 Ollama。

---

## 二、五条技术路线全景对比

| 路线 | 提速机制 | M4 Max 速度参考 | 安装门槛 | 适合谁 |
|---|---|---:|---|---|
| **oMLX + ANE + MTP** | 原生 MTP(k=3) + 神经引擎加速 prefill | **53.3 prose / 72.1 code**（128GB 实测） | 中（brew + 手装 kernel + 改 JSON） | 编码 agent、追求最快 |
| **MTPLX** | 原生 MTP（自动调 depth） | ~2x（M4 Max 估算 45–55） | 低（brew 一键 / DMG） | 要 GUI、要自动调优 |
| **mlx-dspark** | DSpark/DFlash 投机解码 | 3–4x（math/code）；M4 Pro 24–34，M4 Max 更高 | 中 | 数学/编码极限、愿折腾 |
| **Ollama** | MLX 引擎 + NVFP4 + MTP(0.32.13) | 裸跑 13–15；MTP 后潜力 40+ | 最低 | 生态最省心、多模型常驻 |
| **mlx-lm** | 纯 MLX（无 MTP 默认） | 裸跑 ~15（可自接 MTP） | 低 | 自己写脚本、视觉(VLM) |

---

## 三、方案一：oMLX + ANE prefill + 原生 MTP（首选，完整配方）

**实测来源**：Weschera 仓库，2026-08-21 同机同 prompt 实测（M4 Max，macOS 26）。单流、temperature 0、thinking off、320 token 生成、~2.1–2.4K prompt、3 次取均值。

| 配置 | Prose tok/s | Code tok/s | Prefill 4K tok/s |
|---|---:|---:|---:|
| **oMLX 0.6.3rc2 + ANE + MTP k=3** | **53.3** | **72.1** | **273.7** |
| oMLX 0.6.1 + MTP k=3（上一版） | 48.0 | 65.5 | — |
| oMLX 0.6.3rc2 + MTP k=3，ANE off | 47.9 | 47.9 | — |
| vllm-metal 0.3.0（8-bit，无 MTP） | — | 13.2 | — |

> ⚠️ 注意：这个 53.3 是 **128GB** 版 M4 Max 测的，但 decode 是 memory-bound，**48GB（16 核，同为 546GB/s）带宽完全一样，decode 速度一致**。差别只在 context 能开多大（见下）。

### 3.1 安装步骤（逐条命令）

```bash
# 1. 装 oMLX（brew tap 方式，也可从 Releases 下 .dmg）
brew tap jundot/omlx https://github.com/jundot/omlx
brew install jundot/omlx/omlx

# 2. 拉 oQ4e 量化权重（4-bit affine g64，166 个最敏感 tensor 保留 5-bit，官方原版转换）
hf download Jundot/Qwen3.8-27B-oQ4e-mtp

# 3. 装 ANE prefill kernel（drowzeys 的 DualANE 预编译内核，让神经引擎加速 prefill）
curl -sSLO https://github.com/drowzeys/keys-MAC-oMLX-0.6.1-DualANE-Qwen3.8-27B-Abliterated-oQ4e-MTP/releases/download/v1.0/qwen35_prefill-ane-kernel-omlx0.6.1-py311-mlx0.32.0.tar.gz
tar xzf qwen35_prefill-ane-kernel-omlx0.6.1-py311-mlx0.32.0.tar.gz
SITE=$(python3 -c "import omlx, os; print(os.path.dirname(omlx.__file__))")/custom_kernels/qwen35_prefill
cp qwen35_prefill/_ext.cpython-311-darwin.so "$SITE/"
cp qwen35_prefill/*.dylib "$SITE/"
cp qwen35_prefill/*.metallib "$SITE/"

# 4. 改 ~/.omlx/model_settings.json（见 3.2）

# 5. 重启（设置只在启动时生效）
omlx restart

# 6. 验证：OpenAI 兼容 API（端口用 lsof 查，实测落在 127.0.0.1:8083）
curl http://127.0.0.1:8083/v1/models
```

### 3.2 model_settings.json（关键参数，官方配方原样）

```json
{
  "version": "1.0",
  "models": {
    "Jundot--Qwen3.8-27B-oQ4e-mtp": {
      "qwen35_ane_prefill_enabled": true,
      "qwen35_ane_prefill_sequence_length": 2048,
      "qwen35_ane_prefill_fraction": 0.5,
      "qwen35_ane_prefill_max_layers": 64,
      "qwen35_ane_prefill_dual_ane": true,
      "qwen35_ane_prefill_gdn": false,
      "mtp_enabled": true,
      "mtp_num_draft_tokens": 3,
      "context_window": 262144
    }
  }
}
```

**参数逐条说明：**

| 参数 | 值 | 说明 |
|---|---|---|
| `mtp_enabled` | `true` | 开 MTP 投机解码（速度翻倍的头号开关） |
| `mtp_num_draft_tokens` | `3` | MTP draft depth。实测 k=3 最优（prose）；k=4 只对 code 略好、prose 略降 |
| `qwen35_ane_prefill_enabled` | `true` | 用神经引擎（ANE）加速 prefill，prefill 从 ~83 → ~274 tok/s |
| `qwen35_ane_prefill_sequence_length` | `2048` | ANE 处理 prefill 的分段长度 |
| `qwen35_ane_prefill_fraction` | `0.5` | ANE 承担 prefill 的比例（一半丢给 ANE，一半 GPU） |
| `qwen35_ane_prefill_max_layers` | `64` | ANE 加速的 MLP 层数上限 |
| `qwen35_ane_prefill_dual_ane` | `true` | 双 ANE 通道（M4 Max 有两个神经引擎） |
| `qwen35_ane_prefill_gdn` | `false` | GDN（Gated DeltaNet）不交给 ANE（官方配方关闭） |
| `context_window` | `262144` | ⚠️ 官方配方按 128GB 设的。**48GB 建议改小**（见第六节） |

### 3.3 针对 48GB 的调整

- **`context_window` 从 262144 改成 32768 或 65536**。48GB 可用 ~41–43GB，oQ4e 权重 ~16–18GB，开 262K 的 KV cache（即便 hybrid 架构省）也会挤压余量，长会话易触发 swap。**32K–64K 是 48GB 的安全甜点区**。
- 其余 MTP/ANE 参数**照抄官方配方不用动**，因为它们和带宽相关、和内存容量无关。

### 3.4 oMLX 的额外价值

- **分层 KV cache**（hot 内存层 + cold SSD 层）：agent 长会话时，旧上下文自动落到 SSD，切话题也能复用缓存——对 Claude Code 这类编码 agent 是真刚需。
- **连续批处理** + OpenAI 兼容 API：能同时服务多个客户端。
- 菜单栏管理 + 常驻模型 + 按需换模型。

---

## 四、方案二：MTPLX（省心 GUI，同内核）

MTPLX 作者 Youssof Altoukhi 的 verify-shape Metal kernel 就是 oMLX Lightning MTP 的底层。**本质是「oMLX 的 MTP 内核 + 一个开箱即用的 GUI」。**

```bash
# CLI 方式
brew install youssofal/mtplx/mtplx
mtplx start

# 或 pip：python3 -m pip install mtplx
# 或 DMG：mtplx.com/download
```

**核心命令：自动调优 MTP depth（在你这台机器上实测每个 depth，选最快的）**

```bash
mtplx tune --model <model-or-path> --retune
```

**推荐模型**：`Qwen 3.8 27B Optimized Speed`（4-bit dynamic quant，编码速度好、质量佳）。两个变体：`Bare Speed`（爆发快、质量略低、长编码慢）、`Optimized Quality`（8-bit dynamic，完美质量）。M3 及以后自动选 4-bit，M1/M2 自动选 FP16。

**参数特点**：拒绝采样无损，`temperature=0.6, top_p=0.95` 照常有效，不会悄悄改输出分布。macOS 14+、Apple Silicon。

**速度**：16GB M4 mini 1.6x，M5 Max 2.24x，M4 Max 估算 ~2x（45–55 tok/s）。

---

## 五、方案三：mlx-dspark（极致，3–4x 无损）

DeepSeek 的 DSpark + z-lab 的 DFlash 投机解码，MLX 原生移植。无损（target 逐 token 验证）。

**Qwen3.8-27B 实测（M4 Pro，3 次中位数）**：

| 量化 | math 加速 | code 加速 | chat 加速 | 速度 |
|---|---:|---:|---:|---:|
| 8-bit + DFlash2 | **4.06x** | 4.05x | 2.79x | ~24–34 tok/s |
| 4-bit + DFlash2 | 2.63x | 2.62x | 1.68x | ~25–38 tok/s |

> 注意：这是 **M4 Pro**（带宽 273GB/s）测的，M4 Max（546GB/s）会更快。8-bit 在 M4 Max 48GB 上也能装（Q8 约 29GB，余量 ~12GB），math/code 场景 4x 无损加速很诱人。

```bash
# 安装 + 跑 benchmark（三条 prompt：chat/code/math）
mlx-dspark benchmark --trials 3
```

**适合**：重度数学推理/代码生成、能接受 8-bit 占更多内存、愿意折腾。chat 场景加速比（1.68–2.79x）不如 oMLX 的 MTP 稳定。

---

## 六、方案四：Ollama（生态最省心）

```bash
ollama run qwen3.8            # 默认 GGUF
ollama run qwen3.8:27b-mlx    # MLX/NVFP4 版（0.19+ MLX 引擎）
# 接 agent：
ollama launch claude --model qwen3.8
ollama launch opencode --model qwen3.8
```

**关键点：**
- Ollama **0.19 起** Apple Silicon 走 **MLX 引擎**，只对 MLX/safetensors 格式加速（GGUF 不享受）；NVFP4 量化支持。门槛 32GB+ 内存。
- **0.32.13 才正式支持 Qwen3.8 的 MTP**（本周刚落地），速度潜力还没完全释放，可再等一两周。
- 裸跑（无 MTP）仍是 13–15 tok/s。
- 优势：多模型常驻、生态最全、一行命令接各种 agent。

---

## 七、方案五：mlx-lm / mlx-vlm（基础，可控性最强）

```bash
uv tool install mlx-lm
mlx_lm.server --model "mlx-community/Qwen3.8-27B-4bit"   # OpenAI 兼容，localhost:8080/v1
```

- 纯 MLX，裸跑 ~15 tok/s（默认无 MTP，需自己接 MTPLX/oMLX 内核）。
- **mlx-vlm** 支持视觉（Qwen3.8 原生视觉），喂研报截图/Python 脚本处理时有用。
- 适合：要自己写 Python 推理脚本、要视觉多模态、要做二次开发。

---

## 八、针对 M4 Max 48GB 的统一参数建议

**量化格式：**
- 首选 **oQ4e / Q4_K_M（4-bit）**，长上下文零损失的安全线。不要降到 IQ3 以下（长上下文检索有暗伤）。
- 想更高质量 + 48GB 也放得下：Q6_K（22GB）、Q8_0（29GB），余量仍够。
- 视觉/多模态：mlx-community 4bit（约 15GB + vision projector）。

**MTP depth：**
- **k=3** 是 prose 甜点（oMLX 实测）；MTPLX 用 `mtplx tune` 自动选。

**context 长度（48GB 专属）：**
- 安全甜点 **32K–64K**；别照抄 128GB 的 262144 全开（会挤爆余量触发 swap）。

**采样参数（日常对话 vs 编码 agent 分开）：**

| 场景 | temperature | top_p | reasoning/thinking |
|---|---|---|---|
| 日常聊天/写作 | 0.6–0.7 | 0.95 | off（关 thinking，省 token、防死循环） |
| 编码 agent | 0–0.2 | — | off |
| 需要强推理/数学 | 0.3–0.6 | 0.95 | 默认档（**别开 xhigh**，会逼进死循环；low 和 xhigh 在 agentic 任务得分一样） |

**内存上限（可选）：**
- 现代 macOS 15 默认已允许 GPU 用大部分统一内存；如遇「内存没吃满却报 OOM」，可用 `sudo sysctl iogpu.wired_mem_limit` 调高 GPU wired memory 上限。通常不必动。

---

## 九、针对达叔场景的推荐组合

**你的场景**：研究/长文档/agent + 编码，要性能、要接 Alma/Claude Code，已经用过 Ollama/LM Studio/MLX。

**推荐主路线：oMLX + ANE + MTP（k=3）**
- 装好后就是 OpenAI 兼容 API，直接配给 Claude Code / OpenCode / Alma 当本地模型后端。
- 分层 KV cache 对长 agent 会话是刚需。
- 48GB 把 `context_window` 设 65536 即可。

**轻量副路线：Ollama 常驻（多模型切换 + 一行接 agent）**
- 平时跑 Qwen3.6 35B-A3B（MoE，48GB 上 45–57 tok/s）当日常助手，Qwen3.8 留给 oMLX 跑重活。

**要 GUI 尝鲜：MTPLX**（和 oMLX 同内核，装一个对比下体感）。

**坑点提醒：**
1. oMLX 需要 **macOS 15+（Sequoia）、Python 3.11–3.13**；brew 装的可直接跑，从源码 `pip install -e .` 时不带 native kernel，某些模型族会静默掉到慢路径（需装完整 Xcode 编 Metal kernel）。
2. 旧版 llama.cpp / Ollama / LM Studio 报 `unknown model architecture: 'qwen35'`，先升级 runtime。
3. 所有「晒速度」先问开没开 MTP、关没关 thinking、prompt 多长——裸跑 13–15 tok/s 才是基准。

---

## 十、来源清单

1. Weschera 仓库（oMLX+ANE+MTP 实测配方，含 model_settings.json）：https://github.com/Weschera/Qwen3.8-27B-oMLX-MTP-Mac
2. oMLX 本体（jundot/omlx，分层 KV cache + CLI 配置）：https://github.com/jundot/omlx
3. MTPLX（youssofal，GUI + mtplx tune）：https://github.com/youssofal/MTPLX 、https://mtplx.com
4. mlx-dspark（DSpark/DFlash 投机解码）：https://github.com/ARahim3/mlx-dspark
5. Ollama MLX 引擎博客（0.19，NVFP4）：https://ollama.com/blog/mlx
6. Ollama qwen3.8 模型页：https://ollama.com/library/qwen3.8
7. mlx-community Qwen3.8-27B-4bit（mlx-lm/mlx-vlm 用法）：https://huggingface.co/mlx-community/Qwen3.8-27B-4bit
8. 前序 Obsidian 笔记：《Qwen3.8-27B-Mac部署方案成熟度与最优选-2026-08-31》、《Qwen3.8-27B-M4-Max-48GB-部署调研-2026-09-05》

> 备注：速度数据除标注外均为单流 decode，随 prompt 长度、上下文、runtime 版本浮动。oMLX 的 53.3 tok/s 是 128GB 版 M4 Max 实测；48GB（16 核 546GB/s）decode 速度一致，但 context 需设 32–64K。
