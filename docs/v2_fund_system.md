# v2 Fund 系统 (Fund / Strategy / Portfolio / Risk / Execution)

源目录: `v2/fund/`、`v2/brokers/`、`v2/portfolio/`、`v2/risk/`、`v2/pipeline/`、`v2/backtesting/fund.py`、`v2/strategies/`、`v2/funds/`、`v2/run.py`

> **v2.0.0 新增模块，v2.0.1 补上"回测整支基金"**。这是继 [`v2_signals_system.md`](./v2_signals_system.md)（Alpha 模型层）之后，v2 在 2026-07 合并的核心组件——把"单个模型形成观点"接到了"一整支基金的一次完整交易周期"，再接到了"这条流水线循环整段历史"。`ROADMAP.md` 中 `run_cycle`、`Fund` 对象、`Strategy`/组合构建、`Broker` 协议、回测引擎均已从 `⬜ 规划中` 升级为 `✅`/`🚧`。

---

## 1. 模块概述

v2 把基金重建为一个持久化、时点正确的系统，镜像真实资管机构的层级结构：

```
FUND      =  按策略切分资金（对净后总仓位做主风控）
STRATEGY  =  一套模型的混合策略（一个 "pod"）
MODEL     =  一个 Alpha 模型 → 一个 Signal（[-1,+1] 信念值 + 论点）
```

一支基金同时运行两类 pod：**主观型（discretionary）** pod 由 LLM 投资人 Agent（Buffett、Munger、Graham、Lynch、Druckenmiller）担纲，判断力就是其"edge"，可以偏多头或做市场中性混合；**系统型（systematic）** pod 由量化模型（PEAD）驱动——模型本身就是策略，没有人格附着。两者实现同一个 `AlphaModel` 接口，接入同一套引擎不需要任何改动。

**一次完整的交易周期（`run_cycle`）**：

```
时点正确数据 → Alpha 模型 → 组合构建（混合） → 风控（硬限额） → 执行（生成订单） → 成交记录
```

| 目录/文件 | 职责 |
|-----------|------|
| `v2/fund/spec.py` | `ModelSpec`、`BlendPolicy`、`StrategySpec`、`FundSpec`（mandate 数据）+ `Fund`（实例化后的活体基金） |
| `v2/brokers/protocol.py` | `Broker` 协议 |
| `v2/brokers/sim.py` | `SimBroker`——回测用的确定性模拟经纪商 |
| `v2/brokers/models.py` | `Position`、`Order`、`Fill` |
| `v2/portfolio/construction.py` | `blend_signals()`——把一组 `Signal` 混合成目标权重 |
| `v2/risk/limits.py` | `apply_limits()`——对目标权重做硬限额裁剪 |
| `v2/pipeline/run_cycle.py` | `run_cycle()`——一次完整周期的唯一代码路径 |
| `v2/pipeline/execution.py` | `build_orders()`——把目标权重与当前持仓做差，生成订单 |
| `v2/pipeline/models.py` | `CycleRecord`、`StrategyRecord`、`TickerSkip`——一次周期的完整可序列化记录 |
| `v2/backtesting/fund.py` | `backtest_fund()`——`run_cycle` 循环整段历史，`FundBacktestResult`/`rebalance_grid()` |
| `v2/strategies/*.yaml` | 策略库（`fundamental-ls`、`deep-value`、`inflections`、`earnings-drift`） |
| `v2/funds/example.yaml` | 示例基金 mandate |
| `v2/run.py` | 统一 CLI：交互式建基金向导 + 非交互式跑一次周期 |

---

## 2. `FundSpec` 体系（`v2/fund/spec.py`）

**核心设计决策**：mandate 是数据（"Loop-2 ground rule"）——一支基金的完整定义是一份可序列化的 YAML/JSON。交互式向导、未来的 chat LLM、策略生成器，发出的都是同一种格式；下游代码不需要关心这份 mandate 是谁写的。

### 2.1 `ModelSpec`

策略里的一个信号模型（LLM Agent 或量化模型）。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `name` | `str` | — | `ALPHA_MODEL_REGISTRY` 的 key，如 `"buffett"` |
| `weight` | `float` | `1.0`（必须 `> 0`） | 混合权重 |
| `params` | `dict[str, Any]` | `{}` | 传给模型构造函数的关键字参数 |

`model_config = ConfigDict(extra="forbid")`——YAML 里打错字段名会在加载时直接报错，而不是在交易时悄悄失效。

### 2.2 `BlendPolicy`

一个策略内部，各模型观点如何合成为一个"sleeve"（分仓）。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `method` | `Literal["conviction_weighted"]` | `"conviction_weighted"` | 混合方法（目前只有一种） |
| `gross_target` | `float`（`> 0`） | `1.0` | 有观点时希望达到的 `sum(\|weight\|)` |
| `market_neutral` | `bool` | `False` | 是否在缩放前做截面去均值：多最受青睐的名字、空最不受青睐的，形成美元中性的 sleeve |

### 2.3 `StrategySpec`（一个 "pod"）

```python
StrategySpec(
    name="deep-value", display_name="Deep Value", weight=1.0,
    models=[ModelSpec(name="graham", weight=2.0), ModelSpec(name="buffett"), ModelSpec(name="munger")],
    blend=BlendPolicy(method="conviction_weighted", gross_target=1.0),
)
```

- `weight`：这个策略在基金里的资金切片，相对其他策略而言（记账时归一化——`2/2` 和 `1/1` 效果相同）。策略库文件里通常保留默认值，切片是"组建基金"这一步的决定，不是策略本身的属性
- `models`：至少 1 个（`min_length=1`）
- `.title`（属性）：`display_name`，缺省时退回到把 `name` 中的连字符替换成空格并首字母大写
- `.model_weights`（属性）：`{model_name: weight}` 字典，直接喂给 `blend_signals()`

### 2.4 `FundSpec`（完整 mandate）

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `str` | 基金名 |
| `universe` | `list[str]`（至少 1 个） | 股票池；`@field_validator` 自动转大写并检查重复 |
| `strategies` | `list[StrategySpec]`（至少 1 个） | `@field_validator` 检查策略名不重复 |
| `risk` | `RiskLimits` | **主风控**——应用在所有策略合并、净仓之后的总账本上（见第 4 节） |
| `capital` | `float`（`> 0`） | 默认 `100_000.0` |
| `rebalance` | `Literal["daily","weekly","monthly"]` | 默认 `"weekly"`。多久跑一次周期——是 mandate 的选择，不是引擎常量：基本面驱动的基金按周调仓，新闻驱动的基金可以按天。回测器（以及未来的调度守护进程）会遵守它，`run_cycle` 本身从不感知这个字段 |
| `benchmark` | `str` | 默认 `"SPY"`。基金拿什么做基准对比；同时也是回测时交易日网格的来源。`@field_validator` 自动转大写 |

全字段 `extra="forbid"`：YAML 里的拼写错误在加载时（而非交易时）就会报错。

```python
def load_spec(path: str | Path) -> FundSpec: ...      # 加载完整 mandate
def load_strategy(path: str | Path) -> StrategySpec: ...  # 加载策略库里的单个策略
```

### 2.5 `Fund`（活体基金）

```python
Fund(spec: FundSpec, models: dict[str, list[AlphaModel]] | None = None)
```

普通类（非 Pydantic）——模型持有状态（LLM 客户端、prompt 缓存、按 ticker 缓存的财报历史），必须**恰好构造一次**，绝不能每个周期都重建，否则缓存无法跨周期存活。

- `self.strategies: list[tuple[StrategySpec, list[AlphaModel]]]`——每个策略连同其"在编"的模型实例列表
- 生产环境不传 `models`：`Fund.__init__` 按 `strategy.models` 中登记的 `name` 从 `ALPHA_MODEL_REGISTRY` 实例化，未登记的模型名立即抛出 `ValueError`
- `models` 覆盖参数仅供测试注入假模型
- 同一个人格出现在两个策略里会得到两个独立实例——这没问题，因为 `PromptCache` 是按 prompt 内容在磁盘上去重的，第二个实例的调用是缓存命中，不会重复付费

---

## 3. `Broker` 协议与 `SimBroker`（`v2/brokers/`）

镜像 `v2/data/protocol.py` 的 `DataClient` 设计：结构化子类型，无需继承。

### 3.1 `Broker` 协议（`protocol.py`）

```python
@runtime_checkable
class Broker(Protocol):
    def positions(self) -> dict[str, Position]: ...
    def cash(self) -> float: ...
    def place_order(self, order: Order) -> Fill: ...
```

**契约**：`place_order` 要么**完整成交**并返回 `Fill`，要么抛出异常——不允许部分成交，不允许静默丢单（静默丢单会让基金账本和经纪商账本失去同步）。刻意**没有** `equity()` 方法：计算权益需要给持仓定价，而只有流水线知道时点正确的价格；经纪商只负责报告持有什么，流水线负责计算它值多少钱。

### 3.2 数据模型（`models.py`）

| 类 | 字段 | 说明 |
|----|------|------|
| `Position` | `ticker`, `shares`（`int`，带符号） | 负数=空头 |
| `Order` | `ticker`, `side`（`"buy"`/`"sell"`）, `quantity`（`>0`）, `price` | 只有买/卖两种动词——持仓是带符号股数，"做空"就是卖过零，"回补"就是买回趋近零；多空标签只是展示层面的事，不影响执行逻辑 |
| `Fill` | `ticker`, `side`, `quantity`, `price` | 实际成交记录 |

### 3.3 `SimBroker`（`sim.py`）——回测用确定性模拟经纪商

```python
SimBroker(cash: float)
```

- 每笔订单**完全按订单的参考价成交**——这种确定性正是意义所在：同样的订单序列，回测永远重放出同样的账本
- 内存态：`{ticker: signed_shares}` + 现金余额；某只股票仓位归零时从字典中删除该键
- 滑点/交易成本是声明中的未来扩展点，届时只需修改 `place_order` 内部，不触及流水线
- **不建模保证金**：现金允许变负且保持可见；在无杠杆 mandate（`gross_target <= 1`）+ 先卖后买的下单顺序 + 向零取整的仓位计算下，纯多头账本一般不会走到这一步，但这里不做任何强制保证

---

## 4. 组合构建（`v2/portfolio/construction.py`）

`blend_signals()` 是基金的"投资组合经理"：把每个模型的 `Signal` 合并成一个目标权重，纯算术、无 I/O，同样的输入永远产生同样的账本。

```python
def blend_signals(
    signals: list[Signal],
    model_weights: dict[str, float],
    gross_target: float,
    market_neutral: bool = False,
) -> BlendResult
```

**v0 策略：conviction-weighted（信念加权）**：

```
conviction_t = sum(w_m * value_mt for 投票中的模型 m) / sum(w_m)
```

- **弃权信号（`metadata.abstained is True`）从分子和分母中都被排除**——"没有意见"绝不能伪装成"意见：中性"；一个非弃权的 `0.0`（例如 PEAD 不在触发窗口内）是一次真实的中性投票，会稀释结果
- **`market_neutral=True`**：缩放前先做截面去均值——多最受青睐的名字，空最不受青睐的，sleeve 净额为零美元；所有信念一致时去均值后变成一本"平"的账
- **归一化到 `gross_target`**：`weight_t = (去均值后的)conviction_t / sum(|conviction|) * gross_target`；`sum(|conviction|)` 低于 `1e-9` 时（而非严格等于 0——去均值相同的信念会残留 ~1e-16 的浮点噪声）直接输出全零权重，避免把噪声当信号放大成满仓
- **已知的、刻意接受的缺陷**：截面归一化忽略了**绝对**信念强度——一个孤零零的弱观点也会拿到满额 gross target，这一步交给风控阶段去裁剪（"信念负责提案，风控负责处置"）；`evaluate()` 能够度量信念强度后，加一道最低信念阈值是显而易见的下一步

`BlendResult`：`convictions`（缩放前的混合观点）+ `weights`（目标权重，`sum(|weights|) <= gross_target`）。

---

## 5. 风控硬限额（`v2/risk/limits.py`）

"信念负责提案，风控负责处置"：组合构建给出目标权重的提案，这一步按基金的硬限额裁剪——全部是确定性算术，LLM 对账本的影响到 `Signal` 为止，任何裁剪都不可协商。

### 5.1 `RiskLimits`（写在 `FundSpec.risk` 里）

| 字段 | 类型 | 说明 |
|------|------|------|
| `max_position_pct` | `float`（`0 < x <= 1.0`） | 单只股票 `\|weight\|` 上限（占权益的比例） |
| `max_gross_exposure` | `float`（`> 0`） | 全账本 `sum(\|weight\|)` 上限（`1.0` = 不加杠杆） |

### 5.2 `apply_limits(weights, limits) -> RiskResult`

```
1. 逐票裁剪：任何 |weight| 超过 max_position_pct 就裁剪到上限（保留符号），
   每次裁剪记一条 ClampEvent
2. 总额裁剪：若裁剪后 sum(|weight|) 仍超过 max_gross_exposure，
   对全部权重按比例整体缩放
```

**顺序很重要，且这个组合具有幂等性**：第二步只会整体缩小权重，不可能重新触发第一步的单票上限。

**关键设计决策**：被裁掉的敞口**不会重新分配给其他名字**，而是留在现金里。如果重新分配，风控阶段就有可能**增加**某些仓位——这违背了它"只做减法"的职责。

`ClampEvent`：`limit`（`"max_position_pct"` 或 `"max_gross_exposure"`）、`ticker`（组合级裁剪时为 `None`）、`before`、`after`——每一次裁剪都被记录下来，保证可解释。

---

## 6. `run_cycle` 流水线（`v2/pipeline/`）

**基金的心跳**：回测就是在历史上循环调用 `run_cycle` + `SimBroker`；纸面交易是同一个循环接实时时钟 + `PaperBroker`（规划中）；实盘是同一个循环接真实经纪商。变的只是时钟和经纪商，流水线代码永远不变。

`run_cycle` 是流水线里**唯一不纯的部分**——只有它会和数据客户端、经纪商打交道。它委托的每个阶段（`blend_signals`、`apply_limits`、`build_orders`）都是纯函数。**确定性**：给定同样的 spec、日期、经纪商状态、数据响应，返回的 `CycleRecord` 逐字节相同——唯一的例外是 LLM 缓存为冷启动时，Agent 第一次真实调用不确定；一旦 prompt 缓存写入，之后的每次重放都是精确的。

### 6.1 `run_cycle(fund, as_of, broker, data_client) -> CycleRecord`

执行流程：

1. `broker.positions()` 取得当前持仓；`_mark_prices()` 给"股票池 ∪ 持仓"里的每个 ticker 定最新收盘价（见 6.2 节）
2. `equity_before = 现金 + Σ(持仓股数 × 定价)`；若 `<= 0` 立即抛出 `ValueError`（无法对一本非正权益的账本做仓位计算）
3. `tradeable = 股票池中成功定价的 ticker`
4. **逐策略**：该策略的每个在编模型对每个可交易 ticker 调用 `predict()`，收集全部 `Signal`；调用 `blend_signals()` 得到该策略的 sleeve；按 `strategy.weight / 全部策略权重之和` 计算资金切片，累加进 `netted`（跨策略净仓）
5. `apply_limits(netted, spec.risk)` 做主风控裁剪
6. `build_orders()` 生成订单，逐一提交给 `broker.place_order()` 拿到 `Fill`
7. 重新查询经纪商持仓与现金，计算 `nav = 现金 + Σ(持仓股数 × 定价)`
8. 组装并返回完整的 `CycleRecord`

**刻意选择的行为**：
- **目标权重是"期望账本"的完整声明**：如果所有分析师都弃权或给出中性观点，目标权重全为零，基金平仓归零——`CycleRecord` 上每条信号都会标出 `abstained`，供外层调度（如未来的"Day-5 daemon"）决定是否跳过这一次 tick 而不是真的清仓；这个"是否跳过"的判断刻意放在流水线之外
- **股票池里一个既无价格、又不在持仓中的 ticker 会被跳过**（记录原因，其分析师从不会被调用）——退市/停牌属于历史数据中的正常情况；但一个**已持仓**却查不到价格的 ticker 会直接 `raise`——一支基金如果连自己的账本都定不了价，这是基础设施故障，其 NAV 会是谎言

### 6.2 `_mark_prices()`——定价与跳过逻辑（私有）

```python
start = as_of - 7 天（_MARK_LOOKBACK_DAYS）
```

对每个 ticker 取 `[start, as_of]` 区间内最后一根收盘价（覆盖周末、假期集群、短暂停牌）：
- 有价：记入 `marks`
- 无价 + 不在持仓：记入 `skipped`（`TickerSkip(ticker, reason)`），后续不会为它调用任何模型
- 无价 + 在持仓：`raise ValueError`（见上）

### 6.3 `build_orders()`——执行（`v2/pipeline/execution.py`）

```python
def build_orders(target_weights, positions, marks, equity) -> list[Order]
```

- 目标权重是"期望账本"的完整声明：任何持仓中存在、但目标权重里没提到的 ticker，隐含目标为零，平仓单由此自然产生，不需要单独的分支逻辑
- **仓位计算**：`target_shares = int(weight * equity / mark)`——向零取整，永不超出目标；不足一股的零头留在现金里，下个周期重新评估；差额为 0 的不生成订单
- **顺序**：先出全部卖单，再出全部买单，组内按字母序——确定性输出，且卖单腾出的现金可以在**同一个周期内**支付买单
- 在 `marks` 里发生 `KeyError` 说明上游有 bug（`run_cycle` 在调用这里之前已经给每个可交易/持仓的 ticker 都定过价）——任其抛出，不做防御性兜底

### 6.4 `CycleRecord` 与 `StrategyRecord`（`v2/pipeline/models.py`）

`CycleRecord` 是一次周期端到端的完整序列化真相——分析师看到了什么、说了什么、观点如何变成权重、风控裁剪了什么、下了什么单成交了什么、周期结束后账本什么样。`model_dump_json()` 可以完整往返；未来的 `fund why AAPL` 命令只需要读这一份记录就能回答问题。

| 字段 | 类型 | 说明 |
|------|------|------|
| `fund` | `str` | 基金名 |
| `as_of` | `str` | 本次周期的时点日期 |
| `spec` | `FundSpec` | 自包含的审计副本（mandate 本身也存进记录里） |
| `marks` | `dict[str, float]` | 用于定价和 NAV 计算的收盘价 |
| `skipped` | `list[TickerSkip]` | 本周期无法交易的股票池成员及原因 |
| `strategies` | `list[StrategyRecord]` | 每个 sleeve（含每条论点） |
| `target_weights` | `dict[str, float]` | 风控前的净仓（跨策略汇总） |
| `clamps` | `list[ClampEvent]` | 风控触发记录 |
| `final_weights` | `dict[str, float]` | 风控后的目标权重 |
| `equity_before` / `cash_before` | `float` | 周期开始时的权益/现金 |
| `orders` / `fills` | `list[Order]` / `list[Fill]` | 本周期下的单与实际成交 |
| `positions` | `dict[str, int]` | 成交后的带符号持仓股数 |
| `cash` / `nav` | `float` | 周期结束时的现金/净值 |

`StrategyRecord`：`name`、`slice`（归一化后的资金切片）、`signals`（该策略下全部模型 × 全部可交易 ticker 的 `Signal`）、`convictions`（混合前的观点）、`weights`（该策略自己的 sleeve，净仓之前）。

---

## 7. `backtest_fund` — 整支基金的历史回测（`v2/backtesting/fund.py`）

**`run_cycle` 文档字符串许下的承诺**："a backtest is run_cycle in a loop over history with a SimBroker"——这个模块就是那个循环。这里不重新实现任何流水线逻辑：每一个 tick 都是对一个持久化 `SimBroker` 真实调用一次 `run_cycle`，所以任何对单次周期成立的性质（时点正确数据、fail-loud 定价、净仓上的主风控）对回测中的每一个 tick 都天然成立。这是 `engine.py`（`BacktestEngine`，单个 Alpha 模型的回测，见 [`v2_backtesting_system.md`](./v2_backtesting_system.md)）在基金层面的对应物——`BacktestEngine` 用固定机制模拟单个模型的观点，`backtest_fund` 跑的是整个"店"。

> **ROADMAP 补充**：这次合并（`v2.0.1`）把 `run_cycle`/回测收敛的路线图项标为已完成——`backtest_fund` 就是 `run_cycle` 循环历史的那条统一路径；`BacktestEngine.run_alpha()`（`v2/backtesting/engine.py`）作为单模型研究用的旧版专用工具保留下来，两者并存，服务不同的用例。

### 7.1 `backtest_fund(fund, start, end, data_client, *, on_cycle=None) -> FundBacktestResult`

```python
bars = data_client.get_prices(spec.benchmark, start, end)
closes = {日期: 收盘价, ...}          # 限定在 [start, end] 内
grid = rebalance_grid(sorted(closes), spec.rebalance)   # 按调仓频率抽取交易日
```

- **交易日网格来自 mandate 的基准的真实 K 线**，而不是自己实现交易所日历——节假日、缩短的交易周自然就被排除了
- **fail-loud**：基准在 `[start, end]` 窗口内一根 K 线都没有就直接 `raise ValueError`——没有交易网格的回测是基础设施问题，不是"空结果"
- 用**同一个持久化 `SimBroker`** 依次对 `grid` 里的每个日期调用一次 `run_cycle`——持仓和现金跨 tick 延续，基金是在"调仓"而不是每次都从零开始
- `on_cycle(i, n, record)`：每个 tick 结束后触发一次的回调，供进度 UI 使用（见 8.4 节的 `_BacktestBoard`）
- 逐 tick 收集 `nav`（基金净值）与 `benchmark_nav`（基准按同样起始资本换算后的净值，`capital * close_t / close_grid[0]`）

### 7.2 `rebalance_grid(days, cadence) -> list[str]`

从已排序的交易日列表中挑出调仓日：

| `cadence` | 规则 |
|-----------|------|
| `"daily"` | 每个交易日都算 |
| `"weekly"` | 每个 ISO 周的最后一个交易日 |
| `"monthly"` | 每个自然月的最后一个交易日 |

实现上按 `(年, 周)` 或 `(年, 月)` 分组，由于输入已排序，同一组内后写入的日期会覆盖前一个，天然取到"最后一个交易日"。

### 7.3 `FundBacktestMetrics` 与 `FundBacktestResult`

| `FundBacktestMetrics` 字段 | 说明 |
|---------------------------|------|
| `total_return_pct` / `annualized_return_pct` | 总收益率 / 年化收益率 |
| `sharpe_ratio` | 按每期（而非每日）收益率计算，用 `_PERIODS_PER_YEAR[cadence]`（`daily=252`、`weekly=52`、`monthly=12`）年化；样本数 `<=1` 或标准差为 0 时取 `0.0` |
| `max_drawdown_pct` | 对 `[起始资本] + nav` 曲线做历史峰值追踪，逻辑与 `BacktestEngine._compute_metrics` 一致 |
| `benchmark_return_pct` | 基准在同一窗口的总收益率 |
| `excess_return_pct` | 基金总收益率减基准总收益率 |
| `n_cycles` | 回测跑了多少个周期 |
| `n_orders` | 全部周期加总的下单数 |

`FundBacktestResult`（顶层结果，`model_dump_json()` 可完整往返——"收据文件"）：`fund`、`start`/`end`（实际交易的第一/最后一个网格日期）、`rebalance`、`benchmark`、`capital`、`dates`、`nav`（每个 tick 后的基金净值）、`benchmark_nav`、`metrics`、`records`（**每一个 tick 的完整 `CycleRecord`**——因此这份结果里包含了每次调仓背后的每一条论点、每一次风控裁剪、每一笔订单与成交）。

---

## 8. `v2/run.py` — 统一 CLI

早期分立的 `v2/analyze.py`（单模型问一只股票）在 v2.0.0 合并进唯一入口 `v2.run`；v2.0.1 又把"回测一整支基金"也并了进来（`v2/demo/` 的独立演示仪表盘随之被删除，见 [`v2_signals_system.md`](./v2_signals_system.md)）。三种用法背后是**同一套引擎**：交互式向导只是"组装一份 `FundSpec`"的瘦客户端——同样机器可读的 YAML，人可以点选生成，未来 chat LLM 或策略生成器也能生成，引擎只认一种格式。

### 8.1 非交互模式——跑一次周期或跑一次回测

```bash
# 跑一次周期
poetry run python -m v2.run v2/funds/example.yaml --date 2025-06-03

# 回测这份 mandate：从 --start 到 --date，按 mandate 的调仓频率跑 run_cycle
poetry run python -m v2.run v2/funds/example.yaml --backtest --start 2024-01-01 --date 2025-06-03

# 指定 LLM 投资人 Agent 使用的模型
poetry run python -m v2.run v2/funds/example.yaml --model claude-opus-5
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `mandate`（位置参数，可省略） | — | 基金 mandate YAML 路径；省略则进入交互式向导 |
| `--date` | 今日 | as-of 日期（单周期模式）；模型只能看到这一天之前已公开申报的数据 |
| `--backtest` | `False`（flag） | 不跑单次周期，改为回测：从 `--start` 到 `--date` 按 mandate 的 `rebalance` 频率逐一调用 `run_cycle` |
| `--start` | `--date` 往前 `_BACKTEST_WEEKS`（78 周，约 18 个月） | 回测起始日期，仅 `--backtest` 时有效 |
| `--model` | — | LLM 投资人 Agent 使用的模型（如 `claude-opus-5`）；设置后写入环境变量 `V2_LLM_MODEL`，被 `AnthropicLLM` 读取为覆盖值；量化模型忽略此参数 |
| `--out` | — | 额外把结果 JSON 写入这个文件 |

单周期行为：`load_spec()` → `Fund(spec)` → `SimBroker(cash=spec.capital)` → `CachedDataClient(FDClient())` → `run_cycle()`。回测行为：同样加载 `Fund` 后调用 `backtest_fund()`（见第 7 节）。**完整结果 JSON（`CycleRecord` 或 `FundBacktestResult`）打印到 stdout**（可管道给其他工具），进度 spinner 和人类可读摘要打印到 stderr——两者分离是为了让 stdout 保持纯净的 JSON。

### 8.2 交互模式——两件事，一支基金（无参数）

```bash
poetry run python -m v2.run
```

打开后**首先**问一次"投资人 Agent 用哪个模型推理"（`_pick_model`）：

| 选项 | 模型 ID | 说明 |
|------|---------|------|
| Opus 5（默认） | `claude-opus-5` | 能力最强 |
| Sonnet 5 | `claude-sonnet-5` | 均衡档，非最贵档 |

选定后写入 `V2_LLM_MODEL` 环境变量，之后无论是名册预热、正式建基金还是回测，全部 `LLMAgent` 实例统一走这一个模型（量化模型 PEAD 不受影响）。选择模型这一步按 `Esc`/`Ctrl-C` 会直接退出整个交互流程。

再进入主菜单，二选一（"production 与 research lab 并列"，`VISION.md` 的说法）：

- **Build a fund**——走建基金向导，见 8.3 节
- **Backtest a fund**——从已保存的基金里选一个回测，见 8.4 节

`Esc` 在主菜单处直接退出；在某个具体流程里按 `Esc` 回到主菜单。

### 8.3 Build a fund——建基金向导

一个小型状态机，**五步**（每步可按 `Esc` 回退一步，比 v2.0.0 多了第 5 步"调仓频率"）：

1. **`_step_name`**：基金名（自动转小写、空格转连字符）
2. **`_step_tickers`**：股票池（逗号或空格分隔，自动去重、转大写）
3. **`_step_strategies`**：从 `v2/strategies/*.yaml` 策略库里勾选（含一个"Build your own"选项，可以逐个手选模型，哪怕只选 PEAD 一个）
4. **`_step_capital`**：起始资金
5. **`_step_cadence`**：调仓频率，`daily`/`weekly`/`monthly` 三选一，默认 `weekly`

五步结束后：按等额资金切片组装 `FundSpec`（`risk` 用固定的 `DEFAULT_RISK = {max_position_pct: 0.25, max_gross_exposure: 1.0}`；`rebalance` 取第 5 步的选择），写入 `v2/funds/{name}.yaml`（该目录已加入 `.gitignore`，只保留 `example.yaml` 被跟踪），询问是否立即跑第一次周期。

**跑第一次周期时的"名册"UI（`_run_with_roster`）**：先用线程池并发"预热"——每个 Agent 对股票池里的每只票调用一次 `predict()`（只是为了把磁盘缓存焐热，异常被吞掉，不作为真相来源），终端上以 v1 风格显示每个 Agent 的实时状态（`⋯ 排队中` → `⋯ [TICKER] 分析中` → `✓ 完成`）；预热结束后，用一个**全新的** `Fund` 实例和 `SimBroker` 正式跑一次 `run_cycle`——这次读到的全是刚焐热的缓存，瞬间完成，且是唯一的真相来源（fail-loud 错误在这里才会真正出现，不在预热阶段）。

`_print_cycle()` 把 `CycleRecord` 渲染成 v1 风格的分节报告，按 `_REVEAL_DELAY`（0.7 秒）逐节延迟展示：
1. **AGENT ANALYSIS**：每个策略一张表（ticker、Agent、方向 BULLISH/BEARISH/NEUTRAL/ABSTAIN、置信度、理由），标注该策略是主观型还是系统型、是否市场中性、占资金比例
2. **RISK LIMITS**（如有裁剪）：范围、请求值、放行值、触发的限额类型
3. **ORDERS**：买/卖、数量、ticker、价格
4. **PORTFOLIO SUMMARY**：逐 ticker 多空方向、股数、市值、权重，外加 NAV、现金、总敞口（gross）、净敞口（net）

### 8.4 Backtest a fund——研究实验室

`_backtest_saved_fund`：从 `v2/funds/*.yaml` 里挑一支已保存的基金，接着走一个四步状态机（复用同一套 `Esc` 回退状态机模式）：

1. **`_step_pick_fund`**：从已保存的基金 mandate 里选一个（用 `_fund_label` 显示名称、股票池前 4 只、调仓频率）
2. **`_step_tickers`**：预填该基金保存的股票池，本次运行可以临时增删（不修改原 YAML）
3. **`_step_backtest_start`**：回测起始日期，默认今日往前 78 周
4. **`_step_backtest_end`**：回测截止日期，需晚于起始日期

四步完成后，用 `spec.model_copy(update={"universe": ...})` 生成一份**不改动原 YAML 的临时 spec**（浅拷贝、不重新校验），跑 `_run_backtest`。

**`_run_backtest` 的两阶段预热 + 回放**：

1. **`_warm_market_data`**：先并发预取 `run_cycle` 和快照构建将会用到的全部请求（按 `(ticker, 日期分块)` 切任务而非按 ticker 切——这样即使只有一只股票，78 周的历史也能被多个线程同时抓取，而不是全压在一个线程上），用 `rich.Progress` 条显示"Loading market data · N stocks × M cycles"
2. **`_warm_agents`**：名册 UI 在整个历史窗口上重放一遍（每个 Agent × 每个调仓日 × 每只股票各 `predict()` 一次），把 prompt 缓存焐热；这一阶段**没有 dwell 延迟**——未变化的快照是瞬间命中缓存，真正发生的新 LLM 调用自然会让名册的节奏慢下来，不需要人为节流
3. 用**全新的** `Fund` + `_BacktestBoard`（见 8.5 节）跑真正的 `backtest_fund(..., on_cycle=board.tick)`——这是唯一的真相来源

跑完后：`_print_backtest` 打印结果表格（见 8.6 节），并把完整的 `FundBacktestResult` JSON 存到 `v2/funds/{name}-backtest-{YYYY-MM-DD-HHMMSS}.json`（按时间戳命名，重跑同一支基金不会覆盖旧的"收据"）。

### 8.5 `_BacktestBoard`——回测时的实时权益曲线

回放过程中的 Live 仪表盘：顶部四格统计（组合市值、基金收益率、基准收益率、最大回撤），下方是**双线 Unicode 折线图**（`_render_chart`）——基金曲线与基准曲线画在同一组坐标轴上，基金线后画、颜色随盈亏变绿/变红（相对起始资本），基准线固定青色；左侧一列价格刻度（`_money`，超过一万自动显示为 `$xxxk`）。每个 tick 之间有 `_CYCLE_DWELL`（0.08 秒）的最小停留，让曲线"画出来"的过程肉眼可见而不是一闪而过。

### 8.6 `_print_backtest`——最终结果表

一张表：Total Return、Annualized、Sharpe（`>1` 绿色、`>0` 黄色、其余红色）、Max Drawdown、`{benchmark} Return`、Excess（基金减基准，正绿负红）。标题栏附带日期范围、调仓频率、周期数、订单总数。

### 8.7 版本号

`v2/run.py` 顶部硬编码 `VERSION = "2.0.0"`（注释注明需要与 `pyproject.toml` 保持同步；`pyproject.toml` 已升级到 `2.0.1`，这处硬编码尚未跟着更新）——这与合并把项目版本号从日历式（`2026.7.10`）改为语义化版本一致。

---

## 9. 策略库与示例基金（`v2/strategies/`、`v2/funds/`）

策略库是"不需要写代码"的贡献方式：把已有模型打包成一个带混合策略的 YAML，扔进 `v2/strategies/`，建基金向导会自动识别。当前 4 个：

| 文件 | 类型 | 构成 | 特点 |
|------|------|------|------|
| `fundamental-ls.yaml` | 主观型 | 全部 5 位 LLM 人格 | 旗舰主观 pod；`market_neutral: true`，全员给股票池排名，多最受青睐、空最不受青睐 |
| `deep-value.yaml` | 主观型 | Graham（权重 2.0）+ Buffett + Munger | Graham 主导（双倍权重），Buffett/Munger 把关质量 |
| `inflections.yaml` | 主观型 | Druckenmiller + Lynch | 专找基本面变化率的拐点 |
| `earnings-drift.yaml` | 系统型 | PEAD 单模型 | 模型本身就是策略，没有人格附着 |

`v2/funds/example.yaml`：`deep-value`（60% 切片）+ `earnings-drift`（40% 切片）的组合示例，`risk: {max_position_pct: 0.25, max_gross_exposure: 1.0}`，`capital: 100000`，`universe: [AAPL, MSFT, NVDA, GOOGL, AMZN]`。该目录下只有 `example.yaml` 被 git 跟踪，其余（向导生成的用户基金）已加入 `.gitignore`。

主观型/系统型的判定是**派生的，从不显式声明**（`v2/run.py::_strategy_kind`）：一个策略如果全部模型都是 `LLMAgent` 子类，就是主观型；只要混入一个量化模型，就是系统型。

---

## 10. 依赖关系

### 9.1 外部 Python 包

| 包 | 用途 | 所在文件 |
|----|------|---------|
| `pyyaml` | 解析 `FundSpec`/`StrategySpec` YAML | `fund/spec.py` |
| `pydantic` | `FundSpec`、`RiskLimits`、`CycleRecord` 等全部数据模型 | `fund/spec.py`, `risk/limits.py`, `pipeline/models.py`, `brokers/models.py`, `portfolio/construction.py` |
| `questionary` | 交互式向导的文本输入/勾选框 | `run.py` |
| `prompt_toolkit` | 自定义 `Esc` 回退键绑定（`questionary` 的传递依赖） | `run.py` |
| `rich` | 终端富文本（Live 名册、表格、面板） | `run.py` |
| `dotenv`（`python-dotenv`） | 加载 `.env` | `run.py` |

### 9.2 项目内部依赖

| 模块 | 导入项 | 用途 |
|------|--------|------|
| `v2.signals` | `ALPHA_MODEL_REGISTRY`, `LLMAgent` | `Fund` 按名字实例化模型；`run.py` 判断策略类型 |
| `v2.data` | `CachedDataClient`, `FDClient` | `run_cycle` 与 `run.py` 的具体数据源 |
| `v2.models` | `Signal` | `blend_signals`、`CycleRecord` 的信号类型 |

### 9.3 运行环境

- **Python 版本**：3.11+
- **API 密钥**：`FINANCIAL_DATASETS_API_KEY`（数据层）+ `ANTHROPIC_API_KEY`（LLM Agent 策略）
- **缓存目录**：`.v2_cache/data/`、`.v2_cache/llm/`（均已 gitignore）
- **用户基金目录**：`v2/funds/*.yaml` 已 gitignore（仅 `example.yaml` 例外）
