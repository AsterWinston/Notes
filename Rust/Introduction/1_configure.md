Rust 的工具链由三部分组成：`rustup`（版本管理器）、`cargo`（构建系统与包管理器）、`rustc`（编译器）。理解这三者的分工与配置，是搭建 Rust 开发环境的第一步。

# 1. 安装 Rust 编译器

## 1.1 rustup —— 官方推荐安装方式

Rust 官方推荐用 `rustup` 安装，它同时是工具链版本管理器：

```
1.1 下载 rustup-init（Windows 是 rustup-init.exe）
1.2 设置环境变量 CARGO_HOME 和 RUSTUP_HOME
    - 这两个是 rustup 依赖的，执行 rustup-init 时据此确定安装位置
    - 若不设置，则安装到默认位置：<user_name>\.cargo 和 <user_name>\.rustup
1.3 运行 rustup-init 进行安装
```

★ **关键认知**：`rustup` 不只是安装器，更是**工具链管理器**。它允许你同时安装多个版本的 Rust（stable/beta/nightly）、多个目标平台，并按项目切换。`cargo`/`rustc` 实际上是 rustup 管理的"代理"——它们最终调用真正安装的工具链。

## 1.2 关键环境变量

```
CARGO_HOME     cargo 的根目录（默认 <user>\.cargo），存放 registry 缓存、配置
RUSTUP_HOME    rustup 的根目录（默认 <user>\.rustup），存放各版本工具链
PATH           需包含 %CARGO_HOME%\bin，才能调用 cargo/rustc
```

★ **关键认知**：与 Go 的 `GOPATH`/`GOROOT` 类似，`CARGO_HOME` 存放依赖缓存（下载的 crate），`RUSTUP_HOME` 存放工具链本体。理解这两个目录，能解释"依赖下到哪、工具链装在哪"。

## 1.3 国内镜像加速

```
在 %CARGO_HOME%\config.toml 中配置：

[source.crates-io]
replace-with = 'ustc'

[source.ustc]
registry = "sparse+https://mirrors.ustc.edu.cn/crates.io-index/"
```

★ **关键认知**：Rust 的依赖（crate）从 crates.io 下载，国内直连慢。通过源替换（`replace-with`）镜像到中科大/字节等镜像源，可显著加速。`sparse+` 前缀表示使用稀疏索引（新版 cargo 默认，比 git 索引快很多）。

# 2. rustup 常用命令

```
2.1 rustup show              # 显示当前工具链与目标平台
2.2 rustup install nightly   # 安装指定版本工具链
2.3 rustup default stable    # 设置默认工具链
2.4 rustup update            # 更新所有工具链
2.5 rustup component add ... # 添加组件（如 rustfmt、clippy）
2.6 rustup target add ...    # 添加交叉编译目标
2.7 rustup self uninstall    # 卸载 Rust
```

★ **关键认知**：Rust 有 **stable / beta / nightly** 三条发布通道，每 6 周发布一个新 stable。nightly 才能用实验性特性（如 `#![feature(...)]`），生产项目应使用 stable。`rustup override` 还能为单个项目指定工具链（通过 `rust-toolchain.toml`）。

# 3. rustc —— 编译器

## 3.1 直接编译单文件

```
3.1 rustc main.rs            # 编译为可执行文件
3.2 rustc --version          # 查看编译器版本
3.3 rustc -O main.rs         # 开启优化
```

## 3.2 rustc 的产物

```
rustc main.rs  →  main（Windows 为 main.exe）+ 中间产物

流程：源码 → 词法/语法分析 → HIR → MIR → LLVM IR → 机器码
```

★ **关键认知**：`rustc` 是真正的编译器前端，后端用 **LLVM** 生成机器码。`HIR`（高阶 IR）和 `MIR`（中阶 IR）是 Rust 独有的中间表示——借用检查（borrow check）就发生在 MIR 阶段。日常开发几乎不直接用 rustc，而是通过 cargo 间接调用（见下）。

# 4. cargo —— 构建系统与包管理器

## 4.1 常用命令

```
4.1 cargo new <project_name>           # 新建项目（默认二进制 crate）
4.2 cargo new --lib <name>             # 新建库 crate
4.3 cargo build                        # 编译（debug 模式）
4.4 cargo build --release              # 编译（release 模式，开启优化）
4.5 cargo run                          # 编译并运行
4.6 cargo run --release                # release 模式运行
4.7 cargo check                        # ★ 只做类型检查，不生成机器码（快）
4.8 cargo update                       # 更新依赖版本（遵循 Cargo.lock）
4.9 cargo test                         # 运行测试
4.10 cargo doc --open                  # 生成并打开文档
```

★ **关键认知**：`cargo check` 是 Rust 开发中使用最频繁的命令——它只做类型检查和借用检查，不生成机器码，速度比 `cargo build` 快数倍。日常"改代码 → 看能否编译通过"应该用 `cargo check`，而不是 `cargo build`。

## 4.2 debug vs release

```
debug（cargo build）：
  - 开启调试符号，关闭优化
  - 编译快，运行慢
  - 整数溢出会 panic（便于发现问题）

release（cargo build --release）：
  - 开启优化（opt-level=3），去除调试符号
  - 编译慢，运行快
  - 整数溢出静默回绕（wrapping）
```

★ **关键认知**：`debug` 和 `release` 的行为差异不只在性能——**整数溢出**在 debug 下 panic、release 下回绕（wrapping）。这是 Rust 安全设计的一部分：开发期帮你发现溢出 bug，发布期保证性能。

## 4.3 Cargo.toml 与 Cargo.lock

```
Cargo.toml    项目的清单文件，声明依赖和元数据（人写）
Cargo.lock    精确锁定依赖版本与哈希（cargo 自动生成）
```

★ **关键认知**：`Cargo.toml` 是"声明式"的（我要哪些依赖、什么版本范围），`Cargo.lock` 是"锁定式"的（实际解析出的精确版本）。应用项目应提交 `Cargo.lock`（保证可复现构建）；库项目通常不提交（让下游自由选择兼容版本）。

## 4.4 Cargo.toml 结构

```toml
[package]
name = "myapp"
version = "0.1.0"
edition = "2021"          # ★ edition 不是版本，是"语法纪元"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
rand = "0.8"              # 等价于 ^0.8（兼容 0.8.x）

[dev-dependencies]        # 仅测试/示例使用
criterion = "0.5"

[profile.release]         # 自定义编译配置
opt-level = 3
lto = true                # 链接时优化
```

★ **关键认知**：`edition` 是 Rust 的独特设计——它不是编译器版本，而是"**语法纪元**"（2015/2018/2021/2024）。不同 edition 可以用不同的语法规则，但能在同一个项目里互相调用。这解决了"语言演进不能破坏已有代码"的难题：老代码留在老 edition，新代码用新 edition。

# 5. 小结

| 工具 | 职责 | 一句话记忆 |
|------|------|-----------|
| rustup | 工具链管理 | 装/切/更新 Rust 版本 |
| rustc | 编译器 | 前端 + LLVM 后端 |
| cargo | 构建与包管理 | 95% 的命令都通过它 |
| cargo check | 类型检查 | 最快的"编译通过"验证 |
| edition | 语法纪元 | 语言演进不破坏旧代码 |

下一篇进入变量、数据类型与循环，讲清楚 Rust 的不可变绑定、变量遮蔽（shadowing）、标量与复合类型，以及 `if`/`match`/循环表达式。
