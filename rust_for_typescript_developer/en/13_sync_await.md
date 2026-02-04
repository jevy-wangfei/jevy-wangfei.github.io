This is a Rust **async/await** guide tailored for experienced TypeScript developers.

As a senior TS developer you know the Event Loop, Promises, and the microtask queue. Rust’s async model is **fundamentally different** from JS/TS.

In TS, async is a **runtime** feature.
In Rust, async is **compile-time** state-machine rewriting plus a small runtime driver.

We’ll cover **concepts**, **implementation (traits)**, and **comparison** with TS/Go.

---

# Async programming in Rust: async/await

## 1. Why async?

### 1.1 Blocking vs non-blocking

With synchronous I/O (e.g. reading a file, network request), the thread blocks until data is ready.

- **Cost:** Threads are expensive (e.g. ~2MB per thread, context-switch overhead). With 10k connections you can’t afford 10k threads (C10k problem).

### 1.2 Async vs concurrency vs parallelism

- **Concurrency:** Logically doing several things at once (one CPU switching between tasks).
- **Parallelism:** Physically doing several things at once (multiple CPUs).
- **Async:** A programming model for concurrency.

**TS:** Single-threaded concurrency via the Event Loop and I/O waiting.
**Rust:** M:N threading: M green tasks (Futures) on N OS threads.

> M (Tasks/Futures): many async tasks (green threads).
> N (OS Threads): a small number of OS threads (often ≈ CPU cores).

---

## 2. Core difference: Rust Future vs TS Promise

### 2.1 TS: eager execution

In TS, when you create a Promise, it **starts running immediately**.

```typescript
// TS
const p = new Promise((resolve) => {
  console.log("1. Start cooking noodles"); // Runs immediately
  resolve("noodles");
});
console.log("2. Waiting for result");
```

**Order:** 1 → 2. The Promise runs even if you never await it (you just can’t get the result).

### 2.2 Rust: lazy execution

In Rust, an `async` block returns a value that implements the `Future` trait. **Until you poll it (e.g. via an executor or `.await`), it does nothing.**

```rust
// Rust
async fn cook_noodle() {
    println!("1. Start cooking noodles");
}

fn main() {
    let f = cook_noodle(); // Nothing runs here; f is just a Future
    println!("2. Waiting result");

    // Must run it explicitly (e.g. executor::block_on or .await)
    // futures::executor::block_on(f);
}

```

Without an executor, you only see "2"; "1" never runs.

**Analogy:**

- **TS Promise:** Order food; the kitchen starts cooking as soon as you order.
- **Rust Future:** A recipe. You have the recipe, but until a chef (Executor) runs it, no food appears.

---

## 3. Under the hood: the `Future` trait

In Rust’s async world, everything centers on `std::future::Future`.

### 3.1 Definition

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T), // Done; return value
    Pending,  // Not done; keep waiting
}

```

Simple on the surface; this is where Rust’s zero-cost async abstraction lives.

### 3.2 How it runs: poll and waker

In TS, the runtime puts callbacks in the microtask queue.
In Rust there is **no built-in runtime**. A third-party crate (e.g. Tokio, async-std) provides an **Executor**.

For Tokio, see [Migrating to Rust Tokio](https://zhuanlan.zhihu.com/p/1923366370063660292) (Chinese).

**Flow:**

1. **Executor** calls the Future’s `poll`.
2. **Future** checks internal state:
   - If ready → return `Poll::Ready(data)`.
   - If not (e.g. socket still reading) → register a **Waker**, return `Poll::Pending`.
3. **Executor** sees Pending; it may park the thread or run other tasks.
4. **Reactor** (I/O layer) sees that the socket has data and calls the registered **Waker**.
5. **Waker** tells the Executor: "That task can run again."
6. **Executor** calls `poll` again; this time it may get `Poll::Ready`.

### 3.3 Why `Pin`? (Self-referential structs)

The signature uses `self: Pin<&mut Self>`, not plain `&mut Self`. Why?

**Reason: self-referential structs.**

Rust’s `async/await` is implemented by **compiler-generated state machines**. See the widely read article [Why async Rust](https://zhuanlan.zhihu.com/p/1935460387156923268) (Chinese).

```rust
async fn example() {
    let x = [0; 1024]; // Big array
    let y = &x;        // y borrows x (self-reference!)
    some_io().await;   // Yield point
    println!("{:?}", y);
}

```

After compilation, the Future struct roughly looks like:

```rust
struct ExampleFuture {
    state: i32,
    x: [u8; 1024],
    y: *const [u8; 1024], // Pointer into x inside the same struct
}

```

If this struct is **moved** (e.g. from stack to heap, or when a Vec resizes), `x`’s address changes but `y` would still point to the old address (dangling pointer).

**`Pin`:** "Pins" the Future in memory so it is **never moved**. Only then is the internal self-reference safe.

---

## 4. Comparison: Rust vs TS vs Go

| Aspect | TypeScript (Node.js) | Go (goroutines) | Rust (async/await) |
| ---------- | ---------------------------------- | ---------------------------------- | --------------------------------------- |
| **Model** | Event Loop (single thread + callbacks) | Green threads (M:N) | State machines (zero-cost abstraction) |
| **Stack** | Promises on heap | **Stackful** (e.g. 2KB stack per goroutine) | **Stackless** (compiled to fixed-size struct) |
| **Scheduling** | Cooperative (microtask queue) | Preemptive (runtime scheduler) | Cooperative (must await to yield) |
| **Overhead** | Medium (GC + V8) | Low (context switch) | **Very low** (no runtime beyond the state machine) |
| **Cross-thread** | No (need Worker Threads) | Yes (channels) | Yes (Send/Sync traits) |

### 4.1 Go: pros and cons

- **Pros:** "Synchronous-looking code, asynchronous execution." You write `conn.Read()`; it looks blocking but the runtime suspends the goroutine. Low mental load.
- **Cons:** Heavy runtime (GC + scheduler). Each goroutine has at least ~2KB stack (and can grow). That’s a burden for embedded or extreme performance.

### 4.2 Rust: pros and cons

- **Pros:** Zero-cost; no GC. A Future is just a struct. 100k Futures might be a few MB. You control layout.
- **Cons:** More concepts: `Pin`, `Send`, `Sync`, and ecosystem split (Tokio vs async-std). If you do CPU-heavy work inside an async block without yielding, you can block the executor (cooperative scheduling).

---

## 5. Trait pitfalls in practice: `Send` and `Sync`

In TS you don’t think about "can this cross threads?" In Rust async, you must.

### 5.1 Futures across threads

Many executors (e.g. Tokio) are multi-threaded (work stealing). A task might start on one thread and resume on another.

So **the Future and everything it captures must be `Send`.**

```rust
// ❌ Bad
async fn process() {
    let rc = Rc::new(5); // Rc is !Send (refcount not atomic)

    some_io().await; // Yield: state machine stores rc

    println!("{}", rc);
}

tokio::spawn(process()); // Error: Future not Send
```

**Fix:** Use `Arc` (atomic reference counting) instead of `Rc`.

### 5.2 Async trait methods

Before Rust 1.75, you couldn’t write `async fn` directly in a trait.

```rust
// ❌ Used to be unsupported
trait Database {
    async fn fetch(&self, id: i32) -> User;
}

```

**Reason:** `async fn` desugars to `impl Future`; trait methods with `impl Trait` return types made trait objects (dyn Trait) problematic (size unknown).

**Workarounds:**

1. **Macro:** Use `#[async_trait]`; it boxes the future (`BoxFuture`), so you get dynamic dispatch and heap allocation.
2. **Rust 1.75+:** Native `async fn` in traits is supported, but there are still limitations (e.g. RPITIT) and `Send` bounds can be tricky.

---

## 6. Summary: mental map for TS developers

1. **Promise is hot, Future is cold:** A Rust Future doesn’t run until you drive it (e.g. with an executor or `.await`).
2. **No built-in runtime:** The language only gives you the state-machine transformation; you need a crate like `tokio` to run futures.
3. **Future is a state-machine struct:** The compiler turns your `async/await` into an enum state machine.
4. **Pin is for memory safety:** The generated Future may contain self-references; pinning prevents moves that would invalidate them.
5. **Send/Sync are mandatory:** Because futures may move between threads, the compiler enforces that captured data is safe to send across threads.
