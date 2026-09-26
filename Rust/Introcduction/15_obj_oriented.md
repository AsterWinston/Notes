Rust **不是**传统面向对象语言（没有类、没有继承），但它通过**结构体 + impl + trait**支持面向对象编程的三大特性：封装、继承（用组合替代）、多态（用 trait 实现）。理解这种"没有继承的面向对象"，是掌握 Rust 设计哲学的关键。

# 1. 面向对象的三大特性

★ **关键认知**：传统 OOP 的三大特性在 Rust 中的对应：

```
封装（Encapsulation）  → pub 控制可见性（结构体字段默认私有）
继承（Inheritance）    → ❌ Rust 没有继承，用"组合 + trait 默认方法"替代
多态（Polymorphism）   → trait + 泛型（静态分发）/ trait 对象（动态分发）
```

★ **关键认知**：Rust 刻意**不提供继承**，因为继承是"紧耦合"的——子类依赖父类实现，父类改动会波及子类（脆弱基类问题）。Rust 用组合（结构体嵌入）和 trait 共享行为来达到复用目的，耦合更松。

# 2. 封装

## 2.1 私有字段

```rust
pub struct AveragedCollection {
    list: Vec<i32>,       // 私有字段（默认）
    average: f64,         // 私有字段
}

impl AveragedCollection {
    pub fn add(&mut self, value: i32) {   // 公开方法
        self.list.push(value);
        self.update_average();
    }

    pub fn average(&self) -> f64 {        // 公开方法
        self.average
    }

    fn update_average(&mut self) {        // 私有方法
        let total: i32 = self.list.iter().sum();
        self.average = total as f64 / self.list.len() as f64;
    }
}
```

★ **关键认知**：结构体字段默认私有，外部只能通过公开方法访问。这实现了**封装**——内部实现（`list`、`average` 的维护方式）可自由修改，只要公开 API 不变。封装的关键价值是"实现细节与接口解耦"。

# 3. 继承与多态

## 3.1 trait 实现多态

```rust
pub trait Draw {
    fn draw(&self);
}

pub struct Button { pub width: u32 }
pub struct SelectBox { pub options: Vec<String> }

impl Draw for Button {
    fn draw(&self) { /* 绘制按钮 */ }
}
impl Draw for SelectBox {
    fn draw(&self) { /* 绘制下拉框 */ }
}
```

## 3.2 静态分发：泛型

```rust
// 泛型 + trait bound：编译期单态化，静态分发
pub fn draw_all<T: Draw>(items: &[T]) {
    for item in items {
        item.draw();        // 编译期确定调用哪个 draw
    }
}
```

★ **关键认知**：泛型 trait bound 是**静态分发（static dispatch）**——编译期为每个具体类型展开一份代码（单态化），无运行时开销。代价是代码膨胀，且要求集合元素都是同一类型。

## 3.3 动态分发：trait 对象

```rust
// trait 对象：运行时确定具体类型，动态分发
pub struct Screen {
    pub components: Vec<Box<dyn Draw>>,   // 可放任意实现 Draw 的类型
}

impl Screen {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();    // 运行时通过 vtable 查找
        }
    }
}

let screen = Screen {
    components: vec![
        Box::new(Button { width: 50 }),
        Box::new(SelectBox { options: vec![] }),
    ],
};
```

★ **关键认知**：`Box<dyn Draw>` 是 **trait 对象**，支持**动态分发（dynamic dispatch）**——运行时通过虚函数表（vtable）查找具体类型的实现。它允许集合存放**不同类型**的对象（异构集合），代价是一次间接跳转。

## 3.4 静态 vs 动态分发

| 特性 | 静态分发（泛型） | 动态分发（dyn） |
|------|------------------|-----------------|
| 时机 | 编译期单态化 | 运行时 vtable |
| 性能 | 高（可内联） | 略低（间接调用） |
| 代码体积 | 大（每个类型一份） | 小 |
| 异构集合 | ❌ 不支持 | ✅ 支持 |
| 用法 | `T: Draw` | `Box<dyn Draw>` |

★ **关键认知**：选择原则——性能优先、类型统一用**泛型**；需要异构集合、减少代码膨胀用 **trait 对象**。这与 Go 的接口（动态分发）形成对比——Go 只有动态分发，Rust 两种都有。

# 4. 用组合替代继承

## 4.1 结构体嵌入

```rust
struct Engine { hp: u32 }
impl Engine {
    fn start(&self) { println!("engine started"); }
}

struct Car {
    engine: Engine,       // 组合：Car "有一个" Engine
    wheels: u32,
}

impl Car {
    fn start(&self) {
        self.engine.start();   // 显式委托，而非继承
    }
}
```

★ **关键认知**：Rust 用**组合**（"有一个"关系）替代继承（"是一个"关系）。需要复用行为时，显式调用内嵌对象的方法（委托），或用 trait 默认方法。这让类之间的关系更清晰、耦合更松。

## 4.2 trait 默认方法实现代码复用

```rust
trait Greet {
    fn name(&self) -> String;

    fn greet(&self) {                          // 默认方法，提供共享行为
        println!("Hello, {}", self.name());
    }
}

struct Person { name: String }
impl Greet for Person {
    fn name(&self) -> String { self.name.clone() }
}
```

★ **关键认知**：trait 的**默认方法**是 Rust 里"横向代码复用"的主要手段（类似 mixin）。多个类型实现同一 trait，自动获得默认方法，而无需继承。这比继承更灵活（一个类型可实现多个 trait）。

# 5. 小结

| 特性 | Rust 实现 |
|------|-----------|
| 封装 | pub + 私有字段 |
| 继承 | ❌ 用组合 + trait 默认方法替代 |
| 多态（静态） | 泛型 + trait bound |
| 多态（动态） | `Box<dyn Trait>` trait 对象 |
| 代码复用 | 组合、trait 默认方法 |
| 设计哲学 | 组合优于继承，解耦优先 |

下一篇进入 unsafe 与宏，讲清楚 `unsafe` 的五大能力与安全边界，以及声明宏（`macro_rules!`）与过程宏。
