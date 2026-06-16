# Aswath Damodaran 代理 (Aswath Damodaran Agent)

源文件: `src/agents/aswath_damodaran.py`

---

## 1. 模块概述

Aswath Damodaran 代理模拟"估值之王"Damodaran 的投资方法论：基于 CAPM 的 FCFF 折现现金流模型、相对估值和风险分析，通过内在价值与市值的偏离程度（安全边际）生成信号。

**输入**:

| 字段 | 来源 | 说明 |
|------|------|------|
| `data["tickers"]` | AgentState | 待分析股票代码列表 |
| `data["end_date"]` | AgentState | 财务数据截止日期 |
| `data["start_date"]` | AgentState | 起始日期（未使用于此代理） |

**数据获取**:

- `get_financial_metrics`: `period="ttm"`, `limit=5`
- `search_line_items`: 无 period/limit 参数，请求 **8 个字段**: `free_cash_flow`, `ebit`, `interest_expense`, `capital_expenditure`, `depreciation_and_amortization`, `outstanding_shares`, `net_income`, `total_debt`
- `get_market_cap`: 获取当前市值

**输出**:

```python
{"messages": [HumanMessage], "data": state["data"]}
```

每个 ticker 的分析信号写入 `state["data"]["analyst_signals"][agent_id][ticker]`。

**Pydantic 模型**:

```python
class AswathDamodaranSignal(BaseModel):
    signal: Literal["bullish", "bearish", "neutral"]
    confidence: float   # 0-100
    reasoning: str
```

---

## 2. 函数列表

| 函数 | 签名 | 说明 |
|------|------|------|
| `aswath_damodaran_agent` | `(state: AgentState, agent_id: str = "aswath_damodaran_agent") -> dict` | 主入口函数 |
| `analyze_growth_and_reinvestment` | `(metrics: list, line_items: list) -> dict` | 成长性与再投资效率评分 |
| `analyze_risk_profile` | `(metrics: list, line_items: list) -> dict` | 风险画像评分 |
| `analyze_relative_valuation` | `(metrics: list) -> dict` | 相对估值（P/E 历史中位数对比）评分 |
| `calculate_intrinsic_value_dcf` | `(metrics: list, line_items: list, risk_analysis: dict) -> dict` | FCFF DCF 内在价值计算 |
| `estimate_cost_of_equity` | `(beta: float \| None) -> float` | CAPM 权益成本估算 |
| `generate_damodaran_output` | `(ticker: str, analysis_data: dict, state: AgentState, agent_id: str) -> AswathDamodaranSignal` | LLM 信号生成（confidence + reasoning） |

---

## 3. 子分析函数详解

### 3.1 `analyze_growth_and_reinvestment`

**最大得分**: 4

| 条件 | 得分 |
|------|------|
| 收入 CAGR > 8% | +2 |
| 收入 CAGR > 3%（若未得 +2） | +1 |
| 历史 FCFF 末期 > 初期（正增长，使用全部可用期数） | +1 |
| ROIC > 10%（再投资效率） | +1 |

**早期退出守卫**: `if len(metrics) < 2` → 返回 `score=0, "Insufficient history"`

**返回**: `{"score": int, "max_score": 4, "details": str, "metrics": dict}`

---

### 3.2 `analyze_risk_profile`

**最大得分**: 3

| 条件 | 得分 |
|------|------|
| Beta < 1.3 | +1 |
| Debt/Equity < 1 | +1 |
| 利息覆盖率 (EBIT/abs(interest_expense)) > 3× | +1 |

**早期退出守卫**: `if not metrics` → 返回 `score=0, "No metrics"`

内部调用 `estimate_cost_of_equity(beta)` 计算权益成本。

**返回**: `{"score": int, "max_score": 3, "details": str, "beta": float, "cost_of_equity": float}`

---

### 3.3 `analyze_relative_valuation`

**最大得分范围**: -1 ~ +1（可为负）

| 条件 | 得分 |
|------|------|
| TTM P/E < 70% 的 5 年中位数（便宜） | +1 |
| TTM P/E 在 70%~130% 之间（合理） | 0 |
| TTM P/E > 130% 的 5 年中位数（昂贵） | -1 |

**早期退出守卫**:
- `not metrics or len(metrics) < 5` → 返回 `score=0, "Insufficient P/E history"`
- `len(pes) < 5`（过滤 None 后 P/E 数量不足）→ 返回 `score=0, "P/E data sparse"`

**返回**: `{"score": int, "max_score": 1, "details": str}`

---

### 3.4 `calculate_intrinsic_value_dcf`

**DCF 参数**:

| 参数 | 值 | 说明 |
|------|---|------|
| 基准增长率 | 5 年收入 CAGR，上限 12%，默认 4% | 无数据时回退 |
| 终值增长率 | 2.5% | 永续增长率 |
| 预测年数 | 10 年 | 增长率从基准线性衰减至终值 |
| 折现率 | 来自 risk_analysis 的 cost_of_equity，默认 9% | CAPM 权益成本 |

**早期退出守卫**:
- `not metrics or len(metrics) < 2 or not line_items` → `intrinsic_value: None`
- `not fcff0 or not shares` → `intrinsic_value: None, "Missing FCFF or share count"`

> **注意**: `fcff0`（基准 FCF）读取自 `metrics[0].free_cash_flow`（`FinancialMetrics` 对象），而非 `search_line_items` 返回的 line items。但 `shares`（股本数量）读取自 `line_items[0].outstanding_shares`（`search_line_items` 返回值），用于计算 `intrinsic_per_share`。

**返回**: `{"intrinsic_value": float | None, "intrinsic_per_share": float | None, "assumptions": dict, "details": list[str]}`

---

### 3.5 `estimate_cost_of_equity`

CAPM 公式: `r_e = r_f + β × ERP`

| 常量 | 值 |
|------|---|
| 无风险利率 (`r_f`) | 4.0%（10 年美国国债） |
| 股权风险溢价 (`ERP`) | 5.0%（长期美股溢价） |
| 默认 Beta | 1.0（若传入 None） |

默认情况下 cost_of_equity = 0.04 + 1.0 × 0.05 = 9.0%

---

## 4. 信号生成

### 4.1 总分计算

```
total_score = growth_score + risk_score + relative_val_score
```

注意: 三个子分析的分数不做加权平均，而是直接相加传给 LLM。

### 4.2 信号阈值（基于安全边际）

代码根据安全边际预计算信号并作为上下文传入 LLM 提示词，LLM 最终生成包含 signal、confidence 和 reasoning 三个字段的完整输出（`AswathDamodaranSignal`），存储的信号来自 LLM 输出而非预计算值：

| 安全边际 | 信号 |
|---------|------|
| >= 25% | bullish |
| <= -25% | bearish |
| -25% ~ 25% | neutral |

安全边际计算: `margin_of_safety = (intrinsic_value - market_cap) / market_cap`

### 4.3 置信度

由 LLM 生成，类型为 `float`（0-100）。

---

## 5. 分析算法汇总

| 参数 | 值 |
|------|---|
| 子分析数量 | 4（成长性、风险、相对估值、DCF） |
| Bullish 安全边际阈值 | >= 25% |
| Bearish 安全边际阈值 | <= -25% |
| DCF 折现率 | CAPM（约 9%） |
| DCF 增长率上限 | 12% |
| DCF 终值增长率 | 2.5% |
| 相对估值比较基准 | 5 年 P/E 中位数 |

---

## 6. 依赖关系

### 标准库 / 第三方库

| 模块 | 用途 |
|------|------|
| `langchain_core.messages.HumanMessage` | 构建 LangGraph 消息对象 |

### 项目内部依赖

| 模块 | 导入项 | 用途 |
|------|--------|------|
| `src.graph.state` | `AgentState`, `show_agent_reasoning` | 状态类型定义、推理展示 |
| `src.utils.progress` | `progress` | 进度状态更新 |
| `src.utils.api_key` | `get_api_key_from_state` | 从 state 提取 API 密钥 |
| `src.tools.api` | `get_financial_metrics`, `get_market_cap`, `search_line_items` | 财务数据获取 |
| `src.utils.llm` | `call_llm` | LLM 调用封装 |
