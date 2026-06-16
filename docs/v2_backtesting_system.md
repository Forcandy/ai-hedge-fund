# v2 回测系统技术文档 (v2 Backtesting System)

源文件目录: `v2/backtesting/`

---

## 1. 模块概述

### 1.1 核心功能

v2 回测系统是一个**策略无关（Strategy-Agnostic）**的交易模拟引擎。策略负责生成 `TradeSignal` 信号列表；引擎接管一切后续工作——价格查询、仓位大小计算、盈亏结算、权益曲线构建与绩效指标计算。

相较于 v1 的逐日驱动 AI Agent 模式，v2 采用"信号先行、执行分离"的架构：策略可以独立测试和组合，引擎无需感知策略内部逻辑。

### 1.2 设计目标

- 策略层与执行层解耦：策略只生成信号，不负责任何价格或资金操作
- 等额美元仓位管理（Equal-Dollar Position Sizing），简单可比
- 支持多空两个方向
- 自动快照入场日（Snap to next trading day）处理非交易日
- 内置 PEAD（Post-Earnings Announcement Drift）策略作为参考实现

### 1.3 模块文件清单

| 文件 | 职责 |
|------|------|
| `v2/backtesting/engine.py` | `BacktestEngine` 核心引擎类 |
| `v2/backtesting/models.py` | Pydantic 数据模型：`TradeSignal`、`Trade`、`PerformanceMetrics`、`BacktestResult` |
| `v2/backtesting/strategy.py` | `Strategy` 抽象基类 + `PEADStrategy` 实现 |
| `v2/backtesting/__main__.py` | CLI 入口，`poetry run python -m v2.backtesting` |
| `v2/backtesting/__init__.py` | 公开导出所有主要类 |

### 1.4 公开导出（`__init__.py`）

```python
from v2.backtesting import (
    BacktestEngine,
    BacktestResult,
    PerformanceMetrics,
    PEADStrategy,
    Strategy,
    Trade,
    TradeSignal,
)
```

### 1.5 系统角色与流程概览

```
CLI / 调用方
    |
    +-- FDClient（数据层）
    |
Strategy.generate_signals(tickers, fd_client)
    |  -> list[TradeSignal]
    |
BacktestEngine.run(strategy, tickers, fd_client)
    |
    +-- run_signals(signals, fd_client)
         |
         +-- _fill_signal(signal, fd_client)  -> Trade | None
         |    (逐信号：查价格 → 快照入场日 → 计算退出日 → 计算P&L)
         |
         +-- _build_equity_curve(trades)  -> list[float]
         |
         +-- _compute_metrics(trades, equity_curve)  -> PerformanceMetrics
         |
         -> BacktestResult
```

---

## 2. 函数列表

### 2.1 `BacktestEngine`（`engine.py`）

| 方法 | 签名 | 说明 |
|------|------|------|
| `__init__` | `(*, capital=100_000.0, per_trade=10_000.0) -> None` | 引擎构造函数，所有参数为关键字参数 |
| `run` | `(strategy, tickers, fd_client) -> BacktestResult` | 完整管线：生成信号后执行回测 |
| `run_signals` | `(signals, fd_client) -> BacktestResult` | 直接执行预构建信号列表 |
| `_fill_signal` | `(signal, fd_client) -> Trade \| None` | 将单条信号转换为成交交易（私有） |
| `_build_equity_curve` | `(trades) -> list[float]` | 构建权益曲线（私有） |
| `_compute_metrics` | `(trades, equity_curve) -> PerformanceMetrics` | 计算统计绩效指标（私有） |

**模块级辅助函数**（位于 `engine.py` 文件末尾）：

| 函数 | 签名 | 说明 |
|------|------|------|
| `_parse_date` | `(s: str) -> date` | 将 `YYYY-MM-DD` 字符串解析为 `date` 对象，取前 10 字符 |
| `_find_next_trading_day` | `(date_str, trading_days) -> str \| None` | 从有序交易日列表中找到第一个 `>= date_str` 的日期 |

### 2.2 `Strategy` 抽象基类（`strategy.py`）

| 成员 | 类型 | 说明 |
|------|------|------|
| `name` | `@property @abstractmethod -> str` | 策略的人类可读名称 |
| `generate_signals` | `@abstractmethod (tickers, fd_client) -> list[TradeSignal]` | 扫描历史数据，输出信号列表 |

### 2.3 `PEADStrategy`（`strategy.py`）

| 成员 | 类型 | 说明 |
|------|------|------|
| `__init__` | `(*, earnings_limit=8, holding_days=5) -> None` | 关键字参数构造，`earnings_limit` 控制每个 ticker 拉取多少期财报 |
| `name` | `str` | 固定返回 `"pead"` |
| `generate_signals` | `(tickers, fd_client) -> list[TradeSignal]` | 扫描盈余惊喜历史，生成多/空信号 |

### 2.4 Pydantic 模型（`models.py`）

| 类 | 字段 |
|----|------|
| `TradeSignal` | `ticker`, `direction`, `entry_date`, `holding_days`, `metadata` |
| `Trade` | `ticker`, `direction`, `entry_date`, `exit_date`, `entry_price`, `exit_price`, `shares`, `pnl`, `return_pct`, `holding_days`, `metadata` |
| `PerformanceMetrics` | `total_return_pct`, `annualized_return_pct`, `sharpe_ratio`, `max_drawdown_pct`, `win_rate`, `n_trades`, `n_long`, `n_short`, `avg_return_pct`, `avg_holding_days` |
| `BacktestResult` | `trades`, `metrics`, `equity_curve` |

---

## 3. 详解

### 3.1 `BacktestEngine.__init__`

```python
engine = BacktestEngine(capital=100_000.0, per_trade=10_000.0)
```

- `capital`：初始资金，权益曲线起点，默认 `100_000.0`
- `per_trade`：每笔交易的名义美元敞口，用于计算持股数量，默认 `10_000.0`

所有参数均为关键字参数（`*` 分隔符强制），不可使用位置调用。

---

### 3.2 `BacktestEngine.run`

```python
result = engine.run(strategy, ["AAPL", "MSFT"], fd_client)
```

**完整管线**，内部顺序：
1. 调用 `strategy.generate_signals(tickers, fd_client)` 获取信号列表
2. 将信号列表交给 `run_signals(signals, fd_client)` 执行

---

### 3.3 `BacktestEngine.run_signals`

```python
result = engine.run_signals(signals, fd_client)
```

**直接执行预构建信号列表**，适用于测试、手动信号列表，或多策略信号合并后统一执行。

执行流程：
1. 若 `signals` 为空，立即返回 `BacktestResult()`（空结果）
2. 对每条信号调用 `_fill_signal`，收集成功转化为 `Trade` 的结果
3. 若所有信号均无法成交（`trades` 为空），返回空 `BacktestResult()`
4. 按 `entry_date` 对交易排序（升序）
5. 调用 `_build_equity_curve(trades)` 构建权益曲线
6. 调用 `_compute_metrics(trades, equity_curve)` 计算绩效
7. 返回 `BacktestResult(trades=..., metrics=..., equity_curve=...)`

---

### 3.4 `BacktestEngine._fill_signal`

将单条 `TradeSignal` 转换为实际成交的 `Trade`，或在无法成交时返回 `None`。

**价格获取窗口**：

```python
price_start = entry - timedelta(days=5)           # 入场日前 5 日
price_end   = entry + timedelta(days=holding_days * 2 + 10)  # 宽余窗口
```

若 `price_end` 超过今日（`date.today()`），自动截断为今日，防止请求未来数据。

**入场日快照（Snap to Next Trading Day）**：

调用 `_find_next_trading_day(signal.entry_date, trading_days)`，找到价格序列中第一个 `>= signal.entry_date` 的实际交易日作为入场日。若在价格序列范围内找不到有效入场日，返回 `None`，该信号被丢弃。

**退出日计算**：

```python
exit_idx = entry_idx + signal.holding_days
```

以入场日在有序交易日列表中的索引为基准，向后偏移 `holding_days` 个交易日。若 `exit_idx >= len(trading_days)`（价格数据不足），返回 `None`。

**仓位大小（Equal-Dollar Sizing）**：

```python
shares = self._per_trade / entry_price
```

持股数量 = 每笔交易美元额 ÷ 入场价，不取整（保留小数股数），存储时四舍五入到 4 位小数。

**盈亏计算**：

| 方向 | P&L 公式 | 收益率公式 |
|------|----------|----------|
| `long` | `shares * (exit_price - entry_price)` | `(exit_price - entry_price) / entry_price` |
| `short` | `shares * (entry_price - exit_price)` | `(entry_price - exit_price) / entry_price` |

`pnl` 四舍五入到 2 位小数，`return_pct` 四舍五入到 6 位小数。

**返回字段**：完整填充的 `Trade` 对象，`metadata` 从 `signal.metadata` 继承。

---

### 3.5 `BacktestEngine._build_equity_curve`

```python
equity = self._capital
curve = [equity]          # 初始点：起始资金
for t in trades:
    equity += t.pnl
    curve.append(round(equity, 2))
```

- 曲线长度 = `n_trades + 1`（首元素为初始资金，后续每成交一笔追加一个点）
- 交易已按 `entry_date` 排序，曲线反映时间顺序的累计盈亏
- 每个点四舍五入到 2 位小数

---

### 3.6 `BacktestEngine._compute_metrics`

输入：`trades`（已排序）、`equity_curve`（长度 = `n_trades + 1`）

计算以下统计量并返回 `PerformanceMetrics`：

| 指标 | 计算逻辑 |
|------|----------|
| `total_return_pct` | `(final_equity - capital) / capital` |
| `annualized_return_pct` | `(1 + total_return_pct)^(1/years) - 1` |
| `sharpe_ratio` | `(avg / std) * sqrt(trades_per_year)` |
| `max_drawdown_pct` | 遍历权益曲线求峰谷回撤 |
| `win_rate` | 收益率 `> 0` 的交易数 / 总交易数 |
| `n_trades` | 总交易笔数 |
| `n_long` | 多头交易笔数 |
| `n_short` | 空头交易笔数 |
| `avg_return_pct` | 所有交易 `return_pct` 的算术平均 |
| `avg_holding_days` | 所有交易 `holding_days` 的算术平均，四舍五入到 1 位小数 |

---

### 3.7 `Strategy` 抽象基类

自定义策略只需：
1. 继承 `Strategy`
2. 实现 `name` 属性
3. 实现 `generate_signals(tickers, fd_client) -> list[TradeSignal]`

信号顺序不重要，引擎会按 `entry_date` 排序。

---

### 3.8 `PEADStrategy.generate_signals`

对每个 ticker：
1. 调用 `fd_client.get_earnings_history(ticker, limit=self._earnings_limit)` 获取财报历史
2. 过滤：跳过 `filing_date` 或 `quarterly` 为空的记录；跳过 `eps_surprise` 不为 `"BEAT"` 或 `"MISS"` 的记录
3. **45 天过滤**：若 `(filing_date - report_period).days >= 45`，跳过该记录（过晚披露的财报不具备 PEAD 效应）
4. **去重（每个 `report_period` 只保留最优先的一条）**：以 `f"{ticker}:{report_period}"` 为键，按 `source_type` 优先级取最优记录

   | source_type | 优先级值（越小越优先） |
   |-------------|----------------------|
   | `8-K` | 0 |
   | `10-Q` | 1 |
   | `10-K` | 2 |
   | `20-F` | 3 |
   | 其他 | 99 |

5. 生成 `TradeSignal`：

   | 盈余惊喜 | 方向 | `entry_date` |
   |---------|------|--------------|
   | `BEAT` | `long` | `record.filing_date` |
   | `MISS` | `short` | `record.filing_date` |

   `metadata` 中记录 `eps_surprise`、`source_type`、`report_period`。引擎会将 `entry_date` 快照到下一个实际交易日。

---

## 4. Pydantic 模型详解

### 4.1 `TradeSignal`

策略发出的入场指令。引擎处理一切执行细节，策略无需关心价格和资金。

| 字段 | 类型 | 说明 |
|------|------|------|
| `ticker` | `str` | 股票代码，如 `"AAPL"` |
| `direction` | `str` | `"long"` 或 `"short"` |
| `entry_date` | `str` | 期望入场日期，格式 `YYYY-MM-DD`；引擎自动快照到下一个交易日 |
| `holding_days` | `int` | 持有的**交易日**数量（非自然日） |
| `metadata` | `dict[str, Any]` | 策略自定义上下文，默认空字典，传递至 `Trade.metadata` |

---

### 4.2 `Trade`

一笔已完成交易的完整记录。

| 字段 | 类型 | 说明 |
|------|------|------|
| `ticker` | `str` | 股票代码 |
| `direction` | `str` | `"long"` 或 `"short"` |
| `entry_date` | `str` | 实际入场日期（快照后），格式 `YYYY-MM-DD` |
| `exit_date` | `str` | 实际退出日期，格式 `YYYY-MM-DD` |
| `entry_price` | `float` | 入场价（收盘价） |
| `exit_price` | `float` | 退出价（收盘价） |
| `shares` | `float` | 持股数量（`per_trade / entry_price`，精确到 4 位小数） |
| `pnl` | `float` | 美元盈亏（精确到 2 位小数） |
| `return_pct` | `float` | 百分比收益率（精确到 6 位小数，带符号） |
| `holding_days` | `int` | 实际持有交易日数 |
| `metadata` | `dict[str, Any]` | 来自对应 `TradeSignal.metadata`，默认空字典 |

---

### 4.3 `PerformanceMetrics`

一组交易的汇总统计。所有字段均为非 Optional 的 `float` 或 `int`（Pydantic `BaseModel`，无 `total=False`）。

| 字段 | 类型 | 说明 |
|------|------|------|
| `total_return_pct` | `float` | 总收益率（小数形式，如 `0.123` 表示 12.3%） |
| `annualized_return_pct` | `float` | 年化收益率（小数形式） |
| `sharpe_ratio` | `float` | 夏普比率（基于每笔交易收益率年化，精确到 4 位小数） |
| `max_drawdown_pct` | `float` | 最大回撤（小数形式，非负，如 `0.05` 表示 5% 回撤） |
| `win_rate` | `float` | 胜率（精确到 4 位小数） |
| `n_trades` | `int` | 总交易笔数 |
| `n_long` | `int` | 多头交易笔数 |
| `n_short` | `int` | 空头交易笔数 |
| `avg_return_pct` | `float` | 平均每笔收益率（小数形式，精确到 6 位小数） |
| `avg_holding_days` | `float` | 平均持有交易日（精确到 1 位小数） |

---

### 4.4 `BacktestResult`

引擎返回的顶层结果对象。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `trades` | `list[Trade]` | `[]` | 所有成交交易，按 `entry_date` 升序排列 |
| `metrics` | `PerformanceMetrics \| None` | `None` | 信号为空或所有信号均无法成交时为 `None` |
| `equity_curve` | `list[float]` | `[]` | 权益曲线，长度 = `n_trades + 1`，首元素为初始资金 |

---

## 5. 信号生成——PEAD 策略详解

### 5.1 策略原理

PEAD（Post-Earnings Announcement Drift，盈余公告后漂移）是一种成熟的市场异常：股价在盈余公告后往往沿惊喜方向持续漂移数天至数周。v2 实现的逻辑：
- EPS 超预期（BEAT）→ 做多（long）
- EPS 低于预期（MISS）→ 做空（short）

信号入场日为财报**披露日（filing_date）**，由引擎快照到下一个实际交易日，模拟公告发布后第一个开盘日的操作。

### 5.2 去重逻辑

同一公司同一报告期（`report_period`）可能通过多种文件类型（8-K 紧急公告、10-Q 季报、10-K 年报、20-F 外国私人发行人年报）披露同一财报数据。去重规则：

```
key = "{ticker}:{report_period}"
优先保留：8-K（priority=0）> 10-Q（1）> 10-K（2）> 20-F（3）> 其他（99）
```

只保留每个 key 对应 priority 最小的那条记录，确保使用最早披露的信息，符合实际交易中"首次公告"的时效性。

### 5.3 45 天过滤规则

```python
if (filing_date - report_period).days >= 45:
    continue
```

若披露日与报告期结束日之间超过 45 天，说明该财报为补充披露或修订文件，不具备有效的 PEAD 时效性，予以跳过。

### 5.4 信号 metadata 字段

| key | 值 | 说明 |
|-----|----|------|
| `eps_surprise` | `"BEAT"` 或 `"MISS"` | EPS 相对预期的方向 |
| `source_type` | 如 `"8-K"`、`"10-Q"` | 财报文件类型 |
| `report_period` | 如 `"2024-09-30"` | 报告期结束日期 |

这些字段在 CLI 输出中用于彩色显示（`BEAT` 绿色，`MISS` 红色）。

---

## 6. 算法汇总

### 6.1 年化时间跨度

```python
first_entry  = parse_date(trades[0].entry_date)
last_exit    = parse_date(trades[-1].exit_date)
calendar_days = (last_exit - first_entry).days
years = max(calendar_days / 365.25, 0.01)   # 下限 0.01 年，防止除零
```

使用首笔交易入场日到最后一笔交易退出日的**日历天数**（非交易日数），以 365.25 天/年换算，并设 0.01 年下限。

### 6.2 年化收益率

```python
annualized = (1 + total_return_pct) ** (1 / years) - 1
```

标准复利年化公式。`total_return_pct` 基于权益曲线首尾之差相对初始资金的比率。

### 6.3 夏普比率

**基于每笔交易收益率**计算（非日收益率），使用实际交易频率年化：

```python
arr = np.array([t.return_pct for t in trades])   # 每笔交易收益率序列
avg = arr.mean()
std = arr.std(ddof=1) if n > 1 else 1.0           # 样本标准差；单笔交易时 std=1.0 避免除零
trades_per_year = n / years                        # 每年平均交易频率
sharpe = (avg / std) * np.sqrt(trades_per_year) if std > 0 else 0.0
```

**关键差异**：不使用无风险利率，不使用固定年化因子（如 `sqrt(252)`），而是使用实际交易频率 `n / years` 作为年化因子，使结果与策略交易频率自适应匹配。

精确到 4 位小数。

### 6.4 最大回撤

从权益曲线（`list[float]`）逐点遍历计算，使用**历史峰值追踪法**：

```python
peak = equity_curve[0]
max_dd = 0.0
for val in equity_curve:
    if val > peak:
        peak = val
    dd = (peak - val) / peak        # 当前回撤
    if dd > max_dd:
        max_dd = dd
```

- 结果为**非负小数**（如 `0.05` 表示 5% 最大回撤）
- 与 v1 的 `max_drawdown` 不同：v1 乘以 100 返回百分比值（如 `-15.3`）；v2 直接返回小数
- 精确到 6 位小数

### 6.5 胜率

```python
wins = sum(1 for r in returns if r > 0)
win_rate = round(wins / n, 4) if n > 0 else 0.0
```

以收益率严格大于 0 为胜，不计回报率恰好为 0 的交易（平局）。

### 6.6 价格窗口获取

每条信号独立向 FDClient 请求价格数据，请求窗口：
- 起点：`entry_date - 5 日`（覆盖入场前的交易日，确保能找到快照点）
- 终点：`entry_date + holding_days * 2 + 10 日`（覆盖退出日的宽余窗口）
- 终点不超过今日（防止请求未来日期）

### 6.7 仓位等额美元原则

所有交易使用相同名义美元敞口 `per_trade`，不论票价高低。这使得不同价格区间的个股之间的收益率可直接比较，且不会因仓位大小差异干扰整体绩效统计。

---

## 7. CLI 详解（`__main__.py`）

### 7.1 运行命令

```bash
poetry run python -m v2.backtesting
```

### 7.2 默认参数

| 常量 | 值 | 说明 |
|------|-----|------|
| `HOLDING_DAYS` | `5` | PEAD 策略持有交易日数 |
| `CAPITAL` | `100_000.0` | 初始资金 |
| `PER_TRADE` | `10_000.0` | 每笔交易美元敞口 |

### 7.3 默认股票池（共 100 只）

| 板块 | 数量 | 代表股票 |
|------|------|---------|
| 科技 | 21 | AAPL, MSFT, AMZN, GOOGL, META, NVDA, TSLA, NFLX... |
| 金融 | 15 | JPM, GS, BAC, WFC, MS, C, BLK, SCHW... |
| 医疗健康 | 15 | JNJ, PFE, UNH, MRK, LLY, ABBV, TMO... |
| 能源 | 7 | XOM, CVX, COP, SLB, EOG, MPC, PSX |
| 消费/零售 | 15 | HD, LOW, COST, WMT, KO, PEP, MCD... |
| 工业 | 10 | CAT, DE, HON, UPS, RTX, BA, LMT... |
| 媒体/电信 | 7 | DIS, CMCSA, T, VZ, TMUS, CHTR, WBD |
| 其他 | 10 | V, MA, PYPL, NEE, D, SO, DUK, ABNB, COIN, NOW |

### 7.4 执行阶段

**阶段一：扫描财报信号**

逐只股票调用 `PEADStrategy.generate_signals`，终端实时显示进度：

```
  Scanning earnings... [42/100] MSFT
```

完成后显示找到的信号总数。

**阶段二：执行信号**

调用 `engine.run_signals(signals, fd)` 批量执行所有信号。

**阶段三：逐笔动态展示**

遍历已排序交易列表，每加入一笔交易后清屏重绘，展示：
- 顶部面板：组合价值、总收益率、夏普比率、最大回撤、交易笔数、多空分布、胜率、平均收益
- 交易明细表：按时间序逆序排列（最新在上），包含日期、代码、方向、EPS 惊喜类型、入场价、退出价、持股数、P&L、收益率

**颜色方案**：

| 颜色 | 含义 |
|------|------|
| 绿色（`\033[32m`） | 多头（LONG）、正收益、BEAT |
| 红色（`\033[31m`） | 空头（SHORT）、负收益、MISS |
| 青色（`\033[36m`） | 股票代码 |
| 黄色（`\033[33m`） | 夏普 0~1（中性） |
| 灰色（`\033[90m`） | 表头、分隔线 |

---

## 8. 使用示例

### 8.1 基本用法

```python
from v2.data import FDClient
from v2.backtesting import BacktestEngine, PEADStrategy

with FDClient() as fd:
    strategy = PEADStrategy(holding_days=5)
    engine = BacktestEngine(capital=100_000, per_trade=10_000)
    result = engine.run(strategy, ["AAPL", "MSFT", "NVDA"], fd)

print(f"总收益率: {result.metrics.total_return_pct:.2%}")
print(f"夏普比率: {result.metrics.sharpe_ratio:.2f}")
print(f"最大回撤: {result.metrics.max_drawdown_pct:.2%}")
print(f"交易笔数: {result.metrics.n_trades}")
```

### 8.2 直接执行预构建信号

```python
from v2.backtesting.models import TradeSignal
from v2.backtesting import BacktestEngine

signals = [
    TradeSignal(ticker="AAPL", direction="long", entry_date="2024-02-02", holding_days=5),
    TradeSignal(ticker="META", direction="short", entry_date="2024-02-29", holding_days=3),
]

with FDClient() as fd:
    engine = BacktestEngine(capital=50_000, per_trade=5_000)
    result = engine.run_signals(signals, fd)
```

### 8.3 自定义策略

```python
from v2.backtesting.strategy import Strategy
from v2.backtesting.models import TradeSignal

class MomentumStrategy(Strategy):
    @property
    def name(self) -> str:
        return "momentum_52w"

    def generate_signals(self, tickers, fd_client):
        signals = []
        for ticker in tickers:
            # 自定义逻辑：查价格、判断动量、生成信号
            ...
            signals.append(TradeSignal(
                ticker=ticker,
                direction="long",
                entry_date="2024-01-15",
                holding_days=10,
            ))
        return signals
```

---

## 9. 依赖关系

### 9.1 外部 Python 包

| 包 | 用途 | 所在文件 |
|----|------|---------|
| `numpy` | 数组运算（mean、std、sqrt）、夏普比率计算 | `engine.py` |
| `pydantic` | 数据模型定义（`BaseModel`、`Field`） | `models.py` |
| `dotenv`（`python-dotenv`） | CLI 入口加载 `.env` 文件 | `__main__.py` |

### 9.2 项目内部依赖

| 模块 | 导入项 | 用途 | 调用方 |
|------|--------|------|--------|
| `v2.data.client` | `FDClient` | 价格查询（`get_prices`）、财报历史查询（`get_earnings_history`） | `engine.py`, `strategy.py`, `__main__.py` |
| `v2.data` | `FDClient` | 同上（包级别便捷导入） | `__main__.py` |
| `v2.backtesting.models` | `TradeSignal`, `Trade`, `PerformanceMetrics`, `BacktestResult` | 数据模型 | `engine.py`, `strategy.py` |
| `v2.backtesting.strategy` | `Strategy` | 抽象基类 | `engine.py` |
| `v2.backtesting.engine` | `BacktestEngine` | 引擎（CLI 中内部重复导入用于运行时指标计算） | `__main__.py` |

### 9.3 标准库

| 模块 | 用途 | 所在文件 |
|------|------|---------|
| `__future__.annotations` | 延迟类型注解（PEP 563） | `engine.py`, `strategy.py`, `__main__.py` |
| `abc` | `ABC`、`abstractmethod` | `strategy.py` |
| `datetime` | `date`、`datetime`、`timedelta` | `engine.py`, `strategy.py` |
| `logging` | 日志记录 | `engine.py`, `__main__.py` |
| `os` | `os.system("clear")` 用于终端清屏 | `__main__.py` |
| `sys` | 进度输出（`sys.stdout.write`/`flush`） | `__main__.py` |
| `time` | `time.sleep`（动态展示节奏控制） | `__main__.py` |

### 9.4 运行环境

- **Python 版本**：3.11+（使用 `X | Y` 联合类型语法、`from __future__ import annotations`）
- **包管理器**：Poetry
- **API 密钥**：需要 `FINANCIAL_DATASETS_API_KEY` 环境变量（或 `.env` 文件）以访问完整股票数据；免费可用代码：`AAPL`、`GOOGL`、`MSFT`、`NVDA`、`TSLA`
