# v2 数据层 (v2 Data Layer)

源目录: `v2/data/`

---

## 1. 模块概述

v2 数据层为 Financial Datasets API 提供类型安全的 HTTP 客户端，是 v2 流水线的数据获取基础。该层由四个文件组成：

| 文件 | 职责 |
|------|------|
| `client.py` | `FDClient` 类——封装 HTTP 通信、重试逻辑和数据反序列化 |
| `models.py` | 所有 API 响应的 Pydantic 数据模型 |
| `protocol.py` | `DataClient` 协议——定义数据提供者的抽象接口 |
| `__init__.py` | 公开导出所有模型类、`FDClient` 和 `DataClient` |

**设计原则**:

- **无状态客户端**: 不含内存缓存，每次调用直接请求 API，由调用方决定是否缓存
- **不抛异常**: 所有方法在失败时返回空列表或 `None`，从不引发异常
- **类型安全**: 所有响应均通过 Pydantic 模型解析，字段类型明确
- **向前兼容**: 所有模型使用 `extra="ignore"`，API 新增字段不会导致解析失败
- **结构化子类型**: `DataClient` 是 `@runtime_checkable` 协议，任何实现了相应方法的类（无需继承）均可作为数据提供者使用

**公开导出** (`__init__.py`):

```python
from v2.data import (
    FDClient,
    DataClient,
    Price,
    FinancialMetrics,
    InsiderTrade,
    CompanyNews,
    CompanyFacts,
    Earnings,
    EarningsData,
    EarningsRecord,
    Filing,
)
```

---

## 2. FDClient 方法列表

### 2.1 构造与上下文管理

#### `FDClient.__init__(api_key=None, timeout=30.0)`

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `api_key` | `str \| None` | `None` | API 密钥；为 `None` 时自动读取环境变量 `FINANCIAL_DATASETS_API_KEY` |
| `timeout` | `float` | `30.0` | 单次 HTTP 请求超时秒数 |

内部使用 `requests.Session` 保持 HTTP 连接，并将 `X-API-Key` 写入 Session 级别的请求头，所有请求共享该认证头。

**上下文管理器用法**（推荐）:

```python
with FDClient() as fd:
    prices = fd.get_prices("AAPL", "2024-01-01", "2024-12-31")
# Session 在退出时自动关闭
```

也可手动关闭：`fd.close()`

---

### 2.2 公开方法汇总

| 方法 | 签名 | 返回类型 |
|------|------|---------|
| `get_prices` | `(ticker, start_date, end_date, interval="day", interval_multiplier=1)` | `list[Price]` |
| `get_financial_metrics` | `(ticker, end_date, period="ttm", limit=10)` | `list[FinancialMetrics]` |
| `get_news` | `(ticker, end_date, start_date=None, limit=1000)` | `list[CompanyNews]` |
| `get_insider_trades` | `(ticker, end_date, start_date=None, limit=1000)` | `list[InsiderTrade]` |
| `get_company_facts` | `(ticker)` | `CompanyFacts \| None` |
| `get_earnings` | `(ticker)` | `Earnings \| None` |
| `get_earnings_history` | `(ticker, limit=12)` | `list[EarningsRecord]` |
| `get_market_cap` | `(ticker, end_date)` | `float \| None` |

---

### 2.3 各方法详细说明

#### `get_prices(ticker, start_date, end_date, interval="day", interval_multiplier=1) -> list[Price]`

获取 OHLCV 价格 K 线数据，对应 API 端点 `/prices/`。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `ticker` | `str` | — | 股票代码（如 `"AAPL"`） |
| `start_date` | `str` | — | 起始日期（`YYYY-MM-DD`） |
| `end_date` | `str` | — | 截止日期（`YYYY-MM-DD`） |
| `interval` | `str` | `"day"` | 时间粒度（如 `"day"`、`"minute"`） |
| `interval_multiplier` | `int` | `1` | 时间粒度倍数（如 `5` 配合 `"minute"` 表示 5 分钟 K 线） |

---

#### `get_financial_metrics(ticker, end_date, period="ttm", limit=10) -> list[FinancialMetrics]`

获取截至 `end_date` 的财务指标，对应 API 端点 `/financial-metrics/`，按 `report_period_lte` 过滤。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `ticker` | `str` | — | 股票代码 |
| `end_date` | `str` | — | 数据截止日期 |
| `period` | `str` | `"ttm"` | 报告期类型（`"ttm"` 滚动12月、`"annual"` 年报、`"quarterly"` 季报） |
| `limit` | `int` | `10` | 最多返回条数 |

---

#### `get_news(ticker, end_date, start_date=None, limit=1000) -> list[CompanyNews]`

获取公司新闻，对应 API 端点 `/news/`。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `ticker` | `str` | — | 股票代码 |
| `end_date` | `str` | — | 新闻截止日期 |
| `start_date` | `str \| None` | `None` | 新闻起始日期（可选） |
| `limit` | `int` | `1000` | 最多返回条数 |

---

#### `get_insider_trades(ticker, end_date, start_date=None, limit=1000) -> list[InsiderTrade]`

获取内部人士交易记录，对应 API 端点 `/insider-trades/`。使用 `filing_date_lte` 和 `filing_date_gte` 过滤日期范围。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `ticker` | `str` | — | 股票代码 |
| `end_date` | `str` | — | 申报日期上限 |
| `start_date` | `str \| None` | `None` | 申报日期下限（可选） |
| `limit` | `int` | `1000` | 最多返回条数 |

---

#### `get_company_facts(ticker) -> CompanyFacts | None`

获取公司元数据，对应 API 端点 `/company/facts/`。返回单条记录，失败时返回 `None`。

---

#### `get_earnings(ticker) -> Earnings | None`

获取该股票的最新财报数据（单记录模式），对应 API 端点 `/earnings/`。取响应列表的第一条记录。失败或无数据时返回 `None`。

---

#### `get_earnings_history(ticker, limit=12) -> list[EarningsRecord]`

获取历史财报申报记录列表，对应 API 端点 `/earnings/`（扁平历史模式）。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `ticker` | `str` | — | 股票代码 |
| `limit` | `int` | `12` | 最多返回记录数 |

**注意**: 同一 `report_period` 可能出现多条记录，各自对应不同的 `source_type`（如 `8-K`、`10-Q`、`10-K`、`20-F`）。

---

#### `get_market_cap(ticker, end_date) -> float | None`

便捷方法，返回市值。内部按以下优先级获取：

1. 调用 `get_company_facts(ticker)`，若 `facts.market_cap` 不为 `None` 则直接返回
2. 调用 `get_financial_metrics(ticker, end_date, limit=1)`，返回 `metrics[0].market_cap`
3. 两者均失败则返回 `None`

---

## 3. 速率限制与错误处理

### 3.1 `_request` 方法行为

所有 HTTP 请求均通过内部 `_request` 方法发出，该方法实现了统一的重试与错误处理策略：

| 场景 | 行为 |
|------|------|
| HTTP 429（被限速），且重试次数未耗尽 | 按预设延迟等待后重试 |
| HTTP 429，3 次重试全部耗尽 | 记录 warning 日志，返回 `None` |
| HTTP 4xx / 5xx（非 429） | 立即记录 warning 日志，返回 `None`，不重试 |
| 网络异常（`requests.RequestException`） | 立即记录 warning 日志，返回 `None`，不重试 |
| 成功（2xx） | 返回 `requests.Response` 对象 |

**重试延迟序列**: `(5s, 15s, 30s)`，最多重试 3 次。

**重要**: `_request` 方法永不抛出异常，所有公开方法在底层返回 `None` 时均返回空列表或 `None`。

### 3.2 日志记录

客户端使用 Python 标准 `logging` 模块（logger 名称为 `v2.data.client`）：

- 429 重试：`INFO` 级别，记录延迟时间和重试次数
- 重试耗尽：`WARNING` 级别
- HTTP 错误（非 429）：`WARNING` 级别，记录状态码
- 网络异常：`WARNING` 级别，记录异常信息

---

## 4. 数据模型

所有模型定义于 `v2/data/models.py`，均继承自 `pydantic.BaseModel`，并统一设置 `model_config = {"extra": "ignore"}`。

### 4.1 Price

来源：`/prices/` 端点，表示单根 OHLCV K 线。

| 字段 | 类型 | 可空 | 说明 |
|------|------|------|------|
| `open` | `float` | 否 | 开盘价 |
| `close` | `float` | 否 | 收盘价 |
| `high` | `float` | 否 | 最高价 |
| `low` | `float` | 否 | 最低价 |
| `volume` | `int` | 否 | 成交量 |
| `time` | `str` | 否 | 时间戳字符串 |

---

### 4.2 FinancialMetrics

来源：`/financial-metrics/` 端点。`ticker`、`report_period`、`period` 保证非空，其余字段均可为 `None`（API 后端将 NaN/Inf 清洗为 null）。

**标识字段**（非空）:

| 字段 | 类型 | 说明 |
|------|------|------|
| `ticker` | `str` | 股票代码 |
| `report_period` | `str` | 报告期（如 `"2024-12-31"`） |
| `period` | `str` | 期间类型（`"ttm"`、`"annual"`、`"quarterly"`） |
| `currency` | `str \| None` | 货币代码 |

**估值指标**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `market_cap` | `float \| None` | 市值 |
| `enterprise_value` | `float \| None` | 企业价值 |
| `price_to_earnings_ratio` | `float \| None` | 市盈率（P/E） |
| `price_to_book_ratio` | `float \| None` | 市净率（P/B） |
| `price_to_sales_ratio` | `float \| None` | 市销率（P/S） |
| `enterprise_value_to_ebitda_ratio` | `float \| None` | EV/EBITDA |
| `enterprise_value_to_revenue_ratio` | `float \| None` | EV/Revenue |
| `free_cash_flow_yield` | `float \| None` | 自由现金流收益率 |
| `peg_ratio` | `float \| None` | PEG 比率 |

**盈利能力指标**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `gross_margin` | `float \| None` | 毛利率 |
| `operating_margin` | `float \| None` | 营业利润率 |
| `net_margin` | `float \| None` | 净利润率 |
| `return_on_equity` | `float \| None` | 净资产收益率（ROE） |
| `return_on_assets` | `float \| None` | 总资产收益率（ROA） |
| `return_on_invested_capital` | `float \| None` | 投入资本回报率（ROIC） |

**运营效率指标**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `asset_turnover` | `float \| None` | 总资产周转率 |
| `inventory_turnover` | `float \| None` | 存货周转率 |
| `receivables_turnover` | `float \| None` | 应收账款周转率 |
| `days_sales_outstanding` | `float \| None` | 应收账款回收天数（DSO） |
| `operating_cycle` | `float \| None` | 营业周期（天） |
| `working_capital_turnover` | `float \| None` | 营运资本周转率 |

**流动性指标**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `current_ratio` | `float \| None` | 流动比率 |
| `quick_ratio` | `float \| None` | 速动比率 |
| `cash_ratio` | `float \| None` | 现金比率 |
| `operating_cash_flow_ratio` | `float \| None` | 经营现金流比率 |

**杠杆指标**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `debt_to_equity` | `float \| None` | 债务权益比 |
| `debt_to_assets` | `float \| None` | 资产负债率 |
| `interest_coverage` | `float \| None` | 利息覆盖倍数 |

**增长率指标**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `revenue_growth` | `float \| None` | 营收增长率 |
| `earnings_growth` | `float \| None` | 盈利增长率 |
| `book_value_growth` | `float \| None` | 账面价值增长率 |
| `earnings_per_share_growth` | `float \| None` | EPS 增长率 |
| `free_cash_flow_growth` | `float \| None` | 自由现金流增长率 |
| `operating_income_growth` | `float \| None` | 营业收入增长率 |
| `ebitda_growth` | `float \| None` | EBITDA 增长率 |

**每股指标**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `payout_ratio` | `float \| None` | 股息支付率 |
| `earnings_per_share` | `float \| None` | 每股收益（EPS） |
| `book_value_per_share` | `float \| None` | 每股账面价值 |
| `free_cash_flow_per_share` | `float \| None` | 每股自由现金流 |

---

### 4.3 InsiderTrade

来源：`/insider-trades/` 端点，表示单笔内部人士交易记录。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `ticker` | `str` | — | 股票代码（非空） |
| `name` | `str` | — | 交易人姓名（非空） |
| `filing_date` | `str` | — | SEC 申报日期（非空） |
| `is_board_director` | `bool` | `False` | 是否为董事会成员 |
| `issuer` | `str \| None` | `None` | 发行方名称 |
| `title` | `str \| None` | `None` | 职务头衔 |
| `transaction_date` | `str \| None` | `None` | 交易日期 |
| `transaction_type` | `str \| None` | `None` | 交易类型（如买入、卖出） |
| `transaction_shares` | `float \| None` | `None` | 交易股数 |
| `transaction_price_per_share` | `float \| None` | `None` | 每股交易价格 |
| `transaction_value` | `float \| None` | `None` | 交易总金额 |
| `shares_owned_before_transaction` | `float \| None` | `None` | 交易前持股数 |
| `shares_owned_after_transaction` | `float \| None` | `None` | 交易后持股数 |
| `security_title` | `str \| None` | `None` | 证券类型描述 |

---

### 4.4 CompanyNews

来源：`/news/` 端点，表示单条公司新闻。

| 字段 | 类型 | 说明 |
|------|------|------|
| `ticker` | `str` | 股票代码（非空） |
| `title` | `str` | 新闻标题（非空） |
| `source` | `str` | 新闻来源（非空） |
| `date` | `str \| None` | 发布日期 |
| `url` | `str \| None` | 新闻链接 |

---

### 4.5 CompanyFacts

来源：`/company/facts/` 端点，表示公司基础元数据。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `ticker` | `str` | — | 股票代码（非空） |
| `is_active` | `bool` | `True` | 是否为活跃上市公司 |
| `name` | `str \| None` | `None` | 公司全称 |
| `cik` | `str \| None` | `None` | SEC CIK 编号 |
| `sector` | `str \| None` | `None` | 行业板块（如 `"Technology"`） |
| `industry` | `str \| None` | `None` | 细分行业 |
| `category` | `str \| None` | `None` | 公司类别 |
| `exchange` | `str \| None` | `None` | 交易所（如 `"NASDAQ"`） |
| `location` | `str \| None` | `None` | 总部所在地 |
| `sec_filings_url` | `str \| None` | `None` | SEC 申报文件列表 URL |
| `sic_code` | `str \| None` | `None` | SIC 行业代码 |
| `sic_industry` | `str \| None` | `None` | SIC 行业分类名称 |
| `sic_sector` | `str \| None` | `None` | SIC 板块分类名称 |

**注意**: `CompanyFacts` 不包含 `market_cap` 字段。`get_market_cap()` 的"先查 company_facts"逻辑实际上永远走到后备的 `financial_metrics` 分支（因为 `CompanyFacts` 没有 `market_cap` 字段，属性访问会引发 `AttributeError`，或在运行时直接不满足 `facts.market_cap is not None` 的判断）。

---

### 4.6 EarningsData

内嵌于 `Earnings` 和 `EarningsRecord` 的子模型，表示单一报告期的财务数据。所有字段均可为 `None`。

**实际与预测对比**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `revenue` | `float \| None` | 实际营收 |
| `estimated_revenue` | `float \| None` | 预测营收 |
| `revenue_surprise` | `str \| None` | 营收超预期情况（`"BEAT"` / `"MISS"` / `"MEET"`） |
| `earnings_per_share` | `float \| None` | 实际 EPS |
| `estimated_earnings_per_share` | `float \| None` | 预测 EPS |
| `eps_surprise` | `str \| None` | EPS 超预期情况（`"BEAT"` / `"MISS"` / `"MEET"`） |

**利润表**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `net_income` | `float \| None` | 净利润 |
| `gross_profit` | `float \| None` | 毛利润 |
| `operating_income` | `float \| None` | 营业收入 |
| `weighted_average_shares` | `float \| None` | 加权平均股数（基本） |
| `weighted_average_shares_diluted` | `float \| None` | 加权平均股数（摊薄） |
| `free_cash_flow` | `float \| None` | 自由现金流 |

**资产负债表**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `cash_and_equivalents` | `float \| None` | 现金及等价物 |
| `total_debt` | `float \| None` | 总债务 |
| `total_assets` | `float \| None` | 总资产 |
| `total_liabilities` | `float \| None` | 总负债 |
| `shareholders_equity` | `float \| None` | 股东权益 |

**现金流量表**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `net_cash_flow_from_operations` | `float \| None` | 经营活动现金流净额 |
| `capital_expenditure` | `float \| None` | 资本支出 |
| `net_cash_flow_from_investing` | `float \| None` | 投资活动现金流净额 |
| `net_cash_flow_from_financing` | `float \| None` | 融资活动现金流净额 |
| `change_in_cash_and_equivalents` | `float \| None` | 现金变动量 |

**同比变化**（API 无数据时字段缺失）:

| 字段 | 类型 | 说明 |
|------|------|------|
| `revenue_chg` | `float \| None` | 营收同比变化率 |
| `net_income_chg` | `float \| None` | 净利润同比变化率 |
| `operating_income_chg` | `float \| None` | 营业收入同比变化率 |
| `gross_profit_chg` | `float \| None` | 毛利润同比变化率 |
| `free_cash_flow_chg` | `float \| None` | 自由现金流同比变化率 |

---

### 4.7 Earnings

来源：`/earnings/`（单记录模式，即 `get_earnings()` 返回的最新财报）。

| 字段 | 类型 | 说明 |
|------|------|------|
| `ticker` | `str` | 股票代码（非空） |
| `report_period` | `str` | 报告期（非空） |
| `fiscal_period` | `str \| None` | 财政期间（如 `"Q4 2024"`） |
| `currency` | `str \| None` | 货币代码 |
| `quarterly` | `EarningsData \| None` | 当季财务数据 |
| `annual` | `EarningsData \| None` | 年度财务数据 |

---

### 4.8 EarningsRecord

来源：`/earnings/`（历史列表模式，即 `get_earnings_history()` 返回的每条记录）。每条记录对应一份 SEC 申报文件。

| 字段 | 类型 | 说明 |
|------|------|------|
| `ticker` | `str` | 股票代码（非空） |
| `report_period` | `str` | 报告期（非空） |
| `source_type` | `str` | 申报文件类型（非空，如 `"8-K"`、`"10-Q"`、`"10-K"`、`"20-F"`） |
| `filing_date` | `str \| None` | 申报日期 |
| `filing_datetime` | `str \| None` | 申报时间（含时分秒） |
| `filing_window` | `str \| None` | 申报时间窗口标识 |
| `fiscal_period` | `str \| None` | 财政期间 |
| `currency` | `str \| None` | 货币代码 |
| `filing_url` | `str \| None` | SEC EDGAR 申报文件 URL |
| `accession_number` | `str \| None` | SEC 申报登记号 |
| `quarterly` | `EarningsData \| None` | 当季财务数据 |
| `annual` | `EarningsData \| None` | 年度财务数据 |

**注意**: 同一个 `report_period` 可能返回多条记录，分别对应不同的 `source_type`（例如，同一季度可同时存在 `8-K` 和 `10-Q` 两条记录）。

---

### 4.9 Filing

来源：`/filings/` 端点，表示单份 SEC 申报文件元数据。所有字段均可为 `None`（包括 `ticker`）。

| 字段 | 类型 | 说明 |
|------|------|------|
| `ticker` | `str \| None` | 股票代码 |
| `cik` | `str \| None` | SEC CIK 编号 |
| `accession_number` | `str \| None` | 申报登记号 |
| `filing_type` | `str \| None` | 文件类型（如 `"10-K"`、`"8-K"`） |
| `filing_date` | `str \| None` | 申报日期 |
| `report_period` | `str \| None` | 报告期 |
| `document_count` | `int \| None` | 文件包含的文档数量 |
| `is_xbrl` | `bool \| None` | 是否包含 XBRL 结构化数据 |
| `url` | `str \| None` | 申报文件 URL |

---

## 5. DataClient 协议

`DataClient` 定义于 `v2/data/protocol.py`，是一个 `@runtime_checkable` 的 Python Protocol，描述所有数据提供者必须实现的方法签名。

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class DataClient(Protocol):
    def get_prices(self, ticker, start_date, end_date, **kwargs) -> list[Price]: ...
    def get_financial_metrics(self, ticker, end_date, period="ttm", limit=10) -> list[FinancialMetrics]: ...
    def get_news(self, ticker, end_date, start_date=None, limit=1000) -> list[CompanyNews]: ...
    def get_insider_trades(self, ticker, end_date, start_date=None, limit=1000) -> list[InsiderTrade]: ...
    def get_company_facts(self, ticker) -> CompanyFacts | None: ...
    def get_earnings(self, ticker) -> Earnings | None: ...
```

**无需继承**：任何实现了上述方法的类，无论是否继承自 `DataClient`，均可通过 `isinstance(obj, DataClient)` 检查，并在流水线中作为数据提供者使用。

**示例**（自定义数据提供者）:

```python
class YFinanceClient:
    def get_prices(self, ticker, start_date, end_date, **kwargs):
        # 从 yfinance 获取数据，返回 list[Price]
        ...

client: DataClient = YFinanceClient()
assert isinstance(client, DataClient)  # True（结构化子类型）
```

---

## 6. 与 v1 数据层的区别

v1 数据层位于 `src/tools/api.py`，以独立函数形式提供，不封装为类。

| 对比维度 | v1 (`src/tools/api.py`) | v2 (`v2/data/client.py`) |
|----------|------------------------|--------------------------|
| **接口形式** | 独立函数（如 `get_prices(ticker, ...)`) | 类方法（`FDClient().get_prices(ticker, ...)`） |
| **HTTP 连接** | 每次调用单独创建 `requests.get/post` | `requests.Session` 持久连接，性能更优 |
| **认证头** | 每次请求手动传入 `headers` 字典 | 初始化时写入 Session 级别 `X-API-Key` |
| **重试延迟** | `60s → 90s → 120s`（线性递增） | `5s → 15s → 30s`（3 次，更短） |
| **重试后行为** | 返回最终响应对象（可能含错误） | 返回 `None`，永不抛出异常 |
| **内存缓存** | 有（`src/data/cache.py` in-memory cache） | 无（无状态客户端） |
| **盈利数据** | 无 `get_earnings` / `get_earnings_history` | 提供 `get_earnings()` 和 `get_earnings_history()` |
| **公司数据** | 有 `get_company_facts`（通过 `CompanyFactsResponse`） | 有 `get_company_facts`（通过 `CompanyFacts` 直接模型） |
| **LineItem 支持** | 有 `search_line_items`（POST 请求） | 无（v2 不提供此接口） |
| **模型 extra 策略** | `extra="allow"`（允许额外字段，用于 `LineItem`） | `extra="ignore"`（忽略额外字段，向前兼容） |
| **类型注解** | 部分 | 完整（Python 3.10+ union 语法） |
| **上下文管理器** | 不支持 | 支持 `with FDClient() as fd:` |
| **抽象协议** | 无 | `DataClient` Protocol（支持替换数据源） |

---

## 7. 依赖关系

### 7.1 第三方库

| 库 | 用途 |
|----|------|
| `requests` | HTTP 客户端（含 `Session` 持久连接） |
| `pydantic` | 数据模型定义与验证（`BaseModel`） |

### 7.2 标准库

| 模块 | 用途 |
|------|------|
| `os` | 读取 `FINANCIAL_DATASETS_API_KEY` 环境变量 |
| `time` | `time.sleep()` 实现重试等待 |
| `logging` | 结构化日志输出 |
| `typing` | `Protocol`、`runtime_checkable` 用于协议定义 |
| `__future__.annotations` | 启用延迟类型注解求值（PEP 563） |

### 7.3 项目内部依赖

v2 数据层不依赖项目其他模块，是完全自包含的独立层。外部代码通过 `v2.data` 包导入所需类型：

```python
from v2.data import FDClient, DataClient, Price, FinancialMetrics
```
