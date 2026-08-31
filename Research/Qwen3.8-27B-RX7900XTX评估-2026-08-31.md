
> 检索日期：2026-08-31（周一）
> 承接：M4 Max/M5 Ultra/4090/5090/双A40/DGX Spark 调研

---

## 一、规格与价格

- RDNA 3（Navi 31），96 CU / 6144 流处理器，24GB GDDR6，**960GB/s 带宽**（+96MB Infinity Cache），TGP 355W
- **价格（2026-08）**：全新渠道 7800–9400 元，**二手 4200–4800 元**（已停产，质保归零，警惕锁 16GB 的工程样卡）
- 关键：带宽 960 追平 4090 的 1008，显存同样 24GB，价格仅 4090 的 1/3

---

## 二、实测速度（Qwen3.8 27B）

| 阶段 | 速度 | 来源 |
|---|---|---|
| prefill（ROCm） | 600–800 tok/s | Reddit + Google 汇总 |
| decode 裸跑（无 MTP） | 30–45 tok/s | Google 汇总 / vramcalculator |
| decode + MTP | 55–75 tok/s | Reddit r/ROCm（5/8 单卡 75；8/19 XT 版 55-60；8/20 双卡 69） |
| 极限优化（BridgeSpec） | 106–120 tok/s | Reddit 社区 |

---

## 三、最大坑：软件栈（无 CUDA）

| 维度 | ROCm | Vulkan |
|---|---|---|
| prefill | 快（600-800） | 慢 |
| decode | 稳定但偶尔喂不满 GPU | 小上下文峰值略高 |
| 系统 | Linux 才顺 | Windows 省心 |
| 功耗 | 长上下文 2 倍 | 省电 |

- Windows 跑 ROCm 折腾，普通用户走 Vulkan（llama.cpp/LM Studio）但 prefill 慢
- 工具生态二等公民：vLLM/SGLang 优先 CUDA，ROCm 慢半拍或缺失
- 社区共识：4090 赢 CUDA 生态+吞吐，7900 XTX 赢每 GB 价格；4090 实际 AI 负载快 1.5-1.8x（多为软件差距）

---

## 四、对比表

| 维度 | 7900 XTX | 4090 | 5090 | M4 Max 64GB |
|---|---|---|---|---|
| 显存/内存 | 24GB | 24GB | 32GB | 64GB |
| 带宽 | 960GB/s | 1008GB/s | 1792GB/s | 546GB/s |
| prefill | 600-800 | 700-900 | 4000-6000 | 200-274 |
| decode MTP | 55-75 | 75 | 144 | 40-53 |
| 价格 | 二手 4200-4800 | 1.2-1.8万 | 2-3万 | ~2万(二手) |
| 软件栈 | ROCm/Vulkan(坑) | CUDA(成熟) | CUDA(成熟) | MLX(成熟) |
| 长上下文 | ~16K 封顶 | ~16K 封顶 | ~32K | 32-64K |

---

## 五、结论

- ✅ 预算党 + 愿折腾（Linux/ROCm 或 Windows Vulkan）+ 短/中上下文：性价比之王，二手 4200-4800 拿接近 4090 的 decode
- ❌ 要省心 / 跑端到端 agent：24GB 只能 Q4（长上下文 ~16K + 低精度走弯路隐患）+ ROCm 生态二等公民，一路踩坑

**一句话：7900 XTX = 用软件栈的折腾换 1/3 价格，硬件底子能打（960GB/s+24GB），但 ROCm 生态决定它适合懂行肯折腾的预算党，不适合要稳跑长 agent 的场景。**

---

## 六、来源

- 规格/价格：百度聚合（SMZDM/太平洋/中关村/快科技，2026-06~08）
- 实测：Reddit r/ROCm（5/8 Qwen3-27B MTP 75 tok/s；8/19 Qwen3.8 IQ4_XS 600-700 prefill/55-60 decode；8/20 双卡 69）、Google AI 汇总、vramcalculator
- 4090 对比：kunalganglani/convly/compareaihardware 等（2026）
