# Chapter 3: Ownership and Borrowing

This chapter is where Rust's model usually begins to click. Work through it slowly, type the
examples, and read each compiler message to the end. The model is compact—three ownership
rules and one borrowing law—but productivity depends on making it intuitive.

## 3.1 One picture of memory

A running program commonly stores data on the stack or the heap. The **stack** holds values
whose size is fixed and known at compile time, such as an `i32`, a `bool`, or a pair of
floats. Stack storage is fast and is reclaimed automatically when a function returns.

The **heap** holds values whose size may grow or is determined at runtime, such as
user-entered text or an expandable list. Heap storage must be allocated, and someone must
decide when it can be released. Ownership is Rust's answer to that question.

Concretely: `let s = String::from("hello");` creates the bytes `h e l l o` on the heap, and
on the stack a small handle (pointer + length + capacity). "The string" means both parts.

## 3.2 The three rules of ownership

1. Every value has exactly **one owner** (a variable, a struct field, a slot in a
   collection).
2. When the owner goes **out of scope**, the value is **dropped**: memory freed, destructor
   run, right there.
3. Ownership can **move** to a new owner; the old one becomes invalid.

Rule 2 in action:

```rust
{
    // `s` owns the heap allocation.
    let s = String::from("hello");
    println!("{s}");
} // `s` leaves scope, is dropped, and releases the allocation.
```

There is no explicit `free()` call and no garbage collector. The compiler knows where the
owner leaves scope and inserts cleanup at that point.

This pattern is called RAII: resource cleanup is tied to scope. It applies to more than
memory. A `File` closes when dropped, a database connection returns to its pool, and a lock
is released automatically. Many “forgot to close it” bugs disappear because cleanup follows
ownership.

## 3.3 Why assignment moves

What should `let b = a;` do when `a` is a `String`?

Copying only the handle, as a reference-oriented language might, would leave two apparent
owners pointing at one allocation. Both could then try to release it, producing a double
free. Copying the entire heap allocation would be safe, but an ordinary `=` might silently
duplicate a very large value.

Rust chooses a third option: assignment **moves** ownership. After `let b = a;`, `b` owns
the value and `a` is no longer valid:

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

One owner at all times → exactly one free → double-free is _unwritable_, not merely
detected.

If you actually want two independent strings, say so: `let b = a.clone();` — an explicit
deep copy. This is a deliberate design stance: **cheap operations are implicit, expensive
ones are visible.** In code review, every `.clone()` is an honest, greppable confession that
a copy happens.

## 3.4 `Copy` types

```rust
let x = 5;
let y = x;

println!("{x}"); // Valid because `i32` implements `Copy`.
```

Why no error here? An `i32` lives entirely on the stack; there is nothing to free, and
copying 4 bytes costs the same as moving them. Such types implement the `Copy` marker:
assignment duplicates instead of moving. All primitive numbers, `bool`, `char`, and small
combinations of them are `Copy`. Rule of thumb: **owns heap data → moves; pure stack value →
copies.** (`String`, `Vec`, `File`, connections: move.)

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

Correct, but exhausting — must every function eat its arguments and hand something back? No.
This friction is exactly what references remove.

## 3.6 Borrowing: `&` to read, `&mut` to write

A **reference** lends access without transferring ownership. `&value` creates one; a
parameter of type `&String` says "I only need to look":

```rust
fn measure(s: &String) -> usize {
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
    s.push_str(" — Ada");
}

let mut letter = String::from("Dear reader,");
add_signature(&mut letter);
```

Exactly two kinds of borrow exist: `&T` (shared, read-only — many at once) and `&mut T`
(exclusive, read-write — only one). Which brings us to the law.

## 3.7 THE rule

> **At any moment, a value has either any number of `&T` references, or exactly one `&mut T`
> — never both.**

Readers in parallel, or a single writer. Every guarantee Rust makes traces back to this
sentence. Watch it catch a bug that ships to production in other languages:

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

The danger is concrete. `push` may reallocate the vector and move its elements to a larger
memory block, which would leave `first` pointing at released storage. C++ exposes this as
iterator invalidation, while some managed languages detect related cases only at runtime.
Rust rejects the shape before the program runs.

The borrowing rule is therefore not pedantry. Many memory-safety failures reduce to mutating
data while another part of the program still reads it.

Note the last line of the error: a borrow is tracked to its **last use**, not to the end of
scope. Move the `println!` above the `push` and the program compiles — borrows end as early
as possible. The checker is stricter than you fear and smarter than you expect.

> **Try it:** do that reordering. Then try holding two `&mut` to the same vector at once,
> and read that error too.

## 3.8 Lifetimes: references must not outlive their owners

One more hole to close: a reference surviving the thing it points to.

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

The span during which a reference is valid is its **lifetime**, and the rule is exactly
intuition: a reference cannot outlive its referent. The compiler checks this silently,
everywhere. Occasionally — when a _function signature_ relates references in a way it can't
infer — you annotate the relationship:

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

Three facts keep lifetime annotations in perspective. They never extend a value's life; they
only describe relationships that already exist. Most signatures need no explicit annotations
because of lifetime elision. Finally, adding more `'a` labels rarely fixes a design problem:
ask whether the returned value should be a reference at all or should own its data.

## 3.9 `String` vs `&str` — ownership applied to text

Now the week-one stumbling block, which is just this chapter in miniature. `String` is the
**owned** string: heap-allocated, growable, freed when its owner drops. `&str` (say "string
slice") is a **borrowed view**: pointer + length into string data owned by someone else.
String literals are `&str` (their data lives in the program binary).

The working rules, worth memorizing verbatim:

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
says `expected &str, found String` (or the reverse), it now reads as an ownership statement,
not noise: someone wanted a view and got an owner, or vice versa. The same owner/view
pairing exists for collections — `Vec<T>` owns, `&[T]` (a _slice_) is the borrowed view —
with identical rules.

## 3.10 When you fight the borrow checker, the playbook

You will; everyone does. Work this list in order.

**First, inspect the design.** A persistent borrow conflict often means that two parts of
the program both behave as though they own the same value. Consider passing ownership in and
returning it, splitting a struct so borrows do not overlap, or storing items in one
collection and passing indices rather than long-lived references. These changes resolve many
borrow-checker conflicts and often clarify the model.

**Second, clone deliberately.** Duplicating a small `String` to break a tangled borrow is
often the correct engineering trade — visible, measurable, honest. Zero-copy purity is a
hobby; ship, then profile.

**Third, use the ownership vocabulary** for shapes the basic rules can't express — real
tools with named costs, all revisited later: `Box<T>` (heap-allocate, still one owner),
`Rc<T>` / `Arc<T>` (_shared_ ownership via reference counting, freed when the last owner
drops; `Arc` is the thread-safe one — chapter 7), `RefCell<T>` / `Mutex<T>` (_interior
mutability_ — THE rule enforced at **run time** instead of compile time; `RefCell` panics on
violation, `Mutex` makes violators wait).

> **Common mistake:** reaching for tool three first. If `Rc<RefCell<…>>` starts wrapping
> everything, you're rebuilding a garbage-collected language with worse ergonomics — return
> to step one and decide who owns what.

## Summary

Ownership has three rules: each value has one owner, the value is dropped when that owner
leaves scope, and assignment moves ownership unless the type is `Copy`. Borrowing adds one
law: a value may have many shared readers or one exclusive writer, but never both at the
same time. References must also remain shorter-lived than their owners.

`String`/`&str` and `Vec<T>`/`&[T]` are owner-and-view pairs built on the same model. When
the borrow checker resists a design, answer “who owns this?” first, clone deliberately when
appropriate, and reach for shared-ownership or interior-mutability types only after the
simpler options are exhausted.
