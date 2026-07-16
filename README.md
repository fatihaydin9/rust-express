# Rust Express

Rust Express is a practical introduction to Rust for programmers familiar with languages like Python, JavaScript, Go, Java, C#, or C++. It follows how an experienced Rust developer helps a teammate learn the language: build projects quickly, explain differences from familiar languages, and focus on everyday development concepts.

These concepts include ownership, error handling, traits, concurrency, testing, and project structure.

## Who this is for

Prerequisites include basic programming concepts like functions, types, threads, and package managers. You do **not** need experience with Rust, systems programming, or manual memory management.

If Rust behaves differently from other languages, this guide explains why. We use comparisons as learning tools rather than exact matches.

## What this is not

This guide is not a comprehensive language reference. It covers common syntax—like loops, string methods, and patterns—but it does not document every available feature. Instead, it focuses on concepts that change how programmers think about code.

## Structure

The guide has 12 chapters. Read chapters 1–9 in order, as later examples build on earlier concepts. Chapters 10–12 are reference materials; you can read them after chapter 8 and revisit them during real projects.

| # | Chapter | What you get |
|---|---|---|
| 1 | [Quick Start](01-quick-start.md) | The toolchain, a first program, and everyday Cargo commands |
| 2 | [What Makes Rust Different](02-what-makes-rust-different.md) | An overview of the main mental shifts, with links to later chapters |
| 3 | [Ownership and Borrowing](03-ownership-and-borrowing.md) | Moves, references, lifetimes, and the difference between `String` and `&str` |
| 4 | [Modeling Data with Types](04-modeling-data-with-types.md) | Structs, enums, `Option`, pattern matching, iterators, and newtypes |
| 5 | [Error Handling](05-error-handling.md) | `Result`, `?`, error design, `thiserror`, and `anyhow` |
| 6 | [Traits and Generics](06-traits-and-generics.md) | Traits, static and dynamic dispatch, conversions, and interface ownership |
| 7 | [Concurrency and Async](07-concurrency-and-async.md) | Threads, locks, channels, `Send`/`Sync`, Tokio, and common async mistakes |
| 8 | [Projects, Tooling, and Tests](08-projects-tooling-and-tests.md) | Modules, crates, workspaces, layered architecture, and testing tools |
| 9 | [Capstone: A Real Service](09-capstone.md) | The earlier concepts combined in a small service design |
| 10 | [The Ecosystem: Recommended Crates](10-ecosystem-and-crates.md) | Common libraries and a practical way to evaluate unfamiliar crates |
| 11 | [Project Layout and Architecture](11-project-layout-and-architecture.md) | Example layouts for a CLI, a library, and a service |
| 12 | [Best Practices](12-best-practices.md) | Guidance for APIs, performance, security, dependencies, concurrency, and CI |

## Conventions used in the text

Code blocks are small but complete. If a code snippet requires a `use` declaration, the guide includes or mentions it.

The guide includes frequent compiler output to help you learn Rust diagnostics. Read the entire error message, as the best suggestions are often at the end. Note that exact wording might vary between compiler versions.

Three recurring markers appear throughout the guide:

> **If you come from …** — Connects Rust concepts to familiar languages.

> **Common mistake:** — Highlights a pattern that often causes confusion or avoidable problems.

> **Try it:** — Suggests a small experiment to demonstrate a behavior.

## A note about the learning curve

Ownership and borrowing might feel restrictive initially, especially if familiar patterns do not work directly. This gets easier once the ownership model guides how you structure data and APIs.

While the compiler remains strict, its messages become easier to understand. You will learn to recognize which designs naturally fit Rust. This guide aims to make this transition smooth and less frustrating.
