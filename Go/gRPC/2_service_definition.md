`.proto` 写起来像配置，实际上是**线上协议**。字段编号、类型、枚举值一旦发布就进了历史包袱，改错一次可能要背几年。本篇把服务定义里的语法规则和兼容性约束讲透，让你在改 proto 时知道哪些能改、哪些碰不得。

# 1. service 与 message 的组织

## 1.1 一个完整定义

```protobuf
syntax = "proto3";

package user.v1;

option go_package = "example.com/shop/gen/user/v1;userv1";

import "google/protobuf/timestamp.proto";

// 用户服务：按业务域划分 service，而不是按技术层
service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserReply);
  rpc ListUsers (ListUsersRequest) returns (ListUsersReply);
  rpc CreateUser (CreateUserRequest) returns (CreateUserReply);
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserReply {
  User user = 1;
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  google.protobuf.Timestamp created_at = 4;
  repeated string tags = 5;
  map<string, string> attributes = 6;
  UserStatus status = 7;
}

enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;  // 枚举零值必须有，表示未设置
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_DISABLED = 2;
}
```

★ **关键认知**：`service` 的粒度应该按**业务能力**切，而不是按"一张表一个 service"。一个 `UserService` 里放用户相关的增删改查，比 `UserQueryService`、`UserCommandService`、`UserAuthService` 更利于维护，因为调用方只需要一个客户端。

## 1.2 请求/响应消息的命名

约定俗成的命名方式：

| 场景 | 命名 | 示例 |
|------|------|------|
| 方法请求 | `<方法名>Request` | `GetUserRequest` |
| 方法响应 | `<方法名>Reply` / `Response` | `GetUserReply` |
| 领域实体 | 名词 | `User`、`Order` |
| 分页请求 | 独立消息复用 | `PageRequest` |
| 通用响应 | 独立消息复用 | `PageReply` |

⚠ **关键认知**：不要让多个 RPC 共用同一个 `Request`。今天 `GetUser` 和 `DeleteUser` 都收 `UserIdRequest`，明天 `DeleteUser` 要加一个 `hard_delete` 字段，就会污染 `GetUser`。**每个方法一个请求消息**是低成本的隔离。

# 2. 字段编号规则

## 2.1 编号不是随便写的

```protobuf
message Order {
  string id = 1;          // 高频字段用 1~15，编码后 tag 只占 1 字节
  int64 user_id = 2;
  int64 amount = 3;
  string remark = 16;     // 低频、较长的字段往后放
  reserved 4, 6 to 8;     // 4、6、7、8 永久禁用，防止误复用
  reserved "old_status";  // 字段名也保留，防止 JSON 映射冲突
}
```

编号规则总结：

- `1 ~ 15`：编码开销最小，给最常用的字段。
- `16 ~ 2047`：tag 占 2 字节，中频字段。
- `2048` 以上：tag 占更多字节，低频字段。
- `19000 ~ 19999`：Protobuf 内部保留，用了会编译报错。
- 删除字段时用 `reserved` 占位，编号和名字都不能给新字段用。

★ **关键认知**：**编号是传输层的身份，字段名只是代码里的名字**。把 `name = 2` 改成 `user_name = 2` 是兼容的；把 `name = 2` 改成 `name = 5` 是不兼容的，老客户端会把 2 号字段当未知字段丢弃，新服务端读到 0 值。

## 2.2 类型变更的兼容矩阵

| 变更 | 是否兼容 | 说明 |
|------|----------|------|
| `int32` ↔ `int64` | 兼容 | 变长编码，值范围不同但线上可读 |
| `int32` ↔ `uint32` | 部分兼容 | 负数会变成巨大正数 |
| `string` ↔ `bytes` | 兼容 | 二进制层面一致，需自行保证 UTF-8 |
| `int32` ↔ `sint32` | **不兼容** | 编码方式不同，值会错乱 |
| `fixed32` ↔ `sfixed32` | 兼容 | 都是 4 字节 |
| 单字段 ↔ `repeated` | 部分兼容 | proto3 的 packed 编码可能读错 |
| `message` ↔ `string` | **不兼容** | 线格式完全不同 |

⚠ **关键认知**：上表只能当参考，真正的安全线是 **"不改线格式（wire format）"**。拿不准时用 `buf breaking` 做自动化检测，不要靠记忆。

# 3. 字段修饰符

## 3.1 optional：恢复字段存在性

```protobuf
message Profile {
  string nickname = 1;            // 普通字段：0 值无法区分"未传"
  optional string bio = 2;        // 显式存在性：生成指针 + HasBio()
  optional int32 age = 3;
}
```

生成的 Go 代码：

```go
type Profile struct {
    Nickname string  `protobuf:"bytes,1,opt,name=nickname,proto3"`
    Bio      *string `protobuf:"bytes,2,opt,name=bio,proto3,oneof"`
    Age      *int32  `protobuf:"varint,3,opt,name=age,proto3,oneof"`
}

// 使用时需要判空
if p.Bio != nil {
    fmt.Println(*p.Bio)
}
```

★ **关键认知**：`optional` 的价值不是"可以不填"（proto3 字段本来都可选），而是**区分"没传"与"传了零值"**。更新接口里这一点尤其重要：`age = 0` 到底是"把年龄改成 0"还是"这个字段别动"，只有 `optional` 能表达。

## 3.2 repeated：列表

```protobuf
message SearchRequest {
  repeated string keywords = 1;   // 标量列表默认 packed 编码
  repeated Item items = 2;        // 消息列表
}

message Item {
  string sku = 1;
  int32 qty = 2;
}
```

```go
req := &pb.SearchRequest{
    Keywords: []string{"go", "grpc"},
    Items:    []*pb.Item{{Sku: "A-1", Qty: 2}},
}
```

⚠ **关键认知**：proto3 里 `repeated` 标量默认使用 **packed 编码**（连续值打包成一个字段），而 proto2 默认不打包。跨版本混用时要显式写 `[packed = true]`，否则解析可能出问题。

## 3.3 map：键值对

```protobuf
message Config {
  map<string, string> labels = 1;   // 键只能是标量类型，不能是 message
  map<int64, Item> items = 2;       // 值可以是 message
}
```

```go
cfg := &pb.Config{
    Labels: map[string]string{"env": "prod"},
}
```

★ **关键认知**：`map` 在线上等价于 `repeated Entry{key, value}`，**顺序不保证**。如果业务依赖顺序（比如配置优先级），应该用 `repeated` 加显式 `order` 字段，而不是 map。

## 3.4 oneof：互斥字段

```protobuf
message Payment {
  oneof method {
    CardPayment card = 1;
    AlipayPayment alipay = 2;
    WechatPayment wechat = 3;
  }
}

message CardPayment {
  string card_no = 1;
}
```

生成的 Go 代码使用接口表达"当前是哪一个"：

```go
p := &pb.Payment{
    Method: &pb.Payment_Card{
        Card: &pb.CardPayment{CardNo: "6222"},
    },
}

switch m := p.Method.(type) {
case *pb.Payment_Card:
    fmt.Println("card", m.Card.CardNo)
case *pb.Payment_Alipay:
    fmt.Println("alipay")
case nil:
    fmt.Println("未选择支付方式")
}
```

★ **关键认知**：`oneof` 同时只保留一个字段，设置新值会清掉旧值，**还能省内存**（多个字段共享一块存储）。它适合表达"多选一"的语义，而不是"可能同时有多个"的列表。

# 4. 枚举

## 4.1 枚举的硬性规则

```protobuf
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;  // 必须有一个 0 值
  ORDER_STATUS_CREATED = 1;
  ORDER_STATUS_PAID = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_CANCELLED = 4;
}
```

规则：

- 第一个值必须是 `0`，作为默认值。
- 同一 `package` 内枚举值名不能重复（C++ 作用域规则）。
- 枚举值推荐加**前缀**（`ORDER_STATUS_`），避免命名冲突。
- 枚举值编号不要复用，删除时用 `reserved`。

⚠ **关键认知**：proto3 的枚举是**开放枚举**——服务端收到未定义的值不会报错，而是保留数字。这意味着新增枚举值对老客户端是兼容的，但老客户端 switch 时会落到 `default`。所以**永远要处理 default 分支**，不要假设枚举只有已知值。

## 4.2 枚举与 Go 代码

```go
func statusText(s pb.OrderStatus) string {
    switch s {
    case pb.OrderStatus_ORDER_STATUS_CREATED:
        return "已创建"
    case pb.OrderStatus_ORDER_STATUS_PAID:
        return "已支付"
    default:
        return "未知状态"   // 必须兜底，兼容未来新增值
    }
}
```

# 5. 包名与版本

## 5.1 为什么需要版本

```text
proto/
├── user/v1/user.proto
├── user/v2/user.proto
├── order/v1/order.proto
└── common/v1/pagination.proto
```

```protobuf
package user.v1;
option go_package = "example.com/shop/gen/user/v1;userv1";
```

版本化的好处：

- 新老接口可以**并行存在**，服务端同时注册 `UserServiceV1` 和 `UserServiceV2`。
- 全限定方法名带版本：`/user.v1.UserService/GetUser`，路由不会冲突。
- 客户端可以按版本灰度迁移。

★ **关键认知**：版本号代表**兼容性承诺**，不是"代码版本"。`v1` 内只能做向后兼容的增量修改；一旦要删字段、改语义、改类型，就应该开 `v2`，而不是在 `v1` 上硬改。

## 5.2 package 与 go_package 的分工

| 声明 | 作用 |
|------|------|
| `package user.v1` | Protobuf 命名空间，决定全限定服务名 |
| `option go_package` | Go 导入路径与包名，决定生成代码放哪 |
| 目录结构 | 通常与 package 对齐，便于 `buf` 管理 |

⚠ **关键认知**：`package` 和 `go_package` 不一致不会报错，但会让"proto 全限定名"和"Go 导入路径"两套身份脱节，排查注册冲突时很痛苦。建议保持目录、`package`、`go_package` 三者语义一致。

# 6. 向后兼容的实践清单

## 6.1 安全与危险操作

| 操作 | 结论 | 原因 |
|------|------|------|
| 新增字段 | 安全 | 老客户端当未知字段忽略 |
| 删除字段（用 `reserved`） | 安全 | 老客户端读到默认值 |
| 改字段名 | 安全 | 线上只认编号 |
| 改字段编号 | 危险 | 数据错位 |
| 改字段类型（跨线格式） | 危险 | 解析失败或值错乱 |
| 改枚举值编号 | 危险 | 语义漂移 |
| 改 `oneof` 内字段编号 | 危险 | 互斥关系错乱 |
| 改 `service` 方法名 | 危险 | 路由 `/pkg.Service/Method` 变了 |

## 6.2 删除字段的正确姿势

```protobuf
message User {
  int64 id = 1;
  string name = 2;
  // 删除 email 字段：编号和名字都保留，防止后来者复用
  reserved 3;
  reserved "email";
}
```

★ **关键认知**：直接删掉字段，编号会"空出来"。半年后有人觉得 3 号位空着可惜，拿来放 `phone`，老客户端里缓存的 `email` 就会被解析成 `phone`。`reserved` 就是给这个坑上锁。

## 6.3 用 buf 守住兼容性

```bash
# 对比 main 分支，检测破坏性变更
buf breaking --against '.git#branch=main'

# 输出示例
# proto/user/v1/user.proto:12:3: Field "3" with name "email" on message "User" is
# deleted without reserving the number and name.
```

# 7. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| `Field number 19000 is reserved` | 用了内部保留区 | 换编号 |
| `"xxx" is already defined in package` | 枚举值重名 | 加前缀或换 package |
| `Enum value must be zero` | 枚举首值非 0 | 设置 `XXX_UNSPECIFIED = 0` |
| 生成代码找不到类型 | `go_package` 路径不一致 | 统一 import 路径并 `go mod tidy` |
| 服务端收到 0 值 | 老客户端不认识新字段 | 正常，用 `optional` 或默认逻辑兜底 |
| 线上解析乱码 | 改过字段类型或编号 | 回滚，用 `buf breaking` 复查 |

# 8. 练习与自检

1. 设计一个 `OrderService`，包含创建、查询、取消三个方法，要求请求/响应消息各自独立，并给所有字段合理编号。
2. 把 `User.status` 从 `int32` 改成枚举，说明这是否兼容，以及老客户端会看到什么。
3. 用 `reserved` 删除一个字段后重新生成代码，确认 Go 结构体里该字段消失。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 服务粒度 | 按业务能力划分，每方法独立 Request/Reply |
| 编号 | 1~15 高频，删除用 `reserved` 锁住 |
| optional | 用于区分"未传"与"零值" |
| repeated/map/oneof | 列表、键值对、多选一的语义表达 |
| 枚举 | 首值必须 0，永远处理 default |
| 版本 | `v1` 内只做兼容增量，破坏性改动用 `v2` |
| 兼容性 | 认编号不认名字，改编号/类型/方法名最危险 |

下一篇进入四种调用方式，用完整 Go 示例讲清 Unary、Server Streaming、Client Streaming 和 Bidirectional Streaming，以及各自的适用场景。
