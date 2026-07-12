# Chapter 4: Modeling Data with Types

Memory safety is only the starting point. Rust's type system also lets a design rule out
entire categories of logical errors before execution. This chapter covers structs, enums,
`Option`, pattern matching, collections, and iterators, then develops two important design
techniques: newtypes and “parse, don't validate.”

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

Chapter 3 is visible in every signature: a method that reads takes `&self`; one that
modifies takes `&mut self`; a rare one that _consumes_ the struct takes `self` by value.
Ownership rules are part of the API, readable at a glance.

> **If you come from Java/C#:** there are no constructors as a language feature — `new` is
> just a conventional associated function — and no `this` magic; the receiver is an
> explicit, typed parameter.

## 4.2 Enums: variants that carry data

The star of the chapter. A Rust `enum` is not a list of named integers — each variant can
carry its own fields. A struct says _this AND that_; an enum says _this OR that_:

```rust
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Point,
}
```

A `Shape` is exactly one variant at a time, holding exactly the data that variant needs.
Type-theory folks call these _sum types_; the practical effect is that many designs which
need class hierarchies or nullable-field tricks elsewhere become one honest enum here.

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

Two properties make `match` more than a switch. It **destructures** — variant data is pulled
into variables in the same motion. And it is **exhaustive** — the compiler verifies every
variant is handled.

> **Try it:** add `Triangle { base: f64, height: f64 }` to the enum and rebuild. Every
> `match` on `Shape` now reports that the new variant is not covered. The compiler has
> effectively generated a TODO list of every decision affected by the model change.
> Experienced teams use this deliberately: change the enum first, then follow the errors.

Useful extras you'll meet: guards (`Shape::Circle { radius } if *radius > 10.0 => …`), the
catch-all `_`, and `|` for multiple patterns. When only one variant matters, `if let` avoids
ceremony:

```rust
if let Shape::Circle { radius } = &s {
    println!("round, r={radius}");
}
```

## 4.4 `Option<T>`: absence as a type

Rust has no null. If a value can be absent, the type says so — with an ordinary enum from
the standard library:

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

The compiler does not allow access to the inner value until the `None` case has been
considered. A forgotten null check is therefore not a valid program. In everyday code,
combinators are often more concise than a full `match`:

```rust
let name_length = find_user("ada").map(|user| user.name.len());
let user = find_user("ada").unwrap_or_else(User::guest);
let first = list.first();

// Types:
// - `name_length`: Option<usize>
// - `user`: User, with a guest fallback for `None`
// - `first`: Option<&T>
```

Escape hatches exist: `.unwrap()` returns the value or **panics** on `None` — acceptable in
examples and tests, a smell in libraries; `.expect("user must exist because …")` at least
documents the bet. `Option`'s sibling for _failure_ — `Result` — gets all of chapter 5.

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

Ownership shapes the APIs. Pushing moves the value into the collection (the collection is
now the owner). Indexing a `HashMap` with `[]` panics on a missing key, so real code uses
`.get(k)`, which returns — of course — an `Option<&V>`. And iteration comes in three
ownership flavors, chosen by how you write the loop:

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

> **Common mistake:** writing `for t in tags` and being surprised that `tags` is gone
> afterwards. Default to `&tags` unless you mean to consume.

## 4.6 Iterators and closures: the everyday high-level layer

Rust's collection processing looks like functional languages and compiles like C. A
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

Two facts are useful early. Iterator chains are **zero-cost abstractions**: the compiler can
optimize them into the same kind of loop you might write manually, without an interpreter or
mandatory intermediate allocations.

`collect` is type-directed and builds the collection requested by context. When the context
is ambiguous, an explicit annotation such as `.collect::<Vec<_>>()` tells the compiler which
collection to create.

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
type and are interchangeable everywhere. The names improve readability but add no safety, so
swapped arguments still compile.

A **newtype** creates a genuinely distinct type by wrapping one field:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(Uuid);
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct FileId(Uuid);

grant_access(file_id, user_id);
// error[E0308]: mismatched types — expected `UserId`, found `FileId`
```

Same memory layout, zero runtime cost, and the swap is now unwritable. The ceremony (a
constructor, `Display`, occasional `From`) is small and, in codebases with many ID types,
stamped out by a five-line macro. Keep aliases for actual nicknames — the classic legitimate
one being `type Result<T> = std::result::Result<T, MyError>;`.

## 4.8 `derive`: capabilities on request

Those `#[derive(...)]` attributes ask the compiler to write standard-trait implementations
for you: `Debug` (the `{:?}` printout), `Clone`, `Copy` (only for pure-stack types, §3.4),
`PartialEq`/`Eq` (`==`), `Hash` (usable as a map key), `Default`. Two habits: derive what
you _need_, and treat derives on **public** types as API promises — removing one later
breaks your users. Full trait story in chapter 6.

## 4.9 Parse, don't validate

The chapter's closing technique, and the one that most changes how your programs are shaped.
**Validating** checks a value and forgets: `is_valid_email(s)` returns `true`, and three
layers deeper, nervous code re-checks, because a `String` carries no memory of having
passed. **Parsing** converts unvalidated input into a _different type_ whose very existence
is the proof:

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

Validation now happens once, at the boundary where raw input enters. From that point onward,
the `EmailAddress` type carries the guarantee, so functions receiving it do not need to
repeat the check.

Combined with enums, this leads to a central Rust design habit: **push checks to the edges,
encode their results in types, and let the core logic operate on already-valid values.** Use
enums for states and newtypes for constrained values. The compiler can enforce only the
guarantees represented in the type system.

## Summary

Structs combine fields, while enums represent one of several data-carrying alternatives.
Exhaustive `match` expressions turn model changes into compiler-guided work lists, and
`Option` makes absence explicit. Collections and iterators follow the ownership model
without sacrificing low-level performance.

At the design level, use newtypes when raw primitives represent different concepts, and
parse untrusted input at the boundary so valid types carry proof into the core. The next
chapter applies the same approach to failure.
