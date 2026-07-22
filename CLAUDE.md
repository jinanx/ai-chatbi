# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SQLBot is an intelligent data Q&A system (ChatBI) built on LLMs and RAG. Users ask natural language questions and get SQL queries, data results, and visualization charts. Developed by the DataEase open source project group (FIT2CLOUD). Version 1.10.0, licensed under FIT2CLOUD Open Source License (essentially GPLv3 with logo/branding restrictions).

## Build & Development Commands

### Backend (Python FastAPI)

```bash
cd backend

# Install dependencies (use uv, not pip)
uv sync                    # full install
uv sync --extra cpu         # CPU-only PyTorch variant

# Run dev server
uvicorn main:app --host 0.0.0.0 --port 8000 --reload

# Run MCP server separately (port 8001)
uvicorn main:mcp_app --host 0.0.0.0 --port 8001

# Lint & format
ruff check apps scripts common --fix    # lint with auto-fix
ruff format apps scripts common         # format

# Type check
mypy apps scripts common                # strict mode, excludes alembic/venv

# Tests
pytest                                   # run all tests
pytest tests/test_cwe89_escape_fix.py    # run a single test file
pytest -k "test_supplier_config"         # run tests matching a name

# Database migrations
alembic upgrade head                     # apply migrations
```

### Frontend (Vue 3 + TypeScript + Vite)

```bash
cd frontend

npm install              # install dependencies
npm run dev              # dev server (port 5173), runs vue-tsc then vite
npm run build            # production build (vue-tsc + vite build)
npm run lint             # ESLint with --fix (Vue + TS + Prettier rules)
```

Frontend has no test framework configured.

### g2-ssr (Node.js chart image service)

```bash
cd g2-ssr
npm install              # install dependencies
pm2 start app.js         # start on port 3000
```

### Docker

```bash
docker compose up -d     # full stack: PostgreSQL + g2-ssr + MCP + API
```

### Pre-commit hooks

Pre-commit runs `ruff --fix`, `ruff-format`, and standard hooks (trailing whitespace, YAML/TOML checks, large files). Install: `pre-commit install`.

### Spell check

`typos` runs in CI on push/PR. Config in `.typos.toml`.

## Architecture

### Three-service runtime

The app runs four processes via `start.sh`:
1. **PostgreSQL** (internal, port 5432) — SQLBot's own metadata DB (uses SQLModel/SQLAlchemy)
2. **g2-ssr** (Node.js, port 3000) — SSR chart image generation using AntV G2
3. **MCP server** (uvicorn, port 8001) — Model Context Protocol server via `fastapi-mcp`
4. **Main API** (uvicorn, port 8000) — FastAPI application

### Backend structure (`backend/`)

Domain modules live in `apps/`, each following an `api/crud/models/schemas` pattern:

| Module | Purpose |
|--------|---------|
| `chat/` | Core Q&A pipeline. `task/llm.py` is the main orchestrator — runs SQL generation → execution → chart → analysis → prediction through LLM calls. Uses ThreadPoolExecutor (200 workers). |
| `ai_model/` | LLM factory using LangChain. Supports OpenAI-compatible, vLLM, Azure. API keys encrypted at rest (AES). |
| `datasource/` | Multi-database management with connection pooling (LRU eviction, max 500 pools). Supports 15+ DB types. Has embedding-based semantic search for table/datasource matching. |
| `db/` | Database abstraction. `engine.py` provides connection pooling and SQL execution with sqlglot-based injection prevention. Two pool managers: `ConnectionPoolManager` (SQLAlchemy) and `DriverConnectionPoolManager` (native drivers). |
| `template/` | YAML prompt template system. Per-DB-type SQL templates in `templates/sql_examples/` (PostgreSQL, MySQL, Oracle, etc.). |
| `system/` | Users, workspaces, AI model configs, assistants, API keys, auth. JWT token auth via `X-SQLBOT-TOKEN` header. |
| `terminology/` | Custom business terminology for domain-specific SQL generation. |
| `data_training/` | SQL example training data to improve SQL generation accuracy. |
| `mcp/` | MCP protocol endpoints for external integrations (n8n, Dify, MaxKB, DataEase). |
| `dashboard/` | Dashboard management for organizing chart visualizations. |

Shared code in `common/`:
- `core/config.py` — Pydantic settings from env vars
- `core/db.py` — SQLModel engine/session for the app's own PostgreSQL
- `core/security.py` — JWT + bcrypt + MD5 password hashing
- `core/deps.py` — FastAPI dependency injection (session, i18n, current user, current assistant)
- `core/sqlbot_cache.py` — Redis or in-memory caching
- `core/models.py` — SnowflakeBase (all IDs use Snowflake BigInteger IDs)
- `utils/` — AES encryption, i18n, Snowflake ID gen, Excel handling, data formatting

Middleware stack (order matters): CORS → TokenMiddleware → ResponseMiddleware → RequestContextMiddleware

API router prefix: `/api/v1` (configurable via `CONTEXT_PATH` env var).

### Frontend structure (`frontend/src/`)

- `api/` — Axios-based service modules with token auth (`X-SQLBOT-TOKEN` header), retry logic, and streaming via `fetch`
- `views/` — Page components by feature (chat with answer blocks, dashboard, datasource management, system settings, embedded views)
- `stores/` — Pinia stores (user auth, assistant config, chat config, appearance)
- `router/` — Hash-based routing with lazy loading
- `i18n/` — 4 locales: zh-CN, zh-TW, en, ko-KR (with Element Plus locale merging)
- `components/` — Shared components (layout, drawer, filter, rich text with TinyMCE)

Embedded support: dedicated routes for iframe/popup/API embedding with assistant-based auth (`X-SQLBOT-ASSISTANT-TOKEN`).

### Key architectural patterns

1. **LLM pipeline** (`chat/task/llm.py`): Multi-step — select datasource → generate SQL → execute SQL → generate chart → analysis → prediction. Each step logged in `chat_log`. RAG context from terminology embeddings, data training embeddings, table embeddings, and datasource embeddings.

2. **SQL safety**: All user-generated SQL validated as read-only via `sqlglot` parsing before execution. Blocks write operations and dangerous functions per database type.

3. **Connection pooling with LRU eviction**: Thread-safe LRU eviction supporting up to 500 concurrent connections across two pool manager types.

4. **Snowflake IDs**: All entity IDs are Snowflake BigInteger for distributed ID generation.

5. **Template-driven prompts**: YAML templates define system/user prompts for each LLM step, with per-database SQL dialect templates.

6. **Encrypted credentials**: Datasource credentials and API keys encrypted at rest using AES.

7. **Workspace isolation**: Multi-tenant with workspace-scoped data access, role-based permissions, and row-level security filters.

8. **i18n throughout**: Both backend (4 locales, localized OpenAPI docs) and frontend (4 locales with Element Plus locale sync).

## Code Style

### Backend (Python)
- Ruff lint: pycodestyle E/W, pyflakes F, isort I, bugbear B, comprehensions C4, pyupgrade UP, unused args ARG001
- Ruff format for formatting
- Mypy strict mode
- Target Python 3.11 (exact: `==3.11.*`)
- Package manager: `uv` (not pip/poetry)

### Frontend (TypeScript/Vue)
- Prettier: single quotes, no semicolons, trailing commas, 100 char print width
- ESLint: flat config with typescript-eslint, vue plugin, prettier plugin
- CSS: Less preprocessor
- Vue 3 Composition API with `<script setup>`
- Element Plus with auto-import

## Testing

Backend uses pytest + coverage. Root-level `tests/` directory contains:
- `test_cwe89_escape_fix.py` — SQL injection prevention
- `test_supplier_config.py` — Frontend/backend supplier registry consistency
- `test_minimax_integration.py` — LLM supplier configuration

Note: `backend/scripts/test.sh` references `app/` paths from the template fork — the actual codebase uses `apps/`, `common/`, `scripts/` modules. Run `pytest` directly instead of via the script.

## Environment Configuration

Settings loaded from `../.env` (one level above `backend/`) via Pydantic Settings. Key env vars:
- `POSTGRES_SERVER`, `POSTGRES_PORT`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` — internal PostgreSQL
- `CONTEXT_PATH` — API prefix (default empty)
- `FRONTEND_HOST` — CORS origin (default `http://localhost:5173`)
- `BACKEND_CORS_ORIGINS` — additional CORS origins
- `SQLBOT_DOC_ENABLED` — enable/disable OpenAPI docs

## Database Support

SQLBot connects to 15+ external database types for user data querying: PostgreSQL, MySQL, SQL Server, Oracle, ClickHouse, DM (Dameng), Doris, StarRocks, Redshift, Kingbase, Elasticsearch, Hive, Excel/CSV. Each has dialect-specific SQL templates and connection driver support.
