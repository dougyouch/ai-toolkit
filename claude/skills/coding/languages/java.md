# Java Coding Standards

Distilled from two of Doug's own services: `contacts` (a Dropwizard/Guice contact
management API) and `outreach` (a Dropwizard/Guice email/SMS/VOIP service
integrating Twilio, Postmark, Mailgun, and Nylas).

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

## Method Size & Decomposition

- Methods are narrow, typically 5-20 lines, each doing one thing.
- A method that orchestrates several checks or steps stays readable by delegating
  each step to a small private helper (`normalize()`, `toSuppression()`,
  `preferOrgOverGlobal()`) rather than inlining all of it.
- A manager method that wraps a single DAO call and nothing else is fine as a
  one-liner — not every method needs to do more than delegate.

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

Reference: `VoipClient`/`TwilioVoipClient` (outreach) for the SDK-wrapping case;
`GraphQlClient` (contacts) for an HTTP adapter that centralizes exception handling
for an internal service call.

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

    postmarkDomainManager.createDomain(user.getOid(), request.getDomain());
    return Response.ok().build();
}

// Manager — trusts that oid/domain are already valid; no re-checking here
public void createDomain(int oid, String domain) {
    domainDao.insert(oid, domain);
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
