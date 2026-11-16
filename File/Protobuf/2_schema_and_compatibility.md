Protobuf 最贵的错误不是写错语法，而是**改错 schema**。语法错误编译时就报，改错字段编号却可能在线上跑三个月才暴露，而且数据已经写进日志、数据库和消息队列。本篇把兼容性规则讲成可执行的判断标准。

# 1. 为什么兼容性问题特殊

## 1.1 一次发布，多端共存

```text
时间轴 ──────────────────────────────────────────►

客户端 A（旧 schema）  ────────────────┐
客户端 B（新 schema）  ────────┐       │  同时在线
服务端   （新 schema）  ────────┴───────┴──────►
                      │
                      └─ 老数据还可能躺在数据库里
```

移动端、桌面端、第三方接入方不可能同时升级。**任何时刻都存在至少两个版本的 schema 在互相读写**。兼容性不是"要不要考虑"，而是"必须默认成立"。

★ **关键认知**：兼容性的本质是"**同一段字节，在两版代码里都能解析出合理结果**"。新代码读老数据叫**向后兼容**，老代码读新数据叫**向前兼容**。判断任何改动，只要问这两句即可。

## 1.2 破坏通常来自语义

Protobuf 的增量改动大多双向兼容，因为未知字段会被保留（详见第 5 篇）。真正危险的是"字节没变，但字段含义变了"，例如把两个普通字段移进 `oneof`。

# 2. 字段编号规则

## 2.1 编号区间与编码成本

| 区间 | tag 字节数 | 建议用途 |
|------|-----------|----------|
| `1 ~ 15` | 1 | 最高频字段 |
| `16 ~ 2047` | 2 | 中频字段 |
| `2048 ~ 536870911` | 3~5 | 低频字段 |
| `19000 ~ 19999` | 不可用 | Protobuf 内部保留 |

```protobuf
message Event {
  string id      = 1;     // 每条消息都有，放 1
  int64  ts      = 2;     // 高频
  string type    = 3;     // 高频
  string payload = 16;    // 体积大但不必抢 1 字节 tag
  string trace   = 1000;  // 低频调试字段
}
```

★ **关键认知**：编号越小，tag 占的字节越少，但这只在**超高频、超小消息**上才有明显收益。不要为了省字节把所有字段挤进 1~15，正确做法是**按实际频率排序**。

## 2.2 编号是永久承诺

`19000 ~ 19999` 是编译器和运行时的内部保留区，用了会直接编译失败，不需要你手动 `reserved`。真正需要你锁的是**业务字段删除后空出来的编号**。

```protobuf
message User {
  string name  = 2;
  string phone = 2;   // 编译直接报错：编号重复
}
```

编译期能挡住同文件重复，但挡不住"删掉字段后把编号让给新字段"。后者编译能过，线上却会把老数据里的 `name` 解析成 `phone`。

★ **关键认知**：**编号是写入历史的，改不回来**。删除字段时必须同时 `reserved` 编号和字段名，这是唯一可靠的防线。

# 3. 增删改字段

## 3.1 新增字段：默认安全

```protobuf
// v1
message User { int64 id = 1; }

// v2
message User {
  int64  id   = 1;
  string name = 2;   // 新增
}
```

- 新代码读老数据：`name` 不存在，取零值 `""`。
- 老代码读新数据：`name` 是未知字段，保留但不解析。

★ **关键认知**：新增字段安全的前提是**老代码能接受零值**。如果新字段语义是"必须存在"，老数据读出来就是空，业务层要能兜住。这是业务默认值设计的问题，不是 Protobuf 的问题。

## 3.2 删除字段：必须 reserved

```protobuf
message User {
  int64 id = 1;
  reserved 3;            // 原 email 的编号
  reserved "email";      // 字段名一起锁
  reserved 5, 9 to 11;   // 批量锁编号，9~11 含两端
  string phone = 4;
}
```

⚠ **关键认知**：只写 `reserved 3` 不够，还要 `reserved "email"`。有人可能换个编号重新加回同名字段，而 JSON 映射、反射、动态消息都会用到字段名，重复名字仍会引发混乱。**编号和名字一起锁**才是完整做法。

## 3.3 改字段名：二进制安全

```protobuf
string name = 2;        // 旧
string user_name = 2;   // 新，编号没变
```

线上只认编号，改名不影响二进制解析。但会影响 JSON 映射（`protojson` 用 lowerCamel）、生成代码的属性名、日志标签。

★ **关键认知**：改名对**二进制协议**兼容，对**JSON 协议和源码**不兼容。如果服务同时暴露 gRPC 和 HTTP+JSON，改名要按"对外 API 变更"对待。

## 3.4 改字段类型：看线格式

判断标准只有一个：**两种类型的 wire type 和编码方式是否一致**。

| 变更 | 兼容性 | 原因 |
|------|--------|------|
| `int32` → `int64` | 兼容 | 都是 varint，值范围不同 |
| `int32` → `uint32` | 部分兼容 | 负数会被解释成巨大正数 |
| `int32` → `sint32` | 不兼容 | varint 与 ZigZag 编码不同 |
| `string` → `bytes` | 兼容 | 都是长度前缀 + 字节 |
| `fixed32` → `sfixed32` | 兼容 | 都是 4 字节定长 |
| `fixed32` → `int32` | 不兼容 | 定长与变长编码不同 |
| 单值 → `repeated` | 不兼容 | packed 与单值解析不同 |
| `message` → `string` | 不兼容 | wire type 不同 |

```text
wire type（tag 低 3 位）：
0 = varint   int32/int64/uint/bool/enum/sint
1 = 64-bit   fixed64/sfixed64/double
2 = length   string/bytes/message/repeated(packed)
5 = 32-bit   fixed32/sfixed32/float
```

⚠ **关键认知**：只要 wire type 变了，解析器就会把这段字节当成另一种东西读，结果是**静默产生错误数据**，而不是抛异常。类型变更不要凭记忆，用工具检测。

# 4. 兼容性矩阵

## 4.1 常见操作速查

| 操作 | 向后兼容 | 向前兼容 | 结论 |
|------|---------|---------|------|
| 新增字段 | 是 | 是 | 安全 |
| 删除字段并 `reserved` | 是 | 是 | 安全 |
| 删除字段不 `reserved` | 是 | 是 | 当下可用，埋雷 |
| 改字段名 | 是 | 是 | 二进制安全，JSON 不安全 |
| 改字段编号 | 否 | 否 | 禁止 |
| 改字段类型 | 视情况 | 视情况 | 按 wire type 判断 |
| 新增枚举值 | 是 | 是 | 安全，但要有 default |
| 删除枚举值 | 是 | 是 | 需 `reserved` 编号 |
| 改枚举值编号 | 否 | 否 | 禁止 |
| 新增 `oneof` 字段 | 是 | 是 | 安全 |
| 把已有字段移进 `oneof` | 否 | 否 | 禁止 |
| 改 service 方法名 | 否 | 否 | 禁止 |
| 改 package | 否 | 否 | 等于换类型 |

## 4.2 线格式之外的语义兼容

把两个普通字段移进 `oneof`，线上字节完全一样，看起来应该兼容。但 `oneof` 的语义是"同时只有一个"，旧代码可能同时给两个字段赋值，新代码只会保留最后一个。

★ **关键认知**：兼容性有两层——**线格式兼容**和**语义兼容**。前者靠 wire type，后者靠对字段含义的约束。`buf breaking` 能覆盖前者和一部分后者，业务语义仍要靠评审。

# 5. 枚举与 oneof

```protobuf
enum Status {
  STATUS_UNSPECIFIED = 0;
  STATUS_ACTIVE      = 1;
  reserved 2;                  // 删除的编号锁住
  reserved "STATUS_OLD";
  STATUS_DISABLED    = 3;      // 新值只能用没被 reserved 的编号
}

message Payment {
  oneof method {
    Card   card   = 1;
    Alipay alipay = 2;
    reserved 3;                // 曾经是 Wechat
    Bank   bank   = 4;         // 新增
  }
}
```

| 操作 | 结论 |
|------|------|
| 新增枚举值 / `oneof` 字段 | 安全，老代码保留数字或忽略 |
| 删除枚举值 / `oneof` 字段 | 必须 `reserved` 编号 |
| 改枚举值编号 | 禁止，等于改变已有数据语义 |
| 把普通字段移进 `oneof` | 破坏性 |
| 把 `oneof` 拆成普通字段 | 破坏性 |

⚠ **关键认知**：不要写 `if (status == 0)` 这种魔法数字判断，用生成的常量。枚举编号一旦发布就是协议的一部分，和字段编号同等重要。

# 6. 命名规范

## 6.1 字段与消息

| 对象 | 规范 | 示例 |
|------|------|------|
| message | UpperCamelCase | `OrderItem` |
| 字段 | lower_snake_case | `user_id` |
| 枚举类型 | UpperCamelCase | `OrderStatus` |
| 枚举值 | UPPER_SNAKE_CASE，带类型前缀 | `ORDER_STATUS_PAID` |
| service | UpperCamelCase | `OrderService` |
| rpc 方法 | UpperCamelCase | `CreateOrder` |

★ **关键认知**：`.proto` 用 snake_case，生成器会自动转成各语言习惯（Java 的 `getOrderId`、Go 的 `OrderId`、JS 的 `orderId`）。**在 proto 里手写 camelCase 会让转换失真**，比如 `OrderID` 在 Go 里可能变成 `OrderID` 而非常见的 `OrderId`。

## 6.2 版本与包

```text
proto/
├── user/v1/user.proto
├── user/v2/user.proto
├── order/v1/order.proto
└── common/v1/pagination.proto
```

```protobuf
package demo.user.v1;
option go_package = "example.com/demo/gen/user/v1;userv1";
option java_package = "com.example.demo.user.v1";
```

版本目录让新老接口并行，全限定名带版本后路由不冲突，客户端可以灰度迁移。

⚠ **关键认知**：`v1` 代表**兼容性承诺**，不是代码版本号。`v1` 内只能做兼容增量；要删字段、改语义、改类型，就开 `v2`。把 `v1` 当"随时可改"是团队协作里最常见的灾难。

# 7. 用 buf 守住兼容性

```bash
brew install bufbuild/buf/buf          # 安装
buf breaking --against '.git#branch=main'   # 对比 main，检测破坏性变更

# 典型输出
# Field "3" with name "email" on message "User" is deleted
# without reserving the number and name.
```

| 规则集 | 严格程度 | 说明 |
|--------|---------|------|
| `FILE` | 最严 | 任何文件级破坏都报 |
| `PACKAGE` | 中 | 包内兼容即可 |
| `WIRE_JSON` | 宽 | 只看二进制与 JSON 兼容 |
| `WIRE` | 最宽 | 只看二进制 |

★ **关键认知**：把 `buf breaking` 放进 CI，**在合并前拦截**，比线上出问题再回滚便宜两个数量级。规则集至少用 `FILE`，如果对外提供 JSON 接口，用 `WIRE_JSON` 更合适。

# 8. 练习

1. 下面这个改动是否兼容？逐条说明原因：

   ```protobuf
   // v1
   message M { string a = 1; int32 b = 2; }
   // v2
   message M {
     string a = 1;
     reserved 2;
     reserved "b";
     int64  c = 3;
   }
   ```

2. 团队想删掉 `User.email = 3`，有人直接删除没加 `reserved`。三个月后新人把编号 3 分配给 `phone`。请描述线上会发生什么，并写出修复方案。
3. 设计一个 `v2` 版本的 `UserService`，要求老客户端继续可用，新客户端能拿到新增的 `avatar` 字段。说明哪些字段要保留、哪些可以新增。

# 9. 小结

| 主题 | 结论 |
|------|------|
| 编号 | 一旦发布永久占用，1~15 给高频字段 |
| 删除 | 必须 `reserved` 编号和名字 |
| 新增 | 安全，但业务层要能接受零值 |
| 改名 | 二进制安全，JSON 和源码不兼容 |
| 改类型 | 看 wire type，不同则禁止 |
| 枚举 | 首值 0，新增安全，删除要 reserved |
| oneof | 就地加字段安全，移入移出破坏性 |
| 版本 | `v1` 内只做兼容增量，破坏性改动用 `v2` |
| 工具 | `buf breaking` 进 CI，别靠记忆 |

下一篇进入代码生成，讲 `protoc` 的工作原理、各语言插件怎么装、生成的代码长什么样，以及如何把生成过程接进 Maven、Gradle、Go 和 CI。
