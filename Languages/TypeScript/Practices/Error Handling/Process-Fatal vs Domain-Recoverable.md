---
title: Process-Fatal vs Domain-Recoverable
tags: [error-handling, typescript, node, fail-fast, fail-safe]
summary: Invariant violations and unhandled async failures should crash the process (fail-fast); expected failures are typed and handled (fail-safe). Process-fatal is a stability mechanism, not a cleanliness one.
keywords: [unhandled rejection, uncaught exception, fail fast, fail safe, process exit, supervisor, restart policy]
---

*A process whose invariants are broken is producing wrong output. Crashing it is the safe default; trying to keep it running is the dangerous one.*

# Process-Fatal vs Domain-Recoverable

> **Rule:** when an invariant is violated or an asynchronous failure goes unhandled, the process must crash. When an *expected* failure occurs (one the spec lists), it is handled in-band by typed error handling. Mixing the two — catching invariant violations and "logging and continuing" — produces wrong output indefinitely.

A useful frame: there are exactly two categories of failure.

1. **Domain-recoverable failures** — outcomes the spec explicitly anticipates: validation rejected the input, the upstream service returned 503, the user provided a malformed token, the database deadlocked. These are normal operating conditions for the service. They have a typed shape, they are handled by the caller, and they leave the system in a consistent state. The pattern lives in [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]].

2. **Process-fatal failures** — situations that *should not occur*: a `never` branch was reached, an awaited promise rejected with no handler, an internal data structure has an impossible state, an out-of-memory condition. Continuing to run when one of these occurs means the program is producing output without satisfying its preconditions, which is the technical definition of "wrong."

The right response to category 2 is to crash. A supervised process — a container, a systemd unit, a Kubernetes pod, a PM2 supervisor — restarts the crashed process from a known-good state. The wrong response is to catch, log, and continue, because the caught state is by definition one the rest of the program is not designed to operate from.

## Node's structural support for fail-fast

Node.js makes process-fatal an explicit category through several mechanisms:

### Unhandled promise rejections

A promise that rejects with no `catch` handler triggers the `unhandledRejection` event. Modern Node defaults to `--unhandled-rejections=throw`, which converts the unhandled rejection into an uncaught exception and exits the process. Configure this explicitly in production:

```sh
node --unhandled-rejections=throw app.js
```

The alternative — catching `unhandledRejection` and "logging and continuing" — has a long track record of masking real bugs. The promise rejected for a reason the program isn't equipped to understand. Logging it and continuing pretends the program still understands its state.

### Uncaught exceptions

`process.on('uncaughtException', handler)` exists, but the handler should be limited to:

- Emitting a structured log entry.
- Flushing telemetry.
- Calling `process.exit(1)` or letting Node exit naturally.

What it should *not* do is "swallow and continue." The exception bubbled to the top because no in-band handler claimed it. By definition, the program is in a state no handler is designed to operate from.

### Worker thread isolation

For services that need to survive partial failure of *user code* (a plugin host, a sandbox), the right tool is worker thread isolation, not catch-and-continue. The worker crashes; the main process notices, decides whether to spawn a replacement, and stays consistent. This is fail-fast at the worker boundary, fail-safe at the supervisor boundary.

## What "fail-safe" looks like at the product level

"Process-fatal" doesn't mean "user-visible failure." It means "this process exits and a supervisor restarts a fresh one." The user-facing impact depends on the surrounding architecture:

- **Stateless backend services**: the load balancer routes the next request to another replica or to the restarted one. The user sees a brief blip, no incorrect behavior.
- **Stateful services with replication**: failover handles it.
- **Long-running jobs**: a checkpointing strategy lets the restarted worker resume.
- **Real-time clients**: the connection drops; client reconnect logic handles it.

The product-level property is **fail-safe** — the user gets a known-bad outcome (a transient error, a retry, a brief unavailability) rather than an unknown-wrong outcome (incorrect data, a half-completed transaction, a security failure). Fail-fast at the process level is what makes fail-safe at the product level achievable.

## Process-fatal patterns

In TypeScript code, the patterns that *should* be process-fatal:

```ts
// Unreachable code — type system invariant violated
default: {
  const _exhaustive: never = shape;
  throw new Error("Unreachable");
}

// Internal-state invariant
if (this.#state !== "ready") {
  throw new Error(`Operation called in state ${this.#state}, expected "ready"`);
}

// Hard precondition that downstream validation should have caught
if (amount <= 0) {
  throw new Error(`amount must be positive, got ${amount}`);
}

// Top-level entry point
async function main() { /* ... */ }
main().catch(err => {
  // Log structured. Then exit.
  console.error({ event: "fatal", error: err });
  process.exit(1);
});
```

The patterns that *should not* throw:

```ts
// User-supplied data — that's expected failure, return Result
function parseUserInput(raw: unknown): Result<ParsedInput, ParseError> { /* ... */ }

// Network call — handle expected failure, propagate
async function callUpstream(): Promise<Result<Response, UpstreamError>> { /* ... */ }
```

## Composing with observability

Fail-fast is only safe when the process's death is *visible*. The crash must:

- Emit a structured fatal log line before exit (with stack trace, request context if available, version metadata).
- Increment a fatal-crash metric so the rate is observable.
- Trigger an alert if the rate exceeds a known floor.

Otherwise the supervisor restarts the process silently, the symptom is masked, and the bug compounds. This connects to [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]] — fatal-crash rate is a budget burn rate.

## Composing with other practices

- [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]] — what *not* to throw for.
- [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]] — typed linting catches the structural cause of unhandled rejections before runtime.
- [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]] — `useUnknownInCatchVariables` forces catch handlers to narrow before they read, which is what catch handlers should do whether they recover or escalate.
- [[Engineering Philosophy/Principles/Layered Risk Categories]] — process-fatal-vs-recoverable is a runtime-layer concern; getting it wrong is a runtime-layer failure.
- [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]] — crash rate against the budget.

## Related

- [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]]
- [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]]
- [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]]
- [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]]
- [[Languages/Rust/Practices/Error Handling/Result vs Panic]]
- [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]]
