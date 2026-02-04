This is a Rust **Trait** guide tailored for experienced TypeScript developers.

You might think a Trait is just an `interface`. If you use it only that way, you’re using a small part of Rust. In Rust, traits are the **soul of the type system**: code reuse, static polymorphism, dynamic dispatch, and even operator overloading.

We’ll look at design rationale, implementation, and advanced patterns.

---

# The contract of behavior: Rust traits

## 1. Core shift: data-oriented design (DOD)

In TypeScript, OOP is default. We build "fat objects":

```typescript
// TS: data and behavior coupled
class User {
  constructor(public name: string) {}
  serialize() {
    return JSON.stringify(this);
  }
  display() {
    console.log(this.name);
  }
}
```

In Rust we separate **data** and **behavior**:

- **Struct:** Container for data (layout).
- **Trait:** Gives data a capability.

You can "attach" new behavior to a type via a trait without changing the struct definition. Similar to TS Mixins or extension methods, but safer and more powerful.

---

## 2. Basics: more than an interface

### 2.1 Definition and implementation

This part is closest to TS `interface`.

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}

pub struct NewsArticle {
    pub headline: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by ...", self.headline)
    }
}

```

### 2.2 Default implementations

TS interfaces only have signatures. Rust traits can have default method bodies.

```rust
pub trait Summary {
    fn summarize(&self) -> String;

    fn summarize_author(&self) -> String {
        format!("(Read more from {}...)", self.summarize())
    }
}

// Implementors can omit summarize_author and get the default
impl Summary for NewsArticle { ... }

```

**TS comparison:** Similar to an abstract class, but a trait **has no state** (no fields; only methods, constants, associated types). It’s purely a set of behaviors.

---

## 3. Two kinds of polymorphism: static vs dynamic

This is the hardest and most powerful part. TS polymorphism is entirely runtime (prototype lookup). Rust lets you choose: **maximum performance (time for space)** or **maximum flexibility (space for time)**.

### 3.1 Static dispatch — generic bounds

Rust’s default and recommended approach.

```rust
fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
// Or shorthand; still static dispatch:
fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}

let article = NewsArticle { ... };
let tweet = Tweet { ... };

notify(&article); // Call site A
notify(&tweet);   // Call site B
```

**Under the hood (monomorphization):** The compiler sees calls with `NewsArticle` and `Tweet` and generates two functions: `notify_for_NewsArticle` and `notify_for_Tweet`, and replaces calls with the concrete function. No generic `notify` in the final binary.

- **Pros:** Zero-cost abstraction; no vtable; can inline.
- **Cons:** Larger binary.

### 3.2 Dynamic dispatch — trait objects

When you need a **single** collection of different types that all implement `Summary`, generics don’t work (array elements must have the same size). You need **trait objects**: `&dyn Trait` or `Box<dyn Trait>`.

```rust
fn notify_dynamic(item: &dyn Summary) {
    println!("{}", item.summarize());
}

let list: Vec<Box<dyn Summary>> = vec![
    Box::new(NewsArticle { ... }),
    Box::new(Tweet { ... }),
];

```

**Under the hood (vtable):** `Box<dyn Summary>` on the stack is two pointers (e.g. 16 bytes): one to the heap data (e.g. `NewsArticle`), one to the **vtable** with the actual method addresses.

**TS comparison:** Trait objects behave like TS interfaces. V8 uses Hidden Classes and inline caches; Rust makes this cost explicit and controllable.

---

## 4. Associated types vs generic traits

When to use `trait Iterator<T>` vs `trait Iterator { type Item; }`?

### 4.1 Generic trait (`trait Service<T>`)

Generics mean: **one type can have multiple implementations of the same trait for different T.**

**Example:** Conversions with `From<T>`.

> `From` is in the standard library prelude; it’s the standard way to convert values. Unlike TS’s mix of `String(123)`, `Number(str)`, or custom `User.from(data)`, Rust standardizes conversion via the `From` trait.

```rust
struct MyNumber(i32);

pub trait From<T>: Sized {
    fn from(value: T) -> Self;
}

impl From<i32> for MyNumber {
    fn from(item: i32) -> Self {
        MyNumber(item)
    }
}

impl From<String> for MyNumber {
    fn from(item: String) -> Self {
        let n = item.parse().unwrap();
        MyNumber(n)
    }
}

fn main() {
    let num1 = MyNumber::from(100);
    let num2 = MyNumber::from(String::from("200"));
    println!("{:?}, {:?}", num1, num2);
}
```

So `MyNumber` implements both `From<i32>` and `From<String>`.

### 4.2 Associated type (`type Item`)

An associated type means: **for a given type, there is exactly one implementation of this trait** (no generic parameter).

**Example:** `Iterator`.

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

impl Iterator for Counter {
    type Item = u32; // Fixed once
    fn next(&mut self) -> Option<Self::Item> { ... }
}

```

### 4.3 When to use which (input vs output)

- **Generic trait = multiple inputs:** e.g. `From<T>`. Input type `T` can vary; output is `Self`. You need `impl From<A>` and `impl From<B>` for the same type → use generics.
- **Associated type = single output:** e.g. `Iterator`. For one iterator type, the type of items must be fixed. If `Iterator` were generic, `Counter` could implement both `Iterator<u32>` and `Iterator<String>`, and `counter.next()` would be ambiguous. With `type Item`, the compiler knows exactly what `next()` returns → use associated type.

---

## 5. Orphan rule — avoid prototype pollution

In TS you can mutate `String.prototype` (polyfills). Convenient, but two libraries can conflict.

Rust enforces **coherence** with the orphan rule:

**Rule:** In `impl Trait for Type`, either the **Trait** or the **Type** must be defined in the current crate.

- ✅ `impl MyTrait for i32` (trait is ours)
- ✅ `impl Display for MyStruct` (type is ours)
- ❌ `impl Display for String` (both from std — **forbidden**)

**Workaround: newtype pattern.** Wrap the type in a local struct:

```rust
struct Wrapper(String);

impl std::fmt::Display for Wrapper {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "[{}]", self.0)
    }
}

```

---

## 6. Supertraits — "interface inheritance"

Rust has no class inheritance, but traits can depend on other traits.

```rust
use std::fmt::Display;

trait OutlinePrint: Display {
    fn outline_print(&self) {
        let output = self.to_string();
        let len = output.len();
        println!("{}", "*".repeat(len + 4));
        println!("* {} *", output);
        println!("{}", "*".repeat(len + 4));
    }
}

```

**TS comparison:** Like `interface OutlinePrint extends Display`. In Rust you still `impl Display` and `impl OutlinePrint` separately; the compiler doesn’t inherit implementations.

---

## 7. Derivable traits

In TS, deep clone or serialize often needs reflection or manual code. Rust has `#[derive]` for common traits.

```rust
#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

let p1 = Point { x: 1, y: 2 };
let p2 = p1; // Copy; p1 still valid
println!("{:?}", p1);
```

**Common derivable traits:**

- `Debug`: for `{:?}`.
- `Clone`: `.clone()` (deep copy).
- `Copy`: assignment is copy instead of move (only for stack-only data).
- `PartialEq` / `Eq`: `==`.
- `Serialize` / `Deserialize`: (from `serde`) JSON (de)serialization.

---

## Summary: trait cheat sheet for TS developers

| Aspect | TypeScript | Rust | Difference |
| ------------ | ---------------------- | ------------------------------------------ | -------------------------------- |
| **Definition** | `interface` | `trait` | Contract on behavior, not just shape |
| **Implementation** | `class A implements B` | `impl B for A` | Non-invasive; data and behavior separate |
| **Polymorphism** | Only dynamic (runtime) | Static (`<T: Trait>`) or dynamic (`&dyn Trait`) | Default is static for performance |
| **Bounds** | `extends` | Trait bounds | Multiple bounds, associated types |
| **Extension** | Prototype patching | Blanket impl / extension trait | Orphan rule limits who can impl what |
| **Reuse** | Mixins / base class | Default impl | No state inheritance, only behavior |
| **Serialization** | `JSON.stringify` | `#[derive(Serialize)]` | Generated at compile time |

Traits are Rust’s main abstraction tool: they avoid OOP diamond inheritance, give C++-level performance via monomorphization, and keep Haskell-level type safety. Mastering traits is mastering Rust’s "way."
