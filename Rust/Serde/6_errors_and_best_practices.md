序列化代码真正难的地方不在"能跑通"，而在"出错时能不能定位、协议升级时会不会炸、面对不可信输入会不会被打穿"。这一篇把错误上下文、未知字段策略、性能取舍和安全边界收在一起，作为整个 Serde 模块的收尾。

# 1. 错误模型

## 1.1 serde_json 的错误信息

`serde_json::Error` 同时携带错误类别和位置，前面已经见过 `line()`、`column()`、`classify()`。实际使用中建议先分类，再决定是否重试：

```rust
use serde_json::error::Category;

fn parse_user(text: &str) -> Result<User, String> {
    serde_json::from_str(text).map_err(|err| match err.classify() {
        Category::Io => format!("读取失败：{err}"),
        Category::Syntax | Category::Eof => format!("请求体不是合法 JSON：{err}"),
        Category::Data => format!("字段不匹配：{err}"),
    })
}
```

★ **关键认知**：`Data` 类错误只告诉你"某个字段不对"，但不会告诉你**是哪个字段**。字段一多，`invalid type: integer, expected a string at line 1 column 57` 就要靠数第 57 个字符去猜。这是 serde 默认实现的结构性短板，不是配置问题。

## 1.2 路径上下文

需要精确到字段路径时，用 `serde_path_to_error` 包一层：

```toml
[dependencies]
serde_path_to_error = "0.1"
```

```rust
let mut de = serde_json::Deserializer::from_str(text);
let result: Result<User, _> = serde_path_to_error::deserialize(&mut de);

match result {
    Ok(user) => println!("{}", user.name),
    Err(err) => {
        let path = err.path().to_string();   // 如 users[3].email
        let inner = err.into_inner();        // 原始 serde_json::Error
        eprintln!("字段 {path} 解析失败：{inner}");
    }
}
```

★ **关键认知**：`serde_path_to_error` 不改变解析逻辑，只是记录"当前走到哪个字段"。⚠ 它会带来少量额外开销，适合放在网关、配置加载这类"错误必须可诊断"的入口，不必给每个高频小结构体都套上。

# 2. 未知字段

## 2.1 默认行为：忽略

`derive` 生成的 `Deserialize` **默认忽略**输入里多出来的字段：

```rust
#[derive(serde::Deserialize)]
struct User {
    id: u64,
    name: String,
}

// 多余的 role 会被静默丢弃，解析成功
let u: User = serde_json::from_str(r#"{"id":1,"name":"a","role":"admin"}"#).unwrap();
```

这样做的好处是向前兼容：服务端加了字段，老客户端不会因为不认识而失败。

## 2.2 deny_unknown_fields

需要"字段拼错就报错"时，在容器上加属性：

```rust
#[derive(serde::Deserialize)]
#[serde(deny_unknown_fields)]
struct ServerConfig {
    host: String,
    port: u16,
}
```

```text
{"host":"localhost","port":8080}      // 成功
{"host":"localhost","prot":8080}      // 错误：unknown field `prot`
```

★ **关键认知**：`deny_unknown_fields` 适合**自己完全掌控的输入**（配置文件、内部 RPC），能把拼写错误立刻暴露。⚠ 它**不能和 `flatten` 一起用**，也不适合公开 API——客户端多发一个字段就整条请求失败，兼容性极差。

## 2.3 严格与宽松的取舍

| 场景 | 建议 | 理由 |
|------|------|------|
| 应用配置文件 | 严格（`deny_unknown_fields`） | 拼错键名应立刻失败 |
| 内部服务 RPC | 严格 | 双方同步升级，契约明确 |
| 公开 HTTP API | 宽松 | 客户端版本不可控 |
| 日志/埋点 | 宽松 | 字段经常增删 |
| 数据库迁移脚本 | 严格 | 数据错误代价高 |

# 3. 兼容性设计

## 3.1 安全的变更

| 变更 | 是否兼容 | 前提 |
|------|---------|------|
| 新增 `Option<T>` 字段 | 是 | 缺失即 `None` |
| 新增带 `default` 的字段 | 是 | 缺失有兜底 |
| 新增枚举变体 | 视情况 | 消费方要有 `other` 兜底 |
| 删除字段 | 视情况 | 发送方宽松时安全 |
| 字段改名 | 视情况 | 必须加 `alias` |
| 字段改类型 | 否 | 需自定义兼容逻辑 |

## 3.2 危险变更

```rust
// 旧版本
#[derive(serde::Deserialize)]
struct Order {
    id: u64,
}

// 新版本：直接加必填字段 → 旧数据全部解析失败
#[derive(serde::Deserialize)]
struct Order {
    id: u64,
    status: String,   // ✗ 老 JSON 没有这个字段
}
```

正确做法是给新字段兜底：

```rust
#[derive(serde::Deserialize)]
struct Order {
    id: u64,
    #[serde(default = "default_status")]
    status: String,
}

fn default_status() -> String {
    "pending".to_string()
}
```

## 3.3 版本化

协议变动频繁时，显式带版本号比"猜字段"可靠：

```rust
#[derive(serde::Deserialize)]
struct Envelope {
    version: u32,
    #[serde(flatten)]
    payload: serde_json::Value,
}
```

★ **关键认知**：Serde 的属性只能解决"字段级"兼容，解决不了"语义级"变更。⚠ 一旦某个字段的含义变了（比如 `amount` 从"元"改成"分"），加 `alias` 只会让错误数据静默通过，正确做法是换字段名或升版本号。

# 4. 性能

## 4.1 避免中间 Value

最容易被忽视的性能问题，是先转 `Value` 再转结构体：

```rust
// 慢：解析两次，还多一次内存分配
let value: serde_json::Value = serde_json::from_str(text)?;
let user: User = serde_json::from_value(value)?;

// 快：一次解析直接到结构体
let user: User = serde_json::from_str(text)?;
```

```text
直接反序列化：
  JSON 字节 ──解析──> User

经 Value 中转：
  JSON 字节 ──解析──> Value（完整树）──转换──> User
                       ↑ 额外分配 + 类型擦除
```

## 4.2 借用与零拷贝

字段是 `String` 时会为每个字符串分配内存；只读场景可以用借用：

```rust
use std::borrow::Cow;

#[derive(serde::Deserialize)]
struct LogLine<'a> {
    #[serde(borrow)]
    message: Cow<'a, str>,   // 无转义时直接借用输入，不复制
}
```

反序列化入口也分快慢：

| 入口 | 说明 |
|------|------|
| `from_slice(&[u8])` | 从字节直接解析，避免先转成 `String` |
| `from_str(&str)` | 语义更清楚，适合已是字符串的场景 |
| `from_reader` | 必须包 `BufReader`，否则逐字节读极慢 |

## 4.3 写入优化

- 生产环境用 `to_vec` / `to_writer`，不要用 `to_string_pretty`。
- 大对象直接 `to_writer(BufWriter::new(file))`，避免先拼出完整字符串。
- 高频接口如果同一份数据反复输出，可以缓存序列化结果。
- 不要为了"好看"在日志里序列化超大对象，既慢又可能泄露敏感信息。

## 4.4 特性开关的代价

| feature | 收益 | 代价 |
|---------|------|------|
| `preserve_order` | 保留键顺序 | 用 `IndexMap`，内存和哈希开销更大 |
| `arbitrary_precision` | 数字不丢精度 | 数字存储变重，解析变慢 |
| `raw_value` | 延迟解析局部 | API 更复杂，生命周期约束多 |

# 5. 安全

## 5.1 资源耗尽

不可信输入要先限制规模，再谈解析：

- HTTP 层限制 body 大小，别让 1 GB JSON 进到 `from_str`。
- `serde_json` 默认递归深度上限是 128，防止深层嵌套爆栈；确实需要更深时才能开 `unbounded_depth` 并自行评估风险。
- bincode 这类二进制格式的长度前缀可能声明一个巨大的 `Vec` 长度，必须在协议层限制单条消息大小。

⚠ 不要对不可信数据使用 `unwrap()` 或 `expect()`。解析失败是正常输入，不是程序 bug。

## 5.2 重复键与类型陷阱

```json
{"id": 1, "id": 2}
```

- 解析到 `Value` 时，后出现的键覆盖前面的，**不报错**。
- 解析到派生结构体时，重复字段通常报 `duplicate field 'id'`。

两种行为不一致，跨服务传递数据时不要依赖重复键的语义。另外：

- `null` 和"字段缺失"是两回事，`Option<T>` 只解决缺失，不解决 `null` 转非空类型。
- 数字超出目标类型范围会报错，不会静默截断。
- 布尔字段收到字符串 `"true"` 会失败，需要自定义 Visitor。

## 5.3 敏感信息

- 密码、token 用 `#[serde(skip_serializing)]` 或独立 DTO，避免误输出。
- 日志里只打印必要字段，不要 `println!("{:#?}", req)` 整个请求。
- 自定义 `Debug` 时同样要遮蔽敏感字段，`derive(Debug)` 会把它们全打出来。

★ **关键认知**：`skip_serializing` 只控制"输出"，不阻止"输入"。⚠ 如果类型既要接收密码又要避免回显，应该拆成 `CreateUser`（含密码）和 `UserView`（不含密码）两个结构体，而不是在一个结构体上打补丁。

# 6. 最佳实践清单

| 主题 | 做法 |
|------|------|
| 错误 | 分类处理，入口处加路径上下文 |
| 未知字段 | 配置严格，公开 API 宽松 |
| 兼容 | 新字段给 `default`，改名加 `alias` |
| 性能 | 避免 `Value` 中转，读写都缓冲 |
| 借用 | 只读大文本用 `Cow` + `borrow` |
| 安全 | 限制输入大小和深度，不 `unwrap` |
| 敏感 | 独立 DTO，日志脱敏 |
| 一致性 | 全项目统一日期格式和命名风格 |

# 7. 练习

1. 用 `serde_path_to_error` 解析一个嵌套结构，故意让第三层的一个整数字段收到字符串，打印出完整字段路径。
2. 分别给配置结构体加和不加 `deny_unknown_fields`，用带拼写错误的 JSON 测试，比较两边行为。
3. 把一段"先 `Value` 再 `from_value`"的代码改成直接 `from_str`，用 `std::time::Instant` 粗略比较一万次解析的耗时差异。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 错误定位 | 分类 + `line/column`，路径用 `serde_path_to_error` |
| 未知字段 | 默认忽略，`deny_unknown_fields` 需谨慎 |
| 兼容变更 | 加字段给默认值，改名用 `alias` |
| 性能核心 | 拒绝 `Value` 中转，读写都要缓冲 |
| 零拷贝 | `Cow` + `#[serde(borrow)]` |
| 安全 | 限制大小/深度，不信任长度前缀 |
| 敏感数据 | 拆分 DTO，日志脱敏 |

到这里 Serde 模块告一段落：从数据模型和 derive 出发，经过格式、属性、枚举、自定义实现，最后落在错误、性能与安全。下一步可以把这套能力接进 Axum 的 `Json` 提取器，看看请求体解析失败时框架如何把 `serde_json::Error` 转成 HTTP 状态码。
