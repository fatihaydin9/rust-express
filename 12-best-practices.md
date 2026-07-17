# Best practices

This chapter consolidates best practices and design conventions to help you maintain a healthy, robust, and secure Rust codebase. These guidelines are compiled from previous chapters and real-world production experience.

To ensure consistency, you should automate as many of these guidelines as possible in your continuous integration (CI) pipeline.

By the end of this chapter, you will understand:
* Standard API design guidelines.
* Documentation formatting conventions.
* Robust error and panic handling policies.
* Pragmatic performance optimization habits.
* Dependency supply-chain security and safety rules.
* Multi-threading and concurrency guidelines.
* Automated CI pipeline checks and code-review signals.

## API design

The following guidelines help you create ergonomic, readable, and maintainable public APIs:

### 1. Accept borrowed parameters and return owned values
* **Function parameters**: Accept the most general borrowed type possible. For example, prefer **`&str`** over `&String`, and prefer **`&[T]`** (slices) over `&Vec<T>`. This allows callers to pass either owned variables or borrowed views without needing to allocate new memory.
* **Return values**: Return owned values (such as `String` or `Vec<T>`). Returning references binds the output lifetime to your internal data structure lifetimes, which can make future code modifications difficult.

### 2. Use `impl Into<T>` selectively
Using `impl Into<T>` in parameter definitions is highly convenient for callers because it allows them to pass different compatible types directly:

```rust
fn set_name(&mut self, name: impl Into<String>) {
    // The caller can pass either a String or a &str reference.
    self.name = name.into();
}
```

However, overusing this pattern can obscure parameter types in your documentation. Apply this pattern only to frequently used public constructor or configuration APIs where the ergonomic benefit is clear.

### 3. Prevent invalid states
Validate invariants in your constructors and return a `Result` type. Use the newtype pattern and private fields to prevent callers from creating or modifying values without validation. For types with many parameters, combine reasonable default values with the builder pattern.

### 4. Support API evolution with `#[non_exhaustive]`
Add the `#[non_exhaustive]` attribute to public enums and configuration structs if you plan to add fields or variants in future releases. This forces downstream users to include a fallback arm in their `match` expressions and prevents them from instantiating your structs directly, allowing you to update your types without breaking compatibility.

### 5. Follow semantic versioning
Changing a public function signature, removing a derived trait, or adding a trait bound are breaking changes. Use tools like `cargo semver-checks` in your build pipeline to verify compliance automatically.

## Documentation conventions

Document all public items using triple-slash (`///`) comments. Your documentation should begin with a single, standalone summary line because IDE tooltips and documentation search engines often display only this first line.

For functions that can fail or panic, include the following standard markdown headers:
* **`# Errors`**: Explains the scenarios under which the function returns an `Err` variant.
* **`# Panics`**: Explains the conditions under which the function triggers an unrecoverable panic.
* **`# Examples`**: Contains runnable code examples.

Cargo compiles and executes code inside your `# Examples` sections automatically when you run `cargo test`. This guarantees that your documentation examples never become out-of-date.

For library crates, add `#![deny(missing_docs)]` to your `lib.rs` file to turn missing public documentation into compile-time build errors.

## Error and panic handling policies

### Custom error types
Use the `thiserror` crate to define structured, typed error enums in your libraries and domain layers. Use the `anyhow` crate to aggregate error context at your application's outermost entry points (such as CLI drivers or background services).

### Preserve underlying causes
Store underlying errors inside your variants using the `#[from]` or `#[source]` attributes instead of formatting them into raw strings. This preserves the complete cause chain for diagnostics and error reporting.

### Reserve panics for logic errors
Only use `.unwrap()` or `.expect()` in unit tests, examples, or when an established invariant makes failure impossible. When using `.expect()`, always provide a descriptive message explaining *why* the failure is impossible:

```rust
let port = value
    .parse::<u16>()
    .expect("validated configuration must contain a valid TCP port");
```

Never use panics to handle expected runtime failures (such as a missing file or malformed network input). These scenarios should return a `Result` type.

## Performance guidelines

### 1. Benchmark only optimized release builds
Never draw performance conclusions or profile your application using debug builds, which compile without optimizations and can run tens of times slower. Always run performance measurements on builds compiled with the `--release` flag. Use the `criterion` crate for statistical benchmarking and `cargo flamegraph` for profiling.

### 2. Avoid redundant allocations
* **Pass iterators directly**: Avoid collecting items into intermediate vectors only to iterate over them again. Pass the iterator directly to consumers where possible.
* **Preallocate collection capacity**: If you know the final size of a vector or string, allocate its capacity upfront using `with_capacity` to prevent repeated resizing overhead:
  ```rust
  let mut bytes = Vec::with_capacity(expected_size);
  let mut output = String::with_capacity(expected_length);
  ```
* **Buffer input/output**: Wrap file and socket network operations in `BufReader` or `BufWriter` to group small, frequent write or read operations. Avoid allocating memory inside hot loops.

### 3. Optimize on context
* **Clone selectively**: Do not pre-emptively optimize isolated `.clone()` calls on small values. Focus instead on cloning large buffers inside loops.
* **Use copy-on-write**: Use `Cow<'_, str>` for string fields that are usually read-only but occasionally require modification.
* **Avoid unnecessary dynamic dispatch**: The runtime cost of trait objects (`Box<dyn Trait>`) is negligible in I/O-bound applications. Keep dynamic dispatch unless measurements indicate a hot loop bottleneck.

## Security and safety

### 1. Audit your dependencies
Run `cargo audit` in your CI pipeline to detect dependencies with known vulnerabilities. Use `cargo deny` to enforce licensing rules and check for advisory warnings. Inspect your dependency footprint using `cargo tree`.

### 2. Isolate unsafe code
Avoid writing `unsafe` code in standard applications. Where possible, add `#![forbid(unsafe_code)]` to your library root file. 

If you must write `unsafe` code, hide it behind a safe API boundary and add a `// SAFETY:` comment immediately above the block explaining why the operations are safe:

```rust
// SAFETY: `ptr` is non-null, aligned, and valid for `len` initialized bytes.
unsafe {
    // ...
}
```

The comment must explain why the operation cannot violate memory safety, not just restate what the code does.

### 3. Handle secrets securely
* **Redact debugging**: Do not derive `Debug` on types containing sensitive data (such as passwords or API tokens) unless the output is explicitly redacted.
* **Wipe memory**: Use the `zeroize` crate to securely clear sensitive values from memory when variables go out of scope.
* **Constant-time comparisons**: Use constant-time comparison crates (such as `subtle`) for checking credentials to prevent timing-attack analysis.
* **Use standard hashing**: Hash user passwords using dedicated hashing algorithms like Argon2, never general-purpose hashing functions.

### 4. Contain server panics
Configure your server to isolate panics within request handlers instead of terminating the entire process. Web frameworks like Axum support panic-catching middleware (such as `CatchPanic`) that converts a handler panic into an HTTP 500 response and logs the event.

## Dependency management

Each external dependency increases compilation times, security footprint, and future maintenance overhead. Use the standard library where available (for example, use `std::sync::LazyLock` for lazy initialization).

When adding large crates, disable default features and enable only the specific features your application requires. This prevents pulling unused transitive dependencies into your project:

```toml
some-crate = { version = "1", default-features = false, features = ["needed-feature"] }
```

### Workspace dependency policies
* **Centralize versions**: Declare all workspace dependencies in your root `Cargo.toml` file under the `[workspace.dependencies]` table.
* **Update regularly**: Run `cargo outdated` monthly to detect and upgrade outdated packages, avoiding massive migration efforts.
* **Commit lockfiles**: Always commit your `Cargo.lock` file for binary applications to ensure reproducible builds.

## Concurrency guidelines

* **Do not block runtime threads**: Execute long-running synchronous or CPU-heavy work in a dedicated worker thread using `tokio::task::spawn_blocking` or separate thread pools.
* **Do not hold synchronous locks across await boundaries**: Never hold a standard library `std::sync::MutexGuard` lock across an `.await` call. Keep critical sections short, and release locks as early as possible. Use `tokio::sync::Mutex` if a lock must span across `.await` points.
* **Prefer bounded channels**: Always use bounded channels (for example, `tokio::sync::mpsc::channel(1024)`) to create backpressure. Bounded channels prevent memory usage from growing without limit if a consumer slows down.
* **Transition to actors**: If a shared `Mutex` starts to cause code complexity across multiple files, move the state into a single owning task and coordinate updates exclusively using message-passing channels.

## Automate checks in continuous integration

To guarantee code health, enforce checks in your automated CI pipeline rather than relying on manual code reviews.

A standard CI pipeline should run the following four commands on every commit:
```yaml
# .github/workflows/ci.yml
- run: cargo fmt --check
- run: cargo clippy --all-targets -- -D warnings
- run: cargo test --all
- run: cargo audit
```

Pin your compiler version using a `rust-toolchain.toml` file at your repository root to ensure that all developers and CI workers compile your code with the exact same compiler version.

## Code review checklist

When reviewing Rust pull requests, scan for the following common patterns:

*   **Excessive visibility**: Public (`pub`) items that should be restricted to internal crate scope (`pub(crate)`).
*   **Missing type safety**: Type aliases used instead of the newtype pattern for unique identifiers or validated values.
*   **Lossy error chains**: Error variants that format underlying error causes into strings using `format!`, destroying the diagnostics chain.
*   **Incorrect trait ownership**: Traits defined by the provider crate instead of being declared next to the consumer service.
*   **Unclear ownership**: Clusters of `.clone()` calls that indicate a poorly designed data ownership structure.
*   **Asynchronous blockages**: Synchronous mutex locks held across `.await` boundaries, or blocking synchronous function calls executed inside async handlers.
*   **Missing safety annotations**: `unsafe` blocks that lack a preceding `// SAFETY:` comment explaining why the memory operation is sound.
*   **Unbounded channels**: Asynchronous channels created without a deliberate capacity limit.

## Best practices checklist

The following table summarizes the core rules of thumb for maintaining Rust codebases:

| Area | Rule of thumb |
| :--- | :--- |
| **Parameters and returns** | Accept borrowed slices (`&str` or `&[T]`); return owned values (`String`, `Vec<T>`). |
| **Invariants** | Validate primitives on construction; wrap identifiers in custom newtypes. |
| **Public API types** | Use the `#[non_exhaustive]` attribute unless the type shape is permanently closed. |
| **Documentation** | Start with a single summary line; add `# Errors`, `# Panics`, and compiled examples. |
| **Error handling** | Use `thiserror` for structured library errors; use `anyhow` for high-level application reporting. |
| **Panics** | Reserve panics for tests and broken invariants; document `.expect` with a clear explanation. |
| **Performance** | Measure only `--release` builds; avoid unnecessary allocations; buffer small I/O. |
| **Security** | Run vulnerability audits; document unsafe blocks; use constant-time operations for secrets. |
| **Dependencies** | Keep dependencies minimal; disable unused features; centralize workspace versions. |
| **Concurrency** | Use bounded channels to enforce backpressure; do not block async runtime threads. |
| **CI automation** | Verify formatting, deny clippy warnings, run all tests, and check security advisories on every commit. |

## Summary

The fundamental theme of Rust architecture is **moving guarantees earlier in the development lifecycle**:
* Representing business assumptions as compiler-checked types.
* Enforcing architecture boundaries using crate dependencies.
* Verifying code cleanliness and security automatically inside your CI pipeline.

By structuring your systems to let the compiler and tools enforce these boundaries, you write highly robust and maintainable applications.
