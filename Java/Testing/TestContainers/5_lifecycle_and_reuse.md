# 1. 生命周期的三个层次

容器测试慢，往往不是 Docker 慢，而是**启动次数太多**。要控制启动次数，先要分清三个层次的生命周期：JVM 进程、测试类、单个容器实例。三者嵌套，任何一层处理不当都会把启动成本放大。

```text
JVM 进程
 ├── 测试类 A
 │    └── static 容器：类内所有方法共享
 ├── 测试类 B
 │    └── static 容器：与 A 相互独立
 └── Ryuk：JVM 退出后兜底清理
```

`@Testcontainers` 管理的是"测试类"这一层：static 字段在类级别启动一次，实例字段在每个方法前后启动停止。JVM 层则由 Ryuk 兜底。理解这个分层，后面的单例、复用、并行才有落脚点。

## 1.1 容器的完整状态

一个容器的生命周期依次是：**创建 → 启动 → 等待就绪 → 使用 → 停止 → 删除**。Testcontainers 把前三步封装在 `start()` 里，把后两步封装在 `stop()` 里。

| 方法 | 作用 | 说明 |
|------|------|------|
| `start()` | 拉镜像、创建并启动容器、执行等待策略 | 幂等，重复调用无副作用 |
| `isRunning()` | 判断容器是否在运行 | 调试与断言时有用 |
| `getContainerId()` | 获取 Docker 容器 ID | 排查日志时用 |
| `stop()` | 停止并删除容器 | 不会自动重试 |
| `close()` | 同 `stop()` | 实现 `AutoCloseable` |

**注意**：`start()` 失败时不会留下半启动的容器，但会留下**已拉取的镜像**。镜像层是本地缓存，下次启动会快很多，这也是"第一次跑很慢、第二次快"的原因。

# 2. static 容器与实例容器

## 2.1 两种声明方式的行为差异

```java
@Testcontainers
class StaticContainerTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
    // 类内所有测试共享一个容器
}
```

```java
@Testcontainers
class InstanceContainerTest {
    @Container
    PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
    // 每个测试方法前启动、后停止
}
```

| 维度 | static | 实例 |
|------|--------|------|
| 启动次数 | 每类一次 | 每方法一次 |
| 隔离强度 | 中，需清理数据 | 高，容器级隔离 |
| 启动开销 | 低 | 高 |
| 适合 | 绝大多数集成测试 | 强隔离、破坏性测试 |

## 2.2 选择的判断标准

**设计考量**：先问"测试之间会不会互相影响"。影响只来自数据时，用 static 容器加 `TRUNCATE` 就够；影响来自服务配置、数据库扩展、集群拓扑这类无法清理恢复的状态，才值得为每个方法起新容器。大多数团队的实际选择是：**数据库、Kafka、Redis 一律用 static**，破坏性场景单独拆类并用实例容器。

# 3. 单例容器模式

## 3.1 实现方式

当多个测试类需要共享同一个容器时，可以把容器放进一个持有类，在静态块里启动：

```java
public final class SharedContainers {
    public static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("app")
            .withUsername("app")
            .withPassword("app_secret")
            .withInitScript("db/schema.sql");
    static {
        POSTGRES.start();   // 类首次被引用时启动
    }

    private SharedContainers() { }
}
```

测试类只引用、不声明 `@Container`：

```java
@SpringBootTest
class UserRepositoryTest {
    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", SharedContainers.POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", SharedContainers.POSTGRES::getUsername);
        registry.add("spring.datasource.password", SharedContainers.POSTGRES::getPassword);
    }
}
```

## 3.2 为什么不主动 stop

单例容器通常**不在测试结束时停止**，原因有两个。第一，无法确定哪个测试类是"最后一个"，提前停止会让后续测试失败。第二，主动停止需要注册 JVM shutdown hook，而 shutdown hook 在 `kill -9` 时不会执行，可靠性不如 Ryuk。

于是交给 Ryuk：JVM 正常退出或异常终止后，连接断开，Ryuk 删除容器。这是单例模式与 `@Testcontainers` 最大的行为差异——**static 字段由扩展负责停止，单例由 Ryuk 负责清理**。

## 3.3 单例模式的风险

- **状态累积**：所有测试共享一个数据库，数据清理必须严格；
- **调试困难**：容器在类加载时就启动，IDE 里看不到 `@Container` 标记；
- **依赖顺序**：两个测试类并行运行时可能同时触发 `SharedContainers` 的类初始化，JVM 保证只执行一次，但初始化本身会成为串行瓶颈；
- **不适合 CI**：CI 每次是一个新 JVM，单例的优势只在同一进程内成立。

> **行业实践**：模块多、测试类多、镜像大的项目才值得上单例。中小项目用抽象基类共享 static 容器，已经能获得大部分收益，且更符合框架约定。

# 4. reuse 模式

## 4.1 开启方式

reuse 的目标是**跨 JVM 复用容器**：第一次跑测试启动容器后不删除，第二次跑测试直接连上同一个容器。它需要两处配合。

第一，在用户级配置里开启：

```properties
# ~/.testcontainers.properties
testcontainers.reuse.enable=true
```

也可以设置环境变量 `TESTCONTAINERS_REUSE_ENABLE=true`（1.16+）。第二，在容器上声明：

```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
    .withDatabaseName("app")
    .withUsername("app")
    .withPassword("app_secret")
    .withReuse(true);
```

两处缺一不可。只加 `.withReuse(true)` 而配置未开启，容器仍然会被正常删除。

## 4.2 复用如何判定

Testcontainers 根据容器的**配置哈希**匹配可复用容器，包括镜像、环境变量、命令、端口、挂载等。哈希相同且容器仍在运行，就直接复用；否则新建。所以修改了初始化脚本、镜像标签或环境变量，都会导致复用失效——这是符合预期的行为，避免用错配置。

**注意**：复用容器的数据不会自动清理。第一次测试插入的数据，第二次测试仍然存在。要么测试自身幂等，要么每次启动时执行清理。

## 4.3 适用与风险

| 场景 | 是否推荐 | 原因 |
|------|---------|------|
| 本地开发反复跑单个测试 | 推荐 | 省去每次几秒到几十秒的启动 |
| CI 流水线 | 不推荐 | 每次新机器，复用无意义且可能读到脏状态 |
| 需要干净数据库的测试 | 不推荐 | 数据残留导致偶发失败 |
| 大镜像（Oracle、Elasticsearch） | 推荐 | 启动成本高，收益明显 |

> **警告**：reuse 的容器不受 Ryuk 管理，会一直留在机器上直到手动删除。长期使用后 `docker ps` 里会积累一批 `testcontainers/ryuk` 之外的休眠容器，需要定期清理。

```bash
# 查看带 Testcontainers 标签的容器，再用 docker rm -f 清理
docker ps -a --filter "label=org.testcontainers=true"
```

# 5. 并行测试

## 5.1 JUnit 并行配置

JUnit 5 支持类级和方法级并行：

```properties
# src/test/resources/junit-platform.properties
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
junit.jupiter.execution.parallel.mode.classes.default=concurrent
junit.jupiter.execution.parallel.config.strategy=fixed
junit.jupiter.execution.parallel.config.fixed.parallelism=4
```

开启后，多个测试类会同时执行，各自的 static 容器也会同时启动。**容器数量约等于并行度**，内存与 CPU 压力随之上升。

## 5.2 端口与资源

动态端口映射天然解决端口冲突：两个同类容器分别映射到不同的宿主机端口。真正需要担心的是资源：

- 每个 PostgreSQL 容器约占 50–150 MB 内存；
- Kafka 容器启动慢、内存占用高；
- 并行度设得比 CPU 核数还高，容器启动会互相拖慢，总时长反而增加。

**设计考量**：并行度不是越高越好。经验值是 `min(CPU 核数, 4)`，并对 Kafka、Elasticsearch 这类重容器单独降低并发。CI 机器核数少时，宁可串行也不要 OOM。

## 5.3 数据隔离

并行执行时，同一容器内的测试方法如果并发写同一张表，会互相干扰。三种处理方式：

1. **类级并行、方法级串行**：`mode.default=same_thread`，只让不同类并行；
2. **每个类独立容器**：类之间天然隔离，类内串行；
3. **按测试分配 schema 或库名**：同一容器内建多个 schema，方法各用各的。

最简单可靠的是第二种。第一种实现成本最低，但要求类内测试本身不并行。

**注意**：`@Transactional` 回滚在并行测试里并不可靠，因为事务绑定线程，异步或跨线程操作无法回滚。并行场景优先用显式清理或独立 schema。

# 6. 清理策略

## 6.1 Ryuk 的工作方式

Ryuk 是默认开启的。它通过 Docker socket 监听测试进程的连接，连接断开后删除本次会话创建的容器、网络和卷。日志里能看到类似 `Received signal, shutting down` 的记录。

```bash
# 查看 Ryuk 是否在运行，再用 docker logs 看日志
docker ps --filter "name=ryuk"
```

如果需要关闭（例如 CI 不允许挂载 socket）：

```bash
export TESTCONTAINERS_RYUK_DISABLED=true
```

关闭后必须自行保证清理，否则失败的流水线会留下大量容器。

## 6.2 手动与 CI 清理

```bash
# 只清理 Testcontainers 创建的容器，避免误删其他业务容器
docker ps -aq --filter "label=org.testcontainers=true" | xargs -r docker rm -f
```

PowerShell 用 `ForEach-Object { docker rm -f $_ }` 替代 `xargs`。

**警告**：不要用 `docker system prune -a` 作为常规清理手段，它会删除所有未使用的镜像和卷，可能影响同一台机器上的其他项目与构建缓存。

## 6.3 清理时机

| 时机 | 做法 |
|------|------|
| 单个测试类结束 | static 容器由 `@Testcontainers` 停止 |
| JVM 退出 | Ryuk 删除本会话资源 |
| 测试方法之间 | SQL `TRUNCATE` / 删除 key / 重建 topic |
| CI 任务结束 | Ryuk，或显式清理步骤兜底 |

# 7. 启动优化

## 7.1 镜像预热

首次运行的最大开销是拉镜像。可以在 CI 里提前拉取：

```bash
docker pull postgres:16-alpine
docker pull apache/kafka:3.7.0
docker pull redis:7.4-alpine
```

更彻底的做法是使用内部镜像仓库或 registry mirror，把公网拉取变成内网拉取。国内网络环境下，Docker Hub 限流是 CI 失败的高频原因。

## 7.2 减少容器数量

- 能共用一个数据库的测试，不要拆成多个容器；
- 用 schema 隔离代替实例隔离，消息与缓存测试按需分模块跑；
- 避免在 `@WebMvcTest` 等切片测试里启动无关容器。

## 7.3 容器与上下文对齐

Spring 上下文复用和容器复用是两件事，但目标一致：减少重复初始化。让二者对齐的做法是**把容器声明与属性注册都收进基类**，使所有子类的缓存键一致。优化后，N 个测试类从 N 次容器启动加 N 次上下文启动，降到各 1 次。

# 8. 常见错误

| 现象 | 根因 | 处理 |
|------|------|------|
| 每个方法都重启容器 | `@Container` 用了实例字段 | 改为 static |
| 测试结束容器还在 | reuse 开启或 Ryuk 被禁用 | 关闭 reuse，检查 Ryuk 日志 |
| 第二次运行读到脏数据 | reuse 容器保留了上次数据 | 测试前清理，或关闭 reuse |
| 并行测试 OOM | 容器数量超过机器承载 | 降低并行度，减少重容器 |
| 并行测试数据串味 | 同类方法并发写同一张表 | 类级并行、方法串行，或独立 schema |
| CI 偶发拉镜像失败 | 网络或限流 | 预热镜像、配置 mirror |

# 9. 练习与自检

**练习一**：把同一个测试类里的容器字段从 static 改成实例字段，记录测试总耗时变化，并观察 `docker ps` 中容器创建次数的差异。

**练习二**：开启 `testcontainers.reuse.enable=true` 并加 `.withReuse(true)`，连续运行两次测试，确认第二次启动时间下降，并检查数据库中是否残留上次的数据。

**练习三**：开启 JUnit 并行执行，把并行度从 2 调到 8，记录总耗时与内存占用，找出本机的最优并行度。

**自检清单**：

- 能画出 JVM、测试类、容器实例三层生命周期；
- 知道单例容器为什么通常不主动 stop；
- 能解释 reuse 的开启条件与数据残留风险。

# 10. 小结

| 主题 | 核心结论 |
|------|---------|
| static 容器 | 类级启动，配数据清理，是默认选择 |
| 实例容器 | 方法级隔离，只用于破坏性场景 |
| 单例模式 | 跨类共享，靠 Ryuk 清理，注意状态累积 |
| reuse | 跨 JVM 复用，本地有效、CI 慎用、数据不自动清理 |
| 并行 | 动态端口不冲突，真正的瓶颈是内存与隔离 |
| 清理 | Ryuk 兜底，按标签手动清理，避免全量 prune |

单容器的生命周期已经可控，但真实系统往往需要多个容器协同：应用、数据库、缓存、消息一起启动，还要处理网络别名与启动顺序。下一篇进入 Docker Compose 与容器网络。
