# 情绪分析代理 (Sentiment Analyst Agent)

## 1. 模块概述

**源文件**: `src/agents/sentiment.py` (139 行)
**Python 版本**: 3.11+ (间接依赖使用 `float | None` 语法的数据模型)

情绪分析代理是 AI 对冲基金多代理系统中的分析师节点之一。该模块从两个数据源（内幕交易记录、公司新闻）获取数据，通过加权信号融合生成综合交易信号和置信度。

**系统角色**: 在 LangGraph `StateGraph` 工作流中与其他分析师代理并行执行，分析结果写入共享 `AgentState`，供下游风险管理代理和投资组合管理代理消费。

### 输入

| 字段路径 | 类型 | 访问方式 | 说明 |
|---------|------|---------|------|
| `state["data"]["tickers"]` | `list[str]` | `data.get("tickers")` | 待分析的股票代码列表 |
| `state["data"]["end_date"]` | `str` | `data.get("end_date")` | 分析截止日期 |
| `state["data"]["analyst_signals"]` | `dict` | 直接索引 | 共享信号字典 |
| `state["metadata"]["show_reasoning"]` | `bool` | 直接索引 | 是否打印推理过程 |
| `state["metadata"]["request"]` | `object` (可选) | 通过 `get_api_key_from_state` | Web 后端传入的请求对象 |

注意: 本模块对 `state` 和 `data` 使用防御性 `.get()` 模式（第 14-16 行: `state.get("data", {})`、`data.get("end_date")`），但对 `state["metadata"]["show_reasoning"]` 使用直接索引（第 127 行）。

### 输出

返回值类型: `dict`，包含两个键:

```python
{
    "messages": [HumanMessage(content=<JSON>, name="sentiment_analyst_agent")],
    "data": dict,  # 引用 state.get("data", {}) 返回的 data 对象
}
```

每个 ticker 的分析结果结构:

```python
{
    "signal": "bullish" | "bearish" | "neutral",
    "confidence": float,  # 0.0 ~ 100.0
    "reasoning": {
        "insider_trading": {
            "signal": str,
            "confidence": int,
            "metrics": {
                "total_trades": int,
                "bullish_trades": int,
                "bearish_trades": int,
                "weight": float,           # 0.3
                "weighted_bullish": float,
                "weighted_bearish": float,
            }
        },
        "news_sentiment": {
            "signal": str,
            "confidence": int,
            "metrics": {
                "total_articles": int,
                "bullish_articles": int,
                "bearish_articles": int,
                "neutral_articles": int,
                "weight": float,           # 0.7
                "weighted_bullish": float,
                "weighted_bearish": float,
            }
        },
        "combined_analysis": {
            "total_weighted_bullish": float,
            "total_weighted_bearish": float,
            "signal_determination": str,
        }
    }
}
```

---

## 2. 核心函数

### `sentiment_analyst_agent`

**签名** (第 12 行):

```python
def sentiment_analyst_agent(state: AgentState, agent_id: str = "sentiment_analyst_agent") -> dict:
```

**参数**:

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `state` | `AgentState` | (必填) | LangGraph 状态对象 |
| `agent_id` | `str` | `"sentiment_analyst_agent"` | 代理标识符 |

**返回值**: `dict`，包含 `"messages"` 和 `"data"`

**副作用**:
1. 就地修改 `state["data"]["analyst_signals"][agent_id]` 写入分析结果
2. 当 `show_reasoning` 为 `True` 时调用 `show_agent_reasoning()` 打印到标准输出
3. 通过 `progress.update_status()` 更新 Rich 终端进度表

**执行流程**:
1. 使用防御性 `.get()` 从 state 提取 data、tickers、end_date
2. 调用 `get_api_key_from_state(state, "FINANCIAL_DATASETS_API_KEY")` 获取 API 密钥
3. 对每个 ticker 循环:
   - 获取内幕交易数据并转换为信号列表
   - 获取公司新闻并转换为信号列表
   - 加权融合两组信号
   - 计算综合信号和置信度
   - 构建结构化 reasoning 字典
4. 序列化为 JSON，封装为 `HumanMessage` 返回

---

## 3. 分析算法

### 3.1 内幕交易信号转换

**代码位置**: 第 35-36 行

```python
transaction_shares = pd.Series([t.transaction_shares for t in insider_trades]).dropna()
insider_signals = np.where(transaction_shares < 0, "bearish", "bullish").tolist()
```

**逻辑**:
- 从每笔 `InsiderTrade` 提取 `transaction_shares` 字段
- 使用 `pd.Series.dropna()` 过滤掉 `None` 值
- `transaction_shares < 0`（卖出）=> `"bearish"`
- `transaction_shares >= 0`（买入或零）=> `"bullish"`

注意: `transaction_shares == 0` 的交易会被判定为 `"bullish"`（`np.where` 的 else 分支）。

**API 调用**: `get_insider_trades(ticker=ticker, end_date=end_date, limit=1000, api_key=api_key)`

### 3.2 新闻情绪信号转换

**代码位置**: 第 44-46 行

```python
sentiment = pd.Series([n.sentiment for n in company_news]).dropna()
news_signals = np.where(sentiment == "negative", "bearish",
                      np.where(sentiment == "positive", "bullish", "neutral")).tolist()
```

**逻辑**:
- 从每条 `CompanyNews` 提取 `sentiment` 字段（值域: `"positive"` / `"negative"` / `None` 及其他）
- `dropna()` 过滤 `None` 值
- 三路映射: `"negative"` => `"bearish"`、`"positive"` => `"bullish"`、其他 => `"neutral"`

**API 调用**: `get_company_news(ticker, end_date, limit=100, api_key=api_key)`

### 3.3 加权信号融合

**代码位置**: 第 49-68 行

**权重配置**:
- `insider_weight = 0.3` (内幕交易占 30%)
- `news_weight = 0.7` (新闻情绪占 70%)

**加权信号计算**:

```python
bullish_signals = (
    insider_signals.count("bullish") * insider_weight +
    news_signals.count("bullish") * news_weight
)
bearish_signals = (
    insider_signals.count("bearish") * insider_weight +
    news_signals.count("bearish") * news_weight
)
```

注意: 这里的 `bullish_signals` 和 `bearish_signals` 是 `float` 类型的加权值，不是简单计数。neutral 信号不参与加权计算（不计入看涨也不计入看跌）。

**综合信号判定** (第 63-68 行):

```python
if bullish_signals > bearish_signals:
    overall_signal = "bullish"
elif bearish_signals > bullish_signals:
    overall_signal = "bearish"
else:
    overall_signal = "neutral"
```

---

## 4. 信号生成

### 4.1 置信度计算

**代码位置**: 第 71-74 行

```python
total_weighted_signals = len(insider_signals) * insider_weight + len(news_signals) * news_weight
confidence = 0  # 默认值
if total_weighted_signals > 0:
    confidence = round((max(bullish_signals, bearish_signals) / total_weighted_signals) * 100, 2)
```

**公式**: `confidence = round(max(weighted_bullish, weighted_bearish) / total_weighted_signals * 100, 2)`

**边界条件处理**: 通过 `if total_weighted_signals > 0` 条件守卫，避免除零错误。当内幕交易和新闻数据均为空时，`total_weighted_signals` 为 0，置信度保持默认值 0。这不是使用 `max()` 函数防止除零，而是显式的条件判断。

**分母含义**: `total_weighted_signals` 是所有信号数量的加权总和（包括 neutral），不仅是 bullish + bearish。

**示例计算**:
- 10 笔内幕交易（8 bullish, 2 bearish），20 条新闻（12 bullish, 5 bearish, 3 neutral）
- `bullish = 8*0.3 + 12*0.7 = 2.4 + 8.4 = 10.8`
- `bearish = 2*0.3 + 5*0.7 = 0.6 + 3.5 = 4.1`
- `total_weighted = 10*0.3 + 20*0.7 = 3.0 + 14.0 = 17.0`
- `confidence = round(10.8/17.0 * 100, 2) = 63.53`

### 4.2 子信号置信度

reasoning 字典中每个子信号也有独立的 confidence 值:

**内幕交易子信号** (第 81 行):
```python
"confidence": round((max(insider_signals.count("bullish"), insider_signals.count("bearish")) / max(len(insider_signals), 1)) * 100)
```
分母使用 `max(len(insider_signals), 1)` 防止除零。结果用 `round()` 取整为 `int`。

**新闻子信号** (第 94 行): 逻辑相同，使用 `max(len(news_signals), 1)` 防止除零。

### 4.3 reasoning 结构

reasoning 包含三个顶层键:

1. **`"insider_trading"`**: 内幕交易维度的 signal、confidence、metrics
2. **`"news_sentiment"`**: 新闻情绪维度的 signal、confidence、metrics
3. **`"combined_analysis"`**: 加权融合结果，包含 `total_weighted_bullish`、`total_weighted_bearish` 和文本描述

`combined_analysis.signal_determination` 示例值: `"Bullish based on weighted signal comparison"`

---

## 5. 依赖关系

### 直接导入

| 模块 | 导入对象 | 用途 |
|------|---------|------|
| `langchain_core.messages` | `HumanMessage` | 构建 LangGraph 消息对象 |
| `src.graph.state` | `AgentState` | 状态类型定义 (TypedDict) |
| `src.graph.state` | `show_agent_reasoning` | 格式化打印推理过程 |
| `src.utils.api_key` | `get_api_key_from_state` | 从 state 中安全提取 API 密钥 |
| `src.utils.progress` | `progress` | Rich 终端进度追踪单例 |
| `pandas` | `pd` | Series 构造和 `dropna()` 过滤 |
| `numpy` | `np` | `np.where()` 条件向量化映射 |
| `json` | (标准库) | JSON 序列化 |
| `src.tools.api` | `get_insider_trades` | Financial Datasets API -- 内幕交易数据 |
| `src.tools.api` | `get_company_news` | Financial Datasets API -- 公司新闻数据 |

### 间接依赖

| 模块 | 说明 |
|------|------|
| `src.data.models.InsiderTrade` | Pydantic 模型，`get_insider_trades()` 返回元素类型。关键字段: `transaction_shares: float \| None` |
| `src.data.models.CompanyNews` | Pydantic 模型，`get_company_news()` 返回元素类型。关键字段: `sentiment: str \| None` |
| `src.data.cache` | API 层内部缓存，对本模块透明 |
| `rich` | 通过 `progress` 单例间接使用 |
| `pydantic` | 通过数据模型间接使用 |

### `get_api_key_from_state` 详情

**文件**: `src/utils/api_key.py`

```python
def get_api_key_from_state(state: dict, api_key_name: str) -> str:
    if state and state.get("metadata", {}).get("request"):
        request = state["metadata"]["request"]
        if hasattr(request, 'api_keys') and request.api_keys:
            return request.api_keys.get(api_key_name)
    return None
```

本模块在第 17 行调用:
```python
api_key = get_api_key_from_state(state, "FINANCIAL_DATASETS_API_KEY")
```

获取到的 `api_key` 传递给 `get_insider_trades()` 和 `get_company_news()` 两个 API 调用。当 Web 后端传入 request 对象且包含 api_keys 字典时提取密钥；CLI 模式下返回 `None`，API 层回退到环境变量。

### 与 fundamentals_agent 的关键差异

| 方面 | fundamentals_agent | sentiment_agent |
|------|-------------------|-----------------|
| state 访问 | 直接索引 `state["data"]` | 防御性 `state.get("data", {})` |
| pandas/numpy | 不使用 | 使用 |
| 信号聚合 | 简单多数投票（等权） | 加权融合（0.3/0.7） |
| 数据源 | 1 个 (financial_metrics) | 2 个 (insider_trades + company_news) |
| LLM 调用 | 无 | 无 |
