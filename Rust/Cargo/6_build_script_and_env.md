有些 crate 的构建不只是"编译 Rust 代码"：它要调用 C 编译器编译一段 C 代码、要读取系统库的位置、要根据配置文件生成 Rust 源码。这些活儿都交给 **build script（`build.rs`）**。它是 Cargo 在编译你的 crate 之前执行的一段 Rust 程序，本篇讲清楚它什么时候跑、怎么和 Cargo 通信、以及那些"改了文件却不重编"的坑。

# 1. build.rs 是什么

## 1.1 最小示例

```text
项目结构：
mycrate/
├── Cargo.toml
├── build.rs          # 构建脚本，自动被 Cargo 识别
└── src/
    └── lib.rs
```

```rust
// build.rs：编译期执行的普通 Rust 程序
fn main() {
    println!("cargo:warning=正在构建 mycrate");   // 输出警告到 cargo 日志
    println!("cargo:rerun-if-changed=build.rs");  // 只在 build.rs 变化时重跑
}
```

★ **关键认知**：`build.rs` 是**编译期运行的独立程序**，不是你的库代码的一部分。它编译成宿主机（host）的程序，在你的 crate 编译之前执行；它不能 `use` 你的库，只能 `use` `[build-dependencies]` 里的东西。⚠ 它运行在**构建机**上，不是目标机上——交叉编译时这一点尤其重要。

## 1.2 执行时机

```text
cargo build 的执行顺序：

  1. 解析依赖
  2. 编译 build-dependencies
  3. 编译并运行 build.rs（host 程序）
       │
       ├─ 输出 cargo:xxx 指令，告诉 Cargo 该做什么
       └─ 可写文件到 OUT_DIR
  4. 编译你的 crate（此时 OUT_DIR 可供 include!）
  5. 链接
  6. 产出二进制
```

★ **关键认知**：`build.rs` 在**你的 crate 之前**执行，所以它能生成源码、生成链接参数、探测系统环境，把结果通过 `OUT_DIR` 或 `cargo:` 指令交给后续编译。⚠ 它不会在每次 `cargo build` 都跑——Cargo 会缓存结果，只有满足重跑条件（见后文 `rerun-if-changed`）才会重新执行。

# 2. 与 Cargo 通信：cargo: 指令

## 2.1 常用指令

| 指令 | 作用 |
|------|------|
| `cargo:rerun-if-changed=PATH` | PATH 变化时重跑 build.rs |
| `cargo:rerun-if-env-changed=VAR` | 环境变量变化时重跑 |
| `cargo:rustc-link-lib=NAME` | 链接一个库（`-l NAME`） |
| `cargo:rustc-link-search=PATH` | 添加库搜索路径（`-L PATH`） |
| `cargo:rustc-cfg=KEY` | 给 rustc 加 `--cfg KEY` |
| `cargo:rustc-env=KEY=VALUE` | 设置编译期环境变量 |
| `cargo:warning=MESSAGE` | 打印警告（会显示在构建日志） |
| `cargo:metadata=KEY=VALUE` | 传递元数据给下游（配合 `links`） |

## 2.2 输出格式

```rust
// 每条指令一行，必须以 cargo: 开头，println 到 stdout
fn main() {
    println!("cargo:rustc-link-search=native=/usr/local/lib");
    println!("cargo:rustc-link-lib=static=foo");
    println!("cargo:rustc-cfg=has_foo");
    println!("cargo:warning=使用系统库 foo");
}
```

★ **关键认知**：build.rs 的**标准输出被 Cargo 解析为指令**，所以不要在 build.rs 里 `println!` 调试信息——那会被当成指令（无法识别时会警告）。要输出调试信息，用 `eprintln!`（stderr）或 `cargo:warning=`。⚠ 这个坑非常隐蔽：你 `println!("hello")` 后构建日志里多一条莫名其妙的警告，就是因为 Cargo 把 `hello` 当成未知指令。

# 3. OUT_DIR 与代码生成

## 3.1 拿到 OUT_DIR

```rust
// build.rs
use std::env;
use std::fs;
use std::path::Path;

fn main() {
    let out_dir = env::var("OUT_DIR").unwrap();   // Cargo 提供的输出目录
    let dest = Path::new(&out_dir).join("generated.rs");

    let code = r#"
pub fn generated_hello() -> &'static str {
    "来自 build.rs 生成的代码"
}
"#;
    fs::write(&dest, code).unwrap();

    println!("cargo:rerun-if-changed=build.rs");
}
```

```rust
// src/lib.rs
// OUT_DIR 是通过 include! 使用的编译期路径
include!(concat!(env!("OUT_DIR"), "/generated.rs"));

pub fn call_generated() -> &'static str {
    generated_hello()
}
```

★ **关键认知**：`OUT_DIR` 是 Cargo 为**每次构建**分配的目录（类似 `target/debug/build/mycrate-哈希/out`），路径不固定，所以必须用 `env!("OUT_DIR")` 在编译期拼接，不能硬编码。⚠ `include!` 是在**编译期**把生成的文件嵌入，所以生成的文件必须在 rustc 编译你的 crate 之前就存在——这正是 build.rs 先执行的原因。

★ **关键认知**：代码生成不一定非要用 build.rs。如果生成逻辑依赖的是"语法结构"而非"构建环境"，过程宏（proc-macro）更合适；build.rs 适合"编译前必须完成的预处理"（如解析 `.proto`、生成 FFI 绑定）。⚠ 用 build.rs 生成的代码在 IDE 里通常无法跳转，调试体验差，能用宏就用宏。

# 4. 链接系统库

## 4.1 手动链接

```rust
// build.rs
fn main() {
    // 告诉链接器去 /usr/local/lib 找库
    println!("cargo:rustc-link-search=native=/usr/local/lib");
    // 链接静态库 libfoo.a（-l static=foo）
    println!("cargo:rustc-link-lib=static=foo");
    // 链接动态库 libbar.so（-l dylib=bar）
    println!("cargo:rustc-link-lib=dylib=bar");
}
```

```text
链接器实际收到的参数：-L native=/usr/local/lib、-l static=foo、-l dylib=bar
```

## 4.2 用 cc crate 编译 C 代码

```toml
[build-dependencies]
cc = "1.0"
```

```rust
// build.rs：把 src/foo.c 编译成静态库并链接
fn main() {
    cc::Build::new()
        .file("src/foo.c")
        .include("include")          // 头文件目录
        .flag("-Wall")               // 传给 C 编译器的参数
        .compile("foo");             // 产出 libfoo.a 并自动加链接指令

    println!("cargo:rerun-if-changed=src/foo.c");
    println!("cargo:rerun-if-changed=include/foo.h");
}
```
```rust
// src/lib.rs：声明外部 C 函数
extern "C" {
    fn foo_add(a: i32, b: i32) -> i32;   // 对应 foo.c 里的实现
}

pub fn add(a: i32, b: i32) -> i32 {
    unsafe { foo_add(a, b) }             // 调用 C 函数必须 unsafe
}
```

★ **关键认知**：`cc` crate 封装了"找 C 编译器、加参数、编译、生成链接指令"的全过程，是 Rust FFI 生态的事实标准。⚠ 它依赖宿主机上有可用的 C 编译器（`cc` / `clang` / MSVC），交叉编译时还要配 `CC_<target>` 环境变量指向目标平台的交叉编译器。

## 4.3 links 字段

```toml
[package]
name = "foo-sys"
links = "foo"          # 声明这个 crate 链接了原生库 foo
```

```rust
// 下游可以用 DEP_FOO_<KEY> 读取上游 build.rs 的 metadata
// build.rs 里：println!("cargo:metadata=version=1.2.3");
```

★ **关键认知**：`links` 字段保证**同一个原生库在整个依赖图里只被一个 crate 链接**，避免重复链接冲突。它还会让 Cargo 把上游 build.rs 的 `cargo:metadata` 以 `DEP_<LINKS>_<KEY>` 形式传给下游。⚠ 只有 `*-sys` 这类绑定原生库的 crate 才需要 `links`，纯 Rust 库不要写。

# 5. 环境变量

## 5.1 Cargo 提供的编译期变量

| 变量 | 含义 |
|------|------|
| `OUT_DIR` | 构建输出目录 |
| `CARGO_MANIFEST_DIR` | 当前 crate 的 `Cargo.toml` 所在目录 |
| `CARGO_PKG_NAME` | 包名 |
| `CARGO_PKG_VERSION` | 包版本 |
| `CARGO_PKG_VERSION_MAJOR` / `MINOR` / `PATCH` | 版本各段 |
| `TARGET` | 目标三元组（如 `x86_64-unknown-linux-gnu`） |
| `HOST` | 宿主三元组 |
| `PROFILE` | `debug` 或 `release` |
| `OPT_LEVEL` | 优化级别 |
| `NUM_JOBS` | 并行任务数 |

## 5.2 在代码里读取包信息

```rust
// 编译期字符串，零运行时开销
pub const VERSION: &str = env!("CARGO_PKG_VERSION");
pub const NAME: &str = env!("CARGO_PKG_NAME");

pub fn print_info() {
    println!("{} v{}", NAME, VERSION);
}
```

★ **关键认知**：`env!` 是**编译期**读取环境变量，变量不存在会编译失败；`option_env!` 则返回 `Option<&'static str>`，不存在时为 `None`。而 `std::env::var` 是**运行时**读取，两者语义完全不同。⚠ 想读 `OUT_DIR` 这类仅编译期存在的变量，必须用 `env!` / `option_env!`，运行时读不到。

# 6. rerun-if-changed：最常被忽略的一行

## 6.1 默认行为

```text
不写 rerun-if-changed 时：
  Cargo 默认监听"整个 crate 目录"（除 target/）
  → 任何文件变化都会重跑 build.rs
  → 构建变慢，且容易意外触发
```

```rust
// 显式声明依赖的文件
println!("cargo:rerun-if-changed=build.rs");
println!("cargo:rerun-if-changed=proto/api.proto");
println!("cargo:rerun-if-changed=assets/config.toml");
```

★ **关键认知**：一旦你写了**任何一条** `rerun-if-changed`，Cargo 就**不再**默认监听整个目录，只监听你列出的路径。⚠ 这是最经典的坑：加了 `rerun-if-changed=build.rs` 却忘了列 `proto/*.proto`，结果改了 proto 文件却不重新生成代码，调试半天以为是缓存 bug。

## 6.2 监听目录

```rust
println!("cargo:rerun-if-changed=src");        // 监听整个 src 目录
println!("cargo:rerun-if-changed=proto");      // 监听 proto 目录（递归）
```

```rust
// 环境变量变化也要监听
println!("cargo:rerun-if-env-changed=MY_BUILD_FLAG");
```

## 6.3 推荐写法

```rust
fn main() {
    println!("cargo:rerun-if-changed=build.rs");   // 自己
    println!("cargo:rerun-if-changed=proto");      // 输入资源
    println!("cargo:rerun-if-env-changed=PROTOC"); // 影响构建的环境变量
}
```

# 7. 常见错误

```text
错误 1：改了源文件但 build.rs 生成的内容没更新
原因：写了 rerun-if-changed=build.rs，却没监听真正的输入文件
解决：把所有输入路径列进 rerun-if-changed

错误 2：build.rs 里 println! 调试，结果出现奇怪警告
原因：stdout 被 Cargo 当指令解析
解决：调试用 eprintln!，要显示给用户用 cargo:warning=

错误 3：error: failed to run custom build command for `foo-sys`
原因：build.rs 里探测系统库失败、或 C 编译器缺失
解决：看完整 stderr（cargo build -vv），装对应系统库/工具链

错误 4：交叉编译时链接到宿主机的库
原因：build.rs 用了宿主机路径或系统库探测
解决：用 TARGET 环境变量区分目标平台，配交叉编译工具链

错误 5：include!(concat!(env!("OUT_DIR"), ...)) 报文件不存在
原因：build.rs 没生成该文件，或生成路径拼接错
解决：确认 build.rs 执行成功、文件名一致

错误 6：下游拿不到 DEP_XXX 变量
原因：上游没有 links 字段，或没输出 cargo:metadata
解决：上游加 links，下游用 DEP_<LINKS>_<KEY> 读取
```

# 8. 练习

1. 写一个 build.rs，读取 `CARGO_PKG_VERSION`，生成一个 `version.rs`，在库代码里 `include!` 并打印版本号。
2. 用 `cc` crate 编译一个 `add.c`，在 Rust 里通过 `extern "C"` 调用，验证 `cargo run` 输出正确结果。
3. 故意只写 `rerun-if-changed=build.rs`，改一个输入文件后观察 build.rs 不重跑；再补上输入路径验证修复。

# 9. 小结

| 概念 | 要点 |
|------|------|
| build.rs | 编译期执行的 host 程序，先于 crate 编译 |
| 通信方式 | stdout 的 `cargo:` 指令 |
| OUT_DIR | 每次构建独立的输出目录，用 `env!` 拼接 |
| 代码生成 | build.rs 写文件 + `include!` 嵌入 |
| 链接库 | `rustc-link-search` + `rustc-link-lib`，或用 `cc` |
| `links` | 保证原生库只链一次，传递 metadata |
| 编译期变量 | `env!` / `option_env!`，与运行时 `env::var` 不同 |
| `rerun-if-changed` | 写了就只监听列出的路径，漏写是经典坑 |

下一篇进入发布流程，讲清楚 crates.io 的版本语义、`cargo package` / `cargo publish` 的细节、文档规范，以及 yank 与版本回退的正确姿势。
