
> 检索日期：2026-08-31（周一）
> 承接：微信文章横评、首字延迟对比、M4 Max/M5 Ultra/4090/5090 调研

---

## 一、规格与价格

- **GB10（Grace Blackwell）芯片**：20 核 ARM CPU + 128GB 统一内存 + **273GB/s 带宽（有效约 225）** + 1 PetaFLOP FP4 Tensor Core
- 预装 Ubuntu DGX OS，不支持 Windows；尺寸 150×150×50.5mm / 1.2kg
- **价格**：2026-02 全球涨价 17.5% 至 $4,699；中国区京东实售 **¥33,999**（促销 ~¥32,900）

---

## 二、核心矛盾：prefill 快 / decode 慢

| 阶段 | 表现 | 数据 |
|---|---|---|
| **prefill（首字）** | 强 | 65K 输入 1230 tok/s，TTFT 53 秒（Mac M3 Ultra 要 286 秒，快 5-6 倍） |
| **decode 裸跑** | 弱 | 8-13 tok/s（273GB/s 带宽太低，比 Mac 410/546 还低） |
| decode + DFlash2 | 中 | 34-54 tok/s（代码题），开放文本仅 14-16 tok/s |

- FP8：开放文本 14.6-16.8 tok/s，JSON 59.5 / Python 50.6（DFlash 接受率差异）
- INT8 W8A16 + DFlash block 8：16.8K prefill 984 tok/s，512 连续 18.4 tok/s
- 8 并发 FP8：~62 tok/s aggregate（单流仅 ~8 tok/s）

---

## 三、微信文章完整 Codex 任务实测（关键）

| 配置 | 结果 | 备注 |
|---|---|---|
| NVFP4（W4A4） | **中断** | 860 次工具调用（Mac Q8 仅 287）、51 条失败命令、1 亿 token |
| 官方 FP8 | 完成 7:41:45，82 分 | 比双 A40（5:23，87分）慢 2 小时 |
| INT8 W8A16 | 未完成 | 主机网络掉线 |

**核心问题**：NVFP4 低精度量化让 agent 走弯路（prefill 快被多走的弯路吃掉）；FP8 精度高但 decode 慢。

---

## 四、对比表

| 维度 | DGX Spark | M4 Max 64GB | RTX 5090 | 双 A40 |
|---|---|---|---|---|
| 价格 | ¥34,000 | ~¥2万(二手) | ¥2-3万单卡 | 更贵 |
| 内存 | 128GB | 64GB | 32GB | 96GB |
| 带宽 | 273GB/s | 546GB/s | 1792GB/s | 1008×2 |
| prefill | 1230 tok/s | 200-274 | 4000-6000 | 1180 |
| decode 裸跑 | 8-13 | 13-15 | 66 | 21.8 |
| decode 加速后 | 34-54 | 40-53 | 144 | 21.8 |
| 首字 65K | 53秒 | 286秒 | 极快 | 快 |

---

## 五、结论

- ❌ 跑 Codex/agent 干真活：不推荐（NVFP4 走弯路、FP8 慢、端到端输给双 A40 Q8_0）
- ❌ 日常聊天/快输出：不推荐（decode 裸跑 8-13 tok/s，比 Mac 还慢，¥3.4万不值）
- ✅ 唯一适用：长上下文（128GB）+ CUDA 生态（微调/vLLM/SGLang）+ 预算卡 ¥3.4万、上不了双 A40

**一句话：DGX Spark = 首字延迟单项冠军 + 大内存，但 decode 慢 + 低精度量化坑 + 涨价，偏科、非通用最优解。**

---

## 六、来源

- 微信文章（穷举式横评）：https://mp.weixin.qq.com/s/DrF_Mrq453cfVJwpA9-i5g
- DGX Spark 价格/规格：百度聚合 + Google AI 汇总（2026-08）
- NVIDIA Developer Forums（34-38 tok/s 天花板）、Reddit r/LocalLLM（FP8 单流 8 tok/s）
