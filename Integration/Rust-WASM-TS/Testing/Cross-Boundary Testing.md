---
title: Cross-Boundary Testing
tags: [integration, rust, wasm, typescript, testing]
summary: Test in three layers — native Rust + cargo-fuzz, then `wasm-bindgen-test` in Node and headless browsers, then TS-side integration against the generated package.
keywords: [wasm-bindgen-test, cargo-fuzz, native rust tests, integration tests, layered testing, headless browser, node testing]
---

*Three test layers, three different things they can prove. Native Rust tests catch logic; wasm tests catch toolchain drift; TS tests catch boundary-shape drift.*

# Cross-Boundary Testing

> **Rule:** a Rust↔WASM↔TS project tests in three independent layers. Native Rust tests on the pure crate (and `cargo-fuzz` where applicable) prove the algorithm. `wasm-bindgen-test` proves the wasm-target build behaves the same way. TS-side integration tests against the generated package prove the boundary surface is what the application expects. Each layer is independent; gaps at one layer are not closed by piling tests at another.

The Rust↔WASM↔TS pipeline has three places things can go wrong: the Rust logic itself, the wasm-bindgen toolchain compiling and binding it, and the TS-side adapter consuming the generated package. Three test layers are needed because each catches a class of failure the others structurally cannot.

This is the cross-language application of [[Engineering Philosophy/Principles/Evidence Ladder for Testing]] — different layers prove different claims.

## Layer 1 — Native Rust tests on the pure crate

The Rust kernel is, ideally, a normal Rust crate. That means most of its logic can be tested *natively* with `cargo test` — no WASM toolchain involved. This is the cheapest, fastest layer.

```rust
// crates/kernel/src/lib.rs

pub fn parse_header(bytes: &[u8]) -> Result<u32, ParseError> {
    if bytes.len() < 4 {
        return Err(ParseError::TooShort);
    }
    Ok(u32::from_le_bytes([bytes[0], bytes[1], bytes[2], bytes[3]]))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn rejects_short_input() {
        assert!(matches!(parse_header(&[]), Err(ParseError::TooShort)));
        assert!(matches!(parse_header(&[1, 2, 3]), Err(ParseError::TooShort)));
    }

    #[test]
    fn parses_four_bytes() {
        assert_eq!(parse_header(&[0x12, 0x00, 0x00, 0x00]).unwrap(), 0x12);
    }
}
```

This runs as `cargo test` — fast, native, with full debugger / profiler / sanitizer support.

The discipline that makes this layer powerful: **keep the WASM-bound layer thin**. The `#[wasm_bindgen]` annotations sit on a separate function or impl block that *delegates* to the pure Rust logic. The pure logic is what gets exhaustively tested natively; the WASM-bound layer just wires it up.

```rust
// crates/kernel/src/lib.rs

// Pure logic — exhaustively tested natively
pub fn parse_header(bytes: &[u8]) -> Result<u32, ParseError> { /* ... */ }

// Thin WASM-bound layer — just wires up the type translation
#[cfg(target_arch = "wasm32")]
mod wasm {
    use super::*;
    use wasm_bindgen::prelude::*;

    #[wasm_bindgen(js_name = parseHeader)]
    pub fn parse_header_wasm(bytes: &[u8]) -> Result<u32, JsError> {
        parse_header(bytes).map_err(|e| JsError::new(&format!("{:?}", e)))
    }
}
```

The pure `parse_header` is testable on every platform; the WASM-bound version is testable only in a wasm runtime. Most logic bugs live in the pure layer; the wasm layer rarely fails in interesting ways.

### Property tests and fuzzing native

The native Rust layer is also where `proptest` / `quickcheck` and `cargo-fuzz` live:

```rust
// crates/kernel/fuzz/fuzz_targets/parse_header.rs
use kernel::parse_header;
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    // Should never panic — only return Err for invalid input
    let _ = parse_header(data);
});
```

`cargo-fuzz` runs at native speed (much faster than WASM-target fuzzing). For parsers, decoders, and untrusted-input surfaces, fuzz at this layer.

## Layer 2 — `wasm-bindgen-test` for wasm-target behavior

[`wasm-bindgen-test`](https://rustwasm.github.io/wasm-bindgen/wasm-bindgen-test/) runs Rust tests in a wasm runtime — Node.js or a headless browser. It catches issues the native test can't:

- **Toolchain behavior.** Does the wasm-bindgen-emitted glue actually pass arguments through correctly? Does the wasm-target build produce the same output as the native build?
- **Browser / Node runtime differences.** Does the kernel work in browsers' WASM runtime? In Node's? In both?
- **Async behavior with `wasm-bindgen-futures`.** Does the Rust async function correctly produce a JS Promise? Does it await JS Promises correctly?
- **Memory layout differences.** A bug that depends on `usize == u64` (native) vs `usize == u32` (wasm32) shows up here.

```rust
// crates/kernel/tests/wasm_tests.rs (or in src/lib.rs under #[cfg(target_arch = "wasm32")])

use wasm_bindgen_test::*;
wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn parses_in_wasm() {
    let result = kernel::parse_header(&[0x12, 0x00, 0x00, 0x00]);
    assert_eq!(result.unwrap(), 0x12);
}
```

Run with:

```sh
wasm-pack test --node          # in Node.js
wasm-pack test --chrome --headless    # in headless Chromium
wasm-pack test --firefox --headless   # in headless Firefox
```

For a critical-path kernel, run on at least one browser and Node. For a browser-only kernel, run on at least two browsers.

### When wasm-bindgen-test reveals real bugs

In practice, the wasm-bindgen-test layer catches:

- Forgotten `#[wasm_bindgen]` annotations.
- Mismatches between Rust types and the generated TS types.
- Async functions that work natively but hang in WASM (often a missing `JsFuture` await).
- Memory growth bugs (the WASM memory model surfaces issues the native model doesn't).
- Build-profile-specific issues (`panic = "abort"` recovery semantics, see [[Integration/Rust-WASM-TS/Error Handling and DOM/Error Propagation Across Boundary]]).

It is comparatively rare for *logic* bugs to slip past native testing and only show up in wasm-bindgen-test. When that happens, the pure-Rust crate isn't pure enough — there's something the native build doesn't exercise that the wasm build does. Investigate.

## Layer 3 — TS-side integration tests against the generated package

The third layer tests the kernel from the perspective of the TS application that consumes it. This catches:

- **Generated `.d.ts` shape drift.** A Rust change that compiles cleanly may produce a TS surface the application expects to be different. The TS test catches it.
- **Adapter-layer bugs.** The TS application's wrapper around the kernel (the `Result`-shaped facade, the validation, the error mapping) has its own logic.
- **Lifecycle issues.** Does `init()` actually load the WASM? Does `.free()` correctly release memory? Does the kernel survive a long sequence of calls without leaking?
- **Production-shape consumption.** The kernel running through the actual import path the application uses — including any bundler, any wrapper, any lazy-load — not just a directly-imported test fixture.

```ts
// ts-app/test/kernel-integration.test.ts
import { test } from "node:test";
import assert from "node:assert";
import { parseHeader } from "../src/kernel-adapter.js";

test("rejects short input", () => {
  const result = parseHeader(new Uint8Array([1, 2, 3]));
  assert.strictEqual(result.ok, false);
  assert.strictEqual(result.error.kind, "header_too_short");
});

test("parses valid input", () => {
  const result = parseHeader(new Uint8Array([0x12, 0, 0, 0]));
  assert.strictEqual(result.ok, true);
  assert.strictEqual(result.value, 0x12);
});

test("handles repeated calls without leaking", () => {
  for (let i = 0; i < 1000; i++) {
    parseHeader(new Uint8Array([0x12, 0, 0, 0]));
  }
  // Should not throw, should not crash, memory should not grow unboundedly
});
```

These tests run with `node:test` (or whatever the project's runner is — see [[Languages/TypeScript/Practices/Testing/TypeScript Test Tooling]]). They consume the kernel exactly as production will.

For browser-targeted kernels, the same tests can run via Playwright (see TS Test Tooling) to exercise the kernel in a real browser runtime.

## What goes where

Rough allocation of test responsibility:

| Test type | Layer 1 (native Rust) | Layer 2 (wasm-bindgen-test) | Layer 3 (TS integration) |
|---|---|---|---|
| Pure algorithm correctness | ✓ primary | (smoke only) | (smoke only) |
| Edge cases on inputs | ✓ primary | (smoke only) | — |
| Property-based tests | ✓ primary (proptest) | — | (smaller, in TS) |
| Fuzzing | ✓ primary (cargo-fuzz) | — | — |
| Memory layout / `usize` differences | (won't catch) | ✓ primary | (won't catch) |
| Async / Promise bridging | (limited) | ✓ primary | (also exercises) |
| Generated `.d.ts` shape | — | — | ✓ primary |
| Adapter-layer logic | — | — | ✓ primary |
| Production import-path | — | — | ✓ primary |
| Browser runtime behavior | — | ✓ if `--chrome`/`--firefox` | (exercises via Playwright) |

The pattern: most test work is at Layer 1 (native, fast); Layer 2 is the toolchain smoke check; Layer 3 is the production-shape sanity check.

## Test profile and panic strategy

Critical: the wasm-bindgen-test layer should run with the *same* Cargo profile the project ships in production. Panic recovery, optimization-related bugs, and binary-size effects are profile-specific. A test suite that passes under `cargo test` (native, debug) and `wasm-pack test` (debug profile by default) but isn't run against `--release` may miss bugs that only surface at `opt-level = 3` with `panic = "abort"`.

Add a `wasm-pack test --release` step to CI for critical-path kernels. It's slower; it catches what the debug build misses.

## Composing with other practices

- [[Engineering Philosophy/Principles/Evidence Ladder for Testing]] — the universal principle this layered scheme operationalizes.
- [[Engineering Philosophy/Principles/Sharp Oracles]] — every test, every layer, needs a sharp oracle.
- [[Languages/Rust/Practices/Testing/Three Levels of Tests]] — the Rust-side layered testing pattern; the kernel reuses it.
- [[Languages/TypeScript/Practices/Testing/TypeScript Test Tooling]] — the TS-side test runner choices.
- [[Integration/Rust-WASM-TS/Error Handling and DOM/Error Propagation Across Boundary]] — testing panic strategy is profile-specific.
- [[Integration/Rust-WASM-TS/Tooling and Build/WASM Build and Loading]] — the build profile under test.

## Related

- [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]
- [[Languages/Rust/Practices/Testing/Three Levels of Tests]]
- [[Languages/TypeScript/Practices/Testing/TypeScript Test Tooling]]
- [[Integration/Rust-WASM-TS/Error Handling and DOM/Error Propagation Across Boundary]]
- [[Integration/Rust-WASM-TS/Tooling and Build/WASM Build and Loading]]
