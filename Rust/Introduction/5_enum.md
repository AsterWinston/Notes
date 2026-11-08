Rust 的枚举（enum）远比 C 的枚举强大——它的每个变体**可以携带数据**，配合 `match` 模式匹配，能优雅表达"多种可能状态"。`Option<T>` 和 `Result<T, E>` 都是标准库用枚举实现的，是 Rust 错误处理与空值安全的基石。

# 1. enum —— 枚举定义

## 1.1 定义格式

```rust
1.1 格式：
enum <enum_name> {
    <variant1>,
    [<variant2>, ...]
}
```

```rust
enum IpAddrKind {
    V4,
    V6,
}
```

## 1.2 使用枚举

```rust
1.2 使用：<enum_name>::<variant>
```

```rust
let four = IpAddrKind::V4;
let six = IpAddrKind::V6;
```

## 1.3 变体携带数据

★ **关键认知**：Rust 枚举的变体**可以携带数据**（这是它与 C 枚举最大的不同），每个变体可携带不同类型/数量的数据，相当于"标签联合（tagged union）"：

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
    V4Struct { a: u8, b: u8 },
}

let home = IpAddr::V4(127, 0, 0, 1);
let loopback = IpAddr::V6(String::from("::1"));

// 解构
if let IpAddr::V4(a, b, c, d) = home {
    println!("{}.{}.{}.{}", a, b, c, d);
}

// 用 match 解构
match home {
    IpAddr::V4(a, b, c, d) => println!("ipv4"),
    IpAddr::V6(s) => println!("ipv6"),
    IpAddr::V4Struct { a, .. } => println!("{}", a),
}
```

★ **关键认知**：`match` 必须**穷尽**枚举的所有变体（或加 `_` 兜底），否则编译错误。这保证了"新增枚举变体时，所有使用处都必须处理"，从编译期消除遗漏。

## 1.4 impl block

```rust
1.4 为枚举实现方法：
impl <enum_name> {
    fn <method_name>(&self) { <body> }
}
```

```rust
impl IpAddr {
    fn is_loopback(&self) -> bool {
        matches!(self, IpAddr::V4(127, 0, 0, 1) | IpAddr::V6(_))
    }
}
```

★ **关键认知**：枚举可以像结构体一样用 `impl` 块添加方法。`matches!` 宏（`matches!(expr, pattern)`）返回 bool，是 `match` 的简洁写法。

# 2. Option<T>

## 2.1 定义

```rust
// 标准库定义（简化）
enum Option<T> {
    Some(T),
    None,
}
```

★ **关键认知**：`Option<T>` 用枚举表达"**可能有值，也可能没有**"，它是 Rust 处理"空值"的方式。与大多数语言的 `null` 不同，Rust **没有 null**——空值被显式编码为 `None`，编译器强制你处理。这被称为"**十亿美元的错误**"（Tony Hoare 对 null 的反思）的解决方案。

## 2.2 使用 Option

```rust
let some_number: Option<i32> = Some(5);
let absent_number: Option<i32> = None;

// ★ 不能直接把 Option<i32> 当 i32 用
let x: i32 = some_number;      // 编译错误：expected i32, found Option<i32>
```

★ **关键认知**：`Option<T>` 与 `T` 是**不同类型**，编译器不允许把 `Option<i32>` 直接当 `i32` 用。必须通过 `match`/`if let`/`unwrap` 等方式"取出"值——这就强制了空值检查，从类型层面杜绝了空指针解引用。

## 2.3 常用方法

```rust
opt.unwrap()               // 取值，None 则 panic（慎用）
opt.unwrap_or(0)           // 取值，None 则返回默认值
opt.unwrap_or_else(|| ...) // 惰性默认值
opt.map(|x| x + 1)         // 映射内部值
opt.and_then(...)          // 链式操作
opt.is_some() / is_none()  // 判断
```

★ **关键认知**：优先使用 `unwrap_or`、`map`、`?` 等**安全**方法，避免 `unwrap()`（None 时 panic）。生产代码中 `unwrap` 应仅用于"逻辑上不可能为 None"或测试场景。

# 3. match —— 模式匹配

## 3.1 基本格式

```rust
3.1 格式：
match <expr> {
    <pattern1> => { ... },
    <pattern2> => { ... },
    ...
}
// 单个表达式时可省略 {}
```

```rust
match coin {
    Coin::Penny => 1,
    Coin::Nickel => 5,
    Coin::Dime => 10,
    Coin::Quarter => 25,
}
```

## 3.2 通配符与占位符

```rust
3.2 通配符 '_'：
match <expr> {
    _ => { ... },        // 匹配剩余所有情况
}

match x {
    1 => "one",
    2 => "two",
    _ => "other",        // 必须兜底
}
```

★ **关键认知**：`_` 是通配模式（匹配任意值、不绑定），`..` 是"忽略剩余字段"的语法（如 `Point { x, .. }`）。`match` 是表达式，可作为值使用，各分支返回类型必须一致。

## 3.3 高级模式

```rust
match value {
    0 => ...,
    1 | 2 => ...,              // 或模式
    3..=9 => ...,              // 范围模式
    n if n < 0 => ...,         // 匹配守卫
    Some(x) @ 1..=5 => ...,    // 绑定 + 子模式
    (a, b) => ...,             // 解构元组
    [first, .., last] => ...,  // 解构切片
    Point { x, y } => ...,     // 解构结构体
}
```

# 4. if let [else]

## 4.1 简化单分支匹配

```rust
// 用 match 写：
match config {
    Some(value) => println!("{}", value),
    None => (),
}

// 用 if let 简化：
if let Some(value) = config {
    println!("{}", value);
}
```

★ **关键认知**：`if let` 是"**只关心一个模式**"时的简洁写法，等价于只有两个分支的 match。`while let` 则是"当模式匹配时持续循环"（常用于消费迭代器/栈）。

## 4.2 if let else

```rust
if let Some(value) = config {
    println!("{}", value);
} else {
    println!("none");
}
```

## 4.3 while let

```rust
let mut stack = vec![1, 2, 3];
while let Some(top) = stack.pop() {
    println!("{}", top);   // 3, 2, 1
}
```

# 5. 小结

| 概念 | 要点 |
|------|------|
| enum | 变体可携带数据（标签联合） |
| match | 必须穷尽，可解构/守卫/范围 |
| Option<T> | Some/None，替代 null |
| if let | 单模式匹配的简写 |
| while let | 模式匹配循环 |
| matches! | 返回 bool 的 match 宏 |

下一篇进入包、crate 与模块系统，讲清楚 Rust 的代码组织：package、crate、module、`use`/`pub`/`mod`，以及可见性控制。
