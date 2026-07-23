> 一份自研 LuaJIT 期货 K 线内存库的完整设计思路与踩坑总结。核心问题：本地存储、Tick 实时合成、交易时段/trading_day、1m→多周期聚合，以及 vnpy 等成熟框架的实战经验。
> 整理日期：2026-07-23

---

## 一、整体目标与分层架构

在 **LuaJIT** 进程内维护多品种、多周期的 K 线内存表，支持 tick 实时合成、交易时段感知、增量落地与快速回放。面向交易系统，追求低 GC 压力和高吞吐。

**数据分层（热 / 温 / 冷）：**

| 层 | 职责 | 存储 |
|---|---|---|
| 热数据 | 实时读写、指标计算 | 内存 **FFI 定长数组** |
| 温数据 | 当天/近期、边写边看 | **CSV**（LuaJIT 原生追加、pandas 直读） |
| 冷数据 | 长期归档、跨语言分析 | **Parquet**（DuckDB/polars 高性能） |

---

## 二、本地存储格式（可读优先 + Python 友好）

放弃私有二进制，落地采用**文本/列式格式**，让 pandas / DuckDB / Excel 都能直接读。

### 温层：CSV（默认落地）
- LuaJIT 原生 `io.write` 高效追加，append-only 天然契合 K 线。
- 带 `#` 注释头存元信息（symbol/period/tz），pandas 用 `comment='#'` 跳过。
- **bar_time** 写成 `YYYY-MM-DD HH:MM:SS` 可读字符串（想要精度再加一列毫秒整数）。
- 价格按 **tick size** 定点格式化（`%.1f`），避免打印出脏浮点。
- 命名 `{symbol}_{period}_{trading_day}.csv`，按交易日切分。

```python
import pandas as pd
df = pd.read_csv("rb2510_1m_20260723.csv", comment="#", parse_dates=["bar_time"])
```

### 冷层：Parquet（归档 + 分析）
- 列式、zstd 压缩通常比 CSV 小 **5–10 倍**、带 schema、读取快几个数量级。
- **不在 Lua 侧硬写**（无好用库）：Lua 只写 CSV，用独立 DuckDB/Python 脚本转：
  `COPY (SELECT * FROM 'xx.csv') TO 'xx.parquet' (FORMAT parquet, COMPRESSION zstd);`
- 按 `symbol/period/year` 分区，DuckDB 可跨文件查。

> 内存里照样用 FFI 定长 struct 保性能，落地时格式化成 CSV——两者**解耦**，多一步文本序列化换来可读性与互操作。

---

## 三、库文件拆分与 API

```
kline/
├── init.lua        入口，聚合 API
├── types.lua       FFI cdef：Bar struct、枚举
├── bar.lua         单根 bar 读写辅助
├── series.lua      ★核心：单条(symbol,period)时间序列内存管理
├── store.lua       多 Series 容器
├── aggregator.lua  tick→bar 合成状态机
├── calendar.lua    交易日历 + 交易时段
├── period.lua      周期定义 + 时间对齐 + roll-up
├── codec.lua       CSV/JSONL/Parquet 编解码
├── persist.lua     落地调度（flush/切分/加载）
├── config.lua      品种时段表、路径配置
└── util.lua        时间/日志工具
```
最小可用集合：`types + series + store + init`（纯内存表）。

**主要 API 入口（4 组）：**
1. 内存表：`store:series(sym,period)` / `s:append` / `s:last` / `s:slice` / `s:find`(二分)
2. 实时合成：`store:on_tick(tick)` → 返回封口 bar；`store:on_bar_closed(cb)` 回调
3. 落地/加载：`store:persist()` 增量落地、`store:load()` 预热、流式回放
4. 时段：`cal:in_session` / `is_session_end` / `trading_day`

---

## 四、行情字段（Bar struct，定长 88 字节）

| 字段 | 类型 | 说明 |
|---|---|---|
| **bar_time** | int64(ms) | bar 起始时间（时段轴对齐） |
| **trading_day** | int32 | 交易日 YYYYMMDD（夜盘归属单独存） |
| open/high/low/close | double | OHLC |
| **volume** | double | 成交量 |
| turnover | double | 成交额 |
| **open_interest** | double | 持仓量（期货必备，时点值） |
| settlement | double | 结算价（日线才有意义） |
| tick_count | int32 | 本 bar tick 数（数据质量用） |
| flags | uint8 | 位标志：已封口/集合竞价/夜盘 |

> **symbol 不进 struct**，存在 Series header 里共用，不重复存字符串。

---

## 五、Tick 实时合成 K 线（状态机）

核心是"时段感知的 OHLC 状态机"：
1. tick 进来先 `in_session(t)` **过滤脏 tick**（休盘异常价、断线重连过期行情）。
2. `align(t)` 把时间戳对齐到 bar 起始。
3. 未跨界 → 更新 high/low/close，累加 volume/turnover，刷新 open_interest。
4. 跨界 or 命中收盘 → **封口旧 bar**，用当前 tick 初始化新 bar。
5. **休盘 gap 不补空 bar**：时间轴是时段拼接的逻辑轴。

> ⚠️ **volume 用差值累加**：CTP 推的 volume 是**当日累计值**，合成 1m 时必须 `本tick累计 - 上tick累计`，不能直接加。

---

## 六、trading_day 管理（期货跨日核心）

**第一原则：能用交易所/CTP 给的 TradingDay，就绝不自己算。**

- CTP 行情结构里 **ActionDay**（自然日）与 **TradingDay**（交易日）故意不相等。夜盘 22:30 的 tick，ActionDay=当天，TradingDay=下一交易日。
- 自己推算的坑：夜盘归属"下一个交易日"不是简单 +1 天——**周五夜盘跳到下周一，节假日前夜盘跳过整个假期**，必须查交易日历表。
- **约定**：每根 bar 存两列 `bar_time`（自然时间）+ `trading_day`；文件切分、日线聚合、按日查询全用 trading_day。夜盘 bar_time 日期 ≠ trading_day 是**正常的**。跨午夜时合成器判断边界必须走"时段轴"，不能因自然日翻页重置。

### trading_day 对分析与指标的作用
1. **日级聚合的分组键**：日/周/月线都是 `GROUP BY trading_day`。日 K 开盘价应是**夜盘第一根**，用自然日分组会全错。
2. **日内重置类指标的锚点**：**VWAP**、当日高低点、当日涨跌幅、ORB 开盘区间突破，都在 trading_day 翻页时归零。
3. **跳空/缺口计算**：今日开盘 vs 昨收，"昨日"是**上一个 trading_day**（周一的昨日是上周五）。
4. **区分连续 vs 截断**：MA/MACD/RSI 跨日连续；VWAP 日内截断。
5. **回测对齐 / 多品种截面**：按 trading_day 对齐横截面，防未来函数。

---

## 七、1m → 5m/15m 聚合（休盘/午休处理）

**核心方案：时钟对齐（决定归属）+ 休盘边界强制封口（防跨段污染）+ 缺失不补空。**

- **时钟对齐**：bar key = `(分钟数 // 周期) * 周期`。国内时段起点 `09:00/10:30/13:30/21:00` 天然能被 5/15/30 整除，对齐后每段第一根从时段起点干净开始。
- **休盘强制封口**：判定 `in_session(t) 且 not in_session(t+1min)`（下一分钟离开当前时段），命中就立刻封口当前聚合 bar，哪怕没满。
- **缺失 1m 不补空**：某分钟无成交就跳过，5m 用实际存在的 1m 合成。

验证（螺纹钢日盘含 10:15 休盘 + 11:30 午休，225 根 1m = 45 根 5m / 15 根 15m）：

| 周期 | 区间 | 说明 |
|---|---|---|
| 15m | 10:00~10:15 | 休盘边界封口，不并入 10:30 |
| 15m | 10:30~10:45 | 休盘后从时段起点干净开新 |
| 15m | 11:15~11:30 | 午休边界封口 |

三条铁律：**① 缺失不补空 ② 别用"根数对齐"（会把休盘前后合并，也跟文华对不齐）③ volume/turnover 累加、open_interest 取时点值。**

只存 **1m 为 base（真相源）**，5m/15m/30m/1h 由 1m 实时 roll-up 派生；**日线单独存**（绑 trading_day 逻辑）。

---

## 八、成熟框架踩坑参考（vnpy 等）

### 两大对齐流派
- **墙上时钟对齐**（`分钟 % window`）：vnpy 原生、文华、多数软件。简单但需给休盘打补丁，且**只能用整除 60 的窗口**（2/3/5/6/10/15/20/30）。
- **交易时长对齐**（累计交易分钟切窗口）：vnpy 社区 hxxjava 方案。支持 90 分钟/4 小时等任意周期，天然跨休盘连续，但复杂。

### vnpy 已知坑（重点）
1. **休盘导致非常规周期错误**：20 分钟线因 10:15 休盘，原生逻辑会把 10:00-10:14 和 10:30-10:49 错误合成 35 分钟一根。官方补丁是对三大商品所在 **10:14 强制切分**。→ 通用 `in_session` 封口更干净。
2. **小时线多算一根**（Issue #2775）：分钟分支左闭右开 `[0,60)`，小时分支左开右闭 `(0,60]`，**边界语义不统一**。→ 教训：全库统一**左闭右开 `[start,end)`**。
3. **依赖 tick 推动封口**：无成交分钟延迟封口、收盘最后一根不调 `generate()` 就丢失。→ 应做**时段感知的主动封口**。
4. **volume 是当日累计值**：合成时要用差值，直接加会爆表。
5. **只支持整除 60 的窗口**：想做 90 分钟得改交易时长对齐。

### 其他平台
- **天勤 TqSdk**：K 线由**服务器端合成推送**，本地零负担、多合约自动对齐；代价是依赖其服务器。属于与 vnpy（本地合成）不同的哲学。
- **掘金/聚宽/米筐**：平台预合成多周期 bar，用户直接取，但看不到、控制不了合成细节。
- **backtrader**：resample/replay 有 session 概念，但对中国期货午休/夜盘支持不友好，需自定义 TradingCalendar。
- **hxxjava MyBarGenerator**：社区最系统方案，交易时段字符串 `"21:00-23:00,09:00-10:15,10:30-11:30,13:30-15:00"` 驱动 + 交易时长对齐，支持日/周/月/年线；明确**不处理长假**（和所有软件一样）。

### 自研库避坑清单
1. ✅ 1m base + 时钟对齐 roll-up（主流稳妥）
2. ✅ 休盘/午休/收盘边界强制封口（`in_session(t) 且 not in_session(t+1)`）
3. ⚠️ 边界语义全库统一 **左闭右开 `[start,end)`**
4. ⚠️ **主动封口**，不纯靠下一根推动
5. ⚠️ **volume 用差值累加**（CTP 累计值）
6. ✅ trading_day 用 CTP 给的字段
7. ✅ 不处理长假对齐，只处理周末 + 交易日历兜底
8. 🔮 要支持 90 分钟等非整除周期 → 升级交易时长对齐

---

## 来源

- vn.py 社区精选24《针对国内期货市场的K线合成器》：https://zhuanlan.zhihu.com/p/352485736
- vnpy 论坛《彻底解决K线生成器的问题》(hxxjava)：https://www.vnpy.com/forum/topic/30193
- vnpy Issue #2775 小时线多算一根：https://github.com/vnpy/vnpy/issues/2775
- vnpy Issue #1162 无法合成15/30分钟：https://github.com/vnpy/vnpy/issues/1162
- vnpy Issue #957 郑商所收盘数据：https://github.com/vnpy/vnpy/issues/957
- VNPY3.0 解析——K线合成：https://zhuanlan.zhihu.com/p/3004943208
- TqSdk 与 vn.py 的差别：https://doc.shinnytech.com/tqsdk/latest/advanced/for_vnpy_user.html

---

## 九、对齐哲学对比：TradingView vs 国内期货

调研了 **TradingView** 的 K 线聚合逻辑，发现它与国内期货（文华/vnpy）走的是**相反的哲学**，对自研库的对齐策略选择很有参考价值。

### TradingView 的核心行为
- **从交易所日边界（午夜/整点）向下取整对齐**，不做 session 感知的强制封口：
  `bar_start = floor((t - day_anchor) / period) * period + day_anchor`
- 后果：高周期 bar 会"吃掉"开盘时刻——**美股 1 小时图上没有独立的 09:30 bar**，
  开盘被并进 09:00–10:00 那根（社区原话："Hourly bars cut off the 09:30 New York open"）。
- 原因：TradingView 是**全球全品种平台**（股/期/外汇/加密几万标的），
  无法为每个品种维护 session 时段表，只能用统一日边界对齐——简单、一致、可预测，代价是牺牲 session 精确性。

### 周期规范（官方 Pine 文档）
- 无"小时"单位，`1H` 非法，1 小时写 `"60"`（内部把小时当分钟数处理）。
- **分钟支持 1–1440 任意值** → 能做 **45、90 分钟**等非整除周期（比 vnpy 原生只支持整除 60 的更灵活）。
- 秒仅 1/5/10/15/30/45；tick 仅 1/10/100/1000。
- bar 时间戳用**起始时间**（左闭），与国内一致。

### 对比表

| 维度 | TradingView | 国内期货（本设计） |
|---|---|---|
| 对齐锚点 | **日边界向下取整** | **session 起点对齐** |
| 开盘处理 | 吃掉开盘（无独立开盘 bar） | 从时段起点干净开始 |
| 休盘/午休 | 不强制封口 | **休盘边界强制封口** |
| 非整除周期 | 原生支持 1–1440 分钟 | 需交易时长对齐 |
| session 表 | 不维护（全球通用） | 必须维护品种时段表 |
| 设计取向 | 通用、简单、可预测 | 贴合期货交易时段、精确 |

### 给自研库的决策
1. 面向**国内期货专用** → 坚持 **session 对齐 + 休盘强制封口**（比 TV 精确，与文华一致），别学 TV 日边界对齐。
2. 可借鉴 TV 两点：**支持 1–1440 任意分钟周期**（用交易时长对齐做 45/90 分钟）、
   **API 区分已确认/未确认 bar**（回调只在封口触发，避免 repaint 抖动）。
3. 若未来要做跨品种/全球标的通用聚合，再把 TV 式日边界对齐作为可选模式。

### 来源（TradingView）
- 官方 Pine Script 文档 · Timeframes：https://www.tradingview.com/pine-script-docs/concepts/timeframes/
- request.security / 多周期数据：https://www.tradingview.com/pine-script-docs/concepts/other-timeframes-and-data/
- 社区验证"1h 无 09:30 bar"（行为层，非官方逐字）：https://tw.tradingview.com/scripts/sessions/
- 备注：TradingView 未公开高周期对齐逐字算法，"日边界对齐"为社区长期观察的一致行为，可信但属经验性结论。

---

## 九、对齐哲学对比：TradingView vs 国内期货

调研了 **TradingView** 的 K 线聚合逻辑，发现它与国内期货走的是**相反的对齐哲学**，对本库的对齐策略很有参考价值。

### TradingView 的做法（日边界对齐）
- **从交易所日边界（午夜/整点）向下取整**做纯墙上时钟对齐，**不做 session 感知的强制封口**。
- 后果：高周期 bar 会"吃掉"开盘——**美股 1 小时图上没有独立的 09:30 bar**，开盘被并进 09:00–10:00 那根（社区原话："Hourly bars cut off the 09:30 New York open"）。
- 周期规范（官方 Pine 文档）：无"小时"单位，1 小时写 `"60"`；**分钟支持 1–1440 任意值**（能做 45/90 分钟）；秒仅 1/5/10/15/30/45。
- 为什么这么设计：TradingView 是**全球全品种平台**，无法为每个品种维护 session 表，故用统一日边界对齐——简单、一致、可预测，代价是牺牲开盘/收盘精确性（很多 session 指标干脆 1h 以上拒绝工作）。
- bar 时间戳用**起始时间**（左对齐），与国内一致。

### 对比表

| 维度 | TradingView | 国内期货（本库） |
|---|---|---|
| 对齐锚点 | **日边界向下取整** | **session 起点对齐** |
| 开盘处理 | 吃掉开盘（无独立开盘 bar） | 从时段起点干净开始 |
| 休盘/午休 | 不强制封口 | **休盘边界强制封口** |
| 非整除周期 | 原生支持 1–1440 分钟 | 需交易时长对齐 |
| session 表 | 不维护（全球通用） | 必须维护品种时段表 |
| 设计取向 | 通用、简单、可预测 | 贴合期货交易时段、精确 |

### 对本库的启示
1. 本库**国内期货专用**，坚持 **session 起点对齐 + 休盘强制封口**（比 TradingView 精确、与文华一致），不学其日边界对齐。
2. 可借鉴 TradingView 两点：**支持 1–1440 任意分钟周期**（用交易时长对齐做 45/90 分钟）；**API 层区分已确认/未确认 bar**，回调只在封口触发，避免 repaint 式抖动。
3. 若未来要做跨品种/全球标的通用聚合，再把 TradingView 式日边界对齐作为可选模式。

> 置信度说明：TradingView 未公开高周期对齐的逐字算法，"日边界对齐"是社区长期观察的一致行为（可信但属经验结论）；周期规范部分为官方文档确证。

### 来源
- 官方 Pine Script 文档 · Timeframes：https://www.tradingview.com/pine-script-docs/concepts/timeframes/
- 官方 Pine Script 文档 · request.security / Other timeframes：https://www.tradingview.com/pine-script-docs/concepts/other-timeframes-and-data/
- 社区验证"1h 无 09:30 bar / 小时 bar 截断开盘"：https://tw.tradingview.com/scripts/sessions/ 、https://tw.tradingview.com/scripts/%23newyorksession/
