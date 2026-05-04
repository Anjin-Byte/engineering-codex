---
title: Boundary Crossing Cost
tags: [integration, rust, wasm, typescript, boundary, performance]
summary: WASM and JS live in separate memory; object-heavy crossings are slower than byte-oriented. Split a control plane from a data plane; keep APIs coarse.
keywords: [linear memory, js boundary, control plane, data plane, coarse grained api, ffi cost, typed buffer, jsvalue]
---

*JS and WASM live in different memory worlds. Crossings cost real time. Design the boundary so the hot stuff crosses as bytes; the chatty stuff stays on one side.*

# Boundary Crossing Cost

> **Rule:** the JS↔WASM boundary is a designed artifact. Object-heavy crossings (`JsValue` round-trips, struct-shaped arguments) are materially slower than typed-buffer crossings. Hot paths cross as bytes; control-plane configuration crosses as validated structs at low frequency. Coarse-grained APIs that do substantial work per call earn the boundary back.

WebAssembly's linear memory and JavaScript's heap are different memory domains. The wasm-bindgen layer makes them feel like one — a Rust function takes a `&[u8]` and JS passes a `Uint8Array`, no glue visible — but underneath, the data is being transferred (or referenced via handles) across a hard boundary. The cost varies by *what* crosses and *how often*.

## The two boundaries

Two distinct boundaries are involved:

1. **Memory boundary.** WASM linear memory (a resizable `ArrayBuffer` or `SharedArrayBuffer`) is separate from JS objects. Bytes pushed across the memory boundary may be copied (typical) or aliased (in some configurations).

2. **Object boundary.** JS objects passed to WASM are represented in a wasm-bindgen-managed table as `JsValue` handles. The Rust code sees `JsValue`; the actual JS object stays on the JS heap. Each access to its fields is a host call back into JS.

These are different boundaries with different costs. Bytes cross the memory boundary cheaply (memcpy, sometimes zero-copy). Objects don't *cross* the object boundary so much as they're *referenced* across it — and every interaction with the reference is a context switch.

## What's cheap and what's expensive

In rough order, cheapest to most expensive:

| Crossing | Mechanism | Cost |
|---|---|---|
| Numeric scalar (`u32`, `f64`) | Direct WASM ABI | Minimal |
| `&[u8]` / `&mut [u8]` | Memory copy or alias into linear memory | Low (linear in size) |
| `&[T]` of POD structs | Memory copy (with layout conversion if needed) | Low–medium |
| Owned `Vec<u8>` returned to JS | Allocation in linear memory + copy to JS Uint8Array | Medium |
| `String` argument or return | UTF-8 transcoding + memory copy | Medium |
| `JsValue` argument | Handle lookup in wasm-bindgen table | Medium per access |
| Struct exchanged via `serde-wasm-bindgen` | JS object walk + Serde deserialize per field | High |
| Per-field property access on a `JsValue` | One host call per access | Very high |

The pattern: bytes are cheap, scalars are cheap, structures-as-objects are expensive, and object-property access is most expensive of all.

## Control plane vs data plane

The structural fix is to design the boundary as two distinct planes:

### Control plane

Configuration, setup, lifecycle calls. Object-shaped data is fine here because the call frequency is low. Examples:

- Constructor for a kernel handle: `new ImageProcessor(config)`. Called once.
- Reconfiguration: `processor.set_options(opts)`. Called rarely, on user action.
- Query: `processor.get_state() -> State`. Called occasionally for diagnostics.

For control-plane calls, ergonomics matter more than throughput. Use `serde-wasm-bindgen` to exchange Rust structs and JS objects (see [[Integration/Rust-WASM-TS/Boundary Design/Serialization Choice]]). Validate inputs as you go.

### Data plane

The hot path. Each call processes substantial data. Object shapes don't cross; bytes do. Examples:

- Frame processing: `process_frame(input: &[u8], output: &mut [u8]) -> Result<(), JsError>`. Called per frame.
- Codec encode/decode: `encode(input: &[u8]) -> Vec<u8>`. Called per chunk.
- Numeric kernel: `transform(input: &Float32Array, output: &mut Float32Array)`. Called per buffer.

For data-plane calls, the API is buffer-oriented. Sometimes the same call accepts a length parameter to process partial buffers. Sometimes the buffer is reused (the caller passes the same `Uint8Array` repeatedly to avoid reallocation).

## Coarse-grained API discipline

The work-per-call should be substantial. Two examples of the same logical operation, one wrong, one right:

```rust
// ✗ Too fine-grained — boundary cost dominates
#[wasm_bindgen]
pub fn add_pixel(channel_r: u8, channel_g: u8, channel_b: u8) -> Pixel { /* ... */ }
// Caller crosses boundary millions of times.
```

```rust
// ✓ Coarse-grained — one boundary crossing per substantial unit of work
#[wasm_bindgen]
pub fn process_image(
    input: &[u8],
    width: u32,
    height: u32,
    operations: &[u8],         // serialized op list
) -> Result<Vec<u8>, JsError> { /* ... */ }
// Caller crosses boundary once per frame.
```

The second form does an entire image's work per call. The boundary cost is amortized over millions of pixels rather than paid per pixel. Same total work, different boundary throughput.

## Buffer management discipline

A common boundary-cost trap is reallocating buffers on every call. Better patterns:

- **Caller-owned buffers.** The TS side maintains a `Uint8Array` it reuses across calls; the Rust function fills it. No allocation per call.
- **Pooled allocations.** A small pool of reusable buffers on each side, claim/release on each call.
- **Streaming with chunks.** For very large inputs, process in chunks rather than one buffer; each chunk is a coarse-grained call.

```ts
const buffer = new Uint8Array(1024 * 1024);  // allocated once

for (const frame of frames) {
  const written = wasmKernel.process(frame.input, buffer);
  consume(buffer.subarray(0, written));
}
```

The `buffer.subarray(0, written)` is a view, not a copy. The kernel writes directly into the caller's `Uint8Array`, which lives on the JS side but is accessible from WASM via a memory view.

## When the boundary cost is unavoidable

Some workloads inherently have chatty boundaries. Examples:

- A scripting language interpreter where each step is a host call. Each step is too small to amortize boundary cost. WASM may be the wrong runtime; consider an interpreter in JS, or a compiled-to-WASM language.
- Real-time interactive systems where each user input must round-trip through Rust before producing JS output. Look hard at whether the round-trip is structural; sometimes it can be batched.

In these cases the answer is sometimes "don't use WASM here," not "tune the boundary harder." The boundary has a cost floor; below that floor, JS is the right runtime.

## Composing with other practices

- [[Integration/Rust-WASM-TS/Decision and Architecture/When Rust-WASM Is Justified]] — the upstream decision; if the boundary cost dominates, the whole project is wrong.
- [[Integration/Rust-WASM-TS/Decision and Architecture/TypeScript Shell Rust Kernel Architecture]] — the architecture this crossing-cost discipline operationalizes.
- [[Integration/Rust-WASM-TS/Boundary Design/Serialization Choice]] — for the control plane, picks how structs cross.
- [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]] — the API surface is the boundary; making it coarse is design work.
- [[Languages/TypeScript/Practices/Performance/Allocation Hygiene]] — the JS-side allocation discipline that complements buffer reuse.

## Related

- [[Integration/Rust-WASM-TS/Decision and Architecture/When Rust-WASM Is Justified]]
- [[Integration/Rust-WASM-TS/Decision and Architecture/TypeScript Shell Rust Kernel Architecture]]
- [[Integration/Rust-WASM-TS/Boundary Design/Serialization Choice]]
- [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]]
- [[Languages/TypeScript/Practices/Performance/Allocation Hygiene]]
