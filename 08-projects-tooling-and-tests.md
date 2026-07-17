# Projects, tooling, and testing

This chapter explains how to organize, build, and test larger Rust applications. You will learn how to use modules, crates, and workspaces to structure your project. You will also learn how to use standard Rust tools and write fast, deterministic tests using fakes.

By the end of this chapter, you will understand:
* How to organize code within a single crate using modules and visibility rules.
* How to use crates as compilation boundaries to enforce architectural rules.
* How to manage multiple related crates using Cargo workspaces.
* How to design a layered project layout with enforced dependency boundaries.
* How to assemble your application components at the composition root.
* How to use standard Rust formatting, linting, and testing tools.
* How to write unit tests, integration tests, and doc-tests.
* How to test business logic using fakes and assertion macros.

## Modules

A **module** is a namespace that allows you to organize code within a single crate. Modules also control the visibility (privacy) of your code. By default, all items in a module are private and cannot be accessed from outside the module.

The following example defines a module with different visibility levels:

```rust
mod billing {
    // This struct is visible outside of the `billing` module.
    pub struct Invoice {
        /* ... */
    }

    // This function is visible anywhere within the current crate.
    pub(crate) fn tax_rate() -> f32 {
        0.20
    }

    // This function is private to the `billing` module.
    fn round(value: f32) -> f32 {
        /* ... */
    }
}

use billing::Invoice;
```

### Module file mapping
Module structures can match your filesystem structure. When you declare a module with `mod billing;` in `main.rs` or `lib.rs`, Cargo searches for the module code in either:
* `src/billing.rs`
* `src/billing/mod.rs` (used when a module contains sub-modules)

### Recommended visibility practice
Use **`pub(crate)`** to share internal helper functions or data types within your crate. Reserve bare **`pub`** only for items that are part of your crate's public API.

## Crates

A **crate** is the primary unit of compilation and package distribution in Rust. There are two types of crates:
* **Binary crates**: Contain a `src/main.rs` file. They compile into an executable binary and contain a `main` function as the entry point.
* **Library crates**: Contain a `src/lib.rs` file. They expose public functionality to other crates and do not compile into an executable.

A single Cargo package can contain both a library crate and a binary crate.

### Architectural boundaries in crates
A crate cannot use external dependencies unless those dependencies are explicitly listed in its `Cargo.toml` file. This means you can use crate boundaries to enforce architectural rules. For example, if you want to ensure that your core business logic does not depend on a specific database driver, you can place that logic in a separate library crate that omits the database driver from its dependency list. The compiler then enforces this rule automatically.

## Workspaces

A **workspace** allows you to manage multiple related crates together in a single repository. All crates in a workspace share a single root manifest, a single `Cargo.lock` file, and a single build directory. This helps prevent dependency version drift and speeds up compilation times.

The following example shows a root `Cargo.toml` file for a workspace:

```toml
[workspace]
members = ["crates/domain", "crates/store-postgres", "crates/server"]

[workspace.dependencies]
tokio     = { version = "1", features = ["rt-multi-thread", "macros"] }
sqlx      = { version = "0.8", features = ["postgres", "runtime-tokio-rustls"] }
thiserror = "2"
```

To use a shared dependency in a workspace member's `Cargo.toml` file, reference it like this:
```toml
[dependencies]
tokio = { workspace = true }
```

## Layered architecture layout

By combining crate boundaries with consumer-defined traits (see Chapter 6), you can design a highly decoupled, layered architecture:

```text
myapp/
└── crates/
    ├── domain/            # Core business logic (entities, services, errors, traits)
    │                      # Dependencies: thiserror, uuid (no database or web libraries)
    │
    ├── store-postgres/    # Database adapter that implements domain traits
    │                      # Dependencies: domain, sqlx
    │
    └── server/            # Application entry point (routing, configurations, startup)
                           # Dependencies: domain, store-postgres, axum
```

This layout enforces the rule that dependencies must point inward toward the core domain logic:
* **No database dependencies in the domain**: Because the `domain` crate's `Cargo.toml` omits `sqlx`, developers cannot write database-specific code inside the business logic layer.
* **No cyclic dependencies**: Cargo does not allow cyclic dependencies between crates. This prevents tight coupling between layers and keeps your codebase easy to maintain.

## The composition root

In a layered design, you assemble your concrete implementations in a single entry point called the **composition root** (typically the `main` function or a startup helper module). This is where you pass concrete adapters to your generic services:

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let config = Config::from_env()?;
    let pool = sqlx::PgPool::connect(&config.database_url).await?;

    // Create the concrete database adapter.
    let repository = PostgresOrderRepository::new(pool);

    // Wire the generic service with the adapter.
    let orders = OrderService::new(repository);
    
    // Set up the API router and start the server.
    let app = api::router(orders);
    let listener = tokio::net::TcpListener::bind(config.addr).await?;

    axum::serve(listener, app).await?;
    Ok(())
}
```

### Benefits of manual dependency wiring
* **Compile-time safety**: Rust does not require complex runtime dependency-injection frameworks. The compiler checks that all arguments match the expected trait interfaces. If an adapter does not implement a required trait method, the project fails to compile at this wiring step.
* **Simplified testing**: To test the server or services, you can swap the database adapter for a lightweight in-memory fake directly at the call site.

## Developer tools

The following Cargo tools are recommended for daily development and continuous integration (CI) workflows:

*   **`cargo fmt`**: Formats your codebase to match standard Rust formatting style rules. Run `cargo fmt --check` in your CI pipeline to verify formatting.
*   **`cargo clippy`**: Analyzes your code for common mistakes, styling issues, and unidiomatic patterns. Run `cargo clippy -- -D warnings` in your CI pipeline to treat warnings as build errors.
*   **`cargo doc --open`**: Generates searchable HTML documentation from your code's triple-slash (`///`) comments and opens it in your default browser.
*   **`cargo audit`**: Scans your dependency graph against the Rust Security Advisory Database to check for known security vulnerabilities.

## Testing in Rust

Rust and Cargo provide built-in support for three primary forms of testing without requiring external test runners.

### 1. Unit tests
Unit tests verify individual components or functions. They are co-located with the implementation inside the same source file, inside a testing module annotated with `#[cfg(test)]`. This attribute ensures that the test code is excluded from your production builds.

```rust
// src/services/order_service.rs

// This module is only compiled during `cargo test`.
#[cfg(test)]
mod tests {
    use super::*; // Allows tests to access private items in the parent module.

    #[test]
    fn totals_are_summed() {
        assert_eq!(total(&[2, 3]), 5);
    }
}
```

### 2. Integration tests
Integration tests verify that different parts of your library work together. They live in a top-level directory named `tests/` at the root of your crate. Each file in this directory is compiled as a separate binary that can only access the public API of your library.

### 3. Documentation tests (Doc-tests)
Any code blocks included in your triple-slash (`///`) documentation comments are compiled and run automatically when you run `cargo test`. This ensures that your documentation examples remain accurate and up-to-date.

### Common assertion macros
* **`assert!`**: Verifies that a boolean condition evaluates to `true`.
* **`assert_eq!`**: Verifies that two values are equal.
* **`matches!`**: Verifies that an expression matches a specific pattern (highly useful for checking enum variants with data).

## Testing business logic with fakes

Because your core services depend on consumer-owned traits, you do not need complex mocking frameworks to write unit tests. Instead, you can write small, in-memory **fakes** that implement your traits.

The following example defines a fake database repository for testing:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::Mutex;

    #[derive(Default)]
    struct FakeRepo {
        // Use a Mutex to ensure the fake remains thread-safe.
        orders: Mutex<Vec<Order>>,

        // A flag to simulate commit failures.
        fail_commit: bool,
    }

    impl OrderRepository for FakeRepo {
        // The fake does not need a real transaction handle.
        type Tx = ();

        async fn begin(&self) -> Result<Self::Tx, StoreError> {
            Ok(())
        }

        async fn insert_in_tx(
            &self,
            _tx: &mut Self::Tx,
            order: &Order,
        ) -> Result<(), StoreError> {
            self.orders.lock().unwrap().push(order.clone());
            Ok(())
        }

        async fn append_event_in_tx(
            &self,
            _tx: &mut Self::Tx,
            _event: &OrderEvent,
        ) -> Result<(), StoreError> {
            Ok(())
        }

        async fn commit(&self, _tx: Self::Tx) -> Result<(), StoreError> {
            if self.fail_commit {
                Err(StoreError::ConnectionLost)
            } else {
                Ok(())
            }
        }
    }

    #[tokio::test]
    async fn stale_update_reports_who_changed_it() {
        let repository = FakeRepo::with_existing(order_at_version(3));
        let service = OrderService::new(repository);

        let error = service
            .update(cmd_expecting_version(2))
            .await
            .unwrap_err();

        // Use `matches!` to verify the exact error variant and its fields.
        assert!(matches!(
            error,
            OrderError::VersionConflict {
                expected: 2,
                actual: 3,
                ..
            }
        ));
    }
}
```

### Advantages of using fakes
* **Speed and predictability**: These tests run entirely in memory, making them extremely fast and deterministic. They do not depend on external databases or networks.
* **Clear error assertions**: By using the `matches!` macro, you can verify both the exact error variant and the specific data fields returned. This ensures your code returns meaningful errors.
* **Separation of concerns**: Use in-memory fakes to verify your core business logic, and use a separate, small set of integration tests to verify database-specific queries (such as SQL statements running against an actual database container).

## Summary

This chapter discussed project structure, tooling, and testing in Rust:
* **Modules** organize code inside a crate, while **crates** form dependency and compilation boundaries.
* **Workspaces** group multiple related crates in a single repository with shared dependencies and lockfiles.
* **A layered project layout** uses crate boundaries to enforce clean dependencies that point inward toward the core domain logic.
* **The composition root** manually wires adapters and services together at program startup, providing compile-time safety without a runtime framework.
* **Cargo tools** like `cargo fmt`, `cargo clippy`, and `cargo audit` enforce style rules and detect security issues.
* **Rust tests** support co-located unit tests, integration tests, and doc-tests.
* **In-memory fakes** allow you to test complex business logic quickly and deterministically without external infrastructure.
