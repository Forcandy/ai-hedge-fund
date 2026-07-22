# v2 Alpha 模型与 LLM 投资人系统 (v2 Signals & LLM System)

源目录: `v2/signals/`、`v2/llm/`、`v2/features/`、`v2/demo/`

---

## 1. 模块概述

v2 的核心抽象是 **Alpha 模型（`AlphaModel`）**：任何对某只股票形成观点、返回 `Signal`（`[-1, +1]` 区间的信念值 + 文字理由）的组件。两种风格共享同一接口：

- **量化模型（`QuantModel`）**：纯数学/数据驱动，如盈余公告后漂移（PEAD）
- **LLM 投资人 Agent（`LLMAgent`）**：以某位知名投资人的口吻，对逐笔时点正确的基本面快照进行推理，目前有 5 位：Warren Buffett、Charlie Munger、Benjamin Graham、Peter Lynch、Stanley Druckenmiller

两者都能不加区分地接入同一个回测引擎（`BacktestEngine.run_alpha`，见 [`v2_backtesting_system.md`](./v2_backtesting_system.md)），也是 `run_cycle` 流水线（见 [`v2_fund_system.md`](./v2_fund_system.md)）的统一分析师接口——一个模型只需实现一次 `predict()`，就能同时用于单独回测、被某个策略（"pod"）征召、或在实盘/纸面交易中运行。

> **v2.0.0 变更提示**：早期版本的单股票 CLI `v2/analyze.py` 已被删除，其"向某位分析师询问一只股票的时点观点"能力被更完整的 `poetry run python -m v2.run`（交互式建基金 + 跑 `run_cycle`）取代，详见 [`v2_fund_system.md`](./v2_fund_system.md)。

| 目录/文件 | 职责 |
|-----------|------|
| `v2/signals/base.py` | `AlphaModel`（抽象基类）+ `QuantModel`（量化模型公共数值工具） |
| `v2/signals/llm_agent.py` | `LLMAgent`（LLM 投资人 Agent 公共基类，承载全部机械逻辑） |
| `v2/signals/buffett.py` | `BuffettAgent`——Warren Buffett 人格 |
| `v2/signals/munger.py` | `MungerAgent`——Charlie Munger 人格 |
| `v2/signals/graham.py` | `GrahamAgent`——Benjamin Graham 人格 |
| `v2/signals/lynch.py` | `LynchAgent`——Peter Lynch 人格 |
| `v2/signals/druckenmiller.py` | `DruckenmillerAgent`——Stanley Druckenmiller 人格 |
| `v2/signals/pead.py` | `PEADModel`——盈余公告后漂移量化模型 |
| `v2/signals/__init__.py` | 公开导出 + `ALPHA_MODEL_REGISTRY`（6 个模型） |
| `v2/llm/client.py` | `LLMClient` 协议 + `AnthropicLLM` 实现 + `extract_json` |
| `v2/llm/cache.py` | `PromptCache`——按 prompt 内容哈希缓存 LLM 决策到磁盘 |
| `v2/features/snapshot.py` | `FundamentalsSnapshot`——LLM Agent 的共享时点正确输入 |
| `v2/demo/backtest.py` | PEAD 回测演示仪表盘（终端实时展示） |

---

## 2. `AlphaModel` 接口（`v2/signals/base.py`）

```python
class AlphaModel(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...

    @abstractmethod
    def predict(self, ticker: str, date: str, data_client: DataClient) -> Signal: ...
```

**契约**（docstring 摘要）：
- **必须时点正确（point-in-time）**：只能使用 `date <= 传入的 date` 的数据，禁止使用未来信息
- 返回的 `Signal.value` 必须在 `[-1, +1]` 区间；`0.0` 表示"无观点"（弃权，abstain）
- **只形成观点，不决定仓位机制**（时机、大小、持有期）——那是回测引擎 / 未来投资组合构建层的职责，这个分离是刻意的设计决策

`AlphaModel` 有两个已实现的子类分支：

```
AlphaModel (ABC)
  ├─ QuantModel   — 纯 Python 数学（base.py）
  └─ LLMAgent     — LLM 对特征推理（llm_agent.py）
```

### 2.1 `QuantModel`（`base.py`）

量化模型的公共基类，提供共享数值工具（均为 `@staticmethod`，子类可直接调用）：

| 方法 | 签名 | 说明 |
|------|------|------|
| `_safe_float` | `(value, default=0.0) -> float` | 转 float，`None`/NaN/Inf/异常 均返回 `default` |
| `_percentile_rank` | `(value, values) -> float` | `value` 在 `values` 中的百分位排名（0-100） |
| `_normalize_to_signal` | `(raw, low=-1.0, high=1.0) -> float` | 将 `raw` 裁剪到 `[low, high]` |
| `_sigmoid` | `(x, scale=5.0) -> float` | 通过缩放 `tanh` 把无界值映射到 `(-1, +1)` |
| `_compute_rsi` | `(prices: pd.Series, period=14) -> float` | 计算价格序列最新一期的 RSI |

`PEADModel` 是当前唯一的具体 `QuantModel` 实现（见第 4 节）。

---

## 3. `LLMAgent`（`v2/signals/llm_agent.py`）

LLM 投资人 Agent 的公共基类。子类只需定义人格：一个名字 + 一段系统提示词，其余所有机械逻辑（构建快照、调用 LLM、缓存、解析、失败处理）都由基类承担。

```python
class BuffettAgent(LLMAgent):
    @property
    def name(self) -> str:
        return "buffett"

    def get_system_prompt(self) -> str:
        return "You are Warren Buffett..."
```

### 3.1 构造函数

```python
LLMAgent(llm: LLMClient | None = None, cache: PromptCache | None = None)
```

不传时默认使用 `AnthropicLLM()` 和 `PromptCache()`。可注入 mock/自定义实现用于测试。

### 3.2 失败契约（锁定设计决策）

| 失败类型 | 行为 |
|---------|------|
| 数据层异常（如快照数据不足以外的 `FDClientError` 等） | **向上传播（fail loud）**——一次损坏的快照绝不能悄悄变成"中性观点" |
| `InsufficientData`（快照期数不足） | 捕获并转为弃权信号 |
| LLM 调用失败（网络、超时等） | 捕获并转为弃权信号 |
| LLM 响应解析失败（非法 JSON、字段缺失/越界） | 捕获并转为弃权信号，**同时把原始响应连同解析错误一并写入缓存**，留作调试线索 |

弃权信号：`Signal(value=0.0, metadata={"abstained": True, "abstain_reason": ...})`

### 3.3 `predict()` 执行流程

```
build_snapshot(ticker, date, data_client)          # InsufficientData -> 弃权；其余异常传播
  |
system = get_system_prompt()                        # 子类定义的人格
user   = build_user_prompt(snapshot)                 # 默认：snapshot.render()
key    = prompt_key(name, model, system, user)        # 缓存键
  |
命中缓存（PromptCache.get(key) 且含 "parsed"）？
  |-- 是 --> 直接用缓存的 parsed 结果构建 Signal（metadata.cached=True）
  |-- 否 --> llm.complete(system, user)               # 调用失败 -> 弃权
              |
             _parse(response)                          # 解析失败 -> 缓存原始响应+错误，返回弃权
              |
             PromptCache.put(key, {...parsed})          # 写入缓存
              |
             构建 Signal（metadata.cached=False）
```

**响应解析规则**（`_parse`）：从响应中提取 JSON（见 `extract_json`），校验 `signal` 必须是 `"bullish"`/`"bearish"`/`"neutral"` 之一，`confidence` 必须在 `[0, 100]`，否则抛出 `ValueError`（触发弃权路径）。

**Signal.value 换算**：`value = sign(signal) * confidence / 100.0`，其中 `sign` 映射为 `bullish=+1.0`、`neutral=0.0`、`bearish=-1.0`。

### 3.4 子类可覆盖的方法

| 方法 | 默认行为 | 何时覆盖 |
|------|---------|---------|
| `get_system_prompt()` | 无默认实现，**必须**覆盖 | 每个人格都要定义自己的系统提示词 |
| `build_snapshot(ticker, date, data_client)` | 返回共享的 `FundamentalsSnapshot`（`build_snapshot()` 函数，见第 5 节） | 价值/质量类人格无需覆盖；需要推理其他数据（宏观、新闻）的人格才覆盖。文档明确指出：只有当出现第二种快照类型时，才把隐含接口（ticker/as_of/content_hash/render）抽成 Protocol，不要提前抽象 |
| `build_user_prompt(snapshot)` | 返回 `snapshot.render()` | 需要在快照之外补充提示词内容时覆盖 |

---

## 4. LLM 投资人人格（`v2/signals/{buffett,munger,graham,lynch,druckenmiller}.py`）

五个已实现的 LLM 投资人分析师，都是对应投资人公开投资哲学的风格化近似（`VISION.md` 明确说明：这些人格不是真人本人，也不构成对其的背书）。

**每个类只包含一个系统提示词**——全部机械逻辑都在 `LLMAgent` 基类中（见第 3 节），数据全部来自共享的时点正确 `FundamentalsSnapshot`（见第 5 节）。五个人格共享的硬性规则：只能从提供的数据推理，把快照中最新一期的申报日期当作"现在"，不得使用其之后发生的任何知识，不得捏造数字；响应必须是严格 JSON：`{"signal": "bullish"|"bearish"|"neutral", "confidence": <0-100>, "reasoning": "..."}`。

### 4.1 `BuffettAgent`——Warren Buffett（`buffett.py`）

系统提示词要点：能力圈（数据是否足以理解这门生意）→ 护城河（ROE 是否持续高企、利润率是否稳定或改善、是否有定价权）→ 管理层质量（账面价值复利增长、杠杆是否合理、自由现金流是否稳定）→ 财务实力（低负债、健康流动比率、稳定盈利）→ 估值（"合理价格的好公司胜过便宜价格的普通公司"）→ 长期前景（是否愿意持有十年）。

**信号规则**：bullish=优质且持续、价格合理或更好；bearish=业务疲软/恶化，或价格要求完美预期；neutral=证据混杂，或优质公司但价格明显过高。

**置信度量表**：90-100 极强信念（证据充分）；70-89 稳固信念；40-69 混杂；10-39 弱/投机性。

### 4.2 `MungerAgent`——Charlie Munger（`munger.py`）

强调"反过来想"（invert）：先想什么会让这笔投资失败，再看生意质量是否历年持续（不是某一年好）、账面价值是否真实复利增长、估值是否合理。判断不清时明确归入"太难判断"（too-hard pile）并给中性。语气刻意直白、不留余地。

**信号规则**：bullish=毫无疑问的优质生意且价格不离谱；bearish=平庸/恶化的生意、可疑的数字，或估值需要相信蠢事才成立；neutral=太难判断，或优质生意但价格不愿意付。

**置信度量表**：90-100 罕见的、质量与价格都对齐的明显案例；70-89 稳固；40-69 证据混杂；10-39 多半属于"太难判断"。

### 4.3 `GrahamAgent`——Benjamin Graham（`graham.py`）

价值投资之父，作为防御型投资者评估：安全边际优先（P/E、市净率相对保守标准是否够低）→ 财务实力（流动比率 > 1.5、低负债权益比）→ 盈利稳定性（历史记录中是否持续为正、无剧烈波动）→ 对"为成长溢价买单"高度警惕。

**信号规则**：bullish=稳健生意、强健资产负债表、价格提供真实安全边际；bearish=财务薄弱、盈利不稳定，或价格透支了希望而非已证实的业绩（估值过高本身就是利空事实）；neutral=生意健全但安全边际不足。

**置信度量表**：90-100 每条标准都有清晰的量化支撑；70-89 多数标准满足；40-69 混杂；10-39 投机性区域。

### 4.4 `LynchAgent`——Peter Lynch（`lynch.py`）

"了解你持有的东西"：先把公司归类（快速增长者/稳健增长者/缓慢增长者/困境反转），再做 PEG 测试（P/E 相对可见的盈利增长率是否有吸引力），检查故事是否成立（营收增长转化为盈利增长、利润率保持或改善），并避开高负债公司。倡导用大白话讲清楚投资逻辑，讲不清楚就中性。

**信号规则**：bullish=真实可见的盈利增长，且 P/E 尚未提前计入（PEG 有吸引力）；bearish=增长放缓却享受高溢价倍数，或"故事很热但数字在变冷"；neutral=公司不错但已充分定价，或数据不足以判断类别。

**置信度量表**：90-100 经典设置（增长便宜且可见）；70-89 故事好、价格公道；40-69 混杂；10-39 看不清持有的是什么。

### 4.5 `DruckenmillerAgent`——Stanley Druckenmiller（`druckenmiller.py`）

关注拐点而非静态水平：扫描最近几个季度相对更早时期，判断营收增长和利润率是在加速还是转弱，EPS 动能是否在积累；再问"市场已经price in了什么"，寻找拐点与价格错位的不对称机会；只有拐点和价格同时对齐才重仓，否则不出手。文档明确标注了**诚实的能力边界**：该人格目前只对基本面快照推理，尚无宏观、利率或价格走势数据，因此只能从基本面自身的变化趋势中寻找拐点。

**信号规则**：bullish=近期季度出现明显加速、且价格尚未完全反映；bearish=明显恶化或转弱，尤其是价格仍按旧趋势定价时；neutral=看不出明显拐点，或趋势与价格已经完全一致。

**置信度量表**：90-100 明确无误的拐点 + 不对称机会；70-89 稳固的趋势变化；40-69 混杂或为时尚早；10-39 无优势可言。

---

## 5. `FundamentalsSnapshot`（`v2/features/snapshot.py`）

LLM 分析师的共享输入：一位投资人 Agent 在给定日期**被允许知道**的关于一家公司的全部信息——一段财务指标历史（每一行都可证明在 `as_of` 之前已公开，数据层按 `filing_date` 而非 `report_period` 过滤）+ 一些用 Python 预先算好的衍生聚合指标（让 LLM 基于事实推理，而不是重新做算术）。

`content_hash` 是快照内容的稳定哈希，作为 LLM 调用的缓存键——只有当新的财报改变了快照内容，Agent 才会重新推理。

**`content_hash` 与 `render()` 均刻意排除 `as_of` 字段**（`model_dump_json(exclude={"as_of"})`）：两个不同的日期，只要期间没有新的财报申报，看到的基本面数据完全相同，就应该命中同一个缓存键、生成完全相同的 prompt——而不是每换一个日期就重新触发一次真实的 LLM 调用。`.render()` 的文案也相应改为不出现具体日期（"Treat the most recent filing shown as the present"），既保证了同一份基本面数据无论在哪一天问都渲染成完全一致的文本，也避免 LLM 把提示词里的日历日期和该日期之后发生的真实世界事件联系起来。

### 5.1 `build_snapshot(ticker, as_of, data_client, periods=20) -> FundamentalsSnapshot`

```python
metrics = data_client.get_financial_metrics(ticker, as_of, period="ttm", limit=periods)
if len(metrics) < MIN_PERIODS:      # MIN_PERIODS = 4（至少一年的 ttm 数据）
    raise InsufficientData(...)
```

- 数据不足 `MIN_PERIODS`（4 期）时抛出 `InsufficientData`（`LLMAgent.predict` 会捕获并转为弃权）
- 市值取自**最近一期已归档的 `metrics` 行**，而非 `data_client.get_market_cap()`——后者优先取 `company_facts.market_cap`（只有最新值），若用在历史时点会造成未来数据泄漏
- `sector`/`industry` 取自 `get_company_facts()` 的最新值——这是一个已知且文档化的时点正确近似（行业分类变化极慢，可接受）

### 5.2 `FundamentalsSnapshot` 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `ticker` | `str` | 股票代码 |
| `as_of` | `str` | 快照的时点日期 |
| `sector` / `industry` | `str \| None` | 行业分类（近似值，见上） |
| `periods` | `list[PeriodFundamentals]` | 各报告期指标，最新在前 |
| `roe_avg` | `float \| None` | 历史 ROE 均值 |
| `net_margin_avg` | `float \| None` | 历史净利润率均值 |
| `gross_margin_trend` | `float \| None` | 毛利率趋势（最新减最旧） |
| `bvps_cagr` | `float \| None` | 每股账面价值年化复合增长率（季度间隔，年数=`(期数-1)/4`） |
| `debt_to_equity_latest` | `float \| None` | 最新债务权益比 |
| `market_cap_latest` | `float \| None` | 最新市值（见上，来自最近已归档 metrics 行） |

`PeriodFundamentals` 每期字段：`report_period`、`filing_date`、`market_cap`、`price_to_earnings_ratio`、`return_on_equity`、`gross_margin`、`operating_margin`、`net_margin`、`debt_to_equity`、`current_ratio`、`revenue_growth`、`earnings_per_share`、`book_value_per_share`、`free_cash_flow_per_share`。

`.render()` 方法把快照压缩成给 LLM 看的紧凑文本块（公司信息 + 摘要统计 + 逐期指标表格）。

---

## 6. `v2/llm/` — LLM 提供商层

镜像 `v2/data/protocol.py` 的 `DataClient` 设计：Agent 依赖 `LLMClient` 协议，从不直接依赖具体提供商，任何实现了 `complete(system, user) -> str` 方法的类都可以接入。

### 6.1 `LLMClient` 协议（`client.py`）

```python
@runtime_checkable
class LLMClient(Protocol):
    model: str
    def complete(self, system: str, user: str) -> str: ...
```

**契约**：`complete()` 返回模型的原始文本；提供商应在传输失败时抛出异常——由 `LLMAgent` 层决定是否弃权，提供商本身不做这个判断。

### 6.2 `AnthropicLLM`（`client.py`）

通过项目已有的 `langchain-anthropic` 依赖实现（仅用于传输，不使用 LangChain 的结构化输出机制——其强制工具调用模式在 Anthropic 推理模型上会出错，v1 也有相同的绕行方案）。改为在提示词中要求 JSON，自行解析。

```python
AnthropicLLM(model: str | None = None, timeout: float = 60.0, max_tokens: int = 1024)
```

- 需要环境变量 `ANTHROPIC_API_KEY`，缺失时构造函数直接抛出 `ValueError`
- 模型名优先级：显式 `model` 参数 > 环境变量 `V2_LLM_MODEL` > `DEFAULT_MODEL`（`"claude-sonnet-5"`）
- `complete()` 内部处理 Anthropic 推理模型返回的分块内容列表（`list[dict]`），拼接为纯文本

### 6.3 `extract_json(text) -> dict`

从 LLM 响应中提取第一个 JSON 对象，依次尝试三种策略：
1. ` ```json ... ``` ` 代码围栏
2. 整个字符串直接 `json.loads`
3. 从第一个 `{` 开始做括号配对，找到第一个平衡的 `{...}` 块

三种都失败则抛出 `LLMParseError`（`ValueError` 子类）。

### 6.4 `PromptCache`（`cache.py`）

一份 JSON 文件对应一次 LLM 决策，刻意身兼三职（锁定设计决策）：
1. **缓存**：对同一快照重跑回测的某个 Agent 调用成本为 $0
2. **持久化记录**：每个 `Signal` 背后**确切的** prompt + response，用于回放与审计
3. **调试线索**：解析失败时把原始响应留在磁盘上

文件存放于 `.v2_cache/llm/`（已加入 `.gitignore`），以 `(agent, model, prompt)` 的哈希为键。

```python
def prompt_key(agent: str, model: str, system: str, user: str) -> str:
    payload = f"{agent}|{model}|{system}|{user}"
    return hashlib.sha256(payload.encode()).hexdigest()[:24]

class PromptCache:
    def __init__(self, cache_dir: Path | str = DEFAULT_CACHE_DIR) -> None: ...
    def get(self, key: str) -> dict | None: ...   # 损坏的缓存条目视为未命中
    def put(self, key: str, record: dict) -> None: ...  # 自动附加 created_at 时间戳
```

---

## 7. PEAD 模型（`v2/signals/pead.py`）

盈余公告后漂移（Post-Earnings Announcement Drift）——量化模型的代表实现，`QuantModel` 的具体子类，也是回测引擎最初的参考实现。

**原理**：EPS 超预期（BEAT）后做多，低于预期（MISS）后做空，理论依据是市场对盈余惊喜的反应不足，股价会在公告后继续沿惊喜方向漂移数天至数周。

```python
PEADModel(*, earnings_limit: int = 8, signal_window_days: int = 4)
```

### 7.1 `predict(ticker, date, data_client) -> Signal`

1. 获取该股票的"合格事件"列表（见 7.2 节，内部按 ticker 缓存，一次回测中每只股票只请求一次财报历史）
2. **时点过滤**：只保留 `filing_date <= date` 的事件
3. 取其中最新的一次事件；若该事件距 `date` 超过 `signal_window_days`（默认 4 天），返回中性（`0.0`）——即只在"刚得知"的新鲜窗口内触发信号
4. `BEAT -> value=1.0`，`MISS -> value=-1.0`；`reasoning` 与 `metadata`（`eps_surprise`、`source_type`、`report_period`、`filing_date`）随 `Signal` 返回

### 7.2 合格事件筛选（`_qualifying_events`）

对每只 ticker 的原始 `EarningsRecord` 列表做清洗：
1. 丢弃无 `filing_date` 或无 `quarterly` 数据的记录
2. 只保留 `eps_surprise` 为 `"BEAT"` 或 `"MISS"` 的记录
3. **45 天回溯过滤**：若 `(filing_date - report_period).days >= 45`（`_RETROSPECTIVE_CUTOFF_DAYS`），跳过——数据提取器有时会从当期 8-K 中解析出上一季度的对比数据，这类"追溯性"行会被排除
4. **同一 `report_period` 去重**，按来源优先级取最优：

   | source_type | 优先级值（越小越优先） |
   |-------------|----------------------|
   | `8-K` | 0（最早的正式公告） |
   | `10-Q` | 1 |
   | `10-K` | 2 |
   | `20-F` | 3 |
   | 其他 | 99 |

---

## 8. `ALPHA_MODEL_REGISTRY`（`v2/signals/__init__.py`）

```python
ALPHA_MODEL_REGISTRY: dict[str, type[AlphaModel]] = {
    # Quant models
    "pead": PEADModel,
    # LLM investor agents
    "buffett": BuffettAgent,
    "munger": MungerAgent,
    "graham": GrahamAgent,
    "lynch": LynchAgent,
    "druckenmiller": DruckenmillerAgent,
}
```

6 个模型（1 个量化 + 5 个 LLM 人格）。`v2/fund/spec.py` 的 `Fund`（见 [`v2_fund_system.md`](./v2_fund_system.md)）和 `v2/run.py` 的交互式建基金向导都通过这个注册表按名字实例化模型——新增分析师只需在此登记一行即可同时接入回测、策略库 YAML 与建基金 CLI。

**公开导出**：

```python
from v2.signals import (
    AlphaModel, QuantModel, LLMAgent,
    BuffettAgent, MungerAgent, GrahamAgent, LynchAgent, DruckenmillerAgent,
    PEADModel,
    ALPHA_MODEL_REGISTRY,
)
```

---

## 9. 单模型的时点观点 CLI 已并入 `v2.run`

早期版本这里是独立的 `v2/analyze.py`（`poetry run python -m v2.analyze NVDA --agent buffett`）——该文件已在 v2.0.0 删除。同等能力（选一个模型、选一个 as-of 日期、看它对一只股票的时点观点）现在是 `poetry run python -m v2.run` 交互式建基金流程的一部分：建基金时选中的每个策略、每个模型都会对整个股票池跑一遍 `predict()`，并把每一条观点（方向、置信度、理由）展示在"AGENT ANALYSIS"表格里。完整的 CLI 说明见 [`v2_fund_system.md`](./v2_fund_system.md#7-v2runpy--统一-cli)。

---

## 10. `v2/demo/backtest.py` — 演示仪表盘

```bash
poetry run python -m v2.demo.backtest             # 预热缓存 ~20s，可离线运行
poetry run python -m v2.demo.backtest --refresh    # 强制重新拉取数据，重建缓存
```

面向演示场景的前端，底层驱动的是与生产环境相同的 `BacktestEngine`、`PEADModel` 与数据层——这一层只负责固定日期范围、精选 25 只股票池、节奏化的交易重放动画（终端实时仪表盘：运行统计、权益曲线、交易明细带）。

| 常量 | 值 | 说明 |
|------|-----|------|
| `TICKERS` | 25 只精选股票（AAPL、MSFT、NVDA...） | 演示用固定股票池 |
| `START_DATE` | `"2023-07-01"` | — |
| `END_DATE` | `"2026-06-13"`（**固定值，非 `date.today()`**） | 避免演示当天与彩排时缓存键、统计结果不一致 |
| `HOLDING_DAYS` | `5` | — |
| `CAPITAL` | `100_000.0` | — |
| `PER_TRADE` | `10_000.0` | — |
| `EARNINGS_LIMIT` | `40` | 约 3 年的 8-K/10-Q 记录，供 PEAD 回溯 |
| `REPLAY_SECONDS` | `18.0` | 加载 ~2s，总演示时长约 20s |

所有 API 响应均通过 `CachedDataClient` 磁盘缓存：预热后每次重跑（包括正式演示当天）都完全离线，且数字与彩排时完全一致。

---

## 11. 依赖关系

### 11.1 外部 Python 包

| 包 | 用途 | 所在文件 |
|----|------|---------|
| `numpy` | `QuantModel` 数值工具（`_sigmoid`、`_compute_rsi` 等） | `signals/base.py` |
| `pandas` | RSI 等技术指标计算的价格序列输入 | `signals/base.py` |
| `pydantic` | `Signal`、`FundamentalsSnapshot`、`PeriodFundamentals` 等模型 | `models.py`, `features/snapshot.py` |
| `langchain-anthropic` | `AnthropicLLM` 的底层传输 | `llm/client.py` |
| `rich` | demo 的终端富文本输出（spinner、表格、面板） | `demo/backtest.py` |
| `dotenv`（`python-dotenv`） | 加载 `.env` | `demo/backtest.py` |

### 11.2 项目内部依赖

| 模块 | 导入项 | 用途 |
|------|--------|------|
| `v2.data.protocol` | `DataClient` | Alpha 模型 `predict()` 的数据源协议参数 |
| `v2.data` | `CachedDataClient`, `FDClient` | demo 的具体数据源 |
| `v2.models` | `Signal` | Alpha 模型的统一输出类型 |
| `v2.backtesting` | `BacktestEngine` | `demo/backtest.py` 驱动回测 |

> 注：单模型 CLI 相关的 `rich`/`dotenv`/`CachedDataClient` 用法已随 `v2/analyze.py` 的删除转移到 `v2/run.py`（新增 `questionary`、`prompt_toolkit`、`pyyaml` 等依赖），见 [`v2_fund_system.md`](./v2_fund_system.md)。

### 11.3 标准库

| 模块 | 用途 |
|------|------|
| `abc` | `ABC`、`abstractmethod`（`AlphaModel`） |
| `hashlib` | `content_hash`（快照）、`prompt_key`（LLM 缓存）的 SHA-256 |
| `json` | LLM 响应解析、缓存文件读写 |
| `logging` | `LLMAgent` 失败路径的 warning 日志 |
| `os` | 读取 `ANTHROPIC_API_KEY`、`V2_LLM_MODEL` 环境变量 |
| `re` | `extract_json` 的代码围栏正则匹配 |
| `datetime` | PEAD 模型的日期解析与窗口计算 |
| `__future__.annotations` | 延迟类型注解（PEP 563） |

### 11.4 运行环境

- **Python 版本**：3.11+
- **API 密钥**：`FINANCIAL_DATASETS_API_KEY`（数据层）+ `ANTHROPIC_API_KEY`（仅 LLM Agent 需要）
- **缓存目录**：`.v2_cache/data/`（数据层）与 `.v2_cache/llm/`（LLM 决策），均已加入 `.gitignore`
