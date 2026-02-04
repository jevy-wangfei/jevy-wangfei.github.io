Now that you have Struct (space) and Trait (behavior), we focus on the third piece of Rust’s type system—and the hardest: **Time**.

This is the biggest conceptual wall for TS developers. In TS, "time" is effectively unbounded (GC keeps things alive as long as something references them). In Rust, time is **finite and strictly controlled**.

---

# Core reframe: the "trinity" of Rust’s type system (Part 3: Lifetimes)

Rust’s worldview:

1. **Struct + Generics:** The **shape** of data (space).
2. **Traits:** How data **interacts** (behavior).
3. **Lifetimes:** How long data **lives** (time).

**For TS full-stack developers:**

> **Generics `<T>` tell the compiler: "input and output types must match."**
> **Lifetimes `<'a>` tell the compiler: "input and output *scopes* must match."**

Lifetime annotations **never** change runtime behavior or how long a variable actually lives. They only help the **borrow checker** at compile time to prove that your references are safe.

---

### 1 Why do we need lifetimes? (Dangling pointers)

In TS/JS it’s hard to create a dangling pointer; the GC tracks references.

```typescript
// TypeScript
let r;
{
  let x = { val: 5 };
  r = x; // r refers to x
}
// Even though x’s block ended, r still refers to it, so GC won’t collect x yet.
console.log(r.val); // ✅ Safe
```

In Rust, memory is deterministic (RAII). When a variable goes out of scope, its memory is freed.

```rust
// Rust
fn main() {
    let r;                // ---------+-- 'a
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
    // 💀 x is destroyed here  //          |
                          //          |
    println!("{}", r);    // ---------+
}

```

**What the compiler sees:**

1. `r` has lifetime `'a` (whole `main`).
2. `x` has lifetime `'b` (inner block).
3. `r` borrows `x`.
4. **Rule:** The borrowed data (`'b`) must outlive the borrower (`'a`).
5. **Reality:** `'b` is shorter than `'a`.
6. **Conclusion:** Reject the program; prevent use-after-free.

---

### 2 Lifetimes in functions: connecting inputs and outputs

When a function **returns a reference**, the compiler must know where that reference comes from.

#### Example: longest string

```rust
// ❌ Compile error
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

```

**Compiler’s problem:**

- The function returns `&str`.
- There are two input references `x` and `y`.
- The return value might be `x` or `y` (depends on runtime).
- **Question:** If the caller passes `x` that lives 10 seconds and `y` that lives 1 second, how long can the returned reference live? The compiler can’t know which branch is taken at compile time.

#### Solution: explicit annotation

We tell the compiler: **"The returned reference lives at most as long as the shorter of `x` and `y`."**

```rust
// ✅ Generic lifetime 'a
// Read: x and y both live at least 'a; the return value also lives at least 'a.
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

```

**What happens at the call site:**

```rust
fn main() {
    let s1 = String::from("long string"); // s1 lives long
    let result;
    {
        let s2 = String::from("xyz");     // s2 lives short

        // longest(&s1, &s2)
        // Compiler infers 'a = min(lifetime(s1), lifetime(s2)) = lifetime(s2)
        result = longest(s1.as_str(), s2.as_str());

        println!("{}", result); // ✅ OK: result used before s2 dies
    }
    // s2 is dropped
    // println!("{}", result); // ❌ Error: result’s lifetime (tied to 'a) has ended
}

```

---

### 3 Lifetimes in structs (reference-holding pattern)

When a struct **holds a reference**, the struct cannot outlive that reference.

**Rule:** If a struct holds a reference, the struct’s lifetime must be shorter than (or equal to) the reference’s.

```rust
struct UserView<'a> {
    user_ref: &'a User,
}

struct User { username: String }

fn main() {
    let user = User { username: "TS Dev".into() };

    // ✅ OK: view’s lifetime ≤ user’s
    let view = UserView { user_ref: &user };

    // ❌ Invalid (conceptually):
    // let view;
    // {
    //    let temp_user = User { ... };
    //    view = UserView { user_ref: &temp_user };
    // } // temp_user dies; view would be a dangling reference
}

```

**TS analogy:** Like holding a ref to a DOM node in React. `UserView` is like a React ref. If the real node (`User`) is unmounted, using the ref (`UserView`) would be invalid. Rust forbids this at compile time.

---

### 4 Lifetime elision

Why don’t `fn main()` or simple getters need `'a`?
The compiler has three built-in **elision rules**; it fills in lifetimes when they’re unambiguous. If it can’t, it errors.

**Rule 1:** Each reference parameter gets its own lifetime.
`fn foo(x: &i32, y: &i32)` → `fn foo<'a, 'b>(x: &'a i32, y: &'b i32)`

**Rule 2:** If there’s only one input reference, all output references get that lifetime.
`fn return_str(s: &str) -> &str` → `fn return_str<'a>(s: &'a str) -> &'a str`  
(This covers many simple cases.)

**Rule 3 (methods):** If there’s `&self` or `&mut self`, output references get the lifetime of `self`.
Methods usually return references into the struct; so this is the right default.

```rust
impl<'a> UserView<'a> {
    fn get_name(&self) -> &str {
        &self.user_ref.username
    }
}

```

---

### 5 Static lifetime (`'static`)

`'static` means: **"This reference can live for the entire program."**

Two common cases:

1. **String literals:** `let s: &'static str = "Hello";` — stored in the binary’s read-only data (e.g. `.rodata`), never freed.
2. **Global `static`:** Variables declared with `static`.

**Common mistake for TS devs:** When the compiler complains about lifetimes, beginners sometimes add `: &'static str` to silence it. **Don’t.** Use `'static` only when you really have static data (e.g. literals). Forcing a dynamically allocated `String` to `&'static` (e.g. via `Box::leak`) can cause leaks or simply not compile.

---

### 6 Putting it together: Struct + Generic + Trait + Lifetime

**Goal:** A printer that holds a text reference and a delimiter; if the text contains a keyword, return a formatted string; otherwise format the delimiter. Delimiter must implement `Display`.

```rust
use std::fmt::Display;

struct ContentPrinter<'a, T> {
    content: &'a str,
    delimiter: T,
}

impl<'a, T> ContentPrinter<'a, T>
where
    T: Display
{
    fn print_if_contains(&self, keyword: &str) -> String {
        if self.content.contains(keyword) {
            format!("Found: {}", self.content)
        } else {
            format!("Separator: {}", self.delimiter)
        }
    }

    fn get_part(&self) -> &'a str {
        self.content
    }
}

fn main() {
    let text = String::from("Rust is amazing");
    let delimiter = "---";

    let printer = ContentPrinter {
        content: &text,
        delimiter
    };

    println!("{}", printer.print_if_contains("Rust"));
}

```

### Summary: thinking in terms of time

1. **Reference = borrow:** Whenever you see `&`, ask "when does this die?"
2. **Function signature = contract:** `fn foo<'a>(x: &'a str) -> &'a str` is a guarantee: the return value never outlives `x`.
3. **Struct = container:** If the struct holds borrowed data (references), the struct’s lifetime is bounded by those references.

**Final note for TS developers:** In TS you focus on **values**. In Rust you must also focus on **ownership** and **lifetimes**. When you get a lifetime error, treat it as the compiler protecting you from invalid memory access, not as an obstacle to work around.
