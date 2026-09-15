# Python Coding Standards

Read [../SKILL.md](../SKILL.md) first for the universal principles this file makes
concrete in Python.

## File Organization

- Single-responsibility packages, one clear concern per module:
  `http/`, `auth/`, `aws/` in a library; `service/`, `repository/`, `client/`,
  `model/` layers in an application service.
- A file holds one class plus its close helpers — `mapping_service.py` holds only
  `MappingService` (orchestration); `mapping_repository.py` holds only
  `MappingRepository` (pure data access). Splitting a "mapping" concern into a
  service file and a repository file, rather than one `Mapping` god-class, is the
  norm, not the exception.
- Public API surfaced explicitly through `__init__.py` re-exports rather than callers
  reaching into internal modules — imports flow one direction (parent depends on
  child, never the reverse).
- Domain objects segregated by source system when a project talks to more than one
  external system (e.g. `model/crm/`, `model/billing/`, `model/mapping/`).
- No generic `utils`/`helpers` packages. Name a package after the functional purpose
  it serves (`auth/`, `retry/`), not what it vaguely contains — a grab-bag module is
  where dead code and duplicated logic go to hide.
- Imports at the top of the file. A function-level import needs a genuine reason —
  breaking a circular import, or avoiding an expensive/optional dependency on a cold
  path — not just habit; if you can't state the reason, it belongs at the top.

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

## Queries and Mutations Belong on the Model

A query or mutation that's really about one model lives on that model, not on a
generic helper class or a loose service function —
`TenantMapping.get_business_id_for_tenant()`, `Cadence.get_active_for_tenant()`, not
a `mapping_helpers.get_business_id(tenant_id)` reached for from three different call
sites. Callers ask the model, they don't reassemble the model's own query logic
themselves.

If a model method is actually hot enough to need caching, use `cachetools.func`
(e.g. `@ttl_cache`) rather than a hand-rolled module-level dict cache — the latter
has no eviction policy, no thread-safety story, and reinvents what the library
already gives you for free. Don't reach for caching pre-emptively; confirm it's
actually hot first.

A `model.save()` already commits (or the caller's transaction boundary already will)
— a `session.commit()` immediately after is redundant and a sign the transaction
boundary isn't clearly owned by one layer. Flag it in review.

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

Test-only and environment-specific concerns (a sandbox mode, a "don't actually send
this email in staging" switch) are solved at this same infra boundary — the client
or its configuration — not baked into the production data model as an extra column
or a branch in business logic. The model shouldn't know it's being tested against;
the adapter it talks to should be the thing that's different.

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
- Log every exception at the point it's caught, even inside retry logic — a
  `RetryableError` caught by a retry loop still gets a `logger.warning` with the
  attempt number and the underlying cause; swallowing it silently means no one can
  tell how often the call is failing until `RetryLimitReachedError` finally surfaces.
- Every response that isn't a 200 gets a log line stating why before it goes out —
  the `@app.exception_handler` that turns a domain exception into an `HTTPException`
  logs the reason and the ids involved, not just the resulting status code.
- Don't `except` and handle an error inside a repository or utility function just
  because that's where it was raised — it usually can't tell whether the failure is
  retryable or what status code it should become. Let it propagate to the
  route/handler layer (or the retry wrapper) that owns that decision; catching low is
  fine only to attach context and re-raise.

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
def update_status(obj: Widget, new_status: Status) -> None:
    obj.status = new_status
```

`resolve_crm()`-style dependency functions and FastAPI's `Depends()` are exactly the
mechanism for making this the boundary's job: they raise (typically an
`HTTPException`) before the route body ever runs if the object doesn't exist or the
user isn't authorized, so every function downstream of the route can be written as if
its arguments are already known-good.

When the same permission check needs to be reusable from more than one entry point
(a route that already checked it via `Depends()`, *and* a background job or internal
service call that doesn't sit behind that dependency), don't give the bare function a
flag to make the check optional:

```python
# BAD — a flag parameter means the function does two jobs, and every caller has
# to know which one it wants
def update_status(user, obj, new_status, skip_permission_check=False):
    if not skip_permission_check and user.id != obj.user_id:
        raise PermissionError()
    obj.status = new_status

# GOOD — the bare action and the checked wrapper are two small, single-purpose,
# independently testable functions
def update_status(obj: Widget, new_status: Status) -> None:
    obj.status = new_status

def update_status_with_permission_check(
    user: User, obj: Widget, new_status: Status
) -> None:
    if user.id != obj.user_id:
        raise PermissionError()
    update_status(obj, new_status)
```

A route already behind `Depends()` calls `update_status()` directly; a background
job or any caller that isn't already behind that dependency calls
`update_status_with_permission_check()`. Neither needs a flag to tell the other what
it wants.

## Use the SDK's Typed Client

Where a generated typed client exists for a service (the SDK), route every call to
that service through it — flag a hand-rolled HTTP callable, a raw string query, or a
parameter/return typed as `Any` where the client would already give a real type. The
whole point of generating the client is that callers stop hand-maintaining their own
untyped version of the same contract; bypassing it re-introduces the exact class of
bug (a typo'd field name, a wrong type) the client exists to catch at write time.

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
- Reuse the SDK's canonical Pydantic models rather than reinventing a parallel DTO
  for the same data. If you need extra fields, extend the SDK's base model — don't
  bypass it and hand-write a lookalike that will drift the moment the SDK's model
  changes.
- Prefer `None` over sentinel values (`-1`, `""`, `"N/A"`) at API and DB boundaries.
  A sentinel is a magic value every consumer has to know to check for; `None` is
  checked by the type system and by `is None`, not by convention.
- Use `NamedTuple` for anything with three or more positional elements, instead of a
  bare tuple or an untyped `dict`. `result[2]` tells a reader nothing; `result.status`
  does.

## State Transitions: DRY Into One Method

When a state transition is paired with a side effect that must never be forgotten
at a call site — an audit-trail write alongside a status change, a cache
invalidation alongside a save — DRY the transition and its side effect into a single
method, and have every caller go through it. The risk isn't duplicated logic in the
abstract; it's a second call site that changes the state but forgets the write that
was supposed to always come with it. If there's only one method capable of making
the transition, there's no way to make it without the side effect.

## GraphQL Field-Selection Contract

Respect what the client actually asked for. Use `from_model(info)` (or the
equivalent field-selection helper) to conditionally load relationships based on the
GraphQL selection set, rather than a blanket `selectinload()` that eagerly loads
every relationship regardless of whether the query requested it. A blanket eager
load defeats the entire purpose of a field-selection contract and turns a cheap
query into an expensive one for every caller, including the ones that didn't ask for
the extra data.

## Migrations (Alembic)

A migration is justified by real data or a real behavior change — never a "just in
case" migration or a speculative backfill run against data nobody's confirmed needs
it. If you can't point to the query or bug that requires the schema change, it isn't
ready to write yet.

## Extra Scrutiny on Code That Smells AI-Generated

Apply extra scrutiny to code with these tells, whether or not it's known to be
AI-generated — they're the same failure modes either way:

- A hallucinated API — a method or parameter that doesn't exist on the library/SDK
  being called; verify against the actual installed version, don't assume it compiles
  because it reads plausibly.
- An unjustified default value with no comment explaining where it came from.
- A "just in case" migration or backfill with no real data or behavior change behind
  it (see Migrations above).
- A `TYPE_CHECKING`-guarded import with no genuine circular-import reason — if the
  type is actually used at runtime, or there's no cycle to break, it doesn't need the
  guard.

## Restraint Before New Abstractions

Before building a new table, mutation, or helper, ask whether an existing one can
just be extended. A near-duplicate model or a second mutation that does almost what
an existing one does is usually a sign the existing one wasn't found or wasn't
considered, not that the new case is genuinely different. Push back on the new
abstraction in review before it's built and has to be maintained twice. The same
pass should flag dead code with no call sites — code that exists "in case it's
needed" is a liability, not an asset.

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
