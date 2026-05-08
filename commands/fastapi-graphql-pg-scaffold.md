# FastAPI + Strawberry GraphQL + PostgreSQL Project Scaffold

You are creating a small **GraphQL** server from scratch in the current working directory.
Stack: **FastAPI + strawberry-graphql[fastapi] + SQLModel ORM + Alembic + PostgreSQL**.
Ships with a **production-grade multistage Dockerfile + docker-compose** so the project is one command away from running.
Follow every step in order. Do not skip any file. All file contents are specified exactly — write them as-is.

---

## Step 1 — Collect Project Details

Use AskUserQuestion to ask the user:

1. **Project name** — human-readable (e.g. "Task Manager API")
2. **Project description** — one sentence describing what the API does

After receiving answers, derive:

- `PROJECT_NAME` = the exact name the user gave
- `PROJECT_DESCRIPTION` = the exact description the user gave
- `PROJECT_SLUG` = lowercase project name, spaces and hyphens replaced with underscores
- `PROJECT_DB_NAME` = `{PROJECT_SLUG}_db`

Confirm once before creating files: show the four derived values and ask the user to confirm or correct.

Also use today's date as `{TODAY}` in `YYYY-MM-DD` format for the `DECISIONS.md` bootstrap entries.

---

## Step 2 — Create Directory Structure

Docker is the primary dev path for this scaffold (the included `docker-compose.yml` runs the whole stack with hot reload).
A local `.venv` is **optional** — only useful for IDE intellisense and running tests outside Docker.

```bash
mkdir -p app/core app/graphql app/modules app/utils alembic/versions tests .claude/commands .claude/skills .claude/agents
touch app/__init__.py app/core/__init__.py app/graphql/__init__.py app/modules/__init__.py
touch app/utils/.gitkeep tests/.gitkeep
```

If the user wants a local venv too:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

---

## Step 3 — Create Application Files

Write every file below. Replace `{PROJECT_NAME}`, `{PROJECT_SLUG}`, `{PROJECT_DESCRIPTION}`, `{PROJECT_DB_NAME}` with the actual values from Step 1.

---

### `app/core/config.py`

```python
from pydantic import computed_field
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    PROJECT_NAME: str = "{PROJECT_NAME}"
    ENVIRONMENT: str = "local"  # "local" | "production"
    DEBUG: bool = True

    # CORS
    BACKEND_CORS_ORIGINS: list[str] = []

    # Security
    ALLOWED_HOSTS: list[str] = ["*"]

    # Database
    DB_SERVER: str = ""
    DB_PORT: int = 5432
    DB_USER: str = ""
    DB_PASSWORD: str = ""
    DB_NAME: str = ""

    @computed_field
    @property
    def DATABASE_URL(self) -> str:
        return (
            f"postgresql://{self.DB_USER}:{self.DB_PASSWORD}"
            f"@{self.DB_SERVER}:{self.DB_PORT}/{self.DB_NAME}"
        )

    class Config:
        env_file = ".env"


settings = Settings()
```

---

### `app/core/database.py`

```python
from collections.abc import Generator
from sqlmodel import Session, create_engine
from .config import settings

engine = create_engine(settings.DATABASE_URL, echo=False, pool_pre_ping=True)


def get_session() -> Generator[Session, None, None]:
    """FastAPI dependency that yields a SQLModel session and ensures cleanup."""
    with Session(engine) as session:
        yield session
```

---

### `app/graphql/context.py`

```python
from typing import Annotated
from fastapi import Depends
from sqlmodel import Session
from strawberry.fastapi import BaseContext
from app.core.database import get_session


class Context(BaseContext):
    """Per-request GraphQL context. Resolvers access session via `info.context.session`."""

    def __init__(self, session: Session):
        super().__init__()
        self.session = session


async def get_context(
    session: Annotated[Session, Depends(get_session)],
) -> Context:
    return Context(session=session)
```

---

### `app/graphql/schema.py`

```python
import strawberry

# Module Query/Mutation classes are merged in here as features are added.
# Until the first module exists, the schema needs at least one field on Query.
#
# Example after adding modules:
#
#     from strawberry.tools import merge_types
#     from app.modules.users.queries import UsersQuery
#     from app.modules.users.mutations import UsersMutation
#     Query = merge_types("Query", (_RootQuery, UsersQuery))
#     Mutation = merge_types("Mutation", (_RootMutation, UsersMutation))


@strawberry.type
class _RootQuery:
    @strawberry.field
    def ping(self) -> str:
        return "pong"


@strawberry.type
class _RootMutation:
    @strawberry.field
    def _placeholder(self) -> str:
        return "ok"


Query = _RootQuery
Mutation = _RootMutation

schema = strawberry.Schema(query=Query, mutation=Mutation)
```

---

### `app/graphql/router.py`

```python
from strawberry.fastapi import GraphQLRouter
from app.core.config import settings
from .context import get_context
from .schema import schema


graphql_router = GraphQLRouter(
    schema,
    context_getter=get_context,
    graphql_ide="graphiql" if settings.ENVIRONMENT != "production" else None,
)
```

---

### `app/main.py`

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from app.core.config import settings
from app.graphql.router import graphql_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    yield


app = FastAPI(
    title=settings.PROJECT_NAME,
    description="{PROJECT_DESCRIPTION}",
    lifespan=lifespan,
    docs_url=None,   # No REST endpoints — GraphiQL playground at /graphql instead
    redoc_url=None,
)

# Middleware order: outermost runs first. Registration is reverse: last add = outermost.
app.add_middleware(GZipMiddleware, minimum_size=1000)
app.add_middleware(
    CORSMiddleware,
    allow_origins=[str(origin).strip("/") for origin in settings.BACKEND_CORS_ORIGINS],
    allow_origin_regex=r"^https?://(localhost|127\.0\.0\.1)(:[0-9]+)?$",
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
app.add_middleware(TrustedHostMiddleware, allowed_hosts=settings.ALLOWED_HOSTS)

app.include_router(graphql_router, prefix="/graphql")


@app.get("/health")
def health():
    return {"status": "ok", "app": settings.PROJECT_NAME}
```

---

### `alembic/env.py`

```python
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context
from sqlmodel import SQLModel
from app.core.config import settings

config = context.config
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = SQLModel.metadata

# Import all models here so Alembic can detect schema changes:
# from app.modules.users.models import User  # noqa
# from app.modules.tasks.models import Task  # noqa


def run_migrations_offline() -> None:
    context.configure(
        url=settings.DATABASE_URL,
        target_metadata=target_metadata,
        literal_binds=True,
    )
    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    with connectable.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata)
        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

---

### `alembic.ini`

```ini
[alembic]
script_location = alembic
prepend_sys_path = .
version_path_separator = os

sqlalchemy.url =

[post_write_hooks]

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

---

## Step 4 — Create `requirements.txt`

```text
fastapi[standard]>=0.131.0
uvicorn>=0.38.0
gunicorn>=23.0.0
pydantic-settings>=2.9.0
sqlmodel>=0.0.27
psycopg2-binary>=2.9.10
alembic>=1.17.0
strawberry-graphql[fastapi]>=0.260.0
httpx>=0.28.0
python-multipart>=0.0.20
```

---

## Step 5 — Create `.env.example`, `.gitignore`, and `README.md`

### `.env.example`

```text
PROJECT_NAME={PROJECT_NAME}
ENVIRONMENT=local
DEBUG=true
BACKEND_CORS_ORIGINS=[]
ALLOWED_HOSTS=["localhost","127.0.0.1"]

# Database — `DB_SERVER=postgres` is correct when running via docker-compose
# (`postgres` is the service name). Change to `localhost` for local-only dev.
DB_SERVER=postgres
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME={PROJECT_DB_NAME}

# Add when JWT auth is added:
# SECRET_KEY=
# ALGORITHM=HS256
# ACCESS_TOKEN_EXPIRE_MINUTES=30
# REFRESH_TOKEN_EXPIRE_DAYS=7
```

### `.gitignore`

```text
.env
*.pyc
__pycache__/
.pytest_cache/
.mypy_cache/
.ruff_cache/
.venv/
env/
dist/
*.egg-info/
.coverage
htmlcov/
.idea/
.vscode/
*.log
```

### `README.md`

```markdown
# {PROJECT_NAME}

{PROJECT_DESCRIPTION}

## Tech Stack

- FastAPI + Strawberry GraphQL (single `/graphql` endpoint)
- PostgreSQL + SQLModel (ORM) + Alembic (migrations)
- Production-grade multistage Dockerfile + docker-compose

## Quickstart (Docker — primary path)

```bash
cp .env.example .env                    # adjust DB credentials if needed
docker compose up --build
docker compose exec app alembic upgrade head   # after first model is added
```

- GraphiQL playground: http://localhost:8000/graphql
- Health check: http://localhost:8000/health

The app service mounts `./app` and `./alembic` so code edits hot-reload via uvicorn.
Postgres data persists in the named volume `postgres_data`.

## Quickstart (local Python — optional)

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
# Point DB_SERVER=localhost in .env, run a local postgres
alembic upgrade head
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

## Project Docs

- **`ARCHITECTURE.md`** — how the project works and how components connect. Updated whenever structure changes.
- **`DECISIONS.md`** — append-only log of *why* each non-trivial choice was made.
- **`.claude/CLAUDE.md`** — architecture conventions, patterns, and development workflow.
```

---

## Step 6 — Create `DECISIONS.md` and `ARCHITECTURE.md`

Use today's date (`{TODAY}` = `YYYY-MM-DD`) for every bootstrap entry.

---

### `DECISIONS.md`

```markdown
# Decisions

Append-only log of every non-trivial decision made on this project.
Each entry explains *why* a choice was made — and why alternatives were not.

## When to add an entry

- Adding or removing a dependency
- Choosing one library / tool over alternatives
- Introducing or changing an architectural pattern
- Picking an approach where a sensible alternative existed (e.g. JWT vs sessions)
- Setting a non-obvious convention (e.g. all timestamps stored as UTC)

## When NOT to add an entry

- Mechanical scaffolding (new module, new query/mutation, new model class)
- Bug fixes
- Renames, formatting, cleanup

## Entry format

```
## {YYYY-MM-DD} — {Short title}
**Context:** What triggered the decision.
**Decision:** What we chose.
**Alternatives considered:** What else was on the table.
**Why this:** Why the chosen option won.
**Why not others:** Concrete reasons each alternative was rejected.
**Trade-offs accepted:** What we knowingly gave up.
```

---

## {TODAY} — Use FastAPI as the web framework
**Context:** Choosing the Python web framework that hosts the GraphQL endpoint.
**Decision:** FastAPI.
**Alternatives considered:** Django, Flask, Starlette directly.
**Why this:** Native ASGI/async, excellent middleware story, pydantic-settings + pydantic-core integration, low overhead. Strawberry has first-class FastAPI integration via `GraphQLRouter`.
**Why not others:** Django is heavier and its sync-first ORM doesn't fit our async story; Flask lacks built-in async; Starlette is too low-level for typical needs.
**Trade-offs accepted:** Smaller built-in admin/auth story than Django — we add what we need.

## {TODAY} — Use GraphQL (not REST) as the API surface
**Context:** Choosing API style for the project.
**Decision:** GraphQL only — single `/graphql` endpoint.
**Alternatives considered:** REST (FastAPI's native style), gRPC, REST + GraphQL hybrid.
**Why this:** Clients fetch exactly the fields they need (no over/under-fetching); schema is the contract; introspection makes tooling and codegen trivial; mutations + subscriptions in one place.
**Why not REST:** With many related entities, REST endpoints multiply and clients end up making N+1 requests; we'd reinvent field selection via query params.
**Why not gRPC:** Web-client friction, requires extra proxy for browsers, less mature schema-first tooling than GraphQL.
**Why not hybrid:** Two API styles to maintain, two auth flows, doubled documentation surface.
**Trade-offs accepted:** Caching is harder than REST (no per-resource HTTP cache keys); rate-limiting requires query-cost analysis; ad-hoc tooling assumes REST.

## {TODAY} — Use Strawberry as the GraphQL library
**Context:** Choosing a Python GraphQL library.
**Decision:** strawberry-graphql with the [fastapi] extra.
**Alternatives considered:** Ariadne, Graphene, Tartiflette.
**Why this:** Code-first schema using Python dataclasses + type hints — same shape as our SQLModel/Pydantic models. Native FastAPI integration. Active maintenance. Async resolvers are first-class.
**Why not others:** Graphene has a class-heavy API and slower release cadence; Ariadne is schema-first which doubles up on type definitions; Tartiflette is sparsely maintained.
**Trade-offs accepted:** Schema is generated from Python types — to share it with non-Python clients, run a schema-export step.

## {TODAY} — Use PostgreSQL as the database
**Context:** Choosing the relational store.
**Decision:** PostgreSQL.
**Alternatives considered:** MySQL, SQLite, MongoDB.
**Why this:** Mature, strong consistency, rich JSONB support, excellent extension ecosystem (pgvector, PostGIS), proven at scale.
**Why not others:** MySQL has weaker JSON and constraint stories; SQLite doesn't fit multi-process production; MongoDB is non-relational and we want strong schemas + transactions.
**Trade-offs accepted:** Operational overhead vs SQLite — needs a managed service or self-host (handled by docker-compose locally).

## {TODAY} — Use SQLModel as the ORM
**Context:** Need an ORM that integrates with SQLAlchemy power and Pydantic types.
**Decision:** SQLModel.
**Alternatives considered:** Plain SQLAlchemy, Tortoise ORM, Peewee.
**Why this:** Built on SQLAlchemy core, so its full query power is still available. Pydantic-compatible model classes are easy to map into Strawberry types via small `from_model` helpers.
**Why not others:** Plain SQLAlchemy forces a separate Pydantic schema layer; Tortoise has a smaller ecosystem; Peewee is too minimal.
**Trade-offs accepted:** Smaller community than SQLAlchemy; some advanced SQLAlchemy patterns are awkward.

## {TODAY} — Use Alembic for schema migrations
**Context:** Need versioned schema changes.
**Decision:** Alembic.
**Alternatives considered:** Hand-written SQL migrations, third-party SQLModel-specific tools.
**Why this:** Industry standard for SQLAlchemy-based projects, autogenerate works with SQLModel, mature.
**Why not others:** Hand-written SQL is error-prone and lacks autogenerate.
**Trade-offs accepted:** Every model must be imported in `alembic/env.py` for autogenerate to detect it.

## {TODAY} — Modular schema split: `types.py` / `queries.py` / `mutations.py` per module
**Context:** Where to put GraphQL type definitions and resolvers.
**Decision:** Each `app/modules/{feature}/` directory owns its own `types.py`, `queries.py`, `mutations.py`. A central `app/graphql/schema.py` merges the per-module Query/Mutation classes via `strawberry.tools.merge_types`.
**Alternatives considered:** One giant `schema.py`; one `schema.py` per module that defines its own `strawberry.Schema`; co-locating types and resolvers in a single `graphql.py` per module.
**Why this:** Small files, clear ownership, modules can be reviewed independently. The merge happens in one place so adding a feature is a one-line edit to `schema.py`.
**Why not others:** A single big file becomes unreadable past a few features; per-module schemas can't share context easily; co-locating types and resolvers makes resolver files long.
**Trade-offs accepted:** Three files per module instead of one — but each stays small.

## {TODAY} — DB session passed via `info.context.session`
**Context:** How resolvers access the database.
**Decision:** `Context` (subclass of `BaseContext`) holds a per-request `Session`. Built by `get_context` which depends on `get_session`. Resolvers read it via `info.context.session` and pass it to service methods.
**Alternatives considered:** Module-level session, contextvars, querying the FastAPI dep system directly inside resolvers.
**Why this:** Same lifecycle guarantees as the FastAPI dependency system, no implicit globals, easy to override in tests via `app.dependency_overrides[get_session]`.
**Why not others:** Module-level state leaks across requests; contextvars are harder to test and reason about.
**Trade-offs accepted:** Resolvers must accept `info: Info` as their first argument.

## {TODAY} — Resolvers contain no business logic; services do the work
**Context:** Where domain logic lives.
**Decision:** Resolvers only call services and adapt return types. All business logic, DB access, and validation live in `service.py` as static methods on a `{Name}Service` class.
**Alternatives considered:** Logic in resolvers; class-based resolver views; a third "use case" layer.
**Why this:** Resolvers stay one-liners and easy to read; services are testable without spinning up Strawberry; the same services are reusable from CLI commands, scheduled jobs, or background workers.
**Why not others:** Logic in resolvers is hard to test; an extra layer is over-engineering for this project's size.
**Trade-offs accepted:** Two files per feature path.

## {TODAY} — Production Dockerfile uses multistage build with uv
**Context:** Building the production image.
**Decision:** Stage 1 (`builder`) uses `ghcr.io/astral-sh/uv:...-bookworm-slim`, creates a venv at `/opt/venv`, installs deps via `uv pip install -r requirements.txt` with a buildkit cache mount. Stage 2 (`runtime`) is `python:3.13-slim-bookworm`, copies only `/opt/venv` and the app, runs as a non-root `app` user, healthcheck via Python (no `curl` install).
**Alternatives considered:** Single-stage with pip; alpine base; distroless final stage; poetry instead of uv.
**Why this:** uv installs roughly 10× faster than pip, so cold builds are quick. Multistage keeps the runtime image free of build tools and pip caches → smaller, fewer CVEs. `slim-bookworm` is glibc-based so wheels Just Work (no musl compatibility headaches like alpine).
**Why not alpine:** `psycopg2-binary` and many wheels have known musl issues; install often falls back to compiling from source which makes the image bigger and slower to build.
**Why not distroless:** Harder to debug — no shell, no apt. We can revisit when the project is more mature.
**Why not pip:** Slower cold builds, no built-in lock-file resolver of the same quality as uv's.
**Trade-offs accepted:** Pinned uv major version means occasional bumps when uv ships breaking changes.

## {TODAY} — docker-compose runs the production image with dev-friendly overrides
**Context:** How local development happens.
**Decision:** A single `docker-compose.yml` builds and runs the production Dockerfile, then overrides the `command:` to `uvicorn ... --reload` and volume-mounts `./app` and `./alembic` for hot reload. Postgres runs as a sibling service with a healthcheck and `depends_on: condition: service_healthy`.
**Alternatives considered:** Separate `Dockerfile.dev`; two compose files (`compose.yml` + `compose.override.yml`); no compose at all.
**Why this:** One image, one compose file — the user runs `docker compose up --build` and has a working app + DB. Mixes dev and prod surface slightly, but the prod image is unmodified — only the runtime command and source mounts change.
**Why not separate Dockerfile.dev:** Two images to maintain; harder to ensure dev parity with prod.
**Why not split compose files:** Extra cognitive load for a small project; the override pattern shines once compose grows beyond 3-4 services.
**Trade-offs accepted:** The dev compose runs `uvicorn --reload`, which is single-process — good enough for local development; production uses gunicorn workers via the Dockerfile CMD.
```

---

### `ARCHITECTURE.md`

```markdown
# Architecture

How {PROJECT_NAME} is built and how its components connect.

This document is updated every time the project's structure or component
connections change — new module, new external integration, new layer.

---

## Overview

{PROJECT_DESCRIPTION}

A small **GraphQL** server. Single `/graphql` endpoint backed by Strawberry, hosted by FastAPI.
Each feature module owns its own GraphQL types and resolvers; a central schema file merges
them. Resolvers contain no business logic — they call into module services.

---

## Tech Stack

| Layer | Technology |
| --- | --- |
| Web framework | FastAPI |
| GraphQL library | strawberry-graphql[fastapi] |
| Database | PostgreSQL |
| ORM | SQLModel |
| Migrations | Alembic |
| Settings | pydantic-settings |
| Container | multistage Docker (uv builder + python:3.13-slim-bookworm runtime) |
| Local orchestration | docker-compose |

See `DECISIONS.md` for *why* each of these was chosen.

---

## Folder Layout

```
app/
├── core/
│   ├── config.py           # Settings (loaded from .env via pydantic-settings)
│   └── database.py         # SQLAlchemy engine + get_session() dependency
├── graphql/
│   ├── schema.py           # Merges per-module Query/Mutation into the root schema
│   ├── context.py          # Per-request GraphQL context (holds Session)
│   └── router.py           # Strawberry GraphQLRouter mounted at /graphql
├── modules/                # One folder per domain feature
│   └── {feature}/
│       ├── types.py        # @strawberry.type / @strawberry.input definitions
│       ├── queries.py      # @strawberry.type Query with @strawberry.field resolvers
│       ├── mutations.py    # @strawberry.type Mutation with @strawberry.mutation resolvers
│       ├── service.py      # Business logic, DB ops via injected Session
│       └── models.py       # SQLModel table classes (if the feature owns DB tables)
├── utils/                  # Cross-cutting helpers (currently empty)
└── main.py                 # App entry: middleware, GraphQL mount, /health

alembic/
├── env.py                  # Wired to settings.DATABASE_URL + SQLModel.metadata
└── versions/               # Generated migration files

Dockerfile                  # Multistage production image
docker-compose.yml          # postgres + app (dev hot-reload)
.dockerignore               # Excludes everything not needed in the image
```

---

## Component Map

```
┌─────────────────────┐
│  Client (HTTP POST  │
│  to /graphql)       │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Middleware stack    │  TrustedHost → CORS → GZip
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ GraphQLRouter       │  /graphql (POST queries+mutations, GET GraphiQL in dev)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Strawberry Schema   │  Merged Query / Mutation across all modules
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐       ┌─────────────────────┐
│ Module resolver     │ ────▶ │ Module service      │
│ (queries.py /       │       │ (static methods)    │
│  mutations.py)      │       └──────────┬──────────┘
└──────────┬──────────┘                  │
           │                             ▼
           │                  ┌─────────────────────┐
           │                  │ SQLModel ORM        │
           │                  │ (info.context       │
           │                  │  .session)          │
           │                  └──────────┬──────────┘
           │                             │
           │                             ▼
           │                  ┌─────────────────────┐
           │                  │   PostgreSQL        │
           │                  └─────────────────────┘
           │
           ▼
┌─────────────────────┐
│ GraphQL response    │  Serialized by Strawberry
│ (data / errors)     │
└─────────────────────┘
```

---

## Request Flow

1. Client POSTs a GraphQL document to `/graphql`.
2. Middleware stack runs **outer → inner**: TrustedHost → CORS → GZip.
3. `GraphQLRouter` parses the request and runs `get_context(...)`, which depends on `get_session()` to produce a fresh `Session`.
4. Strawberry resolves the operation: each `@strawberry.field` / `@strawberry.mutation` runs its resolver with `info.context.session` available.
5. The resolver calls a service method (`{Name}Service.{action}(session, ...)`).
6. The service does DB work via SQLModel (`session.add`, `session.exec(select(...))`, `session.get`) and returns ORM objects (or raises `Exception` on error — Strawberry surfaces it as a GraphQL `errors[]` entry).
7. The resolver maps ORM objects to Strawberry types (`Item.from_model(...)`) and returns them.
8. Strawberry serializes the response; FastAPI sends it back through the middleware stack.

---

## Database

- **Engine:** Single SQLAlchemy `create_engine(...)` with `pool_pre_ping=True`.
- **Sessions:** `get_session()` yields a fresh `Session` per request.
- **Schema management:** Alembic. After model changes: `docker compose exec app alembic revision --autogenerate -m "..."` then `docker compose exec app alembic upgrade head`.
- **Model registration:** Every SQLModel class with `table=True` must be imported in `alembic/env.py`.

---

## Configuration

All config flows through `app.core.config.settings` (pydantic-settings `BaseSettings`), loaded from `.env`. **Never read `os.environ` directly.** When running via docker-compose, `DB_SERVER=postgres` (the service name); locally, `DB_SERVER=localhost`.

---

## Container Architecture

- **Builder stage:** `ghcr.io/astral-sh/uv:...-bookworm-slim` creates `/opt/venv` and runs `uv pip install -r requirements.txt`.
- **Runtime stage:** `python:3.13-slim-bookworm`, non-root `app` user, copies only `/opt/venv` and source. No build tools, no pip cache, no apt cache.
- **Healthcheck:** Python `urllib.request` against `/health` — no `curl` install needed.
- **Default CMD:** `gunicorn` with uvicorn workers (production). The compose file overrides with `uvicorn --reload` for dev.

---

## Modules

_(none yet — each `add-module` invocation appends an entry here)_

---

## External Integrations

_(none yet — log here when adding Redis, S3, message queues, third-party APIs, etc.)_

---

## Background Workers / Async Jobs

_(none yet — log here when adding Celery, RQ, ARQ, or similar)_
```

---

## Step 7 — Create `Dockerfile`, `docker-compose.yml`, `.dockerignore`

These three files are the heart of this scaffold. They run together — do not modify them piecemeal.

---

### `Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1.7

ARG PYTHON_VERSION=3.13
ARG UV_VERSION=0.5

############################
# Stage 1 — builder
############################
FROM ghcr.io/astral-sh/uv:${UV_VERSION}-python${PYTHON_VERSION}-bookworm-slim AS builder

ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never

WORKDIR /app

COPY requirements.txt .

RUN --mount=type=cache,target=/root/.cache/uv \
    uv venv /opt/venv && \
    uv pip install --python /opt/venv/bin/python -r requirements.txt

############################
# Stage 2 — runtime
############################
FROM python:${PYTHON_VERSION}-slim-bookworm AS runtime

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PATH=/opt/venv/bin:$PATH

RUN groupadd --system app && \
    useradd --system --gid app --home-dir /app --shell /sbin/nologin app

WORKDIR /app

COPY --from=builder /opt/venv /opt/venv
COPY --chown=app:app . .

USER app

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://localhost:8000/health').status == 200 else 1)"

CMD ["gunicorn", "app.main:app", \
     "-w", "4", \
     "-k", "uvicorn.workers.UvicornWorker", \
     "--bind", "0.0.0.0:8000", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

---

### `docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:17-alpine
    container_name: {PROJECT_SLUG}_postgres
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    ports:
      - "${DB_PORT}:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: {PROJECT_SLUG}_app
    env_file: .env
    environment:
      DB_SERVER: postgres
    ports:
      - "8000:8000"
    volumes:
      - ./app:/app/app
      - ./alembic:/app/alembic
      - ./alembic.ini:/app/alembic.ini
    depends_on:
      postgres:
        condition: service_healthy
    command: ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
    restart: unless-stopped

volumes:
  postgres_data:
```

Notes for the user:

- No `version:` key — Compose v2+ doesn't need it.
- The `command:` in the app service overrides the production `CMD` from the Dockerfile so dev gets `--reload`. Strip this block (or remove the `volumes:` mounts for `./app`) for production.
- `$$POSTGRES_USER` is escaped so Compose passes the literal `$POSTGRES_USER` to the container shell.

---

### `.dockerignore`

```text
# VCS
.git
.gitignore
.gitattributes

# Python artifacts
__pycache__
*.py[cod]
*$py.class
*.so
.Python
.venv
venv
env
.pytest_cache
.mypy_cache
.ruff_cache
*.egg-info

# Environment
.env
.env.*
!.env.example

# IDE / OS
.vscode
.idea
*.swp
.DS_Store

# Build / coverage
build
dist
*.egg
.coverage
htmlcov

# Docs / scaffolding (not needed inside the image)
README.md
DECISIONS.md
ARCHITECTURE.md
PROJECT_BLUEPRINT.json
.claude
docs

# Tests
tests

# Logs
*.log

# Docker (not needed inside the image)
Dockerfile*
docker-compose*.yml
compose.yml
.dockerignore

# CI
.github
.gitlab-ci.yml
```

---

## Step 8 — Create `.claude/CLAUDE.md`

```markdown
# CLAUDE.md — {PROJECT_NAME}

{PROJECT_DESCRIPTION}

GraphQL server: FastAPI hosts a single `/graphql` endpoint backed by Strawberry. SQLModel
ORM + Alembic for the data layer. No REST endpoints, no response wrappers, no custom
exception handlers — resolvers return Strawberry types directly and exceptions surface
as GraphQL `errors[]`.

---

## Tech Stack

| Layer | Technology |
| --- | --- |
| Framework | FastAPI |
| GraphQL | strawberry-graphql[fastapi] |
| Database | PostgreSQL |
| ORM / Migrations | SQLModel + Alembic |
| Settings | pydantic-settings |
| Container | multistage Docker + docker-compose |

---

## Critical Rules

1. **All DB access via `info.context.session`** in resolvers; pass it to the service.
2. **Use SQLModel ORM**, not raw SQL. Last resort: `session.exec(text("..."))` from `sqlalchemy`.
3. **Never use `os.environ`.** All config via `from app.core.config import settings`.
4. **Never put logic in a resolver.** Resolvers call services and map types — that's it.
5. **Resolvers raise standard exceptions** for errors; Strawberry surfaces them in `errors[]`. Use `strawberry.exceptions.GraphQLError` for client-facing messages.
6. **Each module owns three GraphQL files**: `types.py`, `queries.py`, `mutations.py`. Plus `service.py` and (if it owns tables) `models.py`.
7. **Register module Query/Mutation in `app/graphql/schema.py`** via `merge_types`. Forgetting this means the resolver never gets called.
8. **Never include `password` in any GraphQL type.** Strawberry types are publicly exposed via introspection.
9. **Use `strawberry.ID` for primary key fields** in types — not `str`. Inputs use `str` or `strawberry.ID` depending on whether the value is meaningful as an ID.
10. **Always commit explicitly** in services. After `session.add(...)`, call `session.commit()` and `session.refresh(obj)` if the resolver needs the populated row back.
11. **Append to `DECISIONS.md` whenever a non-trivial decision is made.** Triggers: adding/removing a dependency, choosing one library/tool over alternatives, introducing or changing an architectural pattern, picking an approach where a sensible alternative existed, setting a non-obvious convention. Do NOT log: mechanical scaffolding (new module / resolver / model), bug fixes, renames, cleanup. Use the `record-decision` skill or write the entry directly using the format at the top of `DECISIONS.md`.
12. **Update `ARCHITECTURE.md` whenever the project's structure or component connections change.** Triggers: adding/removing a module or service, adding an external integration, changing how components communicate, adding a new layer (worker, cache, gateway). Use the `update-architecture` skill or edit the relevant section directly.

---

## Type Pattern (`types.py`)

```python
import strawberry
from typing import Optional


@strawberry.type
class Item:
    id: strawberry.ID
    title: str
    description: Optional[str] = None

    @classmethod
    def from_model(cls, m) -> "Item":
        return cls(id=strawberry.ID(m.id), title=m.title, description=m.description)


@strawberry.input
class ItemCreateInput:
    title: str
    description: Optional[str] = None


@strawberry.input
class ItemUpdateInput:
    id: strawberry.ID
    title: Optional[str] = None
    description: Optional[str] = None
```

## Query Pattern (`queries.py`)

```python
import strawberry
from strawberry.types import Info
from .service import ItemService
from .types import Item


@strawberry.type
class ItemsQuery:
    @strawberry.field
    def items(self, info: Info) -> list[Item]:
        rows = ItemService.list_items(info.context.session)
        return [Item.from_model(r) for r in rows]

    @strawberry.field
    def item(self, info: Info, id: strawberry.ID) -> Item:
        row = ItemService.get_item(info.context.session, str(id))
        return Item.from_model(row)
```

## Mutation Pattern (`mutations.py`)

```python
import strawberry
from strawberry.types import Info
from .service import ItemService
from .types import Item, ItemCreateInput, ItemUpdateInput


@strawberry.type
class ItemsMutation:
    @strawberry.mutation
    def create_item(self, info: Info, input: ItemCreateInput) -> Item:
        row = ItemService.create_item(info.context.session, input)
        return Item.from_model(row)

    @strawberry.mutation
    def update_item(self, info: Info, input: ItemUpdateInput) -> Item:
        row = ItemService.update_item(info.context.session, str(input.id), input)
        return Item.from_model(row)

    @strawberry.mutation
    def delete_item(self, info: Info, id: strawberry.ID) -> bool:
        ItemService.delete_item(info.context.session, str(id))
        return True
```

## Service Pattern (`service.py`)

```python
from sqlmodel import Session, select
from strawberry.exceptions import GraphQLError
from .models import Item


class ItemService:
    @staticmethod
    def create_item(session: Session, payload) -> Item:
        item = Item(title=payload.title, description=payload.description)
        session.add(item)
        session.commit()
        session.refresh(item)
        return item

    @staticmethod
    def list_items(session: Session, skip: int = 0, limit: int = 50) -> list[Item]:
        return session.exec(select(Item).offset(skip).limit(limit)).all()

    @staticmethod
    def get_item(session: Session, item_id: str) -> Item:
        item = session.get(Item, item_id)
        if not item:
            raise GraphQLError("Item not found")
        return item

    @staticmethod
    def update_item(session: Session, item_id: str, payload) -> Item:
        item = ItemService.get_item(session, item_id)
        if payload.title is not None:
            item.title = payload.title
        if payload.description is not None:
            item.description = payload.description
        session.add(item)
        session.commit()
        session.refresh(item)
        return item

    @staticmethod
    def delete_item(session: Session, item_id: str) -> None:
        item = ItemService.get_item(session, item_id)
        session.delete(item)
        session.commit()
```

## Schema Wiring (`app/graphql/schema.py`)

When you add a module's query/mutation classes, merge them in:

```python
import strawberry
from strawberry.tools import merge_types
from app.modules.items.queries import ItemsQuery
from app.modules.items.mutations import ItemsMutation


@strawberry.type
class _RootQuery:
    @strawberry.field
    def ping(self) -> str: return "pong"


@strawberry.type
class _RootMutation:
    @strawberry.field
    def _placeholder(self) -> str: return "ok"


Query = merge_types("Query", (_RootQuery, ItemsQuery))
Mutation = merge_types("Mutation", (_RootMutation, ItemsMutation))

schema = strawberry.Schema(query=Query, mutation=Mutation)
```

---

## Adding a New Feature

**Skills (auto-trigger on natural language):**

- "add a `{feature}` module" / "scaffold a feature module" → `add-module` skill
- "add a `{Name}` model" / "create a SQLModel" → `add-model` skill
- "plan the `{feature}` feature" → `plan-feature` skill
- "we decided to use X" / "switching to Y" → `record-decision` skill
- "added a Redis layer" / "new module just landed" → `update-architecture` skill

**Slash commands (explicit invocation):**

- `/add-resolver {module} {query|mutation} {name} {description}` — add a single resolver
- `/check-conventions [file]` — review against this CLAUDE.md

**Manually:**

1. Create `app/modules/{feature}/types.py`, `queries.py`, `mutations.py`, `service.py`, and `models.py` (if needed)
2. Register Query and Mutation in `app/graphql/schema.py` via `merge_types`
3. If models added: import into `alembic/env.py`, then `docker compose exec app alembic revision --autogenerate -m "..."` then `alembic upgrade head`
4. Test via the GraphiQL playground at `/graphql`

---

## Commands

```bash
# Build + start the whole stack (postgres + app)
docker compose up --build

# Apply migrations
docker compose exec app alembic upgrade head

# New migration
docker compose exec app alembic revision --autogenerate -m "description"

# Tail logs
docker compose logs -f app

# Tear down (keep volume)
docker compose down

# Tear down + delete postgres volume
docker compose down -v

# Run tests inside the container
docker compose exec app pytest -v
```

## Skills (auto-triggered)

| Skill | Trigger phrases | What it does |
| --- | --- | --- |
| `add-module` | "add a tasks module" | Full module skeleton (types/queries/mutations/service) + schema merge |
| `add-model` | "add a Task model" | SQLModel class + Alembic migration |
| `plan-feature` | "plan the auth feature" | Brainstorm + blueprint via `@feature-planner` |
| `record-decision` | "we decided to use X" | Append a new entry to DECISIONS.md |
| `update-architecture` | "new Redis layer added" | Update the relevant section of ARCHITECTURE.md |

## Slash Commands

| Command | Example | What It Does |
| --- | --- | --- |
| `/add-resolver` | `/add-resolver tasks query exportTasks Export tasks as CSV` | Add a single query or mutation to an existing module |
| `/check-conventions` | `/check-conventions app/modules/tasks/queries.py` | Check file for violations |

## Agents

| Agent | Purpose |
| --- | --- |
| `@feature-planner` | Brainstorm + blueprint new features (read-only) |
| `@project-extender` | File-by-file implementation plan from PROJECT_BLUEPRINT.json |
| `@convention-reviewer` | Report convention violations with line numbers |
```

---

## Step 9 — Create `PROJECT_BLUEPRINT.json`

```json
{
  "project_name": "{PROJECT_NAME}",
  "project_slug": "{PROJECT_SLUG}",
  "description": "{PROJECT_DESCRIPTION}",
  "tech_stack": {
    "framework": "FastAPI",
    "graphql": "strawberry-graphql[fastapi]",
    "database": "PostgreSQL",
    "orm": "SQLModel",
    "migrations": "Alembic",
    "container": "multistage Docker (uv builder + python:3.13-slim-bookworm runtime)"
  },
  "conventions": {
    "api_style": "GraphQL only — single /graphql endpoint, no REST",
    "module_layout": "types.py / queries.py / mutations.py / service.py / models.py",
    "schema_assembly": "strawberry.tools.merge_types in app/graphql/schema.py",
    "db_access": "info.context.session passed into service methods",
    "exceptions": "raise strawberry.exceptions.GraphQLError for client-facing errors",
    "config": "from app.core.config import settings (never os.environ)"
  },
  "modules": [],
  "notes": "Feed this file to @project-extender when planning new features."
}
```

---

## Step 10 — Create Project Skills

Skills live in `.claude/skills/{name}/SKILL.md` and auto-trigger when the user's message matches the skill's `description`.

---

### `.claude/skills/add-module/SKILL.md`

```markdown
---
name: add-module
description: Scaffold a complete GraphQL feature module — types.py, queries.py, mutations.py, service.py — and register its Query/Mutation in app/graphql/schema.py. Trigger when the user asks to "add a module", "scaffold a feature module", "create a new module called X", or similar phrasing for spinning up a fresh feature.
---

# Add Module — GraphQL Feature Module

Create a complete feature module from scratch following the patterns in `.claude/CLAUDE.md`.

## Steps

1. If the user did not provide a module name, ask for one with AskUserQuestion.

2. Create:
   - `app/modules/{name}/__init__.py` — empty
   - `app/modules/{name}/types.py` — `{Name}` (output type, with `from_model`), `{Name}CreateInput`, `{Name}UpdateInput`
   - `app/modules/{name}/queries.py` — `{Name}sQuery` with `{name}s` (list) and `{name}` (by id) fields
   - `app/modules/{name}/mutations.py` — `{Name}sMutation` with `create_{name}`, `update_{name}`, `delete_{name}`
   - `app/modules/{name}/service.py` — `{Name}Service` with one static method per resolver

3. Edit `app/graphql/schema.py`:
   - Import the module's Query and Mutation classes
   - Add them to the `merge_types(...)` tuples for `Query` and `Mutation`

4. Confirm all files exist and the schema includes the new module.

5. Append a line to the **Modules** section of `ARCHITECTURE.md` describing the new module's purpose.

6. Remind the user:
   - New DB tables: ask the assistant to "add a `{Name}` model" (triggers `add-model` skill), then `docker compose exec app alembic upgrade head`
   - Exercise the resolvers in the GraphiQL playground at `/graphql`
```

---

### `.claude/skills/add-model/SKILL.md`

```markdown
---
name: add-model
description: Create a SQLModel table class plus its Alembic migration. Trigger when the user asks to "add a model", "create a new SQLModel", "add a {Name} table", or otherwise wants a new database table for the project.
---

# Add Model — SQLModel + Migration

Create a SQLModel table class and generate its Alembic migration.

## Steps

1. Scan `app/modules/` to list existing module directories. Use AskUserQuestion to ask:

   - **Model name** — the class name in singular form (e.g. `Task`, `UserProfile`). Pre-fill from any name in the user's message.
   - **Module** — which module this model belongs to. Show discovered modules + an "Other (new module)" option.

2. Derive:
   - `MODEL_NAME` = PascalCase of the answer (e.g. `task` → `Task`)
   - `MODULE` = chosen module directory
   - `table_name` = snake_case plural (e.g. `tasks`)

3. Ask for fields (optional):
   > Define fields now? Format: `field_name:type` per line or comma-separated.
   > Supported: `str`, `int`, `float`, `bool`, `datetime`, `Optional[str]`, etc.

   Map each to the SQLModel column form (`Optional[str] = Field(default=None)`, `bool = Field(default=False)`, etc.).

4. Create `app/modules/{MODULE}/models.py`:

```python
from typing import Optional
from uuid import uuid4
from sqlmodel import SQLModel, Field
from datetime import datetime


class {MODEL_NAME}(SQLModel, table=True):
    __tablename__ = "{table_name}"

    id: str = Field(default_factory=lambda: uuid4().hex, primary_key=True)
    {fields}
    created_at: datetime = Field(default_factory=datetime.utcnow)
```

5. Import the model in `alembic/env.py`:

```python
from app.modules.{MODULE}.models import {MODEL_NAME}  # noqa
```

6. Generate the migration:

```bash
docker compose exec app alembic revision --autogenerate -m "add_{table_name}_table"
```

7. Read the generated migration file and confirm `upgrade()` and `downgrade()` look correct.

8. Remind the user: `docker compose exec app alembic upgrade head` to apply.

## If the Migration Is Empty

- Is the model imported in `alembic/env.py`?
- Does the model class have `table=True`?

## Rules

- SQLModel class must have `table=True`
- Never edit a migration that has already been applied to production
- `downgrade()` must undo exactly what `upgrade()` does
```

---

### `.claude/skills/plan-feature/SKILL.md`

```markdown
---
name: plan-feature
description: Brainstorm and design a new feature before any code is written. Trigger when the user asks to "plan a feature", "design X", "think through Y before coding", or wants to explore approaches and trade-offs.
---

# Plan Feature

Delegate to the `@feature-planner` agent with the full feature description plus relevant context from the conversation (existing modules, constraints, user preferences).
```

---

### `.claude/skills/record-decision/SKILL.md`

```markdown
---
name: record-decision
description: Append an entry to DECISIONS.md when a project decision is made. Trigger when the user says "we decided", "let's use X", "switching to Y", "going with Z over W", introduces a new dependency, picks a library, sets a non-obvious convention, or otherwise makes a non-trivial choice. Skip mechanical scaffolding (new module, new resolver, bug fix).
---

# Record Decision

Append a new entry to `DECISIONS.md` using the format documented at the top of that file.

## Steps

1. Read `DECISIONS.md` to confirm the format and that no existing entry already covers this decision (if one does, update or replace it instead of duplicating).

2. Gather these from the conversation, asking the user only for what's missing:
   - **Title** — short, e.g. "Add Redis as the cache layer"
   - **Context** — what triggered the decision
   - **Decision** — what was chosen
   - **Alternatives considered** — what else was on the table
   - **Why this** — why the chosen option won
   - **Why not others** — concrete reasons each alternative was rejected
   - **Trade-offs accepted** — what was knowingly given up

3. Append the entry below the most recent one, dated today (`YYYY-MM-DD`).

4. If the decision adds or changes a dependency, also remind the user to update `requirements.txt` and (if relevant) the Dockerfile.
```

---

### `.claude/skills/update-architecture/SKILL.md`

```markdown
---
name: update-architecture
description: Update ARCHITECTURE.md when the project's structure or component connections change. Trigger when adding/removing modules or services, adding external integrations (Redis, S3, queue, third-party API), changing how components communicate, or adding new layers (workers, caches, gateways).
---

# Update Architecture

Update the relevant section of `ARCHITECTURE.md` so it always reflects the current shape of the project.

## Steps

1. Read `ARCHITECTURE.md` and identify which section(s) are affected:
   - New module → `Modules` section
   - New external service (Redis, S3, third-party API) → `External Integrations`
   - New worker / cache / queue → `Background Workers / Async Jobs` or a new section
   - Folder layout change → `Folder Layout`
   - Request flow change → `Request Flow` and `Component Map`

2. Update the affected section. Keep it tight — one or two lines per module/integration; full diagrams only when the request flow itself changes.

3. If the change reflects a non-trivial decision, also invoke the `record-decision` skill.
```

---

## Step 11 — Create Project Commands

---

### `.claude/commands/add-resolver.md`

```markdown
# Add Resolver — Add a Single Query or Mutation to an Existing Module

Usage: /add-resolver <module> <query|mutation> <name> <description>

Example: /add-resolver tasks query exportTasks Export all tasks as CSV
Example: /add-resolver tasks mutation archiveTask Archive a single task by id

Follow the Query / Mutation / Service patterns in `.claude/CLAUDE.md` exactly.

## Steps

1. Read `app/modules/{module}/queries.py`, `mutations.py`, `service.py`, `types.py`.

2. Add the resolver:
   - For `query`: add a `@strawberry.field` method to `{Name}sQuery` in `queries.py`
   - For `mutation`: add a `@strawberry.mutation` method to `{Name}sMutation` in `mutations.py`
   - First parameter is `self`, second is `info: Info`. Pull the session from `info.context.session` and pass it to the service method.

3. Add the corresponding service method in `service.py`.

4. Add new Strawberry types or inputs to `types.py` if the resolver needs them.

5. Confirm the resolver is wired end-to-end and try it in GraphiQL at `/graphql`.
```

---

### `.claude/commands/check-conventions.md`

```markdown
# Check Conventions

Review a file against the project's architecture conventions.

Usage: /check-conventions [file_path?]

Example: /check-conventions app/modules/tasks/queries.py

## Steps

1. If a file path is given, use it. Otherwise, identify the most recently modified `.py` file under `app/`.

2. Delegate to the `@convention-reviewer` agent with the target file path.
```

---

## Step 12 — Create Project Agents

---

### `.claude/agents/feature-planner.md`

```markdown
---
name: feature-planner
description: Strategic planner and brainstormer for new GraphQL features. Use when the user wants to explore, brainstorm, or design a new feature before any code is written. Always reads PROJECT_BLUEPRINT.json first to ground suggestions in the real project state. Invoked by the plan-feature skill.
tools: Read, Glob, Grep
model: opus
---

# Feature Planner

Senior architect. Brainstorm and blueprint new features. Never write code — plans only.

Always read `PROJECT_BLUEPRINT.json` then `.claude/CLAUDE.md` then `ARCHITECTURE.md` before producing output.

## Output (produce all sections)

1. **Feature Summary** — one paragraph; state what is NOT included.
2. **Trade-offs** — two approaches, pros/cons, recommended choice with rationale.
3. **New Files** — `types.py`, `queries.py`, `mutations.py`, `service.py`, `models.py` (if needed).
4. **Files to Edit** — `app/graphql/schema.py`, `alembic/env.py`, `config.py`, `.env.example` as needed.
5. **Data Model** — tables with columns, types, constraints, FKs.
6. **GraphQL Surface** — list each query and mutation, with input types and return types.
7. **Strawberry Types** — output types, input types, enums to define.
8. **Implementation Order** — exact sequence without breaking existing code.
9. **Open Questions** — decisions the user must make before implementation.
10. **Decisions to Record** — note any choices that should produce DECISIONS.md entries.

Check for naming conflicts with existing modules in `PROJECT_BLUEPRINT.json`.
Flag any deviation from project conventions immediately.
```

---

### `.claude/agents/project-extender.md`

```markdown
---
name: project-extender
description: Software architect that produces a complete, file-by-file implementation plan for a new GraphQL feature using PROJECT_BLUEPRINT.json as context. Use after @feature-planner has produced a blueprint and the user is ready for a concrete implementation checklist.
tools: Read, Glob, Grep
model: opus
---

# Project Extender

You are a software architect who plans feature additions for this small FastAPI + Strawberry + PostgreSQL project.

## How to Use

1. Read `PROJECT_BLUEPRINT.json` (always do this first)
2. Read `ARCHITECTURE.md` for current structure
3. Accept the user's feature description

## What You Produce

```
## Feature: {feature name}

### New Files
- app/modules/{module}/types.py
- app/modules/{module}/queries.py
- app/modules/{module}/mutations.py
- app/modules/{module}/service.py
- app/modules/{module}/models.py     (if new tables)

### Files to Edit
- app/graphql/schema.py — register new Query / Mutation via merge_types
- alembic/env.py — import new model
- app/core/config.py — add new env vars (if any)
- .env.example — add new env var examples (if any)
- DECISIONS.md — log any architectural decisions
- ARCHITECTURE.md — update Modules / External Integrations sections

### DB Tables
{table name}: {column list with types}

### GraphQL Surface
| Operation | Type | Returns | Auth |
| --- | --- | --- | --- |
| `tasks` | query | `[Task!]!` | active user |
| `task(id: ID!)` | query | `Task!` | active user |
| `createTask(input: TaskCreateInput!)` | mutation | `Task!` | active user |

### Implementation Order
1. Create SQLModel in models.py (or via the add-model skill)
2. docker compose exec app alembic revision --autogenerate -m "..."
3. docker compose exec app alembic upgrade head
4. Scaffold module via the add-module skill
5. Implement service methods using SQLModel ORM
6. Wire resolvers to call services using info.context.session
7. Register Query/Mutation in app/graphql/schema.py
8. Test via GraphiQL at /graphql
9. Append DECISIONS.md and ARCHITECTURE.md entries as needed
```

## What You Check Against PROJECT_BLUEPRINT.json

- Existing modules (avoid duplicate names or table names)
- Established naming conventions
- Whether deps.py / auth context already exists
```

---

### `.claude/agents/convention-reviewer.md`

```markdown
---
name: convention-reviewer
description: Code reviewer that enforces project architecture conventions for the GraphQL stack. Use proactively after writing or modifying types.py, queries.py, mutations.py, service.py. Reports every violation with file:line and a concrete fix. Invoke via /check-conventions.
tools: Read, Glob, Grep
model: sonnet
---

# Convention Reviewer

Review the files or modules provided and report every violation with its exact location and a concrete fix.

## Review Checklist

### Type Files (`types.py`)

- [ ] Output types use `@strawberry.type`; input types use `@strawberry.input`; enums use `@strawberry.enum`
- [ ] Primary key fields use `strawberry.ID` (not plain `str`)
- [ ] `password` does NOT appear in any output type
- [ ] Output types include a `from_model` classmethod that maps from the SQLModel object
- [ ] Optional fields use `Optional[X] = None` (not `X | None` mixed with default-less)

### Query / Mutation Files (`queries.py`, `mutations.py`)

- [ ] Resolvers take `self, info: Info` as first two params
- [ ] Pull session from `info.context.session` — never call `get_session()` directly inside a resolver
- [ ] No business logic — only call service → map to type → return
- [ ] No ORM calls in resolvers (no `session.exec(select(...))`) — that belongs in the service
- [ ] Return types are Strawberry types (not SQLModel objects)
- [ ] Mutation methods are `@strawberry.mutation`; query methods are `@strawberry.field`

### Service Files (`service.py`)

- [ ] Raises `strawberry.exceptions.GraphQLError` (or a subclass) for client-facing errors
- [ ] Accepts `session: Session` as the first argument of every method
- [ ] Uses SQLModel ORM (`session.add`, `session.exec(select(...))`, `session.get`)
- [ ] Calls `session.commit()` after writes; calls `session.refresh(obj)` if returning the populated row
- [ ] No raw SQL strings unless explicitly justified
- [ ] No `os.environ` usage anywhere

### Schema Wiring (`app/graphql/schema.py`)

- [ ] Every module's Query class is included in the root `merge_types("Query", (...))` tuple
- [ ] Every module's Mutation class is included in the root `merge_types("Mutation", (...))` tuple

### All Files

- [ ] Uses `from app.core.config import settings` — no `os.environ` anywhere
- [ ] No hardcoded credentials, API keys, or connection strings

## Report Format

For each violation:

```
[VIOLATION] {file_path}:{line_number}
Rule:  {rule that was broken}
Found: {actual code}
Fix:   {corrected code}
```

Group violations by file. End with:
```
Review complete: {N} file(s) checked, {X} violation(s) found.
```
```

---

## Step 13 — Final Summary

After all files are created, print this summary:

```
Project scaffold complete: {PROJECT_NAME}

Stack:
  FastAPI + Strawberry GraphQL + PostgreSQL (SQLModel + Alembic)
  Multistage Docker (uv builder + python:3.13-slim-bookworm runtime) + docker-compose

Structure created:
  app/core/        — config + database
  app/graphql/     — schema, context, router (mounted at /graphql)
  app/modules/     — feature modules (empty until first add-module)
  alembic/         — migrations (env.py wired)
  tests/           — empty

Top-level files:
  Dockerfile           — multistage, non-root, healthcheck via Python
  docker-compose.yml   — postgres + app, healthchecks, named volume, hot reload
  .dockerignore        — excludes everything not needed in the image
  .env.example         — DB_SERVER=postgres for compose, .env is gitignored
  DECISIONS.md         — bootstrap entries already logged
  ARCHITECTURE.md      — bootstrap shape already documented

Project tooling (.claude/):
  CLAUDE.md — single source of truth for conventions

  Skills (auto-triggered):
    add-module           — full module skeleton (types/queries/mutations/service) + schema merge
    add-model            — SQLModel class + Alembic migration
    plan-feature         — brainstorm + blueprint via @feature-planner
    record-decision      — append entry to DECISIONS.md
    update-architecture  — update relevant section of ARCHITECTURE.md

  Slash Commands:
    /add-resolver <module> <query|mutation> <name> <description>
    /check-conventions [file]

  Agents:
    @feature-planner     @project-extender     @convention-reviewer

Next steps:
  1. cp .env.example .env             — adjust DB credentials if needed
  2. docker compose up --build        — start postgres + app
  3. (after first model exists)       docker compose exec app alembic upgrade head
  4. Open http://localhost:8000/graphql — GraphiQL playground
  5. Say "add a {feature} module" to scaffold your first feature
```
