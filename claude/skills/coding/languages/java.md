# Java Coding Standards

Read [../SKILL.md](../SKILL.md) first for the universal principles this file makes
concrete in Java.

## File Organization

- One public class or interface per file, strictly.
- Packages follow domain/layer, not type: `data/sms/` groups the SMS DAO, manager,
  and entity together rather than scattering them into global `dao/`, `service/`,
  `model/` packages — a feature's files live next to each other.
- Multi-module Maven layout separates concerns at the build level, not just the
  package level: `OutreachApi` (REST resources), `OutreachData` (managers, DAOs,
  entities), and provider-specific modules (`VoipClient`, `EmailClient`, `SmsClient`)
  are each their own module with their own dependencies.
- Consistent suffixes make a file's role obvious from its name alone: `*Resource`
  (controller), `*Manager` (service/business logic), `*Dao` (data access), `*Client`
  (external adapter), `*Module` (Guice wiring), `*Exception`.
- No `utils`/`helper`/kitchen-sink packages or classes. A grab-bag `SmsUtils` that
  accumulates unrelated static methods is a smell — logic goes on the model it
  concerns (`SmsStatus.isTerminal()`, not `SmsUtils.isTerminalStatus(status)`) or on a
  class named for its one purpose. This is the single most repeated review objection —
  check for the right home before adding to (or creating) a generic helper.

## Method Size & Decomposition

- Methods are narrow, typically 5-20 lines, each doing one thing.
- A method that orchestrates several checks or steps stays readable by delegating
  each step to a small private helper (`normalize()`, `toSuppression()`,
  `preferOrgOverGlobal()`) rather than inlining all of it.
- A manager method that wraps a single DAO call and nothing else is fine as a
  one-liner — not every method needs to do more than delegate.
- Method parameters are ordered broadest scope to narrowest — the tenant id (in a
  multi-tenant system), then any intermediate scoping id, then the specifics of the
  call. A reader (and a diff) should see the blast radius of a call before its
  details.
- Capture the *why* behind a non-obvious decision in the code itself — a comment
  next to the line it justifies — not only in the PR description or a Slack thread.
  Those don't travel with the file; the next person to touch this method won't see
  them.

## The Adapter Pattern (testability boundary)

Define the external dependency as an interface owned by your code, implement it
against the concrete SDK, and bind the implementation via DI — never call a
third-party SDK directly from business logic.

```java
// The interface your code depends on
public interface VoipClient {
    String generateAccessToken(String identity);
    CallStatus getCallStatus(String callSid);
    // ...
}

// The concrete implementation — the only place that imports the Twilio SDK
public class TwilioVoipClient implements VoipClient {
    @Inject
    public TwilioVoipClient(@Named("com.et.twilio.account_sid") String accountSid, ...) {
        ...
    }
    // implements every VoipClient method against com.twilio.*
}

// Wiring — swap the binding in test modules for a fake
public class TwilioVoipClientModule extends AbstractModule {
    @Override
    protected void configure() {
        bind(VoipClient.class).to(TwilioVoipClient.class).asEagerSingleton();
    }
}
```

Business logic (managers, resources) depends only on `VoipClient`, never on
`TwilioVoipClient` or the Twilio SDK directly. A test module rebinds `VoipClient` to
a fake implementation, so nothing under test needs to mock deep into a third-party
client library.

The interface itself must stay provider-agnostic — no Twilio type, status code, or
naming convention should leak into `VoipClient`'s method signatures or return types.
If swapping the provider would mean changing the interface, the abstraction isn't
actually generic yet.

## Configuration-Driven Design

Encode business rules as enum data plus small static methods that interpret it,
rather than as `if`/`else` chains scattered across callers.

```java
public enum SmsStatus {
    PENDING, QUEUED, SENT, DELIVERED, FAILED, ...;

    public static int rank(String status) { ... }               // enforces the state machine
    public static List<String> overridableBy(String status) { ... } // which prior states this may replace
    public static boolean isTerminal(String status) { ... }
}
```

`overridableBy()` is then passed straight into a DAO's `UPDATE ... WHERE status IN
(?)` clause, so a stale webhook callback can never clobber a newer status — the state
machine is enforced by data (`SmsStatus`'s ranking) interpreted generically by the DAO,
not by a growing pile of manager-level conditionals checking "is this transition
allowed."

Configuration values themselves (API keys, feature toggles) are bound once through DI
(`@Provides @Named("com.et.twilio.account_sid")`) and injected wherever needed,
rather than read from environment/config ad hoc at each call site.

Enums and `public static final` constants over magic strings, always. A bare
`"FAILED"` string scattered across callers can't be enforced by the compiler and
drifts silently if one call site typos it; `SmsStatus.FAILED` can't.

## Dependency Injection

- Constructor injection via `@Inject`, never field injection — a class's
  dependencies are visible in its constructor signature, and a test can construct it
  directly with fakes.
- Use `@Named`/custom qualifier annotations (`@OrgId`, `@AuthUser`) to inject scoped
  or contextual values without resorting to thread-locals or static lookups.
- When one manager needs another, inject it — but watch for circular Guice
  dependencies; break the cycle by injecting the narrower collaborator (a DAO)
  instead of the manager that would create the cycle.

## Avoid Defensive Programming Downstream of the Boundary

Permission and validation checks happen once, at the resource (controller) layer,
before the manager is ever called:

```java
// Resource (boundary) — validates, then delegates
public Response createDomain(@Auth User user, DomainRequest request) {
    checkArgumentGivingTreeOwner(user);
    WebPreconditions.checkArgument(user.isSuperUser(), "not authorized");
    validateDomainNameForUser(user, request.getDomain());

    postmarkDomainManager.createDomain(user.getTenantId(), request.getDomain());
    return Response.ok().build();
}

// Manager — trusts that tenantId/domain are already valid; no re-checking here
public void createDomain(int tenantId, String domain) {
    domainDao.insert(tenantId, domain);
}
```

Managers assume their inputs are valid, and DAOs don't re-validate what the manager
already guaranteed. A `BaseResource` base class centralizes the reusable permission
helpers (`checkArgumentGivingTreeUser`, `isValidVolunteerUser`) so every resource
calls the same checks the same way, instead of each resource — or worse, each
manager — reimplementing its own null/permission guard.

Where a lower layer does still guard, it's not defensive redundancy but a concurrency
control the boundary layer can't provide — e.g. the `WHERE status IN (?)` guard from
`SmsStatus.overridableBy()` protects against a race between two concurrent webhook
deliveries, which no amount of controller-level validation up front could catch.

When the same permission check needs to be reusable from a second call site that
isn't already behind a resource — a Sidekiq-style background job, an internal
service call — don't give the manager method a boolean to make the check optional
(`createDomain(int tenantId, String domain, boolean skipPermissionCheck)`). That's a sign
the check belongs in its own method. Add a second, checked entry point instead
(`createDomainWithPermissionCheck(User user, String domain)`) that runs the check —
reusing the same `checkArgumentGivingTreeUser`-style helper the resource layer uses —
and then delegates to the bare `createDomain(int, String)`. Each call site picks the
method that matches whether it's already behind a validated boundary.

## Error Handling

- Define custom exceptions per failure domain (`EmailApiException`,
  `TestOrgSendLimitExceededException`, `UnsubscribeTokenException`), suffixed
  `*Exception`, generally unchecked — don't force every caller up the stack to
  declare or catch something it can't meaningfully recover from.
- Exceptions carry the context needed to act on them (an SDK exception wrapping the
  provider's status code and response body, not just a message string).
- Centralize translation from internal exceptions to HTTP responses in one place —
  a JAX-RS `ExceptionMapper<T>` per exception type — rather than a `try`/`catch` in
  every resource method.
- Catch the specific exception a call can actually throw, never a bare `Exception`
  (or `Throwable`). A blanket catch swallows bugs it was never meant to handle
  alongside the failure it was written for.
- A failure must stay visible. Logging it and moving on — "log-and-vanish" — is
  banned: either send it to Sentry (or the project's equivalent) or rethrow it.
  A caught-and-logged exception with no rethrow and no alerting is a bug that will
  only be found by a customer.
- Log and error messages are human-readable and carry the ids involved (the tenant
  id, the record id, the external call's identifier) — not a bare stack trace or a
  message that only makes sense next to the line that threw it.
- Log every exception at the point it's caught, even when the catch exists only to
  retry (`RetryableSendException` caught inside a retry loop still gets a `log.warn`
  with the attempt count) — a silent retry hides how often the call is actually
  failing until the retry budget runs out with no trail explaining why.
- Every response that isn't a 2xx gets a log line stating the reason before it's
  returned — a `JAX-RS` `ExceptionMapper` logs the mapped exception's message and the
  ids involved when it produces the 4xx/5xx, not just the status code.
- Don't catch an exception in a DAO or utility method just because that's where it's
  thrown. That layer can't tell whether the failure is retryable or what status code
  it should become — only the resource/manager layer, or a dedicated
  `ExceptionMapper`, has that context. Catch low only to wrap with more context and
  rethrow; catch-and-decide belongs at the boundary.

## Multi-Tenancy: Every Query Scoped by Tenant ID

In a multi-tenant system, every query and mutation that touches tenant-scoped data
is filtered by the tenant id. A missing tenant id filter is treated as a
correctness/security bug class — one tenant reading or writing another tenant's
rows — not a style nit, and is called out with the same weight as a SQL injection
finding. This is also why the tenant id is the first parameter in method signatures
(see parameter ordering above): it's the thing a reviewer must be able to confirm is
present and threaded through before looking at anything else.

## Typed POJOs over JsonNode/Map

Model external and internal payloads as typed POJOs, not `JsonNode` or
`Map<String, Object>` passed around and re-parsed at each call site. Validation
(required fields, format, ranges) lives on the model — a constructor or a
`validate()` method the model owns — not scattered across every place that happens
to read the map. A typed field either exists and is well-formed by the time it
reaches business logic, or the model failed to construct in the first place.

## Performance: Reuse and Batch

- Reuse expensive-to-construct objects (`ObjectMapper`, HTTP client instances,
  compiled patterns) as `static final` fields — never re-instantiated per call or
  per request.
- Prefer bulk/batch operations over per-row round trips to a database, search index,
  or distributed job. Count the actual number of round trips a code path makes, not
  just whether it "looks batched" — a loop calling a batched-looking API once per row
  is still N round trips. This applies with particular force to Spark jobs and
  Elasticsearch calls, where triggering an action (a Spark action, an ES request)
  inside a per-row loop is an easy and expensive mistake to make.

## Migration & Schema Hygiene

- Primary keys are `bigint`, not `int` — an `int` PK is a migration waiting to
  happen once the table grows.
- Tables are plural, models are singular (`accounts` table ↔ `Account` model).
- Foreign keys are required (`NOT NULL`) unless the relationship is genuinely
  optional — an optional-looking FK is usually a missing backfill, not a real
  business rule.
- A migration touches only the schema it's justified by — no incidental diffs
  (reordered columns, unrelated index changes) riding along with the change that
  was actually asked for.
- Renaming a column or table on a large, live table can hit lock/timeout limits;
  treat a rename as a multi-step migration (add new, backfill, cut over, drop old),
  not a single blocking `ALTER`.
- Indexing is deliberate — added because a known query needs it, not defensively
  "just in case," and not omitted because "it's a small table today."

## Restraint Before New Abstractions

Before adding a new client, cache, or abstraction layer, check whether one already
exists that can be extended. A second bespoke Redis cache wrapper or a second HTTP
client for a service that already has one is a sign the existing one wasn't found,
not that a new one was needed. Push back on the new abstraction in review — ask the
author to point at what they checked and why it didn't fit — before it gets built
and now has to be maintained twice.

The same restraint applies to duplicated logic that isn't yet a new abstraction: DRY
it into one method or a shared, purpose-named package rather than letting the same
transition/parsing/formatting logic exist in two places that will inevitably drift.
This is not license to create a `utils` package for it — see File Organization above.

## Naming Conventions

- `*Resource` — REST controller/boundary, extends a shared `BaseResource`.
- `*Manager` — business logic / service layer.
- `*Dao` — data access, one per entity.
- `*Client` — adapter wrapping an external SDK/service.
- `*Module` — Guice wiring for a feature/provider.
- `*Exception` — domain-specific exception.
- `*Transcoder` — converts between representations (e.g. domain object ↔ JSON for
  search indexing).

## Testing

- JUnit 5 (Jupiter), Mockito (`@ExtendWith(MockitoExtension.class)`, `@Mock` fields)
  for the collaborators that should stay faked.
- For manager-level tests, prefer a real manager against a real (in-memory H2)
  database over mocking the DAO layer — only the true external boundary (an SDK
  client) gets mocked. A shared base test class provides the in-memory `Jdbi`
  connection and rolls back state after each test.
- Use a builder for constructing test fixtures (`Contact.newBuilder()...build()`)
  instead of long constructor argument lists repeated across tests.
- Aim for meaningful coverage as a floor, not a target — these repos enforce a
  minimum via JaCoCo, but the real goal is that every manager/DAO path has a real
  (not mocked-into-meaninglessness) test exercising it.
