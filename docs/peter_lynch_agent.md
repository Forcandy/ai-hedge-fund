# Peter Lynch Agent 技术文档

## 1. 模块概述

Peter Lynch Agent 是 AI 对冲基金多代理系统中模拟彼得·林奇"以合理价格买入成长股" (GARP) 投资理念的分析代理。该模块从增长性、基本面、估值（以 PEG 比率为核心）、新闻情绪和内部人交易五个维度进行加权评分。

**源文件**: `src/agents/peter_lynch.py`

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
    "reasoning": "林奇风格分析..."
}
```

### 信号数据模型

```python
class PeterLynchSignal(BaseModel):
    signal: Literal["bullish", "bearish", "neutral"]
    confidence: float   # float 类型
    reasoning: str
```

## 2. 核心函数

### `peter_lynch_agent(state: AgentState, agent_id: str = "peter_lynch_agent") -> dict`
主入口函数。遍历所有 ticker，获取数据后执行五维分析，加权评分，通过 LLM 生成信号。

### `analyze_lynch_growth(financial_line_items: list) -> dict`
增长分析。评估收入增长和 EPS 增长。需要至少 2 期数据。原始得分最大 6，标准化到 0-10。

### `analyze_lynch_fundamentals(financial_line_items: list) -> dict`
基本面分析。评估负债率、营业利润率、自由现金流。原始得分最大 6，标准化到 0-10。

### `analyze_lynch_valuation(financial_line_items: list, market_cap: float | None) -> dict`
估值分析（GARP 核心）。计算 P/E、EPS 增长率和 PEG 比率。原始得分最大 5，标准化到 0-10。

### `analyze_sentiment(news_items: list) -> dict`
新闻情绪分析。仅检查 `news.title`（不检查 `news.content` 或 `news.sentiment` 属性），基于负面关键词计数。返回固定分值（3/5/6/8）。

### `analyze_insider_activity(insider_trades: list) -> dict`
内部人交易分析。使用 `transaction_shares` 字段判断买卖方向。返回固定分值（4/5/6/8）。

### `generate_lynch_output(ticker: str, analysis_data: dict[str, any], state: AgentState, agent_id: str) -> PeterLynchSignal`
调用 LLM 生成林奇风格的投资决策。

## 3. 分析算法

### 3.1 数据获取

**关键要点**: 本模块**不调用** `get_financial_metrics` 或 `get_prices`。仅使用以下 API：

| 函数 | 参数 |
|------|------|
| `search_line_items` | `period="annual"`, `limit=5` |
| `get_market_cap` | N/A |
| `get_insider_trades` | `end_date`, `limit=50` |
| `get_company_news` | `end_date`, `limit=50` |

`search_line_items` 请求的字段：
```python
[
    "revenue", "earnings_per_share", "net_income", "operating_income",
    "gross_margin", "operating_margin", "free_cash_flow",
    "capital_expenditure", "cash_and_equivalents", "total_debt",
    "shareholders_equity", "outstanding_shares",
]
```

### 3.2 加权评分

```python
total_score = (
    growth_analysis["score"] * 0.30        # 增长 30%
    + valuation_analysis["score"] * 0.25   # 估值 25%
    + fundamentals_analysis["score"] * 0.20 # 基本面 20%
    + sentiment_analysis["score"] * 0.15    # 情绪 15%
    + insider_activity["score"] * 0.10      # 内部人 10%
)

max_possible_score = 10.0

if total_score >= 7.5:
    signal = "bullish"
elif total_score <= 4.5:
    signal = "bearish"
else:
    signal = "neutral"
```

### 3.3 增长分析 (`analyze_lynch_growth`)

原始得分最大 6，标准化公式：`final_score = min(10, (raw_score / 6) * 10)`。

**收入增长**（基于最新期和最早期比较）：

| 条件 | 原始得分 |
|------|---------|
| 收入增长 > 25% | +3 |
| 收入增长 > 10% | +2 |
| 收入增长 > 2% | +1 |

**EPS 增长**（同样基于最新期和最早期比较）：

| 条件 | 原始得分 |
|------|---------|
| EPS 增长 > 25% | +3 |
| EPS 增长 > 10% | +2 |
| EPS 增长 > 2% | +1 |

增长率计算为简单百分比变化：`(latest - older) / abs(older)`。

### 3.4 基本面分析 (`analyze_lynch_fundamentals`)

原始得分最大 6，标准化公式：`final_score = min(10, (raw_score / 6) * 10)`。

| 检查项 | 条件 | 原始得分 |
|--------|------|---------|
| 负债率 | D/E < 0.5 | +2 |
| 负债率 | D/E < 1.0 | +1 |
| 营业利润率 | > 20% | +2 |
| 营业利润率 | > 10% | +1 |
| 自由现金流 | FCF > 0 | +2 |

### 3.5 估值分析 (`analyze_lynch_valuation`)

原始得分最大 5，标准化公式：`final_score = min(10, (raw_score / 5) * 10)`。

**P/E 计算**: `pe_ratio = market_cap / net_incomes[0]`（仅当最新期净利润为正）。

**EPS 增长率使用 CAGR 公式**：
```python
num_years = len(eps_values) - 1
if latest_eps > 0 and older_eps > 0:
    eps_growth_rate = (latest_eps / older_eps) ** (1 / num_years) - 1
```

如果 `latest_eps <= 0`，回退到简单平均增长：
```python
eps_growth_rate = (latest_eps - older_eps) / (older_eps * num_years)
```

**PEG 计算**: `peg_ratio = pe_ratio / (eps_growth_rate * 100)`。注意增长率乘以 100 转换为百分比形式。

**独立 P/E 评分**（与 PEG 评分独立累加）：

| 条件 | 原始得分 |
|------|---------|
| P/E < 15 | +2 |
| P/E < 25 | +1 |

**PEG 评分**：

| 条件 | 原始得分 |
|------|---------|
| PEG < 1 | +3 |
| PEG < 2 | +2 |
| PEG < 3 | +1 |

P/E 和 PEG 评分是**独立评分**，都计入 `raw_score`，最大合计 5 分。

### 3.6 新闻情绪分析 (`analyze_sentiment`)

使用固定负面关键词列表：`["lawsuit", "fraud", "negative", "downturn", "decline", "investigation", "recall"]`。

**评分使用固定值，非 0-10 标准化**：

| 条件 | 分值 |
|------|------|
| 无新闻数据 | 5（默认中性） |
| 负面新闻 > 30% | 3 |
| 有负面新闻但 <= 30% | 6 |
| 无负面新闻 | 8 |

### 3.7 内部人交易分析 (`analyze_insider_activity`)

**关键要点**: 使用 `transaction_shares` 字段判断买卖方向，**非** `transaction_type`：
```python
for trade in insider_trades:
    if trade.transaction_shares is not None:
        if trade.transaction_shares > 0:   # 正值 = 买入
            buys += 1
        elif trade.transaction_shares < 0:  # 负值 = 卖出
            sells += 1
```

**评分使用固定值**：

| 条件 | 分值 |
|------|------|
| 无交易数据 | 5（默认中性） |
| 买入比 > 70% | 8 |
| 买入比 > 40% | 6 |
| 买入比 <= 40%（以卖出为主） | 4 |
| 无有效买卖交易 | 5 |

## 4. 信号生成

LLM 系统提示词包含林奇的 6 条核心原则：
1. 投资你了解的企业
2. 以合理价格买入成长股（GARP），PEG 比率为核心
3. 寻找"十倍股"潜力
4. 偏好稳定增长
5. 避免高负债
6. 管理层故事

提示词要求以林奇的声音表达：引用 PEG 比率、提及"十倍股"潜力、使用实际和通俗的语言。

**默认回退**: `PeterLynchSignal(signal="neutral", confidence=0.0, reasoning="Error in analysis; defaulting to neutral")`

## 5. 依赖关系

### 内部依赖

| 模块 | 导入内容 |
|------|----------|
| `src.graph.state` | `AgentState`, `show_agent_reasoning` |
| `src.tools.api` | `get_market_cap`, `search_line_items`, `get_insider_trades`, `get_company_news` |
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

### 运行环境

- Python 3.11+
- 依赖管理：Poetry
- 需要 `FINANCIAL_DATASETS_API_KEY`（通过 `get_api_key_from_state` 从 state 获取）
- 需要至少一个 LLM 提供商的 API key
