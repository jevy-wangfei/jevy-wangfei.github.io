作为 TS 开发者，你习惯了 `try/catch`。这套机制在 V8 引擎中工作良好，但它有一个巨大的隐患：**错误是不可见的**。你无法仅通过查看函数签名 `fn readFile(path: string): string` 知道它是否会抛出异常。你必须看文档，或者看源码。

Rust 认为这是糟糕的设计。Rust 采用了 **"错误即值" (Errors as Values)** 的哲学。没有异常（Exceptions），只有返回值。

---

# 显式的艺术：Rust 错误处理深度解析

## 0. 核心范式转移：从 Throw 到 Return

- **TypeScript (隐式控制流)**:

```typescript
// 函数签名没告诉你它会炸
function divide(a: number, b: number): number {
  if (b === 0) throw new Error("Division by zero");
  return a / b;
}

// 调用者必须"记得"去 catch
try {
  const result = divide(10, 0);
} catch (e) {
  // e 是 unknown 或 any，类型丢失了
}
```

- **Rust (显式类型约束)**:

```rust
// 函数签名明确告诉你：我可能会失败
fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        return Err(String::from("Division by zero"));
    }
    Ok(a / b)
} // 无需 try/catch，这是普通的 if/else 逻辑

```

---

## 1. 错误的分类：Panic vs Result

Rust 将错误分为两类：**不可恢复错误** 和 **可恢复错误**。

### 1.1 Panic! (不可恢复错误) —— 类似 `process.exit(1)`

当发生 Bug 时（如数组越界、除以零、内存损坏），Rust 会 `panic!`。这会导致程序立即打印错误堆栈并退出（或者在 WebAssembly 中抛出 JS 异常）。

```rust
fn main() {
    let v = vec![1, 2, 3];
    v[99]; // ❌ Panic: index out of bounds
}

```

**TS 开发者注意**：不要滥用 `panic!`。在 Rust 中，panic 意味着“程序员写了 Bug”或者“环境彻底坏了”。对于“文件没找到”、“网络断开”这种预期内的情况，**绝对不要**使用 panic。

### 1.2 Result (可恢复错误) —— 核心主角

绝大多数错误处理都是通过 `Result` 枚举完成的。

```rust
enum Result<T, E> {
    Ok(T),  // 成功，包含值 T
    Err(E), // 失败，包含错误 E
}

```

这非常像 TS 中的 `type Result<T, E> = { success: true, value: T } | { success: false, error: E }`，但 Rust 提供了极其丰富的组合子方法。

---

## 2. 处理 Result：解包的艺术

当你拿到一个 `Result` 时，你必须处理它，否则编译器会警告你。

### 2.1 笨办法：Pattern Matching

这是最基础的方式，类似于 TS 的 Discriminated Union 检查。

```rust
let f = File::open("hello.txt"); // f 是 Result<File, std::io::Error>

let f = match f {
    Ok(file) => file,
    Err(error) => {
        panic!("Problem opening the file: {:?}", error);
    },
};

```

### 2.2 暴力法：`unwrap` 和 `expect`

有时你确定这不会错，或者你就是想在出错时 crash（比如写原型代码）。
这相当于 TS 的非空断言 `!`，但如果断言失败，程序会挂掉。

- `unwrap()`: 成功则返回值，失败则 panic。
- `expect("msg")`: 同上，但可以自定义 panic 的错误信息（推荐）。

```rust
let f = File::open("hello.txt").unwrap(); // 如果文件不存在，进程直接挂掉
let f = File::open("hello.txt").expect("Failed to open hello.txt"); // 挂掉并打印消息

```

---

## 3. 错误传播：`?` 运算符 (The Question Mark Operator)

这是 Rust 错误处理中最精彩的部分。
在 TS 中，如果你想把错误抛给上层，你只需要不做 `try/catch`，错误会自动冒泡。
在 Rust 中，你需要显式返回 `Err`。为了避免 `match` 地狱，Rust 提供了 `?`。

**场景**：打开文件，读取内容，返回字符串。

**笨办法 (Match Hell)**:

```rust
fn read_username_from_file() -> Result<String, io::Error> {
    let f = File::open("hello.txt");
    let mut f = match f {
        Ok(file) => file,
        Err(e) => return Err(e), // 显式提前返回
    };

    let mut s = String::new();
    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => Err(e), // 显式返回
    }
}

```

**优雅办法 (`?` Operator)**:

```rust
fn read_username_from_file() -> Result<String, io::Error> {
    // 这里的 ? 做了三件事：
    // 1. 如果 Result 是 Ok，提取值赋给 f。
    // 2. 如果 Result 是 Err，立即 return Err(e) 退出当前函数。
    // 3. 自动进行错误类型转换 (调用 From trait)。
    let mut f = File::open("hello.txt")?;

    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}

```

**链式调用 (Chain)**:

```rust
fn read_username_from_file() -> Result<String, io::Error> {
    let mut s = String::new();
    // 极其优雅的链式写法
    File::open("hello.txt")?.read_to_string(&mut s)?;
    Ok(s)
}

```

**TS 类比**：`?` 就像是一个**同步的 `await**`，只不过 `await`等待的是 Promise 完成，而`?` 等待的是 Success，如果遇到 Error 直接抛出。

---

## 4. 组合子 (Combinators)：函数式编程的胜利

作为熟悉 `Array.prototype.map` 的 TS 开发者，你会爱上 Result 的组合子。你不需要 `match` 就能转换数据。

### 4.1 `map` 和 `map_err`

- `map`: 如果是 Ok，就处理值；如果是 Err，直接透传。
- `map_err`: 如果是 Err，处理错误类型（常用于将底层错误转为高层错误）；如果是 Ok，直接透传。

```rust
// TS: const val = res.success ? parse(res.value) : null;
let res: Result<String, _> = Ok("10".to_string());
let num: Result<i32, _> = res.map(|s| s.parse::<i32>().unwrap());

```

### 4.2 `and_then` (FlatMap)

当你的一连串操作每一个都可能出错时，用 `and_then` 链接。

```rust
fn cook_dinner() -> Result<String, String> {
    get_ingredients()           // 返回 Result<Ingredients>
        .and_then(|i| chop(i))  // chop 返回 Result<Chopped>
        .and_then(|c| cook(c))  // cook 返回 Result<Food>
        .and_then(|f| eat(f))
}

```

这消除了嵌套的 `if err != nil`，代码流线型极强。

### 4.3 `unwrap_or` 和 `unwrap_or_else`

提供默认值，类似 TS 的 `||` 或 `??`。

```rust
let config = File::open("config.json")
    .map(|f| parse(f))
    .unwrap_or(default_config()); // 如果中间任何一步出错，使用默认配置

```

---

## 5. 复杂场景：多种错误类型如何处理？

这是 Rust 新手最头疼的地方。
在 TS 中，`catch (e)` 里的 `e` 只要是 `any` 就可以 hold 住所有错误。
在 Rust 中，函数签名 `Result<T, E>` 的 `E` 必须是**一种**具体的类型。如果函数内部既可能报 `IoError` 又可能报 `ParseIntError`，怎么办？

### 5.1 快速方案：Trait Objects (动态分发)

类似于 TS 的 `Error` 接口。我们将所有错误都装箱（Box）成一个“实现了 Error trait 的动态对象”。

```rust
// Box<dyn Error> 意味着：任何实现了 Error 接口的东西
fn run() -> Result<(), Box<dyn std::error::Error>> {
    let f = File::open("config.txt")?; // IoError
    let num: i32 = "NaN".parse()?;     // ParseIntError
    Ok(())
}

```

优点：写起来快。缺点：调用者不知道具体是什么错误，只能打印出来看。

### 5.2 专业方案：自定义 Enum (静态分发)

定义一个包含所有可能性的 Enum，并实现转换。
(手动写很累，推荐使用 `thiserror` 库)

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum MyError {
    #[error("File system error")]
    Io(#[from] std::io::Error), // 自动生成 From impl

    #[error("Parsing error")]
    Parse(#[from] std::num::ParseIntError),
}

fn run() -> Result<(), MyError> {
    // ? 会自动把 IoError 转换成 MyError::Io
    let f = File::open("config.txt")?;
    Ok(())
}

```

### 5.3 应用层方案：Anyhow

如果你在写 Application（而不是 Library），只需要把错误往上抛并打印日志，强烈推荐使用 **`anyhow`** crate。它就像 Rust 版的“万能 Error 容器”，且带有漂亮的 Backtrace。

```rust
use anyhow::Result;

fn main() -> Result<()> {
    let config = read_config()?;
    Ok(())
}

```

---

## 6. 总结：给 TS 开发者的错误处理备忘录

| 场景             | TypeScript            | Rust                       | 备注                 |
| ---------------- | --------------------- | -------------------------- | -------------------- |
| **Bug/不可恢复** | `throw new Error()`   | `panic!()`                 | 数组越界、逻辑死胡同 |
| **可恢复错误**   | `try/catch`           | `Result<T, E>`             | 网络错误、文件缺失   |
| **空值**         | `null` / `undefined`  | `Option<T>`                | 数据库查不到记录     |
| **向上冒泡**     | 自动冒泡              | `?` 操作符                 | 必须显式处理         |
| **获取值/断言**  | `!` (Non-null assert) | `.unwrap()` / `.expect()`  | 风险操作，可能 Crash |
| **默认值**       | `val                  |                            | default`             |
| **多种错误**     | `catch (e: any)`      | `enum MyError` 或 `anyhow` | 强类型约束           |

**深度洞察**：
Rust 的错误处理虽然起步时觉得啰嗦（到处都是 `Result`），但它强迫你在写代码的那一刻就思考：“如果这里失败了，业务逻辑该怎么走？”。这消灭了 TS 项目中 90% 的“未捕获的 Promise 异常”和“undefined is not a function”运行时错误。
