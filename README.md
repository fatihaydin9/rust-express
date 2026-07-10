# Rust for Programmers

This is a practical introduction to Rust for programmers coming from Python, JavaScript, Go,
Java, C#, C++, or another language. It follows the kind of path an experienced Rust
developer might use when onboarding a teammate: start quickly, make the language's
differences explicit, and spend time on the concepts that determine real productivity. Those
concepts include ownership, error handling, traits, concurrency, and project structure.

## Who this is for

You can already program and understand concepts such as functions, types, threads, and
package managers. You do **not** need prior experience with Rust, systems programming, or
manual memory management. Whenever Rust behaves differently from languages you may know, the
text explains that difference directly and gives the reason behind it.

## What this is not

This is not a complete language reference. It uses ordinary syntax—such as loop forms,
string methods, and pattern syntax—without cataloging every variation. The focus stays on
ideas that require a deeper mental shift.

## Structure

Thirteen documents in two parts. Chapters 1–9 form the course: read them in order, because
later chapters use earlier ones constantly. Chapters 10–12 are reference material: dip into
them any time after chapter 8, and return to them while building real projects.

| #   | Chapter                                                                          | What you get                                                                    |
| --- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| 1   | [Quick Start](01-quick-start.md)                                                 | Toolchain installed, first program running, the daily commands                  |
| 2   | [What Makes Rust Different](02-what-makes-rust-different.md)                     | The complete map of differences, each with a pointer to its chapter             |
| 3   | [Ownership and Borrowing](03-ownership-and-borrowing.md)                         | The memory model: moves, references, lifetimes, `String` vs `&str`              |
| 4   | [Modeling Data with Types](04-modeling-data-with-types.md)                       | Structs, enums, `Option`, pattern matching, iterators, newtypes                 |
| 5   | [Error Handling](05-error-handling.md)                                           | `Result`, `?`, designing error types, `thiserror` and `anyhow`                  |
| 6   | [Traits and Generics](06-traits-and-generics.md)                                 | Contracts, zero-cost generics, `dyn`, and who should own an interface           |
| 7   | [Concurrency and Async](07-concurrency-and-async.md)                             | Threads, locks, channels, `Send`/`Sync`, Tokio, the classic mistakes            |
| 8   | [Projects, Tooling, and Tests](08-projects-tooling-and-tests.md)                 | Modules, crates, workspaces, layered architecture, the test toolkit             |
| 9   | [Capstone: A Real Service](09-capstone.md)                                       | Everything combined in one small, realistic design                              |
| 10  | [The Ecosystem: Recommended Crates](10-ecosystem-and-crates.md)                  | Which library to use for what, and how to judge an unfamiliar crate             |
| 11  | [Project Layout and Architecture](11-project-layout-and-architecture.md)         | Copyable file structures (CLI, library, service) and recurring design patterns  |
| 12  | [Best Practices](12-best-practices.md)                                           | API design, performance, security, dependency hygiene, CI — the checklist       |

## Conventions used in the text

Code blocks are intentionally small, but they do not hide important requirements. When an
example needs a `use` declaration, it is shown or explicitly noted. Compiler output also
appears frequently because Rust's diagnostics are part of the learning process; read them to
the end rather than stopping at the first line.

Three recurring markers appear throughout the guide:

> **If you come from …** — a direct comparison to another language's behavior.

> **Common mistake:** — an error most newcomers make; reading these saves you hours.

> **Try it:** — a small experiment: change the code, recompile, watch what happens.

## One honest sentence about the learning curve

During the first one or two weeks, many Rust programmers feel as though they are arguing
with the compiler about ownership. That experience is normal and temporary. It ends not
because the compiler becomes less strict, but because the ownership model becomes intuitive
and you stop reaching for designs it rejects. This guide aims to shorten that transition as
much as written material can.
