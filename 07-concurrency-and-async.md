# Chapter 7: Concurrency and Async

Chapter 2 made a strong claim: safe Rust rejects data races at compile time. This chapter
explains the mechanism and introduces the practical tools built around it, including
threads, locks, channels, async functions, and Tokio. It closes with common mistakes because
Rust eliminates data races, not every possible concurrency bug.

## 7.1 What a data race is, and why it's the worst bug

A **data race** occurs when two threads access the same memory concurrently, at least one
access writes, and no synchronization protects the operation. The result may be incorrect
values, memory corruption, or other undefined behavior. These bugs are especially difficult
because they are intermittent: they can disappear under a debugger and reappear only under
production load. Many languages accept such programs and rely on testing or runtime tools to
find the problem later.

Watch Rust decline the classic version:

```rust
use std::thread;

fn main() {
    let mut counter = 0;
    let mut handles = Vec::new();

    for _ in 0..8 {
        handles.push(thread::spawn(move || {
            // Each closure attempts to take ownership of the same value.
            counter += 1;
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

```text
error[E0382]: use of moved value: `counter`
  |  value moved into closure here, in previous iteration of loop
```

Decode it with chapter 3: `move` gives the closure _ownership_ of `counter`, and one value
has one owner — so the second thread can't also own it. To share across threads you must
**choose a sharing strategy, explicitly, in the types.** Accidental ambient sharing — the
raw material of races — has no syntax. The next two sections are the two strategies.

## 7.2 Strategy A: shared state, behind a lock the compiler respects

Two tools compose. `Arc<T>` ("atomically reference-counted") is shared ownership,
thread-safe edition: `Arc::clone` hands out additional owners; the value frees when the last
drops. `Mutex<T>` holds the value _inside_ and makes threads take turns: `.lock()` blocks
until it's your turn and returns a guard.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = Vec::new();

    for _ in 0..8 {
        // Each thread receives another shared-ownership handle.
        let counter = Arc::clone(&counter);

        handles.push(thread::spawn(move || {
            // Wait for exclusive access to the value.
            let mut value = counter.lock().unwrap();
            *value += 1;
        })); // The guard drops here and unlocks the mutex.
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("{}", *counter.lock().unwrap()); // Always 8.
}
```

Two design details matter. First, **the lock owns the data**: `Mutex<T>` wraps the protected
value, and access is available only through `lock()`. The relationship between state and
synchronization is therefore encoded in the type rather than maintained by convention.

Second, **unlocking is automatic**. The guard releases the mutex when it is dropped, so
early returns do not accidentally leave the lock held. The `.unwrap()` handles lock
poisoning, which records that another thread panicked while holding the mutex.

When reads vastly outnumber writes, `RwLock<T>` allows many readers or one writer — THE rule
from chapter 3, enforced at runtime across threads.

## 7.3 Why the race can't compile: `Send` and `Sync`

The guarantee is built on two automatically derived marker traits. **`Send`** means a value
can be moved safely to another thread, and **`Sync`** means shared references can be used
safely across threads. `thread::spawn` and `tokio::spawn` require the captured state to
satisfy the relevant bounds, and the compiler checks those bounds transitively through every
field.

For example, `Rc<T>` is not `Send` because its reference count is not atomic. Safe Rust
therefore rejects the type shapes that could permit a data race. This is stronger than a
race detector: the invalid program does not build.

Those `Send + Sync` bounds you wrote on chapter 6's traits were exactly this contract:
"implementations must be usable from a multithreaded server," compiler-enforced.

> **Try it:** replace `Arc` with `Rc` in the counter example. The error —
> ``Rc<Mutex<i32>>` cannot be sent between threads safely`` — is the trait system catching,
> at compile time, a subtle refcount race you'd otherwise meet in production.

## 7.4 Strategy B: don't share — send

The alternative to guarding shared state is having none: **channels** move values between
threads, and move semantics make the handoff airtight.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (sender, receiver) = mpsc::channel();

    thread::spawn(move || {
        // The receiver takes ownership of each job.
        for job in receiver {
            println!("processing {job}");
        }
    });

    for job in ["a", "b", "c"] {
        // Ownership of the string moves into the channel.
        sender.send(job.to_string()).unwrap();
    }
}
```

After `send`, ownership has moved into the channel, so the sender cannot continue mutating
the same value. The handoff is enforced rather than documented.

Use **channels for flow**, such as pipelines, worker pools, and fan-out. Use
**`Arc<Mutex<T>>` for small pieces of shared state**, such as counters or configuration
snapshots. When most of a codebase becomes wrapped in `Arc<Mutex<…>>`, treat that as a sign
that ownership boundaries have not been designed clearly.

Two more std tools worth knowing: `std::thread::scope` lets threads borrow from the parent
stack safely (the compiler proves they join before the data dies), and for CPU-parallel data
crunching the `rayon` crate turns `.iter()` into `.par_iter()` — data parallelism in one
token, safe for the same `Send`/`Sync` reasons.

## 7.5 Async: concurrency for waiting

Threads suit work that _computes_. Servers mostly _wait_ — ten thousand connections, each
idle between packets — and ten thousand OS threads is a heavy way to wait. **Async** is the
light way: an `async fn` compiles into a **future** — a paused state machine describing work
— and a runtime (the de-facto standard: **Tokio**) multiplexes huge numbers of them over a
small thread pool. `.await` marks each point where a task says "I'm waiting; run someone
else."

```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let listener = TcpListener::bind("0.0.0.0:8080").await?;

    loop {
        // Waiting for a connection yields control to another task.
        let (socket, _) = listener.accept().await?;

        // Each connection runs in a lightweight Tokio task.
        tokio::spawn(async move {
            if let Err(error) = handle(socket).await {
                eprintln!("connection error: {error}");
            }
        });
    }
}
```

Utilities you'll use daily: `tokio::join!(a, b)` awaits several futures concurrently;
`tokio::select!` races them (first to finish wins — timeouts, cancellation);
`tokio::time::timeout(dur, fut)` wraps any future with a deadline; async channels live in
`tokio::sync` (`mpsc`, plus `oneshot` for request/response and `broadcast` for fan-out).

Three async details often require adjustment.

1. **Futures are lazy.** Constructing a future does not execute it; `.await` or `spawn`
   drives it forward.
2. **Spawned tasks usually must be `Send`.** Tokio may move them between worker threads, so
   the same cross-thread checks still apply.
3. **Cancellation is dropping.** A future that loses a `select!`, reaches a timeout, or
   belongs to a disconnected client may be dropped while suspended. Mandatory cleanup
   belongs in RAII guards, and multi-step invariants need transactional protection.

> **If you come from JS/Python:** same `async/await` surface, two differences underneath —
> laziness (no "fire and it runs anyway"), and true multi-threaded execution (hence `Send`).
> **If you come from Go:** `tokio::spawn` ≈ `go`, Tokio channels ≈ Go channels; the
> difference is that `async fn` is explicit in the type system ("colored functions") rather
> than invisible in goroutines — a real trade-off, with the compensation that blocking vs.
> async is visible in every signature.

## 7.6 The two async foot-guns

Most production async pain reduces to two mistakes; learn them here.

**Don't block the runtime.** A synchronous slow call inside an async task — a blocking DB
driver, heavy CPU work, `std::thread::sleep` — freezes one of the runtime's few worker
threads, stalling _every_ task scheduled there. Route such work through
`tokio::task::spawn_blocking(|| …)`; sleep with `tokio::time::sleep`.

**Don't hold a `std` lock across an `.await`.** Task A takes a `std::sync::Mutex` and
awaits; the runtime schedules task B on the same thread; B tries to lock; B is blocking the
very thread A needs to resume and release. Deadlock — the async classic. Either scope the
guard to drop _before_ the await (an inner `{ }` block), or use `tokio::sync::Mutex` when
the critical section genuinely spans one.

And the promise's honest boundary, stated once: Rust eliminates **data races**, not all
concurrency bugs. Deadlocks, ordering mistakes between separately-locked operations,
starvation — still yours to design away. Rust removed the heisenbugs; the systems thinking
stays.

## Summary

Rust extends ownership across threads. Shared state must be chosen explicitly, and the
chosen strategy is visible in the types: `Arc<Mutex<T>>` represents guarded shared state, a
channel represents ownership transfer, and a `Send` future can migrate between worker
threads. `Send` and `Sync` make data-race-prone shapes invalid in safe code.

Use channels for flow, locks for small shared state, and `spawn_blocking` for synchronous or
CPU-heavy work inside an async application. Do not hold a standard-library lock guard across
`.await`. The next chapter moves from individual operations to whole-project structure.
