# Chapter 2: What Makes Rust Different

This chapter is an orientation map. It introduces ten important differences through small
examples and points to the chapter that explains each one in depth. Read it before trying to
translate familiar patterns directly into Rust syntax; that habit is the source of many
early fights with the compiler.

## 2.1 No garbage collector, ownership instead

Most mainstream memory-safe languages reclaim memory with a runtime garbage collector. Rust
instead tracks a single _owner_ for each value and inserts cleanup when that owner's scope
ends.

The result is deterministic destruction without GC pauses or collector overhead: files close
and locks release at predictable points. The trade-off is that you must learn the ownership
rules, including the fact that assignment moves ownership by default:

```rust
let a = String::from("hi");
let b = a; // Ownership moves to `b`.

println!("{a}"); // Error: `a` is no longer valid.
```

This is the single biggest mental shift, and it gets all of **chapter 3**.

## 2.2 Immutable by default

`let x = 5;` creates an immutable binding. Mutation is explicit: `let mut x = 5;`. The same
principle appears throughout the language in error handling, conversions, and cross-thread
sharing. Rust asks you to declare behavior that many languages allow implicitly, making the
moving parts easier to see during review.

## 2.3 No `null` or `nil`, `Option<T>` instead

Rust has no universal null, nil, or undefined reference. A `String` always contains a
string. When absence is possible, the type says so explicitly through `Option<String>`,
which is either `Some(value)` or `None`. The compiler requires both possibilities to be
handled before the inner value can be used.

```rust
fn find_user(name: &str) -> Option<User> {
    /* ... */
}

if let Some(user) = find_user("ada") {
    greet(&user);
}

// The absence case is explicit rather than hidden behind a nullable reference.
```

Null-pointer exceptions are not a runtime risk; they are a compile error. Details in
**chapter 4**.

> By forcing developers to acknowledge the possibility of absence through the Option<T> enum, Rust shifts the burden of null-checking from runtime (where it crashes your app) to compile time (where it prevents the app from even building).

> **If you come from Java/C#/JS:** every place you'd write a null check, Rust has already
> forced the question at the type level. TypeScript's strict-null-checks and Kotlin's `?`
> types are partial versions of the same idea.

## 2.4 No exceptions, `Result<T, E>` instead

Failure is represented as a return value rather than an invisible control-flow jump. A
fallible function returns `Result<T, E>`, which is either `Ok(value)` or `Err(error)`, and
the signature records both the possibility and the type of failure. The `?` operator
propagates errors without burying the happy path in boilerplate:

```rust
fn load(path: &str) -> Result<Config, ConfigError> {
    // Return early on error; otherwise continue with the extracted string.
    let text = std::fs::read_to_string(path)?;
    parse(&text)
}
```

No invisible jumps from three stack frames down, no forgetting to catch. There _is_ `panic!`
for unrecoverable bugs — closer to an assertion failure than an exception. All of **chapter
5**.

## 2.5 No classes, no inheritance, traits instead

Rust has structs (data) and `impl` blocks (behavior), but no class hierarchy and no
`extends`. Shared behavior is expressed with **traits** — interfaces that any type can
implement, including types you didn't write. Polymorphism happens through generics
(compile-time, zero-cost) or trait objects (runtime dispatch, when needed). If your design
habit is "make a base class," chapter 6 replaces it with something that composes better.
**Chapter 6.**

## 2.6 No implicit conversions, no truthiness

An `i32` never silently becomes an `i64`, a `u8` never becomes a `char`, a number is never
"truthy":

```rust
let a: i32 = 5;

let b: i64 = a; // Error: expected `i64`, found `i32`.
let b: i64 = a.into(); // Explicit, lossless conversion.
let c = a as f64; // Explicit cast; information loss remains visible.

if a {
    // Error: `if` requires a `bool`.
}

if a != 0 {
    // Write the intended comparison explicitly.
}
```

Tedious for the first hour; then you notice a whole family of silent-truncation and
accidental-comparison bugs is gone. Conversions have a proper system (`From`/`Into`, chapter
6).

## 2.7 Almost everything is an expression

`if`, `match`, blocks, loops-with-`break`-values — they produce values. This replaces the
ternary operator and much temporary-variable plumbing:

```rust
let access = match role {
    Role::Admin => Access::Full,
    Role::Member if verified => Access::Standard,
    _ => Access::ReadOnly,
};
```

`match` additionally checks **exhaustiveness**: forget a case and the program doesn't
compile. This turns out to be a refactoring superpower (chapter 4).

## 2.8 Two string types, on purpose

A common first-week stumbling block is the distinction between `String` and `&str`. `String`
owns growable, heap-allocated text, whereas `&str` is a borrowed view consisting of a
pointer and a length. String literals are `&str`.

A useful rule is to **store `String` and accept `&str`** in function parameters. A `&String`
automatically coerces to `&str`; use `.to_string()` or `.to_owned()` to create an owned
value, and `&s` or `.as_str()` to borrow a view. Chapter 3 explains why this distinction
follows directly from ownership.

## 2.9 Data races are compile errors

In every mainstream language, the compiler happily accepts two threads mutating shared data
without synchronization; you meet the bug in production, intermittently. In Rust the
ownership rules extend across threads (via two marker traits, `Send` and `Sync`), and the
racy program _does not build_. You choose a sharing strategy explicitly — a mutex, a
channel, an atomic — and the choice is visible in the types. This is arguably Rust's biggest
practical gift for server software, and it's **chapter 7**.

## 2.10 The compiler is the teacher — and the strictness has a shape

Together, these differences reveal Rust's general character: it asks important design
questions early. Who owns this value? Can this operation fail? Can this value be absent? Is
this state shared across threads?

That front-loaded strictness can slow early prototyping, and the compiler may reject code
that you know is safe until you learn how to express the proof. In return, null
dereferences, use-after-free errors, data races, ignored failures, and forgotten enum cases
move from runtime incidents to compile-time feedback.

## 2.11 Translation table

Rough equivalents, to anchor your instincts. Rows are concepts; use the pointers, not the
analogies, for precision.

| Concept              | Python / JS                 | Go                          | Java / C#           | Rust                                | Chapter |
| -------------------- | --------------------------- | --------------------------- | ------------------- | ----------------------------------- | ------- |
| Absence              | `None` / `null`/`undefined` | `nil`                       | `null`              | `Option<T>`                         | 4       |
| Failure              | exceptions                  | `(val, err)` returns        | exceptions          | `Result<T, E>` + `?`                | 5       |
| Interface            | duck typing / protocols     | implicit interfaces         | `interface`         | `trait` (explicit `impl`)           | 6       |
| Inheritance          | classes                     | struct embedding            | `extends`           | none — traits + composition         | 6       |
| Package tool         | pip / npm                   | go mod                      | Maven/NuGet         | cargo                               | 1, 8    |
| Concurrency unit     | asyncio / event loop        | goroutine + channel         | threads / Tasks     | threads, or Tokio tasks + channels  | 7       |
| Shared mutable state | "be careful"                | "be careful" + `-race` flag | `synchronized` etc. | `Arc<Mutex<T>>` — enforced          | 7       |
| Memory               | GC                          | GC                          | GC                  | ownership, compile-time             | 3       |
| Generic function     | untyped / TS generics       | generics                    | generics (erased)   | generics (monomorphized, zero-cost) | 6       |

## Summary and how to use this map

When a later example feels unexpectedly difficult, return to this map and identify the
underlying difference. Most early problems come from ownership or the distinction between
`String` and `&str`, and they often indicate that an old-language pattern is being
translated rather than redesigned. The remaining chapters expand this map in an order that
begins with memory and builds outward.
