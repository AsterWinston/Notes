结构体（struct）是 Rust 自定义复合类型的核心，配合 `impl` 块定义方法和关联函数。Rust 没有类，用"结构体 + impl + trait"组合实现面向对象式的封装与抽象。

# 1. struct —— 结构体定义

## 1.1 定义格式

```rust
1.1 格式：
struct <struct_name> {
    <field_name>: <type>,
    [<field_name>: <type>, ...]
}
```

```rust
struct User {
    name: String,
    age: u32,
    active: bool,
}
```

## 1.2 创建实例

```rust
1.2 使用结构体：
let [mut] <var_name> = <struct_name> {
    <field_name>: <value>,
    [<field_name>: <value>, ...]
};
```

```rust
let user = User {
    name: String::from("张三"),
    age: 18,
    active: true,
};

// 可变实例（整个实例可变，不能单字段标 mut）
let mut user = User { name: String::from("张三"), age: 18, active: true };
user.age = 19;
```

★ **关键认知**：Rust 的字段可变性**以实例为单位**——要么整个实例 `mut`（所有字段可改），要么完全不可变。没有"某字段可变、某字段不可变"的粒度（与 C# 的 readonly 字段不同）。

## 1.3 访问字段

```rust
1.3 访问字段：<var_name>.<field_name>
```

## 1.4 字段初始化简写

```rust
1.4 当变量名与字段名相同时，可简写：
let age = 18;
let stu = User {
    name: String::from("张三"),
    age,                    // 等价于 age: age
    active: true,
};
```

## 1.5 结构体更新语法

```rust
1.5 更新语法：
let user2 = User {
    name: String::from("李四"),
    ..user1                 // 其余字段取自 user1
};
```

★ **关键认知**：`..user1` 会**移动**未显式赋值的字段（若字段是堆类型如 String）。上例中 user2 用了 user1 的 age/active（Copy），但如果 user1 的 name 也被 `..` 复用，则 user1 会部分失效。这是 move 语义在结构体上的体现。

## 1.6 tuple struct —— 元组结构体

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32);

let black = Color(0, 0, 0);
let p = Point(1, 2);
// 通过 .0 .1 访问：black.0
```

★ **关键认知**：元组结构体有字段但**没有字段名**，适合"轻量包装"（如用结构体区分不同的 ID 类型，避免混用）。即使字段类型相同，不同元组结构体也是不同类型。

## 1.7 Unit-Like Struct —— 类单元结构体

```rust
struct AlwaysEqual;    // 没有任何字段

let subject = AlwaysEqual;
```

★ **关键认知**：**类单元结构体**没有字段，常用于实现某个 trait 但不需要存储数据（如标记类型、泛型占位）。它的名字类似单元类型 `()`。

# 2. trait —— 共享行为

## 2.1 格式化 trait：Display 与 Debug

```
2.1 std::fmt::Display     // 面向用户的格式化，用 {} 打印
2.2 std::fmt::Debug       // 面向调试的格式化，用 {:?} 打印
2.3 #[derive(Debug)]      // 自动为结构体实现 Debug
2.4 {:?}                  // Debug 格式化（单行）
2.5 {:#?}                 // 美化 Debug 格式化（多行缩进）
```

```rust
#[derive(Debug)]
struct Point { x: i32, y: i32 }

let p = Point { x: 1, y: 2 };
println!("{:?}", p);      // Point { x: 1, y: 2 }
println!("{:#?}", p);     // 多行美化
```

★ **关键认知**：`Debug` 可以通过 `#[derive(Debug)]` **自动派生**（编译器生成实现），而 `Display` **不能自动派生**——因为"面向用户的展示格式"因类型而异，必须手写。这与 Go 的 `String()` 方法类似，但 Rust 的 derive 机制更系统化。

## 2.2 手动实现 Display

```rust
use std::fmt;

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
```

## 2.3 derive 可派生的常用 trait

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Default)]
struct Point { x: i32, y: i32 }
```

★ **关键认知**：`derive` 是 Rust 的**过程宏**，编译期自动生成 trait 实现。常用可派生的：`Debug`（调试打印）、`Clone`（显式拷贝）、`Copy`（按位复制）、`PartialEq`/`Eq`（相等比较）、`Hash`（可作 map 键）、`Default`（默认值）。

# 3. method —— 方法

## 3.1 定义格式

```rust
3.1 格式：
impl <struct_name> {
    fn <method_name>([&][mut ]self[, <param>: <type>, ...]) [-> <ret_type>] {
        <method_body>
    }
}
```

```rust
impl Point {
    fn distance(&self) -> f64 {          // &self：不可变借用
        ((self.x.pow(2) + self.y.pow(2)) as f64).sqrt()
    }

    fn shift(&mut self, dx: i32) {       // &mut self：可变借用
        self.x += dx;
    }

    fn consume(self) -> i32 {            // self：获取所有权
        self.x
    }
}
```

## 3.2 self 的三种形式

```
&self       不可变借用（只读，最常用）
&mut self   可变借用（可修改字段）
self        获取所有权（消耗实例，方法调用后实例失效）
```

★ **关键认知**：方法的第一个参数总是 `self` 的某种形式。选择哪种取决于方法是否需要修改/消耗实例——这与 Go 的值/指针接收者类似，但 Rust 更细（多了"消耗所有权"这一档）。

## 3.3 调用方法

```rust
3.2 调用：<var_name>.<method_name>(<args>)
```

★ **关键认知**：Rust 用 `obj.method()` 调用方法（点号），会自动处理引用（autoref）——即使方法签名是 `&self`，调用 `p.distance()` 也会自动加 `&`。这与 Go 自动解引用指针类似。

# 4. associated function —— 关联函数

## 4.1 定义与调用

```rust
4.1 格式（不接收 self）：
impl <struct_name> {
    fn <func_name>(<params>) [-> <ret_type>] {
        <func_body>
    }
}
```

```rust
impl Point {
    // 关联函数：没有 self 参数
    fn new(x: i32, y: i32) -> Point {
        Point { x, y }
    }

    fn origin() -> Point {
        Point { x: 0, y: 0 }
    }
}

let p = Point::new(1, 2);      // 用 :: 调用
let o = Point::origin();
```

★ **关键认知**：关联函数不接收 `self`，通过 `Type::function()` 调用，常用作**构造函数**（Rust 没有内置构造函数，惯例是用 `new`）。这与 Go 的包级函数不同——Rust 把相关的"工厂函数"组织在 `impl` 块内，形成命名空间。

# 5. impl block —— impl 块

```
5.1 每个结构体可以有多个 impl 块（Rust 允许分开书写）
5.2 可以为类型实现方法（impl Type）或实现 trait（impl Trait for Type）
```

```rust
// 多个 impl 块是合法的
impl Point {
    fn x(&self) -> i32 { self.x }
}

impl Point {
    fn y(&self) -> i32 { self.y }
}
```

★ **关键认知**：Rust 允许把方法分散到多个 `impl` 块（常见于泛型约束的场景）。但注意 **orphan rule（孤儿规则）**：只能为"本地类型"实现"本地 trait"，或为"本地类型"实现"外部 trait"——不能为外部类型实现外部 trait。这保证了 trait 实现的一致性和防止冲突。

# 6. 小结

| 概念 | 要点 |
|------|------|
| struct | 命名字段的自定义类型 |
| 实例可变性 | 以实例为单位，整体 mut |
| 更新语法 | `..other` 移动剩余字段 |
| tuple struct | 有字段无字段名 |
| unit struct | 无字段，作标记/占位 |
| Debug/Display | Debug 可 derive，Display 需手写 |
| 方法 | &self / &mut self / self |
| 关联函数 | 无 self，`Type::func()` |
| impl | 可多个块；orphan rule 限制 |

下一篇进入枚举（enum），讲清楚 Rust 强大的枚举（可携带数据）、`Option<T>`、`match` 模式匹配与 `if let`。
