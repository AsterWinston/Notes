Serde 是 Rust 生态里事实上的序列化标准，但它本身不处理任何具体格式——它只定义了一套"数据模型"和一对 trait，把"你的类型长什么样"与"某种格式怎么写"彻底解耦。理解这层分工，是用好 `serde_json` 以及后续所有属性的前提。

# 1. Serde 的定位

## 1.1 它解决什么问题

在没有统一框架的语言里，每接一种格式就要写一套转换代码：JSON 一套、YAML 一套、数据库一套。字段一改，所有转换代码都要跟着改。Serde 把这些重复劳动收敛成一次 `derive`：

```rust
#[derive(serde::Serialize, serde::Deserialize)]
struct User {
    id: u64,
    name: String,
}
```

一次声明，就能被 JSON、YAML、TOML、MessagePack、bincode 等格式复用。关键不在于宏有多神奇，而在于中间那层抽象。

## 1.2 两个核心 trait

```rust
pub trait Serialize {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer;
}

pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>;
}
```

- `Serialize`：把 `&self` 交给一个 `Serializer`，由它决定往哪个格式里写。
- `Deserialize`：从 `Deserializer` 里读出一个 `Self`。
- 两个 trait 都对格式泛型，所以同一份实现可以服务任意后端。

★ **关键认知**：`Serializer` 和 `Deserializer` 不是 serde 自己实现的，而是由**每种格式各自实现**。serde 只规定"调用顺序和值类型"，格式库负责"这些调用最终写成什么字节"。这就是为什么一份 `derive` 能同时支持 JSON 和二进制——类型实现面向的是协议，不是具体格式。

## 1.3 serde 与 serde_json 的分工

```text
你的类型 User
    │  #[derive(Serialize, Deserialize)]
    ▼
serde 数据模型（trait 调用：struct / field / seq / map ...）
    │
    ├── serde_json  →  {"id":1,"name":"alice"}
    ├── serde_yaml  →  id: 1
    ├── toml        →  id = 1
    └── bincode     →  二进制字节
```

| crate | 职责 | 是否需要手动调用 |
|-------|------|-----------------|
| `serde` | trait、数据模型、derive 宏 | 写 `derive` 时用到 |
| `serde_json` | 实现 JSON 的 Serializer/Deserializer，提供 `Value` | 读写 JSON 时用 |
| `serde_yaml` / `toml` | 实现对应格式 | 读写对应格式时用 |

# 2. Serde 的数据模型

## 2.1 29 种值类型

Serde 把 Rust 类型归纳成一组格式无关的"值类型"。格式库只需要支持这些类型，就能覆盖所有 `derive` 出来的结构：

| 分类 | 包含的类型 |
|------|-----------|
| 标量 | `bool`、`i8`/`i16`/`i32`/`i64`/`i128`、`u8`/`u16`/`u32`/`u64`/`u128`、`f32`/`f64`、`char` |
| 字符串与字节 | `String`/`&str`、`&[u8]`/`Vec<u8>` |
| 可选与空 | `Option<T>`、`()`（unit） |
| 结构 | unit struct、newtype struct、tuple struct、struct |
| 序列 | `Vec<T>`、数组、元组 |
| 映射 | `HashMap`、`BTreeMap` |
| 枚举 | unit variant、newtype variant、tuple variant、struct variant |

★ **关键认知**：数据模型是**格式能力的交集**，不是 Rust 类型的全集。比如 `HashMap<u32, String>` 能编译，但 JSON 的键必须是字符串，序列化时会报"key must be a string"。⚠ 选类型时要先想清楚目标格式能不能表达，而不是先想 Rust 能不能编过。

## 2.2 类型到格式的映射

同一个 `User`，在不同格式里的形态不同，但走的是同一套 `serialize_struct` 调用：

```text
Serialize::serialize(&user)
  → serializer.serialize_struct("User", 3)
  → state.serialize_field("id", &1)
  → state.serialize_field("name", &"alice")
  → state.serialize_field("active", &true)
  → state.end()
```

- JSON 把它写成对象 `{...}`。
- YAML 写成缩进映射。
- bincode 写成紧凑的字段序列，连字段名都不存。

# 3. derive 宏

## 3.1 最小可运行示例

`Cargo.toml`：

```toml
[package]
name = "serde-demo"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

`src/main.rs`：

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, PartialEq)]
struct User {
    id: u64,
    name: String,
    active: bool,
}

fn main() {
    let user = User {
        id: 1,
        name: "alice".to_string(),
        active: true,
    };

    // 序列化：结构体 → JSON 字符串
    let json = serde_json::to_string(&user).unwrap();
    println!("{json}");

    // 反序列化：JSON 字符串 → 结构体
    let back: User = serde_json::from_str(&json).unwrap();
    assert_eq!(user, back); // PartialEq 让这里可以直接比较
}
```

运行输出：

```text
{"id":1,"name":"alice","active":true}
```

## 3.2 derive 到底生成了什么

宏展开后大致等价于手写下面两段。先看 `Serialize`：

```rust
impl Serialize for User {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        use serde::ser::SerializeStruct;
        let mut state = serializer.serialize_struct("User", 3)?; // 3 是字段数
        state.serialize_field("id", &self.id)?;                  // 字段名成了字符串
        state.serialize_field("name", &self.name)?;
        state.serialize_field("active", &self.active)?;
        state.end()
    }
}
```

`Deserialize` 会生成一个内部 `Visitor`，逐个匹配字段名，缺失的必填字段报 `missing field`，类型不符报 `invalid type`。所以字段名默认就是 Rust 字段名，大小写、下划线都原样输出——这一点在对接前端时会立刻遇到。

★ **关键认知**：`derive` 生成的是**通用协议代码**，不含任何格式判断。字段名、变体名在序列化时是普通字符串；所以后面 `rename`、`rename_all` 这类属性，本质上只是在改这些字符串，不改变类型系统。

## 3.3 derive 的边界

| 情况 | 行为 |
|------|------|
| 结构体/枚举 | 完整支持 |
| 泛型类型 | 自动加 `T: Serialize` / `T: Deserialize` 约束 |
| `PhantomData<T>` | 自动跳过，不产生字段 |
| 联合体 `union` | 不支持，需手写 |
| 含引用的结构体 | 需要生命周期，常用 `#[serde(borrow)]` |
| 只有 `Serialize` 没有 `Deserialize` | 完全可以，各自独立 |

⚠ `#[derive(Serialize)]` 和 `#[derive(Deserialize)]` 是**两个独立宏**，只写一个就只得到一半能力。报错 `the trait bound User: Deserialize is not satisfied` 时，先检查是不是漏了其中一个。

# 4. 依赖配置

## 4.1 常用写法

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# 需要借用 &str / &[u8] 零拷贝时
serde = { version = "1.0", features = ["derive", "rc"] }
```

`features = ["derive"]` 会启用 `serde_derive` 过程宏。忘了它，`use serde::Serialize` 还能编，但 `#[derive(Serialize)]` 会报"找不到 derive 宏"。

## 4.2 常用 feature

| crate | feature | 作用 |
|-------|---------|------|
| `serde` | `derive` | 启用 derive 宏 |
| `serde` | `std`（默认） | 标准库支持 |
| `serde` | `alloc` | `no_std` + 堆分配 |
| `serde` | `rc` | 支持 `Rc<T>` / `Arc<T>` |
| `serde_json` | `preserve_order` | 用 `IndexMap` 保留键顺序 |
| `serde_json` | `arbitrary_precision` | 数字保留原始字符串精度 |
| `serde_json` | `raw_value` | 延迟解析部分 JSON |

★ **关键认知**：`no_std` 环境下要关掉默认 feature：`default-features = false, features = ["derive", "alloc"]`。⚠ 此时 `serde_json` 也需要 `alloc`，且不能用 `to_string`，只能用 `to_vec` 一类基于 `Vec<u8>` 的接口。

# 5. 常见错误

| 报错 | 原因 | 解决 |
|------|------|------|
| `the trait bound User: Serialize is not satisfied` | 漏 derive，或 derive 的是另一个 trait | 补 `#[derive(Serialize)]` |
| `cannot find derive macro Serialize` | 没开 `derive` feature，或没 `use serde::Serialize` | 加 feature，导入 trait |
| `missing field 'name'` | JSON 缺必填字段 | 补字段，或后面用 `#[serde(default)]` |
| `invalid type: integer, expected a string` | JSON 类型与字段类型不符 | 改字段类型或做自定义反序列化 |
| `key must be a string` | 用了非字符串键的 map | 换 `BTreeMap<String, _>` 或自定义 |
| `lifetime may not live long enough` | 结构体借用了输入字符串 | 改 `String` 或用 `#[serde(borrow)]` |

# 6. 练习

1. 定义一个 `Book { title: String, pages: u32, tags: Vec<String> }`，分别序列化成 JSON 和 YAML，比较两种输出的结构差异。
2. 故意把 JSON 里的 `pages` 改成字符串 `"300"`，观察 `serde_json` 的报错原文，并判断这个错误属于语法错误还是数据错误。
3. 给 `Book` 加上泛型字段 `meta: T`，先不写 `T: Serialize` 约束，看编译器如何提示，再理解 derive 自动加约束的行为。

# 7. 小结

| 概念 | 要点 |
|------|------|
| serde 本体 | 只提供数据模型和 trait，不绑定格式 |
| Serialize | 类型 → `Serializer`，由格式决定输出 |
| Deserialize | 从 `Deserializer` 读出类型 |
| 数据模型 | 29 种值类型，是各格式能力的交集 |
| derive | 生成协议代码，字段名默认原样输出 |
| feature | `derive` 必开，`no_std` 要关默认 feature |

下一篇进入 `serde_json` 与其他格式，讲清楚字符串/文件/流的读写接口、`Value` 动态树、错误定位，以及 JSON、YAML、TOML 的选型差异。
