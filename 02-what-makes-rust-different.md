# What makes Rust different

This chapter provides an overview of the primary differences between Rust and other programming languages. It introduces ten key concepts and points to the chapters that discuss them in detail.

Understanding these concepts before you write code will help you avoid common compilation errors.

## Ownership instead of a garbage collector

Most modern memory-safe programming languages use a runtime garbage collector to reclaim unused memory. Rust does not use a garbage collector. Instead, it uses a set of ownership and borrowing rules that the compiler checks before your program runs.

When a variable that owns a value goes out of scope, Rust automatically runs that value's destructor and releases its resources.

This system provides several benefits:
* **Deterministic cleanup**: Rust releases resources like memory, files, and locks immediately at predictable points.
* **No runtime pauses**: Because there is no garbage collector running in the background, your program does not experience unexpected pauses.

However, you must understand how ownership works. For example, assigning a value to a new variable moves ownership by default for many types, making the original variable invalid:

```rust
let a = String::from("hi");
let b = a; // Ownership moves to `b`.

// println!("{a}"); // Compiler error: `a` is no longer valid.
```

For more details about this model, see Chapter 3.

## Immutable by default

By default, variables in Rust are immutable. This means you cannot change their value after they are bound.

```rust
let x = 5;
// x = 6; // Compiler error: cannot assign twice to immutable variable.
```

If you want to modify a variable, you must explicitly use the `mut` keyword:

```rust
let mut x = 5;
x = 6; // This is valid.
```

This design encourages you to declare your intentions explicitly. Making mutation explicit makes code reviews easier and helps prevent unexpected state changes.

## No null values

Safe Rust does not have a universal `null`, `nil`, or `undefined` value. If a variable is of type `String`, it is guaranteed to contain a valid string.

When a value can be absent, you must represent it using the standard library `Option<T>` enum. An `Option` is either:
* **`Some(value)`**: Contains a value of type `T`.
* **`None`**: Represents the absence of a value.

```rust
fn find_user(name: &str) -> Option<User> {
    // Returns Some(User) if found, or None if not.
}

if let Some(user) = find_user("ada") {
    greet(&user);
}
```

The compiler forces you to handle the `None` case before you can access the inner value. This prevents null pointer exceptions at runtime. For more details, see Chapter 4.

> **If you come from Java, C#, or JavaScript:**
> Instead of using null checks, you use Rust's type system to represent optional values. This is similar to strict null checks in TypeScript or nullable types in Kotlin.

## Result-based error handling

Rust does not use exceptions for standard, recoverable errors. Instead, a function that can fail returns a `Result<T, E>` enum, which is either:
* **`Ok(value)`**: Represents a successful operation.
* **`Err(error)`**: Represents a failed operation.

Because the potential failure is part of the function signature, you must handle the error explicitly.

To keep your code readable, you can use the `?` operator to propagate errors to the calling function:

```rust
fn load(path: &str) -> Result<Config, ConfigError> {
    // The `?` operator returns the error immediately if `read_to_string` fails.
    let text = std::fs::read_to_string(path)?;
    parse(&text)
}
```

For unrecoverable errors (such as a broken program assumption), Rust uses panics. For more details on error handling and designing custom error types, see Chapter 5.

## Composition over class inheritance

Rust does not have classes, class hierarchies, or an `extends` keyword. Instead, Rust uses:
* **Structs**: To define the data and fields of a type.
* **Implementations (`impl` blocks)**: To define the methods for a type.
* **Traits**: To define shared behavior across different types.

Instead of asking what class a type should inherit from, you define what behaviors the type must implement. You can then use traits to achieve polymorphism through static dispatch (resolved at compile time) or dynamic dispatch (resolved at runtime). For more details, see Chapter 6.

## Explicit type conversions

Rust does not perform implicit numeric conversions, and conditions in control flows must evaluate to a boolean (`bool`) type.

```rust
let a: i32 = 5;

// let b: i64 = a; // Compiler error: mismatched types.
let b: i64 = a.into(); // Lossless conversion.
let c = a as f64; // Explicit cast.

// if a { } // Compiler error: expected `bool`, found `i32`.
if a != 0 {
    // This is valid because the condition evaluates to a boolean.
}
```

Rust does support automatic conversions in specific, safe scenarios (such as coercing a `&String` reference to a `&str` reference), but it avoids broad implicit conversions to prevent information loss or ambiguity. For more details, see Chapter 6.

## Expression-based syntax

In Rust, many syntax constructs are expressions that evaluate to a value. This includes `if` blocks, `match` blocks, loops that use `break` with a value, and basic code blocks `{}`.

This expression-based design reduces the need for temporary mutable variables and replaces the need for a separate ternary operator.

```rust
let access = match role {
    Role::Admin => Access::Full,
    Role::Member if verified => Access::Standard,
    _ => Access::ReadOnly,
};
```

Additionally, the `match` operator requires you to handle every possible case. If you miss a case, the compiler generates an error. For more details, see Chapter 4.

## Two main string types

Rust has two primary string types:
* **`String`**: An owned, growable, and heap-allocated UTF-8 text container.
* **`&str`**: A borrowed reference (or slice) to a UTF-8 string owned by another variable.

String literals (such as `"hello"`) have the type `&'static str` because they are stored directly in the compiled binary.

### Recommended guideline
A common practice is to store `String` in structs or long-lived variables, and accept `&str` in function parameters. A reference to a `String` (`&String`) automatically converts to `&str`. 

* To convert a `&str` to a `String`, use `.to_string()` or `.to_owned()`.
* To convert a `String` to a `&str`, borrow it using `&s` or `.as_str()`.

For more details, see Chapter 3.

## Safe concurrency

A data race occurs when two or more threads access the same memory location concurrently, at least one of the accesses is a write, and there is no synchronization. Data races are notoriously difficult to debug because they cause intermittent errors.

Safe Rust prevents data races at compile time. The compiler uses the ownership and borrowing rules, along with two core traits, to ensure safe multi-threaded code:
* **`Send`**: Indicates that a type can transfer ownership safely across thread boundaries.
* **`Sync`**: Indicates that multiple threads can access a type safely through shared references.

To share mutable data across threads safely, you must explicitly use synchronization types such as `Mutex`, channels, or atomic types. For more details, see Chapter 7.

## Compile-time design feedback

Rust's strict rules shift your development effort to the beginning of the design process. The compiler asks you to answer critical questions early:
* Which variable owns this data?
* Can this operation fail?
* Is it possible for this value to be absent?
* Is this data shared or mutated across threads?

Answering these questions can slow down your initial prototyping. However, once your program compiles, the type system has already eliminated a wide category of common bugs:
* Null dereference errors
* Use-after-free and dangling pointer errors
* Double-free errors
* Data races
* Ignored error results
* Unhandled enum variants

## Concept translation table

The following table provides rough equivalents to help you connect familiar programming concepts to Rust:

| Concept | Python / JavaScript | Go | Java / C# | Rust | Chapter |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Absence** | `None` / `null` / `undefined` | `nil` | `null` | `Option<T>` | 4 |
| **Failure** | Exceptions | Multiple return values `(val, err)` | Exceptions | `Result<T, E>` and `?` | 5 |
| **Interface** | Duck typing or protocols | Implicit interfaces | `interface` | `trait` with explicit `impl` | 6 |
| **Inheritance** | Class inheritance | Struct embedding | Class inheritance (`extends`) | Traits and composition | 6 |
| **Package tool** | `pip` or `npm` | `go mod` | `Maven` or `NuGet` | `Cargo` | 1, 8 |
| **Concurrency** | `asyncio` or event loop | Goroutine and channels | Threads or Tasks | Threads or Tokio tasks with channels | 7 |
| **Shared state** | Lock objects or libraries | Mutexes | Synchronization APIs | Synchronization types (such as `Arc<Mutex<T>>`) | 7 |
| **Memory** | Garbage collection | Garbage collection | Garbage collection | Ownership and deterministic cleanup | 3 |
| **Generics** | Dynamic typing | Generics | Generics | Monomorphized generics | 6 |

## Summary

If you encounter difficulties while learning Rust, review this chapter to identify which core difference is causing the issue. Most early compilation issues involve ownership or the distinction between `String` and `&str`. These issues often happen when you try to apply design patterns from garbage-collected languages directly to Rust.

The following chapters explore these differences in detail, starting with memory management and ownership in Chapter 3.
