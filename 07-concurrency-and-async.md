# Concurrency and async

Safe Rust prevents data races at compile time. This chapter explains the mechanisms behind this guarantee and introduces the standard tools for writing safe, concurrent, and asynchronous code in Rust.

By the end of this chapter, you will understand:
* What data races are and how Rust's ownership model prevents them.
* How to share state across threads using `Arc` and `Mutex`.
* How the compiler enforces thread safety using the `Send` and `Sync` traits.
* How to pass messages between threads using channels.
* How to write lightweight, non-blocking code using `async/await` and the Tokio runtime.
* How to avoid common asynchronous performance bottlenecks and deadlocks.

## Data races and thread safety

A **data race** occurs when two or more threads access the same memory location concurrently without synchronization, and at least one of the accesses is a write operation. Data races can cause undefined behavior, including memory corruption and incorrect calculations.

These bugs are difficult to diagnose because they are intermittent. They often disappear during debugging and reappear under production load.

Consider the following example that attempts to modify a shared counter across multiple threads:

```rust
use std::thread;

fn main() {
    let mut counter = 0;
    let mut handles = Vec::new();

    for _ in 0..8 {
        handles.push(thread::spawn(move || {
            // Each thread attempts to take ownership of the same counter.
            counter += 1;
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

If you compile this code, the compiler displays an error:

```text
error[E0382]: use of moved value: `counter`
  |  value moved into closure here, in previous iteration of loop
```

This error occurs because the `move` keyword transfers ownership of `counter` into the thread closure. However, Rust's ownership rules specify that a value can have only one owner at a time. You cannot move the same value into multiple threads.

To share access safely, you must use a synchronization strategy encoded in your program's types.

## Shared state using Arc and Mutex

To share mutable data across multiple threads, you can combine two standard library types:
* **`Arc<T>` (Atomically Reference Counted)**: Provides shared read-only ownership across threads. It tracks the number of active handles. When the last handle is dropped, Rust releases the value's memory.
* **`Mutex<T>` (Mutual Exclusion)**: Guarantees that only one thread can access the inner data at any given time.

The following example demonstrates how to use `Arc` and `Mutex` together to safely update a counter:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // Wrap the integer in a Mutex, then in an Arc for shared ownership.
    let counter = Arc::new(Mutex::new(0));
    let mut handles = Vec::new();

    for _ in 0..8 {
        // Create a new reference-counted pointer to the counter.
        let counter = Arc::clone(&counter);

        handles.push(thread::spawn(move || {
            // Request exclusive access to the counter.
            let mut value = counter.lock().unwrap();
            *value += 1;
        })); // The lock guard is dropped and the mutex is unlocked automatically.
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("{}", *counter.lock().unwrap()); // Prints 8
}
```

### Key design features
* **Type-enforced synchronization**: The `Mutex<T>` wraps the protected data. You can only access the data by calling the `.lock()` method. This ensures that you cannot bypass the lock by mistake.
* **Automatic unlocking**: The `.lock()` method returns a guard type. When this guard goes out of scope, Rust automatically drops it and unlocks the mutex. This prevents deadlocks caused by forgetting to unlock.
* **Lock poisoning handling**: The `.unwrap()` call handles lock poisoning. If a thread panics while holding a lock, the mutex becomes poisoned to prevent other threads from accessing potentially inconsistent data.

If you have data where read operations are much more common than write operations, use **`RwLock<T>`** (Reader-Writer Lock). `RwLock` allows multiple threads to read the data concurrently, but only one thread to write.

## The Send and Sync traits

Rust's compiler prevents data races at compile time using two built-in traits:
* **`Send`**: Indicates that a type can safely transfer ownership across thread boundaries.
* **`Sync`**: Indicates that multiple threads can safely access a type through shared references.

These traits are **marker traits**, which means they do not define any methods. Instead, the compiler automatically implements them for your custom types if all of their fields implement `Send` and `Sync`.

### Example: Rc vs. Arc
The thread-local reference counting pointer `Rc<T>` does not implement `Send` or `Sync` because it updates its reference count using non-atomic operations. If you attempt to share an `Rc` across threads, the compiler generates an error:

> **Try it**:
> Replace `Arc` with `Rc` in the counter example. The compilation fails with an error indicating that `Rc<Mutex<i32>>` cannot be sent between threads safely.

By enforcing these traits, Rust rejects unsafe memory structures before your program runs.

## Message passing with channels

An alternative to sharing state with locks is to transfer ownership of data across threads using **channels**. This approach ensures that you do not share memory directly between threads.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    // Create a Multi-Producer, Single-Consumer (mpsc) channel.
    let (sender, receiver) = mpsc::channel();

    thread::spawn(move || {
        // The receiving thread takes ownership of each job.
        for job in receiver {
            println!("processing {job}");
        }
    });

    for job in ["a", "b", "c"] {
        // Ownership of the String moves into the channel.
        sender.send(job.to_string()).unwrap();
    }
}
```

When you call `send`, ownership of the value moves into the channel. The sender cannot modify or access the value after sending it.

### Guidelines for choosing a concurrency strategy
* Use **channels** for flow-control patterns, such as processing pipelines, background worker pools, or distributing tasks.
* Use **`Arc<Mutex<T>>`** for small, shared state, such as counters, shared configurations, or database connection handles. Avoid wrapping large portions of your codebase in `Arc<Mutex<T>>`.

### Additional concurrency tools
* **`std::thread::scope`**: Allows threads to borrow data from the parent function's stack safely. The compiler guarantees that all scoped threads finish executing before the parent data goes out of scope.
* **`rayon` crate**: Provides a simple way to perform parallel data processing on collections. It allows you to convert standard iterators into parallel iterators by replacing `.iter()` with `.par_iter()`.

## Asynchronous programming

Operating system threads are well-suited for CPU-heavy tasks. However, if your program spends most of its time waiting for network or disk input/output (I/O), creating a separate thread for every connection can use excessive memory and CPU resources.

**Asynchronous programming** (async) provides a more lightweight model:
* An **`async fn`** compiles into a **Future** (a state machine that describes a unit of work).
* An asynchronous runtime (such as **Tokio**) schedules and executes many futures across a small pool of worker threads.
* The **`.await`** keyword pauses execution of the current future when it must wait for an I/O operation, allowing the runtime to run other tasks on the same thread.

The following example demonstrates a basic asynchronous TCP server using Tokio:

```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let listener = TcpListener::bind("0.0.0.0:8080").await?;

    loop {
        // Waiting for a connection yields control to the runtime.
        let (socket, _) = listener.accept().await?;

        // Run each connection in a lightweight, independent task.
        tokio::spawn(async move {
            if let Err(error) = handle(socket).await {
                eprintln!("connection error: {error}");
            }
        });
    }
}
```

### Essential Tokio utilities
* **`tokio::join!`**: Awaits multiple futures concurrently.
* **`tokio::select!`**: Races multiple futures concurrently and executes the block associated with the first future to complete.
* **`tokio::time::timeout`**: Wraps a future with a deadline, returning an error if the future does not complete within the specified duration.
* **Asynchronous channels (`tokio::sync`)**: Includes `mpsc` (multi-producer, single-consumer), `oneshot` (one-to-one messaging), and `broadcast` (one-to-many fan-out).

### Core characteristics of Rust futures
1. **Lazy evaluation**: Futures do not execute automatically when created. You must use `.await` or pass them to a runner (such as `tokio::spawn`) to execute them.
2. **Task migration**: Tokio can move tasks between different worker threads. Therefore, any variables captured by a spawned task must implement the `Send` trait.
3. **Cancellation via dropping**: If a future is cancelled (for example, if it loses a `tokio::select!` race or times out), Rust drops the future immediately. Always place mandatory cleanup in RAII guards to ensure it runs even if a task is cancelled.

> **If you come from JavaScript or Python:**
> Although the `async/await` syntax looks similar, Rust futures are lazy. They do not start running immediately when you call an async function. Additionally, because Rust's async runtime can run tasks on multiple threads, your async variables must implement `Send`.
>
> **If you come from Go:**
> While `tokio::spawn` is similar to starting a goroutine and Tokio channels are similar to Go channels, Rust requires you to explicitly declare async functions. This makes it clear which functions can block and pause execution.

## Common asynchronous mistakes

Although the compiler prevents data races, you must still design your code to avoid standard concurrency issues like deadlocks and performance degradation.

### 1. Blocking the runtime threads
If you execute a long-running synchronous function (such as synchronous disk I/O, a blocking database call, or heavy mathematical calculations) inside an async function, you block the underlying worker thread. This delays all other tasks scheduled on that thread.

* **Solution**: Wrap blocking operations in `tokio::task::spawn_blocking`. Use asynchronous alternatives (such as `tokio::time::sleep` instead of `std::thread::sleep`) where available.

```rust
// Do not do this:
// std::thread::sleep(std::time::Duration::from_secs(1));

// Do this instead:
tokio::time::sleep(std::time::Duration::from_secs(1)).await;
```

### 2. Holding a synchronous lock across an `.await` boundary
If you acquire a standard library mutex lock (`std::sync::MutexGuard`) and then use `.await`, you can cause a deadlock.

For example, Task A acquires the lock and pauses at an `.await` boundary. The runtime schedules Task B on the same thread. If Task B attempts to acquire the same lock, it blocks the thread. Because the thread is blocked, Task A cannot resume to release the lock.

* **Solution**: Scope the lock so that the guard is dropped before the `.await` boundary (for example, by wrapping the critical section in a code block `{}`). If the lock must span across an `.await` call, use Tokio's asynchronous mutex (`tokio::sync::Mutex`) instead.

## Summary

This chapter discussed Rust's multi-threading and asynchronous programming models:
* **Compile-time safety**: Rust uses the `Send` and `Sync` traits to prevent data races before your program compiles.
* **Shared state**: Use `Arc<Mutex<T>>` to protect shared variables, and ensure that unlocking happens automatically through RAII lock guards.
* **Message passing**: Use channels to transfer data ownership between threads.
* **Asynchronous execution**: Use `async/await` and the Tokio runtime to handle I/O-heavy workloads efficiently without allocating excess operating system threads.
* **Avoid pitfalls**: Do not block the runtime threads with synchronous code, and do not hold synchronous locks across `.await` boundaries.
