前五篇分别讲了语法、兼容性、代码生成、线格式和安全。最后这篇把它们收拢成一套可执行的工程实践：schema 怎么组织、Protobuf 和 JSON 与 gRPC 各是什么关系、线上出问题怎么查，以及一个团队从零落地时该按什么顺序做。

# 1. 把 schema 当成产品

## 1.1 schema 是契约，不是配置文件

常见误解是"proto 是我这边用的数据结构，想改就改"；正确认知是"proto 是双方签字的技术合同"。判断标准：如果这个 `.proto` 会被另一个团队、另一个进程或另一个版本引用，它就是**对外契约**，修改要经过兼容性检测、代码评审、版本发布、通知使用方四步。

★ **关键认知**：**改 proto 的成本不在写代码，而在协同**。字段编号一旦发布，就进入永久维护。所以设计阶段多花十分钟想清楚编号和字段，能省掉未来几个月的迁移。

## 1.2 目录组织

```text
proto/
├── common/v1/{pagination,error}.proto
├── user/v1/{user,user_service}.proto
├── user/v2/user.proto
└── order/v1/order.proto
```

组织原则：

| 原则 | 说明 |
|------|------|
| 按业务域分目录 | `user`、`order`，不按技术层 |
| 目录带版本 | `v1`、`v2`，和 `package` 对齐 |
| 消息与服务分离 | `user.proto` 放消息，`user_service.proto` 放 service |
| 公共类型集中 | 分页、错误、时间等放 `common` |

## 1.3 package、目录、生成路径三者一致

```protobuf
// proto/user/v1/user.proto
package demo.user.v1;
option go_package = "example.com/demo/gen/user/v1;userv1";
option java_package = "com.example.demo.user.v1";
```

目录 `proto/user/v1/`、`package demo.user.v1`、Go 路径 `.../gen/user/v1`、Java 包 `com.example.demo.user.v1` 应当是同一套语义。

⚠ **关键认知**：三者不一致**不会报错**，但会让"proto 全限定名"和"代码导入路径"两套身份脱节。排查服务注册冲突、反射找不到类型时，这种脱节会让人抓狂。**保持目录、package、生成路径语义一致**。

# 2. Protobuf 与 JSON 的关系

## 2.1 不是替代关系

| 场景 | 推荐 | 原因 |
|------|------|------|
| 内部服务间 RPC | Protobuf | 体积小、强类型、代码生成 |
| 浏览器直接调用 | JSON | 原生支持，无需生成 |
| 公开 API | JSON 为主 | 可调试、生态广 |
| 日志与事件 | JSON 或 Protobuf | 看下游消费能力 |
| 配置文件 | YAML/JSON | 人可读、可注释 |
| 移动端弱网 | Protobuf | 体积和解析优势明显 |

★ **关键认知**：真实系统里两者**长期共存**。典型架构是：内部 gRPC 用 Protobuf，边缘网关做 Protobuf ↔ JSON 转换，浏览器只看到 JSON。不要把"用了 Protobuf"理解成"所有地方都要用"。

## 2.2 proto3 的 JSON 映射

Protobuf 定义了标准 JSON 映射，`protojson`（Java 里的 `JsonFormat`）负责转换：

`message User { int64 id = 1; string user_name = 2; }` 会映射成 `{"id": "42", "userName": "alice"}`。注意几个反直觉点：

| 规则 | 说明 |
|------|------|
| `int64` 映射成字符串 | 避免 JS 精度丢失 |
| 字段名默认 lowerCamel | `user_name` → `userName` |
| 零值默认省略 | 除非设置 `EmitUnpopulated` |
| 枚举默认输出名字 | `"STATUS_ACTIVE"` 而非数字 |
| `bytes` 用 Base64 | 标准映射 |
| `map` 键会转字符串 | 数字键变成 `"1"` |

Java 用 `JsonFormat.printer().includingDefaultValueFields()` 控制零值输出。

⚠ **关键认知**：**Protobuf 的 JSON 映射和"随便序列化成 JSON"不是一回事**。字段名大小写、int64 转字符串、枚举用名字，这些规则如果和前端约定不一致，联调时会非常痛苦。对外暴露 JSON 时，明确说明用的是 `protojson` 标准映射。

## 2.3 不要把 JSON 塞进 bytes

反模式是 `message Event { bytes payload = 1; }`，里面其实塞的是 JSON。这样做等于放弃了 Protobuf 的所有好处：没有类型、没有代码生成、没有兼容性检查，只剩一个二进制外壳。**要么用结构化消息，要么直接用 JSON**。

# 3. Protobuf 与 gRPC 的关系

## 3.1 分工

```text
.proto
  ├── message  ──►  Protobuf：数据怎么表示、怎么序列化
  └── service  ──►  gRPC：方法怎么调用、怎么传输
```

Protobuf 负责消息结构、序列化和兼容性，不管网络、路由、超时；gRPC 负责传输、流控、拦截器和状态码，不管消息内部结构。

★ **关键认知**：Protobuf 可以**单独使用**——把消息序列化后写文件、进消息队列、存 Redis，都不需要 gRPC。反过来 gRPC 默认用 Protobuf，但协议上也可以换其他编解码器。两者是**正交**的，只是经常一起出现。

## 3.2 service 定义的最小实践

```protobuf
service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserReply);
  rpc ListUsers (ListUsersRequest) returns (ListUsersReply);
  rpc WatchUsers (WatchUsersRequest) returns (stream UserEvent);
}
```

| 约定 | 建议 |
|------|------|
| 每个方法独立 Request/Reply | 避免字段污染 |
| 方法名用动词开头 | `Get`、`List`、`Create`、`Watch` |
| 流式方法显式 `stream` | 客户端/服务端/双向 |
| 错误用状态码 | 不要把所有错误塞进 Reply |
| 分页用公共消息 | `common.v1.PageRequest` |

# 4. 线上调试

## 4.1 没有 schema 时的第一反应

```bash
# 把二进制转成可读结构，不需要 schema
protoc --decode_raw < message.bin
```

输出形如 `1: 150`、`2: "alice"`、`4 { 1: "NY" }`。它只能给出编号和猜测的类型，但足以判断"字段有没有传""大概是什么"。

## 4.2 有 schema 时精确解码

```bash
protoc --decode=demo.user.v1.User \
       -I proto \
       proto/user/v1/user.proto < message.bin
```

输出会精确到字段名和值，例如 `id: 150`、`name: "alice"`、`tags: "admin"`。

## 4.3 gRPC 调试

```bash
# 列出服务（需要服务端开启 reflection）
grpcurl -plaintext localhost:50051 list

# 查看方法
grpcurl -plaintext localhost:50051 list demo.user.v1.UserService

# 调用
grpcurl -plaintext -d '{"id": 42}' \
        localhost:50051 demo.user.v1.UserService/GetUser
```

常用工具就四类：`protoc --decode_raw` 无 schema 看结构，`protoc --decode` 精确解码，`grpcurl` 调 gRPC，`buf curl` 是 buf 生态的客户端。抓包用 Wireshark，需要配 proto 解析。

★ **关键认知**：排障时**先确认字节对不对，再怀疑业务逻辑**。用 `--decode_raw` 看字段编号和值，能快速区分"没发出来""发错字段""解析错类型"三类问题。

## 4.4 常见线上症状对照

| 症状 | 可能原因 | 排查 |
|------|---------|------|
| 字段读出 0 或空 | 对方没传、编号不一致 | `--decode_raw` 看编号 |
| 数字异常大 | 编号复用或类型混用 | 比对双方 proto |
| 内存暴涨 | 超大消息或元素洪水 | 入口大小限制 |
| 栈溢出 | 深递归 | 递归上限 |
| 枚举落到 default | 新值未处理 | 补 default 分支 |
| JSON 字段名对不上 | 大小写或映射规则 | 确认 `protojson` 规则 |

# 5. 性能实践

## 5.1 设计阶段

| 做法 | 收益 |
|------|------|
| 高频字段用 1~15 编号 | 每字段省 1 字节 tag |
| 负数用 `sint` | 从 10 字节降到 1~2 字节 |
| 小整数用 `int32` 而非 `int64` | varint 更短 |
| 大整数或哈希用 `fixed` | 避免 varint 额外字节 |
| 避免无意义的嵌套 | 每层多一个长度前缀 |

## 5.2 运行阶段

```go
buf := make([]byte, 0, 1024)          // 复用一个缓冲区
buf, err := msg.MarshalTo(buf[:0])    // 注意并发安全
```

Java 侧可以复用 `CodedOutputStream` 和 `ByteBuffer`，避免每次新建。

⚠ **关键认知**：**先测量再优化**。Protobuf 的序列化通常不是瓶颈，网络、锁、数据库往往才是。用 pprof 或 async-profiler 确认热点，不要凭感觉改 schema。

## 5.3 不要过度设计

- 不要为了省几字节把字段编号挤成一团，牺牲可读性。
- 不要过早引入 `Any` 或自定义编码。
- 不要把所有消息合并成一个"万能消息"。
- 不要在 hot path 上频繁 `Clone`。

# 6. 落地路线

## 6.1 从零开始的顺序

```text
1 目录与 package 规范
      │
      ▼
2 单一生成入口（buf 优先）
      │
      ▼
3 lint + breaking 进 CI
      │
      ▼
4 公共类型 → 5 业务 proto → 6 生成代码 → 7 监控与脱敏
```

## 6.2 团队约定清单

| 类别 | 约定 |
|------|------|
| 命名 | message UpperCamel，字段 snake_case，枚举值带前缀 |
| 编号 | 1~15 高频，删除必须 `reserved` 编号和名字 |
| 版本 | 破坏性改动用新 `vN`，老版本保留 |
| 生成 | 统一 buf 模板，版本写死，生成代码不手改 |
| 校验 | CI 跑 `buf lint` + `buf breaking` |
| 安全 | 入口限制大小与递归，敏感字段不落日志 |
| 文档 | 每个 service 和关键字段写注释 |
| 评审 | proto 变更单独提 PR，标注兼容性影响 |

## 6.3 一个可用的 buf 配置

```yaml
# buf.yaml
version: v2
modules:
  - path: proto
lint:
  use:
    - STANDARD
  except:
    - PACKAGE_VERSION_SUFFIX
breaking:
  use:
    - FILE
```

```bash
buf lint
buf breaking --against '.git#branch=main'
buf generate
```

★ **关键认知**：这三条命令就是 Protobuf 工程的"编译、测试、构建"。把它们放进 CI 和本地钩子，团队就不会出现"有人改了 proto 没通知"的情况。

# 7. 常见误区

| 误区 | 纠正 |
|------|------|
| "Protobuf 是加密的" | 只是二进制，字符串明文可见 |
| "Protobuf 一定比 JSON 小" | 长字符串场景差距很小 |
| "字段名可以随便改" | 二进制安全，JSON 和源码不兼容 |
| "删了字段就行" | 必须 `reserved` 编号和名字 |
| "未知字段会被丢弃" | proto3 默认保留，可能被夹带 |
| "生成代码可以手改" | 下次生成会被覆盖 |
| "protoc 版本无所谓" | 插件与运行时版本必须对齐 |
| "schema 只约束类型" | 业务范围仍要自己校验 |

# 8. 练习

1. 为一个电商系统设计 proto 目录结构，包含 `user`、`order`、`payment` 三个域，每个域一个 `v1`，再抽出一个 `common` 放分页。写出 `buf.yaml` 和至少一个 `buf.gen.yaml`。
2. 同一个 `User` 消息，分别用 `protojson` 和普通 JSON 库输出，对比 `int64`、枚举、零值字段的差异，并解释为什么会有这些差异。
3. 设计一套线上排障流程：拿到一段无法解析的 gRPC 请求体，从 `--decode_raw` 开始，到定位是"发送端字段错"还是"接收端 schema 旧"，写出每一步命令。

# 9. 小结

| 主题 | 要点 |
|------|------|
| schema 定位 | 对外契约，改动的成本在协同不在代码 |
| 目录 | 按业务域 + 版本组织，目录/package/生成路径一致 |
| 与 JSON | 长期共存，边缘网关做转换，注意 `protojson` 映射规则 |
| 与 gRPC | 正交：Protobuf 管数据，gRPC 管调用 |
| 调试 | `--decode_raw` 看结构，`--decode` 精确解，`grpcurl` 调服务 |
| 性能 | 编号、`sint`、`fixed`、避免深嵌套，先测量再优化 |
| 工程化 | buf 三条命令进 CI，版本写死，生成代码不手改 |
| 安全 | 大小、递归、未知字段、敏感数据、业务校验五道关 |

## 9.1 系列回顾

| 篇目 | 一句话 |
|------|--------|
| 概览与语法 | 先定义 schema，再生成代码，最后传二进制 |
| schema 与兼容性 | 编号是永久承诺，删除必须 `reserved` |
| 代码生成 | protoc 是框架，插件才是生成器，版本要锁 |
| 序列化与线格式 | tag + wire type + varint，理解字节才能判断兼容 |
| 安全与坑点 | 二进制不等于安全，限制、校验、脱敏缺一不可 |
| 最佳实践 | 契约化管理，buf 进 CI，调试先看字节 |

到这里 Protobuf 模块就完整了。下一步可以把它接到 `Go/gRPC` 系列，看服务定义、四种调用方式和拦截器如何与这里的 schema 设计配合；也可以回到 `File/Json`，对照两种格式在 API 选型上的取舍。
