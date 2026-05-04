---
title: Error Propagation Across Boundary
tags: [integration, rust, wasm, error-handling, panic]
summary: Rust returns `Result<T, E>`; `Err` becomes a JS exception caught at containment boundaries. Async via `wasm-bindgen-futures`. Panic strategy is profile-specific.
keywords: [result, jserror, wasm-bindgen-futures, panic abort, panic strategy, exception, async error, containment boundary]
---

*Rust returns Results; JS catches exceptions. wasm-bindgen translates between them. The translation is mechanical — but panic recovery is profile-specific, so test the exact build you ship.*

# Error Propagation Across Boundary

> **Rule:** Rust public functions exported via `#[wasm_bindgen]` return `Result<T, E>`. The `Ok(T)` variant is converted to the JS-side return value normally; the `Err(E)` variant becomes a JavaScript exception. The TS side catches at containment boundaries (per [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]]). `JsError` is the right type for Error-shaped throws. Panics are a separate concern with profile-specific behavior; if recovery matters, test the exact panic strategy and runtime you ship.

The error model spans two languages with different conventions: Rust's typed `Result<T, E>` and JavaScript's untyped `throw`/`catch`. wasm-bindgen bridges them mechanically, but the discipline on each side stays distinct.

## The mapping

A Rust public function:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn parse_header(bytes: &[u8]) -> Result<u32, JsError> {
    if bytes.len() < 4 {
        return Err(JsError::new("header too short"));
    }
    Ok(u32::from_le_bytes([bytes[0], bytes[1], bytes[2], bytes[3]]))
}
```

The TS side:

```ts
import { parse_header } from "./pkg/kernel.js";

// Generated TS signature: parse_header(bytes: Uint8Array): number;
// Note: Result<u32, JsError> in Rust → number return + throw on error in TS.

try {
  const value = parse_header(bytes);
  // ... use value ...
} catch (err: unknown) {
  // err is the JsError-equivalent thrown
  if (err instanceof Error) {
    console.error("parse failed:", err.message);
  }
  // handle the failure
}
```

The Rust `Err` variant becomes a thrown JS exception. The TS side catches it like any other exception. The generated `.d.ts` does not signal that the function may throw — that's a TS limitation, not a wasm-bindgen one. Document the failure modes in the TS application's adapter layer.

## `JsError` vs custom error types

Two ways to produce the JS-side exception:

### `JsError` (recommended for most cases)

`JsError` is wasm-bindgen's convenience type for producing a real JavaScript `Error` object on the JS side:

```rust
return Err(JsError::new("descriptive message"));
```

The TS side gets a thrown `Error` whose `.message` is the string passed in. Useful for messages and stack traces. Good default.

### Custom error types

For richer error information, define a serializable error type:

```rust
use serde::Serialize;
use wasm_bindgen::prelude::*;

#[derive(Serialize)]
#[serde(tag = "kind")]
pub enum ParseError {
    HeaderTooShort { actual: usize, expected: usize },
    InvalidMagic { found: u32, expected: u32 },
    Truncated { at_offset: usize },
}

impl From<ParseError> for JsValue {
    fn from(err: ParseError) -> Self {
        serde_wasm_bindgen::to_value(&err).unwrap()
    }
}

#[wasm_bindgen]
pub fn parse_header(bytes: &[u8]) -> Result<u32, JsValue> {
    if bytes.len() < 4 {
        return Err(ParseError::HeaderTooShort { actual: bytes.len(), expected: 4 }.into());
    }
    // ...
    Ok(0)
}
```

The TS side gets a thrown structured object:

```ts
try {
  const value = parse_header(bytes);
} catch (err: unknown) {
  // err is the structured object: { kind: "HeaderTooShort", actual: 2, expected: 4 }
  if (typeof err === "object" && err !== null && "kind" in err) {
    // narrow on err.kind
  }
}
```

This pattern composes with [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]] — the thrown error is a discriminated union the TS handler can switch over.

## What the TS side should do

[[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]] applies directly. The TS-side discipline:

- **Catch at containment boundaries**, not everywhere. The request handler, the worker message handler, the top-level entry point.
- **Validate the caught value's shape** before acting on it. The thrown error from WASM is `unknown`; narrow before reading.
- **Don't swallow.** A caught WASM error that gets logged-and-continued either belongs to a known recoverable category (handle the recovery explicitly) or is a programmer error (re-throw to the containment boundary).
- **Map to the application's error vocabulary**. The application's own `Result<T, E>` types or error enum types likely don't match WASM's. Translate at the adapter layer.

A typical TS-side adapter:

```ts
import { parse_header } from "./pkg/kernel.js";
import { ok, err, Result } from "./result.js";

type ParseHeaderError =
  | { kind: "header_too_short" }
  | { kind: "invalid_magic" }
  | { kind: "internal" };

export function parseHeader(bytes: Uint8Array): Result<number, ParseHeaderError> {
  try {
    return ok(parse_header(bytes));
  } catch (e: unknown) {
    if (typeof e === "object" && e !== null && "kind" in e) {
      switch (e.kind) {
        case "HeaderTooShort": return err({ kind: "header_too_short" });
        case "InvalidMagic":   return err({ kind: "invalid_magic" });
        default:               return err({ kind: "internal" });
      }
    }
    return err({ kind: "internal" });
  }
}
```

The adapter converts WASM's exception-shaped errors into the application's typed `Result<T, E>`. Consumers downstream of the adapter never see a thrown WASM exception.

## Async errors via `wasm-bindgen-futures`

For async Rust functions:

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;

#[wasm_bindgen]
pub async fn fetch_and_decode(url: &str) -> Result<Vec<u8>, JsError> {
    let resp = JsFuture::from(web_sys::window().unwrap().fetch_with_str(url)).await
        .map_err(|e| JsError::new(&format!("fetch failed: {:?}", e)))?;
    // ...
    Ok(vec![])
}
```

The TS side gets a Promise:

```ts
try {
  const bytes = await fetch_and_decode("/api/data");
} catch (err: unknown) {
  // handle async error the same way as sync
}
```

The exception path is the same; the only difference is that the Promise rejects rather than throwing synchronously. Use `await` + `try/catch` or `.catch()` — both work.

## Panic handling — profile-specific behavior

Panics are *different* from errors. A Rust `panic!` is the analog of "this should not happen"; in WASM compiled with `panic = "abort"`, a panic terminates the WASM instance.

Three configurations to know:

### `panic = "abort"` (size-optimized, common default)

A panic terminates the WASM instance. The TS side observes this as the function not returning — typically as `RuntimeError` or "unreachable" thrown at the call site. The Wasm instance is in an undefined state and should not be reused.

Recovery semantics: the TS side handles the thrown error like any other failure, but the *Wasm instance* is dead. New calls into it may fail unpredictably. The right pattern is to treat a panic as instance-fatal and re-instantiate from the cached `Module` (see [[Integration/Rust-WASM-TS/Tooling and Build/WASM Build and Loading]]).

### `panic = "unwind"` (default for most Rust builds, larger binary)

A panic unwinds the stack. With recent wasm-bindgen versions, some recovery semantics may be available — the function unwinds, the panic surfaces as a thrown exception on the JS side, and the WASM instance may be reusable depending on the panic point.

Recovery semantics: more permissive, but binary size grows. Test the exact behavior for your toolchain version.

### Catching panics in Rust

Some teams use `std::panic::catch_unwind` to catch panics at the public API boundary:

```rust
use std::panic;

#[wasm_bindgen]
pub fn safe_compute(input: &[u8]) -> Result<Vec<u8>, JsError> {
    let result = panic::catch_unwind(|| compute_inner(input));
    match result {
        Ok(Ok(v)) => Ok(v),
        Ok(Err(e)) => Err(e),
        Err(_panic) => Err(JsError::new("internal panic in compute")),
    }
}
```

This works only with `panic = "unwind"`. With `panic = "abort"`, `catch_unwind` doesn't catch — the WASM instance is already dead.

### The standards rule

The source report is explicit on this: "panic strategy is evolving... if you intend to rely on recovery semantics you must test the exact profile you will ship." Translation:

- **Pick a panic strategy explicitly.** Don't accept the default without thinking.
- **Document it in the kernel's API.** Consumers should know whether a panic is recoverable.
- **Test for it.** The wasm-bindgen-test suite (see [[Integration/Rust-WASM-TS/Testing/Cross-Boundary Testing]]) should include panic-and-recovery tests for the exact build profile you ship to production.
- **Don't rely on internet docs over your own measurement.** Behavior varies across wasm-bindgen versions, Rust versions, and panic strategies.

The safe default for production critical-path kernels: `panic = "abort"`, treat panics as instance-fatal, re-instantiate fresh on panic. It's the simplest model and the smallest binary.

## Composing with other practices

- [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]] — the TS-side discipline applied to WASM-thrown errors.
- [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]] — the adapter layer pattern translates WASM exceptions to TS `Result<T, E>`.
- [[Languages/Rust/Practices/Error Handling/Result vs Panic]] — Rust's native partition; the same partition crosses the boundary.
- [[Integration/Rust-WASM-TS/Tooling and Build/WASM Build and Loading]] — the panic strategy is set in the Cargo profile.
- [[Integration/Rust-WASM-TS/Testing/Cross-Boundary Testing]] — the place panic recovery gets tested for the exact profile.

## Related

- [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]]
- [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]]
- [[Languages/Rust/Practices/Error Handling/Result vs Panic]]
- [[Integration/Rust-WASM-TS/Tooling and Build/WASM Build and Loading]]
- [[Integration/Rust-WASM-TS/Testing/Cross-Boundary Testing]]
