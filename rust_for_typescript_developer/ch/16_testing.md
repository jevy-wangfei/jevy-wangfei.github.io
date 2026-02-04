在 JavaScript/TypeScript 的世界里，测试通常是一个庞大的生态系统：你需要 Jest/Vitest/Mocha 作为运行器，需要 Chai/Expect 作为断言库，需要 Istanbul 做覆盖率，可能还需要 ts-jest 来处理转译。配置 `jest.config.js` 往往就是一场噩梦。

在 Rust 中，**测试是第一公民**。测试工具链直接内置在语言和编译器 (`rustc`) 以及包管理器 (`cargo`) 中。你不需要安装任何第三方库就可以编写高鲁棒性的单元测试和集成测试。

---

# 内置的信心：Rust 自动化测试深度解析

## 1. 测试解剖学：从 Jest 到 `#[test]`

在 TS 中，测试通常是一个函数调用 (`it(...)` 或 `test(...)`)。在 Rust 中，测试是一个被**属性 (Attribute)** 标记的普通函数。

### 1.1 基本结构

**TypeScript (Jest):**

```typescript
// math.test.ts
test("adds two numbers", () => {
  expect(2 + 2).toBe(4);
});
```

**Rust:**

```rust
// 这是一个普通的函数，但因为加了 #[test] 属性，
// 只有在运行 `cargo test` 时它才会被编译和运行。
#[test]
fn it_works() {
    let result = 2 + 2;
    assert_eq!(result, 4);
}

```

### 1.2 断言宏 (Assertions)

Rust 没有链式断言 (`expect(x).not.toEqual(y)`), 而是提供了一组简单的宏。

| 概念           | TypeScript (Jest)                 | Rust 宏                        | 备注                                     |
| -------------- | --------------------------------- | ------------------------------ | ---------------------------------------- |
| **真值断言**   | `expect(x).toBeTruthy()`          | `assert!(x)`                   | 接收布尔值，false 则 panic               |
| **相等断言**   | `expect(x).toBe(y)`               | `assert_eq!(x, y)`             | 要求参数实现 `PartialEq` 和 `Debug`      |
| **不等断言**   | `expect(x).not.toBe(y)`           | `assert_ne!(x, y)`             | 同上                                     |
| **自定义信息** | `expect(x).toBe(y, "Custom Msg")` | `assert!(x, "Error: {}", val)` | 所有断言宏都支持格式化字符串作为额外参数 |

**注意**：`assert_eq!` 会自动打印左右两边的值，这就是为什么它要求参数实现 `Debug` trait（大部分基础类型都实现了，自定义结构体需要 `#[derive(Debug)]`）。

### 1.3 失败机制：Panic

在 TS 中，测试失败是因为抛出了 Error。
在 Rust 中，测试失败是因为线程 **Panic** 了。`assert!` 宏在条件不满足时会触发 panic。

---

## 2. 单元测试 (Unit Tests)：私有可见性

这是 Rust 与 TS 最大的工程结构差异。

- **TypeScript**: 通常将测试文件放在 `__tests__` 目录或与源文件同级 `foo.test.ts`。TS 测试通常只能测试 `export` 的公共方法（除非你用 `rewire` 等黑魔法）。
- **Rust**: **单元测试直接写在源代码文件里**。Rust 允许测试私有 (Private) 函数。

### 2.1 惯用写法：`mod tests`

为了不污染正式代码，Rust 习惯在文件底部创建一个子模块专门放测试。

```rust
// src/lib.rs

// 1. 业务逻辑 (可能是私有的)
fn internal_adder(a: i32, b: i32) -> i32 {
    a + b
}

pub fn add_two(a: i32) -> i32 {
    internal_adder(a, 2)
}

// 2. 测试模块
#[cfg(test)] // ✨ 魔法：告诉编译器，只有在 cargo test 时才编译这个模块
mod tests {
    use super::*; // 关键：导入父模块的所有内容 (包括私有函数 internal_adder)

    #[test]
    fn internal() {
        assert_eq!(internal_adder(2, 2), 4); // ✅ 可以测试私有函数！
    }
}

```

**深度解析 `#[cfg(test)]**`：
这相当于 Tree-shaking 在源码级的应用。当你运行 `cargo build` (生产构建) 时，`tests`模块会被完全剔除，不会占用二进制体积。只有`cargo test` 会包含它。

---

## 3. 集成测试 (Integration Tests)：外部使用者视角

如果你想像库的用户一样测试你的代码（只访问 `pub` API），你需要 **集成测试**。

### 3.1 目录结构

Rust 约定项目根目录下的 `tests/` 目录是专门放集成测试的。

```text
my-project/
├── Cargo.toml
├── src/
│   └── lib.rs      (包含单元测试)
└── tests/          (集成测试目录)
    ├── integration_test.rs
    └── common/
        └── mod.rs

```

### 3.2 行为特征

`tests/` 下的每一个 `.rs` 文件都会被编译成一个**独立的 Crate**。
这意味着：

1. 它们必须像外部用户一样 `use my_project;` 导入你的库。
2. 它们只能访问 `pub` 的 API。

**TS 类比**：

- Unit Tests (`src/*.rs`) = `src/*.test.ts`
- Integration Tests (`tests/*.rs`) = `e2e/` 或 `cypress/` 文件夹，或者在一个单独的包里引你的库。

---

## 4. 控制测试执行：Clippy 的指挥棒

`cargo test` 命令非常强大，支持类似 Jest 的过滤功能。

### 4.1 过滤运行

- **运行所有测试**: `cargo test`
- **按名称过滤**: `cargo test add` (运行所有包含 "add" 名字的测试)
- **运行特定文件的测试**: `cargo test --test integration_test` (只运行 `tests/integration_test.rs`)

### 4.2 忽略测试 (`#[ignore]`)

有些测试太慢（比如数据库连接），你不想每次都跑。

```rust
#[test]
#[ignore] // 默认跳过
fn expensive_test() {
    // ...
}

```

要运行这些被忽略的测试：`cargo test -- --ignored`。

### 4.3 捕捉输出

默认情况下，如果测试通过，Rust 会**吞掉**你在代码里写的 `println!` 输出，保持控制台整洁。
如果你想看打印日志：`cargo test -- --show-output`。

---

## 5. 高级测试模式

### 5.1 测试 Panic (`#[should_panic]`)

在 TS 中你用 `expect(() => call()).toThrow()`。
在 Rust 中，你标记测试预期会 panic。

```rust
#[test]
#[should_panic(expected = "Divide by zero")] // 可选：检查 panic 消息包含特定字符串
fn test_divide_by_zero() {
    divide(1, 0);
}

```

### 5.2 使用 `Result` 进行测试

除了 Panic，Rust 测试还可以返回 `Result<(), E>`。这允许你在测试中使用 `?` 运算符，非常优雅。

```rust
#[test]
fn it_works() -> Result<(), String> {
    if 2 + 2 == 4 {
        Ok(())
    } else {
        Err(String::from("two plus two does not equal four"))
    }
}

```

**场景**：当测试中涉及文件操作或解析时，用 `?` 快速失败比写一堆 `unwrap()` 更易读。

---

## 6. Setup / Teardown (钩子函数？)

TS 开发者习惯了 `beforeAll`, `afterEach`。
**Rust 没有内置这些钩子。**

**为什么？**
因为 Rust 拥有 **RAII (资源获取即初始化)** 和 **Drop Trait**。
如果你需要测试后的清理工作（比如关闭数据库连接、删除临时文件），你应该定义一个 Struct，并在它的 `impl Drop` 中编写清理逻辑。当测试函数结束，Struct 离开作用域，清理逻辑自动执行。

对于 Setup，通常编写一个普通的 helper 函数。

```rust
// tests/common/mod.rs
pub fn setup() -> Connection {
    // 初始化数据库连接...
    Connection::new()
}

// tests/my_test.rs
#[test]
fn test_db() {
    let conn = common::setup();
    // 测试逻辑...
} // conn 离开作用域，自动断开连接

```

---

## 7. 总结：给 TS 开发者的测试备忘录

| 特性             | TypeScript (Jest/Vitest)       | Rust (Cargo Test)                  | 关键差异                            |
| ---------------- | ------------------------------ | ---------------------------------- | ----------------------------------- |
| **运行器**       | `npm test` (依赖 package.json) | `cargo test` (内置)                | Rust 无需配置                       |
| **断言**         | `expect(x).toBe(y)`            | `assert_eq!(x, y)`                 | Rust 基于宏，失败即 Panic           |
| **单元测试位置** | `foo.test.ts` (同级文件)       | `src/foo.rs` 内的 `mod tests`      | Rust 可测私有函数，生产环境自动剔除 |
| **集成测试位置** | `tests/` 或 `e2e/`             | 项目根目录 `tests/`                | 视为外部 Crate，只测 `pub` API      |
| **Mocking**      | `jest.fn()`, 动态替换          | 较难。需依赖 Trait 或 `mockall` 库 | 静态语言难以动态修改函数行为        |
| **预期错误**     | `.toThrow()`                   | `#[should_panic]`                  | 属性标记                            |
| **跳过测试**     | `.skip`                        | `#[ignore]`                        | 属性标记                            |

**深度洞察**：
Rust 的测试哲学是**显式且结构化**的。它没有 JS 那种极度的动态灵活性（比如随便 Mock 一个 import），这迫使你在编写代码时就考虑到可测试性（例如使用泛型和 Trait 来解耦依赖，以便注入 Mock 对象）。这虽然增加了初期的编码负担，但生成了架构更优良的代码。
