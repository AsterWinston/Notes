Protobuf（Protocol Buffers）是 Google 开源的结构化数据序列化格式。它不像 JSON 那样直接把字段名写进数据，而是先用 `.proto` 文件定义 schema，再由 `protoc` 生成各语言的代码。理解这套"先定义、再生成、后传输"的流程，是后面所有内容的地基。

# 1. Protobuf 的定位

## 1.1 它解决什么问题

一个服务要把对象发给另一个服务，本质要做三件事：

- 把内存里的对象变成字节流，这叫**序列化**。
- 把字节流还原成对象，这叫**反序列化**。
- 双方对"这段字节里哪个位置是什么字段"达成一致，这叫**契约**。

JSON 用字段名自描述，读起来直观，但每次都把 `"userName"` 这样的字符串重复传一遍，体积大、解析慢。Protobuf 的做法是把字段名换成一个数字编号，并在 schema 里约定编号的含义。

```text
JSON：  {"userName":"alice","age":18}      ← 字段名自带语义，但重复传输
Proto： 0A 05 61 6C 69 63 65 10 12        ← 编号 + 值，字段名在 schema 里
```

★ **关键认知**：Protobuf 不是"更小的 JSON"，而是**带代码生成的二进制契约格式**。它的核心收益来自三处：字段名不参与传输、变长整数编码、schema 驱动生成强类型代码。代价是你必须先有 `.proto` 文件，不能像 JSON 那样随手拼一个对象。

## 1.2 与其他格式的对比

| 维度 | Protobuf | JSON | XML |
|------|----------|------|-----|
| 可读性 | 二进制，人眼不可读 | 好 | 好 |
| 体积 | 小 | 中 | 大 |
| 解析速度 | 快 | 中 | 慢 |
| 类型系统 | 强，编译期检查 | 弱，运行时判断 | 弱，靠 schema |
| 契约 | `.proto` 强制 | 口头或 OpenAPI | XSD/DTD |
| 浏览器支持 | 需库 | 原生 | 原生 |
| 典型场景 | gRPC、内部服务 | Web API、日志 | 文档、SOAP |

⚠ **关键认知**：Protobuf 的可读性差是**故意的取舍**。它面向机器之间传输，不面向人直接调试。需要抓包排查时，要借助 `protoc --decode` 或服务端反射，不能像 JSON 那样直接看。

# 2. proto3 基本结构

## 2.1 第一个 .proto 文件

```protobuf
syntax = "proto3";              // 声明语法版本，必须写在第一行有效语句

package demo.user.v1;           // 命名空间，决定全限定名

option go_package = "example.com/demo/gen/user/v1;userv1";  // Go 代码的导入路径与包名

message User {
  int64  id    = 1;             // 字段类型 名称 = 编号;
  string name  = 2;
  int32  age   = 3;
  bool   active = 4;
}
```

逐段说明：

- `syntax`：不写会默认按 proto2 解析，行为完全不同，务必显式声明。
- `package`：避免不同业务的消息重名，最终类型全名是 `demo.user.v1.User`。
- `option go_package`：只影响 Go 生成代码；其他语言有各自的 option。
- `message`：相当于一个类或结构体，里面是一组带编号的字段。

## 2.2 一个文件里能放什么

```protobuf
syntax = "proto3";
package demo.user.v1;

import "google/protobuf/timestamp.proto";   // 引入外部定义

option java_package = "com.example.demo.user.v1";
option java_multiple_files = true;

enum Status { /* ... */ }
message User { /* ... */ }
service UserService { /* ... */ }
```

一个 `.proto` 文件可以包含 `import`、`option`、`enum`、`message`、`service`。文件本身不产生运行时对象，只是"定义源"。

# 3. message 与字段

## 3.1 字段编号

字段编号是 Protobuf 的灵魂，规则如下：

- 取值范围 `1 ~ 536870911`（2^29 - 1）。
- `19000 ~ 19999` 是 Protobuf 内部保留区，不能用。
- `1 ~ 15` 编码后 tag 只占 1 字节，留给最高频字段。
- 编号一旦上线就不能改，也不能删了再给新字段用。

```protobuf
message Order {
  string id       = 1;    // 高频，1 字节 tag
  int64  user_id  = 2;
  int64  amount   = 3;
  string remark   = 16;   // 低频，2 字节 tag
  reserved 4, 6 to 8;     // 4、6、7、8 永久禁用
  reserved "old_status";  // 字段名也锁住
}
```

★ **关键认知**：**线上只认编号，不认字段名**。把 `name = 2` 改名为 `user_name = 2` 是兼容的；把 `name = 2` 改成 `name = 5` 是破坏性的——老数据里的 2 号字段会被新代码当成未知字段丢弃。

## 3.2 标量类型

| proto 类型 | 说明 | 对应 Java | 对应 Go |
|------------|------|-----------|---------|
| `double` / `float` | 浮点 | `double` / `float` | `float64` / `float32` |
| `int32` / `int64` | 变长有符号，负数编码效率低 | `int` / `long` | `int32` / `int64` |
| `uint32` / `uint64` | 变长无符号 | `int` / `long` | `uint32` / `uint64` |
| `sint32` / `sint64` | ZigZag 编码，适合负数 | `int` / `long` | `int32` / `int64` |
| `fixed32` / `fixed64` | 定长，适合大整数或哈希 | `int` / `long` | `uint32` / `uint64` |
| `sfixed32` / `sfixed64` | 定长有符号 | `int` / `long` | `int32` / `int64` |
| `bool` | 布尔 | `boolean` | `bool` |
| `string` | UTF-8 或 ASCII | `String` | `string` |
| `bytes` | 任意字节 | `ByteString` | `[]byte` |

⚠ **关键认知**：`int32` 存负数会编码成 10 字节（因为负数被当作很大的无符号数做 varint），而 `sint32` 用 ZigZag 编码，负数只占少量字节。**字段可能出现负值时选 `sint32`/`sint64`**，这是最容易被忽略的性能点。

## 3.3 字段修饰符

proto3 的字段默认都是可选的，主要有三种形态：

```protobuf
message Profile {
  string nickname = 1;            // 普通字段，零值不可区分"未传"
  optional string bio = 2;        // 显式存在性，生成 hasXxx()
  repeated string tags = 3;       // 列表，默认 packed 编码
}
```

`optional` 在 proto3 里不是"可以不填"的意思（本来就可以不填），而是**恢复字段存在性**：

```java
// Java 生成代码
if (profile.hasBio()) {          // 能区分"没传"与"传了空字符串"
    System.out.println(profile.getBio());
}
```

★ **关键认知**：普通字段收到 `age = 0` 时，你无法判断对方是"显式传了 0"还是"根本没传这个字段"。在更新类接口里这很危险——把年龄改成 0 和"不要动年龄"是两种语义，只有 `optional` 能表达。

# 4. 复合类型

## 4.1 枚举

```protobuf
enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;   // 必须有一个 0 值
  USER_STATUS_ACTIVE      = 1;
  USER_STATUS_DISABLED    = 2;
}
```

规则：

- 第一个枚举值必须是 `0`，它是默认值。
- 枚举值名在同一 package 内不能重复，所以推荐加前缀。
- proto3 是**开放枚举**：收到未定义的数字不会报错，而是原样保留。

⚠ **关键认知**：因为开放枚举，服务端新增 `USER_STATUS_DELETED = 3` 后，老客户端依然能解析消息，但 `switch` 会落到 `default`。**任何枚举 switch 都必须写 default 分支**，否则新增值会被静默忽略。

## 4.2 map

```protobuf
message Config {
  map<string, string> labels = 1;   // 键只能是标量类型
  map<int64, Item>    items  = 2;   // 值可以是 message
}
```

★ **关键认知**：`map` 在线上等价于 `repeated Entry{key, value}`，**遍历顺序不保证**。如果业务依赖顺序（例如配置优先级），应该用 `repeated` 加一个显式的 `order` 字段，而不是指望 map 的顺序。

## 4.3 oneof

```protobuf
message Payment {
  oneof method {
    CardPayment   card   = 1;
    AlipayPayment alipay = 2;
    WechatPayment wechat = 3;
  }
}
```

`oneof` 表示"这几个字段同时最多只有一个有值"。设置其中一个会自动清空其他。

```go
switch m := p.GetMethod().(type) {
case *pb.Payment_Card:
    fmt.Println("银行卡", m.Card.GetCardNo())
case *pb.Payment_Alipay:
    fmt.Println("支付宝")
case nil:
    fmt.Println("未选择支付方式")   // 必须处理 nil
}
```

★ **关键认知**：`oneof` 不只是语义约束，还**节省内存**——多个字段共享一块存储。它适合表达"多选一"，不适合表达"可能同时有多个"。

## 4.4 嵌套 message

```protobuf
message Order {
  message Item {              // 嵌套定义，全名 Order.Item
    string sku = 1;
    int32  qty = 2;
  }
  repeated Item items = 1;
}
```

嵌套可以避免命名污染，但层次过深会让类型名很长。一般建议：只有确实从属于外层、不会单独复用的类型才嵌套。

# 5. 默认值与存在性

proto3 没有显式的 `null`，每种类型都有零值：

| 类型 | 零值 |
|------|------|
| 数值 | `0` |
| 布尔 | `false` |
| 字符串 | `""` |
| bytes | 空字节串 |
| 枚举 | 编号 `0` 的值 |
| message | `null` / 未设置 |
| repeated / map | 空集合 |

```text
序列化时不写零值字段  →  这是 Protobuf 体积小的原因之一
反序列化时缺失 = 零值  →  这是"无法区分未传"的根源
```

⚠ **关键认知**：proto3 在序列化时会**跳过零值字段**。所以 `{"age": 0}` 编码后可能什么都不剩。需要精确表达"传了 0"时，用 `optional`；需要表达"字段一定存在"时，可以在业务层约定或使用 `google.protobuf.Int32Value` 这类 wrapper 类型。

# 6. 常见错误

| 错误写法 | 报错/后果 | 正确做法 |
|----------|-----------|----------|
| 不写 `syntax` | 按 proto2 解析，`optional` 语义不同 | 首行写 `syntax = "proto3";` |
| 字段编号重复 | 编译报错 | 每个字段唯一编号 |
| 枚举首值非 0 | 编译报错 | 设置 `XXX_UNSPECIFIED = 0` |
| 用 `19000` 编号 | 编译报错 | 换到合法区间 |
| 删除字段不复用 `reserved` | 未来编号被误用，数据错位 | `reserved 编号` 和 `reserved "名字"` |
| `map` 键用 message | 编译报错 | 键只能是整数或字符串 |
| 把字段名当协议 | 改名字以为不兼容 | 记住线上只认编号 |

# 7. 练习

1. 写一个 `Product` 消息，包含 `id`、`name`、`price`（分为单位的整数）、`tags` 列表、`attributes` 键值对，并给出合理编号。思考为什么价格不用 `double`。
2. 设计一个 `Notification` 消息，用 `oneof` 表达"短信、邮件、站内信三选一"，并说明如果未来要支持"同时发短信和邮件"，这个设计要怎么改。
3. 把下面的 proto2 风格字段改写成 proto3，并解释 `required` 为什么被移除：

   ```protobuf
   message Old {
     required string name = 1;
     optional int32  age  = 2;
   }
   ```

# 8. 小结

| 概念 | 要点 |
|------|------|
| 定位 | schema 驱动的二进制契约格式，靠代码生成获得强类型 |
| 字段编号 | 线上身份，1~15 高频，删字段用 `reserved` 锁住 |
| 标量类型 | 负数用 `sint`，大整数或哈希用 `fixed` |
| optional | 区分"未传"与"零值"，更新接口必备 |
| 枚举 | 首值必须 0，开放枚举，switch 必须有 default |
| map | 顺序不保证，依赖顺序时改用 repeated |
| oneof | 多选一，省内存，必须处理 nil |
| 默认值 | 零值不序列化，是体积小的原因，也是歧义的来源 |

下一篇进入 schema 与兼容性，讲清楚字段编号的完整规则、增删字段的安全边界、`reserved` 的用法，以及如何用 `buf breaking` 把兼容性问题挡在合并之前。
