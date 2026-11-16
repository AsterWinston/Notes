
# 1. 场景与目标

这章目标不是把 Compose 写成"生产集群编排"，而是给一套可复制的微服务联调骨架：

1. 一条命令拉起基础设施（MySQL/Redis/配置中心等）。
2. 业务服务在同一网络内通过服务名互访。
3. 用 healthcheck + depends_on 解决"启动顺序"与"基础就绪"的大部分问题。
4. 数据用 volume 持久化，避免每次 up/down 丢库。

Compose v2（`docker compose` 子命令，Go 重写实现）已经把 Compose 从独立 Python 工具变成 Docker CLI 内置能力，compose-spec 也是 OCI 项目，同一份 YAML 在 Docker、Podman、甚至部分 K8s 迁移工具（Kompose）之间有一定互操作性。理解这一点有助于把握 Compose 的定位：**它是"声明式单机运行拓扑"，不是集群平台**。

# 2. 推荐拓扑（示意）

```text
                 ┌───────────────────┐
  Browser/API -> │ gateway (可选)     │
                 └─────────┬─────────┘
                           │
                ┌──────────┴───────────┐
                │ app-net (bridge)     │
                └───┬──────────┬───────┘
                    ▼          ▼
                 ┌──────┐   ┌──────┐    ┌───────┐
                 │ order│──▶│ user │───▶│ redis │
                 └──┬───┘   └──┬───┘    └───────┘
                    │          │
                    └────┬─────┘
                         ▼
                     ┌───────┐
                     │ mysql │
                     └───────┘
```

工程边界：

1. Compose 适合单机联调与小规模服务落地。
2. 需要跨主机调度、弹性伸缩、滚动升级治理时，应该迁移到 Kubernetes 等平台（见 `17_orchestration.md`）。

# 3. services / networks / volumes 三要素组织

一份 compose 文件的顶层只有三大块（加上 `configs`/`secrets` 扩展）：**services** 定义"跑什么"，每个 service 本质是一个"模板化的容器组"，`scale` 时同名 service 的多个副本共享一个网络别名（DNS 轮询到健康实例）；**networks** 定义"怎么连"，默认所有 service 挂进 `<project>_default` 网络，显式声明网络可以做分层隔离；**volumes** 定义"数据放哪"，命名卷由 Docker 管理生命周期，bind mount 则直接映射宿主目录。

**设计考量**：

- **service 名是契约**：网络别名、DNS 名、依赖引用都以 service 名为准，改 service 名等于改服务发现地址，要和服务内配置一起改。
- **网络按信任域划分**：常见做法是 `frontend`（可被入口访问）与 `backend`（数据库、中间件）两张网络，DB 只挂 backend，天然不可从入口侧直连。
- **默认卷 vs bind mount**：数据库数据目录用命名卷（可迁移、权限可控），配置文件与初始化 SQL 用 bind mount 只读挂载。

# 4. 最小可用模板：Spring Boot + MySQL + Redis

```yaml
services:
  mysql:
    image: mysql:8.0.33
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: app_db
    ports:
      - "3306:3306"   # 本地工具连接用；服务间互访不需要 ports
    volumes:
      - mysql-data:/var/lib/mysql
      - ./sql/init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 10
    networks: [app-net]

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 20
    networks: [app-net]

  order-service:
    image: my-registry/order-service:dev
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/app_db
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      SPRING_REDIS_HOST: redis
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    ports:
      - "8081:8080"
    networks: [app-net]

volumes:
  mysql-data:

networks:
  app-net:
    driver: bridge
```

说明：

1. 服务间互访用 `mysql:3306`、`redis:6379`（服务名来自 compose service 名称）。
2. 对外暴露只在你需要从宿主机访问时才开 `ports`。
3. MySQL 初始化脚本用 `/docker-entrypoint-initdb.d`，只在首次初始化数据目录时执行。
4. `docker compose up -d` 拉起，`docker compose ps` 查看状态，`docker compose logs -f order-service` 跟踪单个服务。

# 5. 启动顺序与就绪：你需要解决的不是"谁先起"，而是"谁先可用"

## 5.1 depends_on 的三种语义

`depends_on` 容易被误解为"依赖就绪"，它实际只提供三种控制：

| 写法 | 语义 | 保证程度 |
|------|------|----------|
| `depends_on: [mysql]`（列表） | 只控制**创建顺序** | 最弱，mysql 容器 started 即可，进程可能还没监听 |
| `condition: service_started` | 同上，显式写法 | 依赖容器已启动 |
| `condition: service_healthy` | 等依赖的 healthcheck 通过 | **就绪级**，最常用 |
| `condition: service_completed_successfully` | 等依赖退出码为 0 | 适合初始化任务（建表 Job、数据灌入） |

`service_healthy` 的前提是被依赖方定义了 `healthcheck`，否则 compose 直接报错。Spring Boot 侧推荐用 Actuator：

```yaml
  order-service:
    image: my-registry/order-service:dev
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8080/actuator/health/readiness"]
      interval: 10s
      timeout: 3s
      retries: 12
      start_period: 40s   # 给 JVM 启动留缓冲, 期间失败不计入 retries
```

**start_period**（Compose v2 支持 `start_period`，对应 Dockerfile HEALTHCHECK 的 `--start-period`）是 JVM 应用的关键参数：没有它，健康检查会在应用还没完成类加载时就把容器判为 unhealthy。

## 5.2 业务侧仍要做重试

即使 Compose 等到了 MySQL healthy，也可能存在：初始化脚本还在执行、网络短暂抖动、主从切换。编排层的就绪控制只能消除"必然失败"的启动竞态，不能消除"概率性失败"的运行期抖动，业务侧依旧需要连接重试与超时配置（HikariCP 的 `connectionTimeout`、Resilience4j 重试等）。

> **注意**：`docker compose up --wait` 是 v2 的实用开关，它会阻塞到所有服务 healthy 才返回，CI 里常用它替代手写轮询脚本；但没配 healthcheck 的服务它只等到 started，别把"up --wait 返回了"等同于"整链路可用"。

# 6. profiles 与环境分层

多环境不再复制多份 compose 文件，有两种正交手段：

**profiles** 控制"哪些服务参与本次部署"：

```yaml
services:
  app:
    image: my-app:dev
    networks: [backend]

  mysql:
    image: mysql:8.0.33
    profiles: ["db", "test"]      # 只在启用 db 或 test profile 时创建

  mailhog:
    image: mailhog/mailhog
    profiles: ["test"]            # 联调用邮件挡板, 演示环境不启动

networks:
  backend:
```

```bash
docker compose --profile db up -d        # app + mysql
docker compose --profile test --profile db up -d   # 再加 mailhog
docker compose --profile "*" up -d       # 全部启动
```

**override 文件**做配置分层：`compose.yml` 放公共定义，`compose.override.yml` 自动叠加（开发机本地生效，不提交），`compose.prod.yml` 用 `-f` 显式合并：

```bash
docker compose -f compose.yml -f compose.prod.yml config   # 先渲染检查
docker compose -f compose.yml -f compose.prod.yml up -d
```

合并规则要点：标量后者覆盖前者；`environment`、`labels` 按键合并；`ports`、`volumes`、`command`（short 语法）默认**追加**而非替换，需要精确控制时用 long 语法（带 `!override`、`!reset`、`!merge` 标签）。这正是 `docker compose config` 存在的理由——**永远先看渲染结果再 up**。

# 7. 环境变量插值与注入的区分

compose 文件里有两种"变量"，机制完全不同，混淆它们是排障噩梦：

| 机制 | 语法 | 何时生效 | 来源 |
|------|------|----------|------|
| **插值（interpolation）** | `${VAR}`、`${VAR:-默认值}`、`${VAR:?错误提示}` | compose 解析 YAML 时替换 | `.env` 文件、shell 环境变量 |
| **容器环境变量** | `environment` / `env_file` | 容器进程启动时注入 | YAML 字面量、`.env` 文件、运行时环境 |

```yaml
services:
  app:
    image: my-app:${APP_VERSION:-dev}     # 插值: 决定拉哪个 tag
    env_file:
      - .env.runtime                       # 整个文件注入容器
    environment:
      SPRING_PROFILES_ACTIVE: ${SPRING_PROFILES_ACTIVE:-local}  # 两者叠加: 先插值再注入
```

插值默认值语法：`${VAR:-default}`（未定义或空时用 default）、`${VAR-default}`（仅未定义时用）、`${VAR:?err}`（未定义则报错终止，适合强制密钥）。`.env` 文件放在 compose 文件同级目录会被自动加载；它只影响插值层，想注入容器要显式出现在 `environment`/`env_file` 里。

> **行业实践**：密钥不进 Git 是底线，`.env` 必须列入 `.gitignore`；团队共享的"非敏感默认值"放 `.env.example` 模板，新成员 clone 后复制改名。生产用 compose 时优先 `env_file` 指向平台下发的路径，而不是把值写死在 YAML 里。

# 8. 数据与清理策略

## 8.1 保留数据（常用于开发联调）

```bash
docker compose down
```

`down` 删除容器和网络，但**保留命名卷与镜像**，下次 up 数据还在。

## 8.2 清库重来（谨慎）

```bash
docker compose down -v
```

`-v` 会删除项目创建的命名卷，数据库数据随之消失，执行前确认卷里没有需要保留的东西。更细粒度：`docker volume ls` 找到卷名，`docker volume inspect` 确认归属项目（label `com.docker.compose.project`）再删。

# 9. 常见坑与排查

## 9.1 服务名解析失败

检查点：

1. 是否在同一个网络。
2. 是否误用了多个 compose project——**project name 默认取目录名**，目录改名或在不同路径 checkout 后 up，会创建新的一套网络与卷前缀，旧容器还在跑，表现为"改了 compose 文件不生效"。显式 `name:` 字段或 `-p myproject` 固定项目名。

## 9.2 MySQL 初始化脚本没执行

常见原因：

1. volume 已存在且已有数据，MySQL 不会重复执行 init 脚本。
2. 脚本权限或后缀不符合要求（需要 `.sql`/`.sh`，挂载路径正确）。

## 9.3 端口映射冲突

解决：改宿主机端口，如 `8081:8080`。用 `docker compose config` 检查最终端口，再 `ss -lntp | grep 端口` 找占用者。

## 9.4 改 image tag 后服务没更新

`up -d` 只在配置发生**影响容器定义的变化**时重建容器。改了 `image:` 会重建；只改了远程 tag 指向的内容（同 tag 重新 push）不会感知，需要先 `docker compose pull` 再 up。

# 10. 生产使用 Compose 的边界

有些团队确实把 Compose 用到小规模生产（边缘节点、单机交付的私有化产品），这是合理的，但必须自己补齐 K8s 平台默认提供的那部分能力：

**设计考量**：

- **重启策略只解决进程级自愈**：`restart: unless-stopped` 管不了宿主机宕机，单机 Compose 没有节点级容灾，需要外部监控告警与快速重部署流程。
- **没有滚动发布**：`docker compose up` 重建容器是"先停后起"，有秒级中断；要零停机需要在前面挂 Nginx/LB 做双实例切换，或接受中断窗口。
- **没有期望状态调谐**：手工 `docker rm -f` 掉一个容器后 Compose 不会自动补齐副本，它只在 `up` 时做一次收敛。
- **数据层要显式设计**：数据库单容器单卷意味着卷坏了数据就没了，备份（mysqldump/物理备份 + 对象存储）必须独立成流程。

**适合用 Compose 上生产的场景**：单机部署的软件交付（私有化安装包）、边缘计算节点、开发测试环境、无跨机需求的小型内网服务。**不适合**：多副本高可用、频繁发布、需要弹性伸缩的业务——这些请直接上 Kubernetes。

# 11. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 为什么用 Compose 做联调 | 一键拉起依赖、统一网络与配置、可复现 |
| `depends_on` 能保证可用吗 | 列表形式只保证启动顺序；就绪要 `condition: service_healthy` + 业务重试 |
| profiles 解决什么 | 同一份文件按 profile 控制哪些服务参与部署 |
| `${VAR}` 插值和 environment 区别 | 插值发生在 YAML 解析期（.env/shell），environment 注入容器进程 |
| 生产能用 Compose 吗 | 小规模可以，但自愈、滚动发布、多副本调谐需自建；跨机治理应上 K8s |

# 12. 小结

Compose 微服务编排的知识主线是"**就绪控制 + 配置分层 + 边界认知**"三件事。就绪控制：`depends_on` 的四种 condition 里只有 `service_healthy` 和 `service_completed_successfully` 触及"可用"语义，JVM 服务务必配 `start_period`，业务侧重试不可省。配置分层：profiles 管"哪些服务参与"，override 文件管"同一服务在不同环境长什么样"，插值与 env_file 是两个不同阶段的变量机制，`docker compose config` 是唯一的"事实核查"手段。边界认知：Compose 是单机声明式拓扑，缺节点容灾、滚动发布与调谐循环，小规模生产要用它就必须显式补齐这四类缺口，超出规模就迁移 Kubernetes。仓库（`11_registry.md`）是下一篇的主题：镜像构建出来后如何存放、分发与治理。
