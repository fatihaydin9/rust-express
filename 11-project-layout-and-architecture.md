# Project layout and architecture

This chapter explains how to organize directories and structure code for three common project shapes in Rust: command-line interface (CLI) applications, libraries, and large workspaces. It also describes standard module organization rules, naming conventions, and recurring architectural patterns used in production-grade Rust systems.

By the end of this chapter, you will understand:
* How to lay out directories for a CLI tool or small binary.
* How to design a library crate with a curated public API.
* How to structure a multi-crate workspace for a layered microservice.
* How to organize modules and visibility rules within a single crate.
* Standard architectural patterns like typed configuration, builder patterns, graceful shutdown, and actor-based state management.
* Standard Rust naming conventions for crates, variables, functions, and constructors.

## Layout for command-line tools and small binaries

The following structure is recommended for CLI applications or small executable binaries:

```text
mytool/
├── Cargo.toml
├── README.md
├── src/
│   ├── main.rs           # The entry point (handles parsing and logs errors)
│   ├── lib.rs            # Core application logic
│   ├── cli.rs            # Command-line argument definitions using Clap
│   └── commands/
│       ├── mod.rs
│       ├── sync.rs       # Individual file for a subcommand
│       └── status.rs     # Individual file for a subcommand
└── tests/
    └── cli.rs            # Integration tests that run the compiled binary
```

### The thin entry point rule
To keep your code modular and testable, **do not place business logic in your `main.rs` file**. Instead, your `main.rs` file should only:
1. Parse command-line arguments.
2. Call an entry function defined in your `lib.rs` file.
3. Map any returned errors to system exit codes.

```rust
use clap::Parser;

fn main() -> anyhow::Result<()> {
    let args = mytool::cli::Args::parse();
    mytool::run(args)
}
```

Placing the primary behavior in `lib.rs` allows you to write unit tests for your application logic without spawning a separate operating system process. This layout also makes it easy to reuse your core application code as a library in other projects.

## Layout for library crates

The following structure is recommended for library crates that publish APIs for other applications to use:

```text
mylib/
├── Cargo.toml
├── README.md             # Explains the purpose of the library and contains usage examples
├── src/
│   ├── lib.rs            # Exposes the public modules and contains crate-level documentation
│   ├── error.rs          # Declares the library's custom `thiserror` enum
│   ├── client.rs         # Implements the primary library client
│   └── types.rs          # Declares public and internal data types
├── examples/
│   └── basic.rs          # Demonstrates basic library usage
├── benches/
│   └── throughput.rs     # Benchmarks created using Criterion
└── tests/
    └── integration.rs    # Verification tests using only the public API
```

### Writing library entry files (`lib.rs`)
The `lib.rs` file should contain your crate-level documentation and serve as a gateway to your public types. You can reuse your `README.md` file directly in your source documentation using the following syntax:

```rust
#![doc = include_str!("../README.md")]
```

Structure your `lib.rs` to declare internal modules and re-export only the selected public types. This keeps your internal module tree private and allows you to reorganize your implementation later without breaking callers:

```rust
// Declare internal modules.
mod client;
mod error;
mod types;

// Re-export public items to the root level.
pub use client::Client;
pub use error::Error;
```

With this layout, users can write `use mylib::Client;` directly instead of referencing the private path `use mylib::client::Client;`.

### Recommended practices for libraries
* **Turn on documentation enforcement**: Add `#![deny(missing_docs)]` to your `lib.rs` file to cause compilation to fail if a public API item lacks documentation comments.
* **Keep examples compiled**: The `examples/` directory is automatically compiled when you run tests. This ensures your code examples remain compatible with your API changes.

## Layout for workspace-based services

For larger applications (such as microservices or backend APIs), use a layered Cargo workspace to enforce clean boundaries between your core business rules and infrastructure dependencies:

```text
docstore/
├── Cargo.toml                     # Workspace manifest mapping members and dependencies
├── Cargo.lock                     # Committed to version control for deterministic builds
├── rust-toolchain.toml            # Specifies the exact Rust compiler version
├── deny.toml                      # Defines allowed dependency licenses and warnings
├── docker-compose.yml             # Sets up local databases or other services
├── Dockerfile                     # Multi-stage Docker build for clean runtime images
├── .github/
│   └── workflows/
│       └── ci.yml                 # Automated pipeline for formatting, linting, and testing
│
└── crates/
    ├── domain/                    # Pure business logic (entities, services, errors, traits)
    │   ├── Cargo.toml
    │   └── src/
    │       ├── lib.rs             # Curated exports for the domain crate
    │       ├── ids.rs             # Unique ID newtypes (DocumentId, UserId)
    │       ├── model/
    │       │   ├── mod.rs
    │       │   └── document.rs    # Core Document model and parsing constructors
    │       ├── events.rs          # DocumentEvent enum
    │       ├── errors.rs          # DocumentError using thiserror
    │       └── services/
    │           ├── mod.rs
    │           └── document_service.rs
    │
    ├── store-postgres/            # PostgreSQL adapter that implements domain traits
    │   ├── Cargo.toml
    │   ├── migrations/
    │   │   └── 0001_create_documents.sql # SQL migration scripts
    │   └── src/
    │       ├── lib.rs
    │       └── repository.rs
    │
    ├── store-s3/                  # S3 storage adapter that implements domain blob traits
    │   └── src/
    │       ├── lib.rs
    │       └── blob_store.rs
    │
    └── server/                    # Web routing edge and service initialization
        ├── Cargo.toml
        └── src/
            ├── main.rs            # The composition root
            ├── config.rs          # Parses settings from files or environment variables
            ├── telemetry.rs       # Configures tracing subscribers
            └── api/
                ├── mod.rs          # Assembles the routing table
                ├── documents.rs    # HTTP handlers for /documents endpoints
                └── error_map.rs    # Maps custom domain errors to HTTP status codes
```

### Essential architecture placement rules
* **Traits belong next to their consumers**: Define repository and storage traits inside the `domain` crate, next to the services that use them. Your infrastructure adapter crates (such as `store-postgres`) import the domain crate to implement those traits.
* **Co-locate tests**: Place unit tests inside the same modules they verify. Keep infrastructure integration tests inside their respective adapter crates, and place only thin end-to-end smoke tests in the `server` layer.
* **Keep migrations in the adapter**: Store SQL migration files inside the `store-postgres` crate. This makes the database adapter the clear owner of the schema.
* **Consolidate error mapping**: Keep all error-to-HTTP mapping logic in a single file (`error_map.rs`). Use an exhaustive match to ensure that if you add a new error variant to the domain, the compiler alerts you to assign its corresponding HTTP status code.
* **Use `main.rs` as a composition root**: Your `main.rs` file should only configure startup tasks: reading configuration settings, initializing telemetry, instantiating database adapters, wiring services, and starting the web listener. Do not write business logic inside the startup entry point.

## Module organization rules

When structuring modules inside a single crate, use the modern Rust module layout:
* Place the parent module code in `foo.rs`.
* Place any child modules in a sibling directory named `foo/`.

Avoid placing a `mod.rs` file inside every subdirectory. Using named module files makes it easier to navigate large projects in your editor because you do not have multiple open files named `mod.rs`.

### Visibility guidelines
* **Start private**: Keep variables, functions, and structs private to their module by default.
* **Promote to internal sharing**: Use `pub(crate)` if a sibling module inside the same crate must access the item.
* **Expose to public APIs carefully**: Use bare `pub` only when an item is a deliberate part of your crate's public API contract.
* **Avoid bulk imports**: Do not import modules using glob stars (such as `use crate::prelude::*`) unless the crate specifically requires a prelude module. Explicit, individual imports make code reviews easier, and tools like rust-analyzer manage them automatically.

## Recurring architectural patterns

The following five patterns are commonly used in production-grade Rust services:

### 1. Parse configuration once at startup
Deserialize environment variables and configuration files into a single, typed `Config` struct immediately when your application starts. Do not read environment variables directly from inside your core business logic services.

If a configuration setting is missing or malformed, the application should terminate immediately at startup with a clear error message. This prevents unexpected runtime failures later.

### 2. The builder pattern for complex constructors
Rust does not support default parameter values or named keyword arguments in function calls. If a constructor requires more than a few parameters, use the builder pattern to improve code readability:

```rust
let server = Server::builder()
    .port(8080)
    .timeout(Duration::from_secs(30))
    .build()?;
```

Let the `.build()` method return a `Result` if certain parameter combinations can be invalid. This establishes a single validation boundary for your custom type. You can use helper crates such as `bon` or `typed-builder` to generate builder code automatically.

### 3. Graceful shutdown
Ensure that your application handles terminating signals (such as Ctrl-C or SIGTERM) cleanly using your async runtime:
1. Stop accepting new incoming requests.
2. Allow active, in-flight operations a bounded time window to complete.
3. Flush any pending telemetry or logging buffers before the process terminates.

This is critical for containerized environments like Kubernetes, which expect processes to stop gracefully when receiving signals.

### 4. Transactional outbox with derived projections
To maintain consistency across services, use the transactional-outbox pattern to write your current application state and an immutable audit event together inside the same database transaction.

Once the transaction commits successfully, you can broadcast the audit event to update derived projections (such as a search index, key-value caches, or outbound notifications). Because projections are derived, you can rebuild them at any time from the source event stream.

### 5. Actor-based state management
If sharing variables across threads using standard locking (`Arc<Mutex<T>>`) becomes complex or causes performance bottlenecks, use an actor-based pattern:
1. Spawn a single async task that retains exclusive ownership of the state.
2. Allow other threads and tasks to interact with the state exclusively by sending messages through a Multi-Producer, Single-Consumer (`mpsc`) channel.

This approach removes shared mutable data and locking rules from your core business logic.

## Naming conventions

Following standard ecosystem naming rules ensures that your APIs read consistently:
* **Crates and modules**: Use `snake_case` using short, descriptive nouns (for example, `domain` or `store_postgres`).
* **Types and traits**: Use `CamelCase` (or `PascalCase`). Trait names should describe capabilities (for example, `OrderStore`, `Describe`, or `Serialize`) rather than mimic patterns like `IThing` or `ThingInterface`.

### Standard constructor naming
* Use **`new`** for constructors that cannot fail (infallible).
* Use **`try_new`** for constructors that validate parameters and return a `Result` type (fallible).
* Use **`from_x`** or **`with_x`** when providing alternative constructors or custom configurations.

### Standard type conversion prefixes
* Use **`as_`** for cheap, read-only conversions that return a borrowed reference (for example, `as_str`).
* Use **`to_`** for conversions that require allocating memory or copying data (for example, `to_string`).
* Use **`into_`** for conversions that consume the original variable and return a new type (for example, `into_bytes`).

## Summary

This chapter discussed project structure and architecture guidelines:
* **Layout configurations**: Use three primary shapes: a CLI tool with a thin `main.rs`, a library crate with curated root re-exports, or a multi-crate workspace.
* **Inward-pointing dependencies**: Design workspaces so that dependency arrows point toward the core domain logic.
* **Module cleanliness**: Organize sibling modules using matching folders, keep visibility private by default, and prefer explicit imports over glob stars.
* **System robustness**: Implement configuration validation once at startup, use builders for complex constructors, support graceful shutdown, commit audit events atomically with state, and use channels to pass messages instead of complex shared locks.
