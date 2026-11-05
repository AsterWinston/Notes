# 1. Testcontainers 的定位

集成测试最尴尬的地方在于：单元测试跑得飞快，却证明不了 SQL 能不能执行、消息能不能消费、序列化在真实中间件上是否兼容；而连到一套共享的测试环境又慢、脏、随时可能被别人改坏。Testcontainers 想解决的正是这个夹层问题——**用一次性的、真实的、由测试代码自己管理的 Docker 容器，替代那些"像但不等价"的替身**。

它不是测试框架，也不是 Mock 库，而是一层构建在 Docker 之上的**测试基础设施编排库**。理解这一点，后面的模块划分、生命周期和排错思路都会顺理成章。

## 1.1 集成测试的现实困境

在没有 Testcontainers 的年代，Java 项目通常有三条路，每条都有明显短板。

第一条是**内存数据库**，最典型的是 H2 模拟 PostgreSQL、HSQLDB 模拟 MySQL。它启动快、零依赖，但 SQL 方言、类型系统、锁行为、事务隔离、函数支持都和真实数据库有偏差。测试里能跑的 `MERGE INTO`、`ON CONFLICT`、窗口函数，换到生产库可能直接报语法错误；反过来，生产上能用的特性在 H2 里又缺失。**测试通过反而制造了虚假的安全感**。

第二条是**共享测试环境**。所有开发者连同一套 MySQL、Kafka，好处是真实，坏处是数据互相污染、实例随时被占满、失败原因难以定位。某个人跑了一半的测试留下脏数据，下一个人就会看到莫名其妙的断言失败。

第三条是**纯 Mock**。把 `JdbcTemplate`、`KafkaTemplate` 全部 Mock 掉，测试确实快，但它验证的只是"我调用了一个方法"，而不是"这条 SQL 真的能查出数据"。Repository 层的 Mock 测试几乎不产生价值，因为这一层的本质就是与外部系统交互。

**设计考量**：这三条路的共同问题是把"环境"当成了不可控的外部条件。Testcontainers 的切入点是把环境也变成**测试代码的一部分**——需要什么版本、什么配置、什么初始化脚本，全部写在测试里，随测试生命周期创建和销毁。可重复性由此建立。

## 1.2 Testcontainers 的核心主张

Testcontainers 的核心主张可以拆成四句话：

- **真实优先**：跑的是真正的 PostgreSQL、Kafka、Redis 镜像，不是模拟实现；
- **一次一环境**：每个测试类或每组测试拿到独立容器，互不干扰；
- **代码即配置**：容器版本、端口、环境变量、初始化脚本都用 Java API 或 Compose 文件声明；
- **用完即弃**：测试结束自动停止并删除容器，即使 JVM 崩溃也有兜底回收机制。

这四点合起来，把集成测试从"依赖外部环境"变成"依赖 Docker 这一项基础设施"。只要机器上有可用的 Docker，测试在哪跑都一致。

# 2. Docker 依赖

## 2.1 为什么离不开 Docker

Testcontainers 本身不实现容器，它只是 Docker API 的调用方。测试启动时，库会通过 Docker daemon 拉取镜像、创建容器、映射端口、等待就绪，测试结束后再删除。因此 **Docker（或兼容的容器运行时）是硬性前置条件**，没有它整套机制无法工作。

这一点决定了它的适用场景：开发机需要装 Docker Desktop、Rancher Desktop、Colima 或 Podman；CI 的 runner 需要能访问 Docker daemon。纯 Serverless 构建环境、受限的企业沙箱、没有特权的 Kubernetes Pod，往往需要额外配置才能运行。

```bash
# 确认 Docker 可用，这是排错的第一步
docker version
docker info

# 确认当前用户可以免 sudo 访问 daemon（Linux 常见问题）
docker ps
```

> **行业实践**：团队引入 Testcontainers 前，先把"Docker 可用性"纳入开发环境清单。新成员拉下代码跑不通测试，十有八九不是代码问题，而是 Docker 没启动或没有权限。

## 2.2 Testcontainers 如何发现 Docker

### 2.2.1 环境发现顺序

Testcontainers 会按一定顺序探测 Docker 环境，常见来源包括：

| 来源 | 说明 |
|------|------|
| `DOCKER_HOST` | 显式指定 daemon 地址，如 `tcp://localhost:2375` |
| `~/.testcontainers.properties` | 用户级配置，可指定 `docker.host`、镜像前缀等 |
| Docker Desktop / Rancher Desktop | 读取其默认 socket 或命名管道 |
| Colima / Podman | 通过 socket 路径或环境变量识别 |
| 系统默认 socket | Linux 下 `/var/run/docker.sock` |

Windows 上通常是命名管道 `npipe:////./pipe/docker_engine`，macOS 与 Linux 是 Unix socket。远程 Docker 则通过 `DOCKER_HOST` 指向 TCP 地址，这部分会在 CI 与远程 Docker 一篇里展开。

### 2.2.2 Ryuk 与资源回收

一个容易被忽略的组件是 **Ryuk**。它是 Testcontainers 启动的一个辅助容器，负责在测试 JVM 退出后清理遗留容器。原理很简单：测试进程与 Ryuk 保持一条连接，连接断开（正常结束或进程被杀）后，Ryuk 就删除本次会话创建的容器和网络。

```text
┌──────────────┐   创建/启动   ┌─────────────────┐
│  测试 JVM    │ ────────────▶ │  PostgreSQL 容器 │
│              │               └─────────────────┘
│              │   心跳连接     ┌─────────────────┐
│              │ ────────────▶ │  Ryuk 容器       │
└──────┬───────┘               └────────┬────────┘
       │ JVM 退出 / 连接断开             │
       └───────────────────────────────▶│ 删除本会话资源
```

**设计考量**：为什么需要一个独立容器来做清理？因为测试进程可能被 `kill -9`、IDE 强制停止、CI 超时中断，Java 的 `finally` 和 shutdown hook 都未必执行。把清理逻辑放到另一个进程，才能在 JVM 非正常退出时仍然兜底。代价是某些受限环境（如不允许挂载 Docker socket 的 CI）需要设置 `TESTCONTAINERS_RYUK_DISABLED=true` 并自行保证清理。

# 3. 模块化架构

## 3.1 分层结构

Testcontainers 的 Java 实现是分层加模块化的。核心层只做通用容器编排，具体中间件的能力由独立模块提供。

```text
┌─────────────────────────────────────────────┐
│  测试代码：JUnit 5 + @Testcontainers         │
├─────────────────────────────────────────────┤
│  模块层：postgresql / mysql / kafka / ...    │
│  提供强类型容器类与默认配置                   │
├─────────────────────────────────────────────┤
│  核心层：GenericContainer / 等待策略 / 网络   │
│  生命周期、端口映射、日志、启动检查            │
├─────────────────────────────────────────────┤
│  Docker 客户端层：docker-java                │
├─────────────────────────────────────────────┤
│  Docker daemon / 远程 Docker / Podman        │
└─────────────────────────────────────────────┘
```

**核心层**由 `testcontainers` 这个 artifact 提供，包含 `GenericContainer`、`Network`、`WaitStrategy`、`MountableFile`、`DockerComposeContainer` 等通用能力。任何镜像都能用它启动，只是需要自己写端口、环境变量和等待逻辑。

**模块层**每个中间件一个 artifact，封装了该中间件的默认端口、默认镜像、健康检查与连接信息获取方法。例如 `PostgreSQLContainer` 直接提供 `getJdbcUrl()`、`getUsername()`、`getPassword()`，省掉了手工拼装。

## 3.2 常用模块

| 模块 artifact | 主要类 | 用途 |
|---------------|--------|------|
| `org.testcontainers:testcontainers` | `GenericContainer` | 核心，任意镜像 |
| `org.testcontainers:junit-jupiter` | `@Testcontainers`、`@Container` | JUnit 5 生命周期集成 |
| `org.testcontainers:postgresql` | `PostgreSQLContainer` | PostgreSQL |
| `org.testcontainers:mysql` | `MySQLContainer` | MySQL / MariaDB 兼容用法 |
| `org.testcontainers:kafka` | `KafkaContainer` | Kafka（KRaft 模式） |
| `org.testcontainers:rabbitmq` | `RabbitMQContainer` | RabbitMQ |
| `org.testcontainers:mongodb` | `MongoDBContainer` | MongoDB |
| `org.testcontainers:localstack` | `LocalStackContainer` | AWS 服务模拟 |
| `org.springframework.boot:spring-boot-testcontainers` | `@ServiceConnection` | Spring Boot 自动装配连接信息 |

**设计考量**：模块化让核心保持精简，也避免了"为了用一个 Redis 容器而引入所有中间件依赖"。反过来，模块的维护节奏与核心不完全同步，遇到新版本镜像或新协议时，有时需要用 `GenericContainer` 自己补位。这也是后面 Redis 一篇采用通用容器的原因。

## 3.3 依赖引入

以 Maven 为例，最小可用组合是核心、JUnit 集成和一个数据库模块：

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.20.4</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <version>1.20.4</version>
    <scope>test</scope>
</dependency>
```

若使用 Spring Boot 3.1+ 的 `@ServiceConnection`，还需要：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
```

Spring Boot 的依赖管理（`spring-boot-dependencies`）已经统一了 Testcontainers 的版本，使用 Boot 父 POM 时可以省略 `<version>`。**版本对齐很重要**：核心与各模块版本不一致时，容易出现 `NoSuchMethodError` 或等待策略行为差异。

# 4. 与内存库、Mock 的对比

## 4.1 三类测试替身的差异

| 维度 | 内存数据库（H2） | Mock / Stub | Testcontainers |
|------|------------------|-------------|----------------|
| 真实性 | 部分兼容 | 完全不真实 | 与生产同镜像 |
| 启动速度 | 毫秒级 | 无启动成本 | 秒级到数十秒 |
| 外部依赖 | 无 | 无 | 需要 Docker |
| SQL 方言 | 有偏差 | 不涉及 | 完全一致 |
| 数据隔离 | 每个测试可重建 | 无状态 | 每容器独立 |
| 排错成本 | 中 | 低 | 中（涉及容器） |
| 适用层 | 简单 CRUD | 上层业务逻辑 | 仓储、消息、端到端 |

## 4.2 选型建议

**单元测试仍然优先用 Mock**。Service 层的分支逻辑、异常处理、参数校验，用 Mockito 验证交互足够，也最快。Testcontainers 不该被用来替代单元测试，它的启动成本会让整个测试套件从秒级退化到分钟级。

**Repository 与消息消费者优先用 Testcontainers**。这一层的关键就是"和真实系统打交道"，Mock 掉数据库等于没测。用真实容器验证 SQL、事务、索引、序列化，收益最大。

**不要为了兼容性而坚持 H2**。如果生产是 PostgreSQL，测试就应该用 PostgreSQL。H2 的兼容模式只能掩盖问题，不能证明兼容。

> **行业实践**：常见的分层策略是"大量纯单元测试 + 少量真实依赖集成测试"。集成测试用 Testcontainers 跑在 `*IT` 或 `*IntegrationTest` 命名下，由 Maven Failsafe 或 Gradle 的独立 task 执行，与快速单元测试分开，避免开发者每次保存都等容器启动。

# 5. 适用边界

## 5.1 适合的场景

- **数据库访问层**：验证 SQL、迁移脚本、事务边界、隔离级别；
- **消息队列**：验证生产、消费、序列化、重试、死信；
- **缓存**：验证 Redis 数据结构、过期、Lua 脚本；
- **多服务协作**：用 Docker Compose 拉起应用依赖的一组服务；
- **端到端回归**：启动完整 Spring Boot 应用，通过真实端口发请求。

## 5.2 不适合的场景

- **纯算法与工具类**：没有外部依赖，用 JUnit 即可；
- **需要 GPU、特殊内核模块的场景**：容器并不天然解决；
- **没有 Docker 权限的环境**：受限 CI、部分托管 Kubernetes；
- **对启动时间极端敏感的反馈环**：可以保留少量烟雾测试，而不是全面铺开。

## 5.3 成本认知

引入 Testcontainers 会增加三类成本。第一是**时间成本**：首次拉镜像、每次启动容器都要几秒到几十秒。第二是**资源成本**：并行跑多个数据库容器会吃掉内存，CI 机器规格不足时容易 OOM。第三是**维护成本**：镜像版本、初始化脚本、等待策略都需要随项目演进调整。

**设计考量**：这些成本可以通过上下文复用、容器复用、镜像预热、按需选择模块来控制，但不能完全消除。把 Testcontainers 用在收益最高的地方，比"所有测试都上容器"更务实。

# 6. 常见错误

| 现象 | 根因 | 处理 |
|------|------|------|
| `Could not find a valid Docker environment` | Docker 未启动或当前用户无权限 | 启动 Docker，检查 `docker ps` 与 socket 权限 |
| 测试非常慢 | 每个测试方法都新建容器 | 容器声明为 `static`，复用 Spring 上下文 |
| CI 上镜像拉取失败 | 网络受限或 Docker Hub 限流 | 配置镜像加速、私有 registry、预拉镜像 |
| 容器启动后测试仍连接失败 | 只等待端口开放，未等待服务就绪 | 使用健康检查或日志等待策略 |
| 测试结束容器残留 | Ryuk 被禁用或 JVM 被强杀 | 检查 Ryuk 日志，CI 中增加清理步骤 |
| Windows 上路径挂载失败 | 盘符未共享或路径格式不对 | Docker Desktop 共享目录，使用 `MountableFile` |

# 7. 练习与自检

**练习一**：用 `GenericContainer` 启动一个 `redis:7` 容器，打印容器 ID、映射后的端口和启动日志，然后手动停止。体会核心层与模块层的差别。

**练习二**：写一个最简单的测试，分别用 H2 和 PostgreSQL 容器执行同一条包含 `ON CONFLICT` 的 SQL，观察 H2 是否报错，并记录两者的启动耗时。

**练习三**：在本地故意停止 Docker 后运行测试，记录异常信息；再恢复 Docker，设置 `TESTCONTAINERS_RYUK_DISABLED=true` 运行一次，观察容器是否残留。

**自检清单**：

- 能说清 Testcontainers 与 Docker 的分工，知道它是编排库而不是容器实现；
- 能画出核心层、模块层、Docker 客户端层的调用关系；
- 能解释 Ryuk 存在的必要性；
- 能判断一个测试该用 Mock、H2 还是真实容器。

# 8. 小结

| 主题 | 核心结论 |
|------|---------|
| 定位 | 用一次性真实容器替代不精确的测试替身 |
| 依赖 | 必须有可访问的 Docker daemon，Ryuk 负责兜底清理 |
| 架构 | 核心层做通用编排，模块层提供中间件专用能力 |
| 对比 | Mock 最快但不真实，H2 折中但有方言偏差，容器最真实但有启动成本 |
| 边界 | 仓储、消息、端到端收益最高；纯逻辑测试不必上容器 |

理解了整体定位之后，下一步就是把它用到最典型的场景：数据库。下一篇从 MySQL 与 PostgreSQL 容器开始，讲清初始化脚本、连接串、数据隔离和完整的测试写法。
