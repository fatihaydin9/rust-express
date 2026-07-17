# Ownership and borrowing

This chapter introduces ownership and borrowing, which are the core concepts that define how Rust manages memory. 

To write efficient and safe Rust code, you must understand these concepts. As you read this chapter, examine the code examples and error messages to understand how the Rust compiler enforces these rules.

## Stack and heap memory

A running program uses two parts of memory to store data: the stack and the heap.

### The stack
The stack stores data with a fixed, known size.
* **Characteristics**: Fast allocation and retrieval.
* **Lifetime**: Tied to the execution scope of a function. When a function returns, the program automatically cleans up the stack memory for that function.
* **Stored items**: Primitive types (such as integers and booleans), pointers, and local variables.

### The heap
The heap stores data whose size or lifetime can change at runtime.
* **Characteristics**: Slower allocation than the stack because the operating system must find an empty space of memory.
* **Lifetime**: Determined by the ownership rules of the program.
* **Stored items**: Dynamic data, such as a growable list (`Vec`) or user-input text (`String`).

### Memory representation of a `String`
When you write `let s = String::from("hello");`, Rust allocates memory in two places:
1. **On the heap**: The actual characters `h e l l o`.
2. **On the stack**: A metadata handle that contains:
   * A pointer to the heap memory.
   * The length of the string.
   * The capacity of the string.

Together, the stack handle and the heap data represent the string.

## The three rules of ownership

Rust manages memory using a set of ownership rules. The compiler checks these rules when you build your program.

The three rules of ownership are:
1. Every value in Rust has a variable called its **owner**.
2. A value can have only **one owner** at a time.
3. When the owner goes **out of scope**, Rust automatically drops the value and releases its memory.

### Scope and resource cleanup (RAII)
Consider the following example:

```rust
{
    // The variable `s` owns the heap allocation.
    let s = String::from("hello");
    println!("{s}");
} // `s` goes out of scope here. Rust drops `s` and releases the heap memory.
```

In Rust, you do not need to call a `free` function or use a garbage collector. The compiler determines where a variable goes out of scope and automatically inserts code to clean up the resource.

This pattern is known as **Resource Acquisition Is Initialization (RAII)**. In Rust, RAII applies to all resources:
* **Files**: A `File` handle closes when it goes out of scope.
* **Database connections**: A connection returns to the connection pool when dropped.
* **Mutex locks**: A lock releases automatically when the lock guard goes out of scope.

## Assignment and move semantics

In many programming languages, assigning one variable to another copies a reference or pointer. In Rust, assigning a value to another variable **moves** the ownership of that value.

### Why Rust moves data
Consider this example:
```rust
let a = String::from("hello");
let b = a;
```

If Rust only copied the stack handle, both `a` and `b` would point to the same heap data. When `a` and `b` go out of scope, both would try to release the same memory. This is a critical security bug called a **double-free error**.

To prevent this issue, Rust makes `a` invalid as soon as you assign it to `b`. Ownership of the data moves from `a` to `b`.

### Move error example
If you try to use `a` after moving it, the compiler produces an error:

```rust
let a = String::from("hello");
let b = a;
println!("{a}");
```

```text
error[E0382]: borrow of moved value: `a`
  |
2 |     let b = a;
  |             - value moved here
3 |     println!("{a}");
  |                ^ value borrowed here after move
```

### Deep copying with `clone`
If you need to duplicate the heap data so that both variables remain valid, use the `clone` method:

```rust
let a = String::from("hello");
let b = a.clone(); // Deep copies the heap data.
println!("{a}");   // This is valid.
```

Cloning allocates new heap memory, which has a performance cost. Using `clone` makes this cost explicit in your source code.

## Copy types

Some types do not move on assignment. Instead, they are duplicated automatically.

```rust
let x = 5;
let y = x;
println!("{x}"); // This is valid.
```

This code works because `i32` (an integer) implements the `Copy` trait. Types that implement `Copy` have a small, fixed size on the stack and do not own resources on the heap. Assigning these types performs a bitwise copy of the value on the stack.

### Common `Copy` types
The following types implement the `Copy` trait:
* All integer types (such as `i32`, `u64`).
* All floating-point types (such as `f64`).
* The boolean type (`bool`).
* The character type (`char`).
* Tuples and arrays, if they contain only types that implement `Copy`.

Types that manage heap resources, such as `String` and `Vec`, do not implement `Copy`. These types use move semantics.

## Functions and ownership

Passing a value to a function moves ownership of that value to the function parameter. Returning a value from a function passes ownership to the caller.

```rust
fn shout(s: String) -> String {
    s.to_uppercase() // Ownership of `s` is returned to the caller.
}

fn main() {
    let name = String::from("ada");
    let loud = shout(name); // Ownership of `name` moves into the `shout` function.

    // This is invalid because `name` no longer owns the string.
    // println!("{name}");
}
```

To avoid constantly passing and returning ownership, Rust uses references.

## References and borrowing

A **reference** allows you to access a value without taking ownership of it. Creating a reference is called **borrowing**.

### Shared references (`&T`)
A shared reference allows you to read a value but not modify it. You create a shared reference by prefixing the variable with an ampersand (`&`).

```rust
fn measure(s: &str) -> usize {
    s.len() // Accesses the value without owning it.
}

fn main() {
    let name = String::from("ada lovelace");
    let n = measure(&name); // Passes a shared reference.

    println!("{name}: {n} chars"); // `name` is still valid.
}
```

### Mutable references (`&mut T`)
A mutable reference allows you to modify the borrowed value. To create a mutable reference, both the variable and the reference must use the `mut` keyword.

```rust
fn add_signature(s: &mut String) {
    s.push_str(" - Ada");
}

fn main() {
    let mut letter = String::from("Dear reader,");
    add_signature(&mut letter); // Passes a mutable reference.
}
```

## The borrowing rule

Rust enforces one central rule for borrowing to prevent data race conditions:

> **At any given time, you can have either:**
> * **Any number of shared references (`&T`) to a value.**
> * **Exactly one mutable reference (`&mut T`) to a value.**
>
> **You cannot have both at the same time.**

This rule ensures that data cannot change while other parts of the program are reading it.

### Borrowing conflict example
The following example shows how the compiler enforces this rule:

```rust
let mut items = vec![1, 2, 3];

// Shared reference created here.
let first = &items[0];

// Compiler error: `push` requires a mutable reference.
items.push(4);

println!("{first}");
```

```text
error[E0502]: cannot borrow `items` as mutable because it is also borrowed as immutable
  |
2 |     let first = &items[0];
  |                  ----- immutable borrow occurs here
3 |     items.push(4);
  |     ^^^^^^^^^^^^^ mutable borrow occurs here
4 |     println!("{first}");
  |                ----- immutable borrow later used here
```

If the compiler allowed `push` while `first` was active, the vector might reallocate its heap storage to fit the new element. This reallocation would make the pointer in `first` point to invalid memory, causing undefined behavior.

### Non-lexical lifetimes (NLL)
The Rust compiler tracks references from where they are created to their **last use**, rather than to the end of the enclosing block. This is called a non-lexical lifetime.

If you move the print statement above the mutation, the code compiles successfully:

```rust
let mut items = vec![1, 2, 3];
let first = &items[0]; // Shared reference starts.
println!("{first}");   // Shared reference ends here (last use).

items.push(4);         // This is now valid.
```

## Lifetimes

A **lifetime** is the scope for which a reference remains valid. The compiler checks lifetimes to ensure that a reference never outlives the data it points to.

### Invalid reference example
Consider this example where a reference outlives its data:

```rust
let r;
{
    let x = 5;
    r = &x;
} // `x` is dropped here, but `r` still points to its memory location.

println!("{r}");
```

The compiler rejects this code because `x` does not live as long as `r`:
```text
error[E0597]: `x` does not live long enough
```

### Lifetime annotations in function signatures
Most lifetimes are inferred by the compiler automatically. However, when a function accepts multiple references and returns a reference, you must sometimes specify the relationship between their lifetimes explicitly.

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

The `'a` syntax is a lifetime annotation. In this signature, the annotations specify that the returned reference lives at least as long as the shortest lifetime of the inputs `x` and `y`.

Lifetime annotations do not change how long a value lives. Instead, they describe existing lifetime relationships so the compiler can verify memory safety.

## Strings: `String` vs `&str`

The difference between `String` and `&str` is a practical application of ownership and borrowing.

* **`String`**: An owned, heap-allocated string that is growable and modifiable.
* **`&str`**: An immutable, borrowed view (or slice) into string data owned by another variable.

### Guidelines for strings
* Use `String` when you need to store text in structs or transfer ownership of the data.
* Use `&str` as a function parameter type so the function can accept both `String` references and string literals.

```rust
fn print_label(label: &str) {
    println!("[{label}]");
}

fn main() {
    let owned = String::from("inventory");
    
    // A reference to a `String` automatically converts (coerces) to `&str`.
    print_label(&owned); 
    
    // String literals are already of type `&str`.
    print_label("literal"); 
}
```

The same relationship applies to vectors and slices: `Vec<T>` is the owned container, and `&[T]` is the borrowed view.

## Resolve borrow-checker conflicts

When you encounter borrow-checker errors, use the following three-step process to resolve them:

1. **Examine your program design**: Many borrow conflicts indicate that multiple parts of your code are trying to own or modify the same data. 
   * Consider passing ownership to functions and returning it.
   * Split large structs into smaller, independent structs so that borrows do not overlap.
   * Store data in a single collection and pass indices instead of long-lived references.
2. **Clone data deliberately**: If copying a value simplifies ownership and does not impact performance, use the `clone` method. Keep these copies visible and measure performance to ensure they do not cause bottlenecks.
3. **Use smart pointers**: If simpler designs do not work, use Rust's advanced ownership types:
   * **`Box<T>`**: Allocates a value on the heap with a single owner.
   * **`Rc<T>` / `Arc<T>`**: Implements shared read-only ownership using reference counting. `Arc<T>` is the thread-safe version of `Rc<T>`.
   * **`RefCell<T>` / `Mutex<T>`**: Implements interior mutability, which moves borrow checking from compile time to runtime. `RefCell<T>` is for single-threaded code and panics if borrowing rules are violated. `Mutex<T>` is for multi-threaded code.

> **Common mistake**:
> Avoid using shared ownership and interior mutability types (such as `Rc<RefCell<T>>`) as your default design. Reconsider your architecture first to see if you can achieve a simpler ownership structure.

## Summary

This chapter introduced how Rust manages memory safely:
* **Ownership**: Every value has a single owner. When the owner goes out of scope, Rust cleans up the resource.
* **Borrowing**: You can have multiple shared references (`&T`) or a single mutable reference (`&mut T`) to a value at any time.
* **Lifetimes**: The compiler verifies that references never point to dropped data.
* **String types**: Use `String` for owned text and `&str` for borrowed views.
