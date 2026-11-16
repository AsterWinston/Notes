gRPC 经常被误解成"更快的 HTTP 接口"，但它实际上是一套以接口契约为核心的 RPC 框架。本篇先把 RPC 与 REST 的定位分清，再解释 gRPC 为什么离不开 HTTP/2，最后用一个完整的 proto 与 Maven 配置走通 Java 的代码生成流程。

# 1. RPC 与 REST：先分清问题

## 1.1 什么是 RPC

**RPC（Remote Procedure Call，远程过程调用）** 的目标，是让调用远程服务"看起来像调用本地方法"。调用方只关心方法名、参数和返回值，序列化、连接复用、超时传递这些脏活交给框架：

```java
// 本地调用：进程内直接执行
User user = userRepository.findById(42L);
// RPC 调用：语法接近，背后是一次网络请求
UserReply reply = userServiceBlockingStub.getUser(
        GetUserRequest.newBuilder().setId(42L).build());
```

RPC 的核心抽象是**接口契约**。客户端和服务端共享同一份 `.proto` 文件，编译器据此生成两边的桩代码（stub）。契约一旦确定，双方可以独立开发、独立发布，不必像 REST 那样靠文档口头约定字段名和类型。

**设计考量**：RPC 把"网络调用"包装成"方法调用"，好处是开发体验统一，代价是**网络的不确定性被隐藏了**。本地方法不会因为网络抖动失败，RPC 会。因此 RPC 框架必须把超时、重试、幂等、错误码这些分布式问题显式暴露出来，否则调用方很容易写出"把远程调用当本地调用"的代码。

## 1.2 RPC 与 REST 的对比

REST 不是协议，而是一种**以资源为中心的架构风格**；gRPC 是**以方法调用为中心的框架**。两者解决的问题有重叠，但侧重点不同：

| 维度 | REST over HTTP/JSON | gRPC |
|------|---------------------|------|
| 抽象模型 | 资源 + 动词，如 `GET /users/42` | 服务 + 方法，如 `GetUser` |
| 传输格式 | JSON 文本 | Protobuf 二进制 |
| 传输协议 | HTTP/1.1 或 HTTP/2 | HTTP/2（必须） |
| 接口契约 | OpenAPI/Swagger，常常后补 | `.proto` 强约束，契约先行 |
| 浏览器支持 | 原生支持 | 需要 grpc-web 或网关转码 |
| 流式能力 | 弱，SSE/WebSocket 另说 | 原生支持四种流 |
| 可读性 | 抓包直接看 | 需要工具解码 |
| 典型场景 | 对外开放 API、前后端交互 | 内部服务、低延迟、强类型 |

分界线通常在于**调用方是不是浏览器**。面向公网、需要被任意客户端调用的接口，REST/JSON 的通用性无可替代；服务之间的东西向流量，gRPC 的类型安全、性能和流式能力更划算。生产系统里常见的组合是"边缘 REST，内部 gRPC"。

> **行业实践**：很多团队并不是二选一，而是用 gRPC 做服务间调用，再用 grpc-gateway 或 Envoy 转码对外暴露 REST。这样内部享受强类型和性能，外部仍然保持 HTTP/JSON 的兼容性。

## 1.3 gRPC 的组成

```text
┌──────────────────────────────────────────────┐
│                 应用代码                      │
├──────────────────────────────────────────────┤
│  生成代码（stub）：Client / Server 接口        │
├──────────────────────────────────────────────┤
│  gRPC 运行时：编解码、拦截器、负载均衡、重试     │
├──────────────────────────────────────────────┤
│  HTTP/2：多路复用、头部压缩、流控、Trailers     │
├──────────────────────────────────────────────┤
│  TCP / TLS                                    │
└──────────────────────────────────────────────┘
```

一次 gRPC 调用在 HTTP/2 上表现为：

```text
POST /hello.v1.Greeter/SayHello HTTP/2
:method: POST
:scheme: http
:path: /hello.v1.Greeter/SayHello
content-type: application/grpc
te: trailers
grpc-timeout: 2S
<length-prefixed protobuf body>
```


因此排查 gRPC 问题时，看到 HTTP 200 不代表调用成功，必须看 `grpc-status`。这是与 REST 最直观的差异之一。

# 2. HTTP/2 为什么是 gRPC 的地基

## 2.1 二进制分帧与多路复用

HTTP/1.1 的报文是文本，一条连接同一时刻只能处理一个请求。HTTP/2 把报文拆成**帧（frame）**，每个帧带有 `stream id`，同一条 TCP 连接上可以交错发送多个流：

```text
一条 TCP 连接
┌─────────────────────────────────────────────┐
│ stream 1: HEADERS ─ DATA ─ DATA ─ END       │
│ stream 3:   HEADERS ─ DATA ─ END            │
│ stream 5: HEADERS ─ DATA ─ DATA ─ DATA ─ END│
└─────────────────────────────────────────────┘
     帧按 stream id 归属，接收端重组
```

gRPC 的四种调用方式能成立，靠的就是这个"流"的概念。一个 gRPC 调用占一个 HTTP/2 stream，客户端流、服务端流、双向流都只是在同一个 stream 上多次收发 DATA 帧，而不是建立多条连接。

## 2.2 gRPC 依赖的 HTTP/2 特性

| 特性 | gRPC 的用途 |
|------|-------------|
| 多路复用 | 单连接并发大量 RPC，避免连接数爆炸 |
| 头部压缩（HPACK） | 重复的 `content-type`、`:path` 只传一次 |
| 流控 | 防止发送方压垮接收方 |
| Trailers | 在响应体之后携带 `grpc-status`、`grpc-message` |
| 取消（RST_STREAM） | 客户端取消时精确终止单个调用，不影响同连接其他流 |

**设计考量**：gRPC 选择 HTTP/2 而不是自定义协议，是为了复用成熟的传输层能力，并让代理、负载均衡器有统一的接入方式。代价是链路上的每一跳都必须懂 HTTP/2；任何只支持 HTTP/1.1 的中间件都会让 gRPC 直接失败。

# 3. Protocol Buffers 语法

## 3.1 第一个 proto 文件

新建 `src/main/proto/hello.proto`：

```protobuf
syntax = "proto3";                       // 必须声明版本，否则默认按 proto2 解析
package hello.v1;                        // 包名参与生成代码和全限定方法名
option java_package = "com.example.hello.v1";
option java_multiple_files = true;       // 每个 message 生成独立 Java 文件
option java_outer_classname = "HelloProto";
message HelloRequest {
  string name = 1;                       // 字段编号 1，上线后不可随意修改
}
message HelloReply {
  string message = 1;
}
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}
```

三个 Java 选项的作用：

| 选项 | 作用 | 建议 |
|------|------|------|
| `java_package` | 生成代码的 Java 包名 | 显式指定，避免落到 proto 包名 |
| `java_multiple_files` | 每个 message 单独一个文件 | 新项目开启，便于阅读 |
| `java_outer_classname` | 外层类名，存放描述符 | 显式指定，避免文件名冲突 |

## 3.2 字段编号与类型

字段编号（field number）是线上传输的真正标识，规则如下：

- 范围是 `1 ~ 536870911`，即 2^29 - 1。
- `19000 ~ 19999` 被 Protobuf 内部保留，不能使用。
- `1 ~ 15` 编码后只占 1 字节 tag，适合高频字段。
- 编号一旦发布就**不能复用**。删除字段时应把编号 `reserved` 掉。

常用标量类型与 Java 的映射：

| proto 类型 | Java 类型 | 说明 |
|------------|-----------|------|
| `double` / `float` | `double` / `float` | 浮点数 |
| `int32` / `int64` | `int` / `long` | 变长编码，负数效率低 |
| `uint32` / `uint64` | `int` / `long` | 无符号，Java 用有符号类型承载 |
| `sint32` / `sint64` | `int` / `long` | ZigZag 编码，适合负数 |
| `fixed32` / `fixed64` | `int` / `long` | 定长编码，适合大整数 |
| `bool` | `boolean` | 布尔值 |
| `string` | `String` | 必须是 UTF-8 |
| `bytes` | `ByteString` | 任意字节，避免 `byte[]` 拷贝 |

**性能细节**：`int32` 存负数时会按 64 位补码编码，占用 10 字节。如果字段经常为负，应使用 `sint32` 或 `sint64`，这是 Protobuf 编码里最容易忽略的坑。

## 3.3 proto3 的存在性与复合类型

proto3 的普通标量字段**无法区分"没传"和"传了零值"**。`int32 age = 1` 收到 0 时，你不知道对方是没填，还是确实填了 0。需要区分时，加 `optional`：

```protobuf
message Profile {
  optional int32 age = 1;      // 生成 hasAge()，可以判断是否显式设置
  repeated string tags = 2;    // 生成 ProtocolStringList
  map<string, string> attrs = 3;
  oneof contact {
    string email = 4;
    string phone = 5;
  }
  Status status = 6;
}
enum Status {
  STATUS_UNSPECIFIED = 0;      // proto3 枚举第一个值必须为 0
  STATUS_ACTIVE = 1;
  STATUS_DISABLED = 2;
}
```

几个容易混淆的点：

- `repeated` 默认生成 `List<T>`，不是数组；标量 repeated 在 proto3 中默认使用 packed 编码。
- `map` 在 Java 中生成 `Map<K, V>`，但**不保证顺序**，不要依赖遍历顺序。
- `oneof` 同一时刻只有一个字段有值，生成 `getContactCase()` 用于判断分支。
- 枚举第一个值必须为 0，且建议命名为 `XXX_UNSPECIFIED`，表示"未设置"。

**设计考量**：proto3 去掉 `required` 是刻意的。跨团队、跨版本的契约里，强制字段会让演进变得脆弱；更稳妥的做法是用业务校验或 `google.protobuf` 的 wrapper 类型表达"可缺失"。

## 3.4 兼容性规则

Protobuf 的兼容性建立在"编号不变、语义可扩展"之上：

| 改动 | 是否兼容 | 说明 |
|------|----------|------|
| 新增字段 | 兼容 | 旧客户端忽略未知字段 |
| 删除字段 | 兼容 | 必须 `reserved` 编号和字段名 |
| 修改字段名 | 兼容 | 线上只认编号 |
| 修改字段编号 | 不兼容 | 等于换了一个字段 |
| 修改字段类型 | 多数不兼容 | 如 `int32` 改 `string` |
| 删除枚举值 | 不兼容 | 旧代码可能仍在使用 |


> **行业实践**：把 `.proto` 当作公开 API 管理，使用 `buf breaking` 在 CI 中检测不兼容改动。比事后靠人工 review 可靠得多。

# 4. Maven 插件与代码生成

## 4.1 项目结构

proto 放在 src/main/proto，生成代码输出到 	arget/generated-sources/protobuf，业务代码不要手改生成文件。

## 4.2 pom.xml 配置

```xml
<properties>
    <java.version>17</java.version>
    <grpc.version>1.68.1</grpc.version>
    <protobuf.version>3.25.5</protobuf.version>
</properties>
<dependencies>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty-shaded</artifactId>
        <version>${grpc.version}</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>${grpc.version}</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>${grpc.version}</version>
    </dependency>
    <dependency>
        <groupId>javax.annotation</groupId>
        <artifactId>javax.annotation-api</artifactId>
        <version>1.3.2</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.1</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>
                    com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}
                </protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>
                    io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}
                </pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

几个关键点：

- `os-maven-plugin` 负责识别操作系统和架构，生成 `os.detected.classifier`，用于下载对应平台的 protoc 可执行文件。
- `compile` 生成消息类，`compile-custom` 调用 `protoc-gen-grpc-java` 生成服务桩。
- `javax.annotation-api` 是为了满足生成代码里的 `@javax.annotation.Generated`，JDK 11 之后已不再内置。
- Spring Boot 3.x 也可以使用 `io.github.ascopes:protobuf-maven-plugin`，但在存量项目里 `org.xolstice` 这套配置仍然最常见，两者不要混用。

执行构建：

```bash
mvn clean compile
```

生成的关键文件：

```text
target/generated-sources/protobuf/java/com/example/hello/HelloRequest.java
target/generated-sources/protobuf/java/com/example/hello/HelloReply.java
target/generated-sources/protobuf/grpc-java/com/example/hello/GreeterGrpc.java
```

## 4.3 生成代码的结构

`GreeterGrpc` 是最核心的类，包含：

| 成员 | 用途 |
|------|------|
| `GreeterGrpc.GreeterImplBase` | 服务端继承的基类 |
| `GreeterGrpc.newBlockingStub(channel)` | 阻塞式客户端 |
| `GreeterGrpc.newStub(channel)` | 异步/流式客户端 |
| `GreeterGrpc.newFutureStub(channel)` | 返回 `ListenableFuture` 的客户端 |
| `GreeterGrpc.getServiceDescriptor()` | 服务元数据，注册和反射使用 |

服务端通常这样继承：


注意即使是一元方法，生成的 Java 方法签名也是 `void + StreamObserver`，而不是直接返回对象。这是 gRPC Java 的统一模型：所有调用都通过 `StreamObserver` 回调完成，阻塞式 stub 只是在外面包了一层同步等待。


# 5. 常见错误与排错

| 现象 | 常见原因 | 解决方式 |
|------|----------|----------|
| `protoc-gen-grpc-java: program not found` | 插件未配置或 classifier 错误 | 检查 `os.detected.classifier` 与插件坐标 |
| 生成代码找不到 `javax.annotation.Generated` | 缺少 annotation 依赖 | 引入 `javax.annotation-api` 并设为 provided |
| `UNIMPLEMENTED: Method not found` | 服务端没注册或方法名不匹配 | 检查 `addService` 与 proto 包名 |
| `UNAVAILABLE: io exception` | 端口未监听、TLS 配置不一致 | 先确认端口，再确认明文/TLS |
| `RESOURCE_EXHAUSTED: gRPC message exceeds maximum size` | 消息超过 4 MiB 默认上限 | 调整 `maxInboundMessageSize` 或改流式 |
| 客户端收到 200 却报错 | 只看 HTTP 状态码，忽略 `grpc-status` | 用 `Status.fromThrowable` 取真实状态 |

**排错顺序**：先确认端口和 TLS，再确认 proto 包名与方法全限定名，最后才怀疑业务代码。gRPC 的 `UNIMPLEMENTED` 多数不是网络问题，而是服务没注册或路径对不上。

# 6. 练习与自检

1. 在 `hello.proto` 中新增 `int32 age = 2`，重新生成代码，观察 `getAge()`、`hasAge()` 的差异，并解释 proto3 的零值语义。
2. 把 `SayHello` 的返回类型改成 `stream HelloReply`，只改 proto 并重新生成，观察 `GreeterImplBase` 的方法签名发生了什么变化。
3. 故意把客户端连接端口改成 `50052`，用 `Status.fromThrowable` 打印状态码，解释为什么是 `UNAVAILABLE` 而不是 `NOT_FOUND`。

# 7. 小结

| 主题 | 要点 |
|------|------|
| RPC 定位 | 以方法调用为中心，共享 `.proto` 契约，适合内部服务 |
| HTTP/2 | 多路复用、HPACK、Trailers 是四种流的基础 |
| Protobuf | 编号是线上标识，字段名只影响代码；负数用 `sint` |
| 编译链 | `protoc` 调 `protoc-gen-grpc-java`，生成消息类与服务桩 |
| 兼容性 | 新增字段安全，改编号危险，删除字段必须 `reserved` |
| 排错 | HTTP 200 不代表成功，必须看 `grpc-status` |

下一篇进入 Java 工程实践，从一份 proto 出发，把服务端实现、三种客户端 stub、连接管理和优雅停机串成一个能跑通的最小闭环。
