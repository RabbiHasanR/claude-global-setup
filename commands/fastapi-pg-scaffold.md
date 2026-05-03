# FastAPI + PostgreSQL Project Scaffold

You are creating a complete FastAPI + PostgreSQL project from scratch in the current working directory.
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
mkdir -p app/api app/core app/modules/sql app/utils sql alembic/versions tests .claude/commands .claude/agents
```

Then create empty `__init__.py` files:

```bash
touch app/__init__.py app/api/__init__.py app/core/__init__.py app/modules/__init__.py app/modules/sql/__init__.py
```

Then create placeholder files for empty directories:

```bash
touch app/utils/.gitkeep sql/.gitkeep tests/.gitkeep
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

### `app/core/logging.py`

```python
import logging
import sys
import os
import socket
import structlog
from .config import settings


def add_instance_info(logger, method_name, event_dict):
    event_dict["hostname"] = socket.gethostname()
    event_dict["pid"] = os.getpid()
    return event_dict


def simplify_logger_name(logger, method_name, event_dict):
    full_name = event_dict.get("logger", "")
    if full_name.startswith("uvicorn"):
        event_dict["logger"] = "uvicorn"
    elif full_name.startswith("app"):
        event_dict["logger"] = "app"
    return event_dict


def setup_logging():
    shared_processors = [
        structlog.contextvars.merge_contextvars,
        structlog.stdlib.add_logger_name,
        simplify_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="%Y-%m-%d %H:%M:%S", utc=False),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.UnicodeDecoder(),
        structlog.processors.CallsiteParameterAdder(
            {
                structlog.processors.CallsiteParameter.MODULE,
                structlog.processors.CallsiteParameter.FUNC_NAME,
                structlog.processors.CallsiteParameter.LINENO,
            }
        ),
        add_instance_info,
    ]

    if settings.DEBUG:
        final_processors = shared_processors
        renderer = structlog.dev.ConsoleRenderer(
            colors=True,
            level_styles={
                "debug":    "\033[36m",
                "info":     "\033[32m",
                "warning":  "\033[33m",
                "error":    "\033[31m",
                "critical": "\033[1;41m",
            },
        )
    else:
        final_processors = shared_processors + [structlog.processors.format_exc_info]
        renderer = structlog.processors.JSONRenderer()

    structlog.configure(
        processors=final_processors + [structlog.stdlib.ProcessorFormatter.wrap_for_formatter],
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )

    formatter = structlog.stdlib.ProcessorFormatter(
        foreign_pre_chain=final_processors,
        processor=renderer,
    )

    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(formatter)

    root_logger = logging.getLogger()
    root_logger.handlers = [handler]
    root_logger.setLevel(logging.INFO)

    for _log in ["uvicorn", "uvicorn.error", "uvicorn.access"]:
        _logger = logging.getLogger(_log)
        _logger.handlers = []
        _logger.propagate = True

    return structlog.get_logger()


logger = setup_logging()
```

---

### `app/core/exceptions.py`

```python
from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException as StarletteHTTPException
from .schemas import StandardResponse
from .logging import logger
from .config import settings


def register_exception_handlers(app: FastAPI):

    @app.exception_handler(StarletteHTTPException)
    async def http_exception_handler(request: Request, exc: StarletteHTTPException):
        logger.warning(
            "HTTP Exception",
            method=request.method,
            path=request.url.path,
            status_code=exc.status_code,
            detail=exc.detail,
        )
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

        logger.info(
            "Validation Error",
            method=request.method,
            path=request.url.path,
            errors=error_messages,
        )
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
        logger.error(
            "Global Exception",
            error=str(exc),
            method=request.method,
            path=request.url.path,
            exc_info=exc,
        )
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

### `app/core/middleware.py`

```python
import time
import uuid
import structlog
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request


class LoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        request_id = str(uuid.uuid4())
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(request_id=request_id)

        start = time.perf_counter()
        response = await call_next(request)
        duration_ms = round((time.perf_counter() - start) * 1000, 2)

        structlog.get_logger().info(
            "request_handled",
            method=request.method,
            path=request.url.path,
            status_code=response.status_code,
            duration_ms=duration_ms,
            client_ip=(
                request.headers.get("X-Forwarded-For", "").split(",")[0].strip()
                or (request.client.host if request.client else "unknown")
            ),
            user_agent=request.headers.get("User-Agent", ""),
        )

        response.headers["X-Request-ID"] = request_id
        return response
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
import json
from sqlalchemy import create_engine, text
from sqlmodel import Session
from .config import settings

engine = create_engine(settings.DATABASE_URL, echo=False)


def call_db_function(func_name: str, payload: dict | None = None) -> dict:
    """
    Execute a PostgreSQL stored procedure.
    Every procedure accepts a single JSONB argument and returns a single JSONB result.
    Response envelope: {success, status_code, message, data, errors, meta}
    """
    with Session(engine) as session:
        try:
            result = session.exec(
                text(f"SELECT {func_name}(:payload)"),
                params={"payload": json.dumps(payload or {})},
            )
            row = result.fetchone()
            session.commit()
            return row[0]
        except Exception:
            session.rollback()
            raise
```

---

### `app/modules/sql/service.py`

```python
import os
from pathlib import Path
from sqlalchemy import text
from sqlmodel import Session
from starlette.exceptions import HTTPException as StarletteHTTPException
from app.core.database import engine


def execute_sql_files_from_directory() -> list[str]:
    sql_dir = Path(__file__).resolve().parents[3] / "sql"
    executed: list[str] = []
    with Session(engine) as session:
        try:
            for root, _, files in os.walk(sql_dir):
                for filename in sorted(files):
                    if not filename.endswith(".sql"):
                        continue
                    filepath = Path(root) / filename
                    session.exec(text(filepath.read_text(encoding="utf-8")))
                    executed.append(str(filepath.relative_to(sql_dir)))
            session.commit()
        except Exception as exc:
            session.rollback()
            raise StarletteHTTPException(status_code=500, detail=str(exc)) from exc
    return executed
```

---

### `app/modules/sql/router.py`

```python
from fastapi import Request, Response
from pydantic import BaseModel
from app.core.router import CoreRouter
from app.core.utils import set_response_message
from .service import execute_sql_files_from_directory


class SyncResponse(BaseModel):
    synced: list[str]
    count: int


router = CoreRouter(tags=["SQL"])


@router.post("/sync-functions")
def sync_sql_functions(request: Request, response: Response) -> SyncResponse:
    files = execute_sql_files_from_directory()
    set_response_message(request, f"Synced {len(files)} SQL file(s)")
    response.status_code = 200
    return SyncResponse(synced=files, count=len(files))
```

---

### `app/api/api_v1.py`

```python
from fastapi import APIRouter
from app.modules.sql.router import router as sql_router

api_router = APIRouter()

api_router.include_router(sql_router, prefix="/sql", tags=["SQL"])

# Register feature routers here as the project grows:
# from app.modules.auth.router import router as auth_router
# api_router.include_router(auth_router, prefix="/auth", tags=["Auth"])
#
# from app.modules.users.router import router as users_router
# api_router.include_router(users_router, prefix="/users", tags=["Users"])
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
from app.core.logging import logger
from app.core.exceptions import register_exception_handlers
from app.core.middleware import LoggingMiddleware
from app.api.api_v1 import api_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    logger.info("System Startup", stage="initiating", env=settings.ENVIRONMENT)
    yield
    logger.info("System Shutdown", stage="complete")


app = FastAPI(
    title=settings.PROJECT_NAME,
    description="{PROJECT_DESCRIPTION}",
    lifespan=lifespan,
    docs_url="/docs" if settings.ENVIRONMENT != "production" else None,
    redoc_url=None,
)

register_exception_handlers(app)

# Middleware is executed in reverse registration order (last added = outermost = runs first).
# Request flow: TrustedHostMiddleware → LoggingMiddleware → CORSMiddleware → GZipMiddleware → app
app.add_middleware(GZipMiddleware, minimum_size=1000)
app.add_middleware(
    CORSMiddleware,
    allow_origins=[str(origin).strip("/") for origin in settings.BACKEND_CORS_ORIGINS],
    allow_origin_regex=r"^https?://(localhost|127\.0\.0\.1)(:[0-9]+)?$",
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
app.add_middleware(LoggingMiddleware)
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
structlog>=25.3.0
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

# Add when S3 file storage is added:
# AWS_ACCESS_KEY_ID=
# AWS_SECRET_ACCESS_KEY=
# AWS_STORAGE_BUCKET_NAME=
# AWS_S3_REGION_NAME=us-east-1

# Add when email delivery is added:
# MAIL_API_URL=
# MAIL_API_KEY=
# MAIL_SENDER=
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
- SQLModel + Alembic (migrations)
- Structlog (structured JSON logging)
- All business logic lives in PostgreSQL stored procedures

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

FastAPI + PostgreSQL backend. All business logic lives in PostgreSQL stored procedures.
Python handles HTTP, orchestration, and side effects only.

---

## Tech Stack

| Layer | Technology |
| --- | --- |
| Framework | FastAPI |
| Database | PostgreSQL (stored procedures via PL/pgSQL) |
| ORM / Migrations | SQLModel + Alembic |
| Settings | pydantic-settings |
| Logging | structlog (ProcessorFormatter) |
| Auth | PyJWT (add when needed) |

---

## Critical Rules

1. **Never write raw SQL in Python.** All DB access via `call_db_function()` in `app/core/database.py`.
2. **Never use `os.environ`.** All config via `from app.core.config import settings`.
3. **Never put logic in a router.** Routers call services, services do the work.
4. **Never raise `FastAPI HTTPException`.** Services raise `StarletteHTTPException` from `starlette.exceptions`.
5. **Always use `CoreRouter`.** Never plain `APIRouter` in feature modules.
6. **Always call `set_response_message(request, ...)` before returning** from a router endpoint.
7. **HTTP status code comes from the stored procedure's `status_code` field** — set via injected `Response` object, not hardcoded.
8. **Never include `password` in any response schema.**
9. **Always log with context.** `log = logger.bind(action="...", user_id="...")` then `log.info("event")`.
10. **Email/async side effects are always background tasks.** Never `await` them in the request path.
11. **Never use `response_class=JSONResponse` in path operations and never return `JSONResponse` directly.** Declare the response model as the function's return type annotation instead.
12. **Always declare the response Pydantic model as the function return type.** FastAPI 0.131.0+ serializes declared return types through pydantic-core (Rust) — this is a critical performance optimization. Inject `Response` to set status codes: `response.status_code = result.get("status_code", 200)`. Return the Pydantic model instance directly.

---

## Router Pattern

FastAPI 0.131.0+: declare the return type as a Pydantic model — pydantic-core serializes
the response body in Rust. Never return `JSONResponse` directly. Set status codes via the
injected `Response` object.

```python
from fastapi import Request, Response
from app.core.router import CoreRouter
from app.core.utils import set_response_message
from .service import FeatureService
from . import schemas

router = CoreRouter()


# Single-item endpoint
@router.post("/")
def create_item(
    request: Request, payload: schemas.ItemCreate, response: Response
) -> schemas.ItemResponse:
    result = FeatureService.create_item(payload)
    set_response_message(request, result.get("message", "Created"))
    response.status_code = result.get("status_code", 201)
    return schemas.ItemResponse(**result.get("data", {}))


# List endpoint
@router.get("/")
def get_items(request: Request, response: Response) -> list[schemas.ItemResponse]:
    result = FeatureService.get_items()
    set_response_message(request, result.get("message", "OK"))
    response.status_code = result.get("status_code", 200)
    return [schemas.ItemResponse(**item) for item in (result.get("data") or [])]
```

## Service Pattern

```python
import structlog
from starlette.exceptions import HTTPException as StarletteHTTPException
from app.core.database import call_db_function

logger = structlog.get_logger()

class FeatureService:
    @staticmethod
    def create_item(payload_in: schemas.ItemCreate) -> dict:
        log = logger.bind(action="create_item")
        log.info("create_item_start")
        result = call_db_function("fn_create_item", payload_in.model_dump())
        if not result.get("success"):
            log.warning("create_item_failed", reason=result.get("message"))
            raise StarletteHTTPException(
                status_code=result.get("status_code", 400),
                detail=result.get("message", "Failed"),
            )
        log.info("create_item_success")
        return result
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


class {Name}Filter(BaseModel):
    page: int = Field(default=1, ge=1)
    page_size: int = Field(default=10, ge=1, le=100)
    is_active: Optional[bool] = None
    search: Optional[str] = None
```

## Stored Procedure Response Envelope

Every stored procedure returns:
```json
{
  "success": true,
  "status_code": 201,
  "message": "Created successfully",
  "data": {},
  "errors": [],
  "meta": { "total": 0, "page": 1, "page_size": 10, "total_pages": 0 }
}
```

`status_code` is always used as the HTTP response status code.

## SQL Status Code Guide

| Code | When |
| --- | --- |
| 200 | GET, UPDATE, DELETE success |
| 201 | CREATE success |
| 400 | Validation failure, bad input |
| 401 | Unauthenticated |
| 403 | Forbidden |
| 404 | Not found |
| 409 | Duplicate / conflict |
| 500 | Unexpected DB error (always caught by EXCEPTION handler) |

## SQL Helper Functions

```sql
gen_short_id('bd')                                    -- {cc}__{MMDDHH24MISSUS}
insert_into_table_from_json('table', payload)         -- dynamic INSERT
insert_into_table_from_array_json('table', payloads)  -- bulk INSERT
update_into_table_from_json('table', payload, where)  -- dynamic UPDATE
crypt(payload->>'password', gen_salt('bf'))           -- bcrypt (passwords always hashed in DB)
```

---

## Adding a New Feature

**Shortcut with Claude commands:**

1. `/add-module {feature}` — creates `router.py`, `service.py`, `schemas.py` and registers the router
2. `/add-model {feature}` — creates SQLModel class + Alembic migration; then `alembic upgrade head`
3. `/add-sql {feature} {verb} {noun}` — creates each stored procedure with the JSONB envelope
4. `/sync-sql` — syncs all `.sql` files to the database

To add a single endpoint to an existing module: `/add-endpoint {module} {method} {path} {description}`

**Manually:**

1. Create `app/modules/{feature}/router.py`, `service.py`, `schemas.py`
2. Add `models.py` if new DB tables are needed
3. Write stored procedures in `sql/{feature}/fn_{verb}_{noun}.sql`
4. Register in `app/api/api_v1.py`
5. Run `alembic revision --autogenerate -m "add_{feature}_tables"` then `alembic upgrade head`
6. Add model imports to `alembic/env.py`
7. Sync SQL: `POST /api/v1/sql/sync-functions`

---

## When to Add `app/api/deps.py`

Only create this file when the project needs JWT auth or shared injectable dependencies:

```python
# get_current_user_id          — decodes Bearer token, checks type="access"
# get_current_active_user_id   — calls fn_get_user_status, checks is_active=true
# PermissionChecker(slug, action) — bitwise permission check
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

# Sync stored procedures
POST /api/v1/sql/sync-functions

# Run tests
pytest -v
```

## Slash Commands (Claude Code)

| Command | Example Usage | What It Does |
| --- | --- | --- |
| `/plan-feature` | `/plan-feature user auth with JWT` | Brainstorm + blueprint a new feature before writing code |
| `/plan-db` | `/plan-db multi-tenant isolation` | Design DB schema and stored procedure plan before writing SQL |
| `/add-module` | `/add-module tasks` | Scaffold full module skeleton + register router |
| `/add-endpoint` | `/add-endpoint tasks GET /export Export CSV` | Add endpoint to existing module |
| `/add-model` | `/add-model task` | Create SQLModel class + Alembic migration |
| `/add-sql` | `/add-sql tasks create task` | Create `fn_create_task.sql` |
| `/sync-sql` | `/sync-sql` | Sync all SQL files to DB |
| `/check-conventions` | `/check-conventions app/modules/tasks/router.py` | Check file for violations |

## Agents (Claude Code)

| Agent | Purpose |
| --- | --- |
| `@feature-planner` | Brainstorm and design new features — produces full blueprint before any code is written |
| `@db-planner` | Design DB schemas, stored procedure strategies, and migration sequences before writing SQL |
| `@db-expert` | Write `fn_*.sql` stored procedures from a table and input/output spec |
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
    "migrations": "Alembic",
    "logging": "structlog"
  },
  "conventions": {
    "router": "CoreRouter (never plain APIRouter)",
    "db_access": "call_db_function() only (never raw SQL in Python)",
    "exceptions": "StarletteHTTPException from starlette.exceptions",
    "config": "from app.core.config import settings (never os.environ)",
    "sql_envelope": "success, status_code, message, data, errors, meta"
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
   - `app/modules/{name}/service.py` — `{Name}Service` with one static method per endpoint
   - `app/modules/{name}/schemas.py` — `{Name}Base`, `{Name}Create`, `{Name}Update`, `{Name}Response`, `{Name}Filter`

2. Register in `app/api/api_v1.py`:

```python
from app.modules.{name}.router import router as {name}_router
api_router.include_router({name}_router, prefix="/{name}", tags=["{Name}"])
```

3. Confirm all files exist and router is registered.

4. Remind the user:
   - New DB tables: `/add-model {name}` then `alembic upgrade head`
   - Stored procedures: `/add-sql {name} <verb> <noun>`
   - After SQL: `/sync-sql`
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

3. Add service method to `service.py` following the Service Pattern in CLAUDE.md.

4. Add new schemas to `schemas.py` if needed.

5. If a new stored procedure is needed: `/add-sql {module} <verb> <noun>`

6. Confirm the endpoint is wired end-to-end.
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
from sqlmodel import SQLModel, Field
from datetime import datetime


class {MODEL_NAME}(SQLModel, table=True):
    __tablename__ = "{table_name}"

    id: str = Field(primary_key=True)
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

### `.claude/commands/add-sql.md`

```markdown
# Add SQL Stored Procedure

Create a new PostgreSQL stored procedure with the standard JSONB envelope.

Usage: /add-sql <module> <verb> <noun>

Example: /add-sql tasks create task

Use the SQL template, status codes, and helper functions defined in `.claude/CLAUDE.md`.

## Steps

1. Create `sql/{module}/fn_{verb}_{noun}.sql` following the SQL template in CLAUDE.md.

2. Show the created file path.

3. Run `/sync-sql`.
```

---

### `.claude/commands/sync-sql.md`

```markdown
# Sync SQL Stored Procedures

Sync all `.sql` files from the `sql/` directory into the database.

## Steps

1. Check the server is running:

```bash
curl -s http://localhost:8000/health
```

2. If running, sync:

```bash
curl -X POST http://localhost:8000/api/v1/sql/sync-functions \
     -H "Content-Type: application/json"
```

3. The response lists all synced files and their count. Show it to the user.

4. If the server is not running, start it first:

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

## When to Run

Run `/sync-sql` after:
- Creating a new `.sql` file with `/add-sql`
- Editing an existing stored procedure
- Initial project setup (after writing the first SQL files)
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

### `.claude/commands/plan-db.md`

```markdown
# Plan DB — Brainstorm & Design Database Work

Think through database schema, stored procedures, or migration strategy before writing SQL.

Usage: /plan-db <description>

Example: /plan-db multi-tenant data isolation with row-level security

## Steps

1. Delegate to the `@db-planner` agent with the full description of the DB work.
   Pass along any relevant context (existing tables, feature being built, constraints,
   performance requirements).
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
description: Strategic planner and brainstormer for new features and all project-level work. Use when the user wants to explore, brainstorm, or design a new feature or architectural direction before any code is written. Always reads PROJECT_BLUEPRINT.json first to ground suggestions in the real project state. Invoke via /plan-feature.
tools: Read, Glob, Grep
model: opus
---

# Feature Planner

Senior architect. Brainstorm and blueprint new features. Never write code — plans only.

Always read `PROJECT_BLUEPRINT.json` then `.claude/CLAUDE.md` before producing output.

## Output (produce all sections)

1. **Feature Summary** — one paragraph; state what is NOT included.
2. **Trade-offs** — two approaches, pros/cons, recommended choice with rationale.
3. **New Files** — `router.py`, `service.py`, `schemas.py`, `models.py` (if needed), `fn_*.sql` list.
4. **Files to Edit** — `api_v1.py`, `alembic/env.py`, `config.py`, `.env.example` as needed.
5. **Data Model** — tables with columns, types, constraints, FKs.
6. **API Endpoints** — method, path, auth, permission, description.
7. **Stored Procedures** — function name, inputs, output.
8. **Schemas** — `*Create`, `*Update`, `*Response`, `*Filter` classes needed.
9. **Implementation Order** — exact sequence without breaking existing code.
10. **Open Questions** — decisions the user must make before implementation.

Check for naming conflicts with existing modules in `PROJECT_BLUEPRINT.json`.
Flag any deviation from project conventions immediately.
```

---

### `.claude/agents/db-planner.md`

```markdown
---
name: db-planner
description: Database architect and brainstormer for all PostgreSQL database work. Use when planning table structures, stored procedure strategies, migration sequences, index design, or any DB-level architecture decision before any SQL is written. Invoke via /plan-db.
tools: Read, Glob, Grep
model: sonnet
---

# DB Planner

PostgreSQL database architect. Design schemas and plan stored procedures. Never write SQL — plans only.

Always read `PROJECT_BLUEPRINT.json`, `.claude/CLAUDE.md`, and scan `sql/` before producing output.

## Output (produce all sections)

1. **Schema Design** — per table: columns with types, PKs (`gen_short_id`), FKs, indexes, constraints.
2. **Stored Procedure Plan** — table of: function name, operation, inputs, output, notes.
3. **Migration Sequence** — ordered Alembic migrations with dependencies.
4. **Index Strategy** — which columns, why (query patterns, FK columns, search fields).
5. **Open Questions** — design decisions the user must make before implementation.

## Rules
- IDs: always `gen_short_id('{prefix}')` — never UUID or serial.
- Passwords: always hashed in DB via `crypt(...)`.
- Soft deletes: `is_active = false`, not physical DELETE.
- Every fn_* needs `EXCEPTION WHEN OTHERS THEN` — see envelope format in CLAUDE.md.
```

---

### `.claude/agents/db-expert.md`

```markdown
---
name: db-expert
description: Expert PostgreSQL stored procedure writer. Use when writing or generating fn_*.sql stored procedures with the project's JSONB envelope. Invoke via /add-sql or directly when a stored procedure needs to be created or modified.
tools: Read, Write, Glob, Grep, Bash
model: sonnet
---

# DB Expert

You write PL/pgSQL stored procedures following this project's conventions exactly.

## Input Format

Provide: function (verb + noun), table + columns, inputs from payload, expected output.

Example: `Function: create task | Table: tasks(id, title, description, is_done, created_at) | Inputs: title, description | Output: full row`

## What You Do

Follow the SQL template, JSONB envelope, status codes, and helper functions defined in
`.claude/CLAUDE.md` (sections: Stored Procedure Response Envelope, SQL Status Code Guide,
SQL Helper Functions).

## Naming & Location

File: `sql/{module}/fn_{verb}_{noun}.sql`

After writing, remind: `/sync-sql`
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

You are a software architect who plans feature additions for this FastAPI + PostgreSQL project.
Given a feature description, you read `PROJECT_BLUEPRINT.json` for context and produce a complete,
actionable implementation plan.

## How to Use

1. Read `PROJECT_BLUEPRINT.json` (always do this first — it lists existing modules and conventions)
2. Accept the user's feature description

## What You Produce

A step-by-step implementation plan with exact file paths, covering:

1. **New files to create** — which module files, which SQL files
2. **Existing files to edit** — api_v1.py, alembic/env.py, config.py, .env.example
3. **New DB tables** — model definition, migration needed
4. **Stored procedures** — list of fn_* functions to write (verb + noun + inputs + outputs)
5. **Schemas** — *Create, *Update, *Response, *Filter classes needed
6. **Endpoints** — method, path, auth requirement, permission needed
7. **Order of operations** — exact sequence to implement without breaking anything

## Output Format

```
## Feature: {feature name}

### New Files
- app/modules/{module}/router.py
- app/modules/{module}/service.py
- app/modules/{module}/schemas.py
- app/modules/{module}/models.py   (if new tables needed)
- sql/{module}/fn_create_{noun}.sql
- sql/{module}/fn_get_{noun}s.sql
- sql/{module}/fn_update_{noun}.sql
- sql/{module}/fn_delete_{noun}.sql

### Files to Edit
- app/api/api_v1.py — register new router
- alembic/env.py — import new model
- app/core/config.py — add new env vars (if any)
- .env.example — add new env var examples (if any)

### DB Tables
{table name}: {column list with types}

### Stored Procedures
| Function | Inputs | Output |
| --- | --- | --- |
| fn_create_{noun} | ... | full row |
| fn_get_{noun}s | page, page_size, filters | paginated list + meta |
| fn_get_{noun} | id | single row |
| fn_update_{noun} | id + fields | updated row |
| fn_delete_{noun} | id | confirmation |

### Endpoints
| Method | Path | Auth | Permission |
| --- | --- | --- | --- |
| POST | /api/v1/{module}/ | active user | {module}:create |
| GET  | /api/v1/{module}/ | active user | {module}:view |

### Implementation Order
1. Create SQLModel in models.py
2. Run /add-model
3. Run alembic upgrade head
4. Write stored procedures with /add-sql
5. Run /sync-sql
6. Scaffold module with /add-module
7. Wire service methods to call_db_function()
8. Register router in api_v1.py
9. Test via /docs
```

## What You Check Against PROJECT_BLUEPRINT.json

- Existing modules (avoid duplicate slugs or table names)
- Existing resource bit values (next power of 2 for new permission resources)
- Established naming conventions (fn_{verb}_{noun}, *Create/*Response patterns)
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
- [ ] Injects `Response` and sets `response.status_code = result.get("status_code", ...)` — never hardcodes status
- [ ] Declares Pydantic response model as return type annotation (e.g. `-> schemas.ItemResponse`) — never `JSONResponse`
- [ ] Does NOT use `response_class=JSONResponse` in path operation decorators
- [ ] Does NOT return `JSONResponse(...)` directly
- [ ] No business logic in router — only: call service → set message → set status → return model
- [ ] No direct `call_db_function()` calls in router
- [ ] Dependencies injected via `Depends()`, not called directly

### Service Files (`service.py`)

- [ ] Raises `StarletteHTTPException` from `starlette.exceptions` — NOT `FastAPI HTTPException`
- [ ] Uses `call_db_function()` for all DB access (no raw SQL, no ORM `.select()`)
- [ ] Binds structlog context before logging: `log = logger.bind(action="...")`
- [ ] Logs `{action}_start` and `{action}_success` events
- [ ] Returns `dict` on success
- [ ] No `os.environ` usage anywhere

### Schema Files (`schemas.py`)

- [ ] `password` field does NOT appear in any `*Response` class
- [ ] `*Update` schemas have `model_config = {"extra": "forbid"}`
- [ ] Email fields use `EmailStr` from pydantic
- [ ] Pagination fields use `Field(ge=1, le=100)` bounds

### All Files

- [ ] Uses `from app.core.config import settings` — no `os.environ` anywhere
- [ ] Uses `from app.core.logging import logger` — no bare `logging.getLogger()` calls
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
  sql/           — Stored procedures (empty — add with /add-sql)
  alembic/       — Database migrations (env.py wired)
  tests/         — Test suite (empty — add as features grow)

Project tooling (.claude/):
  CLAUDE.md — single source of truth for architecture patterns and conventions

  Commands:
    /plan-feature <description>                — brainstorm + blueprint a feature before coding
    /plan-db <description>                     — design DB schema and stored procedure plan
    /add-module <name>                         — full module skeleton + registered router
    /add-endpoint <module> <method> <path>     — add endpoint to existing module
    /add-model <name>                          — SQLModel class + Alembic migration
    /add-sql <module> <verb> <noun>            — stored procedure with JSONB envelope
    /sync-sql                                  — sync all SQL files to the database
    /check-conventions [file]                  — check file against CLAUDE.md rules

  Agents:
    @feature-planner     — brainstorm + blueprint new features (opus, read-only)
    @db-planner          — design DB schemas and stored procedure strategies (sonnet, read-only)
    @db-expert           — write fn_*.sql from table + input/output spec
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
