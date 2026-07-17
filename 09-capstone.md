# Capstone project: A versioned document store

This chapter combines the concepts from previous chapters into a single, complete system design: a versioned document store with an audit log.

By examining this project, you will see how each Rust language feature fits into a clean, layered architecture. This chapter also summarizes key Rust design principles, identifies common anti-patterns, and outlines when to choose (or avoid) Rust for your projects.

## Project requirements

The capstone application is a document storage service with the following requirements:
* **Document management**: Users can create and update documents.
* **Optimistic locking**: Updates must use optimistic locking to prevent stale writes. If a write conflict occurs, the system returns detailed error context so the client can handle it instead of silently overwriting newer data.
* **Audit log**: Every change to a document must be recorded in an immutable, append-only audit log.
* **Search index**: The application maintains a search index as a derived, rebuildable projection of the audit log (not as the source of truth).
* **Storage abstraction**: The core business logic must remain independent of specific storage technologies. PostgreSQL and Amazon S3 provide the production storage implementations.

## Workspace structure

The project uses a Cargo workspace to separate core business logic from external adapter implementations:

```text
docstore/
├── Cargo.toml               # Workspace manifest and shared dependency versions
└── crates/
    ├── domain/              # Core logic: types, errors, traits, and services
    │                        # Dependencies: uuid, thiserror (no sqlx or aws-sdk)
    │
    ├── store-postgres/      # Implements domain repository traits using PostgreSQL
    │                        # Dependencies: domain, sqlx
    │
    ├── store-s3/            # Implements domain blob storage traits using Amazon S3
    │                        # Dependencies: domain, aws-sdk-s3
    │
    └── server/              # Application entry point (routing, configurations, startup)
                             # Dependencies: domain, store-postgres, store-s3, axum
```

Dependency arrows point inward toward the domain layer. Because the `domain` crate does not list database or cloud storage SDKs in its `Cargo.toml`, the compiler guarantees that infrastructure-specific code cannot enter your core business logic.

## Domain types

The application uses the newtype pattern to create distinct identifier types, a parsing constructor to validate document names, and an enum to represent audit events:

```rust
pub struct DocumentId(pub Uuid);
pub struct UserId(pub Uuid);

// The private inner field prevents direct instantiation.
pub struct DocumentName(String);

impl DocumentName {
    // This constructor serves as the validation boundary.
    pub fn try_new(raw: &str) -> Result<Self, DomainError> {
        if raw.is_empty() || raw.len() > 255 {
            Err(DomainError::InvalidName)
        } else {
            Ok(Self(raw.to_owned()))
        }
    }

    pub fn as_str(&self) -> &str {
        &self.0
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

## Error design

The domain layer uses a custom enum to represent precise failure states. Each variant carries the specific context required by its handlers:

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

    // Preserves the underlying storage error source chain.
    #[error("storage failure: {0}")]
    Store(#[from] StoreError),
}
```

## Storage abstraction with traits

The domain defines a narrow, transaction-aware trait interface. This trait uses the thread-safety bounds required by the service:

```rust
pub trait DocumentRepository: Send + Sync {
    // The associated type represents an implementation-specific transaction handle.
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

## Document service logic

The `DocumentService` coordinates document updates. It uses optimistic locking to prevent stale writes and ensures atomic persistence of document state and audit logs:

```rust
impl<R: DocumentRepository> DocumentService<R> {
    pub async fn update(
        &self,
        command: UpdateDocument,
    ) -> Result<Document, DocumentError> {
        // 1. Begin a transaction. The concrete transaction type is adapter-specific.
        let mut tx = self.repo.begin().await?;

        // 2. Load the document or return a NotFound error if it does not exist.
        let current = self
            .repo
            .load_for_update(&mut tx, command.id)
            .await?
            .ok_or(DocumentError::NotFound(command.id))?;

        // 3. Prevent stale writes using optimistic locking.
        if current.version != command.expected_version {
            return Err(DocumentError::VersionConflict {
                expected: command.expected_version,
                actual: current.version,
                modified_by: current.updated_by,
                modified_at: current.updated_at,
            });
        }

        // 4. Apply business calculations and generate the audit event.
        let next = current.apply(command)?;
        let event = DocumentEvent::updated(&current, &next);

        // 5. Persist the updated state and the audit event atomically in the transaction.
        self.repo.save_in_tx(&mut tx, &next).await?;
        self.repo.append_event_in_tx(&mut tx, &event).await?;
        
        // 6. Commit the transaction. This consumes the transaction handle `tx`.
        self.repo.commit(tx).await?;

        // 7. Broadcast the event to update the search index projection.
        self.events.broadcast(event);

        Ok(next)
    }
}
```

### Analysis of the update logic
* **Abstract transactions**: The service starts a transaction using `self.repo.begin()`. The domain does not know whether this uses a PostgreSQL connection, an S3 versioning handle, or an in-memory collection.
* **Commit-once protocol**: The `commit` method takes the transaction handle `tx` by value. Move semantics guarantee that you cannot accidentally reuse a transaction after it has been committed.
* **Consistent projections**: The service broadcasts the update event to the search index *only* after the database transaction has committed successfully. This prevents the search index from reflecting uncommitted database changes.

## Testing the service

By using traits to decouple external dependencies, you can test the entire update workflow using an in-memory fake.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::Mutex;

    #[derive(Default)]
    struct FakeRepo {
        documents: Mutex<Vec<Document>>,
        fail_commit: bool,
    }

    impl DocumentRepository for FakeRepo {
        // The fake repository does not need a real transaction handle.
        type Tx = ();

        async fn begin(&self) -> Result<Self::Tx, StoreError> {
            Ok(())
        }

        async fn load_for_update(
            &self,
            _tx: &mut Self::Tx,
            id: DocumentId,
        ) -> Result<Option<Document>, StoreError> {
            let docs = self.documents.lock().unwrap();
            Ok(docs.iter().find(|d| d.id == id).cloned())
        }

        async fn save_in_tx(
            &self,
            _tx: &mut Self::Tx,
            document: &Document,
        ) -> Result<(), StoreError> {
            let mut docs = self.documents.lock().unwrap();
            if let Some(pos) = docs.iter().position(|d| d.id == document.id) {
                docs[pos] = document.clone();
            } else {
                docs.push(document.clone());
            }
            Ok(())
        }

        async fn append_event_in_tx(
            &self,
            _tx: &mut Self::Tx,
            _event: &DocumentEvent,
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
    async fn version_conflict_returns_detailed_context() {
        let repo = FakeRepo::default();
        repo.documents.lock().unwrap().push(Document {
            id: DocumentId(Uuid::nil()),
            name: DocumentName::try_new("plan.md").unwrap(),
            version: 3,
            updated_by: UserId(Uuid::nil()),
            updated_at: Utc::now(),
        });

        let service = DocumentService::new(repo);
        let result = service.update(UpdateDocument {
            id: DocumentId(Uuid::nil()),
            expected_version: 2, // Stale version
            name: "new-plan.md".to_owned(),
        }).await;

        assert!(matches!(
            result,
            Err(DocumentError::VersionConflict { expected: 2, actual: 3, .. })
        ));
    }
}
```

This unit test is entirely self-contained and fast. To run the production application, you wire the generic `DocumentService` with `PostgresDocumentRepository` at your composition root. Swapping from testing mode to production mode requires changing only a single argument where you instantiate your service.

## Core design habits

The following table summarizes the core design guidelines covered throughout this guide:

| Category | Recommended Habit |
| :--- | :--- |
| **Memory** | Resolve ownership structure before handling borrow conflicts (Chapter 3). |
| **Memory** | Enforce either multiple shared read-only references or a single exclusive mutable reference (Chapter 3). |
| **Types** | Accept `&str` in parameters and store `String` in structs (Chapter 3). |
| **Types** | Represent states using enums to make invalid states unrepresentable (Chapter 4). |
| **Types** | Wrap raw identifiers or validated primitives in custom newtype structs (Chapter 4). |
| **Errors** | Model failures as enums where variants carry specific error context (Chapter 5). |
| **Errors** | Preserve original error causes using `#[from]` or `#[source]` attributes (Chapter 5). |
| **Architecture** | Define trait contracts next to the consumer. Infrastructure layer implements them (Chapter 6). |
| **Architecture** | Use associated types to hide adapter details and use move semantics to enforce once-only execution (Chapter 6). |
| **Concurrency** | Share state explicitly in your types. Use channels for data flow and locks for small shared states (Chapter 7). |
| **Concurrency** | Do not execute blocking calls in async workers or hold synchronous locks across `.await` boundaries (Chapter 7). |
| **Layout** | Enforce architectural boundaries using separate package-level crate dependencies (Chapter 8). |
| **Testing** | Use fakes to test business logic quickly and deterministicly in memory (Chapter 8). |

## Common anti-patterns

The following patterns can weaken otherwise clean architectural boundaries:
* **Fake newtypes**: Using type aliases (such as `type UserId = Uuid`) instead of wrapping primitives in tuple structs. Type aliases do not provide compile-time safety against swapped arguments.
* **String-only errors**: Discarding detailed error context and underlying causes by converting all failures to raw strings.
* **Overly broad traits**: Defining massive, provider-owned traits that force consumers to implement unused methods.
* **Leaky crates**: Placing domain rules and database adapter drivers in the same crate, allowing implementation details to mix with core rules.
* **Unbounded shared state**: Using `Arc<Mutex<T>>` throughout your entire application without a clear data ownership hierarchy.
* **Reflexive cloning**: Automatically calling `.clone()` whenever the compiler reports a borrow error instead of clarifying your ownership structure.
* **Blocking async threads**: Running long-running synchronous code inside async workers, blocking the runtime thread pool.
* **HTTP handler logic**: Putting complex business calculations directly inside web routing handlers where they are difficult to unit-test.
* **Document-less unsafe blocks**: Writing `unsafe` blocks without adding comments that document the physical safety invariants being upheld.

## Evaluate when to use Rust

Rust provides significant advantages for applications that require:
* Predictable, low-latency execution without a garbage collector.
* A minimal memory footprint.
* Thread-safe, highly concurrent systems.
* Small, self-contained executable binaries.
* WebAssembly or embedded platform targets.
* High-performance C library interoperability.
* Strong, compiler-enforced business invariants.

### When Rust may not be the optimal choice
Rust may not be the most effective choice for:
* Short utility scripts or throwaway automation tools.
* Quick machine-learning or data-science experiments.
* Rapid prototyping where requirements change constantly and compile-time boundaries add friction.
* Standard database CRUD services where a team is already highly productive in a garbage-collected language like Go, Java, or Node.js.

## Recommended learning resources

To continue your Rust learning path, consult the following resources:
* **The Rust Programming Language**: The official book covers fundamental Rust syntax and language features in detail.
* **Rust by Example**: Provides an interactive, example-driven approach to learning the language.
* **Rust for Rustaceans (by Jon Gjengset)**: Offers a deep dive into advanced traits, lifetime annotations, and async runtime internals.
* **The Tokio Tutorial**: Focuses on building production-grade, asynchronous applications in Rust.
* **The Rustonomicon**: Details the exact memory safety guarantees and rules that you must preserve when writing `unsafe` code.

### Final design movement
The core theme of Rust development is **moving guarantees earlier in the lifecycle**:
* Moving potential runtime issues into compile-time checks using types.
* Moving manual verification into automated unit tests using fakes.
* Moving architectural guidelines into compiler-enforced crate dependencies.

Understanding this design movement will help you write robust, maintainable code in Rust and any other language.
