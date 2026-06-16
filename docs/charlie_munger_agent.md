# Charlie Munger Agent 技术文档

## 1. 模块概述

Charlie Munger Agent 是 AI 对冲基金多代理系统中模拟查理·芒格投资哲学的分析代理。该模块基于芒格的多维分析框架，从护城河强度、管理层质量、业务可预测性和估值四个维度对股票进行加权评分，并结合 LLM 生成投资信号。

**源文件**: `src/agents/charlie_munger.py`

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
    "confidence": 75,       # int 类型，范围 0-100
    "reasoning": "短文本说明"
}
```

### 信号数据模型

```python
class CharlieMungerSignal(BaseModel):
    signal: Literal["bullish", "bearish", "neutral"]
    confidence: int   # 注意: int 类型，非 float
    reasoning: str
```

**关键要点**: `confidence` 字段为 `int` 类型。LLM 提示词中硬编码了 `confidence_hint` 值，要求 LLM 不得修改。

## 2. 核心函数

### `charlie_munger_agent(state: AgentState, agent_id: str = "charlie_munger_agent") -> dict`
主入口函数。遍历所有 ticker，获取五类数据（metrics、line_items、market_cap、insider_trades、company_news），执行四维分析，加权评分后通过 LLM 生成信号。

### `analyze_moat_strength(metrics: list, financial_line_items: list) -> dict`
护城河强度分析。评估 ROIC、定价权、资本密集度、无形资产。原始得分标准化到 0-10。

### `analyze_management_quality(financial_line_items: list, insider_trades: list) -> dict`
管理层质量分析。评估资本配置能力、债务管理、现金管理、内部人交易、股本变化。原始得分标准化到 0-10。返回的 dict 包含额外字段：`insider_buy_ratio`、`recent_de_ratio`、`cash_to_revenue`、`share_count_trend`。

### `analyze_predictability(financial_line_items: list) -> dict`
业务可预测性分析。评估收入稳定性、运营利润稳定性、利润率一致性、现金流可靠性。需要至少 5 期数据。原始得分标准化到 0-10。

### `calculate_munger_valuation(financial_line_items: list, market_cap: float) -> dict`
芒格式估值分析。基于归一化 FCF 的简单估值，不使用复杂 DCF。原始得分标准化到 0-10。返回额外字段：`intrinsic_value_range`、`fcf_yield`、`normalized_fcf`、`margin_of_safety_vs_fair_value`。

### `analyze_news_sentiment(news_items: list) -> str`
新闻情绪分析 **stub 函数**。仅返回占位字符串 `"Qualitative review of {N} recent news items would be needed"`，不执行实际 NLP 分析。

### `compute_confidence(analysis: dict, signal: str) -> int`
计算置信度。基于四个维度得分的加权计算，结合安全边际调整，最终按信号类型限制到特定区间。返回 `int` 类型，范围 10-100。

### `make_munger_facts_bundle(analysis: dict[str, any]) -> dict[str, any]`
构建传递给 LLM 的精简事实包。包含分数、关键比率、布尔标志位（moat_strong、predictable、owner_aligned 等）和截断的分析笔记。

### `generate_munger_output(ticker: str, analysis_data: dict[str, any], state: AgentState, agent_id: str, confidence_hint: int) -> CharlieMungerSignal`
调用 LLM 生成芒格风格的投资决策。**注意**参数 `confidence_hint: int`，该值由 `compute_confidence` 预先计算，在提示词中硬编码，要求 LLM 不修改。

### `_r(x, n=3)`
辅助函数，安全的 round 操作，异常时返回 None。

## 3. 分析算法

### 3.1 加权评分与信号阈值

```python
total_score = (
    moat_analysis["score"] * 0.35 +        # 护城河权重 35%
    management_analysis["score"] * 0.25 +    # 管理层权重 25%
    predictability_analysis["score"] * 0.25 + # 可预测性权重 25%
    valuation_analysis["score"] * 0.15       # 估值权重 15%
)

max_possible_score = 10  # 各子分数已标准化到 0-10

if total_score >= 7.5:
    signal = "bullish"
elif total_score <= 5.5:      # 注意: bearish 阈值是 5.5，不是 4.5
    signal = "bearish"
else:
    signal = "neutral"         # 中性区间: 5.5 < score < 7.5
```

**关键要点**: bearish 阈值为 **5.5**，中性区间为 (5.5, 7.5)。

### 3.2 护城河强度评分 (`analyze_moat_strength`)

原始得分最大 9，标准化公式：`final_score = min(10, score * 10 / 9)`。

| 维度 | 条件 | 原始得分 |
|------|------|---------|
| ROIC 一致性 | >= 80% 期间 ROIC > 15% | +3 |
| ROIC 一致性 | >= 50% 期间 | +2 |
| ROIC 一致性 | > 0 期间 | +1 |
| 定价权 | 源码中条件为 `gross_margins[i] >= gross_margins[i-1]`（list 为新到旧排列），实际统计的是毛利率 **下降** 的期间数 >= 70%；注释写"Improving"但逻辑方向相反（已知源码 bug，行为与文档意图相反） | +2 |
| 定价权 | 平均毛利率 > 30% | +1 |
| 资本密集度 | 平均资本支出/收入 < 5% | +2 |
| 资本密集度 | 平均资本支出/收入 < 10% | +1 |
| R&D 投资 | 有 R&D 支出 | +1 |
| 商誉/无形资产 | 有非 None 的商誉/无形资产记录（未验证 > 0，值为 0 的记录也会触发） | +1 |

### 3.3 管理层质量评分 (`analyze_management_quality`)

原始得分最大 12（可能因惩罚项为负），标准化公式：`final_score = max(0, min(10, score * 10 / 12))`。

| 维度 | 条件 | 原始得分 |
|------|------|---------|
| 现金转换 | FCF/NI > 1.1（仅在 `len(fcf_values) == len(net_income_values)` 时计算） | +3 |
| 现金转换 | FCF/NI > 0.9 | +2 |
| 现金转换 | FCF/NI > 0.7 | +1 |
| 债务管理 | D/E < 0.3 | +3 |
| 债务管理 | D/E < 0.7 | +2 |
| 债务管理 | D/E < 1.5 | +1 |
| 现金管理 | 现金/收入 0.1-0.25 | +2 |
| 现金管理 | 现金/收入 0.05-0.1 或 0.25-0.4 | +1 |
| 内部人交易 | 买入比 > 70% | +2 |
| 内部人交易 | 买入比 > 40% | +1 |
| 内部人交易 | 买入比 < 10% 且卖出 > 5 | -1 |
| 股本管理 | 股本减少 > 5% | +2 |
| 股本管理 | 股本稳定（< 5% 变化） | +1 |
| 股本管理 | 股本增加 > 20% | -1 |

内部人交易分析使用 `transaction_type` 字段（匹配 `buy`/`purchase`/`sell`/`sale`）。

### 3.4 业务可预测性评分 (`analyze_predictability`)

需要至少 5 期数据。原始得分最大 10，标准化公式：`final_score = min(10, score * 10 / 10)` 即直接使用原始分。

| 维度 | 条件 | 原始得分 |
|------|------|---------|
| 收入稳定性 | 平均增长 > 5% 且波动 < 10% | +3 |
| 收入稳定性 | 平均增长 > 0% 且波动 < 20% | +2 |
| 收入稳定性 | 平均增长 > 0% | +1 |
| 运营利润 | 100% 期间为正 | +3 |
| 运营利润 | >= 80% 期间为正 | +2 |
| 运营利润 | >= 60% 期间为正 | +1 |
| 利润率一致性 | 波动 < 3% | +2 |
| 利润率一致性 | 波动 < 7% | +1 |
| 现金流可靠性 | 100% 期间 FCF 为正 | +2 |
| 现金流可靠性 | >= 80% 期间 FCF 为正 | +1 |

### 3.5 芒格式估值评分 (`calculate_munger_valuation`)

需要至少 3 期 FCF 数据。原始得分最大 10，标准化同可预测性。

**归一化 FCF**: 最近 3-5 期 FCF 平均值。

| 维度 | 条件 | 原始得分 |
|------|------|---------|
| FCF 收益率 | > 8% (P/FCF < 12.5x) | +4 |
| FCF 收益率 | > 5% (P/FCF < 20x) | +3 |
| FCF 收益率 | > 3% (P/FCF < 33x) | +1 |
| 安全边际 | 与合理价值（15x FCF）相比 > 30% | +3 |
| 安全边际 | > 10% | +2 |
| 安全边际 | > -10% | +1 |
| FCF 增长趋势 | 近期 > 远期 * 1.2 (>20% 增长) | +3 |
| FCF 增长趋势 | 近期 > 远期 | +2 |

内在价值区间：
```python
conservative_value = normalized_fcf * 10   # 10x FCF
reasonable_value = normalized_fcf * 15     # 15x FCF
optimistic_value = normalized_fcf * 20     # 20x FCF
```

### 3.6 置信度计算 (`compute_confidence`)

```python
quality = 0.35 * moat + 0.25 * mgmt + 0.25 * pred  # 范围 0-8.5
quality_pct = 100 * (quality / 8.5)                   # 范围 0-100

# 安全边际调整 (有界 +/-10pp)
val_adj = max(-10, min(10, mos * 100 / 3))

# 基础置信度
base = 0.85 * quality_pct + 0.15 * (val_score * 10) + val_adj

# 按信号类型限制区间
# bullish: 下限 30-50，上限 69-100（mos<=0 时上限为 69）
# bearish: 下限 10-30，上限 49
# neutral: 下限 50，上限 69

conf = int(round(max(lower, min(upper, base))))
return max(10, min(100, conf))
```

### 3.7 事实包构建 (`make_munger_facts_bundle`)

为 LLM 准备精简数据包，包含：
- 各维度分数（四舍五入到 2 位小数）
- 布尔标志：`moat_strong` (>= 7)、`predictable` (>= 7)、`owner_aligned`、`low_leverage`、`sensible_cash`、`mos_positive`、`fcf_yield_ok` (>= 5%)、`share_count_friendly`
- 分析笔记（截断到 120 字符）

### 3.8 新闻情绪分析 (`analyze_news_sentiment`)

**这是一个 stub 函数**，不执行任何实际 NLP 分析：
```python
def analyze_news_sentiment(news_items: list) -> str:
    if not news_items or len(news_items) == 0:
        return "No news data available"
    return f"Qualitative review of {len(news_items)} recent news items would be needed"
```

返回值仅作为 `analysis_data["news_sentiment"]` 的补充信息存储。

## 4. 信号生成

LLM 提示词简短，要求：
- 角色：查理·芒格
- 仅基于提供的事实做决策
- 推理不超过 120 字符
- 使用提供的 confidence 值，不得修改
- 仅返回 JSON

`confidence_hint` 由 `compute_confidence()` 预先计算并在提示词模板中硬编码：
```python
f'  "confidence": {confidence_hint},\n'
```

**默认回退**: `CharlieMungerSignal(signal="neutral", confidence=confidence_hint, reasoning="Insufficient data")`

## 5. 依赖关系

### 内部依赖

| 模块 | 导入内容 |
|------|----------|
| `src.graph.state` | `AgentState`, `show_agent_reasoning` |
| `src.tools.api` | `get_financial_metrics`, `get_market_cap`, `search_line_items`, `get_insider_trades`, `get_company_news` |
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

**注意**: 本模块不依赖 `math` 模块，也**不存在外部 YAML 配置文件**。所有阈值和权重均硬编码在源代码中。

### API 调用参数

| 函数 | period | limit |
|------|--------|-------|
| `get_financial_metrics` | `"annual"` | 10 |
| `search_line_items` | `"annual"` | 10 |
| `get_market_cap` | N/A | N/A |
| `get_insider_trades` | N/A | 100 |
| `get_company_news` | N/A | **10** |

**关键要点**: `get_company_news` 的 limit 是 **10**，不是 100。

### search_line_items 请求的字段

```python
[
    "revenue", "net_income", "operating_income",
    "return_on_invested_capital", "gross_margin", "operating_margin",
    "free_cash_flow", "capital_expenditure",
    "cash_and_equivalents", "total_debt", "shareholders_equity",
    "outstanding_shares", "research_and_development",
    "goodwill_and_intangible_assets",
]
```

### 运行环境

- Python 3.11+
- 依赖管理：Poetry
- 需要 `FINANCIAL_DATASETS_API_KEY`（通过 `get_api_key_from_state` 从 state 获取）
- 需要至少一个 LLM 提供商的 API key
