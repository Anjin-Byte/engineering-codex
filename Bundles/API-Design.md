---
title: API Design Bundle
tags: [bundle, api-design]
summary: Bundle for designing a public API: predictability over cleverness, validation pushed into construction, and documentation made executable.
source_trigger: "Designing a public API"
bundles: [Predictable APIs, Checked Constructors and Builders, Documentation as Truth]
---

*The through-line: a public API is predictable in shape, validated at the moment of construction, and documented in code that the test runner verifies.*

# API Design Bundle

Read this bundle when designing a public API. It bundles the three disciplines that decide whether the surface a caller sees is trustworthy: naming and behavior must be predictable so callers don't need telepathy; validation must happen at construction so downstream code can trust the type; and the documentation must be executable so it cannot drift from the contract.

---

## Predictable APIs

*Predictable beats clever in public APIs; consumers should be able to guess the shape of the next method correctly.*

Optimize public APIs for **predictability**, not cleverness. Aligns with the Rust API Guidelines on consistency, naming, and dependability.

### What predictable means

A predictable API is one where a competent caller can guess the right thing without reading the implementation.

Concretely:

- **Names imply behavior.** `len()` is cheap. `count_unique()` might not be. If a method called `id()` does a database query, the name is lying.
- **Ownership behavior does not surprise.** A function called `set_name(&mut self, name: String)` taking ownership is fine; if it instead clones internally and the caller didn't expect that, performance goes sideways quietly.
- **Constructors and conversions are unsurprising.** `From<A> for B` should be lossless and obvious. `TryFrom` when failure is possible. `parse` for string-shaped input. Don't invent novel conversion verbs.
- **Methods don't hide expensive work.** If a getter triggers I/O or allocates, the name should hint at it (`fetch_*`, `load_*`, `compute_*`).
- **Defaults are explicit when ambiguity matters.** `Default::default()` is fine for true zero values. For domain types, prefer named constructors so the default isn't invisible.

### Why this is a stability concern

Stability is partly technical and partly social: callers should not need telepathy. APIs that surprise generate bug reports, defensive wrapper code, and cargo-culted workarounds — all of which become load-bearing.

A surprising API is also harder to refactor, because callers have built mental models around the surprises. "Fixing" the surprise becomes a breaking change.

### Naming heuristics

- Use the verb form the standard library uses: `iter`, `into_iter`, `as_str`, `to_string`, `clone`, `with_capacity`. Match those patterns.
- Reserve `unwrap_*`, `expect_*` for explicit panic-on-failure conversions.
- `is_*` returns `bool`. `has_*` returns `bool`. Don't use them for fallible getters.
- Avoid abbreviations unless they're domain-standard (`buf`, `ctx`, `cfg` are fine; `prc`, `mng`, `xfr` are not).

*Source: [[Rust Practices/Foundational/Predictable APIs]]*

---

## Checked Constructors and Builders

*Validate at birth, not on every access; checked constructors and builders make a constructed value trustworthy by definition.*

Prefer **explicit construction** over post-init mutation. Push validation into birth, not into every later access.

### The shift

```rust
// Worse — every field accessible, every combination possible,
// invariants checked nowhere
let mut config = Config::default();
config.port = 8080;
config.fast = true;
config.other_flag = false;

// Better — invariants enforced by build(), invalid combos rejected once
let config = Config::builder()
    .port(Port::new(8080)?)
    .mode(Mode::Fast)
    .build()?;
```

### Why this is more stable

- Invariants are checked **once**, at construction. Downstream code trusts the type.
- The set of legal configurations is enumerated by the builder API. Combinations the builder can't produce are de facto banned.
- Adding a required field is a compile-time refactor (callers fail to build), not a "good luck finding all the call sites" hunt.
- The constructed type can have all-private fields, eliminating "someone mutated this in the middle of an operation" bugs.

### When a constructor is enough

For 1–3 fields with simple validation, a plain `new` is fine:

```rust
impl Config {
    pub fn new(port: Port, mode: Mode) -> Self { /* ... */ }
}
```

Reach for a builder when:
- there are many optional parameters
- some parameters depend on others
- defaults are non-trivial
- adding a field shouldn't break callers

### Builder shape

Two common Rust builder styles:

**Consuming (movable) builder** — typical:
```rust
pub struct ConfigBuilder { /* ... */ }
impl ConfigBuilder {
    pub fn port(mut self, p: Port) -> Self { self.port = Some(p); self }
    pub fn build(self) -> Result<Config, ConfigError> { /* ... */ }
}
```

**Type-state builder** — for required fields the compiler must enforce:
```rust
pub struct ConfigBuilder<HasPort> { /* phantom marker */ }
impl ConfigBuilder<NoPort> {
    pub fn port(self, p: Port) -> ConfigBuilder<HasPort> { /* ... */ }
}
impl ConfigBuilder<HasPort> {
    pub fn build(self) -> Config { /* ... */ }
}
```

The type-state form costs more code but turns "forgot to set port" into a compile error.

### Constructor naming

- `new(...)` — the canonical constructor for the type
- `with_*` — alternative constructors emphasizing a chosen parameter
- `from_*` — conversion-style construction (often pairs with `From`/`TryFrom`)
- `builder()` — returns a builder
- `default()` — zero/identity value (via `Default`); only when "default" is unambiguous

### Anti-pattern

```rust
pub struct Config { pub port: u16, pub fast: bool, pub mode: Mode, /* ... */ }
```

All-public fields look ergonomic but mean every caller can construct any combination, including invalid ones. Save this shape for **plain data records** with no invariants — config DTOs read from disk that you immediately validate into a private-fields version.

*Source: [[Rust Practices/Type-Driven Design/Checked Constructors and Builders]]*

---

## Documentation as Truth

*Doc tests make documentation executable, so prose about invariants and examples becomes a check the compiler runs for you.*

Document **invariants** and **examples** where they live. Because Rust doc examples are compiled and tested, documentation can act as **executable truth** — not decoration.

### What good docs explain

For a public type or function, useful docs cover:

- **What the type guarantees.** "A `Port` is always nonzero." The reader should not need to read the implementation to learn this.
- **What the caller must guarantee.** Preconditions, valid input ranges, expected ordering of method calls.
- **Failure modes.** Which errors are returned when, and what they mean operationally.
- **Examples of intended use.** Short, executable, written at the level of a caller.

This is not the same as describing the implementation. Docs should describe the **contract**, not the body.

### Doc tests are the secret weapon

```rust
/// Loads a [`Config`] from a TOML file.
///
/// # Errors
///
/// Returns [`LoadError::Io`] if the file cannot be read,
/// or [`LoadError::Parse`] if the contents are not valid TOML.
///
/// # Examples
///
/// ```no_run
/// use mycrate::Config;
/// let config = Config::load("app.toml")?;
/// # Ok::<(), mycrate::LoadError>(())
/// ```
pub fn load(path: impl AsRef<Path>) -> Result<Config, LoadError> { /* ... */ }
```

Because the example compiles as part of `cargo test --doc`, it can't drift. Rename `load` and the doc test breaks. Change the error type and the doc test breaks. The docs and the code are *married* instead of "separated but technically still in contact."

### Conventions that make docs more useful

- **Front-load the summary.** First line is a single-sentence description, used in API listings.
- **Use sections.** `# Errors`, `# Panics`, `# Safety`, `# Examples` are conventional and parsed by tools.
- **Link types with `[` `]`.** Rustdoc resolves `[`Type`]` to a clickable link.
- **Document `# Panics` for any function that can panic.** Including for `expect` calls in non-obvious places.
- **Document `# Safety` for every `unsafe fn`.** Non-negotiable. See [[Unsafe Quarantine]].

### Where docs are most valuable

- **Public APIs of adapter crates** — the narrow surface external code consumes. Worth spending time on.
- **[[Newtypes and Domain Types]]** — they encode invariants; the docs should make those invariants legible.
- **Error enums** — what each variant means, when it's returned.
- **Builders and constructors** — required vs optional, validation rules.

Internal helper functions need less doc weight; their callers are nearby and can read the code.

### When to skip docs

- Trivial getters whose name fully describes them (`get`, `len`, `is_empty`).
- Internal `pub(crate)` items used in one place.
- Generated/derived code where the trait's own docs apply.

This is not "skip docs everywhere convenient." It's "don't write empty boilerplate that says what the signature already says."

### CI for docs

Useful additions in CI:

```sh
# Catch broken intra-doc links
RUSTDOCFLAGS="-D warnings" cargo doc --workspace --no-deps

# Doc tests already run via `cargo test --doc` — see [[CI Quality Bar]]
```

These keep documentation honest the same way Clippy keeps code honest.

*Source: [[Rust Practices/Tooling and Quality/Documentation as Truth]]*

---

## Related bundles

- [[Bundles/Module-Design]] — the module shape underneath the API; design them together
- [[Bundles/Error-Handling]] — typed error enums are part of the public API
- [[Bundles/Code-Review]] — what to check when an API change lands in a diff
