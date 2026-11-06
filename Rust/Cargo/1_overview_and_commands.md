很多初学者第一次接触 Rust，会被 `cargo build`、`cargo run`、`cargo check` 这一堆命令搞晕：到底什么时候用哪个？产物去了哪里？为什么改一行代码要等半天？本篇把 Cargo 的定位、日常命令和构建产物讲清楚，让你从"照着教程敲命令"升级到"知道每条命令在干什么"。

# 1. Cargo 是什么

## 1.1 工具链的四件套分工

Rust 官方工具链里有四个角色，理解它们的分工是理解 Cargo 的前提：

| 工具 | 职责 | 类比 |
|------|------|------|
| `rustup` | 安装、切换、更新工具链 | nvm / pyenv |
| `rustc` | 真正的编译器 | gcc / clang |
| `cargo` | 构建系统 + 包管理器 | make + npm |
| `rustfmt` / `clippy` | 格式化 / 静态检查 | prettier / eslint |

日常开发里你几乎只跟 `cargo` 打交道，它背后替你调用 `rustc`、管理依赖、组织构建流程。

★ **关键认知**：`cargo` 不是编译器，它是**构建系统的编排者**。你敲 `cargo build` 时，Cargo 做的是：读取 `Cargo.toml` → 解析依赖 → 下载并编译依赖 → 生成 `rustc` 命令 → 调用 `rustc` 编译你的代码 → 链接产物。真正把 `.rs` 变成机器码的是 `rustc`，Cargo 负责"什么时候、按什么顺序、带什么参数"调用它。

## 1.2 Cargo 解决的三个痛点

在没有 Cargo 的年代（或直接裸用 `rustc`），你需要自己处理：

```text
1. 依赖管理：手动下载 crate、手动指定 --extern 路径、手动处理版本冲突
2. 构建流程：手写编译顺序、手动链接、手动区分 debug/release
3. 项目组织：多文件、多 crate、多目标平台，全靠脚本拼接
```

Cargo 把这三件事统一成"声明式配置 + 一条命令"：依赖写进 `Cargo.toml`，构建配置写进 profile，项目组织写进 workspace。这是 Rust 生态体验好的核心原因之一。

★ **关键认知**：Cargo 的设计哲学是**约定优于配置**。`src/main.rs` 就是二进制入口，`src/lib.rs` 就是库入口，`tests/` 就是集成测试——这些约定让你新建项目后什么都不用配就能跑。⚠ 想打破约定当然可以（用 `[[bin]]`、`[lib]` 显式声明），但绝大多数项目不需要。

# 2. 第一个项目

## 2.1 创建项目

```bash
cargo new hello_cargo          # 新建二进制项目
cargo new --lib my_lib         # 新建库项目
cargo new hello_cargo --vcs none   # 不初始化 git 仓库
cargo init                     # 在已有目录里初始化
```

`cargo new` 默认会初始化一个 git 仓库（如果当前不在仓库内），并生成 `.gitignore`。

## 2.2 目录结构

```text
hello_cargo/
├── Cargo.toml          # 项目清单：名字、版本、依赖
├── Cargo.lock          # 精确锁定依赖版本（自动生成）
├── .gitignore          # 默认忽略 /target
└── src/
    └── main.rs         # 二进制入口，含 fn main()
```

库项目则是 `src/lib.rs`，没有 `main` 函数。一个 package 可以同时有 `src/main.rs` 和 `src/lib.rs`，此时二进制可以复用库里的代码。

★ **关键认知**：`Cargo.toml` 是**人写的声明**，`Cargo.lock` 是**机器生成的锁定**。前者说"我要 `serde` 1.x"，后者记录"这次实际用的是 `serde 1.0.210`"。⚠ 应用项目要提交 `Cargo.lock`（保证队友和你装到一模一样的版本），库项目通常不提交（让下游自己选兼容版本）。

# 3. 常用命令

## 3.1 build 与 run

```bash
cargo build              # debug 构建，产物在 target/debug/
cargo build --release    # release 构建，产物在 target/release/
cargo run                # 构建并运行（debug）
cargo run --release      # 构建并运行（release）
cargo run -- arg1 arg2   # 给程序传参，注意 -- 分隔
```

`cargo run -- xxx` 里的 `--` 很关键：它把 Cargo 自己的参数和程序的参数分开。`cargo run --release` 中的 `--release` 是给 Cargo 的，`cargo run -- --release` 中的 `--release` 才是给程序的。

## 3.2 check：开发期最快的验证

```bash
cargo check              # 只做类型检查 + 借用检查，不生成机器码
cargo check --all-targets    # 连测试、示例、bench 一起检查
cargo check --workspace      # 检查整个 workspace
```

★ **关键认知**：`cargo check` 是 Rust 开发中**使用频率最高**的命令。它跳过代码生成和链接，只跑到类型检查阶段，速度通常比 `cargo build` 快 3-10 倍。改完代码想确认"能不能编译过"，用 `check`；要真正运行或产出二进制，才用 `build`。⚠ `check` 不会发现链接错误和部分代码生成阶段的错误，所以最终交付前仍要跑一次 `build`。

## 3.3 test 与 bench

```bash
cargo test               # 运行单元测试 + 集成测试 + 文档测试
cargo test -- --nocapture    # 显示测试中的 println 输出
cargo test some_name     # 只跑名字匹配 some_name 的测试
cargo bench              # 运行基准测试（需 nightly 或 criterion）
```

## 3.4 fmt 与 clippy

```bash
cargo fmt                # 按官方风格格式化所有代码
cargo fmt -- --check     # 只检查不修改，适合 CI
cargo clippy             # 运行 lint 检查，发现常见错误写法
cargo clippy -- -D warnings   # 把警告当错误，CI 常用
```

★ **关键认知**：`cargo clippy` 不是可选项而是**团队规范的一部分**。它能抓出 `if x == true`、`vec.len() == 0`、不必要的 `clone()` 这类问题。在 CI 里跑 `clippy -- -D warnings`，能让代码质量保持在一条线上。⚠ clippy 会随版本更新新增 lint，升级工具链后可能突然报出新警告，需要预留处理时间。

## 3.5 doc 与其它常用命令

```bash
cargo doc --open             # 生成文档并在浏览器打开
cargo doc --no-deps          # 只生成自己 crate 的文档
cargo add serde              # 添加依赖（自动写 Cargo.toml）
cargo remove serde           # 移除依赖
cargo update                 # 在版本范围内更新依赖
cargo tree                   # 打印依赖树
cargo metadata --format-version 1   # 输出机器可读的项目元数据
```

`cargo add` / `cargo remove` 是 1.62 之后内置的子命令，以前要靠 `cargo-edit` 插件。

## 3.6 命令速查表

| 命令 | 用途 | 何时用 |
|------|------|--------|
| `cargo check` | 类型检查 | 改代码后快速验证 |
| `cargo build` | 生成二进制 | 要运行/交付 |
| `cargo run` | 构建 + 运行 | 本地调试 |
| `cargo test` | 跑测试 | 提交前、TDD |
| `cargo fmt` | 格式化 | 提交前 |
| `cargo clippy` | 静态检查 | 提交前、CI |
| `cargo doc` | 生成文档 | 写库、查 API |
| `cargo tree` | 依赖树 | 排查版本冲突 |

# 4. target 目录与构建产物

## 4.1 产物布局

```text
target/
├── debug/                  # debug 构建产物
│   ├── hello_cargo         # 可执行文件（Windows 为 hello_cargo.exe）
│   ├── deps/               # 依赖的 .rlib / .rmeta 等中间产物
│   ├── incremental/        # 增量编译缓存
│   └── build/              # build.rs 的输出目录
└── release/                # release 构建产物
    └── ...
```

★ **关键认知**：`target/` 是可以随时删除的。它只是构建缓存 + 产物，删掉后下次构建会重新生成。⚠ 但它往往有几个 GB，磁盘紧张时 `cargo clean` 或直接删 `target/` 是最快的瘦身手段；`cargo clean` 会删整个目录，`cargo clean -p 包名` 只清某个包。

## 4.2 增量编译

Cargo 默认开启增量编译（`incremental = true`），把上次编译的中间结果存在 `target/debug/incremental/`，下次只重编改动的部分。

```toml
[profile.dev]
incremental = true      # 默认就是 true，一般不用改
```

★ **关键认知**：增量编译让"改一行、编一次"从几十秒降到几秒，但它**依赖时间戳和文件状态**。如果构建行为诡异（明明改了却没生效），`cargo clean` 后重建往往能解决。⚠ release profile 默认关闭增量编译，因为优化后的增量结果收益不大且容易不一致。

# 5. 工具链与版本

## 5.1 查看版本

```bash
cargo --version     # cargo 1.7x.x
rustc --version     # rustc 1.7x.x
rustup show         # 当前工具链、默认目标平台
```

Cargo 与 rustc 版本是绑定的，由同一个 rustup 工具链提供，单独升级其中一个没有意义。

## 5.2 为项目固定工具链

```toml
# rust-toolchain.toml，放在项目根目录
[toolchain]
channel = "1.75.0"          # 固定版本，团队统一
components = ["rustfmt", "clippy"]
targets = ["x86_64-unknown-linux-gnu"]
```

★ **关键认知**：`rust-toolchain.toml` 让"进入这个目录就自动切到指定工具链"，解决"我这能编、你那编不过"的版本漂移问题。⚠ 它会让 rustup 在你第一次进入目录时尝试下载对应工具链，离线环境下可能卡住，需要提前 `rustup toolchain install`。

# 6. 常见错误

```text
错误 1：error: could not find `Cargo.toml` in ...
原因：不在项目根目录（或子目录），Cargo 靠向上查找 Cargo.toml 定位项目
解决：cd 到项目根目录，或确认 Cargo.toml 存在

错误 2：error: failed to select a version for the requirement ...
原因：依赖版本范围找不到满足条件的版本，常见于手写了一个不存在的版本号
解决：放宽版本约束，或 cargo search / crates.io 查真实版本

错误 3：error: linker `cc` not found
原因：缺少 C 链接器（Windows 上常见于没装 MSVC 构建工具）
解决：安装 Visual Studio Build Tools 的 C++ 组件，或用 GNU 工具链

错误 4：warning: unused variable: `x`
原因：变量没用，是警告不是错误（除非开了 -D warnings）
解决：用下划线前缀 `_x`，或真的去使用它
```

# 7. 练习

1. 用 `cargo new` 建一个项目，把 `main.rs` 改成打印命令行参数（`std::env::args()`），用 `cargo run -- a b c` 观察传参。
2. 故意制造一个类型错误，分别跑 `cargo check` 和 `cargo build`，对比两者的输出和耗时。
3. 在项目里写一个 `tests/` 下的集成测试，用 `cargo test` 跑通，再试 `cargo test 名字` 只跑单个测试。

# 8. 小结

| 概念 | 要点 |
|------|------|
| Cargo 定位 | 构建系统 + 包管理器，编排 rustc |
| `check` | 只类型检查，开发期最快验证 |
| `build` | 生成二进制，区分 debug/release |
| `run --` | `--` 后才是程序参数 |
| `fmt` / `clippy` | 格式化和 lint，CI 必跑 |
| `target/` | 可删的构建缓存与产物 |
| `rust-toolchain.toml` | 固定项目工具链，消除版本漂移 |

下一篇进入 `Cargo.toml` 与依赖管理，讲清楚版本号的语义、可选依赖、dev/build 依赖的区别，以及 path/git 依赖的用法。
