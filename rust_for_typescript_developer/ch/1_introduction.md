这份速查表专为 TypeScript 开发者设计，旨在帮助你利用现有的知识映射到 Rust 的语法。

### 1. 变量与可变性 (Variables & Mutability)

在 TS 中，`const` 意味着引用不可变，但对象内部可变。在 Rust 中，默认全是不可变的（Deeply Immutable）。

| 概念     | TypeScript          | Rust                        | 备注                                              |
| -------- | ------------------- | --------------------------- | ------------------------------------------------- |
| 定义变量 | `let x = 5;`        | `let x = 5;`                | Rust 默认**不可变**                               |
| 可变变量 | `let x = 5; x = 6;` | `let mut x = 5; x = 6;`     | 必须显式加 `mut`                                  |
| 常量     | `const MAX = 100;`  | `const MAX: u32 = 100;`     | Rust 常量必须标注类型，且是编译时常量             |
| 变量遮蔽 | (不推荐)            | `let x = 5; let x = x + 1;` | Rust 允许 Shadowing（重用变量名），常用于类型转换 |

### 2. 基础数据类型 (Data Types)

Rust 是静态强类型语言，但它的类型推断（Type Inference）非常智能。通常你不需要像 Java 那样到处写类型，编译器能根据上下文（比如赋值的值）推导出类型，这点和 TS 非常像。

Rust 的基础类型分为 **标量（Scalar）** 和 **复合（Compound）** 两类。

#### 2.1 标量类型 (Scalars)

代表一个单独的值。

- **整数 (Integers)**:
- **TS:** 只有 `number` (IEEE 754 双精度浮点) 和 `BigInt`。
- **Rust:** 你必须显式选择位宽。
- `i8`, `u8`, `i16`, `u16`, `i32` (默认), `u32`, `i64`, `u64`, `i128`, `u128`。
- **特别关注**: `isize` / `usize`。这就相当于 C++ 的 `size_t`，其长度取决于 CPU 架构（64位机上是 64位）。**在 Rust 中，数组索引和内存偏移必须使用 `usize`。**

- **浮点数 (Floating-Point)**:
- `f32`: 单精度。
- `f64`: 双精度（默认）。**这完全等同于 TS 的 `number`。**

- **布尔 (Booleans)**:
- `bool`: `true` / `false`。占用 1 字节。

- **字符 (Characters)**:
- **TS:** `'a'` 只是一个长度为 1 的字符串（UTF-16）。
- **Rust:** `char` 是 **4 字节** 的 Unicode Scalar Value。它可以直接存储 Emoji `😻`，不仅仅是 ASCII。
- _注意：Rust 的字符串不是 `char` 的数组，而是 UTF-8 字节序列。_

#### 2.2 复合类型 (Compounds)

可以将多个值组合成一个类型，分配在**栈 (Stack)** 上。

- **元组 (Tuples)**:
- **TS:** `[string, number]`。
- **Rust:** `(i32, f64, u8)`。长度固定，类型可以不同。
- **Unit 类型 `()**`: 空元组。类似于 TS 的 `void`，但这在 Rust 中是一个实际存在的“空值”，占用 0 内存。

- **数组 (Arrays)**:
- **TS:** `const arr = [1, 2, 3]` 是动态的堆数组 (List)。
- **Rust:** `[T; N]` 是**固定长度**的。
- `let a = [1, 2, 3, 4, 5];` (类型是 `[i32; 5]`)。
- **关键区别**: Rust 的 Array 分配在**栈**上（如果不大）。你不能 `push` 或 `pop`。
- _如果你想要 TS 那种可变长度数组，请使用标准库的 `Vec<T>`（Vector）。_

#### 2.3 字符串：String vs &str (难点解析)

这是 TS 开发者最容易混淆的概念，因为在 TS 中只有一种 `string`（V8 内部虽然有 Rope/ConsString 优化，但对用户透明）。

- **String (所有者)**:
- **内存位置**: 堆 (Heap)。
- **TS 类比**: 类似于 `new String("...")` 或一个可变的 `StringBuilder`。
- **特性**: 拥有数据的所有权，可以修改（`mut`），可以增长（`push_str`）。它包含三个字段：ptr (指针), len (长度), capacity (容量)。

- **&str (借用者/切片)**:
- **内存位置**: 它本身（胖指针）在栈上，但它**指向**的数据可以在任何地方（静态存储区、堆、栈）。
- **TS 类比**: 一个只读的、定长的“视图窗口” (View)。
- **特性**: 它是一个“胖指针” (Fat Pointer)，包含：ptr (指向数据的起始位置), len (长度)。它不拥有数据，只是借来看一眼。字符串字面量 `"hello"` 的类型就是 `&str`（指向静态内存）。

**代码对比：**

```rust
// String: 我拥有这段内存，我要申请堆空间
let mut s: String = String::from("hello");
s.push_str(" world"); // OK，自动扩容

// &str: 我只是借来看一眼，不负责分配和释放
let slice: &str = &s[0..5]; // 指向 s 的前 5 个字节
// slice.push_str("!"); // ❌ 编译错误，&str 是不可变的

```

### 3. 函数与隐式返回 (Functions)

Rust 可以与TS一样使用 return xxx; (注意分号），也可以采用了类似 Ruby/Lisp 的隐式返回，这在函数式编程中很常见。

```rust
// TypeScript
function add(a: number, b: number): number {
  return a + b;
}

// Rust
// TypeScript
function add(a: number, b: number): number {
  return a + b;
}

// Rust
fn add(a: i32, b: i32) -> i32 {
  reutrn a + b;
}

fn add(a: i32, b: i32) -> i32 {
  a + b  // 注意：没有分号！加上分号就变成了 Statement，返回 Unit ()
}

```

### 4. 结构体与方法 (Structs & Impl)

Rust 没有 `class`。它把**数据定义**和**方法实现**分开了。

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

// 类似于 Extension Method，在外部定义方法
impl User {
    // 关联函数 (类似静态方法 static method)，通常用于构造器
    fn new(name: &str) -> User {
        User { username: name.to_string() }
    }

    // 方法 (Instance method)，&self 类似于 this
    fn describe(&self) {
        println!("User: {}", self.username);
    }
}

```

### 5. 接口 vs Traits

TS 的 Interface 是结构化类型系统（鸭子类型）。Rust 的 Trait 是名义类型系统，必须显式实现。

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

### 6. 枚举与模式匹配 (Enums & Match)

这是 Rust 的杀手级特性。TS 的 Enum 只是数字或字符串映射。Rust 的 Enum 是**代数数据类型 (ADT)**，可以携带数据。

**TypeScript:**

```typescript
type Action =
  | { type: "Quit" }
  | { type: "Move"; x: number; y: number }
  | { type: "Write"; content: string };
// 需要手动判断 type 字段
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
        Action::Move { x, y } => println!("Move to {}, {}", x, y), // 解构
        Action::Write(s) => println!("Write: {}", s),
    }
}
// 编译器会强制你处理所有 case，绝不会漏掉

```

### 7. 空值与错误处理 (No Null, No Exceptions)

Rust 没有 `null`，也没有 `undefined`，更没有 `try/catch`。

- **Option<T>**: 替代 `null/undefined`。Rust 可以与TS一样使用 return xxx; (注意分号），也可以采用了类似 Ruby/Lisp 的隐式返回，这在函数式编程中很常见。
- **Result<T, E>**: 替代异常。Result也是Rust的一个 Enum, Ok<T>和Err<E>是Result Enum的两个case

```rust
// TypeScript:
function find(): string | null {}
// Rust,
fn find_user(id: i32) -> Option<String> {
    if id == 1 {
        Some("Alice".to_string())
    } else {
        None // 类似于 null，但必须显式处理
    }
}

// 使用 match 处理 Option
match find_user(1) {
    Some(name) => println!("Found: {}", name),
    None => println!("No user found"),
}

// 或者使用类似 TS Optional Chaining 的组合子，注意这里|name| println!("Found: {}", name)是Rust闭包
find_user(1).map(|name| println!("Found: {}", name));

```

### 8. 闭包与迭代器 (Closures & Iterators)

语法非常像 TS 的箭头函数。

**TypeScript:**

```typescript
[1, 2, 3].map((x) => x + 1).filter((x) => x > 2);
```

**Rust:**

```rust
let nums = vec![1, 2, 3]; // 宏，用于创建 Vector
let result: Vec<i32> = nums
    .iter()             // 必须先获取迭代器
    .map(|x| x + 1)     // 管道操作，|x| 是闭包参数， x+1是闭包函数
    .filter(|x| *x > 2) // filter 需要解引用，因为 iter 产出的是引用
    .collect();         // 惰性求值！必须调用 collect 才会真正执行循环

```

### 给 TS 资深开发者的建议

1. **关于分号：** 在 Rust 中，表达式（Expression）作为返回值时**不能加分号**，语句（Statement）必须加分号。这是初学者最容易混淆的地方。
2. **关于引用：** 在 TS 中，对象默认是引用传递。在 Rust 中，你需要显式选择是“借用” (`&User`) 还是“移动所有权” (`User`)。如果函数参数不需要修改数据，永远使用 `&`。
3. **泛型：** Rust 的泛型 `<T>` 和 TS 的 `<T>` 几乎一模一样，你会感觉很亲切。

生命周期 (Lifetimes) 是 TS 中完全不存在的概念。
