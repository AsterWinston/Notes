`unsafe` 和宏是 Rust 的两个"元能力"：`unsafe` 打开编译器的安全闸门，让开发者承担原本由编译器保证的责任；宏则支持编译期代码生成，扩展语言语法。二者都是编写底层库和框架的利器，也都需要谨慎使用。

# 1. unsafe

## 1.1 unsafe 的定位

★ **关键认知**：`unsafe` **不是"关闭安全检查"**，而是解锁**五项编译器无法验证的操作**。在 `unsafe` 块中，所有其他安全检查（类型检查、借用检查）**照常进行**：

```
unsafe 解锁的五项能力：
1. 解引用裸指针（*const T / *mut T）
2. 调用 unsafe 函数或方法
3. 访问/修改可变静态变量
4. 实现 unsafe trait（如 Send / Sync）
5. 访问 union 的字段
```

★ **关键认知**：`unsafe` 的哲学是"**把责任从编译器转移给程序员**"。编译器无法自动验证某些底层操作的安全性，于是要求你显式声明"我来保证"，并承担正确性的责任。

## 1.2 裸指针

```rust
let mut num = 5;

let r1 = &num as *const i32;      // 不可变裸指针
let r2 = &mut num as *mut i32;    // 可变裸指针

unsafe {
    println!("{}", *r1);          // 解引用裸指针必须在 unsafe 中
    *r2 = 6;
}
```

★ **关键认知**：裸指针 `*const T`/`*mut T` 可以绕过借用规则（可同时有可变/不可变、可为空、可悬垂），但没有自动内存管理。解引用裸指针必须在 `unsafe` 块中，由程序员保证指针有效。

## 1.3 unsafe 函数

```rust
unsafe fn dangerous() {
    // 调用者必须保证满足前置条件
}

unsafe {
    dangerous();        // 调用 unsafe 函数需要 unsafe 块
}
```

## 1.4 可变静态变量

```rust
static mut COUNTER: u32 = 0;

fn add_to_count(inc: u32) {
    unsafe {
        COUNTER += inc;      // 访问可变静态变量需要 unsafe
    }
}
```

★ **关键认知**：可变静态变量（`static mut`）是全局可变状态，天然有数据竞争风险，因此访问必须在 `unsafe` 中。Rust 推荐用 `Mutex`/原子类型替代可变静态。

## 1.5 unsafe trait

```rust
unsafe trait MyUnsafeTrait { }
unsafe impl MyUnsafeTrait for MyType { }
```

★ **关键认知**：`Send`/`Sync` 就是 unsafe trait——实现它们（即使自动 derive）意味着"向编译器承诺该类型可安全跨线程"，而这个承诺的正确性编译器无法验证。

## 1.6 安全抽象（最佳实践）

```rust
// 正确做法：把 unsafe 封装在安全的 API 后面
pub fn split_at_mut<T>(slice: &mut [T], mid: usize) -> (&mut [T], &mut [T]) {
    let len = slice.len();
    let ptr = slice.as_mut_ptr();

    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}
```

★ **关键认知**：**unsafe 的最佳实践是"安全抽象"**——在底层用 unsafe 实现，但对外暴露**安全的 API**（调用者无需 unsafe）。标准库大量采用这种方式（如 `split_at_mut` 需要 unsafe 内部实现，但对外安全）。原则：**unsafe 的范围越小越好，且必须用注释说明为何安全**。

# 2. 宏

## 2.1 宏的分类

★ **关键认知**：Rust 的宏分两大类：

```
1. 声明宏（Declarative Macro）：macro_rules!，基于模式匹配
2. 过程宏（Procedural Macro）：
   - 派生宏（derive）：#[derive(MyMacro)]
   - 属性宏（attribute）：#[my_attr]
   - 函数宏（function-like）：my_macro!()
```

★ **关键认知**：宏与函数的本质区别——宏是**编译期的代码生成**，可以接受**可变数量、可变类型**的参数，可以生成代码；函数则受固定签名约束。宏的调用用 `!`（如 `println!`）。

## 2.2 声明宏 macro_rules!

```rust
// 定义
macro_rules! say_hello {
    () => {
        println!("Hello!");
    };
}

say_hello!();     // 展开为 println!("Hello!")

// 带参数的声明宏
macro_rules! create_function {
    ($func_name:ident) => {
        fn $func_name() {
            println!("{:?}() called", stringify!($func_name));
        }
    };
}

create_function!(foo);   // 生成 fn foo() { ... }
foo();
```

★ **关键认知**：`macro_rules!` 通过**模式匹配**工作——`($func_name:ident)` 匹配一个标识符，`$func_name` 在展开时代入。`ident` 是片段类型（fragment），类似其他语言的宏变量。声明宏是 Rust 里最常用的宏形式。

## 2.3 常用片段类型

```
$name:ident     标识符
$name:expr      表达式
$name:ty        类型
$name:stmt      语句
$name:block     代码块
$name:pat       模式
$name:tt        单个语法树节点（最灵活）
$name:item      项（函数/结构体等）
```

## 2.4 重复匹配

```rust
macro_rules! my_vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp = Vec::new();
            $( temp.push($x); )*      // $( ... )* 表示重复
            temp
        }
    };
}

let v = my_vec![1, 2, 3];
```

★ **关键认知**：`$( ... ),*` 是**重复匹配**语法——匹配零个或多个逗号分隔的表达式，`$( ... )*` 在展开时同步重复。这是 `vec!` 宏的实现原理。

## 2.5 过程宏

```rust
// 派生宏（最常见）
#[derive(Debug, Clone, PartialEq)]
struct Point { x: i32, y: i32 }

// 属性宏
#[route(GET, "/")]
fn index() { }

// 函数宏
let sql = sql!(SELECT * FROM posts);
```

★ **关键认知**：过程宏是**在独立 crate 中定义**的函数（用 `proc-macro = true`），接收 `TokenStream` 输入、返回 `TokenStream` 输出，编译期运行。它们能实现 `derive`、自定义属性、函数式宏，是 `serde`、`tokio` 等生态的基石。过程宏必须写在独立的 `proc-macro` crate 中。

# 3. 何时使用 unsafe 与宏

★ **关键认知**：

```
使用 unsafe：
  - 与 C 库交互（FFI）
  - 实现底层数据结构（跳过冗余检查）
  - 性能关键且能证明安全的场景
  - 原则：能不用就不用，用时封装成安全 API

使用宏：
  - 需要可变参数（println!）
  - 编译期代码生成（derive）
  - 减少重复样板代码
  - 原则：能用函数/泛型就不用宏（宏更难调试、可读性差）
```

# 4. 小结

| 概念 | 要点 |
|------|------|
| unsafe | 解锁 5 项能力，责任转移给程序员 |
| 裸指针 | *const/*mut，解引用需 unsafe |
| 安全抽象 | unsafe 封装在安全 API 内 |
| macro_rules! | 声明宏，模式匹配 |
| 片段类型 | ident/expr/ty/tt 等 |
| 重复匹配 | $( ... ),* |
| 过程宏 | derive/attribute/function，独立 crate |

至此，Rust 从入门到核心特性的学习路径已经走完。从安装、变量、所有权，到 trait、并发、智能指针、面向对象与 unsafe/宏，你已建立对 Rust 的完整认知框架。
