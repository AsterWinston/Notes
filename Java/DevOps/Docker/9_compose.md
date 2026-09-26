
# 1. Compose 的定位：概念与边界

Docker Compose 用一份 YAML 描述多容器应用，并在**单机**范围内完成：

1. 服务（容器）定义与启动顺序
2. 网络与服务发现（同网络下通过服务名访问）
3. 卷与配置挂载
4. 基础的健康检查与重启策略

Compose 的边界：

1. 它不是集群调度器（跨主机、高可用调度、自动扩缩容、自愈等能力有限）。
2. 适合本地开发、测试环境、小规模部署；更大规模通常会转向 Kubernetes 等编排平台（见 `17_orchestration.md`）。

# 2. Compose 模型：services/networks/volumes

Compose 文件主要由三类对象组成：

```text
docker-compose.yml
  |
  +-- services    -> 容器/服务定义
  +-- networks    -> 网络定义（默认也会自动创建一个网络）
  +-- volumes     -> 命名数据卷定义
```

最小示例：

```yaml
services:
  app:
    image: nginx:1.27
    ports:
      - "8080:80"
```

# 3. 常用字段（按使用频率排序）

## 3.1 image / build

```yaml
services:
  app:
    image: my-app:1.0

  app2:
    build:
      context: .
      dockerfile: Dockerfile
```

工程建议：

1. 开发期可用 `build` 方便迭代；生产更推荐 `image` 指向制品库中的镜像。
2. 镜像 tag 建议可追溯（SemVer + Git SHA），避免只用 `latest`。

## 3.2 ports

```yaml
ports:
  - "8080:8080"
```

提示：ports 是“对外暴露”；容器间互访通常走服务名 + 容器端口，不需要 ports。

## 3.3 environment / env_file

```yaml
environment:
  SPRING_PROFILES_ACTIVE: prod

env_file:
  - .env
```

注意：

1. `.env` 适合本地与测试环境；生产敏感信息要走更严格的密钥管理策略（至少不要提交到 Git）。
2. 变量替换是 Compose 的常见能力：`${VAR}`。

## 3.4 volumes

```yaml
volumes:
  - mysql-data:/var/lib/mysql
  - ./config:/app/config:ro

volumes:
  mysql-data:
```

工程建议：

1. 数据库等状态数据优先用命名 volume。
2. 配置文件、证书等适合 bind mount 且只读。

## 3.5 networks

```yaml
services:
  app:
    networks: [app-net]

networks:
  app-net:
    driver: bridge
```

提示：同一网络内，默认可用服务名互访（如 `mysql:3306`）。

## 3.6 restart

```yaml
restart: unless-stopped
```

# 4. 启动顺序：depends_on + healthcheck（重点）

## 4.1 `depends_on` 的真实含义

`depends_on` 解决的是**容器启动顺序**，不保证“业务已经可用”。

推荐用法是：`depends_on` 配合 `healthcheck`，让依赖服务先变为 healthy，再启动上层服务。

```yaml
services:
  app:
    depends_on:
      mysql:
        condition: service_healthy

  mysql:
    image: mysql:8.0
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
```

工程提醒：即使 `service_healthy` 生效，也不代表你的业务依赖“逻辑层面”就绪（例如 MySQL 已能 ping，但表结构初始化还没完成）。必要时仍需应用侧做重试/熔断。

# 5. 多环境组织方式（推荐两种）

## 5.1 `docker-compose.yml` + override 文件

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

典型策略：

1. `docker-compose.yml` 放通用定义（服务、网络、卷）。
2. `docker-compose.dev.yml` 放本地开发覆盖（bind mount、调试端口）。
3. `docker-compose.prod.yml` 放生产覆盖（镜像 tag、资源限制、重启策略）。

## 5.2 profiles（条件启动）

```yaml
services:
  app:
    profiles: ["prod"]

  debug-tool:
    image: alpine
    profiles: ["debug"]
```

```bash
docker compose --profile debug up -d
```

# 6. 常用命令（以 docker compose 为主）

```bash
docker compose up -d
docker compose up -d --build

docker compose ps
docker compose logs -f app
docker compose exec app sh

docker compose down
docker compose down -v

docker compose config  # 查看渲染后的最终配置（排查变量替换很有用）
```

# 7. 常见坑与排查

## 7.1 服务名访问失败

检查点：

1. 是否在同一个网络。
2. 是否误用 `container_name` 导致命名冲突（团队协作时更常见）。
3. 是否把服务放在不同 compose project 中（网络隔离）。

## 7.2 `depends_on` 不生效/不等待

检查点：

1. 是否配置了 healthcheck。
2. `condition: service_healthy` 是否被当前 compose 版本支持。
3. 即使等待 healthy，也需要业务侧重试。

# 8. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| Compose 解决什么 | 单机多容器声明式启动、网络/卷统一管理 |
| Compose 的边界 | 不负责跨主机调度/复杂自愈/大规模治理 |
| `depends_on` 的含义 | 启动顺序，不等于依赖服务可用 |
| 为什么要 healthcheck | 区分“进程存活”和“服务可用” |
