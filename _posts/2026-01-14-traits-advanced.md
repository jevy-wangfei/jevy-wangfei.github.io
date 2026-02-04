---
title: "11. Advanced Trait Features: Returning Traits and Trait Bounds"
date: 2026-01-14 12:00:00 +1100
categories: [TypeScript-to-Rust Developer]
tags: [Rust, Typescript]
---

## 8. Returning traits (`impl Trait`)

### 8.0 Why can’t we return `-> Trait` directly?

Suppose you have two structs:

- `struct Button` (100 bytes)
- `struct Label` (20 bytes)

Both implement the `Draw` trait.

```rust
// ❌ Compile error!
// Compiler: when calling this function, should I reserve 100 bytes or 20 bytes on the stack?
fn create_component(condition: bool) -> Draw {
    if condition { Button::new() } else { Label::new() }
}

```

`Draw` is a concept (trait), not a concrete type; it has no fixed size (`!Sized`). So you can’t return it directly.

### 8.1 Static return: `impl Trait` (opaque type)

`impl Trait` is an **existential type**.
Meaning: **"I return one concrete type; I just don’t want to write its name in the signature—infer it from the body."**

#### Use case A: Unreadable iterator/closure types

Chained iterators in Rust have huge, unreadable types.

```rust
// Without impl Trait you’d have to write something like:
fn make_iter() -> std::iter::Map<std::iter::Filter<std::vec::IntoIter<i32>, fn(&i32)->bool>, fn(i32)->i32> {
    vec![1, 2, 3].into_iter()
        .filter(|x| x % 2 == 0)
        .map(|x| x * 2)
}

// ✅ With impl Trait: clear and zero-cost
fn make_iter() -> impl Iterator<Item = i32> {
    vec![1, 2, 3].into_iter()
        .filter(|x| x % 2 == 0)
        .map(|x| x * 2)
}

```

#### Use case B: Closures — you must use `impl Trait`

In Rust, every closure has a unique, anonymous type. You can’t name it.

```rust
// ❌ Error: you can’t write the closure’s type
fn returns_closure() -> ??? {
    |x| x + 1
}

// ✅ Correct: tell the compiler "I return something callable"
fn returns_closure() -> impl Fn(i32) -> i32 {
    |x| x + 1
}

```

### 8.2 Limitation: single concrete type

Because `impl Trait` is **static dispatch** (the compiler replaces it with one concrete type), every branch must return the **same** concrete type.

```rust
// ❌ Compile error!
// Branch A returns Button (type A)
// Branch B returns Label  (type B)
// impl Trait stands for one concrete type; compiler can’t make it both.
fn create_component(b: bool) -> impl Draw {
    if b {
        Button::new()
    } else {
        Label::new()
    }
}

```

### 8.3 Escape hatch: dynamic return (`Box<dyn Trait>`)

When you really need to return different types depending on a condition (like in TS), you need **heap allocation** and a fixed-size pointer.

```rust
// ✅ Compiles
// Box<dyn Draw> is a fat pointer (fixed size, e.g. 16 bytes) on the stack.
// The actual Button or Label lives on the heap.
fn create_component(b: bool) -> Box<dyn Draw> {
    if b {
        Box::new(Button::new())
    } else {
        Box::new(Label::new())
    }
}

```

### Summary: mental model for TS developers

1. **`impl Trait` (static):**
   - **What it is:** A "disguise" for one concrete type. The compiler knows who’s behind it.
   - **TS analogy:** Like `type Hidden = ConcreteType` but not exposed.
   - **Performance:** Zero-cost; no heap allocation.
   - **Limit:** All return paths must be the **same** concrete type.

2. **`Box<dyn Trait>` (dynamic):**
   - **What it is:** A **pointer**. The compiler doesn’t know the concrete type, only the trait.
   - **TS analogy:** Standard "return an interface" style.
   - **Performance:** Heap allocation + vtable call.
   - **Capability:** Can return different concrete types (polymorphism).

---

## 9. Fully qualified syntax

This touches a low-level idea in Rust: **universal function call syntax (UFCS)**.

To understand it as a TS developer, we need to drop the idea that "methods belong to objects" and see that methods are just functions.

# Method calls vs fully qualified syntax

## 9.1. Core idea: methods are just functions

In TypeScript/JS, methods live on `prototype`. `person.fly()` is a runtime lookup on `person`’s prototype chain.

In Rust there are no "methods" in that sense.
What you define in an `impl` block compiles to **ordinary functions whose first parameter is `self`**.

### Desugaring

When you write `person.fly()`, the compiler does two things:

1. **Auto reference/dereference:** Based on the signature (`&self`, `&mut self`, `self`), it turns `person` into `&person` (or similar).
2. **Rewrite to function call:** Dot notation is rewritten to a function call.

```rust
// Sugar
person.fly();

// What the compiler really sees (desugared)
Pilot::fly(&person);

```

So Rust can resolve name clashes via traits because they’re just **different functions in different namespaces**.

## 9.2. Three levels of conflict

Suppose `Human` also has its own `fly`. Then we have three `fly` definitions.

```rust
struct Human;

trait Wizard { fn fly(&self); }
trait Pilot { fn fly(&self); }

impl Human {
    fn fly(&self) { println!("Waving arms furiously!"); }
}

impl Wizard for Human {
    fn fly(&self) { println!("Up!"); }
}

impl Pilot for Human {
    fn fly(&self) { println!("Take off!"); }
}

```

### 9.2.1 Level 1: Default priority (inherent impl)

If you call:

```rust
let p = Human;
p.fly(); // Output: "Waving arms furiously!"

```

**Rule:** Rust prefers the type’s **own** methods (`impl Human`) over trait methods.

### 9.2.2 Level 2: Disambiguation

To call a **trait** method, you must name the **trait**:

```rust
Pilot::fly(&p);  // Output: "Take off!"
Wizard::fly(&p); // Output: "Up!"

```

You must pass `&p` explicitly because we’re not using the dot-call sugar (no auto ref).

### 9.2.3 Level 3: Fully qualified syntax

The **fully qualified** form is:

```rust
<Human as Pilot>::fly(&p);

```

**When is this needed?** Usually not. It’s **required** in one case: **associated functions** (no `self`), i.e. "static methods."

If both `Wizard` and `Pilot` have a constructor `new()`:

```rust
trait Wizard { fn new() -> Self; } // no &self
trait Pilot { fn new() -> Self; }

impl Wizard for Human { ... }
impl Pilot for Human { ... }

fn main() {
    // let h = Human::new(); // ❌ Error: Human has no new, and it’s unclear which trait’s new
    // let h = Wizard::new(); // ❌ Error: Wizard::new() return type could be any type implementing Wizard
    // ✅ Must use fully qualified: "treat Human as Wizard and call its new"
    let h = <Human as Wizard>::new();
}
```

## 9.3. Comparison with TypeScript

In TypeScript, if a class implements two interfaces with the same method name, you can only have one implementation:

```typescript
interface Wizard {
  fly(): void;
}
interface Pilot {
  fly(): void;
}

class Human implements Wizard, Pilot {
  fly() {
    // You have to merge behavior here
    console.log("I am flying like... generic human?");
  }
}
```

In Rust you can keep **separate semantics**:

- As a `Wizard`, it flies by magic.
- As a `Pilot`, it flies a plane.
- Two different function addresses; no mixing.

## 9.4. Summary

1. **Methods are functions:** `obj.method()` is sugar for `Trait::method(&obj)` (or `Type::method(&obj)` for inherent impl).
2. **Namespaces:** `impl Human`, `impl Wizard for Human`, `impl Pilot for Human` are three separate namespaces.
3. **Call levels:**
   - `p.fly()` — sugar; prefers inherent impl.
   - `Trait::fly(&p)` — explicit trait; for methods with `self`.
   - `<Type as Trait>::method()` — fully qualified; needed for associated functions (no `self`).

Rust’s design: **when there’s ambiguity, the compiler refuses to guess and forces you to specify the path.**