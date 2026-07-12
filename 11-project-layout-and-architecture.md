# Chapter 11: Project Layout and Architecture

Chapter 8 introduced the principles: crates act as architectural boundaries,
dependencies point inward, and the application has one composition root. This chapter
turns those principles into concrete file layouts for three common project shapes. It
also explains the module conventions and architectural patterns that repeatedly appear
in production Rust services.

## 11.1 Shape A: a CLI tool or small binary

```text
mytool/
├── Cargo.toml
├── README.md
├── src/
│   ├── main.rs           # Thin entry point: parse, call, map the final error
│   ├── lib.rs            # Testable application logic
│   ├── cli.rs            # clap argument definitions
│   └── commands/
│       ├── mod.rs
│       ├── sync.rs       # One file per subcommand
│       └── status.rs
└── tests/
    └── cli.rs            # Integration tests that execute the binary
```

The most important rule is simple: **keep `main.rs` thin**. Even for a small CLI, place
the actual behavior in `lib.rs`. Let `main` parse arguments, call the library, and
convert the final `Result` into the process outcome.

```rust
use clap::Parser;

fn main() -> anyhow::Result<()> {
    let args = mytool::cli::Args::parse();
    mytool::run(args)
}
```

This structure keeps the application logic unit-testable without spawning a process. It
also allows the tool to be reused as a library later if that becomes useful.

## 11.2 Shape B: a library crate

```text
mylib/
├── Cargo.toml
├── README.md             # Can also serve as the crate's front-page docs
├── src/
│   ├── lib.rs            # Crate docs, module declarations, and re-exports
│   ├── error.rs          # The crate's thiserror enum
│   ├── client.rs         # Main behavior
│   └── types.rs          # Supporting public and internal types
├── examples/
│   └── basic.rs          # Run with `cargo run --example basic`
├── benches/
│   └── throughput.rs     # criterion benchmarks
└── tests/
    └── integration.rs    # Tests through the public API
```

A library's `lib.rs` usually begins with `//!` crate-level documentation. You can also
reuse the README directly:

```rust
#![doc = include_str!("../README.md")]
```

The rest of `lib.rs` should mostly contain module declarations and curated re-exports:

```rust
mod client;
mod error;
mod types;

pub use client::Client;
pub use error::Error;
```

Users can now write `mylib::Client` instead of depending on the internal path
`mylib::client::Client`. This keeps the module tree private and gives you room to
reorganize the implementation without breaking callers.

For a library with a deliberately documented public API, consider adding
`#![deny(missing_docs)]`. This turns missing public documentation into a build error.
The `examples/` directory is also useful because those programs can be compiled in CI,
which helps detect examples that no longer match the API.

## 11.3 Shape C: a workspace service

The following tree applies the layered architecture from chapters 8 and 9 to a complete
service:

```text
docstore/
├── Cargo.toml                     # [workspace] and [workspace.dependencies]
├── Cargo.lock                     # Committed because this is an application
├── rust-toolchain.toml            # Pins the compiler version
├── deny.toml                      # cargo-deny policy
├── docker-compose.yml             # Local infrastructure such as PostgreSQL
├── Dockerfile                     # Multi-stage build and slim runtime image
├── .github/
│   └── workflows/
│       └── ci.yml                 # fmt, clippy, tests, and audit
│
└── crates/
    ├── domain/                    # Core rules; no infrastructure I/O
    │   ├── Cargo.toml
    │   └── src/
    │       ├── lib.rs             # Modules and curated re-exports
    │       ├── ids.rs             # DocumentId, UserId, and other newtypes
    │       ├── model/
    │       │   ├── mod.rs
    │       │   └── document.rs    # Document and validated value objects
    │       ├── events.rs          # DocumentEvent enum
    │       ├── errors.rs          # DocumentError using thiserror
    │       └── services/
    │           ├── mod.rs
    │           └── document_service.rs
    │
    ├── store-postgres/            # Adapter implementing domain repository traits
    │   ├── Cargo.toml
    │   ├── migrations/
    │   │   └── 0001_create_documents.sql
    │   └── src/
    │       ├── lib.rs
    │       └── repository.rs
    │
    ├── store-s3/                  # Blob-storage adapter
    │   └── src/
    │       ├── lib.rs
    │       └── blob_store.rs
    │
    └── server/                    # HTTP edge and dependency wiring
        ├── Cargo.toml
        └── src/
            ├── main.rs            # Composition root only
            ├── config.rs          # Environment/files into a typed Config
            ├── telemetry.rs       # tracing subscriber configuration
            └── api/
                ├── mod.rs          # Router assembly
                ├── documents.rs    # /documents handlers
                └── error_map.rs    # Domain errors to HTTP responses
```

Several placement rules are worth stating explicitly.

**Traits live beside their consumers.** A service defines the repository or gateway
behavior it needs, and an adapter implements that contract. A shared `traits` crate
often collects provider-oriented interfaces whose ownership is difficult to identify.

**Tests stay close to the behavior they verify.** Unit tests belong in or beside service
modules. Adapters should include a smaller set of integration tests against real
infrastructure, while the server layer usually needs only a thin set of end-to-end smoke
tests.

**Migrations belong to the adapter that owns the schema.** Keeping SQL migrations in
`store-postgres` makes the ownership boundary explicit.

**Error mapping lives in one place.** An exhaustive match in `error_map.rs` translates
domain errors into HTTP responses. When a new error variant appears, the compiler
identifies the API mapping that still needs a decision.

**`main.rs` performs wiring, not business logic.** It should load configuration, initialize
telemetry, construct adapters and services, assemble the router, and start the server.
Business rules belong in the tested domain or application layers rather than in startup
code.

## 11.4 Module conventions inside a crate

In the modern module style, a module can live in `foo.rs`, while its child modules live
in the sibling directory `foo/`. Prefer this arrangement over placing `mod.rs` in every
directory; using named module files can make a large project easier to navigate than
having many files with the same `mod.rs` name.

Split a module when it contains more than one meaningful concept or has acquired a
second reason to change. Line count alone is not a reliable boundary.

Keep visibility narrow by default. Start with private items, promote them to
`pub(crate)` when sibling modules need access, and treat each bare `pub` as a deliberate
API decision. Similarly, avoid broad preludes such as `use crate::prelude::*` unless the
crate genuinely benefits from one. Explicit imports make reviews easier, and
rust-analyzer can manage them with little effort.

## 11.5 Recurring architectural patterns

Beyond the layered skeleton, five patterns appear in many production Rust services.

### Typed configuration, parsed once

Deserialize environment variables and configuration files into a single `Config` value
during startup. After the opening section of `main`, application code should not read
environment variables directly; it should receive typed configuration.

Fail fast. A missing or malformed setting should stop the process at startup with a
useful message rather than fail at 03:00 when the value is first used.

### Builders for constructors with many options

Rust has no keyword arguments or default parameters. Once a constructor grows beyond a
few related arguments, a builder usually becomes easier to read:

```rust
let server = Server::builder()
    .port(8080)
    .timeout(Duration::from_secs(30))
    .build()?;
```

Let `build()` return `Result` when combinations can be invalid. This creates one
validation point. Crates such as `bon` or `typed-builder` can generate the repetitive
parts.

### Graceful shutdown

Listen for Ctrl-C and SIGTERM through `tokio::signal`. Stop accepting new work, allow
in-flight operations a bounded amount of time to finish, and flush telemetry before the
process exits.

Kubernetes, load balancers, and deployment systems assume this behavior. Adding graceful
shutdown late can require changes across many task-spawning locations, so it is useful
to establish the structure early in a service.

### Atomic state and event writes, followed by projections

The transactional-outbox pattern commits current state and an immutable event in the
same transaction. After the commit succeeds, downstream consumers can build projections
such as search indexes, caches, or notifications.

Those projections are derived and rebuildable rather than authoritative sources of
state. Rust supports this model well because enums make event sets explicit, ownership
helps prevent accidental reuse after commit, and crate boundaries keep projection code
out of the write path.

### Actor-style task ownership

When shared-state locking becomes difficult to reason about, invert the relationship.
Let one spawned task own the state, and let other tasks communicate with it through a
typed `mpsc` channel.

This removes shared mutation and lock poisoning from the design. It is a natural
extension of the “channels for flow” principle from chapter 7. Consider the pattern when
an `Arc<Mutex<_>>` begins to accumulate several unrelated `.lock()` call sites.

## 11.6 Naming conventions

Use `snake_case` for crates and modules, usually with short nouns such as `domain` or
`store_postgres`. Use `CamelCase` for types and traits. Trait names should describe
capabilities—`OrderStore`, `Describe`, or `Serialize`—rather than imitate conventions
such as `IThing` or `ThingInterface`.

Follow standard constructor names:

- `new` for infallible construction
- `try_new` for construction that returns `Result`
- `from_x` and `with_x` for alternative construction or configuration

Conversions should also follow standard-library conventions:

- `as_` for a cheap borrowed view
- `to_` for a conversion that may allocate or copy
- `into_` for a consuming conversion

These conventions are not cosmetic. They make your code read like the ecosystem around
it and reduce the amount of explanation each API requires.

## Summary

Three layouts cover a large share of Rust projects: a CLI with a thin `main`, a library
with a curated public surface, and a layered workspace whose dependencies point toward
the domain.

Within a crate, prefer the `foo.rs` plus `foo/` layout, keep visibility private by
default, and use explicit imports. Across the architecture, reuse typed configuration,
builders, graceful shutdown, transactional outboxes with projections, and actor-style
ownership when shared locking becomes cumbersome.
