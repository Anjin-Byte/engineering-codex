---
title: WASM Build and Loading
tags: [integration, rust, wasm, build, wasm-opt, streaming-compilation]
summary: Maintain two release profiles (speed and size); run `wasm-opt` post-link; prefer `instantiateStreaming` for browser loading; cache the compiled `Module` and transfer it to workers.
keywords: [cargo profile, lto, codegen-units, panic abort, wasm-opt, binaryen, instantiateStreaming, compileStreaming, module caching]
---

*Speed and size aren't the same optimization. Build for the workload — Cargo profile per use case, wasm-opt for the post-link pass, streaming compile and module caching for the loader.*

# WASM Build and Loading

> **Rule:** the WASM build is two profiles, not one. A speed-oriented profile for compute kernels (where runtime perf dominates) and a size-oriented profile for browser-distributed assets (where download time dominates). Both flow through `wasm-opt` post-link. Loading uses `WebAssembly.instantiateStreaming()` and caches the compiled `Module` for reuse across instantiations or workers.

The default Cargo `--release` profile is *neither* the right speed-tuned profile nor the right size-tuned profile for WASM. Adopting a deliberate two-profile setup, plus a post-link `wasm-opt` pass, plus a structured loading strategy, separates "WASM that ships" from "WASM that's fast" from "WASM that's small." Each is a knob.

## Two release profiles

### Speed profile — for compute kernels

When the kernel runtime cost is what matters (codecs, simulators, parsers running on hot paths in long-lived sessions):

```toml
# Cargo.toml
[profile.release]
opt-level = 3
lto = "fat"             # or "thin" for faster builds with most of the win
codegen-units = 1       # max optimization across crate
strip = "debuginfo"     # smaller binary, faster startup parse
panic = "abort"         # smaller binary; no unwinding code
```

This produces a larger `.wasm` (often) but the runtime is faster. The trade-offs:

- `opt-level = 3`: highest optimization. `s` and `z` are *size* levels; for speed, stay at `3`.
- `lto = "fat"` runs link-time optimization across the whole dependency graph. Slow to build; meaningfully faster code. `"thin"` is a faster compromise.
- `codegen-units = 1` lets the optimizer see the entire crate as one unit. Reduces code-generation parallelism (slower build) but eliminates artificial barriers to optimization.
- `panic = "abort"` removes panic-unwinding scaffolding. Smaller binary; no `catch_unwind` recovery. See [[Integration/Rust-WASM-TS/Error Handling and DOM/Error Propagation Across Boundary]] for the panic-strategy implications.

### Size profile — for browser distribution

When download size is the bottleneck (browser apps where users wait for the WASM to arrive over the network):

```toml
[profile.release]
opt-level = "z"         # optimize for size
lto = "fat"
codegen-units = 1
strip = "symbols"       # remove all symbols, not just debuginfo
panic = "abort"
```

This produces a smaller `.wasm`, often at some runtime-speed cost. The trade-offs:

- `opt-level = "z"` — most aggressive size optimization. `"s"` is similar with somewhat better runtime performance.
- `strip = "symbols"` — removes everything (no name mangling preserved). Useful for distribution; awful for debugging production crashes.

Some teams use a third profile (`profile.release-debug` or similar) that's speed-optimized but keeps debug symbols for production diagnostics.

## `wasm-opt` post-link pass

[Binaryen](https://github.com/WebAssembly/binaryen)'s `wasm-opt` is the canonical post-link optimizer for `.wasm` files. The Rust/WASM book recommends running it whether or not Cargo's optimization was already aggressive, because it operates at the WASM-module level (after Rust's compilation) and finds optimizations Cargo can't.

Typical invocations:

```sh
# Speed-oriented: aggressive runtime optimization
wasm-opt -O3 -o kernel.opt.wasm kernel.wasm

# Size-oriented: aggressive size optimization
wasm-opt -Oz -o kernel.min.wasm kernel.wasm

# Both — run twice, with different output paths
```

The Rust/WASM book is explicit that `wasm-opt` can shrink size *and* sometimes improve runtime performance. The pass is fast (seconds for typical kernels) and consistently worth running. Make it part of every release build, not an optional polish.

For wasm-pack-based builds, `wasm-opt` integration is automatic in some versions; verify with `wasm-pack --version` and check the docs. Manual integration is straightforward via build scripts.

## Streaming compilation for browser loading

The browser-side loader has two API choices:

### `WebAssembly.compile()` + `WebAssembly.instantiate()` — the older, two-step path

```ts
const buf = await fetch(url).then(r => r.arrayBuffer());
const mod = await WebAssembly.compile(buf);
const instance = await WebAssembly.instantiate(mod, imports);
```

This requires the entire `.wasm` to be downloaded before compilation begins. For large modules, that means waiting for the full transfer.

### `WebAssembly.compileStreaming()` + `WebAssembly.instantiateStreaming()` — preferred

```ts
const mod = await WebAssembly.compileStreaming(fetch(url));
const instance = await WebAssembly.instantiateStreaming(fetch(url), imports);
```

These accept a `Response` (or a Promise resolving to one) and begin compilation as the bytes arrive. For a network-distributed `.wasm`, this overlaps download time and compile time, materially reducing time-to-first-execution.

Web.dev's WASM performance guidance recommends streaming compilation as the default path. Use it whenever the runtime is a browser; for Node, the loaders wasm-pack generates handle the file-system path automatically.

The server must serve the `.wasm` file with `Content-Type: application/wasm` for streaming to work. A misconfigured server falls back to the buffered path with a warning.

## Module caching across instantiations

A compiled `WebAssembly.Module` can be reused. The `Module` is the parsed and compiled bytecode; an `Instance` is one running copy. For workloads that instantiate frequently (e.g. one instance per worker, one per request batch), compiling once and caching the `Module` is a significant win.

```ts
let cachedModule: WebAssembly.Module | null = null;

async function getModule(): Promise<WebAssembly.Module> {
  if (!cachedModule) {
    cachedModule = await WebAssembly.compileStreaming(fetch(url));
  }
  return cachedModule;
}

async function newInstance(imports: WebAssembly.Imports): Promise<WebAssembly.Instance> {
  const mod = await getModule();
  return WebAssembly.instantiate(mod, imports);
}
```

For workers, the compiled `Module` is a transferable structure: send it via `postMessage` and the worker instantiates from it without re-compiling. This is the highest-value pattern for sustained worker-based compute.

```ts
// main thread
const mod = await WebAssembly.compileStreaming(fetch(url));
worker.postMessage({ type: "init", module: mod });

// worker (does not re-fetch, does not re-compile)
self.onmessage = async (ev) => {
  if (ev.data.type === "init") {
    const instance = await WebAssembly.instantiate(ev.data.module, imports);
    // ... use instance ...
  }
};
```

For long-lived workers, keep the worker alive after init rather than spawning fresh ones per task. Worker startup + module transfer are one-time costs; permanent workers amortize those across all subsequent calls.

## Content Security Policy (CSP) considerations

Browser CSP affects WASM loading. The relevant directives:

- **`script-src 'wasm-unsafe-eval'`** — required for WASM compilation in modern browsers. Narrower than `'unsafe-eval'` (which would also enable arbitrary `eval()`).
- **`script-src 'unsafe-eval'`** — also enables WASM but enables arbitrary `eval()` too. Avoid.
- **`script-src` listing the WASM source URL** — for some configurations, the WASM file needs explicit allowlisting.

Use `'wasm-unsafe-eval'` (modern browsers) over the broader alternatives. CSP-strict environments may require build-pipeline cooperation to inject the right directive.

## What goes in the build pipeline

A standard release-build sequence:

```sh
# 1. Cargo build with the chosen profile
cargo build --release --target wasm32-unknown-unknown

# 2. wasm-bindgen to generate JS glue and .d.ts
wasm-bindgen \
  --target bundler \
  --out-dir pkg \
  target/wasm32-unknown-unknown/release/kernel.wasm

# 3. wasm-opt for post-link optimization
wasm-opt -O3 -o pkg/kernel_bg.opt.wasm pkg/kernel_bg.wasm
mv pkg/kernel_bg.opt.wasm pkg/kernel_bg.wasm

# 4. Verify outputs
ls -la pkg/
```

Or, with `wasm-pack`:

```sh
wasm-pack build --target bundler --release
# wasm-pack handles steps 1-2; wasm-opt may need a separate invocation
# depending on wasm-pack version
wasm-opt -O3 -o pkg/kernel_bg.opt.wasm pkg/kernel_bg.wasm
mv pkg/kernel_bg.opt.wasm pkg/kernel_bg.wasm
```

Wire this into CI as a release-pipeline gate. The `.d.ts` diff is part of the same pipeline (see [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]]).

## Composing with other practices

- [[Integration/Rust-WASM-TS/Tooling and Build/wasm-bindgen and wasm-pack]] — the build steps that produce the input to `wasm-opt`.
- [[Integration/Rust-WASM-TS/Decision and Architecture/Workers Before WASM]] — the loading pattern for kernel-in-worker setups.
- [[Integration/Rust-WASM-TS/Error Handling and DOM/Error Propagation Across Boundary]] — `panic = "abort"` interacts with how panics are observable.
- [[Languages/TypeScript/Workspace/Bundling Decision]] — the TS-side bundling decision is parallel to this one.
- [[Languages/TypeScript/Workspace/Release Engineering/Release Gate Pipeline]] — the WASM build is one stage in the broader release gate.

## Related

- [[Integration/Rust-WASM-TS/Tooling and Build/wasm-bindgen and wasm-pack]]
- [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]]
- [[Integration/Rust-WASM-TS/Decision and Architecture/Workers Before WASM]]
- [[Languages/TypeScript/Workspace/Bundling Decision]]
- [[Languages/TypeScript/Workspace/Release Engineering/Release Gate Pipeline]]
