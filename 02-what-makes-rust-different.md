# Chapter 2: What Makes Rust Different

This chapter provides an orientation map. It introduces ten important differences
through small examples and points to the chapter that develops each idea in more detail.

Understanding these differences before translating familiar patterns directly into Rust
syntax can prevent many early struggles with the compiler.

## 2.1 No garbage collector: ownership instead

Most mainstream memory-safe languages reclaim memory through a runtime garbage
collector. Rust instead uses ownership and borrowing rules that the compiler verifies
before the program runs. When a value's owner leaves its scope, Rust runs the value's
destructor and releases its resources.

This gives Rust deterministic cleanup without garbage-collection pauses. Files close,
locks release, and heap allocations are freed at predictable points. The trade-off is
that you need to understand ownership rules, including the fact that assignment moves
ownership by default for many types:

```rust
let a = String::from("hi");
let b = a; // Ownership moves to `b`.

println!("{a}"); // Error: `a` is no longer valid.
```

For many newcomers, this is the largest mental shift. Chapter 3 develops the model step
by step.

## 2.2 Immutable by default

`let x = 5;` creates an immutable binding. Mutation is explicit: `let mut x = 5;`.

The same preference for explicit behavior appears in error handling, conversions, and
cross-thread sharing. Rust asks you to state choices that some languages allow
implicitly, which makes important changes easier to notice during review.

## 2.3 No universal `null`: `Option<T>` instead

Safe Rust has no universal `null`, `nil`, or `undefined` reference. A `String` contains
a valid string. When a value may be absent, the type expresses that possibility through
`Option<T>`, which is either `Some(value)` or `None`.

```rust
fn find_user(name: &str) -> Option<User> {
    /* ... */
}

if let Some(user) = find_user("ada") {
    greet(&user);
}

// The absence case is explicit rather than hidden behind a nullable reference.
```

The compiler requires the program to account for absence before using the inner value.
This moves many null-related mistakes from runtime failures to compile-time feedback.
Chapter 4 explains `Option`, enums, and pattern matching in more detail.

> **If you come from Java, C#, or JavaScript:** places that would normally require a null
> check are represented in Rust's type system. TypeScript's strict null checks and Kotlin's
> nullable types follow a similar principle.

## 2.4 Recoverable errors use `Result<T, E>`

Rust does not use exceptions for ordinary recoverable errors. A fallible function
usually returns `Result<T, E>`, which is either `Ok(value)` or `Err(error)`. The
signature therefore records both the possibility and the type of failure.

The `?` operator propagates errors while keeping the successful path easy to follow:

```rust
fn load(path: &str) -> Result<Config, ConfigError> {
    let text = std::fs::read_to_string(path)?;
    parse(&text)
}
```

Rust also supports panics, but they are mainly intended for broken assumptions,
programming errors, and situations from which the program cannot reasonably recover.
Chapter 5 covers the distinction and the design of useful error types.

## 2.5 No class inheritance: structs, implementations, and traits

Rust has structs for data and `impl` blocks for behavior, but it does not have
traditional class hierarchies or an `extends` keyword. Shared behavior is expressed
through traits.

A trait can be implemented for a type when Rust's coherence and orphan rules allow it.
Polymorphism is achieved through generics with static dispatch or trait objects with
runtime dispatch when needed.

Instead of asking, "What base class should this inherit from?", Rust encourages you to
ask, "What behavior does this type need to provide?" Chapter 6 develops that approach
and shows how it affects architecture.

## 2.6 No implicit numeric conversions or truthiness

Rust does not silently convert one numeric type into another, and conditions must have
the type `bool`:

```rust
let a: i32 = 5;

let b: i64 = a; // Error: expected `i64`, found `i32`.
let b: i64 = a.into(); // Explicit, lossless conversion.
let c = a as f64; // Explicit cast; the conversion remains visible.

if a {
    // Error: `if` requires a `bool`.
}

if a != 0 {
    // The intended comparison is explicit.
}
```

Rust does support carefully defined coercions, such as converting `&String` to `&str`,
but it avoids broad implicit conversions that could obscure intent or lose information.
Chapter 6 introduces the standard conversion traits.

## 2.7 Many constructs are expressions

Constructs such as `if`, `match`, blocks, and loops that use `break` with a value can
produce values. This often reduces the need for temporary variables and replaces many
uses of a ternary operator:

```rust
let access = match role {
    Role::Admin => Access::Full,
    Role::Member if verified => Access::Standard,
    _ => Access::ReadOnly,
};
```

`match` also checks exhaustiveness. When a case is missing, the compiler points to the
model change that still needs to be handled. Chapter 4 explores this in depth.

## 2.8 Two main string types, on purpose

A common early stumbling block is the distinction between `String` and `&str`. `String`
owns growable, heap-allocated text, whereas `&str` is a borrowed view into valid UTF-8
string data. String literals have the type `&'static str`.

A useful starting rule is to store `String` and accept `&str` in function parameters. A
`&String` usually coerces to `&str`. Use `.to_string()` or `.to_owned()` when you need
an owned value, and borrow with `&s` or `.as_str()` when you need a view.

Chapter 3 explains why this distinction follows directly from ownership.

## 2.9 Data races are compile-time errors in safe Rust

Many languages allow shared mutable state to be accessed without sufficient
synchronization. The resulting bugs can be intermittent and difficult to reproduce.

Safe Rust prevents data races through its ownership, borrowing, and type systems. The
`Send` and `Sync` traits help express which values can cross thread boundaries or be
shared between threads safely. You choose a strategy explicitly, such as a mutex, a
channel, or an atomic type, and that choice appears in the program's types.

Rust does not prevent every concurrency problem. Deadlocks, starvation, and higher-level
ordering mistakes remain possible. Chapter 7 explains both the guarantees and their
limits.

## 2.10 The compiler becomes part of the design process

Together, these differences reveal Rust's general character. The language asks important
design questions early:

- Who owns this value?
- Can this operation fail?
- Can this value be absent?
- Is this state mutable or shared across threads?

This front-loaded strictness can slow early experiments, and the compiler may reject
code that you believe is safe until you learn how to express the relevant relationships
through ownership and types.

In return, many problems—including null dereferences, use-after-free errors, dangling
references, double frees, data races, ignored recoverable errors, and forgotten enum
variants—move from runtime incidents to compile-time feedback.

## 2.11 Translation table

The following table gives rough equivalents to help anchor familiar concepts. The
analogies are useful starting points, but the linked chapters provide the more precise
explanation.

| Concept              | Python / JS                 | Go                          | Java / C#           | Rust                               | Chapter |
| -------------------- | --------------------------- | --------------------------- | ------------------- | ---------------------------------- | ------- |
| Absence              | `None` / `null`/`undefined` | `nil`                       | `null`              | `Option<T>`                        | 4       |
| Failure              | exceptions                  | `(val, err)` returns        | exceptions          | `Result<T, E>` + `?`               | 5       |
| Interface            | duck typing / protocols     | implicit interfaces         | `interface`         | `trait` with an explicit `impl`    | 6       |
| Inheritance          | classes                     | struct embedding            | `extends`           | traits and composition             | 6       |
| Package tool         | pip / npm                   | go mod                      | Maven / NuGet       | Cargo                              | 1, 8    |
| Concurrency unit     | asyncio / event loop        | goroutine + channel         | threads / Tasks     | threads or Tokio tasks + channels  | 7       |
| Shared mutable state | conventions and runtime tools | mutexes and race detector | synchronization APIs | types such as `Arc<Mutex<T>>`      | 7       |
| Memory management    | garbage collection          | garbage collection          | garbage collection  | ownership and deterministic cleanup | 3      |
| Generic function     | dynamic typing / TS generics | generics                   | generics             | monomorphized generics             | 6       |

## Summary and how to use this map

When a later example feels unexpectedly difficult, return to this chapter and identify
the underlying difference. Many early problems involve ownership or the distinction
between `String` and `&str`. They often indicate that a design from another language is
being copied directly rather than adapted to Rust's model.

The remaining chapters expand this map, beginning with memory and ownership and then
moving outward to types, errors, abstractions, concurrency, and project structure.
