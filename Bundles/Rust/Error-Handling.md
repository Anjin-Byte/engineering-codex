---
title: Error Handling Bundle
tags: [bundle, error-handling]
summary: "Bundle for handling errors: when to use Result vs panic, and how typed enum errors live at library boundaries while anyhow stays at the application edge."
source_trigger: "Handling errors"
bundles: [Result vs Panic, Domain Errors at Boundaries]
---

*The through-line: Result is for runtime situations a caller can act on, panic is for impossible invariants, and library boundaries always speak typed enums.*

# Error Handling Bundle

Read this bundle when handling errors. It bundles the two core rules: the choice between `Result` and `panic` (driven by whether a sensible caller could do anything other than crash), and the boundary discipline that keeps `thiserror` enums at every library boundary while `anyhow` is reserved for the application edge.

---

## Result vs Panic

*Result is the API for things that can go wrong; panic is the alarm for things that should be impossible.*

> **Rule:** `Result` for **recoverable failure**. `panic!` for **bugs and impossible invariants**.

### The distinction

- **`Result<T, E>`** — the failure is something a *caller* might reasonably handle. Examples: file not found, network unreachable, malformed input, required device not available.
- **`panic!` / `expect(...)`** — the failure means *the program itself is wrong*. Examples: an internal index map missing a key that was just inserted, an assertion about a private invariant, an unreachable branch in a closed enum.

The test for which to use: **"could a sensible caller do anything other than crash?"** If yes → `Result`. If no → panic.

### Where panics belong

- `unreachable!()` for branches the type system can't prove dead but logic guarantees.
- `assert!`/`debug_assert!` for invariants you want spelled out.
- `expect("message")` over `unwrap()` *only* when the message documents the invariant being relied on (e.g. `.expect("port was checked nonzero in constructor")`).
- Inside private code where invariants are local and the code that maintains them is nearby.

### Where panics do NOT belong

- Library-public functions that handle external input (parsing, deserialization, network).
- Functions whose failure mode the user might want to handle differently (e.g. retry, prompt, fall back).
- Anywhere the failure depends on **runtime data** rather than **programmer mistake**.

For library-shaped code, panic paths should be **rare and loud** — when they fire, they're surfacing a real bug, not a runtime situation.

### On `unwrap()`

`unwrap()` on a `Result` or `Option` says "I'm certain this can't fail." Often that certainty is wrong, and the panic message gives no hint what was assumed.

Prefer:
- `expect("explanation")` — at least the message documents the assumption.
- `?` — when propagation is the right move.
- A real `match` — when both arms have meaningful behavior.

Allow `unwrap()` freely in:
- Tests
- Build automation and one-off scripts
- Examples in docs (where brevity matters more than robustness)

In production code, prefer `expect` over `unwrap` purely for the error-message hygiene.

### A worked example

```rust
// File missing — caller could retry, prompt user, fall back. Result.
fn load_config(path: &Path) -> Result<Config, LoadError> { /* ... */ }

// Index map corrupted despite private construction — programmer error. Panic.
fn lookup(&self, id: VertexId) -> &Vertex {
    self.vertices.get(&id)
        .expect("VertexId can only be obtained from this map")
}
```

The first failure has a sensible caller-side response. The second one shouldn't be possible without a bug, and silently returning `None` would just push the bug downstream.

### Why this matters for stability

Mixing the two roles erodes both. If `Result` is used for "this should never happen," callers get verbose error handling for impossible cases and stop reading it. If `panic` is used for ordinary failures, the program crashes when it should recover.

Keeping the categories sharp keeps both behaviors meaningful.

*Source: [[Rust Practices/Error Handling/Result vs Panic]]*

---

## Domain Errors at Boundaries

*Libraries return precise typed errors; only the application binary collapses them into anyhow for top-level reporting.*

> **Rule:** typed `enum` errors (via `thiserror`) at library boundaries; `anyhow`-style wrappers only at the *application* boundary.

### The pattern

```rust
#[derive(Debug, thiserror::Error)]
pub enum ParseConfigError {
    #[error("missing field: {0}")]
    MissingField(&'static str),
    #[error("invalid port: {0}")]
    InvalidPort(u16),
    #[error("io error: {0}")]
    Io(#[from] std::io::Error),
}
```

This gets you:
- **Testability** — assert on specific variants without string matching.
- **API clarity** — callers see exactly what can go wrong.
- **Future refactoring** — adding a new failure variant is a deliberate API change, not a silent string drift.
- **Pattern-matching at the call site** — callers can handle `MissingField` differently from `Io` cleanly.

### anyhow vs thiserror

- **`thiserror`** — define typed errors at *library* boundaries: per-crate, per-module, per-public-API. Use this for the public surface of any library crate or domain module.
- **`anyhow`** — wrap heterogeneous errors at the *application* boundary, typically in a binary's `main` or top-level orchestration. Convenient for "I just want to bubble a context-rich message to the user and exit."

The mistake to avoid: using `anyhow` deep inside library code. That throws away the structure the rest of the system needs to handle errors precisely.

### Conversion plumbing

`thiserror`'s `#[from]` lets you compose errors without manual `From` impls:

```rust
#[derive(Debug, thiserror::Error)]
pub enum LoadError {
    #[error("io: {0}")]
    Io(#[from] std::io::Error),
    #[error("parse: {0}")]
    Parse(#[from] ParseConfigError),
}
```

Now `?` can convert `std::io::Error` or `ParseConfigError` into `LoadError` without ceremony.

### Adding context

When converting between error types, add context the caller actually needs:

```rust
fn load(path: &Path) -> Result<Config, LoadError> {
    let bytes = std::fs::read(path)?;             // Io variant
    let config = parse(&bytes)?;                   // Parse variant
    Ok(config)
}
```

If the bare `std::io::Error` doesn't tell the caller *which path failed*, wrap it with `.map_err(|e| LoadError::IoAt(path.to_owned(), e))` or attach context via `anyhow::Context` (only at the application boundary).

### Variant naming

- Name variants for the **failure category**, not the underlying cause. `InvalidPort` is more useful than `IntegerError`.
- Keep variant payloads small and structural: the offending field name, the bad value, the path. Resist the urge to embed full backtraces (that's `anyhow`'s job).
- Reserve `Other(String)` as a last resort, not a first reach.

### Boundary, repeated

Each crate boundary is its own opportunity for typed errors. An adapter crate's public functions return its own typed `Error` enum — they do *not* leak the underlying library's error types. A core domain crate's public functions return a domain-specific error. The application binary orchestrates these and finally collapses everything to `anyhow::Error` for the user.

*Source: [[Rust Practices/Error Handling/Domain Errors at Boundaries]]*

---

## Related bundles

- [[Bundles/API-Design]] — error types are part of the public API; design them with the API in mind
- [[Bundles/Module-Design]] — module boundaries are where typed errors live
- [[Bundles/Code-Review]] — what to check for when error-handling code lands in a diff
