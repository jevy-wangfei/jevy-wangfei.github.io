This is a **Rust common collections** guide tailored for experienced TypeScript developers.

In TypeScript/JavaScript you mainly have `Array` (list, stack, queue) and `Object/Map` (key-value). V8 does a lot of dynamic optimization (Hidden Classes, sparse arrays).

In Rust, for zero-cost abstraction and memory safety, the standard library offers finer-grained collections with explicit memory behavior. We've already seen **Vector (`Vec<T>`)** and part of **String**. This section goes deeper into **String** and **HashMap (`HashMap<K, V>`)**.

---

# Rust common collections

## 1. String: UTF-8 pain and love

TS `string` is UTF-16 (legacy from Java).
Rust `String` is **UTF-8**.

That difference makes string operations in Rust feel unintuitive to TS developers: **you cannot index into a string.**

### 1.1 Why is `s[0]` invalid?

In TS, `"你好"[0]` is also wrong ( "你" is two UTF-16 code units), but JS allows it.
Rust refuses to compile `s[0]`.

```rust
let s = String::from("你好");
// let h = s[0]; // ❌ Compile error

```

Reasons:

1. **Bytes:** `String` is a wrapper over `Vec<u8>`. "你" in UTF-8 is 3 bytes `[228, 189, 160]`. `s[0]` would be 228, which has no character meaning.
2. **Performance contract:** Programmers expect indexing to be O(1). In UTF-8, finding the Nth character is O(N). Rust won't pretend an O(N) operation is O(1).

### 1.2 Slicing pitfalls

You can slice, but only on **character boundaries**.

```rust
let hello = "你好";
let s = &hello[0..3]; // ✅ "你" (3 bytes)
// let s = &hello[0..1]; // ❌ Panic! Cut in the middle of a character

```

### 1.3 Iterating over strings

Rust forces you to choose: bytes or characters?

```rust
let s = "你好";

// View 1: Characters (Unicode scalar values)
for c in s.chars() {
    println!("{}", c); // 4 characters
}

// View 2: Bytes
for b in s.bytes() {
    println!("{}", b); // 18 bytes
}

```

### 1.4 Concatenation: `+` vs `format!`

- **TS:** `Hello ${name}`
- **Rust:**

```rust
let s1 = String::from("Hello, ");
let s2 = String::from("world!");

// Method 1: + operator
// Note: s1 is moved! s1 is invalid after this.
// Under the hood: fn add(self, s: &str) -> String
let s3 = s1 + &s2;

// Method 2: format! macro (recommended, like template literals)
// Doesn't take ownership; just references.
let s1 = String::from("Hello");
let s = format!("{}, {}!", s1, s2);

```

## 2. HashMap: strict key-value

TS `Map` or `Object` is very flexible. Rust `HashMap` is also hash-based but has strict rules for **hashing** and **ownership**.

### 2.1 Ownership transfer

This is where TS developers get bitten most.

**TypeScript:**

```typescript
let field = "color";
let map = new Map();
map.set(field, "blue");
console.log(field); // ✅ Still valid
```

**Rust:**

```rust
use std::collections::HashMap;

let field_name = String::from("Favorite color");
let field_value = String::from("Blue");

let mut map = HashMap::new();
// Ownership moves into the map!
map.insert(field_name, field_value);

// println!("{}", field_name); // Compile error: value borrowed here after move

```

**Workaround:** To keep the original variables, `.clone()` them or store references `&str` (and deal with lifetimes). Often the best approach: **let the HashMap own the data (`String`)**.

### 2.2 Getting values: `get`

```rust
// Returns Option<&V>
match map.get(&String::from("Favorite color")) {
    Some(val) => println!("Value: {}", val),
    None => println!("Not found"),
}

```

### 2.3 Entry API: "insert if absent"

In TS, "count word frequency" often looks like:

**TypeScript:**

```typescript
const count = {};
const word = "hello";
if (!count[word]) {
  count[word] = 0;
}
count[word] += 1; // May do two hash lookups
```

**Rust (Entry API):**
Rust has an `entry` API for this case; it does at most one hash lookup.

```rust
let mut scores = HashMap::new();
let team_name = String::from("Blue");

// Read: get entry for "Blue"; if vacant (or_insert), insert 50.
// Returns &mut V
let count = scores.entry(team_name).or_insert(0);

*count += 1;

```

This shows Rust's philosophy: **high-level API, low-level efficiency (one hash lookup).**

### 2.4 Hash: safety vs speed

- **TS (V8):** Fast, deterministic hash (some versions vulnerable to Hash DoS).
- **Rust:** Default is **SipHash**, resistant to Hash DoS (attacker can’t force long chains).
- Cost: Slightly slower.
- Custom: For maximum speed when you trust the input, you can use another hasher (e.g. `FnvHash`).

### Rust vs TypeScript collections (summary)

| Scenario | TypeScript (JS/V8) | Rust (Vec / String / HashMap) | Main difference |
| ----------------------- | ------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| **Resize** | Implicit black box; V8 may switch layouts; behavior not guaranteed | Explicit; `Vec` resize involves memcpy; prefer `Vec::with_capacity(N)` | **Performance** — Rust lets you avoid costly reallocations. |
| **Out-of-bounds `arr[i]`** | Returns `undefined`; silent failure, then "Cannot read prop of undefined" | Either: `v[i]` **panics**, or `v.get(i)` returns **`Option`** — you must handle absence | **Safety** — Rust avoids undefined behavior. |
| **String index `str[0]`** | Allowed (but wrong for multi-byte/emoji); O(1) but misleading | **Compile error**; use `.chars()` or slicing; no O(1) char index for UTF-8 | **Correctness** — UTF-8 is variable-length. |
| **Mutate while iterating** | Allowed; can cause infinite loop or iterator invalidation | **Compile error**; borrow checker forbids mutable access while holding `&vec` | **Memory safety** — avoids dangling pointers from resize. |
| **Map insert** | Reference copy; key/value are copied; originals still valid | **Move**; inserting usually transfers ownership to the map | **RAII** — map is responsible for freeing. |
| **Map "get or insert"** | `if(!has) set; set(get+1)` — 2–3 hash lookups | **Entry API** — `map.entry(k).or_insert(0)` — one lookup, zero-cost abstraction | **Optimization** — higher-level API, fewer instructions. |

**Takeaway:** Treat Rust collections as **resource managers**, not just containers. When you `insert` a `String` into a `HashMap`, you're handing over ownership of that memory. Once you see that, borrow checker errors become guidance, not obstacles.
