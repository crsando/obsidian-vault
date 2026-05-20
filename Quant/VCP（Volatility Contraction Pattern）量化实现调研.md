
> 调研日期：2025-05-20
> 来源：综合 GitHub 开源项目、AmiBroker 论坛、TradingView 社区、Medium/Substack 技术博客、QuantConnect 论坛
> 关键参考：Mark Minervini《Trade Like a Stock Market Wizard》《Think & Trade Like a Champion》

## 一、VCP 的数学本质

**VCP** 是 **Mark Minervini** 系统阐述的价格形态，核心三要素：

| 要素 | 数学描述 | Minervini 原始规则 |
|------|----------|-------------------|
| **价格收缩（T1→T2→T3…）** | 每次回撤幅度 ≈ 前一次的 50%（±合理偏差） | 例: 25% → 15% → 8% → 4% |
| **成交量萎缩** | 收缩区间内量能降至50日均量的 30-50% 以下 | 右侧成交量应接近整个上涨周期以来的最低水平 |
| **收缩次数** | 通常 2-6 次收缩（最常见 2-4 次） | 形成"从左到右逐步收紧"的对称结构 |

### Minervini 标注法（T-P-S）

- **T（Time）**：底部形成以来的天数/周数
- **P（Price）**：最大回撤深度 + 最右侧最小收缩的窄幅度
- **S（Symmetry）**：整个底部过程中的收缩次数

### 前提条件：Stage 2 上升趋势模板（8 条量化标准）

VCP 必须在**上升趋势**中出现（延续形态）：

```
1. 当前价格 > 150日均线 AND > 200日均线
2. 150日均线 > 200日均线
3. 200日均线至少上升1个月
4. 50日均线 > 150日均线 AND > 200日均线
5. 当前价格 > 50日均线
6. 当前价格 > 52周低点 × 1.30（至少高于低点30%）
7. 当前价格 > 52周高点 × 0.75（距高点不超过25%）
8. 相对强度评级 ≥ 70（IBD RS Rating）
```

### 核心量化指标

| 指标 | 计算方式 | VCP 阈值 |
|------|---------|---------|
| **收缩比率** | `contraction[i] / contraction[i-1]` | < 0.5~0.6 |
| **ATR 缩减率** | `ATR_recent / ATR_base_start` | 减少 ≥ 20% |
| **成交量干涸** | `recent_volume / MA(volume, 50)` | < 0.5~0.7 |
| **突破量能** | `breakout_volume / MA(volume, 20)` | ≥ 1.5x |
| **底部时长** | 从高点到突破的交易日数 | ≥ 30 天 |
| **价格紧凑度** | `(HHV - LLV) / Close` 最近 N 天 | < 6~10% |

---

## 二、检测算法：四大流派

### 流派1：规则硬编码（最主流，90% 的实现）

```python
def detect_vcp(prices, highs, lows, volumes):
    # 1. 确认 Stage 2 趋势模板
    if not meets_trend_template(prices): return False
    
    # 2. 找底部起点（近一年最高点）
    base_high = max(highs[-252:])
    
    # 3. 识别收缩序列
    contractions = []
    i = len(prices) - 1
    while i > 0 and len(contractions) < 6:
        swing_high = find_swing_high(highs, i)
        swing_low = find_swing_low(lows, i)
        retracement = (swing_high - swing_low) / swing_high
        contractions.append(retracement)
        i -= period_of_contraction
    
    # 4. 验证逐步收缩（每次 < 前次的 60%）
    valid = all(
        contractions[i] < contractions[i-1] * 0.6
        for i in range(1, len(contractions))
    )
    
    # 5. 成交量干涸
    vol_dryup = recent_volume < avg_volume * 0.7
    
    # 6. 突破检测
    breakout = (close > pivot_high) and (volume > avg_vol * 1.5)
    
    return valid and len(contractions) >= 2 and vol_dryup
```

### 流派2：AmiBroker AFL 经典实现（rocketPower）

最被广泛引用的 VCP 检测代码：

```afl
// 核心参数
Timeframe   = 252;    // 一年回看
VolTf       = 50;     // 50日成交量均线
PVLimit     = 0.10;   // Pivot 区域最大宽度 10%

// 1. 价格在底部范围内
HighPrice = HHV(C, Timeframe);
NearHigh = C < HighPrice AND C > 0.6 * HighPrice;

// 2. 成交量下降（线性回归斜率方法）
Vma = MA(V, VolTf);
VolSlope = LinRegSlope(Vma, VolTf);
VolDecreasing = VolSlope < 0;

// 3. Pivot 质量检测
PivotHighPrice = HHV(H, PivotLength);
PivotLowPrice  = LLV(L, PivotLength);
PivotWidth     = (PivotHighPrice - PivotLowPrice) / C;
IsPivot = PivotWidth < PVLimit AND PivotHighPrice == Ref(H, -PivotLength+1);

// 4. Pivot 区域内成交量全部低于均值
VolDryup = Sum(V < Vma, PivotLength) == PivotLength;

Filter = NearHigh AND VolDecreasing AND IsPivot AND VolDryUp;
```

**关键设计洞察：**
- 用 `LinRegSlope(50日均量, 50)` 检测成交量趋势，比简单比较更鲁棒
- 要求 Pivot 区间高点出现在起始位置（确保收缩方向正确）
- **Pivot 宽度 < 10%**（理想 < 6%）

### 流派3：指标替代法

用现有技术指标当 VCP 代理检测器：

| 指标 | 与 VCP 的对应关系 | 使用方式 |
|------|------------------|---------|
| **布林带宽度（BBW）** | BBW 百分位 → 波动率压缩程度 | BBW 降至历史低位 = 可能的 VCP 紧缩 |
| **ATR 百分位** | 短期 ATR / 长期 ATR | 短期 ATR 显著低于长期 = 收缩中 |
| **TTM Squeeze** | BB 被完全包裹在 Keltner 通道内 | Squeeze 状态 = 极端收缩 |
| **标准差** | 短期 StdDev / 长期 StdDev | 比率下降 = 波动率收缩 |
| **ADX** | 趋势强度衰减 → 收缩中 | ADX < 20 → 横盘收缩中 |
| **KC 宽度** | (Upper_KC - Lower_KC) / EMA | 宽度递减 = VCP 收缩 |

#### TradingView VCS 评分系统（oratnek）

**Volatility Contraction Score** — 0~100 分综合评分：

| 组件 | 权重 | 计算方法 |
|------|------|---------|
| **价格压缩** | 高 | 短期 ATR + StdDev vs 长期均值 |
| **成交量收缩** | 中 | 近期成交量 vs 历史均值 |
| **效率过滤器** | 惩罚项 | 强趋势阶段被惩罚（仅关注压缩） |
| **Higher Low 结构** | 加分/减分 | 近期低点是否高于前期结构性低点 |
| **持续性奖励** | 加分 | 压缩持续越久，分数越高 |

- ≥ 80（绿色）→ **临界紧缩**，极度压缩
- 60-80（蓝色）→ 形态形成中，适合加入观察名单
- < 60（灰色）→ 松散/扩张阶段

### 流派4：规则 + ML 辅助

用 **Isolation Forest** 异常检测剔除假信号，不是直接检测 VCP，而是二次过滤：

```python
from sklearn.ensemble import IsolationForest
model = IsolationForest(contamination=0.1, random_state=42)
df['Anomaly'] = model.fit_predict(df[['ATR', 'Volume']])
# -1 = 异常（如谣言导致的300%量能飙升），剔除假信号
```

---

## 三、SENTINEL PRO 评分系统（105 分制）

将 VCP 量化为评分，最精细的公开实现之一：

| 评分维度 | 分值 | 检测方法 |
|---------|------|---------|
| **紧凑度（Tightness）** | 40分 | 对比 20/30/40/60 日的高低范围，检测波动率是否逐步收缩 |
| **成交量（Volume）** | 30分 | 检测近期量能是否相对历史均值萎缩 |
| **均线排列（MA Alignment）** | 30分 | 确认 50/150/200 日均线完美多头排列 |
| **Pivot 加分** | 5分 | 测量当前价格与近期高点（pivot 点位）的接近程度 |

- GitHub: https://github.com/EMMA019/US-stocks
- 配合 **GitHub Actions** 每日自动扫描 + **LINE** 通知 + **DeepSeek API** AI 定性诊断

---

## 四、完整 Python 实现参考（tikamalma）

```python
import yfinance as yf
import pandas as pd
import numpy as np
import talib as ta
from sklearn.ensemble import IsolationForest

LOOKBACK_PERIOD = "6mo"
MIN_BASE_DURATION = 30
ATR_PERIOD = 14
KC_PERIOD = 20
VOLUME_SPIKE_MULTIPLIER = 1.5

def analyze_vcp(symbol, df):
    # 1. 上升趋势验证（至少30%涨幅 + 价格在50MA上方）
    price_increase = (df['Close'].iloc[-1] - df['Close'].iloc[0]) / df['Close'].iloc[0]
    if price_increase < 0.3 or df['Close'].iloc[-1] < df['MA50'].iloc[-1]:
        return False
    
    # 2. 收缩序列检测
    contractions = []
    closes = df['Close'].values
    i = len(df) - 1
    while i > 0 and len(contractions) < 6:
        if closes[i] < closes[i-1]:
            start = i
            while i > 0 and closes[i] < closes[i-1]:
                i -= 1
            end = i
            high = df['High'].iloc[start:end+1].max()
            low = df['Low'].iloc[start:end+1].min()
            retracement = (high - low) / high
            if contractions and retracement > contractions[-1]['retracement'] * 0.6:
                break
            contractions.append({'retracement': retracement})
        i -= 1
    
    # 3. 验证收缩递减 + Keltner 通道宽度递减
    valid = all(
        contractions[i]['retracement'] < contractions[i-1]['retracement'] * 0.6
        for i in range(1, len(contractions))
    )
    
    # 4. Isolation Forest 异常检测（剔除假信号）
    model = IsolationForest(contamination=0.1)
    anomaly_score = model.fit_predict(df[['ATR', 'Volume']])
    
    # 5. 突破检测
    resistance = df['High'].iloc[-20:-1].max()
    breakout = (df['Close'].iloc[-1] > resistance) and \
               (df['Volume'].iloc[-1] > df['Volume'].rolling(20).mean().iloc[-1] * 1.5)
    
    return valid and breakout
```

**量化规则汇总：**
- 价格收缩比：每次 **< 前一次的 50-60%**
- ATR 缩减：底部期间 ATR 降低 **≥ 20%**
- 成交量：收缩期低于均值 **30-50%**
- 突破量能：**≥ 20日均量的 1.5 倍**
- 底部时长：**≥ 30 个交易日**
- 至少 **2 次有效收缩**

---

## 五、入场 / 出场信号的程序化逻辑

### Pivot 点位计算

```python
# 方法1：最后收缩区间的最高点
pivot_price = max(highs[-pivot_length:])

# 方法2：最近N日阻力位
pivot_price = max(highs[-20:-1])

# 方法3（Minervini 原始）：最后一次收缩的摆动高点
```

### 入场条件

```python
entry_trigger = (
    close > pivot_price                            # 价格突破 Pivot
    and volume > ma(volume, 20) * 1.5              # 量能放大 ≥ 1.5x
    and close > open                               # 阳线确认
)
# 买入方式：设置 Buy-Stop 单在 Pivot 价位上方
buy_stop_price = pivot_price * 1.005
```

### 止损

```python
# 方法1：Pivot 区间最低点
stop_loss = min(lows[-pivot_length:])

# 方法2（Minervini 建议）：止损距买入价不超过 5-7%（理想）到 10%（最大）
stop_loss = entry_price * 0.93  # 7% 止损

# 仓位计算
shares = min(
    (total_capital * account_risk_pct) / (entry - stop),  # 风险反推
    (total_capital * 0.4) / entry                         # 最大仓位限制
)
```

### 出场

```python
# R 倍数目标
target = entry + (entry - stop) * R_multiple  # 常用 R=2 或 R=3

# 跟踪止损
trailing_stop = highest_since_entry * (1 - trailing_pct)
```

---

## 六、回测数据（公开较稀缺）

| 来源 | 数据 |
|------|------|
| **Mark Minervini 本人** | 5 年回报 +33,544%（自述），但这是整体系统（VCP + SEPA + 仓位 + 择时）的结果 |
| **PineScriptForge（玉米期货）** | 胜率 44.7%，盈亏比 1.35 |
| **SENTINEL PRO（GLW 实盘）** | +12.3%，持仓 4 天 |
| **社区共识** | 突破策略胜率普遍 **40-50%**，靠盈亏比盈利 |

⚠️ 公开回测极少发布完整 Sharpe / 最大回撤数据；幸存者偏差严重。

---

## 七、开源项目一览

| 项目 | 语言 | 特色 |
|------|------|------|
| [shiyu2011/cookstock](https://github.com/shiyu2011/cookstock) | Python | Stage 2 + VCP + GPT 新闻分析，每日自动筛选 |
| [EMMA019/US-stocks](https://github.com/EMMA019/US-stocks) | Python | 105 分评分 + Streamlit UI + GitHub Actions |
| [jeffreyrdcs/stock-vcpscreener](https://github.com/jeffreyrdcs/stock-vcpscreener) | Python | Dash 可视化 + 市场广度指标 |
| [marco-hui-95/vcp_screener](https://github.com/marco-hui-95/vcp_screener.github.io) | Python | FinViz 预筛 + 收缩计数到 Excel |
| [crankycandle/volatility-contraction-pattern](https://github.com/crankycandle/volatility-contraction-pattern) | Python | 基于 carlam.net 的简洁实现 |
| [tradermonty/claude-trading-skills](https://github.com/tradermonty/claude-trading-skills) | Python | Claude AI + VCP 筛选 + Alpaca 交易执行 |
| rocketPower AFL | AFL | [AmiBroker Forum](https://forum.amibroker.com/t/18720)，最经典检测算法 |
| Amphibiantrading VCP | Pine | [TradingView](https://tw.tradingview.com/script/J1tqSCqR)，5 层收缩追踪 |
| oratnek VCS | Pine | [TradingView](https://www.tradingview.com/script/5eg3OfM6)，0-100 评分 |
| tikamalma VCP Scanner | Python | [Substack](https://tikamalma.substack.com/p/understanding-basics-of-vcp-and-creating)，含 ML + 完整代码 |

---

## 八、推荐实现路径

```
第1层：Stage 2 趋势模板 → 过滤掉 90%+ 的票
  ↓
第2层：VCP 核心检测（ATR 收缩 + 量能下降斜率 + Pivot 紧凑度）
  ↓
第3层：信号确认（突破量 ≥ 1.5x + Pivot 宽度 < 10%）
  ↓
第4层：入场执行（Buy-Stop 在 Pivot 上方，止损 -7%）
```

### 必备技术指标

```python
atr_14 = ta.ATR(high, low, close, 14)     # 波动率度量
ma_50  = ta.SMA(close, 50)                 # 趋势确认
ma_150 = ta.SMA(close, 150)                # 趋势确认
ma_200 = ta.SMA(close, 200)                # 趋势确认
vol_ma = ta.SMA(volume, 50)                # 成交量基准
bbw    = (bb_upper - bb_lower) / bb_mid    # 布林带宽度
kc_width = (kc_upper - kc_lower) / ema_20  # Keltner 宽度
vol_slope = linregress(vol_ma, 50).slope   # 成交量趋势
```

### 关键风险提示

- **VCP 不是孤立策略**，必须嵌入完整交易系统（选股 + 择时 + 仓位 + 风控）
- **假突破率高**，胜率约 40-50%，靠盈亏比盈利
- **牛市效果远好于震荡/熊市**
- 收缩比阈值（0.5 vs 0.6）、Pivot 长度（3 vs 5 天）等**参数敏感**，需针对标的回测调优
