# Chapter 5: Error Handling

Rust does not use exceptions for ordinary recoverable failures. Failure is a **return
value**, which makes error handling part of the data model rather than an invisible
runtime mechanism. This chapter begins with `Result` and `?`, then moves to error types
that remain useful during production incidents and the common roles of `thiserror` and
`anyhow`.

## 5.1 Two kinds of failure, two tools

Rust distinguishes between broken assumptions and failures that callers may reasonably
handle. A **panic** usually represents an internal invariant violation or programming
error. Depending on the build configuration, it may unwind the current thread or abort
the process. Out-of-bounds indexing is one example.

Conditions such as a missing file, a network timeout, malformed input, or a version
conflict are expected parts of many programs. They are normally represented as typed
return values. A useful question is: _could the caller reasonably respond to this
condition?_ If so, return it as an error value rather than panicking.

> **If you come from Go:** `Result` serves a role similar to a `(value, err)` return, while
> the type system and the `?` operator provide standard handling and propagation. **If you
> come from Java, Python, or JavaScript:** recoverable failure is visible in the return type
> rather than being communicated through exceptions.

## 5.2 `Result<T, E>`: failure you can hold

`Result` is an ordinary generic enum from the standard library:

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

Everything chapter 4 taught about enums applies: exhaustive matching, combinators
(`.map`, `.unwrap_or_else`, `.ok()` to convert into an `Option`, and `opt.ok_or(err)`
the other way). `.unwrap()` / `.expect("why this can't fail")` exist here too, with the
same policy: fine in tests and examples, a smell in libraries.

## 5.3 `?`: propagation without the pyramid

The `?` operator provides a concise form of early error propagation. The following two
functions have the same behavior:

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
`Err`, it returns early. During that return, it can convert the error into the
function's error type when a suitable `From` implementation exists.

The happy path therefore remains linear, while every possible propagation point stays
visible in the source. Errors can travel upward through several layers, but each hop is
explicit and typed.

Note: `?` works inside any function that returns `Result` (or `Option`) — including
`main`, which may be declared `fn main() -> Result<(), anyhow::Error>` so scripts get
`?` for free.

## 5.4 Designing the error type, three drafts

Because errors are values, their structure is part of the program's design. The
following example develops an error type in three stages.

**Draft 1 — a string error:** `Result<Doc, String>`. This is easy to create, but callers
cannot reliably branch on prose and can usually do little more than display or log it.

**Draft 2 — name the cases.** An enum gives each failure a stable identity:

```rust
enum DocumentError {
    NotFound,
    PermissionDenied,
    VersionConflict,
}
```

Callers can match now. But put yourself at the _receiving_ end of `VersionConflict`
(optimistic locking: someone saved before you). What do you tell the user? Which version
exists now? Who changed it? The error does not contain that context, so the handler may
need another query and the information may no longer be available.

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

A practical default is one error enum per module or service, expressed at that layer's
level of abstraction. One application-wide enum often becomes too broad, while a
separate enum for every function usually adds unnecessary ceremony.

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

Each `#[error("…")]` is the human-readable message with the variant's fields
interpolated. The crate derives the standard error traits while leaving the variants and
their meaning under your control.

## 5.6 Wrapping causes: preserve the error chain

The last variant answers the layered-system question: _what do I do with errors from the
layer below me?_ A common but lossy approach is to convert the lower-level error into a
string:

```rust
#[error("storage failure: {0}")]
Store(String), // The original error type and source chain are lost.
```

Converting the underlying error to a `String` discards useful structure. The original
**type** is no longer available for matching, and the **source chain** used by error
reporters is also lost:

```text
storage failure
├─ caused by: connection reset by peer
└─ caused by: os error 104
```

Store the original error instead:

```rust
#[error("storage failure")]
Store(#[from] sqlx::Error),
```

`#[from]` stores the original error without flattening it and preserves the source
chain. It also generates the `From` conversion that allows `?` to cross the layer
boundary automatically.

Treat the cause chain as diagnostic evidence: preserve the original error rather than
paraphrasing it into a string. When automatic conversion is not appropriate, mark the
underlying field with `#[source]` to preserve the same relationship.

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

When a new variant is added, this exhaustive match identifies the boundary mapping that
still needs a status code.

## 5.7 `anyhow`: for the edges

One ecosystem convention separates typed internal errors from application-level
reporting. When another piece of code consumes an error, it usually needs a concrete
enum so it can branch on variants. When the final consumer is a human reading CLI output
or a top-level log, `anyhow` offers a flexible error type with convenient context and
little ceremony:

```rust
use anyhow::Context;

fn main() -> anyhow::Result<()> {
    let config = load_config().context("while loading configuration")?;
    let database = connect(&config.db_url).context("while connecting to database")?;

    run(config, database)
}
```

A common ecosystem convention is to use **`thiserror` for structured library and domain
errors, and `anyhow` at application boundaries where a human will consume the report.**

> **Common mistake:** exposing `anyhow::Error` from a library API whose callers need to match
> specific cases, or defining a large custom error hierarchy for a small application script
> that only needs contextual reporting.

## Summary

Failure is data. It appears in signatures through `Result`, propagates explicitly
through `?`, and is modeled with variants that carry the information handlers need.
Preserve underlying causes with `#[from]` rather than reducing them to strings, and map
domain errors once at the outer boundary.

Use `thiserror` where code needs structured errors and allow `anyhow` at human-facing
application edges. The next chapter introduces traits, the mechanism that carries these
contracts across abstraction boundaries.
