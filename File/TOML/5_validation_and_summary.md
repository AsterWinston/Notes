TOML 解析器说"语法正确"，只代表文件能被读成字典，不代表配置能用。端口写成字符串、时间少了时区、整数超出目标类型，这些问题都要靠校验和类型边界来兜底。本篇讲 schema 与类型校验、TOML 1.0 的版本兼容、常见坑，并用自检题把整个模块串起来。

# 1. 为什么"能解析"不等于"配置正确"

## 1.1 三层校验

```text
第一层：语法
  TOML 解析器能读吗？
  例：引号是否闭合、表是否重复定义
       │
       ▼
第二层：结构与类型
  字段是否齐全、类型是否匹配？
  例：port 必须是整数，server 必须有 host
       │
       ▼
第三层：业务语义
  值是否合理、字段之间是否一致？
  例：port 在 1-65535，生产环境不能使用默认密码
```

★ **关键认知**：三层校验分别由不同工具负责。解析器只管第一层，schema 管第二层，应用代码管第三层。⚠ 只做第一层就上线，等于把配置错误推迟到运行时。

## 1.2 类型错误的代价

```toml
[server]
port = "8080"          # 字符串，不是整数
```

Python 里 `config["server"]["port"] + 1` 会抛 `TypeError`；Rust 反序列化到 `u16` 会直接报 `invalid type: string "8080"`。表面上只是多了两个引号，实际会让服务在启动或第一个请求时失败。

```toml
[start]
at = "2026-01-01T00:00:00Z"   # 加了引号，变成字符串
```

去掉引号才是日期类型。★ **关键认知**：TOML 的类型由**写法**决定，不由字段名决定。⚠ 字段叫 `port` 不代表值一定是数字，叫 `at` 也不代表值一定是时间。

# 2. Schema 与类型校验

## 2.1 JSON Schema 约束

JSON Schema 可以描述 TOML 的结构。下面这份 schema 要求 `server` 必填，`port` 是 1 到 65535 的整数，`log_level` 只能取枚举值：

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
        "host": { "type": "string", "minLength": 1 },
        "port": { "type": "integer", "minimum": 1, "maximum": 65535 }
      },
      "additionalProperties": false
    },
    "log_level": {
      "type": "string",
      "enum": ["debug", "info", "warn", "error"]
    }
  }
}
```

关联到 TOML：

```toml
#:schema ./config.schema.json

log_level = "info"        # 根表键必须写在第一个表头之前

[server]
host = "0.0.0.0"
port = 8080
```

运行 `taplo lint config.toml`，越界、缺字段、多余字段都会报出来。

## 2.2 应用层校验

Schema 难以表达跨字段规则，应用层还要再校验一次。Python 用 Pydantic：

```python
from typing import Literal
from pydantic import BaseModel, Field

class Server(BaseModel):
    host: str = Field(min_length=1)
    port: int = Field(ge=1, le=65535)

class Config(BaseModel):
    server: Server
    log_level: Literal["debug", "info", "warn", "error"] = "info"

config = Config(**toml_data)      # 类型或范围不对会立即抛异常
```

Rust 用 `serde` 加类型约束：

```rust
#[derive(Debug, serde::Deserialize)]
#[serde(deny_unknown_fields)]
struct Server {
    host: String,
    port: u16,          // 天然限制在 0-65535，仍需额外排除 0
}
```

★ **关键认知**：`deny_unknown_fields`、Pydantic 的 `extra="forbid"`、schema 的 `additionalProperties: false` 是同一个目的：**让拼错的字段尽早暴露**。⚠ 默认忽略未知字段会让 `prot = 8080` 静默失效，比直接报错更难查。

## 2.3 校验时机

| 时机 | 工具 | 作用 |
|------|------|------|
| 编辑时 | 编辑器 + Taplo LSP | 即时提示 |
| 提交前 | pre-commit | 拦住格式和 schema 错误 |
| CI | `taplo fmt --check`、`taplo lint` | 统一门禁 |
| 启动时 | Pydantic / serde / 手写校验 | 挡住业务规则 |
| 运行时 | 监控与告警 | 发现配置漂移 |

# 3. 类型与边界

## 3.1 整数

TOML 1.0 的整数按 **64 位有符号整数**设计：

```toml
small = -42
big = 9_223_372_036_854_775_807
hex = 0xFF
octal = 0o755
binary = 0b1010
```

- 十进制整数不允许前导零，`0123` 非法。
- 下划线只能放在数字之间，`1_000` 合法，`_1000` 非法。
- Rust 映射到 `i64` 以外的类型（如 `u16`）时，超出范围会反序列化失败。

⚠ **关键认知**：Python 的 `int` 本身没有固定位宽，但不要因此依赖超范围值。跨语言传递时，对方可能只支持 64 位，配置应在所有目标运行时都能表示。

## 3.2 浮点数

```toml
ratio = 0.75
scientific = 6.022e23
infinity = inf
not_a_number = nan
```

金额不要用浮点数：

```toml
# 不推荐
price = 19.99

# 推荐：用整数分，或明确使用十进制类型
price_cents = 1999
```

★ **关键认知**：`0.1 + 0.2` 在二进制浮点里不等于 `0.3`，金额、税率、积分这类需要精确小数的字段，应该用整数最小单位或 `Decimal` / `rust_decimal`。⚠ `inf` 和 `nan` 是合法 TOML，但很多 JSON 序列化器无法表示，跨格式转换时要特别处理。

## 3.3 日期时间

```toml
offset = 2026-01-01T00:00:00Z          # 带时区，绝对时刻
offset2 = 2026-01-01T08:00:00+08:00    # 同一时刻的另一种写法
local_dt = 2026-01-01T00:00:00         # 本地时间，无时区
local_date = 2026-01-01                # 只有日期
local_time = 00:00:00                  # 只有时间
```

Python 中带偏移的会得到 aware `datetime`，本地日期时间得到 naive `datetime`。Rust 中用 `toml::value::Datetime` 保留原始信息。

★ **关键认知**：`local_dt` 不是"默认 UTC"，它只是没有时区信息。⚠ 服务器分布在多个时区时，同一个 `local_dt` 含义不同；需要绝对时刻就写偏移，需要业务本地时间就明确业务时区。

## 3.4 字符串

```toml
basic = "第一行\n第二行"        # 支持转义
literal = 'C:\Users\alice'      # 原样保留，适合路径和正则
multi = """
多行文本
"""
```

- 文件必须是 UTF-8。
- 基本字符串里 `\` 是转义符，Windows 路径要么双写，要么用单引号。
- 多行字符串会去掉紧跟开引号的第一个换行。
- 不要在文件开头加 BOM，部分解析器会把它当成非法字符。

# 4. 版本兼容

## 4.1 TOML 1.0 与旧版

TOML 1.0 在 2021 年发布，是目前大多数解析器的目标版本。几个兼容点：

| 版本 | 状态 | 注意事项 |
|------|------|---------|
| 0.4 | 旧版 | 不支持本地日期、本地时间等写法 |
| 0.5 | 旧版 | 部分解析器仍在使用，混合类型数组等行为可能不同 |
| 1.0 | 当前稳定版 | 本模块所有示例以此为准 |
| 1.1 | 草案 | 不要在生产配置里依赖未稳定的新特性 |

★ **关键认知**：TOML 1.0 允许混合类型数组，例如 `[1, "two"]`；老解析器可能拒绝。⚠ 如果配置要同时被多种语言读取，尽量只使用 1.0 的核心子集：字符串、数字、布尔、日期、同构数组、普通表、表数组。

## 4.2 常见解析器

| 语言 | 库 | 说明 |
|------|----|------|
| Python | `tomllib` | 3.11+ 标准库，只读；旧版本用 `tomli` |
| Rust | `toml` | 常与 `serde` 搭配，`toml_edit` 用于保留格式 |
| Go | `BurntSushi/toml`、`pelletier/go-toml/v2` | 后者性能更好，API 不同 |
| Java | `tomlj` | 读取为主，注意版本对 1.0 的支持 |
| JavaScript | `smol-toml`、`@iarna/toml` | 前端工具链常用 |

## 4.3 向前兼容策略

- 新配置优先使用 TOML 1.0 的稳定子集。
- 在 CI 里用目标语言的解析器各读一遍，不要只测一种。
- 固定解析器版本，升级时跑回归用例。
- 不依赖"某个解析器恰好接受"的宽松行为。

# 5. 常见坑清单

| 坑 | 现象 | 正确做法 |
|----|------|---------|
| 日期加引号 | 得到字符串 | 去掉引号 |
| 本地时间当 UTC | 跨时区时间偏移 | 明确时区，必要时写偏移 |
| 重复键 | 解析失败或覆盖行为不一 | 合并键，用 lint 检查 |
| 内联表跨行 / 尾逗号 | 解析失败 | 单行书写，去掉尾逗号 |
| 内联表后续扩展 | 重复定义错误 | 改用标准表 |
| 表数组子表挂错元素 | 配置挂到上一个元素 | 每个 `[[...]]` 后重新声明 |
| Windows 路径用双引号 | `\U` 被当转义 | 用单引号或双写反斜杠 |
| 想写 `null` | 解析失败 | 省略键，由程序给默认值 |
| 数组合并误以为会拼接 | 线上列表变短 | 环境文件写完整列表 |
| 环境变量覆盖后不转类型 | 数字变字符串 | 覆盖后重新校验 |
| 用浮点存金额 | 精度误差 | 整数分或 Decimal |
| 把 TOML 校验当 Cargo 校验 | 清单语义错误没发现 | 跑 `cargo metadata` |

# 6. 自检题

1. **问**：第一个表头之后的键属于哪张表？ **答**：属于该表头声明的表，直到下一个表头出现；根表键必须写在第一个表头之前。
2. **问**：`[[servers]]` 后面的 `[servers.tls]` 会挂到哪个元素？ **答**：最近一个 `[[servers]]` 元素，不是全部元素。
3. **问**：`at = "2026-01-01T00:00:00Z"` 是什么类型？ **答**：字符串，不是日期时间。
4. **问**：`2026-01-01T00:00:00` 和 `2026-01-01T00:00:00Z` 有什么区别？ **答**：前者是本地时间，没有时区；后者是带偏移的绝对时刻。
5. **问**：内联表 `point = { x = 1 }` 之后还能写 `point.y = 2` 吗？ **答**：不能，内联表定义后封闭。
6. **问**：TOML 有 `null` 吗？ **答**：没有，可选字段通过"允许缺失"表达。
7. **问**：深度合并时数组会拼接吗？ **答**：不会，数组整体替换。
8. **问**：`tomllib` 能用文本模式打开文件吗？ **答**：不能，必须用二进制模式。

# 7. 综合练习

1. 找出下面配置的错误并改正：

   ```toml
   name = demo
   port = 08080
   path = "C:\logs\app.log"
   [server]
   host = "0.0.0.0"
   [server]
   timeout = null
   at = "2026-01-01T00:00:00Z"
   ```

2. 为一份包含 `[server]`、`[database]`、`[[upstreams]]` 的配置写 JSON Schema，要求必填字段、端口范围、枚举值和 `additionalProperties: false`，然后用 `taplo lint` 验证。

3. 设计一个兼容性检查脚本：用 Python 的 `tomllib` 和 Rust 的 `toml` 各读同一份配置，确认两边都能解析，并比较日期字段的类型差异。

# 8. 小结

| 主题 | 关键结论 |
|------|---------|
| 三层校验 | 语法 → 结构类型 → 业务语义 |
| Schema | 管结构、必填、范围、未知字段 |
| 应用校验 | 管跨字段规则和业务约束 |
| 整数 | 64 位有符号，注意目标类型范围 |
| 浮点 | 金额不用浮点，注意 `inf` / `nan` |
| 日期 | 带偏移是绝对时刻，本地时间无时区 |
| 字符串 | UTF-8，路径优先用单引号 |
| 版本 | 以 1.0 为准，保守使用核心子集 |
| 工具 | Taplo 管格式与 schema，CI 做门禁 |

TOML 模块到这里结束。下一步可以回到 File 板块的 JSON、YAML 篇做横向对比，或把本篇的 schema 与 CI 检查直接接到自己的项目里。
