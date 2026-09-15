# Ruby Coding Standards

Distilled from four of Doug's own gems: `dynamic-active-model`, `schema`, `db-purger`,
and `client-api-builder`. These are library/gem-shaped standards — for
migration/Rake-task conventions in a specific application codebase, that's a separate,
narrower concern and lives elsewhere; don't conflate the two.

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

Real examples from Doug's gems, for reference:

- **client-api-builder** — `ClientApiBuilder::NetHTTP::Request` wraps `Net::HTTP`
  behind `request` / `stream` / `stream_to_file`, included into any class that
  `include`s `Router`. Specs use WebMock against real HTTP calls, not mocks of the
  adapter. The adapter also owns boundary security concerns (path traversal checks,
  default-secure SSL options) — the boundary is the natural place to enforce them
  once, rather than at every call site.
- **db-purger** — `Executor` takes `database`, `plan`, and `options` via constructor
  injection, and exposes an `error_io=` setter purely so specs can capture output
  without touching global state.
- **dynamic-active-model** — `Database` and `Factory` take `connection_options`
  through the constructor rather than reading global configuration.

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

Real examples from Doug's gems: client-api-builder's `route`/`section` DSL generates
every client method from declared endpoints rather than hand-written per-endpoint
code; db-purger's `PlanBuilder` declares a purge plan as data that `Executor`
interprets, including nested child/foreign tables, without the executor knowing
anything about a specific schema; schema's field declarations drive parsing and
validation from a declared type instead of per-field custom logic.

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
