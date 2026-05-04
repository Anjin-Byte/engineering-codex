---
title: Deadline Propagation with AbortSignal
tags: [async, typescript, node, abort-signal, deadline, timeout]
summary: Every outbound IO accepts an `AbortSignal` and runs under a deadline; unbounded fan-out is forbidden — bounded concurrency is mandatory.
keywords: [abort signal, abort controller, timeout, deadline, fetch timeout, bounded concurrency, fan out, cancellation]
---

*If a function can hang, it does. The only protection is a deadline that fires regardless. `AbortSignal.timeout()` is the deadline.*

# Deadline Propagation with AbortSignal

> **Rule:** every outbound IO call — `fetch`, database query, message-bus publish, child-process spawn, file read of unknown size — accepts an `AbortSignal` and runs under an explicit deadline. Fan-out concurrency is bounded; unbounded `Promise.all` over an external collection is forbidden.

A function with no timeout is a function that hangs forever when the dependency is slow, broken, or under load. In a synchronous language this manifests as a wedged thread; in Node, it manifests as a request that never completes, a connection that never returns, a worker that never shuts down. The runtime cannot decide when to give up — the application has to.

[`AbortSignal`](https://nodejs.org/api/globals.html#class-abortsignal) is the standard primitive. `AbortSignal.timeout(delay)` (added in Node v17.3.0 / v16.14.0) returns a signal that auto-aborts after a specified number of milliseconds. The full signal API supports cancellation from any source — explicit `AbortController.abort()`, parent context teardown, request cancellation — making it a single cancellation channel for the whole IO graph.

## The pattern

### Single outbound call

```ts
const response = await fetch(url, {
  signal: AbortSignal.timeout(5000),
});
```

If the fetch takes more than 5 seconds, the signal fires, `fetch` rejects with an `AbortError`, and the caller can handle the timeout as an expected failure (see [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]]).

### Composing deadlines

A request handler typically has its own deadline. Children should inherit it:

```ts
async function handleRequest(req: Request) {
  const signal = AbortSignal.any([
    req.signal,                       // upstream client cancellation
    AbortSignal.timeout(2000),        // local deadline
  ]);

  const upstream = await fetch(internalServiceUrl, { signal });
  const dbResult = await db.query(sql, { signal });
  // ...
}
```

`AbortSignal.any([...])` (Node v20+) returns a signal that fires when *any* of its inputs fire. The handler now respects both the client's cancellation and its own deadline budget.

For Node versions before 20 or for finer control, an `AbortController` wired to multiple sources works the same way:

```ts
const controller = new AbortController();
req.signal.addEventListener("abort", () => controller.abort(req.signal.reason));
const timer = setTimeout(() => controller.abort(new Error("Deadline exceeded")), 2000);
try {
  await fetch(url, { signal: controller.signal });
} finally {
  clearTimeout(timer);
}
```

### Bounded fan-out

```ts
// ✗ Unbounded — N parallel requests, where N is data-driven
const results = await Promise.all(
  items.map(item => fetch(`/api/${item.id}`, { signal: AbortSignal.timeout(5000) })),
);
```

If `items.length` is 100,000, you've opened 100,000 simultaneous requests against a downstream service. You've also DoS'd it. And exhausted local file descriptors. And exhausted any rate limiter. The deadline doesn't help — the saturation is upstream.

```ts
// ✓ Bounded — N parallel, with a configured concurrency cap
import { withConcurrency } from "./bounded-concurrency"; // your helper or library

const results = await withConcurrency(items, 8, item =>
  fetch(`/api/${item.id}`, { signal: AbortSignal.timeout(5000) })
);
```

The 8 here is project-specific — it depends on the downstream service's capacity, the local resource budget, and the SLO. The discipline is independent of the number: *fan-out always has a cap*, even when the cap is conservative.

Many libraries provide bounded-concurrency helpers (`p-limit`, `p-map`, `p-queue`); writing one is also straightforward.

## What "every outbound IO" includes

The discipline is universal at the IO boundary:

- **HTTP / RPC clients** — `fetch`, axios, gRPC, undici. All accept `signal`.
- **Database drivers** — most modern Node drivers accept `signal` or have an equivalent timeout option.
- **Message-bus clients** — Kafka, NATS, RabbitMQ, Redis. Some have native timeouts; for those that don't, wrap the operation in an `AbortSignal`-aware promise.
- **Child processes** — `child_process.spawn` and `execFile` accept `signal`.
- **File reads of unknown size** — for streaming reads of files that may be very large, plumb the signal through the reader.
- **DNS lookups, TLS handshakes** — both can hang. Set explicit timeouts at the socket level if your client library doesn't do it.

What does *not* need a deadline: pure-CPU local work (it can't hang on IO), in-process queue dequeues that are explicitly designed to block (use a different cancellation mechanism), and child operations whose deadline is enforced by an outer signal.

## Why deadlines, not retries

A timeout *gives up*. A retry *tries again*. The two are different decisions, and the timeout decision is upstream:

- A 1-second timeout with no retry: the call gives up at 1s, and the caller decides what happens next.
- A 1-second timeout with 3 retries: each attempt has 1s, total budget is up to 3s plus jitter.
- No timeout with 3 retries: each attempt hangs forever, retries never trigger.

The first two are reasonable; the third is what you have if you skip deadlines. Always set the deadline first; layer retries on top if appropriate.

## Honest deadlines

A deadline is only honest if the runtime can honor it. [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]] makes the point that a saturated event loop delays every timer, including `AbortSignal.timeout`. A 1-second deadline on a service whose loop delay is regularly 200ms is in practice a 1.2-second deadline. Either tighten the deadline-setting to account for known loop delay, or fix the loop saturation.

This is a structural reason to monitor event-loop delay: it tells you whether the deadlines you wrote down are still the deadlines the runtime is honoring.

## Composing with other practices

- [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]] — a floating promise has no awaiter, so the timeout's rejection has no handler. The lint catches the structural cause.
- [[Languages/TypeScript/Practices/Async and Concurrency/Async Context Propagation]] — deadline and context usually share a scope; both belong in the same boundary block.
- [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]] — the deadline's honesty depends on loop health.
- [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]] — a timeout is an expected failure, not an invariant violation. The caller's response should be typed.
- [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]] — the SLO defines the deadline budget for the whole request; sub-deadlines are pieces of that budget.

## Related

- [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]]
- [[Languages/TypeScript/Practices/Async and Concurrency/Async Context Propagation]]
- [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]]
- [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]]
- [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]]
