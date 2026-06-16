# 成长分析师代理 (Growth Analyst Agent)

源文件: `src/agents/growth_agent.py`

---

## 1. 模块概述

成长分析师代理专注于识别具有强劲成长趋势、合理估值和健康财务状况的公司。代理完全基于数值计算（不调用 LLM），通过加权综合评分生成信号。

**重要**: 此代理不使用 `generate_output` / LLM 调用，信号由纯数学计算决定。

**输入**:

| 字段 | 来源 | 说明 |
|------|------|------|
| `data["tickers"]` | AgentState | 待分析股票代码列表 |
| `data["end_date"]` | AgentState | 财务数据截止日期 |

**数据获取**:

- `get_financial_metrics`: `period="ttm"`, `limit=12`（获取 3 年 TTM 数据）
- `get_insider_trades`: `limit=1000`

**早期退出守卫**: `if not financial_metrics or len(financial_metrics) < 4` → 跳过该 ticker，状态更新为 "Failed: Not enough financial metrics"

**输出**:

```python
{"messages": [HumanMessage], "data": state["data"]}
```

每个 ticker 的信号写入 `state["data"]["analyst_signals"][agent_id][ticker]`，结构：

```python
{
    "signal": "bullish" | "bearish" | "neutral",
    "confidence": int,  # 0-100
    "reasoning": {
        "historical_growth": dict,
        "growth_valuation": dict,
        "margin_expansion": dict,
        "insider_conviction": dict,
        "financial_health": dict,
        "final_analysis": {
            "signal": str,
            "confidence": int,
            "weighted_score": float
        }
    }
}
```

---

## 2. 函数列表

| 函数 | 签名 | 说明 |
|------|------|------|
| `growth_analyst_agent` | `(state: AgentState, agent_id: str = "growth_analyst_agent") -> dict` | 主入口函数 |
| `_calculate_trend` | `(data: list[float \| None]) -> float` | 线性回归斜率辅助函数 |
| `analyze_growth_trends` | `(metrics: list) -> dict` | 成长趋势评分（0-1.0） |
| `analyze_valuation` | `(metrics) -> dict` | 估值评分（0-1.0），接收单个 FinancialMetrics 对象 |
| `analyze_margin_trends` | `(metrics: list) -> dict` | 利润率趋势评分（0-1.0） |
| `analyze_insider_conviction` | `(trades: list) -> dict` | 内部人信念评分（0-1.0） |
| `check_financial_health` | `(metrics) -> dict` | 财务健康评分（0-1.0），接收单个 FinancialMetrics 对象 |

---

## 3. 辅助函数

### `_calculate_trend(data: list[float | None]) -> float`

线性回归斜率，用于判断指标是否在改善。

**早期退出**: `if len(clean_data) < 2` 或 `ZeroDivisionError` → 返回 `0.0`

---

## 4. 子分析函数详解

### 4.1 `analyze_growth_trends`

**满分**: 1.0

**返回**: `{"score": float, "revenue_growth": float, "revenue_trend": float, "eps_growth": float, "eps_trend": float, "fcf_growth": float, "fcf_trend": float}`

| 维度 | 条件 | 得分 |
|------|------|------|
| **收入增长**（最新） | > 20% | +0.40 |
| | > 10% | +0.20 |
| 收入趋势（斜率）为正 | `revenue_trend > 0` | +0.10 |
| **EPS 增长**（最新） | > 20% | +0.25 |
| | > 10% | +0.10 |
| EPS 趋势为正 | `eps_trend > 0` | +0.05 |
| **FCF 增长**（最新） | > 15% | +0.10 |

---

### 4.2 `analyze_valuation`

**满分**: 1.0

**返回**: `{"score": float, "peg_ratio": float | None, "price_to_sales_ratio": float | None}`

| 维度 | 条件 | 得分 |
|------|------|------|
| **PEG 比率** | < 1.0 | +0.50 |
| | < 2.0 | +0.25 |
| **市销率 (P/S)** | < 2.0 | +0.50 |
| | < 5.0 | +0.25 |

---

### 4.3 `analyze_margin_trends`

**满分**: 1.0

**返回**: `{"score": float, "gross_margin": float, "gross_margin_trend": float, "operating_margin": float, "operating_margin_trend": float, "net_margin": float, "net_margin_trend": float}`

| 维度 | 条件 | 得分 |
|------|------|------|
| **毛利率**（最新） | > 50% | +0.20 |
| 毛利率趋势为正 | `gross_margin_trend > 0` | +0.20 |
| **运营利润率**（最新） | > 15% | +0.20 |
| 运营利润率趋势为正 | `operating_margin_trend > 0` | +0.20 |
| **净利润率趋势** | `net_margin_trend > 0` | +0.20 |

---

### 4.4 `analyze_insider_conviction`

**满分**: 1.0

**返回**: `{"score": float, "net_flow_ratio": float, "buys": float, "sells": float}`

注意: `buys` 和 `sells` 为交易价值（美元金额）的累计值，来自 `transaction_value` 字段，而非交易次数。

使用 `transaction_shares` 符号区分买入（正）和卖出（负）：

```
net_flow_ratio = (buys - sells) / (buys + sells)
```

| 条件 | 得分 |
|------|------|
| net_flow_ratio > 0.5 | 1.0 |
| net_flow_ratio > 0.1 | 0.7 |
| > -0.1 且 <= 0.1（中性） | 0.5 |
| net_flow_ratio < -0.1 | 0.2 |

---

### 4.5 `check_financial_health`

**满分**: 1.0（从 1.0 开始扣分，下限 0.0）

**返回**: `{"score": float, "debt_to_equity": float | None, "current_ratio": float | None}`

| 维度 | 条件 | 扣分 |
|------|------|------|
| **负债/权益** | > 1.5（高杠杆） | -0.50 |
| | > 0.8（中等杠杆） | -0.20 |
| **流动比率** | < 1.0（流动性不足） | -0.50 |
| | < 1.5（流动性偏低） | -0.20 |

---

## 5. 信号生成

### 5.1 加权总分计算（满分 1.0）

```
weighted_score = (
    growth_score   * 0.40 +
    valuation_score * 0.25 +
    margin_score   * 0.15 +
    insider_score  * 0.10 +
    health_score   * 0.10
)
```

| 子分析 | 权重 |
|--------|------|
| 成长趋势 | 40% |
| 估值 | 25% |
| 利润率趋势 | 15% |
| 内部人信念 | 10% |
| 财务健康 | 10% |

### 5.2 信号阈值

| 条件 | 信号 |
|------|------|
| weighted_score > 0.6 | bullish |
| weighted_score < 0.4 | bearish |
| 0.4 <= weighted_score <= 0.6 | neutral |

### 5.3 置信度公式

```
confidence = round(abs(weighted_score - 0.5) * 2 * 100)
```

即: 得分距离 0.5 越远，置信度越高；得分恰好为 0.5 时置信度为 0。

---

## 6. 分析算法汇总

| 参数 | 值 |
|------|---|
| 子分析数量 | 5（成长、估值、利润率、内部人、健康） |
| 加权满分 | 1.0 |
| Bullish 阈值 | > 0.6 |
| Bearish 阈值 | < 0.4 |
| 置信度公式 | `abs(score - 0.5) * 2 * 100` |
| 早期退出 | < 4 期财务指标 |
| 数据周期 | ttm，12 期 |
| LLM 调用 | 无（纯计算） |

---

## 7. 依赖关系

### 标准库

| 模块 | 用途 |
|------|------|
| `__future__.annotations` | 启用延迟类型注解求值（Python 3.7+ PEP 563） |
| `json` | 序列化分析结果 |
| `statistics` | 已导入但未使用（未来计算预留，目前 `growth_agent.py` 中无实际调用） |

### 项目内部依赖

| 模块 | 导入项 | 用途 |
|------|--------|------|
| `src.graph.state` | `AgentState`, `show_agent_reasoning` | 状态类型定义、推理展示 |
| `src.utils.progress` | `progress` | 进度状态更新 |
| `src.utils.api_key` | `get_api_key_from_state` | 从 state 提取 API 密钥 |
| `src.tools.api` | `get_financial_metrics`, `get_insider_trades` | 财务指标和内部人交易数据 |
| `langchain_core.messages` | `HumanMessage` | 构建消息对象 |
