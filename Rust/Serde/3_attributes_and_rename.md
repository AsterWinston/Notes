实际项目里，Rust 的命名习惯和外部协议往往对不上：结构体字段是 `snake_case`，接口要 `camelCase`；老版本字段叫 `user_name`，新版本改成了 `username`；某些字段只在序列化时出现，某些字段缺失时要有默认值。这些都不需要手写 `impl`，serde 的属性系统就是为它们准备的。

# 1. 属性写在哪儿

## 1.1 三种作用位置

`#[serde(...)]` 可以写在三个层级，作用范围不同：

| 位置 | 写法示例 | 作用 |
|------|---------|------|
| 容器（struct/enum） | `#[serde(rename_all = "camelCase")]` | 影响所有字段/变体 |
| 字段 | `#[serde(rename = "userName")]` | 只影响当前字段 |
| 变体（enum） | `#[serde(rename = "move")]` | 只影响当前变体 |

```rust
#[derive(serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "camelCase")]       // 容器级：所有字段转 camelCase
struct UserProfile {
    user_id: u64,                         // → userId
    #[serde(rename = "displayName")]      // 字段级：覆盖容器规则
    display_name: String,
    is_active: bool,                      // → isActive
}
```

★ **关键认知**：字段级属性**优先于**容器级属性。理解这一点，就能用容器级规则统一大多数字段，只对个别例外做覆盖，而不是每个字段都手写 `rename`。

# 2. 重命名

## 2.1 rename

最直接的方式，一个字段一个名字：

```rust
#[derive(serde::Serialize, serde::Deserialize)]
struct User {
    #[serde(rename = "id")]
    user_id: u64,
    #[serde(rename = "userName")]
    name: String,
}
```

序列化输出 `{"id":1,"userName":"alice"}`，反序列化也认这两个名字。`rename` 同时作用于两个方向；如果只想改一个方向：

```rust
#[serde(rename(serialize = "userName", deserialize = "user_name"))]
```

## 2.2 rename_all

批量转换命名风格，常用值如下：

| 取值 | `user_name` 变成 |
|------|-----------------|
| `"PascalCase"` | `UserName` |
| `"camelCase"` | `userName` |
| `"snake_case"` | `user_name` |
| `"SCREAMING_SNAKE_CASE"` | `USER_NAME` |
| `"kebab-case"` | `user-name` |
| `"SCREAMING-KEBAB-CASE"` | `USER-NAME` |

```rust
#[derive(serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "camelCase")]
struct ApiUser {
    user_id: u64,
    display_name: String,
}
// {"userId":1,"displayName":"alice"}
```

⚠ `rename_all` 写在 enum 上时，改的是**变体名**，不是变体内部字段。要让变体内部字段也转换，需要 `rename_all_fields`。

## 2.3 rename_all_fields

针对 enum 的 struct variant 内部字段（serde 1.0.184+）：

```rust
#[derive(serde::Serialize)]
#[serde(rename_all = "snake_case")]        // 变体名：create_user
#[serde(rename_all_fields = "camelCase")]  // 变体字段：userId
enum Command {
    CreateUser { user_id: u64 },
}
// {"create_user":{"userId":1}}
```

# 3. 跳过与默认值

## 3.1 skip 家族

三个属性容易混，用表格分清：

| 属性 | 序列化 | 反序列化 | 反序列化时缺失 |
|------|--------|---------|---------------|
| `skip` | 跳过 | 跳过 | 用 `Default::default()` |
| `skip_serializing` | 跳过 | 正常读取 | 正常要求必填 |
| `skip_deserializing` | 正常输出 | 跳过 | 用 `Default::default()` |

```rust
#[derive(serde::Serialize, serde::Deserialize)]
struct Account {
    id: u64,
    #[serde(skip)]                 // 前后端都不传，反序列化时取默认值
    internal_flag: bool,
    #[serde(skip_serializing)]     // 只读字段：能反序列化，不往外写
    password_hash: String,
}
```

★ **关键认知**：`skip` 在反序列化方向要求类型实现 `Default`，因为字段不会从输入里读，只能凭空造一个。⚠ 用 `skip` 隐藏敏感字段时，它仍然可能被反序列化输入覆盖——要彻底隔离，应该用独立 DTO，而不是依赖属性。

## 3.2 skip_serializing_if

只在满足条件时跳过输出，常用于省略空值：

```rust
#[derive(serde::Serialize)]
struct Profile {
    name: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    avatar: Option<String>,
    #[serde(skip_serializing_if = "Vec::is_empty")]
    tags: Vec<String>,
}
```

```rust
let p = Profile { name: "alice".into(), avatar: None, tags: vec![] };
// {"name":"alice"}，两个空字段都被省略
```

也可以接自定义函数：

```rust
fn is_zero(n: &u32) -> bool {
    *n == 0
}

#[derive(serde::Serialize)]
struct Counter {
    #[serde(skip_serializing_if = "is_zero")]
    count: u32,
}
```

⚠ `skip_serializing_if` **只影响序列化**，不影响反序列化。省略的字段在读取方仍然按必填处理，除非同时加了 `#[serde(default)]`。

## 3.3 default

字段缺失时填充默认值，而不是报 `missing field`：

```rust
#[derive(serde::Deserialize)]
struct Config {
    #[serde(default)]
    retries: u32,                          // 缺失 → 0
    #[serde(default = "default_port")]
    port: u16,                             // 缺失 → 8080
    #[serde(default = "Vec::new")]
    hosts: Vec<String>,                    // 缺失 → 空数组
}

fn default_port() -> u16 {
    8080
}
```

也可以写在容器上，让**所有**字段缺失时都取默认值：

```rust
#[derive(serde::Deserialize, Default)]
#[serde(default)]
struct Options {
    debug: bool,
    timeout: u64,
}
```

★ **关键认知**：`default` 只在字段**完全缺失**时生效。如果 JSON 里写了 `"retries": null`，那不是缺失，而是类型错误（`null` 不能变成 `u32`）。⚠ 想同时接受 `null` 和缺失，字段类型要用 `Option<T>`，不要指望 `default`。

# 4. 展平与别名

## 4.1 flatten

把一个结构体的字段"摊平"进外层对象：

```rust
#[derive(serde::Serialize, serde::Deserialize)]
struct Pagination {
    page: u32,
    size: u32,
}

#[derive(serde::Serialize, serde::Deserialize)]
struct Response {
    total: u64,
    #[serde(flatten)]
    page: Pagination,
}
// {"total":100,"page":1,"size":20}，而不是嵌套的 {"page":{"page":1,...}}
```

`flatten` 还能收集"未知字段"：

```rust
use std::collections::HashMap;

#[derive(serde::Serialize, serde::Deserialize)]
struct Event {
    name: String,
    #[serde(flatten)]
    extra: HashMap<String, serde_json::Value>, // 其余键全部收进这里
}
```

★ **关键认知**：`flatten` 依赖格式的**自描述**能力，需要先读出所有键再分发。⚠ 它和 `deny_unknown_fields` **不能同时使用**（derive 会直接报错），因为"收集未知字段"和"拒绝未知字段"逻辑冲突；在 bincode 这类非自描述格式上也不可用。

## 4.2 alias

`alias` 只在反序列化时生效，可以写多个，用于兼容历史字段名：

```rust
#[derive(serde::Deserialize)]
struct User {
    #[serde(alias = "user_name", alias = "userName")]
    username: String,
}
```

```json
{"username": "a"}   // 认
{"user_name": "b"}  // 认
{"userName": "c"}   // 认
```

序列化时永远输出 `username`。⚠ 如果多个别名字段同时出现在输入里，serde 会按声明顺序取第一个匹配，后面的被当作未知字段忽略（或触发 `deny_unknown_fields`），不要依赖这种歧义输入。

# 5. 字段兼容实战

## 5.1 改名不停机

服务端升级字段名时，最稳的做法是"新名输出、旧名照收"：

```rust
#[derive(serde::Serialize, serde::Deserialize)]
struct Article {
    #[serde(rename = "title", alias = "name", alias = "article_title")]
    title: String,
    #[serde(default)]
    tags: Vec<String>,
}
```

- 新客户端读 `title`。
- 老客户端仍发 `name`，靠 `alias` 兼容。
- 老数据缺 `tags`，靠 `default` 兜底。

## 5.2 常见组合限制

| 组合 | 是否可用 | 说明 |
|------|---------|------|
| `flatten` + `deny_unknown_fields` | 否 | derive 直接报错 |
| `flatten` + 非自描述格式 | 否 | 无法预读键 |
| `skip` + 无 `Default` | 否 | 反序列化无法造值 |
| `default` + 显式 `null` | 不生效 | `null` 不是缺失 |
| `alias` + 序列化 | 不生效 | 别名只用于读取 |

# 6. 常见错误

| 报错/现象 | 原因 | 解决 |
|-----------|------|------|
| `unknown field 'user_name'` | 只改了 `rename` 没加 `alias` | 加 `alias`，或统一客户端 |
| `missing field 'x'` | 字段缺失且无默认值 | `default` 或改 `Option` |
| `#[serde(flatten)] cannot be used with deny_unknown_fields` | 组合冲突 | 二选一 |
| 反序列化 `null` 报错 | `default` 不处理 `null` | 类型改 `Option<T>` |
| enum 内部字段没转命名 | 只写了 `rename_all` | 补 `rename_all_fields` |
| `skip` 字段编译报 `Default` 未实现 | 反序列化需要默认值 | 实现 `Default` 或换用 `skip_deserializing` + 默认值 |

# 7. 练习

1. 定义一个 `ServerConfig`，要求：字段输出为 `kebab-case`；`timeout` 缺失时默认 30；`token` 为空字符串时不输出。
2. 用 `flatten` 把 `Address { city, street }` 嵌进 `Order`，再序列化，确认输出是平铺的。
3. 模拟一次字段改名：旧名 `mobile`、新名 `phone`，要求新代码输出 `phone`，但仍能读取旧数据里的 `mobile`，写出完整结构体并测试两个方向的输入。

# 8. 小结

| 属性 | 方向 | 典型用途 |
|------|------|---------|
| `rename` | 双向 | 单个字段/变体改名 |
| `rename_all` | 双向 | 批量命名风格转换 |
| `rename_all_fields` | 双向 | enum 变体内部字段转换 |
| `skip` | 双向 | 完全忽略字段 |
| `skip_serializing_if` | 仅序列化 | 省略空值 |
| `default` | 仅反序列化 | 缺失时兜底 |
| `flatten` | 双向 | 平铺嵌套结构 |
| `alias` | 仅反序列化 | 兼容旧字段名 |

下一篇进入枚举的序列化，讲清楚外部、内部、相邻、无标签四种表示形式，以及它们在 API 版本演进中的取舍。
