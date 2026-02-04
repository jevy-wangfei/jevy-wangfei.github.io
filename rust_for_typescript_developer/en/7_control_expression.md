For JS/TS developers, these are usually separate: `if` is a "statement" for control flow, `1+1` is an "expression" for computation. Rust inherits from functional languages (Haskell, Scala, etc.): **control flow is data flow**. Aside from `let` and `fn` declarations, almost every block is an expression and has a value.

# Rust core idea

**Expression-oriented control flow.** In Rust, control flow isn’t only the "steering wheel" of execution; it’s also the "pipeline" that produces data.

So you can assign almost any block (`if`, `match`, `{}`) directly to a variable or use it as a function return value. That removes the "declare a variable, then assign in different branches" pattern and encourages immutability.

## 1. Statements vs expressions

To see why we put them together, we need the distinction.

### 1.1 Traditional view (JS/TS)

In TypeScript, control flow is mostly **statements**. Statements do things but don’t produce a value.

```typescript
// TS style
let result; // 1. Must declare a mutable variable (breeding ground for side effects)
if (condition) {
  result = 10; // 2. Assign in branches
} else {
  result = 20;
}
// result may implicitly be undefined, or need complex inference
```

### 1.2 Rust view (expression-first)

In Rust, control constructs (`if`, `match`, `loop`) are **expressions**. They "evaluate" and "return" a value.

```rust
// Rust style
// 1. Compute result directly from if, bind to immutable result
let result = if condition {
    10
} else {
    20
};
// result is fixed from the start and immutable.

```

**Why "expression" and "control flow" together:** In Rust, **control flow constructs are expressions that produce values.** They’re the same thing.

## 2. How control flow becomes expressions

### 2.1 `if` is a full-blown ternary

Rust has no `condition ? true : false`. `if` is enough.

- **TS:** `if` is a statement; no return value.
- **Rust:** `if` is an expression; value is the last expression in the branch.

```rust
let status = if score > 60 { "Pass" } else { "Fail" };

```

**Constraint:** Because it’s an expression, the `if` and `else` branches must have the **same type**. You can’t return `i32` in one and `String` in the other (unless wrapped in an enum).

### 2.2 `match` is a powerful pattern-based expression

This is the heart of Rust’s control flow. It’s not only branching; it’s a **data extractor** and **transformer**.

```rust
let input = Option::Some(100);

// match is an expression
let number = match input {
    Some(n) => n * 2, // Bind n, compute n*2, that’s the value
    None => 0,
};
// number is inferred as i32

```

### 2.3 `loop` can return a value

TS `while/for` just loop. Rust’s `loop` (infinite loop) can return a value like a function. Useful for retry logic.

```rust
let mut counter = 0;

// This loop eventually produces an i32
let result = loop {
    counter += 1;
    if counter == 10 {
        break counter * 2; // ✅ break carries a value out, like return
    }
};

assert_eq!(result, 20);

```

### 2.4 Blocks `{}` are expressions

Often overlooked: a block `{ ... }` is an expression; its value is the **last line** (no semicolon).

```rust
let y = {
    let x = 3;
    x + 1 // No semicolon -> value is 4
};

println!("y is {}", y); // y is 4

```

**Use case:** Isolate a small scope inside a function, compute a value, without polluting the outer scope.

## 3. The role of the semicolon (`;`)

In Rust, the semicolon isn’t just "end of line"; it’s an **operator**.

- **No semicolon** = **expression** — value is "returned" from the block.
- **With semicolon** = **statement** — value is discarded; the block yields unit `()`.

## 4. Why design it this way? (For senior devs)

You might ask: why change the familiar `return` style?

1. **Functional heritage:** Rust is influenced by OCaml/Haskell. In those languages, everything is a computation and has a value.
2. **Less mutable state (immutability):**
   - **TS:** `let x; if (a) x=1; else x=2;` — `x` must be mutable and has an "uninitialized" period.
   - **Rust:** `let x = if a {1} else {2};` — `x` can be immutable and is **initialized in one step**. That removes many state-sync bugs.
3. **Type system:** The compiler can check that all branches produce compatible types.

## 5. Summary

Merge your idea of "control flow" and "expression":

- In TS you often: **do A, then do B, then return result.**
- In Rust you: **build one big expression from `if`, `match`, and blocks, and that expression evaluates to the result.**
