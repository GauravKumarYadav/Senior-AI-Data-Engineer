# FastAPI Core

> Your complete interview-prep guide — from first route to production-grade auth.

---

## Screen 1 — Why FastAPI?

### What Is FastAPI?

FastAPI is a modern, high-performance Python web framework for building APIs. It was created by Sebastián Ramírez and first released in late 2018. It sits on top of two powerhouse libraries: **Starlette** (for the async web layer) and **Pydantic** (for data validation and serialization). That combination is the secret sauce — you get the raw speed of an ASGI server with the developer experience of automatic type checking and documentation.

Here's the shortest possible FastAPI app:

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello, world"}
```

Run it with `uvicorn main:app --reload` and you instantly get:

- A working JSON API at `http://localhost:8000`
- Interactive Swagger docs at `http://localhost:8000/docs`
- ReDoc documentation at `http://localhost:8000/redoc`
- An OpenAPI 3.1 JSON schema at `http://localhost:8000/openapi.json`

You wrote **three lines of business logic** and got all of that for free. That's the pitch.

### Why Is It So Popular?

Three pillars keep coming up in every conversation about FastAPI:

1. **Async-first design.** FastAPI is built on ASGI (Asynchronous Server Gateway Interface), not the older WSGI standard. Every endpoint can be `async def` natively, which means you can handle thousands of concurrent I/O-bound requests without threads. This matters enormously for microservices that call databases, external APIs, or message queues.

2. **Automatic OpenAPI documentation.** The moment you add type hints and Pydantic models, FastAPI generates complete, interactive API documentation. No Swagger config files. No manual YAML. The code *is* the documentation.

3. **Pydantic-powered validation.** Request bodies, query parameters, path parameters, headers, cookies — everything is validated through Python type hints and Pydantic models at the framework level. Invalid data never reaches your business logic.

> 💡 **Interview Gold**: When asked "Why FastAPI?", hit these three points in order: *async performance*, *auto-generated docs*, and *type-safe validation*. Then add: "It also has first-class dependency injection, which makes testing and composability trivial." That fourth point separates you from someone who just read the README.

### FastAPI vs Flask vs Django REST Framework

| Dimension | Flask | Django REST | FastAPI |
|---|---|---|---|
| Protocol | WSGI (sync) | WSGI (sync) | ASGI (async) |
| Validation | Manual / Marshmallow | Serializers | Pydantic (auto) |
| API docs | Flask-Swagger (addon) | drf-spectacular | Built-in OpenAPI |
| ORM | SQLAlchemy (BYO) | Django ORM (built-in) | SQLAlchemy / any |
| Auth | Flask-Login (addon) | Built-in (sessions) | OAuth2 / JWT (BYO) |
| Learning curve | Low | Medium-high | Low-medium |
| Performance | Moderate | Moderate | High |
| Best for | Simple services | Full-stack + API | High-perf APIs |

**Flask** is the classic micro-framework. It's battle-tested, has a massive ecosystem, and is dead simple. But it's synchronous by default (you can bolt on `async` with Quart, but it's not native), and it has no built-in validation or documentation.

**Django REST Framework (DRF)** is the heavyweight. If you need an admin panel, an ORM, migrations, user authentication, and a full-stack app *plus* an API — DRF is hard to beat. But it's synchronous, opinionated, and heavy for pure API services.

**FastAPI** is the modern middle ground. You get Flask-level simplicity with DRF-level features (validation, auth patterns, docs) and performance that's in the same ballpark as Node.js and Go for I/O-bound workloads. The trade-off? It's younger, the ecosystem is smaller, and you bring your own ORM.

> 🎯 **Aha!** FastAPI isn't "better" than Django or Flask — it's *purpose-built for APIs*. If someone asks you to build a full website with server-rendered templates, Django is probably the better call. FastAPI shines when you're building JSON APIs, microservices, and ML model endpoints.

### Key Selling Points an Interviewer Wants to Hear

When you're in an interview and someone asks "Why did you choose FastAPI for this project?", here's your checklist:

- **Type safety end-to-end.** Request validation, response serialization, and IDE autocompletion — all from the same type hints.
- **Performance.** Benchmarks consistently show FastAPI handling 2-5× the throughput of Flask for I/O-bound workloads thanks to async.
- **Developer productivity.** Less boilerplate means fewer bugs and faster iteration. The auto-docs alone save hours of Swagger maintenance.
- **Dependency injection.** Built-in DI makes testing easy (swap real DB for mock), promotes separation of concerns, and handles resource lifecycle (open/close DB connections).
- **Standards-based.** Generates OpenAPI 3.1 and JSON Schema — your frontend team can auto-generate TypeScript clients from your API spec.
- **Production-ready ecosystem.** Uvicorn + Gunicorn for serving, Pydantic for validation, SQLAlchemy for data access, all well-supported.

---

## Screen 2 — Request/Response Models & Routing

### Pydantic v2 Models — The Foundation

Every FastAPI application lives and dies by its Pydantic models. These are Python classes that define the shape of your data, validate it automatically, and serialize it for responses. Pydantic v2 (which FastAPI 0.100+ uses) is a ground-up rewrite with a Rust-powered core that's 5-50× faster than v1.

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from datetime import datetime
from enum import Enum


class Priority(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"


class TaskCreate(BaseModel):
    """Request model for creating a new task."""

    title: str = Field(
        ...,
        min_length=1,
        max_length=200,
        description="The task title",
        examples=["Fix login bug"],
    )
    description: str | None = Field(
        default=None,
        max_length=2000,
        description="Optional detailed description",
    )
    priority: Priority = Field(
        default=Priority.MEDIUM,
        description="Task priority level",
    )
    tags: list[str] = Field(
        default_factory=list,
        max_length=10,
        description="Up to 10 tags",
    )

    model_config = {
        "str_strip_whitespace": True,
        "json_schema_extra": {
            "examples": [
                {
                    "title": "Fix login bug",
                    "priority": "high",
                    "tags": ["backend", "auth"],
                }
            ]
        },
    }

    @field_validator("tags")
    @classmethod
    def lowercase_tags(cls, v: list[str]) -> list[str]:
        return [tag.lower().strip() for tag in v]

    @model_validator(mode="after")
    def high_priority_needs_description(self) -> "TaskCreate":
        if self.priority == Priority.HIGH and not self.description:
            raise ValueError(
                "High-priority tasks require a description"
            )
        return self
```

Let's break down what's happening:

- **`Field(...)`** — The `...` (Ellipsis) means the field is required. You can also set `default=` for optional fields or `default_factory=` for mutable defaults like lists.
- **`field_validator`** — Validates a single field. In Pydantic v2, these are `@classmethod` decorated with `@field_validator("field_name")`. The `mode="before"` option lets you transform data before type coercion.
- **`model_validator`** — Validates across multiple fields. `mode="after"` means all fields are already parsed; `mode="before"` gives you raw input data.
- **`model_config`** — Replaces the old `class Config:` inner class. Common settings include `str_strip_whitespace`, `strict`, and `from_attributes` (for ORM mode).

> 💡 **Interview Gold**: Know the difference between `field_validator` and `model_validator`. The interviewer will ask. Field validators operate on a single field. Model validators see the entire model — use them for cross-field validation like "end_date must be after start_date."

### Response Models

You typically want a *different* model for responses than for requests. The response includes fields the server generates (ID, timestamps) and may exclude sensitive fields.

```python
class TaskResponse(BaseModel):
    id: int
    title: str
    description: str | None
    priority: Priority
    tags: list[str]
    created_at: datetime
    updated_at: datetime

    model_config = {"from_attributes": True}


class TaskListResponse(BaseModel):
    tasks: list[TaskResponse]
    total: int
    page: int
    page_size: int
```

The `from_attributes = True` setting (formerly `orm_mode`) lets Pydantic read data from ORM objects (like SQLAlchemy models) by accessing attributes instead of dictionary keys.

### Path Parameters, Query Parameters & Request Body

FastAPI uses function signature inspection to determine where each parameter comes from:

```python
from fastapi import FastAPI, Query, Path, Body, HTTPException, status

app = FastAPI()


@app.get("/tasks/{task_id}", response_model=TaskResponse)
async def get_task(
    task_id: int = Path(
        ..., gt=0, description="The ID of the task to retrieve"
    ),
):
    """Fetch a single task by ID."""
    task = await db.get_task(task_id)
    if not task:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Task {task_id} not found",
        )
    return task


@app.get("/tasks", response_model=TaskListResponse)
async def list_tasks(
    page: int = Query(default=1, ge=1, description="Page number"),
    page_size: int = Query(default=20, ge=1, le=100),
    priority: Priority | None = Query(default=None),
    search: str | None = Query(
        default=None, min_length=2, max_length=100
    ),
):
    """List tasks with pagination and optional filters."""
    tasks, total = await db.list_tasks(
        page=page,
        page_size=page_size,
        priority=priority,
        search=search,
    )
    return TaskListResponse(
        tasks=tasks, total=total, page=page, page_size=page_size
    )


@app.post(
    "/tasks",
    response_model=TaskResponse,
    status_code=status.HTTP_201_CREATED,
)
async def create_task(task: TaskCreate):
    """Create a new task. The request body is validated by Pydantic."""
    return await db.create_task(task)
```

**The rules are simple:**

| Parameter location | How FastAPI knows |
|---|---|
| Path parameter | Name matches `{name}` in route |
| Request body | It's a Pydantic `BaseModel` |
| Query parameter | Everything else (primitives) |

> 🎯 **Aha!** If your endpoint has a parameter that's a plain type (`int`, `str`, `bool`) and it's not in the URL path, FastAPI treats it as a query parameter. If it's a Pydantic model, it's the request body. You don't need to explicitly annotate most of the time — but `Query()`, `Path()`, and `Body()` let you add validation and documentation.

### Status Codes and Response Model Exclusion

```python
@app.put("/tasks/{task_id}", response_model=TaskResponse)
async def update_task(
    task_id: int,
    task: TaskCreate,
):
    updated = await db.update_task(task_id, task)
    if not updated:
        raise HTTPException(status_code=404, detail="Task not found")
    return updated


@app.delete(
    "/tasks/{task_id}", status_code=status.HTTP_204_NO_CONTENT
)
async def delete_task(task_id: int):
    deleted = await db.delete_task(task_id)
    if not deleted:
        raise HTTPException(status_code=404, detail="Task not found")
    # 204 = no response body
```

You can also control which fields appear in the response without creating a separate model:

```python
@app.get(
    "/tasks/{task_id}/summary",
    response_model=TaskResponse,
    response_model_exclude={"description", "tags"},
)
async def get_task_summary(task_id: int):
    return await db.get_task(task_id)
```

Use `response_model_include` and `response_model_exclude` sparingly — separate response models are usually cleaner. But for quick projections, these are handy.

---

## Screen 3 — Dependency Injection & Middleware

### Understanding Depends()

Dependency Injection (DI) is FastAPI's superpower for composability and testability. The `Depends()` function tells FastAPI: "Before running this endpoint, call this function first and inject the result."

```python
from fastapi import Depends, FastAPI, HTTPException, status
from typing import Annotated

app = FastAPI()


# ---------- Simple dependency ----------
async def get_db_session():
    """Create a database session for each request."""
    session = AsyncSession(engine)
    try:
        yield session  # <-- yield makes this a "generator dependency"
        await session.commit()
    except Exception:
        await session.rollback()
        raise
    finally:
        await session.close()


# Type alias for cleaner signatures
DBSession = Annotated[AsyncSession, Depends(get_db_session)]


# ---------- Auth dependency ----------
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: DBSession = ...,  # sub-dependency!
) -> User:
    """Validate JWT and return the current user."""
    payload = decode_jwt(token)
    user = await db.get(User, payload["sub"])
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found",
        )
    return user


CurrentUser = Annotated[User, Depends(get_current_user)]


# ---------- Using them ----------
@app.get("/tasks", response_model=list[TaskResponse])
async def list_tasks(
    db: DBSession,
    user: CurrentUser,
):
    """Both db and user are injected automatically."""
    return await db.execute(
        select(Task).where(Task.owner_id == user.id)
    )
```

> 💡 **Interview Gold**: The `Annotated[Type, Depends(fn)]` pattern (introduced in FastAPI 0.95+) is the modern way to declare dependencies. It's cleaner than the old `param: Type = Depends(fn)` style because the annotation is reusable via type aliases. If an interviewer sees you use `Annotated`, they'll know you're current.

### How Yield Dependencies Work

A yield dependency is FastAPI's equivalent of a context manager. Everything *before* the `yield` runs before the endpoint. Everything *after* the `yield` runs after the response is sent. This is perfect for database sessions, file handles, and any resource that needs cleanup.

```python
async def get_db_session():
    session = async_session_maker()
    try:
        yield session           # <-- endpoint runs here
        await session.commit()  # runs AFTER endpoint returns
    except Exception:
        await session.rollback()
        raise
    finally:
        await session.close()   # cleanup always runs
```

The execution order is:

1. `session = async_session_maker()` — setup
2. `yield session` — FastAPI injects `session` into the endpoint
3. Endpoint runs and returns a response
4. `await session.commit()` — teardown (happy path)
5. `await session.close()` — final cleanup

If the endpoint raises an exception, the `except` block catches it, rolls back, and re-raises.

### Sub-Dependencies — Building a Chain

Dependencies can depend on other dependencies. FastAPI resolves the entire graph automatically.

```python
async def get_settings() -> Settings:
    """App configuration — cached per-request by default."""
    return Settings()


async def get_db_session(
    settings: Annotated[Settings, Depends(get_settings)],
):
    engine = create_async_engine(settings.database_url)
    async with AsyncSession(engine) as session:
        yield session


async def get_current_user(
    db: Annotated[AsyncSession, Depends(get_db_session)],
    token: Annotated[str, Depends(oauth2_scheme)],
) -> User:
    # ... validate token, fetch user from db
    return user


async def require_admin(
    user: Annotated[User, Depends(get_current_user)],
) -> User:
    if not user.is_admin:
        raise HTTPException(status_code=403, detail="Admin required")
    return user


@app.delete("/admin/tasks/{task_id}")
async def admin_delete_task(
    task_id: int,
    admin: Annotated[User, Depends(require_admin)],
    db: Annotated[AsyncSession, Depends(get_db_session)],
):
    """Only admins can delete tasks."""
    # `db` is the SAME session instance as the one in require_admin
    # FastAPI caches dependencies within a request by default
    await db.execute(delete(Task).where(Task.id == task_id))
```

> 🎯 **Aha!** FastAPI caches dependency results per-request by default. In the example above, `get_db_session` is called only **once** even though both `get_current_user` and the endpoint depend on it. You get the same session instance. To disable caching, use `Depends(get_db_session, use_cache=False)`.

### Middleware

Middleware wraps every request/response cycle. It runs *before* your endpoint and *after* your endpoint — like a Russian nesting doll.

#### CORS Middleware

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://your-frontend.walmart.com",
        "http://localhost:3000",
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

> 💡 **Interview Gold**: Never use `allow_origins=["*"]` in production with `allow_credentials=True`. It's a security hole. Browsers will reject it anyway (the CORS spec forbids wildcard origins with credentials). Always list your specific origins.

#### Custom Middleware — Logging & Timing

```python
import time
import logging
from starlette.middleware.base import (
    BaseHTTPMiddleware,
    RequestResponseEndpoint,
)
from starlette.requests import Request
from starlette.responses import Response

logger = logging.getLogger(__name__)


class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(
        self, request: Request, call_next: RequestResponseEndpoint
    ) -> Response:
        start = time.perf_counter()
        response = await call_next(request)
        elapsed_ms = (time.perf_counter() - start) * 1000

        logger.info(
            "%s %s completed in %.2fms (status %d)",
            request.method,
            request.url.path,
            elapsed_ms,
            response.status_code,
        )
        response.headers["X-Process-Time-Ms"] = f"{elapsed_ms:.2f}"
        return response


app.add_middleware(TimingMiddleware)
```

#### Auth Middleware (Header-Based API Key)

```python
class APIKeyMiddleware(BaseHTTPMiddleware):
    EXCLUDED_PATHS = {"/docs", "/redoc", "/openapi.json", "/health"}

    async def dispatch(
        self, request: Request, call_next: RequestResponseEndpoint
    ) -> Response:
        if request.url.path in self.EXCLUDED_PATHS:
            return await call_next(request)

        api_key = request.headers.get("X-API-Key")
        if not api_key or api_key not in VALID_API_KEYS:
            return Response(
                content='{"detail": "Invalid API key"}',
                status_code=401,
                media_type="application/json",
            )
        return await call_next(request)
```

**Middleware execution order** matters. Middleware is applied in reverse order of how you add it — the *last* middleware you add is the *outermost* wrapper (runs first). Think of it like stacking decorators.

---

## Screen 4 — Async, Performance & Streaming

### async def vs def — When to Use Each

This is one of the most commonly misunderstood topics in FastAPI, and interviewers love it.

```python
# ✅ Use async def for I/O-bound operations
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    # awaiting an async database call — non-blocking
    user = await db.fetch_one("SELECT * FROM users WHERE id=$1", user_id)
    return user


# ✅ Use plain def for CPU-bound or blocking I/O
@app.get("/report")
def generate_report():
    # This calls a synchronous library (e.g., pandas)
    # FastAPI runs this in a thread pool automatically
    df = pd.read_sql("SELECT * FROM sales", sync_engine)
    return df.to_dict(orient="records")
```

**The rules:**

| Scenario | Use | Why |
|---|---|---|
| Async DB driver (asyncpg) | `async def` | Non-blocking I/O |
| Async HTTP client (httpx) | `async def` | Non-blocking I/O |
| Sync DB driver (psycopg2) | `def` | FastAPI threads it |
| CPU-heavy (pandas, ML) | `def` | FastAPI threads it |
| Mixed sync + async | `def` | Don't block the loop |

> 💡 **Interview Gold**: "If I define an endpoint with `def`, FastAPI doesn't block the event loop — it runs the function in an external threadpool. If I use `async def`, I *must* use `await` for all I/O, otherwise I'll block the entire event loop and kill throughput for all concurrent requests." This answer shows you understand the async model deeply.

### Connection Pooling with asyncpg

For Postgres, `asyncpg` is the gold standard async driver. Pair it with SQLAlchemy's async engine for connection pooling:

```python
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    async_sessionmaker,
    AsyncSession,
)

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/mydb",
    pool_size=20,         # persistent connections
    max_overflow=10,      # burst connections
    pool_timeout=30,      # seconds to wait for a connection
    pool_recycle=1800,    # recycle connections every 30 min
    echo=False,           # set True for SQL logging
)

async_session_maker = async_sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False
)
```

> 🎯 **Aha!** `pool_size=20` means 20 persistent connections. `max_overflow=10` means up to 30 total under load. If you have 4 Uvicorn workers, that's 120 connections to your database. Size your pool based on your DB's `max_connections` setting divided by the number of app instances.

### Background Tasks

For fire-and-forget work (sending emails, logging analytics, cache warming), use `BackgroundTasks`:

```python
from fastapi import BackgroundTasks


async def send_welcome_email(email: str, name: str) -> None:
    """Runs AFTER the response is sent to the client."""
    async with httpx.AsyncClient() as client:
        await client.post(
            "https://email-service.internal/send",
            json={
                "to": email,
                "template": "welcome",
                "vars": {"name": name},
            },
        )


@app.post("/users", status_code=201)
async def create_user(
    user: UserCreate,
    background_tasks: BackgroundTasks,
    db: DBSession,
):
    new_user = await db.create_user(user)
    # This runs after the 201 response is already sent
    background_tasks.add_task(
        send_welcome_email, new_user.email, new_user.name
    )
    return new_user
```

The client gets their `201 Created` immediately. The email sends in the background. If you need robust job queues with retries and persistence, graduate to **Celery** or **ARQ**.

### File Uploads and Form Data

```python
from fastapi import File, UploadFile, Form


@app.post("/upload")
async def upload_file(
    file: UploadFile = File(..., description="The file to upload"),
    category: str = Form(..., description="File category"),
):
    # UploadFile is async-friendly and spools large files to disk
    contents = await file.read()
    size_mb = len(contents) / (1024 * 1024)

    if size_mb > 50:
        raise HTTPException(
            status_code=413,
            detail="File too large. Max 50MB.",
        )

    # Save to object storage, database, etc.
    path = f"/storage/{category}/{file.filename}"
    async with aiofiles.open(path, "wb") as f:
        await f.write(contents)

    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size_mb": round(size_mb, 2),
    }
```

> 🎯 **Aha!** `UploadFile` is better than `bytes` for file uploads. `bytes` loads the entire file into memory at once. `UploadFile` uses a `SpooledTemporaryFile` that spills to disk when the file exceeds a threshold (default 1MB). For large files, you can read in chunks with `await file.read(chunk_size)`.

### StreamingResponse for LLM Token Output

This is increasingly common in modern backends — streaming tokens from an LLM to the client as they're generated:

```python
from fastapi.responses import StreamingResponse
import asyncio


async def stream_llm_response(prompt: str):
    """Simulate an LLM generating tokens one at a time."""
    # In production, this would call OpenAI, Bedrock, or
    # an internal LLM Gateway endpoint
    tokens = f"The answer to '{prompt}' is a thoughtful response.".split()
    for token in tokens:
        yield f"data: {token}\n\n"  # SSE format
        await asyncio.sleep(0.05)   # simulate token generation delay
    yield "data: [DONE]\n\n"


@app.post("/chat/stream")
async def chat_stream(prompt: str):
    return StreamingResponse(
        stream_llm_response(prompt),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # disable nginx buffering
        },
    )
```

The client consumes this with `EventSource` or `fetch` with a readable stream. Each token arrives as a Server-Sent Event (SSE). The `X-Accel-Buffering: no` header is critical if you're behind Nginx — without it, Nginx buffers the entire response and defeats the purpose of streaming.

### WebSockets

For true bidirectional real-time communication (chat, live dashboards, collaborative editing):

```python
from fastapi import WebSocket, WebSocketDisconnect


class ConnectionManager:
    def __init__(self):
        self.active_connections: list[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)


manager = ConnectionManager()


@app.websocket("/ws/chat/{room_id}")
async def websocket_endpoint(
    websocket: WebSocket,
    room_id: str,
):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(
                f"Room {room_id}: {data}"
            )
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast("A user left the chat")
```

> 💡 **Interview Gold**: Know when to use WebSockets vs SSE. **SSE** is simpler (server → client only, auto-reconnect, works over HTTP/2) and is perfect for LLM streaming, live feeds, and notifications. **WebSockets** are bidirectional (client ↔ server) and are needed for chat, gaming, and collaborative editing. Most interviewers appreciate when you know the right tool for the right job.

### Lifespan Events

The modern way to handle startup/shutdown logic (replacing the deprecated `@app.on_event` decorators):

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI


@asynccontextmanager
async def lifespan(app: FastAPI):
    # --- STARTUP ---
    print("🚀 Starting up...")
    app.state.db_engine = create_async_engine(DATABASE_URL)
    app.state.redis = await aioredis.from_url(REDIS_URL)
    app.state.http_client = httpx.AsyncClient()

    yield  # <-- app runs here

    # --- SHUTDOWN ---
    print("🛑 Shutting down...")
    await app.state.http_client.aclose()
    await app.state.redis.close()
    await app.state.db_engine.dispose()


app = FastAPI(lifespan=lifespan)
```

This is a context manager pattern. Everything before `yield` runs at startup. Everything after runs at shutdown. Resources like database engines, Redis connections, and HTTP clients are created once and shared across all requests via `app.state`.

> 🎯 **Aha!** The `lifespan` pattern is better than `on_event("startup")` / `on_event("shutdown")` for two reasons: (1) it's a single function, so you can't accidentally forget the shutdown handler, and (2) variables created before `yield` are naturally in scope after `yield` for cleanup. The old decorators are deprecated as of FastAPI 0.109.

---

## Screen 5 — Authentication & Security

### OAuth2 with JWT — The Complete Flow

This is the single most commonly asked FastAPI interview topic. Let's build a complete JWT auth system.

#### Step 1: Dependencies and Configuration

```python
from datetime import datetime, timedelta, timezone
from typing import Annotated

import jwt
from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from passlib.context import CryptContext
from pydantic import BaseModel

# Configuration — in production, load from environment variables
SECRET_KEY = "your-256-bit-secret"  # os.environ["JWT_SECRET"]
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")

app = FastAPI()
```

#### Step 2: Models

```python
class Token(BaseModel):
    access_token: str
    token_type: str = "bearer"


class TokenPayload(BaseModel):
    sub: str          # subject (user ID or username)
    exp: datetime     # expiration
    iat: datetime     # issued at


class UserInDB(BaseModel):
    username: str
    hashed_password: str
    is_active: bool = True
    is_admin: bool = False
```

#### Step 3: Token Creation and Validation

```python
def create_access_token(
    subject: str,
    expires_delta: timedelta | None = None,
) -> str:
    now = datetime.now(timezone.utc)
    expire = now + (expires_delta or timedelta(minutes=15))
    payload = {
        "sub": subject,
        "exp": expire,
        "iat": now,
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)


def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)


def hash_password(password: str) -> str:
    return pwd_context.hash(password)
```

#### Step 4: The Auth Dependency

```python
async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
) -> UserInDB:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(
            token, SECRET_KEY, algorithms=[ALGORITHM]
        )
        username: str | None = payload.get("sub")
        if username is None:
            raise credentials_exception
    except jwt.ExpiredSignatureError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token has expired",
            headers={"WWW-Authenticate": "Bearer"},
        )
    except jwt.InvalidTokenError:
        raise credentials_exception

    user = await get_user_from_db(username)
    if user is None:
        raise credentials_exception
    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Inactive user",
        )
    return user


CurrentUser = Annotated[UserInDB, Depends(get_current_user)]
```

#### Step 5: The Login Endpoint

```python
@app.post("/auth/token", response_model=Token)
async def login(
    form_data: Annotated[
        OAuth2PasswordRequestForm, Depends()
    ],
):
    """OAuth2 compatible token login."""
    user = await get_user_from_db(form_data.username)
    if not user or not verify_password(
        form_data.password, user.hashed_password
    ):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    access_token = create_access_token(
        subject=user.username,
        expires_delta=timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),
    )
    return Token(access_token=access_token)
```

#### Step 6: Protected Routes

```python
@app.get("/users/me")
async def read_users_me(current_user: CurrentUser):
    return current_user


@app.get("/admin/dashboard")
async def admin_dashboard(
    current_user: CurrentUser,
):
    if not current_user.is_admin:
        raise HTTPException(status_code=403, detail="Admin only")
    return {"message": "Welcome, admin"}
```

> 💡 **Interview Gold**: Always include the `WWW-Authenticate: Bearer` header in 401 responses. It's required by the OAuth2 spec (RFC 6750) and tells the client *how* to authenticate. Forgetting this header is a common mistake that interviewers notice.

### API Key Authentication

For service-to-service communication where OAuth2 is overkill:

```python
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key")

# In production, store hashed keys in a database
VALID_API_KEYS = {
    "sk_live_abc123": "service-a",
    "sk_live_def456": "service-b",
}


async def verify_api_key(
    api_key: Annotated[str, Depends(api_key_header)],
) -> str:
    service_name = VALID_API_KEYS.get(api_key)
    if not service_name:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid API key",
        )
    return service_name


@app.get("/internal/data")
async def get_internal_data(
    service: Annotated[str, Depends(verify_api_key)],
):
    return {"caller": service, "data": "sensitive stuff"}
```

### Rate Limiting with slowapi

Protect your endpoints from abuse:

```python
from slowapi import Limiter
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter


@app.exception_handler(RateLimitExceeded)
async def rate_limit_handler(request, exc):
    return JSONResponse(
        status_code=429,
        content={
            "detail": "Rate limit exceeded. Try again later."
        },
    )


@app.post("/auth/token")
@limiter.limit("5/minute")  # max 5 login attempts per minute
async def login(request: Request, form_data: ...):
    ...


@app.get("/api/search")
@limiter.limit("30/minute")
async def search(request: Request, q: str):
    ...
```

> 🎯 **Aha!** Rate limiting the `/auth/token` endpoint is **critical**. Without it, an attacker can brute-force passwords. A good default is 5 attempts per minute per IP. For general API endpoints, 30-60 requests per minute per IP is reasonable for most applications.

### HTTPS and Security Headers

In production, always run behind a reverse proxy (Nginx, AWS ALB) that terminates TLS. Add security headers:

```python
from starlette.middleware.trustedhost import TrustedHostMiddleware

# Only allow requests to your actual domain
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["api.yourapp.walmart.com", "localhost"],
)


@app.middleware("http")
async def add_security_headers(request: Request, call_next):
    response = await call_next(request)
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "1; mode=block"
    response.headers["Strict-Transport-Security"] = (
        "max-age=31536000; includeSubDomains"
    )
    response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
    return response
```

> 💡 **Interview Gold**: When asked about FastAPI security, go beyond just JWT. Mention: (1) HTTPS everywhere, (2) CORS properly configured, (3) rate limiting on auth endpoints, (4) security headers, (5) input validation via Pydantic, (6) parameterized queries for SQL injection prevention. Showing breadth here signals production experience.

---

## Screen 6 — Quiz Time

Test yourself on everything we've covered. Try to answer before looking at the solution.

---

### Q1: What happens when you define a FastAPI endpoint with `def` instead of `async def`?

- A) The endpoint will not work and FastAPI raises a startup error.
- B) FastAPI runs it in an external threadpool so it doesn't block the event loop.
- C) FastAPI automatically converts it to an async function using `asyncio.run()`.
- D) The endpoint runs on the main thread, blocking all other requests until it completes.

**Answer: B**

FastAPI detects whether your endpoint is `async def` or `def`. If it's a regular `def`, FastAPI runs it in an external threadpool (using `anyio.to_thread.run_sync`) so it doesn't block the main async event loop. This is why synchronous libraries like `pandas` or `psycopg2` work fine in FastAPI — they get their own thread. However, if you use `async def` and then call a blocking function *without* `await`, you *will* block the loop (that's option D's trap — it's what happens when you misuse `async def`, not `def`).

---

### Q2: In FastAPI's dependency injection system, what does `Depends(get_db, use_cache=False)` do?

- A) It disables caching globally for all dependencies in the application.
- B) It ensures `get_db` is called fresh for this specific injection point, even if another dependency in the same request already resolved it.
- C) It prevents the dependency result from being stored in Redis.
- D) It makes the dependency return `None` if it was already called.

**Answer: B**

By default, FastAPI caches dependency results within a single request. If two dependencies both depend on `get_db`, the function is called once and the result is shared. Setting `use_cache=False` forces a fresh call for that specific injection point. This is rarely needed but useful when you intentionally want separate instances (e.g., two independent database sessions for different databases within the same request).

---

### Q3: Which Pydantic v2 feature replaces the old `class Config: orm_mode = True`?

- A) `model_config = {"from_attributes": True}`
- B) `model_config = {"orm_mode": True}`
- C) `@model_validator(mode="orm")`
- D) `class Settings(BaseSettings): orm_mode = True`

**Answer: A**

In Pydantic v2, the inner `class Config` has been replaced with the `model_config` dictionary. The `orm_mode` setting was renamed to `from_attributes` to better describe what it does — it tells Pydantic to read data from object attributes (like SQLAlchemy model instances) rather than dictionary keys. Option B uses the old name in the new syntax and would raise a warning or error.

---

### Q4: You need to send data from the server to the client as an LLM generates tokens. Which approach is most appropriate?

- A) WebSocket connection with bidirectional messaging.
- B) `StreamingResponse` with `text/event-stream` media type (Server-Sent Events).
- C) Long polling with repeated `GET` requests every 100ms.
- D) Return the complete response after all tokens are generated.

**Answer: B**

Server-Sent Events (SSE) via `StreamingResponse` is the standard pattern for LLM token streaming. It's server-to-client only (which is all you need — the client sends the prompt once, then listens), it auto-reconnects on connection drops, and it works over standard HTTP. WebSockets (A) add unnecessary complexity for a unidirectional stream. Long polling (C) wastes bandwidth and adds latency. Waiting for completion (D) defeats the purpose of streaming and creates a poor user experience with long wait times.

---

### Q5: What is the correct way to handle application startup and shutdown in modern FastAPI (0.109+)?

- A) Use `@app.on_event("startup")` and `@app.on_event("shutdown")` decorators.
- B) Override the `__init__` method of the FastAPI class.
- C) Use the `lifespan` async context manager parameter when creating the FastAPI app.
- D) Create a `startup.py` module that runs before `uvicorn` starts.

**Answer: C**

The `lifespan` parameter accepts an async context manager function. Everything before `yield` runs at startup; everything after runs at shutdown. This replaced the `@app.on_event` decorators, which are deprecated as of FastAPI 0.109. The lifespan approach is superior because: (1) startup and shutdown logic lives in one function, (2) resources created at startup are naturally in scope for cleanup at shutdown, and (3) it follows Python's standard context manager pattern. Example: `app = FastAPI(lifespan=lifespan)`.

---

## 🏁 Wrap-Up: Your FastAPI Interview Cheat Sheet

Here's what to remember walking into that interview:

1. **FastAPI = Starlette + Pydantic.** It's async-first, type-safe, and auto-documents your API.
2. **Pydantic v2 models** are the backbone — `field_validator` for single fields, `model_validator` for cross-field, `model_config` for settings.
3. **Dependency injection** with `Depends()` is the killer feature. Yield dependencies handle resource lifecycle. `Annotated` types are the modern pattern.
4. **`async def` for I/O-bound, `def` for CPU-bound.** FastAPI handles both correctly — but never call blocking code inside `async def`.
5. **Lifespan events** replace `on_event`. Use them for database engines, Redis connections, and HTTP clients.
6. **JWT auth** follows a clear pattern: `OAuth2PasswordBearer` → decode token → return user → inject as dependency.
7. **Always mention security breadth**: CORS, rate limiting, security headers, input validation, parameterized queries.

Good luck. You've got this. 🐍
