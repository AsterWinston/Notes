TOML 的语法只是前半程，真正落到项目里是"用什么库读、读出来是什么类型、写回时会不会丢注释"。Python 3.11 起标准库自带 `tomllib`，Rust 生态则习惯用 `toml` + `serde` 把配置映射成结构体。本篇用同一份配置走一遍两条路线，再看 `pyproject.toml` 和 `Cargo.toml`。

# 1. Python 读取 TOML

## 1.1 tomllib 的定位

Python 3.11 把 TOML 解析器收进了标准库，模块名是 `tomllib`。它只负责**读取**，不负责写入。

- `tomllib.load(fp)` 接收**二进制**文件对象，不是文本模式。
- `tomllib.loads(s)` 接收字符串，适合处理网络或内存里的内容。
- 低于 3.11 的版本需要安装第三方库 `tomli`，API 基本一致。

★ **关键认知**：标准库只读不写是刻意设计。TOML 写回涉及注释、空行、键顺序等格式保留问题，标准库不想绑定一种风格；需要写入时再选专门的库。

## 1.2 最小读取示例

```toml
title = "demo"

[database]
host = "127.0.0.1"
port = 5432
timeout_ms = 3_000

[[servers]]
name = "alpha"

[[servers]]
name = "beta"
```

```python
import tomllib
from pathlib import Path

with Path("config.toml").open("rb") as f:   # 必须用 rb，不能用 r
    config = tomllib.load(f)
print(config["title"])                      # demo
print(config["database"]["port"])           # 5432
print(config["servers"][0]["name"])         # alpha
```

⚠ **关键认知**：把 `open("config.toml")` 写成文本模式会得到 `TypeError`，因为 `tomllib` 需要自己按 UTF-8 解码字节流。

## 1.3 类型映射

| TOML 类型 | Python 类型 | 说明 |
|-----------|-------------|------|
| 字符串 | `str` | 统一为 Unicode 字符串 |
| 整数 | `int` | Python 整数无固定位宽 |
| 浮点数 | `float` | 可用 `parse_float` 换成 `Decimal` |
| 布尔 | `bool` | `true` / `false` |
| 带偏移日期时间 | `datetime.datetime` | `tzinfo` 不为空 |
| 本地日期时间 | `datetime.datetime` | `tzinfo` 为空 |
| 本地日期 | `datetime.date` | 只有年月日 |
| 本地时间 | `datetime.time` | 只有时分秒 |
| 数组 | `list` | 元素类型可混合 |
| 表 | `dict` | 键为字符串 |
| 表数组 | `list[dict]` | 每个元素是一张表 |

★ **关键认知**：TOML 的"本地日期时间"和"带偏移日期时间"都会变成 `datetime.datetime`，区别只在 `tzinfo`。⚠ 直接比较两者会抛 `TypeError`，跨时区逻辑必须显式补时区。

金额或高精度小数用 `parse_float` 避免二进制浮点误差：

```python
from decimal import Decimal

config = tomllib.loads(text, parse_float=Decimal)
price = config["price"]        # Decimal("19.99")
```

## 1.4 错误处理

解析失败时抛 `tomllib.TOMLDecodeError`，异常信息里通常带行列位置：

```python
import tomllib
try:
    data = tomllib.loads("port = 08080")   # 前导零，非法
except tomllib.TOMLDecodeError as exc:
    print(f"配置格式错误：{exc}")
```

配置读不出来意味着服务无法按预期启动，**快速失败**比带着半份配置运行安全。

## 1.5 写回 TOML

需要生成或修改 TOML 时，标准库帮不上忙：

| 库 | 特点 | 适用场景 |
|----|------|---------|
| `tomli-w` | 轻量，API 简单，不保留注释 | 程序生成新配置 |
| `tomlkit` | 保留注释、空行和键顺序 | 修改已有配置文件 |

```python
import tomlkit
from pathlib import Path

doc = tomlkit.parse(Path("config.toml").read_text(encoding="utf-8"))
doc["database"]["port"] = 5433          # 注释和排版保持不变
Path("config.toml").write_text(tomlkit.dumps(doc), encoding="utf-8")
```

★ **关键认知**：配置文件是**人写的**时，优先用 `tomlkit` 做修改；`tomli-w` 会把注释和原有排版全部丢掉。⚠ 用通用 TOML 库批量改写 `Cargo.toml`、`pyproject.toml` 也有同样风险，Rust 侧对应的是 `toml_edit`。

## 1.6 pyproject.toml 示例

`pyproject.toml` 是 PEP 518 / PEP 621 定义的 Python 项目元数据文件，本质仍是 TOML：

```toml
[build-system]
requires = ["hatchling"]              # 构建后端依赖
build-backend = "hatchling.build"     # 真正执行打包的模块

[project]
name = "order-service"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
  "httpx>=0.27",
  "pydantic>=2.7,<3",
]
```

逐段说明：

- `[build-system]` 告诉 pip 用哪个后端构建，缺了它 pip 会回退到旧式 `setup.py`。
- `[project]` 是 PEP 621 的标准元数据，`dependencies` 里每项都是 PEP 508 依赖表达式。
- 实际项目还可以增加 `[project.optional-dependencies]`（可选依赖组）和 `[project.scripts]`（命令行入口）。
- `[tool.*]` 是各工具自己的命名空间，TOML 不限制内容，工具自己校验。

★ **关键认知**：TOML 只保证"语法合法"，`pyproject.toml` 的字段含义由 PEP 和构建后端决定。⚠ 字段名写错、版本约束写错，TOML 解析器不会报错，要到 `pip install` 或构建阶段才暴露。

# 2. Rust 使用 toml + serde

## 2.1 两个库的分工

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
toml = "0.8"
```

- `toml` 负责 TOML 语法解析和序列化。
- `serde` 负责把解析出来的数据映射到你的结构体。

★ **关键认知**：`toml` 不强制你用 `serde`，也可以直接操作 `toml::Value`；但项目配置几乎都会定义结构体，因为**编译期字段检查**比运行时 `value["server"]["port"].as_integer()` 可靠得多。

## 2.2 定义结构体并反序列化

```toml
[server]
host = "0.0.0.0"
port = 8080
workers = 4
[database]
url = "postgres://localhost/app"
read-timeout-ms = 5_000
```

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]        // 出现未声明字段时直接报错
struct Config {
    server: Server,
    database: Database,
}

#[derive(Debug, Deserialize)]
struct Server {
    host: String,
    port: u16,
    #[serde(default = "default_workers")]
    workers: usize,
}
fn default_workers() -> usize { 4 }

#[derive(Debug, Deserialize)]
struct Database {
    url: String,
    #[serde(rename = "read-timeout-ms")]
    read_timeout_ms: u64,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let text = std::fs::read_to_string("config.toml")?;
    let config: Config = toml::from_str(&text)?;
    Ok(())
}
```

几个映射规则：

- TOML 表对应结构体，表数组对应 `Vec<T>`。
- `#[serde(default)]` 让字段缺失时使用默认值，也可以指定默认函数。
- 字段名和 TOML 键不一致时用 `#[serde(rename = "...")]`。
- 可选字段用 `Option<T>`，缺失时得到 `None`。

★ **关键认知**：`serde` 默认**忽略**结构体里没有声明的字段。想让拼错的键立刻暴露，加上 `#[serde(deny_unknown_fields)]`；⚠ 这个属性不会自动继承到嵌套结构体，需要逐层添加。

## 2.3 类型映射与日期

| TOML 类型 | Rust 常用类型 | 备注 |
|-----------|--------------|------|
| 字符串 | `String` / `&str` | 配置结构体一般用 `String` |
| 整数 | `i64`、`u16`、`usize` 等 | 超出目标类型范围会报错 |
| 浮点数 | `f64` | 金额建议用整数分或 `rust_decimal` |
| 布尔 | `bool` | 只接受 `true` / `false` |
| 日期时间 | `toml::value::Datetime` | 需要时再转 `chrono` 或 `time` |
| 数组 | `Vec<T>` | 元素类型由 `T` 决定 |
| 表 | 结构体或 `HashMap` | 键不固定时用 map |
| 表数组 | `Vec<T>` | 每个元素映射成一张表 |

⚠ **关键认知**：TOML 日期时间走 `toml::value::Datetime` 最稳，它保留"有没有时区"的信息。直接映射到 `chrono::DateTime<Utc>` 需要额外 serde 支持，本地时间转 UTC 必须显式指定时区。

## 2.4 序列化与保留注释

`toml::to_string_pretty` 是"重新生成"，原有注释和键顺序不会保留。要像编辑器一样修改现有文件，用 `toml_edit`：

```rust
use toml_edit::DocumentMut;

let mut doc = std::fs::read_to_string("Cargo.toml")?
    .parse::<DocumentMut>()?;
doc["package"]["version"] = toml_edit::value("0.2.0");
std::fs::write("Cargo.toml", doc.to_string())?;
```

★ **关键认知**：`toml` 负责**数据映射**，`toml_edit` 负责**格式保留**。Cargo 自己就是用 `toml_edit` 改清单的，所以 `cargo add` 不会把注释和排版全部冲掉。

## 2.5 Cargo.toml 的结构

```toml
[package]
name = "demo"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1.0", features = ["derive"], optional = true }

[features]
default = []
json = ["dep:serde"]
```

- `edition` 是**字符串** `"2021"`，不是整数 `2021`。
- `features` 的值是数组，`dep:serde` 是 Cargo 1.60+ 的语法。
- Cargo 还支持 `[[bin]]` 表数组和 `[profile.release]` 嵌套表，结构规则与上面一致。

★ **关键认知**：合法 TOML 不一定合法清单。`name = 123` 在 TOML 层面没问题，Cargo 会报"期望字符串"；`version = "abc"` 也可能被 Cargo 的语义检查拒绝。⚠ 不要用通用 TOML 校验器替代 `cargo metadata`。

# 3. 两种语言落地对比

| 维度 | Python | Rust |
|------|--------|------|
| 标准库支持 | 3.11+ 自带 `tomllib` | 无，需 `toml` crate |
| 读取方式 | `tomllib.load` / `loads` | `toml::from_str` / `from_path` |
| 写入方式 | `tomli-w`、`tomlkit` | `toml::to_string`、`toml_edit` |
| 结构约束 | 运行时字典访问，易漏键 | 编译期结构体字段检查 |

# 4. 常见错误

```text
错误 1：Python 用文本模式打开
现象：TypeError: File must be opened in binary mode
修复：open(path, "rb") 或 tomllib.loads(path.read_text())

错误 2：Rust 报 missing field `workers`
原因：TOML 里没写，结构体也没给默认值
修复：加 #[serde(default)] 或 #[serde(default = "函数名")]

错误 4：写回后注释全部消失
原因：用了 tomli-w / toml::to_string
修复：需要保留格式时改用 tomlkit / toml_edit
```

# 5. 动手练习

1. 用 Python 读取包含 `[[servers]]` 的 TOML，打印每个服务器的 `name` 和 `weight`，并把 `weight` 转成 `Decimal`。

2. 给 Rust 的 `Config` 增加可选的 `[cache]` 表，字段 `ttl_s` 缺失时默认为 60；用 `Option` 和 `serde(default)` 各实现一次，比较两者语义差别。

3. 从一份真实 `Cargo.toml` 中读出 `package.name`、`package.version` 和所有依赖名，分别用 Python 与 Rust 实现。

# 6. 小结

| 概念 | 要点 |
|------|------|
| `tomllib` | 3.11+ 标准库，只读，二进制模式 |
| 类型映射 | 日期分 aware / naive，数组变 list |
| 写回 | `tomli-w` 简单，`tomlkit` 保留格式 |
| `pyproject.toml` | PEP 621 元数据 + 工具命名空间 |
| `toml` + `serde` | 解析与结构映射分离 |
| `toml_edit` | 修改清单时保留注释与排版 |
| `Cargo.toml` | 合法 TOML 之外还有 Cargo 语义校验 |

下一篇进入配置实践，讲清楚配置文件怎么分层、环境差异怎么覆盖、密钥为什么不能写进 TOML，以及默认值和启动校验应该放在哪一层。
