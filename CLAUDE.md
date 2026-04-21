# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install (development)
pip install -e .
pip install -r requirements.txt   # full deps including web UI

# Run
wyckoff                            # start TUI (CLI entry point)
streamlit run streamlit_app.py    # start web UI

# Tests
pytest                             # run all tests
pytest tests/test_wyckoff_engine.py          # single test file
pytest tests/test_wyckoff_engine.py::test_fn  # single test function

# Daily pipeline (also runs via GitHub Actions)
python -u scripts/daily_job.py --logs logs/daily_job.log
```

Requires Python >= 3.11. Configuration via `.env` (copy from `.env.example`): `SUPABASE_URL`, `SUPABASE_KEY`, `GEMINI_API_KEY` are the essential ones.

## Architecture

Two user-facing entry points share the same core logic:

- **CLI TUI** (`wyckoff` command): `cli/__main__.py` → `cli/tui.py` (Textual full-screen) → `cli/agent.py` (ReAct loop) + `cli/tools.py` (`ToolRegistry`)
- **Web UI** (`streamlit_app.py` + `pages/`): Google ADK `LlmAgent` from `agents/wyckoff_chat_agent.py`, tools from `agents/chat_tools.py`

Both surfaces share the same 12 tool functions (`agents/chat_tools.py`) and system prompts (`core/prompts.py`). The CLI has a 13th tool (`check_background_tasks`) and supports background threading for long-running tools.

### Layer Map

```
cli/ + streamlit_app.py   ← user interfaces
agents/                   ← LLM agent wiring (ADK for web, bare ReAct loop for CLI)
core/                     ← thin re-export layer (delegates to scripts/ and tools/)
scripts/                  ← heavy implementations (funnel, backtest, rebalancer, daily job)
tools/                    ← lower-level quant primitives (ranker, market regime, report builder)
integrations/             ← all external I/O (data sources, Supabase, LLM clients)
utils/                    ← notifications, trading calendar, helpers
```

`core/` modules are intentionally thin forwarding shims — the actual logic lives in `scripts/` and `tools/`. For example, `core/funnel_pipeline.py` re-exports from `scripts/wyckoff_funnel.py` and `tools/candidate_ranker.py`.

### The 5-Layer Wyckoff Funnel

Implemented in `scripts/wyckoff_funnel.py`, orchestrated via `core/funnel_pipeline.py`, parameterized by `FunnelConfig` (~60 knobs) in `core/wyckoff_engine.py`:

| Layer | Name | Logic |
|-------|------|-------|
| L1 | Quality filter | Remove ST/BSE/STAR, market cap < 3.5B CNY, daily turnover < 50M |
| L2 | Six channels | Markup / Ignition / Lurking / Accumulation / Low-volume / Defended |
| L2.5 | Markup detection | MA50 cross MA200 + angle verification |
| L3 | Sector rotation | Keep top-N industries by L2 pass-through + RPS momentum |
| L4 | Micro triggers | Spring / LPS / SOS / EVR (Effort vs Result) |
| L5 | AI judgment | LLM classifies into three camps (Invalidated / Building Cause / On Springboard) |

L4 signals enter a confirmation state machine (`core/signal_confirmation.py`): `pending → confirmed` (price confirmation within 1-3 days) or `expired` (TTL varies by signal type).

### Data Flow

**Stock history:** `integrations/stock_hist_repository.py` — Supabase cache first → gap-fill from data source fallback chain (tushare → akshare → baostock → efinance) → write-back cache.

**Auth and user credentials:** Supabase RLS-protected. API keys/tokens stored in `user_settings` table, fetched with 5-minute in-memory cache. CLI uses `~/.wyckoff/session.json` for JWT persistence.

**LLM calls:** `integrations/llm_client.py` (direct, 8+ providers) or `integrations/llm_adapter.py` (LiteLLM unified, enabled via `LITELLM_ENABLED=1`). CLI providers in `cli/providers/` implement a uniform `chat_stream()` interface returning typed chunks: `thinking_delta | text_delta | tool_calls | usage`.

### GitHub Actions Automation

Nine workflows in `.github/workflows/`. Key scheduled jobs:
- **18:25 BJT (Sun-Thu):** full pipeline — funnel → AI report → strategy → Feishu/Telegram push
- **08:20 BJT (Mon-Fri):** premarket risk check (A50 + VIX, 4-tier alert)
- **23:05 BJT daily:** Supabase cache maintenance

Scripts triggered by Actions use `service_role_key` (bypasses Supabase RLS); CLI/Web use user JWT (subject to RLS).

### CLI Background Task Architecture

Long-running tools (`screen_stocks`, `generate_ai_report`, `generate_strategy_decision`) are submitted to `cli/background.py` (`BackgroundTaskManager`) as daemon threads and return immediately with a `task_id`. On completion, a callback injects results into the TUI message queue. Users can query progress via `check_background_tasks`.

### Key Files Quick Reference

| File | Role |
|------|------|
| `core/wyckoff_engine.py` | L1–L4 filter logic, FunnelConfig, trigger detection (Spring/SOS/LPS/EVR), exit signals |
| `core/prompts.py` | All LLM system prompts |
| `agents/chat_tools.py` | 12 tool function implementations shared by web and CLI |
| `cli/tools.py` | ToolRegistry — JSON schemas for LLM, tool dispatch, BACKGROUND_TOOLS set |
| `cli/tui.py` | Textual TUI app, chat rendering, `/login /model /new /help` commands |
| `cli/agent.py` | Bare ReAct loop (up to 15 rounds), thinking display, tool call orchestration |
| `integrations/data_source.py` | Unified OHLCV fetcher with 4-source fallback chain + baostock circuit breaker |
| `integrations/supabase_client.py` | Supabase client factory (service-role vs user JWT) |
| `scripts/wyckoff_funnel.py` | Full 5-layer funnel implementation |
| `scripts/step4_rebalancer.py` | Position rebalancing decisions (EXIT/TRIM/HOLD/PROBE/ATTACK) via LLM |
| `scripts/daily_job.py` | GitHub Actions entry point: Step2 → Step3 → Step4 → push notifications |