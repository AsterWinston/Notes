写了一个不错的 crate，怎么让别人用上？答案通常是发布到 **crates.io**——Rust 的官方包注册中心。发布看着只是 `cargo publish` 一条命令，但版本号一旦发出去就永久占用、不能修改，只能 yank 或发新版本。本篇讲清楚发布前的准备、打包的细节、版本语义的约束，以及发布后怎么补救。

# 1. 注册中心与发布模型

## 1.1 crates.io 的定位

```text
crates.io 的角色：
  ├─ 托管 crate 源码包（.crate 文件，本质是 tar.gz）
  ├─ 托管索引（让 cargo 能解析版本和依赖）
  └─ 托管文档（docs.rs 自动构建）
```

```text
发布与使用流程：

  作者                          使用者
   │                              │
   ├─ cargo publish ──> crates.io │
   │                      │       │
   │                      ▼       │
   │                  索引更新 ──> cargo add / cargo build
   │                              │
   └─ docs.rs 自动构建文档 ──────> 浏览 API
```

★ **关键认知**：crates.io 是**不可变（immutable）**的——一个版本发布后**永远不能覆盖或删除**（只能 yank）。这是为了保证依赖它的项目永远能拉到一模一样的源码。⚠ 所以发布前必须用 `cargo package` + `--dry-run` 检查，一旦发错版本号，只能发一个新的 patch 版本修正。

## 1.2 cargo publish 做什么

`cargo publish` 依次完成：校验必填字段与版本号 → 打包 `.crate` → 用包内容构建验证 → 上传 → 更新索引 → docs.rs 异步构建文档。

# 2. 发布前的必要准备

## 2.1 Cargo.toml 必填与推荐字段

```toml
[package]
name = "my-crate"                    # 全局唯一，小写连字符
version = "0.1.0"
edition = "2021"
rust-version = "1.75"                # MSRV，下游据此判断能否用
description = "一句话说明这个 crate 做什么"   # 必填
license = "MIT OR Apache-2.0"        # 必填（或用 license-file）
repository = "https://github.com/you/my-crate"
documentation = "https://docs.rs/my-crate"
readme = "README.md"
keywords = ["cli", "parser"]         # 最多 5 个
categories = ["command-line-utilities"]
```

★ **关键认知**：`description` 和 `license` 是**发布强制字段**，缺一个 `cargo publish` 就会报错。`license = "MIT OR Apache-2.0"` 是 Rust 生态的惯例双许可——使用者可以任选其一，兼容性最好。⚠ `license-file` 和 `license` 不能同时写，二选一。

## 2.2 版本号的语义约束

```text
semver：MAJOR.MINOR.PATCH

  1.0.0 → 1.0.1   修复 bug，兼容
  1.0.0 → 1.1.0   新增功能，兼容
  1.0.0 → 2.0.0   破坏性变更，不兼容

0.x 阶段：
  0.1.0 → 0.1.1   视为兼容
  0.1.0 → 0.2.0   视为不兼容（Cargo 按新 major 对待）
```

★ **关键认知**：crates.io 不允许重新发布同一版本号，所以**发布前想清楚版本号**。破坏性变更必须升 major（1.x 之后），因为 Cargo 的 `^` 语义会让下游自动接受 minor/patch 升级——你若在 patch 版本里塞破坏性变更，会悄悄破坏大量下游。⚠ 0.x 阶段虽然宽松，但 `0.1` → `0.2` 仍会让下游的 `^0.1` 失效，升级前看 changelog。

## 2.3 依赖的发布约束

```toml
[dependencies]
# 路径依赖必须同时给 version，否则发布失败
mylib = { version = "0.1", path = "../mylib" }

# git 依赖不能发布（crates.io 不接受）
# bad = { git = "https://github.com/x/y" }   ← 发布时会被拒绝
```

★ **关键认知**：发布时 Cargo 会把 `path` 依赖替换成 crates.io 上的版本依赖，所以 `version` 必须写且与 crates.io 上已发布的版本一致。⚠ git 依赖无法发布——如果确实要用未发版的依赖，要么先把它发到 crates.io，要么用 `[patch]` 在本地覆盖（但 `[patch]` 不会被打包）。

# 3. 打包：cargo package

## 3.1 先 dry-run

```bash
cargo package --list          # 列出会被打进包的文件
cargo package --dry-run       # 打包但不发布，验证能否成功
cargo package                 # 生成 target/package/my-crate-0.1.0.crate
```

```text
输出的 .crate 是 tar.gz，解压后可看到实际内容：
  my-crate-0.1.0/
  ├── Cargo.toml        # 被规范化（补全元信息）
  ├── Cargo.toml.orig   # 原始清单
  ├── src/
  └── README.md
```

★ **关键认知**：`cargo package --list` 是你发布前最该跑的命令。它会告诉你**到底哪些文件会被上传**，能抓出"忘了包含 LICENSE"、"把巨大的测试数据打进去了"、"把 `.env` 泄露了"这类问题。⚠ `package` 会在隔离环境里重新构建一次，能提前发现"本地能编、打包后编不过"的问题（常见于漏了 `include` 的资源文件）。

## 3.2 控制打包内容

```toml
[package]
include = ["src/**", "Cargo.toml", "README.md", "LICENSE*"]   # 白名单
# exclude = ["tests/fixtures/**", "benches/**"]              # 黑名单
```

```text
默认规则：
  ├─ 包含：src/、Cargo.toml、README、LICENSE、build.rs
  ├─ 排除：target/、.git/、Cargo.lock（库）、隐藏文件
  └─ 规则：include 与 exclude 不能同时用
```

★ **关键认知**：`include` 是白名单，一旦设置，**只有列出的文件会被打包**，连 `Cargo.toml` 都要显式写（或不写时会自动补）。⚠ 这是双刃剑：白名单能防泄露、控制体积，但漏列一个运行必需的文件就会导致"发布后下游编译失败"。

## 3.3 examples、tests、benches 的处理

`examples/`、`tests/`、`benches/` 默认会被打包；⚠ 若它们用了 `dev-dependencies`，`cargo package` 验证时会尝试编译，依赖没声明全就会失败。

# 4. 发布

## 4.1 登录与 token

```bash
cargo login                      # 交互式粘贴 token（存到 ~/.cargo/credentials.toml）
cargo login <token>              # 直接传入（不推荐，会进 shell 历史）
```

```text
Token 管理：在 crates.io 账户页生成（可限定 scope 和过期时间）；
存在 ~/.cargo/credentials.toml（不要提交到 git）；CI 用 CARGO_REGISTRY_TOKEN 注入。
```

★ **关键认知**：token 等同于账号密码，**绝不能进版本库**。CI 里用仓库 Secret 注入 `CARGO_REGISTRY_TOKEN`，本地用 `cargo login` 存到用户目录。⚠ 一旦 token 泄露，立即去 crates.io 撤销——攻击者可以用它发布恶意版本，而版本一经发布无法删除。

## 4.2 执行发布

```bash
cargo publish --dry-run          # 完整走一遍流程但不真的上传
cargo publish                    # 正式发布
cargo publish -p my-crate        # workspace 里指定成员
```

```bash
# 发布 workspace 里多个 crate 时，注意顺序
cargo publish -p core            # 先发被依赖的
cargo publish -p api             # 再发依赖 core 的
cargo publish -p cli
```

★ **关键认知**：workspace 发布必须**按依赖拓扑顺序**进行——被依赖的 crate 必须先上 crates.io，否则依赖它的 crate 发布时会找不到版本。⚠ 发布不是原子的，中途失败要手动补发剩下的，建议用一个发布脚本明确顺序。

## 4.3 发布后的传播延迟

`cargo publish` 成功后，网页立即可见，但索引更新要几十秒、docs.rs 构建要几分钟，国内镜像还有同步延迟。

★ **关键认知**：刚发布的版本可能短暂"查不到"，尤其在国内镜像源上，同步有延迟。⚠ 遇到 `cargo add` 找不到刚发的版本，先等一两分钟，或临时用官方源验证，不要急着以为发布失败而重复发布（会报"版本已存在"）。

# 5. 文档与 docs.rs

## 5.1 文档注释

```rust
/// 把两个数相加。
///
/// # 示例
///
/// ```
/// let sum = my_crate::add(1, 2);
/// assert_eq!(sum, 3);
/// ```
///
/// # Panics
///
/// 当结果溢出时 panic。
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

★ **关键认知**：文档注释里的代码块会被 `cargo test` 当作**文档测试**执行。这既是文档又是测试——示例写错了 CI 会挂。⚠ 文档测试默认在独立 crate 里编译，所以示例里的 `use` 要写完整路径（`my_crate::add`），不能依赖当前作用域。

## 5.2 docs.rs 配置

```toml
[package.metadata.docs.rs]
all-features = true                       # 构建所有 feature 的文档
rustdoc-args = ["--cfg", "docsrs"]        # 配合 #[cfg_attr(docsrs, doc(cfg(...)))]
targets = ["x86_64-unknown-linux-gnu"]
```

```rust
#![cfg_attr(docsrs, feature(doc_cfg))]

#[cfg(feature = "json")]
#[cfg_attr(docsrs, doc(cfg(feature = "json")))]
pub fn to_json() {}
```

★ **关键认知**：docs.rs 默认只构建默认 feature，`all-features = true` 能让所有 feature 下的 API 都出现在文档里。`doc(cfg(...))` 会标注"这个 API 需要哪个 feature"，对使用者非常友好。⚠ `doc(cfg)` 是 nightly 特性，需要 `#![cfg_attr(docsrs, feature(doc_cfg))]`，且只在 docs.rs 构建时启用，不影响 stable 编译。

# 6. yank 与版本补救

## 6.1 yank 是什么

```bash
cargo yank --version 0.1.1        # 撤回某个版本
cargo yank --version 0.1.1 --undo # 取消撤回
```

```text
yank 的效果：
  ├─ 新项目无法再解析到这个版本
  ├─ 已有 Cargo.lock 的项目仍能拉到（不破坏构建）
  └─ 版本本身没有被删除，只是"标记为不推荐"
```

★ **关键认知**：yank 是"**软撤回**"，不是删除。它的设计目的是"阻止新项目用这个坏版本，但不破坏已锁定它的项目"。⚠ yank 不能修复已发布的代码——如果版本里有严重 bug 或安全问题，正确做法是 yank 掉并**立刻发一个新的 patch 版本**，同时在 README / changelog 里说明。

## 6.2 发布事故处理表

| 事故 | 处理 |
|------|------|
| 版本号写错（如 0.1.0 发成 0.0.1） | 无法改，直接发正确的新版本，yank 错的 |
| 漏打包文件导致编不过 | yank + 发 patch |
| 严重安全漏洞 | yank + 发 patch + 公告（RUSTSEC） |

★ **关键认知**：`Cargo.lock` 会记住被 yank 的版本，所以 yank 不会让已有项目突然编不过——这是"稳定性优先"的设计。⚠ 但 `cargo update` 会避开被 yank 的版本，所以 yank 后下游升级会自然跳到新版本。

# 7. 常见错误

```text
错误 1：error: missing field `description`
原因：必填字段缺失
解决：补 description / license，或 license-file

错误 2：error: crates.io is not a valid registry for ...
原因：依赖里有 git 或没有 version 的 path 依赖
解决：改成 crates.io 版本依赖

错误 3：error: version `0.1.0` already exists
原因：版本号已被占用，crates.io 不可变
解决：升 patch 版本再发

错误 4：error: failed to verify package tarball
原因：打包后构建失败，通常漏了文件或 dev 依赖没声明全
解决：cargo package --list 检查，cargo package 本地复现

错误 5：下游报 "can't find crate" 但本地正常
原因：漏打包了某个文件，或 include 白名单漏项
解决：解压 .crate 检查内容，补 include

错误 6：token 无效 / 401
原因：token 过期、被撤销、或 CARGO_REGISTRY_TOKEN 没注入
解决：重新 cargo login，CI 检查 Secret 配置
```

# 8. 发布检查清单

```text
发布前：
  □ description / license / repository 齐全
  □ 版本号符合 semver，且未发布过
  □ path 依赖都带了 version，无 git 依赖
  □ cargo package --list / --dry-run 通过
  □ cargo test / clippy / fmt 全绿
  □ README、LICENSE、CHANGELOG 齐全

发布后：
  □ 网页可见，索引已更新
  □ docs.rs 构建成功
  □ 试一遍 cargo add 新版本并编译
```

# 9. 练习

1. 建一个库 crate，补全必填字段，用 `cargo package --list` 查看默认打包内容，再用 `include` 白名单对比。
2. 写一段带文档测试的 `///` 注释，故意写错示例里的断言，验证 `cargo test` 会失败。
3. 模拟"发错版本"：本地打两个版本（0.1.0、0.1.1），思考如何用 yank 处理 0.1.0，写出操作步骤。

# 10. 小结

| 概念 | 要点 |
|------|------|
| crates.io | 不可变注册中心，版本不能覆盖 |
| 必填字段 | `description` + `license` |
| semver | 破坏性变更升 major，patch 不能破坏 |
| `cargo package` | 先 `--list`、再 `--dry-run` |
| include/exclude | 白名单/黑名单，不能同时用 |
| 发布顺序 | workspace 按依赖拓扑发 |
| docs.rs | `all-features = true`、`doc(cfg)` |
| yank | 软撤回，不删除，不破坏已有 lock |
| token | 绝不入库，泄露立即撤销 |

下一篇进入 CI、缓存与故障排查，讲清楚 GitHub Actions 里的 Rust 构建、`Swatinem/rust-cache` 的用法、离线构建、交叉编译，以及依赖冲突和链接错误的排查套路。
