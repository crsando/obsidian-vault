
> 检索日期：2026-08-31（周一）
> 承接：36GB / 64GB 调研、部署方案成熟度

---

## 一、核心原理

**首字延迟（TTFT）= prefill 阶段 = 算力密集（compute-bound），与显存带宽基本无关。**

- decode（生成）是 memory-bound：速度 ≈ 带宽 ÷ 权重字节数。
- prefill（首字）是 compute-bound：由 GPU 算力（FLOPs）决定，量化帮不上忙。
- Mac GPU 算力天生弱 → 首字延迟是 Mac 通病，换任何 Mac（含 M5 Ultra）都治不好。

---

## 二、对比表

| 维度 | M5 Ultra 96GB | RTX 4090 24GB | RTX 5090 32GB |
|---|---|---|---|
| 内存/显存 | 96GB 统一内存 | 24GB GDDR6X | 32GB GDDR7 |
| 内存带宽 | 1.2TB/s | 1008GB/s | 1792GB/s |
| 算力（决定首字延迟） | 弱 | 强 | 最强（Blackwell FP4/FP8） |
| **首字延迟 TTFT（4K prompt）** | **约 7–15 秒** | **约 1.1–1.5 秒** | **约 0.3–0.4 秒** |
| prefill 速度 | ~200–500 tok/s | ~700–900 tok/s | ~4000–6000 tok/s |
| decode（MTP 开） | 48–80 tok/s | ~75 tok/s | ~144 tok/s |
| 长上下文 | 强（32K–256K） | 弱（~16K 封顶） | 中（~32K） |
| 参考价格 | ¥46,999（整机） | ~¥1.2–1.8万（单卡） | ~¥2–3万（单卡） |

---

## 三、结论

1. **解决首字延迟唯一解 = N 卡，首选 RTX 5090**：首字 0.3–0.4 秒，decode 144 tok/s，双料冠军，但 32GB 显存限长上下文。
2. **M5 Ultra 96GB 解决的是容量 + decode，不是首字延迟**：prefill 仍秒级（Mac 通病），花 ¥46,999 首字该等还是等。
3. **RTX 4090 性价比之选**：首字 1 秒出头、decode 75 tok/s，但 24GB 显存长上下文 ~16K 封顶。

---

## 四、来源

- M5 Ultra 96GB 规格/价格（国行 ¥46,999、1.2TB/s）：百度聚合 + 中文社区（2026-08 末）
- RTX 5090/4090 prefill/TTFT/decode：Google AI 汇总 + Reddit r/LocalLLaMA（2026-08-22，262K context vLLM 实测）
- M4 Max prefill 基线（200–274 tok/s）：oMLX 官方 benchmark + Weschera 配方
- 前序：`Qwen3.8-27B-M4-Max-36GB-可行性调研`、`Qwen3.8-27B-Mac部署方案成熟度与最优选`
