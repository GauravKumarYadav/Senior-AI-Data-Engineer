---
tags: [backend, fastapi, supplementary]
phase: supplementary
status: not-started
priority: low
---

# 🌐 Backend — FastAPI

> **Phase:** Supplementary (Good-to-Have) | **Priority:** Low
> **Related:** [[02 - Model Serving]], [[01 - LLM Fundamentals]], [[01 - Python Core]]

---

## Checklist

### FastAPI Core
- [ ] Request/response models with Pydantic v2
- [ ] Path params, query params, request body
- [ ] Dependency injection: `Depends()`, scoped dependencies
- [ ] Middleware: CORS, logging, timing, authentication
- [ ] Background tasks: `BackgroundTasks` for async post-processing
- [ ] File uploads, form data handling

### Async & Performance
- [ ] Async endpoints: `async def` with `await`
- [ ] Connection pooling: `asyncpg` for Postgres, `databases` library
- [ ] Streaming responses: `StreamingResponse` for LLM token output
- [ ] WebSockets: real-time communication (chat, live updates)
- [ ] Lifespan events: startup/shutdown for resource management

### Authentication & Security
- [ ] OAuth2 with JWT: `OAuth2PasswordBearer`, token validation
- [ ] API key authentication: header-based
- [ ] Rate limiting: `slowapi` or custom middleware
- [ ] HTTPS, security headers

### API Design
- [ ] REST resource naming: nouns, plural, nested resources
- [ ] Pagination: cursor-based vs offset-based
- [ ] Filtering & sorting: query parameter conventions
- [ ] Versioning: URL (/v1/) vs header-based
- [ ] Error responses: consistent format, meaningful messages
- [ ] OpenAPI spec: auto-generated docs, Swagger UI, ReDoc

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] FastAPI docs: https://fastapi.tiangolo.com/
- [ ] FastAPI tutorial (official — excellent)
