# 估值分析师代理 (Valuation Analyst Agent)

源文件: `src/agents/valuation.py`

---

## 1. 模块概述

估值分析师代理实现四种互补的估值方法论，通过加权聚合计算内在价值与市值的偏离程度，生成买入/卖出/持有信号。本模块不直接调用 LLM，完全基于财务数据的数值计算。

**四种估值模型及权重**:

| 模型 | 权重 | 函数 |
|------|------|------|
| 增强型 DCF（多场景） | 0.35 | `calculate_dcf_scenarios` -> `calculate_enhanced_dcf_value` |
| 所有者收益 (Owner Earnings) | 0.35 | `calculate_owner_earnings_value` |
| EV/EBITDA 隐含价值 | 0.20 | `calculate_ev_ebitda_value` |
| 剩余收益模型 (RIM) | 0.10 | `calculate_residual_income_value` |

**重要说明**: 代理实际调用的 DCF 函数是 `calculate_dcf_scenarios`（第 111 行），而**非** `calculate_intrinsic_value`。`calculate_intrinsic_value` 存在于文件中（第 259-280 行）但未被调用，是一个遗留的经典 DCF 实现。

**输入**:

| 字段 | 来源 | 说明 |
|------|------|------|
| `data["tickers"]` | AgentState | 待分析的股票代码列表 |
| `data["end_date"]` | AgentState | 财务数据截止日期 |
| `FINANCIAL_DATASETS_API_KEY` | state / 环境变量 | Financial Datasets API 密钥 |

**数据获取**:

- `get_financial_metrics`: `period="ttm"`, `limit=8`
- `search_line_items`: `period="ttm"`, `limit=8`，请求 **12 个字段**: `free_cash_flow`, `net_income`, `depreciation_and_amortization`, `capital_expenditure`, `working_capital`, `total_debt`, `cash_and_equivalents`, `interest_expense`, `revenue`, `operating_income`, `ebit`, `ebitda`
- `get_market_cap`: 获取当前市值用于信号判定

**输出**:

```python
{
    "messages": [msg],  # 标准单消息返回（非拼接模式）
    "data": data,       # 包含 analyst_signals[agent_id] = valuation_analysis
}
```

每个 ticker 的分析结果结构:
```python
{
    "signal": "bullish" | "bearish" | "neutral",
    "confidence": int,  # 0-100
    "reasoning": {
        "dcf_analysis": {"signal": ..., "details": ...},
        "owner_earnings_analysis": {"signal": ..., "details": ...},
        "ev_ebitda_analysis": {"signal": ..., "details": ...},
        "residual_income_analysis": {"signal": ..., "details": ...},
        "dcf_scenario_analysis": {  # 第 5 个 key
            "bear_case": "$...",
            "base_case": "$...",
            "bull_case": "$...",
            "wacc_used": "...",
            "fcf_periods_analyzed": int
        }
    }
}
```

注意: `show_reasoning` 使用 `.get()` 安全访问模式（第 212 行: `state["metadata"].get("show_reasoning")`），与技术分析师代理的直接索引方式不同。

---

## 2. 核心函数

### 2.1 `valuation_analyst_agent(state, agent_id="valuation_analyst_agent") -> dict`

主入口函数。

**执行流程**:

1. 提取 `end_date`、`tickers`，获取 API 密钥（第 24-27 行）
2. 对每个 ticker:
   - 获取 `financial_metrics`（`limit=8`, `period="ttm"`）（第 34-40 行）
   - **守卫**: `if not financial_metrics: continue`（第 41 行）——无财务指标数据则跳过
   - 获取 `line_items`（12 个字段, `limit=8`, `period="ttm"`）（第 48-68 行）
   - **守卫**: `if len(line_items) < 2: continue`（第 69 行）——需要当期和前期数据，不足 2 期则跳过
   - 计算 working capital 变化（处理 None 值，默认 0）（第 78-81 行）
   - 调用四种估值模型（第 84-134 行）
   - 获取独立市值 `get_market_cap`（第 139 行）
   - **守卫**: `if not market_cap: continue`（第 140-142 行）——无法获取市值则跳过（无法计算 gap）
   - **守卫**: `total_weight == 0` 时跳过（第 152-154 行）——所有估值模型均未能返回正值（即没有任何方法给出有效估值），`total_weight = sum(weight for methods with value > 0) = 0`，此时跳过该 ticker
   - 计算各模型与市值的 gap、加权 gap、信号和置信度（第 144-164 行）
   - 构建 reasoning 字典，包含第 5 个 key `dcf_scenario_analysis`（第 167-201 行）
3. 构建 `HumanMessage` 并返回（第 211-220 行）

---

### 2.2 `calculate_owner_earnings_value(net_income: float | None, depreciation: float | None, capex: float | None, working_capital_change: float | None, growth_rate: float = 0.05, required_return: float = 0.15, margin_of_safety: float = 0.25, num_years: int = 5) -> float`

**巴菲特所有者收益估值法**。

```python
owner_earnings = net_income + depreciation - capex - working_capital_change
```

**包含终值计算**:

```python
# 显式增长期（1 到 num_years 年）
pv = sum(owner_earnings * (1+growth_rate)^yr / (1+required_return)^yr for yr in 1..5)

# 终值: Gordon 增长模型
terminal_growth = min(growth_rate, 0.03)
term_val = (owner_earnings * (1+growth_rate)^num_years * (1+terminal_growth)) / (required_return - terminal_growth)
pv_term = term_val / (1+required_return)^num_years

intrinsic = (pv + pv_term) * (1 - margin_of_safety)
```

**参数默认值**:
- `growth_rate`: 来自 `most_recent_metrics.earnings_growth`，不可用时默认 `0.05`
- `required_return`: `0.15`（15%）
- `margin_of_safety`: `0.25`（25% 安全边际）
- `num_years`: `5` 年显式预测期
- `terminal_growth`: `min(growth_rate, 0.03)`，上限 3%

**前置校验**: 所有四个输入必须为 `int` 或 `float` 类型，`owner_earnings <= 0` 则返回 0。

---

### 2.3 `calculate_intrinsic_value(free_cash_flow, growth_rate=0.05, discount_rate=0.10, terminal_growth_rate=0.02, num_years=5) -> float`

**经典 DCF 模型（未被代理调用）**。

文件中存在但在 `valuation_analyst_agent` 中**未被使用**。代理实际调用的是 `calculate_dcf_scenarios` -> `calculate_enhanced_dcf_value` 链路。此函数可能是早期版本的遗留实现。

---

### 2.4 `calculate_wacc(market_cap, total_debt, cash, interest_coverage, debt_to_equity, beta_proxy=1.0, risk_free_rate=0.045, market_risk_premium=0.06) -> float`

**加权平均资本成本 (WACC)**，使用 CAPM 模型。

**权益成本 (Cost of Equity)**:
```python
cost_of_equity = risk_free_rate + beta_proxy * market_risk_premium
# 默认: 0.045 + 1.0 * 0.06 = 0.105 (10.5%)
```

注意: `beta_proxy` 默认为 1.0，代理调用时未传入实际 beta 值，因此所有股票使用相同的市场 beta。

**债务成本 (Cost of Debt)**:
```python
if interest_coverage > 0:
    cost_of_debt = max(risk_free_rate + 0.01, risk_free_rate + (10 / interest_coverage))
else:
    cost_of_debt = risk_free_rate + 0.05  # 默认利差 5%
```

利息覆盖率越高，债务成本越低。`10 / interest_coverage` 给出利差（非百分比）：覆盖率 10x 时原始利差 = 1.0（100%）；覆盖率 2x 时原始利差 = 5.0（500%）。但输出经 `min(max(wacc, 0.06), 0.20)` 封顶，实际 WACC 不超过 20%，因此高利差被自动压制。

**权重计算**:
```python
net_debt = max((total_debt or 0) - (cash or 0), 0)
total_value = market_cap + net_debt
weight_equity = market_cap / total_value
weight_debt = net_debt / total_value
```

**税盾**: 假设 25% 企业税率:
```python
wacc = weight_equity * cost_of_equity + weight_debt * cost_of_debt * 0.75
```

**输出限制**: `min(max(wacc, 0.06), 0.20)`，下限 6%，上限 20%。

---

### 2.5 `calculate_fcf_volatility(fcf_history) -> float`

**自由现金流波动性**，以变异系数 (CV) 衡量。

- 历史数据少于 3 期: 返回 0.5（中等波动性）
- 正 FCF 少于 2 期: 返回 0.8（高波动性）
- 正常情况: `min(stdev(positive_fcf) / mean(positive_fcf), 1.0)`
- 异常时: 返回 0.5

---

### 2.6 `calculate_enhanced_dcf_value(fcf_history, growth_metrics, wacc, market_cap, revenue_growth=None) -> float`

**增强型多阶段 DCF 模型**。

**三阶段增长**:

| 阶段 | 年份 | 增长率计算 |
|------|------|-----------|
| 高增长期 | 1-3 | `min(revenue_growth or 0.05, 0.25)`；大盘股(>500亿)上限 10% |
| 过渡期 | 4-7 | `(high_growth + 0.03) / 2`，逐年衰减: `transition_growth * (8-year)/4` |
| 终值 | 7年后 | `min(0.03, high_growth * 0.6)` |

**基准 FCF**:
```python
base_fcf = max(fcf_current, fcf_avg_3yr * 0.85)  # 取当前 FCF 与 3 年均值 85% 的较大值
```

**终值计算**:
```python
final_fcf = base_fcf * (1+high_growth)^3 * (1+transition_growth)^4
terminal_value = (final_fcf * (1+terminal_growth)) / (wacc - terminal_growth)
pv_terminal = terminal_value / (1+wacc)^7
```

当 `wacc <= terminal_growth` 时，调整 `terminal_growth = wacc * 0.8` 以避免除零或负值。

**质量调整**:
```python
quality_factor = max(0.7, 1 - fcf_volatility * 0.5)
return (pv + pv_terminal) * quality_factor
```

FCF 波动性越高，估值折扣越大（最多打 7 折）。

---

### 2.7 `calculate_dcf_scenarios(fcf_history, growth_metrics, wacc, market_cap, revenue_growth=None) -> dict`

**多场景 DCF 分析**。代理在第 111 行直接调用此函数。

三个场景的调整参数:

| 场景 | 增长率调整 | WACC 调整 | 终值调整 |
|------|----------|----------|---------|
| Bear（悲观） | x 0.5 | x 1.2 | x 0.8 |
| Base（基准） | x 1.0 | x 1.0 | x 1.0 |
| Bull（乐观） | x 1.5 | x 0.9 | x 1.2 |

注意: `terminal_adj` 参数被传入 `scenarios` 字典但在 `calculate_enhanced_dcf_value` 中**未被使用**。

每个场景调用 `calculate_enhanced_dcf_value`，传入调整后的 `revenue_growth` 和 `wacc`。

**概率加权**:
```python
expected_value = bear * 0.2 + base * 0.6 + bull * 0.2
```

**返回值**:
```python
{
    'scenarios': {'bear': float, 'base': float, 'bull': float},
    'expected_value': float,  # 概率加权期望值
    'range': bull - bear,     # 估值区间宽度
    'upside': bull,           # 乐观情景值
    'downside': bear          # 悲观情景值
}
```

---

### 2.8 `calculate_ev_ebitda_value(financial_metrics) -> float`

**EV/EBITDA 隐含权益价值**。

```python
ebitda_now = enterprise_value / enterprise_value_to_ebitda_ratio  # 反推当前 EBITDA
med_mult = median([m.enterprise_value_to_ebitda_ratio for m in financial_metrics if ...])  # 历史中位数倍数
ev_implied = med_mult * ebitda_now
net_debt = enterprise_value - market_cap
return max(ev_implied - net_debt, 0)
```

使用历史 EV/EBITDA 中位数作为公允倍数，反推隐含权益价值。返回值下限为 0。

---

### 2.9 `calculate_residual_income_value(market_cap, net_income, price_to_book_ratio, book_value_growth=0.03, cost_of_equity=0.10, terminal_growth_rate=0.03, num_years=5) -> float`

**剩余收益模型 (Edwards-Bell-Ohlson)**。

```python
book_val = market_cap / price_to_book_ratio
ri0 = net_income - cost_of_equity * book_val  # 初始剩余收益
```

如果 `ri0 <= 0`（收益不足以覆盖权益成本），返回 0。

```python
# 显式期: ri 按 book_value_growth 增长并折现
pv_ri = sum(ri0 * (1+book_value_growth)^yr / (1+cost_of_equity)^yr for yr in 1..5)

# 终值
term_ri = ri0 * (1+book_value_growth)^(num_years+1) / (cost_of_equity - terminal_growth_rate)
pv_term = term_ri / (1+cost_of_equity)^num_years

intrinsic = (book_val + pv_ri + pv_term) * 0.8  # 20% 安全边际
```

---

## 3. 分析算法

### 3.1 综合信号生成

**Gap 计算**（每种估值方法独立）:
```python
gap = (estimated_value - market_cap) / market_cap  # 正值表示低估
```

**加权 Gap**:
```python
weighted_gap = sum(weight * gap for valid methods) / total_weight
```

仅价值大于 0 的方法参与计算（`total_weight` 为这些方法的权重之和）。

**信号阈值**:
- `weighted_gap > 0.15` -> bullish（低估超过 15%）
- `weighted_gap < -0.15` -> bearish（高估超过 15%）
- 否则 -> neutral

**置信度公式**:
```python
confidence = round(min(abs(weighted_gap) / 0.30 * 100, 100))
```

即: gap 达到 30% 时置信度为 100%，线性缩放，上限 100。

### 3.2 每种方法的独立信号

reasoning 中每种方法也有独立的信号判定（第 186-189 行）:
```python
"bullish" if gap > 0.15 else "bearish" if gap < -0.15 else "neutral"
```

使用与综合信号相同的 15% 阈值。

### 3.3 DCF 场景分析详情

reasoning 中包含第 5 个 key `dcf_scenario_analysis`（第 195-201 行）:
```python
"dcf_scenario_analysis": {
    "bear_case": "$...",      # 悲观情景估值
    "base_case": "$...",      # 基准情景估值
    "bull_case": "$...",      # 乐观情景估值
    "wacc_used": "...",       # 使用的 WACC
    "fcf_periods_analyzed": n # FCF 历史期数
}
```

---

## 4. 信号生成汇总

| 参数 | 值 |
|------|---|
| Bullish 阈值 | weighted_gap > 0.15 |
| Bearish 阈值 | weighted_gap < -0.15 |
| Confidence 公式 | `round(min(abs(weighted_gap)/0.30*100, 100))` |
| DCF 权重 | 0.35 |
| Owner Earnings 权重 | 0.35 |
| EV/EBITDA 权重 | 0.20 |
| Residual Income 权重 | 0.10 |
| DCF Bear 概率 | 0.20 |
| DCF Base 概率 | 0.60 |
| DCF Bull 概率 | 0.20 |
| WACC 下限 | 6% |
| WACC 上限 | 20% |
| Owner Earnings 安全边际 | 25% |
| Residual Income 安全边际 | 20% |

---

## 5. 依赖关系

### 标准库 / 第三方库

| 模块 | 用途 |
|------|------|
| `__future__.annotations` | 启用延迟类型注解求值（Python 3.7+ PEP 563） |
| `json` | 序列化分析结果 |
| `statistics` | `statistics.median` 计算 EV/EBITDA 中位数倍数，`statistics.mean`/`stdev` 计算 FCF 波动性 |
| `langchain_core.messages.HumanMessage` | 构建 LangGraph 消息对象 |

### 项目内部依赖

| 模块 | 导入项 | 用途 |
|------|--------|------|
| `src.graph.state` | `AgentState`, `show_agent_reasoning` | 状态类型定义、推理展示输出 |
| `src.utils.progress` | `progress` | 进度状态更新（用于 UI 前端展示） |
| `src.utils.api_key` | `get_api_key_from_state` | 从 state 中安全提取 API 密钥 |
| `src.tools.api` | `get_financial_metrics`, `get_market_cap`, `search_line_items` | 获取财务指标、市值和明细科目数据 |
