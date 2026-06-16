# Rakesh Jhunjhunwala 代理 (Rakesh Jhunjhunwala Agent)

源文件: `src/agents/rakesh_jhunjhunwala.py`

---

## 1. 模块概述

Rakesh Jhunjhunwala 代理模拟印度"股市大牛"Rakesh Jhunjhunwala 的投资风格：关注高成长性、强盈利能力、健康资产负债表，并通过 DCF 内在价值与安全边际判断信号。

**输入**:

| 字段 | 来源 | 说明 |
|------|------|------|
| `data["tickers"]` | AgentState | 待分析股票代码列表 |
| `data["end_date"]` | AgentState | 财务数据截止日期 |

**数据获取**:

- `get_financial_metrics`: `period="ttm"`, `limit=5`
- `search_line_items`: 无 period/limit 参数（使用默认值），请求 **13 个字段**: `net_income`, `earnings_per_share`, `ebit`, `operating_income`, `revenue`, `operating_margin`, `total_assets`, `total_liabilities`, `current_assets`, `current_liabilities`, `free_cash_flow`, `dividends_and_other_cash_distributions`, `issuance_or_purchase_of_equity_shares`
- `get_market_cap`: 获取当前市值

**输出**:

```python
{"messages": [HumanMessage], "data": state["data"]}
```

**Pydantic 模型**:

```python
class RakeshJhunjhunwalaSignal(BaseModel):
    signal: Literal["bullish", "bearish", "neutral"]
    confidence: float   # 0-100
    reasoning: str
```

---

## 2. 函数列表

| 函数 | 签名 | 说明 |
|------|------|------|
| `rakesh_jhunjhunwala_agent` | `(state: AgentState, agent_id: str = "rakesh_jhunjhunwala_agent") -> dict` | 主入口函数 |
| `analyze_profitability` | `(financial_line_items: list) -> dict` | 盈利能力评分（ROE、运营利润率、EPS CAGR） |
| `analyze_growth` | `(financial_line_items: list) -> dict` | 成长性评分（收入/净利润 CAGR） |
| `analyze_balance_sheet` | `(financial_line_items: list) -> dict` | 资产负债表评分 |
| `analyze_cash_flow` | `(financial_line_items: list) -> dict` | 现金流评分 |
| `analyze_management_actions` | `(financial_line_items: list) -> dict` | 管理层行为评分（回购/增发） |
| `assess_quality_metrics` | `(financial_line_items: list) -> float` | 综合质量评分（0-1.0） |
| `calculate_intrinsic_value` | `(financial_line_items: list, market_cap: float) -> float \| None` | DCF 内在价值计算 |
| `analyze_rakesh_jhunjhunwala_style` | `(financial_line_items: list, owner_earnings: float = None, intrinsic_value: float = None, current_price: float = None) -> dict[str, any]` | 综合分析封装 |
| `generate_jhunjhunwala_output` | `(ticker: str, analysis_data: dict, state: AgentState, agent_id: str) -> RakeshJhunjhunwalaSignal` | LLM 信号生成 |

---

## 3. 子分析函数详解

### 3.1 `analyze_profitability`

**早期退出守卫**: `if not financial_line_items` → `score=0, "No profitability data available"`

| 维度 | 条件 | 得分 |
|------|------|------|
| **ROE** | > 20%（优秀） | +3 |
| | > 15%（良好） | +2 |
| | > 10%（一般） | +1 |
| **运营利润率** | > 20% | +2 |
| | > 15% | +1 |
| **EPS CAGR** | > 20% | +3 |
| | > 15% | +2 |
| | > 10% | +1 |

满分约 8 分（无封顶）。

**注意**：
- ROE 使用推导公式：`shareholders_equity = total_assets - total_liabilities`，无 `shareholders_equity` 字段直接使用。
- EPS CAGR 有次级守卫：过滤出正值 EPS 后，需 `len(positive_eps_values) >= 3` 才进行 CAGR 计算，否则追加 "Insufficient EPS data for growth analysis"。

---

### 3.2 `analyze_growth`

**早期退出守卫**: `if len(financial_line_items) < 3` → `score=0, "Insufficient data for growth analysis"`

| 维度 | 条件 | 得分 |
|------|------|------|
| **收入 CAGR** | > 20% | +3 |
| | > 15% | +2 |
| | > 10% | +1 |
| **净利润 CAGR** | > 25% | +3 |
| | > 20% | +2 |
| | > 15% | +1 |
| **增长一致性** | >= 80% 的年份均实现增长 | +1 |

满分约 7 分（无封顶）。

**注意**：收入和净利润的 CAGR 均有次级守卫——过滤出正值后，需 `len(positive_values) >= 3` 才进行 CAGR 计算，否则追加 "Insufficient ... data" 说明。增长一致性同样基于已过滤的正值收入序列。

---

### 3.3 `analyze_balance_sheet`

**早期退出守卫**: `if not financial_line_items` → `score=0, "No balance sheet data"`

| 维度 | 条件 | 得分 |
|------|------|------|
| **负债/总资产** | < 0.5 | +2 |
| | < 0.7 | +1 |
| **流动比率** | > 2.0 | +2 |
| | > 1.5 | +1 |

满分 4 分。

---

### 3.4 `analyze_cash_flow`

**早期退出守卫**: `if not financial_line_items` → `score=0, "No cash flow data"`

| 维度 | 条件 | 得分 |
|------|------|------|
| **自由现金流** | FCF > 0 | +2 |
| **股息分配** | 有股息支付（负值） | +1 |

满分 3 分。

---

### 3.5 `analyze_management_actions`

**早期退出守卫**: `if not financial_line_items` → `score=0, "No management action data"`

| 维度 | 条件 | 得分 |
|------|------|------|
| **股份变动** | `issuance_or_purchase_of_equity_shares < 0`（回购） | +2 |
| | `issuance_or_purchase_of_equity_shares == 0`（明确无变动） | +1 |
| | `issuance_or_purchase_of_equity_shares > 0`（增发） | 0 |
| | `issuance is None`（无数据） | 0 |

满分 2 分。

---

### 3.6 `assess_quality_metrics`

**早期退出守卫**: `if not financial_line_items` → `return 0.5`（中性分，非 0.0）

返回 0-1.0 的综合质量评分，取 3 个维度的平均值：

| 维度 | 评分规则 |
|------|---------|
| ROE | > 20% → 1.0；> 15% → 0.8；> 10% → 0.6；< 10% → 0.3；负权益 → 0.0 |
| 负债/资产 | < 0.3 → 1.0；< 0.5 → 0.7；< 0.7 → 0.4；>= 0.7 → 0.1 |
| 增长一致性 | 实现增长的年份占比（0-1.0） |

---

### 3.7 `calculate_intrinsic_value`

基于 5 年盈利增长的 DCF 模型。

**早期退出守卫**:
- `if not financial_line_items or not market_cap` → `return None`
- `if not net_income or net_income <= 0` → `return None`
- `if len(net_incomes) < 2` → 使用保守倍数（12×）作为备用

**DCF 参数**（基于质量评分动态调整）:

| 质量等级 | 折现率 | 终值倍数 |
|---------|--------|---------|
| 高质量（>= 0.8） | 12% | 18× |
| 中等（0.6-0.8） | 15% | 15× |
| 较低（< 0.6） | 18% | 12× |

**增长率**: 基于历史 CAGR 动态调整的可持续增长率，4 档规则：
- CAGR > 25%：保守上限 20%
- 15% < CAGR <= 25%：使用历史 CAGR × 80%
- 5% < CAGR <= 15%：使用历史 CAGR × 90%
- CAGR <= 5%：下限 5%（通货膨胀保底）

5 年后采用终值倍数。

---

## 4. 信号生成

### 4.1 总分计算

```
total_score = profitability + growth + balance_sheet + cash_flow + management
max_score = 24  # 8 + 7 + 4 + 3 + 2
```

### 4.2 信号阈值（安全边际优先）

```
if margin_of_safety >= 30%:   signal = "bullish"
elif margin_of_safety <= -30%: signal = "bearish"
else:
  if quality_score >= 0.7 AND total_score >= 60% of max: signal = "bullish"
  elif quality_score <= 0.4 OR total_score <= 30% of max: signal = "bearish"
  else: signal = "neutral"
```

### 4.3 置信度公式

```
if margin_of_safety exists:
  confidence = min(max(abs(margin_of_safety) * 150, 20), 95)  # 20-95% 范围
else:
  confidence = min(max((total_score / max_score) * 100, 10), 80)  # 10-80% 范围
```

---

## 5. 分析算法汇总

| 参数 | 值 |
|------|---|
| 子分析数量 | 5（盈利、成长、资产负债、现金流、管理层）+ 质量 + DCF |
| 总分满分 | 24 |
| Bullish 安全边际阈值 | >= 30% |
| Bearish 安全边际阈值 | <= -30% |
| 高质量阈值 | quality_score >= 0.7 |
| 数据周期 | ttm，5 期 |

---

## 6. 依赖关系

### 项目内部依赖

| 模块 | 导入项 | 用途 |
|------|--------|------|
| `src.graph.state` | `AgentState`, `show_agent_reasoning` | 状态类型定义、推理展示 |
| `src.utils.progress` | `progress` | 进度状态更新 |
| `src.utils.api_key` | `get_api_key_from_state` | 从 state 提取 API 密钥 |
| `src.tools.api` | `get_financial_metrics`, `get_market_cap`, `search_line_items` | 财务数据获取 |
| `src.utils.llm` | `call_llm` | LLM 调用封装 |
