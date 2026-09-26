Rust 的代码组织围绕 **package → crate → module** 三层展开，配合 `use`/`pub` 控制引用与可见性。理解这套体系，是构建多文件、多模块项目的基础。

# 1. package

★ **关键认知**：`package` 是 cargo 的概念（**不是** Rust 语言概念），用于构建、测试、共享 crate。它是 `Cargo.toml` 所描述的一个整体：

```
1.1 每个 package 包含一个 Cargo.toml，描述如何构建这些 crate
1.2 只能包含 0~1 个 library crate
1.3 可以包含任意数量的 binary crate
1.4 必须包含至少一个 crate（library 或 binary）
```

★ **关键认知**：用 `cargo new myproj` 创建的 package，默认 `src/main.rs` 是一个 binary crate 的根；若加 `--lib`，则 `src/lib.rs` 是 library crate 的根。一个 package 可以同时有 `src/main.rs` 和 `src/lib.rs`。

# 2. crate

★ **关键认知**：`crate` 是编译器眼中的**编译单元**，是一棵模块树，可产生一个 library 或一个可执行文件：

```
2.1 分类：
    2.1.1 binary crate（可执行，必须有 main 函数）
    2.1.2 library crate（库，供其他 crate 引用）
2.2 crate root：
    2.2.1 是源码文件，编译器从这里开始组织 crate 的根 module
    2.2.2 binary crate 的根通常是 src/main.rs
    2.2.3 library crate 的根通常是 src/lib.rs
```

```
crate 的模块树示意：
crate (root: src/main.rs 或 src/lib.rs)
 ├── mod front_of_house
 │    ├── mod hosting
 │    └── mod serving
 └── mod back_of_house
      └── mod cook
```

★ **关键认知**：crate root 文件是模块树的根，`mod` 声明的子模块挂在它下面。整个 crate 用 **crate 关键字**（`crate::`）引用根。crate 是 Rust 里"编译 + 链接 + 优化"的基本单位。

# 3. module

★ **关键认知**：`module`（模块）用于**控制代码的组织、作用域和私有路径**，是 crate 内的命名空间：

```rust
// 内联模块
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}
```

```
3.1 模块可以嵌套
3.2 模块定义了一个命名空间，其中的项通过路径访问
3.3 模块可以写在同一个文件里（{ }），也可以写成独立文件
```

★ **关键认知**：用 `mod xxx;`（不带 `{ }`）可以声明"引出一个独立文件中的模块"。编译器会自动查找 `xxx.rs` 或 `xxx/mod.rs`。这是大项目分文件组织的标准方式。

## 3.1 模块的文件组织

```
src/
├── main.rs              // crate root
├── garden.rs            // mod garden;  → 内容放这里
└── garden/
    ├── vegetables.rs    // mod vegetables;（嵌套在 garden 内）
    └── mod.rs           // garden 的旧式入口（2018 后可用 garden.rs 代替）
```

★ **关键认知**：Rust 2018 起，`foo.rs` + `foo/` 目录是推荐写法（替代旧的 `foo/mod.rs`）。`mod garden;` 会优先找 `garden.rs`，再找 `garden/mod.rs`。

# 4. pub —— 可见性

## 4.1 默认私有

★ **关键认知**：Rust 中所有项（模块、函数、结构体、字段、枚举变体）**默认私有**：

```
4.1 模块内的项默认对父模块私有
4.2 用 pub 关键字使其公开
4.3 结构体字段默认私有，需逐个标 pub（或用 pub(crate)）
4.4 枚举变体默认公开（若枚举本身 pub）
```

```rust
mod my_mod {
    pub fn public_fn() {}         // 对外可见
    fn private_fn() {}            // 仅本模块可见

    pub struct Foo {
        pub x: i32,               // 字段公开
        y: i32,                   // 字段私有
    }
}
```

## 4.2 可见性范围

```rust
pub                   // 任何地方可见
pub(crate)            // 仅当前 crate 内可见
pub(super)            // 仅父模块可见
pub(in path)          // 仅指定路径内可见
pub(self)             // 等价私有
```

★ **关键认知**：Rust 的可见性是**精细可控**的——`pub(crate)` 实现"包内公开、包外私有"（类似 Go 的 internal），`pub(super)` 实现"对父模块公开"。父模块**不能**访问子模块的私有项，但子模块可以访问祖先模块的项。

# 5. use

★ **关键认知**：`use` 用于**把路径引入当前作用域**，避免每处都写完整路径：

```rust
use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();   // 引入后直接用
}
```

```
5.1 use crate::...        // crate 根开始的绝对路径（crate 是根的关键字）
5.2 use 可以在模块/函数内局部引入
5.3 惯用法：引入函数时引入其父模块；引入结构体/枚举时直接引入本身
```

★ **关键认知**：惯用法上，引入**函数**时通常 `use path::module;` 然后用 `module::func()`（明确来源）；引入**结构体/枚举**时直接 `use path::Type;`。这不是硬规则，但能提升可读性。

# 6. mod

★ **关键认知**：`mod` 关键字有两种用法：

```rust
mod inline_mod {          // 1. 内联定义模块
    pub fn f() {}
}

mod external_mod;         // 2. 声明外部文件模块（找 external_mod.rs 或 external_mod/mod.rs）
```

# 7. crate 关键字

★ **关键认知**：`crate` 在路径中表示**当前 crate 的根**，用于写绝对路径：

```rust
use crate::front_of_house::hosting;   // 从 crate 根开始
```

它相当于文件系统中的 `/`（根目录），保证路径不受当前模块位置影响。

# 8. super

★ **关键认知**：`super` 表示**父模块**，相当于文件系统的 `..`：

```rust
mod child {
    fn call_parent() {
        super::parent_fn();    // 调用父模块的函数
    }
}
```

# 9. self

★ **关键认知**：`self` 在路径中表示**当前模块**：

```rust
mod my_mod {
    pub fn a() {}
    pub fn b() {
        self::a();     // 等价于直接调用 a()
    }
}
```

# 10. as

★ **关键认知**：`as` 用于**重命名引入**，解决命名冲突或简化：

```rust
use std::fmt::Result;
use std::io::Result as IoResult;   // 用 as 重命名，避免冲突

use std::collections::HashMap as Map;
```

# 11. pub use —— 重导出

★ **关键认知**：`pub use` 把引入的项**再公开出去**，实现"**重导出（re-export）**"。这是设计清晰 API 的关键：

```rust
// 在 crate 根
pub use crate::front_of_house::hosting;   // 外部可直接 use mycrate::hosting

// 典型应用：把深层模块的项提到 crate 顶层，简化使用者路径
pub use self::inner::PublicType;
```

★ **关键认知**：`pub use` 让 crate 可以"对外暴露漂亮的 API，内部保持清晰的模块层次"。用户在 `use mycrate::PublicType` 时，不需要知道它在内部 `inner` 模块里。这是 Rust 库设计的常用手法。

# 12. 小结

| 概念 | 要点 |
|------|------|
| package | cargo 概念，含 1 个 Cargo.toml |
| crate | 编译单元，模块树的根 |
| module | 命名空间，控制组织与可见性 |
| pub | 默认私有，pub 公开 |
| pub(crate) | 包内公开、包外私有 |
| use | 引入路径到作用域 |
| crate/super/self | 根/父/当前模块 |
| as | 重命名引入 |
| pub use | 重导出，简化 API |

下一篇进入常用集合，讲清楚 `Vec<T>`、`String`/`&str`、`HashMap<K, V>` 的用法与底层特点。
