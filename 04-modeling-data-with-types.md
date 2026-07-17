# Modeling data with types

Rust's type system does more than ensure memory safety. You can also use types to represent your business or domain rules. This helps the compiler prevent invalid states before your program runs.

By the end of this chapter, you will understand:
* How to group data using structs and define behaviors using methods.
* How to use enums to represent alternative data structures.
* How to use pattern matching (`match` and `if let`) to safely access variant data.
* How to represent the absence of data using the `Option` type.
* How collections like `Vec` and `HashMap` interact with ownership.
* How to use iterators and closures for clean, functional data processing.
* How to design safer APIs using the newtype pattern and the "parse, don't validate" technique.

## Structs and methods

A **struct** (short for structure) allows you to group related fields into a single custom type. You define the behavior for a struct inside an `impl` (implementation) block.

The following example defines a `Document` struct and its associated methods:

```rust
// The derive attribute automatically implements common traits. 
// For more details, see the section on derive attributes.
#[derive(Debug, Clone)]
struct Document {
    name: String,
    size_bytes: u64,
}

impl Document {
    // This is an associated function that acts as a constructor.
    // By convention, constructors are named `new`.
    fn new(name: String, size_bytes: u64) -> Self {
        Self { name, size_bytes }
    }

    // This method borrows the struct as read-only.
    fn describe(&self) -> String {
        format!("{} ({} bytes)", self.name, self.size_bytes)
    }

    // This method borrows the struct as mutable to modify its data.
    fn rename(&mut self, new_name: String) {
        self.name = new_name;
    }
}
```

You can use the struct and call its methods like this:

```rust
let mut document = Document::new("plan.md".to_string(), 1024);
document.rename("plan-v2.md".to_string());
println!("{}", document.describe());
```

### Ownership in method signatures
The ownership rules from Chapter 3 are reflected in method signatures:
* **`&self`**: Borrows the value as read-only. This is the most common receiver type for reading data.
* **`&mut self`**: Borrows the value as mutable. Use this when the method needs to modify fields.
* **`self`**: Takes ownership of the value. This consumes the struct, making it invalid after the method call.

These receiver options make ownership guarantees part of your public API contract.

> **If you come from Java or C#:**
> Rust does not have a dedicated `constructor` keyword. Instead, creating a struct constructor is a naming convention using an associated function (usually named `new`). You must also explicitly declare the method receiver (`self`, `&self`, or `&mut self`) as the first parameter of your methods.

## Enums: data-carrying variants

In Rust, **enums** (enumerations) are highly versatile. Unlike enums in other languages that are limited to a list of named integers, Rust enums can carry custom data in each variant.

A struct represents an **AND** relationship (contains *this* field and *that* field). An enum represents an **OR** relationship (contains *either* this variant or *that* variant).

```rust
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Point,
}
```

A variable of type `Shape` is exactly one of these variants at any given time. It stores only the data required for that specific variant. In computer science, these are called **sum types**. They help you avoid complex class hierarchies or multiple nullable fields.

## Pattern matching with `match`

The `match` control flow operator allows you to compare a value against a series of patterns and execute code based on the matching pattern.

```rust
fn area(shape: &Shape) -> f64 {
    match shape {
        Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
        Shape::Rectangle { width, height } => width * height,
        Shape::Point => 0.0,
    }
}
```

The `match` operator provides two major advantages over standard switch statements:
* **Destructuring**: It automatically extracts the inner data of a variant into local variables.
* **Exhaustiveness checking**: The compiler verifies that your `match` expression handles every possible enum variant. If you miss a variant, the program does not compile.

> **Try it**:
> Add a `Triangle { base: f64, height: f64 }` variant to the `Shape` enum. If you try to compile, the compiler points out every `match` expression where `Triangle` is not handled.

### Advanced pattern matching techniques
* **Match guards**: You can add an `if` condition to a match arm:
  ```rust
  match shape {
      Shape::Circle { radius } if *radius > 10.0 => { /* ... */ },
      _ => { /* ... */ }
  }
  ```
* **The catch-all pattern (`_`)**: Matches any remaining values that were not explicitly listed.
* **The multiple pattern operator (`|`)**: Matches any of the specified patterns in a single arm.
* **`if let` expressions**: If you only need to handle a single variant and ignore the rest, use `if let` for more concise code:
  ```rust
  if let Shape::Circle { radius } = &s {
      println!("round, r={radius}");
  }
  ```

## Representing absence with `Option<T>`

Rust does not have a `null` or `nil` value. Instead, when a value can be absent, Rust uses the standard library enum `Option<T>`:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

Using `Option` makes the possibility of an absent value explicit in your code. You cannot use the inner value of type `T` until you check for and handle the `None` case.

```rust
fn find_user(name: &str) -> Option<User> {
    // Returns Some(User) if found, or None if not found.
}

match find_user("ada") {
    Some(user) => greet(&user),
    None => println!("no such user"),
}
```

### Useful `Option` methods and combinators
For simple operations, you can use built-in combinators instead of writing a full `match` statement:
* **`.map`**: Applies a function to the inner value if it is `Some`.
  ```rust
  let name_length = find_user("ada").map(|user| user.name.len()); // Option<usize>
  ```
* **`.unwrap_or_else`**: Returns the inner value of `Some`, or evaluates a fallback closure if the value is `None`.
  ```rust
  let user = find_user("ada").unwrap_or_else(User::guest); // Returns User
  ```
* **`.unwrap()`**: Returns the inner value or panics if the value is `None`. Use `.unwrap()` only in tests, examples, or when you are certain the value cannot be `None`.
* **`.expect("message")`**: Works like `.unwrap()`, but allows you to specify a custom panic message to document why the value must be present.

## Collections and ownership

The two most common collection types in Rust are `Vec<T>` (a growable array) and `HashMap<K, V>` (a key-value map).

```rust
use std::collections::HashMap;

let mut tags: Vec<String> = Vec::new();
tags.push("draft".to_string()); // Ownership of the string moves into the vector.

let mut counts: HashMap<String, u32> = HashMap::new();

// The Entry API allows you to update or insert values efficiently.
*counts.entry("rust".to_string()).or_insert(0) += 1;
```

### Iteration and ownership
Ownership determines how you iterate over collections. There are three ways to write a loop:

1. **Iterate by shared reference (`&tags`)**:
   ```rust
   for tag in &tags {
       // `tag` is of type `&String`. The vector remains valid for use after the loop.
   }
   ```
2. **Iterate by mutable reference (`&mut tags`)**:
   ```rust
   for tag in &mut tags {
       // `tag` is of type `&mut String`. You can modify the string values in place.
   }
   ```
3. **Iterate by value (`tags`)**:
   ```rust
   for tag in tags {
       // `tag` is of type `String`. The loop consumes the vector, making it invalid afterward.
   }
   ```

> **Common mistake**:
> Writing `for tag in tags` when you still need to use the `tags` vector after the loop. To avoid moving ownership out of the collection, use `for tag in &tags` instead.

## Iterators and closures

Rust provides high-level functional APIs, such as closures and iterators, that compile down to highly efficient native code.

* **Closure**: An anonymous, inline function. For example, `|x| x * 2`.
* **Iterator**: A sequence of values that you can process step-by-step.

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

### Key characteristics of iterators
* **Lazy evaluation**: Iterators are lazy. They do not perform any calculations or allocate memory until you call a consuming method, such as `.sum()` or `.collect()`.
* **Zero-cost abstractions**: The compiler optimizes iterator chains into machine code that is often as fast as a hand-written `for` loop.
* **Type-directed collection**: The `.collect()` method compiles based on the expected return type. If the type is ambiguous, you can specify it using the "turbofish" syntax:
  ```rust
  let names = users.iter().map(|u| u.name.as_str()).collect::<Vec<_>>();
  ```

## The newtype pattern vs type aliases

It is common to create descriptive names for primitive types. However, type aliases can lead to bugs if they are not used carefully.

### The danger of type aliases
Consider the following code that uses type aliases:

```rust
pub type UserId = Uuid;
pub type FileId = Uuid;

fn grant_access(user: UserId, file: FileId) {
    // ...
}

// These values are swapped by mistake, but the compiler does not catch it.
grant_access(file_id, user_id);
```

Because `type` only creates an alias, both `UserId` and `FileId` are still treated as the same underlying `Uuid` type. The compiler does not prevent you from swapping them.

### Creating distinct types with the newtype pattern
A **newtype** is a tuple struct with a single field that wraps an existing type. This creates a completely new type that the compiler treats as distinct.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(pub Uuid);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct FileId(pub Uuid);

// Swapping these arguments now triggers a compile-time error.
// grant_access(file_id, user_id);
// error[E0308]: mismatched types - expected `UserId`, found `FileId`
```

The newtype pattern provides type safety with zero runtime overhead. The compiler optimizes the wrapper struct away so that it has the same memory representation as the inner type.

## The `derive` attribute

The `#[derive(...)]` attribute asks the compiler to automatically write basic trait implementations for your custom types.

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Default)]
struct Config {
    port: u16,
    host: String,
}
```

### Common derivable traits
* **`Debug`**: Formats the type for output using the `{:?}` placeholder.
* **`Clone`**: Allows explicit deep-copying of the value.
* **`Copy`**: Allows bitwise duplication on assignment (only works if all fields implement `Copy`).
* **`PartialEq` / `Eq`**: Implements equality comparisons (`==`).
* **`Hash`**: Allows the type to be used as a key in a `HashMap`.
* **`Default`**: Provides a standard default constructor (for example, numeric fields default to `0`, strings default to empty).

### Best practices
* Derive only the traits that your type genuinely needs.
* If a struct or enum is part of your public API, remember that changing derived traits is a breaking change for your users.

## Parse, don't validate

"Parse, don't validate" is a design technique that shifts validation to the boundaries of your program and encodes the result in your types.

A standard validation function returns a boolean value:
```rust
fn is_valid_email(s: &str) -> bool {
    // ...
}
```
If this function returns `true`, you still have a raw `String`. Other parts of your program must re-run this validation check because they cannot be sure that the string is still valid.

With **parsing**, you convert the raw input into a distinct type. The very existence of this type is proof that the validation succeeded.

```rust
pub struct EmailAddress(String);

impl EmailAddress {
    // This constructor is the only public way to create an EmailAddress.
    pub fn try_new(raw: &str) -> Result<Self, InvalidEmail> {
        if is_valid_email(raw) {
            Ok(Self(raw.to_owned()))
        } else {
            Err(InvalidEmail)
        }
    }

    // Provides a safe read-only view of the inner string.
    pub fn as_str(&self) -> &str {
        &self.0
    }
}

fn send_welcome(to: &EmailAddress) {
    // You do not need to validate the email here because the type system guarantees its validity.
}
```

### Benefits of parsing
* **Single point of validation**: Validation happens once at the input boundary.
* **Type-enforced safety**: The core logic of your application operates only on pre-validated types, making your program much more robust.
* **Improved performance**: Eliminates redundant validation checks throughout your codebase.

## Summary

This chapter discussed how to use Rust's type system to write safe and structured code:
* **Structs** group related fields, and **methods** define behaviors using explicit ownership receivers (`&self`, `&mut self`, or `self`).
* **Enums** represent alternative data-carrying structures. Combined with exhaustive `match` expressions, they make code modifications highly manageable.
* **`Option`** replaces null values, forcing you to handle the possibility of absent data explicitly.
* **Collections** and **iterators** allow you to write high-level functional code without sacrificing low-level performance.
* **The newtype pattern** prevents swapped argument bugs by wrapping primitives in distinct structs.
* **The "parse, don't validate" technique** encourages you to validate raw input at the program boundary and encode successful checks directly in your types.
