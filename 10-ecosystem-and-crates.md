# Chapter 10 — The Ecosystem: Recommended Crates

Rust's standard library is intentionally small. It does not include an HTTP client, an async runtime, serialization formats, or random-number generation. The broader ecosystem fills those gaps, and in many categories the community has converged on a de facto standard.

This chapter provides a practical map of that ecosystem: the crates an experienced Rust developer is likely to consider first, together with criteria for evaluating crates that are not on the list.

A note on volatility: these recommendations reflect the ecosystem in early 2026. Most have been stable choices for years, but ecosystems evolve. The categories and evaluation criteria in §10.9 will age better than any individual crate name.

## 10.1 The core stack

| Category | Crate | Why it is a common default |
|---|---|---|
| Async runtime | **tokio** | The dominant runtime; most async libraries integrate with it |
| Serialization | **serde** with `serde_json` | One derive-based framework that supports many formats |
| Web framework | **axum** | Composable, type-driven, and built by the Tokio team |
| HTTP client | **reqwest** | An ergonomic async client built on the same ecosystem |
| Database | **sqlx** | Async, SQL-first access with optional compile-time query checking |
| Errors | **thiserror** and **anyhow** | Typed errors for libraries; flexible context at application boundaries |
| Diagnostics | **tracing** with `tracing-subscriber` | Structured, async-aware events and spans |
| CLI parsing | **clap** in derive mode | Declarative arguments, validation, and generated help text |
| Date and time | **chrono** | Widely used date, time, and timezone types |
| Identifiers | **uuid** | Standard UUID support with optional `serde` and version features |

If you remember only one part of this chapter, remember this table. It covers most of the dependencies used in ordinary Rust backend work.

## 10.2 `serde`: the crate almost every project uses

Serde is important enough to deserve its own section. A pair of derives can make a type serializable and deserializable across JSON, TOML, YAML, MessagePack, and many other formats through companion crates.

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize, Serialize)]
struct Order {
    id: u64,
    customer: String,

    // Missing input uses the field's Default value.
    #[serde(default)]
    notes: Option<String>,

    // Keep Rust naming internally while matching the external format.
    #[serde(rename = "amountCents")]
    amount_cents: u64,
}

let order: Order = serde_json::from_str(payload)?;
let json = serde_json::to_string_pretty(&order)?;
```

The design fits Rust's broader philosophy. `from_str` returns `Result<Order, _>`, so malformed input never becomes a partially valid `Order`. Serde's attributes—such as `rename_all = "camelCase"`, `deny_unknown_fields`, and `skip_serializing_if`—handle most wire-format differences declaratively.

## 10.3 `tracing`: diagnostics that understand async work

`println!`-based debugging becomes difficult as soon as multiple tasks interleave. The `tracing` crate adds structured events and **spans**. A span represents a named period of work, and nested events retain the context of the request or operation that produced them.

```rust
use tracing::{info, instrument};

#[instrument(skip(repo), fields(document_id = %id))]
async fn update_document(repo: &Repo, id: DocumentId) {
    info!(version = 3, "loaded current document");
}
```

A corresponding log event can now include the function span, the document identifier, and structured fields such as `version`.

Configure the subscriber once in `main`:

```rust
tracing_subscriber::fmt()
    .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
    .init();
```

You can then control verbosity through the `RUST_LOG` environment variable. Production systems can switch to JSON output without changing every call site. Introduce spans early; retrofitting request context after a service grows is far more difficult.

## 10.4 The web pair: `axum` and `sqlx`

The following handler shows the basic shape of an axum endpoint:

```rust
async fn get_document(
    State(service): State<DocumentService<PgRepo>>,
    Path(id): Path<Uuid>,
) -> Result<Json<Document>, ApiError> {
    let document = service.get(DocumentId::from(id)).await?;
    Ok(Json(document))
}

let app = Router::new()
    .route("/documents/:id", get(get_document))
    .with_state(service);
```

Extractors such as `State`, `Path`, and `Json` are ordinary types in the function signature. There is no reflection-based wiring: if the handler compiles, its inputs and outputs satisfy the framework's contracts.

SQLx can optionally check SQL against a real schema at compile time:

```rust
let document = sqlx::query_as!(
    DocumentRow,
    "SELECT id, name, version FROM documents WHERE id = $1",
    id,
)
.fetch_optional(&pool)
.await?;
```

A misspelled column can therefore become a build error rather than a production failure. The result is also explicit: `fetch_optional` returns `Option<DocumentRow>` when no row is found.

These are defaults, not the only valid choices. `actix-web` is an older and production-proven web framework, while `diesel` and `sea-orm` are reasonable options when you prefer query builders or ORM-style abstractions.

## 10.5 CLI applications with `clap`

```rust
use clap::Parser;
use std::path::PathBuf;

#[derive(Parser)]
#[command(version, about = "Sync documents to the archive")]
struct Args {
    /// Directory containing the source documents.
    path: PathBuf,

    /// Print planned actions without executing them.
    #[arg(long)]
    dry_run: bool,
}

let args = Args::parse();
```

Documentation comments become command-line help text. `clap` also generates `--help`, `--version`, validation errors, and usage output. For more polished terminal applications, consider `indicatif` for progress bars and `owo-colors` for styled output.

## 10.6 Testing tools beyond the standard library

Rust's built-in test framework covers most routine needs, but several additions are especially useful:

- **`proptest`** provides property-based testing. You state an invariant—such as `parse(render(x)) == x`—and the library generates many inputs. When it finds a failure, it attempts to shrink the input to a minimal counterexample. This is particularly valuable for parsers, state machines, and event/projection invariants.
- **`insta`** provides snapshot testing. It compares a value with a reviewed snapshot committed to the repository, which works well for rendered output and API responses.
- **`criterion`** provides statistically informed benchmarks and is the usual answer to “is this change actually faster?”
- **`mockall`** generates trait mocks. Use it when hand-written fakes become cumbersome, but first consider whether the mocked trait has grown too broad.

## 10.7 Concurrency and data utilities

Several smaller crates repeatedly appear in production code:

- **`rayon`** turns sequential iterator pipelines into data-parallel work, often by replacing `.iter()` with `.par_iter()`.
- **`dashmap`** provides a concurrent map for cases where `Arc<RwLock<HashMap<_, _>>>` becomes a bottleneck.
- **`crossbeam`** offers richer channels and scoped-thread utilities.
- **`itertools`** adds iterator adapters that are not present in the standard library, including `chunks`, `sorted_by_key`, and `join`.
- **`regex`** provides a linear-time regular-expression engine that avoids catastrophic backtracking.

Many older `once_cell` use cases can now be expressed with standard-library types such as `std::sync::LazyLock`.

## 10.8 Supporting crates by task

For layered configuration from files and environment variables, consider **`figment`** or **`config`**; for `.env` files, use **`dotenvy`**. For gRPC, the usual choice is **`tonic`**. Axum includes WebSocket support, while **`object_store`** provides a uniform API for S3, Google Cloud Storage, Azure, and local storage. The official **`aws-sdk-s3`** crate is appropriate when you need AWS-specific capabilities.

For metrics, use **`metrics`** with a suitable exporter such as Prometheus. For compile-time-checked HTML templates, consider **`askama`**. Common security-related crates include **`argon2`** for password hashing, **`jsonwebtoken`** for JWTs, and **`zeroize`** for clearing sensitive values from memory.

Cross-format configuration parsing usually returns to the same foundation: Serde combined with crates such as `toml` or a YAML implementation.

## 10.9 How to evaluate an unfamiliar crate

When the map ends, use the following criteria as a compass.

First, inspect release activity. A year without a release may be concerning for foundational infrastructure, but it can be perfectly acceptable for a small, finished utility. Next, look at download trends and reverse dependencies. A crate used by major ecosystem projects carries a stronger maintenance signal than raw download counts alone.

Then inspect its documentation on docs.rs. Strong crates usually provide useful examples on the front page. Poor documentation creates an ongoing maintenance cost, even when the implementation itself is sound. Read the README for comparisons, trade-offs, and explicit non-goals; honest limitations are often a positive sign.

Finally, consider structure and policy. Prefer crates with a manageable number of transitive dependencies because every dependency expands compile time and supply-chain exposure. Use `cargo tree` to inspect the graph and `cargo deny` to enforce license and advisory policies. The community-maintained list at **blessed.rs** is also a useful cross-check.

> **Common mistake:** Do not search for a single batteries-included equivalent of Rails or Spring. Rust projects usually compose focused crates: axum handles routing, sqlx handles data access, and tracing handles diagnostics. This requires more explicit assembly, but it also leaves far less hidden behavior to debug.

## Summary

A strong default backend stack is Tokio, Serde, axum, sqlx, reqwest, thiserror or anyhow, tracing, clap, chrono, and uuid. Serde and tracing are especially useful from the first day of a project.

Outside that map, evaluate crates by their maintenance activity, reverse dependencies, documentation, transitive dependency weight, and licensing. Rust's ecosystem favors composing small, focused crates rather than waiting for one framework to own the entire application.
