# Phil Fisher 代理 (Phil Fisher Agent)

源文件: `src/agents/phil_fisher.py`

---

## 1. 模块概述

Phil Fisher 代理模拟"闲聊投资法"创始人 Phil Fisher 的投资风格：注重高质量成长企业、稳定利润率、高效管理层，以及通过内部人交易和市场情绪作为辅助信号。

**输入**:

| 字段 | 来源 | 说明 |
|------|------|------|
| `data["tickers"]` | AgentState | 待分析股票代码列表 |
| `data["end_date"]` | AgentState | 财务数据截止日期 |

**数据获取**:

- `search_line_items`: `period="annual"`, `limit=5`，请求 **13 个字段**: `revenue`, `net_income`, `earnings_per_share`, `free_cash_flow`, `research_and_development`, `operating_income`, `operating_margin`, `gross_margin`, `total_debt`, `shareholders_equity`, `cash_and_equivalents`, `ebit`, `ebitda`
- `get_market_cap`: 获取当前市值
- `get_insider_trades`: `limit=50`
- `get_company_news`: `limit=50`

**注意**: 此代理**不调用** `get_financial_metrics`，所有财务数据来自 `search_line_items`。

**输出**:

```python
{"messages": [HumanMessage], "data": state["data"]}
```

**Pydantic 模型**:

```python
class PhilFisherSignal(BaseModel):
    signal: Literal["bullish", "bearish", "neutral"]
    confidence: float   # 0-100
    reasoning: str
```

---

## 2. 函数列表

| 函数 | 签名 | 说明 |
|------|------|------|
| `phil_fisher_agent` | `(state: AgentState, agent_id: str = "phil_fisher_agent") -> dict` | 主入口函数 |
| `analyze_fisher_growth_quality` | `(financial_line_items: list) -> dict` | 成长质量评分 |
| `analyze_margins_stability` | `(financial_line_items: list) -> dict` | 利润率稳定性评分 |
| `analyze_management_efficiency_leverage` | `(financial_line_items: list) -> dict` | 管理效率与杠杆评分 |
| `analyze_fisher_valuation` | `(financial_line_items: list, market_cap: float \| None) -> dict` | 估值评分 |
| `analyze_insider_activity` | `(insider_trades: list) -> dict` | 内部人交易活动评分 |
| `analyze_sentiment` | `(news_items: list) -> dict` | 新闻情绪评分 |
| `generate_fisher_output` | `(ticker: str, analysis_data: dict[str, any], state: AgentState, agent_id: str) -> PhilFisherSignal` | LLM 信号生成（接收全部 ticker 的 analysis_data 字典） |

---

## 3. 子分析函数详解

所有子分析函数返回归一化至 0-10 的得分（原始分 / 满分 × 10），但内部人交易和情绪分析直接返回固定值（4/5/6/8）。

### 3.1 `analyze_fisher_growth_quality`

**原始满分**: 9；归一化至 0-10

**早期退出守卫**: `if not financial_line_items or len(financial_line_items) < 2` → `score=0`

| 维度 | 条件 | 得分 |
|------|------|------|
| **收入 CAGR** | > 20% | +3 |
| | > 10% | +2 |
| | > 3% | +1 |
| **EPS CAGR** | > 20% | +3 |
| | > 10% | +2 |
| | > 3% | +1 |
| **研发强度** | 3%~15% 收入（最优区间） | +3 |
| | > 15%（过重） | +2 |
| | > 0% | +1 |

---

### 3.2 `analyze_margins_stability`

**原始满分**: 6；归一化至 0-10

**早期退出守卫**: `if not financial_line_items or len(financial_line_items) < 2` → `score=0`

| 维度 | 条件 | 得分 |
|------|------|------|
| **运营利润率趋势** | 稳定或改善（最新 >= 最早） | +2 |
| | 为正但下降 | +1 |
| **毛利率绝对水平** | > 50% | +2 |
| | > 30% | +1 |
| **多年利润率稳定性**（需 >= 3 期数据） | 标准差 < 2% | +2 |
| | 标准差 < 5% | +1 |

---

### 3.3 `analyze_management_efficiency_leverage`

**原始满分**: 6；归一化至 0-10

**早期退出守卫**: `if not financial_line_items` → `score=0`

| 维度 | 条件 | 得分 |
|------|------|------|
| **ROE** | > 20% | +3 |
| | > 10% | +2 |
| | > 0% | +1 |
| **负债/权益** | < 0.3 | +2 |
| | < 1.0 | +1 |
| **FCF 一致性** | > 80% 期间 FCF 为正 | +1 |

---

### 3.4 `analyze_fisher_valuation`

**原始满分**: 4；归一化至 0-10

**早期退出守卫**: `if not financial_line_items or market_cap is None` → `score=0`

| 维度 | 条件 | 得分 |
|------|------|------|
| **市盈率 (P/E)** | < 20 | +2 |
| | < 30 | +1 |
| **市现率 (P/FCF)** | < 20 | +2 |
| | < 30 | +1 |

---

### 3.5 `analyze_insider_activity`

**早期退出守卫**: `if not insider_trades` → 返回 `score=5`（中性默认值）

使用 `transaction_shares` 的符号判断买入/卖出：

| 条件 | 得分 |
|------|------|
| 买入比例 > 70% | 8 |
| 买入比例 > 40% | 6 |
| 买入比例 <= 40% | 4 |
| 无交易数据 | 5（中性） |

---

### 3.6 `analyze_sentiment`

**早期退出守卫**: `if not news_items` → 返回 `score=5`（中性默认值）

通过检查 **新闻标题**（`news.title`）中的负面关键词判断情绪：

**负面关键词**: `["lawsuit", "fraud", "negative", "downturn", "decline", "investigation", "recall"]`

| 条件 | 得分 |
|------|------|
| 负面标题占比 > 30% | 3 |
| 负面标题占比 > 0% | 6 |
| 无负面标题 | 8 |

注意: 此函数仅检查 `news.title` 文本，不使用 `news.sentiment` 属性。

---

## 4. 信号生成

### 4.1 加权总分计算（满分 10.0）

```
total_score = (
    growth_score     * 0.30 +
    margins_score    * 0.25 +
    mgmt_score       * 0.20 +
    valuation_score  * 0.15 +
    insider_score    * 0.05 +
    sentiment_score  * 0.05
)
```

| 子分析 | 权重 |
|--------|------|
| 成长质量 | 30% |
| 利润率稳定性 | 25% |
| 管理效率 | 20% |
| 估值 | 15% |
| 内部人交易 | 5% |
| 情绪 | 5% |

### 4.2 信号阈值

| 条件 | 信号 |
|------|------|
| total_score >= 7.5 | bullish |
| total_score <= 4.5 | bearish |
| 其他 | neutral |

### 4.3 置信度

由 LLM 生成，类型为 `float`（0-100）。

---

## 5. 分析算法汇总

| 参数 | 值 |
|------|---|
| 子分析数量 | 6（成长、利润率、管理、估值、内部人、情绪） |
| 加权满分 | 10.0 |
| Bullish 阈值 | >= 7.5 |
| Bearish 阈值 | <= 4.5 |
| 内部人和情绪默认分 | 5（无数据时中性） |
| 数据周期 | annual，5 年 |
| 不使用 get_financial_metrics | 仅依赖 search_line_items |

---

## 6. 依赖关系

### 项目内部依赖

| 模块 | 导入项 | 用途 |
|------|--------|------|
| `src.graph.state` | `AgentState`, `show_agent_reasoning` | 状态类型定义、推理展示 |
| `src.utils.progress` | `progress` | 进度状态更新 |
| `src.utils.api_key` | `get_api_key_from_state` | 从 state 提取 API 密钥 |
| `src.tools.api` | `get_market_cap`, `search_line_items`, `get_insider_trades`, `get_company_news` | 财务数据获取 |
| `src.utils.llm` | `call_llm` | LLM 调用封装 |
| `statistics` | `pstdev` | 标准库，计算运营利润率的总体标准差（`analyze_margins_stability` 第 311 行） |
