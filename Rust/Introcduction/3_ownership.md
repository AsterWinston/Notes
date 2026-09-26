所有权（ownership）是 Rust 最核心、最独特的概念，也是它无需 GC 就能保证内存安全的关键。所有权的三条规则配合借用检查器（borrow checker），在编译期就杜绝了悬垂指针、内存泄漏和双重释放。

# 1. stack

## 1.1 栈的特性

```
栈（stack）：后进先出（LIFO），存储大小已知、固定的数据
  - 分配/释放极快：只需移动栈指针
  - 数据必须编译期确定大小
  - 函数调用、局部变量、基本类型都在栈上

示例：i32、bool、char、定长数组、元组（元素都是栈类型时）
```

★ **关键认知**：栈分配只是栈指针的加减，近乎零成本。Rust 编译器会尽可能把数据放在栈上——能放栈上就不放堆上。

# 2. heap

## 2.1 堆的特性

```
堆（heap）：存储大小未知或运行期才能确定的数据
  - 分配需要向分配器申请，有开销
  - 通过指针访问（栈上存指针，堆上存数据）
  - String、Vec<T>、Box<T> 的底层数据都在堆上

分配流程：
1. 分配器在堆上找一块足够大的空间
2. 标记为已用，返回指向它的指针
3. 指针（固定大小）存在栈上
```

★ **关键认知**：栈 vs 堆的根本区别是**大小是否编译期已知**。Rust 的所有权系统正是为了解决"堆数据何时释放、由谁释放"的问题——不需要 GC，而是通过编译期的所有权规则在确定的时机插入释放代码。

# 3. String

```rust
3.1 String::from("hello")           // 从字面量创建
3.2 s.clone()                       // 深拷贝（复制堆数据）
3.3 s.clear()                       // 清空（变为空串）
3.4 s.push_str("world")             // 追加字符串
3.5 s.len()                         // 字节长度
```

★ **关键认知**：字符串字面量 `&str` 是**栈上/静态区**的不可变引用；`String` 是**堆上**的可变、可增长字符串。`String` 在栈上存三部分：**指针 + 长度 + 容量**（ptr/len/cap），实际字节存在堆上。这与 Go 的 string header 类似，但 Go 的 string 不可变且缺 cap。

# 4. move —— 移动语义

## 4.1 赋值即移动

```rust
let s1 = String::from("hello");
let s2 = s1;              // s1 被"移动"到 s2，s1 失效

println!("{}", s1);       // 编译错误：value borrowed here after move
```

★ **关键认知**：对于堆类型，赋值/传参是**移动（move）**而非拷贝——所有权从 s1 转移给 s2，s1 立即失效。这避免了两者同时持有同一块堆内存、导致**重复释放（double free）**。这是 Rust 与 C++ 智能指针 `unique_ptr` 类似的思路，但由编译器强制。

## 4.2 移动的内存示意

```
let s2 = s1 之后：

s1（失效）          s2
┌────┬────┬───┐    ┌────┬────┬───┐
│ptr │len │cap│    │ptr │len │cap│
└─┬──┴────┴───┘    └─┬──┴────┴───┘
  └────────同一指针────┘
                 │
                 ▼
              堆上的 "hello"
（Rust 在移动后令 s1 失效，保证只有一个所有者）
```

# 5. shallow copy

```
浅拷贝：只复制栈上的指针/len/cap，不复制堆数据
```

★ **关键认知**：Rust **从不隐式做浅拷贝**——这正是 `let s2 = s1` 变成 move 而非浅拷贝的原因。因为浅拷贝会让两个变量指向同一堆内存，造成"两个所有者"的歧义。Rust 用 move 消除了这个隐患。需要真正独立的数据，才显式调用 `.clone()`。

# 6. copy trait

## 6.1 实现 Copy 的类型

```rust
let x = 5;
let y = x;               // x 仍可用（Copy 语义，按位复制）
println!("{}", x);       // 合法

// 实现了 Copy 的类型：所有整数/浮点/布尔/char/元组（元素都 Copy）/数组（元素 Copy）
```

★ **关键认知**：实现了 `Copy` trait 的类型在赋值时**按位复制**，原变量仍然可用（因为数据全在栈上，无堆共享问题）。`Copy` 与 `Drop` **互斥**——实现了 `Drop` 的类型不能实现 `Copy`（有堆资源需要释放）。`String`/`Vec` 没有 Copy，所以是 move。

## 6.2 Clone vs Copy

```rust
#[derive(Clone, Copy)]
struct Point { x: i32, y: i32 }   // 可以同时 derive

let p = Point { x: 1, y: 2 };
let q = p;      // Copy：p 仍可用
```

★ **关键认知**：`Copy` 是"隐式、栈上的按位复制"；`Clone` 是"显式、可包含堆数据深拷贝"。`Copy` 是 `Clone` 的子集（Copy 隐含 Clone）。自定义类型可用 `#[derive(Copy, Clone)]` 自动实现。

# 7. drop trait

## 7.1 自动释放

```rust
{
    let s = String::from("hello");   // s 进入作用域
    // 使用 s
}   // ★ 作用域结束，Rust 自动调用 s.drop()，释放堆内存
```

★ **关键认知**：`Drop` trait 定义了"值离开作用域时自动执行的清理逻辑"。Rust 在编译期确定每个值的生命周期，在离开作用域时**自动插入 drop 调用**——这就是 Rust 无需 GC 却能精确释放内存的原因（称为 RAII）。

## 7.2 自定义 Drop

```rust
struct MyRes { }
impl Drop for MyRes {
    fn drop(&mut self) {
        println!("releasing resource");
    }
}
```

★ **关键认知**：drop 的调用顺序是**后进先出**（LIFO）——后声明的变量先释放。可以用 `std::mem::drop(x)` 提前释放（其实是移动进 `drop` 函数）。⚠ 无法手动调用 `.drop()` 方法（会与自动 drop 冲突）。

# 8. function and ownership

```rust
fn takes_ownership(s: String) { /* s 在函数结束时被 drop */ }
fn makes_copy(n: i32) { /* n 是 Copy，调用方仍可用 */ }

let s = String::from("hello");
takes_ownership(s);      // s 被 move 进函数，之后 s 失效
let n = 5;
makes_copy(n);           // n 是 Copy，仍可用
```

★ **关键认知**：把堆类型的值传入函数会**移动所有权**，函数结束后该值被 drop。若想让调用方继续使用，要么传引用（借用），要么函数把所有权返回。这是 Rust 函数设计的核心考量。

## 8.1 返回所有权

```rust
fn give_ownership() -> String {
    let s = String::from("hello");
    s                    // 返回 s，所有权移交给调用方（不 drop）
}
```

# 9. reference —— 引用与借用

## 9.1 不可变引用

```rust
9.1.1 &<type>        // 引用类型，如 &String
9.1.2 &<var>         // 取引用（借用），如 &s
9.1.3 *<var>         // 解引用，如 *r
```

```rust
fn calculate_length(s: &String) -> usize {   // s 是借用，不获取所有权
    s.len()
}   // s 离开作用域，但因为没有所有权，不 drop

let s = String::from("hello");
let len = calculate_length(&s);   // 借用 s
println!("{}", s);                // s 仍可用
```

★ **关键认知**：**引用（reference）** 让你在不获取所有权的情况下使用值，称为**借用（borrowing）**。借用不转移所有权，所以原变量在借用结束后仍可用。默认引用是**不可变**的——不能通过它修改数据。

## 9.2 可变引用

```rust
9.2.1 mut <var>            // 原变量必须声明为 mut
9.2.2 &mut <var>           // 取可变引用
9.2.3 &mut <type>          // 可变引用类型
9.2.4 *<var>               // 解引用修改
```

```rust
fn change(s: &mut String) {
    s.push_str(", world");
}

let mut s = String::from("hello");   // 必须 mut
change(&mut s);                       // 传可变引用
```

★ **关键认知**：可变引用允许修改借用的数据，但原变量必须声明为 `mut`。可变引用有严格限制（见引用规则），这是 Rust 在编译期防止数据竞争的核心。

# 10. dangling reference —— 悬垂引用

```rust
fn dangle() -> &String {           // 错误：返回局部变量的引用
    let s = String::from("hello");
    &s                             // s 在函数结束时被 drop，引用悬垂
}
```

```
编译器报错：missing lifetime specifier / `s` does not live long enough

正确写法：返回所有权（而不是引用）
fn no_dangle() -> String {
    let s = String::from("hello");
    s                              // 移动所有权出去，s 不 drop
}
```

★ **关键认知**：**悬垂引用（dangling reference）** 是指向已被释放内存的引用。C/C++ 里这是运行时崩溃/安全漏洞，Rust 在**编译期**通过生命周期检查直接拒绝。这是 Rust "无 GC 又内存安全"的核心承诺——**编译器保证引用始终有效**。

# 11. slice

## 11.1 字符串切片

```rust
11.1 &s[x..y]     // [x, y) 范围
11.2 &s[x..]      // 从 x 到末尾
11.3 &s[..y]      // 从开始到 y
11.4 &s[..]       // 整个字符串
```

```rust
let s = String::from("hello world");
let hello = &s[0..5];
let world = &s[6..11];
```

★ **关键认知**：切片（slice）是对**序列一部分**的引用（`&str`、`&[T]`），它不含所有权，只是"指向 + 长度"。字符串切片 `&str` 的索引按**字节**计算，⚠ 若切在多字节字符中间会 **panic**（因为 UTF-8 边界）。

## 11.2 slice of other type

```rust
let arr = [1, 2, 3, 4, 5];
let slice: &[i32] = &arr[1..3];    // [2, 3]
```

★ **关键认知**：切片的类型是 `&[T]`，它比 `&Vec<T>` 更通用（能同时接受数组和 Vec）。函数参数优先用 `&[T]` 而非 `&Vec<T>`，用 `&str` 而非 `&String`——这体现了 Rust 的"deref coercion"（解引用强制转换）。

# 12. reference rule —— 引用规则

```
12.1 引用必须有效：不能创建悬垂引用（Dangling Reference）
12.2 可变性规则（同一作用域内）：
     12.2.1 可以同时存在多个不可变引用
     12.2.2 同一时刻最多一个可变引用（防止数据竞争）
     12.2.3 可变引用和不可变引用不能共存
12.3 引用的作用域：引用的生命周期必须长于其使用范围，否则触发编译错误
```

```rust
// 合法：多个不可变引用
let r1 = &s;
let r2 = &s;    // OK

// 非法：可变与不可变共存
let r1 = &s;
let r2 = &mut s;   // 编译错误
```

★ **关键认知**：这条规则（"**要么多个不可变引用，要么一个可变引用**"）是 Rust 在编译期消除数据竞争的关键。它对应读写锁的语义：读读共享、读写互斥、写写互斥。**借用检查器（borrow checker）** 在编译期验证这条规则，因此 Rust 的并发代码天然无数据竞争。

# 13. deref coercion —— 解引用强制转换

```rust
fn hello(name: &str) { }

let s = String::from("world");
hello(&s);        // &String 自动转成 &str，无需 &s[..]
```

★ **关键认知**：**Deref coercion（解引用强制转换）** 是编译器自动应用的转换：`&String` → `&str`、`&Vec<T>` → `&[T]`、`&Box<T>` → `&T`。它让函数签名可以用更通用的引用类型（如 `&str`），而调用方能直接传 `&String`。底层依赖 `Deref` trait（见智能指针章节）。

# 14. 小结

| 概念 | 要点 |
|------|------|
| 栈/堆 | 栈大小已知、快；堆大小未知、有指针 |
| move | 堆类型赋值即转移所有权，原变量失效 |
| Copy | 栈类型按位复制，原变量仍可用 |
| Drop | 离开作用域自动释放（RAII） |
| 借用 | &T 不转移所有权 |
| 引用规则 | 多读 XOR 一写，编译期防数据竞争 |
| 悬垂引用 | 编译期拒绝，保证引用有效 |
| 切片 | &str / &[T]，对部分序列的引用 |
| deref coercion | &String→&str 等自动转换 |

下一篇进入结构体与方法，讲清楚 struct、impl 块、方法、关联函数，以及用 trait 实现共享行为。
