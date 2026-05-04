---
title: Object Shape Stability
tags: [performance, typescript, v8, hidden-class, advanced]
summary: V8's hidden-class optimization rewards objects whose property set, order, and types are stable across instances. Polymorphic shape on hot paths costs measurable performance — most teams will never need to optimize for this; the few that do should know the model.
keywords: [v8, hidden classes, inline caches, object shape, monomorphic, polymorphic, megamorphic, hot path]
---

*V8 makes objects with the same shape fast. Objects whose shape varies — properties added in different orders, optional fields randomly present — pay an indirection cost. Most code doesn't notice. Hot paths can.*

# Object Shape Stability

> **Rule:** for hot-path code only — code that's been profiled and shown to be performance-sensitive — design objects so their shape is consistent across instances. For everything else, optimize for clarity and trust V8's optimizer.

This is an advanced / optional practice. Knowing the V8 model is occasionally useful; coding to it pre-emptively is almost always a mistake. The note exists for the cases where measurement has shown that property-access cost dominates.

## The V8 model

V8 represents objects through *hidden classes* (also called Maps or Shapes). Each object instance has a hidden class, and the hidden class records the layout — which properties exist at which offsets. Objects created the same way share a hidden class; V8 can then generate optimized code for property access on that class because the offsets are known.

Hot property access goes through *inline caches* — V8 records the hidden class of objects seen at a given access site and emits specialized code for that class. Three regimes:

- **Monomorphic** — the access site has only ever seen objects of one hidden class. Fastest; direct offset access.
- **Polymorphic** — the access site has seen 2–4 hidden classes. Branching code; slower but still fast.
- **Megamorphic** — the access site has seen 5+ hidden classes. Full-lookup code; significantly slower.

The discipline is: keep hot-path access sites monomorphic, or at worst lightly polymorphic.

## What disturbs hidden classes

Objects diverge into different hidden classes when:

### Property addition order differs

```ts
// Object A's hidden class
const a = {};
a.x = 1;
a.y = 2;

// Object B's hidden class — different from A's, even though both have {x, y}
const b = {};
b.y = 2;
b.x = 1;
```

Two objects with the same final properties can have different hidden classes if their construction order differs.

### Properties added after construction

```ts
function makeShape() {
  const o = { x: 0, y: 0 };
  return o;
}

const a = makeShape();         // hidden class: {x, y}
const b = makeShape();
b.z = 1;                       // hidden class: {x, y, z} — different from a's
```

Adding a property after construction transitions the object to a new hidden class. Functions that mutate object shape post-construction force every consumer to handle the resulting polymorphism.

### Type changes on a property

```ts
const a = { count: 0 };       // count: SMI (small integer)
const b = { count: 0 };
b.count = "zero";             // count: string — different hidden class
```

Changing the *type* of a property's value (number → string, integer → float in some cases) can force a hidden-class transition.

### Conditional property assignment

```ts
function makeUser(input: UserInput): User {
  const u: any = { id: input.id };
  if (input.email) u.email = input.email;
  if (input.phone) u.phone = input.phone;
  return u;
}
```

Four hidden classes for four input shapes (just `id`, `id+email`, `id+phone`, `id+email+phone`). At a downstream access site that handles all four, the access is polymorphic.

## What stable shapes look like

Three patterns produce monomorphic hot paths:

### Initialize all fields at construction, in the same order

```ts
function makeUser(input: UserInput): User {
  return {
    id: input.id,
    email: input.email ?? null,
    phone: input.phone ?? null,
  };
}
```

Every `User` has the same hidden class regardless of input. Optional properties are explicitly `null` rather than absent. Downstream code that reads `.email` is monomorphic.

### Use classes when the shape is well-defined

```ts
class User {
  constructor(
    public id: UserId,
    public email: Email | null,
    public phone: Phone | null,
  ) {}
}
```

Class instances created by `new` get a stable hidden class shared across all instances (within a class).

### Use TypeScript discriminated unions where shapes genuinely vary

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number };
```

Each variant has its own stable shape. The dispatch on `kind` (see [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]) routes to per-variant code paths, each monomorphic. This is faster *and* type-safer than a single shape with optional fields per variant.

## Profile before optimizing

The cost of polymorphic access is real but usually modest. Before reshaping code for monomorphism:

1. **Profile.** Use V8's tick profiler (`node --prof`) or Chrome DevTools' Performance panel. Find where the time actually goes.
2. **Look for IC misses.** V8 logs (`--trace-ic`) show which access sites are polymorphic / megamorphic. The output is verbose but informative.
3. **Measure the change.** A reshape that "should" speed things up may not; measure with a stable benchmark before and after.

Almost every team that reaches for shape-stability optimization is solving the wrong problem. Profile carefully; the bottleneck is usually I/O, GC pressure, or algorithmic, not access-site polymorphism.

## When this matters

Honest cases:

- **Hot inner loops in compute-heavy code.** Image processing, numerical kernels, parsers, codecs — code where the same shape is accessed millions of times per request.
- **High-throughput backend services where event-loop time per request matters.** A 5% per-request improvement compounds.
- **Long-running processes where peak throughput is bounded by per-call CPU time.**

Cases that don't matter:

- **Most application code.** Polymorphism is the GC's problem to optimize, not yours.
- **Code where I/O dominates.** Database calls and HTTP requests are 1000× slower than any property-access cost.
- **Cold paths.** Shape stability matters where access happens often. A function called once per request is "often" only at very high QPS.

## V8 documentation

The canonical reference is V8's "Fast properties in V8" blog post (https://v8.dev/blog/fast-properties) and the broader [V8 documentation](https://v8.dev/docs). The exact representation has changed over V8 versions and will continue to change; the *principle* (stable shapes get fast code) is durable.

## Composing with other practices

- [[Languages/TypeScript/Practices/Performance/Allocation Hygiene]] — both notes operate at the V8-runtime level; allocation pressure and shape polymorphism often correlate.
- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]] — discriminated unions naturally produce stable shapes per variant; coding to the type system aligns with shape stability.
- [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]] — performance regressions often surface as event-loop-delay growth.
- [[Engineering Philosophy/Principles/Reference Implementation as Oracle]] — optimized code with shape-stability tweaks should keep a slower reference for differential testing.

## Related

- [[Languages/TypeScript/Practices/Performance/Allocation Hygiene]]
- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]
- [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]]
- [[Engineering Philosophy/Principles/Reference Implementation as Oracle]]
