# Ruby Coding Standards

These are library/gem-shaped standards — for migration/Rake-task conventions in a
specific application codebase, that's a separate, narrower concern and lives
elsewhere; don't conflate the two.

Read [../SKILL.md](../SKILL.md) first for the universal principles this file makes
concrete in Ruby.

## File Organization

- One class or module per file. Filename is the snake_case of the class/module name:
  `PurgeTable` → `purge_table.rb`, `NetHTTP::Request` → `net_http_request.rb`.
- A single entry-point file (`lib/<gem_name>.rb`) declares the module and `autoload`s
  every component, so nothing loads until it's used and the entry file stays tiny:

  ```ruby
  module MyGem
    autoload :Config,   'my_gem/config'
    autoload :Executor, 'my_gem/executor'
  end
  ```

- Subdirectories mirror nested modules: `lib/gem/parsers/common.rb` →
  `Gem::Parsers::Common`.
- Every file starts with `# frozen_string_literal: true`.
- A file mixing more than one class, or growing past roughly 150-200 lines, is a
  signal to split by concern — e.g. a router-style DSL class gets split into
  `router.rb`, `nested_router.rb`, `section.rb`, `query_params.rb` rather than staying
  one large class.

## Method Size & Decomposition

- Public methods are short (roughly 5-15 lines) and orchestrate; they call
  well-named private helpers instead of inlining logic.
- Boolean methods end in `?` (`skip_table?`, `unique_index?`, `foreign_tables?`).
- A `!`-suffixed method is the raising variant of a safer counterpart
  (`valid_model!` raises where the underlying check would just report).
- Extract a helper the moment a method does two distinct things — helper names should
  make the orchestrating method readable as a short list of steps.

## The Adapter Pattern (testability boundary)

This is the standard's centerpiece: wrap every external boundary (HTTP, DB,
filesystem) behind a small module with a handful of methods, included into the class
that needs it via a `self.included(base)` hook. Production code gets the real
implementation; specs stub the wire, not the abstraction.

```ruby
module MyGem
  module Http
    module Request
      def request(method:, uri:, body: nil, headers: {})
        # real Net::HTTP call lives here, and only here
      end

      def stream(method:, uri:, &block)
        # chunked variant, delegates to #request
      end
    end
  end

  class Client
    def self.included(base)
      base.include Http::Request
    end
  end
end
```

In specs, stub at the network boundary (WebMock's `stub_request`) rather than mocking
the adapter module itself — the adapter exists to make the real boundary thin enough
that stubbing it is cheap and realistic, so use that, don't route around it with mocks.

The adapter is also the natural place to enforce boundary security concerns once
(path traversal checks, default-secure SSL options) rather than at every call site.
Collaborators like a database connection or an options hash arrive through the
constructor rather than global configuration, and a setter meant purely for specs to
capture output (`error_io=`) is fine as long as it doesn't touch global state.

## Configuration-Driven Design

Where a family of similar cases would otherwise mean a family of similar methods
(or a growing `case`/`if` chain inside one method), declare the differences as data
and let one reusable class or method interpret that data uniformly. New cases become
new configuration, not new code paths — which also means they're covered by the same
tests that already exercise the interpreter, with just a new fixture.

```ruby
# Declarative — adding an endpoint is a config line, not a new method
class Client
  include ClientApiBuilder::Router
  base_url 'https://api.example.com'
  route :get, 'users/:id'
  route :post, 'users'
end
```

```ruby
# Declarative — the purge behavior is data the Executor interprets uniformly
plan = MyGem::PlanBuilder.build do
  base_table(:companies, :id)
  child_table(:employments, :company_id, batch_size: 2)
end
MyGem::Executor.new(database: db, plan: plan).purge!
```

```ruby
# Declarative — parsing rules declared per field, not hand-rolled per model
class Invoice
  include Schema::Model
  field :price,     :integer
  field :issued_on, :date
end
```

The testability payoff: a method whose behavior comes entirely from its
configuration input can be tested by feeding it every configuration permutation that
matters, in isolation, without touching the method itself or stubbing internals for
each case.

## Dependency Injection Beyond HTTP

- Required collaborators arrive through the constructor
  (`Executor.new(database:, plan:, options:)`), not via globals or class-level state.
- Configuration-heavy objects are built through a DSL/builder rather than literals,
  so specs can construct realistic fixtures inline:

  ```ruby
  plan = MyGem::PlanBuilder.build do
    base_table(:companies, :id)
    child_table(:employments, :company_id, batch_size: 2)
  end
  ```

- Serialization/formatting strategy is configurable at the class level as a symbol,
  method name, or `Proc` (client-api-builder's `body_builder` / `query_builder`),
  which makes it trivial to swap in a test-only formatter without subclassing.

## Error Handling

- Define one base error for the gem (`MyGem::Error < StandardError`), with specific
  subclasses carrying whatever context makes them actionable
  (`UnexpectedResponse` carries the response object; `ModelNotFound` carries the
  lookup key).
- During multi-step validation or parsing, accumulate into an errors object instead
  of raising on the first failure — let the caller see everything wrong at once.
- Raise only at an explicit boundary method, and prefer the bang-method convention to
  make that boundary visible in the caller: `parsed!` / `valid_model!` raise;
  `parsing_errors` / `errors` just report.
- Guard clauses (`return unless ...`, `next if ...`) over nested conditionals for
  early-exit validation.
- Log every exception at the point it's rescued, even when the rescue exists only to
  retry (`retry if attempts < max_attempts`) — logging the attempt count and the
  underlying error. A silent retry hides how often the call is actually failing until
  the retry budget is exhausted with nothing to explain why.
- Every response that isn't a 200 gets a log line stating the reason before it's
  returned — a controller rescuing into a 4xx/5xx logs the cause and the ids
  involved, not just the status code.
- Don't rescue in a model or service method just because that's where the error was
  raised — it usually can't tell whether the failure is retryable or what status the
  caller should get. Let it propagate to the controller (or the job's retry
  wrapper) that owns that decision; rescuing low is fine only to add context and
  re-raise.

## Avoid Defensive Programming Downstream of the Boundary

Permission and existence checks belong once, at the boundary that owns them — a
controller, a Pundit policy, a `find!`/`find_by!` that raises when the record isn't
there. A utility or service method that receives an already-validated object should
not re-check it:

```ruby
# BAD — defensive checks hide the real bug and fail silently
def update_status(user, object, new_status)
  return unless object
  return unless user&.can?(:update, object)
  object.update!(status: new_status)
end

# GOOD — the controller already authorized the user and loaded the record
# (or raised ActiveRecord::RecordNotFound / Pundit::NotAuthorizedError trying).
# This method trusts both and fails loudly (NoMethodError) if that trust is broken.
def update_status(object, new_status)
  object.update!(status: new_status)
end
```

The `return unless ...` guard-clause style earlier in this file (under Error
Handling) is for the validation/parsing layer itself deciding whether *its own* input
is well-formed — not for a downstream utility method re-guarding against a
precondition its caller was already responsible for. If `object` can legitimately be
`nil` by the time `update_status` is called, that's a bug in the caller to fix, not a
case for `update_status` to handle gracefully.

When the same permission check needs to be reusable from more than one entry point —
a controller action that's already behind Pundit, *and* a service call or Sidekiq job
that isn't — don't give the bare method a flag to make the check optional:

```ruby
# BAD — a flag parameter means the method does two jobs, and every caller has to
# know which one it wants
def update_status(user, object, new_status, skip_permission_check: false)
  raise PermissionError unless skip_permission_check || user.id == object.user_id
  object.update!(status: new_status)
end

# GOOD — the bare action and the checked wrapper are two small, single-purpose,
# independently testable methods
def update_status(object, new_status)
  object.update!(status: new_status)
end

def update_status_with_permission_check(user, object, new_status)
  raise PermissionError unless user.id == object.user_id
  update_status(object, new_status)
end
```

A controller action already behind Pundit calls `update_status` directly; a Sidekiq
job or any caller that isn't already behind that authorization calls
`update_status_with_permission_check`. Neither needs a flag to tell the other what it
wants.

## Modules & Composition

- Prefer `include`/`extend` with a `self.included(base)` hook over subclassing.
- Name mixins for the role they play, not just "stuff that got shared":
  `Helper` (shared private methods), `Validator`, `Builder`, `Subscriber`
  (notification observers), `Adapter`/`Request` (boundary wrapper).
- Where side effects need to be observable in tests without mocking internals, use
  `ActiveSupport::Notifications` (or an equivalent pub/sub) and assert against what a
  subscriber actually received.

## Testing

- RSpec. `spec/` mirrors `lib/` file-for-file — `lib/my_gem/executor.rb` ↔
  `spec/my_gem/executor_spec.rb`.
- Prefer asserting on real, observable state:

  ```ruby
  expect { subject }.to change { TestDB::Event.count }.by(-4)
  ```

  over verifying that a mock received a particular call.
- Stub the network at the wire (WebMock `stub_request`) rather than mocking the
  adapter class — the adapter's whole purpose is to make that safe and cheap.
- Build fixtures via the gem's own builder/DSL inline in the spec instead of loading
  fixture files, when the DSL supports it.
- Use `SecureRandom` for test data or dynamically-named modules/classes per example,
  to keep tests isolated from each other without a shared-state cleanup dance.

## Style

- 2-space indentation.
- Symbols over strings for hash keys.
- `&&`/`||` over `and`/`or`.
- String interpolation over concatenation.
- `# frozen_string_literal: true` (and `freeze` on standalone string constants).
- Descriptive, action-oriented class names (`PurgeTable`, not `TablePurger`).
