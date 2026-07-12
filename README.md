# Rust Express

Rust Express is a practical introduction to Rust for programmers coming from Python,
JavaScript, Go, Java, C#, C++, or another language. It follows the path an experienced
Rust developer might use when helping a teammate get started: build something quickly,
explain where Rust differs from familiar languages, and spend more time on the ideas
that shape everyday development.

Those ideas include ownership, error handling, traits, concurrency, testing, and project
structure.

## Who this is for

You should already be comfortable with general programming concepts such as functions,
types, threads, and package managers. You do **not** need previous experience with Rust,
systems programming, or manual memory management.

When Rust behaves differently from languages you may know, the guide explains the
difference and the reason behind it. The comparisons are learning aids rather than exact
equivalences.

## What this is not

This is not a complete language reference. It uses common syntax, including loops,
string methods, and patterns, without cataloging every available form. The focus remains
on the concepts that usually require a change in how programmers think about code.

## Structure

The guide contains twelve chapters. Chapters 1–9 form the main course and are best read
in order because later examples build on earlier concepts. Chapters 10–12 work more like
reference material: you can read them after chapter 8 and revisit them while working on
real projects.

| #   | Chapter                                                                          | What you get                                                                   |
| --- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| 1   | [Quick Start](01-quick-start.md)                                                 | The toolchain, a first program, and the Cargo commands used every day          |
| 2   | [What Makes Rust Different](02-what-makes-rust-different.md)                     | An overview of the main mental shifts, with links to later chapters            |
| 3   | [Ownership and Borrowing](03-ownership-and-borrowing.md)                         | Moves, references, lifetimes, and the difference between `String` and `&str`  |
| 4   | [Modeling Data with Types](04-modeling-data-with-types.md)                       | Structs, enums, `Option`, pattern matching, iterators, and newtypes            |
| 5   | [Error Handling](05-error-handling.md)                                           | `Result`, `?`, error design, `thiserror`, and `anyhow`                         |
| 6   | [Traits and Generics](06-traits-and-generics.md)                                 | Traits, static and dynamic dispatch, conversions, and interface ownership      |
| 7   | [Concurrency and Async](07-concurrency-and-async.md)                             | Threads, locks, channels, `Send`/`Sync`, Tokio, and common async mistakes      |
| 8   | [Projects, Tooling, and Tests](08-projects-tooling-and-tests.md)                 | Modules, crates, workspaces, layered architecture, and testing tools           |
| 9   | [Capstone: A Real Service](09-capstone.md)                                       | The earlier ideas combined in a small service design                           |
| 10  | [The Ecosystem: Recommended Crates](10-ecosystem-and-crates.md)                  | Common libraries and a practical way to evaluate unfamiliar crates            |
| 11  | [Project Layout and Architecture](11-project-layout-and-architecture.md)         | Example layouts for a CLI, a library, and a service                            |
| 12  | [Best Practices](12-best-practices.md)                                           | Guidance for APIs, performance, security, dependencies, concurrency, and CI    |

## Conventions used in the text

Code blocks are intentionally small, but they do not omit requirements that affect the
example. When a snippet needs a `use` declaration, the guide shows it or states that it
is required.

Compiler output appears often because learning to read Rust diagnostics is part of
learning the language. Read the complete message rather than stopping at its first line;
the most useful suggestion is often near the end. Exact wording may vary slightly
between compiler versions.

Three recurring markers appear throughout the guide:

> **If you come from …** — a comparison that connects Rust to a familiar language.

> **Common mistake:** — a pattern that often causes confusion or avoidable problems.

> **Try it:** — a small experiment that helps make the behavior concrete.

## A note about the learning curve

Ownership and borrowing can feel restrictive at first, especially when familiar designs
do not translate directly. This becomes easier as the ownership model starts to guide
the way you structure data and APIs.

The compiler does not become less strict, but its messages become easier to interpret,
and you begin to recognize which designs fit the model naturally. This guide is intended
to make that transition more understandable and less frustrating.
