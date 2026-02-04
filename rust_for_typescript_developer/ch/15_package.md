作为一个习惯了 NPM 生态和 ES Modules 的开发者，你可能认为“模块”就是文件，“包”就是 `node_modules` 里的文件夹。但在 Rust 中，这套体系是为了**编译性能**和**严格的边界控制**而设计的。

TypeScript 的模块系统是**基于文件系统 (File-system based)** 的（只要文件存在，就可以 import）。
Rust 的模块系统是**基于显式挂载 (Explicit Mounting)** 的（文件存在没用，必须显式声明它是模块树的一部分）。

---

# 组织代码的艺术：从 NPM 到 Cargo

## 1. 宏观层级：Package vs Crate vs Module

首先，我们需要对齐术语。在 TS 中，概念比较模糊（Project, Package, Module 经常混用）。Rust 有严格的层级定义：

| 概念               | TypeScript / Node.js 对应物 | Rust 定义                                                                      | 关键文件                      |
| ------------------ | --------------------------- | ------------------------------------------------------------------------------ | ----------------------------- |
| **Package**        | NPM Package                 | 一个项目工程，由 `Cargo.toml` 描述。可以包含 1 个库 Crate 和多个二进制 Crate。 | `Cargo.toml`                  |
| **Crate** (板条箱) | Build Entry / Bundle        | **编译单元**。编译器一次处理一个 Crate。分 Library 和 Binary 两种。            | `src/main.rs` 或 `src/lib.rs` |
| **Module**         | ES Module (单个 `.ts` 文件) | 代码的组织单元，用于划分作用域和隐私边界。                                     | `mod` 关键字                  |

### 1.1 核心差异：Crate 是什么？

在 TS 中，你运行 `tsc` 编译整个项目。
在 Rust 中，`rustc` 一次编译一个 Crate。

- **Binary Crate**: 编译成可执行文件 (`src/main.rs`)。
- **Library Crate**: 编译成库，供其他 Crate 引用 (`src/lib.rs`)。

一个 Package 必须有一个 `Cargo.toml`，它可以包含：

- **最多一个** Library Crate。
- **任意多个** Binary Crates (放在 `src/bin/` 下)。

---

## 2. 模块系统 (Modules)：最反直觉的部分

这是 TS 开发者最容易撞墙的地方。

### 2.1 显式挂载 (Explicit Declaration)

在 TS 中，你在文件夹里新建 `utils.ts`，然后直接 `import { func } from './utils'`，一切正常。

在 Rust 中，如果你新建 `src/utils.rs`，编译器**完全看不到它**，就像它不存在一样。你必须在它的**父模块**（通常是 `main.rs` 或 `lib.rs`）中显式声明它：

```rust
// src/main.rs

// 这一行告诉编译器：
// "去把 utils.rs (或 utils/mod.rs) 的内容加载进来，
// 并作为当前 crate 下的一个子模块，名字叫 utils"
mod utils;

fn main() {
    utils::do_something();
}

```

**心智模型**：Rust 的模块是一棵**逻辑树**。文件系统只是这棵树的一种存储形式。你必须手动把文件“挂载”到树的节点上。

### 2.2 模块定义的两种方式

假设我们要构建 `crate::front_of_house::hosting` 结构。

**方式 A: 内联 (Inline) —— 类似 TS 的 Namespace**
适合很小的模块。

```rust
// src/lib.rs
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

```

**方式 B: 文件系统映射 (File System)**
这更像 TS 的习惯。

```text
src/
├── lib.rs
└── front_of_house.rs (或者 front_of_house/mod.rs)

```

```rust
// src/lib.rs
mod front_of_house; // 声明：内容在别的文件里，去找吧

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
}

```

**注意**：Rust 2018 以后，不再强制要求 `front_of_house/mod.rs`（类似 JS 的 `index.js`），你可以直接用 `front_of_house.rs`。但如果你有子模块，还是需要文件夹结构。

---

## 3. 路径与引用 (Paths & Use)

### 3.1 绝对路径 vs 相对路径

- **TS**:
- 绝对: `import ... from '@/utils'` (需要配置 tsconfig paths)
- 相对: `import ... from '../../utils'`

- **Rust**:
- **`crate::` (绝对路径)**: 从 Crate 的根 (`main.rs`/`lib.rs`) 开始。等同于 TS 的项目根目录别名 `@/`。
- **`super::` (相对路径)**: 父级模块。等同于 TS 的 `../`。
- **`self::` (相对路径)**: 当前模块。等同于 TS 的 `./`。

```rust
// 使用绝对路径 (推荐)
crate::front_of_house::hosting::add_to_waitlist();

// 使用相对路径
super::hosting::add_to_waitlist();

```

### 3.2 `use` 关键字

`use` 类似于 TS 的 `import`，或者是 `const x = require(...)` 的简化版，它创建一个**软链接（Symlink）**到当前作用域。

```rust
// TS: import { add_to_waitlist } from './front_of_house/hosting';
use crate::front_of_house::hosting::add_to_waitlist;

pub fn eat_at_restaurant() {
    add_to_waitlist();
}

```

### 3.3 重导出 (`pub use`) —— Facade 模式

在 TS 中，为了让 API 更好看，我们经常在 `index.ts` 里写：
`export * from './internal-module';`

在 Rust 中，这就是 `pub use`。这被称为 **Re-exporting**。
它允许你解耦**内部代码结构**和**对外公开 API**。

```rust
// src/lib.rs
mod front_of_house; // 内部实现，可能是私有的

// 对外暴露，用户可以直接 use my_crate::hosting;
// 而不需要知道它实际是在 front_of_house 里
pub use crate::front_of_house::hosting;

```

---

## 4. 可见性 (Visibility)：隐私的艺术

TS 只有 `public` (默认) 和 `private` (编译期检查，运行时无)。Rust 的隐私检查是**强制的**。

### 4.1 默认私有

Rust 中，所有定义（函数、结构体、模块、字段）默认都是 **Private** 的。父模块无法访问子模块的私有内容，但子模块可以访问父模块的所有内容（上下文可见性）。

### 4.2 `pub` 关键字

- **TS**: `export const foo = ...`
- **Rust**: `pub fn foo() ...`

### 4.3 细粒度控制 (TS 没有的功能)

Rust 允许你把可见性限制在特定的范围内，这在大型项目中非常有用：

- `pub`: 对所有人可见。
- `pub(crate)`: **只对当前 Crate 可见**。这对外部用户是私有的，但对你项目里的其他模块是公开的。（TS 中我们常渴望这种 "Package Private" 的功能）。
- `pub(super)`: 只对父模块可见。

```rust
mod back_of_house {
    pub struct Breakfast {
        pub toast: String,      // 顾客可以选面包类型
        seasonal_fruit: String, // 顾客不能选水果，厨师根据季节决定 (Private)
    }

    impl Breakfast {
        // 必须提供构造函数，因为外部无法直接初始化包含 private 字段的结构体
        pub fn summer(toast: &str) -> Breakfast {
            Breakfast {
                toast: String::from(toast),
                seasonal_fruit: String::from("peaches"),
            }
        }
    }
}

pub fn eat_at_restaurant() {
    let mut meal = back_of_house::Breakfast::summer("Rye");
    meal.toast = String::from("Wheat"); // ✅ 可以改公有字段
    // meal.seasonal_fruit = String::from("blueberries"); // ❌ 编译错误：字段私有
}

```

---

## 5. Workspaces：Monorepo 原生支持

你肯定用过 Lerna, Nx, Turborepo 或者 Pnpm Workspaces。
Cargo 原生内置了 **Workspaces**。

### 场景

你的项目 `my-app` 包含：

1. 一个核心算法库 (`core`)
2. 一个后端 API (`server`)
3. 一个 CLI 工具 (`cli`)

### 目录结构

```text
my-app/
├── Cargo.toml (Workspace Root)
├── crates/
│   ├── core/
│   │   └── Cargo.toml
│   ├── server/
│   │   └── Cargo.toml
│   └── cli/
│       └── Cargo.toml

```

### 配置

根目录 `Cargo.toml`:

```toml
[workspace]
members = [
    "crates/core",
    "crates/server",
    "crates/cli",
]

```

在 `crates/server/Cargo.toml` 中引用 `core`：

```toml
[dependencies]
core = { path = "../core" }

```

**优势**：

- **共享 `Cargo.lock**`：保证所有子包依赖版本一致。
- **共享 `target` 目录**：公共依赖（如 `serde`）只需要编译一次，所有子包复用。TS 的 `node_modules` 还需要 hoisted 才能做到。

---

## 6. 引入外部包 (External Packages)

这就是 Rust 版的 `npm install`。

1. 打开 `Cargo.toml`。
2. 在 `[dependencies]` 下添加：

```toml
rand = "0.8.5"

```

3. 在代码中直接使用：

```rust
use rand::Rng; // 这里的 rand 指向的是 Cargo.toml 里定义的外部 crate

```

**Std vs External**:

- `std::...`: 标准库（类似于 JS 的 `Math`, `JSON` 等内置对象，但 Rust 需要显式 use）。
- `rand::...`: 第三方库（类似于 `node_modules`）。

---

## 总结：给 TS 开发者的对照表

| 特性         | TypeScript                   | Rust                     | 核心逻辑                                   |
| ------------ | ---------------------------- | ------------------------ | ------------------------------------------ |
| **引入**     | `import { x } from './file'` | `mod file; use file::x;` | TS 基于文件查找；Rust 基于模块树挂载       |
| **项目根**   | `package.json`               | `Cargo.toml`             | 依赖管理与构建配置                         |
| **命名空间** | namespace (不常用) / File    | `mod`                    | 逻辑隔离                                   |
| **默认可见** | 文件内可见 / export 公开     | 私有 (Private)           | Rust 默认封闭，TS 默认封闭但 `export` 泛滥 |
| **包内共享** | 无（只能不 export 或用注释） | `pub(crate)`             | 允许 Crate 内部随意访问，对外隐藏          |
| **Monorepo** | Pnpm/Yarn Workspaces         | Cargo Workspaces         | 共享依赖构建缓存                           |
| **入口**     | `index.ts`                   | `lib.rs` / `main.rs`     | 模块树的根节点                             |

**深度洞察**：
Rust 这种繁琐的 `mod` 声明看似多余，实际上它让编译器能够**构建一个确定的依赖图**，而不需要像 Webpack 那样去扫描分析文件的 `import` 语句。这也是 Rust 编译虽慢（在做很多检查），但链接和死代码消除（Dead Code Elimination）非常高效的原因。

**最后一步**：
当你写 Rust 时，不要上来就写代码。先在脑子里（或纸上）画出模块树：
`Crate Root -> mod A -> mod B -> struct MyData`。
确定好树结构后，再用 `mod` 关键字把它们串起来。
