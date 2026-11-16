当项目从"一个 crate"长成"核心库 + 命令行 + 服务端 + 若干工具"时，如果每个 crate 各自为政，你会遇到版本不一致、重复编译、依赖树分裂等问题。Cargo 的 **workspace** 就是为这种"多个相关 crate 一起开发"设计的。本篇讲清楚 workspace 的成员组织、依赖共享、虚拟 workspace 的取舍，以及目录怎么摆。

# 1. 为什么需要 workspace

## 1.1 多个 crate 分开管理的痛点

```text
假设有三个 crate：core、api、cli，cli 和 api 都依赖 core。

分开管理（各自独立 Cargo.toml）：
  ├─ 三份 Cargo.lock，core 的版本可能各自锁定不同版本
  ├─ 三个 target/ 目录，core 被编译三遍
  └─ 升级 core 要三个仓库分别改、分别提交
```

```text
Workspace 管理：
  ├─ 一份 Cargo.lock，全工作区版本一致
  ├─ 一个 target/，core 只编译一次（缓存共享）
  └─ core 改动一次，所有成员同时可见
```

★ **关键认知**：workspace 解决的核心问题是**一致性 + 缓存共享**，不是"目录好看"。它让多个 crate 像一个大项目一样被解析依赖、编译、测试，但每个 crate 仍然是独立可发布的包。⚠ workspace 不是"合并成一个 crate"，成员之间仍然是独立的库/二进制，只是共享构建上下文。

## 1.2 一个 workspace 的解剖

```text
workspace/
├── Cargo.toml          # workspace 根清单（可同时是某个 package）
├── Cargo.lock          # 全工作区共享一份
├── target/             # 全工作区共享一个
├── crates/
│   ├── core/
│   │   ├── Cargo.toml  # 成员清单
│   │   └── src/lib.rs
│   ├── api/
│   │   ├── Cargo.toml  # 依赖 core：{ path = "../core" }
│   │   └── src/lib.rs
│   └── cli/
│       ├── Cargo.toml  # 依赖 core 和 api
│       └── src/main.rs
```

# 2. 定义 workspace

## 2.1 根清单

```toml
# workspace/Cargo.toml
[workspace]
members = [
    "crates/core",
    "crates/api",
    "crates/cli",
]
resolver = "2"          # Rust 2021 起默认 2，显式写上更清楚
```

根 `Cargo.toml` 只写 `[workspace]`，不写 `[package]`，就是**虚拟 workspace**（virtual workspace）——它本身不是包，只是成员容器。

★ **关键认知**：`members` 支持 glob 吗？**不直接支持**。Cargo 的 `members` 是路径列表，但你可以用 `"crates/*"` 这种通配符（Cargo 支持 glob 展开）。⚠ 不过通配符会误收不想纳入的目录（比如刚建的实验性 crate），显式列出成员更可控。

## 2.2 成员清单

```toml
# crates/core/Cargo.toml
[package]
name = "core"
version = "0.1.0"
edition = "2021"
```

```toml
# crates/cli/Cargo.toml
[package]
name = "cli"
version = "0.1.0"
edition = "2021"

[dependencies]
core = { path = "../core" }   # 成员之间用 path 依赖
api = { path = "../api" }
```

成员之间用 `path` 依赖，Cargo 会自动把 path 解析为 workspace 内的成员，不需要额外声明。

## 2.3 排除成员

```toml
[workspace]
members = ["crates/*"]
exclude = ["crates/experimental"]   # 通配符里排除某个目录
```

⚠ `exclude` 只对 glob 展开有效，显式列出的成员不会被 `exclude` 移除。

# 3. 虚拟 workspace vs 根包 workspace

## 3.1 两种形态

| 形态 | 根 Cargo.toml | 适用场景 |
|------|---------------|---------|
| 虚拟 workspace | 只有 `[workspace]` | 纯多 crate 项目，根目录不产出产物 |
| 根包 workspace | `[package]` + `[workspace]` | 根目录本身就是一个 crate，其他是附属 |

根包 workspace 示例：

```toml
# 根目录就是一个 crate，同时是 workspace 根
[package]
name = "app"
version = "0.1.0"
edition = "2021"

[dependencies]
core = { path = "crates/core" }   # 相对根目录

[workspace]
members = ["crates/core", "crates/cli"]
```

★ **关键认知**：根包 workspace 里，根目录**自动是成员**，不需要写进 `members`。这是新手常困惑的点：明明 `members` 没列根包，`cargo build` 却编译了它。⚠ 虚拟 workspace 没有 `[package]`，所以在根目录直接 `cargo run` 会报"no bin target"，必须用 `-p` 指定成员或用 `default-members`。

## 3.2 default-members

```toml
[workspace]
members = ["crates/core", "crates/api", "crates/cli"]
default-members = ["crates/cli"]    # 不带 -p 时默认操作 cli
```

```bash
cargo build              # 只构建 default-members（cli）
cargo build --workspace  # 构建全部成员
```

# 4. 共享依赖版本

## 4.1 workspace.dependencies

```toml
# 根 Cargo.toml
[workspace]
members = ["crates/core", "crates/api", "crates/cli"]

[workspace.dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
anyhow = "1.0"
core = { path = "crates/core" }     # 也可以把成员放进来
```

```toml
# crates/cli/Cargo.toml
[dependencies]
serde = { workspace = true }        # 继承根目录的版本和 feature
tokio = { workspace = true }
core = { workspace = true }
```

★ **关键认知**：`[workspace.dependencies]` 是**版本的单一事实来源**。成员用 `workspace = true` 继承，避免"core 用 serde 1.0.100、cli 用 serde 1.0.150"这种版本漂移。⚠ 继承的是**版本约束和 feature**，不是锁定的精确版本；实际版本仍由 workspace 的 `Cargo.lock` 统一决定。

## 4.2 成员覆盖 feature

```toml
# 成员可以追加 feature，但不能改 version
[dependencies]
serde = { workspace = true, features = ["rc"] }
```

成员可以在此基础上加 feature，但不能覆盖版本——否则 Cargo 会报错。这是有意设计：版本统一，feature 可以按需叠加。

## 4.3 共享 package 元信息

```toml
# 根 Cargo.toml
[workspace.package]
version = "0.3.0"
edition = "2021"
license = "MIT OR Apache-2.0"
authors = ["Team <team@example.com>"]
repository = "https://github.com/example/repo"
```

```toml
# 成员
[package]
name = "core"
version.workspace = true
edition.workspace = true
license.workspace = true
```

★ **关键认知**：`[workspace.package]` 让所有成员的版本、许可证、作者等信息集中管理。Rust 生态常见做法是"全工作区统一版本号"，发布时一起 bump。⚠ 这意味着成员不能有独立版本号——如果你的 crate 需要各自独立演进，就别用 `workspace.package`，改为各写各的。

# 5. 常用操作

## 5.1 指定成员

```bash
cargo build -p core          # 只构建 core
cargo test -p api            # 只测 api
cargo run -p cli             # 运行 cli
cargo build -p core -p api   # 构建多个成员
```

## 5.2 全工作区操作

```bash
cargo build --workspace           # 构建所有成员
cargo test --workspace            # 测试所有成员
cargo check --workspace --all-targets
cargo clippy --workspace -- -D warnings
cargo publish -p core             # 只发布 core
```

## 5.3 查看依赖关系

```bash
cargo tree -p cli            # 看 cli 的依赖树
cargo tree -d                # 找重复依赖（同一 crate 多版本）
cargo metadata --no-deps     # 列出所有成员
```

★ **关键认知**：`cargo tree -d` 是 workspace 排障的利器。它列出"同一个 crate 出现多个版本"的地方——这往往是因为不同成员对同一依赖写了不同版本范围，是编译变慢和二进制膨胀的常见原因。⚠ 找到后应该用 `[workspace.dependencies]` 统一，而不是逐个成员硬改。

# 6. 目录组织建议

## 6.1 常见布局

```text
方案 A：crates/ 子目录（推荐）
workspace/
├── Cargo.toml
└── crates/
    ├── core/
    ├── api/
    └── cli/

方案 B：成员平铺在根目录
workspace/
├── Cargo.toml
├── core/
├── api/
└── cli/

方案 C：按职责分层
workspace/
├── Cargo.toml
├── libs/            # 库
│   ├── core/
│   └── utils/
└── apps/            # 二进制
    ├── cli/
    └── server/
```

★ **关键认知**：目录结构没有标准答案，但**"库和二进制分开"**是普遍共识。库成员放 `libs/` 或 `crates/`，可执行文件放 `apps/` 或 `bins/`，一眼就能看出谁会被发布、谁只是工具。⚠ 别把成员塞进 `src/` 里——Cargo 会把它当成普通模块，而不是独立 crate。

# 7. 常见错误

```text
错误 1：error: package `x` is a member of the wrong workspace
原因：某个成员的 Cargo.toml 里写了 [workspace] 指向别处，或在别的 workspace 根下
解决：删除成员里的 [workspace]，让它归属外层

错误 2：error: multiple workspace roots found
原因：嵌套目录各有一个 [workspace]，Cargo 不知道用哪个
解决：检查是否在成员目录里误建了 workspace 根

错误 3：cargo run 报 no bin target named ...
原因：虚拟 workspace 里根目录没有二进制，且没指定 -p
解决：cargo run -p cli，或配置 default-members

错误 4：workspace 成员版本不一致，发布时报依赖找不到
原因：成员 A 依赖 core 0.2，但 core 实际版本是 0.3
解决：统一用 [workspace.package] 管版本，或用 [workspace.dependencies] 的 path 依赖
```

# 8. 练习

1. 建一个虚拟 workspace，包含 `core`（lib）和 `cli`（bin），让 `cli` 依赖 `core`，用 `cargo run -p cli` 跑通。
2. 在根清单加 `[workspace.dependencies]` 声明 `serde`，两个成员都用 `workspace = true` 继承，用 `cargo tree` 确认版本一致。
3. 给 workspace 配 `default-members = ["crates/cli"]`，验证 `cargo build` 与 `cargo build --workspace` 的差异。

# 9. 小结

| 概念 | 要点 |
|------|------|
| workspace | 多 crate 共享锁文件、target、依赖版本 |
| 虚拟 workspace | 根只有 `[workspace]`，需 `-p` 指定成员 |
| 根包 workspace | 根目录本身是成员，自动纳入 |
| `workspace.dependencies` | 版本单一事实来源，成员用 `workspace = true` |
| `workspace.package` | 统一版本、许可证等元信息 |
| `-p` / `--workspace` | 精确操作某成员 / 全部成员 |
| `cargo tree -d` | 排查重复依赖 |

下一篇进入 feature，讲清楚 feature 如何做条件编译、默认 feature 的取舍、feature 组合爆炸问题，以及如何避免"feature 冲突"这个生态级难题。
