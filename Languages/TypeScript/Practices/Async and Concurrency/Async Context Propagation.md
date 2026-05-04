---
title: Async Context Propagation
tags: [async, typescript, node, async-local-storage, context]
summary: Use `AsyncLocalStorage` for request-scoped state — correlation IDs, authorization context, tenancy, idempotency keys — so context survives `await` boundaries without being threaded through every signature.
keywords: [async_hooks, async local storage, context propagation, correlation id, request context, tenancy, opentelemetry]
---

*Request-scoped state has to follow the request through every `await`. Threading it as an argument is mechanical; `AsyncLocalStorage` makes it implicit and reliable.*

# Async Context Propagation

> **Rule:** request-scoped state — correlation IDs, authorization context, tenant identity, idempotency keys, observability spans — propagates through async control flow via [`AsyncLocalStorage`](https://nodejs.org/api/async_context.html). Any code path that drops this context is a structural bug, not just an oversight, because it produces logs without correlation, decisions without authorization, and traces with broken parents.

`AsyncLocalStorage` is in `node:async_hooks`, stability 2 — Stable since v16.4.0. The Node docs describe it as creating "stores that stay coherent through asynchronous operations." It is the closest thing Node has to a thread-local: a value bound to the *async causal chain* rather than the call stack, and visible to any code descended from the binding regardless of how many `await`s, `setTimeout` calls, or microtask hops happened in between.

## The pattern

```ts
import { AsyncLocalStorage } from "node:async_hooks";

interface RequestContext {
  correlationId: string;
  tenantId: TenantId;
  user: AuthenticatedUser;
}

export const requestContext = new AsyncLocalStorage<RequestContext>();

// At the request boundary
async function handleRequest(req: Request, res: Response) {
  const ctx: RequestContext = {
    correlationId: req.header("x-correlation-id") ?? uuid(),
    tenantId: parseTenantId(req),
    user: await authenticate(req),
  };

  await requestContext.run(ctx, async () => {
    // Inside this async causal chain, requestContext.getStore() returns ctx
    // — across awaits, timers, microtasks, child function calls, anywhere.
    await routeRequest(req, res);
  });
}

// Anywhere downstream, even five awaits deep
function audit(action: string) {
  const ctx = requestContext.getStore();
  if (ctx) logger.info({ action, correlationId: ctx.correlationId, user: ctx.user.id });
}
```

Inside the `run` block, `requestContext.getStore()` returns the bound `ctx` from any async descendant. Outside, it returns `undefined`. This makes the context *available* without making it *passable* — there is no parameter to forget to forward, no class field to thread through.

## Why this beats threading arguments

The alternative — passing a `ctx` parameter through every function call — has three structural problems:

1. **Every function gains a parameter that is rarely read.** Layers in the call graph that don't directly use the context still have to forward it. Adding a new context field requires updating every signature.
2. **One forgotten forward leaks the bug everywhere downstream.** A logger that runs without a correlation ID for the rest of the request produces logs you can't correlate. The bug is structural and invisible until you go looking.
3. **Third-party code can't be threaded.** Express middleware, an ORM, a message-bus client — none of them know about your context type. They run between your code's awaits and lose access.

`AsyncLocalStorage` makes the context survive across all of these. The third-party library is *inside* the async causal chain; the context is visible to anything `getStore()` is called from inside the `run` block.

## What belongs in async context

The criterion is "state whose scope is the operation, not the request handler." Good candidates:

- **Correlation / trace IDs** for log and trace correlation across services.
- **Authentication and authorization context** — current user, roles, scopes.
- **Tenant identity** in a multi-tenant system.
- **Idempotency keys** when the request is doing something at-most-once.
- **OpenTelemetry context** — the active span, the trace context, the baggage.
- **A request-scoped database transaction** when transaction boundaries align with request boundaries.

What does *not* belong:

- **Mutable state shared across async branches.** The store is per-async-chain; concurrent branches get the same reference. If you mutate the store, you race.
- **Persistent state.** Anything that should survive past the request boundary belongs in storage, not in async context.
- **Computed values that are cheap to recompute.** Don't memoize into context; compute on read.

## Pitfalls

### Forgetting to bind

`AsyncLocalStorage.getStore()` returns `undefined` outside any `run` block. Code that reads the store must handle the `undefined` case:

```ts
const ctx = requestContext.getStore();
if (!ctx) {
  // Not running inside a request — typical for boot-time code, scheduled jobs
  // not associated with a request, etc. Decide explicitly what's right.
  return;
}
```

A common bug: a job spawned by a request kicks off a fire-and-forget background task with `void doBackgroundWork()` that runs *outside* the `run` block. The background task has no context. Decide explicitly: re-bind the context for the background task, or accept that it has none.

### Mutation surprises

The store is the same object reference across the entire async causal chain. Two branches that both read and write fields on the store can race:

```ts
// ✗ Two parallel branches mutating the same store
await Promise.all([
  doStep1(),  // sets ctx.lastStep = "step1"
  doStep2(),  // sets ctx.lastStep = "step2"
]);
// ctx.lastStep is one or the other, depending on scheduling
```

The fix is to treat the store as effectively immutable after the bind, or to scope mutations using nested `run` blocks.

### Performance

`AsyncLocalStorage` has measurable overhead, particularly with many concurrent contexts. For a typical API service this is well under the noise floor; for very hot paths (sub-millisecond handlers, high-frequency polling loops) it's worth measuring. The Node team has continued to optimize the underlying hooks, but a project at the very tail of the latency budget should benchmark.

## OpenTelemetry integration

OpenTelemetry's JavaScript SDK uses `AsyncLocalStorage` as its context-propagation backend on Node. If you adopt OpenTelemetry, you get correlation-ID-style propagation as part of the tracing infrastructure — you don't need to roll your own for trace context. You may still want a separate `AsyncLocalStorage` instance for application-level fields (user, tenant, idempotency key) that are not OpenTelemetry-native.

## Composing with other practices

- [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]] — context propagation is only as good as the awaits. A floating promise drops out of the async chain that owns the context. Fix the float first.
- [[Languages/TypeScript/Practices/Async and Concurrency/Deadline Propagation with AbortSignal]] — deadlines and request context are usually scoped together; both belong in the same `run` block.
- [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]] — a saturated loop disturbs context propagation only in the trivial sense (timers fire late); the chain itself remains coherent. The two concerns are independent.
- [[Engineering Philosophy/Principles/Layered Risk Categories]] — context loss is a concurrency-/context-layer failure mode.

## Related

- [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]]
- [[Languages/TypeScript/Practices/Async and Concurrency/Deadline Propagation with AbortSignal]]
- [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]]
- [[Engineering Philosophy/Principles/Layered Risk Categories]]
