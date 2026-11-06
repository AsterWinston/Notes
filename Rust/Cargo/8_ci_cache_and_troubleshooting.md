本地构建一切正常，推到 CI 却挂了；依赖明明在本地缓存里，离线机器上却拉不下来；想交叉编译到 ARM，结果链接器报一堆看不懂的错。这类问题大多不是 Rust 代码本身的问题，而是 **Cargo 的构建环境**问题。本篇讲 CI 配置、缓存策略、离线构建、交叉编译，以及依赖冲突和链接错误的排查套路，作为本模块的收尾。

# 1. CI 里的 Rust 构建

## 1.1 一个可用的 GitHub Actions 配置

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: 安装 Rust 工具链
        uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
      - name: 缓存 Cargo 与 target
        uses: Swatinem/rust-cache@v2
      - name: 格式检查
        run: cargo fmt --all -- --check
      - name: Lint
        run: cargo clippy --workspace --all-targets -- -D warnings
      - name: 测试
        run: cargo test --workspace --all-features
      - name: 构建
        run: cargo build --workspace --release
```

★ **关键认知**：CI 的第一步不是 `cargo build`，而是 `cargo fmt --check` 和 `cargo clippy -D warnings`。这两步最快，能在最便宜的环境里挡掉大部分低级问题，避免跑完十分钟测试才发现格式不对。⚠ `-D warnings` 会让警告变成错误，升级工具链后可能突然挂——要么锁工具链版本，要么给 clippy 加 `--cap-lints allow` 临时放行。

## 1.2 矩阵与 feature 组合

```yaml
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        rust: [stable, beta]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ matrix.rust }}
      - uses: Swatinem/rust-cache@v2
      - run: cargo test --workspace
```

```yaml
      - run: cargo test                                  # 默认 feature
      - run: cargo test --no-default-features            # 无默认 feature
      - run: cargo test --all-features                   # 全部 feature
```

★ **关键认知**：跨平台矩阵能抓出"只在 Windows 编不过"这类平台相关代码问题（`cfg` 写错、路径分隔符、原生库链接）。库 crate 还必须测试默认、无默认、全开三组 feature，否则 `#[cfg(feature)]` 剔除的分支会悄悄腐坏。⚠ 矩阵会成倍消耗 CI 分钟数，常见折中是"主平台跑全量测试，其他平台只跑 `cargo check`"。

# 2. 缓存策略

## 2.1 该缓存什么

| 路径 | 内容 | 是否缓存 |
|------|------|---------|
| `~/.cargo/registry/` | 下载的 crate 源码与索引 | 是 |
| `~/.cargo/git/` | git 依赖的克隆 | 是 |
| `target/` | 编译产物与增量缓存 | 是（按需） |

```text
缓存收益：registry / git 省下载时间（国内尤其明显），target 省编译时间（大项目收益最大）。
```

## 2.2 rust-cache 与手动缓存

```yaml
      - uses: Swatinem/rust-cache@v2
        with:
          workspaces: ". -> target"       # workspace 路径映射
          cache-on-failure: true          # 失败也保存缓存
          key: ${{ matrix.os }}-${{ matrix.rust }}
```

```yaml
      - uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
          restore-keys: ${{ runner.os }}-cargo-
```

★ **关键认知**：缓存 key 用 `Cargo.lock` 的哈希，保证"依赖变了就重建、没变就命中"；`restore-keys` 提供模糊匹配，让新 key 没命中时至少能复用旧缓存做增量。`Swatinem/rust-cache` 已自动处理这些。⚠ `target/` 在不同 Rust 版本间不完全兼容，矩阵构建时要按 `rust` 版本分 key，否则会复用出诡异错误。

# 3. 离线构建

## 3.1 vendor 依赖

```bash
cargo vendor                                # 依赖源码复制到 vendor/
cargo vendor vendor > .cargo/config.toml    # 同时输出配置片段
cargo build --offline                        # 强制离线构建
```

```toml
# .cargo/config.toml
[source.crates-io]
replace-with = "vendored-sources"

[source.vendored-sources]
directory = "vendor"
```

★ **关键认知**：`cargo vendor` 把依赖源码**完整复制进仓库**，配合源替换实现真正的离线构建和"依赖固化"。这适合内网、审计要求高、需要长期归档的场景。⚠ vendor 目录会显著增大仓库体积（几百 MB 很常见），且每次改依赖都要重新 vendor 并提交。

## 3.2 .cargo/config.toml 的优先级

```text
配置优先级（从低到高）：
  1. $CARGO_HOME/config.toml           全局
  2. 项目根 .cargo/config.toml          项目
  3. 环境变量（CARGO_*）
  4. 命令行参数
```

★ **关键认知**：项目级 `.cargo/config.toml` 会覆盖全局配置，所以镜像源、linker、rustflags 这类环境相关设置，团队协作时要明确"写在哪一层"。⚠ 把个人镜像配置写进项目级 config 并提交，会强迫所有队友用同一个源，可能反而变慢或不可用。

## 3.3 离线排查

```text
报错：failed to get `serde` as a dependency ... offline mode
原因：本地缓存里没有该依赖，且 --offline 禁止联网
解决：联网跑一次 cargo fetch 预热，或配置 vendor，或去掉 --offline
```

```bash
cargo fetch                                      # 只下载依赖，不编译
cargo fetch --target x86_64-unknown-linux-gnu    # 预取指定平台的依赖
```

# 4. 交叉编译

## 4.1 基本流程

```bash
rustup target add aarch64-unknown-linux-gnu    # 安装目标平台标准库
cargo build --target aarch64-unknown-linux-gnu
```

```text
交叉编译三要素：目标三元组、目标平台标准库、目标平台链接器/C 工具链。
```

★ **关键认知**：`rustup target add` 只装了**标准库**，不装链接器。链接阶段需要一个能产出目标平台二进制的链接器（如 `aarch64-linux-gnu-gcc`），否则会报 `linker not found`。⚠ 纯 Rust 依赖好办，一旦依赖里有 C 代码（`*-sys` 系列），还要配对应的交叉 C 编译器。

## 4.2 配置链接器

```toml
# .cargo/config.toml
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"

[target.x86_64-pc-windows-gnu]
linker = "x86_64-w64-mingw32-gcc"

[target.x86_64-unknown-linux-gnu]
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=lld"]
```

## 4.3 交叉编译的常见障碍

| 障碍 | 表现 | 解决 |
|------|------|------|
| 缺标准库 | `can't find crate for std` | `rustup target add` |
| 缺链接器 | `linker not found` | 装交叉工具链并配置 |
| C 依赖 | `cc` 找不到编译器 | 设 `CC_<target>` |
| 系统库 | `cannot find -lfoo` | 交叉编译该库或用纯 Rust 替代 |
| pkg-config | 找到宿主机库 | 设 `PKG_CONFIG_ALLOW_CROSS` / sysroot |

★ **关键认知**：交叉编译失败十有八九卡在**链接**而不是编译。`rustc` 能编出目标平台的 `.o`，但链接器、C 工具链、系统库必须齐备。⚠ `cc` crate 会自动读取 `CC_<target>` 环境变量（下划线和连字符的写法都要注意），配错名字会静默回退到宿主机编译器，产出错误架构的二进制。

# 5. 常见依赖错误

## 5.1 版本解析失败

```text
报错：failed to select a version for the requirement `foo = "^2.0"`
      candidate versions found which didn't match: 1.9.0, 1.8.0
原因：crates.io 上没有 2.x 版本，或镜像源索引过期
解决：cargo search foo 查真实版本；放宽约束；检查镜像同步状态
```

## 5.2 重复依赖

```bash
cargo tree -d                # 列出重复 crate
cargo tree -i serde          # 谁依赖了 serde
```

```text
典型原因：版本范围互斥、上游长期未升级、git 依赖与 crates.io 版本并存。
影响：编译变慢、体积增大、类型不兼容（两个版本的 T 不是同一个 T）。
```

★ **关键认知**：重复依赖最隐蔽的危害是**类型不兼容**：`serde 1.0` 的 `Serialize` 和 `serde 0.9` 的 `Serialize` 是两个不同的 trait，混用会报"trait not implemented"，哪怕名字一模一样。⚠ 排查时先 `cargo tree -d` 定位是哪个上游引入了旧版本，再决定是升级上游、用 `[patch]` 覆盖，还是接受它。

## 5.3 [patch] 覆盖依赖

```toml
[patch.crates-io]
serde = { git = "https://github.com/serde-rs/serde", branch = "master" }
foo = { path = "../foo" }          # 或指向本地路径做临时调试
```

★ **关键认知**：`[patch]` 能在不改动依赖树结构的前提下，把某个 crate 换成另一个来源（git / path）。它常用于"上游还没发版，但我需要这个修复"。⚠ `[patch]` 的版本必须与原来要求的版本**兼容**，否则 Cargo 会忽略它；调试完记得移除，否则发布时会有隐患。

# 6. 常见链接错误

```text
错误 1：undefined reference to `foo`
原因：声明了 extern "C" 函数，但没链接实现，或链接顺序不对
解决：检查 rustc-link-lib、C 源文件是否编译、符号名是否匹配

错误 2：cannot find -lfoo
原因：链接器搜索路径里没有 libfoo
解决：加 rustc-link-search，或安装系统库开发包（-dev / -devel）

错误 3：duplicate symbol
原因：同一个符号被链接了两次（常见于 links 配错、静态库重复）
解决：检查 links 字段，确认原生库只被一个 crate 链接
```

★ **关键认知**：链接错误的关键信息往往在**第一行**——`undefined reference` 后面跟的符号名，直接告诉你是哪个函数没找到。用 `nm -C libfoo.a | grep 符号名` 能确认库到底导出了什么。⚠ 不要一上来就怀疑 Rust 代码，90% 的链接错误是"库没找到"或"符号名不匹配"（C++ 的 name mangling 尤其常见）。

# 7. 一个排查流程图

```text
构建失败
   │
   ├─ 编译错误（error[E0xxx]）？──> 代码/类型问题，按报错改
   │
   ├─ 依赖解析错误？
   │     ├─ 版本找不到 → 查 crates.io / 放宽约束
   │     └─ 重复依赖 → cargo tree -d → 升级/[patch]
   │
   ├─ build.rs 失败？──> cargo build -vv 看 stderr → 缺工具链/库
   │
   ├─ 链接错误？
   │     ├─ cannot find -l → 补搜索路径
   │     └─ undefined reference → 查符号与链接顺序
   │
   └─ 环境问题？
         ├─ 离线 → cargo fetch / vendor
         └─ 交叉编译 → 目标标准库 + 链接器 + CC
```

# 8. 练习

1. 给一个项目写 GitHub Actions 配置，包含 fmt、clippy、test、三组 feature 测试，并用 `Swatinem/rust-cache` 缓存。
2. 用 `cargo vendor` 把依赖固化到本地，断网后 `cargo build --offline` 验证能构建。
3. 故意引入一个重复依赖（同时依赖同一 crate 的两个 0.x 版本），用 `cargo tree -d` 定位并用 `[patch]` 或升级消除。

# 9. 小结

| 主题 | 要点 |
|------|------|
| CI 顺序 | fmt → clippy → test → build，快的放前面 |
| 矩阵 | 跨平台/跨版本，抓平台相关 bug |
| feature 测试 | 默认 / 无默认 / 全开三组 |
| 缓存 key | 用 `Cargo.lock` 哈希，按工具链分 |
| vendor | 完全离线、依赖固化，代价是仓库变大 |
| 交叉编译 | 标准库 + 链接器 + C 工具链三件套 |
| 依赖冲突 | `cargo tree -d` + `[patch]` |
| 链接错误 | 先看符号名，`-vv` 看完整命令 |

# 10. 模块总结

Cargo 这个模块从"一条命令怎么用"讲到"整个生态怎么协作"，脉络可以收成一条线：单项目（命令、manifest、profile）→ 多项目（workspace、features、共享依赖）→ 发布（package、publish、yank）→ 工程化（CI、缓存、交叉编译）。

| 篇目 | 核心收获 |
|------|---------|
| 1 概览与命令 | 知道每条命令在干什么、产物在哪 |
| 2 清单与依赖 | 版本语义、依赖分类、来源 |
| 3 workspace | 多 crate 共享锁与缓存 |
| 4 feature | 加法原则、resolver 2、避免冲突 |
| 5 profile | opt/lto/codegen-units 的取舍 |
| 6 build.rs | 编译期脚本、OUT_DIR、rerun-if-changed |
| 7 发布 | 不可变注册中心、yank、文档 |
| 8 CI 与排障 | 缓存、离线、交叉编译、链接错误 |

Cargo 的复杂度不在命令本身，而在**版本、feature、构建环境**这三张网的交叉点。真正用好它的标志，不是背下所有参数，而是遇到"编不过/编得慢/编出来不对"时，能快速判断问题落在哪一层：依赖解析、编译、链接，还是环境。下一篇可以回到 Rust 语言主线，继续深入所有权与生命周期的细节。
