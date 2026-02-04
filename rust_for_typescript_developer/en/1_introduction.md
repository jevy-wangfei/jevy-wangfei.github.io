This cheat sheet is designed for TypeScript developers, aiming to help you map your existing knowledge to Rust syntax.

### 1. Variables & Mutability

In TS, `const` means the reference is immutable, but the object's interior is mutable. In Rust, everything is immutable by default (Deeply Immutable).

| Concept | TypeScript | Rust | Notes |
| -------- | ------------------- | --------------------------- | ------------------------------------------------- |
| Define variable | `let x = 5;` | `let x = 5;` | Rust defaults to **immutable** |
| Mutable variable | `let x = 5; x = 6;` | `let mut x = 5; x = 6;` | Must explicitly add `mut` |
| Constant | `const MAX = 100;` | `const MAX: u32 = 100;` | Rust constants require a type and are compile-time constants |
| Variable shadowing | (not recommended) | `let x = 5; let x = x + 1;` | Rust allows Shadowing (reusing variable names), often used for type conversion |

### 2. Basic Data Types

Rust is a statically strongly-typed language, but its type inference is very smart. Usually you don't need to write types everywhere like in Java; the compiler can infer types from context (e.g. the assigned value), which is very similar to TS.

Rust's basic types fall into **Scalar** and **Compound** categories.

#### 2.1 Scalar Types

Represent a single value.

- **Integers**:
- **TS:** Only `number` (IEEE 754 double-precision float) and `BigInt`.
- **Rust:** You must explicitly choose bit width.
- `i8`, `u8`, `i16`, `u16`, `i32` (default), `u32`, `i64`, `u64`, `i128`, `u128`.
- **Special note**: `isize` / `usize`. These are like C++'s `size_t`; their length depends on CPU architecture (64 bits on 64-bit machines). **In Rust, array indices and memory offsets must use `usize`.**

- **Floating-Point**:
- `f32`: Single precision.
- `f64`: Double precision (default). **This is equivalent to TS's `number`.**

- **Booleans**:
- `bool`: `true` / `false`. Occupies 1 byte.

- **Characters**:
- **TS:** `'a'` is just a string of length 1 (UTF-16).
- **Rust:** `char` is a **4-byte** Unicode Scalar Value. It can store Emoji `😻` directly, not just ASCII.
- _Note: Rust strings are not arrays of `char`; they are UTF-8 byte sequences._

#### 2.2 Compound Types

Combine multiple values into one type, allocated on the **Stack**.

- **Tuples**:
- **TS:** `[string, number]`.
- **Rust:** `(i32, f64, u8)`. Fixed length, types may differ.
- **Unit type `()`**: Empty tuple. Similar to TS's `void`, but in Rust it's a real "empty value" that occupies 0 memory.

- **Arrays**:
- **TS:** `const arr = [1, 2, 3]` is a dynamic heap array (List).
- **Rust:** `[T; N]` is **fixed length**.
- `let a = [1, 2, 3, 4, 5];` (type is `[i32; 5]`).
- **Key difference**: Rust arrays are allocated on the **stack** (if not too large). You cannot `push` or `pop`.
- _If you want a TS-style variable-length array, use the standard library's `Vec<T>` (Vector)._

#### 2.3 Strings: String vs &str (Tricky Part)

This is the concept TS developers find most confusing, because TS has only one `string` (V8 has Rope/ConsString optimizations internally, but they're transparent to users).

- **String (Owner)**:
- **Memory location**: Heap.
- **TS analogy**: Like `new String("...")` or a mutable `StringBuilder`.
- **Characteristics**: Owns the data, can be modified (`mut`), can grow (`push_str`). It has three fields: ptr (pointer), len (length), capacity (capacity).

- **&str (Borrower/Slice)**:
- **Memory location**: The fat pointer itself is on the stack, but the data it **points to** can be anywhere (static storage, heap, stack).
- **TS analogy**: A read-only, fixed-length "view window".
- **Characteristics**: It's a "fat pointer" containing: ptr (start of data), len (length). It doesn't own data; it just borrows a view. String literals `"hello"` have type `&str` (pointing to static memory).

**Code comparison:**

```rust
// String: I own this memory, I allocate on the heap
let mut s: String = String::from("hello");
s.push_str(" world"); // OK, grows automatically

// &str: I'm just borrowing a view, I don't allocate or free
let slice: &str = &s[0..5]; // Points to first 5 bytes of s
// slice.push_str("!"); // ❌ Compile error: &str is immutable

```

### 3. Functions & Implicit Return

Rust can use `return xxx;` like TS (note the semicolon), or use implicit return like Ruby/Lisp, which is common in functional programming.

```rust
// TypeScript
function add(a: number, b: number): number {
  return a + b;
}

// Rust
fn add(a: i32, b: i32) -> i32 {
  return a + b;
}

fn add(a: i32, b: i32) -> i32 {
  a + b  // Note: no semicolon! With a semicolon it becomes a Statement and returns Unit ()
}

```

### 4. Structs & Methods

Rust has no `class`. It separates **data definition** and **method implementation**.

**TypeScript:**

```typescript
class User {
  constructor(public username: string) {}

  describe() {
    console.log(`User: ${this.username}`);
  }
}
```

**Rust:**

```rust
struct User {
    username: String,
}

// Like Extension Methods, methods defined externally
impl User {
    // Associated function (like static method), often used as constructor
    fn new(name: &str) -> User {
        User { username: name.to_string() }
    }

    // Method (instance method), &self is like this
    fn describe(&self) {
        println!("User: {}", self.username);
    }
}

```

### 5. Interface vs Traits

TS's Interface is a structural type system (duck typing). Rust's Trait is nominal; implementation must be explicit.

**Rust:**

```rust
trait Printable {
    fn format(&self) -> String;
}

impl Printable for User {
    fn format(&self) -> String {
        format!("User: {}", self.username)
    }
}

```

### 6. Enums & Pattern Matching

This is one of Rust's killer features. TS enums are just number or string mappings. Rust enums are **Algebraic Data Types (ADT)** and can carry data.

**TypeScript:**

```typescript
type Action =
  | { type: "Quit" }
  | { type: "Move"; x: number; y: number }
  | { type: "Write"; content: string };
// Must manually check type field
```

**Rust:**

```rust
enum Action {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
}

fn handle(action: Action) {
    match action {
        Action::Quit => println!("Quit"),
        Action::Move { x, y } => println!("Move to {}, {}", x, y), // Destructuring
        Action::Write(s) => println!("Write: {}", s),
    }
}
// The compiler forces you to handle all cases; you can't miss any

```

### 7. No Null, No Exceptions

Rust has no `null`, no `undefined`, and no `try/catch`.

- **Option<T>**: Replaces `null/undefined`. Rust can use `return xxx;` like TS (note the semicolon), or use implicit return like Ruby/Lisp, common in functional programming.
- **Result<T, E>**: Replaces exceptions. Result is also a Rust Enum; Ok<T> and Err<E> are its two variants.

```rust
// TypeScript:
function find(): string | null {}
// Rust
fn find_user(id: i32) -> Option<String> {
    if id == 1 {
        Some("Alice".to_string())
    } else {
        None // Like null, but must be handled explicitly
    }
}

// Handle Option with match
match find_user(1) {
    Some(name) => println!("Found: {}", name),
    None => println!("No user found"),
}

// Or use combinators like TS Optional Chaining; note |name| println!("Found: {}", name) is a Rust closure
find_user(1).map(|name| println!("Found: {}", name));

```

### 8. Closures & Iterators

Syntax is very similar to TS arrow functions.

**TypeScript:**

```typescript
[1, 2, 3].map((x) => x + 1).filter((x) => x > 2);
```

**Rust:**

```rust
let nums = vec![1, 2, 3]; // Macro to create a Vector
let result: Vec<i32> = nums
    .iter()             // Must get an iterator first
    .map(|x| x + 1)     // Pipeline; |x| is closure param, x+1 is closure body
    .filter(|x| *x > 2) // filter needs dereference because iter yields references
    .collect();         // Lazy! Must call collect to actually run the loop

```

### Tips for Senior TS Developers

1. **On semicolons:** In Rust, an expression used as a return value **must not** have a semicolon; statements must have one. This is the most common beginner gotcha.
2. **On references:** In TS, objects are passed by reference by default. In Rust, you explicitly choose "borrow" (`&User`) or "move ownership" (`User`). If the function doesn't need to modify data, always use `&`.
3. **Generics:** Rust's generics `<T>` are almost the same as TS's `<T>`; they'll feel familiar.

Lifetimes are a concept that doesn't exist in TS at all.
