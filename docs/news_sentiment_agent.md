# 新闻情绪分析师代理 (News Sentiment Agent)

源文件: `src/agents/news_sentiment.py`

---

## 1. 模块概述

新闻情绪分析师代理通过分析公司相关新闻的情绪（正面/负面/中性），生成基于新闻舆论的买入/卖出/持有信号。对无预置情绪标签的文章，代理使用 LLM 进行分类。

**与 `sentiment_analyst_agent` 的区别**:
- `news_sentiment_agent`（本模块）：纯新闻文本驱动，调用 LLM 对无标签文章分类
- `sentiment_analyst_agent`（`src/agents/sentiment.py`）：融合内部人交易（30%）+ 新闻情绪（70%），不调用 LLM

**输入**:

| 字段 | 来源 | 说明 |
|------|------|------|
| `data["tickers"]` | AgentState | 待分析股票代码列表 |
| `data["end_date"]` | AgentState | 财务数据截止日期 |

**数据获取**:

- `get_company_news`: `limit=100`（获取最近 100 条新闻）

**输出**:

```python
{"messages": [HumanMessage], "data": state["data"]}
```

每个 ticker 的信号写入 `state["data"]["analyst_signals"][agent_id][ticker]`，结构：

```python
{
    "signal": "bullish" | "bearish" | "neutral",
    "confidence": float,  # 0-100
    "reasoning": {
        "news_sentiment": {
            "signal": str,
            "confidence": float,
            "metrics": {
                "total_articles": int,
                "bullish_articles": int,
                "bearish_articles": int,
                "neutral_articles": int,
                "articles_classified_by_llm": int
            }
        }
    }
}
```

**Pydantic 模型**（LLM 输出结构）:

```python
class Sentiment(BaseModel):
    sentiment: Literal["positive", "negative", "neutral"]
    confidence: int   # 0-100
```

---

## 2. 函数列表

| 函数 | 签名 | 说明 |
|------|------|------|
| `news_sentiment_agent` | `(state: AgentState, agent_id: str = "news_sentiment_agent") -> dict` | 主入口函数 |
| `_calculate_confidence_score` | `(sentiment_confidences: dict, company_news: list, overall_signal: str, bullish_signals: int, bearish_signals: int, total_signals: int) -> float` | 置信度计算 |

---

## 3. 核心逻辑

### 3.1 新闻处理流程

1. **获取新闻**: `get_company_news(ticker, end_date, limit=100)`，若无新闻则默认空列表
2. **识别无情绪标签文章**: 仅在**最新 10 条**（`company_news[:10]`）中找出没有预置 `sentiment` 字段（`sentiment is None`）的文章
3. **LLM 分类**（最多 5 篇）: 对上述无标签文章取前 5 篇，使用 LLM 基于**标题**（仅标题，不含正文）分类情绪，返回 `Sentiment` 模型
4. **情绪映射**: `"negative"` → `"bearish"`；`"positive"` → `"bullish"`；`"neutral"` → `"neutral"`
5. **多数投票**: 对**全部 `company_news`**（最多 100 条）统计 bullish/bearish/neutral 信号数量，取最多者作为整体信号

> **注意**: LLM 分类的候选池是最新 10 条文章（非全部 100 条），再从中最多取 5 篇无标签文章分类；投票/聚合步骤使用全部获取的文章（最多 100 条）。

### 3.2 信号确定规则

```
if bullish_signals > bearish_signals:  signal = "bullish"
elif bearish_signals > bullish_signals: signal = "bearish"
else:                                   signal = "neutral"
```

---

## 4. `_calculate_confidence_score` 置信度计算

**参数**: sentiment_confidences（LLM 为每篇文章返回的置信度字典）、新闻列表、整体信号、各类别计数

**流程**:

1. **若 total_signals == 0**: 返回 `0.0`
2. **若有 LLM 置信度分数**:
   - 找出与 overall_signal 一致的文章
   - 计算这些文章的平均 LLM 置信度
   - 混合公式: `0.7 × avg_llm_confidence + 0.3 × (max(bullish, bearish) / total × 100)`
3. **备用方案**（无 LLM 置信度）:
   - `(max(bullish_signals, bearish_signals) / total_signals) × 100`

---

## 5. 关键常量

| 参数 | 值 | 说明 |
|------|---|------|
| 新闻获取上限 | 100 条 | `get_company_news limit` |
| 投票/聚合范围 | 全部 100 条 | `company_news` 全量统计 |
| LLM 候选池 | 最新 10 条文章（`company_news[:10]`） | 仅在此范围内识别无标签文章 |
| LLM 分类上限 | 候选池中最多 5 条无标签文章 | 节约 Token |
| LLM 置信度权重 | 70% | 混合公式中 LLM 占比 |
| 信号比例权重 | 30% | 混合公式中信号占比 |

---

## 6. 信号生成汇总

| 参数 | 值 |
|------|---|
| 信号来源 | 新闻情绪多数投票 |
| 置信度范围 | 0-100 |
| LLM 分类数量 | 最多 5 篇/ticker |
| 无新闻时默认信号 | neutral, confidence=0 |
| 情绪标签类型 | `Sentiment.confidence` 为 `int` |

---

## 7. 依赖关系

### 项目内部依赖

| 模块 | 导入项 | 用途 |
|------|--------|------|
| `src.graph.state` | `AgentState`, `show_agent_reasoning` | 状态类型定义、推理展示 |
| `src.utils.progress` | `progress` | 进度状态更新 |
| `src.utils.api_key` | `get_api_key_from_state` | 从 state 提取 API 密钥 |
| `src.tools.api` | `get_company_news` | 新闻数据获取 |
| `src.utils.llm` | `call_llm` | LLM 调用封装（用于情绪分类） |
| `src.data.models` | `CompanyNews` | 新闻数据类型 |
| `langchain_core.messages` | `HumanMessage` | 构建消息对象 |

### 外部依赖

| 库 | 导入项 | 用途 |
|----|--------|------|
| `pydantic` | `BaseModel`, `Field` | `Sentiment` 模型定义 |
| `typing_extensions` | `Literal` | 情绪类型约束（`"positive"/"negative"/"neutral"`） |
| `pandas` | `pd` | 情绪数据聚合处理 |
| `numpy` | `np` | 数值计算（置信度统计） |
