# 1. 为什么需要 Docker Compose

单个容器能解决"一个依赖"，但真实系统的测试往往同时需要数据库、缓存、消息，甚至还要把被测应用本身跑在容器里。用多个 `GenericContainer` 手工拼装当然可行，但网络、别名、启动顺序、环境变量会迅速变成一堆重复代码。Docker Compose 本来就是为"一组服务"设计的描述格式，Testcontainers 提供了把它直接搬进测试的能力。

Compose 模块的价值不在于"少写几行"，而在于**复用团队已有的编排描述**。开发环境用同一份 compose 文件，测试环境也用它，配置漂移会小很多。

# 2. DockerComposeContainer 基础

## 2.1 最小示例

把 compose 文件放在 `src/test/resources` 下，测试里引用它：

```java
import org.testcontainers.containers.DockerComposeContainer;

@Testcontainers
class ComposeSmokeTest {

    @Container
    static DockerComposeContainer<?> compose =
        new DockerComposeContainer<>(new File("src/test/resources/docker-compose-test.yml"))
            .withExposedService("postgres", 5432)
            .withExposedService("redis", 6379)
            .withLocalCompose(true);

    @Test
    void should_start_all_services() {
        String host = compose.getServiceHost("postgres", 5432);
        int port = compose.getServicePort("postgres", 5432);
        assertThat(host).isNotBlank();
        assertThat(port).isPositive();
    }
}
```

`withLocalCompose(true)` 表示调用宿主机上的 `docker compose` 命令，而不是再启动一个 compose 容器。它要求机器上安装了 Compose v2，但兼容性更好、日志更直观，测试环境推荐开启。

## 2.2 compose 文件

```yaml
# src/test/resources/docker-compose-test.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app_secret
    ports:
      - "5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 2s
      timeout: 3s
      retries: 20

  redis:
    image: redis:7.4-alpine
    ports:
      - "6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 2s
      timeout: 3s
      retries: 20
```

注意 `ports` 写成 `"5432"` 而不是 `"5432:5432"`。**只声明容器端口，让 Docker 分配随机宿主机端口**，这样并行测试不会冲突；Testcontainers 的 `getServicePort` 会返回实际映射端口。

## 2.3 暴露服务与等待

`withExposedService` 除了登记端口，还可以附带等待策略：

```java
@Container
static DockerComposeContainer<?> compose =
    new DockerComposeContainer<>(new File("src/test/resources/docker-compose-test.yml"))
        .withExposedService("postgres", 5432,
            Wait.forListeningPort().withStartupTimeout(Duration.ofMinutes(2)))
        .withExposedService("redis", 6379,
            Wait.forLogMessage(".*Ready to accept connections.*\\n", 1))
        .withLocalCompose(true);
```

**设计考量**：Compose 会并行启动所有服务，`depends_on` 默认只保证"先启动"，不保证"已就绪"。给每个关键服务配等待策略，比在测试里 `Thread.sleep` 可靠得多。

## 2.4 获取连接地址

`getServiceHost(service, port)` 返回宿主机地址，`getServicePort(service, port)` 返回映射端口，`getContainerByServiceName(service)` 返回容器对象、方便读日志。拿到 host 和 port 后，拼出 JDBC URL 或客户端配置，再注入 Spring：

```java
@DynamicPropertySource
static void props(DynamicPropertyRegistry registry) {
    String host = compose.getServiceHost("postgres", 5432);
    int port = compose.getServicePort("postgres", 5432);
    registry.add("spring.datasource.url",
        () -> "jdbc:postgresql://" + host + ":" + port + "/app");
    registry.add("spring.datasource.username", () -> "app");
    registry.add("spring.datasource.password", () -> "app_secret");
}
```

# 3. 多容器依赖与启动顺序

## 3.1 depends_on 的局限

```yaml
services:
  app:
    image: my-app:latest
    depends_on:
      - postgres
      - redis
```

`depends_on` 的短语法只控制启动顺序：Docker 先启动 postgres，再启动 app，但不会等 postgres 真正可以接受连接。app 启动太快时，第一轮连接往往失败。如果应用没有重试逻辑，容器会直接退出。

## 3.2 用健康检查表达就绪

Compose v2 支持长语法，等待健康状态：

```yaml
services:
  app:
    image: my-app:latest
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 2s
      retries: 20
  redis:
    image: redis:7.4-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 2s
      retries: 20
```

**注意**：`condition: service_healthy` 需要 Compose v2 与较新的 Docker Engine。老版本会忽略 condition，退化成普通启动顺序，表现为"本地能过、CI 偶发失败"。

## 3.3 应用侧仍要重试

即使配了健康检查，也建议应用在启动阶段对数据库连接做有限重试。原因有三：健康检查有间隔窗口；服务可能在运行中重启；不同环境的资源差异会改变启动时间。**把就绪判断同时放在编排层和应用层，是更稳的做法**。

> **行业实践**：集成测试里的"启动顺序"问题，十有八九不是顺序本身，而是"没有重试"。与其精确控制先后，不如让每个组件都能容忍依赖短暂不可用。

# 4. 网络与别名

## 4.1 默认网络

Compose 会为整个项目创建一个默认网络，项目内所有服务自动加入。服务之间用**服务名**互相访问：

```text
compose 默认网络（project_default）
  app ──▶ postgres:5432 / redis:6379     ← 服务名即 DNS 别名
宿主机 ──▶ localhost:<随机映射端口>       ← 测试 JVM 走这条
```

服务名就是 DNS 别名，由 Docker 内嵌 DNS 解析。所以 compose 里的应用配置应该写 `jdbc:postgresql://postgres:5432/app`，而不是 `localhost`。

## 4.2 容器名与服务名

Testcontainers 的 `withExposedService` 接受服务名。早期版本要求传容器名，形如 `<project>_<service>_1` 或 `<project>-<service>-1`，这也是很多旧教程写 `postgres_1` 的原因。两种写法的区别在于：

服务名是 compose 文件的 key（如 `postgres`），容器名则是 Docker 实际生成的 `testcontainers-postgres-1`。不确定时用 `docker compose ps` 查看真实名称。**新项目统一用服务名**，可读性更好。

## 4.3 宿主机与容器内地址不同

这是 Compose 测试最常见的困惑来源：

- **测试代码在宿主机 JVM 里**：用 `getServiceHost` + `getServicePort`，即映射后的地址；
- **另一个容器在 compose 网络里**：用服务名 + 容器端口，如 `postgres:5432`。

同一个服务有两套地址，混用就会连不上。判断标准是"发起连接的进程在哪"：在 Docker 外就用映射端口，在 Docker 内就用服务名。

# 5. 完整示例：数据库 + 缓存

沿用 2.2 的 compose 文件，给 postgres 加一个 `volumes` 挂载初始化脚本即可。`./db/schema.sql` 是**相对 compose 文件所在目录**的路径，脚本应放在 `src/test/resources/db/schema.sql`。

## 5.1 测试类

```java
@SpringBootTest
@Testcontainers
class CacheAsideIntegrationTest {

    @Container
    static DockerComposeContainer<?> compose =
        new DockerComposeContainer<>(new File("src/test/resources/docker-compose-test.yml"))
            .withExposedService("postgres", 5432)
            .withExposedService("redis", 6379)
            .withLocalCompose(true);

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        String pgHost = compose.getServiceHost("postgres", 5432);
        int pgPort = compose.getServicePort("postgres", 5432);
        registry.add("spring.datasource.url",
            () -> "jdbc:postgresql://" + pgHost + ":" + pgPort + "/app");
        registry.add("spring.datasource.username", () -> "app");
        registry.add("spring.datasource.password", () -> "app_secret");

        registry.add("spring.data.redis.host",
            () -> compose.getServiceHost("redis", 6379));
        registry.add("spring.data.redis.port",
            () -> compose.getServicePort("redis", 6379));
    }

    @Autowired
    private ProductService productService;

    @Test
    void should_hit_database_then_cache() {
        productService.get(1L);   // 第一次走数据库
        productService.get(1L);   // 第二次命中缓存

        // 断言放在业务层，不直接依赖 Redis 客户端
    }
}
```

**注意**：在 `@DynamicPropertySource` 里调用 `getServiceHost` 时，容器必须已经启动。`@Container` static 字段由扩展在类级别启动，早于 Spring 上下文创建，所以这里能拿到值。若改成手动 `start()`，要确保顺序。

# 6. 常见坑

## 6.1 Compose 版本与字段

不同版本的 Compose 对 `version` 字段、`condition`、`profiles` 的支持不一致。测试专用 compose 文件**尽量保持简单**：不用 profiles、不用扩展字段、不依赖最新语法。把它当成"最小可运行描述"，而不是生产编排的完整复制。

## 6.2 相对路径与打包

相对路径以 compose 文件所在目录为基准。如果 compose 文件被打进 jar，classpath 资源没有稳定的文件系统路径，相对挂载会失效。解决办法是把 compose 文件放在项目目录（如 `src/test/resources`）并用 `new File(...)` 引用，而不是用 `getResourceAsStream`。

## 6.3 端口写法

- `"5432"`：只暴露容器端口，宿主机随机分配，推荐；
- `"5432:5432"`：固定绑定，容易冲突，不推荐；
- 忘记写 `ports`：`withExposedService` 拿不到映射，测试连不上。

## 6.4 平台架构

Apple Silicon 上拉取 `amd64` 镜像会走模拟层，启动慢甚至失败。优先选择有 `arm64` 变体的镜像，或在 compose 里显式声明 `platform: linux/amd64`（接受性能损失）。CI 与本地架构不一致时，这类问题尤其隐蔽。

# 7. 与单容器方案的取舍

| 维度 | 多个 GenericContainer | DockerComposeContainer |
|------|----------------------|------------------------|
| 描述方式 | Java API | YAML 文件 |
| 复用团队配置 | 差 | 好 |
| 网络与别名 | 手工创建 Network | 自动 |
| 启动顺序 | 手工 dependsOn + 等待 | compose 原生支持 |
| 调试 | 直观 | 需要看 compose 日志 |
| 适用 | 少量、结构简单的依赖 | 多服务、已有 compose 资产 |

**设计考量**：如果只是"一个数据库 + 一个 Redis"，用两个 `GenericContainer` 更轻，启动也更快。当服务数量超过三个、或者团队已经有维护良好的 compose 文件时，Compose 模块的收益才明显。

# 8. 常见错误

| 现象 | 根因 | 处理 |
|------|------|------|
| `no such service` | 传了容器名但版本只认服务名 | 用 `docker compose ps` 确认名称 |
| 应用连不上数据库 | 容器内写了 `localhost` | 容器内用服务名，宿主机用映射地址 |
| 初始化脚本没执行 | 挂载路径相对位置错误 | 以 compose 文件目录为基准检查 |
| 启动超时 | 只等端口，未等服务就绪 | 配 healthcheck 或 `Wait` 策略 |
| 并行测试端口冲突 | compose 写了固定端口映射 | 改成只声明容器端口 |
| 测试结束容器残留 | compose 未随扩展停止 | 确认 `@Container` static，检查 Ryuk |

# 9. 练习与自检

**练习一**：写一个包含 PostgreSQL 与 Redis 的 compose 文件，用 `DockerComposeContainer` 启动，分别从宿主机和另一个容器访问，验证两套地址的差异。

**练习二**：给 PostgreSQL 加 healthcheck，在 compose 里用 `depends_on: condition: service_healthy` 控制一个初始化容器，观察去掉 healthcheck 后的失败现象。

**练习三**：把端口从 `"5432"` 改成 `"5432:5432"`，同时运行两个测试类，记录端口冲突的报错信息，再改回随机端口。

**自检清单**：

- 能说清服务名与容器名的区别；
- 知道宿主机访问和容器内访问为什么地址不同；
- 能解释 `depends_on` 为什么不等于"依赖就绪"。

# 10. 小结

| 主题 | 核心结论 |
|------|---------|
| Compose 模块 | 复用 YAML 编排，适合多服务依赖 |
| 暴露服务 | `withExposedService` 登记端口与等待策略 |
| 启动顺序 | `depends_on` 只保证顺序，健康检查才保证就绪 |
| 网络 | 容器内用服务名，宿主机用映射端口 |
| 端口 | 只声明容器端口，交给 Docker 随机分配 |
| 取舍 | 服务少用单容器，服务多且有 compose 资产才上 Compose |

本地能跑通只是第一步，CI 上往往没有现成的 Docker Desktop，还需要处理 DinD、远程 daemon、镜像缓存与资源限制。下一篇进入 CI 与远程 Docker。
