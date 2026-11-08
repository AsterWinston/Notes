# 1. Testcontainers 概览

Mock 解决了隔离，但隔离的代价是**真实中间件的行为永远没被测到**——H2 与 MySQL 的 SQL 方言差异、Kafka 的分区与消费位移、Redis 的过期语义，都是 Mock 和内嵌替代品覆盖不了的盲区。**Testcontainers 的思路是反过来的：用 Docker 在测试期启动真实的中间件容器，测试结束即销毁**，让集成测试既真实又可重复。它基于 JUnit Jupiter 扩展模型与 Spring 的上下文机制接线，1.19+ 版本对 JUnit 5.10、Spring Boot 3.2+ 均有完善支持。

## 1.1 典型场景

- **数据库**：MySQL、PostgreSQL、Oracle（`testcontainers/mysql` 等专用模块，内置 JDBC URL、用户名密码的便捷访问器）；
- **消息队列**：Kafka（`KafkaContainer`）、RabbitMQ、Pulsar；
- **缓存与存储**：Redis、Elasticsearch、MinIO（S3 兼容）；
- **任意 Docker 镜像**：`GenericContainer` 兜底，可跑自研依赖服务的真实镜像做契约验证。

判断"要不要上 Testcontainers"的口径与第 18 篇的 Mock 边界正好互补：**单元测试 Mock，集成测试真容器**。测试金字塔中层的集成测试，就是 Testcontainers 的主场。

## 1.2 依赖引入

```xml
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>junit-jupiter</artifactId>   <!-- @Testcontainers/@Container 扩展 -->
  <version>1.19.7</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>postgresql</artifactId>      <!-- 专用模块，按中间件选型 -->
  <version>1.19.7</version>
  <scope>test</scope>
</dependency>
```

Spring Boot 项目在 `spring-boot-dependencies` BOM 管理下可省略版本号；Boot 3.1+ 另有 `spring-boot-testcontainers` 模块提供 `@ServiceConnection`（见 2.3）。

# 2. 基本用法

## 2.1 @Testcontainers 与 @Container

`@Testcontainers` 激活 JUnit Jupiter 扩展，`@Container` 标注的字段由扩展托管生命周期：**static 字段的容器全类共享（类前启动、类后停止），实例字段的容器每个测试方法各起一个**。前者快，适合只读或能自清理的依赖；后者隔离最彻底，适合容器状态就是测试状态的场景（如每个用例要一个空数据库）。

```java
import org.junit.jupiter.api.Test;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.containers.PostgreSQLContainer;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
class UserRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:15")  // 镜像 tag 必须固定，禁用 latest
                    .withDatabaseName("app_test")
                    .withUsername("test")
                    .withPassword("test")
                    .withInitScript("schema.sql");    // classpath 下的初始化脚本，启动时执行

    @Test
    void should_persist_and_query_user() throws Exception {
        try (Connection conn = DriverManager.getConnection(
                postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword())) {
            conn.createStatement().execute(
                    "INSERT INTO users(id, name) VALUES (1, 'tom')");
            ResultSet rs = conn.createStatement()
                    .executeQuery("SELECT name FROM users WHERE id = 1");
            rs.next();
            assertThat(rs.getString("name")).isEqualTo("tom");
        }
    }
}
```

**关键点：容器端口是随机的。** `PostgreSQLContainer` 把容器内 5432 映射到宿主机随机端口，`getJdbcUrl()` 返回的是映射后的真实连接串。任何硬编码 `localhost:5432` 的写法都测不到这个容器——这也是 `@DynamicPropertySource`/`@ServiceConnection` 存在的根本原因。

## 2.2 与 Spring Boot 集成：@DynamicPropertySource

容器启动后才知道端口，而 Spring 容器初始化需要 `spring.datasource.url`——时序上是"先有鸡还是先有蛋"。`@DynamicPropertySource` 就是解药：它在 ApplicationContext 刷新前回调，把**惰性求值的属性**（Supplier）注册进 Environment，取值发生在容器启动之后：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest
@Testcontainers
class OrderRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:15");

    @DynamicPropertySource
    static void registerPgProps(DynamicPropertyRegistry registry) {
        // 传方法引用而非立即取值——求值发生在容器就绪之后
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void should_save_order() {
        orderRepository.save(new Order(1L, "CREATED"));
        assertThat(orderRepository.findById(1L)).isPresent();
    }
}
```

> **注意**：`@DynamicPropertySource` 的每个不同注册组合都会改变 Spring 上下文指纹，导致上下文缓存失效（见第 17 篇）。把容器和属性注册抽到一个共享基类或统一配置类，是控制容器型测试套件启动次数的关键手段。

## 2.3 @ServiceConnection：Boot 3.1+ 的零样板接线

Spring Boot 3.1 引入 `@ServiceConnection`（`spring-boot-testcontainers` 模块），把"从容器读连接信息、写进对应属性"的样板代码全部自动化——**支持清单内的容器（Postgres、MySQL、Redis、Kafka、RabbitMQ、Elasticsearch 等）只要标注 `@ServiceConnection`，连接属性自动注册**，不再需要手写 `@DynamicPropertySource`：

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest
@Testcontainers
class OrderRepositoryIT {

    @Container
    @ServiceConnection // 自动生成 spring.datasource.url/username/password，无需 @DynamicPropertySource
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:15");

    // ...测试体与 2.2 相同
}
```

对清单外的自定义容器，仍可用 `@DynamicPropertySource`，或实现 `ConnectionDetails` 接口扩展。`@ServiceConnection` 是当前的推荐默认——更少样板、更少拼错属性名的机会。

## 2.4 生命周期与就绪判定

Testcontainers 启动容器后并非立即放行，而是执行 **wait strategy** 等就绪：数据库容器等"能接受 JDBC 连接"，`GenericContainer` 默认等日志出现 `Listening on port`，可显式指定：

```java
new GenericContainer<>("my-service:1.2.3")
        .withExposedPorts(8080)
        .waitingFor(Wait.forHttp("/actuator/health")   // 以健康检查端点为准
                        .forStatusCode(200)
                        .withStartupTimeout(Duration.ofSeconds(60)));
```

自研镜像务必暴露健康端点并配置对应 wait strategy，否则会出现"容器端口开了但应用还没初始化完"的偶发失败——这类 Flaky 几乎总是就绪判定不严造成的。

# 3. 性能：启动开销与复用策略

## 3.1 单例容器模式（静态复用）

`@Container` 托管的 static 容器仍是"每个测试类起一次"。容器多了以后（Postgres + Kafka + Redis），类与类之间的重复启动成为套件耗时大头。**单例容器模式（Singleton Container Pattern）把容器提升为跨类共享的 JVM 级 static 实例**，并绕过扩展托管、自己控制生命周期：

```java
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.utility.DockerImageName;

public abstract class AbstractIntegrationTest {

    // 整个 JVM（整个测试套件）只启动一次的共享容器
    static final PostgreSQLContainer<?> POSTGRES;

    static {
        POSTGRES = new PostgreSQLContainer<>(
                DockerImageName.parse("postgres:15"));
        POSTGRES.start();
        // 不调用 stop()：注册 JVM 关闭钩子（Ryuk）在测试进程结束时回收，
        // 这样容器能活过所有测试类，实现最大化复用
    }

    @DynamicPropertySource
    static void registerProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }
}
```

所有集成测试继承该基类（或用静态工具类持有容器），配合 Spring 上下文缓存，**容器只启动一次、上下文只构建一次**，N 个测试类的边际成本趋近于零。代价是共享状态：测试之间必须自己保证数据隔离——每个测试用事务回滚（`@Transactional`）、独立 schema、或 `@Sql` 重建数据，**不能假设数据库是空的**。

> **行业实践**：中大型工程的主流选择是"单例容器 + 每个测试方法自带数据（Arrange 阶段插入、事务回滚或 truncate 清理）"。判断标准：套件里集成测试类超过 10 个，就应该考虑单例化；否则 `@Container static` 的类级复用已够用。

## 3.2 reuse：跨 JVM 复用的容器

`withReuse(true)` + `~/.testcontainers.properties` 里的 `testcontainers.reuse.enable=true` 让容器**在多次测试运行（多个 JVM 进程）之间存活**：Testcontainers 给容器打上哈希标签，下次运行时若发现同配置的容器已在运行就直接连接。本地开发反复跑测试时体验极佳（省掉每次 10-20 秒的启动），但：

- 它**不保证状态干净**——上一个 JVM 留下的数据还在，测试必须自清理；
- CI 上通常没有收益（每次构建都是新环境），反而增加不确定性；
- reuse 与单例模式是不同层面的复用：前者跨进程，后者进程内。

**推荐口径：本地开 reuse，CI 不开。**

## 3.3 启动性能的其他杠杆

- **镜像预热**：CI 缓存基础镜像层（GitHub Actions 的 docker layer cache），避免每次构建重新 pull；
- **`TESTCONTAINERS_RYUK_DISABLED=true`**：Ryuk 是负责清理孤儿容器的守护容器，受限 CI 环境（无特权）里启动它可能失败或拖慢，禁用后需确保 CI workspace 本身是一次性的；
- **测试分级**：Testcontainers 套件打 `@Tag("integration")`，与毫秒级单测分开执行——单测每次提交都跑，集成测试按分支/定时跑；
- **并行执行的端口与资源**：Testcontainers 用随机映射端口，JUnit 5 并行（第 13 篇）不会引起端口冲突，但 N 个并行容器会成倍吃内存，需给 Docker daemon 配置足够资源并控制并行度。

```text
复用层级（从细到粗）：

  @Container 实例字段 → 每个测试方法一个容器   （最隔离，最慢）
  @Container static   → 每个测试类一个容器     （类内共享）
  单例容器模式         → 每个 JVM 一个容器      （套件内共享，主流选择）
  withReuse(true)     → 跨 JVM 共享            （本地开发利器，CI 慎用）
```

# 4. 与 H2/内嵌库的对比

集成测试用"内嵌替代品"（H2 内存库、embedded-redis）曾是主流，Testcontainers 出现后这场对比有了清晰答案：**内嵌库快但失真，真容器慢但真实**。

| 维度 | H2（MySQL 兼容模式） | 内嵌 Redis 等 | Testcontainers |
|------|---------------------|--------------|----------------|
| 启动速度 | 极快（进程内） | 快 | 慢（镜像拉取 + 容器启动，秒级到分钟级） |
| SQL 方言保真 | 低：JSON 函数、`ON DUPLICATE KEY`、窗口函数细节、锁行为均有差异 | 中 | 高：就是生产同款镜像 |
| Flyway/Liquibase 脚本 | 常需为 H2 单独维护一套方言脚本 | — | 直接跑生产脚本 |
| 环境要求 | 无 | 无 | Docker daemon |
| CI 适配 | 简单 | 简单 | 需要 Docker 可用的 runner |
| 数据隔离 | 进程内，天然干净 | 同左 | 需自管理（事务回滚/truncate） |

典型失真案例：H2 的 MySQL 模式不支持 `utf8mb4` 排序规则细节与部分函数，测试全绿、上线报错；PostgreSQL 根本没有像样的内嵌替代品。**结论倾向明确：只要 CI 环境能跑 Docker，数据库集成测试就应该用 Testcontainers；H2 只留给"验证 JPA 映射本身"这类对 SQL 方言不敏感的轻量切片（`@DataJpaTest` 默认行为）**。

> **设计争议**：`@DataJpaTest` 默认用内嵌库替换真实数据源（`@AutoConfigureTestDatabase`），这被批评为"默认给了一个失真的测试环境"。社区的主流修正是在测试配置里全局关掉替换（`replace = Replace.NONE`）并配合单例 Postgres 容器，让所有 JPA 测试都跑在真实方言上——慢一点，但绿得可信。

# 5. CI 环境要求

Testcontainers 需要一个可用的 Docker daemon，这在各类 CI 上的落地方式不同：

- **GitHub Actions**：ubuntu-latest runner 自带 Docker，开箱即用；
- **GitLab CI**：两种主流方案——挂载宿主机 socket（`docker:dind` service 或 `/var/run/docker.sock` 卷挂载，后者更快但要求 runner 特权）；
- **Jenkins**：agent 机器装 Docker，或 Kubernetes 插件里用 sidecar 方式的 dind；
- **Kubernetes 原生 runner**：可用 Testcontainers Cloud 或远程 Docker host（`DOCKER_HOST` 环境变量指向远端 daemon）。

```yaml
# GitLab CI 示例：dind service
integration-test:
  image: maven:3.9-eclipse-temurin-17
  services:
    - docker:24-dind          # 提供 Docker daemon
  variables:
    DOCKER_HOST: tcp://docker:2375   # 指向 dind service
    DOCKER_TLS_CERTDIR: ""           # 内网禁用 TLS，简化配置
    TESTCONTAINERS_RYUK_DISABLED: "true" # dind 里 Ryuk 常需特权，直接禁用
  script:
    - mvn -B verify -Dgroups=integration
```

**本地开发的前置检查**：Docker Desktop（Win/Mac）或 daemon（Linux）必须运行中，Testcontainers 启动时会自检并在失败时给出明确报错（`Could not find a valid Docker environment`）。Windows 上注意 WSL2 后端的文件系统性能问题——镜像与卷放 WSL2 内比放 Windows 盘快得多。

# 6. 常见问题排查

| 症状 | 根因与解法 |
|------|-----------|
| `Could not find a valid Docker environment` | Docker 未启动/未装/当前用户不在 docker 组；本地起 Docker Desktop，Linux `sudo usermod -aG docker $USER` |
| 容器起了但连接被拒 | wait strategy 未就绪即放行；显式配置 `Wait.forHttp/forLogMessage` |
| 测试偶发失败、重跑就过 | 共享容器数据污染；补事务回滚/truncate，或检查单例模式下的测试间隔离 |
| CI 里镜像拉取超时 | 配置镜像仓库代理/缓存；私有 registry 凭据用 `~/.testcontainers.properties` 或环境变量注入 |
| Ryuk 启动失败 | 受限环境禁用：`TESTCONTAINERS_RYUK_DISABLED=true`，并确保环境一次性 |
| 本机端口被占用报错 | Testcontainers 默认随机端口，出现占用多为 `withFixedExposedPort` 误用——删掉固定端口 |

# 7. 小结

| 主题 | 关键结论 |
|------|---------|
| 定位 | 集成测试用真实容器替代 Mock/内嵌库，消除方言与行为失真 |
| 基本用法 | `@Testcontainers` + `@Container`（static 类级/实例方法级）；端口随机，连接串从容器访问器取 |
| Spring 接线 | `@DynamicPropertySource` 惰性注册属性；Boot 3.1+ 首选 `@ServiceConnection` 零样板 |
| 性能 | 单例容器模式（JVM 级 static）是套件提速主力；`reuse` 跨 JVM 复用适合本地不适合 CI |
| 对比 H2 | 能跑 Docker 就用真容器；H2 只留给方言不敏感的轻量切片 |
| CI | 需要 Docker daemon：GHA 自带、GitLab 用 dind、受限环境禁 Ryuk |

下一篇是整个 JUnit 系列的收官：最佳实践、坑与高频面试题。
