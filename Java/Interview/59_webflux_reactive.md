# 1. 知识点
- **同步阻塞 vs 异步非阻塞**：同步阻塞模型中每个请求占用一个线程，线程在 I/O 操作期间等待阻塞；异步非阻塞模型中少量线程处理大量请求，线程在等待 I/O 时释放去处理其他请求，数据就绪后通过回调/事件通知。
- **响应式编程（Reactive Programming）**：基于数据流和变化传播的异步编程范式。以异步非阻塞方式处理数据流，关注数据流的转换和组合而非控制流。
- **背压（Backpressure）**：下游消费者能向上游生产者发出信号，告知其处理能力（能消费多少数据），上游据此控制数据生产速率，防止下游被压垮（如内存溢出）。通过 `request(n)` 方法实现流量控制。
- **Reactive Streams 规范**：JVM 平台异步流处理的规范标准，定义了四个核心接口和一系列交互规则，是 Reactor、RxJava 等实现的共同基础。
- **四大核心接口**：`Publisher<T>`（发布者，生产数据）、`Subscriber<T>`（订阅者，消费数据）、`Subscription`（订阅关系，控制数据请求和取消）、`Processor<T,R>`（既是发布者又是订阅者，中间处理器）。
- **Subscriber 生命周期**：`onSubscribe(Subscription s)` → `request(n)` 触发数据流 → `onNext(T item)` 接收数据（可多次）→ `onComplete()` 正常结束 → `onError(Throwable t)` 异常结束。
- **Mono**：Reactor 核心类，表示 0-1 个数据的异步序列。类比 Java 中的 `Optional`，但支持异步。常用操作：`just()`、`empty()`、`error()`、`fromCallable()`、`defer()`。
- **Flux**：Reactor 核心类，表示 0-N 个数据的异步序列。类比 Java 中的 `Stream`，但支持异步和背压。常用操作：`just()`、`fromIterable()`、`fromStream()`、`range()`、`interval()`、`create()`。
- **创建操作符**：`just()`（给定值）、`fromIterable()`（从集合）、`create()`（编程式创建，需手动管背压，支持多线程）、`generate()`（同步逐个生成）、`interval()`（按时间间隔生成递增 Long）、`defer()`（延迟创建，每次订阅重新生成序列）。
- **转换操作符**：`map()`（同步转换，1:1）、`flatMap()`（异步转换，1:N，返回 Publisher，不保证顺序）、`concatMap()`（1:N，保证顺序）、`flatMapMany()`（Mono 转 Flux）、`transform()`（函数式组合操作链）、`cast()`（类型转换）、`buffer()`（按批收集）。
- **过滤操作符**：`filter()`（条件过滤）、`take()`（取前 N 个）、`takeLast()`（取后 N 个）、`skip()`（跳过前 N 个）、`distinct()`（去重）、`distinctUntilChanged()`（连续相同去重）、`sample()`（采样）。
- **组合操作符**：`merge()`（合并多个流，无序交织）、`concat()`（串联多个流，保持顺序）、`zip()`（配对组合，一一对应）、`combineLatest()`（取各流最新值组合）、`switchMap()`（切换映射，新值到则取消旧订阅）、`then()`（忽略元素，完成后发信号）。
- **错误处理操作符**：`onErrorReturn()`（出错返回默认值）、`onErrorResume()`（出错切换到备选流）、`onErrorMap()`（错误类型转换）、`retry()`（出错重试）、`doOnError()`（出错时的副作用操作）。
- **副作用操作符**：`doOnNext()`（每个元素发出时）、`doOnComplete()`（完成时）、`doOnError()`（出错时）、`doOnSubscribe()`（订阅时）、`doOnRequest()`（请求时）、`doOnCancel()`（取消时）、`doFinally()`（无论如何都会执行）、`log()`（日志记录）。
- **subscribeOn vs publishOn**：`subscribeOn` 影响**上游**操作符所在的线程（订阅链起始线程）；`publishOn` 影响**下游**操作符所在的线程（从该点之后的链）。`subscribeOn` 在整个链中只需一次（多个时第一个有效），`publishOn` 可在链中多次切换。
- **Schedulers（调度器/线程模型）**：`Schedulers.immediate()`（当前线程立即执行）、`Schedulers.single()`（单线程复用，适合小任务）、`Schedulers.parallel()`（固定大小线程池 = CPU 核数，适合 CPU 密集型）、`Schedulers.boundedElastic()`（弹性线程池，有界 10*CPU 核数上限，适合 I/O 密集/阻塞调用，替代已弃用的 elastic()）。
- **背压策略**：`limitRate(n)`（限制预取数量，下游每次请求不超过 n）、`limitRequest(n)`（限制总请求数）、`onBackpressureBuffer()`（缓冲未消费元素，可设最大容量）、`onBackpressureDrop()`（缓冲区满时丢弃最新）、`onBackpressureLatest()`（缓冲区满时保留最新丢弃旧数据）、`onBackpressureError()`（溢出时丢异常）。
- **WebFlux**：Spring 5 引入的响应式 Web 框架，基于 Reactor 和 Netty（默认），非阻塞 I/O。不使用 Servlet API，通过 `HttpHandler` 适配容器。支持注解式和函数式两种编程模型。
- **RouterFunction + HandlerFunction**：函数式编程模型。`RouterFunctions.route()` 定义路由（URL、方法、Accept 匹配），`HandlerFunction<ServerResponse>` 处理请求，替代 `@RequestMapping` 注解方式。
- **WebClient**：响应式 HTTP 客户端，替代 `RestTemplate`（阻塞式）。非阻塞、支持流式传输。核心方法：`get()`/`post()`/`put()`/`delete()`、`retrieve()`（提取响应体）、`exchange()`（获取完整响应含状态头和 Cookie）、`bodyToMono()`/`bodyToFlux()`（解码响应）、`onStatus()`（错误处理）。
- **R2DBC（Reactive Relational Database Connectivity）**：响应式关系型数据库连接规范，完全异步非阻塞，不依赖 JDBC。驱动有 `r2dbc-postgresql`、`r2dbc-mysql`、`r2dbc-h2` 等。`DatabaseClient` 是便捷 API，`R2dbcRepository` 类似 JPA 的 Repository。
- **Spring Data Reactive**：支持 MongoDB Reactive（`ReactiveMongoRepository`）、Redis Reactive（`ReactiveRedisTemplate`）、Cassandra Reactive、Couchbase Reactive、Elasticsearch Reactive 等。
- **WebFlux vs Spring MVC**：WebFlux 使用 Netty/Undertow，完全响应式栈；Spring MVC 使用 Tomcat，基于 Servlet 规范（阻塞式）。两者不能同时使用同一个接口，但可在同一应用中并存区分路径。
- **WebFlux 适用场景**：高并发网关和 API、推送通知、实时数据仪表盘、微服务间大量非阻塞调用、接口聚合（并行调用多个后端服务不阻塞）、文件上传/下载、SSE（Server-Sent Events）、WebSocket 通信。
- **Netty**：高性能异步事件驱动的网络应用框架，NIO 基础，是 WebFlux 的默认嵌入式容器。单线程的事件循环处理所有 I/O 操作，避免了传统线程池模式的上下文切换开销。
- **错误处理链路**：在 WebFlux 中异常不通过 `@ExceptionHandler` 抛给容器，而要通过返回的 `Mono.error()` 或 `Flux.error()` 传播，由 `@ControllerAdvice` 处理或函数式端点中通过 `onErrorMap/onErrorResume` 处理。
- **Server-Sent Events（SSE）**：服务器向浏览器推送事件的单向通信协议。WebFlux 中 `produces = MediaType.TEXT_EVENT_STREAM_VALUE` 的 `Flux` 返回 SSE 流。
- **WebFlux 安全**：Spring Security 提供 `@EnableWebFluxSecurity` 和 `ServerHttpSecurity` 支持 WebFlux，不使用 Servlet Filter，使用 `WebFilter`。
- **热流 vs 冷流**：冷流（Cold）每次订阅重新发送完整数据序列（如 HTTP 请求的响应）；热流（Hot）无论是否有订阅者都在产生数据，订阅者只能收到订阅之后的数据（如股票价格推送、传感器数据）。冷流通过 `publish()` + `connect()` 或 `share()` 可转为热流。

# 2. 面试题

## 2.1 什么是响应式编程？与传统命令式编程有什么区别？
**响应式编程**是一种以**数据流和变化传播**为核心的异步编程范式。开发者声明数据如何转换和组合（声明式），框架负责在数据到达时触发处理管线。

**与传统命令式的区别**：
| 维度 | 命令式（传统 MVC） | 响应式（WebFlux） |
|------|-------------------|-------------------|
| 线程模型 | 一个请求一个线程（Thread-per-Request） | 少量线程处理大量请求（Event Loop） |
| I/O 处理 | 阻塞等待（线程被挂起） | 非阻塞（线程释放处理其他请求） |
| 代码风格 | 顺序执行、同步风格 | 声明式链式调用、回调/函数式 |
| 资源利用 | 线程上下文切换开销大 | 高效利用 CPU，吞吐量高 |
| 数据方式 | 拉模式（Pull） | 推模式（Push），数据推送到订阅者 |
| 调试难度 | 较简单（栈跟踪清晰） | 较困难（异步回调栈难追踪，需 Reactor 调试工具） |

## 2.2 什么是背压（Backpressure）？在 Reactor 中如何实现？
**背压**是响应式流中下游消费者告知上游生产者自己能处理多少数据的反馈机制，防止生产者发送过快导致消费者缓冲区溢出或 OOM。

在 Reactive Streams 规范中，通过 `Subscription.request(n)` 实现：
- 下游 `Subscriber` 调用 `subscription.request(5)` 告知上游"我最多能处理 5 个"。
- 上游最多发送 5 个 `onNext()` 后暂停，等待下游再次 request。

**Reactor 中的背压操作符**：
- `limitRate(n)`：限制上游每次预取不超过 n。
- `onBackpressureBuffer(maxSize)`：当生产者快于消费者时，缓冲未处理的元素。
- `onBackpressureDrop()`：缓冲区满时丢弃新到达的元素。
- `onBackpressureLatest()`：缓冲区满时保留最新到达的元素，丢弃旧的。
- `onBackpressureError()`：缓冲区满时抛出异常。

```
Flux.range(1, 1000)
    .onBackpressureBuffer(100)  // 缓冲区大小 100
    .limitRate(20)              // 每次 request(20)
    .subscribe(data -> { ... });
```

## 2.3 Mono 和 Flux 有什么区别？分别适用什么场景？
| 维度 | Mono | Flux |
|------|------|------|
| 数据数量 | 0 或 1 个 | 0 到 N 个 |
| 类比 | `Optional<T>`（异步版） | `Stream<T>`（异步版） |
| 典型场景 | 单个资源获取、保存操作、删除 | 列表查询、流式数据处理、事件流 |
| 常用方法 | `just()`、`empty()`、`map()`、`flatMap()` | `just()`、`fromIterable()`、`filter()`、`collectList()` |

**场景选择**：
```java
// Mono: 查询单个用户
Mono<User> getUser(Long id) {
    return userRepository.findById(id);
}

// Flux: 查询所有用户、监听消息流
Flux<User> getAllUsers() {
    return userRepository.findAll();
}

Flux<Message> streamMessages() {
    return messageRepository.findByTopic(topic);
}
```

## 2.4 subscribeOn 和 publishOn 有什么区别？
- **`subscribeOn`**：指定**订阅操作**（整个链的起始点）在哪个 Scheduler 上执行。它影响**上游**直到第一个 `publishOn` 的操作。在整个链中通常只需要一个 `subscribeOn`。
- **`publishOn`**：指定**之后的**链上操作在哪个 Scheduler 上执行。它影响**下游**从当前点开始的所有操作。可以在链中使用多个 `publishOn` 在不同的 Scheduler 之间切换。

```java
Flux.just("data")
    .map(this::heavyCpuTask)       // 在 subscribeOn 指定的线程
    .publishOn(Schedulers.single())  // 切换线程
    .map(this::lightTask)          // 在 single 线程
    .subscribeOn(Schedulers.parallel()) // 设置上游的线程
    .subscribe();
```

**记住**：`subscribeOn` 管上游，`publishOn` 管下游。

## 2.5 Schedulers 有哪些类型？如何选择合适的 Scheduler？
| Scheduler | 特性 | 适用场景 |
|-----------|------|---------|
| `immediate()` | 当前线程执行，不切换 | 不需要异步，直接在调用线程执行 |
| `single()` | 单个可复用线程 | 小任务、需要使用同一线程的场景 |
| `parallel()` | 固定大小 = CPU 核数 | CPU 密集型（计算、编解码） |
| `boundedElastic()` | 弹性线程池，空闲线程回收，上限 10 * CPU 核 | I/O 密集型（数据库、HTTP 调用）、阻塞操作 |
| `elastic()` | 无限弹性线程，已弃用（用 boundedElastic 替代） | 已不推荐 |
| `fromExecutor()` | 自定义线程池 | 需要自定义线程参数时 |

**选择原则**：CPU 密集型用 `parallel()`；I/O 密集型或调用阻塞 API 用 `boundedElastic()`；不要用 `elastic()`。

## 2.6 WebFlux 和 Spring MVC 有什么区别？如何选择？
| 维度 | Spring MVC | Spring WebFlux |
|------|------------|---------------|
| 编程模型 | 命令式、同步阻塞 | 响应式、异步非阻塞 |
| 底层容器 | Tomcat（基于 Servlet） | Netty/Undertow（非 Servlet） |
| 线程模型 | Thread-per-Request | Event Loop + Worker |
| 并发能力 | 取决于线程池大小 | C10K/C100K 级别 |
| 数据库 | JDBC / JPA（阻塞） | R2DBC / Reactive MongoDB |
| 学习曲线 | 平缓 | 较陡 |
| 生态成熟度 | 非常成熟 | 快速增长中 |

**选择建议**：
- 使用 Spring MVC 如果：已有阻塞依赖（JDBC / JPA）、传统 Web 应用、团队不熟悉响应式。
- 使用 WebFlux 如果：高并发网关/API、微服务间调用、流数据实时推送、团队有能力理解响应式。
- **不要两者混用于同一接口**。但可以在同一个应用中分别处理不同路径。

## 2.7 WebFlux 中的 RouterFunction 和注解式 Controller 有什么区别？
**注解式 Controller**（传统方式）：
```java
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public Mono<User> getUser(@PathVariable Long id) {
        return userService.getUser(id);
    }
}
```

**函数式 RouterFunction**：
```java
@Bean
public RouterFunction<ServerResponse> userRoutes(UserHandler handler) {
    return RouterFunctions.route()
        .GET("/users/{id}", handler::getUser)
        .GET("/users", handler::listUsers)
        .POST("/users", handler::createUser)
        .build();
}
```

| 维度 | 注解式 | 函数式 |
|------|--------|--------|
| 代码风格 | 声明注解、反射调用 | 显式声明、方法引用 |
| 灵活性 | 简单直观，适合标准 REST | 更灵活，易于动态路由 |
| 类型安全 | 运行时检查 | 编译期检查更严格 |
| 复杂性 | 低，易上手 | 稍高，适合复杂路由需求 |
| 中间件 | Interceptor（类 Spring MVC） | `filter()` 在路由链上 |

## 2.8 WebClient 如何替代 RestTemplate？有什么优势？
**RestTemplate（不推荐）**：
```java
User user = restTemplate.getForObject("/users/{id}", User.class, 1L);
```

**WebClient（推荐）**：
```java
WebClient client = WebClient.create("http://localhost:8080");

Mono<User> userMono = client.get()
    .uri("/users/{id}", 1L)
    .retrieve()
    .onStatus(status -> status.isError(), resp -> 
        resp.bodyToMono(String.class).map(BusinessException::new))
    .bodyToMono(User.class);
```

**WebClient 优势**：
- 非阻塞异步，不占用线程等待响应。
- 支持流式传输（Flux 响应体）。
- 链式 API 和 Reactive 操作符（retry、timeout、fallback）。
- 支持请求/响应拦截过滤器。
- RestTemplate 已进入维护模式，新功能不添加。

## 2.9 如何配置 WebFlux 中的 CORS？
```java
@Configuration
public class WebFluxConfig implements WebFluxConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://example.com")
                .allowedMethods("GET", "POST")
                .allowedHeaders("*")
                .allowCredentials(true);
    }
}
```

或使用 `CorsWebFilter`：
```java
@Bean
public CorsWebFilter corsWebFilter() {
    CorsConfiguration config = new CorsConfiguration();
    config.addAllowedOrigin("http://example.com");
    config.addAllowedMethod("*");
    config.addAllowedHeader("*");
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return new CorsWebFilter(source);
}
```

## 2.10 WebFlux 中如何进行统一异常处理？
**注解式 Controller**：使用 `@ControllerAdvice` + `@ExceptionHandler`（与 Spring MVC 一致，但返回 `Mono/Flux`）。

**函数式端点**：在 RouterFunction 中使用 `onErrorResume`/`onErrorMap`：
```java
@Bean
public RouterFunction<ServerResponse> routes() {
    return RouterFunctions.route()
        .GET("/users", request -> 
            userRepository.findAll()
                .onErrorResume(e -> Flux.error(new BusinessException("查询失败")))
        )
        .onError(BusinessException.class, (e, request) -> 
            ServerResponse.status(400)
                .bodyValue(Map.of("error", e.getMessage()))
        )
        .build();
}
```

## 2.11 R2DBC 是什么？与 JDBC 有什么不同？
| 维度 | JDBC | R2DBC |
|------|------|-------|
| I/O 模式 | 同步阻塞 | 异步非阻塞 |
| 返回类型 | 原始 Java 对象 | `Mono<>` 或 `Flux<>` |
| 线程模型 | 一个请求占用一个线程 | Event Loop 模型，线程复用 |
| 事务管理 | `@Transactional + PlatformTransactionManager` | `@Transactional + ReactiveTransactionManager` |
| 连接池 | HikariCP、Druid | `r2dbc-pool` |
| 规范成熟度 | 成熟稳定 | 发展中 |
| 驱动支持 | 几乎所有数据库 | PostgreSQL、MySQL、H2、MSSQL、MariaDB |

```java
// R2DBC Repository
public interface UserRepository extends ReactiveCrudRepository<User, Long> {
    Flux<User> findByNameContaining(String name);
}
```

Spring Data R2DBC 不支持 JPA 的实体关联（`@OneToMany`、`@ManyToOne` 等），需手动处理。

## 2.12 Reactor 中的错误处理操作符有哪些？如何处理？
| 操作符 | 作用 |
|-------|------|
| `onErrorReturn(default)` | 发生错误时返回默认值 |
| `onErrorResume(fallbackFunction)` | 发生错误时切换到备选 Publisher |
| `onErrorMap(transformFunction)` | 将错误转换为另一种类型再抛出 |
| `retry(n)` | 出错时重试 n 次 |
| `retryWhen(companion)` | 条件重试（如指数退避） |
| `doOnError(consumer)` | 错误时的副作用（如记录日志） |
| `doFinally(consumer)` | 无论如何都执行（类似 finally） |

```java
userRepository.findById(id)
    .switchIfEmpty(Mono.error(new NotFoundException("用户不存在")))  // 空转错误
    .onErrorMap(e -> new BusinessException("查询失败", e))            // 错误类型转换
    .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)))               // 指数退避重试
    .doOnError(e -> log.error("查询用户失败: {}", id, e));            // 日志记录
```

## 2.13 冷流（Cold）和热流（Hot）的区别？
| 维度 | 冷流 Cold | 热流 Hot |
|------|----------|---------|
| 数据生成时机 | 每次订阅时重新生成 | 无论是否有订阅者都在生成 |
| 数据共享 | 每个订阅者独立接收完整序列 | 所有订阅者共享同一数据源 |
| 类比 | HTTP 请求（每次请求返回最新数据） | 广播电台（订阅者只能收到之后的内容） |
| Reactor 默认 | Mono/Flux 默认为冷流 | 通过 `ConnectableFlux`、`share()` 转为热流 |

```java
// 冷流转热流
ConnectableFlux<String> hot = source.publish();
hot.connect();  // 开始发射数据（此时订阅者可收到 connect 之后的数据）

// 另一种方式
Flux<String> hot = source.share();  // 第一个订阅者触发 connect，后续订阅者共享
```

## 2.14 WebFlux 中如何处理大文件上传和下载？
**文件上传**（Buffer 模式）：
```java
@PostMapping("/upload")
public Mono<Void> upload(@RequestPart("file") FilePart filePart) {
    Path path = Paths.get("/uploads/" + filePart.filename());
    return filePart.transferTo(path);
}
```

**文件下载**（流式响应）：
```java
@GetMapping("/download/{filename}")
public Mono<ResponseEntity<Resource>> download(@PathVariable String filename) {
    Resource resource = new FileSystemResource("/uploads/" + filename);
    return Mono.just(ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, 
                    "attachment; filename=\"" + filename + "\"")
            .body(resource));
}
```

通过 `FilePart.transferTo()` 和 `ZeroCopyHttpOutputMessage`（零拷贝写入），WebFlux 可以利用操作系统的零拷贝特性提升大文件传输效率。

## 2.15 WebFlux 中如何使用 Server-Sent Events（SSE）实现实时推送？
```java
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> streamEvents() {
    return Flux.interval(Duration.ofSeconds(1))
               .map(seq -> ServerSentEvent.<String>builder()
                       .id(String.valueOf(seq))
                       .event("periodic-event")
                       .data("Tick #" + seq)
                       .build());
}
```

也可以返回 `Flux<String>` 直接作为 SSE 流：
```java
@GetMapping(value = "/notifications", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> notifications() {
    return notificationService.stream()
               .map(Notification::getMessage);
}
```

## 2.16 如何将阻塞操作包装为响应式非阻塞？
当需要调用不可避免的阻塞 API（如旧版 JDBC、文件 IO）时：
```java
public Mono<User> getUserById(Long id) {
    return Mono.fromCallable(() -> {
        // 这里是阻塞调用，在 boundedElastic 线程上执行
        return jdbcTemplate.queryForObject("SELECT * FROM user WHERE id = ?", 
                new UserRowMapper(), id);
    })
    .subscribeOn(Schedulers.boundedElastic()); // 关键：将阻塞操作放在独立的弹性线程池
}
```

`Mono.fromCallable()` 将 Callable 包装为 Mono，`subscribeOn(Schedulers.boundedElastic())` 确保阻塞调用运行在专用线程池上，不阻塞 Event Loop 的主线程。

## 2.17 WebFlux 的 Security 配置与 Spring MVC 有什么区别？
| 维度 | Spring MVC | WebFlux |
|------|------------|---------|
| 注解 | `@EnableWebSecurity` | `@EnableWebFluxSecurity` |
| 配置类 | `SecurityFilterChain` Bean | `SecurityWebFilterChain` Bean |
| 配置对象 | `HttpSecurity` | `ServerHttpSecurity` |
| 认证对象 | `UsernamePasswordAuthenticationToken` | 相同 |
| 过滤器 | Servlet Filter | `WebFilter` |
| 方法安全 | `@PreAuthorize` | `@PreAuthorize`（相同） |

```java
@EnableWebFluxSecurity
public class SecurityConfig {
    @Bean
    public SecurityWebFilterChain securityFilterChain(ServerHttpSecurity http) {
        return http.authorizeExchange()
                .pathMatchers("/public/**").permitAll()
                .anyExchange().authenticated()
                .and()
                .oauth2ResourceServer().jwt()
                .and().and().build();
    }
}
```

## 2.18 什么是 Reactor 的操作符融合（Operator Fusion）？
操作符融合是 Reactor 的性能优化技术，将连续的多个操作符合并为一个以减少内存开销和对象创建。

- **宏融合（Macro-fusion）**：将 `just`、`map`、`filter` 等 1:1 操作符在汇编阶段合并为一条流水线（pipeline），一个元素从流水线的开始直接传递到结束，无需中间对象。
- **微融合（Micro-fusion）**：在消息发出/接收时共享队列，如 `publishOn` 和内层操作符合用一个线程队列。

开发者无需显式使用，Reactor 自动开启，但对性能理解有帮助。不是所有操作符都能融合，如 `flatMap` 涉及异步边界就不能线性融合。

## 2.19 WebFlux 中如何实现请求参数校验？
```java
@PostMapping("/users")
public Mono<User> createUser(@Valid @RequestBody Mono<User> userMono) {
    return userMono.flatMap(user -> {
        // 如果 Mono 中的对象校验失败，会自动转换为 WebExchangeBindException
        return userRepository.save(user);
    });
}
```

使用 `@Valid` + `@Validated`（Spring 校验）和 `@NotNull`、`@NotBlank`、`@Size` 等 JSR-303 注解。校验失败时 WebFlux 自动返回 400 Bad Request。

函数式端点中的校验：
```java
public Mono<ServerResponse> createUser(ServerRequest request) {
    return request.bodyToMono(User.class)
        .flatMap(user -> {
            Set<ConstraintViolation<User>> violations = validator.validate(user);
            if (!violations.isEmpty()) {
                return ServerResponse.badRequest().bodyValue(violations);
            }
            return userRepository.save(user)
                .flatMap(saved -> ServerResponse.ok().bodyValue(saved));
        });
}
```

## 2.20 Reactor 中如何测试响应式代码？
使用 `StepVerifier`（Reactor Test 提供）：
```java
@Test
void testFlux() {
    Flux<Integer> flux = Flux.just(1, 2, 3, 4, 5);
    
    StepVerifier.create(flux)
        .expectNext(1)
        .expectNext(2, 3)
        .expectNextMatches(n -> n >= 4)
        .expectComplete()
        .verify();
}

@Test
void testError() {
    Mono<Object> mono = Mono.error(new RuntimeException("error"));
    
    StepVerifier.create(mono)
        .expectError(RuntimeException.class)
        .verify();
}

@Test
void testWithVirtualTime() {
    // 模拟时间快进，无需真实等待
    StepVerifier.withVirtualTime(() -> 
        Flux.interval(Duration.ofSeconds(10)).take(1))
        .thenAwait(Duration.ofSeconds(10))
        .expectNext(0L)
        .expectComplete()
        .verify();
}
```

`StepVerifier` 是声明式的，通过 `.expectNext()`、`.expectComplete()`、`.expectError()` 等断言表达式逐步验证数据流中的事件序列。
