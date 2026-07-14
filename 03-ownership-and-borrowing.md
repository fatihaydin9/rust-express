# Chapter 3: Ownership and Borrowing

This chapter introduces the model that shapes much of Rust programming. Work through the
examples slowly and read each compiler message to the end. The basic rules are compact,
but they become useful only when ownership and borrowing start to feel natural.

## 3.1 One picture of memory

A running program commonly uses both stack and heap storage. The **stack** is well
suited to fixed-size local values and bookkeeping such as pointers and lengths. Its
storage is tied to function calls and is reclaimed automatically as those calls return.

The **heap** is used for allocations whose size or lifetime is determined at runtime,
such as user-entered text or a growable list. Those allocations need a clear rule for
when their resources can be released. Ownership provides that rule in Rust.

Concretely: `let s = String::from("hello");` creates the bytes `h e l l o` on the heap,
and on the stack a small handle (pointer + length + capacity). "The string" means both
parts.

## 3.2 The three rules of ownership

1. Every value has exactly **one owner** (a variable, a struct field, a slot in a
   collection).
2. When the owner goes **out of scope**, the value is **dropped**. Its destructor runs
   and any owned resources are released at that point.
3. Ownership can **move** to a new owner; the old one becomes invalid.

Rule 2 in action:

```rust
{
    // `s` owns the heap allocation.
    let s = String::from("hello");
    println!("{s}");
} // `s` leaves scope, is dropped, and releases the allocation.
```

There is no explicit `free()` call and no garbage collector. The compiler knows where
the owner leaves scope and inserts cleanup at that point.

This pattern is called RAII: resource cleanup is tied to scope. It applies to more than
memory. A `File` closes when dropped, a database connection returns to its pool, and a
lock is released automatically. This reduces the number of places where cleanup can be
forgotten because resource release follows scope and ownership.

## 3.3 Why assignment moves

What should `let b = a;` do when `a` is a `String`?

Copying only the handle, as a reference-oriented language might, would leave two
apparent owners pointing at one allocation. Both could then try to release it, producing
a double free. Copying the entire heap allocation would be safe, but an ordinary `=`
might silently duplicate a very large value.

Rust chooses a third option: assignment **moves** ownership. After `let b = a;`, `b`
owns the value and `a` is no longer valid:

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
  |
help: consider cloning the value if the performance cost is acceptable
```

Because only the current owner performs cleanup, this particular double-free pattern is
prevented in safe Rust.

When two independent strings are needed, use `let b = a.clone();`. The allocation and
copy are explicit, which makes their cost visible in the source and easier to notice
during review.

## 3.4 `Copy` types

```rust
let x = 5;
let y = x;

println!("{x}"); // Valid because `i32` implements `Copy`.
```

Why is there no error here? An `i32` has a small, fixed-size representation and does not
own a resource that needs special cleanup. Types with this behavior can implement the
`Copy` marker, so assignment duplicates the value instead of invalidating the original.

Primitive numeric types, `bool`, `char`, and tuples or arrays made entirely from `Copy`
types are common examples. Types that manage heap allocations or other resources, such
as `String`, `Vec`, and `File`, generally move instead.

## 3.5 Functions move too

Passing an argument is an assignment; so is returning:

```rust
fn shout(s: String) -> String {
    // The function owns `s` and returns a new owned string.
    s.to_uppercase()
}

let name = String::from("ada");
let loud = shout(name);

// Error: ownership of `name` moved into `shout`.
// println!("{name}");
```

Returning ownership can be useful, but doing it for every read-only function would make
APIs awkward. References allow a function to use a value without taking ownership of it.

## 3.6 Borrowing: `&` to read, `&mut` to write

A **reference** lends access without transferring ownership. `&value` creates one; a
parameter of type `&str` says "I only need to read this text":

```rust
fn measure(s: &str) -> usize {
    // The function borrows the string instead of owning it.
    s.len()
}

let name = String::from("ada lovelace");
let n = measure(&name);

// `name` remains valid because no ownership transfer occurred.
println!("{name}: {n} chars");
```

To lend _with permission to modify_, both sides say so explicitly:

```rust
fn add_signature(s: &mut String) {
    s.push_str(" - Ada");
}

let mut letter = String::from("Dear reader,");
add_signature(&mut letter);
```

Ordinary safe references come in two forms: `&T` provides shared read access, while
`&mut T` provides exclusive mutable access. Their relationship is captured by the
central borrowing rule.

## 3.7 The central borrowing rule

> **At a given time, a value can have any number of shared `&T` references or one
> exclusive `&mut T` reference, but not both.**

This can be summarized as multiple readers or one writer. Much of Rust's safe borrowing
behavior follows from this rule. The following example shows why the restriction
matters:

```rust
let mut items = vec![1, 2, 3];

// Shared borrow into the vector's current storage.
let first = &items[0];

// Error: `push` needs an exclusive mutable borrow of `items`.
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

The danger is concrete. `push` may reallocate the vector and move its elements to a
larger memory block, which would leave `first` pointing at released storage. C++ exposes
this as iterator invalidation, while some managed languages detect related cases only at
runtime. Rust rejects the shape before the program runs.

The borrowing rule addresses a concrete safety problem. Many memory-safety failures
reduce to mutating data while another part of the program still reads it.

Note the last line of the error: a borrow is tracked to its **last use**, not to the end
of scope. Move the `println!` above the `push` and the program compiles — borrows end as
early as possible. This non-lexical lifetime behavior keeps a borrow active only for as
long as it is used.

> **Try it:** do that reordering. Then try holding two `&mut` to the same vector at once,
> and read that error too.

## 3.8 Lifetimes: references must not outlive their owners

The next rule prevents a reference from remaining valid after the value it points to has
been dropped.

```rust
let r;
{
    let x = 5;
    r = &x;
} // `x` is dropped here while `r` still points to it.

println!("{r}");
```

```text
error[E0597]: `x` does not live long enough
```

The span during which a reference is valid is its **lifetime**. A reference cannot
outlive the value it refers to, and the compiler checks this relationship throughout the
program. Most lifetimes are inferred. When a function signature relates several
references in a way the compiler cannot infer on its own, the relationship is written
explicitly:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

Read `'a` as a label for some span of validity. The signature says: given two references
that are valid over a shared span, the function returns a reference valid over that same
span.

Three facts keep lifetime annotations in perspective. They do not extend a value's life;
they only describe relationships that already exist. Most signatures need no explicit
annotations because of lifetime elision. Finally, adding more `'a` labels rarely fixes a
design problem: ask whether the returned value should be a reference at all or should
own its data.

## 3.9 `String` vs `&str` : ownership applied to text

The distinction between `String` and `&str` is ownership applied to text. `String` is
the
**owned** string: heap-allocated, growable, freed when its owner drops. `&str` (say "string
slice") is a **borrowed view**: pointer + length into string data owned by someone else.
String literals have the type `&'static str`; their data is stored with the program and
is available for its entire execution.

The following rules are useful starting points:

```rust
fn print_label(label: &str) {
    println!("[{label}]");
}

// Store owned strings in structs and long-lived values.
let owned = String::from("inventory");

// Accept borrowed string slices in parameters.
print_label(&owned); // `&String` coerces to `&str`.
print_label("literal"); // String literals already have type `&str`.

let owned_copy: String = "hi".to_string(); // `&str` to `String`: allocates.
let view: &str = owned.as_str(); // `String` to `&str`: borrowed view.
```

**Accept `&str`, store `String`**, and let the coercion handle the rest. When the compiler
says `expected &str, found String` (or the reverse), it now reads as an ownership
statement, as an ownership mismatch: one side expects a borrowed view while the other
provides an owned value, or vice versa. The same owner/view pairing exists for
collections ( `Vec<T>` owns, `&[T]` (a _slice_) is the borrowed view ) with identical
rules.

## 3.10 A playbook for borrow-checker conflicts

Borrow conflicts are common while learning Rust. The following steps provide a practical
order for resolving them.

**First, inspect the design.** A persistent borrow conflict often means that two parts of
the program both behave as though they own the same value. Consider passing ownership in
and returning it, splitting a struct so borrows do not overlap, or storing items in one
collection and passing indices rather than long-lived references. These changes resolve
many borrow-checker conflicts and often clarify the model.

**Second, clone deliberately.** Duplicating a small value to simplify ownership can be a
reasonable engineering trade-off. Keep the copy visible, and use measurement to decide
whether its cost matters.

**Third, use the appropriate ownership tools** when the simpler model is not enough. Each
of the following types expresses a specific ownership or mutability requirement:
`Box<T>` (heap-allocate, still one owner), `Rc<T>` / `Arc<T>` (_shared_ ownership via
reference counting, freed when the last owner drops; `Arc` is the thread-safe one —
chapter 7), `RefCell<T>` / `Mutex<T>` (_interior mutability_ — THE rule enforced at
**run time** instead of compile time; `RefCell` panics on violation, `Mutex` makes
violators wait).

> **Common mistake:** reaching for shared ownership and interior mutability before clarifying
> the basic ownership model. If `Rc<RefCell<…>>` begins to appear throughout the design,
> return to the first step and reconsider which component should own the data.

## Summary

Ownership has three rules: each value has one owner, the value is dropped when that
owner leaves scope, and assignment moves ownership unless the type is `Copy`. Borrowing
adds one central rule: a value may have shared readers or one exclusive writer, but not
both at the same time. References must also remain shorter-lived than their owners.

`String`/`&str` and `Vec<T>`/`&[T]` are owner-and-view pairs built on the same model.
When the borrow checker resists a design, answer “who owns this?” first, clone
deliberately when appropriate, and reach for shared-ownership or interior-mutability
types only after the simpler options are exhausted.
