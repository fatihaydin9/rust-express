# Chapter 9: Capstone: A Real Service

This chapter assembles the guide into one realistic design: a **versioned document store
with an audit trail**. Each earlier concept now carries part of the architecture. The
final sections collect the main principles, identify recurring anti-patterns, discuss
when Rust may be the wrong choice, and suggest where to continue.

## 9.1 Requirements

Users can create and update documents. Updates use _optimistic locking_, so a stale
write returns enough context for the client to respond instead of silently overwriting
newer data. Each change is recorded in an immutable audit trail. A search index is
maintained as a derived, rebuildable view rather than the source of truth. Postgres and
S3 provide the storage implementations, while the core logic remains independent of
those technologies.

## 9.2 Workspace (ch. 8)

```
docstore/
├── Cargo.toml               # workspace and shared dependency versions
└── crates/
    ├── domain/               # types, errors, traits, services — no sqlx, no aws-sdk
    ├── store-postgres/       # implements the repository trait
    ├── store-s3/             # implements the blob trait
    └── server/               # axum + composition root
```

Dependency arrows point inward. Because the `domain` crate does not declare `sqlx`,
SQL-specific code cannot be added there without changing its manifest and dependency
boundary.

## 9.3 Types (ch. 4)

The domain uses newtype identifiers to distinguish arguments, a parsed name type to
represent validated input, and an enum whose event variants carry the data they need:

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

Each variant carries the context its handlers need, and the storage error is preserved
as the underlying cause:

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

The consumer defines a focused, transaction-aware contract with the thread-safety bounds
required by the service:

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
3. **③** performs optimistic locking and constructs the conflict error while the
   expected version, actual version, author, and timestamp are all available.
4. **④** applies pure domain logic to ordinary values, which keeps the rule easy to
   test.
5. **⑤–⑦** persist the new state and its audit event atomically. Because `commit(tx)`
   consumes `Tx`, a committed transaction cannot be reused.
6. **⑧** broadcasts the event to the search-index task. The index is a rebuildable
   projection of the audit stream rather than an authoritative store.

If the event log must be append-only, enforce that requirement in the database as well
as in application code, for example by restricting `UPDATE` and `DELETE` permissions. A
comment alone does not protect the invariant.

## 9.7 The proof (ch. 8)

A small `FakeRepo` can use `type Tx = ()` and expose a `fail_commit` switch. Two focused
tests are then enough to pin the main contracts: the happy path persists both the
document and its event, while a stale update returns
`VersionConflict { expected: 2, actual: 3, .. }`, verified with `matches!`. No database is
required.

In the `server` crate, `main` passes `PostgresDocumentRepository` to the same generic
service. Switching between the test and production worlds changes one argument at the
composition root.

The design can now be reviewed by asking where each requirement is enforced. ID mix-ups:
newtypes. Invalid names: the parsing constructor. Unhandled events: exhaustive `match`.
Ignored failures: `Result` + `?`. Double-commit: move semantics on `Tx`. Domain touching
SQL: crate walls. Races in the fan-out: `Send`/`Sync`. Many of these guarantees are
represented in types or crate dependencies and are therefore checked by the compiler.
The result comes from several language features working together.

## 9.8 Principles at a glance

| Habit                                                               | From  |
| ------------------------------------------------------------------- | ----- |
| Clarify ownership before resolving borrow conflicts                | ch. 3 |
| Use shared readers or one exclusive writer                         | ch. 3 |
| Accept `&str`, store `String`; owner/view pairs everywhere          | ch. 3 |
| Model states as enums; make illegal states unrepresentable          | ch. 4 |
| Use newtypes when identifiers or validated values need distinction  | ch. 4 |
| Errors are designed data: variants carry what handlers need         | ch. 5 |
| Preserve underlying causes with `#[from]` or `#[source]`            | ch. 5 |
| `thiserror` where code consumes errors; `anyhow` at human edges     | ch. 5 |
| The consumer owns the trait; infrastructure implements it           | ch. 6 |
| Abstract the transaction; let moves enforce commit-once             | ch. 6 |
| Sharing is a choice, visible in types; channels for flow            | ch. 7 |
| Avoid blocking the runtime or holding a std lock across `.await`    | ch. 7 |
| Use crate dependencies to enforce important architectural arrows    | ch. 8 |
| Keep concrete wiring in one composition root                        | ch. 8 |
| Mock the trait, assert the variant — tests as contracts             | ch. 8 |

## 9.9 Recurring anti-patterns

The following patterns deserve review because they often weaken otherwise useful
boundaries:

- A fake newtype such as `type UserId = Uuid`, which changes the name but not the type.
- String-only errors that discard the underlying type and source chain.
- A provider-owned, overly broad trait that gives every consumer more operations than it
  needs.
- An overly broad crate in which domain rules and SQL implementation details share the
  same dependency boundary.
- `Arc<Mutex<…>>` used throughout the system without a clear ownership boundary.
- Reflexive `.clone()` calls used only to silence the borrow checker. Deliberate clones
  are valid; unexamined clones accumulate design debt.
- Blocking operations inside async tasks.
- Business logic concentrated in HTTP handlers, beyond the reach of fast unit tests.
- `unsafe` blocks without a comment explaining the invariant they uphold.

All of these patterns can compile. Rust provides a stronger safety baseline, but the
quality of the architecture still depends on the decisions expressed through types,
modules, and dependencies.

## 9.10 When not to choose Rust

Rust justifies its learning and development cost when the problem benefits from
predictable latency without a garbage collector, a tight memory footprint,
race-resistant concurrency, small static binaries, WebAssembly or embedded targets, C
interoperability, or compiler-enforced invariants in a long-lived system.

It may be the wrong trade for throwaway scripts, rapid machine-learning experiments,
GUI-heavy applications, or a deadline-driven CRUD service maintained by a team already
highly productive in a garbage-collected language. Choose Rust because its guarantees
match concrete failure modes in the problem, and be able to name those guarantees.

## 9.11 Where to continue

Useful next resources include:

- _The Rust Programming Language_, the official book, for the syntax and language
  coverage this guide intentionally skips.
- **Rust by Example** for a more example-driven path.
- _Rust for Rustaceans_ by Jon Gjengset for deeper treatment of traits, API lifetimes,
  and async internals.
- The **Tokio tutorial** for production-oriented asynchronous Rust.
- The _Rustonomicon_ for understanding the guarantees that `unsafe` code must preserve.
  It is most useful for learning the boundary before writing unsafe code.

Then build a system with a real architectural seam, such as a service that uses both a
database and a queue. Rust's design instincts are learned most effectively through
repeated, concrete feedback from the compiler.

## Final takeaway

A recurring Rust design habit is **moving guarantees earlier**: from production
incidents into tests, from tests into types, and from types into compile-time ownership
checks. Each chapter illustrated that movement by encoding states as enums, failures as
variants, interfaces as consumer-owned traits, thread safety as bounds, and
architectural rules as crate dependencies.

Learn this design movement rather than memorizing syntax alone. The habit remains useful
even when you return to another language.
