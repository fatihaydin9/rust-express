# Chapter 1: Quick Start

By the end of this chapter, you will have Rust installed, a small program running, a
dependency in use, and a working set of everyday Cargo commands. The setup is
intentionally short so that you can begin experimenting with the language quickly.

## 1.1 Install

The recommended installer is `rustup`, Rust's toolchain manager. On macOS and Linux,
run:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

On Windows, download and run `rustup-init.exe` from rustup.rs. The installation
includes:

```text
rustc      the compiler (usually invoked through Cargo)
cargo      the build tool, package manager, and test runner
rustfmt    the formatter that applies Rust's standard style
clippy     the linter for common mistakes and unidiomatic patterns
```

You can update the toolchain later with `rustup update`.

For editor support, install **rust-analyzer**, the official Rust language server. In VS
Code, the extension has the same name. It shows inferred types, reports errors while you
type, and offers automatic imports. It is worth treating rust-analyzer as part of the
standard setup because its type hints, navigation, and diagnostics make early learning
considerably easier.

> **If you come from Node or Python:** Cargo combines several responsibilities that are
> often handled by separate tools, including dependency management, builds, tests, and
> lockfile generation.

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
edition = "2024"        # the current default edition for new projects

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

Formatting uses `{}` placeholders, as in `println!("{} + {} = {}", 2, 3, 2 + 3);`.
Variables can also be embedded directly: `println!("{name}");`.

## 1.3 The daily commands

| Command                 | What it does                                    | When                                                     |
| ----------------------- | ----------------------------------------------- | -------------------------------------------------------- |
| `cargo check`           | Type-checks without producing a binary          | During development for a fast feedback loop              |
| `cargo run`             | Builds and runs the program                     | When you want to observe program behavior                |
| `cargo test`            | Runs the test suite                             | As tests are added from chapter 8 onward                 |
| `cargo build --release` | Creates an optimized build in `target/release/` | For performance measurements and production artifacts    |
| `cargo fmt`             | Formats the project                             | Before committing and in CI                              |
| `cargo clippy`          | Checks common mistakes and style issues         | Regularly and in CI with `-D warnings` when appropriate  |
| `cargo doc --open`      | Builds and opens API documentation              | When documenting or exploring code                       |

> **Common mistake:** benchmarking a debug build. Debug builds prioritize fast compilation
> and diagnostics rather than runtime speed, so they can be substantially slower. Use
> `--release` before drawing performance conclusions.

## 1.4 Adding a dependency

Libraries ("crates") live on crates.io, the community registry. Add one from the command
line:

```bash
$ cargo add rand
```

Cargo adds the current compatible `rand` release to `[dependencies]`. A simple example
is:

```rust
fn main() {
    let n: u32 = rand::random_range(1..=100);
    println!("secret number: {n}");
}
```

`cargo run` fetches, compiles, and caches the dependency. Cargo resolves versions
according to semantic-versioning rules and records the exact dependency graph in
`Cargo.lock`. Applications should normally commit that file so local development, CI,
and deployments resolve the same dependency versions.

Two conventions are worth learning early. A project is a _binary crate_ when it contains
`src/main.rs`, and a _library crate_ when it contains `src/lib.rs`; one package may
contain both.

Rust uses `snake_case` for functions and variables, and `CamelCase` for types. The
compiler warns when names do not follow these conventions.

## 1.5 Reading compiler errors

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
source location, an explanation of what the compiler believes happened, and often a
`help:` section containing a direct correction.

Build two habits from the first day. First, read the diagnostic to the bottom because
the final lines frequently contain the solution. Second, when an error remains unclear,
run `rustc --explain E0615` to read a longer explanation of that error category.
Compiler messages are an important learning resource in Rust, so this guide includes
them often and explains how to read them.

## 1.6 A brief look at the syntax

The following example introduces a few common forms so that later chapters are easier to
read. Each concept is explained in more detail when it becomes relevant:

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
variables rarely need annotations. `if` expressions and blocks produce values, which
removes the need for a ternary operator. Numeric conversions are also explicit: an `i32`
does not silently become an `i64` or `f64`. The next chapter places these differences on
one map.

## Summary

You now have the toolchain, a project, a dependency, and the basic command loop. Use
`cargo check` while writing and `cargo run` when you need output. Run `cargo fmt` and
`cargo clippy` before committing, and use release builds for performance measurements.
Keep rust-analyzer enabled in the editor. The next chapter maps the major ways Rust differs
from other languages.
