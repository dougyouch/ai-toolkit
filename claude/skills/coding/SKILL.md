---
name: coding
description: Doug's personal multi-language coding standards — one class per file, small testable methods, adapter pattern at I/O boundaries, composition over inheritance. Use before writing or modifying application/library code (not one-off scripts) in any language. Loads a language-specific file for idiom-level detail; falls back to the universal principles here when no language file exists yet.
metadata:
  type: coding-standards
  source: distilled from dynamic-active-model, schema, db-purger, and client-api-builder (github.com/dougyouch)
---

# Coding Standards

A layered skill. This file holds the principles that hold across every language Doug
writes in. Each principle exists because it showed up independently, more than once,
across his own gems — this is not a generic style guide, it is a description of how
he actually builds software when there's no deadline forcing shortcuts.

Before writing or modifying code:

1. Identify the language.
2. Read the matching file in `languages/` if one exists (see table below) — it has
   concrete syntax, idioms, and code examples for that language.
3. If no language file exists yet, apply the universal principles below directly, and
   mention to Doug that this language doesn't have a dedicated file yet — it's worth
   adding once there's a real repo of his to draw the patterns from, rather than
   guessing at conventions he hasn't actually chosen.

| Language | File |
|---|---|
| Ruby | `languages/ruby.md` |
| Python | `languages/python.md` |
| Java | `languages/java.md` |
| *(others)* | not yet authored — apply the principles below |

## Universal Principles

### 1. One class or module per file

The filename mirrors the class/module name, and directory structure mirrors
namespace nesting. A single small entry-point file lazily loads the rest (e.g. Ruby's
`autoload`) so nothing pays for code it doesn't use, and adding a class never means
editing a growing central file.

### 2. Small files, on purpose

Favor many small files over a few large ones. A file that mixes more than one
responsibility, or that's grown long enough to lose at a glance, is a signal to
split it. This isn't just aesthetic: it's what keeps a file small enough that both a
human and an AI coding agent can hold the whole thing in working memory while editing
it, instead of reasoning about a fragment of a much larger file.

### 3. Small, focused methods

Public methods read as orchestration — a handful of calls to well-named private
helpers, not inlined logic. Each helper does one thing and is named for what it does
or checks (a boolean helper reads as a question). If a method needs a comment to
explain what its middle section does, that section is a helper that hasn't been
extracted yet.

### 4. Adapter pattern at every I/O boundary — the core testability move

Anywhere code reaches an external system — an HTTP client, a database connection,
the filesystem, a queue — wrap that boundary behind a small, swappable adapter with a
handful of methods. Production code includes/injects the real adapter; tests
substitute a fake or stub the boundary itself (e.g. WebMock at the wire), never the
business logic sitting behind it.

This is *the* pattern that makes the rest of the testing story possible: business
logic never talks to a concrete `Net::HTTP` or a concrete DB driver directly, so
tests never need to mock deep into a third-party library to exercise it.

### 5. Composition over inheritance

Prefer mixins/modules with an explicit hook (e.g. Ruby's `self.included(base)`) over
class hierarchies. Behavior is assembled from small, independently testable pieces,
not inherited from a base class that accumulates responsibilities over time.

### 6. Dependency injection for anything a test needs to control

Required collaborators come in through the constructor, not reached for globally.
Configuration-heavy objects are built through a builder/DSL rather than hardcoded
literals, so tests can construct realistic fixtures inline instead of loading files
or reaching into internals.

### 7. Code through configuration, not conditionals

Behavior differences between callers should be expressed as data — a hash, a
declared field, a DSL block — that one reusable function or class interprets, rather
than as branching logic bolted onto that function for every new case. A route
declared once, a plan built declaratively, a field's parsing rule declared once: each
turns "add a new case" into "add a config value" instead of "modify a function's
internals."

This is a direct testability lever, not just a style preference: a function whose
entire behavior is parameterized by its input is trivial to test exhaustively — feed
it every configuration permutation that matters — without forking the function or
stubbing internals for each branch. It's also what makes a function reusable across
call sites that need slightly different behavior: they pass different configuration
into the same code path instead of each getting their own near-duplicate function.

### 8. Errors: accumulate, then raise at a boundary

Prefer collecting errors into a structured errors object through a multi-step
process (parsing, validation) and raising only at an explicit, clearly-named boundary
method (a bang-method convention works well: the non-bang form reports, the bang form
raises). Don't raise on the first bad input if the caller would benefit from seeing
everything wrong at once. Define a small exception hierarchy scoped to the
library/module, with a common base error, and give specific exceptions the context
they need to be actionable (e.g. carry the response object, the offending record).

### 9. Validate at the boundary; let internals trust their inputs

Permission checks, existence checks, and input validation belong at the boundary
where untrusted input enters the system — an API controller, a queue consumer entry
point — not scattered through every downstream method the input eventually reaches.
A utility method like `update_status(user, object, new_status)` should not re-check
that `user` is allowed to do this or that `object` isn't null; by the time it's
called, the caller has already established both. If they haven't, the method should
fail loudly — a `NoMethodError`, a `NullPointerException`, an `AttributeError` — not
silently return or guard around the bad state.

This is deliberate, not an oversight. A defensive `if object.nil? then return` inside
a utility method doesn't fix a bug, it hides one: the caller had no business calling
with a null object, and swallowing that there means the real defect — whatever
produced a null where the invariant said there wouldn't be one — surfaces later,
further from its cause, or never surfaces at all. Let internal code fail hard and
immediately at the point an invariant breaks; that's what makes a failure traceable
back to its actual cause instead of a confusing downstream symptom.

This is not a license to skip validation — it means validation happens exactly once,
at the layer that owns it (a controller, a serializer, a request-validation layer),
and every layer beneath it is written as if that validation already happened, because
it did.

### 10. Tests mirror source, and exercise real behavior

Test files mirror the source tree file-for-file. Prefer asserting on real, observable
state changes over verifying that a mock was called — this is only possible because
of the adapter pattern above, which makes the actual boundary (network, filesystem)
cheap and safe to exercise in tests via stubs at the wire rather than deep mocks of
your own code.

### 11. Naming communicates role

A small, consistent vocabulary of suffixes tells a reader what a class does without
opening it: something that builds is a `Builder`, something that validates is a
`Validator`, something that wraps an external call is a `Client`/`Request`/`Adapter`,
something observing events is a `Subscriber`. Pick the vocabulary per project and use
it consistently rather than inventing a new word for the same role in every file.

### 12. Log every exception, and log every non-success response

Every time an exception is thrown, log it — including when the catch exists only to
retry. A retry that swallows the exception silently hides how often it's actually
failing; by the time the retry budget is exhausted, there's no record of why. Every
request that doesn't return a 200 back to its caller gets a log line stating why —
the actual reason (validation failure, downstream timeout, permission denial), not
just the status code — so an incident doesn't start from a blank stack trace.

### 13. Compose a checked wrapper instead of adding a flag to skip a check

When some call sites need a check (permission, feature-gating, rate limiting) before
an action and others don't, don't bolt a boolean onto the action to make the check
optional — `update_status(user, object, new_status, skip_permission_check)`. A flag
like that means the method is doing two jobs at once, and every caller now has to
know which one it needs and pass the right flag to get it. It's also a sign the
method was never properly laid out in the first place.

Split it into two small methods instead: a bare method that does exactly what it's
told (`update_status(object, new_status)`), and a separate, reusable method that
wraps it with the check and raises if the check fails
(`update_status_with_permission_check(user, object, new_status)` — something like
`raise PermissionError unless user.id == object.user_id`, then a call to the bare
method). A caller already behind a validated boundary calls the bare method
directly; a caller that needs the check reusable across multiple entry points calls
the wrapper. Both stay single-purpose and independently testable, and neither needs
a flag to know which behavior the other wanted.

### 14. Let exceptions bubble up to where they can be handled properly

Don't catch and handle an exception at the low-level method where it's thrown just
because it's convenient to do so there. That method usually can't tell whether the
failure is retryable, whether it should produce a 4xx or a 5xx, or whether a fallback
exists — only the caller, closer to the boundary (the controller, the job's retry
wrapper, the request handler), has enough context to decide. Catch low only to add
context and rethrow, or to log per Principle 12 before letting it propagate; catch
and swallow only at the boundary that can actually turn the failure into a decision —
a retry, an error status code, a fallback path.
