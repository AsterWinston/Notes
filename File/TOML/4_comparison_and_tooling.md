TOML 不是唯一的配置格式，选错格式的代价往往在半年后才显现：YAML 的隐式类型让 `NO` 变成布尔值，JSON 不能写注释，INI 没有嵌套。本篇把 TOML 和 YAML、JSON、INI 放在一起比较，再讲格式化、schema 校验和编辑器支持这套工具链。

# 1. 四种格式的定位

## 1.1 一句话对比

| 格式 | 最适合 | 不适合 | 代表文件 |
|------|--------|--------|---------|
| TOML | 人手写的应用配置 | 深层嵌套、多文档 | `Cargo.toml`、`pyproject.toml` |
| YAML | 复杂编排、多文档 | 需要严格类型的小配置 | Kubernetes、GitHub Actions |
| JSON | 机器传输、接口数据 | 需要注释的配置 | API 响应、`package.json` |
| INI | 简单节配置、遗留系统 | 嵌套、数组、日期 | Windows 应用配置 |

★ **关键认知**：格式没有绝对优劣，只有**结构与维护者的匹配度**。TOML 用"多写一点语法"换"少一点歧义"，适合配置项有限、需要人频繁修改的场景；⚠ 数据层级很深时，TOML 的表头会变得难读，这时 YAML 或 JSON 更合适。

## 1.2 TOML 与 YAML

| 维度 | TOML | YAML |
|------|------|------|
| 层级表达 | 表头 `[a.b]` | 缩进 |
| 类型 | 显式，字符串必须加引号 | 隐式推断，`NO` 可能是布尔 |
| 注释 | `#` | `#` |
| 锚点与复用 | 不支持 | 支持 `&` / `*` / `<<` |
| 多文档 | 不支持 | 支持 `---` |
| 数组 | 方括号，可跨行 | `-` 列表或行内数组 |
| 可读性 | 啰嗦但结构清楚 | 紧凑但缩进敏感 |
| 典型场景 | 工程配置 | 编排、CI、复杂清单 |

★ **关键认知**：YAML 的表达力更强，代价是**隐式规则更多**；TOML 的表达力弱一些，但几乎不会"解析成功却含义变了"。⚠ 不要因为 YAML 能写锚点就把 TOML 配置机械地转成 YAML，复用逻辑往往需要重新设计。

## 1.3 TOML 与 JSON

| 维度 | TOML | JSON |
|------|------|------|
| 注释 | 支持 | 不支持 |
| 键 | 可裸写 | 必须双引号 |
| 尾逗号 | 数组允许 | 不允许 |
| 日期时间 | 原生类型 | 只能字符串 |
| 根节点 | 必须是表 | 任意值 |
| `null` | 没有 | 有 |
| 典型场景 | 配置 | 传输 |

JSON 适合机器之间传递数据，因为解析器无处不在、格式足够简单。TOML 适合人改配置，因为注释、裸键、日期类型都能省事。

## 1.4 TOML 与 INI

INI 只有"节 + 键值"，没有标准类型、没有嵌套、没有数组。TOML 看起来像加强版 INI，但两者并不等价：

- TOML 的表可以嵌套，INI 的节通常是平的。
- TOML 有整数、浮点、布尔、日期，INI 的值基本是字符串。
- TOML 不允许重复键，INI 的实现则各有各的规则。
- TOML 用 `[[items]]` 表达表数组，INI 没有对应语法。

★ **关键认知**：把 INI 改成 TOML 不是改后缀那么简单，**值的类型和嵌套结构都要重新设计**。⚠ 反过来，把 TOML 降级成 INI 会丢失类型和层级，通常不可逆。

## 1.5 选型决策

```text
数据主要给机器读？
├── 是 → JSON（接口、缓存、日志）
└── 否 → 是否需要复杂编排 / 多文档 / 锚点？
    ├── 是 → YAML（K8s、CI、Compose）
    └── 否 → 是否是简单节配置的遗留系统？
        ├── 是 → INI
        └── 否 → TOML（工程配置、应用配置）
```

# 2. 格式化

## 2.1 为什么需要格式化

格式化解决三个问题：减少无意义的 diff、统一团队风格、让 review 聚焦在值而不是空格。TOML 没有官方格式化器，社区事实标准是 **Taplo**。

## 2.2 Taplo

安装方式任选：

```bash
cargo install taplo-cli --locked       # Rust 工具链
npm install -g @taplo/cli              # Node 环境
brew install taplo                     # macOS
```

常用命令：

```bash
taplo fmt config.toml                  # 就地格式化
taplo fmt --check                      # 只检查，不修改；CI 用
taplo lint config.toml                 # 结合 schema 做校验
```

项目级配置 `.taplo.toml`：

```toml
[formatting]
align_entries = false                  # 不对齐等号，减少无意义 diff
array_trailing_comma = true            # 多行数组保留尾逗号
compact_arrays = false                 # 数组不强制压成一行
```

★ **关键认知**：格式化只改变排版，**不改变语义，也不做业务校验**。`taplo fmt` 通过不代表配置能启动；⚠ 反过来，格式不一致也不代表配置有错，别把两者混为一谈。

## 2.3 其他格式化工具

| 工具 | 形态 | 说明 |
|------|------|------|
| Taplo | CLI + LSP | TOML 生态最完整，格式化、lint、schema 一体 |
| dprint | CLI + 插件 | 多语言统一格式化，适合已有 dprint 的团队 |
| Prettier 插件 | Node 插件 | 前端团队顺手使用，TOML 支持依赖社区插件 |

团队只需要选一个，并在 CI 里固定版本。不要让每个人用不同工具互相覆盖格式。

# 3. Schema 校验

## 3.1 为什么 TOML 需要 schema

TOML 解析器只检查语法。`port = "8080"` 是合法 TOML，但业务需要的是整数；`[server]` 少写一个 `host` 也能解析，但服务起不来。Schema 负责**结构、类型、必填项、取值范围**。

## 3.2 Taplo + JSON Schema

写一份 `config.schema.json`：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["server"],
  "properties": {
    "server": {
      "type": "object",
      "required": ["host", "port"],
      "properties": {
        "host": { "type": "string" },
        "port": { "type": "integer", "minimum": 1, "maximum": 65535 }
      },
      "additionalProperties": false
    }
  }
}
```

在 TOML 文件顶部用指令关联 schema：

```toml
#:schema ./config.schema.json

[server]
host = "0.0.0.0"
port = 8080
```

然后运行：

```bash
taplo lint config.toml
```

★ **关键认知**：`#:schema` 是 Taplo 的约定，不是 TOML 标准。换一个解析器，这行只是普通注释；⚠ schema 路径通常相对于 TOML 文件本身，不是相对于当前工作目录，CI 里尤其要注意。

## 3.3 应用内校验

Schema 能挡住大部分结构和类型问题，但跨字段规则、环境相关规则仍要在应用里做：

- Python：Pydantic 模型 + `Field` 约束。
- Rust：`serde` 反序列化 + `deny_unknown_fields`，必要时用 `validator`。
- 启动时校验失败就退出，不要带着半份配置继续跑。

★ **关键认知**：schema 校验是**第一道门**，应用校验是**最后一道门**。⚠ 只做其中一道都不够：只有 schema 挡不住业务规则，只有应用校验则错误发现得太晚。

# 4. 编辑器与 CI

## 4.1 VS Code

安装 **Even Better TOML**（Taplo 的 VS Code 封装），即可获得补全、悬停、格式化和 schema 校验。可以在 `settings.json` 里关联 schema：

```json
{
  "evenBetterToml.schema.associations": {
    "config/*.toml": "./schemas/config.schema.json"
  }
}
```

## 4.2 JetBrains

IntelliJ IDEA、CLion、PyCharm 安装 **TOML** 插件，支持语法高亮、格式化和 JSON Schema 映射。在 `Settings > Languages & Frameworks > Schemas and DTDs > JSON Schema Mappings` 里把 schema 绑到文件模式。

## 4.3 Neovim / Helix

用 Taplo 的 LSP：

```bash
taplo lsp stdio
```

在 Neovim 的 `lspconfig` 或 Helix 的 `languages.toml` 里把 `toml` 的 language server 指向该命令，即可获得诊断和补全。

## 4.4 CI 中固定检查

```yaml
name: toml-check
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm install -g @taplo/cli
      - run: taplo fmt --check
      - run: taplo lint
```

★ **关键认知**：编辑器里的实时提示是**开发体验**，CI 里的检查才是**质量门禁**。⚠ 只依赖编辑器，意味着没装插件的人可以提交格式错误或 schema 不合法的配置。

# 5. 工具对比

| 工具 | 格式化 | Schema 校验 | LSP | 适合 |
|------|--------|------------|-----|------|
| Taplo | 是 | 是 | 是 | 大多数 TOML 项目 |
| dprint | 是 | 否 | 否 | 已有 dprint 的多语言仓库 |
| Prettier 插件 | 是 | 否 | 否 | 前端项目顺手格式化 |
| `toml-test` | 否 | 否 | 否 | 测试 TOML 解析器实现 |

# 6. 常见问题

| 问题 | 原因 | 处理 |
|------|------|------|
| `taplo lint` 通过但程序启动失败 | schema 不覆盖业务规则 | 增加应用层校验 |
| CI 报 schema 找不到 | 路径相对位置写错 | 用相对 TOML 文件的路径 |
| 格式化产生巨大 diff | 团队此前没统一格式 | 先单独提交一次格式化 |
| 编辑器有提示但 CI 不检查 | 只装了插件 | 在 CI 跑 `taplo fmt --check` 和 `lint` |
| Cargo.toml 通过 TOML 校验却构建失败 | Cargo 有额外语义规则 | 用 `cargo metadata` 验证 |
| YAML 转 TOML 后锚点消失 | TOML 不支持锚点 | 展开复用内容或改加载逻辑 |

# 7. 动手练习

1. 把一段包含注释、数组和嵌套映射的 YAML 改写成 TOML，列出转换过程中丢失或需要重新表达的特性。

2. 给一份 `config.toml` 写 JSON Schema，要求 `port` 在 1 到 65535 之间，并用 `taplo lint` 验证一个越界值。

3. 在项目里配置 `taplo fmt --check` 和 `taplo lint` 的 CI 步骤，故意提交一个格式错误和一个类型错误，确认两者都能被拦住。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 选型 | 配置优先 TOML，编排用 YAML，传输用 JSON，遗留用 INI |
| 格式化 | Taplo 是事实标准，CI 用 `--check` |
| Schema | `#:schema` 关联 JSON Schema，`taplo lint` 校验 |
| 分层校验 | schema 管结构，应用管业务 |
| 编辑器 | VS Code / JetBrains / Neovim 都可通过 Taplo 获得 LSP |
| CI | 格式与 schema 都要卡，不能只靠编辑器 |

下一篇进入校验与总结，讲清楚类型边界、版本兼容、常见坑，并用一组自检题把整个 TOML 模块串起来。
