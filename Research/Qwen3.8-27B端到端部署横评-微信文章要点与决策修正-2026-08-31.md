
> 检索日期：2026-08-31（周一）
> 来源：微信文章 https://mp.weixin.qq.com/s/DrF_Mrq453cfVJwpA9-i5g
> 承接：36GB/64GB 调研、首字延迟对比

---

## 一、文章核心结论

穷举式横评三套硬件（M3 Ultra 256GB / 双 A40 48GB×2 / DGX Spark GB10 128GB）× 6 引擎 × 7 量化，最终反直觉结论：

> **65K 输入 prefill 最快的 DGX Spark（比 Mac 快 5 倍），完整 Codex 任务反而输；端到端赢的是最朴素的双 A40 + llama.cpp Q8_0。**

**Agent 真实速度 = 每步多快 × 走多少步 × 链路能否坚持到最后**

---

## 二、关键数据

### Mac 死穴：长输入 prefill 分钟级
- M3 Ultra 256GB（oMLX Q8）：65K 输入 prefill 229 tok/s，**TTFT 286 秒（近 5 分钟）**
- 65K 请求总 293 秒，其中 286 秒在等首 token；decode 从 18→50 tok/s 只省几秒

### DGX Spark：prefill 快 5-6 倍，但 NVFP4 让 agent 走弯路
- 65K prefill：Spark 1230 tok/s（TTFT 53s）vs Mac 220 tok/s（TTFT 298s）
- NVFP4 完整任务 860 次工具调用才中断 vs Mac Q8 仅 287 次——低精度量化让 agent 变笨、多试错

### 双 A40 + Q8_0：端到端赢家
- Q8_0 单卡：prompt 1182.9 tok/s，decode 21.77 tok/s，权重 27.1GB
- 双卡 96GB 扛 25 万 token 长上下文；完整任务墙钟 **5:23:47（最短）**，缓存命中率 98.85%

### 各量化完整任务对照
| 配置 | 墙钟 | 总 Token | 编程分 |
|---|---|---|---|
| 双 A40 Q8_0 | **5:23:47** | 3422万 | 87 |
| M3 Ultra oQ8e-MTP | 6:38:25 | **3380万(最低)** | 92 |
| M3 Ultra MTPLX INT4 | 7:35:02 | 4631万 | 89 |
| DGX Spark NVFP4 | 中断 | 1.008亿 | — |
| DGX Spark FP8 | 7:41:45 | 6885万 | 82 |

---

## 三、对之前决策的修正

1. **"5090 是首字延迟最优解"→ 方向对但漏了关键**：prefill 快 5 倍的 DGX Spark 反而输在 NVFP4 低精度导致 agent 走弯路。
2. **4090（24GB）更谨慎**：24GB 只能 Q4，低精度 + 长上下文 ~16K 封顶双重劣势。
3. **5090（32GB）局限**：32GB 装 Q8_0（27GB）+ KV 就满，25 万 token 长 agent 吃紧。
4. **双 A40（48GB×2）盖章最优**：Q8_0 精度 + 96GB 显存，端到端最短墙钟。
5. **Mac 定位**：短上下文 decode 峰值漂亮（MTPLX 63-95 tok/s），长输入 prefill 分钟级是死穴。

---

## 四、其他易踩坑结论

- 不要用 Aggregate TPS 当单请求速度（Spark 4 并发 90 tok/s ≠ 单请求 90）
- 不要只汇报最有利提示词（JSON 近 60 tok/s，开放文本仅 15）
- 先测正确性再测速度（损坏权重曾造出"100% 接受率"其实是更快犯错）
- 双卡 layer split ≠ P/D 分离（无 P2P 时传输 65-109MB/s，P/D 反而更慢）
- 缓存命中率 97-99% 挡不住一次致命冷 prefill（长任务 20 万 token 后 cache 失效 = 几分钟假死）
- API 兼容/稳定性是性能一部分（MTPLX 峰值快但 Responses API 会中途停；oMLX 慢但能跑完）

---

## 五、最终落点

**跑 Codex/agent 干真活：优先大显存 + 高精度量化（双 A40 或等效，Q8_0，llama.cpp），别盯 tok/s 和单卡 5090。**
