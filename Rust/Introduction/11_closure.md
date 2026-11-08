闭包（closure）是可以捕获环境的匿名函数，迭代器（iterator）是惰性求值的序列处理抽象。两者配合，构成了 Rust 强大的函数式编程能力。

# 1. closure —— 闭包

## 1.1 基本语法

```rust
// 闭包用 |参数| 定义
let add_one = |x| x + 1;
let result = add_one(5);        // 6

// 可标注类型（通常省略，编译器推断）
let add = |x: i32| -> i32 { x + 1 };

// 多行闭包
let calc = |x, y| {
    let sum = x + y;
    sum * 2
};
```

★ **关键认知**：闭包是**匿名函数**，与普通函数的最大区别是**能捕获定义处作用域的变量**。闭包用 `||` 而非 `()` 包裹参数，类型通常可省略（由使用处推断）。

## 1.2 捕获环境

```rust
let x = 10;
let print_x = || println!("{}", x);   // 捕获外部的 x
print_x();
```

## 1.3 三种捕获方式

★ **关键认知**：闭包捕获变量有**三种方式**，对应它如何使用被捕获的值，编译器会自动选择"最宽松的够用方式"：

```
捕获方式（优先级从弱到强）：
1. 不可变借用（&T）   —— 只读取
2. 可变借用（&mut T） —— 修改
3. 获取所有权（T）    —— 消耗/移出

编译器按"能满足需求的最弱方式"捕获。
```

```rust
let mut s = String::from("hello");
let read = || println!("{}", s);          // 不可变借用
let mut modify = || s.push_str("!");      // 可变借用
let consume = move || drop(s);            // 获取所有权
```

# 2. Fn trait

★ **关键认知**：闭包实现了三种 `Fn` trait 之一（或多种），对应不同的捕获方式。这是"闭包作为参数"的类型基础：

```
2.1 Fn      —— 以不可变引用捕获（可多次调用，不修改）
2.2 FnMut   —— 以可变引用捕获（可多次调用，可修改）
2.3 FnOnce  —— 获取所有权（只能调用一次，会消耗捕获的值）
```

```
trait 层级关系（包含关系）：
FnOnce
  └── FnMut
        └── Fn

所有 Fn 都实现 FnMut，所有 FnMut 都实现 FnOnce。
所以：要求 FnOnce 的地方能传 Fn，反之不行。
```

```rust
fn call_fn<F: Fn()>(f: F) { f(); }              // 只接受不可变捕获
fn call_fnmut<F: FnMut()>(mut f: F) { f(); }    // 接受可变捕获
fn call_once<F: FnOnce()>(f: F) { f(); }        // 接受所有权捕获
```

★ **关键认知**：选择哪个 trait 约束取决于闭包"最多需要什么权限"。函数参数应使用**最小够用**的约束——能用 `Fn` 就不要用 `FnMut`，能用 `FnMut` 就不要用 `FnOnce`，这样能接受更广泛的闭包。

## 2.1 移动闭包：move

```rust
let s = String::from("hello");

// move 关键字强制闭包获取所有权
let owns = move || println!("{}", s);
owns();
// println!("{}", s);   // 错误：s 已被 move 进闭包

// 经典场景：把闭包传给线程
use std::thread;
let handle = thread::spawn(move || {
    println!("{}", s);   // 线程可能比 s 活得久，必须 move
});
handle.join().unwrap();
```

★ **关键认知**：`move` 强制闭包**获取被捕获变量的所有权**（而非借用）。这在**多线程**场景是必须的——线程的生命周期不可预测，借用局部变量不安全，必须转移所有权。

# 3. iterator —— 迭代器

## 3.1 Iterator trait

```rust
pub trait Iterator {
    type Item;                                    // 关联类型
    fn next(&mut self) -> Option<Self::Item>;     // 唯一必须实现的方法
}
```

★ **关键认知**：迭代器的核心是 `next()` 方法，返回 `Option<Item>`——有值返回 `Some`，结束返回 `None`。所有其他迭代器方法（`map`、`filter`、`collect`……）都是基于 `next()` 的默认方法。

## 3.2 创建迭代器

```rust
let v = vec![1, 2, 3];
let iter = v.iter();            // 不可变引用迭代（&T）
let iter = v.iter_mut();        // 可变引用迭代（&mut T）
let iter = v.into_iter();       // 获取所有权迭代（T）
```

★ **关键认知**：三种迭代方式的区别是**所有权**——`iter()` 借用、`iter_mut()` 可变借用、`into_iter()` 消耗。`for x in collection` 底层会调用 `into_iter()`。

## 3.3 迭代器适配器（惰性）

★ **关键认知**：迭代器适配器是**惰性的**——它们返回新的迭代器，不立即执行，直到被消费：

```rust
let v = vec![1, 2, 3, 4, 5];

let result: Vec<i32> = v.iter()
    .map(|x| x * 2)          // 映射
    .filter(|x| *x > 5)      // 过滤
    .collect();              // ★ 消费，触发执行
// [6, 8, 10]
```

```
常用适配器（返回新迭代器，惰性）：
map       映射
filter    过滤
filter_map 过滤+映射
flat_map  展平
take(n)   取前 n 个
skip(n)   跳过前 n 个
enumerate 带索引
zip       拉链合并
chain     连接
rev       反转
peekable  可窥视
```

## 3.4 消费者（触发执行）

```
常用消费者（消耗迭代器，产生结果）：
collect()     收集为集合（Vec/HashMap/...）
sum() / product()   求和/求积
count()       计数
fold()        折叠（归约）
for_each()    对每个元素执行
any() / all() 任一/全部满足
find() / position()  查找
max() / min() 最值
```

```rust
let sum: i32 = vec![1, 2, 3].iter().sum();              // 6
let count = vec![1, 2, 3].iter().filter(|x| **x > 1).count();  // 2
let total = vec![1, 2, 3].iter().fold(0, |acc, x| acc + x);    // 6
```

## 3.5 自定义迭代器

```rust
struct Counter { count: u32 }

impl Iterator for Counter {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}
```

★ **关键认知**：实现 `Iterator` 只需实现 `next()`，就能免费获得所有适配器方法。这是 Rust trait 系统的强大之处——一个方法换来一整套 API。

# 4. 零成本抽象

★ **关键认知**：迭代器链与手写循环**性能相当**（甚至更好）——编译器会内联所有适配器、消除中间迭代器，把链式调用优化成高效的循环。这就是 Rust "零成本抽象"的典型体现：抽象不带来运行时开销。

```rust
// 这两种写法编译后性能几乎相同
let sum: i32 = (1..=100).filter(|x| x % 2 == 0).sum();

let mut sum = 0;
for x in 1..=100 {
    if x % 2 == 0 { sum += x; }
}
```

# 5. 小结

| 概念 | 要点 |
|------|------|
| 闭包 | 匿名函数，可捕获环境 |
| 捕获 | 不可变借用 / 可变借用 / 所有权 |
| Fn / FnMut / FnOnce | 对应三种捕获，层级包含 |
| move | 强制获取所有权（多线程必备） |
| Iterator | next() 返回 Option |
| 适配器 | 惰性，返回新迭代器 |
| 消费者 | 触发执行（collect/fold/sum） |
| 零成本 | 迭代器链优化成高效循环 |

下一篇进入 cargo 与 crate 进阶，讲清楚 release 构建、crate 类型、pub use 重导出、workspace 多包管理、安装二进制 crate。
