# Quick start

This chapter helps you get started with the Rust programming language. By the end of this chapter, you will be able to do the following:
* Install Rust.
* Run a basic program.
* Add an external dependency.
* Use common Cargo commands.

## Install Rust

To install Rust, use `rustup`, which is the official toolchain manager for Rust.

### On macOS and Linux
Run the following command in your terminal:
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### On Windows
Download and run the `rustup-init.exe` installer from [rustup.rs](https://rustup.rs/).

### Included tools
The installation includes the following tools:
* **`rustc`**: The Rust compiler. In most cases, you run the compiler through Cargo rather than running it directly.
* **`cargo`**: The build tool, package manager, and test runner.
* **`rustfmt`**: The code formatter that applies the standard Rust code style.
* **`clippy`**: The linter that checks your code for common mistakes and unidiomatic patterns.

To update your Rust installation to the latest version, run the following command:
```bash
rustup update
```

### Editor support
To get features like type hints, syntax checking, and code navigation in your editor, install **rust-analyzer**. This is the official Rust language server. If you use VS Code, install the extension named **rust-analyzer**. 

> **If you come from Node.js or Python:** 
> In Node.js or Python, you often use separate tools for package management, building, testing, and formatting. In Rust, Cargo handles all of these tasks.

## Create your first project

To create and run a new Rust project, run the following commands in your terminal:

```bash
cargo new hello
cd hello
cargo run
```

Your terminal shows output similar to the following:
```text
   Compiling hello v0.1.0
    Finished `dev` profile
     Running `target/debug/hello`
Hello, world!
```

### Project structure
The `cargo new` command creates a directory named `hello` with the following structure:
```text
hello/
├── Cargo.toml        The project manifest (metadata and dependencies)
└── src/
    └── main.rs       The source code file (contains the main function)
```

### The `Cargo.toml` file
The `Cargo.toml` file contains the configuration for your project:
```toml
[package]
name = "hello"
version = "0.1.0"
edition = "2024"        # The current default edition for new projects

[dependencies]
```

### The `src/main.rs` file
The `src/main.rs` file contains the entry point of your program:
```rust
fn main() {
    println!("Hello, world!");
}
```

In Rust, `println!` is a **macro**, not a regular function. The exclamation mark (`!`) identifies it as a macro. At this stage, you can think of it as a function that prints text to the screen.

To format text, use curly braces `{}` as placeholders:
```rust
println!("{} + {} = {}", 2, 3, 2 + 3);
```

You can also place variables directly inside the placeholders:
```rust
println!("{name}");
```

## Common Cargo commands

The following table lists the Cargo commands you use most often during daily development:

| Command | Action | Usage |
| :--- | :--- | :--- |
| `cargo check` | Checks for compilation and type errors. Does not produce a binary file. | Use during development for quick feedback. |
| `cargo run` | Builds and runs your program. | Use to verify program behavior. |
| `cargo test` | Runs tests in your project. | Use when you add tests (see Chapter 8). |
| `cargo build --release` | Creates an optimized binary in the `target/release/` directory. | Use for production deployment or when measuring performance. |
| `cargo fmt` | Formats all code in the project to match the standard style. | Run before you commit code or in your continuous integration (CI) pipeline. |
| `cargo clippy` | Checks your code for common mistakes and styling issues. | Run regularly during development and in your CI pipeline. |
| `cargo doc --open` | Generates HTML documentation for your project and opens it in a browser. | Use to explore APIs and code structure. |

> **Common mistake**:
> Do not use a debug build to measure performance. Debug builds prioritize fast compilation and detailed diagnostics, making them much slower. Always use a release build (`cargo build --release`) to measure performance.

## Add a dependency

In Rust, external libraries are called **crates**. These crates are hosted on [crates.io](https://crates.io/), the community package registry.

To add a crate to your project, run the `cargo add` command. For example, to add the `rand` crate for generating random numbers:
```bash
cargo add rand
```

This command adds the specified crate to the `[dependencies]` section of your `Cargo.toml` file.

The following example shows how to use the `rand` crate:
```rust
fn main() {
    let n: u32 = rand::random_range(1..=100);
    println!("secret number: {n}");
}
```

When you run `cargo run`, Cargo automatically downloads, compiles, and caches the dependency. Cargo resolves crate versions using semantic versioning (SemVer) rules. It records the exact version of every dependency in a file named `Cargo.lock`. 

Always commit the `Cargo.lock` file to your version control system (such as Git). This ensures that your local development environment, CI system, and production builds use the exact same dependency versions.

### Key concepts and naming conventions
* **Binary and Library Crates**: A project is a **binary crate** if it contains `src/main.rs`. It is a **library crate** if it contains `src/lib.rs`. A single Rust package can contain both.
* **Naming Conventions**: Rust uses `snake_case` for functions, variables, and module names. It uses `CamelCase` (also called `PascalCase`) for types, such as structs and enums. The compiler displays warning messages if your code does not follow these naming rules.

## Read compiler errors

The Rust compiler provides detailed and helpful error messages. Consider the following example with an error:

```rust
fn main() {
    let name = String::from("ada");

    // This refers to the method instead of calling it because it lacks parentheses.
    let upper = name.to_uppercase;
}
```

If you compile this code, the compiler displays the following error:

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

### Anatomy of a Rust compiler error
Rust compiler errors follow a consistent structure:
* **Error code**: A unique code, such as `E0615`.
* **Source location**: The exact file, line number, and character position of the error.
* **Explanation**: A description of what caused the error.
* **Help section**: Suggested changes to fix the issue.

### Recommended practices
* **Read the full message**: Always read the compiler error message to the end. The most useful details and solutions are often at the bottom.
* **Use the explain tool**: If an error message is not clear, run the `rustc --explain` command followed by the error code. For example:
  ```bash
  rustc --explain E0615
  ```
  This command displays a detailed explanation and code examples to help you understand the error category.

## Basic syntax

The following example demonstrates several common Rust syntax patterns. These concepts are explained in detail in later chapters:

```rust
// Parameter types follow their names. The `->` syntax specifies the return type.
fn add(a: i32, b: i32) -> i32 {
    // If you omit the semicolon from the final expression, Rust returns its value.
    a + b
}

fn main() {
    // Rust infers the types of local variables automatically.
    let x = add(2, 3);

    // Variables are immutable by default. Use `mut` to allow modification.
    let mut count = 0;
    count += 1;

    // `if` is an expression that returns a value.
    let label = if x > 4 { "big" } else { "small" };

    // Half-open ranges (0..3) exclude the upper bound (returns 0, 1, and 2).
    for i in 0..3 {
        println!("{i}: {label}, count={count}");
    }
}
```

### Key takeaways
Keep the following concepts in mind as you begin writing Rust:
* **Type inference**: Rust is statically typed, but the compiler can infer most local variable types. You do not need to add type annotations to every local variable.
* **No ternary operator**: Because `if` statements are expressions, they return a value directly. This removes the need for a separate ternary operator.
* **Explicit conversions**: Rust does not perform implicit numeric conversions. For example, the compiler does not silently convert an `i32` to an `i64` or an `f64`. You must convert types explicitly.

## Summary

This chapter covered the essential tools and workflow for Rust development.

### Summary checklist
* Use `cargo check` regularly during development for quick feedback on your code.
* Use `cargo run` to build and run your program.
* Run `cargo fmt` and `cargo clippy` before you commit code to ensure standard formatting and style.
* Use release builds (`cargo build --release`) for production deployments and performance testing.
* Keep **rust-analyzer** enabled in your editor for type hints and navigation.

The next chapter maps the primary differences between Rust and other programming languages.
