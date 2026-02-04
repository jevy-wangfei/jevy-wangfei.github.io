这是一篇为资深 TypeScript 开发者定制的 **Rust 结构体 (Structs)** 深度解析文章。

在 TypeScript 中，我们每天都在处理 Object、Interface 和 Class。你可能认为 Rust 的 Struct 只是变了名字的 Object，如果这样想，那就 **大错特错。**。

阅读本章及后面两章，需要先清空个人对Struct和Trait的想象，以学习着的姿态，逐渐理解与TS的区别。

在 V8 引擎中，对象是一个巨大的 Hash Map（或者通过 Hidden Class 优化的 lookup table），充满了动态性和元数据。而在 Rust 中，Struct 是**精确的内存布局**。如果 TS 的 Interface 是“协议”，那么 Rust 的 Struct 就是“蓝图”。

---

# 数据的蓝图：Rust 结构体 (Structs) 深度解析

## 1. 定义：名义类型 vs 结构化类型

这是 TS 开发者进入 Rust 的第一个文化冲击。

### TypeScript (Structural Typing / 鸭子类型)

在 TS 中，只要两个接口长得一样，它们就是兼容的。

```typescript
interface User {
  name: string;
  age: number;
}
interface Person {
  name: string;
  age: number;
}

let u: User = { name: "Alice", age: 30 };
let p: Person = u; // 合法，因为形状（Shape）一样
```

### Rust (Nominal Typing / 名义类型)

在 Rust 中，即使字段一模一样，名字不同就是不同的类型。

```rust
struct User {
    name: String,
    age: u8,
}

struct Person {
    name: String,
    age: u8,
}

let u = User { name: String::from("Alice"), age: 30 };
// let p: Person = u; // 编译错误！User 不是 Person

```

**深度视角**：这种严格性是为了保证**内存安全**和**语义清晰**。Rust 编译器不会进行隐式的类型转换，除非你显式实现 `From` 或 `Into` trait。

>

    简单来说，`From` 和 `Into` 是 Rust 中用于**类型转换**的一对“双生” Trait。

    * **`From<T>` (实现端)**：定义“如何从 A 变成 B”。它是**主动**的。
    * **规则**：只要你为 `B` 实现了 `From<A>`，Rust 就会**自动**为 `A` 实现 `Into<B>`。
    * **习惯**：永远优先实现 `From`，因为它的逻辑最清晰。


    * **`Into<T>` (调用端)**：定义“如何把自己变成 B”。它是**被动**的。

---

## 2. 实例化与字段简写：熟悉的配方

Rust 借鉴了现代 JS/TS 的一些语法糖，这让你会感到很亲切。

### 2.1 基础实例化与简写

```rust
fn build_user(email: String, username: String) -> User {
    User {
        email,      // 字段简写 (Field Init Shorthand)，和 TS 一样
        username,
        active: true,
        sign_in_count: 1,
    }
}

```

### 2.2 更新语法 (Struct Update Syntax)

在 TS 中你习惯用 Spread Operator (`...`)。Rust 使用 `..` 语法, 但必须放在最后。

**TypeScript:**

```typescript
const user2 = { ...user1, email: "new@example.com" };
```

**Rust:**

```rust
let user2 = User {
    email: String::from("new@example.com"),
    ..user1 // 必须放在最后
};

```

**注意：所有权陷阱！**
在 TS 中，`...` 是浅拷贝。
在 Rust 中，`..user1` 会发生 **Move (移动)**。
如果 `user1` 中有堆内存字段（如 `String`），它们的所有权会被转移给 `user2`。
如果 `user1` 中有栈内存字段（如 `int32`基础类型），它们会被复制，所有权不会被转移。

- 执行完这行代码后，`user1.username` 已经**失效**了（因为它被 move 给了 user2）。
- 但 `user1.active` (bool) 和 `user1.sign_in_count` (u64) 依然可用（因为它们实现了 Copy trait，是栈上复制）。

---

## 3. 内存布局：元组结构体与单元结构体

Rust 提供了两种特殊的 Struct，对应不同的内存布局需求。

### 3.1 元组结构体 (Tuple Structs) —— "具名元组"

TS 的 Tuple `[number, number]` 只是一个数组。Rust 的 Tuple Struct 是有名字的类型。

**场景**：你需要区分 `Color(0,0,0)` 和 `Point(0,0,0)`。在 TS 中这很难强制区分，在 Rust 中这零成本。

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

let black = Color(0, 0, 0);
let origin = Point(0, 0, 0);
// black == origin // 类型不匹配，安全！

```

**内存视角**：它们在内存中就是紧挨着的三个 `i32`，没有任何字段名的开销。

### 3.2 单元结构体 (Unit-Like Structs) —— "空接口"

```rust
struct AlwaysEqual;

```

这看起来毫无意义？在 TS 中相当于 `interface AlwaysEqual {}`。
**用途**：它不占任何内存（0字节），主要用于**实现 Trait**。比如你需要一个“占位符”来代表某种行为（如 Mock 对象），或者作为标记类型（Marker Type）。

---

## 4. 方法 (Methods)：解构 Class

TS 的 `class` 把数据（props）和行为（methods）绑在一起。Rust 将它们物理分离。

### 4.1 `impl` 块

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

// 行为定义在这里
impl Rectangle {
    // 这是一个方法 (Method)
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

```

### 4.2 `self` 的三种形态 (关键概念)

在 TS 中，`this` 是隐含的。在 Rust 中，`self` 是显式的第一个参数。它决定了调用方法时**所有权**如何变化。

| 参数写法    | 等同的 TS 概念    | Rust 含义                                                            | 常用度                |
| ----------- | ----------------- | -------------------------------------------------------------------- | --------------------- |
| `&self`     | `this` (Readonly) | **借用**。只读访问，不拿走所有权。                                   | ⭐⭐⭐⭐⭐ (90% 情况) |
| `&mut self` | `this`            | **可变借用**。允许修改结构体内部字段。                               | ⭐⭐⭐                |
| `self`      | N/A               | **夺取所有权**。方法调用后，原实例被销毁（消费）。通常用于转换类型。 | ⭐                    |

**代码示例**：

```rust
impl Rectangle {
    // 只读
    fn area(&self) -> u32 { ... }

    // 修改
    fn resize(&mut self, factor: u32) {
        self.width *= factor;
    }

    // 消费 (Destructor 模式)
    fn destroy(self) {
        // self 离开作用域，内存被释放
        println!("Rectangle is gone");
    }
}

let mut rect = Rectangle { width: 10, height: 10 };
rect.resize(2); // OK
rect.destroy(); // OK
// rect.area(); // 错误！rect 已经被 destroy 消费掉了

```

### 4.3 关联函数 (Associated Functions) —— 静态方法

TS 的 `static` 方法，在 Rust 中就是不带 `self` 的函数。

```rust
impl Rectangle {
    // 类似于 static new(size: number)
    fn square(size: u32) -> Rectangle {
        Rectangle { width: size, height: size }
    }
}

// 调用方式：使用 ::
let sq = Rectangle::square(10);

```

---

## 5. 所有的权与生命周期陷阱 (The References Trap)

这是新手必踩的坑。TS 开发者习惯把对象传来传去。

**错误示范**：

```rust
struct User {
    username: &str, // ❌ 试图在结构体里存引用
    email: &str,
}

```

**编译器咆哮**：`missing lifetime specifier`。

**原因解析**：
如果在 Struct 中存引用（`&str`），Rust 必须确保**结构体活得不能比引用的数据久**。否则结构体就会变成“悬垂指针”的容器。为了处理这个，你需要引入生命周期标注 `'a`，这对初学者来说太复杂了。

**Senior TS 建议**：
在入门阶段，结构体字段永远使用**拥有所有权**的类型 (`String`, `Vec<T>`)，而不是切片/引用 (`&str`, `&[T]`)。

```rust
struct User {
    username: String, // 安全，User 拥有这个字符串
    email: String,
}

```

---

## 6. 打印与调试 (Debug Trait)

在 JS 中，你习惯了 `console.log(obj)`。
在 Rust 中，直接 `println!("{}", user)` 会报错，因为 `User` 默认没有实现 `Display` trait（转成字符串的能力， 相当于Java里的toString()）。

**解决方案：派生宏 (Derive Macro)**
Rust 允许你通过注解自动生成代码。

```rust
#[derive(Debug)] // ✨ 魔法：自动实现调试打印功能
struct User {
    name: String,
}

let u = User { name: String::from("Rust") };

// {:?} 是调试格式输出
println!("{:?}", u); // User { name: "Rust" }

// {:#?} 是美化输出 (Pretty Print)
println!("{:#?}", u);
/*
User {
    name: "Rust"
}
*/

```

> 注意：
>
> - println! 里的 ! 表示它是宏。
> - {:#?} 里的 ? 表示调试模式输出。
> - 代码逻辑中的 ? 是错误自动转发。

---

## 7. 内存对齐 (Memory Alignment) —— Senior知识点

在 TS 中，对象属性的顺序不影响内存大小（V8 内部处理）。
在 Rust 中，Struct 的字段顺序**影响内存大小**。这是因为 **Padding (内存填充)**。

CPU 喜欢按字长（如 8 字节）读取内存。

```rust
// 布局优化前
struct Bad {
    a: u8,   // 1 byte
    b: u64,  // 8 bytes (需要前面补 7 bytes padding 才能对齐)
    c: u8,   // 1 byte  (需要后面补 7 bytes padding 凑齐 alignment)
} // 总大小可能是 24 bytes

// 布局优化后
struct Good {
    b: u64, // 8
    a: u8,  // 1
    c: u8,  // 1 (紧跟 a 后面)
    // padding 6 bytes
} // 总大小 16 bytes

```

虽然 Rust 编译器会自动优化字段顺序（Repacking），但在涉及 FFI（调用 C 语言库）时，你需要手动控制布局 (`#[repr(C)]`)。作为高级开发者，意识到“字段也是有物理体积的”至关重要。

---

## 总结

对于 TS 全栈开发者，理解 Rust Struct 的关键在于：

1. **它不是对象**：没有 Hidden Class，没有动态属性添加。它是编译期确定的内存块。
2. **Move 语义无处不在**：使用 `..update` 语法时，小心字段所有权的转移。
3. **分离数据与行为**：`struct` 存数据，`impl` 写逻辑。
4. **`self` 的控制权**：明确区分 `&self` (读)、`&mut self` (写) 和 `self` (消费)。

掌握了 Struct，你就掌握了 Rust 数据建模的基石。通常会结合 **Enum** 和 **Trait**，构建出比 TS 更加类型安全和富有表现力的系统。
