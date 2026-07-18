# CLAUDE.md

Factory Inventory Management System — a full-stack demo app (built for a Claude Code workshop) with a Vue 3 frontend, a Python FastAPI backend, and in-memory mock data loaded from JSON (no database). It covers inventory tracking, order management, demand forecasting, backlog monitoring, spending analytics, and quarterly/monthly reports, with English/Japanese localization.

> This is the top-level overview. Deeper, task-specific guidance lives in **`server/CLAUDE.md`** (FastAPI backend) and **`client/CLAUDE.md`** (Vue frontend) — read those before doing substantial work in either half.

## Critical Tool Usage Rules

### Subagents
Use the Task tool with these specialized subagents for appropriate tasks:

- **vue-expert**: Use for Vue 3 frontend features, UI components, styling, and client-side functionality
  - Examples: Creating components, fixing reactivity issues, performance optimization, complex state management
  - **MANDATORY RULE: ANY time you need to create or significantly modify a .vue file, you MUST delegate to vue-expert**
- **code-reviewer**: Use after writing significant code to review quality and best practices
- **security-auditor**: Use for a fast security review of changed files (focuses only on the diff, not the whole codebase)
- **Explore**: Use for understanding codebase structure, searching for patterns, or answering questions about how components work
- **general-purpose**: Use for complex multi-step tasks or when other agents don't fit

### Skills
- **backend-api-test** skill: Use when writing or modifying tests in `tests/backend` directory with pytest and FastAPI TestClient

### MCP Tools
- **ALWAYS use GitHub MCP tools** (`mcp__github__*`) for ALL GitHub operations
  - Exception: Local branches only - use `git checkout -b` instead of `mcp__github__create_branch`
- **ALWAYS use Playwright MCP tools** (`mcp__playwright__*`) for browser testing
  - Test against: `http://localhost:3000` (frontend), `http://localhost:8001` (API)

### Slash Commands
Custom commands live in `.claude/commands/`:
- `/start` – kill anything on ports 3000/8001, then start both servers
- `/stop` – stop the frontend and backend servers
- `/test` – run the full test suite with reporting
- `/optimize` – scan for and remove dead code / optimize the codebase
- `/demo-branch` – create an auto-incrementing `demo-branch-N`
- `/reset-branch` – switch to `main`, delete the current branch, and close its PRs (destructive)

## Stack
- **Frontend**: Vue 3 (Composition API) + Vue Router + Vite + axios — port 3000
- **Backend**: Python FastAPI + Pydantic + uvicorn (Python ≥ 3.11), managed with `uv` — port 8001
- **Data**: JSON files in `server/data/`, loaded into memory at startup via `server/mock_data.py` (changes never persist — restart reloads from disk)

## Quick Start

```bash
# One command (macOS/Linux) — starts both servers
./scripts/start.sh          # stop with ./scripts/stop.sh

# Backend (manual)
cd server
uv run python main.py       # http://localhost:8001, docs at /docs

# Frontend (manual)
cd client
npm install && npm run dev  # http://localhost:3000
```

On Windows the shell scripts don't work — run the manual backend and frontend commands in separate terminals.

## Architecture & Data Flow

```
Vue view/component
  → useFilters() composable (shared filter state)
  → client/src/api.js (axios, maps filters to query params, drops 'all')
  → FastAPI endpoint (server/main.py)
  → in-memory list filtering (apply_filters / filter_by_month)
  → Pydantic response_model validation
  → back to the view, held in refs; derived values via computed properties
```

**Reactivity rule**: raw server data lives in `ref`s (e.g. `allOrders`, `inventoryItems`); everything derived (filtered lists, totals, chart data) is a `computed`. Never mutate a computed.

## Project Structure

```
├── server/                 # FastAPI backend
│   ├── main.py             # ALL endpoints, Pydantic models, filter helpers
│   ├── mock_data.py        # loads server/data/*.json into module-level lists
│   ├── generate_data.py    # script to (re)generate mock data
│   ├── data/*.json         # source of truth for all app data
│   ├── pyproject.toml      # deps + dev-deps (uv), requirements.txt mirrors runtime deps
│   └── CLAUDE.md           # backend-specific guidance
├── client/                 # Vue 3 frontend
│   ├── src/
│   │   ├── main.js         # app entry + Vue Router route table
│   │   ├── App.vue         # shell: nav tabs, profile menu, tasks, global styles
│   │   ├── api.js          # centralized axios API client
│   │   ├── views/*.vue     # page-level components
│   │   ├── components/*.vue# reusable UI (filter bar, detail modals, profile, i18n switcher)
│   │   ├── composables/    # useFilters, useI18n, useAuth
│   │   ├── locales/        # en.js, ja.js translation dictionaries
│   │   └── utils/currency.js
│   ├── vite.config.js      # dev server pinned to port 3000
│   └── CLAUDE.md           # frontend-specific guidance
├── tests/backend/          # pytest + FastAPI TestClient
├── scripts/                # start.sh / stop.sh
└── .claude/                # commands, agents, skills, hooks
```

## API Endpoints (server/main.py)

All list endpoints filter over in-memory data; missing/`all` query params mean "no filter".

| Method & Path | Filters | Notes |
|---|---|---|
| `GET /` | — | API banner/version |
| `GET /api/inventory` | warehouse, category | No time dimension (inventory has no date) |
| `GET /api/inventory/{item_id}` | — | 404 if not found |
| `GET /api/orders` | warehouse, category, status, month | |
| `GET /api/orders/{order_id}` | — | 404 if not found |
| `GET /api/demand` | — | Demand forecasts |
| `GET /api/backlog` | — | Adds computed `has_purchase_order` flag per item |
| `GET /api/dashboard/summary` | warehouse, category, status, month | Aggregates value, low-stock, pending, backlog counts |
| `GET /api/spending/summary` | — | |
| `GET /api/spending/monthly` | — | |
| `GET /api/spending/categories` | — | |
| `GET /api/spending/transactions` | — | |
| `GET /api/reports/quarterly` | — | Computes per-quarter revenue, fulfillment rate |
| `GET /api/reports/monthly-trends` | — | Month-over-month order/revenue trends |

**Filtering internals**: `apply_filters()` handles warehouse/category/status (case-insensitive, skips `'all'`); `filter_by_month()` matches a direct month (`2025-09`) or a quarter (`Q1-2025`..`Q4-2025` via `QUARTER_MAP`) against each item's `order_date`.

## Frontend

**Routed views** (`client/src/main.js`): `/` Dashboard · `/inventory` · `/orders` · `/spending` · `/demand` · `/reports`. These six are the nav tabs in `App.vue`.

> `client/src/views/Backlog.vue` exists but is **not routed**. Backlog data is surfaced inline on the Dashboard (backlog table + `BacklogDetailModal`), not as its own page.

**Composables** (`client/src/composables/`):
- `useFilters()` – singleton filter state: `selectedPeriod`, `selectedLocation`, `selectedCategory`, `selectedStatus`. `getCurrentFilters()` maps UI names to API params — **`selectedLocation` → `warehouse`, `selectedPeriod` → `month`**. Keep that mapping in mind when tracing a filter.
- `useI18n()` – `t(key, params)` translation lookup with English fallback; locale persisted to `localStorage` (`app-locale`).
- `useAuth()` – mock signed-in user (John Doe / 田中 太郎), including a client-side `tasks` array.

**Localization**: English (`en.js`) and Japanese (`ja.js`). Currency follows locale automatically — `en → USD`, `ja → JPY` (see `useI18n` + `utils/currency.js`). Any user-facing string should go through `t()` and exist in both locale files.

**Components** (`client/src/components/`): `FilterBar`, `LanguageSwitcher`, `ProfileMenu`, `ProfileDetailsModal`, `TasksModal`, and detail modals (`InventoryDetailModal`, `ProductDetailModal`, `CostDetailModal`, `BacklogDetailModal`).

## Filter System

Four global filters flow through every filterable view via `useFilters()`:

| UI filter | State ref | API param | Values (from data) |
|---|---|---|---|
| Time Period | `selectedPeriod` | `month` | `2025-01`…`2025-12`, `Q1-2025`…`Q4-2025` |
| Warehouse / Location | `selectedLocation` | `warehouse` | San Francisco, Tokyo, London |
| Category | `selectedCategory` | `category` | Circuit Boards, Sensors, Actuators, Controllers, Power Supplies |
| Order Status | `selectedStatus` | `status` | Delivered, Shipped, Processing, Backordered |

Inventory ignores `month`/`status` (no time or status dimension). Revenue goal on the Dashboard: **$800K per month**, i.e. **$9.6M** across all 12 months.

## Data Files (`server/data/`)

Orders span the full year **2025-01 → 2025-12**. Approximate sizes: `inventory.json` (~32 items), `orders.json` (~250), `transactions.json` (~56), `demand_forecasts.json` (~9), `backlog_items.json` (~4), `spending.json` (object with `spending_summary` / `monthly_spending` / `category_spending`), `purchase_orders.json` (currently empty `[]`). Regenerate with `server/generate_data.py`.

## Testing

- Backend tests: `tests/backend/` (`test_dashboard.py`, `test_inventory.py`, `test_misc_endpoints.py`), shared `client` fixture in `conftest.py` (FastAPI `TestClient`; adds `server/` to `sys.path`).
- Config: `tests/pytest.ini` (`testpaths = backend`, verbose, `asyncio_mode = auto`).
- Run: `cd tests && uv run pytest backend/ -v` (or the `/test` command). Follow the **backend-api-test** skill when adding tests.
- There are **no frontend unit tests**; use Playwright MCP for browser-level checks.

## Known Gotchas & Conventions

1. **Missing backend endpoints (frontend calls them anyway)**: `client/src/api.js` defines `getTasks/createTask/deleteTask/toggleTask` (`/api/tasks*`) and `createPurchaseOrder/getPurchaseOrderByBacklogItem` (`/api/purchase-orders*`), but **`server/main.py` implements none of them**. These calls 404 and are swallowed by `try/catch` (see `App.vue` task handlers) — Tasks work off client-side mock data in `useAuth`. If you build out Tasks or Purchase Orders, add the FastAPI endpoints first. Note `main.py` already declares `PurchaseOrder` / `CreatePurchaseOrderRequest` models with no route using them.
2. **v-for keys**: use a stable unique key (`sku`, `id`, `month`), never the array `index`.
3. **Validate dates before parsing** (`new Date(...)` → check `!isNaN(getTime())` before `.getMonth()`).
4. **Keep Pydantic models in sync** with JSON structure — update `main.py` models when `server/data/*.json` fields change, then restart the server to reload data.
5. **Filter on copies, never mutate** the module-level lists in `mock_data.py`.
6. **Reports.vue** calls `/api/reports/*` with axios directly rather than through `api.js` — mirror whichever pattern the file you're editing already uses.
7. **CORS** is wide open (`allow_origins=["*"]`) — demo only, not production-safe. There is no auth, rate limiting, or persistence.

## Design System
- Colors: slate/gray neutrals (`#0f172a`, `#64748b`, `#e2e8f0`); accent green `#059669`/`#10b981`
- Status colors: green / blue / yellow / red
- Charts: hand-built SVG; CSS Grid for layout, Flexbox for arrangement
- Scoped styles per component; global styles in `client/src/App.vue`
- **No emojis in the UI**

## Tooling Notes (`.claude/`)
- **Agents**: `vue-expert` (sonnet), `code-reviewer` (sonnet), `security-auditor` (haiku) — see subagent rules above.
- **MCP servers**: `playwright` + `github`, configured in `.mcp.json` (GitHub needs `GITHUB_PERSONAL_ACCESS_TOKEN` in the env).
- **Hook**: `.claude/hooks/post-tool-use.sh` logs tool usage to `logs/`. It is a standalone script and is **not wired into a `settings.json`**, so it is dormant unless you register it as a `PostToolUse` hook.
