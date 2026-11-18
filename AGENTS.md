# REMY AI BRAIN

---

You are an expert software engineer.

This is the entry point project for Remy AI, here you will get enough context of the different repos/projects related to Remy AI.

You know how each project is structured, which libraries they use, they architectures or patterns they use and the frameworks they are built in.

---

## How to help

- Perform tasks along the projects
- act as a pair programmer or critic depending on the task 

---

### **Projects structures**

**remy.ai** is an agentic AI receptionist for restaurants. It takes orders and answers menu and service questions, and it talks to customers over Telegram. The system is split into two repos:

| Project | Role | Main repo | Orca worktree | Default branch |
|---|---|---|---|---|
| **remy_ai** | The agent ("the brain"): LangGraph workflows, LLM calls, RAG | `~/remy_ai` | `~/orca/workspaces/remy_ai/featherduster` | `main` |
| **remy_api** | REST API + Telegram gateway ("the body"): DB, CRUD, webhook | `~/remy_api` | `~/orca/workspaces/remy_api/silverside` | `main` |

Both use **Python 3.14** and **uv** (`uv.lock`, `.python-version`), and keep secrets in an untracked `.env`.
GitHub owner: `diego3008`. Branch names are `feature/<name>` or `diego3008/<name>`, and work merges through PRs.

> Snapshot taken 2026-11-17. remy_ai at `9f1eae5` (`feature/api-retrieval`, same as `main`). remy_api at
> `f670bea` (`origin/main`, PR #3 merged). The local `main` checkout in `~/remy_api` was behind `origin/main`
> (at `e32d274`), so pull before you work. Check `git log` before relying on the details below.

---

## How the two projects connect

```text
Telegram user
    │  message
    ▼
Telegram Bot API ──POST /webhook/telegram──►  remy_api (FastAPI)
                                                 │ 1. normalize Update -> IncomingMessage
                                                 │ 2. find/create LangGraph thread by metadata {telegram_chat_id}
                                                 │ 3. client.runs.wait(thread, assistant_id, {"messages":[user msg]})
                                                 ▼
                                        remy_ai (LangGraph API server, `langgraph dev`)
                                                 │ main_graph: add_message -> categorize -> (business_inquiry | respond)
                                                 │ business_inquiry subgraph -> RAG over ChromaDB (menu.json)
                                                 ▼
                                        final state.messages -> last "ai" message
                                                 │
remy_api ◄───────────────────────────────────────┘
    │ bot.send_message(chat_id, text, parse_mode=MARKDOWN)
    ▼
Telegram user
```

- remy_api talks to remy_ai **only** through the `langgraph-sdk` HTTP client (`LANGGRAPH_API_URL`).
- Conversation memory lives in the **LangGraph checkpointer** (threads), one thread per Telegram chat.
  remy_api does not store message history itself.
- Planned direction: remy_ai gets **tools that call remy_api's REST endpoints** (`/api/v1/...`) to read menus and
  create customers and orders. `Order.thread_id` already exists to link an order to its LangGraph thread, and
  remy_ai's `src/helpers/url_querystring.py` (builds a URL with query params from a model) is there for this.
- ⚠️ Watch the assistant ID: remy_api defaults to `LANGGRAPH_ASSISTANT_ID="agent"`, but remy_ai's `langgraph.json`
  registers the graph as **`main_graph`**. For runs to work, remy_api's `.env` must set `LANGGRAPH_ASSISTANT_ID=main_graph`,
  or the graph key must change.

---

## remy_ai — the agent

### Tech stack
| Area | Tooling |
|---|---|
| Language / packaging | Python ≥3.14, uv |
| Agent framework | **LangGraph** (`langgraph`, `langgraph-prebuilt`, `langgraph-checkpoint`) |
| Serving | **LangGraph API / CLI** (`langgraph-cli`, `langgraph-api`, `langgraph-runtime-inmem`) via `langgraph.json` |
| LLM | **LangChain** + `langchain-openai` → `ChatOpenAI(model="gpt-6-luna")` |
| Structured output | Pydantic v2 models + `llm.with_structured_output(...)` |
| RAG | **ChromaDB** (`PersistentClient` at `./.chroma`), `sentence-transformers` embeddings (`BAAI/bge-m3`), `langchain-text-splitters` (`RecursiveCharacterTextSplitter`, 500/50) |
| Observability | LangSmith |
| Tests | pytest (+ `monkeypatch`, `unittest.mock.MagicMock`) |

### File structure
```text
remy_ai/
├── langgraph.json              # Registers graph "main_graph" -> ./src/graphs/main_graph.py:main_graph; env=.env
├── pyproject.toml              # Dependencies (uv)
├── uv.lock
├── main.py                     # Placeholder (prints hello); not the entry point
├── AGENTS.md / README.md
├── .gitignore                  # .env, .langgraph_api/, __pycache__, .chroma/
├── src/
│   ├── state.py                # TypedDict states: MainState, BusinessInquiryState (messages uses add_messages reducer)
│   ├── prompts.py              # Prompt templates: MESSAGE_WRITER_PROMPT, MESSAGE_CATEGORIZER_PROMPT, BUSINESS_INQUIRY_PROMPT
│   ├── structured_outputs.py   # MessageCategory enum + CategorizerMessageOutput (Pydantic)
│   ├── graphs/
│   │   ├── main_graph.py       # MainGraph class: entry graph + route_by_category conditional edge
│   │   └── business_inquiry_graph.py  # BusinessInquiryGraph: subgraph START -> business_inquiry (rag_answer_node) -> END
│   ├── nodes/
│   │   ├── __init__.py         # NODES registry dict: add_message, respond, categorize
│   │   ├── add_message_node.py # Copies last message into state.current_message
│   │   ├── message_categorizer.py  # LLM classifier -> state.message_category
│   │   ├── respond_node.py     # Generic LLM reply over state.messages (fallback for all non-inquiry categories)
│   │   └── rag_answer_node.py  # @tool search_restaurant_info + node that answers from retrieved context
│   ├── helpers/
│   │   ├── __init__.py         # helpers registry dict: url_query, build_recent_history
│   │   ├── message_history.py  # build_recent_history(): last 6 msgs as "User:/Agent:" text (+ optional summary)
│   │   └── url_querystring.py  # build_url_with_model(): merge model fields into a URL query string
│   ├── utils/
│   │   ├── __init__.py         # utils registry dict of RAG functions
│   │   └── rag_utils.py        # Chroma client/collection, chunking, ingest, query, retrieve_context()
│   └── data/
│       └── menu.json           # Knowledge base (Spanish): "restaurant" info + "faqs" [{id, tema, pregunta, respuesta}]
└── tests/
    ├── test_main_graph_routing.py  # Router unit tests, graph wiring, end-to-end with faked nodes/LLM
    └── test_rag_feature.py         # rag_utils, tool, and rag_answer_node tests (written red -> green -> refactor)
```
Note: the README mentions `src/models/` (models mirroring the API), but that folder does **not** exist yet.

### Architecture
**Main graph** (`MainState`):
```text
START -> add_message -> categorize --route_by_category--> business_inquiry (subgraph) -> END
                                                      └─> respond -> END
```
- `categorize` sorts the latest message (using the last 6 messages as history) into a `MessageCategory`:
  `new_order`, `order_change`, `confirmation`, `decline`, `customer_complaint`, `customer_feedback`,
  `greeting`, `business_inquiry`.
- Only `business_inquiry` has its own flow today. **Every other category falls through to `respond`**,
  a generic `ChatOpenAI` call on the raw history. New flows (orders, complaints…) are meant to be added as
  new subgraphs plus new branches in `route_by_category`.
- **business_inquiry subgraph** (`BusinessInquiryState`): `rag_answer_node` calls the `search_restaurant_info`
  tool, which calls `retrieve_context()`. That creates the Chroma `menu` collection and fills it from `menu.json`
  if it is empty, then returns the top 4 chunks. The node runs `BUSINESS_INQUIRY_PROMPT | ChatOpenAI` and returns
  `messages`, `retrieved_context` and `current_flow="business_inquiry"`.
- The subgraph is added directly as a node (`workflow.add_node("business_inquiry", business_inquiry_graph)`).
  State keys that `MainState` and `BusinessInquiryState` share flow through.
- No checkpointer is configured in code. Persistence comes from the LangGraph API server that runs the graph.

### Patterns & conventions
- **Graph-as-class**: each graph is a class whose `__init__` builds a `StateGraph` and stores `self.graph = workflow.compile()`.
  The module exports a compiled instance (`main_graph = MainGraph().graph`), which `langgraph.json` points to.
- **Registry dicts** (`NODES`, `helpers`, `utils`) are the lookup layer. Nodes are fetched with `NODES["..."]`
  when the graph is built, so tests can `monkeypatch.setitem(NODES, ...)` before building `MainGraph()`.
  Register new nodes there.
- **Nodes are plain functions** `(state) -> dict` that return partial state updates (preferred). Note that
  `message_categorizer_node` currently mutates and returns the whole state; don't copy that style.
- **LCEL chains**: `PromptTemplate | llm` or `PromptTemplate | llm.with_structured_output(Model)`. Prompts live in
  `src/prompts.py` as module constants with a `Name / Role / Goal` layout. Customer-facing language is Spanish;
  the agent answers in the customer's language.
- **Structured output** uses Pydantic models and enums in `src/structured_outputs.py`. A new category means updating
  the enum, the categorizer prompt and the router.
- **LangChain `@tool`** wraps capabilities. The node currently invokes the tool directly; there is no LLM tool-calling yet.
- **Testing**: tests never call the LLM or retrieval for real. Patch the name in the module that uses it, e.g.
  `monkeypatch.setattr(rag_node_module, "ChatOpenAI", FakeLLM)` and
  `monkeypatch.setattr(rag_node_module, "retrieve_context", ...)`. Chroma collections are `MagicMock`s.
  Tests are written TDD style (red → green → refactor).

### Commands
```bash
uv sync                      # install deps
uv run langgraph dev         # run the LangGraph API server locally (Studio + HTTP API, needs .env with OPENAI_API_KEY)
uv run pytest                # tests (run from repo root; tests import `src.*`)
```
`.chroma/` is created on the first retrieval. Delete it to force re-ingestion after editing `menu.json`.

---

## remy_api — REST API & Telegram gateway

### Tech stack
| Area | Tooling |
|---|---|
| Language / packaging | Python ≥3.14, uv |
| Web framework | **FastAPI** (`fastapi[standard]`), uvicorn |
| ORM | **SQLAlchemy 2.x async** (`Mapped` / `mapped_column`, `AsyncSession`), `greenlet` (`sqlmodel` is a dependency but unused) |
| Database | **PostgreSQL**: `asyncpg` for the app, `psycopg` (sync) for Alembic |
| Migrations | **Alembic** (autogenerate from `Base.metadata`) |
| Validation / config | Pydantic v2 schemas, `pydantic-settings` (`.env`) |
| Messaging | `python-telegram-bot` v22 (webhook mode, no updater) |
| Agent client | `langgraph-sdk` |
| Tests | pytest + `pytest-asyncio` (`asyncio_mode=auto`), `httpx` `AsyncClient` + `ASGITransport`, `aiosqlite` |

### File structure
```text
remy_api/
├── main.py                     # FastAPI app: lifespan (PTB app + Telegram webhook setup, engine dispose), routers, GET /health
├── pyproject.toml              # Deps + dev group; pytest config (asyncio_mode=auto, pythonpath=.)
├── alembic.ini
├── alembic/
│   ├── env.py                  # Uses Settings.sync_database_url (psycopg); target_metadata = Base.metadata
│   └── versions/               # conversations, businesses, businesses.phone, menu/dish/customer/order(+items)
├── docs/
│   └── webhook-fix-notes.md    # Telegram flood-control fix (only set_webhook when URL changed; catch RetryAfter)
├── CLAUDE.md                   # Project overview + "adding a new resource" checklist
├── src/
│   ├── config.py               # Settings (BaseSettings) + @lru_cache get_settings(); builds async/sync DB URLs
│   ├── db.py                   # @lru_cache get_engine()/get_sessionmaker() (expire_on_commit=False), get_session() dependency
│   ├── api/
│   │   ├── __init__.py         # Exports api_router, telegram_router
│   │   ├── telegram.py         # POST /webhook/telegram — always returns 200
│   │   ├── agent_client.py     # LangGraph SDK client: thread per chat (metadata search), runs.wait, extract last AI msg
│   │   └── routes/             # REST CRUD, all under /api/v1
│   │       ├── __init__.py     # Builds api_router from resource routers
│   │       ├── _common.py      # SessionDep, PaginationDep, get_or_404, commit_or_409, delete_or_404, apply_updates
│   │       ├── businesses.py   # /businesses  (filter: b_active)
│   │       ├── menus.py        # /menus       (filter: business_id)
│   │       ├── dishes.py       # /dishes      (filters: menu_id, business_id)
│   │       ├── customers.py    # /customers   (filters: business_id, phone)
│   │       ├── orders.py       # /orders (filters: business_id, customer_id, status, thread_id) + /orders/{id}/items
│   │       └── conversations.py# /conversations
│   ├── models/                 # SQLAlchemy ORM models
│   │   ├── base.py             # Base (DeclarativeBase), TimestampMixin, Money = Numeric(10, 2)
│   │   ├── enums.py            # Channel, Fulfillment, OrderStatus + _enum() helper (store values in named PG enums)
│   │   ├── business.py         # Businesses (name, phone, b_active, timezone="America/Monterrey")
│   │   ├── menu.py             # Menu -> business (CASCADE)
│   │   ├── dish.py             # Dish -> menu (CASCADE), price >= 0, is_available
│   │   ├── customer.py         # Customer -> business (CASCADE), unique (business_id, phone)
│   │   ├── order.py            # Order -> business/customer (RESTRICT), delivery needs address, thread_id, recalculate_total()
│   │   ├── order_item.py       # OrderItem -> order (CASCADE), dish (RESTRICT), unit_price snapshot, quantity > 0
│   │   ├── conversations.py    # Conversations (chat_id, thread_id, parent_thread_id, message_count…)
│   │   └── incoming_message.py # Pydantic (not a table): IncomingMessage (platform-agnostic), AgentResponse
│   └── schemas/                # Pydantic I/O models per resource: <Name>Create / <Name>Update / <Name>Read
│       ├── base.py             # ORMModel (from_attributes=True) — base for all *Read
│       └── business.py, menu.py, dish.py, customer.py, order.py, conversation.py
└── tests/
    ├── conftest.py             # `client` fixture: in-memory SQLite (FK pragma on) or Postgres via TEST_DATABASE_URL
    └── test_crud.py            # business, menu/dish, customer uniqueness, order lifecycle, order validation, conversations
```

### Architecture
- **Layered**: `routes` (HTTP + business rules) → `schemas` (Pydantic contracts) → `models` (ORM) → `db` (async session).
  There is no separate service or repository layer. Validation and domain rules live in the route modules
  (e.g. `orders.py::_build_item`, `_check_delivery_address`).
- **Two routers** are mounted in `main.py`: `telegram_router` (`/webhook`) and `api_router` (`/api/v1`).
- **Lifespan**: builds a python-telegram-bot `Application` (`.updater(None)`) and stores it in `app.state.ptb`.
  It registers the webhook at `{WEBHOOK_URL}/webhook/telegram` only if the URL changed, starts PTB, and disposes
  the engine on shutdown.
- **Telegram webhook**: `Update.de_json` → ignore non-text → `IncomingMessage` → `run_agent()` → `send_message(MARKDOWN)`.
  The handler is wrapped in try/except and **always returns 200**, because Telegram retries on any other status.
- **Domain model**: `Businesses 1─* Menu 1─* Dish`, `Businesses 1─* Customer 1─* Order 1─* OrderItem *─1 Dish`.
  Order rules:
  - The dish must belong to the order's business and be available.
  - `unit_price` is copied from `Dish.price` when the item is added.
  - Every item change calls `order.recalculate_total()`.
  - A delivery order without an address takes `customer.default_address`, and must end up with an address
    (enforced in code and by a DB check constraint).
- **The DB owns delete behaviour**: `delete_or_404` uses a Core `DELETE`, so `ON DELETE CASCADE/RESTRICT`
  decides what happens, and RESTRICT violations become **409**.
- **Settings** env vars: `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET`, `LANGGRAPH_API_URL`, `LANGGRAPH_API_KEY`,
  `LANGGRAPH_ASSISTANT_ID`, `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT`, `DB_NAME`, `WEBHOOK_URL`.

### Patterns & conventions
- **Shared CRUD helpers in `routes/_common.py`**. Use them instead of rewriting the same logic:
  - `SessionDep` / `PaginationDep` (`offset` ≥0, `limit` 1–200, default 50) as `Annotated` dependencies
  - `get_or_404(session, Model, id, *options)`
  - `commit_or_409(session)`: IntegrityError → rollback + 409
  - `delete_or_404(session, Model, id)`
  - `apply_updates(obj, payload)`: PATCH semantics via `model_dump(exclude_unset=True)`; an explicit `null` on a NOT NULL column → 422
- **Endpoint shape per resource**: `GET ""` (list, filters as optional query params, ordered), `POST ""` (201),
  `GET /{id}`, `PATCH /{id}` (partial), `DELETE /{id}` (204). Routers: `APIRouter(prefix="/<plural>", tags=["<Plural>"])`.
- **Async rules**: no lazy loading. Eager-load relationships with `selectinload` (see `_load_order`, which also uses
  `populate_existing=True`). Sessions use `expire_on_commit=False`. Re-fetch after commit before returning.
- **Models**:
  - SQLAlchemy 2.0 typed style.
  - `from __future__ import annotations` + `TYPE_CHECKING` imports for relationships.
  - `TimestampMixin` for created/updated, `Money` for currency.
  - Python `str` enums stored by **value** via `_enum(EnumCls, "pg_enum_name")`.
  - Constraints named explicitly (`ck_...`, `uq_...`, `ix_...`).
  - Legacy exceptions: `Businesses` (plural class name, no server-side timestamp defaults, so the route sets `func.now()`)
    and `Conversations`.
- **Schemas**: `XCreate(BaseModel)`, `XUpdate(BaseModel)` with all fields optional, `XRead(ORMModel)`. Export new schemas
  from `src/schemas/__init__.py` and new models from `src/models/__init__.py`; Alembic autogenerate only sees exported models.
- **Config and clients** are cached singletons via `@lru_cache` factories (`get_settings`, `get_engine`, `get_agent_client`).
- **Tests** call the API through the `client` fixture, which overrides the `get_session` dependency. They use SQLite by
  default; set `TEST_DATABASE_URL=postgresql+asyncpg://...` (a migrated, disposable DB) to test against real Postgres.
- Errors use `HTTPException` with a plain-English `detail` (e.g. `"Dish 3 not found"`).

### Adding a new resource (from remy_api CLAUDE.md)
1. Model in `src/models/`, export in `src/models/__init__.py`, then `uv run alembic revision --autogenerate -m "..."`.
2. `Create` / `Update` / `Read` schemas in `src/schemas/`, export in `src/schemas/__init__.py`.
3. Router in `src/api/routes/` using `_common.py` helpers; register it in `src/api/routes/__init__.py`.
4. Tests in `tests/`; run `uv run pytest`.

### Commands
```bash
uv sync
uv run fastapi dev main.py          # dev server with reload (webhook is only re-registered if the URL changed)
uv run alembic upgrade head         # apply migrations (uses DB_* from .env)
uv run alembic revision --autogenerate -m "msg"
uv run pytest                       # SQLite by default; TEST_DATABASE_URL for Postgres
```
`WEBHOOK_URL` must be a public HTTPS URL (e.g. a tunnel) for Telegram to reach `/webhook/telegram`.

---

## Known gaps & gotchas (verify before relying on them)
- **Assistant ID mismatch**: remy_api defaults to `"agent"`, but remy_ai's graph is `"main_graph"` (see above).
- **`TELEGRAM_WEBHOOK_SECRET`** is loaded but never passed to `set_webhook(secret_token=...)` or checked against the
  `X-Telegram-Bot-Api-Secret-Token` header, so anyone can post to the webhook.
- `agent_client._get_thread_id(chat_id: int)` receives a **str** chat_id from `IncomingMessage`; thread metadata stores strings.
- The `Conversations` model types `chat_id` as a UUID (Telegram chat IDs are ints), and the agent flow doesn't use it:
  the thread lookup goes through LangGraph metadata instead.
- Telegram replies use the legacy `ParseMode.MARKDOWN`, so LLM output with unbalanced `*` or `_` can make `send_message` fail.
- `runs.wait` runs inside the webhook request, so a slow agent run can exceed Telegram's webhook timeout.
- remy_ai:
  - `respond_node` ignores `MESSAGE_WRITER_PROMPT` (no system prompt).
  - The model name `gpt-6-luna` is hard-coded in three nodes.
  - RAG retrieval re-creates the Chroma client and embedding model on every call.
  - There are no remy_api tools yet, so order categories are classified but not handled.
- A third project, `~/remyai_webapp` (orca: `~/orca/projects/remyai_webapp`), exists but isn't documented here yet.

---

## Working method: backlog + vertical slices

Postponed or planned work is tracked in this repo, so each session can pick it up without re-deriving context.

- **Backlog:** `docs/backlog.md`. Read it at the start of a session. Each item (`B-1`, `B-2`, …) has a status
  (`todo` · `doing` · `done` · `shelved`), the repo it touches, a link to its plan, and a table of slices.
  Update it whenever a slice changes state, and add new postponed work there in the same format.
- **Priority rule:** get the core flow (Telegram → remy_api → remy_ai) working first, then harden. Don't push
  hardening items (e.g. the webhook secret) ahead of work that blocks the working flow.
- **Vertical slices:** every task is a thin, independently shippable behavior change that cuts through all the layers
  it needs (test → code → config/docs), described by what someone can **observe** when it is done. Don't split work by
  layer ("add the model", then "add the route").
- **One slice at a time:** do one slice, run the tests, and stop for Diego's review before starting the next. Don't
  chain slices unprompted.
- **Plans live in `docs/<topic>/`:** `plan.md` (slices as tasks, TDD steps, global constraints, review focus) and
  `test-plan.md` (the test cases and their status). Example: `docs/webhook-secret/`.
- **Traffic-light test plans:** ⚪ not started, 🔴 failing or blocked, 🟡 passing with limited evidence (fake bot only,
  manual check pending), 🟢 passing with the strongest evidence available. A slice is ready for review only when none of
  its cases are 🔴, and every 🟡 names the check that will turn it 🟢.
- **Credentials:** never handle real secrets (bot token, webhook secret). Prepare the commands and have Diego run the live steps.
- **Commits:** work on a branch off `origin/main` (the local `main` may be behind), commit when asked, and never push or
  open a PR without being asked.

---

## How to act on tasks
1. **Identify the target repo first.** Agent behaviour, prompts, graphs and RAG → **remy_ai**. Endpoints, DB schema,
   migrations and Telegram → **remy_api**. Features that span both (e.g. "the agent can place orders") usually need
   an endpoint in remy_api plus a tool, node or subgraph in remy_ai.
2. Work in the repo's own worktree/branch, branch off `main`, never commit to `main` directly; open a PR.
3. **Follow the existing patterns above**: `_common.py` helpers, Create/Update/Read schemas, graph-as-class, the `NODES`
   registry, prompts in `prompts.py`, partial-state returns.
4. **TDD**: write or extend tests first. In remy_ai, always fake `ChatOpenAI` and retrieval; in remy_api, use the `client` fixture.
   Run `uv run pytest` in the repo you changed before claiming done.
5. Every schema change in remy_api comes with an Alembic migration. Review the autogenerated output (named PG enums,
   check constraints) before committing.
6. Keep the API contract and the agent in sync: when an endpoint or field changes, check remy_ai for callers,
   including uses of `Order.thread_id` and thread metadata.
7. Customer-facing text is Spanish (a Monterrey, MX restaurant, `America/Monterrey` timezone); code, comments and commits are in English.
8. Never commit `.env`, `.chroma/`, or `.langgraph_api/`.
9. Update this file when architecture, structure, or conventions change.
