# Chapter 1: Quick Start

By the end of this chapter, you will install Rust, run a small program, add a dependency, and learn everyday Cargo commands. We keep this setup short so you can start experimenting with the language quickly.

## 1.1 Install

Use `rustup`, the official Rust toolchain manager, to install Rust. On macOS and Linux, run this command:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

On Windows, download and run `rustup-init.exe` from [rustup.rs](https://rustup.rs). The installation includes the following tools:

```text
rustc      the compiler (usually run through Cargo)
cargo      the build tool, package manager, and test runner
rustfmt    the formatter that applies standard Rust styling
clippy     the linter that catches common mistakes and bad patterns
```

To update your toolchain later, run `rustup update`.

For editor support, install **rust-analyzer**, the official Rust language server (available as an extension in VS Code). It displays inferred types, reports errors as you type, and manages automatic imports. We highly recommend installing rust-analyzer; its type hints, code navigation, and diagnostics make learning Rust much easier.

> **If you come from Node or Python:** Cargo handles several tasks that require separate tools in other languages. It manages dependencies, builds code, runs tests, and generates lockfiles.

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

The `cargo new` command creates two main files:

```text
hello/
├── Cargo.toml        the manifest: metadata + dependencies
└── src/
    └── main.rs       the code; fn main() is the entry point
```

Your `Cargo.toml` file looks like this:

```toml
[package]
name = "hello"
version = "0.1.0"
edition = "2024"        # the current default edition for new projects

[dependencies]
```

And your `src/main.rs` file:

```rust
fn main() {
    println!("Hello, world!");
}
```

The `!` in `println!` means it is a *macro*, not a function. You do not need to worry about this distinction yet; just think of it as "print with formatting." 

Use `{}` placeholders to format output, like this: `println!("{} + {} = {}", 2, 3, 2 + 3);`. You can also embed variables directly into the string: `println!("{name}");`.

## 1.3 The daily commands

| Command                 | What it does                                    | When to use it                                           |
| ----------------------- | ----------------------------------------------- | -------------------------------------------------------- |
| `cargo check`           | Type-checks code without building a binary      | During development for fast feedback                     |
| `cargo run`             | Builds and runs the program                     | To test and observe program behavior                     |
| `cargo test`            | Runs the test suite                             | When testing code (covered in Chapter 8)                 |
| `cargo build --release` | Creates an optimized build in `target/release/` | For performance testing and production releases          |
| `cargo fmt`             | Formats the project code                        | Before committing code and in CI pipelines               |
| `cargo clippy`          | Checks for common mistakes and style issues     | Regularly, and in CI (use `-D warnings` when needed)     |
| `cargo doc --open`      | Builds and opens the API documentation          | When exploring or writing documentation                  |

> **Common mistake:** Benchmarking a debug build. Debug builds prioritize fast compile times over runtime speed, making them noticeably slower. Always use `--release` before testing performance.

## 1.4 Adding a dependency

Rust libraries are called "crates." You can find them on [crates.io](https://crates.io), the community registry. To add a crate, use the command line:

```bash
$ cargo add rand
```

Cargo automatically adds the latest compatible version of `rand` to your `[dependencies]`. Here is a simple example using the new crate:

```rust
fn main() {
    let n: u32 = rand::random_range(1..=100);
    println!("secret number: {n}");
}
```

When you type `cargo run`, Cargo fetches, compiles, and caches the new dependency. It uses semantic versioning rules to resolve versions and saves the exact dependency graph in a `Cargo.lock` file. You should commit this file to your version control system. This ensures that your local environment, CI pipelines, and deployments all use the exact same dependency versions.

Learn these two conventions early: 
1. A project with `src/main.rs` is a *binary crate* (an executable). A project with `src/lib.rs` is a *library crate*. A single package can contain both. 
2. Rust uses `snake_case` for functions and variables, and `CamelCase` for types. The compiler will warn you if you do not follow these rules.

## 1.5 Reading compiler errors

Let's create a deliberate mistake to see how Rust handles errors:

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

Rust error messages (diagnostics) follow a consistent format. They include an error code (like `E0615`), the exact file location, an explanation of the problem, and often a `help:` section with a suggested fix.

Build these two habits immediately: 
1. **Read the entire error message.** The solution is often at the very end. 
2. **Ask the compiler for help.** If you do not understand an error, run `rustc --explain <Error_Code>` (for example, `rustc --explain E0615`) to get more details. 

Compiler messages are excellent learning tools, so this guide uses them frequently.

## 1.6 A brief look at the syntax

This example introduces basic Rust syntax to help you read the code in later chapters. We will explain each concept in detail when it becomes relevant:

```rust
// Parameter types follow their names; `->` introduces the return type.
fn add(a: i32, b: i32) -> i32 {
    // The final expression is returned automatically because it has no semicolon.
    a + b
}

fn main() {
    // Local types are usually inferred.
    let x = add(2, 3);

    // You must declare variables as mutable (`mut`) if you want to change them.
    let mut count = 0;
    count += 1;

    // `if` is an expression, meaning it produces a value.
    let label = if x > 4 { "big" } else { "small" };

    // Half-open ranges exclude the upper bound: this prints 0, 1, 2.
    for i in 0..3 {
        println!("{i}: {label}, count={count}");
    }
}
```

Keep these three details in mind: 
1. Types are static but usually inferred, so you rarely need to annotate local variables. 
2. `if` statements act as expressions and produce values, which replaces the need for a ternary operator (like `? :`). 
3. Numeric conversions are explicit: an `i32` will not silently convert into an `i64` or `f64`. 

The next chapter will map out these differences more clearly.

## Summary

You now have a working Rust toolchain, a new project, a dependency, and the basic commands. Use `cargo check` frequently as you write code, and use `cargo run` to test the output. Run `cargo fmt` and `cargo clippy` before saving your work, and use `--release` builds to test performance. Finally, keep rust-analyzer running in your editor. The next chapter explains how Rust differs from other programming languages.
