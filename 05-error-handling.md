# Chapter 5: Error Handling

Rust does not use exceptions for ordinary recoverable failures. Failure is a **return
value**, which makes error handling part of the data model rather than an invisible runtime
mechanism. This chapter begins with `Result` and `?`, then moves to error types that remain
useful during production incidents and the common roles of `thiserror` and `anyhow`.

## 5.1 Two kinds of failure, two tools

**Unrecoverable failures** represent bugs or violated internal invariants for which
continuing would be unsafe. These use `panic!`, which stops the current thread and normally
unwinds its stack. Out-of-bounds indexing is one example. Think of panic as an assertion
failure, not a substitute for ordinary error handling.

**Recoverable** failures — file missing, network timeout, malformed input, version conflict
— are _expected events_, and expected events are **data**: returned, typed, handled.

The dividing question: _could a caller reasonably want to react to this?_ If yes, it's a
value. Almost everything interesting is a value.

> **If you come from Go:** `Result` is your `(value, err)` pair, formalized — with the
> improvements that ignoring it is a compiler warning, and propagation is one character
> instead of three lines. **If you come from Java/Python/JS:** signatures now _show_
> fallibility, and nothing jumps invisibly over your stack frames.

## 5.2 `Result<T, E>`: failure you can hold

An ordinary enum from the standard library — no magic:

```rust
enum Result<T, E> {
    Ok(T),  // Success, carrying the value.
    Err(E), // Failure, carrying the error.
}
```

```rust
fn read_config(path: &str) -> Result<Config, ConfigError> {
    /* ... */
}

match read_config("app.toml") {
    Ok(config) => start(config),
    Err(error) => eprintln!("could not load config: {error}"),
}
```

Everything chapter 4 taught about enums applies: exhaustive matching, combinators (`.map`,
`.unwrap_or_else`, `.ok()` to convert into an `Option`, and `opt.ok_or(err)` the other way).
`.unwrap()` / `.expect("why this can't fail")` exist here too, with the same policy: fine in
tests and examples, a smell in libraries.

## 5.3 `?`: propagation without the pyramid

The operator that makes `Result` livable. These two functions are identical:

```rust
// Expanded form.
fn load_expanded(path: &str) -> Result<Config, ConfigError> {
    let text = match std::fs::read_to_string(path) {
        Ok(text) => text,
        Err(error) => return Err(error.into()),
    };

    parse(&text)
}

// Idiomatic form with `?`.
fn load(path: &str) -> Result<Config, ConfigError> {
    let text = std::fs::read_to_string(path)?;
    parse(&text)
}
```

The `?` operator performs three steps. On `Ok`, it extracts the value and continues. On
`Err`, it returns early. During that return, it can convert the error into the function's
error type when a suitable `From` implementation exists.

The happy path therefore remains linear, while every possible propagation point stays
visible in the source. Errors can travel upward through several layers, but each hop is
explicit and typed.

Note: `?` works inside any function that returns `Result` (or `Option`) — including `main`,
which may be declared `fn main() -> Result<(), anyhow::Error>` so scripts get `?` for free.

## 5.4 Designing the error type, three drafts

Errors are values, so _you design them_, and the quality shows in production logs. Watch one
evolve.

**Draft 1 — the lazy baseline:** `Result<Doc, String>`. Compiles; useless. Callers can't
branch on prose, only print it. You've rebuilt untyped exceptions.

**Draft 2 — name the cases.** It's an enum, obviously:

```rust
enum DocumentError {
    NotFound,
    PermissionDenied,
    VersionConflict,
}
```

Callers can match now. But put yourself at the _receiving_ end of `VersionConflict`
(optimistic locking: someone saved before you). What do you tell the user? Which version
exists now? Who changed it? The error knows nothing; you must re-query — if you still can.

**Draft 3 — the principle: each variant carries what its handler needs to act.** The
information was in hand at the failure site. Don't discard it there:

```rust
pub enum DocumentError {
    NotFound(DocumentId),
    PermissionDenied { doc: DocumentId, user: UserId },
    VersionConflict {
        expected: i32,
        actual: i32,
        modified_by: UserId,
        modified_at: DateTime<Utc>,
    },
}
```

The richer variant gives each handler enough information to act. A UI can explain who
changed the document and when, a synchronization client can choose between retrying and
merging, and a log entry can describe the incident without another query. Chapter 4's
newtype IDs also remain intact inside the error.

This habit—placing actionable context in each variant—is one of the clearest differences
between weak and strong error designs.

Granularity advice from practice: one error enum **per module or service**, at that layer's
level of abstraction — not one giant enum for the whole application (whose matches rot into
`_ => unreachable!()`), and not one enum per function (ceremony).

## 5.5 `thiserror`: the boilerplate, derived

Errors should print nicely (`Display`) and integrate with the ecosystem
(`std::error::Error`). Writing that by hand is boilerplate; the standard tool is the
`thiserror` crate:

```rust
#[derive(Debug, thiserror::Error)]
pub enum DocumentError {
    #[error("document {0} not found")]
    NotFound(DocumentId),

    #[error(
        "version conflict: expected v{expected}, found v{actual} \
         (by {modified_by} at {modified_at})"
    )]
    VersionConflict {
        expected: i32,
        actual: i32,
        modified_by: UserId,
        modified_at: DateTime<Utc>,
    },

    #[error("storage failure")]
    Store(#[from] sqlx::Error),
}
```

Each `#[error("…")]` is the human-readable message with the variant's fields interpolated.
That's the whole crate: your enum, your design — minus the ceremony.

## 5.6 Wrapping causes: keep the chain, never flatten it

The last variant answers the layered-system question: _what do I do with errors from the
layer below me?_ There is a tempting wrong answer, common in the wild:

```rust
#[error("storage failure: {0}")]
Store(String), // The original error type and source chain are lost.
```

Converting the underlying error to a `String` amputates it: its **type** is gone (no more
matching "was that a unique-constraint violation?"), and its **source chain** is gone — the
linked list of causes that error reporters walk to print:

```text
storage failure
├─ caused by: connection reset by peer
└─ caused by: os error 104
```

The right answer costs nothing:

```rust
#[error("storage failure")]
Store(#[from] sqlx::Error),
```

`#[from]` stores the original error without flattening it and preserves the source chain. It
also generates the `From` conversion that allows `?` to cross the layer boundary
automatically.

Treat the cause chain as evidence: **wrap it rather than paraphrasing it.** When automatic
conversion is not appropriate, mark the underlying field with `#[source]` to preserve the
same relationship.

At the outermost boundary, map errors _once_, exhaustively — for example, domain error →
HTTP status:

```rust
match e {
    DocumentError::NotFound(_)             => 404,
    DocumentError::PermissionDenied { .. } => 403,
    DocumentError::VersionConflict { .. }  => 409,
    DocumentError::Store(_)                => 500,
}
```

Add a variant next month, and this match refuses to compile until you've chosen its status
code — chapter 4's exhaustiveness doing maintenance for you.

## 5.7 `anyhow`: for the edges

One ecosystem convention separates typed internal errors from application-level reporting.
When another piece of code consumes an error, it usually needs a concrete enum so it can
branch on variants. When the final consumer is a human reading CLI output or a top-level
log, `anyhow` offers a flexible error type with convenient context and little ceremony:

```rust
use anyhow::Context;

fn main() -> anyhow::Result<()> {
    let config = load_config().context("while loading configuration")?;
    let database = connect(&config.db_url).context("while connecting to database")?;

    run(config, database)
}
```

Rule of thumb, near-universal in the ecosystem: **`thiserror` for libraries and domain
logic; `anyhow` allowed at application edges.**

> **Common mistake:** the directions reversed — `anyhow` leaking out of a library's public
> API (callers can no longer branch), or thirty-variant `thiserror` ceremony inside a
> ten-line script.

## Summary

Failure is data. It appears in signatures through `Result`, propagates explicitly through
`?`, and is modeled with variants that carry the information handlers need. Preserve
underlying causes with `#[from]` rather than reducing them to strings, and map domain errors
once at the outer boundary.

Use `thiserror` where code needs structured errors and allow `anyhow` at human-facing
application edges. The next chapter introduces traits, the mechanism that carries these
contracts across abstraction boundaries.
