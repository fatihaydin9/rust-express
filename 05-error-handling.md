# Error handling

Rust does not use exceptions to handle errors. Instead, Rust represents errors as ordinary values. This makes error handling an explicit part of your application's data model and control flow.

By the end of this chapter, you will understand:
* The difference between unrecoverable panics and recoverable errors.
* How to use the `Result` enum to represent operations that can fail.
* How to use the `?` operator to propagate errors cleanly.
* How to design informative and actionable custom error types.
* How to use the `thiserror` crate to implement standard error traits automatically.
* How to preserve the original error cause chain across application layers.
* When to use `thiserror` for structured library errors and `anyhow` for application entry points.

## Panics and recoverable errors

Rust distinguishes between two categories of failure: unrecoverable errors (panics) and recoverable errors.

### Unrecoverable errors (Panics)
A **panic** represents a bug or an invariant violation in your code.
* **When it occurs**: For example, when you access a vector index that is out of bounds or divide an integer by zero.
* **What happens**: The program displays an error message, unwinds the stack to clean up resources, and terminates the execution.
* **When to use**: Use panics only for logic errors and broken assumptions from which your program cannot recover.

### Recoverable errors
A recoverable error represents an expected failure scenario that your application can handle.
* **When it occurs**: For example, when a file is missing, a network request times out, or user input is malformed.
* **What happens**: The function returns a value that represents the failure.
* **When to use**: If the calling function can reasonably respond to the failure (for example, by prompting the user for a different filename or retrying a request), you must return a recoverable error value.

> **If you come from Go:**
> The `Result` type works similarly to returning a value and an error tuple `(value, err)`. However, Rust provides compiler checks and a dedicated propagation operator (`?`) to streamline the workflow.
>
> **If you come from Java, Python, or JavaScript:**
> In Rust, recoverable errors are visible in the function signatures. You do not use exceptions or try-catch blocks.

## Recoverable errors with `Result<T, E>`

The `Result<T, E>` type is a standard library enum defined as follows:

```rust
enum Result<T, E> {
    Ok(T),  // Represents success and contains the output value of type `T`.
    Err(E), // Represents failure and contains the error details of type `E`.
}
```

Because `Result` is an enum, you can use pattern matching or standard combinators to handle its variants explicitly:

```rust
fn read_config(path: &str) -> Result<Config, ConfigError> {
    // ...
}

fn main() {
    match read_config("app.toml") {
        Ok(config) => start(config),
        Err(error) => eprintln!("Failed to load configuration: {error}"),
    }
}
```

### Common `Result` methods and combinators
Instead of writing a full `match` statement, you can use built-in combinators:
* **`.map`**: Transforms the successful value inside `Ok` if present.
* **`.unwrap_or_else`**: Returns the successful value, or evaluates a fallback closure if the variant is `Err`.
* **`.ok()`**: Converts a `Result<T, E>` into an `Option<T>` (discards the error details).
* **`Option::ok_or`**: Converts an `Option<T>` into a `Result<T, E>` by providing a fallback error.

> **Best practice**:
> Avoid using `.unwrap()` or `.expect()` in library APIs. These methods panic on failure and should be reserved for unit tests, example code, or when a code invariant guarantees success.

## Error propagation with the `?` operator

The `?` operator provides a clean way to return errors early to the calling function. The following two examples perform the exact same steps:

```rust
// Long-form manual propagation
fn load_expanded(path: &str) -> Result<Config, ConfigError> {
    let text = match std::fs::read_to_string(path) {
        Ok(text) => text,
        Err(error) => return Err(error.into()), // Returns early on error
    };

    parse(&text)
}

// Idiomatic propagation using the `?` operator
fn load(path: &str) -> Result<Config, ConfigError> {
    let text = std::fs::read_to_string(path)?; // Returns early on error
    parse(&text)
}
```

### Steps performed by the `?` operator
When you append `?` to a `Result` value, Rust performs the following actions:
1. **Unwraps `Ok`**: If the result is `Ok(value)`, the program extracts the inner value and continues execution.
2. **Returns early on `Err`**: If the result is `Err(error)`, the program immediately stops execution and returns the error to the caller.
3. **Converts error types automatically**: During the early return, the `?` operator automatically converts the error value into the caller's expected error type, if a valid `From` trait implementation exists.

This design keeps the successful path linear and readable while ensuring that every potential error propagation point is marked clearly.

## Designing effective error enums

Because errors are ordinary values, you must design their structure to support robust handling. The following stages demonstrate how to design an effective error type:

### Stage 1: String-based errors (Not recommended)
```rust
fn load_document(id: DocumentId) -> Result<Document, String> {
    // ...
}
```
String-based errors are quick to write but difficult to handle. The calling function cannot easily analyze or branch on text messages, leaving it unable to programmatically recover.

### Stage 2: Basic enums (Partially recommended)
```rust
enum DocumentError {
    NotFound,
    PermissionDenied,
    VersionConflict,
}
```
An enum allows the caller to match variants explicitly. However, this structure lacks necessary context. If a version conflict occurs, the caller does not know which version is currently active or who made the modification.

### Stage 3: Context-rich enums (Highly recommended)
Include specific, actionable fields in your enum variants so that callers can respond programmatically:

```rust
pub enum DocumentError {
    NotFound(DocumentId),
    PermissionDenied { 
        doc: DocumentId, 
        user: UserId 
    },
    VersionConflict {
        expected: i32,
        actual: i32,
        modified_by: UserId,
        modified_at: DateTime<Utc>,
    },
}
```

This context-rich structure provides several benefits:
* **Actionable feedback**: Your user interface can explain precisely who changed the document and when.
* **Programmatic recovery**: Synchronization clients can inspect the version fields to determine whether to retry or merge changes.
* **Detailed diagnostics**: System logs can record full error attributes without needing to run separate database queries.

### Architectural guideline
Define one custom error enum per module or service layer. An application-wide error enum is often too broad, while a separate enum for every single function introduces unnecessary complexity.

## Implementing errors with `thiserror`

All custom error types should implement the standard library `std::fmt::Display` and `std::error::Error` traits. Implementing these traits manually requires significant boilerplate code. You can use the `thiserror` crate to generate these implementations automatically:

```rust
#[derive(Debug, thiserror::Error)]
pub enum DocumentError {
    #[error("document {0} not found")]
    NotFound(DocumentId),

    #[error(
        "version conflict: expected v{expected}, found v{actual} \
         (modified by {modified_by} at {modified_at})"
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

The `#[error("...")]` attribute defines the user-facing format string, interpolating the variant's fields automatically. The crate derives the required traits while keeping your custom error variants clean and structured.

## Preserving the error chain

In layered applications, you must determine how to handle errors returned from lower-level layers (such as a database).

### Lossy conversion (Not recommended)
Avoid converting underlying errors into plain strings:
```rust
#[error("storage failure: {0}")]
Store(String), // This discards the original error type.
```
Converting to a string discards the underlying error type, preventing callers from matching on it. It also breaks the error source chain used by diagnostics tools.

### Structured wrapping (Recommended)
Store the original error type directly in your variant using the `#[from]` or `#[source]` attributes:

```rust
#[error("storage failure")]
Store(#[from] sqlx::Error),
```

The `#[from]` attribute:
* Preserves the full error type and its underlying cause chain.
* Generates a `From` trait implementation, allowing the `?` operator to convert the lower-level error to your custom error type automatically.

### Mapping errors at boundaries
At the outer boundary of your system (such as an HTTP routing handler), map your custom domain errors to external representations (such as HTTP status codes) exhaustively:

```rust
match error {
    DocumentError::NotFound(_)             => 404,
    DocumentError::PermissionDenied { .. } => 403,
    DocumentError::VersionConflict { .. }  => 409,
    DocumentError::Store(_)                => 500,
}
```

Exhaustive matches ensure that if you add a new error variant, the compiler forces you to define its corresponding boundary mapping code.

## Structured errors vs. application reporting

Rust applications commonly use different error handling tools depending on where the error is consumed:

* **In libraries and domain layers**: Use **`thiserror`** to define typed, structured enums. Callers of your library need distinct variants so they can match on them and recover programmatically.
* **At application boundaries and entry points**: Use **`anyhow`** to handle errors. Entry points (such as the `main` function or command-line parsers) usually focus on presenting readable reports and diagnostic context to humans, rather than recovering programmatically.

The following example demonstrates using `anyhow` at the application entry point to add diagnostic context:

```rust
use anyhow::Context;

fn main() -> anyhow::Result<()> {
    let config = load_config().context("failed to load application configuration")?;
    let database = connect(&config.db_url).context("failed to connect to database")?;

    run(config, database)
}
```

### Guidelines for anyhow vs. thiserror
* Do not expose `anyhow::Error` in public library APIs where callers must match on specific failure variants.
* Do not design complex custom error hierarchies for small scripts or simple applications that only require high-level, human-readable logging. Use `anyhow` instead.

## Summary

This chapter discussed error handling in Rust:
* **Recoverable errors** are represented explicitly as values using the `Result<T, E>` enum, while **panics** represent unrecoverable invariant violations.
* **The `?` operator** simplifies error propagation by automatically returning errors early and converting types using the `From` trait.
* **Effective error enums** store detailed, context-rich fields in each variant to support programmatic error recovery and UI messaging.
* **The `thiserror` crate** eliminates trait boilerplate by deriving standard error traits automatically using format string attributes.
* **The `#[from]` attribute** wraps lower-level errors to preserve the complete cause chain for diagnostics.
* **Use `thiserror`** to declare structured error enums in domain layers and libraries, and use **`anyhow`** to format human-readable error reports at your application's entry boundaries.
