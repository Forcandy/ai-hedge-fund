# 技术分析师代理 (Technical Analyst Agent)

源文件: `src/agents/technicals.py`

---

## 1. 模块概述

技术分析师代理是一个纯量化的、不依赖 LLM 的分析模块。它从历史价格数据中计算多种技术指标，执行五种独立交易策略，最终通过加权集成方式合并为一个综合信号。

**设计特点**:

- 不调用 LLM，完全基于数值计算生成信号
- 在所有分析师代理中**唯一**需要 `start_date` 的代理（其他代理仅使用 `end_date`）
- 返回值采用 `state["messages"] + [message]` 的拼接方式，**将已有消息全部保留后追加新消息**；其他分析师代理返回 `[message]`（由 LangGraph 的 `operator.add` reducer 负责合并）

**输入**:

| 字段 | 来源 | 说明 |
|------|------|------|
| `data["tickers"]` | AgentState | 待分析的股票代码列表 |
| `data["start_date"]` | AgentState | 历史数据起始日期（**必需**，其他分析师不需要此字段） |
| `data["end_date"]` | AgentState | 历史数据截止日期 |
| `FINANCIAL_DATASETS_API_KEY` | state / 环境变量 | Financial Datasets API 密钥 |

**输出**:

```python
{
    "messages": state["messages"] + [message],  # 拼接已有消息 + 新 HumanMessage
    "data": data,  # 包含 analyst_signals[agent_id] = technical_analysis
}
```

每个 ticker 的分析结果结构:
```python
{
    "signal": "bullish" | "bearish" | "neutral",
    "confidence": int,  # 0-100
    "reasoning": {
        "trend_following": {"signal": ..., "confidence": ..., "metrics": ...},
        "mean_reversion": {"signal": ..., "confidence": ..., "metrics": ...},
        "momentum": {"signal": ..., "confidence": ..., "metrics": ...},
        "volatility": {"signal": ..., "confidence": ..., "metrics": ...},
        "statistical_arbitrage": {"signal": ..., "confidence": ..., "metrics": ...},
    }
}
```

**关于返回值的重要说明**: `AgentState.messages` 声明为 `Annotated[Sequence[BaseMessage], operator.add]`，即 LangGraph 会自动将返回的 `messages` 列表与现有列表进行 `+` 拼接。因此本代理返回 `state["messages"] + [message]` 会导致**已有消息被重复**——现有消息被返回后再被 reducer 追加一次。这是一个与其他分析师代理不一致的行为。在系统中，仅有 `risk_manager` 和 `portfolio_manager`（流水线后续阶段节点）使用相同的拼接模式。

---

## 2. 核心函数

### 2.1 `safe_float(value, default=0.0) -> float`

安全地将值转换为浮点数，处理 NaN、None 及类型异常情况。

```python
def safe_float(value, default=0.0):
```

使用 `pd.isna()` 和 `np.isnan()` 双重检查，捕获 `ValueError`、`TypeError`、`OverflowError`。在所有策略函数的 metrics 输出中使用，防止 pandas/numpy 标量导致 JSON 序列化失败。

---

### 2.2 `technical_analyst_agent(state: AgentState, agent_id: str = "technical_analyst_agent") -> dict`

主入口函数。按 ticker 逐一调用五种策略函数，加权合并后写入 `state["data"]["analyst_signals"]`。

**执行流程**:

1. 从 `data` 中提取 `start_date`、`end_date`、`tickers`（第 44-47 行）
2. 通过 `get_api_key_from_state` 获取 API 密钥（第 48 行）
3. 对每个 ticker:
   - 调用 `get_prices(ticker, start_date, end_date, api_key)` 获取历史价格（第 56-61 行）
   - **无数据守卫**：若 `not prices`，打印 "Failed: No price data found" 并 `continue` 跳过该 ticker（不生成任何信号）（第 63-65 行）
   - 调用 `prices_to_df(prices)` 转换为 DataFrame（第 68 行）
   - **注意**：对数据量不足的情况（如价格历史 < 126 bars），各策略函数内部使用 `safe_float()` 处理 NaN 值，不会报错但可能产生 NaN 或默认值驱动的信号
   - 依次计算五种策略信号（第 71-83 行）
   - 使用 `weighted_signal_combination()` 合并（第 95-104 行）
   - 用 `normalize_pandas()` 转换 metrics 后存入分析结果（第 107-137 行）
4. 构建 `HumanMessage`，内容为 JSON 序列化的分析结果（第 141-144 行）
5. 如果 `state["metadata"]["show_reasoning"]` 为真，调用 `show_agent_reasoning`（第 146 行）
   - 注意：此处直接用 `["show_reasoning"]` 索引访问，不同于 valuation agent 使用 `.get()` 方法，当 metadata 中不存在该 key 时会抛出 `KeyError`
6. 将信号写入 `state["data"]["analyst_signals"][agent_id]`（第 150 行）
7. 返回 `{"messages": state["messages"] + [message], "data": data}`（第 154-157 行）

---

### 2.3 `normalize_pandas(obj)`

**JSON 序列化辅助函数**（正确性要求，非内存优化）。递归地将 pandas `Series` 转为 `list`、`DataFrame` 转为 `dict` 记录列表，使输出可被 `json.dumps` 序列化。

```python
def normalize_pandas(obj):
    if isinstance(obj, pd.Series):
        return obj.tolist()
    elif isinstance(obj, pd.DataFrame):
        return obj.to_dict("records")
    elif isinstance(obj, dict):
        return {k: normalize_pandas(v) for k, v in obj.items()}
    elif isinstance(obj, (list, tuple)):
        return [normalize_pandas(item) for item in obj]
    return obj
```

此函数仅在构建 reasoning 字典时对各策略的 `metrics` 字段调用。如果不做转换，`json.dumps(technical_analysis)` 会因 pandas 类型不可序列化而抛出 `TypeError`。

---

### 2.4 策略函数

#### `calculate_trend_signals(prices_df) -> dict`

趋势跟踪策略，使用多时间框架 EMA 和 ADX。

- 计算 EMA-8、EMA-21、EMA-55 三条指数移动平均线
- 计算 14 周期 ADX（趋势强度）
- 信号逻辑:
  - `short_trend = ema_8 > ema_21`（短期趋势），`medium_trend = ema_21 > ema_55`（中期趋势）
  - 两者最新值均为 True -> bullish，均为 False -> bearish，否则 neutral
  - confidence = `adx[-1] / 100.0`（趋势强度归一化到 0-1），neutral 时固定为 0.5
- 输出 metrics: `adx`, `trend_strength`

---

#### `calculate_mean_reversion_signals(prices_df) -> dict`

均值回归策略，使用 Z-score 和布林带。

**关键区别**: Z-score 和布林带使用**不同的窗口期**，它们是完全独立的计算:

- **Z-score**: 50 周期均线 (`ma_50`) 和 50 周期标准差 (`std_50`)

  ```python
  ma_50 = prices_df["close"].rolling(window=50).mean()
  std_50 = prices_df["close"].rolling(window=50).std()
  z_score = (prices_df["close"] - ma_50) / std_50
  ```

- **布林带**: 20 周期窗口（`calculate_bollinger_bands` 默认参数 `window=20`）

  ```python
  bb_upper, bb_lower = calculate_bollinger_bands(prices_df)  # window=20
  ```

- `price_vs_bb`: 当前价格在布林带中的相对位置（0 = 下轨，1 = 上轨）

  ```python
  price_vs_bb = (close[-1] - bb_lower[-1]) / (bb_upper[-1] - bb_lower[-1])
  ```

- 信号逻辑:
  - Z-score < -2 **且** price_vs_bb < 0.2 -> bullish（严重超卖）
  - Z-score > 2 **且** price_vs_bb > 0.8 -> bearish（严重超买）
  - 否则 -> neutral
  - confidence = `min(abs(z_score) / 4, 1.0)`，neutral 时固定 0.5
- 输出 metrics: `z_score`, `price_vs_bb`, `rsi_14`, `rsi_28`

注意: RSI-14 和 RSI-28 被计算并包含在 metrics 中，但**并未参与信号判定逻辑**，仅作为参考指标输出。

---

#### `calculate_momentum_signals(prices_df) -> dict`

多因子动量策略。

- 计算三个时间窗口的价格动量（滚动日收益率求和，近似累计收益率；使用 `pct_change().rolling(N).sum()`，非精确复利累计收益）:
  - `mom_1m`: 21 日（约 1 个月交易日）
  - `mom_3m`: 63 日（约 3 个月）
  - `mom_6m`: 126 日（约 6 个月）
- 计算成交量动量: `volume_momentum = volume / volume.rolling(21).mean()`
- 加权动量得分: `momentum_score = 0.4 * mom_1m + 0.3 * mom_3m + 0.3 * mom_6m`（取最新值）
- 成交量确认: `volume_momentum[-1] > 1.0`（当前成交量高于 21 日均量）
- 信号逻辑:
  - score > 0.05 且成交量确认 -> bullish
  - score < -0.05 且成交量确认 -> bearish
  - 否则 -> neutral（包括动量显著但成交量未确认的情况）
  - confidence = `min(abs(momentum_score) * 5, 1.0)`，neutral 时固定 0.5
- 输出 metrics: `momentum_1m`, `momentum_3m`, `momentum_6m`, `volume_momentum`

---

#### `calculate_volatility_signals(prices_df) -> dict`

波动率交易策略。

- 日收益率: `returns = close.pct_change()`
- 历史波动率: `hist_vol = returns.rolling(21).std() * sqrt(252)`（年化，21 日窗口）
- 波动率长期均值: `vol_ma = hist_vol.rolling(63).mean()`（63 日窗口，约 3 个月）
- 波动率机制: `vol_regime = hist_vol / vol_ma`（当前波动率相对于长期均值的比率）
- 波动率 Z-score: `vol_z_score = (hist_vol - vol_ma) / hist_vol.rolling(63).std()`
- ATR 比率: `atr_ratio = atr / close`（ATR 相对于价格的比例）
- 信号逻辑:
  - vol_regime < 0.8 且 vol_z < -1 -> bullish（低波动率机制，预期波动率扩张有利于做多）
  - vol_regime > 1.2 且 vol_z > 1 -> bearish（高波动率机制，预期波动率收缩或下行风险）
  - 否则 -> neutral
  - confidence = `min(abs(vol_z) / 3, 1.0)`，neutral 时固定 0.5
- 输出 metrics: `historical_volatility`, `volatility_regime`, `volatility_z_score`, `atr_ratio`

---

#### `calculate_stat_arb_signals(prices_df) -> dict`

统计套利信号，基于价格行为的统计特性分析。

- 计算 63 日滚动偏度 (`skew`) 和峰度 (`kurt`)
- 通过 `calculate_hurst_exponent` 计算 Hurst 指数
- 信号逻辑:
  - Hurst < 0.4 且 skew > 1 -> bullish（序列具有均值回复特性 + 正偏，当前可能被低估）
  - Hurst < 0.4 且 skew < -1 -> bearish（序列具有均值回复特性 + 负偏，当前可能被高估）
  - 否则 -> neutral（包括 Hurst >= 0.4 即趋势性或随机游走序列的情况）
  - confidence = `(0.5 - hurst) * 2`（Hurst 越小于 0.5，置信度越高），neutral 时固定 0.5
- 输出 metrics: `hurst_exponent`, `skewness`, `kurtosis`

---

### 2.5 `weighted_signal_combination(signals, weights) -> dict`

加权信号合并函数。

- 信号映射: bullish=1, neutral=0, bearish=-1
- 加权求和公式: `weighted_sum += numeric_signal * weight * confidence`
- 总权重: `total_confidence += weight * confidence`
- 归一化: `final_score = weighted_sum / total_confidence`（当 total_confidence > 0 时）
- 最终信号阈值:
  - final_score > 0.2 -> bullish
  - final_score < -0.2 -> bearish
  - 否则 -> neutral
- 返回 confidence = `abs(final_score)`

**策略权重分配**:

| 策略 | 权重 | 说明 |
|------|------|------|
| 趋势跟踪 (trend) | 0.25 | 与动量并列最高权重 |
| 均值回归 (mean_reversion) | 0.20 | 中等权重 |
| 动量 (momentum) | 0.25 | 与趋势并列最高权重 |
| 波动率 (volatility) | 0.15 | 辅助策略 |
| 统计套利 (stat_arb) | 0.15 | 辅助策略 |

---

## 3. 分析算法——技术指标函数

### 3.1 `calculate_rsi(prices_df, period=14) -> pd.Series`

**相对强弱指数 (RSI)**。

```python
delta = prices_df["close"].diff()
gain = delta.where(delta > 0, 0).fillna(0)
loss = (-delta.where(delta < 0, 0)).fillna(0)
avg_gain = gain.rolling(window=period).mean()
avg_loss = loss.rolling(window=period).mean()
rs = avg_gain / avg_loss
rsi = 100 - (100 / (1 + rs))
```

使用简单移动平均（SMA）而非 Wilder 指数平滑计算 `avg_gain` / `avg_loss`。当 `avg_loss == 0` 时 `rs` 为 `inf`，RSI 计算结果为 100（数学上正确：完全无下跌区间）。

在本模块中被调用两次：`calculate_rsi(prices_df, 14)` 和 `calculate_rsi(prices_df, 28)`，分别用于均值回归策略的 14 日和 28 日 RSI。

---

### 3.2 `calculate_bollinger_bands(prices_df, window=20) -> tuple[pd.Series, pd.Series]`

**布林带**。

```python
sma = prices_df["close"].rolling(window).mean()
std_dev = prices_df["close"].rolling(window).std()
upper_band = sma + (std_dev * 2)
lower_band = sma - (std_dev * 2)
```

标准 2 倍标准差布林带，默认窗口期 20。返回上轨和下轨两个 Series。与 Z-score 的 50 周期窗口**相互独立**，这是有意的设计——Z-score 捕捉较长期的均值偏离，布林带捕捉较短期的价格极端。

---

### 3.3 `calculate_ema(df, window) -> pd.Series`

**指数移动平均线 (EMA)**。

```python
return df["close"].ewm(span=window, adjust=False).mean()
```

使用 pandas 内建 EWM，`adjust=False` 表示使用递归计算方式，衰减因子 `alpha = 2 / (span + 1)`。在趋势策略中被调用三次：EMA-8、EMA-21、EMA-55。

---

### 3.4 `calculate_adx(df, period=14) -> pd.DataFrame`

**平均方向指数 (ADX)**。

计算步骤:
1. **True Range**: `max(high-low, |high-prev_close|, |low-prev_close|)`
2. **方向移动**: `+DM`（上涨幅度大于下跌幅度且为正时取上涨幅度），`-DM`（下跌幅度大于上涨幅度且为正时取下跌幅度）
3. **方向指标**: `+DI = 100 * EWM(+DM, span=period) / EWM(TR, span=period)`，`-DI` 同理
4. **DX**: `100 * |+DI - -DI| / (+DI + -DI)`
5. **ADX**: `EWM(DX, span=period)`

返回包含 `adx`、`+di`、`-di` 三列的 DataFrame。

**已知问题——无零除保护**（第 480 行）:

```python
df["dx"] = 100 * abs(df["+di"] - df["-di"]) / (df["+di"] + df["-di"])
```

当 `+DI == 0` 且 `-DI == 0` 时（即 `+DM` 和 `-DM` 均为 0，发生在价格完全不变的区间），分母为零，将产生 `inf` 或 `NaN`。后续 `EWM` 会传播 `NaN`，但最终在输出层通过 `safe_float()` 被转换为默认值 `0.0`。

**副作用**: 此函数直接修改传入的 DataFrame，添加 `high_low`、`high_close`、`low_close`、`tr`、`up_move`、`down_move`、`plus_dm`、`minus_dm`、`+di`、`-di`、`dx`、`adx` 共 12 个临时列，会污染原始 `prices_df`。这些列在后续其他策略函数执行时仍然存在于 DataFrame 中。

---

### 3.5 `calculate_atr(df, period=14) -> pd.Series`

**真实波动幅度均值 (ATR)**。

```python
high_low = df["high"] - df["low"]
high_close = abs(df["high"] - df["close"].shift())
low_close = abs(df["low"] - df["close"].shift())
ranges = pd.concat([high_low, high_close, low_close], axis=1)
true_range = ranges.max(axis=1)
return true_range.rolling(period).mean()
```

使用 SMA 计算 ATR（标准 Wilder 方法使用 EMA）。使用局部变量，不修改传入的 DataFrame。

---

### 3.6 `calculate_hurst_exponent(price_series, max_lag=20) -> float`

**Hurst 指数**——判断时间序列的长期记忆特性。

**重要: 实际使用方差缩放法 (Variance Scaling)，而非 R/S 分析法**（尽管注释提及 "R/S calculation"）。

```python
lags = range(2, max_lag)  # lags: 2, 3, ..., 19
tau = [max(1e-8, np.sqrt(np.std(np.subtract(price_series[lag:], price_series[:-lag]))))
       for lag in lags]
reg = np.polyfit(np.log(lags), np.log(tau), 1)
return reg[0]  # 斜率即 Hurst 指数
```

算法原理:
1. 对每个滞后期 `lag`（从 2 到 19），计算价格差分序列: `price_series[lag:] - price_series[:-lag]`
2. 计算差分序列的标准差 `np.std(...)`
3. 对标准差取平方根 `np.sqrt(std(...))`——注意这是 `sqrt(std)`，不是直接取 `std`
4. 添加下界 `max(1e-8, ...)` 防止 `log(0)`
5. 在 `log(lags)` vs `log(tau)` 上做最小二乘线性拟合（`np.polyfit`），斜率为 Hurst 指数
6. 计算失败时（`ValueError` 或 `RuntimeWarning`）返回 0.5（随机游走假设）

**解读**:
- H < 0.5: 均值回复序列（统计套利策略关注此区间）
- H = 0.5: 随机游走
- H > 0.5: 趋势序列

---

## 4. 信号生成

### 4.1 各策略信号阈值汇总

| 策略 | Bullish 条件 | Bearish 条件 | Confidence 公式 | Neutral Confidence |
|------|-------------|-------------|----------------|-------------------|
| 趋势跟踪 | EMA8>EMA21 且 EMA21>EMA55 | EMA8<EMA21 且 EMA21<EMA55 | ADX/100 | 0.5 |
| 均值回归 | Z<-2 且 BB位置<0.2 | Z>2 且 BB位置>0.8 | min(\|z\|/4, 1.0) | 0.5 |
| 动量 | score>0.05 且量确认 | score<-0.05 且量确认 | min(\|score\|*5, 1.0) | 0.5 |
| 波动率 | regime<0.8 且 volZ<-1 | regime>1.2 且 volZ>1 | min(\|volZ\|/3, 1.0) | 0.5 |
| 统计套利 | Hurst<0.4 且 skew>1 | Hurst<0.4 且 skew<-1 | (0.5-hurst)*2 | 0.5 |

### 4.2 综合信号合并

- bullish: `final_score > 0.2`
- bearish: `final_score < -0.2`
- neutral: `-0.2 <= final_score <= 0.2`
- 最终 confidence 在主函数中: `round(combined_signal["confidence"] * 100)` 转为整数百分比

---

## 5. 依赖关系

### 标准库 / 第三方库

| 模块 | 用途 |
|------|------|
| `math` | `math.sqrt(252)` 年化波动率系数 |
| `json` | 序列化分析结果为 JSON 字符串 |
| `pandas` | DataFrame 操作、滚动窗口计算、EWM |
| `numpy` | 数组运算、`np.polyfit` 线性拟合、`np.where` 条件选择、`np.subtract` 差分 |
| `langchain_core.messages.HumanMessage` | 构建 LangGraph 消息对象 |

### 项目内部依赖

| 模块 | 导入项 | 用途 |
|------|--------|------|
| `src.graph.state` | `AgentState`, `show_agent_reasoning` | 状态类型定义、推理展示输出 |
| `src.utils.api_key` | `get_api_key_from_state` | 从 state 中安全提取 API 密钥 |
| `src.tools.api` | `get_prices`, `prices_to_df` | 获取历史价格数据并转为 DataFrame（包含 open/high/low/close/volume 列） |
| `src.utils.progress` | `progress` | 进度状态更新（用于 UI 前端展示） |
