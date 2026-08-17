---
title: Tick Volume Profile 每日研究流程与本地浏览网站
created: 2026-08-17
tags:
  - 期货
  - 量化
  - MarketProfile
  - Research
  - Workflow
status: active
---

# Tick Volume Profile 每日研究流程与本地浏览网站

关联方法论：[[纯日盘 Tick Volume Profile 支撑阻力识别与有效性验证]]

## 当前阶段目标

当前阶段不急着做复杂统计或交易策略，而是建立一套每天都能稳定重复的研究闭环：

```text
收集新一天 tick
→ 检查数据
→ 生成 1min
→ 运行 Tick Volume Profile
→ 识别 support/resistance zones
→ 生成 HTML/PNG
→ 肉眼复盘
→ 记录人工判断和问题
```

目标是先把数据、算法、图表和人工判断之间的关系跑顺，并积累多个交易日的典型案例。

## 每日目录规范

每个交易日独立存放：

```text
~/Downloads/YYYYMMDD/
├── *.csv
├── 1min/
└── day_market_profile/
    ├── selected_ticks/
    ├── selected_contracts.csv
    ├── normalization_report.csv
    ├── profiles.csv
    ├── levels.csv
    ├── events.csv
    ├── summary.csv
    ├── result.json
    └── visual/
        ├── index.html
        ├── manifest.json
        ├── details/
        └── images/
```

其中：

- 根目录 CSV 是原始 tick，只读，不覆盖；
- `1min/` 是全品种 1min K线归档；
- `day_market_profile/` 是纯日盘 Profile、level 和事件结果；
- `visual/` 是每日总览、单品种详情和静态 PNG。

## 第一步：检查原始数据

拿到新一天数据后先检查：

```text
文件数量是否合理
CSV 表头是否一致
TradingDay 是否正确
单文件 InstrumentID 是否一致
是否有空文件或异常小文件
tick 时间是否倒退
Volume/Turnover 是否异常重置
```

这一阶段只回答“数据能不能研究”，暂时不看行情结果。

## 第二步：生成 1min K线

```bash
DATE=20260109
RAW="$HOME/Downloads/$DATE"

python3 tools/tick_data/convert_to_1m.py \
  "$RAW" \
  "$RAW/1min" \
  --workers 8
```

检查：

```text
源文件数 == 输出文件数
bad_price == 0
backwards == 0
counter_reset 数量是否异常
1min Bar 数量是否符合交易时段
```

1min 当前不是 Profile 的直接输入，但它是独立归档和视觉对照数据。

## 第三步：运行 Tick Volume Profile

```bash
python3 tools/market_profile/market_profile.py \
  "$RAW" \
  "$RAW/day_market_profile" \
  --overwrite
```

流程包括：

```text
纯日盘品种筛选
上午流动性过滤
代表合约选择
Tick 标准化
LTP / Interval-VWAP Profile
sigma 0/1/2/4 平滑
VPOC / VAH / VAL
Persistent HVN / LVN
上午冻结 level
下午 first-touch 事件
```

首先检查：

```text
selected_contracts.csv
normalization_report.csv
result.json
```

重点观察：

- 代表合约是否符合当天流动性常识；
- 活跃品种是否被意外过滤；
- LONG_GAP、reset、VWAP fallback 是否异常增加；
- Profile volume 是否守恒；
- Level 数量是否明显异常。

## 第四步：生成 HTML 和图片

```bash
python3 tools/market_profile/visualize.py \
  "$RAW/day_market_profile" \
  "$RAW/day_market_profile/visual" \
  --overwrite
```

打开：

```text
day_market_profile/visual/index.html
```

先浏览每日总览，再进入感兴趣或异常的单品种详情。

## 第五步：固定肉眼检查顺序

### 第一轮：只检查 Tick 和 1min K线

```text
1min K线和 Tick path 是否一致
是否有明显跳点、断流、错误尖峰
休盘位置是否正确
Volume 和 Open Interest 是否异常
```

这一轮只判断数据和显示是否可信。

### 第二轮：检查 Volume Profile

```text
Raw Profile 形状是否自然
LTP 和 VWAP 形状是否接近
sigma 1/2/4 后主要峰谷是否稳定
是否有单个异常 tick 形成假尖峰
Profile 是否过于稀疏或过度平滑
```

### 第三轮：检查 Zone 位置

先不看 bounce/break 结论，只判断位置是否合理：

```text
VPOC 是否位于主要成交密集区
VAH/VAL 是否包围主要 Value Area
HVN 是否确实对应成交接受区域
LVN 是否位于两个成交峰之间
Zone 是否过宽或过窄
Zone 数量是否过多
相邻 Zone 合并是否合理
```

### 第四轮：检查下午价格反应

```text
价格是否真的从区间外进入 zone
Touch 方向是否正确
Bounce 是否是清晰、快速的拒绝
Break 是否有明确穿透和延续
LVN 被进入后是否快速穿越
价格是否长时间在 zone 内反复纠缠
```

## 人工评价标准

每个被触碰的 zone 简单分级：

```text
A：Zone 位置合理，价格出现清晰、快速反应
B：Zone 位置合理，但反应一般或存在震荡
C：Zone 勉强合理，反应不清晰
D：Zone 明显不合理，或由数据异常产生
```

行为标签：

```text
clean_bounce
weak_bounce
immediate_break
choppy
no_touch
data_problem
```

不要只记录“成功/失败”，否则会丢失价格路径信息。

## 人工 Review 记录

建议每个交易日生成或维护 `review.csv`：

```text
trading_day
instrument
node_type
center
lower
upper
role
auto_outcome
manual_grade
manual_behavior
profile_quality
zone_quality
comment
image_path
reviewed_at
```

示例：

```text
20260109,IF2603,HVN+VPOC,4726.9,4724.3,4729.5,
support,bounce,A,clean_bounce,good,good,
首次进入后快速反弹,images/IF2603_20260109.png,2026-08-17
```

## 发现问题时的分类

不要看到怪 level 就马上调参数。先分类：

```text
数据问题：
  tick 缺失、时间错误、Volume 重置、价格异常

Profile 问题：
  LTP 分配偏差、VWAP fallback、平滑失真

Level 问题：
  错误峰谷、zone 太宽、节点错误合并

事件问题：
  touch 方向错误、bounce/break 阈值不合理

可视化问题：
  available_time、价格轴、休盘压缩、marker 错误
```

只修改对应层，不要混着调。

## 参数修改纪律

当前参数先固定：

```text
sigma = 0/1/2/4
persistence >= 3
LTP/VWAP distance <= 2 ticks
Value Area = 70%
prominence share >= 0.25%
valley depth >= 25%
```

至少连续观察 10-20 个交易日后，再总结重复问题。

如果修改参数：

1. 记录修改原因；
2. 提升配置版本；
3. 用新参数重跑之前所有日期；
4. 新旧结果并排比较；
5. 不针对单独某一天做定向调参。

## 每周阶段总结

每积累 5-10 个交易日，整理：

```text
哪些 node type 肉眼上最合理
哪些品种 Profile 最稳定
哪些品种不适合当前参数
最常见的数据问题
最常见的错误 level 类型
自动 outcome 与人工判断的分歧
需要优先修的数据、算法或可视化问题
```

当前阶段只做简单计数和典型截图，不需要复杂统计。

# Node.js 本地浏览网站构想

## 可行性

完全可行，而且很适合当前研究阶段。

网站不需要复制或上传数据，可以直接读取本机按交易日保存的目录。它的作用是把现在的命令行和静态 HTML 串成一个统一入口：

```text
扫描本地数据目录
→ 展示已有交易日
→ 检查处理状态
→ 触发 Python pipeline
→ 浏览每日总览和详情图
→ 填写人工 review
→ 保存 CSV/JSON 标注
```

## 推荐技术栈

第一版保持简单：

```text
Node.js 20+
TypeScript
Fastify 或 Express
SQLite
原生 HTML/CSS/TypeScript，或轻量 React/Vite
现有 ECharts 页面继续复用
Python pipeline 作为子进程执行
```

更推荐：

```text
Fastify + TypeScript + SQLite + React/Vite
```

理由：

- Fastify 适合本地 API、目录扫描和任务日志；
- SQLite 用于保存人工 review、运行记录和���置版本；
- React 用于日期、品种、review 表单和任务状态；
- ECharts 详情页已经完成，可以先嵌入 iframe，避免重新写图表；
- Python 继续负责行情处理和 Profile 计算，Node.js 不重复实现算法。

## 第一版页面

### 交易日列表

```text
日期
原始 tick 是否存在
1min 是否完成
Profile 是否完成
Visual 是否完成
人工 review 进度
```

### 单日总览

直接嵌入：

```text
YYYYMMDD/day_market_profile/visual/index.html
```

同时展示：

```text
selected contracts
levels/events 数量
数据质量异常
当天 review 完成比例
```

### 单品种详情

```text
左侧：现有 ECharts 详情页
右侧：人工 review 表单
```

表单包括：

```text
manual_grade
manual_behavior
profile_quality
zone_quality
comment
```

### Pipeline 运行页

```text
选择交易日
运行 1min
运行 Profile
生成 Visual
查看实时 stdout/stderr
显示成功/失败状态
```

## Node.js 与 Python 的边界

Node.js 负责：

```text
目录扫描
本地 API
任务编排
进程日志
页面展示
人工标注
SQLite 查询
```

Python 负责：

```text
Tick 清洗
1min 聚合
Profile 计算
Level 识别
Event 识别
HTML/PNG 生成
```

不要用 Node.js 重写现有 Python 算法。网站应该是研究工作台，不是第二套计算引擎。

## 本地 API 草案

```text
GET  /api/days
GET  /api/days/:date
POST /api/days/:date/run/1min
POST /api/days/:date/run/profile
POST /api/days/:date/run/visual
GET  /api/days/:date/tasks
GET  /api/days/:date/instruments
GET  /api/days/:date/reviews
PUT  /api/days/:date/reviews/:instrument/:zoneId
```

静态结果：

```text
GET /data/:date/visual/*
```

## Review 数据存储

人工标注建议以 SQLite 为主，同时支持导出 CSV：

```sql
review (
  trading_day TEXT,
  instrument TEXT,
  zone_id TEXT,
  node_type TEXT,
  center REAL,
  lower REAL,
  upper REAL,
  auto_outcome TEXT,
  manual_grade TEXT,
  manual_behavior TEXT,
  profile_quality TEXT,
  zone_quality TEXT,
  comment TEXT,
  reviewed_at TEXT,
  PRIMARY KEY (trading_day, instrument, zone_id)
)
```

`zone_id` 应由以下字段稳定生成：

```text
trading_day + instrument + node_type + lower + upper + available_time
```

## 安全边界

网站只监听本机：

```text
127.0.0.1
```

不要默认监听 `0.0.0.0`，也不要将原始 tick 目录暴露为任意文件浏览器。

API 只允许访问配置好的数据根目录，例如：

```text
~/Downloads
```

所有日期参数必须校验为：

```text
YYYYMMDD
```

Python 子进程使用参数数组调用，禁止拼接 shell 命令，避免路径和命令注入。

## 推荐开发顺序

### 第一阶段：只读浏览

```text
扫描交易日
显示处理状态
浏览现有 index.html
打开单品种详情
```

### 第二阶段：Pipeline 编排

```text
网页触发三条 Python 命令
显示实时日志
记录运行状态和退出码
```

### 第三阶段：人工 Review

```text
Zone 列表
人工等级和行为标签
评论
Review 进度
CSV 导出
```

### 第四阶段：研究汇总

```text
按日期、品种、node type 筛选
查看人工 A/B/C/D 分布
自动 outcome 与人工判断对照
典型截图收藏
```

## 当前建议

本地网站值得做，但第一版不要做成复杂平台。

最小可用版本只需要：

1. 自动扫描交易日；
2. 显示每个日期的处理状态；
3. 嵌入现有 visual/index.html 和 details；
4. 提供人工 review 表单；
5. 一键运行 1min、Profile、Visual 三步 pipeline；
6. 将 review 保存到 SQLite 并可导出 CSV。

这正好支持当前“不断收集 tick、重复运行、肉眼观察、积累判断”的研究阶段。