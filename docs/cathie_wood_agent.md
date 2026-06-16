# Cathie Wood 代理 (Cathie Wood Agent)

源文件: `src/agents/cathie_wood.py`

---

## 1. 模块概述

Cathie Wood 代理模拟 ARK Invest 创始人 Cathie Wood 的投资风格：专注颠覆性创新技术、指数级增长曲线、大型 TAM（AI、机器人、基因组学、金融科技、区块链），通过综合评分生成信号。

**输入**:

| 字段 | 来源 | 说明 |
|------|------|------|
| `data["tickers"]` | AgentState | 待分析股票代码列表 |
| `data["end_date"]` | AgentState | 财务数据截止日期 |

**数据获取**:

- `get_financial_metrics`: `period="annual"`, `limit=5`
- `search_line_items`: `period="annual"`, `limit=5`，请求 **12 个字段**: `revenue`, `gross_margin`, `operating_margin`, `debt_to_equity`, `free_cash_flow`, `total_assets`, `total_liabilities`, `dividends_and_other_cash_distributions`, `outstanding_shares`, `research_and_development`, `capital_expenditure`, `operating_expense`
- `get_market_cap`: 获取当前市值

**输出**:

```python
{"messages": [HumanMessage], "data": state["data"]}
```

**Pydantic 模型**:

```python
class CathieWoodSignal(BaseModel):
    signal: Literal["bullish", "bearish", "neutral"]
    confidence: float   # 0-100
    reasoning: str
```

---

## 2. 函数列表

| 函数 | 签名 | 说明 |
|------|------|------|
| `cathie_wood_agent` | `(state: AgentState, agent_id: str = "cathie_wood_agent") -> dict` | 主入口函数 |
| `analyze_disruptive_potential` | `(metrics: list, financial_line_items: list) -> dict` | 颠覆性潜力评分 |
| `analyze_innovation_growth` | `(metrics: list, financial_line_items: list) -> dict` | 创新增长能力评分 |
| `analyze_cathie_wood_valuation` | `(financial_line_items: list, market_cap: float) -> dict` | DCF 估值评分 |
| `generate_cathie_wood_output` | `(ticker: str, analysis_data: dict[str, any], state: AgentState, agent_id: str = "cathie_wood_agent") -> CathieWoodSignal` | LLM 信号生成（接收全部 ticker 的 analysis_data 字典） |

---

## 3. 子分析函数详解

### 3.1 `analyze_disruptive_potential`

**原始满分**: 12（代码归一化基数）；归一化至 0-5：`(raw_score / 12) * 5`。注意：基于各评分规则，理论原始最高分为 14（评分规则可叠加至 14 分），故归一化分最高可达约 5.83（略超 5.0）。

**早期退出守卫**: `if not metrics or not financial_line_items` → 返回 `score=0, "Insufficient data"`

| 维度 | 条件 | 得分 |
|------|------|------|
| **收入增长加速** | 增长率加速（最新 > 最早，反序排列） | +2 |
| **收入增长绝对值** | 最新增长 > 100% | +3 |
| | 最新增长 > 50% | +2 |
| | 最新增长 > 20% | +1 |
| **毛利率扩张** | 扩张幅度 > 5% | +2 |
| | 扩张幅度 > 0% | +1 |
| **高绝对毛利率** | 毛利率 > 50% | +2 |
| **运营杠杆** | 收入增长 > 运营费用增长 | +2 |
| **研发投入** | 研发 / 收入 > 15% | +3 |
| | 研发 / 收入 > 8% | +2 |
| | 研发 / 收入 > 5% | +1 |

---

### 3.2 `analyze_innovation_growth`

**原始满分**: 15；归一化至 0-5：`(raw_score / 15) * 5`

**早期退出守卫**: `if not metrics or not financial_line_items` → 返回 `score=0, "Insufficient data"`

| 维度 | 条件 | 得分 |
|------|------|------|
| **研发增长趋势** | 研发增长 > 50% | +3 |
| | 研发增长 > 20% | +2 |
| | 研发强度提升（末期 > 首期） | +2 |
| **FCF 质量** | FCF 增长 > 30% 且所有期均正 | +3 |
| | >= 75% 期间 FCF 为正 | +2 |
| | > 50% 期间 FCF 为正 | +1 |
| **运营效率** | 运营利润率 > 15% 且持续改善 | +3 |
| | 运营利润率 > 10% | +2 |
| | 运营利润率改善 | +1 |
| **资本配置** | CAPEX 强度 > 10% 且 CAPEX 增长 > 20% | +2 |
| | CAPEX 强度 > 5% | +1 |
| **增长再投资** | 分红支付率 < 20% FCF | +2 |
| | 分红支付率 < 40% FCF | +1 |

---

### 3.3 `analyze_cathie_wood_valuation`

**早期退出守卫**:
- `if not financial_line_items or market_cap is None` → `score=0, "Insufficient data"`
- `if fcf <= 0` → `score=0, "No positive FCF for valuation"`

**DCF 参数**:

| 参数 | 值 |
|------|---|
| 年化增长率 | 20% |
| 折现率 | 15% |
| 终值倍数 | 25× |
| 预测年数 | 5 年 |

**得分逻辑（基于安全边际 MOS）**:

| 条件 | 得分 |
|------|------|
| MOS > 50% | +3 |
| MOS > 20% | +1 |
| MOS <= 20% | 0 |

**返回**: `{"score": int, "details": str, "intrinsic_value": float, "margin_of_safety": float}`

---

## 4. 信号生成

### 4.1 总分计算

```
total_score = disruptive_score + innovation_score + valuation_score
max_possible_score = 15
```

### 4.2 信号阈值

| 条件 | 信号 |
|------|------|
| total_score >= 0.7 × 15 = 10.5 | bullish |
| total_score <= 0.3 × 15 = 4.5 | bearish |
| 其他 | neutral |

### 4.3 置信度

由 LLM 生成，类型为 `float`（0-100）。

---

## 5. 分析算法汇总

| 参数 | 值 |
|------|---|
| 子分析数量 | 3（颠覆性潜力、创新增长、估值） |
| 最大总分 | 15（代码硬编码值） |
| 实际可达最高分 | ≈13.83（`analyze_cathie_wood_valuation` 返回原始分 0/1/3，**未**归一化至 0-5；`analyze_innovation_growth` 归一化至 0-5（raw max=15）；`analyze_disruptive_potential` 以 12 为基数归一化，但基于各评分规则理论 raw max 为 14，故归一化后最高约 5.83；实际总分约 5.83+5+3≈13.83） |
| Bullish 阈值 | >= 10.5（70% of max） |
| Bearish 阈值 | <= 4.5（30% of max） |
| DCF 增长率 | 20%（固定） |
| DCF 折现率 | 15%（固定） |
| DCF 终值 | 25× 年化 FCF |
| 数据周期 | annual，5 年 |

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
