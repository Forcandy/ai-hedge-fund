# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture Overview

AI-powered hedge fund system with two interfaces:

1. **Web Application** (`app/`) - React/FastAPI with visual flow builder (React Flow)
2. **CLI Interface** (`src/`) - Python application using LangGraph for multi-agent coordination

### How the Agent System Works

The core pattern is a LangGraph `StateGraph` where all selected analyst agents run **in parallel**, then feed into risk manager → portfolio manager sequentially.

**State shape** (`src/graph/state.py`):
```python
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]  # accumulated messages
    data: Annotated[dict, merge_dicts]      # tickers, portfolio, analyst_signals, dates
    metadata: Annotated[dict, merge_dicts]  # show_reasoning, model_name, model_provider
```

**Workflow** (`src/main.py` → `create_workflow()`):
1. Start node initializes state with tickers, portfolio, dates
2. All selected analysts execute in parallel (each reads market data, returns signals)
3. Risk manager aggregates signals, calculates position limits
4. Portfolio manager makes final buy/sell/hold decisions
5. Output: JSON with trading decisions per ticker

**Agent types** (`src/agents/`, registered in `src/utils/analysts.py`):
- 12 famous investor agents (Buffett, Lynch, Graham, etc.) - each mimics an investing style
- 4 specialist analysts (technical, fundamentals, valuation, sentiment)
- News sentiment analyst, Growth analyst
- Risk manager + Portfolio manager (always run, not selectable)

Each agent is a function `(AgentState) -> dict` that updates state with its signals.

### Data Layer

- **API client** (`src/tools/api.py`) - Financial Datasets API with rate limiting (exponential backoff 60s→150s)
- **Cache** (`src/data/cache.py`) - In-memory cache for prices, metrics, line items, insider trades, news
- **Models** (`src/data/models.py`) - Pydantic models for all financial data types
- Free tickers (no API key): AAPL, GOOGL, MSFT, NVDA, TSLA

### LLM Configuration

Models configured in `src/llm/models.py`, loaded from JSON files (`api_models.json`, `ollama_models.json`).

Supported providers: OpenAI, Anthropic, Groq, DeepSeek, Google Gemini, xAI, Azure OpenAI, Ollama, OpenRouter, GigaChat.

`get_model(model_name, model_provider)` instantiates the correct LangChain chat model.

### Web Backend (`app/backend/`)

FastAPI with SQLAlchemy + Alembic migrations. Key tables:
- `hedge_fund_flows` - Saved React Flow configurations (nodes, edges, viewport)
- `hedge_fund_flow_runs` - Execution runs
- `hedge_fund_flow_run_cycles` - Analysis cycles within runs
- `api_keys` - Stored API keys

Routes in `app/backend/routes/`, services in `app/backend/services/`.

### Web Frontend (`app/frontend/src/`)

React 18 + Vite. Key patterns:
- React Flow for visual workflow builder (custom nodes in `components/nodes/`)
- Shadcn/Radix UI components (`components/ui/`)
- Context providers for state (`contexts/` - flow, nodes, tabs)
- Resizable split panels layout

### Backtesting System (`src/backtesting/`)

Engine runs the agent workflow over historical periods. Entry point: `src/backtester.py`. Components: engine, portfolio state, metrics, benchmarks, CLI output.

## Development Commands

### Backend
```bash
cd app/backend && poetry run fastapi dev main.py          # Start dev server
cd app/backend && poetry run alembic upgrade head          # Run migrations
cd app/backend && poetry run alembic revision --autogenerate -m "desc"  # New migration
```

### Frontend
```bash
cd app/frontend && npm run dev      # Dev server (Vite, port 5173)
cd app/frontend && npm run build    # Production build
cd app/frontend && npm run lint     # ESLint
```

### CLI
```bash
poetry run python src/main.py --ticker AAPL,MSFT,NVDA
poetry run python src/main.py --ticker AAPL --ollama                    # Local LLMs
poetry run python src/main.py --ticker AAPL --start-date 2024-01-01 --end-date 2024-03-01
poetry run python src/main.py --ticker AAPL --show-reasoning            # Show agent reasoning
poetry run python src/backtester.py --ticker AAPL,MSFT,NVDA             # Backtester
```

### Full Stack
```bash
./run.sh    # Mac/Linux (from app/ directory)
run.bat     # Windows
```

### Code Quality
```bash
poetry run black .          # Format (420 char line length)
poetry run isort .          # Sort imports
poetry run flake8 .         # Lint
poetry run pytest           # All tests
poetry run pytest tests/test_api_rate_limiting.py -v   # Single test file
poetry run pytest -k "test_name"                       # Single test by name
```

## Environment Variables

See `.env.example`. At minimum, set one LLM provider key (e.g., `OPENAI_API_KEY`). Set `FINANCIAL_DATASETS_API_KEY` for tickers beyond the free five.

## Code Style

- Python: Black with 420 char line length, isort for imports
- Python 3.11+, managed with Poetry
- Frontend: ESLint, Tailwind CSS
