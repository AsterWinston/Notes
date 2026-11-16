`serde` 定义了"怎么序列化"，真正把字节读进来、写出去的是格式库。`serde_json` 是其中使用频率最高的一个：HTTP 接口、配置文件、日志、跨语言数据交换都绕不开它。本篇把 JSON 的读写接口、动态 `Value` 树、错误定位讲透，再横向对比 YAML 与 TOML，让你按场景选对格式。

# 1. serde_json 的读写接口

## 1.1 字符串与字节

最常用的四个函数，输入输出都是内存中的连续数据：

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
}

fn main() {
    let user = User { id: 1, name: "alice".into() };

    // 序列化到 String，适合直接放进 HTTP body
    let text = serde_json::to_string(&user).unwrap();

    // 带缩进，适合写给人看或落盘
    let pretty = serde_json::to_string_pretty(&user).unwrap();

    // 序列化到 Vec<u8>，适合网络发送、写入 Buffer
    let bytes = serde_json::to_vec(&user).unwrap();

    // 反序列化：字符串与字节切片各有一个入口
    let a: User = serde_json::from_str(&text).unwrap();
    let b: User = serde_json::from_slice(&bytes).unwrap();

    println!("{text}\n{pretty}");
    println!("{} {}", a.name, b.name);
}
```

★ **关键认知**：`to_string` 和 `to_vec` 只是同一套序列化逻辑的两种"落点"。`String` 保证 UTF-8，`Vec<u8>` 少一层 UTF-8 校验，在"直接写 socket / 写文件"的场景更自然。⚠ 生产中不要对不可信数据用 `unwrap()`，解析失败应该转成业务错误，而不是 panic。

## 1.2 文件与流

数据量大时，避免先拼成一个巨大的 `String`，直接对接 `Read`/`Write`：

```rust
use std::fs::File;
use std::io::{BufReader, BufWriter};

fn write_file(user: &User) -> std::io::Result<()> {
    let file = File::create("user.json")?;
    let mut writer = BufWriter::new(file);              // 包一层缓冲，减少系统调用
    serde_json::to_writer_pretty(&mut writer, user)?;   // 序列化错误可转成 io::Error
    Ok(())
}

fn read_file() -> Result<User, Box<dyn std::error::Error>> {
    let file = File::open("user.json")?;
    let reader = BufReader::new(file);                  // from_reader 每次读一个字节，必须缓冲
    let user = serde_json::from_reader(reader)?;
    Ok(user)
}
```

★ **关键认知**：`serde_json::from_reader` 内部按字节逐个读取，**不缓冲**。直接传 `File` 会触发大量系统调用，慢到怀疑人生；正确做法是外面套 `BufReader`，或者先用 `read_to_string` 再 `from_str`。

## 1.3 接口对比

| 函数 | 输入/输出 | 适用场景 | 注意 |
|------|-----------|---------|------|
| `to_string` | `&T` → `String` | HTTP body、日志 | 分配一次字符串 |
| `to_vec` | `&T` → `Vec<u8>` | 网络、二进制缓冲 | 最通用 |
| `to_writer` | `&T` + `Write` | 大对象落盘 | 自己负责缓冲 |
| `from_str` | `&str` → `T` | 已读入内存的文本 | 输入必须完整 |
| `from_slice` | `&[u8]` → `T` | 字节缓冲 | 会校验 UTF-8 |
| `from_reader` | `Read` → `T` | 大文件、 socket | 必须包 `BufReader` |

# 2. Value 动态树

## 2.1 五种形态

当结构在编译期不确定，或者只想"先解析再挑字段"，用 `serde_json::Value`：

```rust
pub enum Value {
    Null,
    Bool(bool),
    Number(Number),
    String(String),
    Array(Vec<Value>),
    Object(Map<String, Value>),
}
```

```rust
let value: serde_json::Value =
    serde_json::from_str(r#"{"id":1,"tags":["rust","json"]}"#).unwrap();

assert!(value.is_object());
assert_eq!(value["id"].as_u64(), Some(1));
assert_eq!(value["tags"][0].as_str(), Some("rust"));
```

`Number` 是独立类型，不会直接暴露成 `f64`，所以大整数不会像 JavaScript 那样悄悄丢精度：

```rust
let n: serde_json::Value = serde_json::from_str("9007199254740993").unwrap();
println!("{}", n.as_u64().unwrap()); // 9007199254740993，完整保留
```

⚠ `as_u64()` 对负数和浮点返回 `None`；`as_f64()` 对某些超大整数会丢精度。需要严格区分整数与浮点时，先用 `is_i64`/`is_u64`/`is_f64` 判断。

## 2.2 取值与索引

两种风格，语义不同：

| 写法 | 缺失时 | 非对象时 | 建议 |
|------|--------|---------|------|
| `value["key"]` | 返回 `Null` | 返回 `Null` | 快速原型 |
| `value.get("key")` | 返回 `None` | 返回 `None` | 需要区分"缺失"时 |
| `value.pointer("/a/b")` | 返回 `None` | 返回 `None` | 深层路径 |

```rust
let v: serde_json::Value = serde_json::from_str(r#"{"user":{"name":"alice"}}"#).unwrap();

let name = v.pointer("/user/name").and_then(|x| x.as_str());
assert_eq!(name, Some("alice"));
```

★ **关键认知**：`value["key"]` 在键不存在时**不会 panic**，而是返回 `Value::Null`。这很方便，也很危险——拼错字段名会静默得到 `Null`，类型检查帮不了你。⚠ 对可靠性要求高的代码用 `get` + `ok_or`，把"缺失"显式暴露成错误。

## 2.3 json! 宏与转换

构造动态 JSON 时，`json!` 宏比手写枚举舒服得多：

```rust
use serde_json::json;

let id = 42;
let body = json!({
    "id": id,
    "name": "alice",
    "tags": ["rust", "json"],
    "meta": null,
});
```

在 `Value` 和强类型之间自由转换：

```rust
let user: User = serde_json::from_value(body.clone())?; // Value → 强类型
let value = serde_json::to_value(&user)?;              // 强类型 → Value
```

# 3. 错误处理

## 3.1 错误的四个分类

`serde_json::Error` 可以通过 `classify()` 判断来源：

| 分类 | 含义 | 典型原因 |
|------|------|---------|
| `Io` | 底层读写失败 | 文件不存在、网络断开 |
| `Syntax` | JSON 语法错误 | 少逗号、引号不配对 |
| `Data` | 语法正确但类型/字段不符 | 字段缺失、类型不匹配 |
| `Eof` | 输入提前结束 | 空文件、截断的响应 |

```rust
use serde_json::error::Category;

match serde_json::from_str::<User>(text) {
    Ok(user) => println!("{}", user.name),
    Err(err) => match err.classify() {
        Category::Io => eprintln!("IO 错误：{err}"),
        Category::Syntax => eprintln!("语法错误：{err}"),
        Category::Data => eprintln!("数据不匹配：{err}"),
        Category::Eof => eprintln!("内容不完整：{err}"),
    },
}
```

## 3.2 定位与上报

语法错误可以拿到行列号，排查时先看报错位置附近：

```rust
let bad = r#"{"id": 1, "name": }"#;
let err = serde_json::from_str::<User>(bad).unwrap_err();
println!("第 {} 行，第 {} 列：{}", err.line(), err.column(), err);
// 第 1 行，第 20 列：expected value at line 1 column 20
```

★ **关键认知**：`Data` 类错误**没有精确列号**，因为它是反序列化到某个字段时才发现的。想定位到具体字段路径（如 `users[3].email`），需要额外的 `serde_path_to_error` crate，单靠 `serde_json` 做不到。

# 4. 其他格式

## 4.1 YAML

YAML 擅长表达层次化配置，支持注释，可读性比 JSON 好：

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_yaml = "0.9"
```

```rust
let yaml = serde_yaml::to_string(&user)?;
let user: User = serde_yaml::from_str(&yaml)?;
```

```yaml
# 输出形态
id: 1
name: alice
```

⚠ `serde_yaml` 已停止维护（仓库归档），新项目可以评估 `serde_norway` 等兼容替代。YAML 还有著名的"隐式类型"问题：`yes`、`on`、`1.0` 可能被解析成布尔或数字，字符串字段务必加引号。

## 4.2 TOML

TOML 是 Cargo 清单的格式，适合"人类手写配置"：

```rust
let config = Config { port: 8080, debug: true };
let text = toml::to_string(&config)?;   // 注意不是 to_string_pretty
let back: Config = toml::from_str(&text)?;
```

```toml
port = 8080
debug = true
```

TOML 的顶层必须是表（table），不能直接序列化一个裸 `Vec` 或标量；日期时间有原生类型，这一点比 JSON 省事。

## 4.3 格式选型

| 格式 | 可读性 | 注释 | 自描述 | 适合 |
|------|--------|------|--------|------|
| JSON | 中 | 否 | 是 | 接口传输、跨语言 |
| YAML | 高 | 是 | 是 | 复杂配置、K8s |
| TOML | 高 | 是 | 是 | 项目清单、扁平配置 |
| bincode | 无 | 否 | 否 | 内部高性能通信 |
| MessagePack | 低 | 否 | 是 | 体积敏感的场景 |

★ **关键认知**：自描述（self-describing）意味着格式里带了类型信息，反序列化时可以按名字找字段。⚠ `bincode` 这类非自描述格式**强依赖字段顺序**：结构体加一个字段，旧数据就全读错了，所以只适合"两端同时升级"的内部协议。

# 5. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `trailing characters` | `from_str` 后还有多余内容 | 清洗输入或用流式解析 |
| `EOF while parsing` | 输入被截断、空文件 | 检查网络/文件完整性 |
| `missing field` | JSON 缺必填字段 | 补字段或用 `default` |
| `invalid type` | 值类型不符 | 改类型或自定义反序列化 |
| 大整数变科学计数 | 用了 `as_f64` | 用 `as_u64` 或字符串传输 |
| 读取文件极慢 | `from_reader` 没缓冲 | 包 `BufReader` |

# 6. 练习

1. 写一个函数，读取 NDJSON 文件（每行一个 `User`），用 `BufReader` + 逐行 `from_str` 统计用户总数。
2. 用 `Value` 解析一份结构不确定的 JSON，取出所有 `tags` 数组并去重，再转回 `Vec<String>`。
3. 分别用 `to_string` 和 `to_string_pretty` 序列化同一个结构体，比较字节大小，思考线上接口该用哪个。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 序列化入口 | `to_string` / `to_vec` / `to_writer` |
| 反序列化入口 | `from_str` / `from_slice` / `from_reader` |
| 流式读取 | `from_reader` 必须包 `BufReader` |
| `Value` | 动态树，`[]` 缺失返回 `Null`，`get` 返回 `Option` |
| 错误分类 | Io / Syntax / Data / Eof |
| 格式选择 | 传输用 JSON，配置看 YAML/TOML，内部二进制用 bincode |

下一篇进入 serde 属性，讲清楚 `rename`、`rename_all`、`skip`、`default`、`flatten`、`alias` 各自解决什么字段兼容问题，以及它们之间的组合限制。
