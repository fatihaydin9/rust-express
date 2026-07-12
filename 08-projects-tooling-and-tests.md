# Chapter 8: Projects, Tooling, and Tests

The guide now moves beyond single-file examples. This chapter covers modules, crates,
workspaces, project tooling, and tests. Its central idea is that **Rust can encode
architectural boundaries as compiler-enforced dependency rules rather than leaving them only
in diagrams.**

## 8.1 Modules: rooms inside one building

Inside one compilation unit, code organizes into **modules** — namespaces with visibility
rules:

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

Module names usually follow the file layout. A `mod billing;` declaration in `main.rs` or
`lib.rs` loads `src/billing.rs`, or `src/billing/mod.rs` when the module is represented by a
directory.

The `pub(crate)` visibility level is especially useful: the entire crate may use the item,
but external crates may not. Prefer it for shared internals, and reserve bare `pub` for an
intentionally designed public API.

## 8.2 Crates: the wall the compiler patrols

A **crate** is Rust's unit of compilation and distribution. Binary crates contain a
`src/main.rs` entry point, while library crates expose code through `src/lib.rs`; one
package may contain both.

The important architectural property is simple: **a crate cannot use a dependency that is
absent from its `Cargo.toml`.** When the domain crate does not declare `sqlx`, importing
SQLx there becomes a compile error rather than a convention that reviewers must remember.

## 8.3 Workspaces: many crates, one truth

A **workspace** develops several crates together — one root manifest, one shared
`Cargo.lock`, one build cache:

```toml
[workspace]
members = ["crates/domain", "crates/store-postgres", "crates/server"]

[workspace.dependencies]           # single source of version truth
tokio     = { version = "1", features = ["rt-multi-thread", "macros"] }
sqlx      = { version = "0.8", features = ["postgres", "runtime-tokio-rustls"] }
thiserror = "2"
```

Members reference them with `tokio = { workspace = true }`. The payoff of
`[workspace.dependencies]` is the end of _version drift_ — no three crates on three subtly
incompatible `tokio`s — and one place to audit or upgrade anything.

## 8.4 The layered layout: architecture as build success

Combine §8.2 with chapter 6's consumer-owned traits and a layout falls out:

```
myapp/
└── crates/
    ├── domain/            # entities, errors, services, THE TRAITS
    │                      #   deps: thiserror, uuid — NO sqlx, NO axum
    ├── store-postgres/    # implements domain's repository traits
    │                      #   deps: domain, sqlx
    └── server/            # HTTP, config, startup — wires everything
                           #   deps: domain, store-postgres, axum
```

The layout encodes one rule: dependency arrows point inward toward the domain. In
ports-and-adapters terminology, chapter 6's traits are the ports and the storage crates are
adapters.

Rust adds enforcement. Because the domain manifest omits `sqlx`, SQL-specific code cannot
enter the domain while the project still builds. Cargo also rejects cyclic crate
dependencies, preventing one common form of architectural erosion at the package level.

Sizing advice: split by _rate and reason of change_, not dogma. A `domain` several times
larger than any adapter is healthy; ten micro-crates for a small app is ceremony.

## 8.5 The composition root: wiring without a framework

Exactly one place knows every concrete type — `main` (or a `bootstrap` module beside it),
where chapter 6's generic services meet real implementations:

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

No dependency-injection container, no reflection: **injection is passing an argument**, and
the whole graph is type-checked. If the adapter drifts out of sync with the trait, the build
breaks _here_, naming the missing method. Swapping Postgres for an in-memory store is a
different argument at this one call site — which is exactly how the tests below work.

## 8.6 The tooling belt

Four tools should be part of a professional Rust project from the beginning:

- **`cargo fmt`** applies Rust's canonical format. Run `cargo fmt --check` in CI.
- **`cargo clippy`** detects non-idiomatic code and many real bug patterns. A strict CI
  command is `cargo clippy -- -D warnings`.
- **`cargo doc --open`** renders `///` comments into browsable API documentation. Code
  examples in documentation can be compiled and executed as doc-tests by `cargo test`.
- **`cargo audit`** checks the dependency graph against the Rust security advisory database.

## 8.7 Tests: built in, three kinds

`cargo test` runs three kinds of tests with zero framework decisions. **Unit tests** live
_inside the file they test_, in a module compiled only for testing:

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

Co-location makes tests the first documentation a reader meets; `#[cfg(test)]` makes them
cost zero in release builds. **Integration tests** live in a top-level `tests/` directory —
each file a separate binary seeing only your _public_ API. **Doc-tests** are §8.6's compiled
examples. The assertion macros are `assert!`, `assert_eq!`, and — for enums with data —
`matches!`, which you're about to need.

## 8.8 The payoff: testing the domain with a fifteen-line fake

Here chapters 5, 6, and 7 pay out together. `OrderService` depends on a trait _we_ defined,
so a test double is a small struct — no mocking framework, no database, no Docker:

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

This test exercises pure business logic without infrastructure. It is fast, deterministic,
and safe to run in parallel. `type Tx = ()` removes transaction mechanics from the fake,
while `fail_commit` makes a failure path easy to reproduce.

The `matches!` assertion checks both the error variant and its important fields. The test
therefore verifies the failure contract that an API or UI will depend on, not merely that
“some error” occurred.

Two closing calibrations. A modest band of **integration tests against real Postgres** (in a
container) still belongs in the store crate — the fake proves your logic, not your SQL. And
if testing ever feels hard in a Rust codebase, the fix is almost never a bigger mocking
framework; it's a **missing trait boundary** — go back to chapter 6.5.

## Summary

Modules organize code within a crate, while crates form dependency boundaries that the
compiler can enforce. Workspaces provide one lockfile, build cache, and shared dependency
policy. A layered crate layout can turn “the domain does not know the database” into a build
rule rather than a guideline.

Wire concrete implementations once at the composition root, and use formatting, linting,
documentation tests, and dependency audits as CI gates. Consumer-owned traits then make the
domain fast to test with small fakes and precise `matches!` assertions. The final chapter
assembles these ideas into one service design.
