# v2 回测系统技术文档 (v2 Backtesting System)

源文件目录: `v2/backtesting/`

---

## 1. 模块概述

### 1.1 核心功能

v2 回测系统是一个**Alpha 模型无关（Alpha-Model-Agnostic）**的交易模拟引擎。引擎直接驱动任意 `AlphaModel`（定义于 `v2/signals/base.py`，量化模型与 LLM 投资人 Agent 的共同接口）在历史交易日网格上逐日形成观点（`Signal`），并接管一切后续工作——价格查询、入场时机判断、仓位大小计算、盈亏结算、权益曲线构建与绩效指标计算。

> **架构变更提示**：早期版本的 v2 回测系统曾采用 `Strategy` 抽象基类 + `PEADStrategy` + `TradeSignal` 的"批量生成信号 → 引擎执行"两阶段架构（`v2/backtesting/strategy.py`）。该文件已被删除，取而代之的是与 `v2/signals/` 模块共享的 `AlphaModel` 接口——量化模型（`QuantModel`，如 `PEADModel`）与 LLM 投资人 Agent（`LLMAgent`，如 `BuffettAgent`）现在使用同一套接口、同一个回测引擎，无需再为每种模型单独适配"生成信号"逻辑。

### 1.2 设计目标

- **观点与执行分离**：`AlphaModel.predict(ticker, date, data_client)` 只负责形成观点（`Signal`，`[-1, +1]` 区间的信念值），不涉及任何价格或资金操作；引擎逐日调用该接口并自行决定何时开仓、平仓
- 等额美元仓位管理（Equal-Dollar Position Sizing），简单可比
- 支持多空两个方向
- 边沿触发（Edge-Triggered）开仓：只有当模型的观点从"无信号"重新转为"有信号"时才开新仓，避免同一信号窗口内重复开仓
- 同一套引擎既可驱动纯数学的量化模型（`PEADModel`），也可驱动调用 LLM 的投资人 Agent（`BuffettAgent`）

### 1.3 模块文件清单

| 文件 | 职责 |
|------|------|
| `v2/backtesting/engine.py` | `BacktestEngine` 核心引擎类 |
| `v2/backtesting/models.py` | Pydantic 数据模型：`Trade`、`PerformanceMetrics`、`BacktestResult` |
| `v2/backtesting/__main__.py` | CLI 入口，`poetry run python -m v2.backtesting`（PEAD 演示） |
| `v2/backtesting/__init__.py` | 公开导出所有主要类 |

`AlphaModel`、`QuantModel`、`LLMAgent`、`PEADModel`、`BuffettAgent`、`Signal` 已迁移至 `v2/signals/` 与顶层 `v2/models.py`，详见 [`v2_signals_system.md`](./v2_signals_system.md)。

### 1.4 公开导出（`__init__.py`）

```python
from v2.backtesting import (
    BacktestEngine,
    BacktestResult,
    PerformanceMetrics,
    Trade,
)
```

### 1.5 系统角色与流程概览

```
CLI / 调用方
    |
    +-- DataClient（数据层，FDClient 或 CachedDataClient）
    |
    +-- AlphaModel 实例（PEADModel、BuffettAgent 等，来自 v2.signals）
    |
BacktestEngine.run_alpha(model, tickers, data_client, start_date, end_date, ...)
    |
    +-- _trade_ticker(model, ticker, data_client, ...)  # 对每只股票独立执行
         |
         +-- 逐交易日调用 model.predict(ticker, date, data_client)  -> Signal
         |    （边沿触发：只在从"无观点"转为"有观点"时开仓）
         |
         +-- _build_trade(...)  -> Trade | None
         |    （按 holding_days 计算退出日，等额美元定价）
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
| `run_alpha` | `(model, tickers, data_client, start_date, end_date, *, threshold=0.0, holding_days=5) -> BacktestResult` | 唯一的公开入口：驱动 `AlphaModel` 逐日形成观点并执行回测 |
| `_trade_ticker` | `(model, ticker, data_client, start_date, end_date, *, threshold, holding_days) -> list[Trade]`（私有） | 对单只股票走完整交易日网格，开仓/平仓 |
| `_build_trade` | `(ticker, direction, entry_date, exit_date, price_map, holding_days, reasoning, metadata) -> Trade \| None`（私有） | 将一次开平仓事件转换为成交 `Trade` |
| `_build_equity_curve` | `(trades) -> list[float]`（私有） | 构建权益曲线 |
| `_compute_metrics` | `(trades, equity_curve) -> PerformanceMetrics`（私有） | 计算统计绩效指标 |

**模块级辅助函数**（位于 `engine.py` 文件末尾）：

| 函数 | 签名 | 说明 |
|------|------|------|
| `_parse_date` | `(s: str) -> date` | 将 `YYYY-MM-DD` 字符串解析为 `date` 对象，取前 10 字符 |

> `run`、`run_signals`、`_fill_signal`、`_find_next_trading_day` 等方法（旧 `Strategy` 架构下的方法名）已不存在——引擎不再区分"生成信号"和"执行信号"两个阶段，`run_alpha` 内部直接逐日调用 `model.predict()`。

### 2.2 Pydantic 模型（`models.py`）

| 类 | 字段 |
|----|------|
| `Trade` | `ticker`, `direction`, `entry_date`, `exit_date`, `entry_price`, `exit_price`, `shares`, `pnl`, `return_pct`, `holding_days`, `reasoning`, `metadata` |
| `PerformanceMetrics` | `total_return_pct`, `annualized_return_pct`, `sharpe_ratio`, `max_drawdown_pct`, `win_rate`, `n_trades`, `n_long`, `n_short`, `avg_return_pct`, `avg_holding_days` |
| `BacktestResult` | `trades`, `metrics`, `equity_curve` |

> `TradeSignal` 模型已删除——`AlphaModel.predict()` 的返回类型是 `v2.models.Signal`（跨 `v2/signals/`、`v2/backtesting/` 共用），不再是回测子模块私有的信号类型。

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

### 3.2 `BacktestEngine.run_alpha`

```python
result = engine.run_alpha(
    PEADModel(), ["AAPL", "MSFT"], data_client,
    "2024-06-01", "2024-12-01", holding_days=5,
)
```

**唯一的公开入口**，对 `tickers` 中每只股票独立调用 `_trade_ticker`，汇总所有交易后统一计算权益曲线和绩效指标。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `model` | `AlphaModel` | — | 待回测的 Alpha 模型实例（如 `PEADModel()`、`BuffettAgent()`） |
| `tickers` | `list[str]` | — | 回测股票池 |
| `data_client` | `DataClient` | — | 数据源（`FDClient` 或 `CachedDataClient` 均可） |
| `start_date` | `str` | — | 信号评估起始日期（`YYYY-MM-DD`） |
| `end_date` | `str` | — | 信号评估截止日期（`YYYY-MM-DD`） |
| `threshold` | `float` | `0.0` | 触发交易所需的最小 `\|conviction\|`；`0.0` 表示任何非零观点都会触发 |
| `holding_days` | `int` | `5` | 每笔持仓的持有交易日数 |

**执行流程**：
1. 对 `tickers` 中每只股票调用 `_trade_ticker`，收集所有 `Trade`
2. 若无任何成交，返回空 `BacktestResult()`
3. 按 `entry_date` 对全部交易排序（升序）
4. 调用 `_build_equity_curve(trades)` 构建权益曲线
5. 调用 `_compute_metrics(trades, equity_curve)` 计算绩效
6. 返回 `BacktestResult(trades=..., metrics=..., equity_curve=...)`

---

### 3.3 `BacktestEngine._trade_ticker`

对单只股票走完整交易日网格，逐日调用 `model.predict()` 形成观点，并按边沿触发规则开平仓。

**价格获取窗口（带 padding）**：

```python
end_padded = end_date + timedelta(days=holding_days * 2 + 10)
if end_padded > today:
    end_padded = today
prices = data_client.get_prices(ticker, start_date, end_padded)
```

向后多取 `holding_days * 2 + 10` 天的价格，确保 `end_date` 附近开仓的交易仍有足够的未来交易日可用于平仓定价；`end_padded` 不超过今日，避免请求未来数据。若该股票无任何价格数据，直接返回空列表。

**交易日网格**：

```python
all_days = sorted(price_map)                                    # 价格序列覆盖的全部交易日（含 padding）
grid = [d for d in all_days if start_date <= d <= end_date]      # 实际逐日评估信号的范围
```

**边沿触发开仓（Edge-Triggered Arming）**：

引擎维护一个 `armed` 布尔标志（初始为 `True`）：

- 仅当 `armed=True` 且 `abs(signal.value) > threshold` 时才开新仓；开仓后立即将 `armed` 置为 `False`
- 只有当某一天的信号重新回落到 `abs(signal.value) <= threshold`（"无观点"）时，才将 `armed` 重新置为 `True`

这确保了同一次持续信号（如连续多天维持看多）不会被反复开仓；模型必须先"归零"，再重新触发，才会形成第二笔交易。

**开仓与跳过持有期**：

```python
direction = "long" if signal.value > 0 else "short"
entry_idx = all_days.index(d)
exit_idx = entry_idx + holding_days
if exit_idx >= len(all_days):
    break   # 剩余价格数据不足以平仓，终止该股票的扫描
```

开仓后，网格指针直接跳到平仓日（`all_days[exit_idx]`）对应的位置继续扫描，不会在持仓期间的每一天重复评估信号（也就不会产生重叠仓位）。

---

### 3.4 `BacktestEngine._build_trade`

将一次开平仓事件（entry_date、exit_date、direction）转换为成交的 `Trade`，或在价格缺失时返回 `None`。

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

**返回字段**：完整填充的 `Trade` 对象；`reasoning` 与 `metadata` 直接取自触发该笔交易的 `Signal.reasoning` / `Signal.metadata`（例如 PEAD 模型会带上 `eps_surprise`、`source_type`、`report_period`；LLM Agent 会带上 `confidence`、`model`、`cached` 等）。

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

（这套计算逻辑与旧版 `Strategy` 架构完全一致，未随本次重构改变。）

---

## 4. Pydantic 模型详解

### 4.1 `Trade`

一笔已完成交易的完整记录。

| 字段 | 类型 | 说明 |
|------|------|------|
| `ticker` | `str` | 股票代码 |
| `direction` | `str` | `"long"` 或 `"short"` |
| `entry_date` | `str` | 实际入场日期，格式 `YYYY-MM-DD` |
| `exit_date` | `str` | 实际退出日期，格式 `YYYY-MM-DD` |
| `entry_price` | `float` | 入场价（收盘价） |
| `exit_price` | `float` | 退出价（收盘价） |
| `shares` | `float` | 持股数量（`per_trade / entry_price`，精确到 4 位小数） |
| `pnl` | `float` | 美元盈亏（精确到 2 位小数） |
| `return_pct` | `float` | 百分比收益率（精确到 6 位小数，带符号） |
| `holding_days` | `int` | 实际持有交易日数 |
| `reasoning` | `str \| None` | 触发该笔交易的 `Signal.reasoning`（Alpha 模型给出的理由；LLM Agent 场景下是持仓论点） |
| `metadata` | `dict[str, Any]` | 来自 `Signal.metadata`，默认空字典（Alpha 模型自定义上下文） |

---

### 4.2 `PerformanceMetrics`

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

### 4.3 `BacktestResult`

引擎返回的顶层结果对象。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `trades` | `list[Trade]` | `[]` | 所有成交交易，按 `entry_date` 升序排列 |
| `metrics` | `PerformanceMetrics \| None` | `None` | 无任何成交交易时为 `None` |
| `equity_curve` | `list[float]` | `[]` | 权益曲线，长度 = `n_trades + 1`，首元素为初始资金 |

---

## 5. 用 PEAD 模型驱动回测

`PEADModel`（`v2/signals/pead.py`）是引擎最初的参考实现，详细的信号生成逻辑（EPS 惊喜方向判定、45 天过滤、多来源去重）已迁移至 [`v2_signals_system.md`](./v2_signals_system.md#pead-模型-v2signalspeadpy) 文档说明。本节仅说明它如何与回测引擎交互：

- `PEADModel.predict(ticker, date, data_client)` 若在 `signal_window_days`（默认 4 天）内发现符合条件的最新财报事件，返回 `value = ±1.0`；否则返回 `0.0`（无观点）
- 引擎按 `threshold=0.0` 的默认设置，任何非零观点都会触发边沿触发开仓逻辑（见 3.3 节）
- `Signal.metadata` 中的 `eps_surprise`、`source_type`、`report_period` 会原样保存到 `Trade.metadata`，供 CLI 输出彩色展示

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

### 6.6 价格窗口获取（含 padding）

每只股票独立向 `DataClient` 请求一次价格数据（而非逐信号请求），请求窗口：
- 起点：`start_date`
- 终点：`end_date + holding_days * 2 + 10 日`（覆盖回测末尾开仓交易的平仓日）
- 终点不超过今日（防止请求未来日期）

### 6.7 仓位等额美元原则

所有交易使用相同名义美元敞口 `per_trade`，不论票价高低。这使得不同价格区间的个股之间的收益率可直接比较，且不会因仓位大小差异干扰整体绩效统计。

---

## 7. CLI 详解（`__main__.py`）

### 7.1 运行命令

```bash
poetry run python -m v2.backtesting
```

> 另有一个面向演示场景的独立入口 `poetry run python -m v2.demo.backtest`（终端实时仪表盘、25 只股票精选池、可离线重放），详见 [`v2_signals_system.md`](./v2_signals_system.md#demo-展示层-v2demo) 或 `v2/README.md`。两者驱动的都是同一个 `BacktestEngine` + `PEADModel`，区别仅在于展示层与股票池/日期范围的取舍。

### 7.2 默认参数

| 常量 | 值 | 说明 |
|------|-----|------|
| `HOLDING_DAYS` | `5` | PEAD 模型持有交易日数 |
| `CAPITAL` | `100_000.0` | 初始资金 |
| `PER_TRADE` | `10_000.0` | 每笔交易美元敞口 |
| `START_DATE` | `"2024-06-01"` | 回测起始日期 |
| `END_DATE` | `date.today().isoformat()` | 回测截止日期（默认今日） |

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

**阶段一：逐股票回测**

对每只股票独立调用 `engine.run_alpha(model, [ticker], fd, START_DATE, END_DATE, holding_days=HOLDING_DAYS)`（`model = PEADModel()`），终端实时显示进度：

```
  Backtesting PEAD alpha... [42/100] MSFT
```

完成后显示找到的交易总数。

**阶段二：逐笔动态展示**

将所有股票产生的交易按 `entry_date` 排序后，逐笔重放：每加入一笔交易后清屏重绘，展示：
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

### 8.1 基本用法（量化模型）

```python
from v2.data import FDClient
from v2.backtesting import BacktestEngine
from v2.signals import PEADModel

with FDClient() as fd:
    engine = BacktestEngine(capital=100_000, per_trade=10_000)
    result = engine.run_alpha(
        PEADModel(), ["AAPL", "MSFT", "NVDA"], fd,
        "2024-06-01", "2024-12-01", holding_days=5,
    )

print(f"总收益率: {result.metrics.total_return_pct:.2%}")
print(f"夏普比率: {result.metrics.sharpe_ratio:.2f}")
print(f"最大回撤: {result.metrics.max_drawdown_pct:.2%}")
print(f"交易笔数: {result.metrics.n_trades}")
```

### 8.2 使用磁盘缓存加速重跑

```python
from v2.data import CachedDataClient, FDClient
from v2.backtesting import BacktestEngine
from v2.signals import PEADModel

with FDClient() as raw:
    fd = CachedDataClient(raw)   # 首次跑真实请求，重跑走磁盘缓存
    engine = BacktestEngine()
    result = engine.run_alpha(PEADModel(), ["AAPL"], fd, "2024-01-01", "2024-06-01")
```

### 8.3 自定义 Alpha 模型（量化或 LLM 均可）

```python
from v2.signals.base import QuantModel
from v2.models import Signal

class MomentumModel(QuantModel):
    @property
    def name(self) -> str:
        return "momentum_52w"

    def predict(self, ticker, date, data_client) -> Signal:
        # 自定义逻辑：查价格、判断动量、形成观点
        ...
        return Signal(model_name=self.name, ticker=ticker, date=date, value=1.0)
```

自定义模型只需实现 `AlphaModel.predict(ticker, date, data_client) -> Signal`，即可原样传入 `engine.run_alpha()`，无需修改引擎代码。参见 [`v2_signals_system.md`](./v2_signals_system.md) 了解 `AlphaModel` / `QuantModel` / `LLMAgent` 的完整接口说明。

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
| `v2.data.protocol` | `DataClient` | 数据提供者协议（引擎不再直接依赖具体的 `FDClient`） | `engine.py` |
| `v2.data` | `FDClient` | 具体数据源（包级别便捷导入） | `__main__.py` |
| `v2.signals.base` | `AlphaModel` | Alpha 模型抽象接口 | `engine.py` |
| `v2.signals` | `PEADModel` | 具体量化模型 | `__main__.py` |
| `v2.backtesting.models` | `Trade`, `PerformanceMetrics`, `BacktestResult` | 数据模型 | `engine.py` |
| `v2.backtesting.engine` | `BacktestEngine` | 引擎（CLI 中内部重复导入用于运行时指标计算） | `__main__.py` |

### 9.3 标准库

| 模块 | 用途 | 所在文件 |
|------|------|---------|
| `__future__.annotations` | 延迟类型注解（PEP 563） | `engine.py`, `__main__.py` |
| `datetime` | `date`、`datetime`、`timedelta` | `engine.py` |
| `logging` | 日志记录 | `engine.py`, `__main__.py` |
| `os` | `os.system("clear")` 用于终端清屏 | `__main__.py` |
| `sys` | 进度输出（`sys.stdout.write`/`flush`） | `__main__.py` |
| `time` | `time.sleep`（动态展示节奏控制） | `__main__.py` |

### 9.4 运行环境

- **Python 版本**：3.11+（使用 `X | Y` 联合类型语法、`from __future__ import annotations`）
- **包管理器**：Poetry
- **API 密钥**：需要 `FINANCIAL_DATASETS_API_KEY` 环境变量（或 `.env` 文件）以访问完整股票数据；免费可用代码：`AAPL`、`GOOGL`、`MSFT`、`NVDA`、`TSLA`
