---
title: Generated TypeScript Surfaces
tags: [integration, rust, wasm, typescript, generated-code, api]
summary: wasm-bindgen's generated `.d.ts` is a public API; review its diff on every change and design Rust exports to produce a usable TS surface.
keywords: [d.ts, declaration file, generated bindings, wasm-bindgen output, api stability, public surface, abi]
---

*The .d.ts wasm-bindgen emits is the contract your TypeScript application sees. Design it like a contract — review it, gate on its shape, evolve it deliberately.*

# Generated TypeScript Surfaces

> **Rule:** the `.d.ts` declarations wasm-bindgen emits are a public API. Each change to a Rust `#[wasm_bindgen]` export changes the TS surface; each change is reviewed for shape stability the same way a manually-written API would be. The generated surface is also an artifact: gate on its diff in CI.

When wasm-bindgen processes a Rust crate, it produces three artifacts: the `.wasm` binary, a `.js` glue file that handles the JS-side runtime, and a `.d.ts` declaration file that types the generated JS for TypeScript consumers. The `.d.ts` is what a TS application sees when it `import`s the generated package. Treating it as an emergent side-effect of the Rust code rather than a designed artifact is the most common cause of surprised TS consumers.

## What the generated surface looks like

For a Rust function:

```rust
#[wasm_bindgen]
pub fn process_image(
    input: &[u8],
    width: u32,
    height: u32,
) -> Result<Vec<u8>, JsError> {
    // ...
}
```

wasm-bindgen emits something like:

```ts
// pkg/image_kernel.d.ts (generated)
export function process_image(
  input: Uint8Array,
  width: number,
  height: number,
): Uint8Array;
```

The TS signature uses idiomatic JS types: `Uint8Array` for `&[u8]`, `number` for `u32`. `Result<T, JsError>` becomes a non-`Result` return type with errors thrown as exceptions (see [[Integration/Rust-WASM-TS/Error Handling and DOM/Error Propagation Across Boundary]]).

For exported types, wasm-bindgen emits classes:

```rust
#[wasm_bindgen]
pub struct ImageProcessor {
    inner: Inner,
}

#[wasm_bindgen]
impl ImageProcessor {
    #[wasm_bindgen(constructor)]
    pub fn new(config: JsValue) -> Result<ImageProcessor, JsError> { /* ... */ }

    pub fn process(&mut self, input: &[u8]) -> Result<Uint8Array, JsError> { /* ... */ }

    pub fn close(self) { /* ... */ }
}
```

becomes:

```ts
export class ImageProcessor {
  constructor(config: any);
  process(input: Uint8Array): Uint8Array;
  close(): void;
  free(): void;  // wasm-bindgen-managed memory cleanup
}
```

The `.free()` method is added by wasm-bindgen for explicit memory management. Consumers should call it when done with the object, or use `using` (TS 5.2+) for deterministic cleanup.

## The "this is an API" rule

The generated `.d.ts` is what every downstream TypeScript consumer sees. Treating it like the side-effect of a Rust crate produces three failure modes:

1. **Surprise.** A small Rust refactor (renaming a public method, adding a parameter) shifts the TS surface. Consumers break, often without warning.
2. **Bad ergonomics.** Rust idioms that feel natural in Rust translate awkwardly to TS — `Option<T>` becomes nullable; complex enums become magic constants; tuple return types become arrays. A TS consumer's experience is worse than necessary.
3. **API drift.** Without explicit review, the generated surface accumulates inconsistencies. Some functions return `Uint8Array`; some return `Array<number>`; the `.d.ts` is a museum of historical decisions.

The fix: review the generated surface as part of every Rust change to a `#[wasm_bindgen]` boundary. Design for the consumer.

## Designing for the TS consumer

The Rust side decides what the TS side sees. Some heuristics:

### Use `Vec<u8>` returning `Uint8Array`, not `Vec<i32>` returning `Int32Array`

Buffer types should match what the consumer naturally has. JS-native buffer types (`Uint8Array`, `Float32Array`) are the path of least friction.

### Avoid `Option<T>` in public signatures when you can

`Option<T>` becomes `T | undefined` in TS. Often a `Result<T, E>` (which becomes throw-or-return) is clearer for the consumer than a maybe-null return. When `Option` is genuinely the right shape (a successful "no result" outcome), document it.

### Avoid raw `JsValue` in public signatures

`JsValue` is `any` from the TS perspective. A function returning `JsValue` gives the consumer no type information at all. Use a typed Rust struct with `#[derive(Serialize, Deserialize)]` and `serde-wasm-bindgen` to produce a properly-typed JS object on the TS side (see [[Integration/Rust-WASM-TS/Boundary Design/Serialization Choice]]).

### Use `#[wasm_bindgen(getter)]` and `#[wasm_bindgen(setter)]` for property access

Rust struct fields can be exposed as TS-side properties rather than methods, which reads better:

```rust
#[wasm_bindgen]
impl Frame {
    #[wasm_bindgen(getter)]
    pub fn width(&self) -> u32 { self.width }

    #[wasm_bindgen(getter)]
    pub fn height(&self) -> u32 { self.height }
}
```

Generates:

```ts
export class Frame {
  readonly width: number;
  readonly height: number;
}
```

A TS consumer reads `frame.width` rather than `frame.width()`. Small improvement; pays back over many call sites.

### Use `#[wasm_bindgen(typescript_custom_section)]` for richer types

For declarations the wasm-bindgen mapping doesn't produce well, you can inject custom TypeScript directly:

```rust
#[wasm_bindgen(typescript_custom_section)]
const TS_APPEND_CONTENT: &'static str = r#"
export type Color = "red" | "green" | "blue";
"#;
```

Use sparingly; this is an escape hatch, not a primary path.

## Gating on shape stability

The `.d.ts` is an artifact. Gate on it in CI:

1. **Commit the generated `.d.ts`** (alongside or separately from the `.wasm` and `.js`) so a diff is visible on every change.
2. **Block merges that change the `.d.ts` without explicit acknowledgment.** A reviewer must consciously approve the API change. Hidden-via-codegen API changes are the failure this gate catches.
3. **Version the package strictly.** A `.d.ts` change that removes or renames a member is a breaking change. The package version (whether internal or public-npm) reflects that.
4. **Document the `.d.ts` consumer surface separately from the implementation.** Even if Rust docstrings are extracted, the consumer-facing docs live with the TS package.

## Versioning and ABI stability

WebAssembly itself doesn't have a stable cross-version ABI for high-level data types — `wasm-bindgen` versions can produce slightly different glue. Pinning the wasm-bindgen version, the rustc version, and the wasm-opt version is part of the build's reproducibility:

- Pin wasm-bindgen to a specific minor (e.g. `wasm-bindgen = "=0.2.95"`).
- Pin the rustc toolchain via `rust-toolchain.toml`.
- Pin wasm-opt version (Binaryen) in the build script.

When upgrading any of these, the `.d.ts` diff should be reviewed exactly as if it were a manual API change. Most upgrades are no-op for the surface; some are not.

## What about TS consumers using the package?

The TS application that imports the generated package should:

- **Not modify the generated files.** Treat them as build output.
- **Wrap them in an application-side adapter** if the generated API is awkward for the consumer's domain. Don't expose the `wasm-bindgen` shape directly to the rest of the app.
- **Validate inputs going into the kernel** at the adapter layer (see [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]]).
- **Call `.free()` on long-lived kernel objects** (or use `using`) to release linear-memory allocations. wasm-bindgen does not garbage-collect Rust-side allocations; the TS side must own the lifecycle.

## Composing with other practices

- [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]] — the API shape is what makes calls coarse or chatty.
- [[Integration/Rust-WASM-TS/Boundary Design/Serialization Choice]] — the serialization choice affects what the generated `.d.ts` looks like.
- [[Integration/Rust-WASM-TS/Tooling and Build/wasm-bindgen and wasm-pack]] — the tooling that emits the `.d.ts`.
- [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]] — the discipline TS consumers apply when calling into the kernel.
- [[Languages/Rust/Practices/Foundational/Predictable APIs]] — the same posture, applied to the Rust public surface that produces the TS surface.

## Related

- [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]]
- [[Integration/Rust-WASM-TS/Boundary Design/Serialization Choice]]
- [[Integration/Rust-WASM-TS/Tooling and Build/wasm-bindgen and wasm-pack]]
- [[Integration/Rust-WASM-TS/Error Handling and DOM/Error Propagation Across Boundary]]
- [[Languages/Rust/Practices/Foundational/Predictable APIs]]
