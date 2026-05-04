---
title: Error Handling Bundle
tags: [bundle, error-handling, typescript]
summary: "Pre-bundled error handling for TypeScript: Result for expected failures, process-fatal for invariant violations, and runtime validation at every external boundary."
source_trigger: "Handling errors and runtime validation"
bundles: [Result Pattern for Expected Failures, Process-Fatal vs Domain-Recoverable, Runtime Validation at Boundaries]
---

*The through-line: expected failures are typed and handled, invariant violations crash the process, and external data is `unknown` until a schema parses it — three disciplines, one coherent error model.*

# Error Handling Bundle (TypeScript)

Read this bundle when handling errors or wiring runtime validation in a TypeScript service. It bundles the three notes that decide what crashes, what is typed and recovered, and what data is allowed to enter the typed core in the first place.

---

## Result Pattern for Expected Failures

*Failures the spec lists belong in the type system. The compiler can force every caller to handle them. `throw` is for failures the spec never anticipated.*

> If the spec says "this can fail with X, Y, or Z," the function returns a `Result<T, E>` with `E` enumerating those failure modes. The compiler then refuses to let any caller forget to handle them. `throw` is reserved for situations the spec did not anticipate — invariant violations, programmer errors, broken assumptions.

### The pattern

```ts
type Ok<T> = { ok: true; value: T };
type Err<E> = { ok: false; error: E };
type Result<T, E> = Ok<T> | Err<E>;

const ok = <T>(value: T): Ok<T> => ({ ok: true, value });
const err = <E>(error: E): Err<E> => ({ ok: false, error });
```

A function that can fail with a known set of reasons returns one:

```ts
type TransferError =
  | { kind: "insufficient_funds"; available: Cents; required: Cents }
  | { kind: "account_frozen"; account: AccountId }
  | { kind: "amount_invalid"; amount: Cents };

function transferFunds(
  from: AccountId, to: AccountId, amount: Cents,
): Result<Receipt, TransferError> {
  if (amount <= 0) return err({ kind: "amount_invalid", amount });
  // ...
  return ok(receipt);
}
```

Every caller is now compelled by the type system to inspect `result.ok` before reading `result.value`. The discriminated-union exhaustiveness pattern (see [[Bundles/TypeScript/Module-Design]]) applies directly: a switch on `error.kind` exhaustiveness-checks, so adding a new failure variant becomes a compile error at every consumer.

### Why this beats throw-and-catch

- **The signature lies with throw.** `function transferFunds(...): Receipt` looks total; the failure shape is invisible.
- **The catch site loses the type.** `catch (e)` is `unknown`; the handler must guard-narrow.
- **Forgetting to catch is invisible** until the next `await` boundary turns it into an unhandled rejection.

`Result<T, E>` makes all three failures impossible to overlook.

### When to throw anyway

Throws still belong, but only for *invariant violations* and *programmer errors*:
- Unreachable code (the `assertNever` helper).
- Required preconditions failing (an obviously-invalid value that should have been validated upstream).
- Resource-exhaustion conditions the spec says are unrecoverable.

These should be process-fatal — see the next note.

### Result libraries

`neverthrow`, `ts-results`, `fp-ts`/`effect`. The vault doesn't prescribe a library; the hand-rolled `Ok | Err` discriminated union is sufficient for projects that want minimal dependencies. What matters is the *discipline*.

*Source: [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]]*

---

## Process-Fatal vs Domain-Recoverable

*A process whose invariants are broken is producing wrong output. Crashing it is the safe default; trying to keep it running is the dangerous one.*

> When an invariant is violated or an asynchronous failure goes unhandled, the process must crash. When an *expected* failure occurs (one the spec lists), it is handled in-band by typed error handling. Mixing the two — catching invariant violations and "logging and continuing" — produces wrong output indefinitely.

### Two categories

**Domain-recoverable failures** — outcomes the spec explicitly anticipates: validation rejected the input, the upstream service returned 503, the database deadlocked. These are normal operating conditions. They have a typed shape, they are handled by the caller, they leave the system in a consistent state. `Result<T, E>` handles them.

**Process-fatal failures** — situations that should not occur: a `never` branch was reached, an awaited promise rejected with no handler, an internal data structure has an impossible state, an out-of-memory condition. Continuing to run when one of these occurs means the program is producing output without satisfying its preconditions, which is the technical definition of "wrong."

The right response to category 2 is to crash. A supervised process (container, systemd unit, Kubernetes pod, PM2) restarts from a known-good state. The wrong response is to catch, log, and continue, because the caught state is by definition one the rest of the program is not designed to operate from.

### Node's structural support for fail-fast

- **Unhandled promise rejections.** Configure `--unhandled-rejections=throw` explicitly. Don't catch `unhandledRejection` and "log and continue."
- **Uncaught exceptions.** `process.on('uncaughtException', handler)` should log + flush telemetry + `process.exit(1)`. Not "swallow and continue."
- **Worker thread isolation.** For partial-failure scenarios (plugin host, sandbox), let the worker crash; the main process spawns a replacement.

### What "fail-safe" looks like at the product level

"Process-fatal" doesn't mean user-visible failure. It means "this process exits and a supervisor restarts a fresh one." The user-facing impact depends on the surrounding architecture — load balancer routing, replication, checkpointing, client reconnect logic. The product-level property is **fail-safe**: a known-bad outcome rather than an unknown-wrong one.

### Process-fatal patterns vs not

Process-fatal:

```ts
// Unreachable code
default: {
  const _exhaustive: never = shape;
  throw new Error("Unreachable");
}

// Top-level entry point
main().catch(err => {
  console.error({ event: "fatal", error: err });
  process.exit(1);
});
```

Not process-fatal — return Result instead:

```ts
function parseUserInput(raw: unknown): Result<ParsedInput, ParseError> { /* ... */ }
async function callUpstream(): Promise<Result<Response, UpstreamError>> { /* ... */ }
```

### Composing with observability

Fail-fast is only safe when the process's death is *visible*. The crash must emit a structured fatal log line, increment a fatal-crash metric, and trigger an alert if the rate exceeds a known floor. Fatal-crash rate is a budget burn rate.

*Source: [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]]*

---

## Runtime Validation at Boundaries

*Types are compile-time. The runtime sees only bytes, JSON, env strings. Convert bytes to typed values explicitly, at the boundary, with a schema.*

> Every external input enters the program as `unknown` and is parsed against an executable schema into a typed value before the typed core sees it.

The compiler can prove that values of type `User` always have the fields `User` declares. It cannot prove that what `JSON.parse(req.body)` returns is a `User`. The two facts are independent.

### The pattern

```ts
type User = { id: UserId; email: Email; createdAt: Date };
const userSchema: Schema<User> = /* ... */;

async function getUser(req: Request): Promise<Result<User, ParseError>> {
  const raw: unknown = await req.json();
  const parsed = userSchema.parse(raw);
  if (!parsed.ok) return err({ kind: "invalid_payload", details: parsed.error });
  return ok(parsed.value);
}
```

### Why "unknown first"

A common anti-pattern: `const body = await req.json() as User`. The cast is a lie. The runtime may have given a string, a partial object, a string-encoded number, or a deliberately hostile payload. The type system prevented none of this. Force the schema to do the proof.

### Library landscape

- **Ajv** — JSON Schema and JTD, cross-language. Pick for external contracts.
- **Zod** — TS-first, fluent. Pick for internal services.
- **Valibot** — tree-shakeable. Pick when bundle size matters.
- **ArkType** — TS-syntax inference. Pick when type fidelity dominates.

For external contracts, default to Ajv + JSON Schema. For internal services, default to Zod. Pick one project-wide; don't mix three.

### Boundaries to validate

- HTTP request bodies and query parameters
- Message-bus payloads
- Environment variables
- CLI arguments
- Feature-flag payloads
- Persisted state on read-back
- File imports
- WebSocket frames, gRPC payloads

Any data crossing the program's outer boundary belongs in this list.

### Cite-able primary source

The OWASP Input Validation Cheat Sheet's recommendation — validate syntactic and semantic correctness of all external input as early as possible using allowlists — is exactly what schema-at-the-boundary implements.

*Source: [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]]*

---

## Related bundles

- [[Bundles/TypeScript/Module-Design]] — module boundaries are where typed errors live
- [[Bundles/TypeScript/Code-Review]] — what to check when error-handling lands in a diff
- [[Bundles/TypeScript/Test-Design]] — testing the error paths
