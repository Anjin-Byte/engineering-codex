---
title: DOM Ownership Boundary
tags: [integration, rust, wasm, typescript, dom, web-sys]
summary: TypeScript owns DOM mutation, event wiring, and UI state. Rust may compute UI data but does not own the DOM by default; `web-sys` exists but rarely pays back.
keywords: [dom ownership, web-sys, jscast, rust frontend, ui state machine, dom mutation, kernel ui split]
---

*Rust computes the data; TypeScript paints the screen. The DOM lives in the host's runtime — let it live there.*

# DOM Ownership Boundary

> **Rule:** in a Rust↔WASM↔TS system, TypeScript owns DOM mutation, event handling, and UI state machines. Rust may compute *data that drives the UI*. Rust does not own the DOM by default. Exceptions exist (Rust-first frontend frameworks, Yew, Leptos), but for projects whose primary paradigm is "TS application with a Rust kernel," the DOM stays on the TS side.

The DOM is a JavaScript-native API surface. WASM can reach the DOM through `web-sys` and `JsCast`, but doing so trades a clean separation for ongoing complexity that doesn't typically pay back. The default policy line — *Rust may compute UI data; TypeScript owns DOM mutation and event wiring unless a conscious exception is approved* — keeps the architecture matched to what each runtime is good at.

## Why TypeScript owns the DOM

Three structural reasons:

1. **The DOM was designed for JavaScript.** Every API, every event, every property accessor is shaped around JS idioms. Crossing the WASM boundary to make a DOM call adds friction with no compensating benefit — the DOM call is the same speed either way; you've just added a JS↔WASM context switch.

2. **UI state is chatty.** A typical UI mutation involves dozens of small operations: read a value, update a class, attach a listener, emit a custom event, schedule a relayout. Each is fine in JS; each is a boundary crossing in WASM-driven UI. See [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]].

3. **The frontend ecosystem is JavaScript-shaped.** Frameworks (React, Vue, Solid, Svelte), routing libraries, state management, design systems, accessibility tooling — they all expect to be the DOM's primary consumer. Putting Rust in that role means rebuilding adapters for every piece of the ecosystem.

## What Rust *can* do for the UI

The right split is "Rust computes; TS renders." Concretely:

- **Layout calculation.** A Rust kernel computes layout metrics; TS applies them as CSS or absolute positioning.
- **Image / canvas pixel work.** Rust processes pixels; TS calls `putImageData` on a canvas.
- **Audio buffer generation.** Rust computes audio samples; TS feeds them to `AudioContext`.
- **Data transformation for visualization.** Rust filters, aggregates, or transforms data; TS feeds the result to a charting library.
- **Real-time simulation update.** Rust steps a simulation; TS reads the new state and updates the rendering.

In each case Rust produces a `Uint8Array` or `Float32Array` (or similar typed buffer), and the TS side does the actual DOM work. The boundary stays narrow; the DOM stays in JS.

## When Rust *should not* touch the DOM

Anti-patterns to recognize:

- **Rust rendering a list of items.** The list is a DOM tree; rendering it from Rust means crossing the boundary per item or building the entire tree in WASM and pushing it across in one giant call. Either way, the JS side has to consume it. Render in TS.
- **Rust handling click events.** An event listener registered from Rust requires the boundary on every click. The TS side already has a perfectly good event loop. Listen in TS; call into Rust if a click triggers WASM-side computation.
- **Rust managing form state.** Form state is exactly the kind of UI state that's small, frequent, and ecosystem-integrated. Manage in TS.
- **Rust controlling routing.** Routing in single-page apps interacts with browser history, scroll management, focus, accessibility announcements. All JS-shaped. Route in TS.
- **Rust managing animations.** `requestAnimationFrame` is JS-native; running an animation loop from Rust adds boundary crossings per frame for no benefit.

## When Rust DOM access is actually warranted

A few cases where reaching for `web-sys` from Rust is the right call:

- **Tight render loop with isolated DOM target.** A WebGL / canvas application where the Rust kernel owns a single `<canvas>` and the rest of the page is in TS. Even here, the boundary is "TS gives Rust a handle to one canvas," not "Rust controls the page."
- **Truly Rust-first frontends.** Yew, Leptos, Dioxus, and similar are genuine alternatives to React/Vue/etc. and own DOM management end-to-end. If you've adopted one of these, the rules of this note are different — DOM management is the framework's responsibility, just in Rust. But that's a different architecture from the TS-shell-with-Rust-kernel pattern this scope is built around.
- **Performance-critical specific operations.** Reading large arrays from canvas pixel data, manipulating typed arrays in-place. Sometimes one specific DOM operation is hot enough to justify the boundary crossing. Profile first.

In each case, the warranted use is *narrow and specific*, not "we may as well control the DOM from Rust."

## `web-sys` and `JsCast` — the tools when you need them

When Rust does need DOM access:

- **[`web-sys`](https://rustwasm.github.io/wasm-bindgen/web-sys/index.html)** provides Rust bindings to Web APIs. It's massive — thousands of types — and feature-gated. Enable only the parts you actually use:

```toml
[dependencies.web-sys]
version = "0.3"
features = [
  "Window",
  "Document",
  "Element",
  "HtmlCanvasElement",
  "CanvasRenderingContext2d",
  "ImageData",
]
```

Each feature increases binary size; only enable what's needed.

- **`JsCast`** provides type-checked downcasting from generic `Element` references to specific types like `HtmlCanvasElement`:

```rust
use wasm_bindgen::JsCast;
use web_sys::{HtmlCanvasElement, window};

let canvas = window()
    .and_then(|w| w.document())
    .and_then(|d| d.get_element_by_id("my-canvas"))
    .and_then(|el| el.dyn_into::<HtmlCanvasElement>().ok())
    .ok_or_else(|| JsError::new("canvas not found or wrong type"))?;
```

Two important methods:

- **`.dyn_into::<T>()` / `.dyn_ref::<T>()`** — checked cast; returns `Result` / `Option`. Use these for any DOM element whose type isn't statically known.
- **`.unchecked_into::<T>()` / `.unchecked_ref::<T>()`** — unchecked cast; trusts the caller. Use only when you've already verified the type some other way.

The standard rule: **prefer `dyn_into` / `dyn_ref` over the unchecked variants** when the value's type isn't compile-time guaranteed. The unchecked path is faster but produces undefined behavior on a type mismatch.

## Minimal `web-sys` feature set discipline

The `web-sys` feature list grows easily. Discipline:

- **Start with no features**, add only what compiles when tried.
- **Periodically audit.** Are the features you enabled still used?
- **Document each one.** A `web-sys` feature is a dependency on a specific browser API. The one-line comment per feature pays back when the codebase grows.

```toml
[dependencies.web-sys]
version = "0.3"
features = [
  "Window",                    # entry point for browser APIs
  "Document",                  # for element lookup
  "HtmlCanvasElement",         # the kernel's canvas target
  "CanvasRenderingContext2d",  # for direct pixel write
  "ImageData",                 # for pixel buffer transfer
]
```

A feature list with comments is a feature list someone can shrink later.

## Cross-cutting: the kernel never owns user state

A useful sub-rule: even when Rust does compute UI data, *application state* (selected item, current route, user preferences, modal open/closed) lives in TS. The Rust kernel is stateless across calls (or has small computational state — e.g. an in-progress simulation) but does not hold "what the user clicked" or "what's currently visible."

The state-ownership boundary keeps reasoning simple: TS-side debugging shows the state; the kernel is referentially transparent given its inputs. If a bug surfaces in the UI, the state is in TS DevTools, not behind a WASM-instance pointer.

## Composing with other practices

- [[Integration/Rust-WASM-TS/Decision and Architecture/TypeScript Shell Rust Kernel Architecture]] — the architecture this rule expresses.
- [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]] — DOM operations are *exactly* the kind of chatty boundary that's expensive.
- [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]] — the Rust API exposed to TS should not include DOM-typed values; it should expose buffers and configs.
- [[Languages/TypeScript/Practices/Async and Concurrency/Async Context Propagation]] — request-scoped state lives in TS; the kernel doesn't see it.

## Related

- [[Integration/Rust-WASM-TS/Decision and Architecture/TypeScript Shell Rust Kernel Architecture]]
- [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]]
- [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]]
- [[Integration/Rust-WASM-TS/Tooling and Build/wasm-bindgen and wasm-pack]]
