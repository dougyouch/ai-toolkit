# Python Coding Standards

Distilled from Doug's own repos: `et-python-sdk` (an SDK — the primary source for the
HTTP-adapter pattern) and `et-crm-integrations` (a FastAPI service showing the
same principles applied at application scale).

Read [../SKILL.md](../SKILL.md) first for the universal principles this file makes
concrete in Python.

## File Organization

- Single-responsibility packages, one clear concern per module:
  `et/http/`, `et/auth/`, `et/aws/` in the SDK; `service/`, `repository/`, `client/`,
  `model/` layers in the CRM integration service.
- A file holds one class plus its close helpers — `mapping_service.py` holds only
  `MappingService` (orchestration); `mapping_repository.py` holds only
  `MappingRepository` (pure data access). Splitting a "mapping" concern into a
  service file and a repository file, rather than one `Mapping` god-class, is the
  norm, not the exception.
- Public API surfaced explicitly through `__init__.py` re-exports rather than callers
  reaching into internal modules — imports flow one direction (parent depends on
  child, never the reverse).
- Domain objects segregated by source system when a project talks to more than one
  external system (e.g. `model/crm/`, `model/et/`, `model/mapping/`).

## Method Size & Decomposition

- Methods stay focused, roughly 15-25 lines, one concern each.
- Complex operations delegate to private helpers, marked with a leading underscore:
  `_fetch_and_shape()`, `_prepare()`, `_handle_response()`, `_read_nango_settings()`.
- A method that both fetches and transforms is a signal to split fetch from shape —
  e.g. a public method delegates entirely to a private one that does the real work,
  so the public method reads as a one-line policy statement (add caching, apply
  auth) around the private implementation.
- 404/not-found from an external call is handled by returning `None`, not raising —
  reserve exceptions for actually-exceptional conditions, not expected absence.

## The Adapter Pattern (testability boundary)

Wrap every external HTTP/API boundary behind a base client class that owns the
request/response pipeline once; concrete clients subclass it and add nothing but
endpoint methods.

```python
class BaseHttpClient:
    def _prepare(self, method, path, **kwargs):
        ...  # build the request, apply retry/auth config

    def _handle_response(self, response):
        ...  # raise typed errors, or return parsed body

    def request(self, method, path, **kwargs):
        req = self._prepare(method, path, **kwargs)
        return self._handle_response(self._send(req))


class HttpClient(BaseHttpClient):
    def _send(self, req):
        return httpx.request(**req)


class AsyncHttpClient(BaseHttpClient):
    async def _send(self, req):
        return await self._async_client.request(**req)
```

The sync and async variants share the entire pipeline (retries, error translation)
and differ only in how the request is actually sent — the boundary that would
otherwise be hardest to test is the *only* thing that changes between them.

In tests, patch the underlying transport (`httpx.request`) or build a mock response
with a small test helper (`mock_http_response(...)`), rather than mocking the client
class itself — same principle as stubbing at the wire in Ruby's WebMock usage: the
adapter exists to make that safe.

Real examples from Doug's repos:

- **et-python-sdk** — `BaseHttpClient` (`et/http/_base.py`) centralizes
  `_prepare`/`_handle_response`/retry logic; `HttpClient` and `AsyncHttpClient`
  inherit it and only override the send mechanism. Tests patch `httpx.request`
  directly or use `mock_http_response()` from `et/testutils/helpers.py`.
- **et-crm-integrations** — `EvertrueAPIClient` / `NangoAPIClient` wrap their
  respective external APIs behind a client class; `RETRYABLE_STATUS_CODES` is
  declared once and checked by membership, not duplicated per client.

## Configuration-Driven Design

Prefer a declared config object (a Pydantic model, a dict, a registry) that generic
code interprets, over hardcoded values or branching baked into a function.

```python
class HttpClientConfig(BaseModel):
    max_attempts: int
    retry_backoff_seconds: float
    retryable_status_codes: set[int]


class DnaClient(BaseHttpClient):
    __config_key__ = "dna"  # HttpComponent.load() applies config by this key
```

A client declares *which* configuration applies to it (`__config_key__`) and the
base class' generic retry/request logic reads that configuration — adding a new
client with different retry behavior is a new config value, not a new branch in the
base client.

The same idea shows up as plain data structures when a full config model would be
overkill: `TOPIC_VERSIONS` (a dict mapping topic name to version) validated by
membership rather than an `if/elif` chain per topic; `NUMERIC_GIVING_FIELDS` /
`BOOLEAN_GIVING_FIELDS` (frozensets) driving field-type handling by lookup instead of
per-field conditionals; enums for domain state (`Status.DRAFT` / `Status.PUBLISHED`)
backed by a plain string column so new states don't require a schema migration.

## Dependency Injection

- FastAPI's `Depends()` for request-scoped collaborators — a plain function
  (`get_dna_client()`, `resolve_crm()`) builds the dependency, and routers ask for it
  by type rather than importing/constructing it directly.
- Dependencies that are expensive to build are lazy singletons — constructed on
  first use inside the dependency function, not at import time.
- Repositories take the DB `Session` as an explicit parameter rather than opening
  their own connection, so the caller controls the transaction boundary and tests
  can inject an in-memory/rollback-scoped session.
- In tests, override the dependency at the app level
  (`app.dependency_overrides[require_super_user] = lambda: SUPER_USER`) rather than
  monkeypatching the function it replaces — the override is explicit, scoped to the
  test, and reads as configuration rather than a patch.

## Error Handling

- Define a small exception hierarchy per concern, distinguishing retryable from
  terminal failure explicitly (`RetryableError` vs. `RetryLimitReachedError`), so
  retry logic can branch on exception type instead of inspecting status codes
  ad hoc.
- Base error classes carry the data a caller needs to act on the failure —
  `ResponseError` wraps the actual `httpx.Response`; a conflict error
  (`StaleRevisionConflict`) carries the current revision, the attempted update, and
  who made it, as plain attributes, not just a message string.
- Define domain exceptions in the module that raises them, not in a shared
  `exceptions.py` dumping ground — `DraftAlreadyExistsError` lives next to the
  repository method that raises it.
- Translate domain exceptions to transport-level responses (HTTP status codes) in one
  central place (a registered `@app.exception_handler(ExceptionType)` per exception
  type), not scattered `try/except` blocks in every route.

## Avoid Defensive Programming Downstream of the Boundary

Permission and existence checks belong once, at the boundary — a FastAPI dependency,
a route handler — not re-checked inside every service/utility function that receives
an already-validated object:

```python
# BAD — defensive checks hide the real bug and fail silently
def update_status(user, obj, new_status):
    if obj is None:
        return
    if not user.can_update(obj):
        return
    obj.status = new_status

# GOOD — a Depends() dependency already 404'd if the object was missing, and
# another already 403'd if the user lacked permission. This function trusts both
# and fails loudly (AttributeError) if that trust is ever broken.
def update_status(user: User, obj: Widget, new_status: Status) -> None:
    obj.status = new_status
```

`resolve_crm()`-style dependency functions and FastAPI's `Depends()` are exactly the
mechanism for making this the boundary's job: they raise (typically an
`HTTPException`) before the route body ever runs if the object doesn't exist or the
user isn't authorized, so every function downstream of the route can be written as if
its arguments are already known-good.

## Type Hints & Models

- Full type annotations everywhere, including generics for reusable abstractions
  (`Consumer[PayloadT]`, `Message[PayloadT]`).
- Pydantic models for anything crossing a boundary: request/response DTOs, config
  objects. Keep the ORM model (SQLAlchemy) and the DTO (Pydantic) as separate
  classes even when their fields overlap — the DTO is the contract, the ORM model is
  storage detail.
- Use a lenient base model (`ConfigDict(extra="allow")`) for parsing external APIs
  you don't control, and a strict model for your own internal schemas — don't let an
  upstream API adding a field break your parsing.

## Naming Conventions

- **Service** — business orchestration (`MappingService`).
- **Repository** — pure data access, no business logic (`MappingRepository`).
- **Client** — wraps an external API (`EvertrueAPIClient`, `AuthClient`).
- **Config** / **Component** — declared configuration and the thing that applies it
  (`HttpClientConfig`, `HttpComponent`).
- **Consumer** / **Producer** — message-queue handlers (`ContactsImportConsumer`).
- **Error** — domain exceptions, always a noun phrase describing what went wrong
  (`DraftAlreadyExistsError`, `StaleRevisionConflict`).
- Private helpers: leading underscore (`_prepare`, `_fetch_and_shape`).

## Testing

- pytest, with `unittest.mock` for patching and pytest fixtures for setup/teardown.
- Mark slow/integration tests explicitly (`@pytest.mark.integration`) so the default
  run stays fast and network-free; unit tests run against an in-memory SQLite DB
  rather than a real one.
- Patch at the transport boundary (`httpx.request`) or the injected dependency
  (FastAPI `dependency_overrides`), not at the business-logic call site — this keeps
  tests honest about what's actually being faked.
- Small helper functions for building consistent test fixtures with overrideable
  defaults (`_row(**overrides)`) beat copy-pasted literal dicts across test cases.
