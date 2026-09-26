# 1. Testcontainers 概览

Testcontainers 基于 Docker 提供可复用的测试基础设施容器，适合集成测试。

## 1.1 典型场景

- 数据库（MySQL、PostgreSQL）
- 消息队列（Kafka、RabbitMQ）
- 外部服务依赖

# 2. 基本用法

```java
@Testcontainers
class UserRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:15");

    @Test
    void should_save_user() { }
}
```

## 2.1 与 Spring Boot 集成

```java
@DynamicPropertySource
static void register(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", postgres::getJdbcUrl);
    registry.add("spring.datasource.username", postgres::getUsername);
    registry.add("spring.datasource.password", postgres::getPassword);
}
```

# 3. 复用与性能

## 3.1 容器复用

```properties
testcontainers.reuse.enable=true
```

## 3.2 并行执行注意点

- 避免端口冲突
- 控制资源占用

# 4. 常见问题

- Docker 未启动导致测试失败
- CI 环境需要启用 Docker-in-Docker 或远程 Docker
