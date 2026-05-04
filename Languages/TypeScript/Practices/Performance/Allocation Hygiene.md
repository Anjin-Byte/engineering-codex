---
title: Allocation Hygiene
tags: [performance, typescript, node, gc, advanced]
summary: JavaScript is garbage-collected; the only performance lever is allocation behavior. For hot paths, minimize per-iteration allocation, avoid retention in closures or caches, and measure the heap before changing flags.
keywords: [garbage collection, gc, allocation, heap, memory leak, retention, closures, caches, hot path]
---

*You don't manage memory in JavaScript; you manage allocation. The hot loop that allocates a new closure every iteration is fine in development and a heap-pressure problem in production.*

# Allocation Hygiene

> **Rule:** for code on a hot path — request handlers under load, message-bus consumers, tight processing loops — minimize per-iteration allocation, avoid accidental retention in closures and caches, and prefer stable object shapes. For everything else, optimize for clarity, not for the GC.

This is an advanced / optional practice. Most TypeScript code does not need to think about allocation; the GC is good and the cost of premature optimization is real. The note exists for the cases where measurement has shown that allocation pressure is an actual constraint — typically high-throughput backend services or long-running processes that exhibit slow memory growth.

## What "allocation hygiene" means

JavaScript exposes no manual memory control. Every value with non-primitive identity (arrays, objects, functions, closures) is a heap allocation. The GC eventually reclaims unreferenced heap, but the *rate* of GC and the *pause times* are functions of allocation pressure and live-set size. Hot code that allocates aggressively pays in:

- More frequent young-generation GC (cheap individually, expensive in aggregate).
- Larger old-generation collections (expensive individually) when live objects survive to old gen.
- Higher peak heap usage, which can push the process into OOM territory under load.

Hygiene is the discipline of *avoiding allocation you don't actually need* — not an attempt to outwit the GC.

## Common allocation traps

### Per-iteration closures

```ts
// ✗ Allocates a new closure every iteration
items.forEach(item => {
  process(item, ctx);
});

// ✓ One closure, captured once
const handle = (item: Item) => process(item, ctx);
items.forEach(handle);

// ✓✓ Or a plain for-loop with no closure at all
for (let i = 0; i < items.length; i++) {
  process(items[i], ctx);
}
```

For a 1000-item list, the difference is 1000 closures vs 0–1. In a hot path that runs millions of times, the difference is measurable.

### Per-iteration array / object literals

```ts
// ✗ Allocates a new tuple every iteration
for (const item of items) {
  emit([item.id, item.score]);
}

// ✓ Reuse a single buffer object if the consumer doesn't retain it
const buf = { id: "", score: 0 };
for (const item of items) {
  buf.id = item.id;
  buf.score = item.score;
  emit(buf);  // only safe if emit doesn't retain a reference
}
```

The buffer trick is risky — only safe when the consumer reads-and-releases. When in doubt, allocate.

### Concatenation in a loop

```ts
// ✗ Each += allocates a new string
let result = "";
for (const item of items) {
  result += item.toString() + ",";
}

// ✓ Build an array, join once
const parts: string[] = [];
for (const item of items) {
  parts.push(item.toString());
}
const result = parts.join(",");
```

Modern V8 has optimizations for some string-concatenation patterns, but the explicit-array approach is reliably fast and reads as clearly.

### Hidden allocations

Some operations allocate where it isn't obvious:

- `arr.slice()` — copies the array.
- `arr.concat(other)` — copies both arrays into a new one.
- `Object.assign({}, base, override)` — new object.
- `{ ...base, ...override }` — new object.
- `arr.map(fn)` — new array of return values.
- `JSON.parse(str)` — new object graph.
- `String.split` — new array.

These are correct in most code. In a hot loop, they're allocations that compound.

## Retention bugs

A different failure mode: code that holds references it shouldn't, preventing the GC from reclaiming. Common shapes:

### Caches without bounds

```ts
// ✗ Unbounded cache; grows forever
const cache = new Map<string, Result>();

function compute(key: string): Result {
  if (!cache.has(key)) {
    cache.set(key, expensive(key));
  }
  return cache.get(key)!;
}
```

Use a bounded cache (LRU or TTL) for memoization. `lru-cache` is the standard option.

### Closure capture of large scopes

```ts
function setup(largeData: Data[]): () => void {
  return () => {
    console.log("ping");      // largeData stays in scope, alive forever
  };
}
```

If `largeData` isn't actually used, refactor so it's not captured. If only a small piece is used, capture just the piece.

### Event-emitter listeners not removed

```ts
// ✗ Listener stays attached; the closure (and its captured scope) lives forever
emitter.on("event", () => doWork(largeData));
```

Long-lived emitters with short-lived listeners need explicit `off` / `removeListener`. Or use `once` if a single fire is enough.

### Async-iteration buffers

Streams, async iterators, and back-pressure-naive code can accumulate buffers when consumer is slower than producer. The fix is consumer-side back-pressure, not GC tuning.

## Diagnosing in production

When you suspect allocation pressure but haven't yet confirmed:

1. **Run with `--max-old-space-size=N`** to set a heap limit, then watch whether the process approaches it. A process that grows toward the limit and stays there is leaking; a process that grows, GCs, and stays steady is just allocating actively.
2. **Use `--heapsnapshot-near-heap-limit=N`** to dump a heap snapshot just before OOM. Open in Chrome DevTools; the largest retainers are the suspects.
3. **Use `--trace-gc`** for a coarse view of how often and how long GC is running. Frequent old-gen collections are a sign of pressure.
4. **Profile with `--inspect`** and Chrome DevTools' Memory panel. Take snapshots, compare, find the growing references.

Don't change `--max-old-space-size` or `--initial-old-space-size` based on intuition. Measure first.

## When to *not* think about allocation

Most TypeScript code. The cost of allocation-aware coding is readability; the benefit is performance only on hot paths. A request handler that runs once per request and allocates twenty objects in the process is almost certainly fine. Optimize when measurement shows you need to.

For request-scoped state propagation, use `AsyncLocalStorage` (see [[Languages/TypeScript/Practices/Async and Concurrency/Async Context Propagation]]) — it allocates, but it's the right shape and doesn't need hand-optimization.

## Composing with other practices

- [[Languages/TypeScript/Practices/Performance/Object Shape Stability]] — V8 hidden-class optimization rewards stable object shapes; allocation hygiene composes with shape discipline.
- [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]] — long GC pauses show up as event-loop-delay spikes; the correlation is observable.
- [[Engineering Philosophy/Principles/Reference Implementation as Oracle]] — when an optimized hot path has measurable performance gain, keep the simpler reference for differential testing.
- [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]] — performance optimization can introduce subtle bugs; types prove no behavioral guarantee, only structural.

## Related

- [[Languages/TypeScript/Practices/Performance/Object Shape Stability]]
- [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]]
- [[Engineering Philosophy/Principles/Reference Implementation as Oracle]]
- [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]]
