---
tags:
  - 量化
  - 回测
  - tick
  - 中金所
  - 数据
created: 2026-11-03
source: Alma 整理
project: ~/src/kline
---

# 中金所历史 Tick 数据：清洗结果与使用指南

> [!summary] 一句话
> 2022–2025 四年中金所 8 个品种的 tick 已清洗成 Parquet，放在 `~/data/kline/ctp_tick/CFFEX/`，共 **969 个交易日、4.74 亿行、约 2.9G**。字段与 watcher 的 `ctp_tick` 对齐，用 DuckDB 直接 SQL 查询即可。

## 1. 今天做了什么（2026-11-03）

1. **摸清原始数据**：192.168.5.13 的 `/opt/xunlei/downloads/<年>/` 下是按月打包的 zip，每个交易日、每个合约一个 15 列 CSV（CTP 字段子集）。2025 年全市场解压约 352G，中金所只占 5%。
2. **确定范围**：只做中金所（IF/IH/IC/IM、T/TF/TS/TL），只要 tick 表，不做日表，不在本机保留原始 CSV。
3. **定清洗原则**：坏数据直接丢弃、计入统计表；清洗后不依赖原始数据；异常靠事后看统计表发现。
4. **写清洗程序** `kline/cffex.py` + 12 个单测，单日约 2 秒，一年约 80–110 秒。
5. **清洗完 2022–2025**：

| 年份 | 交易日 | 行数 | 丢弃 | 体积 |
| --- | --- | --- | --- | --- |
| 2022 | 242 | 119,408,235 | 0 | 598M |
| 2023 | 242 | 127,792,408 | 112 | 631M |
| 2024 | 242 | 108,361,524 | 56 | 788M |
| 2025 | 243 | 118,811,982 | 0 | 851M |

**过程中发现并处理的问题：**

- **TS（二债）最小变动价位在 20231107 由 0.005 改为 0.002**。一开始按 0.002 处理，2022、2023 年会误丢三千多万行；现在按日期取值。
- **20251103 的 `IC2511` 原文件是空的**：从同日的 `IC当月连续` 文件补回 25,093 行。
- **涨跌停价有浮点噪声**（如 `3452.6000000000013`）：价格统一规整到最小变动价位。
- **丢掉的 168 行都该丢**：盘前 07:0x 的空快照、收盘后重复写入的快照。
- 量类字段（成交量、挂单量、持仓）全年逐行核查，没有空值、负数或小数。

## 2. 数据在哪

```text
~/data/kline/
├── ctp_tick/CFFEX/<年>/<YYYYMMDD>.parquet   tick，一个交易日一个文件，含当天全部合约
├── stats/CFFEX/<年>/<YYYYMMDD>.parquet      统计，每个交易日、每个合约一行
└── logs/build_<年>.log                      清洗日志
```

| 品种 | 上市 | 每天合约数 | 乘数 | 最小变动价位 | 连续竞价 |
| --- | --- | --- | --- | --- | --- |
| IF、IH | 早于 2022 | 4 | 300 | 0.2 | 09:30–11:30、13:00–15:00 |
| IC | 早于 2022 | 4 | 200 | 0.2 | 同上 |
| IM | 2022-07-22 | 4 | 200 | 0.2 | 同上 |
| T、TF | 早于 2022 | 3 | 10000 | 0.005 | 09:30–11:30、13:00–15:15 |
| TS | 早于 2022 | 3 | 20000 | 0.005 → 0.002（20231107 起） | 同上 |
| TL | 2023-04-21 | 3 | 10000 | 0.01 | 同上 |

国债到期合约在最后交易日（合约月第二个周五）只交易到 11:30。

## 3. 字段

| 列 | 类型 | 说明 |
| --- | --- | --- |
| `ts` | timestamp(us, UTC) | 交易所时间。查询前 `SET TimeZone='Asia/Shanghai'` 就显示为北京时间 |
| `symbol` | string | 合约，如 `IF2509` |
| `trading_day` | string | `YYYYMMDD`。中金所没有夜盘，就是 `ts` 的日期 |
| `product` | string | 品种，如 `IF` |
| `last_price` | double | 最新价，已规整到最小变动价位 |
| `volume` | int64 | **当日累计**成交量，不是增量 |
| `turnover` | double | **当日累计**成交额 |
| `open_interest` | int64 | 持仓量 |
| `bid_price1` / `ask_price1` | double，**可空** | 买一 / 卖一价；NULL 表示该方向没有挂单（通常是封板） |
| `bid_volume1` / `ask_volume1` | int64 | 买一 / 卖一量；价格为 NULL 时为 0 |
| `seq` | int32 | 该合约当天在原文件里的行号，**同一时刻内排序用** |
| `flags` | uint8 | 时段标记，见下 |

`flags` 按位：

| 位 | 值 | 含义 |
| --- | --- | --- |
| AUCTION | 1 | 09:29:00 的集合竞价结果 |
| BREAK | 2 | 午休时段（11:30:01–12:59:59） |
| POST_CLOSE | 4 | 收盘后的快照（股指 15:00 后、国债 15:15 后、国债到期日 11:30 后） |
| NO_TRADE | 8 | 当天还没成交过（`volume = 0`） |

> [!tip] 最常用的过滤
> 只要连续竞价时段：`flags & 7 = 0`。
> 想把集合竞价也算进开盘：`flags & 6 = 0`。

## 4. 怎么用

工具：DuckDB（Python 或命令行都行），直接读 Parquet，不用导入。在 `~/src/kline` 下用 `uv run python` 即可，依赖已装好。

```python
import duckdb
con = duckdb.connect()
con.execute("SET TimeZone='Asia/Shanghai'")
TICK = "read_parquet('~/data/kline/ctp_tick/CFFEX/*/*.parquet')"
STATS = "read_parquet('~/data/kline/stats/CFFEX/*/*.parquet')"
```

按 `symbol`、`trading_day` 过滤会利用文件统计信息跳过无关数据，跨四年查也很快（实测扫 2024 全年 IF+IH 计数 0.07 秒）。

### 4.1 单合约回放

```sql
SELECT * FROM read_parquet('~/data/kline/ctp_tick/CFFEX/2025/*.parquet')
WHERE symbol = 'IF2509' AND trading_day BETWEEN '20250801' AND '20250831'
  AND flags & 7 = 0
ORDER BY ts, seq;
```

转成 pandas：在 `con.sql(...)` 后面加 `.df()`。

### 4.2 1 分钟 K 线

成交量和成交额都是当日累计，**先差分再聚合**：

```sql
WITH t AS (
  SELECT ts, seq, last_price,
         volume   - lag(volume,   1, 0) OVER w AS dvol,
         turnover - lag(turnover, 1, 0) OVER w AS dturn
  FROM read_parquet('~/data/kline/ctp_tick/CFFEX/2025/20250902.parquet')
  WHERE symbol = 'IF2509'
  WINDOW w AS (ORDER BY seq)
)
SELECT time_bucket(INTERVAL 1 MINUTE, ts) AS bar,
       arg_min(last_price, seq) AS open,  max(last_price) AS high,
       min(last_price)          AS low,   arg_max(last_price, seq) AS close,
       sum(dvol) AS vol,
       sum(dturn) / nullif(sum(dvol), 0) / 300 AS vwap   -- 300 = IF 乘数
FROM t GROUP BY bar ORDER BY bar;
```

- 差分要在过滤 `flags` **之前**做，否则午休、收盘后的成交量会并到下一根 K 线。
- 09:29 的集合竞价会单独成一根 K 线；不想要就在外层过滤，或把它并入 09:30。
- VWAP 要除以合约乘数，见第 2 节的表。

### 4.3 每天的主力合约

统计表里有每个合约当天收盘的累计成交量，按它选：

```sql
SELECT trading_day, arg_max(symbol, last_volume) AS main
FROM read_parquet('~/data/kline/stats/CFFEX/*/*.parquet')
WHERE product = 'IF'
GROUP BY trading_day ORDER BY trading_day;
```

> [!warning] 防未来数据
> 上面用的是**当天**收盘成交量，回测时当天开盘并不知道。实际使用要整体错一天：用前一交易日选出的合约作为当天的主力，或者改用持仓量。

### 4.4 Python 里直接拿 Arrow 表

```python
import pyarrow.dataset as ds
import pyarrow.compute as pc

d = ds.dataset("/Users/ralmia/data/kline/ctp_tick/CFFEX/2025", format="parquet")
tb = d.to_table(filter=(pc.field("symbol") == "IM2512") & (pc.field("flags") == 0))
```

## 5. 回测时要注意

- **排序键是 `(ts, symbol, seq)`**。中金所每秒两次推送，同一合约同一时刻可能有多行，靠 `seq` 区分先后。
- **`ts` 是交易所时间，不是你收到行情的时间**。历史数据没有本机接收时间，下单延迟要自己建模（可以参考线上 QuestDB `insert_at - ts` 的分布）。
- **盘口为 NULL 就不能在那个方向成交**。比如 20250407 股指跌停时买一为空，不要前向填充，否则回测会在跌停板上卖出。
- **tick 是快照，不是逐笔**。两次快照之间的成交细节看不到，K 线的高低点也可能漏掉瞬时极值。
- **持仓量统计只用连续竞价时段**。收盘后的快照里偶尔有持仓调整（到期归零等）。
- **合约选择不要依赖数据商的“连续”文件**，自己按成交量或持仓量推算（见 4.3）。

## 6. 发现数据异常时

统计表 `stats` 每个合约每天一行：`rows`、`dropped`（及各原因）、`first_ts`、`last_ts`、`max_gap_s`（连续竞价内最大断档秒数）、`last_volume`、`last_open_interest`、`recovered`。

```sql
-- 行数明显偏少的日子（建议只看股指，国债成交清淡，天然波动大）
WITH d AS (
  SELECT trading_day, product, sum(rows) AS n
  FROM read_parquet('~/data/kline/stats/CFFEX/*/*.parquet')
  WHERE product IN ('IF', 'IH', 'IC', 'IM')
  GROUP BY trading_day, product
)
SELECT *, n / median(n) OVER (PARTITION BY product) AS ratio
FROM d QUALIFY ratio < 0.7 ORDER BY ratio;

-- 有丢弃或长时间断档的合约日
SELECT * FROM read_parquet('~/data/kline/stats/CFFEX/*/*.parquet')
WHERE dropped > 0 OR max_gap_s > 60 ORDER BY trading_day;
```

第二条在四年数据上命中 4,871 个合约日，其中 93% 是国债，大多是远月或临近到期、几乎没人交易的合约，属正常。股指只有约 130 个，主力合约各 6 个左右。所以**实际排查时建议只看当天的主力合约**（第 4.3 节），噪声会少很多。

第一条的结果几乎全是 IH：它是股指里成交最清淡的，行情平淡时行数只有中位数的六成多。

## 7. 重跑和扩展

```bash
cd ~/src/kline
uv run python -m kline.cffex --year 2025 --jobs 4          # 已有的交易日自动跳过
uv run python -m kline.cffex --days 20250902 --force        # 覆盖指定交易日
uv run python -m kline.cffex --month 202511 --force         # 覆盖一个月
uv run python scripts/dry_run.py 2021                       # 只预检不写文件
```

- 需要先在 Mac 上挂载 SMB：`/Volumes/xunlei-downloads`。
- 原始 zip 删掉后就无法重跑，规则改了只能在现有 Parquet 上修补。
- 2021 年还没处理；其他交易所、2026 年的补数据（格式不同）都不在范围内。

## 相关

- 方案文档：`~/src/kline/docs/02-中金所tick清洗方案.md`
- 前期分析（含全市场数据特征）：`~/src/kline/docs/01-前期分析与规划.md`
- 线上采集：`~/src/ctp.luajit/examples/watcher`（字段口径来源）
- [[Tick Volume Profile 每日研究流程与本地浏览网站]]
