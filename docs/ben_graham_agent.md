# Ben Graham Agent 技术文档

## 1. 模块概述

Ben Graham Agent 是 AI 对冲基金多代理系统中模拟本杰明·格雷厄姆价值投资理念的分析代理。该模块严格遵循格雷厄姆的经典原则：收益稳定性、财务实力、内在价值折价和安全边际。

**源文件**: `src/agents/ben_graham.py`

### 输入格式

`AgentState` 类型字典，包含：
- `data.tickers`: 股票代码列表 (`list[str]`)
- `data.end_date`: 分析截止日期 (`str`)
- `data.analyst_signals`: 分析师信号字典 (`dict`)
- `metadata.show_reasoning`: 是否展示推理过程 (`bool`)

### 输出格式

返回 `dict`，包含 `messages` 和 `data`。每个 ticker 的信号结构：

```python
{
    "signal": "bullish" | "bearish" | "neutral",
    "confidence": 78.5,     # float 类型，范围 0-100
    "reasoning": "格雷厄姆风格分析..."
}
```

### 信号数据模型

```python
class BenGrahamSignal(BaseModel):
    signal: Literal["bullish", "bearish", "neutral"]
    confidence: float   # 注意: float 类型
    reasoning: str
```

**注意**: 与 WarrenBuffettSignal 不同，BenGrahamSignal 的 `confidence` 是 `float` 类型。默认回退值为 `confidence=0.0`。

## 2. 核心函数

### `ben_graham_agent(state: AgentState, agent_id: str = "ben_graham_agent") -> dict`
主入口函数。遍历所有 ticker，依次获取数据、执行三维分析、通过 LLM 生成信号。

### `analyze_earnings_stability(metrics: list, financial_line_items: list) -> dict`
分析收益稳定性。检查多年期 EPS 正值记录和 EPS 增长趋势。最大得分 **4**。

### `analyze_financial_strength(financial_line_items: list) -> dict`
分析财务实力。检查流动比率、债务比率和股息记录。最大得分 5。

### `analyze_valuation_graham(financial_line_items: list, market_cap: float) -> dict`
格雷厄姆估值分析。包含 NCAV 检查、格雷厄姆数字计算、安全边际评估。最大得分 7。

### `generate_graham_output(ticker: str, analysis_data: dict[str, any], state: AgentState, agent_id: str) -> BenGrahamSignal`
调用 LLM 生成格雷厄姆风格的投资决策。注意：`analysis_data` 参数是主函数中累积的多 ticker 分析字典，函数将**整个** `analysis_data` 字典序列化（`json.dumps(analysis_data, indent=2)`）后传入 LLM 提示词，而非仅传入 `analysis_data[ticker]` 对应的单只股票数据。

## 3. 分析算法

### 3.1 收益稳定性评分 (`analyze_earnings_stability`)

需要 `metrics` 和 `financial_line_items` 两个参数。从 `financial_line_items` 中提取 `earnings_per_share` 数据。

**最大得分为 4（3 + 1），而非 5**：

| 检查项 | 条件 | 得分 |
|--------|------|------|
| EPS 全部为正 | `positive_eps_years == total_eps_years` | +3 |
| EPS 80% 以上为正 | `positive_eps_years >= total_eps_years * 0.8` | +2 |
| EPS 增长 | `eps_vals[0] > eps_vals[-1]`（最新 > 最旧） | +1 |

需要至少 2 期 EPS 数据。如果数据不足，返回 `score=0`。

### 3.2 财务实力评分 (`analyze_financial_strength`)

最大得分 5：

| 检查项 | 条件 | 得分 |
|--------|------|------|
| 流动比率 >= 2.0 | `current_assets / current_liabilities >= 2.0` | +2 |
| 流动比率 >= 1.5 | 同上 `>= 1.5` | +1 |
| 债务比率 < 0.5 | `total_liabilities / total_assets < 0.5` | +2 |
| 债务比率 < 0.8 | 同上 `< 0.8` | +1 |
| 股息记录 | 多数年份支付股息（`div_paid_years >= len(div_periods) // 2 + 1`） | +1 |

股息判断：`dividends_and_other_cash_distributions < 0` 表示支付了股息（资金流出）。

### 3.3 格雷厄姆估值评分 (`analyze_valuation_graham`)

最大得分 7（4 + 3 或其他组合）：

**NCAV 检查**：
```python
net_current_asset_value = current_assets - total_liabilities
```

| 条件 | 得分 |
|------|------|
| NCAV > market_cap（经典格雷厄姆深度价值） | +4 |
| NCAV 每股 >= 2/3 每股价格（部分净净值折价） | +2 |

**格雷厄姆数字**：
```python
graham_number = math.sqrt(22.5 * eps * book_value_per_share)
# 仅当 eps > 0 且 book_value_ps > 0 时计算
```

**安全边际（基于格雷厄姆数字）**：
```python
margin_of_safety = (graham_number - current_price) / current_price
```

| 条件 | 得分 |
|------|------|
| 安全边际 > 50% | +3 |
| 安全边际 > 20% | +1 |

### 3.4 总分与信号判定

```python
total_score = earnings_analysis["score"] + strength_analysis["score"] + valuation_analysis["score"]
max_possible_score = 15  # 硬编码值

if total_score >= 0.7 * 15:    # >= 10.5 -> bullish
    signal = "bullish"
elif total_score <= 0.3 * 15:  # <= 4.5 -> bearish
    signal = "bearish"
else:
    signal = "neutral"
```

**注意**: `max_possible_score` 硬编码为 15，对应三个分析各 5 分的设计意图。但实际上 `analyze_earnings_stability` 最大只能得 4 分，`analyze_valuation_graham` 最大可得 7 分（4+3），因此实际可能总分范围为 0-16。硬编码的 15 导致阈值计算存在轻微偏差。

### 3.5 API 调用参数

本模块使用 `period="annual"`：

| 函数 | period | limit |
|------|--------|-------|
| `get_financial_metrics` | `"annual"` | 10 |
| `search_line_items` | `"annual"` | 10 |
| `get_market_cap` | N/A | N/A |

### search_line_items 请求的字段

```python
[
    "earnings_per_share", "revenue", "net_income", "book_value_per_share",
    "total_assets", "total_liabilities", "current_assets", "current_liabilities",
    "dividends_and_other_cash_distributions", "outstanding_shares"
]
```

## 4. 信号生成

LLM 系统提示词包含格雷厄姆的 5 条核心原则：
1. 坚持安全边际，低于内在价值买入
2. 强调财务实力（低杠杆、充足流动资产）
3. 偏好多年稳定收益
4. 考虑股息记录
5. 避免投机或高增长假设

提示词要求详细推理，引用具体数字和格雷厄姆的阈值标准。

**默认回退**: `BenGrahamSignal(signal="neutral", confidence=0.0, reasoning="Error in generating analysis; defaulting to neutral.")`

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
| `pydantic` | `BaseModel` |
| `typing_extensions` | `Literal` |
| `json` | JSON 序列化 |
| `math` | `math.sqrt` 用于格雷厄姆数字计算 |

### 运行环境

- Python 3.11+
- 依赖管理：Poetry
- 需要 `FINANCIAL_DATASETS_API_KEY`（通过 `get_api_key_from_state` 从 state 获取）
- 需要至少一个 LLM 提供商的 API key
