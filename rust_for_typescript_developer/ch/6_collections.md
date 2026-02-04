这是一篇为资深 TypeScript 开发者深度定制的 **Rust 常用集合 (Common Collections)** 解析文章。

在 TypeScript/JavaScript 的世界里，你只有两种主要的容器：`Array`（既是列表也是栈也是队列）和 `Object/Map`（键值对）。V8 引擎在背后为你做了大量的动态优化（如 Hidden Classes, Sparse Arrays）。

但在 Rust 中，为了零成本抽象和内存安全，标准库提供了更细粒度、内存行为更明确的集合类型。我们已经了解了：**Vector (`Vec<T>`)**和部分**String (`String`)**，本节将再次深入了解**String (`String`)**，和 **HashMap (`HashMap<K, V>`)**。

---

# Rust 常用集合解析

## 1. String: UTF-8 的痛与爱

TS 的 `string` 是 UTF-16 编码（受 Java 早期影响）。
Rust 的 `String` 是 **UTF-8** 编码。

这一区别导致了 Rust 的字符串操作对 TS 开发者来说非常反直觉：**你不能通过索引访问字符串。**

### 1.1 为什么 `s[0]` 是非法的？

在 TS 中，`"你好"[0]` 也是乱码（因为“你”在 UTF-16 中占两个单元），但 JS 允许你这么做。
Rust 拒绝编译 `s[0]`。

```rust
let s = String::from("你好");
// let h = s[0]; // ❌ 编译错误

```

原因：

1. **字节 (Bytes)**: `String` 是 `Vec<u8>` 的封装。“你”在 UTF-8 中是 3 个字节 `[228, 189, 160]`。如果你取 `s[0]` 拿到 228，这没有任何语义意义。
2. **性能承诺**: 程序员预期索引操作是 O(1) 的。但由于 UTF-8 是变长的，要找到第 N 个字符，必须从头遍历扫描。Rust 不会把一个 O(N) 的操作伪装成 O(1)。

### 1.2 切片 (Slicing) 的危险

你可以切片，但必须切在**字符边界**上。

```rust
let hello = "你好";
let s = &hello[0..3]; // ✅ "你" (占3字节)
// let s = &hello[0..1]; // ❌ Panic! 切在了字符的中间

```

### 1.3 遍历字符串

Rust 强迫你选择视角：是看字节，还是看字符？

```rust
let s = "你好";

// 视角 1: 字符 (Unicode Scalar Values)
for c in s.chars() {
    println!("{}", c); // 打印出 4 个字符
}

// 视角 2: 字节 (Bytes)
for b in s.bytes() {
    println!("{}", b); // 打印出 18 个字节
}

```

### 1.4 拼接：`+` vs `format!`

- **TS**: `Hello ${name}`
- **Rust**:

```rust
let s1 = String::from("Hello, ");
let s2 = String::from("world!");

// 方法 1: + 运算符
// 注意：s1 所有权被移交走了！s1 变废了。
// 底层调用 fn add(self, s: &str) -> String
let s3 = s1 + &s2;

// 方法 2: format! 宏 (推荐，类似 Template Literals)
// 不会夺走所有权，只是引用。
let s1 = String::from("Hello");
let s = format!("{}, {}!", s1, s2);

```

## 2. HashMap: 严格的键值对

TS 的 `Map` 或 `Object` 非常灵活。Rust 的 `HashMap` 同样基于 Hash 表实现，但它对 **Hash 算法** 和 **所有权** 有严格要求。

### 2.1 所有权转移 (Ownership Transfer)

这是最容易坑 TS 开发者的点。

**TypeScript:**

```typescript
let field = "color";
let map = new Map();
map.set(field, "blue");
console.log(field); // ✅ 依然可用
```

**Rust:**

```rust
use std::collections::HashMap;

let field_name = String::from("Favorite color");
let field_value = String::from("Blue");

let mut map = HashMap::new();
// 所有权转移！field_name 和 field_value 被 Move 进了 map
map.insert(field_name, field_value);

// println!("{}", field_name); // 编译错误：value borrowed here after move

```

**解决方案**：如果你想保留原变量，必须 `.clone()` 它们，或者存引用 `&str`（但这需要处理复杂的生命周期）。通常建议：**让 HashMap 拥有数据 (`String`)**。

### 3.2 获取值：`get`

```rust
// 返回 Option<&V>
match map.get(&String::from("Favorite color")) {
    Some(val) => println!("Value: {}", val),
    None => println!("Not found"),
}

```

### 2.3 Entry API: 优雅的 "不存在则插入"

在 TS 中，我们要实现“统计单词词频”，通常这么写：

**TypeScript:**

```typescript
const count = {};
const word = "hello";
if (!count[word]) {
  count[word] = 0;
}
count[word] += 1; // 这里的查询其实可能进行了两次 Hash 查找
```

**Rust (Entry API):**
Rust 提供了一个视图（View）API，叫 `entry`，专门处理这种情况，且只计算一次 Hash，性能更高。

```rust
let mut scores = HashMap::new();
let team_name = String::from("Blue");

// 读作：获取 "Blue" 的 Entry，如果是空 (or_insert)，就插入 50。
// 返回一个可变引用 &mut V
let count = scores.entry(team_name).or_insert(0);

// 直接通过解引用修改值
*count += 1;

```

这一行代码体现了 Rust 的性能：**高层的抽象（API 易用），底层的极致（只做一次 Hash 查找）。**

### 2.4 Hash 算法：安全 vs 速度

- **TS (V8)**: 为了快，使用确定性算法（但在某些版本可能遭受 Hash DoS 攻击）。
- **Rust**: 默认使用 **SipHash**。它能抵抗 Hash DoS 攻击（即黑客构造大量 Key 让你的 Hash Map 退化成链表）。
- 代价：稍微慢一点点。
- 自定义：如果你像 V8 那样追求极致速度且信任数据源，可以替换 Hasher（例如 `FnvHash`）。

### Rust vs TypeScript 常用集合行为对照表

| 场景                    | TypeScript (JS/V8)                                                                                | Rust (Vec / String / HashMap)                                                                                           | 核心差异 (Why?)                                                                                          |
| ----------------------- | ------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| **内存扩容**            | **隐式黑盒**                                                                                      | <br>V8 自动管理，可能在 Packed/Holey 模式间切换，性能不可预测。                                                         | **显式控制**<br> <br>`Vec` 自动扩容涉及内存搬运(Memcpy)。<br><br>推荐用 `Vec::with_capacity(N)` 预分配。 | **性能 (Performance)**<br> <br>Rust 让你避免昂贵的内存重新分配开销。 |
| **数组越界** `arr[i]`   | **返回 undefined**<br><br>运行时静默失败，容易导致后续的 `Cannot read prop of undefined`。        | **二选一**<br><br>1. `v[i]`: **Panic** (崩溃)，假设你逻辑没错。<br><br>2. `v.get(i)`: 返回 **`Option`**，强制处理空值。 | **安全 (Safety)**<br><br>Rust 拒绝“未定义行为”，要么崩溃，要么显式处理。                                 |
| **字符串索引** `str[0]` | **允许 (但有坑)**<br><br>返回 UTF-16 码元。对于 Emoji (🍎) 会返回乱码，虽然是 O(1) 但可能是错的。 | **编译错误**<br><br>禁止索引。必须用 `.chars()` (迭代) 或切片。<br><br>防止将 O(N) 操作误认为是 O(1)。                  | **正确性 (Correctness)**<br><br>Rust 字符串是 UTF-8 (变长)，无法在 O(1) 时间定位第 N 个字符。            |
| **遍历中修改**          | **允许 (有副作用)**<br><br>编译通过。运行时可能导致死循环、跳过元素或迭代器失效。                 | **编译错误**<br><br>借用检查器拦截：不能在持有不可变引用 (`&vec`) 时获取可变引用 (`vec.push`)。                         | **内存安全 (Memory Safety)**<br><br>防止扩容导致的悬垂指针 (Dangling Pointer)。                          |
| **Map 插入**            | **引用复制**<br><br>Key/Value 只是引用的拷贝，原对象依然可用。                                    | **所有权转移 (Move)**<br><br>默认情况下，插入 Map 意味着把变量的所有权“上交”给 Map 管理。                               | **资源管理 (RAII)**<br><br>Map 负责释放内存。                                                            |
| **Map 统计**            | **多次 Hash**<br><br>`if(!has) set; set(get+1)`<br><br>需要 2-3 次 Hash 查找。                    | **一次 Hash (Entry API)**<br><br>`map.entry(k).or_insert(0)`<br><br>直接操作底层内存桶位，零成本抽象。                  | **极致优化 (Zero-cost)**<br><br>用更高级的 API 换取更少的 CPU 指令。                                     |

**深度建议：**
当你使用 Rust 集合时，不要把它仅仅当作数据的容器，而要把它当作**资源的管理器**。当你把一个 String `insert` 到 HashMap 时，你不仅仅是存了个值，你是把这块内存的管理权交接给了这个 Map。理解了这一点，借用检查器的报错就不再是阻碍，而是提示。
