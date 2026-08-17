---
title: 纯日盘 Tick Volume Profile 支撑阻力识别与有效性验证
created: 2026-08-17
tags:
  - 期货
  - 量化
  - MarketProfile
  - Research
status: developing
---

# 纯日盘 Tick Volume Profile 支撑阻力识别与有效性验证

## 研究目标

目标是从国内期货日内行情中提取可检验的 support/resistance zone，而不是只画出视觉上“看起来有效”的水平线。

整个问题分成三层：

1. 如何从 tick 数据构建可靠的 Volume Profile。
2. 如何对 Profile 进行平滑并识别稳定的 VPOC、VAH/VAL、HVN、LVN 和 profile edge。
3. 如何用严格的样本外事件定义，判断这些 zone 是否比随机价格区或简单基线更有效。

> [!warning]
> 当前只有 `20260109` 一个交易日。现有结果只能验证数据处理和事件定义是否闭环，不能证明策略有效，也不能估计稳定胜率。

## 数据背景

原始行情约每 500ms 一条，是 CTP 风格的行情快照，不是真正逐笔成交：

```text
TradingDay
InstrumentID
UpdateTime
UpdateMillisec
LastPrice
Volume          # 当日累计成交量
Turnover        # 当日累计成交额
OpenInterest
BidPrice1 / AskPrice1
```

项目 K 线规范要求：

- 时间区间统一左闭右开。
- tick 的 `Volume/Turnover` 是交易日累计值。
- K 线中的 `volume/turnover` 是区间增量。
- 第一笔有效 tick 只建立累计值基准。
- 计数器回退时负差截为 0。
- 非交易时段行情不更新差分基准。
- 不补不存在的空分钟。

## 为什么选择 Tick Volume Profile，而不是 1min Volume Profile

1min Bar 只有：

```text
OHLC + 该分钟总成交量
```

如果一分钟的价格范围是 `[3200, 3210]`，成交量是 1000 手，仅凭 1min 数据无法知道成交量具体分布在哪些价位。只能人为选择：

- 全部分配到 close；
- 均匀分配到 `[low, high]`；
- 围绕 HLC3 或估算 VWAP 做 Kernel 分配。

这些假设会制造或抹掉 HVN/LVN。

Tick 快照虽然仍不是逐笔成交，但可以计算：

```text
delta_volume   = max(Volume[t] - Volume[t-1], 0)
delta_turnover = max(Turnover[t] - Turnover[t-1], 0)
```

然后把两次快照之间新增的 volume 分配到 LastPrice 或 interval VWAP。相比 1min，价格分辨率和成交路径信息都更高。

## 第一阶段范围：只研究纯日盘品种

暂时排除夜盘，以减少跨自然日、交易日归属和夜盘/日盘 Profile 拼接造成的干扰。

### 确认的纯日盘品种池

| 交易所 | 品种 |
|---|---|
| CFFEX | IF IC IH IM T TF TS TL |
| DCE | BB FB JD LH LG |
| CZCE | AP CJ JR LR PK PM RI RS SF SM WH |
| SHFE/INE | WR EC |
| GFEX | SI LC PS PT PD |

`PL` 和 `RR` 被明确排除，因为 `20260109` 的源数据在 21:00 后存在真实 `delta_volume > 0`，不能按旧印象归为纯日盘。

### 代表合约选择

不能把所有月份与“主力连续”同时作为独立样本，否则同一市场行情会被重复统计。

每个品种只选一个具体合约：

```text
representative_contract =
    argmax(上午有效时段内 sum(delta_volume))
```

只使用上午数据选择合约，避免泄漏下午信息。主力连续文件只作结果对照，不参与统计。

默认流动性过滤：

```text
total morning volume >= 1000
nonzero delta-volume updates >= 1000
active-minute coverage >= 60%
unique traded price bins >= 8
```

当前最终保留 21 个代表合约：

```text
AP605 CJ605 ec2604 IC2603 IF2603 IH2603 IM2603
jd2603 lc2605 lh2603 pd2606 PK603 ps2605 pt2606
SF603 si2605 SM603 T2603 TF2603 TL2603 TS2603
```

`LG` 在改为只用上午流动性后，因为 nonzero updates 不足 1000 被排除。

## Tick 标准化

只对已经通过价格、时间顺序和交易时段检查的 tick 做差分：

```text
delta_volume   = max(volume[t] - volume[t-1], 0)
delta_turnover = max(turnover[t] - turnover[t-1], 0)
```

记录但不静默隐藏的质量问题：

```text
VOLUME_RESET
TURNOVER_RESET
LONG_GAP
VWAP_FALLBACK_LTP
```

标准化输出字段：

```text
trading_day
instrument
event_time
millis_of_day
session
last_price
volume
turnover
delta_volume
delta_turnover
interval_vwap
open_interest
gap_millis
quality_flags
```

## 两套 Volume Profile

### LTP Profile

```text
profile_ltp[round_to_tick(LastPrice)] += delta_volume
```

优点：直接、稳定，最接近行情快照终点。

缺点：如果 500ms 内发生多笔不同价格的成交，全部 volume 会被压到最后一个价格。

### Interval-VWAP Profile

```text
interval_vwap =
    delta_turnover / (delta_volume * turnover_divisor)

profile_vwap[round_to_tick(interval_vwap)] += delta_volume
```

需要区分：

- `contract_multiplier`：合约经济价值乘数；
- `turnover_divisor`：源行情从成交额恢复价格时所需的除数。

多数交易所二者相同。当前郑商所源数据的 Turnover 已按价格口径归一，因此显式使用：

```text
turnover_divisor = 1
```

如果 interval VWAP 非有限、非正，或者距 LastPrice 超过配置阈值，则回退到 LastPrice，并标记 `VWAP_FALLBACK_LTP`。

这样可以保证：

```text
sum(Profile_LTP) == sum(Profile_VWAP) == morning delta volume
```

## 为什么需要平滑

Raw Profile 以一个 `tick_size` 为 bin，容易出现大量单格尖峰：

- 500ms 快照把区间成交量压到一个 LastPrice；
- 低流动性品种的成交档位稀疏；
- 单笔大成交可能形成偶发局部峰值；
- 直接找局部最大值会产生过多假 HVN/LVN。

但只选一个平滑参数也不可靠。平滑带宽稍微改变，峰谷可能移动、消失或合并。

因此保留 raw Profile，同时构造多尺度 Gaussian smoothing：

```text
sigma = 0 / 1 / 2 / 4 ticks
```

要求：

- 价格 bin 始终是一个 tick；
- Gaussian convolution 后重新归一化；
- 所有尺度严格保持总 volume；
- raw Profile 用于标准 VPOC、VAH、VAL；
- smoothed Profile 只用于结构检测。

## Profile 节点定义

### VPOC

Raw LTP Profile 中 volume 最大的价格 bin。

如果最大值形成连续 plateau，保留完整 plateau zone，并用 volume-weighted center 表示中心。

如果存在多个不连续的同高峰，不能把它们跨过中间低谷合并成一个 plateau。应选择最接近 Profile volume center 的那个峰，平手时采用确定性的价格顺序。

### Value Area、VAH、VAL

默认 Value Area 覆盖 70% raw volume。

从 raw VPOC 开始，比较左右相邻 bin 的 volume，优先加入 volume 更高的一侧，直到累计达到目标。

```text
VAH = Value Area 上边界
VAL = Value Area 下边界
```

### Persistent HVN

各尺度分别检测局部 peak，并计算：

```text
peak prominence
half-prominence width
node volume share
```

初始门槛：

```text
prominence share >= 0.25% total profile volume
至少在 3/4 个尺度出现
跨尺度中心偏移 <= 2 ticks
LTP 与 VWAP Profile 中距离 <= 2 ticks
```

只有跨尺度、跨方法都稳定的节点才称为 persistent HVN。

### Persistent LVN

LVN 只在两个相邻 persistent HVN 之间寻找。

每个 smoothing scale 都独立寻找 LTP/VWAP valley，并要求：

```text
LTP/VWAP valley 距离 <= 2 ticks
valley depth >= 25%
至少在 3 个尺度稳定
```

不能用两侧 HVN 的 persistence 冒充 LVN 自身的 persistence。

### Profile Edge

Raw Profile 最低和最高的有成交价格边界。

它们可能表示价格极端或分布截断，但其经济含义和 HVN/LVN 不同，必须单独标记。

## 不同节点的市场含义

不要把所有节点预先永久标记成 support 或 resistance。

```text
VPOC / HVN center：accepted value，常表现为价格磁铁或均值回归目标
HVN outer edge：balance boundary，可能产生反应
VAH / VAL：value boundary
LVN：rejection / fast-travel zone
Profile edge：极端接受度边界
```

同一个 zone 的角色取决于价格接近方向：

```text
价格从 zone 上方进入 -> candidate support
价格从 zone 下方进入 -> candidate resistance
```

LVN 尤其具有双重性质：

- 从外部首次接近时可能 rejection；
- 一旦被接受并进入，可能快速穿越到下一个 HVN。

## Zone，而不是假精确价格线

每个候选保存：

```text
center
lower
upper
node_types
source
available_time
raw_volume_share
prominence_share
width_ticks
scale_persistence
ltp_vwap_distance_ticks
```

重叠或相邻 zone 可以合并并保留 confluence 类型，例如：

```text
HVN+VPOC
HVN+VAH
```

但 acceptance node（HVN/VPOC）不能和 rejection node（LVN）合并，否则经济含义互相冲突。

## 当前 smoke test

使用上午数据构建并冻结 Profile：

```text
商品：09:00-10:15, 10:30-11:30
股指：09:30-11:30
国债：09:30-11:30
available_time = 11:30:00
```

下午验证：

```text
商品：13:30-15:00
股指：13:00-15:00
国债：13:00-15:15
```

最终结果：

```text
selected contracts: 21
profile groups: 168 = 21 contracts * 2 methods * 4 scales
frozen zones: 117
afternoon first-touch events: 68
bounce: 36
break: 31
timeout: 1
```

Resolved bounce rate：

```text
36 / (36 + 31) = 53.7%
```

> [!danger]
> 53.7% 不能解释成有效胜率。当前只有一天，没有 matched random baseline，事件存在品种和交易日聚类，而且不同 zone 的宽度与反应阈值不同。

## 如何定义一次 Touch、Bounce 和 Break

每个 zone 每天下午只统计第一次从区间外进入的 touch。

当前反应阈值基于上午 1min median range：

```text
reaction_ticks =
    max(2, round(0.50 * median_range_ticks))

penetration_ticks =
    max(1, round(0.25 * median_range_ticks))
```

Support：

```text
先到 upper + reaction -> bounce
先到 lower - penetration -> break
```

Resistance 反向处理。

到下午收盘仍未触发则为 timeout，同时记录：

```text
MFE
MAE
time_to_resolution
```

## 如何判断 Support/Resistance 是否有效

### 核心定义

一个 level 只有满足下面条件，才可以称为“有效”：

> 在完全样本外的数据中，相比条件匹配的随机价格 zone，能够稳定提高 bounce probability，或者改善 MFE/MAE 与扣除成本后的 expectancy。

不能简单使用 `bounce rate > 50%`，因为 zone 宽度、approach direction、市场趋势和 barrier 距离都会改变随机基线。

### 主检验应使用对称 First-Passage Barrier

为了单独检验 level 的阻挡能力，主实验建议改用对称 barrier：

```text
support:
  bounce = 先到 upper + k * volatility
  break  = 先到 lower - k * volatility

resistance:
  bounce = 先到 lower - k * volatility
  break  = 先到 upper + k * volatility
```

参数网格：

```text
k = 0.25 / 0.5 / 1.0
timeout = 15 / 30 / 60 min
volatility = 上午 median range 或 ATR
```

当前不对称阈值可以保留为策略型实验，但不应作为纯 level 有效性的主结论。

### Matched Random Baseline

每个真实 level 生成多个条件匹配的随机 zone：

```text
同一品种、同一交易日
相同 available_time
相同 zone width
距下午开盘价相近
位于上午价格区间的相似分位
相同 approach direction
相同 volatility regime
```

核心指标：

```text
bounce_lift =
P(bounce | profile level)
- P(bounce | matched random level)
```

还要加入简单基线：

```text
上午 High / Low
上午 VWAP
均匀随机价格 zone
```

如果复杂 Profile 不优于上午 High/Low/VWAP，则增加的复杂度没有价值。

## 评价指标

### 预测能力

```text
touch rate
bounce probability
break probability
bounce lift
odds ratio
```

### 价格路径

```text
MFE
MAE
MFE / MAE
time-to-bounce
time-to-break
穿越 LVN 后的移动速度
```

### 稳定性

```text
按品种
按月份
按趋势/震荡环境
按高低波动环境
按上午成交量分位
```

### 经济性

```text
扣除手续费和滑点后的 expectancy
stop/target 盈亏分布
最大连续亏损
成交可实现性与容量
```

预测有效性与交易有效性必须分开报告。

## Feature 有效性与 Calibration

需要检验 Profile 强度特征是否有单调预测关系：

```text
scale_persistence 越高，bounce lift 是否越高？
prominence 越大，效果是否越强？
LTP/VWAP distance 越小，是否更稳定？
HVN+VPOC 是否优于单独 HVN？
LVN 首次测试是否优于后续测试？
zone age 增加后是否衰减？
```

可以按 score 或单个 feature 分成五组。如果最高组和最低组之间没有稳定单调差异，当前 strength score 就没有预测意义。

## Ablation Test

至少比较：

```text
A. raw VPOC/VAH/VAL
B. LTP Profile only
C. Interval-VWAP Profile only
D. LTP/VWAP consensus
E. no smoothing
F. sigma 1/2/4 multi-scale persistence
G. 上午 High/Low/VWAP
H. matched random levels
```

重点回答：

1. 平滑是否真正提高样本外 bounce lift？
2. 多尺度 persistence 是否优于单一 sigma？
3. 双 Profile 一致性是否值得牺牲 level 数量？
4. 复杂节点是否优于简单上午 High/Low/VWAP？

## Walk-Forward 设计

优先采用：

```text
D 日完整 Profile -> D+1 验证
```

也可保留：

```text
D 日上午 Profile -> D 日下午验证
```

参数选择必须按时间向前滚动，例如：

```text
训练 60 日
验证 20 日
测试 20 日
```

最终保留一段从未参与任何调参的 holdout period。

代表合约也必须 point-in-time：

```text
上午 -> 下午：只能用上午 volume 选择
D -> D+1：可以用 D 日信息选择 D+1 合约
上午盘中：应使用前一日主力规则
```

## 统计检验

事件不是独立样本。同一天、同一品种、相邻 zone 之间高度相关。

建议：

```text
按 trading_day + root 做 clustered bootstrap
Wilson interval 或 Beta posterior
Permutation test：真实 level vs matched random level
多参数实验使用 Benjamini-Hochberg FDR
```

特征模型示例：

```text
bounce ~ node_type
       + prominence
       + scale_persistence
       + ltp_vwap_distance
       + zone_width
       + distance_from_open
       + volatility
       + liquidity
```

标准误应按品种和交易日聚类，或者使用 random effects。

### 样本量直觉

如果基线 bounce probability 约为 50%：

```text
检测 55% vs 50%：约需 780 个独立事件
检测 60% vs 50%：约需 190 个独立事件
```

实际事件存在聚类，所需样本数会更多。当前 68 个事件远远不足。

## 有效性验收标准

某类 level 至少满足以下条件，才暂时称为有效：

1. 完全样本外 `bounce_lift > 0`。
2. Clustered bootstrap 95% CI 下界大于 0。
3. 相比上午 High/Low/VWAP 仍存在增量。
4. 在多数核心品种和多个时间区间方向一致。
5. 参数上下浮动约 20% 后结论不消失。
6. Prominence、persistence 等强度特征与效果有合理单调关系。
7. 如果目标是交易，扣除手续费和滑点后 expectancy 大于 0。

## 当前实现位置

项目目录：

```text
/Users/mingdaqiu/Develop/kline.luajit/tools/market_profile/
```

主要文件：

```text
market_profile.py       # 完整流水线
contracts.json          # 品种、时段、tick size、multiplier 和参数
README.md               # 实现文档
test_market_profile.py  # 自动测试
```

正式结果：

```text
/Users/mingdaqiu/Downloads/20260109/day_market_profile/
```

关键产物：

```text
selected_contracts.csv
selected_ticks/*.csv.gz
normalization_report.csv
profiles.csv
levels.csv
events.csv
summary.csv
report.html
run_config.json
result.json
```

## 下一步建议

1. 收集��少 60-120 个交易日的同口径 tick 数据。
2. 实现对称 first-passage barrier。
3. 实现 matched random level 生成器。
4. 加入上午 High/Low/VWAP 基线。
5. 建立按 trading_day 向前滚动的 walk-forward pipeline。
6. 对 raw、single-scale、multi-scale 和 dual-profile 做 ablation。
7. 使用 clustered bootstrap 和 permutation test 输出置信区间。
8. 最后才加入手续费、滑点和完整交易规则。

现阶段最重要的问题不是“53.7% 能不能赚钱”，而是：

> Persistent Tick Volume Profile zone 是否在严格 point-in-time、matched baseline 和多日样本外环境下，提供了可重复的增量预测信息。