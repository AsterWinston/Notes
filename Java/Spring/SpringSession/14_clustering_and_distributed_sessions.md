# 1. 分布式 Session 的核心挑战

在分布式系统中，Session 管理面临以下挑战：

| 挑战 | 说明 |
|------|------|
| **高可用** | Session 存储需要多副本，防止单点故障 |
| **一致性** | 多个实例看到同一个 Session 的状态应一致 |
| **负载均衡** | 用户请求可能被路由到不同实例 |
| **扩缩容** | 实例增减时 Session 不受影响 |
| **故障恢复** | 实例宕机后 Session 不丢失 |
| **跨区域** | 多数据中心 / 多可用区场景 |

---

# 2. 负载均衡与 Session 亲和性

## 2.1 Sticky Session（黏性会话）

```
用户请求 → 负载均衡器 → ╔═══════════╗
                         ║ 实例 A    ║
                         ║ (Session) ║ ← 每次路由到同一实例
                         ╚═══════════╝
                              ↓
                         实例 B (不会收到请求)
```

**配置**（Nginx）：

```nginx
upstream backend {
    ip_hash;  # 按客户端 IP 做哈希
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}
```

**优缺点**：

| 优点 | 缺点 |
|------|------|
| 实现简单 | 实例宕机时 Session 丢失 |
| 无需外部存储 | 流量不均匀（IP 哈希分布不均） |
| 无网络开销 | 无法弹性伸缩 |
| | 对代理环境不友好 |

## 2.2 Session 共享（Spring Session）

```
用户请求 → 负载均衡器 → 实例 A ─┐
                               ├→ Redis/DB (Session)
                        实例 B ─┘

每个实例都能访问同一份 Session 数据
```

**配置**（Nginx）：

```nginx
upstream backend {
    # 不需要 ip_hash，任意轮询即可
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}
```

**优势**：真正的无状态，实例可随意扩缩容。

---

# 3. 集群环境下的特殊考虑

## 3.1 Session 并发修改

```java
// 问题：两个请求几乎同时修改同一 Session
// 请求 1：session.setAttribute("cart", cartItems1);
// 请求 2：session.setAttribute("cart", cartItems2);

// Redis: 后写入覆盖先写入，无并发控制
// JDBC: 后 UPDATE 覆盖先 UPDATE（取决于事务隔离级别）
```

**解决方案**：

```java
// 方案一：Redis WATCH + MULTI/EXEC 乐观锁
// 方案二：使用更细粒度的 Key
session.setAttribute("cart:timestamp", System.currentTimeMillis());
// 方案三：应用层合并冲突
```

## 3.2 跨实例事件一致性

```
实例 A 使 Session 过期 → 实例 B 可能在几毫秒后仍认为 Session 有效
```

**影响范围**：短暂不一致通常是可接受的。如果对一致性要求极高，可以考虑：

- 每次读写都实时查询 Redis（避免本地缓存 Session）
- 使用分布式锁保护 Session 操作

## 3.3 Session 预热

新实例加入集群时，没有任何 Session 的本地缓存：

```java
// 懒加载（默认）：首次请求时从 Redis 加载
// 无需额外处理，Spring Session 会自动从 Redis 加载
```

---

# 4. 多数据中心部署

## 4.1 架构方案

```
┌────────── 数据中心 A ──────────┐
│  ┌──────┐  ┌──────┐          │
│  │ App1 │  │ App2 │          │
│  └──┬───┘  └──┬───┘          │
│     └────┬────┘               │
│          ▼                    │
│    ┌──────────┐               │
│    │ Redis 主 │               │
│    └────┬─────┘               │
└─────────┼─────────────────────┘
          │ 同步
┌─────────┼─────────────────────┐
│    ┌────▼─────┐               │
│    │ Redis 从 │               │
│    └────┬─────┘               │
│          ▼                    │
│  ┌──────┐  ┌──────┐          │
│  │ App3 │  │ App4 │          │
│  └──────┘  └──────┘          │
└────────── 数据中心 B ──────────┘
```

## 4.2 Redis 异地多活方案

| 方案 | 说明 |
|------|------|
| **Redis 主从复制** | 数据中心 A 为 Master，B 为 Slave（需处理切换） |
| **Redis 集群 + Gossip** | 跨数据中心部署 Redis Cluster（延迟高） |
| **独立的 Redis + 事件同步** | 每个数据中心独立 Redis，通过消息队列同步变更 |
| **全球分布式 Redis** | 使用 Redis Enterprise 的 Active-Active 功能 |
| **GEO 路由** | 按地理位置将用户路由到最近的数据中心 |

---

# 5. Kubernetes 环境部署

## 5.1 Deployment 配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: app
          image: spring-app:latest
          env:
            - name: SPRING_DATA_REDIS_HOST
              value: redis-service
            - name: SPRING_SESSION_TIMEOUT
              value: 30m
          ports:
            - containerPort: 8080
```

## 5.2 Redis Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  selector:
    app: redis
  ports:
    - port: 6379
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
```

## 5.3 K8s 环境特殊配置

```yaml
# application-k8s.yml
spring:
  session:
    redis:
      namespace: k8s:session:${HOSTNAME}  # 使用 Pod 名称做命名空间
  data:
    redis:
      host: ${REDIS_HOST:redis-service}
      port: 6379
      timeout: 3000ms
      lettuce:
        pool:
          max-active: 8
          max-idle: 4
          min-idle: 2
```

---

# 6. 容错与降级

## 6.1 Redis 不可用时的降级策略

```java
@Component
public class SessionResilienceManager {

    @Autowired
    private SessionRepository<?> primaryRepository;

    @Autowired(required = false)
    private SessionRepository<?> fallbackRepository;

    public Session findById(String id) {
        try {
            return primaryRepository.findById(id);
        } catch (Exception e) {
            logger.error("Primary session store unavailable", e);
            // 降级到备用存储
            if (fallbackRepository != null) {
                return fallbackRepository.findById(id);
            }
            // 或返回 null（用户需重新登录）
            return null;
        }
    }
}
```

## 6.2 断路器模式

```java
@Configuration
public class CircuitBreakerSessionConfig {

    @Bean
    @Primary
    public SessionRepository<?> resilientSessionRepository(
            RedisSessionRepository redisRepository,
            JdbcSessionRepository jdbcRepository) {

        return new ResilientSessionRepository(
            redisRepository,    // 主
            jdbcRepository,     // 备用
            3,                  // 失败阈值
            Duration.ofSeconds(30) // 熔断时间
        );
    }
}
```

## 6.3 启动时的 Session 可用性检查

```java
@Component
public class SessionHealthIndicator implements HealthIndicator {

    @Autowired
    private SessionRepository<?> sessionRepository;

    @Override
    public Health health() {
        try {
            Session session = sessionRepository.createSession();
            sessionRepository.deleteById(session.getId());
            return Health.up()
                .withDetail("sessionStore", "available")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .build();
        }
    }
}
```

---

# 7. 会话迁移与滚动更新

## 7.1 蓝绿部署中的 Session

```
蓝环境 (旧版本)              绿环境 (新版本)
┌────────────┐              ┌────────────┐
│  App v1.0  │              │  App v2.0  │
│  Session:  │              │  Session:  │
│  userId="A"│              │  userId="A"│
└─────┬──────┘              └─────┬──────┘
      └──────────┬───────────────┘
                 │
         ┌───────▼───────┐
         │  Redis (共享) │
         │  Session 数据  │
         └───────────────┘
```

**关键要求**：
- Session 中的对象在新旧版本间**序列化兼容**
- Cookie 配置保持一致（名称、域名、路径等）
- 建议发布前先清除所有活跃 Session

## 7.2 滚动更新策略

```yaml
# K8s 滚动更新
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # 最多多出 1 个 Pod
    maxUnavailable: 0    # 不能有不可用的 Pod
```

**滚动更新期间的 Session 处理**：

1. 旧版本 Pod 持有一些用户的 Session
2. 新版本 Pod 启动并加入集群
3. 用户请求可能路由到新版本 Pod
4. 新版本 Pod 从 Redis 加载 Session
5. 如果序列化兼容，用户无感知
6. 如果序列化不兼容，用户被要求重新登录

---

# 8. 大规模集群实践

## 8.1 百万级 Session 存储

| 策略 | 说明 |
|------|------|
| **Redis 集群** | 16384 个 Slot 自动分片，支持水平扩展 |
| **Session 命名空间** | 按业务模块分 namespace，物理隔离 |
| **压缩序列化** | 使用 Kryo 或自定义压缩降低存储大小 |
| **合理 TTL** | 精简过期时间，及时清理无用 Session |
| **索引优化** | 索引 Key 也设置 TTL，定期清理孤儿索引 |

## 8.2 监控指标

```
应监控的关键指标：
├── 活跃 Session 总数
├── Session 创建速率（create/s）
├── Session 销毁速率（destroy/s）
├── Session 查找延迟（P50/P95/P99）
├── Redis 连接数和等待数
├── 过期 Session 清理速率
└── 按用户的 Session 数量分布
```
