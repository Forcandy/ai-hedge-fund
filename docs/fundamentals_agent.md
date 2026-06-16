# 基本面分析代理 (Fundamentals Analyst Agent)

## 1. 模块概述

**源文件**: `src/agents/fundamentals.py` (164 行)
**Python 版本**: 3.11+ (使用 `float | None` 联合类型语法)

基本面分析代理是 AI 对冲基金多代理系统中的分析师节点之一。该模块从 Financial Datasets API 获取财务指标数据，通过四个维度（盈利能力、成长性、财务健康度、估值比率）对股票进行规则驱动的评估，生成综合交易信号和置信度。

本模块不使用 LLM，不涉及任何并发/缓存机制（无 `@lru_cache`、无 `ThreadPoolExecutor`），是纯规则引擎。

**系统角色**: 在 LangGraph `StateGraph` 工作流中，与其他分析师代理并行执行。分析结果写入共享 `AgentState`，供下游风险管理代理和投资组合管理代理消费。

### 输入

| 字段路径 | 类型 | 说明 |
|---------|------|------|
| `state["data"]["tickers"]` | `list[str]` | 待分析的股票代码列表 |
| `state["data"]["end_date"]` | `str` | 分析截止日期 |
| `state["data"]["analyst_signals"]` | `dict` | 共享信号字典，代理写入结果 |
| `state["metadata"]["show_reasoning"]` | `bool` | 是否向标准输出打印推理过程 |
| `state["metadata"]["request"]` | `object` (可选) | Web 后端传入，包含 `api_keys` 字典 |

### 输出

返回值类型: `dict`，包含两个键:

```python
{
    "messages": [HumanMessage(content=<JSON>, name="fundamentals_analyst_agent")],
    "data": dict,  # 原始 data 字典，analyst_signals 已被就地更新
}
```

每个 ticker 的分析结果结构:

```python
{
    "signal": "bullish" | "bearish" | "neutral",
    "confidence": float,  # 0.0 ~ 100.0
    "reasoning": {
        "profitability_signal": {"signal": str, "details": str},
        "growth_signal": {"signal": str, "details": str},
        "financial_health_signal": {"signal": str, "details": str},
        "price_ratios_signal": {"signal": str, "details": str},
    }
}
```

注意: reasoning 字典的第四个键名是 `"price_ratios_signal"`，不是 `"valuation_signal"`。

---

## 2. 核心函数

### `fundamentals_analyst_agent`

**签名** (第 11 行):

```python
def fundamentals_analyst_agent(state: AgentState, agent_id: str = "fundamentals_analyst_agent") -> dict:
```

**参数**:

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `state` | `AgentState` | (必填) | LangGraph 状态对象，TypedDict 包含 `messages`、`data`、`metadata` |
| `agent_id` | `str` | `"fundamentals_analyst_agent"` | 代理标识符，用于进度追踪和信号存储键名 |

**返回值**: `dict`，包含 `"messages"` (list[HumanMessage]) 和 `"data"` (dict)

**副作用**:
1. 就地修改 `state["data"]["analyst_signals"][agent_id]` 写入分析结果
2. 当 `show_reasoning` 为 `True` 时调用 `show_agent_reasoning()` 打印到标准输出
3. 通过 `progress.update_status()` 更新 Rich 终端进度表

**执行流程**:
1. 从 state 提取 `tickers`、`end_date`（直接索引 `state["data"]`，非防御性 `.get()`）
2. 调用 `get_api_key_from_state(state, "FINANCIAL_DATASETS_API_KEY")` 获取 API 密钥
3. 对每个 ticker 循环:
   - 调用 `get_financial_metrics(ticker, end_date, period="ttm", limit=10, api_key=api_key)`
   - 若返回空列表则 `continue` 跳过该 ticker
   - 取 `financial_metrics[0]` 作为最新指标
   - 依次执行四项分析，每项向 `signals` 列表追加一个信号字符串
   - 简单多数投票确定综合信号
   - 计算置信度
4. 将全部 ticker 结果序列化为 JSON，封装为 `HumanMessage` 返回

---

## 3. 分析算法

### 3.1 盈利能力分析 (Profitability Analysis)

**代码位置**: 第 44-60 行

**指标与阈值**:

| 指标 | 字段 | 阈值 | 含义 |
|------|------|------|------|
| 净资产收益率 (ROE) | `return_on_equity` | > 0.15 (15%) | 强劲的股东回报 |
| 净利润率 | `net_margin` | > 0.20 (20%) | 健康的利润水平 |
| 营业利润率 | `operating_margin` | > 0.15 (15%) | 良好的运营效率 |

**计分逻辑** (第 54 行):

```python
profitability_score = sum(metric is not None and metric > threshold for metric, threshold in thresholds)
```

每个指标先检查 `metric is not None`，再与阈值比较。通过一项得 1 分，满分 3 分。当指标为 `None` 时该项不计分（得 0）。

**信号判定** (第 56 行):
- `score >= 2` => `"bullish"`
- `score == 0` => `"bearish"`
- 其他（即 `score == 1`） => `"neutral"`

**reasoning 键**: `"profitability_signal"`

### 3.2 成长性分析 (Growth Analysis)

**代码位置**: 第 62-79 行

**指标与阈值**:

| 指标 | 字段 | 阈值 |
|------|------|------|
| 营收增长率 | `revenue_growth` | > 0.10 (10%) |
| 盈利增长率 | `earnings_growth` | > 0.10 (10%) |
| 账面价值增长率 | `book_value_growth` | > 0.10 (10%) |

**计分与信号判定**: 与盈利能力分析完全一致的模式 -- `sum()` 计分，含 None 安全检查，`>= 2` 看涨，`== 0` 看跌。

**reasoning 键**: `"growth_signal"`

### 3.3 财务健康度分析 (Financial Health)

**代码位置**: 第 81-100 行

此维度使用独立的 `if` 语句逐项累加，逻辑与前两项不同:

```python
health_score = 0
if current_ratio and current_ratio > 1.5:       # 流动比率 > 1.5
    health_score += 1
if debt_to_equity and debt_to_equity < 0.5:      # 负债权益比 < 0.5
    health_score += 1
if free_cash_flow_per_share and earnings_per_share and free_cash_flow_per_share > earnings_per_share * 0.8:
    health_score += 1                            # FCF 转化率 > 80%
```

**None 安全机制**: 使用 Python 真值检查（`if current_ratio and ...`），当值为 `None` 或 `0` 时条件不成立，该项不计分。第三项要求 `free_cash_flow_per_share` 和 `earnings_per_share` 同时为非空非零。

**信号判定**: 同上，`>= 2` 看涨，`== 0` 看跌。

**reasoning 键**: `"financial_health_signal"`

### 3.4 估值比率分析 (Price Ratios Analysis)

**代码位置**: 第 102-119 行

**指标与阈值**:

| 指标 | 字段 | 阈值 |
|------|------|------|
| 市盈率 (P/E) | `price_to_earnings_ratio` | > **25** |
| 市净率 (P/B) | `price_to_book_ratio` | > 3 |
| 市销率 (P/S) | `price_to_sales_ratio` | > 5 |

注意: P/E 阈值是 **25**（第 109 行 `(pe_ratio, 25)`），不是 20。

**计分逻辑** (第 113 行):

```python
price_ratio_score = sum(metric is not None and metric > threshold for metric, threshold in thresholds)
```

**信号反转逻辑** (第 115 行) -- 此处与前三项方向**相反**:

```python
signals.append("bearish" if price_ratio_score >= 2 else "bullish" if price_ratio_score == 0 else "neutral")
```

高 `price_ratio_score`（多项估值指标超过阈值）意味着股票**被高估**，产生**看跌** (`"bearish"`) 信号。所有比率均低于阈值时产生**看涨** (`"bullish"`) 信号。这是因为阈值衡量的是"价格是否过高"，超过越多越贵。

**reasoning 键**: `"price_ratios_signal"`（不是 `"valuation_signal"`）

---

## 4. 信号生成

### 4.1 综合信号 -- 简单多数投票法

**代码位置**: 第 122-131 行

```python
bullish_signals = signals.count("bullish")
bearish_signals = signals.count("bearish")

if bullish_signals > bearish_signals:
    overall_signal = "bullish"
elif bearish_signals > bullish_signals:
    overall_signal = "bearish"
else:
    overall_signal = "neutral"
```

四个维度各产生一个信号，`signals` 列表长度固定为 4。综合判定仅比较看涨与看跌的**计数**，不涉及权重或加权求和。当看涨与看跌数量相等时判定为 `"neutral"`。

### 4.2 置信度计算

**代码位置**: 第 134-135 行

```python
total_signals = len(signals)  # 固定为 4
confidence = round(max(bullish_signals, bearish_signals) / total_signals, 2) * 100
```

**公式**: `confidence = round(max(bullish_count, bearish_count) / 4, 2) * 100`

**取值范围**:

| 主导信号数 | round 结果 | 置信度 | 典型场景 |
|-----------|-----------|--------|---------|
| 4 | 1.0 | 100.0 | 四项一致 |
| 3 | 0.75 | 75.0 | 三项一致 |
| 2 | 0.5 | 50.0 | 2 bullish + 2 bearish 或 2 + 2 neutral |
| 1 | 0.25 | 25.0 | 仅一项非 neutral |
| 0 | 0.0 | 0.0 | 全部为 neutral |

注意: 置信度反映主导信号的占比。当 2 bullish + 2 bearish 时，`max(2,2)/4 = 0.5`，置信度为 50.0，但综合信号为 `"neutral"`。

### 4.3 数据缺失处理

- `get_financial_metrics()` 返回空列表时:  `continue` 跳过该 ticker，不写入结果字典
- 各指标 `None` 值通过 `metric is not None and metric > threshold` 安全忽略，不计入得分
- reasoning 的 details 字段对 `None` 值显示 `"N/A"`:
  ```python
  (f"ROE: {return_on_equity:.2%}" if return_on_equity else "ROE: N/A")
  ```

---

## 5. 依赖关系

### 直接导入

| 模块 | 导入对象 | 用途 |
|------|---------|------|
| `langchain_core.messages` | `HumanMessage` | 构建 LangGraph 消息对象 |
| `src.graph.state` | `AgentState` | 状态类型定义 (TypedDict) |
| `src.graph.state` | `show_agent_reasoning` | 格式化打印推理过程到标准输出 |
| `src.utils.api_key` | `get_api_key_from_state` | 从 state.metadata.request.api_keys 提取 API 密钥 |
| `src.utils.progress` | `progress` | Rich 终端进度追踪单例 (AgentProgress 实例) |
| `json` | (标准库) | JSON 序列化 |
| `src.tools.api` | `get_financial_metrics` | Financial Datasets API 调用 |

### 间接依赖

| 模块 | 说明 |
|------|------|
| `src.data.models.FinancialMetrics` | Pydantic 模型，`get_financial_metrics()` 返回的元素类型。所有财务字段类型为 `float \| None` |
| `src.data.cache` | API 层内部的内存缓存，对本模块透明 |
| `rich` | 通过 `progress` 单例间接使用，渲染终端进度表 |
| `pydantic` | 通过 `FinancialMetrics` 间接使用 |

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

当 Web 后端传入 request 对象且包含 `api_keys` 字典时，提取指定名称的密钥。CLI 模式下返回 `None`，此时 API 层会回退到环境变量。

### 不存在的依赖

本模块**不使用**:
- `pandas` / `numpy`
- `@lru_cache` 装饰器
- `ThreadPoolExecutor` 或任何并发原语
- LLM 调用（纯规则驱动）
