# Warren Buffett Agent 技术文档

## 1. 模块概述

Warren Buffett Agent 是 AI 对冲基金多代理系统中模拟沃伦·巴菲特投资理念的分析代理。该模块基于巴菲特的核心投资原则（基本面分析、护城河评估、管理层质量、定价能力、账面价值增长、内在价值计算）对股票进行综合评估，最终通过 LLM 生成投资信号。

**源文件**: `src/agents/warren_buffett.py`

### 输入格式

`AgentState` 类型字典，包含：
- `data.tickers`: 股票代码列表 (`list[str]`)
- `data.end_date`: 分析截止日期 (`str`)
- `data.analyst_signals`: 分析师信号字典 (`dict`)
- `metadata.show_reasoning`: 是否展示推理过程 (`bool`)

### 输出格式

返回 `dict`，包含：
- `messages`: `list[HumanMessage]`，内容为 JSON 序列化的分析结果
- `data`: 更新后的状态数据，含 `analyst_signals["warren_buffett_agent"]`

每个 ticker 的信号结构：
```python
{
    "signal": "bullish" | "bearish" | "neutral",
    "confidence": 75,       # int 类型，范围 0-100
    "reasoning": "短文本说明"
}
```

### 信号数据模型

```python
class WarrenBuffettSignal(BaseModel):
    signal: Literal["bullish", "bearish", "neutral"]
    confidence: int = Field(description="Confidence 0-100")  # 注意: int 类型
    reasoning: str = Field(description="Reasoning for the decision")
```

**关键要点**: `confidence` 字段为 `int` 类型，非 `float`。默认回退值为 `confidence=50`。

## 2. 核心函数

### `warren_buffett_agent(state: AgentState, agent_id: str = "warren_buffett_agent") -> dict`
主入口函数。遍历所有 ticker，依次获取数据、调用各分析子函数、通过 LLM 生成信号。

### `analyze_fundamentals(metrics: list) -> dict[str, any]`
分析基本面指标（ROE、负债率、营业利润率、流动比率）。使用最新一期 metrics 数据。

### `analyze_consistency(financial_line_items: list) -> dict[str, any]`
分析盈利一致性。需要至少 4 期数据。检查 net_income 是否逐期递增。最大得分 3。

### `analyze_moat(metrics: list) -> dict[str, any]`
评估竞争护城河。需要至少 5 期数据，max_score=5。从 ROE 一致性、营业利润率稳定性、资产效率、竞争地位稳定性四个维度评估。

### `analyze_pricing_power(financial_line_items: list, metrics: list) -> dict[str, any]`
分析定价能力。基于毛利率趋势和平均毛利率水平。最大得分 5。

### `analyze_book_value_growth(financial_line_items: list) -> dict[str, any]`
分析每股账面价值增长。需要至少 3 期数据。最大得分 5。

### `_calculate_book_value_cagr(book_values: list) -> tuple[int, str]`
辅助函数，计算账面价值 CAGR 并返回 (评分, 推理文本)。

### `analyze_management_quality(financial_line_items: list) -> dict[str, any]`
分析管理层质量。检查股票回购和分红记录。max_score=2。

### `calculate_owner_earnings(financial_line_items: list) -> dict[str, any]`
计算所有者收益。公式：净利润 + 折旧摊销 - 维护性资本支出 - 营运资本变化。需要至少 2 期数据。

### `estimate_maintenance_capex(financial_line_items: list) -> float`
估算维护性资本支出。三种方法取中位数（需 3 期以上历史数据）或取较大值。

### `calculate_intrinsic_value(financial_line_items: list) -> dict[str, any]`
三阶段 DCF 模型计算内在价值。需要至少 3 期数据。

### `generate_buffett_output(ticker: str, analysis_data: dict[str, any], state: AgentState, agent_id: str = "warren_buffett_agent") -> WarrenBuffettSignal`
调用 LLM 生成最终投资决策。提示词要求推理在 120 字符以内。

## 3. 分析算法

### 3.1 基本面评分 (`analyze_fundamentals`)

基于最新一期 metrics，最大实际得分为 7：

| 指标 | 阈值 | 得分 |
|------|------|------|
| ROE > 15% | `return_on_equity > 0.15` | +2 |
| 负债权益比 < 0.5 | `debt_to_equity < 0.5` | +2 |
| 营业利润率 > 15% | `operating_margin > 0.15` | +2 |
| 流动比率 > 1.5 | `current_ratio > 1.5` | +1 |

### 3.2 一致性评分 (`analyze_consistency`)

需要至少 4 期 `net_income` 数据。

- 逐期递增（`earnings_values[i] > earnings_values[i+1]`，从最新到最旧）：+3
- 最大得分：3

**关键特性**: `consistency_analysis["score"]` 计入 `total_score`，但**不计入** `max_possible_score`。这意味着一致性评分是一个额外加分项。

### 3.3 护城河评分 (`analyze_moat`)

需要至少 5 期 metrics，`max_score = 5`：

| 维度 | 条件 | 得分 |
|------|------|------|
| ROE 一致性 | >= 80% 期间 ROE > 15% | +2 |
| ROE 一致性 | >= 60% 期间 ROE > 15% | +1 |
| 营业利润率稳定性 | 平均 > 20% 且近期 >= 远期 | +1 |
| 资产效率 | 任一期 asset_turnover > 1.0 | +1 |
| 竞争地位稳定性 | 综合稳定性 > 70% | +1 |

最终 `moat_score = min(moat_score, 5)`。

### 3.4 管理层质量评分 (`analyze_management_quality`)

`max_score = 2`：

| 行为 | 条件 | 得分 |
|------|------|------|
| 股票回购 | `issuance_or_purchase_of_equity_shares < 0` | +1 |
| 分红 | `dividends_and_other_cash_distributions < 0` | +1 |

### 3.5 定价能力评分 (`analyze_pricing_power`)

最大得分 5：

- 毛利率趋势部分（最多 +3）：
  - 近期比远期提升 > 2%: +3
  - 近期比远期提升（< 2%）: +2
  - 稳定（差异 < 1%）: +1
- 平均毛利率部分（最多 +2）：
  - 平均毛利率 > 50%: +2
  - 平均毛利率 > 30%: +1

### 3.6 账面价值增长评分 (`analyze_book_value_growth`)

最大得分 5。一致性部分（增长率 = 增长期数 / 总期数 - 1）：
- >= 80%: +3; >= 60%: +2; >= 40%: +1

CAGR 部分（`_calculate_book_value_cagr`）：
- CAGR > 15%: +2; CAGR > 10%: +1; 从负到正: +3

### 3.7 总分与最大可能分数

```python
total_score = (
    fundamental_analysis["score"] +      # 0-7 实际上限
    consistency_analysis["score"] +       # 0-3，计入 total 但不计入 max
    moat_analysis["score"] +              # 0-5
    mgmt_analysis["score"] +              # 0-2
    pricing_power_analysis["score"] +     # 0-5
    book_value_analysis["score"]          # 0-5
)

max_possible_score = (
    10 +                                  # fundamental_analysis 名义上限
    moat_analysis["max_score"] +          # 5
    mgmt_analysis["max_score"] +          # 2
    5 +                                   # pricing_power
    5                                     # book_value_growth
)
# max_possible_score = 27，不含 consistency
```

### 3.8 三阶段 DCF 内在价值计算

**历史增长率**：
- 使用最近 5 期净利润 CAGR：`((latest / oldest) ^ (1/years)) - 1`
- 限制在 `[-0.05, 0.15]`
- 乘以 0.7（30% 保守折扣）得到 `conservative_growth`

**三阶段参数**：

| 阶段 | 增长率 | 年数 |
|------|--------|------|
| 阶段1 (高增长) | `min(conservative_growth, 0.08)` | 5 年 |
| 阶段2 (过渡) | `min(conservative_growth * 0.5, 0.04)` | 5 年 |
| 终端 | 固定 `0.025` | 永续 |

**折现率**: 固定 `0.10`（代码中定义了 `base_discount_rate = 0.09` 但实际未使用，直接赋值 `discount_rate = 0.10`）。

**终端价值**: Gordon 增长模型 `terminal_earnings / (discount_rate - terminal_growth)`。

**最终返回值**: `intrinsic_value` 字段使用 `conservative_intrinsic_value = intrinsic_value * 0.85`（额外 15% 保守折扣）。

### 3.9 安全边际

```python
margin_of_safety = (intrinsic_value - market_cap) / market_cap
```

其中 `intrinsic_value` 为已打折 15% 的保守值。

## 4. 信号生成

最终信号由 LLM 生成。系统提示词规则：

- **Bullish**: 优质企业 AND `margin_of_safety > 0`
- **Bearish**: 差劲企业 OR 明显高估
- **Neutral**: 好企业但 `margin_of_safety <= 0`，或证据混合

**置信度区间（由提示词定义）**：

| 范围 | 含义 |
|------|------|
| 90-100 | 能力圈内的卓越企业，价格有吸引力 |
| 70-89 | 有护城河的好企业，估值合理 |
| 50-69 | 信号混合，需要更多信息 |
| 30-49 | 超出专业范围或基本面堪忧 |
| 10-29 | 差劲企业或严重高估 |

**默认回退**: `WarrenBuffettSignal(signal="neutral", confidence=50, reasoning="Insufficient data")`

提示词要求推理不超过 120 字符，不得捏造数据，仅返回 JSON。

## 5. 依赖关系

### 内部依赖

| 模块 | 导入内容 |
|------|----------|
| `src.graph.state` | `AgentState`, `show_agent_reasoning` |
| `src.tools.api` | `get_financial_metrics`, `get_market_cap`, `search_line_items` |
| `src.utils.llm` | `call_llm` |
| `src.utils.progress` | `progress` |
| `src.utils.api_key` | `get_api_key_from_state` |

### 外部依赖

| 库 | 用途 |
|----|------|
| `langchain_core.prompts` | `ChatPromptTemplate` |
| `langchain_core.messages` | `HumanMessage` |
| `pydantic` | `BaseModel`, `Field` |
| `typing_extensions` | `Literal` |
| `json` | JSON 序列化 |

### API 调用参数

本模块使用 `period="ttm"` (trailing twelve months)，与其他代理使用 `"annual"` 不同：

| 函数 | period | limit |
|------|--------|-------|
| `get_financial_metrics` | `"ttm"` | 10 |
| `search_line_items` | `"ttm"` | 10 |
| `get_market_cap` | N/A | N/A |

### search_line_items 请求的字段

```python
[
    "capital_expenditure",
    "depreciation_and_amortization",
    "net_income",
    "outstanding_shares",
    "total_assets",
    "total_liabilities",
    "shareholders_equity",
    "dividends_and_other_cash_distributions",
    "issuance_or_purchase_of_equity_shares",
    "gross_profit",
    "revenue",
    "free_cash_flow",
]
```

### 运行环境

- Python 3.11+
- 依赖管理：Poetry
- 需要 `FINANCIAL_DATASETS_API_KEY` 环境变量（通过 `get_api_key_from_state` 从 state 获取）
- 需要至少一个 LLM 提供商的 API key
