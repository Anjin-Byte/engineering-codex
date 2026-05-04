---
title: Serialization Choice
tags: [integration, rust, wasm, typescript, serialization]
summary: Three boundary-serialization choices — `serde-wasm-bindgen` (ergonomic default), FlatBuffers (zero-copy hot binary), Protobuf (cross-service schemas). Pick by ergonomics, throughput, or schema longevity.
keywords: [serde-wasm-bindgen, flatbuffers, protobuf, jsvalue, bincode, schema, serialization, boundary format]
---

*Three formats, three priorities. Default to ergonomics; promote to a binary schema when the boundary becomes hot or when the schema must outlive the project.*

# Serialization Choice

> **Rule:** the boundary serialization format is a deliberate choice, not a default. `serde-wasm-bindgen` is the right default for control-plane Rust↔JS object exchange. FlatBuffers is the right answer when the boundary is hot binary data and zero-copy access matters. Protocol Buffers is the right answer when a durable cross-service schema dominates.

The choice depends on what the boundary is *for*. Control-plane traffic (configuration, lifecycle, occasional structured data) optimizes for developer ergonomics; data-plane traffic (the hot path; megabytes per call) optimizes for throughput; cross-service traffic (a schema shared with other services in other languages) optimizes for schema interoperability and stability.

## The three formats

### `serde-wasm-bindgen` — ergonomic default

The wasm-bindgen guide describes `serde-wasm-bindgen` as a way to "serialize arbitrary Serde-supported types to and from `JsValue`." It uses Rust's `serde` framework on the Rust side, producing idiomatic JS objects on the JS side and vice versa.

```rust
use serde::{Deserialize, Serialize};
use serde_wasm_bindgen::{to_value, from_value};
use wasm_bindgen::prelude::*;

#[derive(Serialize, Deserialize)]
pub struct Config {
    pub width: u32,
    pub height: u32,
    pub mode: String,
}

#[wasm_bindgen]
pub fn configure(input: JsValue) -> Result<JsValue, JsError> {
    let config: Config = from_value(input)?;
    // ... use config ...
    let result = serde_json::json!({ "ok": true });
    Ok(to_value(&result)?)
}
```

The TS side passes a normal object; the Rust side gets a typed `Config`. No JSON intermediary. Smaller code size and often better performance than JSON-string round-tripping; *not* zero-copy and not the fastest possible binary path.

**Pick `serde-wasm-bindgen` when:**
- The boundary is the control plane (configuration, lifecycle, rare calls).
- Developer ergonomics matter; the API is read by humans regularly.
- Throughput is bounded by the rest of the work, not by the serialization step.
- The data shapes evolve frequently and a binary schema would slow iteration.

### FlatBuffers — zero-copy hot binary

[FlatBuffers](https://flatbuffers.dev/) is a schema-based binary format with a unique property: data can be read directly from the serialized buffer without parsing into native objects. The buffer *is* the in-memory representation. For hot exchange of structured binary payloads between Rust and JS, this can eliminate a copy.

```fbs
// schema.fbs
table Frame {
  width:uint32;
  height:uint32;
  pixels:[uint8];
}
root_type Frame;
```

Generated Rust and TS bindings let each side read fields from a shared `ArrayBuffer` without deserialization. The buffer transfers (zero-copy if it's a transferable) and both sides interpret the same bytes.

**Pick FlatBuffers when:**
- The boundary is hot data-plane traffic.
- Payloads are large and structured (codec frames, simulation state, parsed records).
- Zero-copy access is a measured win, not a hope.
- Both sides have flatbuffers tooling available.

**Don't pick FlatBuffers when:**
- The data is unstructured bytes (just use `&[u8]`).
- The schema changes frequently — adding fields is fine; restructuring is harder than `serde`.
- The team isn't fluent with the schema-compile workflow.

### Protocol Buffers — durable cross-service schema

[Protocol Buffers](https://protobuf.dev/) is the mature, ecosystem-rich schema format used widely for cross-service communication. Less zero-copy than FlatBuffers but with broader ecosystem support, including gRPC integration and rich tooling.

```proto
// schema.proto
syntax = "proto3";

message Frame {
  uint32 width = 1;
  uint32 height = 2;
  bytes pixels = 3;
}
```

**Pick Protobuf when:**
- The schema must be shared across services in multiple languages.
- The same data shape exists at the WASM boundary *and* at a service-RPC boundary (you may as well use one format).
- Long-term schema evolution and backward compatibility matter.
- gRPC is already in the architecture.

**Don't pick Protobuf when:**
- The data crosses only the JS↔WASM boundary; a cross-service format adds overhead without benefit.
- The hot path is the bottleneck — Protobuf encode/decode is slower than FlatBuffers' zero-copy.
- The schema is single-team-owned and evolves freely; Protobuf's rigor is a tax in that case.

## Decision matrix

| Boundary character | Recommended format | Reasoning |
|---|---|---|
| Control plane: configs, lifecycle, status queries | `serde-wasm-bindgen` | Ergonomics > throughput |
| Data plane: small structured records, hot but not extreme | `serde-wasm-bindgen` for prototypes; profile and consider FlatBuffers | Default to easy; promote when measurement justifies |
| Data plane: large binary payloads, hot path, zero-copy beneficial | FlatBuffers | Zero-copy access is the win |
| Hot binary path with no structure | Native `&[u8]` / `Uint8Array` | Schema is overhead; bytes are bytes |
| Cross-service schema also used at WASM boundary | Protobuf | Use the same format end-to-end |
| Cross-service schema, WASM boundary is hot path | Profile both | Sometimes Protobuf is fine; sometimes a separate FlatBuffers shim per WASM call is faster |

## Anti-patterns

- **Multiple formats in one boundary.** Pick one per logical surface. Mixing formats produces conversion costs that defeat the optimization.
- **Choosing a binary format prematurely.** `serde-wasm-bindgen` is fine until profile shows the boundary is the bottleneck. Premature schema-compilation is overhead.
- **Re-implementing JSON.** TypeScript and Rust both have excellent JSON support; the wasm-bindgen layer can pass JSON strings. But this is rarely the right path because each side then parses JSON, paying the cost twice. `serde-wasm-bindgen` skips the JSON intermediate.
- **Using `JsValue` everywhere.** The wasm-bindgen `JsValue` type is *opaque on the Rust side*; every property access is a host call back to JS. Fine for occasional control-plane work; awful for hot paths.

## What about `bincode`, `MessagePack`, `CBOR`?

These are reasonable alternatives in narrow cases:

- **`bincode`** — Rust-native binary serde. Useful for Rust↔Rust boundaries (e.g., serializing for storage); not a primary choice for the JS↔WASM boundary because the JS side has no native bincode tooling.
- **`MessagePack`** — JSON-shaped but binary. Reasonable when JSON-like ergonomics are wanted with smaller wire size. Slower than FlatBuffers for hot paths.
- **`CBOR`** — IETF-standardized MessagePack-equivalent. Good for cross-protocol contexts.

None of these supplant the three primary choices for the JS↔WASM boundary in typical projects.

## Composing with other practices

- [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]] — the upstream decision: control plane (where serialization shape matters less) vs data plane (where it matters most).
- [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]] — the choice of serialization format affects the generated TS API.
- [[Engineering Philosophy/Principles/Layered Risk Categories]] — the boundary is part of the type-model layer; getting the format wrong shows up as decoded-data confusion.

## Related

- [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]]
- [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]]
- [[Integration/Rust-WASM-TS/Tooling and Build/wasm-bindgen and wasm-pack]]
- [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]]
