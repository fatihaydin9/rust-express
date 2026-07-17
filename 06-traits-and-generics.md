# Traits and generics

Rust does not use class inheritance. Instead, the primary tool for abstraction in Rust is the **trait**. A trait defines a set of methods that a type must implement to provide a specific capability.

By the end of this chapter, you will understand:
* How to define and implement traits to share behavior across different types.
* How to use generics and trait bounds for static dispatch.
* How to use trait objects (`dyn Trait`) for dynamic dispatch.
* How to use common standard library traits like `From` and `Into`.
* How to design your application architecture using consumer-defined traits.
* How to abstract complex implementation details (such as database transactions) using associated types.

## Shared behavior with traits

A trait declares operations a type must support:

```rust
trait Describe {
    fn describe(&self) -> String;

    // This is a default method. Implementing types can use this default
    // implementation or override it with their own behavior.
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

Using traits, unrelated types like `City` and `Comet` can share the same behavior without inheriting from a common parent class.

> **If you come from Go:**
> Traits work similarly to interfaces in Go. However, in Rust, you must implement a trait explicitly using the `impl Trait for Type` syntax. This makes it easier to navigate your codebase and find implementations.
>
> **If you come from Java or C#:**
> Traits are similar to interfaces with default methods. However, because Rust does not support class inheritance, you use composition and traits to structure your application.

### The orphan rule
To maintain consistency across libraries, Rust enforces the **orphan rule**:
* You can implement any trait on your custom types.
* You can implement your custom traits on any external types (such as types from the standard library or another crate).
* You **cannot** implement an external trait on an external type. For example, you cannot implement a trait from a third-party crate on the standard library `Vec` type.

To work around this limitation, use the newtype pattern (see Chapter 4) to wrap the external type in a local struct, then implement the trait on the wrapper.

## Generics and trait bounds

Generics allow you to write functions and data structures that work with multiple types. You can use **trait bounds** to specify that a generic parameter must support a particular trait.

```rust
fn announce<T: Describe>(item: &T) {
    println!("Now arriving: {}", item.describe());
}
```

In this example, `<T: Describe>` is a trait bound. It specifies that the `announce` function accepts any type `T` that implements the `Describe` trait.

### Advanced trait bounds
* **Multiple bounds**: You can combine multiple trait bounds using the `+` operator. For example, `T: Describe + Clone`.
* **Where clauses**: For complex bounds, use a `where` clause to make your function signature easier to read:
  ```rust
  fn process<T, U>(item: &T, config: &U)
  where
      T: Describe + Clone,
      U: Debug,
  {
      // ...
  }
  ```

### Monomorphization
When you compile your program, Rust uses a process called **monomorphization** for generics. The compiler analyzes your code and generates a copy of the generic function for each concrete type used. 

This results in **static dispatch**. Because the compiler knows the exact types at compile time, it can optimize calls directly, avoiding runtime lookup overhead. This provides maximum performance but can increase binary size.

## Dynamic dispatch with trait objects

If you need to work with multiple types that implement the same trait at runtime, you must use a **trait object**. You create a trait object using the `dyn` keyword, usually wrapped in a pointer type such as `Box` or a reference.

```rust
let items: Vec<Box<dyn Describe>> = vec![
    Box::new(city),
    Box::new(comet),
];

for item in &items {
    // The program selects the correct implementation at runtime.
    println!("{}", item.describe());
}
```

### Static vs. dynamic dispatch
The following table compares static dispatch (generics) and dynamic dispatch (trait objects):

| Feature | Static dispatch (Generics) | Dynamic dispatch (Trait objects) |
| :--- | :--- | :--- |
| **How it works** | The compiler generates duplicate code for each concrete type. | The program uses a pointer lookup table (**vtable**) at runtime. |
| **Compilation** | Direct function calls; supports aggressive compiler optimizations. | Indirect function calls; slightly slower per-call performance. |
| **Flexibility** | Types must be resolved at compile time. | Allows storing different types in the same collection at runtime. |
| **Syntax** | `<T: Describe>` | `&dyn Describe` or `Box<dyn Describe>` |

## Standard library traits

Rust uses traits to define many standard operations:
* **`Display`**: Formats types for user-facing output (`println!("{}", item)`).
* **`Debug`**: Formats types for programmer-facing diagnostics (`println!("{:?}", item)`).
* **`PartialEq` / `Eq`**: Implements equality comparisons (`==` and `!=`).
* **`Iterator`**: Defines sequences that can be used in `for` loops.
* **`Drop`**: Defines clean-up logic executed when a value goes out of scope.

### Type conversions with `From` and `Into`
The `From` and `Into` traits define how to convert one type into another safely. If you implement the `From` trait for your type, Rust automatically provides the corresponding `Into` implementation.

```rust
impl From<Uuid> for UserId {
    fn from(value: Uuid) -> Self {
        UserId(value)
    }
}

// You can perform the conversion using either method:
let id: UserId = uuid.into();
let id = UserId::from(uuid);
```

For conversions that can fail, use the `TryFrom` and `TryInto` traits, which return a `Result` type.

## Interface design: consumer-defined traits

When you design interfaces to abstract external dependencies (such as a database), you can choose who defines the interface. In Rust, the recommended pattern is **dependency inversion**, where the consumer defines the trait.

### Path A: Infrastructure-defined interfaces (Not recommended)
In this approach, the database or storage layer defines a broad interface (such as a `Repository`), and your business logic services import this interface. 

This model makes your core domain logic depend on infrastructure concepts and can introduce unnecessary database details into your domain tests.

### Path B: Consumer-defined interfaces (Recommended)
In this approach, the business logic service defines the exact, minimal interface it needs. The storage layer then implements this interface.

```rust
// Defined in your domain layer, next to your OrderService.
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

This design provides three major benefits:
1. **Reversed dependency direction**: Your domain logic defines the interface, and your database adapter depends on the domain to implement it. This allows your domain logic to compile without database dependencies.
2. **Narrow interfaces**: The trait contains only the specific operations required by the service, rather than every possible database operation. Small, service-specific traits are easier to maintain than a single broad interface.
3. **Simplified testing**: You can easily create an in-memory test double of the `OrderStore` trait for your unit tests without needing a running database.

## Abstracting transactions with associated types

Sometimes, your business logic must coordinate multiple actions in a single atomic transaction. However, the domain logic should not know the database-specific implementation details of a transaction.

You can use **associated types** to solve this problem. Associated types allow a trait to declare a placeholder type that each implementation defines.

```rust
pub trait OrderRepository: Send + Sync {
    // Each database implementation defines its own concrete transaction type.
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

Your service can now coordinate transactional operations without knowing the underlying implementation details:

```rust
pub async fn place(&self, command: PlaceOrder) -> Result<Order, OrderError> {
    let order = Order::try_new(command)?;
    let event = OrderEvent::placed(&order);

    let mut tx = self.repo.begin().await?;
    self.repo.insert_in_tx(&mut tx, &order).await?;
    self.repo.append_event_in_tx(&mut tx, &event).await?;

    // The order and the event commit as an atomic unit.
    self.repo.commit(tx).await?;

    Ok(order)
}
```

### Key design details
* **Replaceable transaction handles**: A PostgreSQL implementation can define `type Tx = sqlx::Transaction<...>`, while a test double can use a unit type `type Tx = ()`.
* **Compile-time safety**: The `commit` method consumes the transaction parameter by value (`tx` instead of `&mut tx`). Because ownership moves into the function, you cannot reuse a transaction after it has been committed. This prevents double-commit errors at compile time.

### Associated types vs. generic parameters
* Use **associated types** when an implementation of a trait has exactly one corresponding type (for example, a repository has exactly one transaction type).
* Use **generic parameters** when a single implementation of a trait must support multiple types.

## Summary

This chapter discussed traits and generics in Rust:
* **Traits** define shared capabilities across different types.
* **Generics** provide static dispatch for maximum performance, while **trait objects** (`dyn Trait`) provide dynamic dispatch for runtime flexibility.
* **Standard vocabulary traits** (such as `From` and `Into`) establish standard conversion rules.
* **Consumer-defined traits** invert architectural dependencies, keeping your domain logic clean and decoupled from infrastructure details.
* **Associated types** let you abstract complex concepts like transactions while maintaining compile-time safety rules.
