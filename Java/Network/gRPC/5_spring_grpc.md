在纯 Java 里，gRPC 服务的启动、依赖注入、配置、优雅停机都要自己写。Spring gRPC 把这些工作交给容器：服务实现就是一个 Bean，拦截器也是 Bean，端口和 TLS 通过配置文件管理。本篇以 Spring Boot 3.x 与 Spring gRPC 1.0.x 为准，同时说明存量项目常见的 net.devh starter。

# 1. 为什么需要 Spring 集成

裸 gRPC 的启动代码通常是这样的：

```java
Server server = ServerBuilder.forPort(50051)
        .addService(new UserServiceImpl(new UserRepository(...)))
        .build()
        .start();
```

问题是：

- 依赖要手动 new，无法使用 Spring 的事务、缓存、配置。
- 拦截器、线程池、TLS 都写在 `main` 方法里，环境切换困难。
- 服务启动顺序、优雅停机、健康检查需要自己接。
- 测试时难以替换依赖。

Spring gRPC 的目标是让 gRPC 服务像 `@RestController` 一样自然：加注解、注入依赖、写配置，剩下的交给自动配置。

# 2. 两种技术路线

Java 生态里有两套常见的 Spring gRPC 集成，名字很像但配置完全不同：

| 维度 | 官方 Spring gRPC | net.devh grpc-spring-boot-starter |
|------|------------------|-----------------------------------|
| 坐标 | `org.springframework.grpc:spring-grpc-spring-boot-starter` | `net.devh:grpc-server-spring-boot-starter` |
| 维护方 | Spring 官方 | 社区 |
| 配置前缀 | `spring.grpc.*` | `grpc.*` |
| 服务注解 | `@GrpcService` 或普通 `@Service` | `@GrpcService` |
| 客户端注解 | `@ImportGrpcClients`、`GrpcChannelFactory` | `@GrpcClient` |
| 适用 | 新项目、Spring Boot 3.x | 存量项目、需要成熟社区方案 |

**重要**：两套 starter 不要同时引入。它们的自动配置、注解全限定名、配置前缀都不同，混用会出现服务注册不上、端口被抢、客户端注入失败等难以定位的问题。新项目优先选官方 Spring gRPC。

# 3. 官方 Spring gRPC 最小示例

## 3.1 依赖

```xml
<properties>
    <java.version>17</java.version>
    <spring-grpc.version>1.0.1</spring-grpc.version>
    <grpc.version>1.68.1</grpc.version>
    <protobuf.version>3.25.5</protobuf.version>
</properties>
<dependencies>
    <dependency>
        <groupId>org.springframework.grpc</groupId>
        <artifactId>spring-grpc-spring-boot-starter</artifactId>
        <version>${spring-grpc.version}</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-services</artifactId>
        <version>${grpc.version}</version>
    </dependency>
</dependencies>
```

`grpc-services` 提供健康检查与反射服务，便于用 `grpcurl` 调试。proto 代码生成仍使用上一篇介绍的 `protobuf-maven-plugin`，Spring 不会替你做代码生成。

## 3.2 配置

```yaml
spring:
  application:
    name: user-server
  grpc:
    server:
      port: 9090
```

默认端口是 9090。如果同时提供 HTTP 接口，Web 端口仍是 `server.port`（例如 8080），两个端口互不影响。

## 3.3 服务实现

```java
package com.example.user;
import com.example.user.v1.GetUserRequest;
import com.example.user.v1.UserReply;
import com.example.user.v1.UserServiceGrpc;
import io.grpc.Status;
import io.grpc.stub.StreamObserver;
import org.springframework.grpc.server.service.GrpcService;
@GrpcService
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {
    private final UserRepository userRepository;
    // 构造器注入，和普通 Spring Bean 完全一致
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    @Override
    public void getUser(GetUserRequest request, StreamObserver<UserReply> responseObserver) {
        repository.findById(request.getId())
                .ifPresentOrElse(user -> {
                    responseObserver.onNext(UserReply.newBuilder()
                            .setId(user.id())
                            .setName(user.name())
                            .setEmail(user.email())
                            .build());
                    responseObserver.onCompleted();
                }, () -> responseObserver.onError(Status.NOT_FOUND
                        .withDescription("用户不存在: " + request.getId())
                        .asRuntimeException()));
    }
}
```

`@GrpcService` 同时具备组件扫描语义。如果你不加它，只要把服务实现声明为普通的 `@Service` Bean 也可以，因为 Spring gRPC 会把容器里所有 `BindableService` 绑定到服务器。`@GrpcService` 的价值在于可以按服务配置拦截器。

## 3.4 客户端配置

官方 Spring gRPC 用 `GrpcChannelFactory` 创建 channel，也可以依赖默认的 stub 扫描：

```yaml
spring:
  grpc:
    client:
      channel:
        user:
          target: static://localhost:9090
```

```java
import com.example.user.v1.UserServiceGrpc;
import org.springframework.grpc.client.GrpcChannelFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
@Configuration
public class GrpcClientConfig {
    @Bean
    UserServiceGrpc.UserServiceBlockingStub userStub(GrpcChannelFactory channels) {
        // "user" 对应 spring.grpc.client.channel.user 这组配置
        return UserServiceGrpc.newBlockingStub(channels.createChannel("user"));
    }
}
```

使用：

```java
@Service
public class UserFacade {
    private final UserServiceGrpc.UserServiceBlockingStub stub;
    public UserFacade(UserServiceGrpc.UserServiceBlockingStub stub) {
        this.stub = stub;
    }
    public UserReply getUser(long id) {
        return stub.withDeadlineAfter(2, java.util.concurrent.TimeUnit.SECONDS)
                .getUser(GetUserRequest.newBuilder().setId(id).build());
    }
}
```

Spring Boot 默认等价于在启动类上加了 `@ImportGrpcClients`，它会扫描生成的 stub 类型并注册 Bean。需要自定义扫描包或绑定关系时，再显式加注解：

```java
@ImportGrpcClients(basePackages = "com.example.user.v1")
@SpringBootApplication
public class UserClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserClientApplication.class, args);
    }
}
```

**注意**：`@GrpcClient` 是 net.devh starter 的注解，官方 Spring gRPC 没有这个注解。看到教程里用 `@GrpcClient("user")`，先确认项目用的是哪套 starter。

# 4. Bean 集成

## 4.1 服务实现作为 Bean

Spring gRPC 的核心规则只有一条：**容器里的每个 `BindableService` Bean 都会被绑定到 gRPC 服务器**。生成的 `XxxImplBase` 实现了 `BindableService`，所以继承它的类天然满足条件。

这意味着：

- 可以在服务实现里注入 Repository、`RestClient`、事务管理器。
- 可以用 `@Transactional`、`@Cacheable` 等 Spring 注解。
- 可以用 `@Profile` 控制不同环境是否注册。
- 测试时可以用 `@MockBean` 替换依赖。

## 4.2 全局拦截器

```java
package com.example.user;
import io.grpc.Metadata;
import io.grpc.ServerCall;
import io.grpc.ServerCallHandler;
import io.grpc.ServerInterceptor;
import org.springframework.core.annotation.Order;
import org.springframework.grpc.server.GlobalServerInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
@Configuration
public class GrpcInterceptorConfig {
    @Bean
    @Order(100)
    @GlobalServerInterceptor
    ServerInterceptor loggingInterceptor() {
        return new LoggingServerInterceptor();
    }
    @Bean
    @Order(200)
    @GlobalServerInterceptor
    ServerInterceptor authInterceptor() {
        return new AuthServerInterceptor(tokenVerifier());
    }
    @Bean
    TokenVerifier tokenVerifier() {
        return token -> "valid-user"; // 示例：真实项目应校验 JWT 签名
    }
}
```

`@Order` 越小越靠外。日志放在外层记录总耗时，鉴权放在内层靠近业务，与上一篇的顺序原则一致。Spring gRPC 会按 `@Order` 排序后统一应用，不需要操心 `intercept` 与 `interceptForward` 的倒序陷阱。

## 4.3 按服务配置拦截器

只给某个服务加拦截器：

```java
@GrpcService(interceptors = AuditServerInterceptor.class)
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {
    // 只有这个服务会经过 AuditServerInterceptor
}
```

`@GrpcService` 的 `interceptors` 接收拦截器类型，Spring 会从容器里取对应 Bean。全局拦截器与按服务拦截器同时存在时，默认全局先执行，再执行服务级。

# 5. 配置项

官方 Spring gRPC 的常用配置：

| 配置 | 作用 |
|------|------|
| `spring.grpc.server.port` | gRPC 监听端口，默认 9090 |
| `spring.grpc.server.address` | 绑定地址，默认所有网卡 |
| `spring.grpc.server.inbound.message.max-size` | 入站消息上限 |
| `spring.grpc.server.keepalive.time` | 服务端 keepalive 间隔 |
| `spring.grpc.server.keepalive.timeout` | keepalive 探测超时 |
| `spring.grpc.server.shutdown.grace-period` | 关闭时等待在途请求的时长 |
| `spring.grpc.client.channel.<name>.target` | 客户端目标地址 |
| `spring.grpc.client.channel.<name>.ssl.bundle` | 客户端 TLS 证书 bundle |
| `spring.grpc.server.ssl.bundle` | 服务端 TLS 证书 bundle |

客户端的 `target` 支持多种 scheme：

- `static://localhost:9090`：固定地址。
- `dns:///user-service:9090`：通过 DNS 解析。
- `in-process:test`：进程内测试，不占端口。

**注意**：`static://` 是地址解析方式，不是"使用明文"的开关。是否加密由 `ssl.bundle` 等 TLS 配置决定，不要把它和 `usePlaintext()` 混为一谈。

# 6. 与 Spring Boot 生命周期

## 6.1 启动顺序

```text
Spring 容器启动
    │
    ├── 扫描 @GrpcService / BindableService
    ├── 创建服务 Bean，完成依赖注入
    ├── 创建全局/服务级拦截器 Bean
    ├── 配置线程池、TLS、消息大小
    ▼
GrpcServerLifecycle 启动 Server
    │
    ├── 绑定端口
    ├── 开始接收请求
    └── 注册健康检查与反射服务
    ▼
应用就绪
```

关键点：**服务 Bean 初始化完成之后，gRPC Server 才会启动**。因此不要在服务构造器里发起依赖 gRPC 服务自身的调用，否则会卡在启动阶段。

## 6.2 优雅停机

Spring 关闭时，`GrpcServerLifecycle` 会调用 `Server.shutdown()`，等待在途请求结束，再销毁 Bean。需要控制等待时长时，配置 `spring.grpc.server.shutdown.grace-period`。

如果自定义了线程池，要确保：

- 线程池不是守护线程，否则 JVM 可能在请求完成前退出。
- 关闭顺序是"先停 server，再停线程池"。
- 不要在 `@PreDestroy` 里调用阻塞的 gRPC 方法。

> **行业实践**：Kubernetes 下把 `terminationGracePeriodSeconds` 设置得比 gRPC 关闭等待时间略长，并配合 readiness probe 先从 Service 摘流，避免发布期间出现 `UNAVAILABLE`。

## 6.3 与 Web 共存

三种部署方式：

| 方式 | 端口 | 说明 |
|------|------|------|
| 独立 Netty 端口 | gRPC 9090 + Web 8080 | 默认，最简单，互不影响 |
| Servlet 模式 | 与 Web 共用 8080 | 需要容器开启 HTTP/2，受容器能力限制 |
| 进程内 | 不占端口 | 测试或同进程模块调用 |

大多数项目采用独立端口。只有必须复用同一端口、或受网络策略限制时，才考虑 Servlet 模式。Servlet 模式下部分 gRPC 配置会失效，排错时要先确认当前用的是哪种传输。

# 7. 存量方案：net.devh starter

旧项目里的 net.devh starter 使用 `grpc.server.port`、`grpc.client.<name>.address` 和 `@GrpcClient` 注入，配置前缀与官方 `spring.grpc.*` 完全不同。迁移时依赖、注解、配置、TLS 属性要一起改。

# 8. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 服务启动但客户端 `UNIMPLEMENTED` | 服务 Bean 没被扫描到 | 检查包路径与 `@GrpcService` |
| 端口被占用 | 9090 与别的进程冲突 | 改 `spring.grpc.server.port` |
| 客户端注入失败 | 用了 `@GrpcClient` 但引入的是官方 starter | 改用 `GrpcChannelFactory` 或换 starter |
| 客户端连不上 | `target` 写错或缺少 channel 配置 | 检查 `spring.grpc.client.channel.<name>.target` |
| 两个 starter 冲突 | 同时引入官方与 net.devh | 只保留一套 |
| 健康检查失败 | 没引入 `grpc-services` | 加依赖并确认自动配置生效 |
| 停机时大量 `UNAVAILABLE` | 关闭等待时间太短 | 调整 grace period 与 K8s 摘流顺序 |
| 拦截器不生效 | 拦截器不是 Bean 或没加 `@GlobalServerInterceptor` | 补注解并确认包扫描 |

# 9. 练习与自检

1. 用官方 Spring gRPC 搭一个服务端与客户端，要求服务实现注入一个 `Map` 作为内存存储，验证依赖注入生效。
2. 注册两个全局拦截器，用 `@Order` 控制顺序，打印日志验证"外层先进入、后返回"。
3. 把官方 starter 换成 net.devh starter，记录需要改动的依赖、注解和配置项，总结迁移检查清单。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 官方方案 | `spring-grpc-spring-boot-starter`，配置前缀 `spring.grpc.*` |
| 服务注册 | 任何 `BindableService` Bean 都会被绑定 |
| 客户端 | `GrpcChannelFactory` 或默认 stub 扫描 |
| 拦截器 | `@GlobalServerInterceptor` + `@Order` |
| 生命周期 | 容器启动后拉起 Server，关闭时优雅停机 |
| 选型 | 新项目用官方，存量项目先确认是否 net.devh |

下一篇进入错误处理与超时，讲清 `Status` 体系、错误详情、Deadline、取消传播，以及重试与幂等的边界。
