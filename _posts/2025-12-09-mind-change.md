---
title: "2. Mindset Shift from TypeScript to Rust"
date: 2025-12-09 12:00:00 +1100
categories: [TypeScript-to-Rust Developer]
tags: [Rust, Typescript]
---

As experienced full-stack JavaScript/TypeScript developers, we're used to the V8 engine managing memory behind the scenes and to TypeScript's flexible structural type system.

To understand Rust from a top-level design perspective, you need to adjust several core mental models. You can think of Rust as **TypeScript without garbage collection (GC), plus mandatory memory rules**.

### 1. Core Paradigm Shift: From "GC as Safety Net" to "Ownership"

In TypeScript/Node.js, variables live on the stack (primitives) or heap (objects), and you rarely care when they're destroyed because GC handles it.

- **TypeScript:** "I create an object, pass it around; GC handles reference counting or mark-and-sweep."
- **Rust:** "A resource (memory, file handle) must have exactly one clear Owner."

**Mental shift for TS developers:**
Imagine that when writing TS, the compiler enforced: **an object can only be assigned to one variable at a time.**

```typescript
// Simulating Rust behavior in TS
let a = { name: "Rust" };
let b = a; // In Rust this is a "Move". 'a' is immediately invalid and can't be used again.
console.log(a); // Compile error! Ownership was transferred to 'b'.
```

Rust's "Borrow Checker" is like an extremely strict ESLint rule that, at **compile time**, ensures no dangling pointers and no data races, so you don't need a runtime GC.

### 2. Type System

This is where TS developers get tripped up most.

- **TypeScript:** If it walks like a duck and quacks like a duck, it's a duck. If the fields match, `Interface A` can be assigned to `Interface B`.
- **Rust:** Even if two structs have identical fields, they are different types.

**Trait vs Interface:**
Rust's `Trait` is similar to TS's `Interface` but more like a set of **extension methods**. In TS you implement an interface when you define the class. In Rust you can implement a Trait for an existing type (even a primitive like `i32`) "after the fact."

- **TS:** `class User implements JsonSerializable { ... }`
- **Rust:** Define `struct User` first, then elsewhere `impl JsonSerializable for User`. This separates data (Struct) and behavior (Trait) completely—more decoupled than OOP.

### 3. Error Handling: `try/catch` vs `Result<T, E>`

TypeScript inherits JS's exception model; any function can `throw` at any time, and you must always be on guard.

Rust follows a functional approach: **no exceptions, only return values**.

- **TS:**

```typescript
try {
  const data = readFile("test.txt");
} catch (e) {
  // Don't even know what type e is
}
```

- **Rust:**
  Any fallible operation returns a `Result<Success, Error>` enum. You **must** handle the `Error` branch (or propagate it with the `?` operator), or the code won't compile. This removes runtime surprises like "Undefined is not a function".

### 4. Async Model: Eager vs Lazy

- **TypeScript (Promise):** As soon as you create `new Promise(...)`, it starts executing.
- **Rust (Future):** Calling an `async` function returns a `Future` (like a JS Promise), but it **does nothing** until you explicitly `await` it or hand it to an executor (e.g. Tokio). This enables very efficient zero-cost abstraction but has slightly more mental overhead than JS's Event Loop.

### 5. Toolchain: Cargo vs NPM

As a senior TS developer, you'll love Cargo.

- **NPM:** `package.json` + `node_modules` + `webpack/vite` + `eslint` + `prettier` + `jest`...
- **Cargo:** It's Rust's package manager, build tool, test runner, and doc generator. It's like a better-designed NPM: works out of the box, with more predictable and fast dependency management.

### Summary: Learning Path

With full-stack TS experience, don't treat Rust as "C++ replacement"; treat it as **"strongly-typed systems-level Node.js"**.

**Use [The Rust Programming Language](https://doc.rust-lang.org/book/title-page.html) and [Rust by Example](https://doc.rust-lang.org/rust-by-example/):** Focus on **Lifetimes** and **Smart Pointers (`Box`, `Rc`, `Arc`)**, which have no equivalent in TS.

You don't need to relearn programming logic; you only need to learn how to "please" the very strict compiler. Once it compiles, the code usually runs correctly.

... [Rust for TypeScript Developers](https://www.youtube.com/watch?v=hcK3N_Y3hzM) ...

This video by well-known developer ThePrimeagen is aimed at TypeScript developers and compares Rust and TS mindsets; it fits a TS/JS background well.