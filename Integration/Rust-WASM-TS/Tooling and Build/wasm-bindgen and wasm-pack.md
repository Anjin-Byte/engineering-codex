---
title: wasm-bindgen and wasm-pack
tags: [integration, rust, wasm, tooling, wasm-bindgen, wasm-pack]
summary: `wasm-bindgen` generates the JS glue and TS declarations; `wasm-pack` runs the build and packaging; `wasm-bindgen-futures` bridges Rust `Future`s and JS `Promise`s.
keywords: [wasm-bindgen, wasm-pack, wasm-bindgen-futures, js-sys, web-sys, target bundler, target nodejs, target web]
---

*wasm-bindgen makes the boundary; wasm-pack ships it. Different tools, different jobs. Knowing which one is responsible for what is the foundation of an unfrustrating Rust↔WASM toolchain.*

# wasm-bindgen and wasm-pack

> **Rule:** in a Rust↔WASM↔TS project, `wasm-bindgen` owns the interop layer between Rust and JS (generates the JS glue and `.d.ts` declarations), and `wasm-pack` owns the packaging and build workflow (compiles to `.wasm`, generates package metadata, emits target-specific loading strategies). They are complementary; neither replaces the other.

The naming is unfortunate — both have "wasm" in the name and both come from the same project (`rustwasm`) — but they're two different tools that compose. A useful mental model:

- **`wasm-bindgen`** is what *makes Rust callable from JS at all*. It's the macro layer (`#[wasm_bindgen]`) plus a code generator that emits the JS-side glue code wrapping the WASM ABI.
- **`wasm-pack`** is what *makes the result an npm-shaped package*. It runs Cargo, runs `wasm-bindgen`, generates `package.json`, and chooses a loading strategy based on the target environment.

You can use `wasm-bindgen` without `wasm-pack` (running Cargo and `wasm-bindgen-cli` manually). You cannot use `wasm-pack` without `wasm-bindgen` — the package always includes wasm-bindgen output.

## The three tools to know

### `wasm-bindgen` — the interop layer

[wasm-bindgen](https://rustwasm.github.io/wasm-bindgen/) does several things:

1. **Macro layer.** `#[wasm_bindgen]` annotations on Rust functions, structs, and impls are read by a procedural macro. The macro generates the boilerplate that exposes the Rust item to JS.
2. **Code generation.** A separate `wasm-bindgen-cli` tool processes the compiled `.wasm` and emits a `.js` glue file plus a `.d.ts` declaration file alongside it.
3. **Type-bridging conventions.** It defines how Rust types map to JS types (`&[u8]` ↔ `Uint8Array`, `String` ↔ `string`, `Result<T, E>` ↔ throw-or-return).
4. **Companion crates.**
   - `js-sys` — bindings to standard ECMAScript globals (`Array`, `Object`, `Promise`, `Math`, etc.).
   - `web-sys` — bindings to Web APIs (DOM, fetch, WebGL, etc.). Feature-gated; enable only the parts you use.

A typical Rust file using wasm-bindgen:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn process(input: &[u8]) -> Result<Vec<u8>, JsError> {
    // ...
    Ok(vec![])
}
```

That's the entire surface from the developer's perspective. The macro and the code generator do everything else.

### `wasm-pack` — the build and package workflow

[wasm-pack](https://rustwasm.github.io/wasm-pack/) is a build tool that:

1. **Runs `cargo build` with the WASM target.**
2. **Runs `wasm-bindgen-cli` on the resulting `.wasm`** to produce JS glue and TS declarations.
3. **Generates a `package.json`** with appropriate `main`, `module`, `types`, `files`, and `sideEffects` fields.
4. **Selects a loading strategy** based on the target flag.

The targets are the primary user-visible knob:

| Target | Use case |
|---|---|
| `--target bundler` | For consumption by a bundler (Webpack, Vite, esbuild). The default; produces ESM that the bundler resolves. |
| `--target nodejs` | For Node.js consumption. Uses `require`/CommonJS or ESM depending on package config. |
| `--target web` | For direct browser consumption without a bundler. Includes manual `init` for fetching the `.wasm` file. |
| `--target no-modules` | For older browser environments without ES modules. Emits a single global. |
| `--target deno` | For Deno consumption. |

Each target produces a different loading strategy in the JS glue. The choice depends on how the package will be consumed.

```sh
# Browser-bundled consumption
wasm-pack build --target bundler --release

# Node service consumption
wasm-pack build --target nodejs --release

# Browser direct (no bundler)
wasm-pack build --target web --release
```

### `wasm-bindgen-futures` — async bridging

[wasm-bindgen-futures](https://docs.rs/wasm-bindgen-futures/) bridges Rust `Future`s and JavaScript `Promise`s:

- **Rust → JS**: a Rust `async fn` exported via `#[wasm_bindgen]` returns a `Promise` to JS callers.
- **JS → Rust**: a `JsFuture` wraps a JS Promise so Rust async code can `await` it.

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;

#[wasm_bindgen]
pub async fn fetch_and_process(url: &str) -> Result<Vec<u8>, JsError> {
    let resp = JsFuture::from(web_sys::window().unwrap().fetch_with_str(url)).await?;
    // ...
    Ok(vec![])
}
```

Without `wasm-bindgen-futures`, async crossing the boundary is impossible. With it, `async` just works on both sides. The cost is a small runtime; well worth it for any non-trivial async workflow.

## When to use which

The default workflow:

```
Rust source           ────► cargo build (--target wasm32-unknown-unknown)
   │                                │
   │                                ▼
#[wasm_bindgen]       ────► wasm-bindgen-cli (called by wasm-pack)
   macros                           │
                                    ▼
                                Output: .wasm + .js + .d.ts + package.json
```

`wasm-pack` runs the whole pipeline. For most projects, `wasm-pack build --target X` is the entire build step.

You'd reach for raw `wasm-bindgen-cli` (without `wasm-pack`) when:

- Your build pipeline already produces an npm package and you want only the glue (e.g. Vite plugins, Rollup plugins).
- You need a non-standard packaging that wasm-pack doesn't emit.
- You're building a custom toolchain.

For any new project, start with `wasm-pack`; reach for raw `wasm-bindgen-cli` only when wasm-pack actively gets in the way.

## Project layout that supports both

A typical project layout:

```
project-root/
├── crates/
│   └── kernel/
│       ├── Cargo.toml          # crate-type = ["cdylib", "rlib"]
│       └── src/
│           └── lib.rs          # #[wasm_bindgen] exports
├── pkg/                         # wasm-pack output (gitignored or build-only)
│   ├── kernel.js
│   ├── kernel.d.ts
│   ├── kernel_bg.wasm
│   └── package.json
├── ts-app/
│   ├── package.json             # depends on "../crates/kernel/pkg" or similar
│   ├── tsconfig.json
│   └── src/
└── package.json                 # workspace root if applicable
```

The Rust crate has `crate-type = ["cdylib", "rlib"]` in `Cargo.toml`:

- `cdylib` — produces the `.wasm` artifact.
- `rlib` — keeps the crate usable as a normal Rust library, which lets you write native Rust tests against the same code (see [[Integration/Rust-WASM-TS/Testing/Cross-Boundary Testing]]).

The TS application consumes the `pkg/` output as if it were any npm package. The exact import path depends on the wasm-pack target and the workspace configuration.

## Pinning versions

Reproducible builds require pinning the toolchain:

```toml
# rust-toolchain.toml (in project root)
[toolchain]
channel = "1.78.0"  # or whatever stable version
targets = ["wasm32-unknown-unknown"]
```

```toml
# crates/kernel/Cargo.toml
[dependencies]
wasm-bindgen = "=0.2.95"          # pin minor + patch
js-sys = "0.3"
wasm-bindgen-futures = "0.4"

[dependencies.web-sys]
version = "0.3"
features = ["Window", "Performance"]  # explicit feature list
```

The `wasm-bindgen` macro and the `wasm-bindgen-cli` must be the same version. wasm-pack handles this automatically; manual setups need to match versions explicitly. Mismatch produces cryptic glue errors.

## Open questions in the toolchain

The wasm-pack documentation has historically been fragmented across `rustwasm.github.io/wasm-pack/` and older relocations. The wasm-bindgen guide is similarly maintained but references some legacy patterns. When in doubt:

- Verify the canonical URL on `rustwasm.github.io` (the project's umbrella).
- Check the version-specific documentation; some features (panic recovery, async behavior) are version-sensitive.
- Run `wasm-pack --version` and `wasm-bindgen --version` and check the corresponding release notes when behavior surprises you.

This is one of the topics flagged in the source report's "open questions" — the toolchain documentation is real but fragmented, and your project's standard should anchor on the exact versions you ship.

## Composing with other practices

- [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]] — what wasm-bindgen actually emits.
- [[Integration/Rust-WASM-TS/Tooling and Build/WASM Build and Loading]] — the build profiles and post-link optimization that follow wasm-pack.
- [[Integration/Rust-WASM-TS/Error Handling and DOM/Error Propagation Across Boundary]] — how `Result<T, E>` and `JsError` work in wasm-bindgen.
- [[Integration/Rust-WASM-TS/Testing/Cross-Boundary Testing]] — the testing tools that complement these build tools.
- [[Languages/Rust/Workspace/Cargo Workspace Configuration]] — Cargo configuration for the WASM crate.

## Related

- [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]]
- [[Integration/Rust-WASM-TS/Tooling and Build/WASM Build and Loading]]
- [[Integration/Rust-WASM-TS/Error Handling and DOM/Error Propagation Across Boundary]]
- [[Integration/Rust-WASM-TS/Testing/Cross-Boundary Testing]]
- [[Languages/Rust/Workspace/Cargo Workspace Configuration]]
