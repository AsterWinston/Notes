Cargo 的 feature 是 Rust 生态里最强大也最容易踩坑的机制之一：它让你按需编译代码、裁剪依赖，但"feature 是加法"这条规则一旦被违反，整个依赖图都会出问题。本篇从 feature 的定义讲到 resolver 2 的解析规则，再讲 feature 冲突的成因与规避。

# 1. feature 是什么

## 1.1 最小示例

```toml
# Cargo.toml
[features]
default = ["json"]              # 默认启用的 feature 集合
json = ["dep:serde_json"]       # 启用它，就引入 serde_json 依赖
extra = []                      # 纯开关，不关联依赖
```

```rust
// src/lib.rs
#[cfg(feature = "json")]
pub fn to_json<T: serde::Serialize>(v: &T) -> String {
    serde_json::to_string(v).unwrap()   // 只有启用 json 时才编译
}

#[cfg(feature = "extra")]
pub fn extra_api() { /* ... */ }
```

```bash
cargo build                      # 启用默认 feature（json）
cargo build --no-default-features   # 一个 feature 都不开
cargo build --features extra     # 在默认基础上追加 extra
cargo build --no-default-features --features json   # 只要 json
```

★ **关键认知**：feature 是**编译期的条件开关**，不是运行时的配置。`#[cfg(feature = "...")]` 在编译前就把不满足条件的代码整段剔除，所以未启用的 feature 对应的代码**根本不会进入二进制**，也不会参与类型检查。⚠ 这意味着未启用 feature 的代码可能悄悄腐坏（长期没人编译），CI 要覆盖多组 feature 组合。

## 1.2 feature 的三种作用

| 作用 | 写法 | 说明 |
|------|------|------|
| 条件编译 | `#[cfg(feature = "x")]` | 控制代码是否参与编译 |
| 引入依赖 | `x = ["dep:foo"]` | 开关依赖是否被编译 |
| 传递开关 | `x = ["other/x"]` | 启用依赖 other 的 x feature |

# 2. 默认 feature

## 2.1 default 的语义

```toml
[features]
default = ["std"]               # 绝大多数库的默认配置
std = []
alloc = []
```

```bash
cargo build                          # 等价于 --features default
cargo build --no-default-features    # 关闭 default
```

★ **关键认知**：`default` 只是一个普通 feature 名，Cargo 约定"没显式指定时启用名为 `default` 的 feature"。它没有任何特殊语法地位。⚠ 但下游可以 `default-features = false` 关掉它，所以**你的 API 必须在无默认 feature 时也能编译**，否则下游想裁剪依赖就会失败。

## 2.2 库作者的最佳实践

```toml
[dependencies]
serde = { version = "1.0", optional = true, default-features = false }

[features]
default = ["std"]
std = ["serde?/std"]            # serde? 表示"如果 serde 已启用，才启用它的 std"
alloc = ["serde?/alloc"]

# no_std 支持：默认关掉 std，但保留 alloc
```

```text
推荐组合：
  default = ["std"]
  std     = ["alloc"]
  alloc   = []
  serde   = ["dep:serde"]
```

★ **关键认知**：写库时，把 `std` 做成可关闭的 feature 是 `no_std` 支持的标准做法。`serde?/std` 里的 `?` 表示**弱依赖启用**：只有当 serde 这个 optional 依赖已经被启用时，才去开它的 std feature。⚠ 不用 `?` 写成 `serde/std` 会强制启用 serde，破坏"没开 json 就不引入 serde"的设计。

# 3. feature 的解析规则

## 3.1 feature 是加法（additive）

```text
规则：feature 只能"增加"能力和依赖，不能"移除"或"替换"行为。
```

```rust
// 错误示范：feature 改变行为
#[cfg(feature = "fast")]
fn compute() -> u64 { /* 快速但精度低 */ }

#[cfg(not(feature = "fast"))]
fn compute() -> u64 { /* 慢但精确 */ }
```

★ **关键认知**：这是 feature 机制最重要的约束——**feature 必须是可叠加的**。因为同一个 crate 在依赖图里可能被多个包依赖，各自开启不同 feature，Cargo 会把它们的 feature **求并集**。如果两个 feature 互斥（一个要"精确"、一个要"快速"），并集后就无法确定用哪个实现。⚠ 需要"二选一"的配置，应该用运行时配置、泛型参数或独立 crate，而不是 feature。

## 3.2 feature 统一（unification）

```text
依赖图：
  app
  ├── lib-a ──> shared (features = ["x"])
  └── lib-b ──> shared (features = ["y"])

结果：shared 同时启用 x 和 y（求并集），只编译一次。
```

★ **关键认知**：同一个 crate 在依赖图里**只会被编译一次**，其 feature 是所有依赖方要求的并集。这保证了二进制里不会有"两份 shared"。⚠ 副作用是：你只想用 `x`，但别的依赖开了 `y`，你也被动获得了 `y`——这就是"feature 泄漏"。

## 3.3 resolver 2 的改进

```toml
[package]
edition = "2021"     # 默认 resolver 2
```

```text
resolver 1（旧）：feature 在"整个依赖图"层面统一，
                  包括 target-specific、dev、build 依赖，容易过度合并

resolver 2（新）：区分平台与上下文
  ├─ target-specific 依赖的 feature 不跨平台合并
  ├─ dev/build 依赖的 feature 不与普通依赖合并
  └─ 更少的意外 feature 泄漏
```

★ **关键认知**：Rust 2021 edition 默认 `resolver = "2"`，它把 feature 统一下沉到更精确的上下文，避免了"Windows 才用的依赖把 Linux 构建的 feature 也带开"这类问题。⚠ 工作区的 resolver 由根清单决定，成员不能各自设；虚拟 workspace 必须在根写 `resolver = "2"`，否则默认可能是 1。

# 4. feature 组合与条件编译

## 4.1 组合的命名约定

```toml
[features]
default = ["full"]
full    = ["json", "yaml", "logging"]
json    = ["dep:serde_json"]
yaml    = ["dep:serde_yaml"]
logging = ["dep:log"]
```

★ **关键认知**：用"元 feature"（如 `full`）打包常用组合，是给下游提供便利的常见手法。⚠ 但不要做"互斥组合"（如 `json` 和 `yaml` 只能二选一），这违反加法原则；如果实在需要，就让它们各自独立、允许同时开启。

## 4.2 `cfg` 的组合表达式

```rust
#[cfg(all(feature = "json", target_os = "linux"))]
fn linux_json() {}

#[cfg(any(feature = "json", feature = "yaml"))]
fn any_format() {}

#[cfg(not(feature = "std"))]
extern crate alloc;
```

| 组合子 | 含义 |
|--------|------|
| `all(a, b)` | 同时满足 |
| `any(a, b)` | 满足任一 |
| `not(a)` | 取反 |

## 4.3 在代码里检测 feature

```rust
#[cfg(feature = "json")]
pub const HAS_JSON: bool = true;

#[cfg(not(feature = "json"))]
pub const HAS_JSON: bool = false;

// 运行时可读，编译期已确定
pub fn supports_json() -> bool { HAS_JSON }
```

# 5. feature 冲突：成因与规避

## 5.1 冲突长什么样

```text
场景：库 A 定义 feature "use-tls"，库 B 定义 feature "no-tls"
      app 同时依赖 A 和 B，两者都依赖同一个网络底层 crate
      → 底层 crate 无法同时满足"要 TLS"和"不要 TLS"
```

```text
症状：
  ├─ 编译报 "multiple versions of crate" 或符号重复
  ├─ 运行时行为不符合预期（TLS 被意外启用）
  └─ 二进制体积异常膨胀
```

★ **关键认知**：feature 冲突的根源是**有人把 feature 当配置项用**。修复思路有三条：① 把它改成运行时的 `Config` 结构体；② 把两种实现拆成两个 crate（如 `foo-native` / `foo-rustls`），让用户选依赖；③ 用 trait + 泛型在类型层面区分。⚠ 不要在同一个 crate 里用 feature 表达"互斥模式"，这是生态里最常被吐槽的设计。

## 5.2 排查工具

```bash
cargo tree -e features          # 按 feature 维度展示依赖树
cargo tree -f "{p} {f}"         # 显示每个包的 feature
cargo build -v                  # 看实际传给 rustc 的 --cfg feature=...
```

## 5.3 最小化 feature 的实践

```toml
# 下游想要最小依赖时
[dependencies]
big-lib = { version = "1.0", default-features = false, features = ["core"] }
```

★ **关键认知**：`default-features = false` 是控制二进制体积和编译时间的第一手段。⚠ 关掉默认 feature 后，某些 API 会消失（"为什么会话 `use` 不到 X"），要对照文档确认所需 feature 是否已开启。

# 6. 常见错误

```text
错误 1：error: feature `x` includes `dep:y`, but `y` is not an optional dependency
原因：dep: 只能用于 optional 依赖
解决：把依赖改成 optional = true，或直接用 "y" 引用

错误 2：cargo build --features x 报 "package does not contain this feature"
原因：feature 名拼错，或该 feature 属于依赖而不是当前包
解决：cargo metadata 里查 features 列表；依赖的 feature 用 "dep/feature" 语法

错误 3：no_std 项目编译时报 "can't find crate for `std`"
原因：依赖默认开了 std，没有关掉
解决：依赖改成 default-features = false，并按需开 alloc

错误 4：workspace 里 resolver 默认是 1，出现 feature 泄漏
原因：虚拟 workspace 根没写 resolver = "2"
解决：根 Cargo.toml 的 [workspace] 加 resolver = "2"

错误 5：CI 只测了默认 feature，关掉后编不过
原因：feature 组合未覆盖
解决：CI 加 --no-default-features 和 --all-features 两轮
```

# 7. 练习

1. 给一个库加 `json` / `yaml` 两个 feature，各自引入一个 optional 依赖，用 `cargo tree -e features` 观察开启前后的差异。
2. 写一个 `no_std` 友好的 feature 结构（`std` / `alloc` / `default`），用 `cargo build --no-default-features` 验证能编译。
3. 故意制造一个"互斥 feature"，观察两个下游同时开启时的行为，然后改造成运行时配置。

# 8. 小结

| 概念 | 要点 |
|------|------|
| feature | 编译期条件开关，不进入二进制 |
| default | 普通 feature 名，可被下游关闭 |
| additive | feature 只能加，不能互斥 |
| 统一 | 同一 crate 只编译一次，feature 取并集 |
| resolver 2 | 按平台/上下文隔离 feature |
| `dep:` | 显式关联 optional 依赖，避免同名 feature |
| `serde?/std` | 弱依赖启用，依赖开了才传递 |
| 排查 | `cargo tree -e features` |

下一篇进入 profile 与构建优化，讲清楚 dev/release 的默认参数、LTO、codegen-units、strip、panic 策略，以及构建缓存如何影响编译速度。
