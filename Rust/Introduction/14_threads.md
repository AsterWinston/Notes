Rust 的并发安全由**所有权系统 + `Send`/`Sync` trait** 在编译期保证，被称为"无畏并发（fearless concurrency）"——编译器帮你杜绝数据竞争。

# 1. multithreads —— 多线程

## 1.1 创建线程

```rust
use std::thread;
use std::time::Duration;

let handle = thread::spawn(|| {
    for i in 1..10 {
        println!("spawned thread: {}", i);
        thread::sleep(Duration::from_millis(1));
    }
});

handle.join().unwrap();    // 等待子线程结束
```

★ **关键认知**：`thread::spawn` 创建新线程，返回 `JoinHandle`。`handle.join()` 阻塞等待该线程结束并返回其结果（`Result`）。主线程结束时，所有未 join 的子线程会被强制终止。

## 1.2 move 闭包

```rust
let v = vec![1, 2, 3];

let handle = thread::spawn(move || {   // ★ 必须 move，转移 v 的所有权
    println!("{:?}", v);
});
handle.join().unwrap();
```

★ **关键认知**：传给线程的闭包通常必须用 `move`——因为线程可能比当前函数活得更久，借用局部变量会悬垂。`move` 把所有权转移进线程，由编译器保证安全。

## 1.3 消息传递：channel

```rust
use std::sync::mpsc;

let (tx, rx) = mpsc::channel();     // 发送端 tx，接收端 rx

thread::spawn(move || {
    tx.send(String::from("hello")).unwrap();
});

let received = rx.recv().unwrap();  // 阻塞接收
println!("{}", received);
```

★ **关键认知**：`mpsc`（multiple producer, single consumer）是多生产者单消费者通道。`send` 转移值的所有权，`recv` 阻塞接收（`try_recv` 非阻塞）。这与 Go 的 channel 理念一致："通过通信共享内存"。

## 1.4 多生产者

```rust
let (tx, rx) = mpsc::channel();
let tx2 = tx.clone();               // 克隆发送端，支持多生产者

thread::spawn(move || tx.send(1).unwrap());
thread::spawn(move || tx2.send(2).unwrap());

for v in rx {                       // 迭代接收
    println!("{}", v);
}
```

# 2. shared —— 共享状态

## 2.1 Mutex<T>

```rust
use std::sync::Mutex;

let m = Mutex::new(5);

{
    let mut num = m.lock().unwrap();   // ★ 获取锁，返回 MutexGuard
    *num = 6;
}   // MutexGuard 离开作用域，自动解锁

println!("{:?}", m);
```

★ **关键认知**：`Mutex<T>` 通过 `lock()` 获取锁，返回 `MutexGuard`（智能指针）。**MutexGuard 实现了 Drop**——离开作用域自动解锁，从语言层面杜绝"忘记解锁"的死锁。⚠ 若持锁线程 panic，锁会"中毒"（poisoned），后续 `lock()` 返回 Err。

## 2.2 Arc<Mutex<T>> 多线程共享可变状态

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));   // Arc 共享所有权 + Mutex 可变
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    });
    handles.push(handle);
}

for handle in handles {
    handle.join().unwrap();
}
println!("Result: {}", *counter.lock().unwrap());   // 10
```

★ **关键认知**：多线程共享可变状态的**标准模式是 `Arc<Mutex<T>>`**——`Arc` 提供跨线程的共享所有权（原子计数），`Mutex` 提供互斥访问。这是 Rust 里对应 `Rc<RefCell<T>>` 的多线程版本。

## 2.3 RwLock

```
RwLock<T>     // 读写锁：多读共享、写独占
```

★ **关键认知**：读多写少的场景用 `RwLock`（比 Mutex 并发读性能更好）。`read()` 返回读守卫，`write()` 返回写守卫，都由 Drop 自动释放。

# 3. sync and send

## 3.1 两个标记 trait

★ **关键认知**：`Send` 和 `Sync` 是 Rust 的**标记 trait（marker trait）**，没有方法，用于在类型层面表达"能否安全跨线程"：

```
3.1 Send：类型的所有权可以安全地转移到另一个线程
3.2 Sync：类型可以安全地被多个线程同时引用（&T 是 Send 的）

直觉理解：
Send  = 可以"搬"到另一个线程
Sync  = 可以"共享"给多个线程
```

```
常见类型：
i32, String, Vec<T>        Send + Sync（大部分类型）
Rc<T>                      既非 Send 也非 Sync（引用计数非原子）
Rc<RefCell<T>>             非 Send（不能跨线程）
Arc<T>                     Send + Sync（T 满足时）
Mutex<T>                   Sync（若 T: Send）
裸指针 *const T            非 Send 非 Sync
```

★ **关键认知**：绝大多数类型自动实现 `Send`/`Sync`（由编译器自动推导）。只有包含非线程安全组件（如 `Rc`、裸指针）的类型才不满足。当你看到"`Rc<T> cannot be sent between threads safely`"时，就是它在起作用。

## 3.2 手动实现（unsafe）

```rust
unsafe impl Send for MyType {}
unsafe impl Sync for MyType {}
```

★ **关键认知**：可以手动 `unsafe impl` 实现 `Send`/`Sync`，但必须**由你保证**类型确实线程安全——编译器无法验证，责任转移到开发者。这是 unsafe 的典型场景。

# 4. 无畏并发

★ **关键认知**：Rust 的"无畏并发"来自**编译期的静态检查**：

```
1. 所有权系统：同一数据同一时刻只有一个可变借用 → 无数据竞争
2. Send/Sync：非线程安全类型无法跨线程 → 编译期拦截
3. 类型系统：数据竞争在编译期就报错，而非运行期崩溃
```

结果：只要代码编译通过，就**不会出现数据竞争**。这是 Rust 相对 C/C++/Go 在并发安全上的巨大优势——把"运行时才暴露的竞态 bug"变成了"编译期错误"。

# 5. 小结

| 概念 | 要点 |
|------|------|
| thread::spawn | 创建线程，move 闭包 |
| join | 等待线程结束 |
| mpsc channel | 多生产者单消费者消息传递 |
| Mutex<T> | 互斥锁，Guard 自动解锁 |
| Arc<Mutex<T>> | 多线程共享可变状态 |
| RwLock | 读写锁 |
| Send | 可跨线程转移所有权 |
| Sync | 可被多线程共享引用 |
| 无畏并发 | 编译期消除数据竞争 |

下一篇进入面向对象，讲清楚 Rust 如何用结构体 + trait + 泛型实现封装、继承（组合）与多态，以及 trait 对象的动态分发。
