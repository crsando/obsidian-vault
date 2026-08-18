---
title: Tick Volume Profile 识别 Support Resistance 的基本逻辑
created: 2026-08-18
tags:
  - 期货
  - 量化
  - MarketProfile
  - SupportResistance
  - Research
status: active
---

# Tick Volume Profile 识别 Support Resistance 的基本逻辑

关联笔记：

- [[纯日盘 Tick Volume Profile 支撑阻力识别与有效性验证]]
- [[Tick Volume Profile 每日研究流程与本地浏览网站]]

## 一句话概括

> 先把“市场在哪些价格上最愿意成交”画成一张价格地形图，再从稳定的山峰、山谷和边界中提取值得观察的价格区域，最后观察价格重新接近这些区域后，是反弹、突破还是反复震荡。

当前项目严格来说使用的是 **Tick Volume Profile**，不是经典的 TPO Market Profile：

```text
TPO Market Profile：统计价格停留时间
Tick Volume Profile：统计不同价格上的成交量
```

## 原始数据是什么

原始数据约每 500ms 一条，是 CTP 风格的行情快照，不是真正逐笔成交：

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

其中 `Volume` 不是这一笔的成交量，而是当天累计值。

例如：

```text
09:30:00  Volume = 100
09:30:00  Volume = 105
09:30:01  Volume = 112
```

真正新增的成交量是：

```text
0、5、7
```

所以项目首先计算：

```text
delta_volume = max(当前累计 Volume - 上一笔累计 Volume, 0)
```

规则：

- 第一笔有效 Tick 只建立累计值基准；
- 计数器回退时负差截为 0；
- 非交易时段 Tick 不更新差分基准；
- 时间倒退和无效价格会被过滤。

## 把新增成交量放到价格上

项目同时构建两张 Profile。

### LTP Profile

LTP 即 Last Traded Price。

```text
某个快照新增成交 10 手
LastPrice = 4727
→ 把 10 手记到 4727
```

公式：

```text
profile_ltp[round_to_tick(LastPrice)] += delta_volume
```

优点：简单、稳定，直接反映每个行情快照结束时的价格。

局限：500ms 内可能发生多笔不同价格成交，但新增成交量会全部压在最后一个价格上。

### Interval-VWAP Profile

项目还用新增成交额和新增成交量计算两次 Tick 之间的平均成交价：

```text
interval_vwap =
    delta_turnover / (delta_volume × turnover_divisor)
```

再把新增成交量记到 interval VWAP：

```text
profile_vwap[round_to_tick(interval_vwap)] += delta_volume
```

如果 interval VWAP 不合理，会回退到 LastPrice，并标记 `VWAP_FALLBACK_LTP`，保证成交量不丢失。

为什么要做两张 Profile？

```text
LTP Profile：价格快照终点
VWAP Profile：快照间成交的平均位置
```

如果两张 Profile 在相近价格都出现稳定节点，这个节点比只在一张图里出现更可信。

## 把 Profile 理解成价格地形图

假设统计结果为：

```text
价格    成交量
4724     300
4725     600
4726    1200
4727    1800
4728    1300
4729     700
4730     200
```

横向画出来后，4727 附近像一座山峰。

可以把市场想象成一个商场：

```text
成交量大的价格：人群愿意停留和交易的热门区域
成交量小的价格：大家快速经过、不愿停留的通道
Profile 边缘：市场当天探索到的价格极端
```

## 为什么要平滑

Raw Profile 以一个 `tick_size` 为价格格，容易出现毛刺：

- 一笔大成交可能形成孤立尖峰；
- 500ms 内多笔成交被压在一个 LastPrice；
- 低流动性品种的价格档位稀疏；
- 直接找局部最大值会产生很多假节点。

但只使用一个平滑参数也不可靠。参数稍微改变，峰谷可能移动、合并或消失。

所以项目同时观察：

```text
sigma = 0 / 1 / 2 / 4 ticks
```

可以理解成从不同距离看同一片地形：

```text
sigma=0：贴近地面，细节多但噪声也多
sigma=1：轻度平滑
sigma=2：观察主要结构
sigma=4：只看更大的山脉
```

如果一个峰谷只在某个 sigma 下出现，它可能是噪声。

如果它在至少 3 个尺度都存在，而且中心位置变化不大，才认为它是 `persistent node`。

平滑只用于识别结构：

```text
VPOC、VAH、VAL：仍从 Raw LTP Profile 计算
HVN、LVN：使用多尺度平滑结果识别
```

## 识别哪些关键节点

### VPOC

```text
Volume Point of Control
整个上午 Profile 中成交量最大的价格
```

它代表上午市场最主要的价值中心，更像“市场最认同的价格”。

VPOC 往往有价格磁铁作用，但不一定天然就是阻力。

### HVN

```text
High Volume Node
局部成交量山峰
```

HVN 表示市场在这里成交充分、接受度高。

常见含义：

- 价格平衡区域；
- 均值回归目标；
- 离开后再次返回时可能产生反应；
- HVN 外边缘可能比中心更像支撑阻力。

项目要求 HVN：

```text
具有足够 prominence
至少在 3 个平滑尺度存在
跨尺度中心偏移不大
LTP/VWAP 两张 Profile 位置接近
```

### LVN

```text
Low Volume Node
两个成交量山峰之间的低成交山谷
```

LVN 表示市场过去不愿停留的区域。

它具有双重性质：

```text
从外部首次接近：可能发生 rejection
一旦被接受并进入：可能快速穿越到下一个 HVN
```

项目只在两个相邻 persistent HVN 之间寻找 LVN，并要求 LVN 自己也跨多个尺度稳定。

### VAH / VAL

项目从 VPOC 开始向上下扩展，直到包含约 70% 的上午成交量：

```text
VAH = Value Area High
VAL = Value Area Low
```

它们是主要价值区域的上下边界，通常比 VPOC 中心更像传统的支撑阻力边界。

### Profile Edge

```text
上午 Profile 的最高和最低有成交价格边界
```

它们表示上午价格探索的极端区域。

Edge 的经济含义和 HVN/LVN 不同，因此必须单独标记。

## 为什么输出 Zone，而不是一条精确价格线

真实市场很少精确到某一个价格立即反转。

项目输出的是区间：

```text
lower = 4724.3
center = 4726.9
upper = 4729.5
```

而不是：

```text
4726.9000 是一条神奇支撑线
```

Zone 的宽度来自 Profile 节点本身的宽度、plateau 或边界范围。

如果多个证据靠近，会合并并保留 confluence：

```text
HVN + VPOC
HVN + VAH
```

但 HVN 和 LVN 不会随便合并，因为：

```text
HVN：接受区域
LVN：低接受、拒绝或快速穿越区域
```

两者语义相反。

## Support 和 Resistance 不是永久属性

项目先识别的是“重要价格区域”，然后根据价格接近方向赋予角色。

```text
价格从 Zone 上方向下进入
→ Candidate Support

价格从 Zone 下方向上进入
→ Candidate Resistance
```

例如同一个 4727 Zone：

```text
价格从 4740 跌到 4727
→ 候选支撑

价格从 4715 涨到 4727
→ 候选阻力
```

所以一个 Zone 不是永远的 support，也不是永远的 resistance。

## 避免未来数据泄漏

当前研究流程严格使用上午数据识别节点：

```text
上午 Tick
→ 构建 Volume Profile
→ 11:30 冻结 Zones
```

然后用下午数据观察：

```text
下午 Tick
→ 第一次 Touch
→ Bounce / Break / Timeout
```

Zone 只从 `available_time=11:30` 之后开始显示，不会横贯上午。

否则视觉上会让人误以为上午早些时候已经知道这些 level，形成隐性 look-ahead。

## Touch、Bounce 和 Break

每个 Zone 每天下午只优先统计第一次从区间外进入的 Touch。

对于 Support：

```text
先明显向上离开 Zone
→ Bounce

先明显向下穿透 Zone
→ Break
```

对于 Resistance，方向相���。

直到收盘没有明确结果：

```text
Timeout
```

同时记录：

```text
MFE：向有利方向最多走了多少
MAE：向不利方向最多走了多少
Resolution time：多久得到结果
```

## 当前完整流程

```text
原始 Tick
→ 过滤无效和非交易时段数据
→ 计算 delta_volume / delta_turnover
→ 构建 LTP / Interval-VWAP Profile
→ sigma 0/1/2/4 多尺度平滑
→ 识别 VPOC、VAH/VAL、HVN、LVN、Edges
→ 筛选跨尺度、跨方法稳定节点
→ 合并为价格 Zones
→ 11:30 冻结
→ 下午识别 Touch、Bounce、Break
→ 生成 HTML/PNG
→ 人工 Review
```

## 项目真正表达的是什么

项目不是在说：

```text
“这里一定上涨，那里一定下跌。”
```

项目真正表达的是：

> 上午市场在这些价格形成了明显、稳定、可解释的成交结构；下午价格重新接近时，这些区域值得重点观察。

算法首先保证：

- 节点从哪里来；
- 使用了哪些数据；
- 什么时间开始可用；
- Zone 的上下边界是什么；
- 价格从哪一侧进入；
- Touch 后发生了什么。

至于某类 Zone 是否具有稳定预测能力，要通过后续多个交易日的重复运行、肉眼 Review 和案例积累来判断。

## 常见误区

### 误区一：成交量最大的位置一定是强支撑

不一定。VPOC/HVN 更多表示“市场接受”，可能是价格磁铁，也可能被快速穿越。

### 误区二：LVN 一定反弹

不一定。LVN 从外部接近时可能 rejection；一旦被接受，反而可能快速穿越。

### 误区三：平滑越强越稳定

过度平滑会把真实的多个节点合并成一个宽峰。项目使用多尺度一致性，而不是无限加大 sigma。

### 误区四：Bounce 数量超过 Break 就说明有效

单日事件数量太少，而且 Zone 宽度、价格趋势和阈值都会影响结果。当前阶段只用来检查逻辑和积累案例。

### 误区五：图上看起来挡住价格，就是事先可用的支撑阻力

必须检查 `available_time`。如果 Level 使用了 Touch 之后的数据构造，再漂亮也是未来函数。

## 代码位置

```text
Tick 转 1min：
tools/tick_data/convert_to_1m.py

Profile、Level 与 Event：
tools/market_profile/market_profile.py

合约、时段和参数：
tools/market_profile/contracts.json

HTML/PNG 可视化：
tools/market_profile/visualize.py

本地研究工作台：
tools/market_profile_web/
```