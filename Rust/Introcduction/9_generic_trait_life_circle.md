泛型、trait、生命周期是 Rust 类型系统的三大支柱：泛型实现代码复用，trait 定义共享行为，生命周期保证引用的有效性。它们共同支撑起 Rust 的"零成本抽象"与内存安全。

# 1. generic —— 泛型

★ **关键认知**：泛型让"同一套逻辑适用于多种类型"，且在 Rust 中是**零成本抽象**——编译期通过**单态化（monomorphization）** 为每个具体类型生成一份代码，运行时无任何额外开销（不同于 Java 的类型擦除）。

## 1.1 泛型函数

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

## 1.2 泛型结构体

```rust
struct Point<T> {
    x: T,
    y: T,
}

// 多个类型参数
struct Pair<T, U> {
    first: T,
    second: U,
}
```

## 1.3 泛型枚举

```rust
enum Option<T> { Some(T), None }
enum Result<T, E> { Ok(T), Err(E) }   // 标准库就是泛型枚举
```

## 1.4 泛型 impl

```rust
impl<T> Point<T> {
    fn x(&self) -> &T { &self.x }     // 对所有 T 生效
}

impl Point<f64> {
    fn distance(&self) -> f64 {       // 只对 T=f64 生效
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

★ **关键认知**：可以为"所有 T"或"特定 T"实现方法。注意泛型类型参数在 impl 中必须声明（`impl<T> Point<T>`），否则 `T` 未定义。

## 1.5 单态化

```
单态化（monomorphization）：
编译期为每个用到的具体类型生成一份专用代码。

largest::<i32>  → 生成一份处理 i32 的机器码
largest::<char> → 生成一份处理 char 的机器码

代价：二进制可能变大（代码膨胀）
收益：运行时零开销
```

★ **关键认知**：这是 Rust 泛型与 Java 泛型的根本差异——Java 擦除类型、运行时装箱拆箱有开销；Rust 编译期展开、运行时和手写具体类型一样快。

# 2. trait and trait bounds

## 2.1 定义 trait

```rust
2.1 定义：
[pub] trait <trait_name> {
    fn <method_name>(&self) -> <type>;                       // 只有签名
    fn <method_name>(&self) -> <type> { ... }                // 提供默认实现
}
```

```rust
pub trait Summary {
    fn summarize(&self) -> String;

    fn summarize_author(&self) -> String {
        String::from("(anonymous)")
    }
}
```

★ **关键认知**：trait 定义"共享行为的契约"，类似其他语言的接口（interface）。它**可以有默认方法实现**，实现者只需覆盖关心的部分。

## 2.2 实现 trait

```rust
impl <trait_name> for <type> {
    fn <method_name>(&self) -> <type> { ... }
}
```

```rust
struct NewsArticle { headline: String }

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}", self.headline)
    }
}
```

## 2.3 注意事项

```
2.3.1 默认实现的方法可以调用 trait 中的其他方法，即使这些方法没有默认实现
2.3.2 无法从方法的重写里调用默认实现（需要完全替换，或用完全限定语法）
```

## 2.4 trait 作参数

```rust
// impl Trait 语法（简单，适合单一 trait）
fn notify(item: &impl Summary) {
    println!("{}", item.summarize());
}

// trait bound 等价写法
fn notify<T: Summary>(item: &T) {
    println!("{}", item.summarize());
}
```

★ **关键认知**：`impl Trait` 与 trait bound 等价。`impl Trait` 更简洁（适合参数简单的情况），trait bound 更灵活（可命名、可复用、可加多个约束）。两者都在编译期单态化。

## 2.5 trait bound

```rust
// 多个约束
fn notify<T: Summary + Display>(item: &T) { }

// impl Trait 写法
fn notify(item: &(impl Summary + Display)) { }
```

## 2.6 使用 + 指定多个 trait

```rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: T, u: U) { }
```

## 2.7 使用 where 子句

```rust
// where 子句：约束多了之后更清晰
fn some_function<T, U>(t: T, u: U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
    0
}
```

## 2.8 trait 作返回类型

```rust
// impl Trait 作为返回类型
fn returns_summarizable() -> impl Summary {
    NewsArticle { headline: String::from("x") }
}
```

★ **关键认知**：`-> impl Trait` 只能返回**确定的同一种类型**，返回可能不同类型的代码会报错（即使它们都实现了该 trait）。因为编译器需要知道统一的具体类型来做单态化。需要返回多种类型时用 `Box<dyn Trait>`。

# 3. life circle —— 生命周期

★ **关键认知**：生命周期（lifetime）是**编译期的引用有效性检查**，确保引用不会悬垂。它**不改变运行时代码**，只是编译器的静态分析。多数情况可省略（生命周期省略规则）。

## 3.1 生命周期标注

```rust
// 'a 是生命周期参数，表示"返回的引用活得和输入一样久"
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

★ **关键认知**：生命周期标注**不改变引用的实际存活时间**，只是告诉编译器"这些引用之间的生命周期约束关系"。`'a` 的实际含义是"**取所有被标注引用中最短的那个**"。

## 3.2 取短的

```
3.1 取短的：'a 具体等于多个输入生命周期中"最短"的那个
    函数返回的引用的有效范围，受限于最短的输入生命周期
```

## 3.3 省略规则

★ **关键认知**：编译器有三条**生命周期省略规则**，满足则无需手写标注：

```
3.2.1 每个引用类型的参数都有自己的生命周期参数
      fn foo(x: &str, y: &str)  →  fn foo<'a, 'b>(x: &'a str, y: &'b str)
3.2.2 如果只有一个输入生命周期参数，那么它被赋给所有输出生命周期参数
      fn foo(x: &str) -> &str   →  fn foo<'a>(x: &'a str) -> &'a str
3.2.3 如果有多个输入生命周期，但其中一个是 &self 或 &mut self（方法），
      那么 self 的生命周期被赋给所有输出生命周期参数
```

## 3.4 结构体中的生命周期

```rust
struct Excerpt<'a> {
    part: &'a str,       // 结构体持有引用，必须标注生命周期
}
```

★ **关键认知**：只要结构体**持有引用字段**，就必须标注生命周期，保证结构体实例不会比它引用的数据活得更久。

## 3.5 静态生命周期

```rust
let s: &'static str = "hello";   // 'static：整个程序运行期都有效
```

★ **关键认知**：`'static` 表示"引用在整个程序生命周期内有效"。字符串字面量都是 `'static`（存在二进制只读段）。⚠ 慎用 `'static` 约束，它会强迫调用方提供永久有效的引用。

# 4. 小结

| 概念 | 要点 |
|------|------|
| 泛型 | 编译期单态化，零成本抽象 |
| trait | 共享行为契约，可有默认实现 |
| impl Trait | 作参数/返回类型的语法糖 |
| where | 复杂约束的清晰写法 |
| 生命周期 | 编译期引用有效性检查 |
| 省略规则 | 三条规则，满足则免标注 |
| 'static | 程序全程有效 |

下一篇进入测试，讲清楚 Rust 的单元测试、集成测试、断言宏、`#[should_panic]`、`#[ignore]`，以及测试组织方式。
