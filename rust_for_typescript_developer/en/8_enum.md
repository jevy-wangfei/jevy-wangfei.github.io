This is a **Rust enums and pattern matching** guide for experienced TypeScript developers.

As a TS developer you may have a love-hate relationship with TS `enum`: it’s a runtime object, sometimes compiled to an IIFE or numbers. In practice, the TypeScript community prefers **discriminated unions**: `type State = { kind: 'A' } | { kind: 'B', payload: string }`.

Rust enums are the **full evolution** of TS discriminated unions, and together with pattern matching they form the backbone of Rust’s type system.

---

# The crown of the type system: Rust enums and pattern matching

## 1. Concept alignment: this isn’t the enum you know

In C++ or Java, enums are named integers. In TypeScript they’re string or number mappings.
In Rust, enums are **Algebraic Data Types (ADTs)**.

### 1.1 Carrying data

In TS you’d write:

**TypeScript (discriminated union):**

```typescript
type Message =
  | { kind: "Quit" }
  | { kind: "Move"; x: number; y: number }
  | { kind: "Write"; content: string }
  | { kind: "ChangeColor"; r: number; g: number; b: number };

function handle(msg: Message) {
  if (msg.kind === "Move") {
    console.log(msg.x, msg.y); // TS narrows the type
  }
}
```

**Rust (enum):**
Rust fuses the "tag" and the "payload" into one type.

```rust
enum Message {
    Quit,                       // No data
    Move { x: i32, y: i32 },    // Anonymous struct
    Write(String),              // Tuple
    ChangeColor(i32, i32, i32), // Tuple of three i32
}

```

**Mental model:** The enum definition doesn’t just define values; it defines **the shape of the data**. The `String` in `Write(String)` isn’t a separate heap object; it’s embedded in the enum’s layout.

---

## 2. Memory layout: how Rust stores enums

- **TypeScript:** Objects are heap hash maps. `{ kind: 'Move', x: 1, y: 2 }` stores property names (or Hidden Class pointer), `kind`, `x`, `y`.
- **Rust:** **Tagged union** layout.
  - **Discriminant (tag):** A small integer (often 1 byte) indicating which variant (Quit=0, Move=1, …).
  - **Payload (union):** Right after the tag; size is the **largest** variant.
  - **Padding:** For alignment.

**Example:** If `Quit` has 0 bytes, `Write(String)` has 24 bytes (String = ptr+cap+len), `Move` has 8 bytes, then each `Message` might be about 32 bytes on the stack (tag + padding + max payload).

**Conclusion:** Rust enum access is fast: contiguous stack memory, no property lookup.

---

## 3. Option: beyond the "billion dollar mistake"

TS has strict null checks, but `null` and `undefined` still exist as special values.
Rust removes `null` and uses a standard enum:

```rust
enum Option<T> {
    None,
    Some(T),
}

```

- **TS:** `let x: string | null = null;` — you must remember `if (x)`.
- **Rust:** `let x: Option<String> = None;` — you **cannot** use `x` as a String directly. The compiler forces you to unwrap or handle `None`.

That forces handling of absence at compile time and avoids runtime "Cannot read property of undefined".

---

## 4. `match`: switch on steroids

TS `switch` is value equality (`===`) and easy to forget `break`.
Rust `match` is **pattern matching** and is an expression.

### 4.1 Exhaustiveness

```rust
fn process(msg: Message) {
    match msg {
        Message::Quit => println!("Quit"),
        Message::Move { x, y } => println!("Move to {}, {}", x, y),
        Message::Write(text) => println!("Text: {}", text),
        // ❌ Compile error: missing `ChangeColor` branch
    }
}

```

**TS comparison:** In TS you might use `default: assertNever(msg)` to catch missing cases. In Rust, exhaustiveness is default.

### 4.2 Destructuring in patterns

`Message::Move { x, y }` binds `x` and `y` directly; cleaner than `case 'Move': const {x,y} = msg;` in TS.

### 4.3 Match guards

You can add extra conditions:

```rust
match msg {
    Message::Move { x, y } if x > 100 => println!("Big move!"),
    Message::Move { x, y } => println!("Normal move"),
    _ => (), // _ is wildcard, like default
}

```

### 4.4 Binding (@)

Sometimes you want to test a value and also bind it to a name:

```rust
match msg {
    Message::Move { x: id_variable @ 3..=7, y } => {
        println!("Found an id in range: {}", id_variable)
    },
    _ => (),
}

```

---

## 5. `if let`: sugar for one variant

When you only care about **one** variant and want to ignore the rest, `match` is verbose.

**TS:**

```typescript
if (msg.kind === "Write") {
  console.log(msg.content);
}
```

**Rust (`match`):**

```rust
match msg {
    Message::Write(text) => println!("{}", text),
    _ => (), // Boilerplate
}

```

**Rust (`if let`):**

```rust
if let Message::Write(text) = msg {
    println!("{}", text);
}

```

`if let` takes a pattern and an expression. You give up exhaustiveness for brevity.

---

## 6. Example: from TS to Rust

Design an HTTP request state machine.

### TypeScript

```typescript
type RequestState =
  | { status: "Idle" }
  | { status: "Loading" }
  | { status: "Success"; data: string }
  | { status: "Error"; error: Error };

function render(state: RequestState) {
  if (state.status === "Success") {
    return `Data: ${state.data}`;
  }
  // ...
}
```

### Rust

```rust
enum RequestState {
    Idle,
    Loading,
    Success(String),
    Error(String),
}

fn render(state: RequestState) -> String {
    match state {
        RequestState::Idle => String::from("Idle"),
        RequestState::Loading => String::from("Loading..."),
        RequestState::Success(data) => format!("Data: {}", data),
        RequestState::Error(err) => format!("Error: {}", err),
    }
}

```

**Important:** When you `match state`, if `state` holds heap data (e.g. `Success(String)`), ownership **moves** into the match arm. To only read and not consume, match on a **reference**:

```rust
fn render(state: &RequestState) {
    match state {
        RequestState::Success(s) => println!("Data: {}", s),
        _ => (),
    }
}

```

---

## 7. Summary

For TS developers, the key points are:

1. **Forget TS enum:** Rust enums are supercharged discriminated unions, not integer mappings.
2. **Embrace pattern matching:** Use `match` to destructure and branch; avoid ad-hoc property checks.
3. **Get used to Option/Result:** No null, no throw; only `None` and `Err`. That changes how you handle errors.
4. **Watch ownership:** Matching an enum can move data. To avoid that, match on `&Enum`.

Rust enums + match don’t just tidy code; when you add a new variant, the compiler tells you exactly where to update, enabling **fearless refactoring**.
