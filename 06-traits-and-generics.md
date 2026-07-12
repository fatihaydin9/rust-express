# Chapter 6: Traits and Generics

Rust has no class inheritance. Its main abstraction mechanism is the **trait**. This
chapter begins with trait basics and ends with an architectural question that shapes
entire codebases: who should define an interface? Chapters 8 and 9 build directly on
that answer.

## 6.1 A trait is a promised capability

A trait declares operations a type can promise to support:

```rust
trait Describe {
    fn describe(&self) -> String;

    // Implementations may override this default method.
    fn describe_loudly(&self) -> String {
        self.describe().to_uppercase()
    }
}

struct City {
    name: String,
    population: u64,
}

struct Comet {
    name: String,
    period_years: f64,
}

impl Describe for City {
    fn describe(&self) -> String {
        format!("{} (pop. {})", self.name, self.population)
    }
}

impl Describe for Comet {
    fn describe(&self) -> String {
        format!("comet {} ({}y period)", self.name, self.period_years)
    }
}
```

The two unrelated types now share a capability without requiring a common base class.

> **If you come from Go:** traits play a role similar to interfaces, but implementations are
> written explicitly with forms such as `impl Describe for City`, which makes the
> relationship easy to search and navigate.
>
> **If you come from Java or C#:** traits resemble interfaces with default methods, but Rust
> does not use class inheritance. You can also implement a trait that you define for types
> from another crate, subject to the orphan rule described below.

One boundary to know: the **orphan rule**. You may implement your trait for any type, or
anyone's trait for your type — but not a foreign trait for a foreign type (otherwise two
crates could give `Vec<u8>` conflicting behaviors). The standard workaround is chapter
4's newtype: wrap the foreign type, implement on the wrapper.

## 6.2 Generics: "any type that can…"

A function can accept _any type that keeps a promise_:

```rust
fn announce<T: Describe>(item: &T) {
    println!("Now arriving: {}", item.describe());
}

announce(&city);
announce(&comet);

// Error: `i32` does not implement `Describe`.
announce(&42);
```

`<T: Describe>` is a **trait bound**. Multiple bounds combine with `+`, as in
`T: Describe + Clone`, while more complex constraints are usually moved to a `where`
clause.

The compiler normally monomorphizes generic code by generating a specialized version for
each concrete type used. This enables direct-call optimization and avoids the virtual
dispatch required by trait objects. The final performance still depends on the code and
should be measured when it matters.

## 6.3 `dyn`: dispatch decided at runtime

Sometimes the concrete type genuinely varies at runtime — a config file picks the
implementation, or one list holds several types. That's a **trait object**:

```rust
let items: Vec<Box<dyn Describe>> = vec![Box::new(city), Box::new(comet)];

for item in &items {
    // The concrete implementation is selected through a vtable at runtime.
    println!("{}", item.describe());
}
```

`dyn Trait` routes calls through a lookup table (like virtual methods elsewhere): one
compiled copy, a small per-call cost, more runtime flexibility. A useful default is to
use generics when the concrete type is known at compile time and `dyn Trait` when
implementations must vary at runtime or cross a plugin-like boundary. Neither form is
universally better; they make different trade-offs.

## 6.4 The traits you already use — and `From`/`Into`

Traits are used throughout the standard library, including in familiar operations:
`println!("{}")` calls `Display`, `{:?}` calls `Debug`, `==` calls `PartialEq`, `for`
calls `Iterator`, `+` calls `Add`, drop-cleanup calls `Drop`. The `#[derive(...)]`
attribute from §4.8 machine-writes the mechanical ones.

One pair deserves its own paragraph: **`From` and `Into`**, the standard vocabulary for
type conversion. Implement `From`, get `Into` for free:

```rust
impl From<Uuid> for UserId {
    fn from(value: Uuid) -> Self {
        UserId(value)
    }
}

let id: UserId = uuid.into();
let id = UserId::from(uuid); // Equivalent explicit form.
```

This is also the machinery behind two things you've met: `?` converts error types via
`From` (chapter 5's `#[from]` generates exactly this impl), and APIs accept flexible
inputs with `impl Into<String>` parameters. When you need a conversion that can _fail_,
the fallible twins are `TryFrom`/`TryInto`, returning `Result`.

## 6.5 The architectural question: who defines the interface?

The same trait mechanism also affects architecture. Suppose an `OrderService` needs to
persist orders. There are two common ways to place the storage abstraction, and they
produce different dependency directions.

**Path A:** the storage layer defines a broad `Repository` interface and services import
that abstraction. This can make business logic depend on infrastructure concepts and may
bring more database-related surface area into tests than the service actually needs.

**Path B (the pattern to learn): the consumer defines the trait.** The service declares,
right next to itself, the _minimal_ contract it needs; storage _implements_ it:

```rust
// Defined in the domain, next to `OrderService`.
// The consumer owns the interface.
pub trait OrderStore: Send + Sync {
    async fn insert(&self, order: &Order) -> Result<(), StoreError>;
    async fn find(&self, id: OrderId) -> Result<Option<Order>, StoreError>;
}

pub struct OrderService<S: OrderStore> {
    store: S,
}

impl<S: OrderStore> OrderService<S> {
    pub async fn place(&self, command: PlaceOrder) -> Result<Order, OrderError> {
        let order = Order::try_new(command)?;
        self.store.insert(&order).await?;
        Ok(order)
    }
}
```

This choice has three important consequences.

1. The **dependency arrow reverses**. The domain defines the contract, and the Postgres
   adapter depends on the domain to implement it. The domain can therefore compile
   without database dependencies.
2. The interface remains **narrow**. It contains the two operations this service needs
   rather than every operation a storage layer might expose. Several small
   consumer-specific traits are usually better than one broad repository interface.
3. **Testing becomes simple**. A small in-memory implementation of `OrderStore` can
   replace the database in unit tests. Chapter 8 develops this payoff in detail.

This is an example of _dependency inversion_. Rust can express it without a runtime
container: the generic parameter `S` receives the implementation at startup, and the
compiler checks that it satisfies the trait.

The `Send + Sync` bounds state that implementations must support the thread-sharing
pattern expected by a multithreaded server. The compiler verifies the requirement
structurally. Chapter 7 explains them fully.

## 6.6 Associated types: abstracting even the transaction

Transactions provide a more demanding example of a consumer-owned trait. The service
must write the order **and** its audit event — together or not at all. That's a database
transaction; but the domain must not know what a transaction concretely _is_. An
**associated type** lets the trait name the unknown — "each implementation will fill in
one concrete type here":

```rust
pub trait OrderRepository: Send + Sync {
    // Each implementation chooses its transaction-handle type.
    type Tx: Send;

    async fn begin(&self) -> Result<Self::Tx, StoreError>;

    async fn insert_in_tx(
        &self,
        tx: &mut Self::Tx,
        order: &Order,
    ) -> Result<(), StoreError>;

    async fn append_event_in_tx(
        &self,
        tx: &mut Self::Tx,
        event: &OrderEvent,
    ) -> Result<(), StoreError>;

    async fn commit(&self, tx: Self::Tx) -> Result<(), StoreError>;
}
```

The service can now coordinate an atomic operation without knowing the concrete
transaction type:

```rust
pub async fn place(&self, command: PlaceOrder) -> Result<Order, OrderError> {
    let order = Order::try_new(command)?;
    let event = OrderEvent::placed(&order);

    let mut tx = self.repo.begin().await?;
    self.repo.insert_in_tx(&mut tx, &order).await?;
    self.repo.append_event_in_tx(&mut tx, &event).await?;

    // State and event commit as one atomic unit.
    self.repo.commit(tx).await?;

    Ok(order)
}
```

Two details are worth studying. A Postgres implementation may define
`type Tx = sqlx::Transaction<…>`, while a test double can use `type Tx = ()`. Even the
transaction representation is replaceable.

`commit(tx)` also consumes the transaction by value. Ownership moves into the call, so
committed transactions cannot be reused. Consuming the transaction expresses a
commit-at-most-once protocol in the type system without requiring a separate runtime
flag. This small example combines moves, rich types, `?`, and associated types in one
design.

(Associated type vs. another generic parameter: if each implementation has exactly
**one** natural answer — a repository has one transaction type — use an associated type.
If one implementation should work for many types, use a generic parameter.)

## Summary

Traits define explicit capabilities. Generics provide static dispatch, `dyn Trait`
provides runtime dispatch, and `From`/`Into` establish a shared conversion vocabulary.

The architectural principle to carry forward is interface ownership: **the consumer
defines the smallest trait it needs, and infrastructure implements it.** Associated
types can hide implementation-specific concepts such as transactions, while ownership
can enforce protocols such as commit-once. The next chapter explains the `Send + Sync`
bounds and Rust's concurrency model.
