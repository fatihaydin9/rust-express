# Chapter 9: Capstone: A Real Service

This chapter assembles the guide into one realistic design: a **versioned document store
with an audit trail**. Each earlier concept now carries part of the architecture. The final
sections collect the main principles, identify recurring anti-patterns, discuss when Rust
may be the wrong choice, and suggest where to continue.

## 9.1 Requirements

Users can create and update documents. Updates use _optimistic locking_, so a stale write
fails with enough context for the client to respond rather than overwriting newer data.
Every change is recorded in an immutable audit trail. A search index is maintained as a
derived view, never as the source of truth. Postgres and S3 provide storage, but **the core
logic must not depend on those technologies.**

## 9.2 Workspace (ch. 8)

```
docstore/
├── Cargo.toml               # workspace; [workspace.dependencies] = one version truth
└── crates/
    ├── domain/               # types, errors, traits, services — no sqlx, no aws-sdk
    ├── store-postgres/       # implements the repository trait
    ├── store-s3/             # implements the blob trait
    └── server/               # axum + composition root
```

Dependency arrows point inward, and the compiler patrols them: `domain` doesn't declare
`sqlx`, so SQL cannot leak into business logic.

## 9.3 Types (ch. 4)

Newtype IDs so arguments can't be swapped; a parsed name whose existence proves validity;
events as an enum carrying exactly what each variant needs:

```rust
pub struct DocumentId(Uuid);
pub struct UserId(Uuid);

// The private field prevents construction without validation.
pub struct DocumentName(String);

impl DocumentName {
    pub fn try_new(raw: &str) -> Result<Self, DomainError> {
        // Parse untrusted input into a valid domain value.
        /* ... */
    }
}

pub struct Document {
    pub id: DocumentId,
    pub name: DocumentName,
    pub version: i32,
    pub updated_by: UserId,
    pub updated_at: DateTime<Utc>,
}

pub enum DocumentEvent {
    Created {
        id: DocumentId,
        by: UserId,
        at: DateTime<Utc>,
    },
    Updated {
        id: DocumentId,
        from: i32,
        to: i32,
        by: UserId,
        at: DateTime<Utc>,
    },
}
```

## 9.4 Errors (ch. 5)

Variants carry what handlers need; the storage cause is wrapped whole, never flattened:

```rust
#[derive(Debug, thiserror::Error)]
pub enum DocumentError {
    #[error("document {0} not found")]
    NotFound(DocumentId),

    #[error("version conflict: expected v{expected}, found v{actual}")]
    VersionConflict {
        expected: i32,
        actual: i32,
        modified_by: UserId,
        modified_at: DateTime<Utc>,
    },

    // The underlying source chain remains available.
    #[error("storage failure")]
    Store(#[from] StoreError),
}
```

## 9.5 The port (ch. 6)

The consumer defines its contract — narrow, transaction-aware, thread-promising:

```rust
pub trait DocumentRepository: Send + Sync {
    type Tx: Send;

    async fn begin(&self) -> Result<Self::Tx, StoreError>;

    async fn load_for_update(
        &self,
        tx: &mut Self::Tx,
        id: DocumentId,
    ) -> Result<Option<Document>, StoreError>;

    async fn save_in_tx(
        &self,
        tx: &mut Self::Tx,
        document: &Document,
    ) -> Result<(), StoreError>;

    async fn append_event_in_tx(
        &self,
        tx: &mut Self::Tx,
        event: &DocumentEvent,
    ) -> Result<(), StoreError>;

    async fn commit(&self, tx: Self::Tx) -> Result<(), StoreError>;
}
```

## 9.6 The heart: `update`, annotated

```rust
impl<R: DocumentRepository> DocumentService<R> {
    pub async fn update(
        &self,
        command: UpdateDocument,
    ) -> Result<Document, DocumentError> {
        // ① Begin a transaction whose concrete type is adapter-specific.
        let mut tx = self.repo.begin().await?;

        // ② Convert absence into a typed domain error.
        let current = self
            .repo
            .load_for_update(&mut tx, command.id)
            .await?
            .ok_or(DocumentError::NotFound(command.id))?;

        // ③ Reject stale writes through optimistic locking.
        if current.version != command.expected_version {
            return Err(DocumentError::VersionConflict {
                expected: command.expected_version,
                actual: current.version,
                modified_by: current.updated_by,
                modified_at: current.updated_at,
            });
        }

        // ④ Apply pure domain logic.
        let next = current.apply(command)?;
        let event = DocumentEvent::updated(&current, &next);

        // ⑤–⑦ Persist state and audit event atomically.
        self.repo.save_in_tx(&mut tx, &next).await?;
        self.repo.append_event_in_tx(&mut tx, &event).await?;
        self.repo.commit(tx).await?;

        // ⑧ Update derived projections after the authoritative commit.
        self.events.broadcast(event);

        Ok(next)
    }
}
```

The numbered steps connect directly to earlier chapters:

1. **①** opens a transaction whose concrete type remains hidden from the domain.
2. **②** converts absence into a typed `NotFound` error through `ok_or` and `?`.
3. **③** performs optimistic locking and constructs the conflict error while the expected
   version, actual version, author, and timestamp are all available.
4. **④** applies pure domain logic to ordinary values, which keeps the rule easy to test.
5. **⑤–⑦** persist the new state and its audit event atomically. Because `commit(tx)`
   consumes `Tx`, a committed transaction cannot be reused.
6. **⑧** broadcasts the event to the search-index task. The index is a rebuildable
   projection of the audit stream rather than an authoritative store.

One operational note from the field: if the event log must be append-only, enforce it **in
the database** (revoke `UPDATE`/`DELETE` on the table). A comment saying "append-only" is a
wish, not a property.

## 9.7 The proof (ch. 8)

A small `FakeRepo` can use `type Tx = ()` and expose a `fail_commit` switch. Two focused
tests are then enough to pin the main contracts: the happy path persists both the document
and its event, while a stale update returns
`VersionConflict { expected: 2, actual: 3, .. }`, verified with `matches!`. No database is
required.

In the `server` crate, `main` passes `PostgresDocumentRepository` to the same generic
service. Switching between the test and production worlds changes one argument at the
composition root.

Step back and audit _who_ holds each requirement. ID mix-ups: newtypes. Invalid names: the
parsing constructor. Unhandled events: exhaustive `match`. Ignored failures: `Result` + `?`.
Double-commit: move semantics on `Tx`. Domain touching SQL: crate walls. Races in the
fan-out: `Send`/`Sync`. Every guarantee is held by the **compiler** somewhere. No single
feature did this — the accumulation is the language.

## 9.8 The principles, one card

| Habit                                                               | From  |
| ------------------------------------------------------------------- | ----- |
| Decide ownership first; borrow-checker fights are ambiguity reports | ch. 3 |
| Readers XOR one writer — the rule beneath everything                | ch. 3 |
| Accept `&str`, store `String`; owner/view pairs everywhere          | ch. 3 |
| Model states as enums; make illegal states unrepresentable          | ch. 4 |
| Newtype your IDs and parsed values — an alias protects nothing      | ch. 4 |
| Errors are designed data: variants carry what handlers need         | ch. 5 |
| Wrap causes with `#[from]`; never flatten chains to strings         | ch. 5 |
| `thiserror` where code consumes errors; `anyhow` at human edges     | ch. 5 |
| The consumer owns the trait; infrastructure implements it           | ch. 6 |
| Abstract the transaction; let moves enforce commit-once             | ch. 6 |
| Sharing is a choice, visible in types; channels for flow            | ch. 7 |
| Never block the runtime; never hold a std lock across `.await`      | ch. 7 |
| Crates are the architecture; the arrow rule lives in Cargo.toml     | ch. 8 |
| One composition root; injection = passing arguments                 | ch. 8 |
| Mock the trait, assert the variant — tests as contracts             | ch. 8 |

## 9.9 Anti-patterns: recognize on sight

Recognize these recurring anti-patterns:

- A fake newtype such as `type UserId = Uuid`, which changes the name but not the type.
- String-only errors that discard the underlying type and source chain.
- A provider-owned god-trait that forces every consumer to depend on a broad interface.
- A god-crate in which domain rules and SQL implementation details share one manifest.
- `Arc<Mutex<…>>` applied everywhere because ownership was never decided.
- Reflexive `.clone()` calls used only to silence the borrow checker. Deliberate clones are
  valid; unexamined clones accumulate design debt.
- Blocking operations inside async tasks.
- Business logic concentrated in HTTP handlers, beyond the reach of fast unit tests.
- `unsafe` blocks without a comment explaining the invariant they uphold.

All of these patterns can compile. Rust raises the minimum safety level, but engineering
discipline still determines whether the compiler supports the design or merely tolerates it.

## 9.10 When not to choose Rust

Rust justifies its learning and development cost when the problem benefits from predictable
latency without a garbage collector, a tight memory footprint, race-resistant concurrency,
small static binaries, WebAssembly or embedded targets, C interoperability, or
compiler-enforced invariants in a long-lived system.

It may be the wrong trade for throwaway scripts, rapid machine-learning experiments,
GUI-heavy applications, or a deadline-driven CRUD service maintained by a team already
highly productive in a garbage-collected language. Choose Rust because its guarantees match
concrete failure modes in the problem, and be able to name those guarantees.

## 9.11 Where to continue

Useful next resources include:

- _The Rust Programming Language_, the official book, for the syntax and language coverage
  this guide intentionally skips.
- **Rust by Example** for a more example-driven path.
- _Rust for Rustaceans_ by Jon Gjengset for deeper treatment of traits, API lifetimes, and
  async internals.
- The **Tokio tutorial** for production-oriented asynchronous Rust.
- The _Rustonomicon_ for understanding the guarantees that `unsafe` code must preserve. Read
  it first to understand the boundary, not merely to cross it.

Then build a system with a real architectural seam, such as a service that uses both a
database and a queue. Rust's design instincts are learned most effectively through repeated,
concrete feedback from the compiler.

## The set's last takeaway

Rust's deepest habit is **moving guarantees earlier**: from production incidents into tests,
from tests into types, and from types into compile-time ownership checks. Each chapter
illustrated that movement by encoding states as enums, failures as variants, interfaces as
consumer-owned traits, thread safety as bounds, and architectural rules as crate
dependencies.

Learn this design movement rather than memorizing syntax alone. The habit remains useful
even when you return to another language.
