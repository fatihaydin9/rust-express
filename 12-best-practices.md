# Chapter 12: Best Practices

This chapter consolidates the conventions that help a Rust codebase remain healthy over time. Most sections compress ideas from earlier chapters into rules that can be reviewed or automated; the rest comes from recurring production experience.

Read it once now, return to it while building, and encode as much as possible in CI so the checks do not depend on memory.

## 12.1 API design

### Accept borrowed inputs and return owned values

Function parameters should usually accept the most general borrowed form that fits the operation. Prefer `&str` over `&String` and `&[T]` over `&Vec<T>`. Callers can then pass either owned collections or borrowed views.

Return values are usually easier to evolve when they are owned, such as `String` or `Vec<T>`. Returning a reference ties the output lifetime to internal data and can make later refactoring more difficult.

### Use `impl Into<_>` selectively

A signature such as the following lets callers provide either `&str` or `String`:

```rust
fn set_name(&mut self, name: impl Into<String>) {
    self.name = name.into();
}
```

This is convenient on frequently used public APIs, but it can add noise and obscure types when applied everywhere. Use it when the ergonomic benefit is clear.

### Make invalid states unconstructible

Use constructors that validate invariants and return `Result` for domain values. Newtypes and private fields prevent callers from bypassing those checks. For types with many optional settings, combine sensible defaults with a builder.

The goal is twofold: invalid values should be difficult or impossible to create, while valid values should remain easy to express.

### Leave public APIs room to grow

Apply `#[non_exhaustive]` to public enums and configuration-like structs when downstream code should not assume the set of variants or fields is permanently closed.

This requires downstream matches to include a fallback arm and prevents direct construction of some public structs. The small inconvenience can preserve compatibility when a later release adds a variant or field.

### Treat semantic versioning as an API contract

Changing a public signature, removing a derive, or tightening a trait bound can be a breaking change. Use `cargo semver-checks` before publishing a library release to detect many of these changes mechanically.

## 12.2 Documentation

Every public item should have `///` documentation whose first line is a complete, standalone summary. That line is what documentation search results and IDE tooltips often display.

For fallible or panicking functions, use the conventional sections:

- `# Errors` explains when the function returns an error.
- `# Panics` explains conditions that can trigger a panic.
- `# Examples` contains runnable code blocks.

Documentation examples are compiled and executed by `cargo test`, which prevents them from silently becoming stale. Library crates should also consider `#![deny(missing_docs)]` in `lib.rs` so undocumented public items fail the build.

Treat docs.rs as part of the interface. Write documentation for a reader who has not opened the implementation.

## 12.3 Error and panic discipline

Library and domain code should return typed errors, commonly defined with `thiserror`. Keep `anyhow` at application boundaries where flexible context and error aggregation are more useful than a stable public error type.

Preserve underlying causes with `#[from]` or `#[source]`; do not reduce them to formatted strings. Structured error chains retain information for both diagnostics and callers.

Reserve `.unwrap()` and `.expect()` for tests, examples, and cases that are genuinely impossible after an invariant has been established. Even then, prefer an explanatory message:

```rust
let port = value
    .parse::<u16>()
    .expect("validated configuration must contain a valid TCP port");
```

A library that panics because of ordinary bad input has an API bug. Panics should represent broken internal assumptions, not expected failure modes.

## 12.4 Performance

### Measure optimized builds

Measure before optimizing, and measure with `--release`. Debug builds may be tens of times slower, so performance impressions formed there are unreliable. Use Criterion for benchmarks and tools such as `cargo flamegraph` for profiling.

### Apply low-cost habits consistently

Prefer iterator pipelines over unnecessary intermediate collections. Rather than calling `collect()` only to iterate again, pass the iterator directly into the consumer when possible.

Preallocate when the final size is reasonably known:

```rust
let mut bytes = Vec::with_capacity(expected_size);
let mut output = String::with_capacity(expected_length);
```

Wrap file and socket I/O with `BufReader` or `BufWriter` when performing many small operations. Avoid repeated `format!` calls and allocations in proven hot loops.

### Keep contextual optimizations contextual

A `.clone()` is acceptable until measurement shows that it matters. Focus on repeated cloning of large buffers in loops rather than isolated `String` clones in request handlers.

Use `Cow<'_, str>` when a value is usually borrowed but occasionally needs ownership. The per-call virtual dispatch cost of `Box<dyn Trait>` is generally insignificant in I/O-heavy code and deserves attention mainly inside very hot per-element loops.

> **Common mistake:** Do not micro-optimize allocations while every database request performs an unindexed `SELECT *`. Rust already makes ordinary computation fast; production bottlenecks are frequently I/O. Timed `tracing` spans will show where the time actually goes.

## 12.5 Security and robustness

### Protect the dependency supply chain

Run `cargo audit` in CI to detect known vulnerable dependencies. Use `cargo deny` to enforce license, advisory, and dependency policies. Before adopting a dependency, inspect its transitive graph with `cargo tree`.

### Isolate unsafe code

Application crates should normally contain no unsafe code. Use `#![forbid(unsafe_code)]` where that policy is appropriate.

When unsafe code is genuinely necessary, hide it behind a safe API and document the invariant immediately above the block:

```rust
// SAFETY: `ptr` is non-null, aligned, and valid for `len` initialized bytes.
unsafe {
    // ...
}
```

The comment should explain why the operation is sound, not merely restate what the block does.

### Validate trust boundaries through types

Everything crossing a trust boundary should be parsed into invariant-preserving types. Use strict Serde options such as `deny_unknown_fields` when silently accepting unknown input would be dangerous.

### Handle secrets deliberately

Do not derive `Debug` for a type that contains secrets unless the output is safely redacted. Clear sensitive values with `zeroize` when appropriate, use constant-time comparison functions such as those in `subtle`, and hash passwords with a password-hashing algorithm such as Argon2 rather than a general-purpose hash.

### Contain panics in servers

A panic in one request handler should not terminate the process. Tower middleware such as `CatchPanic` can convert the failure into an HTTP 500 response while `tracing` records the incident. This is containment, not a substitute for fixing the underlying bug.

## 12.6 Dependency hygiene

Every dependency adds compile time, supply-chain exposure, and future upgrade work. Prefer the standard library when it already solves the problem; for example, use `std::sync::LazyLock` instead of adding a crate solely for lazy initialization.

For large dependencies, consider disabling default features and enabling only what the project uses:

```toml
some-crate = { version = "1", default-features = false, features = ["needed-feature"] }
```

Feature flags are additive across the dependency graph. One dependency that enables a broad default can therefore pull in functionality—or even another async runtime—that the application never intended to use.

Centralize shared versions in `[workspace.dependencies]` so the workspace has one source of truth. Upgrade dependencies on a schedule, for example by reviewing `cargo outdated` monthly, rather than allowing years of drift to turn routine maintenance into a migration project.

Applications should commit `Cargo.lock`. Libraries historically omitted it from published packages, although keeping it in the repository is now common and useful for reproducible development and CI.

## 12.7 Concurrency discipline

Do not block the async runtime. Move synchronous or CPU-intensive work to `spawn_blocking` or to a dedicated worker pool.

Never hold a `std::sync` lock guard across `.await`. Keep critical sections small: prepare work outside the lock, acquire it only to inspect or replace shared state, and release it before awaiting anything.

Prefer bounded channels:

```rust
let (tx, rx) = tokio::sync::mpsc::channel(1024);
```

A bounded channel creates backpressure. When the consumer slows down, producers observe latency instead of allowing memory usage to grow without limit.

When a shared `Mutex` reaches several unrelated call sites, consider moving the state into one owning task and communicating through typed messages. Design the shutdown path before the application accumulates many independent task-spawning locations.

## 12.8 Encode the practices in CI

Any rule that a machine can verify should be checked by a machine. A minimal pipeline contains four gates:

```yaml
# .github/workflows/ci.yml
- run: cargo fmt --check
- run: cargo clippy --all-targets -- -D warnings
- run: cargo test --all
- run: cargo audit
```

A more mature pipeline can add `cargo deny check`, `cargo semver-checks` for library releases, and a `--release` build so optimization-dependent code paths are compiled regularly.

Pin the toolchain with `rust-toolchain.toml` so CI and developer machines use the same compiler version. The general principle is to move checks earlier: a rule enforced in CI becomes part of the engineering system, while a rule documented only in a wiki remains optional in practice.

## 12.9 Rust-specific code-review signals

In addition to ordinary review concerns, several Rust patterns are worth scanning for:

- A `pub` item that only needs `pub(crate)`, which may indicate accidental API growth
- A `type X = Y` alias being used where a newtype is needed for actual type safety
- An error variant that converts its cause to `format!` and loses the error chain
- A trait defined by the provider rather than by the consumer that needs the behavior
- Clusters of `.clone()` calls that may reveal an unclear ownership model
- A lock guard held across `.await`
- Business logic inside an HTTP handler
- An `unsafe` block without a `SAFETY:` explanation
- An unbounded channel without a deliberate capacity strategy

Once these patterns have names, most take only seconds to notice during review.

## 12.10 One-page checklist

| Area | Rule of thumb |
|---|---|
| Parameters and returns | Accept `&str` or `&[T]`; usually return owned values |
| Invariants | Parse into validated types; use newtypes for identifiers |
| Public enums and structs | Use `#[non_exhaustive]` unless the shape is intentionally closed |
| Documentation | Begin with a summary; add `# Errors`, `# Panics`, and compiled examples |
| Errors | Use `thiserror` in libraries and domains; use `anyhow` at application edges |
| Panics | Reserve them for tests and broken invariants; explain every `expect` |
| Performance | Measure `--release`; avoid unnecessary collections; buffer small I/O |
| Security | Audit dependencies, document unsafe invariants, and handle secrets explicitly |
| Dependencies | Keep them few, trim features, centralize versions, and upgrade regularly |
| Concurrency | Use bounded channels; avoid blocking and locks across `.await` |
| CI | Run fmt, clippy with warnings denied, tests, and vulnerability checks |

## Closing note

Most of these rules restate one idea from chapters 3–8: move guarantees earlier. Encode assumptions in types, isolate dependencies behind crate boundaries, and automate checks in CI. When those guarantees become part of the structure, the language and toolchain carry much of the discipline for you.
