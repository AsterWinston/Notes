`.proto` 本身不能运行，它只是一份契约。真正干活的是 `protoc` 和你装的插件：它们把 schema 翻译成 Java、Go、Python 的强类型代码。很多人卡在"生成不出来"或"生成出来对不上"，问题几乎都出在插件路径、import 路径和版本不一致这三处。

# 1. protoc 的工作原理

## 1.1 一条流水线

```text
user.proto
    │
    ▼
┌──────────────┐   解析、校验、构建
│   protoc     │   ───────────────►  FileDescriptorProto
│  前端解析器   │                      （描述符，语言无关）
└──────┬───────┘
       │ 通过 CodeGeneratorRequest 交给插件
       ▼
┌──────────────┐
│ 插件（protoc-gen-xxx）│  按语言生成代码
└──────┬───────┘
       ▼
  xxx.pb.go / Xxx.java / xxx_pb2.py
```

关键点：

- `protoc` 负责**解析和校验**，它自己只知道 C++、Java、Python 等内置语言。
- 其他语言（Go、Rust、Dart）通过**插件**扩展。
- 插件本质是一个可执行文件，从 stdin 读 `CodeGeneratorRequest`，往 stdout 写 `CodeGeneratorResponse`。

★ **关键认知**：`protoc` 不是编译器，它更像**代码生成框架**。`protoc-gen-go` 这样的插件、以及内置的 Java/Python 生成器，才是真正写文件的部分。所以"protoc 版本"和"插件版本"是两套独立的版本号，都要管。

## 1.2 描述符：一切生成物的源头

```bash
protoc --descriptor_set_out=user.desc user.proto   # 导出二进制描述符
protoc --decode_raw < message.bin                  # 无 schema 查看结构
```

描述符是语言无关的元数据树，记录每个 message、字段、编号、类型。代码生成、gRPC 反射、动态消息、`buf` 检测都建立在它上面。

⚠ **关键认知**：调试"生成代码为什么不对"时先看描述符，`buf build -o -` 可以把它导出来，比翻生成代码快。

# 2. 安装与最小示例

## 2.1 安装 protoc

```bash
brew install protobuf                      # macOS
winget install --id Google.Protobuf        # Windows
protoc --version                           # 验证
```

## 2.2 一个最小的 Go 生成

```bash
# 1. 安装 Go 插件
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# 2. 确保 $GOPATH/bin 在 PATH 里
export PATH="$PATH:$(go env GOPATH)/bin"

# 3. 生成
protoc --go_out=gen --go_opt=paths=source_relative \
       --go-grpc_out=gen --go-grpc_opt=paths=source_relative \
       proto/user/v1/user.proto
```

生成结果：

```text
gen/
└── proto/user/v1/
    ├── user.pb.go        # 消息、序列化、Getter
    └── user_grpc.pb.go   # 服务端接口、客户端 Stub
```

⚠ **关键认知**：`protoc-gen-go` 必须能被 `PATH` 找到，且文件名必须严格是 `protoc-gen-go`，`protoc` 才能通过 `--go_out` 找到它。报 `protoc-gen-go: program not found` 时，99% 是 `$GOPATH/bin` 没进 `PATH`。

# 3. 各语言插件

## 3.1 Go

| 插件 | 作用 | 安装 |
|------|------|------|
| `protoc-gen-go` | 消息、枚举、序列化 | `go install google.golang.org/protobuf/cmd/protoc-gen-go@latest` |
| `protoc-gen-go-grpc` | service 与客户端 | `go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest` |

```protobuf
option go_package = "example.com/demo/gen/user/v1;userv1";
```

- 分号前是导入路径，分号后是包名。
- 不写 `go_package` 时，新版 `protoc-gen-go` 会直接报错。

## 3.2 Java

Java 的生成器**内置在 protoc 里**，不需要额外插件：

```bash
protoc --java_out=src/main/java proto/user/v1/user.proto
```

常用 option：

```protobuf
option java_package = "com.example.demo.user.v1";
option java_outer_classname = "UserProto";
option java_multiple_files = true;   // 每个 message 一个文件
```

★ **关键认知**：`java_multiple_files = true` 几乎是现代 Java 项目的默认选择。不开的话所有 message 都挤在一个 `UserProto.User` 内部类里，import 写起来很别扭。

## 3.3 Python

```bash
protoc --python_out=gen proto/user/v1/user.proto
```

Python 不需要插件，但要装运行时库 `pip install protobuf`。生成的 `user_pb2.py` 是描述符的序列化结果，运行时动态构建类，所以不能像普通 Python 源码那样直接读。

其他语言各有插件：Rust 用 `protoc-gen-prost`，Dart 用 `protoc-gen-dart`，C# 内置 `--csharp_out`。

⚠ **关键认知**：不同插件的参数风格不一样。Go 用 `--go_opt`，Java 用纯 `--java_out`，`buf` 则在 `buf.gen.yaml` 里统一描述。**不要把 `--go_opt` 套到 Java 上**，`protoc` 会报未知参数。

# 4. 生成代码的结构

## 4.1 Go 生成物

```go
// user.pb.go（节选，示意）
type User struct {
    state         protoimpl.MessageState
    Id            int64  `protobuf:"varint,1,opt,name=id,proto3"`
    Name          string `protobuf:"bytes,2,opt,name=name,proto3"`
    Bio           *string `protobuf:"bytes,3,opt,name=bio,proto3,oneof"`
    unknownFields protoimpl.UnknownFields   // 未知字段保留区
}

func (x *User) GetId() int64 {
    if x != nil { return x.Id }
    return 0
}
```

要点：

- 结构体字段导出，但**不要直接改** `state` 和 `unknownFields`。
- 每个字段都有 `GetXxx()`，nil 安全，推荐在业务代码里用 Getter。
- `optional` 字段生成指针，配合 `HasXxx()` 判断存在性。
- 结构体带 protobuf tag，序列化时按 tag 里的编号和 wire type 工作。

## 4.2 Java 生成物

Java 生成的是**不可变消息 + Builder**，字段私有，只能通过 Builder 组装：

```java
User user = User.newBuilder()
    .setId(42L)
    .setName("alice")
    .build();
```

★ **关键认知**：Java 消息一旦 `build()` 就不可变，修改必须 `toBuilder()`。这不是设计缺陷，而是为了保证线程安全和哈希稳定。**别在循环里反复 build**，先把数据攒进 Builder，最后 build 一次。

## 4.3 生成代码该不该进版本库

| 做法 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| 提交生成代码 | 构建简单，无需装 protoc | 仓库膨胀，易过期 | 小团队、发布包 |
| 构建时生成 | 永远和 proto 一致 | 构建依赖工具链 | 中大型项目 |
| 发布到制品库 | 多语言共享 | 需要发版流程 | 平台化团队 |

⚠ **关键认知**：无论选哪种，**生成代码都不要手改**。手改会在下次生成时被覆盖，而且不会有人记得为什么改。要改行为就改 proto 或写包装层。

# 5. 构建集成

## 5.1 buf：推荐的统一入口

```yaml
# buf.gen.yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go:v1.36.0
    out: gen
    opt:
      - paths=source_relative
  - remote: buf.build/grpc/go:v1.5.1
    out: gen
    opt:
      - paths=source_relative
  - remote: buf.build/protocolbuffers/java:v3.25.3
    out: gen-java
```

```bash
buf generate                    # 按 buf.gen.yaml 生成
buf generate --template buf.gen.ci.yaml
```

buf 的好处：插件版本写死、无需本地装 protoc、和 lint/breaking 统一管理。**新项目优先用 buf，不要从裸 protoc 起步**。

## 5.2 Maven

```xml
<plugin>
  <groupId>org.xolstice.maven.plugins</groupId>
  <artifactId>protobuf-maven-plugin</artifactId>
  <version>0.6.1</version>
  <configuration>
    <protocArtifact>
      com.google.protobuf:protoc:3.25.3:exe:${os.detected.classifier}
    </protocArtifact>
  </configuration>
  <executions>
    <execution>
      <goals><goal>compile</goal><goal>test-compile</goal></goals>
    </execution>
  </executions>
</plugin>
```

★ **关键认知**：构建集成最重要的不是"能跑"，而是**可复现**。protoc 版本、插件版本、生成参数必须写进配置文件并提交，让任何人 clone 下来执行一条命令得到完全相同的代码。

# 6. 版本管理

## 6.1 三套版本要一起管

| 组件 | 示例 | 说明 |
|------|------|------|
| protoc | `3.25.3` | 解析器版本 |
| 语言插件 | `protoc-gen-go v1.36.0` | 生成器版本 |
| 运行时库 | `protobuf-java 3.25.3` | 反序列化实现版本 |

```text
插件 v1.36  ──生成──►  代码调用新 API
                              │
                              ▼
                   项目把运行时降到 v1.20
                              │
                              ▼
                    编译通过，运行报 NoSuchMethodError
```

⚠ **关键认知**：运行时库版本**不能低于**生成代码所要求的版本。常见事故是 `pom.xml` 里传递依赖把 `protobuf-java` 降到了旧版，运行时立刻炸。用 `mvn dependency:tree` 或 `go mod graph` 锁定版本。

## 6.2 锁定与升级

```bash
# Go：锁 protoc 插件版本（tools.go 或 go.mod）
go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.36.0

# buf：远程插件带明确版本号
# remote: buf.build/protocolbuffers/go:v1.36.0

# Java：protoc artifact 固定版本
# com.google.protobuf:protoc:3.25.3:exe
```

升级策略：

- 小版本可以跟，但要在 CI 里跑一轮 `buf breaking` 和单测。
- 大版本（如 Java 3.x → 4.x）改动大，单独排期。
- 团队内所有人生成代码必须用同一套版本，否则会出现"我这边能编译，你那边不行"。

# 7. 常见问题

| 现象 | 原因 | 解决 |
|------|------|------|
| `protoc-gen-go: program not found` | `$GOPATH/bin` 不在 PATH | 加 PATH，或写绝对路径 `--plugin=` |
| `option go_package is required` | 新版插件强制要求 | 在 proto 里补 `go_package` |
| `No such file or directory: user.proto` | 没有 `-I` 指定导入根 | 加 `-I proto`，路径写相对根 |
| import 找不到 `google/protobuf/*.proto` | 没装 well-known types | 用 buf，或指定 protoc 自带 include |
| Java 生成到奇怪包名 | 缺 `java_package` | 显式声明 |
| 生成代码编译报 `Unresolved reference` | 生成到错误目录 | 检查 `--*_out` 与导入路径 |
| 运行时报 `NoSuchMethodError` | 运行时库版本过低 | 升级并统一依赖 |

★ **关键认知**：`-I`（或 `--proto_path`）是所有 import 的根。`protoc -I proto proto/user/v1/user.proto` 和 `protoc proto/user/v1/user.proto` 生成的 import 语句可能不一样。**路径写错不会报错，但生成代码的包路径会歪**，这是最隐蔽的一类问题。

# 8. 练习

1. 写一个 `common/v1/page.proto`，定义 `PageRequest` 和 `PageReply`，再写 `user/v1/user.proto` 用 `import` 引用它。用 `-I proto` 生成 Go 代码，观察两个文件的 import 路径。
2. 把同一个 proto 分别生成 Java 和 Go，对比：Java 的 Builder 模式、Go 的结构体 + Getter，各自如何表达 `optional` 字段。
3. 故意把 `--go_out` 指向错误目录，观察生成结果，说明 `paths=source_relative` 和默认模式的区别。

# 9. 小结

| 主题 | 要点 |
|------|------|
| protoc | 解析与校验，插件负责生成 |
| 插件 | 本质是可执行文件，通过 stdin/stdout 通信 |
| Go | `protoc-gen-go` + `protoc-gen-go-grpc`，必须配 `go_package` |
| Java | 内置生成器，不可变消息 + Builder |
| Python | 内置生成器，`_pb2.py` 是描述符包装 |
| 构建 | 优先 buf，版本写死，保证可复现 |
| 版本 | protoc、插件、运行时三套一起管 |
| 生成代码 | 不要手改，要么改 proto，要么写包装层 |

下一篇进入序列化和线格式，拆开 varint、tag、长度前缀，手算一遍字节，搞清楚为什么 Protobuf 比 JSON 小、在哪些场景其实并不小。
