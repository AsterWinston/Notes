`derive` 能覆盖绝大多数结构体，但总有它表达不了的规则：金额要写成 `"12.34"` 而不是分、时间要 Unix 时间戳、`u128` 要转成字符串防止前端丢精度、字段值要落在合法区间。这些都需要绕过默认实现，自己控制序列化过程。

# 1. 什么时候需要手写

| 场景 | `derive` 的表现 | 需要自定义 |
|------|----------------|-----------|
| 金额/精度 | 直接输出 `i64` 分 | 输出带小数点的字符串 |
| 日期时间 | 依赖类型自身实现 | 指定时间戳或格式 |
| 超大整数 | 输出 JSON 数字 | 输出字符串 |
| 枚举的线上值 | 用 Rust 变体名 | 映射成数字/旧名称 |
| 字段校验 | 只做类型检查 | 检查范围、长度、格式 |
| 兼容多种输入 | 只接受一种类型 | 数字和字符串都接受 |

★ **关键认知**：自定义序列化的核心不是"写更多代码"，而是**在类型和格式之间插入一层翻译**。能用一个独立函数表达的，优先用 `with`/`serialize_with`；只有整段逻辑都要变时，才手写 trait。

# 2. 手写 Serialize

## 2.1 调用流程

手写 `Serialize` 时，你拿到的是格式库给的 `Serializer`，要做的是"用哪类值类型来描述自己"：

```text
serde_json::to_string(&money)
  → Money::serialize(serializer)
  → serializer.serialize_str("12.34")      // 选择描述方式
  → serde_json 把字符串写成 "12.34"
```

可用的描述方法很多，常用的有：

| 方法 | 输出的 JSON |
|------|------------|
| `serialize_str` | 字符串 |
| `serialize_i64` / `serialize_u64` | 整数 |
| `serialize_seq` / `serialize_map` | 数组 / 对象 |
| `serialize_struct` | 带字段名的对象 |

## 2.2 金额示例

用一个以"分"为单位的类型，序列化成两位小数的字符串：

```rust
use serde::Serialize;

#[derive(Debug, PartialEq)]
struct Money(i64); // 内部以分存储，避免浮点误差

impl Serialize for Money {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        let sign = if self.0 < 0 { "-" } else { "" };
        let abs = self.0.abs();
        // 150 → "1.50"，-5 → "-0.05"
        let text = format!("{sign}{}.{:02}", abs / 100, abs % 100);
        serializer.serialize_str(&text)
    }
}

fn main() {
    let price = Money(1250);
    assert_eq!(serde_json::to_string(&price).unwrap(), "\"12.50\"");
}
```

★ **关键认知**：`serialize_str` 里的字符串生命周期只在调用期间有效，serde 会立刻把它写出去或复制，不会存引用。所以可以安全地用 `format!` 造临时字符串。⚠ 但如果你序列化的是 `&str`，要保证引用活到序列化结束，否则编译器会拦下来。

# 3. 手写 Deserialize

## 3.1 Visitor 模式

`Deserialize` 比 `Serialize` 绕，因为它要处理"输入到底是什么类型"。serde 用 **Visitor 模式**把"可能的类型"分发成一组回调：

```text
Deserializer 读到 token
  → 根据 token 类型调用 Visitor 的 visit_* 方法
     ├── visit_bool
     ├── visit_i64 / visit_u64 / visit_f64
     ├── visit_str / visit_string
     ├── visit_seq
     └── visit_map
  → 你的 Visitor 返回目标类型
```

一个能同时接受数字和数字字符串的端口类型：

```rust
use serde::de::{self, Visitor};
use std::fmt;

#[derive(Debug, PartialEq)]
struct Port(u16);

struct PortVisitor;

impl<'de> Visitor<'de> for PortVisitor {
    type Value = Port;

    fn expecting(&self, f: &mut fmt::Formatter) -> fmt::Result {
        f.write_str("0 到 65535 之间的整数或数字字符串")
    }

    fn visit_u64<E: de::Error>(self, value: u64) -> Result<Port, E> {
        u16::try_from(value)
            .map(Port)
            .map_err(|_| E::custom("端口超出 u16 范围"))
    }

    fn visit_str<E: de::Error>(self, value: &str) -> Result<Port, E> {
        value.parse::<u16>().map(Port).map_err(E::custom)
    }
}

impl<'de> serde::Deserialize<'de> for Port {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        deserializer.deserialize_any(PortVisitor)
    }
}
```

`expecting` 不是摆设：类型完全对不上时，serde 会用它拼出 `invalid type: ..., expected 0 到 65535 之间的整数或数字字符串`。

⚠ `deserialize_any` 依赖**自描述格式**，JSON 可以，bincode 不行。二进制格式要改用 `deserialize_u64` 或 `deserialize_str`，只接受一种类型。

# 4. with 模块

## 4.1 serialize_with 与 deserialize_with

只改一个方向时，这两个属性最省事：

```rust
fn serialize_upper<S>(value: &String, serializer: S) -> Result<S::Ok, S::Error>
where
    S: serde::Serializer,
{
    serializer.serialize_str(&value.to_uppercase())
}

#[derive(serde::Serialize)]
struct Tag {
    #[serde(serialize_with = "serialize_upper")]
    name: String,
}
```

```rust
#[derive(serde::Deserialize)]
struct Age {
    #[serde(deserialize_with = "deserialize_age")]
    value: u8,
}

fn deserialize_age<'de, D>(deserializer: D) -> Result<u8, D::Error>
where
    D: serde::Deserializer<'de>,
{
    let age = u8::deserialize(deserializer)?;
    if !(1..=150).contains(&age) {
        return Err(serde::de::Error::custom("年龄必须在 1 到 150 之间"));
    }
    Ok(age)
}
```

## 4.2 with 模块

两个方向都要改时，把函数放进模块，用 `#[serde(with = "模块名")]` 一次挂上：

```rust
mod ts_seconds {
    use chrono::{DateTime, TimeZone, Utc};
    use serde::{Deserialize, Deserializer, Serializer};

    pub fn serialize<S>(dt: &DateTime<Utc>, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_i64(dt.timestamp())
    }

    pub fn deserialize<'de, D>(deserializer: D) -> Result<DateTime<Utc>, D::Error>
    where
        D: Deserializer<'de>,
    {
        let secs = i64::deserialize(deserializer)?;
        Utc.timestamp_opt(secs, 0)
            .single()
            .ok_or_else(|| serde::de::Error::custom("时间戳超出可表示范围"))
    }
}

#[derive(serde::Serialize, serde::Deserialize)]
struct Event {
    #[serde(with = "ts_seconds")]
    created_at: chrono::DateTime<chrono::Utc>,
}
```

`chrono` 也自带 `chrono::serde::ts_seconds`、`ts_milliseconds` 等模块，直接写 `#[serde(with = "chrono::serde::ts_seconds")]` 即可；需要保留时区偏移时用默认的 RFC 3339。整个项目只选一种，不要混用。

## 4.3 三种方式对比

| 方式 | 作用范围 | 适合 |
|------|---------|------|
| `serialize_with` | 仅序列化 | 只改输出 |
| `deserialize_with` | 仅反序列化 | 只做校验/兼容 |
| `with = "module"` | 双向 | 日期、大整数等成套转换 |

# 5. 实战

## 5.1 大整数

`u128` 在多数前端会丢精度，用字符串传输：

```rust
mod u128_string {
    use serde::{Deserialize, Deserializer, Serializer};

    pub fn serialize<S>(value: &u128, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_str(&value.to_string())
    }

    pub fn deserialize<'de, D>(deserializer: D) -> Result<u128, D::Error>
    where
        D: Deserializer<'de>,
    {
        let text = String::deserialize(deserializer)?;
        text.parse().map_err(serde::de::Error::custom)
    }
}

#[derive(serde::Serialize, serde::Deserialize)]
struct Balance {
    #[serde(with = "u128_string")]
    amount: u128,
}
```

## 5.2 字段校验

校验放在反序列化里，能让非法数据在入口就被拒绝：

```rust
fn deserialize_ratio<'de, D>(deserializer: D) -> Result<f64, D::Error>
where
    D: serde::Deserializer<'de>,
{
    let ratio = f64::deserialize(deserializer)?;
    if !(0.0..=1.0).contains(&ratio) {
        return Err(serde::de::Error::custom("ratio 必须在 0 到 1 之间"));
    }
    Ok(ratio)
}

#[derive(serde::Deserialize)]
struct Threshold {
    #[serde(deserialize_with = "deserialize_ratio")]
    ratio: f64,
}
```

★ **关键认知**：`serde::de::Error::custom` 是自定义错误的标准出口，它会保留当前位置信息，最终变成 `ratio 必须在 0 到 1 之间 at line 1 column 20`。⚠ 校验只保证"单字段合法"，跨字段约束（比如 `end > start`）需要在整个结构体反序列化完成后再检查。

# 6. 常见错误

| 报错/现象 | 原因 | 解决 |
|-----------|------|------|
| `invalid type: string, expected u64` | 只实现了数字读取 | Visitor 加 `visit_str` |
| `deserialize_any is not supported` | 用了非自描述格式 | 改用具体 `deserialize_*` |
| `custom error` 没有指到字段 | 跨字段校验写太晚 | 在字段级函数里检查 |
| 时间差 8 小时 | 用了本地时区 | 统一 `DateTime<Utc>` |
| 大整数变 `1e19` | 前端按 double 解析 | 改字符串传输 |
| `with` 模块找不到函数 | 函数不是 `pub` 或名字不对 | 模块内导出 `serialize`/`deserialize` |

# 7. 练习

1. 实现一个 `Percentage(u8)`，要求序列化成 `"12%"`，反序列化同时接受 `"12%"` 和 `12`，非法值返回 `custom` 错误。
2. 给一个用户列表结构体加上 `#[serde(with = "chrono::serde::ts_milliseconds")]` 的注册时间字段，分别用秒级和毫秒级数据测试，观察错误。
3. 写一个 `deserialize_with` 校验手机号：必须是 11 位数字且以 `1` 开头，写出对应的错误提示并测试空字符串和带字母的输入。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 手写 Serialize | 选择 `serialize_*` 方法描述自己 |
| 手写 Deserialize | 用 Visitor 接收多种输入类型 |
| `deserialize_any` | 仅自描述格式可用 |
| `with` 模块 | 双向转换，模块需导出两个函数 |
| 日期 | 统一 RFC 3339 或时间戳，别混用 |
| 大整数 | `u128`/`i128` 用字符串传输 |
| 校验 | `Error::custom` 保留位置信息 |

下一篇进入错误处理与最佳实践，讲清楚未知字段策略、错误上下文、性能取舍和安全边界，并给出一份可以直接照着做的检查清单。
