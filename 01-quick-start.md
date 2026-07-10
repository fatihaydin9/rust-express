# Chapter 1 — Quick Start

By the end of this chapter, you will have Rust installed, a program running, a dependency
working, and the everyday Cargo commands at hand. The initial setup should take about
fifteen minutes.

## 1.1 Install

One command installs everything, on any OS:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

(Windows: download `rustup-init.exe` from rustup.rs.) This gives you `rustup`, the toolchain
manager. Through it you get:

```text
rustc      the compiler (you will almost never call it directly)
cargo      build tool + package manager + test runner — your main interface
rustfmt    the formatter (one canonical style, no debates)
clippy     the linter (hundreds of "you probably meant…" checks)
```

You can update the toolchain later with `rustup update`.

For editor support, install **rust-analyzer**, the official Rust language server. In VS
Code, the extension has the same name. It shows inferred types, reports errors while you
type, and offers automatic imports. Treat it as part of the standard setup: writing Rust
without it makes the learning experience unnecessarily difficult.

> **If you come from Node/Python:** cargo is npm/pip, virtualenv, the build system, and the
> test runner in one binary, with a lockfile by default. There is no separate tool to
> choose.

## 1.2 First project

```bash
$ cargo new hello
$ cd hello
$ cargo run
   Compiling hello v0.1.0
    Finished `dev` profile
     Running `target/debug/hello`
Hello, world!
```

`cargo new` created two files:

```text
hello/
├── Cargo.toml        the manifest: metadata + dependencies
└── src/
    └── main.rs       the code; fn main() is the entry point
```

`Cargo.toml` looks like this:

```toml
[package]
name = "hello"
version = "0.1.0"
edition = "2021"        # language edition — just leave the default

[dependencies]
```

And `src/main.rs`:

```rust
fn main() {
    println!("Hello, world!");
}
```

The `!` in `println!` indicates that it is a _macro_ rather than a function. That
distinction does not matter yet, so you can read it as “print with formatting.”

Formatting uses `{}` placeholders, as in `println!("{} + {} = {}", 2, 3, 2 + 3);`. Variables
can also be embedded directly: `println!("{name}");`.

## 1.3 The daily commands

| Command                 | What it does                                    | When                                                     |
| ----------------------- | ----------------------------------------------- | -------------------------------------------------------- |
| `cargo check`           | Type-checks without producing a binary          | Constantly — it's fast; this is your feedback loop       |
| `cargo run`             | Build + run                                     | When you want output                                     |
| `cargo test`            | Runs all tests                                  | You'll write tests from chapter 8 on                     |
| `cargo build --release` | Optimized build in `target/release/`            | Benchmarks and shipping — debug builds are _much_ slower |
| `cargo fmt`             | Formats the whole project                       | Before every commit                                      |
| `cargo clippy`          | Lints; catches real bugs and non-idiomatic code | Regularly; in CI as `cargo clippy -- -D warnings`        |
| `cargo doc --open`      | Builds and opens your project's API docs        | When exploring dependencies too                          |

> **Common mistake:** benchmarking a debug build. Debug builds skip optimization entirely
> and can be 10–100× slower. Any performance impression you form must come from `--release`.

## 1.4 Adding a dependency

Libraries ("crates") live on crates.io, the community registry. Add one from the command
line:

```bash
$ cargo add rand
```

This writes `rand = "0.9"` into `[dependencies]`. Use it:

```rust
use rand::Rng;

fn main() {
    let n: u32 = rand::rng().random_range(1..=100);
    println!("secret number: {n}");
}
```

`cargo run` fetches, compiles, and caches the dependency. Cargo resolves versions according
to semantic-versioning rules and records the exact dependency graph in `Cargo.lock`. Commit
that file for applications so every environment builds the same graph.

Two conventions are worth learning early. A project is a _binary crate_ when it contains
`src/main.rs`, and a _library crate_ when it contains `src/lib.rs`; one package may contain
both.

Rust uses `snake_case` for functions and variables, and `CamelCase` for types. The compiler
warns when names do not follow these conventions.

## 1.5 Reading compiler errors (a skill, not a chore)

Make a deliberate mistake:

```rust
fn main() {
    let name = String::from("ada");

    // Missing parentheses: this refers to the method instead of calling it.
    let upper = name.to_uppercase;
}
```

```text
error[E0615]: attempted to take value of method `to_uppercase` on type `String`
 --> src/main.rs:3:22
  |
3 |     let upper = name.to_uppercase;
  |                      ^^^^^^^^^^^^ method, not a field
  |
help: use parentheses to call the method
  |
3 |     let upper = name.to_uppercase();
  |                                  ++
```

Rust diagnostics follow a consistent structure: an error code such as `E0615`, the exact
source location, an explanation of what the compiler believes happened, and often a `help:`
section containing a direct correction.

Build two habits from the first day. First, read the diagnostic to the bottom because the
final lines frequently contain the solution. Second, when an error remains unclear, run
`rustc --explain E0615` to read a longer explanation of that error category. Compiler
messages are one of Rust's main teaching tools, so this guide quotes them often.

## 1.6 A 60-second feel for the syntax

You'll learn constructs as concepts need them, but here's the flavor, so later examples read
smoothly:

```rust
// Parameter types follow their names; `->` introduces the return type.
fn add(a: i32, b: i32) -> i32 {
    // The final expression is returned because it has no semicolon.
    a + b
}

fn main() {
    // Local types are usually inferred.
    let x = add(2, 3);

    // Mutation must be declared explicitly.
    let mut count = 0;
    count += 1;

    // `if` is an expression and therefore produces a value.
    let label = if x > 4 { "big" } else { "small" };

    // Half-open ranges exclude the upper bound: 0, 1, 2.
    for i in 0..3 {
        println!("{i}: {label}, count={count}");
    }
}
```

Three details are worth remembering. Types are static but usually inferred, so local
variables rarely need annotations. `if` expressions and blocks produce values, which removes
the need for a ternary operator. Numeric conversions are also explicit: an `i32` never
silently becomes an `i64` or `f64`. The next chapter places these differences on one map.

## Summary

You now have the toolchain, a project, a dependency, and the basic command loop. Use
`cargo check` while writing, `cargo run` when you need output, `cargo fmt` and
`cargo clippy` before committing, and release builds for performance measurements. Keep
rust-analyzer enabled in the editor. The next chapter maps the major ways Rust differs from
other languages.
