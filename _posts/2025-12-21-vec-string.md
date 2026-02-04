---
title: "5. Deep Dive into Vec and String"
date: 2025-12-21 12:00:00 +1100
categories: [TypeScript-to-Rust Developer]
tags: [Rust, Typescript]
---

Rust's strings are so different from TS that we devote a separate chapter to them.

## Strings: Buffer vs View

In TS/JS (V8), `string` is a black box. You write `const s = "hello" + "world"`; V8 might optimize it to a ConsString (tree) or allocate and copy—all transparent to you.

In Rust, for performance and precise control, strings are split into two distinct concepts: **owner (`String`)** and **borrower (`&str`)**. Before diving into String, we look at Vec.

### 1. Vec<T>

In Rust, **`Vec<T>`** (Vector) is a core data structure: a **variable-length dynamic array**.

Unlike a fixed array (`[T; n]`), a `Vec` can grow and shrink at runtime; its data lives on the **heap**.

#### 1.1 Core properties of Vec

- **Dynamic growth:** You can add elements with `push`; it handles allocation.
- **Contiguous memory:** Elements are laid out contiguously for fast random access.
- **Homogeneous:** A `Vec` holds only one type `T`.
- **Ownership:** When a `Vec` goes out of scope, its elements are dropped (memory freed).

#### 1.2 Memory layout (triple)

Under the hood, a `Vec` is three values on the **stack**:

1. **Pointer:** To the start of the heap allocation.
2. **Length:** Number of elements currently in the vector.
3. **Capacity:** Maximum elements without reallocating.

#### 1.3 Basic usage

```rust
fn main() {
    // 1. New empty Vec (type annotation needed)
    let mut v: Vec<i32> = Vec::new();

    // 2. Macro for create + init
    let mut v2 = vec![1, 2, 3];

    // 3. Add elements
    v2.push(4);
    v2.push(5);

    // 4. Read elements
    // A: Indexing (panics if out of bounds)
    let third = &v2[2];
    println!("Third element: {}", third);

    // B: .get (returns Option, safer)
    match v2.get(10) {
        Some(val) => println!("Value: {}", val),
        None => println!("Out of bounds!"),
    }

    // 5. Iterate
    for x in &v2 {
        println!("{}", x);
    }
}

```

#### 1.4 Vec vs Array

| Property | Array | Vec |
| ------------ | ---------------------- | -------------------------- |
| **Length** | Fixed (compile-time) | Dynamic (runtime) |
| **Memory** | Usually stack | Heap |
| **Use case** | Known fixed size | Unknown or changing size |

#### 1.5 Performance tip

When length exceeds capacity, `Vec` reallocates and copies. If you know the size in advance, use `Vec::with_capacity(n)` to avoid repeated allocations.

### 2. String: The owner

- **What it is:** A wrapper around `Vec<u8>`. A **heap-allocated**, growable, UTF-8 byte buffer.
- **Layout:**
  - **Stack:** A fat pointer: `ptr`, `len`, `capacity`.
  - **Heap:** The actual bytes (e.g. "Hello").
- **TS analogy:** Like Node's `Buffer` or a mutable `StringBuilder`.
- **Role:** It owns the data. When it goes out of scope it frees the heap (Drop).

### 3. &str: The view

- **What it is:** A **string slice**. It doesn't own data; it's a "window" onto some memory.
- **Layout:**
  - **Stack:** Fat pointer: `ptr`, `len`. **No `capacity`.**
  - **Points to:** Data that may be on heap, stack, or static (see below).
- **TS analogy:** Like `Uint8Array.subarray()` — a lightweight reference; **no deep copy**.

### 4. Where does `let a = "abc"` live?

This trips up beginners.

```rust
let a = "abc";
// a has type &'static str

```

Two things happen in memory:

1. **Data in static:** The bytes `'a','b','c'` are baked into the binary at compile time (**.rodata**). They exist from program start until exit.
2. **Variable on stack:** The variable `a` is an `&str` fat pointer on the stack; its `ptr` points into that static region.

**Why type `&str`?** Rust can't store "unsized" data directly on the stack. The data type is `str`; we can't hold it, only a reference `&str`. Because the data lives in static and never dies, its lifetime is `'static`.

### 5. Best-practice rule of thumb

Given the layout:

> **Rule: Prefer `&str` for function parameters; prefer `String` for struct fields.**

#### Scenario A: Function parameters use `&str`

**Reason: Deref coercion**

The compiler allows **`&String` to coerce to `&str`**.

```rust
fn print_msg(msg: &str) { // ✅ Accepts any view
    println!("{}", msg);
}

let s1 = String::from("Heap Data"); // Heap String
let s2 = "Static Data";             // Static &str

print_msg(&s1); // ✅ Coerces &String → &str (O(1))
print_msg(s2);  // ✅ Already &str

```

If the parameter were `String`, passing `s2` would require `.to_string()` and an expensive heap allocation (malloc + memcpy).

#### Scenario B: Struct fields use `String`

**Reason: Ownership and lifetime**

```rust
struct User {
    username: String, // User owns the name; it lives as long as User
}

// Unless you're advanced, avoid this
struct UserRef<'a> {
    username: &'a str, // User borrows; you must say how long the name lives
}

```

Storing `&str` in a struct means the struct **cannot outlive** the borrowed data. Once the owner (e.g. a `String`) is dropped, the struct must be invalid. That leads to lifetime annotations and extra complexity.

### Summary

- **String** = **landlord** (owns the house on the heap).
- **&str** = **tenant/viewer** (has key `ptr` and lease length `len`; the "house" can be the landlord's (String on heap) or "public housing" (literal in static)).