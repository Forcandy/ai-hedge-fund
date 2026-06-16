# Mohnish Pabrai 代理 (Mohnish Pabrai Agent)

源文件: `src/agents/mohnish_pabrai.py`

---

## 1. 模块概述

Mohnish Pabrai 代理模拟"Dhandho 投资者"Mohnish Pabrai 的投资哲学：寻找"低风险高不确定性"机会，注重下行保护、FCF 收益率和资本翻倍潜力。

**输入**:

| 字段 | 来源 | 说明 |
|------|------|------|
| `data["tickers"]` | AgentState | 待分析股票代码列表 |
| `data["end_date"]` | AgentState | 财务数据截止日期 |

**数据获取**:

- `get_financial_metrics`: `period="annual"`, `limit=8`
- `search_line_items`: `period="annual"`, `limit=8`，请求 **15 个字段**: `revenue`, `gross_profit`, `gross_margin`, `operating_income`, `operating_margin`, `net_income`, `free_cash_flow`, `total_debt`, `cash_and_equivalents`, `current_assets`, `current_liabilities`, `shareholders_equity`, `capital_expenditure`, `depreciation_and_amortization`, `outstanding_shares`
- `get_market_cap`: 获取当前市值

**输出**:

```python
{"messages": [HumanMessage], "data": state["data"]}
```

**Pydantic 模型**:

```python
class MohnishPabraiSignal(BaseModel):
    signal: Literal["bullish", "bearish", "neutral"]
    confidence: float   # 0-100
    reasoning: str
```

---

## 2. 函数列表

| 函数 | 签名 | 说明 |
|------|------|------|
| `mohnish_pabrai_agent` | `(state: AgentState, agent_id: str = "mohnish_pabrai_agent") -> dict` | 主入口函数 |
| `analyze_downside_protection` | `(financial_line_items: list) -> dict` | 下行保护（资产负债表强度） |
| `analyze_pabrai_valuation` | `(financial_line_items: list, market_cap: float \| None) -> dict` | FCF 收益率估值 |
| `analyze_double_potential` | `(financial_line_items: list, market_cap: float \| None) -> dict` | 资本翻倍潜力评估 |
| `generate_pabrai_output` | `(ticker: str, analysis_data: dict, state: AgentState, agent_id: str) -> MohnishPabraiSignal` | LLM 信号生成 |

---

## 3. 子分析函数详解

### 3.1 `analyze_downside_protection`

**最大得分**: 10（使用 `min(10, score)` 封顶）

**早期退出守卫**: `if not financial_line_items` → 返回 `score=0, "Insufficient data"`

| 维度 | 条件 | 得分 |
|------|------|------|
| **净现金头寸** | 净现金 > 0（现金 - 债务 > 0） | +3 |
| **流动比率** | >= 2.0 | +2 |
| | >= 1.2 | +1 |
| **负债/权益** | < 0.3 | +2 |
| | < 0.7 | +1 |
| **FCF 稳定性** | 最近 3 年平均 FCF > 0 且 >= 更早期参考值（3 年平均或最早单期） | +2 |
| | 正但存在衰退 | +1 |

---

### 3.2 `analyze_pabrai_valuation`

**最大得分**: 10（使用 `min(10, score)` 封顶）

**早期退出守卫**:
- `if not financial_line_items or market_cap is None or market_cap <= 0` → `score=0`
- `if not fcf_values or len(fcf_values) < 3` → `score=0, "Insufficient FCF history"`
- `if normalized_fcf <= 0` → `score=0, "Non-positive normalized FCF"`

归一化 FCF = 最多 5 年平均值（`sum(fcf_values[:min(5, len)]) / min(5, len)`）。

| 维度 | 条件 | 得分 |
|------|------|------|
| **FCF 收益率** | > 10% | +4 |
| | > 7% | +3 |
| | > 5% | +2 |
| | > 3% | +1 |
| **资本支出强度** | Capex / 收入 < 5%（轻资产） | +2 |
| | Capex / 收入 < 10% | +1 |

**返回**: `{"score": int, "details": str, "fcf_yield": float | None, "normalized_fcf": float | None}`

---

### 3.3 `analyze_double_potential`

**最大得分**: 10（使用 `min(10, score)` 封顶）

**早期退出守卫**: `if not financial_line_items or market_cap is None or market_cap <= 0` → `score=0`

| 维度 | 条件 | 得分 |
|------|------|------|
| **收入增长** | > 15% | +2 |
| | > 5% | +1 |
| **FCF 增长** | > 20% | +3 |
| | > 8% | +2 |
| | > 0% | +1 |
| **FCF 收益率** | > 8% | +3 |
| | > 5% | +1 |

---

## 4. 信号生成

### 4.1 总分计算（加权平均）

```
total_score = downside["score"] * 0.45 + valuation["score"] * 0.35 + double_potential["score"] * 0.20
max_score = 10
```

| 子分析 | 权重 |
|--------|------|
| 下行保护 | 45% |
| FCF 估值 | 35% |
| 翻倍潜力 | 20% |

### 4.2 信号阈值

| 条件 | 信号 |
|------|------|
| total_score >= 7.5 | bullish |
| total_score <= 4.0 | bearish |
| 其他 | neutral |

### 4.3 置信度

由 LLM 生成，类型为 `float`（0-100）。

---

## 5. 分析算法汇总

| 参数 | 值 |
|------|---|
| 子分析数量 | 3（下行保护、FCF 估值、翻倍潜力） |
| 每个子分析满分 | 10（封顶） |
| 加权满分 | 10 |
| Bullish 阈值 | >= 7.5 |
| Bearish 阈值 | <= 4.0 |
| FCF 归一化 | 最多 5 年平均值 |
| 数据周期 | annual，8 年 |

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
