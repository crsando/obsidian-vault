# Qwen3.8-27B 部署于 Mac Studio M3 Ultra 96GB：深度调研报告

调研截止：2026-08-30，UTC+8。本文明确区分官方资料、第三方实测、工程推算和财务假设。

## 一、执行结论

目标型号完整名称是 Qwen3.8-27B，截至 2026-08-30 已存在于 Qwen 官方 Hugging Face、GitHub 和 ModelScope 仓库。它是 27B 稠密语言模型加视觉编码器，不是 Qwen3-30B-A3B，也不是 Qwen3-32B。

对 M3 Ultra 96GB Mac Studio 的判断：

- 可部署性高。4-bit 权重约 16-20GB，装入 96GB 统一内存没有问题。
- 普通路径体验中等。公开普通 4-bit、无 MTP 参考约 12.1-12.3 tok/s。
- 优化路径体验高。目标机器公开 oMLX 记录显示，4-bit oQ4e 加 Lightning MTP 在 4K 上下文为 59.4 tok/s，32K 上下文为 53.9 tok/s。
- 研究实用性高。适合中文/英文资料综合、代码、结构化抽取、图表和文档图片分析、私密资料处理。
- 本地模型没有自动实时联网能力。最新事实、网页、引用和数值计算必须由 Web 工具、数据源和 Python/Polars/DuckDB 支撑。
- 32K 是稳妥默认值。官方 262K 原生和 1M 扩展是模型能力上限，不等于 96GB 机器可以在 1M 上下文下快速、稳定、低延迟运行。
- 单纯省同档 API 钱不划算。按当前 Qwen3.8-27B 市场 API 参考价，轻/中/大三档约省 RMB 29/171/571 token 账单；若单独购买约 RMB 33,000 的机器并按五年折旧，约每月 1.1 亿总 token 才接近回本。

核心判断：购买理由应主要是隐私、离线、长期可用、无额度限制和本地研究流水线，而不是低价 Qwen API 的纯财务套利。

## 二、模型身份与能力

### 2.1 官方模型信息

来源：

- 模型卡：https://huggingface.co/Qwen/Qwen3.8-27B
- 官方仓库：https://github.com/AlibabaCloud-Official/Qwen3.8-27B
- 官方配置：https://huggingface.co/Qwen/Qwen3.8-27B/raw/main/config.json
- ModelScope：https://modelscope.cn/models/Qwen/Qwen3.8-27B

| 项目 | 已核实值 |
|---|---:|
| 语言模型参数 | 27B；HF 参数数 27,781,427,952 |
| 模型类型 | Causal Language Model with Vision Encoder |
| 层数 | 64 |
| 隐藏维度 | 5,120 |
| 层结构 | 16 个 full-attention + 48 个 linear-attention/Gated DeltaNet |
| Full attention | Q=24、KV=4、head dim=256 |
| Linear attention | key heads=16、value heads=48、head dim=128 |
| MTP | 包含训练过的 Multi-Token Prediction head |
| 原生上下文 | 262,144 tokens |
| 扩展上下文 | 最高约 1,000,000 tokens，需 YaRN/RoPE |
| 视觉 | 原生图片、视频理解 |
| 思考 | 默认开启 |
| 推理深度 | xhigh、medium、low |
| 历史推理 | preserve_thinking 默认开启 |
| 许可证 | Apache-2.0 |

官方配置把模型标识为 Qwen3_5ForConditionalGeneration/qwen3_5。混合注意力是它适合长上下文和本地部署的关键：大部分层用线性注意力的固定状态，只有每 4 层中的 1 层使用常规 full attention。

### 2.2 对研究工作的实际价值

适合：

- 研报、论文、新闻、公告的中英文总结、对比和批注。
- 事实、人物、日期、数字、观点和表格字段抽取。
- 图表、截图、扫描文档图片的辅助分析。
- Python、SQL、Polars、DuckDB 数据处理代码。
- 研究提纲、假设树、反方论证、风险清单和 JSON/Markdown 输出。
- 私密笔记、内部资料和本地数据集处理。
- 通过工具完成文件处理和多步分析。

不能独立替代：

- 最新网络信息和实时行情。
- 可信引用和来源核验。
- 财务收益率、估值和时间序列的最终计算。
- 任意复杂且长时间无人值守的 agent。
- 云端模型的 SLA、托管升级和成熟工具生态。

对达叔的资料研究，推荐流程是：MinerU 解析复杂 PDF -> Markdown/表格结构化 -> Qwen3.8 综合分析 -> Polars/DuckDB/Python 数值计算 -> Web/行情数据源补最新事实 -> 保留原始 URL 和页码/段落。

## 三、能力证据与边界

### 3.1 官方 benchmark

以下是 Qwen 官方模型卡的结果，是模型卡自报的能力上限信号，不是 M3 Ultra 96GB 上的 4-bit 实测。

| 能力 | Benchmark | 分数 |
|---|---|---:|
| 终端 agent 编程 | Terminal Bench 2.1 | 73.0 |
| Agent 编程 | SWE-bench Pro | 61.7 |
| 仓库级代码生成 | NL2Repo-Bench | 42.3 |
| 软件工程 | QwenSWEBench | 79.0 |
| 长流程办公 | CoWorkBench | 70.7 |
| 指令遵循 | IFBench | 79.5 |
| 科学推理 | GPQA Diamond | 89.2 |
| 综合困难问题 | HLE | 30.8 |
| 竞赛编程 | LiveCodeBench v6 | 90.3 |
| 电脑操作 | OSWorld-Verified | 84.3 |
| 浏览器操作 | WebArena-Verified | 64.8 |
| 多模态软件工程 | SWE-MM | 38.6 |
| 文档理解 | OmniDocBench 1.5 | 91.1 |
| 图表分析，带代码解释器 | CharXiv (RQ) | 90.2 |
| 视觉数学，带代码解释器 | MathVision | 94.6 |

限制在于：测试使用特定 harness、prompt、采样温度和上下文；部分对比对象是闭源模型；量化、工具解析器和失败重试都会改变实际结果。OpenRouter 页面另列 Artificial Analysis 来源的 GPQA 90.5%、HLE 33.9%、AA-LCR 77.3%、GDPval-AA 52.1% 和 SciCode 44.7%，可以作为第二组市场侧信号，但不是个人工作流保证。

### 3.2 Thinking 影响

默认 xhigh thinking 会增加 reasoning token、等待时间和上下文占用；preserve_thinking 会把历史推理带入后续请求。独立 PDF 摘要建议关闭历史思考保留；复杂多步任务再保留。官方也提醒，降低 reasoning effort 不一定线性降低总耗时，因为失败和重试可能反而增加总 token。

## 四、M3 Ultra 96GB 硬件适配

Apple 官方规格：https://support.apple.com/en-us/122211

Apple 发布稿：https://www.apple.com/newsroom/2025/03/apple-unveils-new-mac-studio-the-most-powerful-mac-ever/

目标配置是 28 核 CPU（20 性能核加 8 能效核）、60 核 GPU、32 核 Neural Engine、96GB 统一内存、819GB/s 带宽、1TB SSD。最大持续系统功率为 480W。

Apple 宣称 M3 Ultra 在 LM Studio 中运行数百亿参数以上 LLM 时，token generation 最多比 M1 Ultra 快 16.9 倍。这是 Apple 的产品级对比，不是 Qwen3.8-27B 的速度承诺。

### 4.1 文件体积

| 权重/量化 | 公开体积或理论值 | 证据类型 |
|---|---:|---|
| 官方 BF16 Transformers | 约 55.6GB | HF 文件/参数元数据 |
| mlx-community 4-bit | 约 16.1GB | HF API 文件体积 |
| Unsloth UD-Q4_K_M GGUF | 约 16.46GB | HF API 文件体积 |
| Unsloth UD-Q4_K_XL GGUF | 约 17.56GB | HF API 文件体积 |
| Unsloth UD-Q5_K_M GGUF | 约 19.77GB | HF API 文件体积 |
| Unsloth UD-Q6_K GGUF | 约 21.98GB | HF API 文件体积 |
| Unsloth UD-Q8_K_L GGUF | 约 28.05GB | HF API 文件体积 |
| Jundot oQ4e FP16-MTP | 约 17.9GB | 模型页 |
| Youssofal oQ4e MTP | 约 20.4GB | 模型页 |

4-bit 权重装入 96GB 没有问题；运行时还要加视觉编码器、MTP、Metal/MLX workspace、KV cache、GDN 状态、操作系统和其他应用。

### 4.2 KV cache 推算

根据官方 config，常规 full attention 只有 16 层：

~~~text
16 层 × 2(K/V) × 4 KV heads × 256 head_dim × 2 bytes(BF16)
≈ 65,536 bytes/token
~~~

仅计算 conventional full-attention KV，不包括 GDN 状态、激活、视觉 token 和运行时开销：

| 上下文 | BF16 KV only | FP8 KV only | 4-bit KV only |
|---:|---:|---:|---:|
| 262K | 约 16GiB | 约 8GiB | 约 4GiB |
| 1M | 约 61GiB | 约 30.5GiB | 约 15.3GiB |

所以正常工作区建议 8K-32K；32K-64K 是长文档区；64K-128K 需要先测；262K/1M 只作为实验目标。

## 五、目标机器性能证据

### 5.1 精确 96GB/60 核的 4K 实测

来源：https://omlx.ai/benchmarks/performance/ldaoefgn

硬件为 M3 Ultra 60c、96GB；oMLX 0.6.2、macOS 15.3.2；Qwen3.8-27B-oQ4e-mtp 4-bit；Lightning MTP 开启、3 个 draft token、Qwen ANE prefill 开启。

- Prompt processing：376.7 tok/s。
- Generation：59.4 tok/s。
- TTFT：10,874ms。
- Peak footprint：43.14GB。
- MLX active peak：23.62GB。
- System used peak：64.88GB。
- Thermal：Nominal。

这说明优化路径在目标硬件上确实有很好的交互速度，但 4K 输入本身仍可能需要约 10.9 秒才开始输出。

### 5.2 精确 96GB/60 核的 32K 实测

来源：https://omlx.ai/benchmarks/performance/ol9dvoa6

- Prompt processing：290.0 tok/s。
- Generation：53.9 tok/s。
- TTFT：113,010ms，约 113 秒。
- MLX peak：21.7GB。
- System used peak：73.05GB。
- Thermal：Nominal。
- Batch 1：51.4 tok/s；batch 2：68.1 tok/s；batch 4 总吞吐 127.3 tok/s。

32K 输入的 TTFT 约 113 秒是研究工作流必须正视的数字。重复使用同一前缀时，prompt cache 可以明显改善体验。

### 5.3 普通无 MTP 基线

来源：https://omlx.ai/benchmarks/performance/zj1o8yi2 和 https://omlx.ai/benchmarks/performance/vebz51hi

M3 Ultra 80c/512GB 上，普通 4-bit Qwen3.8-27B、无 MTP 的公开记录约为 12.1-12.3 tok/s，1K-16K 上下文均在这个量级。

这不是严格 A/B，因为硬件 GPU 核数、模型 artifact、运行时版本和设置不同。严谨表述是：普通 4-bit 路径约十几 tok/s；专门 oQ4e/MTP 路径在目标机器公开记录中约 50-60 tok/s。

### 5.4 高精度参考

来源：https://omlx.ai/benchmarks/performance/1gfqfbbm

M3 Ultra 60c/96GB 上，oQ8e/fp16-MTP 在 64K 上下文记录为 23.2 tok/s、40.6GB peak。8-bit 可作为质量对照组，但不宜作为日常默认。

## 六、部署路线与稳定性

| 路线                            | 建议          | 说明                                       |
| ----------------------------- | ----------- | ---------------------------------------- |
| MLX-VLM + mlx-community 4-bit | 先做 baseline | Apple Silicon 原生，图片路径明确                  |
| oQ4e/MTP + oMLX               | 日常性能版       | OpenAI/Anthropic API、batch、cache、MTP、VLM |
| llama.cpp + Unsloth GGUF      | 兼容性回退       | Metal、GGUF 和生态最广                         |
| LM Studio                     | GUI 快速验证    | 方便锁模型、改上下文和做对照                           |
| Ollama                        | 简单文本体验      | tag、量化和默认参数会变化                           |
| vLLM/SGLang                   | 不作为 Mac 首选  | 更适合 Linux/NVIDIA 服务端                     |
|                               |             |                                          |

来源：

- MLX-LM：https://github.com/ml-explore/mlx-lm
- llama.cpp：https://github.com/ggml-org/llama.cpp
- oMLX：https://github.com/jundot/omlx
- Unsloth：https://unsloth.ai/docs/models/qwen3.8

截至调研日，高性能 oMLX 路径仍有风险：

- issue 3233 报告 ANE prefill 加 Lightning MTP 在长 agent 会话中可能出现上下文腐化、空 completion 和工具调用损坏；关闭 ANE、保留 MTP 的一次 8 步会话约 41 tok/s 并完成。
- issue 3242 报告不同 Qwen 家族模型同进程加载时的 class-wide patch 碰撞，另一个模型可能持续失败，重启服务才能恢复。

推荐 profile：MTP 开启，长 agent 先关闭 ANE prefill，context 先设 32K，固定回归任务后再升级 runtime。

## 七、三档工作量与本地处理时间

假设输出 token 包括 reasoning token，输入/输出比例为 5:1：

| 档位 | 月输入 | 月输出 | 月总 token | 按目标机 4K 实测粗算 |
|---|---:|---:|---:|---:|
| 少量 | 5M | 1M | 6M | 约 8.4 小时 |
| 中等 | 30M | 6M | 36M | 约 50.2 小时 |
| 大规模 | 100M | 20M | 120M | 约 167.3 小时 |

处理时间使用 376.7 PP tok/s 和 59.4 TG tok/s 粗算。若主要是 32K 长文档，按 290/53.9 粗算，大约为 9.9/59.7/198.9 小时。真实时间还会受 cache、图片/视频、工具调用、失败重试和并发影响。

## 八、API 价格与月度 token 节省

### 8.1 价格口径

Qwen3.8-27B 开源 checkpoint 的官方同名按量 API 价格没有在本次核查中确认到独立公开价目；官方模型卡写 Qwen Cloud hosted version 将提供。使用 OpenRouter 当前市场参考：

来源：https://openrouter.ai/qwen/qwen3.8-27b

- Qwen3.8-27B：输入 $0.25/M，输出 $3.00/M。
- 页面加权实际价格约为输入 $0.2168/M、输出 $2.927/M。

对照价：

| 模型 | 输入 | 输出 | 来源 |
|---|---:|---:|---|
| Qwen3.8-27B | $0.25/M | $3.00/M | OpenRouter |
| Qwen3.8 Flash | $0.15/M | $0.47/M | https://openrouter.ai/qwen/qwen3.8-flash |
| Claude Sonnet 5 | $2/M | $10/M | https://openrouter.ai/anthropic/claude-sonnet-5 |
| GPT-5.5 | $5/M | $30/M | https://openrouter.ai/openai/gpt-5.5 |
| Gemini 3.1 Pro | $2/M | $12/M | https://openrouter.ai/google/gemini-3.1-pro-preview |

汇率采用 2026-08-28 的 1 USD = RMB 6.7209：https://api.frankfurter.app/latest?from=USD&to=CNY

QwenCloud Token Plan 是 credits 订阅，不是单纯文本 token：Lite 约 $8/月（限时 $6）、Standard 约 $25（限时 $18）、Pro 约 $80（限时 $68），分别是 2,500/10,000/40,000 credits 每 7 天，并包含多模型和工具。来源：https://docs.qwencloud.com/token-plan/overview

### 8.2 毛 API 账单，也就是本地可避免的 token 费用

公式：

~~~text
API 月成本 = 输入_M × 输入单价 + 输出_M × 输出单价
~~~

| 使用量 | 总 token | Qwen3.8-27B | Qwen3.8 Flash | Claude Sonnet 5 | GPT-5.5 | Gemini 3.1 Pro |
|---|---:|---:|---:|---:|---:|---:|
| 少量：5M+1M | 6M | RMB 28.6 | RMB 8.2 | RMB 134.4 | RMB 369.6 | RMB 147.9 |
| 中等：30M+6M | 36M | RMB 171.4 | RMB 49.2 | RMB 806.5 | RMB 2,217.9 | RMB 887.2 |
| 大规模：100M+20M | 120M | RMB 571.3 | RMB 164.0 | RMB 2,688.4 | RMB 7,393.0 | RMB 2,957.2 |

如果比较的是同一个 Qwen3.8-27B API，本地部署每月少买 token 的毛金额约为 29/171/571 元。这不是扣除硬件后的净收益，也不代表本地模型与 GPT/Claude 完全等价。

### 8.3 输出比例敏感性

按 100M 总 token：

- 输入/输出 9:1：约 RMB 353。
- 输入/输出 4:1：约 RMB 538。
- 输入/输出 1:1：约 RMB 1,092。

应从 API usage 导出 input、output、reasoning、cache-hit 和 retry，而不是只看可见回答字数。

## 九、硬件成本、回本和购买判断

### 9.1 机器价格

2025 年中国上市报道列出 M3 Ultra 96GB/1TB 起售价 RMB 32,999：https://www.sohu.com/a/867445923_120914498

同配置 2025 年底零售促销也记录为 RMB 32,999：https://www.smzdm.com/p/165533658/

2026 年实际购买可能是清库存、二手或涨价，应以实际成交价替换。

### 9.2 所有权假设

若专门为了本地模型购买，按 RMB 32,999、五年直线折旧、不计残值：

- 设备折旧约 RMB 550/月。
- 电费通常小于折旧。Apple 只给出 480W 最大持续功率；目标 Qwen3.8 96GB 实测页没有墙上功耗。
- 参考同类 M3 Ultra 本地模型，生成窗口常见约 38-76W，但不能冒充目标机实测。
- 活跃推理电费通常是每月十几元级；常开、满载、同时跑多个任务则更高。
- 为了保守，把折旧、电费、维护取 RMB 600/月。

### 9.3 同模型 API 的净结果

| 档位 | 毛 API 节省 | 扣 RMB 600 固定成本后的近似净值 |
|---|---:|---:|
| 少量 | RMB 28.6 | -RMB 571.4 |
| 中等 | RMB 171.4 | -RMB 428.6 |
| 大规模 | RMB 571.3 | -RMB 28.7 |

4:1 输入/输出时：

~~~text
(4 × $0.25 + 1 × $3.00) / 5 = $0.80/M total tokens
$0.80 × 6.7209 ≈ RMB 5.38/M total tokens
RMB 600 / 5.38 ≈ 111.6M total tokens/month
~~~

回本阈值约为：输入密集 9:1 时 170M/月；平衡 4:1 时 112M/月；输出/思考密集 1:1 时 55M/月。

若 Mac Studio 本来就承担日常工作，不能把全部折旧都计入模型；此时本地推理的增量成本主要是电费、存储、维护和你的时间，经济性会明显改善。

### 9.4 最终购买建议

建议购买/部署，如果你重视本地隐私、会长期使用、机器还承担其他工作，或当前主要替代的是 Claude/GPT 级高价 API；这时本地 Qwen3.8 可以成为高频研究工人，云端模型负责最终复核和困难任务。

不建议仅为省 Qwen token 购买，如果当前每月只有几百万到几千万 token，或大部分任务可以用 Qwen3.8 Flash/低价 API 完成。轻度和中度下，单看 token 账单无法支持 RMB 33,000 的硬件投资。

推荐落地配置：标准 MLX 4-bit 做 baseline；oQ4e/MTP 加 oMLX 做性能版；32K context；长 agent 先关闭 ANE prefill；保留 llama.cpp/GGUF 作为回退；MinerU 加 Polars/DuckDB 加 Web/数据源负责完整研究链路。

## 十、来源清单

1. Qwen 官方模型卡：https://huggingface.co/Qwen/Qwen3.8-27B
2. Qwen 官方仓库：https://github.com/AlibabaCloud-Official/Qwen3.8-27B
3. Qwen 官方 config：https://huggingface.co/Qwen/Qwen3.8-27B/raw/main/config.json
4. MLX 4-bit：https://huggingface.co/mlx-community/Qwen3.8-27B-4bit
5. Unsloth：https://unsloth.ai/docs/models/qwen3.8
6. Apple specs：https://support.apple.com/en-us/122211
7. Apple launch：https://www.apple.com/newsroom/2025/03/apple-unveils-new-mac-studio-the-most-powerful-mac-ever/
8. oMLX 96GB/4K：https://omlx.ai/benchmarks/performance/ldaoefgn
9. oMLX 96GB/32K：https://omlx.ai/benchmarks/performance/ol9dvoa6
10. oMLX 无 MTP 参考：https://omlx.ai/benchmarks/performance/zj1o8yi2
11. oMLX 无 MTP 参考二：https://omlx.ai/benchmarks/performance/vebz51hi
12. oMLX 高精度参考：https://omlx.ai/benchmarks/performance/1gfqfbbm
13. oMLX 项目：https://github.com/jundot/omlx
14. oMLX ANE/MTP 稳定性 issue：https://github.com/jundot/omlx/issues/3233
15. oMLX 多模型稳定性 issue：https://github.com/jundot/omlx/issues/3242
16. MLX-LM：https://github.com/ml-explore/mlx-lm
17. llama.cpp：https://github.com/ggml-org/llama.cpp
18. OpenRouter Qwen3.8-27B：https://openrouter.ai/qwen/qwen3.8-27b
19. OpenRouter Qwen3.8 Flash：https://openrouter.ai/qwen/qwen3.8-flash
20. OpenRouter Claude Sonnet 5：https://openrouter.ai/anthropic/claude-sonnet-5
21. OpenRouter GPT-5.5：https://openrouter.ai/openai/gpt-5.5
22. OpenRouter Gemini 3.1 Pro：https://openrouter.ai/google/gemini-3.1-pro-preview
23. QwenCloud Token Plan：https://docs.qwencloud.com/token-plan/overview
24. USD/CNY：https://api.frankfurter.app/latest?from=USD&to=CNY
25. 中国上市价格报道：https://www.sohu.com/a/867445923_120914498
26. 零售促销记录：https://www.smzdm.com/p/165533658/
