# Chapter 8: Projects, Tooling, and Tests

The guide now moves beyond single-file examples. This chapter covers modules, crates,
workspaces, project tooling, and tests. Its central idea is that crate dependencies can
turn some architectural boundaries into rules checked by the compiler rather than
conventions shown only in diagrams.

## 8.1 Modules: organization within a crate

Inside one compilation unit, code organizes into **modules** — namespaces with
visibility rules:

```rust
mod billing {
    // Visible outside the `billing` module.
    pub struct Invoice {
        /* ... */
    }

    // Visible anywhere inside the current crate.
    pub(crate) fn tax_rate() -> f32 {
        0.20
    }

    // Private to the `billing` module.
    fn round(value: f32) -> f32 {
        /* ... */
    }
}

use billing::Invoice;
```

Module names usually follow the file layout. A `mod billing;` declaration in `main.rs`
or `lib.rs` loads `src/billing.rs`, or `src/billing/mod.rs` when the module is
represented by a directory.

The `pub(crate)` visibility level is especially useful: the entire crate may use the
item, but external crates may not. Prefer it for shared internals, and reserve bare
`pub` for an intentionally designed public API.

## 8.2 Crates: compilation and dependency boundaries

A **crate** is Rust's unit of compilation and distribution. Binary crates contain a
`src/main.rs` entry point, while library crates expose code through `src/lib.rs`; one
package may contain both.

The important architectural property is simple: **a crate cannot use a dependency that
is absent from its `Cargo.toml`.** When the domain crate does not declare `sqlx`,
importing SQLx there becomes a compile error rather than a convention that reviewers
must remember.

## 8.3 Workspaces: several crates managed together

A **workspace** develops several crates together — one root manifest, one shared
`Cargo.lock`, one build cache:

```toml
[workspace]
members = ["crates/domain", "crates/store-postgres", "crates/server"]

[workspace.dependencies]           # shared dependency versions
tokio     = { version = "1", features = ["rt-multi-thread", "macros"] }
sqlx      = { version = "0.8", features = ["postgres", "runtime-tokio-rustls"] }
thiserror = "2"
```

Members reference them with `tokio = { workspace = true }`. `[workspace.dependencies]`
reduces version drift by giving workspace members one place to select, audit, and
upgrade shared dependencies.

## 8.4 A layered layout with enforced dependencies

Combine §8.2 with chapter 6's consumer-owned traits and a layout falls out:

```
myapp/
└── crates/
    ├── domain/            # entities, errors, services, and consumer-owned traits
    │                      #   deps: thiserror, uuid; no sqlx or axum
    ├── store-postgres/    # implements domain's repository traits
    │                      #   deps: domain, sqlx
    └── server/            # HTTP, config, startup — wires everything
                           #   deps: domain, store-postgres, axum
```

The layout encodes one rule: dependency arrows point inward toward the domain. In
ports-and-adapters terminology, chapter 6's traits are the ports and the storage crates
are adapters.

Rust adds enforcement. Because the domain manifest omits `sqlx`, SQL-specific code
cannot enter the domain while the project still builds. Cargo also rejects cyclic crate
dependencies, preventing one common form of architectural erosion at the package level.

Choose crate boundaries according to why code changes and which dependencies it needs. A
small application may not benefit from many tiny crates, while a larger domain can
reasonably be much bigger than each adapter.

## 8.5 The composition root: wiring without a framework

In this design, concrete implementations are assembled in one place: `main`, or a nearby
`bootstrap` module. This is where the generic services from chapter 6 meet the
production adapters:

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let config = Config::from_env()?;
    let pool = sqlx::PgPool::connect(&config.database_url).await?;

    // Concrete adapter implementing the domain trait.
    let repository = PostgresOrderRepository::new(pool);

    // The generic service is wired at the composition root.
    let orders = OrderService::new(repository);
    let app = api::router(orders);
    let listener = tokio::net::TcpListener::bind(config.addr).await?;

    axum::serve(listener, app).await?;
    Ok(())
}
```

No runtime dependency-injection container is required here. The implementation is passed
as an argument, and the resulting graph is checked by the type system. If the adapter
drifts out of sync with the trait, the build breaks _here_, naming the missing method.
Swapping Postgres for an in-memory store is a different argument at this one call site —
which is exactly how the tests below work.

## 8.6 Project tools

The following tools are useful defaults for most Rust projects:

- **`cargo fmt`** applies Rust's canonical format. Run `cargo fmt --check` in CI.
- **`cargo clippy`** detects non-idiomatic code and many real bug patterns. A strict CI
  command is `cargo clippy -- -D warnings`.
- **`cargo doc --open`** renders `///` comments into browsable API documentation. Code
  examples in documentation can be compiled and executed as doc-tests by `cargo test`.
- **`cargo audit`** checks the dependency graph against the Rust security advisory
  database.

## 8.7 Three forms of tests

Rust and Cargo support three common forms of tests without requiring an external test
runner. **Unit tests** live _inside the file they test_, in a module compiled only for
testing:

```rust
// src/services/order_service.rs, at the bottom of the same file.
#[cfg(test)]
mod tests {
    use super::*; // Tests can access private items in the parent module.

    #[test]
    fn totals_are_summed() {
        assert_eq!(total(&[2, 3]), 5);
    }

    #[tokio::test]
    async fn placing_an_order_stores_it() {
        /* ... */
    }
}
```

Co-location keeps tests near the implementation, while `#[cfg(test)]` excludes the test
module from normal non-test builds. **Integration tests** live in a top-level `tests/`
directory — each file a separate binary seeing only your _public_ API. **Doc-tests** are
§8.6's compiled examples. The assertion macros are `assert!`, `assert_eq!`, and — for
enums with data — `matches!`, which you're about to need.

## 8.8 Testing the domain with a small fake

The ideas from chapters 5, 6, and 7 now combine in a testable design. `OrderService`
depends on a trait _we_ defined, so a test double is a small struct — no mocking
framework, no database, no Docker:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::Mutex;

    #[derive(Default)]
    struct FakeRepo {
        // Guarded interior mutability keeps the fake thread-safe.
        orders: Mutex<Vec<Order>>,

        // Explicit failure injection for the commit path.
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

This test exercises business logic without starting external infrastructure. It is
small, deterministic, and suitable for parallel test execution. `type Tx = ()` removes
transaction mechanics from the fake, while `fail_commit` makes a failure path easy to
reproduce.

The `matches!` assertion checks both the error variant and its important fields. The
test therefore verifies the failure contract that an API or UI will depend on, not
merely that “some error” occurred.

Two closing calibrations. A modest band of **integration tests against real Postgres**
(in a container) still belongs in the store crate — the fake proves your logic, not your
SQL. When business logic is difficult to test without infrastructure, review whether a
focused trait boundary is missing before introducing a larger mocking framework.

## Summary

Modules organize code within a crate, while crates form dependency boundaries that the
compiler can enforce. Workspaces provide one lockfile, build cache, and shared
dependency policy. A layered crate layout can turn “the domain does not know the
database” into a build rule rather than a guideline.

Wire concrete implementations once at the composition root, and use formatting, linting,
documentation tests, and dependency audits as CI gates. Consumer-owned traits then make
the domain fast to test with small fakes and precise `matches!` assertions. The final
chapter assembles these ideas into one service design.
