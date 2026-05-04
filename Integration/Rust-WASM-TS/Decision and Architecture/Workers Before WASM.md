---
title: Workers Before WASM
tags: [integration, typescript, worker, wasm, escalation]
summary: Before introducing Rust/WASM, exhaust the TypeScript-plus-workers-plus-transferable-buffers escalation. Workers may suffice for medium-hot work; WASM wins only when the kernel is heavy enough to amortize boundary cost.
keywords: [worker_threads, web workers, transferable buffers, typed array, escalation, performance ladder, parallelism]
---

*Workers and `TypedArray` solve a surprising amount of performance work without a language switch. Try them first; reach for Rust/WASM only if measurement says you must.*

# Workers Before WASM

> **Rule:** the performance escalation order for CPU-bound TypeScript code is: (1) algorithmic improvement → (2) `TypedArray` and allocation discipline → (3) workers with transferable buffers → (4) Rust/WASM. Each step has materially lower complexity cost than the next; reaching for step 4 without exhausting steps 1–3 wastes engineering budget.

The performance instinct in many teams is to jump from "this is slow in JS" directly to "let's add Rust." Sometimes that's the right move. More often, the slow code is slow for reasons earlier escalations would solve — pathological allocation, an O(n²) loop, or main-thread contention. WASM doesn't fix those; it just hides them.

## The escalation ladder

### Step 1 — Algorithmic improvement

Before anything else: is the algorithm itself the bottleneck? An O(n²) inner loop replaced with O(n log n) is a bigger win than any language switch. Profile before you optimize, *and* profile before you decide which optimization. The hottest function in the profile may have a 100× algorithmic improvement available that no amount of WASM will deliver.

This step is free (it's just code) and most often the largest win. Skipping past it is the most common mistake.

### Step 2 — `TypedArray` and allocation discipline

If the algorithm is sound, the next axis is *how it operates on data*:

- Use `TypedArray` (Uint8Array, Float32Array, etc.) and `ArrayBuffer` for binary or numeric data instead of arrays of objects. The memory layout is contiguous, allocation is one shot, GC pressure drops.
- Avoid per-iteration object/closure allocation in hot loops (see [[Languages/TypeScript/Practices/Performance/Allocation Hygiene]]).
- Keep object shapes stable for V8's hidden-class optimizer (see [[Languages/TypeScript/Practices/Performance/Object Shape Stability]]).
- Use string-builder patterns (`Array.join`) rather than `+=` concatenation in loops.

This step is also relatively free — it's discipline applied to the existing code. The wins are sometimes 2–5×, sometimes a small constant factor; either way, it's a prerequisite for measuring whether the next steps are needed.

### Step 3 — Workers with transferable buffers

If the algorithm is sound and the data discipline is good but the work is still expensive enough to block the event loop, the next step is to move it off the main thread:

- **Web Workers** (browser) and **`worker_threads`** (Node) provide a separate JS execution context. The worker has its own event loop; the main thread stays responsive.
- **Transferable buffers** (`postMessage(buf, [buf])`) move ownership of an `ArrayBuffer` from one thread to another *without copying*. For binary payloads, this is the right pattern.
- **`SharedArrayBuffer`** allows true shared memory between threads when the worker model isn't enough. Requires cross-origin isolation in browsers; Node has fewer constraints. Use sparingly — it changes the memory model.

Workers have real cost: setup time per worker, message-passing overhead per call, complexity of state management across threads. They earn their keep when the work-per-call is large enough that the overhead amortizes, the same calculus as WASM.

### Step 4 — Rust/WASM

If steps 1–3 are exhausted *and* profiling still shows a CPU-bound kernel that matters, *and* the kernel is coarse-grained enough to amortize the JS↔WASM boundary cost, *and* a Rust ecosystem provides leverage, *then* Rust/WASM is justified. See [[Integration/Rust-WASM-TS/Decision and Architecture/When Rust-WASM Is Justified]] for the full decision rule.

WASM kernels typically run in workers themselves (combining steps 3 and 4). The shell-kernel architecture (see [[Integration/Rust-WASM-TS/Decision and Architecture/TypeScript Shell Rust Kernel Architecture]]) treats the WASM module as a kernel that may live anywhere — main thread or worker.

## Why workers beat WASM for medium-hot work

The cost of a WASM kernel is:

- **Boundary cost per call.** Even with `wasm-bindgen`'s optimizations, calls cross a JS↔WASM context switch. Object-heavy boundaries are slow (see [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]]).
- **Build complexity.** Two toolchains, two test pipelines, generated TS surfaces to maintain.
- **Cognitive load.** Engineers need to be Rust-fluent to maintain the kernel, or the kernel becomes a black box owned by one person.

The cost of a worker is:

- **Setup cost** (one-time per worker, roughly milliseconds).
- **Per-message cost** (transferable buffers are zero-copy but messages still cross thread boundaries).
- **Complexity of state management** across threads.

For work that's "expensive but not extreme" — say, parsing a megabyte JSON, encoding a few-second audio clip with a JS-implemented codec, running a moderate compute loop — workers are often enough. The work-per-call is large enough that worker overhead amortizes; you don't need WASM's compute density.

WASM wins decisively when the work-per-call is *extreme* (gigabytes processed per call, or compute-intensive enough that JS's general-purpose JIT can't compete) or when an existing Rust crate provides functionality you'd otherwise re-implement.

## When workers don't help

Workers do not help when:

- **The bottleneck is I/O**, not CPU. A worker waiting on the same network call has not made anything faster. (Worse: it added cost.)
- **The work is many tiny calls.** Worker message-passing per call defeats the model; the main thread spends all its time talking to the worker.
- **State is heavily shared and mutable.** SharedArrayBuffer can fix this but reintroduces the data-race risks the JS single-threaded model was designed to avoid.
- **Memory pressure is the issue**, not CPU. A worker has its own heap; total memory may go up, not down.

In each case, the right answer is upstream of workers — algorithmic, allocation, or architectural.

## When the escalation order doesn't apply

A few cases skip directly to step 4:

- **An existing Rust crate** is the right tool and re-implementing in TS would cost weeks (ImageMagick equivalents, audio codecs, scientific libraries).
- **Determinism or numerical precision** requires a model JS doesn't natively provide (bit-exact float, constant-time crypto).
- **Memory layout** requirements (`#[repr(C)]`, packed structs) that JS engines cannot guarantee.

In these cases, the decision is settled by the dependency or correctness requirement, not by performance comparison.

## Composing with other practices

- [[Integration/Rust-WASM-TS/Decision and Architecture/When Rust-WASM Is Justified]] — the formal decision rule for step 4.
- [[Integration/Rust-WASM-TS/Decision and Architecture/TypeScript Shell Rust Kernel Architecture]] — the architecture once you do reach step 4.
- [[Languages/TypeScript/Practices/Performance/Allocation Hygiene]] — the discipline for step 2.
- [[Languages/TypeScript/Practices/Performance/Object Shape Stability]] — the discipline for step 2.
- [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]] — the metric that tells you when steps 1–3 are no longer enough.
- [[Engineering Philosophy/Principles/Reference Implementation as Oracle]] — keep the simpler implementation alongside any escalated one as the correctness oracle.

## Related

- [[Integration/Rust-WASM-TS/Decision and Architecture/When Rust-WASM Is Justified]]
- [[Integration/Rust-WASM-TS/Decision and Architecture/TypeScript Shell Rust Kernel Architecture]]
- [[Languages/TypeScript/Practices/Performance/Allocation Hygiene]]
- [[Languages/TypeScript/Practices/Performance/Object Shape Stability]]
- [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]]
