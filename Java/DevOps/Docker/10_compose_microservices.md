
# 1. 场景与目标

这章目标不是把 Compose 写成“生产集群编排”，而是给一套可复制的微服务联调骨架：

1. 一条命令拉起基础设施（MySQL/Redis/配置中心等）。
2. 业务服务在同一网络内通过服务名互访。
3. 用 healthcheck + depends_on 解决“启动顺序”与“基础就绪”的大部分问题。
4. 数据用 volume 持久化，避免每次 up/down 丢库。

# 2. 推荐拓扑（示意）

```text
                 +-------------------+
  Browser/API -> | gateway (optional)|
                 +---------+---------+
                           |
                +----------+-----------+
                | app-net (bridge)     |
                +---+----------+---+---+
                    |          |   |
                 +--v--+    +--v--+| +--------+
                 |order|    |user |+-+  redis |
                 +--+--+    +--+--+  +---+----+
                    |          |         |
                    +----+-----+---------+
                         |
                      +--v--+
                      |mysql|
                      +-----+
```

工程边界：

1. Compose 适合单机联调与小规模服务落地。
2. 需要跨主机调度、弹性伸缩、滚动升级治理时，应该迁移到 Kubernetes 等平台（见 `17_orchestration.md`）。

# 3. 最小可用模板：Spring Boot + MySQL + Redis

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

# 4. 启动顺序与就绪：你需要解决的不是“谁先起”，而是“谁先可用”

## 4.1 健康检查的设计原则

健康检查应满足：

1. 足够轻量（不要每次都跑重查询）。
2. 与“依赖就绪”相关（MySQL 用 ping，HTTP 服务用 health endpoint）。
3. interval/timeout/retries 要与实际启动耗时匹配。

## 4.2 业务侧仍要做重试

即使 Compose 等到了 MySQL healthy，也可能存在：

1. 初始化脚本还在执行。
2. 网络抖动/短暂抖动。

业务侧依旧需要重试与超时（尤其是数据库连接、注册中心连接）。

# 5. 配置与密钥：避免“写死在 compose 里”

建议：

1. `.env` 适合本地与测试。
2. 不要把密码写进镜像或提交到 Git。

`.env` 示例：

```text
MYSQL_ROOT_PASSWORD=devpass
```

# 6. 数据与清理策略

## 6.1 保留数据（常用于开发联调）

```bash
docker compose down
```

## 6.2 清库重来（谨慎）

```bash
docker compose down -v
```

# 7. 常见坑与排查

## 7.1 服务名解析失败

检查点：

1. 是否在同一个网络。
2. 是否误用了多个 compose project（默认 project name 不同会导致网络隔离）。

## 7.2 MySQL 初始化脚本没执行

常见原因：

1. volume 已存在且已有数据，MySQL 不会重复执行 init 脚本。
2. 脚本权限或后缀不符合要求。

## 7.3 端口映射冲突

解决：改宿主机端口，如 `8081:8080`。

# 8. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 为什么用 Compose 做联调 | 一键拉起依赖、统一网络与配置、可复现 |
| `depends_on` 能保证可用吗 | 只能辅助，真正可用要 healthcheck + 业务重试 |
| 微服务联调的关键点 | 网络（服务名）、配置注入、依赖就绪、数据卷 |
