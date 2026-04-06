# APIs & Microservices

> A comprehensive study guide for backend engineering interviews. Master REST design, microservice patterns, and inter-service communication — the building blocks of every modern distributed system.

---

## Screen 1 — REST API Design Principles

### What REST Actually Means

REST (Representational State Transfer) isn't a protocol — it's an **architectural style** defined by Roy Fielding in his 2000 doctoral dissertation. When interviewers ask about REST, they want to know that you understand the constraints: client-server separation, statelessness, cacheability, a uniform interface, and layered system architecture.

In practice, most "REST APIs" are really "HTTP APIs inspired by REST." True REST compliance (including hypermedia/HATEOAS) is rare in the wild. That's okay — what matters is that you can design clean, predictable, resource-oriented APIs.

> 💡 **Interview Gold**: If asked "Is your API truly RESTful?", a strong answer is: "We follow REST conventions for resource naming, HTTP verbs, and status codes, but we don't implement full HATEOAS. Most production APIs are pragmatically RESTful rather than academically pure."

---

### Resource Naming Conventions

Resources are the **nouns** of your API. They represent entities — users, orders, products. Every URL should describe *what* you're operating on, never *how* you're operating on it.

**The rules:**

1. **Use nouns, not verbs.** The HTTP method *is* the verb.
2. **Use plural nouns** for collections.
3. **Use lowercase with hyphens** for multi-word resources (`line-items`, not `lineItems`).
4. **Nest resources** to show relationships, but keep nesting shallow (max 2 levels deep).
5. **Use path parameters** for identity, query parameters for filtering.

```
✅ Good resource URLs:

GET    /api/v1/users                       # List all users
GET    /api/v1/users/42                     # Get user 42
GET    /api/v1/users/42/orders              # List orders for user 42
GET    /api/v1/users/42/orders/7            # Get order 7 for user 42
POST   /api/v1/users                       # Create a new user
PUT    /api/v1/users/42                     # Replace user 42
PATCH  /api/v1/users/42                     # Partially update user 42
DELETE /api/v1/users/42                     # Delete user 42

❌ Bad URLs (verb-based, inconsistent):

GET    /api/v1/getUser/42
POST   /api/v1/createUser
POST   /api/v1/user/42/deleteOrder/7
GET    /api/v1/Users/42/OrderList
```

> 🎯 **Aha!** When nesting goes beyond two levels (e.g., `/users/42/orders/7/items/3/reviews`), it's a signal to promote a sub-resource to a top-level resource. You can always use `/api/v1/order-items/3` with a query param `?order_id=7` instead of deep nesting.

---

### HTTP Methods Mapped to CRUD

Each HTTP method has specific semantics. Using them correctly makes your API predictable to any developer who's worked with HTTP before.

| Method | CRUD Operation | Request Body? | Typical Response |
|---------|----------------|---------------|-------------------|
| GET | Read | No | 200 + resource(s) |
| POST | Create | Yes | 201 + created resource |
| PUT | Full Replace | Yes | 200 + updated resource |
| PATCH | Partial Update | Yes | 200 + updated resource |
| DELETE | Delete | Rarely | 204 (no content) |

**PUT vs PATCH — the distinction that trips people up:**

- **PUT** replaces the entire resource. If you omit a field, it gets set to null/default. Think "overwrite."
- **PATCH** updates only the fields you send. Everything else stays as-is. Think "merge."

```json
// PUT /api/v1/users/42  — full replacement
// If you omit "phone", it's gone.
{
  "name": "Alice Chen",
  "email": "alice@example.com",
  "phone": "+1-555-0100"
}

// PATCH /api/v1/users/42  — partial update
// Only email changes; name and phone are untouched.
{
  "email": "alice.chen@newdomain.com"
}
```

---

### Status Codes: Your API's Vocabulary

Status codes tell the client *what happened* without the client needing to parse the body. Here's your cheat sheet, organized by the scenarios you'll actually encounter:

**2xx — Success:**

| Code | Name | When to Use |
|------|------|-------------|
| 200 | OK | General success (GET, PUT, PATCH) |
| 201 | Created | Resource created (POST); include `Location` header |
| 204 | No Content | Success with no body (DELETE, some PUTs) |

**4xx — Client Error (the client did something wrong):**

| Code | Name | When to Use |
|------|------|-------------|
| 400 | Bad Request | Malformed JSON, missing required fields |
| 401 | Unauthorized | No auth credentials / invalid token |
| 403 | Forbidden | Valid auth, but insufficient permissions |
| 404 | Not Found | Resource doesn't exist at this URI |
| 409 | Conflict | State conflict (duplicate email, version mismatch) |
| 422 | Unprocessable Entity | Syntactically valid but semantically wrong (e.g., age = -5) |

**5xx — Server Error (we screwed up):**

| Code | Name | When to Use |
|------|------|-------------|
| 500 | Internal Server Error | Unhandled exception — something broke |
| 502 | Bad Gateway | Upstream service returned an invalid response |
| 503 | Service Unavailable | Server is overloaded or down for maintenance |

> 💡 **Interview Gold**: A common interview question is "What's the difference between 401 and 403?" The answer: **401 means "I don't know who you are"** (authentication failure). **403 means "I know who you are, but you can't do this"** (authorization failure). A 401 should prompt the client to re-authenticate; a 403 means retrying with the same credentials won't help.

> 🎯 **Aha!** The debate between 400 and 422 is real. Use **400** when the request body can't even be parsed (malformed JSON, wrong content type). Use **422** when the body *parses* fine but fails business validation (email format is wrong, quantity is negative). FastAPI and Rails both prefer 422 for validation errors.

---

### Idempotency — Why It Matters

An operation is **idempotent** if calling it once produces the same result as calling it N times. This is critical for reliability in distributed systems where network failures cause retries.

| Method | Idempotent? | Why |
|--------|-------------|-----|
| GET | ✅ Yes | Reading data doesn't change it |
| PUT | ✅ Yes | Replacing a resource with the same data = same result |
| DELETE | ✅ Yes | Deleting an already-deleted resource = still deleted |
| PATCH | ⚠️ Depends | `{"email": "new@x.com"}` is idempotent; `{"views": "+1"}` is not |
| POST | ❌ No | Creating a resource twice = two resources |

```bash
# Idempotent: Safe to retry on timeout
PUT /api/v1/users/42
# If the network drops after the server processes this,
# retrying is safe — you'll get the same user state.

# NOT idempotent: Dangerous to retry blindly
POST /api/v1/orders
# Retrying could create duplicate orders!
# Solution: Use an idempotency key header.
```

**The Idempotency Key Pattern (making POST safe to retry):**

```bash
POST /api/v1/orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{
  "product_id": 101,
  "quantity": 2
}
```

The server stores this key. If the same key comes in again, it returns the original response instead of creating a duplicate. Stripe popularized this pattern, and it's now considered best practice for any non-idempotent operation in payment or order systems.

> 💡 **Interview Gold**: If asked "How do you prevent duplicate orders in a distributed system?", talk about idempotency keys. The server hashes the key, checks a fast store (Redis), and either processes the request or returns the cached response. This is a real-world pattern, not a textbook exercise.

---

## Screen 2 — Pagination, Filtering & Versioning

### Pagination: Offset vs Cursor

When a collection has thousands (or millions) of items, you can't return them all. Pagination is how you slice the data into manageable pages.

#### Offset-Based Pagination

The simplest approach: `?page=3&page_size=20` or `?offset=40&limit=20`.

```bash
GET /api/v1/products?page=3&page_size=20
```

```json
{
  "data": [ /* 20 products */ ],
  "meta": {
    "page": 3,
    "page_size": 20,
    "total_count": 1542,
    "total_pages": 78
  }
}
```

**Pros:**
- Simple to implement (`SELECT ... LIMIT 20 OFFSET 40`)
- Users can jump to any page directly
- Easy to display "Page 3 of 78"

**Cons:**
- **Performance degrades** with large offsets (`OFFSET 100000` = DB scans 100K rows and throws them away)
- **Inconsistent results** when data changes between requests (items shift between pages, causing duplicates or missed items)
- Expensive `COUNT(*)` for total count on large tables

#### Cursor-Based Pagination

Uses an opaque cursor (usually a Base64-encoded identifier) pointing to a position in the dataset. The client says "give me 20 items *after* this cursor."

```bash
GET /api/v1/products?limit=20&after=eyJpZCI6IDQyfQ==
```

```json
{
  "data": [ /* 20 products */ ],
  "cursors": {
    "after": "eyJpZCI6IDYyfQ==",
    "has_next": true
  }
}
```

**Pros:**
- **Consistent performance** regardless of position (uses `WHERE id > 42 LIMIT 20` — index-backed)
- **Stable results** even when data changes between requests
- Works beautifully with real-time feeds (social timelines, activity logs)

**Cons:**
- Can't jump to an arbitrary page
- No total count without a separate query
- Slightly more complex to implement

**Python cursor pagination implementation:**

```python
import base64
import json
from fastapi import FastAPI, Query
from pydantic import BaseModel

app = FastAPI()


def encode_cursor(data: dict) -> str:
    """Encode pagination state into an opaque cursor."""
    return base64.urlsafe_b64encode(
        json.dumps(data).encode()
    ).decode()


def decode_cursor(cursor: str) -> dict:
    """Decode an opaque cursor back into pagination state."""
    return json.loads(
        base64.urlsafe_b64decode(cursor.encode()).decode()
    )


@app.get("/api/v1/products")
async def list_products(
    limit: int = Query(default=20, le=100),
    after: str | None = Query(default=None),
):
    # Decode cursor to get the last seen ID
    last_id = 0
    if after:
        cursor_data = decode_cursor(after)
        last_id = cursor_data["id"]

    # Query with cursor (index-friendly!)
    query = """
        SELECT id, name, price
        FROM products
        WHERE id > :last_id
        ORDER BY id ASC
        LIMIT :limit + 1
    """
    # Fetch limit + 1 to check if there's a next page
    rows = await db.fetch_all(query, {"last_id": last_id, "limit": limit})

    has_next = len(rows) > limit
    items = rows[:limit]  # Trim the extra row

    next_cursor = None
    if has_next and items:
        next_cursor = encode_cursor({"id": items[-1]["id"]})

    return {
        "data": items,
        "cursors": {
            "after": next_cursor,
            "has_next": has_next,
        },
    }
```

> 💡 **Interview Gold**: "When would you use offset vs cursor pagination?" Use **offset** for admin dashboards, back-office tools, and small datasets where jumping to a specific page matters. Use **cursor** for user-facing feeds, infinite scroll, real-time data, and any dataset where performance at scale matters. If you're paginating more than ~10K rows regularly, cursor wins.

---

### Filtering and Sorting via Query Parameters

Clean filtering keeps your API flexible without creating dozens of endpoints.

```bash
# Filter by status AND sort by creation date (descending)
GET /api/v1/orders?status=active&sort=-created_at

# Multiple filters
GET /api/v1/products?category=electronics&min_price=50&max_price=200&in_stock=true

# Field selection (sparse fieldsets) — reduce payload size
GET /api/v1/users/42?fields=id,name,email
```

**Sorting conventions:**
- `sort=created_at` → ascending (default)
- `sort=-created_at` → descending (prefix with `-`)
- `sort=status,-created_at` → multi-field sort

**Date range filtering:**

```bash
GET /api/v1/orders?created_after=2025-01-01T00:00:00Z&created_before=2025-12-31T23:59:59Z
```

> 🎯 **Aha!** Never expose raw SQL operators in query params (`?price__gt=50`). It leaks your implementation and opens injection risks. Instead, use semantic parameter names like `min_price`, `max_price`, `created_after`, `created_before`. They're self-documenting and safe.

---

### API Versioning Strategies

APIs evolve. Breaking changes are inevitable. Versioning lets old clients keep working while new clients use new features.

#### Strategy 1: URL Path Versioning (Most Common)

```
GET /api/v1/users/42
GET /api/v2/users/42
```

**Pros:** Explicit, visible, easy to route, easy to cache.
**Cons:** URL changes for every version; can lead to code duplication.
**Who uses it:** Twitter, Stripe, GitHub (partially).

#### Strategy 2: Header Versioning

```bash
GET /api/users/42
Accept: application/vnd.myapi.v2+json
```

**Pros:** Clean URLs; version is a concern of content negotiation.
**Cons:** Harder to test (can't just change the URL in a browser), harder to cache.
**Who uses it:** GitHub (primary method).

#### Strategy 3: Query Parameter Versioning

```
GET /api/users/42?version=2
```

**Pros:** Easy to switch versions for testing.
**Cons:** Easy to forget; mixes concerns into query string.
**Who uses it:** Google APIs (some), Amazon.

**The pragmatic recommendation:**

URL path versioning (`/v1/`) is the industry default. It's explicit, discoverable, and works with every HTTP client, cache, and proxy without special configuration. Start there unless you have a compelling reason not to.

> 💡 **Interview Gold**: "When do you introduce v2?" The answer isn't "when we add features." New fields and new endpoints are **non-breaking** — add them to v1. You only version when you make **breaking changes**: removing a field, changing a field's type, renaming an endpoint, or altering the response structure. The goal is to delay v2 as long as possible through additive, backward-compatible design.

---

### HATEOAS (Awareness Level)

HATEOAS (Hypermedia As The Engine Of Application State) is the idea that API responses should include **links** telling the client what it can do next. It's the most debated REST constraint.

```json
{
  "id": 42,
  "name": "Alice Chen",
  "email": "alice@example.com",
  "_links": {
    "self": { "href": "/api/v1/users/42" },
    "orders": { "href": "/api/v1/users/42/orders" },
    "update": { "href": "/api/v1/users/42", "method": "PATCH" },
    "delete": { "href": "/api/v1/users/42", "method": "DELETE" }
  }
}
```

In practice, very few APIs implement full HATEOAS. It adds complexity, increases payload size, and most frontend developers prefer reading API docs over parsing hypermedia links. But you should know what it is — it comes up in interviews as a "have you heard of this?" question.

---

## Screen 3 — Error Handling & Documentation

### Consistent Error Response Format

Nothing frustrates API consumers more than inconsistent error responses. One endpoint returns `{"error": "not found"}`, another returns `{"message": "Not Found", "code": 404}`, and a third returns a raw string. Don't be that API.

#### RFC 7807 — Problem Details for HTTP APIs

RFC 7807 (now superseded by RFC 9457, same concept) defines a standard error format. It's the gold standard for API error responses.

```json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 422,
  "detail": "The 'email' field must be a valid email address.",
  "instance": "/api/v1/users",
  "errors": [
    {
      "field": "email",
      "message": "Must be a valid email address",
      "value": "not-an-email"
    },
    {
      "field": "age",
      "message": "Must be a positive integer",
      "value": -5
    }
  ]
}
```

**The key fields:**

| Field | Required? | Purpose |
|-------|-----------|---------|
| `type` | Yes | A URI that identifies the error type (can be a docs link) |
| `title` | Yes | A short, human-readable summary |
| `status` | Yes | The HTTP status code (mirrored in body for convenience) |
| `detail` | Yes | A human-readable explanation specific to this occurrence |
| `instance` | No | The URI of the request that caused the error |
| `errors` | No (extension) | Array of field-level validation errors |

**FastAPI implementation of consistent error handling:**

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

app = FastAPI()


class APIError(Exception):
    """Base exception for all API errors."""

    def __init__(
        self,
        status: int,
        error_type: str,
        title: str,
        detail: str,
    ):
        self.status = status
        self.error_type = error_type
        self.title = title
        self.detail = detail


@app.exception_handler(APIError)
async def api_error_handler(request: Request, exc: APIError):
    return JSONResponse(
        status_code=exc.status,
        content={
            "type": f"https://api.example.com/errors/{exc.error_type}",
            "title": exc.title,
            "status": exc.status,
            "detail": exc.detail,
            "instance": str(request.url),
        },
    )


@app.exception_handler(RequestValidationError)
async def validation_error_handler(
    request: Request, exc: RequestValidationError
):
    return JSONResponse(
        status_code=422,
        content={
            "type": "https://api.example.com/errors/validation-failed",
            "title": "Validation Failed",
            "status": 422,
            "detail": "One or more fields failed validation.",
            "instance": str(request.url),
            "errors": [
                {
                    "field": ".".join(str(loc) for loc in err["loc"]),
                    "message": err["msg"],
                }
                for err in exc.errors()
            ],
        },
    )
```

> 💡 **Interview Gold**: When discussing error handling, mention that you always include a **correlation ID** (also called trace ID or request ID) in error responses. This lets support engineers trace a user's complaint back to the exact log entry. Generate it as middleware: `X-Request-ID: 550e8400-e29b-41d4-a716-446655440000`.

**More error response examples for common scenarios:**

```json
// 401 Unauthorized — Missing or expired token
{
  "type": "https://api.example.com/errors/authentication-required",
  "title": "Authentication Required",
  "status": 401,
  "detail": "The access token is expired. Please refresh your token.",
  "instance": "/api/v1/users/42"
}

// 409 Conflict — Duplicate resource
{
  "type": "https://api.example.com/errors/duplicate-resource",
  "title": "Duplicate Resource",
  "status": 409,
  "detail": "A user with email 'alice@example.com' already exists.",
  "instance": "/api/v1/users"
}

// 429 Too Many Requests — Rate limit exceeded
{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "detail": "You have exceeded 100 requests per minute. Retry after 23 seconds.",
  "instance": "/api/v1/products"
}
```

---

### OpenAPI / Swagger Auto-Generated Docs

OpenAPI (formerly Swagger) is the standard for describing REST APIs. It's a YAML/JSON specification that describes your endpoints, request/response schemas, authentication, and more.

**FastAPI generates OpenAPI specs automatically** from your type hints and Pydantic models. This is one of its killer features:

```python
from pydantic import BaseModel, EmailStr

class UserCreate(BaseModel):
    name: str
    email: EmailStr
    age: int

class UserResponse(BaseModel):
    id: int
    name: str
    email: str

@app.post(
    "/api/v1/users",
    response_model=UserResponse,
    status_code=201,
    summary="Create a new user",
    description="Register a new user account. Returns the created user.",
    tags=["Users"],
)
async def create_user(user: UserCreate) -> UserResponse:
    ...
```

This auto-generates an interactive spec at `/docs` (Swagger UI) and `/redoc` (ReDoc).

### ReDoc vs Swagger UI

| Feature | Swagger UI | ReDoc |
|---------|-----------|-------|
| Try-it-out (live API calls) | ✅ Yes | ❌ No |
| Visual design | Functional | Beautiful, clean |
| Three-panel layout | No | Yes (nav, content, examples) |
| Best for | Developers testing | Published documentation |
| Search | Basic | Excellent full-text |

**Best practice:** Use **Swagger UI** during development (interactive testing is invaluable) and **ReDoc** for published, external-facing documentation.

### API Documentation Best Practices

1. **Every endpoint needs a description** — not just the method and URL.
2. **Include request/response examples** — real, copy-pasteable JSON.
3. **Document all error codes** an endpoint can return.
4. **Show authentication requirements** per endpoint.
5. **Provide curl examples** — the universal API testing language.
6. **Keep docs in sync with code** — auto-generation (FastAPI, SpringDoc) is the only reliable way.

```bash
# Good documentation includes curl examples like this:
curl -X POST https://api.example.com/api/v1/users \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice Chen",
    "email": "alice@example.com",
    "age": 30
  }'
```

> 🎯 **Aha!** The best API documentation is the one that's never out of date. This means generating it from code, not writing it by hand. If your docs and code can diverge, they *will* diverge. FastAPI + Pydantic models make this virtually impossible — the types ARE the docs.

---

## Screen 4 — Microservice Architecture Patterns

### Monolith vs Microservices

This is the most important architectural decision you'll discuss in interviews. There's no universally correct answer — it depends on your team, product, and scale.

**Monolith:**

```
┌─────────────────────────────────────────────┐
│              Monolith Application            │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│  │  Users   │ │  Orders  │ │   Payments   │ │
│  │  Module  │ │  Module  │ │   Module     │ │
│  └────┬─────┘ └────┬─────┘ └──────┬───────┘ │
│       └─────────────┴──────────────┘         │
│                Shared Database               │
└─────────────────────────────────────────────┘
```

**Microservices:**

```
┌──────────┐    ┌──────────┐    ┌──────────────┐
│  Users   │    │  Orders  │    │   Payments   │
│ Service  │◄──►│ Service  │◄──►│   Service    │
│  (API)   │    │  (API)   │    │   (API)      │
└────┬─────┘    └────┬─────┘    └──────┬───────┘
     │               │                 │
┌────▼─────┐    ┌────▼─────┐    ┌──────▼───────┐
│ Users DB │    │ Orders DB│    │ Payments DB  │
└──────────┘    └──────────┘    └──────────────┘
```

| Factor | Monolith | Microservices |
|--------|----------|---------------|
| Team size | < 10 engineers | 10+ (multiple teams) |
| Deploy speed | One deploy, everything ships | Independent deploys per service |
| Complexity | Low operational overhead | High (networking, observability, etc.) |
| Data consistency | Easy (single DB, ACID transactions) | Hard (eventual consistency, sagas) |
| Scaling | Scale entire app | Scale individual services |
| New project? | ✅ Almost always start here | ❌ Don't start here |

> 💡 **Interview Gold**: "Start with a monolith, extract microservices when you have a reason." The reason is never "because Netflix does it." Good reasons include: a module needs to scale independently, a team needs to deploy independently, or a bounded context has truly different data/technology requirements. Martin Fowler calls this the "Monolith First" approach.

---

### Service Decomposition

How do you decide what becomes its own service? Two guiding principles:

**1. Bounded Contexts (from Domain-Driven Design)**

A bounded context is a boundary within which a particular domain model is defined and consistent. "User" means different things in different contexts:

- **Authentication context:** User = credentials, tokens, sessions
- **Billing context:** User = payment methods, invoices, subscription tier
- **Social context:** User = profile, followers, posts

Each of these can be a separate service with its own data model.

**2. Single Responsibility at the Service Level**

Each service should have one reason to change. If you're modifying the payments service every time the catalog team ships a feature, your boundaries are wrong.

**Signs you should extract a service:**
- A module has a different scaling profile (e.g., image processing is CPU-heavy)
- A different team owns it and deploys on a different schedule
- It uses a fundamentally different technology (e.g., ML model serving)
- It has a different data retention or compliance requirement

---

### API Gateway Pattern

An API Gateway sits between clients and your microservices. It's the single entry point that handles cross-cutting concerns so individual services don't have to.

```
                    ┌─────────────────────┐
   Clients ────────►│    API Gateway      │
   (Web, Mobile,    │  ┌───────────────┐  │
    Partners)       │  │ Auth / JWT    │  │
                    │  │ Rate Limiting │  │
                    │  │ Routing       │  │
                    │  │ SSL Termination│  │
                    │  │ Request Log   │  │
                    │  └───────────────┘  │
                    └──────┬──┬──┬────────┘
                           │  │  │
              ┌────────────┘  │  └────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────────┐
        │  Users   │   │  Orders  │   │   Payments   │
        │ Service  │   │ Service  │   │   Service    │
        └──────────┘   └──────────┘   └──────────────┘
```

**What the gateway does:**
- **Routing:** Maps `/api/v1/users/*` → Users Service, `/api/v1/orders/*` → Orders Service
- **Authentication:** Validates JWTs once at the edge, passes user identity downstream
- **Rate limiting:** Protects services from abuse (100 requests/minute per client)
- **Request aggregation:** Combines responses from multiple services into one (BFF pattern)
- **SSL termination:** Handles HTTPS so internal services can speak plain HTTP

**Popular options:** Kong, AWS API Gateway, Envoy, Traefik, NGINX (with Lua).

> 🎯 **Aha!** The API Gateway can become a **bottleneck and single point of failure** if not designed carefully. Always deploy it with redundancy (multiple instances behind a load balancer). Avoid putting business logic in the gateway — it should be a thin routing/auth layer, not a "smart pipe."

---

### Service Discovery

In a microservice world, services need to find each other. Hardcoding IP addresses doesn't work when services scale up/down dynamically.

**DNS-Based Discovery (simplest):**
Each service registers a DNS name. Kubernetes does this automatically — `orders-service.default.svc.cluster.local` resolves to the current pods.

**Service Registry (Consul, Eureka):**
Services register themselves on startup and deregister on shutdown. Other services query the registry to find healthy instances.

**Kubernetes Services:**
K8s Services provide a stable virtual IP that load-balances across pods. This is the most common approach in modern deployments.

```yaml
# Kubernetes Service — other services reach orders via
# http://orders-service:8080
apiVersion: v1
kind: Service
metadata:
  name: orders-service
spec:
  selector:
    app: orders
  ports:
    - port: 8080
      targetPort: 8080
```

---

### Circuit Breaker Pattern

When Service A calls Service B, and Service B is slow or down, Service A can exhaust its connection pool waiting. The circuit breaker prevents this cascade failure.

```
State Machine:

  CLOSED ──── failure threshold reached ────► OPEN
  (normal)                                    (fail fast,
     ▲                                         don't call)
     │                                            │
     │                                   timeout expires
     │                                            │
     └──── success ◄──── HALF-OPEN ◄──────────────┘
                         (test with
                          one request)
```

**How it works:**
1. **CLOSED** (normal): Requests flow through. Track failure count.
2. **OPEN** (fail fast): After N failures, stop calling the downstream service. Return a fallback response immediately. This prevents timeouts from piling up.
3. **HALF-OPEN** (testing): After a cooldown period, allow one test request. If it succeeds, go back to CLOSED. If it fails, stay OPEN.

```python
# Using the 'circuitbreaker' library
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30)
async def call_payment_service(order_id: str) -> dict:
    """Call payment service with circuit breaker protection."""
    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.post(
            "http://payments-service:8080/api/v1/charges",
            json={"order_id": order_id},
        )
        response.raise_for_status()
        return response.json()
```

> 💡 **Interview Gold**: When discussing circuit breakers, mention **bulkheads** as a complementary pattern. A bulkhead isolates failures by giving each downstream service its own connection pool. If the payment service is slow, it exhausts *its* pool but doesn't affect connections to the inventory service. Think of it like watertight compartments in a ship.

---

### Saga Pattern for Distributed Transactions

In a monolith, you wrap related operations in a database transaction. In microservices, there's no distributed ACID transaction across services. The Saga pattern is the solution.

A saga is a sequence of local transactions where each step triggers the next, and each step has a **compensating transaction** (an undo) if something fails.

**Example: "Place Order" saga:**

| Step | Service | Action | Compensation (undo) |
|------|---------|--------|---------------------|
| 1 | Orders | Create order (PENDING) | Cancel order |
| 2 | Inventory | Reserve items | Release reservation |
| 3 | Payments | Charge card | Issue refund |
| 4 | Orders | Confirm order (CONFIRMED) | — |

If step 3 (charge card) fails, the saga runs compensations in reverse: release reserved items (undo step 2), cancel order (undo step 1).

**Two coordination strategies:**

**Choreography** (event-driven, no central coordinator):
Each service publishes an event when it finishes, and the next service listens for that event.

```
Orders ──publishes──► "OrderCreated"
                            │
Inventory ◄────listens──────┘
         ──publishes──► "ItemsReserved"
                            │
Payments ◄────listens───────┘
         ──publishes──► "PaymentCharged"
                            │
Orders   ◄────listens───────┘ (confirms order)
```

**Orchestration** (central saga coordinator):
A central orchestrator service tells each participant what to do and handles compensations.

```
                 ┌──────────────────┐
                 │  Saga            │
                 │  Orchestrator    │
                 └──┬────┬────┬────┘
                    │    │    │
          step 1    │    │    │  step 3
                    ▼    │    ▼
              Orders     │    Payments
                         │
                   step 2│
                         ▼
                     Inventory
```

| Factor | Choreography | Orchestration |
|--------|-------------|---------------|
| Coupling | Loose (services are independent) | Tighter (orchestrator knows all steps) |
| Complexity | Hard to trace across services | Easy to understand the full flow |
| Best for | Simple sagas (2-3 steps) | Complex sagas (4+ steps, branching) |
| Debugging | Requires distributed tracing | Centralized log of saga state |

> 🎯 **Aha!** Choreography sounds elegant but becomes a nightmare to debug with more than 3-4 services. You end up with "event spaghetti" — no one knows the full order of operations without reading every service's event handlers. For anything complex, orchestration is more maintainable. Think of it as the difference between a jazz improvisation and a symphony with a conductor.

---

## Screen 5 — Communication Patterns & Beyond REST

### Synchronous: REST vs gRPC

REST over HTTP/JSON is the default for public APIs and simple service-to-service calls. But for internal microservice communication where performance matters, **gRPC** is a serious contender.

**gRPC (Google Remote Procedure Call):**
- Uses **Protocol Buffers** (protobuf) for serialization — binary, strongly typed, 10x smaller than JSON
- Runs over **HTTP/2** — multiplexed streams, header compression
- Supports **four communication patterns:** unary, server streaming, client streaming, bidirectional streaming
- **Code generation** — define your service in a `.proto` file, generate client/server stubs in any language

```protobuf
// orders.proto — service definition
syntax = "proto3";

service OrderService {
  // Unary RPC — standard request/response
  rpc GetOrder (GetOrderRequest) returns (Order);

  // Server streaming — one request, stream of responses
  rpc WatchOrderStatus (GetOrderRequest) returns (stream OrderStatus);
}

message GetOrderRequest {
  string order_id = 1;
}

message Order {
  string id = 1;
  string user_id = 2;
  repeated OrderItem items = 3;
  string status = 4;
}
```

| Factor | REST (HTTP/JSON) | gRPC (HTTP/2 + Protobuf) |
|--------|-----------------|--------------------------|
| Payload size | Larger (text-based JSON) | ~10x smaller (binary protobuf) |
| Speed | Good | Excellent (2-10x faster) |
| Streaming | Not native (WebSocket workaround) | Native (4 streaming patterns) |
| Browser support | ✅ Native | ⚠️ Requires grpc-web proxy |
| Human-readable | ✅ JSON is easy to debug | ❌ Binary (need tools to inspect) |
| Schema evolution | Manual (hope clients handle new fields) | Built-in (field numbers, backward compat) |
| Best for | Public APIs, web frontends | Internal service-to-service, high throughput |

> 💡 **Interview Gold**: "When would you choose gRPC over REST?" Answer: For **internal** service-to-service communication where latency and bandwidth matter — especially polyglot environments where you need type-safe contracts across languages. Keep REST for public/external APIs because browsers speak HTTP/JSON natively and developers expect it.

---

### Asynchronous: Message Queues & Event Streaming

Not every service call needs an immediate response. Asynchronous messaging **decouples** services in both time and availability.

**Message Queue (Point-to-Point):**
A producer sends a message to a queue; exactly one consumer picks it up. Good for task distribution.

```
Producer ──► [ Queue ] ──► Consumer

Examples: RabbitMQ, AWS SQS, Redis Streams
```

**Use cases:** Background jobs (send email, resize image), work distribution across workers.

**Event Streaming (Pub/Sub + Log):**
A producer publishes events to a topic; multiple consumer groups can each read all events independently. The log is persistent — consumers can replay history.

```
Producer ──► [ Topic / Partition Log ] ──► Consumer Group A
                                       ──► Consumer Group B
                                       ──► Consumer Group C

Examples: Apache Kafka, AWS Kinesis, Redpanda
```

**Use cases:** Event-driven architectures, audit trails, analytics pipelines, real-time data replication between services.

**Key differences:**

| Factor | Message Queue (SQS, RabbitMQ) | Event Stream (Kafka) |
|--------|-------------------------------|----------------------|
| Delivery | Each message consumed once | Each consumer group reads independently |
| Retention | Deleted after consumption | Retained for configured duration (days/weeks) |
| Replay | ❌ Can't re-read consumed messages | ✅ Consumers can seek to any offset |
| Ordering | Best-effort (FIFO with limits) | Guaranteed within a partition |
| Best for | Task queues, work distribution | Event sourcing, data pipelines, audit logs |

```python
# Example: Publishing an event to Kafka after order creation
from aiokafka import AIOKafkaProducer
import json

async def publish_order_event(order: dict) -> None:
    producer = AIOKafkaProducer(
        bootstrap_servers="kafka:9092",
        value_serializer=lambda v: json.dumps(v).encode(),
    )
    await producer.start()
    try:
        await producer.send_and_wait(
            topic="order-events",
            key=order["id"].encode(),  # Partition by order ID for ordering
            value={
                "event_type": "OrderCreated",
                "timestamp": "2026-04-06T18:00:00Z",
                "data": order,
            },
        )
    finally:
        await producer.stop()
```

> 🎯 **Aha!** The golden rule of async messaging: **design for at-least-once delivery**. Networks are unreliable, and message brokers may deliver a message more than once. Your consumers must be **idempotent** — processing the same message twice should produce the same result. Use a deduplication table or idempotency keys.

---

### Event-Driven Architecture: Event Sourcing & CQRS

These are advanced patterns you should know at an awareness level for interviews.

**Event Sourcing:**
Instead of storing the current state of an entity, you store the **sequence of events** that led to that state. The current state is derived by replaying events.

```
Traditional:  Account { balance: 150 }

Event Sourced:
  1. AccountOpened    { initial_deposit: 100 }
  2. MoneyDeposited   { amount: 200 }
  3. MoneyWithdrawn   { amount: 150 }
  // Current balance = 100 + 200 - 150 = 150
```

**Why?** Complete audit trail, ability to rebuild state at any point in time, natural fit for event-driven systems. **Tradeoff:** Complex to query (you need projections), eventual consistency, steep learning curve.

**CQRS (Command Query Responsibility Segregation):**
Separate your read model from your write model. Writes go through command handlers that emit events. Reads are served by optimized, denormalized read models (projections) built from those events.

```
Commands (writes) ──► Write Model ──► Events ──► Read Model ──► Queries (reads)
                     (normalized)                (denormalized,
                                                  optimized for
                                                  specific queries)
```

**Why?** Read and write workloads can be scaled independently. The read model can be optimized for specific query patterns without affecting write performance.

---

### GraphQL Basics

GraphQL lets the **client** define exactly what data it needs, solving REST's over-fetching and under-fetching problems.

```graphql
# Client requests exactly the fields it needs
query {
  user(id: 42) {
    name
    email
    orders(first: 5) {
      id
      total
      status
    }
  }
}
```

```json
// Server returns exactly what was requested — no more, no less
{
  "data": {
    "user": {
      "name": "Alice Chen",
      "email": "alice@example.com",
      "orders": [
        { "id": "1", "total": 59.99, "status": "delivered" },
        { "id": "2", "total": 124.50, "status": "shipped" }
      ]
    }
  }
}
```

**Resolver pattern:** Each field in your schema has a resolver function that knows how to fetch that data. GraphQL calls the right resolvers based on the client's query.

| Factor | REST | GraphQL |
|--------|------|---------|
| Data fetching | Fixed response shape per endpoint | Client defines response shape |
| Over-fetching | Common (get 20 fields, need 3) | Eliminated by design |
| Under-fetching | Common (need 3 calls for 1 view) | One query, one round-trip |
| Caching | Simple (HTTP caching by URL) | Complex (no URL-based caching) |
| Error handling | HTTP status codes | Always 200; errors in response body |
| Learning curve | Low | Medium-high |
| Best for | Simple CRUD, public APIs | Complex UIs, mobile (bandwidth sensitive) |

> 💡 **Interview Gold**: "When would you use GraphQL over REST?" Answer: When you have **multiple frontend clients** (web, mobile, TV) that need different slices of the same data. Instead of building a custom REST endpoint for each client, GraphQL lets each client query exactly what it needs. Facebook created it for exactly this reason — their mobile app needed much less data than the web app.

---

### WebSocket for Real-Time Communication

HTTP is request-response. WebSocket provides a **persistent, full-duplex connection** for real-time, bidirectional communication.

```
HTTP:      Client ──request──► Server ──response──► Client
           (connection closes)

WebSocket: Client ◄────────── persistent connection ──────────► Server
           (either side can send at any time)
```

**Use cases:** Chat applications, live dashboards, collaborative editing, multiplayer games, stock tickers, notification systems.

```python
# FastAPI WebSocket example
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

class ConnectionManager:
    def __init__(self):
        self.active: list[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active.remove(websocket)

    async def broadcast(self, message: str):
        for connection in self.active:
            await connection.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws/notifications")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"User says: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

**WebSocket vs SSE (Server-Sent Events):**

| Factor | WebSocket | SSE |
|--------|-----------|-----|
| Direction | Bidirectional | Server → Client only |
| Protocol | WebSocket (ws://) | HTTP (regular GET) |
| Reconnection | Manual | Automatic (built-in) |
| Best for | Chat, collaboration | Notifications, live feeds, dashboards |

> 🎯 **Aha!** If you only need server-to-client updates (dashboards, notifications), **SSE is simpler and sufficient**. It works over regular HTTP, handles reconnection automatically, and is easier to scale behind load balancers. Only reach for WebSocket when you need true bidirectional communication.

---

## Screen 6 — Quiz Time

Test your understanding with these interview-style questions. Try to answer before peeking at the solution!

---

### Q1: You're designing a REST API endpoint that allows a client to partially update a user's profile (e.g., change only the email address). Which HTTP method and status code combination is most appropriate?

A) `POST /api/v1/users/42` → 200 OK
B) `PUT /api/v1/users/42` → 200 OK
C) `PATCH /api/v1/users/42` → 200 OK
D) `PATCH /api/v1/users/42` → 201 Created

**Answer: C**

> **Explanation:** PATCH is specifically designed for partial updates — you send only the fields you want to change. PUT requires the full resource representation (it's a full replace). POST is for creating new resources, not updating existing ones. The correct status code is 200 OK (successful update), not 201 Created (which indicates a new resource was created). This distinction between PUT and PATCH is a very common interview question.

---

### Q2: Your e-commerce platform processes orders through multiple microservices (Orders, Inventory, Payments). You need to ensure that if payment fails, the reserved inventory is released and the order is cancelled. Which pattern best solves this?

A) Two-phase commit (2PC) across all three databases
B) Saga pattern with compensating transactions
C) Shared database across all three services
D) Synchronous REST calls with retry logic

**Answer: B**

> **Explanation:** The Saga pattern handles distributed transactions by defining a sequence of local transactions, each with a compensating action. If payment fails, the saga triggers compensations: release inventory, cancel order. Two-phase commit (2PC) doesn't scale well across microservices and creates tight coupling. A shared database defeats the purpose of microservices (independent data ownership). Synchronous retries don't address the fundamental problem of rolling back completed steps.

---

### Q3: You're paginating a social media feed that receives hundreds of new posts per minute. Users scroll through the feed using infinite scroll. Which pagination strategy is most appropriate?

A) Offset-based pagination with `?page=1&page_size=20`
B) Cursor-based pagination with `?after=eyJpZCI6IDQyfQ==&limit=20`
C) No pagination — return all posts and let the frontend handle it
D) Random sampling of posts per request

**Answer: B**

> **Explanation:** Cursor-based pagination is ideal for real-time feeds with frequent inserts. It provides stable results even when new posts are added between requests (no duplicates or missed posts). Offset-based pagination would cause items to shift between pages as new posts arrive — a user might see the same post twice or miss posts entirely. The cursor acts as a stable bookmark in the dataset, and the underlying query (`WHERE id > cursor LIMIT 20`) uses an index efficiently regardless of position.

---

### Q4: Your API gateway is routing requests to a downstream payment service that has become slow and unresponsive. Requests are timing out after 30 seconds, causing your gateway's thread pool to fill up. Which pattern prevents this cascade failure?

A) Increase the timeout to 60 seconds
B) Add more instances of the API gateway
C) Implement the circuit breaker pattern
D) Switch from REST to gRPC for faster communication

**Answer: C**

> **Explanation:** The circuit breaker pattern detects that the downstream service is failing and "opens the circuit" — subsequent requests fail fast with a fallback response instead of waiting for a timeout. This prevents the gateway's connection pool from being exhausted by slow calls. Increasing timeouts makes the problem worse. Adding more gateway instances delays the problem but doesn't solve it. Switching to gRPC might be marginally faster but doesn't help when the downstream service is fundamentally unresponsive.

---

### Q5: A mobile app needs to display a user's profile, their 3 most recent orders, and their notification count — all on a single screen. With a traditional REST API, this requires 3 separate HTTP requests. Which technology would let the client fetch all this data in a single request?

A) gRPC with server streaming
B) GraphQL with a nested query
C) REST with cursor-based pagination
D) WebSocket with a subscription

**Answer: B**

> **Explanation:** GraphQL allows the client to define a single query that fetches data from multiple related resources in one round-trip: `{ user(id: 42) { name, orders(first: 3) { id, total }, notificationCount } }`. This solves the "under-fetching" problem where REST requires multiple endpoints for a single view. gRPC could aggregate this but would require a specific server-side method. Pagination doesn't address the multi-resource problem. WebSocket is for real-time streams, not one-time data fetching. GraphQL was specifically designed by Facebook to solve this exact mobile-app data-fetching challenge.

---

## 🏁 Module Summary

You've covered the full landscape of API design and microservice architecture:

| Topic | Key Takeaway |
|-------|-------------|
| REST Design | Resources are nouns; HTTP methods are verbs; status codes tell the story |
| Pagination | Cursor for feeds and scale; offset for admin UIs and small datasets |
| Error Handling | RFC 7807 gives you a consistent, professional error format |
| Microservices | Start monolith; extract when you have a reason; sagas replace transactions |
| Communication | REST for public, gRPC for internal, async messaging for decoupling |
| Real-Time | WebSocket for bidirectional; SSE for server-push |

> 💡 **Interview Gold**: The meta-skill interviewers are looking for isn't memorization — it's **tradeoff analysis**. Every pattern in this module has pros and cons. When you say "I'd use X because of Y tradeoff," you sound like a senior engineer. When you say "I'd always use X," you sound like you read a blog post. Practice articulating *when* to use each pattern and *why*.

Good luck with your interviews. You've got this. 🚀
