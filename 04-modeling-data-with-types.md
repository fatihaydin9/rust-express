# Chapter 4: Modeling Data with Types

Memory safety is only one use of Rust's type system. Types can also represent domain
rules so that many invalid states are rejected before the program runs. This chapter
covers structs, enums, `Option`, pattern matching, collections, and iterators, then
develops two important design techniques: newtypes and “parse, don't validate.”

## 4.1 Structs and methods

A `struct` groups fields; behavior goes in `impl` blocks:

```rust
// Derive common capabilities automatically. See section 4.8.
#[derive(Debug, Clone)]
struct Document {
    name: String,
    size_bytes: u64,
}

impl Document {
    // `new` is a conventional constructor name.
    fn new(name: String, size_bytes: u64) -> Self {
        Self { name, size_bytes }
    }

    // `&self` borrows the value for reading.
    fn describe(&self) -> String {
        format!("{} ({} bytes)", self.name, self.size_bytes)
    }

    // `&mut self` borrows the value for modification.
    fn rename(&mut self, new_name: String) {
        self.name = new_name;
    }
}

let mut document = Document::new("plan.md".to_string(), 1024);
document.rename("plan-v2.md".to_string());
println!("{}", document.describe());
```

The ownership model from chapter 3 is visible in these signatures: a method that reads
takes `&self`; one that modifies takes `&mut self`; a rare one that _consumes_ the
struct takes `self` by value. Ownership choices therefore become part of the method's
public contract and are visible in its signature.

> **If you come from Java or C#:** Rust does not have constructors as a separate language
> feature. `new` is a naming convention for an associated function, and the method receiver
> is written explicitly as `self`, `&self`, or `&mut self`.

## 4.2 Enums: variants that carry data

A Rust `enum` can represent more than a list of named integer values. Each variant may
carry its own data. A struct says _this AND that_; an enum says _this OR that_:

```rust
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Point,
}
```

A `Shape` is exactly one variant at a time, holding exactly the data that variant needs.
In type theory these are called _sum types_. In practice, they often replace designs
that would otherwise use class hierarchies or several nullable fields.

## 4.3 `match`: destructure + exhaustiveness

```rust
fn area(shape: &Shape) -> f64 {
    match shape {
        Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
        Shape::Rectangle { width, height } => width * height,
        Shape::Point => 0.0,
    }
}
```

Two properties make `match` more than a switch. It **destructures** — variant data is
pulled into variables in the same motion. And it is **exhaustive** — the compiler
verifies every variant is handled.

> **Try it:** add `Triangle { base: f64, height: f64 }` to the enum and rebuild. Each
> exhaustive `match` on `Shape` reports that the new variant is not covered. These errors
> identify the places where the model change still needs to be handled.

Useful extras include guards (`Shape::Circle { radius } if *radius > 10.0 => …`), the
catch-all `_`, and `|` for multiple patterns. When only one variant matters, `if let`
avoids the need for a full `match`:

```rust
if let Shape::Circle { radius } = &s {
    println!("round, r={radius}");
}
```

## 4.4 `Option<T>`: absence as a type

Rust has no null. If a value can be absent, the type says so — with an ordinary enum
from the standard library:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

```rust
fn find_user(name: &str) -> Option<User> {
    /* ... */
}

match find_user("ada") {
    Some(user) => greet(&user),
    None => println!("no such user"),
}
```

The inner value cannot be used as a `T` until the possibility of `None` has been handled
or transformed. This makes absence part of the program's control flow rather than a
hidden runtime condition. In everyday code, combinators are often more concise than a
full `match`:

```rust
let name_length = find_user("ada").map(|user| user.name.len());
let user = find_user("ada").unwrap_or_else(User::guest);
let first = list.first();

// Types:
// - `name_length`: Option<usize>
// - `user`: User, with a guest fallback for `None`
// - `first`: Option<&T>
```

Convenience methods also exist. `.unwrap()` returns the value or panics on `None`, so it
is most appropriate in tests, examples, and situations where an invariant has already
made absence impossible. `.expect("reason")` serves the same purpose while documenting
that invariant. `Option`'s sibling for _failure_ — `Result` — gets all of chapter 5.

## 4.5 Collections, with ownership in mind

The two workhorses are `Vec<T>` (growable array) and `HashMap<K, V>`:

```rust
use std::collections::HashMap;

let mut tags: Vec<String> = Vec::new();
tags.push("draft".to_string()); // Ownership moves into the vector.

let mut counts: HashMap<String, u32> = HashMap::new();

// The entry API performs an upsert without a separate lookup.
*counts.entry("rust".to_string()).or_insert(0) += 1;
```

Ownership shapes the APIs. Pushing moves the value into the collection (the collection
is now the owner). Indexing a `HashMap` with `[]` panics when the key is missing. Code
that expects absence usually uses `.get(k)`, which returns `Option<&V>`. And iteration
comes in three ownership flavors, chosen by how you write the loop:

```rust
for tag in &tags {
    // `tag: &String`; the vector remains available.
}

for tag in &mut tags {
    // `tag: &mut String`; values can be modified in place.
}

for tag in tags {
    // `tag: String`; the loop consumes the vector.
}
```

> **Common mistake:** writing `for tag in tags` when the vector is still needed afterward.
> Use `&tags` for shared iteration, `&mut tags` for mutable iteration, and `tags` when the
> loop should consume the collection.

## 4.6 Iterators and closures: the everyday high-level layer

Rust's iterator APIs provide a functional style while still compiling to ordinary native
code. A
**closure** is an inline function (`|x| x * 2`); an **iterator chain** transforms lazily and
materializes at `collect`:

```rust
let paid_total: u64 = orders
    .iter()
    .filter(|order| order.status == Status::Paid)
    .map(|order| order.amount_cents)
    .sum();

let names: Vec<&str> = users
    .iter()
    .map(|user| user.name.as_str())
    .collect();
```

Iterator chains are lazy and usually avoid intermediate collections unless `collect` or
a similar operation requests one. The compiler can often optimize them into code
comparable to a hand-written loop, although measurement remains the right way to
evaluate a hot path.

`collect` is type-directed and builds the collection requested by context. When the
context is ambiguous, an explicit annotation such as `.collect::<Vec<_>>()` tells the
compiler which collection to create.

## 4.7 The newtype pattern — and the alias trap

A design-level bug now. Your system has users and documents, both keyed by UUIDs, and
someone "improves readability":

```rust
pub type UserId = Uuid; // Alias only; this is still `Uuid`.
pub type FileId = Uuid;

fn grant_access(user: UserId, file: FileId) {
    /* ... */
}

// The arguments are swapped, but the aliases do not prevent it.
grant_access(file_id, user_id);
```

A `type` declaration creates only an alias. `UserId` and `FileId` remain the same `Uuid`
type and are interchangeable everywhere. The names improve readability but add no
safety, so swapped arguments still compile.

A **newtype** creates a genuinely distinct type by wrapping one field:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(Uuid);
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct FileId(Uuid);

grant_access(file_id, user_id);
// error[E0308]: mismatched types — expected `UserId`, found `FileId`
```

The wrapper is typically optimized to the same representation as the inner value, while
the compiler now rejects the swapped arguments. The wrapper may need a constructor and a
few trait implementations such as `Display` or `From`. In codebases with many identifier
types, a small macro can reduce this repetition. Keep aliases for actual nicknames — the
classic legitimate one being `type Result<T> = std::result::Result<T, MyError>;`.

## 4.8 `derive`: capabilities on request

Those `#[derive(...)]` attributes ask the compiler to write standard-trait
implementations for you: `Debug` (the `{:?}` printout), `Clone`, `Copy` (only for
pure-stack types, §3.4), `PartialEq`/`Eq` (`==`), `Hash` (usable as a map key),
`Default`. Two habits: derive what you _need_, and treat derives on **public** types as
API promises — removing one later breaks your users. Full trait story in chapter 6.

## 4.9 Parse, don't validate

The final technique is often summarized as "parse, don't validate." It changes where
validation happens and what later functions are allowed to receive. A boolean validation
function such as `is_valid_email(s)` reports whether a value is valid, but the original
`String` does not record that result. Later layers may therefore repeat the same check.
**Parsing** converts unvalidated input into a _different type_ whose very existence is
the proof:

```rust
pub struct EmailAddress(String);

impl EmailAddress {
    // The private field makes this constructor the only public entry point.
    pub fn try_new(raw: &str) -> Result<Self, InvalidEmail> {
        if looks_like_email(raw) {
            Ok(Self(raw.to_owned()))
        } else {
            Err(InvalidEmail)
        }
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

fn send_welcome(to: &EmailAddress) {
    // No repeated validation is necessary.
}
```

Validation now happens once, at the boundary where raw input enters. From that point
onward, the `EmailAddress` type records that the check succeeded, so functions receiving
it do not need to validate the same condition again.

Combined with enums, this leads to a central Rust design habit: **push checks to the
edges, encode their results in types, and let the core logic operate on already-valid
values.** Use enums for states and newtypes for constrained values. The compiler can
enforce only the guarantees represented in the type system.

## Summary

Structs combine fields, while enums represent one of several data-carrying alternatives.
Exhaustive `match` expressions turn model changes into compiler-guided work lists, and
`Option` makes absence explicit. Collections and iterators follow the ownership model
without sacrificing low-level performance.

At the design level, use newtypes when raw primitives represent different concepts, and
parse untrusted input at the boundary so valid types carry proof into the core. The next
chapter applies the same approach to failure.
