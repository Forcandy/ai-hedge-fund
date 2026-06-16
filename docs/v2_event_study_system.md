# v2 事件研究系统技术文档 (Event Study System)

源模块: `v2/event_study/`

---

## 1. 模块概述

### 1.1 核心功能

v2 事件研究系统围绕财报事件计算**累积异常收益率（CAR，Cumulative Abnormal Returns）**。系统使用市场模型（OLS 线性回归拟合个股收益与 SPY 基准收益的关系）在估计窗口（`[-250, -11]` 个交易日）内拟合参数，再在事件窗口（`[0, +20]` 个交易日）内计算每日异常收益，最终累加得到三个标准 CAR 窗口的估计值：`[0,+1]`、`[0,+5]`、`[0,+20]`。

### 1.2 设计目标

- 以财报日为事件锚点，评估市场对财报信息的价格反应
- 支持按披露文件类型（8-K、10-Q、10-K、20-F）分组的横截面统计
- 提供 OLS 回归、t 检验、百分位 Bootstrap 置信区间等完整统计推断
- 支持 EPS 惊喜过滤（BEAT / MISS / MEET）
- 提供三种 matplotlib 可视化图表
- 通过 CLI 一键批量运行

### 1.3 系统角色

本模块独立于 LangGraph 多智能体流水线，直接调用 `v2.data.FDClient` 获取价格与财报数据，适合用于离线量化研究与策略验证。

### 1.4 输入/输出

**输入**：
- 股票代码列表（`tickers`）
- `FDClient` 实例（v2 数据层 API 客户端）
- 可选参数：财报条数限制、市场基准代码、Bootstrap 次数、随机种子、EPS 惊喜过滤

**输出**：
- `EventStudyResult`：包含逐条 `EventCAR` 事件记录、按 `source_type` 分组的 `AggregateResult` 统计、以及跳过的 ticker 列表

### 1.5 模块文件清单

| 文件 | 职责 |
|------|------|
| `v2/event_study/engine.py` | `compute_car()` 公共 API + 所有内部处理函数 |
| `v2/event_study/models.py` | 所有 Pydantic 数据模型定义 |
| `v2/event_study/stats.py` | 纯统计函数（OLS、t 检验、Bootstrap） |
| `v2/event_study/plot.py` | matplotlib 可视化函数 |
| `v2/event_study/__main__.py` | CLI 入口（`poetry run python -m v2.event_study`） |
| `v2/event_study/__init__.py` | 公共导出定义 |

---

## 2. 架构设计

### 2.1 处理流水线

```
CLI / 外部调用
    |
compute_car()                    # engine.py 公共 API
    |
    +-- 获取 SPY 收盘价序列（一次性）
    |
    +-- 对每个 ticker:
    |       _compute_ticker_events()
    |           |
    |           +-- 获取财报历史记录
    |           +-- _filter_retrospective()   # 过滤追溯性披露
    |           +-- 获取个股价格
    |           +-- 构建对齐收益序列
    |           +-- 对每条财报记录:
    |                   _process_event()
    |                       |
    |                       +-- _find_event_idx()   # 定位事件日
    |                       +-- fit_market_model()  # OLS 估计窗口
    |                       +-- compute_abnormal_returns()
    |                       +-- sum_car()  x3 窗口
    |                       +-- 返回 EventCAR
    |
    +-- _aggregate()             # 横截面聚合
            |
            +-- ttest_cars()     # t 检验
            +-- bootstrap_ci()   # Bootstrap 置信区间
            +-- 返回 list[AggregateResult]
    |
返回 EventStudyResult
```

### 2.2 关键常量（engine.py）

| 常量 | 值 | 含义 |
|------|----|------|
| `_MARKET_TICKER` | `"SPY"` | 默认市场基准代码 |
| `_ESTIMATION_START` | `-250` | 估计窗口起点（相对事件日的交易日偏移） |
| `_ESTIMATION_END` | `-11` | 估计窗口终点（相对事件日的交易日偏移） |
| `_MIN_ESTIMATION_DAYS` | `200` | 估计窗口最少有效交易日数 |
| `_MAX_EVENT_WINDOW` | `20` | 事件窗口最大长度（交易日） |
| `_RETROSPECTIVE_CUTOFF_DAYS` | `45` | 追溯性披露判定阈值（自然日） |
| `_CAR_WINDOWS` | `[(0,1),(0,5),(0,20)]` | 三个标准 CAR 计算窗口 |

---

## 3. 函数列表

### 3.1 engine.py 函数一览

| 函数 | 可见性 | 返回类型 | 说明 |
|------|--------|---------|------|
| `compute_car(...)` | 公共 | `EventStudyResult` | 主入口，驱动完整 CAR 计算流水线 |
| `_compute_ticker_events(...)` | 内部 | `list[EventCAR]` | 处理单个 ticker 的所有财报事件 |
| `_process_event(...)` | 内部 | `EventCAR \| None` | 处理单条财报记录，返回 EventCAR 或 None |
| `_aggregate(...)` | 内部 | `list[AggregateResult]` | 横截面聚合，按 source_type 分组统计 |
| `_filter_retrospective(...)` | 内部 | `list[EarningsRecord]` | 过滤追溯性披露记录 |
| `_find_event_idx(...)` | 内部 | `int \| None` | 将事件日期对齐到下一个交易日索引 |
| `_parse_date(...)` | 内部 | `date` | 解析日期字符串为 `datetime.date` |

### 3.2 stats.py 函数一览

| 函数 | 返回类型 | 说明 |
|------|---------|------|
| `fit_market_model(stock_returns, market_returns)` | `MarketModelFit` | OLS 拟合市场模型 |
| `compute_abnormal_returns(stock_returns, market_returns, alpha, beta)` | `np.ndarray` | 计算每日异常收益率序列 |
| `sum_car(daily_ar, start, end)` | `float` | 对指定窗口的日异常收益求和 |
| `ttest_cars(cars)` | `tuple[float, float]` | 单样本 t 检验（H0: 均值=0） |
| `bootstrap_ci(cars, n_bootstrap, confidence, rng_seed)` | `BootstrapCI` | 百分位 Bootstrap 置信区间 |

### 3.3 plot.py 函数一览

| 函数 | 返回类型 | 说明 |
|------|---------|------|
| `plot_car_by_source(result)` | `Figure` | 分组条形图：各 source_type × 窗口的平均 CAR |
| `plot_car_distribution(result, window)` | `Figure` | 各 source_type 的 CAR 分布直方图 |
| `plot_cumulative_ar(result, source_type)` | `Figure` | 日均 CAR 路径折线图（含 ±1 SE 带） |

---

## 4. 子函数详解

### 4.1 `compute_car()` — 公共 API

```python
def compute_car(
    tickers: list[str],
    fd_client: FDClient,
    *,
    earnings_limit: int = 12,
    market_ticker: str = "SPY",
    n_bootstrap: int = 10_000,
    rng_seed: int | None = None,
    require_eps_surprise: bool = False,
) -> EventStudyResult
```

**参数说明**：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `tickers` | `list[str]` | 必填 | 待分析的股票代码列表 |
| `fd_client` | `FDClient` | 必填 | v2 数据层 API 客户端实例 |
| `earnings_limit` | `int` | `12` | 每个 ticker 最多处理的财报记录数 |
| `market_ticker` | `str` | `"SPY"` | 市场基准代码，用于获取基准收益序列 |
| `n_bootstrap` | `int` | `10_000` | Bootstrap 重采样次数 |
| `rng_seed` | `int \| None` | `None` | 随机数种子，设置后可复现 Bootstrap 结果 |
| `require_eps_surprise` | `bool` | `False` | 若为 `True`，过滤掉无 EPS 惊喜标签的事件 |

**执行流程**：
1. 调用 `fd_client` 获取 `market_ticker`（默认 SPY）完整收盘价序列
2. 遍历每个 ticker，调用 `_compute_ticker_events()`；若异常则记录到 `skipped_tickers`
3. 若设置 `require_eps_surprise=True`，过滤 `eps_surprise` 为 `None` 的事件
4. 调用 `_aggregate()` 按 `source_type` 横截面聚合
5. 返回 `EventStudyResult`

---

### 4.2 `_compute_ticker_events()` — 单 Ticker 处理

```python
def _compute_ticker_events(
    ticker: str,
    fd_client: FDClient,
    spy_closes: dict[str, float],  # 日期字符串 → 收盘价映射（YYYY-MM-DD）
    *,
    earnings_limit: int = 12,
) -> list[EventCAR]
```

**执行流程**：
1. 通过 `fd_client` 获取该 ticker 的财报历史记录（最多 `earnings_limit` 条）
2. 调用 `_filter_retrospective()` 过滤追溯性披露（`filing_date - report_period >= 45` 自然日）
3. 通过 `fd_client` 获取该 ticker 的历史收盘价
4. 计算个股日收益率序列与对应的 SPY 日收益率序列（时间对齐）
5. 构建 `return_days`（有序交易日列表）和 `day_to_idx`（日期→索引映射）
6. 对每条过滤后的财报记录调用 `_process_event()`，收集非 `None` 结果
7. 返回该 ticker 的 `list[EventCAR]`

---

### 4.3 `_process_event()` — 单事件处理

```python
def _process_event(
    record: EarningsRecord,
    stock_returns: np.ndarray,
    spy_returns: np.ndarray,
    return_days: list[str],
    day_to_idx: dict[str, int],
) -> EventCAR | None
```

**执行流程**：
1. 调用 `_find_event_idx()` 将事件日（`filing_date`）对齐到最近的交易日索引
2. 确定估计窗口索引范围：`[event_idx + _ESTIMATION_START, event_idx + _ESTIMATION_END]`
3. 验证估计窗口的有效交易日数 ≥ `_MIN_ESTIMATION_DAYS`（200）；不足则返回 `None`
4. 调用 `fit_market_model()` 对估计窗口内的个股与 SPY 收益做 OLS 拟合，得到 `alpha`、`beta`、`r_squared`
5. 确定事件窗口范围：`[event_idx, event_idx + _MAX_EVENT_WINDOW]`
6. 调用 `compute_abnormal_returns()` 计算事件窗口内每日 AR
7. 调用 `sum_car()` 三次，分别计算 `car_0_1`、`car_0_5`、`car_0_20`
8. 若事件窗口长度不足，对应 CAR 字段设为 `None`
9. 返回 `EventCAR` 数据对象

**返回 None 的条件**：
- 事件日找不到对应交易日（且未来 4 日内也无交易日）
- 估计窗口有效数据点 < 200
- 收益序列中数据缺失导致无法拟合

---

### 4.4 `_aggregate()` — 横截面聚合

```python
def _aggregate(
    events: list[EventCAR],
    n_bootstrap: int,
    rng_seed: int | None,
) -> list[AggregateResult]
```

**执行流程**：
1. 按 `source_type`（8-K、10-Q、10-K、20-F）对 `events` 分组
2. 对每个分组、每个 CAR 窗口（`[0,+1]`、`[0,+5]`、`[0,+20]`）：
   - 收集该窗口下所有非 `None` 的 CAR 值
   - 计算均值、标准差
   - 调用 `ttest_cars()` 进行 t 检验
   - 调用 `bootstrap_ci()` 计算 95% 置信区间
   - 构建 `WindowStats` 对象
3. 构建 `AggregateResult` 对象，包含该 `source_type` 的所有窗口统计
4. 返回 `list[AggregateResult]`

---

### 4.5 `_filter_retrospective()` — 追溯性披露过滤

```python
def _filter_retrospective(
    records: list[EarningsRecord],
) -> list[EarningsRecord]
```

**过滤规则**：

丢弃满足以下条件的财报记录：

```
filing_date - report_period >= 45 自然日
```

**设计逻辑**：财报的披露日期（`filing_date`）相比报告期末（`report_period`）滞后超过 45 天，通常意味着该财报是追溯性补报而非常规定期报告，其市场信息含量已被大幅提前消化，不适合作为标准事件研究的触发点。

---

### 4.6 `_find_event_idx()` — 事件日交易日对齐

```python
def _find_event_idx(
    event_date: str,
    return_days: list[str],
    day_to_idx: dict[str, int],
) -> int | None
```

**对齐规则**：
- 若 `event_date` 本身在 `return_days` 中，直接返回对应索引
- 否则向后最多尝试 **4 个自然日**，找到第一个存在于 `return_days` 中的日期并返回其索引
- 若 4 日内均无交易日，返回 `None`（该事件将被跳过）

**设计意图**：财报通常在盘后或周末披露，下一个交易日才会在价格中体现；4 日的宽松容差可以覆盖长假期等特殊情况。

---

### 4.7 `_parse_date()` — 日期解析

```python
def _parse_date(s: str) -> date
```

将字符串解析为 `datetime.date` 对象，供内部统一使用。

---

## 5. Pydantic 模型（models.py）

### 5.1 模型一览

| 模型 | 说明 |
|------|------|
| `MarketModelFit` | 市场模型 OLS 拟合结果 |
| `EventCAR` | 单条财报事件的 CAR 计算结果 |
| `BootstrapCI` | Bootstrap 置信区间 |
| `WindowStats` | 单个 CAR 窗口的横截面统计 |
| `AggregateResult` | 单个 source_type 的全部窗口聚合结果 |
| `EventStudyResult` | 整个事件研究的最终输出 |

---

### 5.2 `MarketModelFit`

```python
class MarketModelFit(BaseModel):
    alpha: float            # OLS 截距项（个股超额收益常数项）
    beta: float             # 市场因子载荷（个股对市场收益的敏感度）
    r_squared: float        # 拟合优度 R²
    n_obs: int              # 参与拟合的有效观测数
```

---

### 5.3 `EventCAR`

```python
class EventCAR(BaseModel):
    ticker: str                         # 股票代码
    event_date: str                     # 事件日（对齐后的交易日，YYYY-MM-DD 字符串）
    source_type: str                    # 文件类型：8-K / 10-Q / 10-K / 20-F
    report_period: str                  # 财报报告期末日期（YYYY-MM-DD 字符串）
    eps_surprise: str | None = None     # EPS 惊喜标签：BEAT / MISS / MEET / None
    market_model: MarketModelFit        # 市场模型拟合结果（含 alpha、beta、R²）
    daily_ar: list[float]               # 事件窗口 [0,+20] 内的逐日异常收益率
    car_0_1: float | None = None        # CAR[0,+1]，数据不足时为 None
    car_0_5: float | None = None        # CAR[0,+5]，数据不足时为 None
    car_0_20: float | None = None       # CAR[0,+20]，数据不足时为 None
```

---

### 5.4 `BootstrapCI`

```python
class BootstrapCI(BaseModel):
    lower: float                # 置信区间下界
    upper: float                # 置信区间上界
    confidence: float = 0.95   # 置信水平（默认 95%）
    n_bootstrap: int = 10_000  # Bootstrap 重采样次数
```

---

### 5.5 `WindowStats`

```python
class WindowStats(BaseModel):
    window: str             # 窗口描述，如 "[0,+1]"、"[0,+5]"、"[0,+20]"
    n_events: int           # 该窗口下有效事件数（CAR 非 None 的记录数）
    mean_car: float         # 平均 CAR
    std_car: float          # CAR 标准差
    t_stat: float           # t 统计量（H0: 均值=0）
    p_value: float          # 双尾 p 值
    ci: BootstrapCI         # Bootstrap 95% 置信区间
```

---

### 5.6 `AggregateResult`

```python
class AggregateResult(BaseModel):
    source_type: str                    # 文件类型：8-K / 10-Q / 10-K / 20-F
    n_events: int                       # 该 source_type 下的总事件数
    windows: list[WindowStats]          # 三个 CAR 窗口的统计结果列表
```

---

### 5.7 `EventStudyResult`

```python
class EventStudyResult(BaseModel):
    events: list[EventCAR]              # 所有成功处理的事件记录
    aggregates: list[AggregateResult]   # 按 source_type 分组的聚合统计
    skipped_tickers: list[str]          # 处理失败（异常）被跳过的 ticker 列表
```

---

## 6. 统计方法（stats.py）

### 6.1 `fit_market_model()` — OLS 市场模型拟合

```python
def fit_market_model(
    stock_returns: np.ndarray,
    market_returns: np.ndarray,
) -> MarketModelFit
```

**方法**：OLS（普通最小二乘法），使用 `np.linalg.lstsq`。

**设计矩阵**：

```
X = [1, R_mkt]    # 第一列为常数项 1，第二列为市场日收益率
y = R_stock       # 个股日收益率
```

**求解**：

```
[alpha, beta] = lstsq(X, y)
R² = 1 - SS_res / SS_tot
```

**参数说明**：
- `alpha`：截距项，表示个股在市场收益为零时的预期收益（Jensen's alpha 近似）
- `beta`：斜率，表示个股收益对市场收益变动的敏感度
- `r_squared`：模型对个股收益变动的解释比例

---

### 6.2 `compute_abnormal_returns()` — 异常收益率计算

```python
def compute_abnormal_returns(
    stock_returns: np.ndarray,
    market_returns: np.ndarray,
    alpha: float,
    beta: float,
) -> np.ndarray
```

**公式**：

```
AR_t = R_stock_t - (alpha + beta * R_mkt_t)
```

**含义**：个股实际收益率与市场模型预期收益率之差，剔除了市场系统性波动的影响。

---

### 6.3 `sum_car()` — CAR 累加

```python
def sum_car(daily_ar: np.ndarray, start: int, end: int) -> float
```

**公式**：

```
CAR[start, end] = sum(AR_t)  for t in [start, end]  （含端点）
```

**实现**：`daily_ar[start:end+1].sum()`，对事件窗口内各日异常收益率进行简单累加。

---

### 6.4 `ttest_cars()` — 单样本 t 检验

```python
def ttest_cars(cars: np.ndarray) -> tuple[float, float]
```

**原假设**：`H0: E[CAR] = 0`（即财报事件不产生异常收益）

**返回值**：`(t_stat, p_value)`，双尾检验。

**公式**：

```
t = mean(CAR) / (std(CAR) / sqrt(n))
```

使用 `scipy.stats.ttest_1samp` 实现，自由度为 `n - 1`。

---

### 6.5 `bootstrap_ci()` — 百分位 Bootstrap 置信区间

```python
def bootstrap_ci(
    cars: np.ndarray,
    n_bootstrap: int = 10_000,
    confidence: float = 0.95,
    rng_seed: int | None = None,
) -> BootstrapCI
```

**方法**：百分位 Bootstrap（Percentile Bootstrap）。

**步骤**：
1. 设置随机种子（若提供）
2. 有放回地随机重采样 `n_bootstrap` 次，每次抽取与原样本等量的数据
3. 计算每次重采样的均值，得到 Bootstrap 均值分布
4. 取分布的 `(1-confidence)/2` 和 `1-(1-confidence)/2` 分位数作为置信区间上下界

**优点**：不依赖正态分布假设，对 CAR 分布偏斜或厚尾情况更稳健。

---

## 7. 可视化（plot.py）

### 7.1 `plot_car_by_source()` — 分组条形图

```python
def plot_car_by_source(result: EventStudyResult) -> Figure
```

**图表类型**：分组条形图（Grouped Bar Chart）

**横轴**：CAR 窗口（`[0,+1]`、`[0,+5]`、`[0,+20]`）

**纵轴**：平均 CAR（百分比）

**分组**：按 `source_type`（8-K、10-Q、10-K、20-F）着色

**误差线**：Bootstrap 95% 置信区间（`BootstrapCI.lower`、`BootstrapCI.upper`）

**用途**：直观比较不同文件类型的市场反应强度与方向

---

### 7.2 `plot_car_distribution()` — CAR 分布直方图

```python
def plot_car_distribution(
    result: EventStudyResult,
    window: str = "[0,+1]",
) -> Figure
```

**图表类型**：分面直方图（按 `source_type` 分面）

**参数**：
- `window`：要展示分布的 CAR 窗口，默认 `"[0,+1]"`

**用途**：展示各 `source_type` 下指定窗口 CAR 的经验分布形态，辅助判断正态性与异常值

---

### 7.3 `plot_cumulative_ar()` — 日均 CAR 路径图

```python
def plot_cumulative_ar(
    result: EventStudyResult,
    source_type: str | None = None,
) -> Figure
```

**图表类型**：折线图 + 阴影带

**横轴**：事件窗口相对日（0 到 +20）

**纵轴**：日均累积异常收益率（跨所有事件的截面均值）

**阴影带**：±1 标准误差带（`mean ± SE`，其中 `SE = std / sqrt(n)`）

**参数**：
- `source_type`：若指定则只展示该文件类型的事件；若为 `None` 则展示所有事件

**用途**：观察市场对财报事件的动态响应模式——是否存在提前反应、当日跳升或事后漂移

---

## 8. 算法汇总

### 8.1 完整处理管道逐步说明

| 步骤 | 操作 | 关键参数/判断 |
|------|------|--------------|
| 1 | 获取 SPY 全区间收盘价 | 一次性获取，供所有 ticker 共用 |
| 2 | 获取 ticker 财报历史 | `earnings_limit`（默认 12 条） |
| 3 | 追溯性过滤 | `filing_date - report_period >= 45` 天则丢弃 |
| 4 | 获取个股收盘价，计算日收益率 | 与 SPY 对齐同一时间轴 |
| 5 | 定位事件日交易日索引 | 向后最多尝试 4 自然日 |
| 6 | 截取估计窗口数据 | 相对事件日 `[-250, -11]` 个交易日 |
| 7 | 验证有效数据点 ≥ 200 | 不足则跳过该事件 |
| 8 | OLS 拟合市场模型 | `np.linalg.lstsq`，设计矩阵 `[1, R_mkt]` |
| 9 | 计算事件窗口异常收益 | `AR_t = R_stock_t - (alpha + beta * R_mkt_t)` |
| 10 | 累加 CAR | `CAR[0,1]`、`CAR[0,5]`、`CAR[0,20]` |
| 11 | 按 source_type 分组横截面聚合 | 计算均值、标准差、t 检验、Bootstrap CI |
| 12 | 构建 EventStudyResult 返回 | 含事件列表、聚合结果、跳过 ticker |

### 8.2 估计窗口与事件窗口示意

```
相对事件日（交易日偏移）:

...-250          -11  -10    -1   0   +1   +5         +20
  |________________|       |    |___|____|            |
  估计窗口（OLS拟合）        缓冲  事件日  CAR[0,1]  CAR[0,20]
  [-250, -11]              [-10,-1]
  （至少 200 个交易日）
```

缓冲带 `[-10, -1]` 不参与估计窗口，也不参与事件窗口，用于避免事件提前泄露污染模型参数。

### 8.3 追溯性过滤规则说明

```
filing_date - report_period:
    < 45 天  →  保留（正常定期财报）
    ≥ 45 天  →  丢弃（追溯性补报，信息可能已提前消化）
```

### 8.4 EPS 惊喜标签含义

| 标签 | 含义 |
|------|------|
| `BEAT` | 实际 EPS 高于分析师预期 |
| `MISS` | 实际 EPS 低于分析师预期 |
| `MEET` | 实际 EPS 等于分析师预期 |
| `None` | 无 EPS 惊喜数据（设置 `require_eps_surprise=True` 时过滤） |

---

## 9. CLI 使用（`__main__.py`）

### 9.1 启动命令

```bash
poetry run python -m v2.event_study
```

### 9.2 CLI 默认配置

| 配置项 | 值 |
|--------|-----|
| 股票池 | 100 个 ticker（与回测系统相同的标准股票池） |
| `EARNINGS_LIMIT` | `8` |
| `n_bootstrap` | `10_000` |
| `rng_seed` | `42` |
| EPS 惊喜过滤 | 启用（`require_eps_surprise=True`） |

### 9.3 CLI 输出内容

逐事件表格，每行包含：

| 列名 | 说明 |
|------|------|
| Ticker | 股票代码 |
| Event Date | 对齐后的交易日事件日期 |
| Source Type | 文件类型（8-K / 10-Q / 10-K / 20-F） |
| EPS Surprise | BEAT / MISS / MEET |
| CAR[0,1] | 两日累积异常收益率 |
| CAR[0,5] | 五日累积异常收益率 |
| CAR[0,20] | 二十日累积异常收益率 |
| Beta | 市场模型 beta 系数 |
| R² | 市场模型拟合优度 |

---

## 10. 依赖关系

### 10.1 内部依赖

| 模块 | 用途 |
|------|------|
| `v2.data.FDClient` | Financial Datasets API 客户端，获取价格与财报历史数据 |

### 10.2 外部库依赖

| 库 | 用途 |
|----|------|
| `numpy` | OLS 求解（`np.linalg.lstsq`）、数组运算、收益率计算 |
| `scipy.stats` | 单样本 t 检验（`ttest_1samp`） |
| `matplotlib` | 三种可视化图表生成 |
| `pydantic` | 数据模型定义与验证（`BaseModel`） |

### 10.3 标准库依赖

| 库 | 用途 |
|----|------|
| `datetime` | 日期类型（`date`）处理 |
| `typing` | 类型注解（`list`、`dict`、`tuple` 等） |
