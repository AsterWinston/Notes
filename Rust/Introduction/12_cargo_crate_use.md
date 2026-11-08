本章聚焦 cargo 的进阶用法与 crate 的组织：从 release 构建、crate 类型，到重导出、workspace 多包管理与安装二进制工具。

# 1. cargo release

```
1.1 cargo build              # debug 构建（快，无优化，溢出 panic）
1.2 cargo build --release    # release 构建（慢，优化，溢出回绕）
1.3 cargo run --release      # release 模式运行
```

★ **关键认知**：release 构建开启 `opt-level=3` 优化，运行速度可比 debug 快数十倍。发布产品、基准测试必须用 `--release`。⚠ debug/release 的整数溢出行为不同（debug panic、release 回绕），这是常见的行为差异陷阱。

## 1.1 自定义 profile

```toml
[profile.release]
opt-level = 3        # 优化级别（0-3, "s" 尺寸, "z" 极致尺寸）
lto = true           # 链接时优化（跨 crate 优化，编译慢但快）
codegen-units = 1    # 减少并行代码单元，优化更好
panic = "abort"      # panic 直接终止，体积更小
strip = true         # 去除符号
```

★ **关键认知**：`lto`（链接时优化）能让编译器跨 crate 边界内联优化，配合 `codegen-units = 1` 可榨取极致性能，代价是编译时间显著变长。是"性能 vs 编译速度"的取舍。

# 2. crate

## 2.1 crate 类型

```
2.1 binary crate（二进制 crate）
    - 有 src/main.rs，包含 fn main()
    - 编译为可执行文件
2.2 library crate（库 crate）
    - 有 src/lib.rs
    - 编译为 .rlib，供其他 crate 依赖
```

★ **关键认知**：一个 package 可以同时包含 library crate（`src/lib.rs`）和 binary crate（`src/main.rs`），甚至多个 binary（`src/bin/xxx.rs`）。binary crate 可以依赖同 package 的 library crate。

## 2.2 crate 依赖声明

```toml
[dependencies]
serde = "1.0"                                  # 版本范围 ^1.0
rand = { version = "0.8", features = ["std"] } # 指定 feature
local = { path = "../local" }                  # 本地路径依赖
gitdep = { git = "https://github.com/x/y" }    # git 依赖

[dev-dependencies]                             # 仅测试/示例
criterion = "0.5"

[build-dependencies]                           # build.rs 使用
cc = "1.0"
```

```rust
// 使用依赖
use serde::Serialize;
```

★ **关键认知**：`[dependencies]` 分三类：普通依赖、`[dev-dependencies]`（仅测试/示例/bench）、`[build-dependencies]`（仅构建脚本 build.rs）。这隔离了不同阶段的依赖。crate 来自 crates.io（默认）、本地路径或 git。

# 3. pub use —— 重导出

```rust
// 内部模块层次
mod inner {
    pub struct PublicType;
}

// 在 crate 根重导出
pub use inner::PublicType;
```

★ **关键认知**：`pub use` 把内部的项**重新公开**到更浅的位置，让使用者可以直接 `use mycrate::PublicType`，而无须知道它在 `inner` 中。这是设计"外观整洁、内部模块化"的 API 的常用手段，也支持 API 演进（内部结构变了，对外路径不变）。

# 4. cargo workspace

## 4.1 创建 workspace

```toml
# 根 Cargo.toml
[workspace]
members = [
    "crates/core",
    "crates/api",
    "crates/cli",
]

[workspace.dependencies]           # ★ 统一管理依赖版本
serde = "1.0"
```

```
目录结构：
workspace/
├── Cargo.toml          # workspace 清单
├── Cargo.lock          # ★ 整个 workspace 共享一个锁文件
├── crates/
│   ├── core/
│   │   └── Cargo.toml
│   ├── api/
│   └── cli/
```

★ **关键认知**：workspace 让多个相关 crate 共享一个 `Cargo.lock`、一个 `target/` 目录（共享构建缓存）、统一的依赖版本。成员间可用 `path` 依赖。它解决"多个 crate 协同开发"的一致性问题。

## 4.2 workspace 操作

```bash
cargo build -p core       # 只构建某个成员
cargo test --workspace    # 测试所有成员
cargo run -p cli          # 运行某个成员
```

★ **关键认知**：`-p <package>` 指定操作某个成员。`[workspace.dependencies]` 让所有成员引用同一版本（成员用 `serde = { workspace = true }`），避免版本分裂。

# 5. install binary crate

```bash
cargo install ripgrep        # 从 crates.io 安装二进制工具
cargo install --path .       # 从本地目录安装
cargo install --list         # 列出已安装的工具
cargo uninstall ripgrep
```

★ **关键认知**：`cargo install` 把二进制 crate 编译并安装到 `~/.cargo/bin`（需在 PATH 中）。这是安装 Rust 命令行工具的标准方式（如 `ripgrep`、`fd`、`bat`、`cargo-watch`）。它与 Go 的 `go install` 类似。

# 6. 小结

| 概念 | 要点 |
|------|------|
| release | -O 优化，发布必用 |
| profile | 自定义优化/lto/panic 策略 |
| crate 类型 | binary / library |
| 依赖分类 | dependencies / dev / build |
| pub use | 重导出，清爽 API |
| workspace | 多 crate 共享锁与缓存 |
| cargo install | 安装二进制工具 |

下一篇进入智能指针，讲清楚 `Box<T>`、`Deref`/`Drop` trait、`Rc<T>` 共享所有权、`RefCell<T>` 内部可变性。
