Rust 的变量默认**不可变**，类型系统严格且支持强大的类型推断。本章从变量绑定讲起，覆盖标量/复合类型、函数、以及 `if`/`match`/循环这些控制流表达式。

# 1. let —— 变量绑定

## 1.1 可变与不可变绑定

```rust
let mut x = 5;   // 1.1 可变绑定
x = 6;           // 合法：mut 允许重新赋值

let y = 5;       // 1.2 不可变绑定
y = 6;           // 编译错误：cannot assign twice to immutable variable
```

★ **关键认知**：Rust 变量**默认不可变**（immutable），这是语言的核心安全设计之一。要修改必须显式写 `mut`。这与大多数语言"默认可变"相反——强制开发者显式表达"这个值会变"，让并发安全和逻辑推理更容易。

## 1.2 类型推断与显式标注

```rust
let x = 5;              // 推断为 i32（默认整型）
let y: u64 = 5;         // 显式标注类型
let z: f64 = 3.14;      // 浮点显式标注
```

★ **关键认知**：Rust 编译器有强大的类型推断（基于 Hindley-Milner 算法），能自动推断出变量类型，多数情况无需标注。但推断依赖于**使用上下文**——如果一个变量从未被使用到需要确定类型的地方，就必须显式标注。

# 2. const —— 常量

```
2.1 必须标注数据类型
2.2 命名规范：全大写字母，单词用 '_' 分隔
2.3 不能用 let 重新赋值（无法被遮蔽）
2.4 值必须是编译期常量表达式
```

```rust
const MAX_POINTS: u32 = 100_000;
const PI: f64 = 3.14159;
```

★ **关键认知**：`const` 与 `let` 有本质区别——`const` 在编译期求值并**内联到使用处**，没有运行时内存分配；`let` 是运行时绑定。`const` 可以定义在全局作用域，且其值必须是编译期可计算的常量表达式。

# 3. shadowing —— 变量遮蔽

```rust
let x = 1;
// x = 2;        // 错误：不能重新赋值
let x = 2;       // 合法：用新的 let 遮蔽（shadow）旧变量

let spaces = "   ";
let spaces = spaces.len();   // ★ 甚至可以改变类型！String(str) → usize
```

★ **关键认知**：**遮蔽（shadowing）** 允许用同名 `let` 声明一个新变量，旧变量被"遮蔽"（仍存在但不可访问）。它与 `mut` 的区别是：遮蔽会创建**新变量**，因此可以**改变类型**（上例从 `&str` 变成 `usize`），而 `mut` 不能改变类型。这是 Rust 里非常实用的惯用法。

# 4. data type —— 数据类型

## 4.1 类型标注

```
4.1 type annotation：Rust 是静态类型语言，每个值都有编译期确定的类型
```

## 4.2 scalar type（标量类型）

### 4.2.1 integer（整型）

```
4.2.1.1 变量形式：<i|u><size>，size 可为 8/16/32/64/128
        另有 isize / usize，长度取决于机器字长（64 位系统上为 64 位）
        - i<size> 范围：-2^(n-1) ~ 2^(n-1)-1
        - u<size> 范围：0 ~ 2^n-1
4.2.1.2 字面量：除 byte 外都可用类型后缀，如 123u16（值为 123，类型 u16）
        整型字面量默认类型为 i32
4.2.1.2.1 十进制：789_123（可用 '_' 分隔，便于阅读）
4.2.1.2.2 十六进制：0xff
4.2.1.2.3 八进制：0o123
4.2.1.2.4 二进制：0b1010_0101
4.2.1.2.5 字节：b'A'（仅 u8）
```

```rust
let a = 98_222;       // 十进制
let b = 0xff;         // 十六进制
let c = 0o77;         // 八进制
let d = 0b1111_0000;  // 二进制
let e = b'A';         // 字节字面量（u8），值为 65
```

★ **关键认知**：整数溢出在 **debug 模式 panic**、**release 模式回绕（wrapping）**。需要显式控制溢出行为时，用标准库方法：`wrapping_add`（回绕）、`checked_add`（溢出返回 None）、`saturating_add`（饱和到边界）、`overflowing_add`（返回溢出标志）。

### 4.2.2 float（浮点型）

```
4.2.2.1 f32：单精度（4 字节）
4.2.2.2 f64：双精度（8 字节），浮点字面量默认类型
```

```rust
let x = 3.0;        // f64
let y: f32 = 3.0;   // f32
let z = 3.0_f64;
```

★ **关键认知**：Rust 浮点遵循 **IEEE-754** 标准，`f32`/`f64` 都有 NaN、±infinity。浮点比较不能用 `==` 做精确比较（0.1+0.2 != 0.3），需用误差范围或 `approx` 库。

### 4.2.3 bool（布尔型）

```
4.2.3.1 true
4.2.3.2 false
```

★ **关键认知**：Rust 的 `bool` 与整数**严格分离**，不存在隐式转换。`if 1 {}` 编译错误，必须 `if true {}`。

### 4.2.4 char（字符型）

```
占 4 字节，是 Unicode 标量值（Unicode Scalar Value）
可表示英文字母、中日韩字符、零宽空白、emoji 等
用单引号：'a'、'我'、'😊'
```

★ **关键认知**：Rust 的 `char` 是 **4 字节**的 Unicode 标量值（U+0000 ~ U+D7FF 及 U+E000 ~ U+10FFFF），与 C 的 1 字节 `char` 完全不同。字符串 `&str` 是 UTF-8 编码的字节序列，所以"字符数"与"字节数"不等——详见字符串与集合章节。

## 4.3 composite type（复合类型）

### 4.3.1 Tuple（元组）

```rust
let tup: (i32, f64, u8) = (500, 6.4, 1);   // 创建（可标注类型）
let (x, y, z) = tup;                       // 解构（pattern matching）
let first = tup.0;                         // 用 .index 访问
```

### 4.3.2 Array（数组）

```rust
let a: [i32; 5] = [1, 2, 3, 4, 5];   // 定长数组
let b = [3; 5];                       // [3, 3, 3, 3, 3]，语法糖

let first = a[0];                     // 用下标访问
```

★ **关键认知**：数组**存储在栈上**，长度固定且是类型的一部分（`[i32; 5]` 和 `[i32; 6]` 是不同类型）。越界访问在**编译期能查出常量下标**，运行期下标会 **panic**（带边界检查）。需要可变长度的场景用 `Vec<T>`（堆分配，见集合章节）。

# 5. function —— 函数

## 5.1 定义与命名

```
5.1 命名规范：全小写字母，单词用 '_' 分隔（snake_case）
5.2 格式：
fn <func_name>([<param_name>: <type>, ...]) [-> <ret_val_type>] {
    <func_body>
    [<expr>]        // 最后一个表达式作为返回值，不加分号
}
```

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b           // 无分号，作为返回值（表达式）
}

fn greet(name: &str) {
    println!("Hello, {}", name);   // 无返回值，返回 ()
}
```

★ **关键认知**：Rust 是**表达式语言**——函数体最后一个**不带分号**的表达式就是返回值；加分号会变成语句，返回 `()`。这是 Rust 与 C 系语言最大的语法差异之一，也导致了"少写一个分号却报类型错误"的常见困惑。

## 5.2 语句 vs 表达式

```rust
let x = 5;          // 语句（statement），无返回值
let y = {
    let z = 3;
    z + 1           // 表达式（expression），块的值 = 4
};
```

★ **关键认知**：**语句**执行动作、无返回值；**表达式**求值、有返回值。Rust 中几乎所有东西都是表达式（`if`、`match`、块 `{}` 都能返回值），这让代码可以写得极简且函数式。注意：`let` 是语句，不能写 `let x = (let y = 5);`。

## 5.3 参数与返回值

```rust
fn five() -> i32 { 5 }              // 无参数，返回 5
fn max(a: i32, b: i32) -> i32 {     // 多参数
    if a > b { a } else { b }       // if 是表达式，直接返回
}
```

# 6. comment —— 注释

```
6.1 普通注释：// 单行，/* 多行 */
6.2 文档注释：/// 为下一项生成文档，//! 为所在项（如 crate/module）生成文档
6.3 cargo doc 可据文档注释生成 HTML 文档
```

★ **关键认知**：`///` 文档注释支持 **Markdown**，且其中的代码块会被 **`cargo test` 当作文档测试（doctest）执行**——文档即测试，保证示例不会过时。这是 Rust 工程质量的一大特色。

# 7. if expression —— if 表达式

## 7.1 基本形式

```rust
if x > 0 {
    println!("positive");
} else if x < 0 {
    println!("negative");
} else {
    println!("zero");
}
```

★ **关键认知**：`if` 的条件**必须是 `bool`**，不接受"非 0 即真"。且 `if` 是**表达式**，可以直接赋值：`let n = if cond { 1 } else { 2 };`——但两个分支的类型必须相同。

## 7.2 if let

```rust
if let Some(x) = opt {
    println!("{}", x);      // 只关心 Some 分支
} else {
    println!("none");
}
```

# 8. match

## 8.1 基本形式

```rust
match x {
    1 => println!("one"),
    2 | 3 => println!("two or three"),   // 多模式
    4..=6 => println!("4 to 6"),         // 范围模式
    _ => println!("other"),              // 通配
}
```

★ **关键认知**：`match` **必须穷尽（exhaustive）**所有可能值，否则编译错误。这强制你处理所有情况（用 `_` 兜底或列全），从编译期杜绝了"漏判分支"的 bug。`match` 也是表达式，常用于解构枚举和 `Option`/`Result`。

## 8.2 match 的绑定与守卫

```rust
match value {
    Some(x) if x > 10 => println!("big {}", x),   // 匹配守卫
    Some(x) => println!("{}", x),                 // 绑定变量
    None => println!("none"),
}
```

# 9. circulate —— 循环

## 9.1 loop

```rust
let mut count = 0;
let result = loop {          // loop 是表达式，可带 break 返回值
    count += 1;
    if count == 10 {
        break count * 2;     // 返回值 20
    }
};
```

★ **关键认知**：`loop` 是无限循环，也可是表达式——`break value` 可以把值作为 `loop` 的结果返回。这是 Rust 独有的能力，`while`/`for` 都不能带值 break。

## 9.2 while

```rust
let mut n = 3;
while n != 0 {
    println!("{}", n);
    n -= 1;
}
```

## 9.3 for 与 Range

```rust
for i in 0..5 {          // 0,1,2,3,4（左闭右开）
    println!("{}", i);
}
for i in 0..=5 {         // 0..=5（闭区间）
    println!("{}", i);
}
for i in (1..4).rev() {  // 反向：3,2,1
    println!("{}", i);
}
for x in &arr {          // 遍历数组（借用）
    println!("{}", x);
}
```

★ **关键认知**：`for` 是 Rust 最常用的循环，它遍历**迭代器（Iterator）**。`0..5` 是 `Range`，`.rev()` 反转。`for` 比 `while` 更安全（不会因手写索引越界），性能也更好（编译器能优化掉边界检查）。

## 9.4 循环标签

```rust
'outer: for i in 0..3 {
    for j in 0..3 {
        if i * j > 2 {
            break 'outer;    // 跳出外层循环
        }
        continue 'outer;     // 跳到外层下一次迭代
    }
}
```

# 10. 小结

| 概念 | 要点 |
|------|------|
| let | 默认不可变，`mut` 才可变 |
| const | 编译期常量，必须标注类型 |
| shadowing | 新 `let` 遮蔽旧变量，可改类型 |
| 标量 | 整数/浮点/布尔/char（4 字节） |
| 复合 | 元组（异构）、数组（定长同构） |
| 函数 | 表达式语言，末表达式即返回值 |
| if | 条件必须 bool，是表达式 |
| match | 必须穷尽，可解构绑定 |
| 循环 | loop（可 break 带值）/while/for |

下一篇进入 Rust 的核心——所有权（ownership），讲清楚栈与堆、move 语义、Copy/Drop trait、借用与引用规则、以及切片，这是 Rust 内存安全的基石。
