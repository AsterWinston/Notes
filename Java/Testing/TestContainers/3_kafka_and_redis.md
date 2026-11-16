# 1. 消息与缓存的测试难点

数据库之外，最常见的两个外部依赖是消息队列和缓存。它们的难点与数据库不同：数据库是同步请求响应，消息系统是**异步**的，缓存则对**连接时序和数据结构**敏感。一个 Kafka 测试如果只验证"发送方法没抛异常"，几乎等于没测；真正要验证的是消息被写入主题、被消费者按预期读到、反序列化正确。

Redis 的测试常卡在两类细节上：容器启动很快但连接时机不对，以及不同版本的命令行为有差异（例如 `SETEX` 与 `SET ... EX`、Lua 脚本的返回值类型）。用真实镜像能把这些不确定性一次性消除。

# 2. Kafka 容器

## 2.1 两种 KafkaContainer

Testcontainers 1.20 时代存在两套实现，容易混淆。

| 实现 | 包名 | 默认镜像 | 运行模式 |
|------|------|---------|---------|
| 旧版 | `org.testcontainers.containers.KafkaContainer` | Confluent 镜像 | 依赖 ZooKeeper 或外部配置 |
| 新版 | `org.testcontainers.kafka.KafkaContainer` | Apache Kafka 官方镜像 | KRaft，无需 ZooKeeper |

新项目应使用新版，对应 Kafka 3.x 之后的 KRaft 模式，启动更快、组件更少。构造器要求传入 `DockerImageName`：

```java
KafkaContainer kafka = new KafkaContainer(
    DockerImageName.parse("apache/kafka:3.7.0"));
```

## 2.2 最小示例

```java
@Testcontainers
class KafkaSmokeTest {

    @Container
    static KafkaContainer kafka =
        new KafkaContainer(DockerImageName.parse("apache/kafka:3.7.0"));

    @Test
    void should_expose_bootstrap_servers() {
        // 已包含随机映射端口，可直接交给客户端
        assertThat(kafka.getBootstrapServers()).isNotBlank();
    }
}
```

`getBootstrapServers()` 返回类似 `localhost:49153` 的地址，应用侧直接用它创建 `Producer`、`Consumer` 或配置 `spring.kafka.bootstrap-servers`。**不要自己拼 `getHost() + ":9092"`**，容器内的 9092 映射到宿主机时是随机端口。

## 2.3 等待就绪

Kafka 模块内置了等待策略，通常不需要配置。用 `GenericContainer` 自己搭时就必须显式等待：

```java
GenericContainer<?> kafka = new GenericContainer<>("apache/kafka:3.7.0")
    .withExposedPorts(9092)
    .waitingFor(Wait.forLogMessage(".*Kafka Server started.*\\n", 1))
    .withStartupTimeout(Duration.ofMinutes(2));
```

**设计考量**：Kafka 的"端口已监听"不等于"可以收发消息"。broker 启动后会经历日志恢复、controller 选举、元数据加载，这段时间端口是开的，但生产请求可能被拒绝。日志等待比端口等待更可靠。

## 2.4 生产者消费者测试

下面验证消息从生产到消费的完整链路：

```java
@Testcontainers
class OrderEventKafkaTest {

    @Container
    static KafkaContainer kafka =
        new KafkaContainer(DockerImageName.parse("apache/kafka:3.7.0"));

    @Test
    void should_round_trip_message() throws Exception {
        String topic = "order-created";
        String bootstrap = kafka.getBootstrapServers();

        // 1. 显式建主题，避免自动创建带来的分区差异
        try (AdminClient admin = AdminClient.create(Map.of("bootstrap.servers", bootstrap))) {
            admin.createTopics(List.of(new NewTopic(topic, 1, (short) 1))).all().get();
        }

        // 2. 生产一条消息
        try (Producer<String, String> producer = new KafkaProducer<>(Map.of(
                "bootstrap.servers", bootstrap,
                "key.serializer", StringSerializer.class.getName(),
                "value.serializer", StringSerializer.class.getName()))) {
            producer.send(new ProducerRecord<>(topic, "order-1", "{\"id\":1}")).get();
        }

        // 3. 从最早偏移量消费，轮询直到拿到消息
        try (Consumer<String, String> consumer = new KafkaConsumer<>(Map.of(
                "bootstrap.servers", bootstrap,
                "group.id", "test-group",
                "auto.offset.reset", "earliest",
                "key.deserializer", StringDeserializer.class.getName(),
                "value.deserializer", StringDeserializer.class.getName()))) {
            consumer.subscribe(List.of(topic));
            List<ConsumerRecord<String, String>> records = new ArrayList<>();
            Awaitility.await().atMost(Duration.ofSeconds(20)).until(() -> {
                consumer.poll(Duration.ofMillis(500)).forEach(records::add);
                return !records.isEmpty();
            });

            assertThat(records).hasSize(1);
            assertThat(records.get(0).key()).isEqualTo("order-1");
            assertThat(records.get(0).value()).contains("\"id\":1");
        }
    }
}
```

消费端用 `Awaitility` 轮询，不要用 `Thread.sleep(5000)`。固定睡眠要么太短导致偶发失败，要么太长拖慢套件；轮询在成功时立即返回，失败时给出明确超时。

> **行业实践**：涉及消息的断言，尽量验证"最终达到某个状态"而不是"立刻达到"。异步系统天然有延迟，用 `Awaitility` 包装能显著降低测试的不稳定率。

# 3. Redis 容器

## 3.1 用 GenericContainer 启动

Testcontainers 核心不包含官方 Redis 模块（社区有 `com.redis:testcontainers-redis`），最简单可靠的方式是用 `GenericContainer`：

```java
@Testcontainers
class RedisSmokeTest {

    @Container
    static GenericContainer<?> redis =
        new GenericContainer<>(DockerImageName.parse("redis:7.4-alpine"))
            .withExposedPorts(6379)
            .waitingFor(Wait.forLogMessage(".*Ready to accept connections.*\\n", 1));

    @Test
    void should_ping() {
        try (Jedis jedis = new Jedis(redis.getHost(), redis.getMappedPort(6379))) {
            assertThat(jedis.ping()).isEqualTo("PONG");
        }
    }
}
```

`redis:7.4-alpine` 体积小、启动快。日志里出现 `Ready to accept connections` 才代表可以接受命令，仅用 `Wait.forListeningPort()` 也可以，但日志等待更稳。

## 3.2 密码与配置

需要密码时通过启动命令传入，连接端用 `RedisStandaloneConfiguration` 设置同一个密码：

```java
GenericContainer<?> redis = new GenericContainer<>("redis:7.4-alpine")
    .withExposedPorts(6379)
    .withCommand("redis-server", "--requirepass", "test-secret")
    .waitingFor(Wait.forLogMessage(".*Ready to accept connections.*\\n", 1));

RedisStandaloneConfiguration config =
    new RedisStandaloneConfiguration(redis.getHost(), redis.getMappedPort(6379));
config.setPassword("test-secret");
LettuceConnectionFactory factory = new LettuceConnectionFactory(config);
factory.afterPropertiesSet();
```

**注意**：Redis 官方镜像默认没有配置文件，参数都要通过命令行或挂载配置文件传入。挂载自定义 `redis.conf` 时，记得让 `bind` 监听 `0.0.0.0`，否则宿主机映射端口连不上。

## 3.3 常用数据结构测试

Redis 的价值在于数据结构语义，测试应覆盖命令本身的边界：

```java
@Test
void should_support_expiry_and_lua() {
    try (Jedis jedis = new Jedis(redis.getHost(), redis.getMappedPort(6379))) {
        jedis.setex("session:1", 60, "user-1");
        assertThat(jedis.ttl("session:1")).isBetween(1L, 60L);
        assertThat(jedis.get("session:1")).isEqualTo("user-1");

        // Lua 返回值类型由脚本决定，容易与 Java 类型不匹配
        Object result = jedis.eval("return redis.call('SET', KEYS[1], ARGV[1]) and 1",
            List.of("lock:1"), List.of("owner-1"));
        assertThat(result).isEqualTo(1L);
    }
}
```

**设计考量**：Lua 脚本、事务 `MULTI/EXEC`、发布订阅这几类能力，在不同客户端和 Redis 版本上行为差异较大，用真实容器测试比 Mock 更有意义。

# 4. 端口与网络

## 4.1 随机端口映射

所有容器的 `getMappedPort(容器内端口)` 都返回宿主机上的随机端口。好处是并行测试不会冲突、同一镜像能跑多个实例；代价是调试时必须拿到真实端口，不能凭记忆访问 `localhost:6379`。

```text
┌──────────────────────────────┐
│ 宿主机                        │
│   localhost:49153 ───────────┼──▶ 容器内 6379 (Redis)
│   localhost:49154 ───────────┼──▶ 容器内 6379 (另一个 Redis)
└──────────────────────────────┘
```

## 4.2 容器间网络与别名

两个容器互相访问时，创建共享网络并设置别名：

```java
Network network = Network.newNetwork();

GenericContainer<?> redis = new GenericContainer<>("redis:7.4-alpine")
    .withNetwork(network)
    .withNetworkAliases("cache")
    .withExposedPorts(6379);

GenericContainer<?> app = new GenericContainer<>("my-app:latest")
    .withNetwork(network)
    .dependsOn(redis);
```

在 `app` 内部可以用 `cache:6379` 访问 Redis。`withNetworkAliases` 只在**同一网络内**有效，宿主机测试仍然要用 `getHost()` + `getMappedPort()`。

## 4.3 Kafka 的 listener 陷阱

Kafka 返回给客户端的地址由 `advertised.listeners` 决定，而不是客户端连接时使用的地址。配置不对时会出现"能连上 broker，但拿到的元数据指向不可达地址"的怪象。

KafkaContainer 默认把 advertised listener 配成宿主机可访问的地址，所以宿主机测试正常。如果消费者本身也是容器，就需要额外的内部 listener：

```text
宿主机测试  ──▶ localhost:49153 ──▶ Kafka (EXTERNAL listener)
应用容器    ──▶ kafka:29092     ──▶ Kafka (INTERNAL listener)
```

> **行业实践**：跨容器访问 Kafka 时优先用 Docker Compose，并在 compose 文件里显式声明 `KAFKA_LISTENERS` 与 `KAFKA_ADVERTISED_LISTENERS`。手工用多个 `GenericContainer` 拼接虽然可行，但配置容易遗漏。

# 5. 完整示例：订单事件链路

生产端只是 `KafkaTemplate.send("order-created", ...)`，消费端是 `@KafkaListener(topics = "order-created")` 把消息收集到列表。真正需要测试的是二者在同一上下文中的协作：

## 5.1 测试

```java
@SpringBootTest
@Testcontainers
class OrderEventFlowTest {
    @Container
    static KafkaContainer kafka =
        new KafkaContainer(DockerImageName.parse("apache/kafka:3.7.0"));

    @DynamicPropertySource
    static void kafkaProps(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired
    private OrderEventPublisher publisher;

    @Autowired
    private OrderEventConsumer consumer;
    @Test
    void should_consume_published_event() {
        publisher.publish(42L);

        Awaitility.await().atMost(Duration.ofSeconds(20)).untilAsserted(() ->
            assertThat(consumer.received()).anyMatch(msg -> msg.contains("\"orderId\":42")));
    }
}
```

这段测试把发送与消费放在同一个 Spring 上下文里，验证的是应用自身的序列化、监听器配置和主题路由，而不是 Kafka 客户端 API。**这才是集成测试真正该覆盖的内容**。

# 6. 常见错误

| 现象 | 根因 | 处理 |
|------|------|------|
| 消费端一直拿不到消息 | 生产后才订阅、`auto.offset.reset` 配置不当 | 先订阅再生产，或设为 `earliest` |
| `Connection to node -1 could not be established` | 使用了不可达的 advertised listener | 用 `getBootstrapServers()`，跨容器配内部 listener |
| 测试偶发失败 | 用固定 `sleep` 等待异步结果 | 改用 `Awaitility` 轮询 |
| Redis 连接被拒绝 | 只等端口，未等 `Ready to accept connections` | 加日志等待策略 |
| Redis 认证失败 | 容器设了密码但客户端没传 | 检查 `RedisStandaloneConfiguration` |
| Kafka 启动超时 | 资源不足或镜像未预热 | 加大超时、预拉镜像、减少并行 |

# 7. 练习与自检

**练习一**：用 `KafkaContainer` 启动 broker，创建 3 个分区的主题，生产 10 条带 key 的消息，验证同一 key 的消息落在同一分区。

**练习二**：用 `GenericContainer` 启动带密码的 Redis，分别用 Jedis 和 Lettuce 连接，验证错误密码抛异常、正确密码能 `PING`。

**练习三**：创建共享 `Network`，启动 Redis 与一个自定义应用容器，让应用容器通过别名 `cache` 访问 Redis，宿主机测试通过映射端口访问，比较两种访问方式。

**自检清单**：

- 能区分新旧两个 `KafkaContainer`，知道该用哪个包；
- 能解释 `getBootstrapServers()` 为什么不能手工拼；
- 知道消息测试为什么必须用轮询而非固定睡眠。

# 8. 小结

| 主题 | 核心结论 |
|------|---------|
| Kafka 容器 | 新项目用 `org.testcontainers.kafka.KafkaContainer` + KRaft |
| 等待策略 | 端口就绪不等于服务就绪，优先日志或健康检查 |
| 消息测试 | 显式建主题，轮询消费，验证端到端语义 |
| Redis | 用 `GenericContainer`，等待 `Ready to accept connections` |
| 网络 | 宿主机用映射端口，容器间用共享网络别名 |
| 稳定性 | 异步断言一律用 `Awaitility`，拒绝固定睡眠 |

单个容器解决"依赖真实"，但把它们接进 Spring Boot 还需要处理属性注入与上下文复用。下一篇进入 Spring Boot 集成，讲清 `@DynamicPropertySource`、`@ServiceConnection` 与测试上下文的复用机制。
