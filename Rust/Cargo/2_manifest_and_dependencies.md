`Cargo.toml` 是 Rust 项目的"身份证 + 采购清单"：它既声明这个包是谁（名字、版本、edition），也声明它依赖谁（依赖名、版本、来源、feature）。看似简单的键值对，背后有一套版本语义和依赖分类规则。本篇把清单文件和依赖声明讲透，让你写依赖时不再靠复制粘贴。

# 1. Cargo.toml 的整体结构

## 1.1 一个完整的最小清单

```toml
[package]
name = "demo"                  # 包名，crates.io 上全局唯一
version = "0.1.0"              # 语义化版本
edition = "2021"               # 语法纪元，不是编译器版本
rust-version = "1.75"          # 最低支持的 Rust 版本（MSRV）
description = "一个演示项目"    # 发布到 crates.io 时必填
license = "MIT OR Apache-2.0"  # 双许可证是 Rust 生态惯例
authors = ["Your Name <you@example.com>"]

[dependencies]
serde = { version = "1.0", features = ["derive"] }

[dev-dependencies]
criterion = "0.5"

[build-dependencies]
cc = "1.0"

[[bin]]
name = "demo"
path = "src/main.rs"
```

★ **关键认知**：`edition` 不是编译器版本，而是**语法纪元**（2015/2018/2021/2024）。它决定这门"方言"用哪套语法规则，比如 2021 的闭包捕获规则、`array.into_iter()` 的行为都变了。同一个项目里，不同 crate 可以是不同 edition 并互相调用——这正是 Rust 让语言演进不破坏旧代码的机制。

## 1.2 各 section 的职责

| 段落 | 作用 | 是否必填 |
|------|------|---------|
| `[package]` | 包元信息（名字、版本、作者、许可证） | 是 |
| `[dependencies]` | 运行时依赖 | 否 |
| `[dev-dependencies]` | 测试、示例、bench 专用 | 否 |
| `[build-dependencies]` | `build.rs` 专用 | 否 |
| `[features]` | 特性开关定义 | 否 |
| `[profile.*]` | 编译优化配置 | 否 |
| `[[bin]]` / `[lib]` | 显式声明目标 | 否（有约定） |
| `[workspace]` | 多包工作区 | 否 |

# 2. 依赖版本语义

## 2.1 语义化版本回顾

```text
版本号 X.Y.Z：
  X（major）  不兼容的 API 变更
  Y（minor）  向后兼容的新功能
  Z（patch）  向后兼容的 bug 修复
```

## 2.2 Cargo 的版本约束写法

```toml
[dependencies]
a = "1.2.3"        # 等价于 ^1.2.3：>=1.2.3, <2.0.0
b = "^1.2"         # >=1.2.0, <2.0.0（1.x 内任意）
c = "~1.2.3"       # >=1.2.3, <1.3.0（只允许 patch 升级）
d = "=1.2.3"       # 精确锁定，几乎不用
e = ">=1.2, <1.5"  # 显式范围
f = "*"            # 任意版本，强烈不推荐
```

★ **关键认知**：Cargo 的默认 `"1.2.3"` 是**脱字符语义（caret）**，即允许 `1.x` 内升级而锁定 major 版本。这遵循"semver 承诺"：依赖作者承诺 minor/patch 升级不破坏 API。⚠ 但世界上总有作者不遵守 semver，所以 `Cargo.lock` 才是真正的保险——它把实际版本钉死，测试通过后再更新。

## 2.3 0.x 版本的特殊性

```text
"0.8"  →  >=0.8.0, <0.9.0     # 0.x 时 minor 被视为不兼容边界
"0.0.3" → >=0.0.3, <0.0.4     # 0.0.x 时 patch 也被视为不兼容边界
```

★ **关键认知**：`0.x` 版本的兼容边界和 `1.x` 不同——`0.8` 升到 `0.9` 可能带来破坏性变更。这是 Cargo 对"未稳定 API"的保护：库作者在 1.0 之前不负兼容承诺。⚠ 这也是为什么大量 Rust 库长期停在 `0.x`，升级时要格外看 changelog。

## 2.4 Cargo.lock 与 update

```bash
cargo update                 # 在 Cargo.toml 允许范围内升级到最新
cargo update -p serde        # 只更新 serde
cargo update -p serde --precise 1.0.200   # 精确指定
```

`cargo update` 只改 `Cargo.lock`，不改 `Cargo.toml`。如果最新版超出了 `Cargo.toml` 的范围，它不会升上去。

# 3. 依赖的三种分类

## 3.1 普通、dev、build 依赖

```toml
[dependencies]
serde = "1.0"            # 编译进最终产物，运行时需要

[dev-dependencies]
criterion = "0.5"        # 只用于测试/示例/bench
pretty_assertions = "1"

[build-dependencies]
cc = "1.0"               # 只用于 build.rs，不进入最终产物
```

```text
编译阶段依赖可见性：

  build.rs ──用──> [build-dependencies]
     │
     ▼ 生成代码 / 配置
  你的 crate ──用──> [dependencies]
     │
     ▼
  测试代码 ──用──> [dev-dependencies] + [dependencies]
```

★ **关键认知**：三种依赖的隔离是**编译期**的，不是运行时的。`build-dependencies` 只在编译 `build.rs` 时可用，不会进入你的库；`dev-dependencies` 只在测试/示例/bench 目标里可用，不会污染发布产物。⚠ 但 `dev-dependencies` 会被 `cargo test` 全部编译，测试依赖太重会拖慢 CI。

## 3.2 optional 依赖与 feature

```toml
[dependencies]
serde = { version = "1.0", optional = true }

[features]
default = []
json = ["dep:serde"]        # 打开 json feature 才引入 serde
```

```rust
// 只有开了 feature 才编译这段
#[cfg(feature = "json")]
pub fn to_json<T: serde::Serialize>(v: &T) -> String {
    serde_json::to_string(v).unwrap()
}
```

★ **关键认知**：`optional = true` 让依赖变成"可选"，默认不编译；`dep:serde` 语法（Cargo 1.60+）表示"这个 feature 直接启用依赖 serde"，而不会隐式产生同名 feature。⚠ 老写法 `json = ["serde"]` 会同时创建一个叫 `serde` 的 feature，容易和依赖名混淆，新项目一律用 `dep:` 前缀。

# 4. 依赖来源

## 4.1 三种来源对比

| 来源 | 写法 | 适用场景 |
|------|------|---------|
| crates.io | `serde = "1.0"` | 绝大多数公开库 |
| 本地路径 | `foo = { path = "../foo" }` | 同 workspace / 本地开发 |
| git | `foo = { git = "https://..." }` | 用未发布的修复或分支 |

## 4.2 路径依赖

```toml
[dependencies]
mylib = { path = "../mylib" }
```

```text
项目布局：
workspace/
├── app/
│   ├── Cargo.toml        # mylib = { path = "../mylib" }
│   └── src/main.rs
└── mylib/
    ├── Cargo.toml
    └── src/lib.rs
```

路径依赖是本地的、不走网络，改 `mylib` 后 `app` 会重新编译。发布到 crates.io 时，Cargo 要求路径依赖**同时给出 version**，否则打包失败：

```toml
mylib = { version = "0.1", path = "../mylib" }
```

## 4.3 git 依赖

```toml
[dependencies]
# 默认用主分支最新提交
foo = { git = "https://github.com/user/foo" }

# 锁定分支 / tag / 具体 commit
bar = { git = "https://github.com/user/bar", branch = "dev" }
baz = { git = "https://github.com/user/baz", tag = "v1.2.0" }
qux = { git = "https://github.com/user/qux", rev = "abc1234" }

# 指定子目录（仓库里是 workspace 时）
sub = { git = "https://github.com/user/mono", package = "sub-crate" }
```

★ **关键认知**：git 依赖适合"上游还没发版、但我需要这个修复"的过渡期。⚠ 它的代价是可复现性差：`branch` 会随上游变动，只有 `rev`（commit hash）是真正锁定的。生产项目若必须用 git 依赖，务必写 `rev`，并在上游发版后尽快切回 crates.io。

## 4.4 依赖重命名

```toml
[dependencies]
serde1 = { package = "serde", version = "1.0" }
serde2 = { package = "serde", version = "0.9" }
```

```rust
use serde1::Serialize;   // 代码里用重命名后的名字
```

★ **关键认知**：`package` 字段让你在**同一个项目里同时依赖同一个 crate 的两个不兼容版本**（比如迁移期同时用 serde 0.9 和 1.0）。⚠ 这会让二进制体积和编译时间翻倍，只能作为迁移的临时手段，不要长期保留。

# 5. 其它常用字段

## 5.1 目标声明

```toml
[lib]
name = "mylib"          # 库名（默认取 package name 的下划线形式）
crate-type = ["cdylib", "rlib"]   # 同时产出 C 动态库和 Rust 库

[[bin]]
name = "tool"
path = "src/bin/tool.rs"
```

## 5.2 条件依赖（按平台）

```toml
[target.'cfg(windows)'.dependencies]
winapi = "0.3"

[target.'cfg(unix)'.dependencies]
libc = "0.2"

[target.x86_64-unknown-linux-gnu.dependencies]
openssl = "0.10"
```

★ **关键认知**：`[target.'cfg(...)'.dependencies]` 让依赖**只在特定平台编译**，这是写跨平台库的必备手段。⚠ `cfg` 里的条件在"解析依赖"阶段就生效，所以不满足条件的依赖根本不会被下载，不是编译期才跳过。

# 6. 常见错误

```text
错误 1：error: dependency 'foo' is optional but not enabled by any feature
原因：optional 依赖没有在 [features] 里被引用
解决：加一个 feature，用 "dep:foo" 或 "foo" 启用它

错误 2：error: failed to parse manifest: expected a table
原因：TOML 语法错，常见于把字符串写成数组、漏引号
解决：用 cargo metadata 验证，编辑器装 TOML 插件

错误 3：error: the lock file needs to be updated but --locked was passed
原因：Cargo.lock 过期，CI 用 --locked 时暴露
解决：本地跑 cargo update 后提交新的 Cargo.lock

错误 4：依赖版本冲突，同一个 crate 出现多个版本
原因：不同依赖要求互斥的版本范围
解决：cargo tree -d 查看重复依赖，升级或替换冲突方
```

# 7. 练习

1. 建一个项目，尝试 `serde = "1"`、`serde = "~1.0.100"`、`serde = "=1.0.100"` 三种写法，用 `cargo tree` 观察实际选中的版本差异。
2. 给一个项目加 `optional` 依赖，用 `--features` 开关前后分别 `cargo build`，观察编译时间与产物变化。
3. 用 `[target.'cfg(windows)'.dependencies]` 加一个只在 Windows 生效的依赖，在 Linux 环境（或交叉检查）下确认它未被编译。

# 8. 小结

| 概念 | 要点 |
|------|------|
| edition | 语法纪元，不是编译器版本 |
| 默认版本 | `"1.2.3"` 是 caret，允许 1.x 升级 |
| 0.x 语义 | minor 被视为不兼容边界 |
| 依赖分类 | 普通 / dev / build，编译期隔离 |
| optional + feature | `dep:xxx` 是现代写法 |
| 路径依赖 | 发布时需同时给 version |
| git 依赖 | 生产环境务必用 `rev` 锁定 |
| `package` 重命名 | 同名多版本共存，仅限迁移 |

下一篇进入 workspace，讲清楚多 crate 如何共享锁文件与构建缓存、虚拟 workspace 的组织方式，以及成员间的依赖如何声明。
