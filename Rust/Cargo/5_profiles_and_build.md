同一个程序，`cargo build` 和 `cargo build --release` 出来的东西性能能差几十倍，体积也可能差一个数量级。这背后是 **profile** 在控制编译器的优化策略。理解 profile，你才能在"编得快"和"跑得快、体积小"之间做出有依据的取舍，而不是无脑加 `lto = true`。

# 1. profile 是什么

## 1.1 内置的四种 profile

| profile | 触发方式 | 默认特征 |
|---------|---------|---------|
| `dev` | `cargo build` | 无优化、带调试符号、溢出检查 |
| `release` | `cargo build --release` | `opt-level=3`、无调试信息、溢出回绕 |
| `test` | `cargo test` | 继承 dev，但开启测试 harness |
| `bench` | `cargo bench` | 继承 release |

```toml
# Cargo.toml
[profile.dev]
opt-level = 0

[profile.release]
opt-level = 3
```

★ **关键认知**：`dev` 和 `release` 的差异**不只是性能**。dev 下整数溢出会 panic（帮你尽早发现 bug），release 下会静默回绕（wrapping）；dev 带完整调试符号，release 默认没有。⚠ 所以"我本地 release 跑没问题"不能替代 dev 下的测试——两种 profile 的行为可能不同。

## 1.2 dev 的默认值

```toml
[profile.dev]
opt-level = 0            # 不优化，编译最快
debug = true             # 完整调试符号
debug-assertions = true  # 开启 debug_assert! 和溢出检查
overflow-checks = true   # 整数溢出 panic
incremental = true       # 增量编译
codegen-units = 256      # 大量并行单元，编译快
```

## 1.3 release 的默认值

```toml
[profile.release]
opt-level = 3            # 最高优化
debug = false            # 无调试符号
debug-assertions = false
overflow-checks = false  # 溢出回绕
incremental = false      # 关增量，保证优化一致性
codegen-units = 16       # 少量并行单元，优化更好
lto = false              # 默认不做跨 crate LTO
panic = "unwind"         # panic 时展开栈
strip = "none"           # 保留符号
```

★ **关键认知**：`opt-level = 3` 是"以速度为目标"，但编译器不总是越大越好。`opt-level = "s"` 优化体积，`opt-level = "z"` 极致压缩体积（可能更慢）。⚠ 对 WebAssembly、嵌入式、CLI 工具，`"s"` / `"z"` 往往比 `3` 更合适——先明确你优化的是速度还是体积。

# 2. 关键参数逐个讲

## 2.1 opt-level

```toml
[profile.release]
opt-level = 3      # 速度优先（默认）
# opt-level = "s"  # 体积优先
# opt-level = "z"  # 极致体积
# opt-level = 1    # 轻度优化，编译快
```

| 取值 | 目标 | 适用 |
|------|------|------|
| `0` | 不优化 | dev |
| `1` | 轻度 | 编译时间敏感 |
| `2` | 较优 | 一般发布 |
| `3` | 最优速度 | 计算密集 |
| `"s"` | 体积 | wasm、嵌入式 |
| `"z"` | 极致体积 | 空间极受限 |

## 2.2 LTO（链接时优化）

```toml
[profile.release]
lto = true          # 等价于 "fat"：跨所有 crate 优化
# lto = "thin"      # 折中：并行、较快、效果接近 fat
# lto = "off"       # 关闭
```

```text
无 LTO：
  crate A ──编译──> A.o
  crate B ──编译──> B.o
  A.o + B.o ──链接──> 可执行文件（跨 crate 内联受限）

有 LTO：
  所有 crate 的 IR ──一起优化──> 可执行文件（可跨 crate 内联、去重）
```

★ **关键认知**：LTO 让编译器**跨 crate 边界**做内联和死代码消除，是榨取性能的常用手段。⚠ 代价是编译时间和内存显著上升，`lto = true`（fat）在大型项目上可能让链接阶段从几秒变成几分钟。日常发布用 `"thin"` 更平衡，只有追求极致性能才用 `true`。

## 2.3 codegen-units

```toml
[profile.release]
codegen-units = 1     # 整个 crate 一个代码生成单元，优化最彻底
```

```text
codegen-units = 16（默认）：
  crate 被切成 16 块并行编译 ──> 编译快，但跨块优化受限

codegen-units = 1：
  整块编译 ──> 优化更好，但编译慢、无法并行
```

★ **关键认知**：`codegen-units` 和 `lto` 是"编译速度 vs 运行性能"的两个旋钮。`codegen-units = 1` 配合 `lto = true` 是极致性能组合，但编译时间可能翻好几倍。⚠ 大多数项目用默认值即可，只有在基准测试证明有收益时才调。

## 2.4 strip

```toml
[profile.release]
strip = true          # 去除所有符号（等价 "symbols"）
# strip = "debuginfo" # 只去调试信息，保留符号表
# strip = "none"      # 不去（默认）
```

★ **关键认知**：`strip = true` 能显著减小二进制体积（常常减半），但去掉符号后**崩溃栈里只剩地址**，排查线上问题会很痛苦。⚠ 推荐做法：发布时 `strip = "debuginfo"` 保留符号表，或单独保存一份带符号的构建产物用于事后分析。

## 2.5 panic 策略

```toml
[profile.release]
panic = "abort"      # panic 直接 abort，不展开栈
# panic = "unwind"   # 默认，panic 时展开栈，可被 catch_unwind 捕获
```

| 策略 | 体积 | 行为 | 限制 |
|------|------|------|------|
| `unwind` | 较大 | 展开栈，`catch_unwind` 可用 | 默认 |
| `abort` | 较小 | 立即终止进程 | 不能用 `catch_unwind` |

★ **关键认知**：`panic = "abort"` 能减小体积、略微提速，但会**禁用 `catch_unwind`**，也会让依赖里依赖展开行为的库（如某些测试框架）出问题。⚠ 库 crate 不应该设置 `panic` 策略——它只对最终二进制生效，库里写会被忽略或警告。

# 3. 自定义 profile

## 3.1 派生 profile

```toml
[profile.release-lto]
inherits = "release"      # 继承 release 的全部设置
lto = true
codegen-units = 1
```

```bash
cargo build --profile release-lto
```

★ **关键认知**：`inherits` 让你在保留 release 默认值的基础上只改几项，避免重复抄一整段配置。自定义 profile 的输出目录是 `target/<profile名>/`，不是 `target/release/`。⚠ 自定义 profile 名字不能是 `dev` / `release` / `test` / `bench` 这些内置名。

## 3.2 按包覆盖 profile

```toml
# 只对某个依赖降低优化，加快编译
[profile.dev.package."*"]
opt-level = 2          # 依赖用 2，自己的代码用 0

[profile.dev.package.serde]
opt-level = 3
```

★ **关键认知**：`[profile.dev.package."*"]` 是加速 dev 构建的经典技巧——依赖通常不变，用较高优化编译一次后长期复用；自己的代码频繁改动，保持 `opt-level = 0` 编得快。⚠ 这会拉长首次构建时间，适合依赖多、迭代频繁的中大型项目。

## 3.3 build 覆盖

```toml
[profile.dev.build-override]
opt-level = 3          # 让 build.rs 和过程宏编译得更快执行
```

`build-override` 控制 build script 和 proc-macro 的编译参数。它们在编译期执行，优化它们能加快整体构建。

# 4. 构建缓存

## 4.1 缓存的位置与失效

```text
target/debug/
├── deps/            # 依赖产物（.rlib / .rmeta）
├── incremental/     # 增量编译缓存
├── .fingerprint/    # 每个 crate 的"指纹"，判断是否需要重编
└── build/           # build.rs 输出
```

Cargo 通过 **fingerprint（指纹）** 判断某个 crate 是否需要重编：源码哈希、依赖版本、编译参数、环境变量、Rust 版本……任一变化都会导致失效。

★ **关键认知**：Cargo 的缓存粒度是 **crate**，不是文件。改一行代码，只有你所在的 crate 需要重编（加上依赖它的 crate），依赖本身不动。⚠ 但改了 `Cargo.toml`、feature、profile 或升级工具链，往往会导致大量 crate 重编——这也是"为什么我只改了一行却编了十分钟"的常见原因。

## 4.2 加速构建的手段

```bash
cargo build -j 8               # 限制并行任务数（默认用满 CPU）
cargo build --timings          # 生成构建耗时报告（target/cargo-timings/）
sccache                       # 跨项目共享编译缓存（需单独安装配置）
```

```toml
[profile.dev]
debug = 0                      # 减少调试信息生成，加快链接
```

★ **关键认知**：`cargo build --timings` 会生成一个 HTML 报告，直观展示每个 crate 的编译耗时和并行度。排查"构建慢"时，先看这个报告，而不是凭感觉加 `-j`。⚠ 常见瓶颈是链接阶段和 proc-macro 编译，前者可以换更快的链接器（mold/lld），后者可以通过 `build-override` 优化。

## 4.3 换链接器

```toml
# .cargo/config.toml
[target.x86_64-unknown-linux-gnu]
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=lld"]
```

```toml
# Windows + MSVC 可用 lld-link
[target.x86_64-pc-windows-msvc]
linker = "lld-link"
```

★ **关键认知**：大型项目的链接阶段经常是构建瓶颈，换成 `lld` 或 `mold` 能把链接时间从几十秒降到几秒。⚠ 换链接器属于"环境级配置"，写进 `.cargo/config.toml` 会影响所有项目，团队协作时要确认大家环境一致。

# 5. 常见错误

```text
错误 1：panic = "abort" 后 catch_unwind 编译失败
原因：abort 策略下没有栈展开
解决：改回 unwind，或移除 catch_unwind 依赖

错误 2：strip = true 后崩溃日志只有地址
原因：符号被剥离
解决：改用 strip = "debuginfo"，或单独保留符号文件

错误 3：自定义 profile 找不到产物
原因：产物在 target/<profile名>/，不是 target/release/
解决：确认路径，或用 cargo build --profile xxx --message-format=json 拿路径

错误 4：只改 profile 却触发全量重编
原因：profile 属于编译参数，会改变指纹
解决：属正常现象，接受一次全量；CI 里缓存 target/ 可缓解

错误 5：lto = true 后链接 OOM
原因：fat LTO 内存占用高
解决：改用 lto = "thin"，或增大机器内存/减少 codegen-units
```

# 6. 练习

1. 建一个计算密集的小程序，分别用 `dev`、`release`、`release + lto + codegen-units=1` 构建，用 `time` 对比运行耗时。
2. 给 release 加 `strip = true`，对比加与不加时的二进制体积（Windows 看 `.exe` 大小）。
3. 用 `cargo build --timings` 生成报告，找出构建中最耗时的 crate，尝试用 `[profile.dev.package."*"]` 调整优化级别。

# 7. 小结

| 参数 | 作用 | 建议 |
|------|------|------|
| `opt-level` | 优化目标 | 速度用 3，体积用 `"s"` / `"z"` |
| `lto` | 跨 crate 优化 | 日常 `"thin"`，极致用 `true` |
| `codegen-units` | 并行单元数 | 极致性能设 1，其余默认 |
| `strip` | 去符号 | 发布用 `"debuginfo"` 保留符号表 |
| `panic` | 展开/终止 | 二进制可 abort，库不要设 |
| `build-override` | 加速 build.rs | 依赖多时有用 |
| `--timings` | 构建分析 | 排查慢构建第一步 |

下一篇进入 build script 与环境变量，讲清楚 `build.rs` 的生命周期、如何链接系统库、如何做代码生成，以及 `rerun-if-changed` 为什么必须写。
