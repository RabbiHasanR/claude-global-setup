# FastAPI + PostgreSQL (ORM) Project Scaffold

You are creating a small FastAPI + PostgreSQL project from scratch in the current working directory.
This scaffold uses **SQLModel ORM + Alembic** for all database operations — no stored procedures, no logging.
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
  (e.g. "Task Manager API" → `task_manager_api`)
- `PROJECT_DB_NAME` = `{PROJECT_SLUG}_db`

Confirm once before creating files: show the three values and ask the user to confirm or correct.

---

## Step 2 — Create Virtual Environment and Directory Structure

First, create and activate the Python virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Confirm activation succeeded — the prompt should show `(.venv)`. Then create the project directories:

```bash
mkdir -p app/api app/core app/modules app/utils alembic/versions tests .claude/commands .claude/agents
```

Then create empty `__init__.py` files:

```bash
touch app/__init__.py app/api/__init__.py app/core/__init__.py app/modules/__init__.py
```

Then create placeholder files for empty directories:

```bash
touch app/utils/.gitkeep tests/.gitkeep
```

---

## Step 3 — Create Application Files

Write every file below. Replace `{PROJECT_NAME}`, `{PROJECT_SLUG}`, `{PROJECT_DESCRIPTION}`, `{PROJECT_DB_NAME}` with the actual values collected in Step 1.

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

### `app/core/schemas.py`

```python
from typing import Any
from pydantic import BaseModel


class StandardResponse(BaseModel):
    success: bool
    message: str
    data: Any | None = None
    errors: list[str] | None = None

    model_config = {"populate_by_name": True}
```

---

### `app/core/exceptions.py`

```python
from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException as StarletteHTTPException
from .schemas import StandardResponse
from .config import settings


def register_exception_handlers(app: FastAPI):

    @app.exception_handler(StarletteHTTPException)
    async def http_exception_handler(request: Request, exc: StarletteHTTPException):
        return JSONResponse(
            status_code=exc.status_code,
            content=StandardResponse(
                success=False,
                message=exc.detail if isinstance(exc.detail, str) else "Request failed",
                errors=[str(exc.detail)],
            ).model_dump(exclude_none=True),
        )

    @app.exception_handler(RequestValidationError)
    async def validation_exception_handler(request: Request, exc: RequestValidationError):
        error_messages = []
        for error in exc.errors():
            field = ".".join(str(x) for x in error["loc"])
            msg = error["msg"]
            error_messages.append(f"{field}: {msg}")

        return JSONResponse(
            status_code=422,
            content=StandardResponse(
                success=False,
                message="Validation Error",
                errors=error_messages,
            ).model_dump(exclude_none=True),
        )

    @app.exception_handler(Exception)
    async def global_exception_handler(request: Request, exc: Exception):
        return JSONResponse(
            status_code=500,
            content=StandardResponse(
                success=False,
                message="Internal Server Error",
                errors=(
                    [str(exc)]
                    if settings.DEBUG
                    else ["An unexpected error occurred. Please contact support."]
                ),
            ).model_dump(exclude_none=True),
        )
```

---

### `app/core/router.py`

```python
import json
from typing import Callable
from fastapi import APIRouter, Request, Response
from fastapi.routing import APIRoute
from fastapi.responses import JSONResponse, StreamingResponse
from .schemas import StandardResponse


class StandardizedRoute(APIRoute):
    def get_route_handler(self) -> Callable:
        original_route_handler = super().get_route_handler()

        async def custom_route_handler(request: Request) -> Response:
            result = await original_route_handler(request)

            if isinstance(result, StreamingResponse):
                return result

            if isinstance(result, JSONResponse):
                response_data = json.loads(result.body)
                message = getattr(request.state, "response_message", "Operation successful")

                wrapped = StandardResponse(
                    success=True,
                    message=message,
                    data=response_data,
                    errors=None,
                )

                new_headers = dict(result.headers)
                new_headers.pop("content-length", None)

                return JSONResponse(
                    status_code=result.status_code,
                    content=wrapped.model_dump(exclude_none=True),
                    headers=new_headers,
                )

            return result

        return custom_route_handler


class CoreRouter(APIRouter):
    def __init__(self, *args, **kwargs):
        kwargs["route_class"] = StandardizedRoute
        super().__init__(*args, **kwargs)
```

---

### `app/core/utils.py`

```python
from fastapi import Request
from pydantic import ValidationError


def set_response_message(request: Request, message: str):
    """Set the success message that StandardizedRoute will include in the wrapped response."""
    request.state.response_message = message


def format_pydantic_errors(e: ValidationError) -> list[str]:
    error_messages = []
    for err in e.errors():
        msg = err.get("msg")
        loc = err.get("loc")
        clean_loc = " -> ".join(str(l) for l in loc) if loc else "field"
        error_messages.append(f"Validation Error in '{clean_loc}': {msg}")
    return error_messages
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

### `app/api/api_v1.py`

```python
from fastapi import APIRouter

api_router = APIRouter()

# Register feature routers here as the project grows:
# from app.modules.users.router import router as users_router
# api_router.include_router(users_router, prefix="/users", tags=["Users"])
#
# from app.modules.tasks.router import router as tasks_router
# api_router.include_router(tasks_router, prefix="/tasks", tags=["Tasks"])
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
from app.core.exceptions import register_exception_handlers
from app.api.api_v1 import api_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    yield


app = FastAPI(
    title=settings.PROJECT_NAME,
    description="{PROJECT_DESCRIPTION}",
    lifespan=lifespan,
    docs_url="/docs" if settings.ENVIRONMENT != "production" else None,
    redoc_url=None,
)

register_exception_handlers(app)

# Middleware is executed in reverse registration order (last added = outermost = runs first).
# Request flow: TrustedHostMiddleware → CORSMiddleware → GZipMiddleware → app
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

app.include_router(api_router, prefix="/api/v1")


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

DB_SERVER=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=
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

- FastAPI + PostgreSQL
- SQLModel (ORM) + Alembic (migrations)
- All database operations via SQLModel sessions — no raw SQL, no stored procedures

## Setup

```bash
cp .env.example .env        # fill in DB credentials
pip install -r requirements.txt
alembic upgrade head        # after first model is added
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Visit http://localhost:8000/docs

## Development

See `.claude/CLAUDE.md` for architecture conventions, patterns, and development workflow.
```

---

## Step 6 — Create `.claude/CLAUDE.md`

```markdown
# CLAUDE.md — {PROJECT_NAME}

{PROJECT_DESCRIPTION}

Small FastAPI + PostgreSQL backend. All database access goes through the SQLModel ORM
with sessions injected as FastAPI dependencies.

---

## Tech Stack

| Layer | Technology |
| --- | --- |
| Framework | FastAPI |
| Database | PostgreSQL |
| ORM / Migrations | SQLModel + Alembic |
| Settings | pydantic-settings |
| Auth | PyJWT (add when needed) |

---

## Critical Rules

1. **All DB access via injected `Session`.** Use `session: Session = Depends(get_session)` in routers and pass the session into the service.
2. **Use SQLModel ORM**, not raw SQL strings. If raw SQL is unavoidable, use `session.exec(text("..."))` from `sqlalchemy`.
3. **Never use `os.environ`.** All config via `from app.core.config import settings`.
4. **Never put logic in a router.** Routers call services, services do the work.
5. **Never raise `FastAPI HTTPException`.** Services raise `StarletteHTTPException` from `starlette.exceptions`.
6. **Always use `CoreRouter`.** Never plain `APIRouter` in feature modules.
7. **Always call `set_response_message(request, ...)` before returning** from a router endpoint.
8. **Set HTTP status code via the injected `Response` object** — do not hardcode it on the path operation decorator.
9. **Never include `password` in any response schema.**
10. **Never use `response_class=JSONResponse` in path operations and never return `JSONResponse` directly.** Declare the response model as the function's return type annotation instead.
11. **Always declare the response Pydantic model as the function return type.** FastAPI 0.131.0+ serializes declared return types through pydantic-core (Rust) — this is a critical performance optimization.
12. **Always commit explicitly** in services. After `session.add(...)`, call `session.commit()` and `session.refresh(obj)` if you need the populated row back.

---

## Router Pattern

FastAPI 0.131.0+: declare the return type as a Pydantic model — pydantic-core serializes
the response body in Rust. Never return `JSONResponse` directly. Set status codes via the
injected `Response` object.

```python
from fastapi import Depends, Request, Response
from sqlmodel import Session
from app.core.router import CoreRouter
from app.core.database import get_session
from app.core.utils import set_response_message
from .service import ItemService
from . import schemas

router = CoreRouter()


# Single-item endpoint
@router.post("/")
def create_item(
    request: Request,
    payload: schemas.ItemCreate,
    response: Response,
    session: Session = Depends(get_session),
) -> schemas.ItemResponse:
    item = ItemService.create_item(session, payload)
    set_response_message(request, "Item created")
    response.status_code = 201
    return schemas.ItemResponse.model_validate(item)


# List endpoint
@router.get("/")
def get_items(
    request: Request,
    response: Response,
    session: Session = Depends(get_session),
) -> list[schemas.ItemResponse]:
    items = ItemService.get_items(session)
    set_response_message(request, "OK")
    response.status_code = 200
    return [schemas.ItemResponse.model_validate(i) for i in items]
```

## Service Pattern

```python
from sqlmodel import Session, select
from starlette.exceptions import HTTPException as StarletteHTTPException
from .models import Item
from . import schemas


class ItemService:
    @staticmethod
    def create_item(session: Session, payload: schemas.ItemCreate) -> Item:
        item = Item(**payload.model_dump())
        session.add(item)
        session.commit()
        session.refresh(item)
        return item

    @staticmethod
    def get_items(session: Session, skip: int = 0, limit: int = 50) -> list[Item]:
        return session.exec(select(Item).offset(skip).limit(limit)).all()

    @staticmethod
    def get_item(session: Session, item_id: str) -> Item:
        item = session.get(Item, item_id)
        if not item:
            raise StarletteHTTPException(status_code=404, detail="Item not found")
        return item

    @staticmethod
    def update_item(session: Session, item_id: str, payload: schemas.ItemUpdate) -> Item:
        item = ItemService.get_item(session, item_id)
        data = payload.model_dump(exclude_unset=True, exclude={"id"})
        for key, value in data.items():
            setattr(item, key, value)
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

## Schemas Pattern

```python
from typing import Optional
from pydantic import BaseModel, Field


class {Name}Base(BaseModel):
    pass  # add shared fields here


class {Name}Create({Name}Base):
    pass


class {Name}Update(BaseModel):
    id: str
    model_config = {"extra": "forbid"}
    # all other fields optional


class {Name}Response({Name}Base):
    id: str
    model_config = {"from_attributes": True}


class {Name}Filter(BaseModel):
    page: int = Field(default=1, ge=1)
    page_size: int = Field(default=10, ge=1, le=100)
    is_active: Optional[bool] = None
    search: Optional[str] = None
```

## Status Code Guide

| Code | When |
| --- | --- |
| 200 | GET, UPDATE, DELETE success |
| 201 | CREATE success |
| 400 | Validation failure, bad input |
| 401 | Unauthenticated |
| 403 | Forbidden |
| 404 | Not found |
| 409 | Duplicate / conflict |
| 500 | Unexpected error |

---

## Adding a New Feature

**Shortcut with Claude commands:**

1. `/add-module {feature}` — creates `router.py`, `service.py`, `schemas.py` and registers the router
2. `/add-model {feature}` — creates SQLModel class + Alembic migration; then `alembic upgrade head`

To add a single endpoint to an existing module: `/add-endpoint {module} {method} {path} {description}`

**Manually:**

1. Create `app/modules/{feature}/router.py`, `service.py`, `schemas.py`
2. Add `models.py` if new DB tables are needed
3. Register in `app/api/api_v1.py`
4. Run `alembic revision --autogenerate -m "add_{feature}_tables"` then `alembic upgrade head`
5. Add model imports to `alembic/env.py`

---

## When to Add `app/api/deps.py`

Only create this file when the project needs JWT auth or shared injectable dependencies:

```python
# get_current_user_id          — decodes Bearer token, checks type="access"
# get_current_active_user_id   — checks user.is_active=true
```

Add `SECRET_KEY`, `ALGORITHM`, and token expiry fields to `app/core/config.py` at the same time.

---

## Commands

```bash
# Start dev server
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Start production server
gunicorn app.main:app -w 4 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000

# New migration
alembic revision --autogenerate -m "description"

# Apply migrations
alembic upgrade head

# Run tests
pytest -v
```

## Slash Commands (Claude Code)

| Command | Example Usage | What It Does |
| --- | --- | --- |
| `/plan-feature` | `/plan-feature user auth with JWT` | Brainstorm + blueprint a new feature before writing code |
| `/add-module` | `/add-module tasks` | Scaffold full module skeleton + register router |
| `/add-endpoint` | `/add-endpoint tasks GET /export Export CSV` | Add endpoint to existing module |
| `/add-model` | `/add-model task` | Create SQLModel class + Alembic migration |
| `/check-conventions` | `/check-conventions app/modules/tasks/router.py` | Check file for violations |

## Agents (Claude Code)

| Agent | Purpose |
| --- | --- |
| `@feature-planner` | Brainstorm and design new features — produces full blueprint before any code is written |
| `@project-extender` | Plan full feature implementation using `PROJECT_BLUEPRINT.json` as context |
| `@convention-reviewer` | Review module files and report all convention violations |

## First Run

```bash
python3 -m venv .venv
source .venv/bin/activate         # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env              # fill in DB credentials
alembic upgrade head              # after first model is added
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Visit http://localhost:8000/docs
```

---

## Step 7 — Create `PROJECT_BLUEPRINT.json`

```json
{
  "project_name": "{PROJECT_NAME}",
  "project_slug": "{PROJECT_SLUG}",
  "description": "{PROJECT_DESCRIPTION}",
  "tech_stack": {
    "framework": "FastAPI",
    "database": "PostgreSQL",
    "orm": "SQLModel",
    "migrations": "Alembic"
  },
  "conventions": {
    "router": "CoreRouter (never plain APIRouter)",
    "db_access": "SQLModel ORM via injected Session (never raw SQL strings)",
    "exceptions": "StarletteHTTPException from starlette.exceptions",
    "config": "from app.core.config import settings (never os.environ)"
  },
  "modules": [],
  "notes": "Feed this file to @project-extender when planning new features."
}
```

---

## Step 8 — Create Project Commands

### `.claude/commands/add-module.md`

```markdown
# Add Module — FastAPI Feature Module

Create a complete feature module from scratch.

Usage: /add-module <name>

Example: /add-module tasks

Follow all patterns in `.claude/CLAUDE.md` exactly — Router, Service, and Schemas.

## Steps

1. Create:
   - `app/modules/{name}/__init__.py` — empty
   - `app/modules/{name}/router.py` — `CoreRouter` with CRUD: `POST /`, `GET /`, `GET /{id}`, `PATCH /`, `DELETE /{id}`
   - `app/modules/{name}/service.py` — `{Name}Service` with one static method per endpoint, all using injected `Session`
   - `app/modules/{name}/schemas.py` — `{Name}Base`, `{Name}Create`, `{Name}Update`, `{Name}Response`, `{Name}Filter`

2. Register in `app/api/api_v1.py`:

```python
from app.modules.{name}.router import router as {name}_router
api_router.include_router({name}_router, prefix="/{name}", tags=["{Name}"])
```

3. Confirm all files exist and router is registered.

4. Remind the user:
   - New DB tables: `/add-model {name}` then `alembic upgrade head`
   - Verify endpoints at http://localhost:8000/docs
```

---

### `.claude/commands/add-endpoint.md`

```markdown
# Add Endpoint — Add to Existing Module

Add a single endpoint to an existing feature module.

Usage: /add-endpoint <module> <method> <path> <description>

Example: /add-endpoint tasks GET /export Export all tasks as CSV

Follow Router Pattern and Service Pattern in `.claude/CLAUDE.md` exactly.

## Steps

1. Read `app/modules/{module}/router.py`, `service.py`, `schemas.py`.

2. Add endpoint to `router.py` following the Router Pattern in CLAUDE.md.
   Inject `session: Session = Depends(get_session)` and pass it to the service method.

3. Add service method to `service.py` following the Service Pattern in CLAUDE.md.
   Use SQLModel ORM operations (`session.add`, `session.exec(select(...))`, etc.).

4. Add new schemas to `schemas.py` if needed.

5. Confirm the endpoint is wired end-to-end.
```

---

### `.claude/commands/add-model.md`

```markdown
# Add Model — SQLModel + Migration

Create a SQLModel table class and generate its Alembic migration.

Usage: /add-model <name>

Example: /add-model task

## Steps

1. Scan `app/modules/` to list existing module directories.
   Use AskUserQuestion to ask:

   - **Model name** — the class name in singular form (e.g. `Task`, `UserProfile`)
     Pre-fill from the argument if one was given.
   - **Module** — which module this model belongs to.
     Show the discovered modules as options plus an "Other (new module)" option.

   Example question:
   > Which module should the `Task` model go in?
   > Options: [tasks] [users] [auth] [Other — enter name]

2. Derive:
   - `MODEL_NAME` = PascalCase of the answer (e.g. `task` → `Task`)
   - `model_name` = snake_case singular (e.g. `task`)
   - `MODULE` = the chosen module directory name (e.g. `tasks`)
   - `table_name` = snake_case plural (e.g. `tasks`)

3. Ask for fields (optional — user may skip):

   > Do you want to define fields now? (You can always add them manually later.)
   > If yes, list them in this format: `field_name:type` — one per line or comma-separated.
   >
   > Supported types: `str`, `int`, `float`, `bool`, `datetime`, `Optional[str]`, `Optional[int]`, etc.
   > Mark a field optional by wrapping its type with `Optional[...]`.
   >
   > Example input:
   > `title:str, description:Optional[str], is_done:bool, due_date:Optional[datetime]`

   If the user provides fields, map each to its SQLModel column:
   - `str` → `str`
   - `Optional[str]` → `Optional[str] = Field(default=None)`
   - `bool` → `bool = Field(default=False)`
   - `int` → `int`
   - `datetime` → `datetime`
   - `Optional[datetime]` → `Optional[datetime] = Field(default=None)`

   If the user skips, leave a `# Add your domain fields here` comment.

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

   Where `{fields}` is the generated field lines from step 3, e.g.:

```python
    title: str
    description: Optional[str] = Field(default=None)
    is_done: bool = Field(default=False)
    due_date: Optional[datetime] = Field(default=None)
```

   Only import `Optional` if at least one optional field is declared.
   Only import `datetime` if at least one datetime field is declared.

5. Import the model in `alembic/env.py`:

```python
from app.modules.{MODULE}.models import {MODEL_NAME}  # noqa
```

6. Generate the migration:

```bash
alembic revision --autogenerate -m "add_{table_name}_table"
```

7. Show the path of the generated file (e.g. `alembic/versions/abc123_add_{table_name}_table.py`).

8. Read the file and confirm `upgrade()` and `downgrade()` look correct.

9. Remind the user: `alembic upgrade head` to apply.

## If the Migration Is Empty

Check:

- Is the model imported in `alembic/env.py`? Add: `from app.modules.{MODULE}.models import {MODEL_NAME}  # noqa`
- Does the model class have `table=True`?

## Rules

- SQLModel class must have `table=True`
- Never edit a migration that has already been applied to production
- `downgrade()` must undo exactly what `upgrade()` does
```

---

### `.claude/commands/plan-feature.md`

```markdown
# Plan Feature — Brainstorm & Design New Feature

Think through a new feature strategically before any code is written.

Usage: /plan-feature <feature description>

Example: /plan-feature user authentication with JWT and role-based permissions

## Steps

1. Delegate to the `@feature-planner` agent with the full feature description.
   Pass along any additional context from the conversation (existing modules, constraints,
   user preferences, anything that affects scope or design).
```

---

### `.claude/commands/check-conventions.md`

```markdown
# Check Conventions

Review a file against the project's architecture conventions.

Usage: /check-conventions [file_path?]

Example: /check-conventions app/modules/tasks/router.py

## Steps

1. If a file path is given, use it.
   Otherwise, identify the most recently modified `.py` file under `app/`.

2. Delegate to the `@convention-reviewer` agent with the target file path.
```

---

## Step 9 — Create Project Agents

Each agent file uses YAML frontmatter to declare its `name`, `description`, `tools`, and `model`
so Claude Code knows when to delegate automatically and which model and tool set to use.

---

### `.claude/agents/feature-planner.md`

```markdown
---
name: feature-planner
description: Strategic planner and brainstormer for new features. Use when the user wants to explore, brainstorm, or design a new feature before any code is written. Always reads PROJECT_BLUEPRINT.json first to ground suggestions in the real project state. Invoke via /plan-feature.
tools: Read, Glob, Grep
model: opus
---

# Feature Planner

Senior architect. Brainstorm and blueprint new features. Never write code — plans only.

Always read `PROJECT_BLUEPRINT.json` then `.claude/CLAUDE.md` before producing output.

## Output (produce all sections)

1. **Feature Summary** — one paragraph; state what is NOT included.
2. **Trade-offs** — two approaches, pros/cons, recommended choice with rationale.
3. **New Files** — `router.py`, `service.py`, `schemas.py`, `models.py` (if needed).
4. **Files to Edit** — `api_v1.py`, `alembic/env.py`, `config.py`, `.env.example` as needed.
5. **Data Model** — tables with columns, types, constraints, FKs.
6. **API Endpoints** — method, path, auth, description.
7. **Schemas** — `*Create`, `*Update`, `*Response`, `*Filter` classes needed.
8. **Implementation Order** — exact sequence without breaking existing code.
9. **Open Questions** — decisions the user must make before implementation.

Check for naming conflicts with existing modules in `PROJECT_BLUEPRINT.json`.
Flag any deviation from project conventions immediately.
```

---

### `.claude/agents/project-extender.md`

```markdown
---
name: project-extender
description: Software architect that produces a complete, file-by-file implementation plan for a new feature using PROJECT_BLUEPRINT.json as context. Use after @feature-planner has produced a blueprint and the user is ready for a concrete implementation checklist with exact file paths and ordered steps.
tools: Read, Glob, Grep
model: opus
---

# Project Extender

You are a software architect who plans feature additions for this small FastAPI + PostgreSQL (ORM) project.
Given a feature description, you read `PROJECT_BLUEPRINT.json` for context and produce a complete,
actionable implementation plan.

## How to Use

1. Read `PROJECT_BLUEPRINT.json` (always do this first — it lists existing modules and conventions)
2. Accept the user's feature description

## What You Produce

A step-by-step implementation plan with exact file paths, covering:

1. **New files to create** — which module files
2. **Existing files to edit** — api_v1.py, alembic/env.py, config.py, .env.example
3. **New DB tables** — model definition, migration needed
4. **Schemas** — *Create, *Update, *Response, *Filter classes needed
5. **Endpoints** — method, path, auth requirement
6. **Order of operations** — exact sequence to implement without breaking anything

## Output Format

```
## Feature: {feature name}

### New Files
- app/modules/{module}/router.py
- app/modules/{module}/service.py
- app/modules/{module}/schemas.py
- app/modules/{module}/models.py   (if new tables needed)

### Files to Edit
- app/api/api_v1.py — register new router
- alembic/env.py — import new model
- app/core/config.py — add new env vars (if any)
- .env.example — add new env var examples (if any)

### DB Tables
{table name}: {column list with types}

### Endpoints
| Method | Path | Auth |
| --- | --- | --- |
| POST | /api/v1/{module}/ | active user |
| GET  | /api/v1/{module}/ | active user |

### Implementation Order
1. Create SQLModel in models.py (or via /add-model)
2. Run alembic revision --autogenerate -m "..."
3. Run alembic upgrade head
4. Scaffold module with /add-module
5. Implement service methods using SQLModel ORM (session.add / select / etc.)
6. Wire router to call service with injected Session
7. Register router in api_v1.py
8. Test via /docs
```

## What You Check Against PROJECT_BLUEPRINT.json

- Existing modules (avoid duplicate slugs or table names)
- Established naming conventions (*Create/*Response patterns)
- Whether deps.py already exists (if not, note it needs creating for auth)
```

---

### `.claude/agents/convention-reviewer.md`

```markdown
---
name: convention-reviewer
description: Code reviewer that enforces project architecture conventions. Use proactively after writing or modifying router.py, service.py, or schemas.py. Reports every violation with exact file:line location and a concrete fix. Invoke via /check-conventions.
tools: Read, Glob, Grep
model: sonnet
---

# Convention Reviewer

You are a code reviewer who enforces the project's architecture conventions. Review the files
or modules provided and report every violation with its exact location and a concrete fix.

## Review Checklist

### Router Files (`router.py`)

- [ ] Uses `CoreRouter` — not plain `APIRouter`
- [ ] Every endpoint calls `set_response_message(request, ...)` before returning
- [ ] Injects `Response` and sets `response.status_code = ...` — never hardcodes status on the decorator
- [ ] Injects `session: Session = Depends(get_session)` for any DB-touching endpoint
- [ ] Declares Pydantic response model as return type annotation (e.g. `-> schemas.ItemResponse`) — never `JSONResponse`
- [ ] Does NOT use `response_class=JSONResponse` in path operation decorators
- [ ] Does NOT return `JSONResponse(...)` directly
- [ ] No business logic in router — only: call service → set message → set status → return model
- [ ] No direct ORM queries in router (no `session.exec(select(...))`) — that belongs in the service
- [ ] Dependencies injected via `Depends()`, not called directly

### Service Files (`service.py`)

- [ ] Raises `StarletteHTTPException` from `starlette.exceptions` — NOT `FastAPI HTTPException`
- [ ] Accepts `session: Session` as the first argument of every method
- [ ] Uses SQLModel ORM (`session.add`, `session.exec(select(...))`, `session.get`, etc.)
- [ ] Calls `session.commit()` after writes; calls `session.refresh(obj)` if returning the populated row
- [ ] No raw SQL strings (no `session.exec(text("..."))`) unless explicitly justified
- [ ] No `os.environ` usage anywhere

### Schema Files (`schemas.py`)

- [ ] `password` field does NOT appear in any `*Response` class
- [ ] `*Update` schemas have `model_config = {"extra": "forbid"}`
- [ ] `*Response` schemas have `model_config = {"from_attributes": True}` (so they can be built from ORM objects)
- [ ] Email fields use `EmailStr` from pydantic
- [ ] Pagination fields use `Field(ge=1, le=100)` bounds

### All Files

- [ ] Uses `from app.core.config import settings` — no `os.environ` anywhere
- [ ] No hardcoded credentials, API keys, or connection strings

## What to Review

You can review:
- Single file: `app/modules/tasks/router.py`
- Full module: `app/modules/tasks/`
- Multiple modules: `app/modules/`

Ask the user what to review if not specified.

## Report Format

For each violation found:

```
[VIOLATION] {file_path}:{line_number}
Rule:  {rule that was broken}
Found: {actual code}
Fix:   {corrected code}
```

Group violations by file. After each file, show a pass/fail count.

If no violations in a file: `✓ {file_path} — clean`

## Summary Line

End your review with:
```
Review complete: {N} file(s) checked, {X} violation(s) found.
```
```

---

## Step 10 — Final Summary

After all files are created, print this summary:

```
Project scaffold complete: {PROJECT_NAME}

Structure created:
  app/           — FastAPI application (all boilerplate ready)
  alembic/       — Database migrations (env.py wired)
  tests/         — Test suite (empty — add as features grow)

Project tooling (.claude/):
  CLAUDE.md — single source of truth for architecture patterns and conventions

  Commands:
    /plan-feature <description>                — brainstorm + blueprint a feature before coding
    /add-module <name>                         — full module skeleton + registered router
    /add-endpoint <module> <method> <path>     — add endpoint to existing module
    /add-model <name>                          — SQLModel class + Alembic migration
    /check-conventions [file]                  — check file against CLAUDE.md rules

  Agents:
    @feature-planner     — brainstorm + blueprint new features (opus, read-only)
    @project-extender    — full implementation plan from PROJECT_BLUEPRINT.json
    @convention-reviewer — review files, report violations with line numbers

Next steps:
  1. source .venv/bin/activate        — activate virtual environment (Windows: .venv\Scripts\activate)
  2. pip install -r requirements.txt  — install dependencies into the venv
  3. cp .env.example .env             — fill in DB credentials
  4. uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
  5. Visit http://localhost:8000/docs
  6. Use /add-module to scaffold your first feature
```
