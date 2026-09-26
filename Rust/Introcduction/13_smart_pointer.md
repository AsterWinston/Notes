智能指针（smart pointer）是"像指针一样，但带有额外元数据和能力"的数据结构。Rust 标准库提供了几种关键智能指针，用来处理堆分配、共享所有权和内部可变性——它们配合所有权系统，扩展了内存管理的能力。

# 1. smart pointer

★ **关键认知**：智能指针本质是**实现了 `Deref` 和 `Drop` 的结构体**：

```
1.1 Deref trait    —— 让智能指针能像引用一样使用（支持 * 解引用、deref coercion）
1.2 Drop trait     —— 离开作用域时自动清理堆资源（RAII）

普通引用 &T：只借用，不拥有，不释放
智能指针：拥有数据，负责释放，可带额外能力
```

```
常用智能指针：
Box<T>       独占所有权的堆分配
Rc<T>        引用计数的共享所有权（单线程）
Arc<T>       原子引用计数（多线程）
RefCell<T>   内部可变性（运行时借用检查）
```

# 2. Box<T> —— 堆上数据的独占指针

## 2.1 基本用法

```rust
let b = Box::new(5);          // 把 5 分配到堆上，b 是 Box<i32>
println!("{}", *b);           // 解引用，得到 5
```

★ **关键认知**：`Box<T>` 把数据放在**堆上**，自身（指针）在栈上，拥有数据的**独占所有权**。离开作用域时自动释放堆内存。它是"最简单的智能指针"。

## 2.2 典型用途

```rust
// 1. 递归类型（大小无法编译期确定）
enum List {
    Cons(i32, Box<List>),     // 用 Box 打破无限大小
    Nil,
}

// 2. 大对象转移所有权时避免拷贝（只移动 8 字节指针）
let big = Box::new([0u8; 1_000_000]);

// 3. trait 对象（动态分发）
let obj: Box<dyn Draw> = Box::new(Circle {});
```

★ **关键认知**：`Box` 最经典的用途是**递归类型**——普通递归枚举 `Cons(i32, List)` 的大小无限，编译报错；用 `Box<List>`（指针大小固定）打破递归，让类型大小可确定。

# 3. deref trait

## 3.1 解引用强制转换

★ **关键认知**：`Deref` trait 让智能指针能像普通引用一样使用。它支持 **deref coercion（解引用强制转换）**——编译器自动把 `&Box<T>` 转成 `&T`：

```rust
fn hello(name: &str) { }

let s = String::from("world");
hello(&s);                  // &String → &str（自动转换）
let b = Box::new(String::from("world"));
hello(&b);                  // &Box<String> → &str（链式转换）
```

```
deref coercion 转换链：
&Box<String> → &String → &str
```

## 3.2 手动实现 Deref

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> Deref for MyBox<T> {
    type Target = T;
    fn deref(&self) -> &T {
        &self.0
    }
}

let x = MyBox(5);
assert_eq!(5, *x);          // *x 展开为 *(x.deref())
```

★ **关键认知**：`*x` 由编译器展开为 `*(x.deref())`。实现 `Deref` 后，你的类型就能参与自动解引用和 deref coercion，用起来像引用一样自然。

# 4. drop trait

★ **关键认知**：`Drop` trait 定义"值离开作用域时的自动清理逻辑"，是 RAII 的核心：

```rust
struct CustomSmartPointer { data: String }

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping with data `{}`", self.data);
    }
}

{
    let c = CustomSmartPointer { data: String::from("x") };
}   // 作用域结束，自动调用 drop
```

```
4.1 drop 调用顺序：后进先出（LIFO）
4.2 可用 std::mem::drop(x) 提前释放
4.3 不能手动调用 .drop() 方法
```

★ **关键认知**：Drop 让 Rust 能精确、自动地释放资源（内存、文件句柄、锁等），无需 GC。这是"零开销、确定性析构"的关键。⚠ 实现了 `Drop` 的类型不能再实现 `Copy`。

# 5. Rc<T> —— 引用计数共享所有权

## 5.1 基本用法

```rust
use std::rc::Rc;

let a = Rc::new(String::from("shared"));
let b = a.clone();          // 克隆 Rc，增加引用计数（不是克隆数据！）
let c = Rc::clone(&a);      // 等价写法，惯用

println!("count = {}", Rc::strong_count(&a));   // 3
```

★ **关键认知**：`Rc<T>`（Reference Counting）实现**多所有权共享**——通过引用计数跟踪有多少个 `Rc` 指向同一数据，计数归零时释放。`Rc::clone` 只增加计数（廉价），**不复制底层数据**。

## 5.2 单线程限制

★ **关键认知**：`Rc<T>` **不是线程安全的**（计数非原子操作），只能在**单线程**中使用。多线程场景用 `Arc<T>`（Atomic Rc），它用原子操作维护计数，可跨线程共享。

# 6. RefCell<T> —— 内部可变性

## 6.1 内部可变性

★ **关键认知**：`RefCell<T>` 提供**内部可变性（interior mutability）**——在持有不可变引用时，也能修改内部数据。它把借用检查**从编译期推迟到运行时**：

```rust
use std::cell::RefCell;

let c = RefCell::new(5);

*c.borrow_mut() += 10;          // 通过不可变绑定的 c 修改内部值
println!("{}", c.borrow());     // 15
```

```
borrow()      返回 Ref<T>，运行时检查"不可变借用"
borrow_mut()  返回 RefMut<T>，运行时检查"可变借用"
```

★ **关键认知**：`RefCell` 在运行时执行借用规则——如果同时存在可变借用和不可变借用（违反规则），会在**运行时 panic**（而非编译错误）。它在"编译期无法证明安全、但逻辑上安全"的场景有用。

## 6.2 Rc<RefCell<T>> 组合

```rust
use std::rc::Rc;
use std::cell::RefCell;

// 共享 + 可变：多个所有者都能修改同一数据
let shared = Rc::new(RefCell::new(vec![1, 2, 3]));

let a = Rc::clone(&shared);
let b = Rc::clone(&shared);

a.borrow_mut().push(4);         // a 修改
b.borrow_mut().push(5);         // b 修改
println!("{:?}", shared.borrow());   // [1, 2, 3, 4, 5]
```

★ **关键认知**：`Rc<RefCell<T>>` 是 Rust 里实现"**共享可变状态**"的经典组合——`Rc` 提供共享所有权，`RefCell` 提供可变性。多线程下对应 `Arc<Mutex<T>>` 或 `Arc<RwLock<T>>`。

# 7. 汇总对比

| 指针 | 所有权 | 可变性 | 线程安全 | 典型场景 |
|------|--------|--------|----------|----------|
| Box<T> | 独占 | 随绑定 | 是（若 T 是） | 递归/大对象/trait 对象 |
| Rc<T> | 共享（计数） | 不可变 | 否 | 单线程多所有者 |
| Arc<T> | 共享（原子计数） | 不可变 | 是 | 多线程多所有者 |
| RefCell<T> | 独占 | 内部可变 | 否 | 运行时借用检查 |
| Cell<T> | 独占 | 内部可变（Copy） | 否 | 简单复制类型 |
| Rc<RefCell<T>> | 共享 | 内部可变 | 否 | 单线程共享可变 |
| Arc<Mutex<T>> | 共享 | 锁保护 | 是 | 多线程共享可变 |

# 8. 小结

| 概念 | 要点 |
|------|------|
| 智能指针 | 实现 Deref + Drop 的结构体 |
| Box<T> | 独占堆分配，递归类型 |
| Deref | 解引用强制转换 |
| Drop | 离开作用域自动清理 |
| Rc/Arc | 引用计数共享所有权 |
| RefCell | 内部可变性，运行时检查 |

下一篇进入多线程，讲清楚 `thread::spawn`、共享状态（`Mutex`/`Arc`）、以及 `Send`/`Sync` trait 如何保证线程安全。
