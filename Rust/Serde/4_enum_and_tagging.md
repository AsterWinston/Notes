枚举是 serde 里最容易出问题的部分：同一个 `enum`，换一种 `tag` 属性，JSON 结构就完全不同，跨语言对接时对方能不能解析、以后能不能加变体，全取决于这个选择。本篇把四种表示形式逐一拆开，给出各自适合的协议风格和踩坑点。

# 1. 四种表示总览

先看同一个枚举在四种模式下的输出：

```rust
#[derive(serde::Serialize, serde::Deserialize)]
enum Message {
    Quit,
    Write(String),
    Move { x: i32, y: i32 },
}
```

| 模式 | 属性 | `Move { x: 1, y: 2 }` 的输出 |
|------|------|----------------------------|
| 外部标签 | 默认 | `{"Move":{"x":1,"y":2}}` |
| 内部标签 | `#[serde(tag = "type")]` | `{"type":"Move","x":1,"y":2}` |
| 相邻标签 | `#[serde(tag = "t", content = "c")]` | `{"t":"Move","c":{"x":1,"y":2}}` |
| 无标签 | `#[serde(untagged)]` | `{"x":1,"y":2}` |

★ **关键认知**：四种模式的本质区别只有两个问题——**变体名放在哪里**、**变体内容是否单独包一层**。外部标签把名字当外层键；内部标签把名字塞进内容对象；相邻标签用两个平级键分别装名字和内容；无标签则完全丢掉名字。

# 2. 外部标签

## 2.1 形态

不加任何属性时的默认行为：

```rust
#[derive(serde::Serialize)]
enum Shape {
    Circle(f64),                       // 新类型变体
    Rect { w: f64, h: f64 },           // 结构体变体
    Point,                             // 单元变体
}
```

```json
{"Circle": 1.5}
{"Rect": {"w": 10.0, "h": 5.0}}
"Point"
```

不同变体形态的输出规则：

| 变体形态 | 输出 |
|---------|------|
| 单元变体 | 字符串 `"Point"` |
| 新类型变体 | `{"Variant": 内容}` |
| 元组变体 | `{"Variant": [a, b, c]}` |
| 结构体变体 | `{"Variant": {字段...}}` |

## 2.2 优缺点

- 优点：不需要额外的 `tag` 字段，结构最紧凑；新增变体不影响旧变体的解析。
- 缺点：变体名是 JSON 的键，跨语言时对方必须知道所有变体名；对 TypeScript 等语言生成类型也稍麻烦。

⚠ 单元变体序列化成裸字符串，而不是对象。如果前端统一按对象处理，`"Point"` 会让 `JSON.parse` 之后拿到的不是对象，需要额外判断。

# 3. 内部标签

## 3.1 tag

把变体名写进内容对象的某个字段：

```rust
#[derive(serde::Serialize, serde::Deserialize)]
#[serde(tag = "type")]
enum Event {
    Login { user_id: u64 },
    Logout { user_id: u64 },
    Ping,
}
```

```json
{"type":"Login","user_id":1}
{"type":"Logout","user_id":1}
{"type":"Ping"}
```

这是 REST API 里最常见的风格，因为"类型字段 + 扁平字段"很像普通对象。

## 3.2 限制

★ **关键认知**：内部标签要求变体内容能"展开成一个 map"。因此它**只支持单元变体和结构体变体**；新类型变体只有在内部是结构体/映射时才可用；元组变体和"内部是标量"的新类型变体会在序列化时报错。

```rust
#[derive(serde::Serialize)]
#[serde(tag = "type")]
enum Bad {
    Values(Vec<i32>),   // ✗ 元组/序列无法与 tag 合并
    Code(u32),          // ✗ 标量无法与 tag 合并
}
```

⚠ 报错信息通常是 `cannot serialize tagged newtype variant Bad::Values containing a sequence`。看到它别怀疑语法，直接换相邻标签或无标签。

另一个坑是字段冲突：

```rust
#[derive(serde::Serialize)]
#[serde(tag = "type")]
enum Conflict {
    A { r#type: String },  // ✗ 和 tag 字段重名
}
```

内部标签会把 `type` 提到同一层，重名时 derive 会报冲突。设计协议时要给 `tag` 选一个业务字段不会用到的名字。

# 4. 相邻标签

## 4.1 tag 与 content

名字和内容各占一个键：

```rust
#[derive(serde::Serialize, serde::Deserialize)]
#[serde(tag = "t", content = "c")]
enum Payload {
    Text(String),
    Numbers(Vec<i32>),
    Empty,
}
```

```json
{"t":"Text","c":"hello"}
{"t":"Numbers","c":[1,2,3]}
{"t":"Empty"}
```

单元变体没有 `content` 键，只有 `tag`。

## 4.2 适用场景

相邻标签是"最不挑内容类型"的方案：标量、数组、对象都能塞进 `content`，同时保留了显式的类型字段。

| 需求 | 推荐模式 |
|------|---------|
| 内容可能是数组或标量 | 相邻标签 |
| 前端希望扁平对象 | 内部标签（仅结构体变体） |
| 追求最小体积 | 外部标签 |
| 对接只认裸数据的老系统 | 无标签 |

★ **关键认知**：相邻标签的 `content` 是独立一层，所以变体内部字段**不会和 tag 冲突**。这是它比内部标签更稳的地方，代价是多一层嵌套。

# 5. 无标签

## 5.1 untagged

完全省略变体名，只输出内容：

```rust
#[derive(serde::Serialize, serde::Deserialize)]
#[serde(untagged)]
enum Id {
    Number(u64),
    Text(String),
}
```

```json
42
"abc"
```

## 5.2 匹配顺序

反序列化时，serde 按**声明顺序**逐个尝试，第一个成功就返回：

```rust
#[derive(serde::Deserialize)]
#[serde(untagged)]
enum Value {
    Int(i64),
    Float(f64),
}
// 输入 1.5：Int 失败 → Float 成功
// 输入 1：Int 成功，不会尝试 Float
```

⚠ 顺序写反会静默改变语义：

```rust
#[derive(serde::Deserialize)]
#[serde(untagged)]
enum Bad {
    Any(serde_json::Value),  // ✗ 永远第一个成功，后面的变体形同虚设
    Special(String),
}
```

★ **关键认知**：无标签的错误信息非常差——所有变体都失败时只报 `data did not match any variant of untagged enum X`，不告诉你差在哪。⚠ 调试无标签枚举时，先临时改成相邻标签，看清结构再换回去。

# 6. 版本演进

## 6.1 新增变体

| 模式 | 新增变体的影响 |
|------|---------------|
| 外部标签 | 旧代码遇到未知变体报 `unknown variant`，但旧变体不受影响 |
| 内部标签 | 同上，且新变体必须是结构体/单元形态 |
| 相邻标签 | 同上，最灵活 |
| 无标签 | 新变体若排在末尾通常安全；排前面可能截胡旧输入 |

```rust
// 兼容未知变体的写法：加一个兜底变体
#[derive(serde::Deserialize)]
#[serde(tag = "type")]
enum Event {
    Login { user_id: u64 },
    #[serde(other)]
    Unknown,                 // 只对内部标签/相邻标签有效，且必须是单元变体
}
```

★ **关键认知**：`#[serde(other)]` 是内部标签和相邻标签的"未知变体兜底"，遇到不认识的 `type` 时落到 `Unknown`，而不是报错。⚠ 它**不能**用于外部标签和无标签；外部标签要兜底只能自己实现 `Deserialize`。

## 6.2 修改变体字段

- 加字段：给新字段 `#[serde(default)]`，旧数据仍能解析。
- 删字段：旧数据会多出未知字段，默认被忽略；开了 `deny_unknown_fields` 就会失败。
- 改字段名：用 `alias` 同时接受新旧名字。
- 改变体名：用 `#[serde(rename = "new", alias = "old")]`。

# 7. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| `cannot serialize tagged newtype variant` | 内部标签遇到元组/标量变体 | 改相邻标签 |
| `unknown variant 'X'` | 对方发了新变体 | 加 `#[serde(other)]` 或升级枚举 |
| `data did not match any variant` | 无标签全部失败，信息被吞 | 临时改相邻标签调试 |
| 无标签选错变体 | 宽泛变体排在前面 | 把具体变体放前面 |
| tag 字段冲突 | 变体里有同名字段 | 换 tag 名 |
| 单元变体输出是字符串 | 外部标签的默认行为 | 需要对象就换相邻标签 |

⚠ `deny_unknown_fields` 与无标签枚举组合时，判断会变得更严格也更难调试，跨版本兼容场景不建议一起用。

# 8. 练习

1. 用外部标签定义 `Command { Add(u32), Remove(u32), Clear }`，写出三种变体的 JSON 输出，再改成相邻标签，对比差异。
2. 定义一个内部标签的 `Notification`，包含 `Email { to: String }` 和 `Sms { phone: String }`，并加 `#[serde(other)] Unknown`，测试未知 `type` 的行为。
3. 用无标签实现 `StringOrNumber`，故意把 `serde_json::Value` 变体放在第一位，观察它如何"吃掉"所有输入，并解释原因。

# 9. 小结

| 模式 | 类型字段 | 内容层 | 适合 |
|------|---------|--------|------|
| 外部标签 | 外层键 | 有 | 简洁、内部协议 |
| 内部标签 | 内容内 | 无 | 扁平 REST 对象 |
| 相邻标签 | 独立键 | 有 | 内容类型不受限 |
| 无标签 | 无 | 无 | 对接裸数据、联合类型 |

下一篇进入自定义序列化，讲清楚手写 `Serialize`/`Deserialize`、`with` 模块，以及日期、大整数和字段校验这些 derive 覆盖不了的场景。
