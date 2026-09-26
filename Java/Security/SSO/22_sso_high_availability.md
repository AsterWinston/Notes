
# 1. 概述

SSO 认证中心是整个系统群的"咽喉"。一旦认证中心不可用，所有依赖它的系统都无法登录。因此 SSO 的高可用设计是整个架构中最关键的一环。

```
┌──────────────────────────────────────────────────────────────────┐
│                    SSO 高可用架构                                 │
│                                                                   │
│                         ┌──────────┐                              │
│                         │   DNS    │                              │
│                         │ 轮询/加权│                              │
│                         └────┬─────┘                              │
│                              │                                    │
│                    ┌─────────┴─────────┐                          │
│                    ▼                   ▼                          │
│              ┌──────────┐        ┌──────────┐                    │
│              │  L4 LB   │◄──────►│  L4 LB   │                    │
│              │ (Nginx)  │ 健康检查│ (Nginx)  │                    │
│              └────┬─────┘        └────┬─────┘                    │
│                   │                   │                            │
│       ┌───────────┼───────────────────┼───────────┐              │
│       ▼           ▼                   ▼           ▼              │
│  ┌────────┐ ┌────────┐          ┌────────┐ ┌────────┐           │
│  │CAS Svr1│ │CAS Svr2│          │CAS Svr3│ │CAS Svr4│           │
│  └───┬────┘ └───┬────┘          └───┬────┘ └───┬────┘           │
│      │          │                   │          │                  │
│      └──────────┼───────────────────┼──────────┘                  │
│                 │                   │                              │
│        ┌────────┴─────────┐ ┌──────┴──────────┐                  │
│        │   Redis Cluster  │ │   DB (主从)      │                  │
│        │   (Ticket/会话)  │ │   (用户/审计)    │                  │
│        └──────────────────┘ └─────────────────┘                   │
└──────────────────────────────────────────────────────────────────┘
```

# 2. 认证中心集群化

## 2.1 CAS Server 集群

```yaml
# CAS Server 集群配置点
# 1. Ticket Registry 使用 Redis 共享
# 2. Service Registry 可使用数据库/JPA 共享
# 3. 无状态 Session（无本地文件依赖）

# cas.properties（所有节点相同配置）
cas.ticket.registry.redis.host=redis-cluster.example.com
cas.ticket.registry.redis.port=6379
cas.ticket.registry.redis.password=${REDIS_PASSWORD}
cas.ticket.registry.redis.pool.max-active=20

cas.service-registry.jpa.url=jdbc:mysql://db-cluster.example.com:3306/cas
cas.service-registry.jpa.user=${DB_USER}
cas.service-registry.jpa.password=${DB_PASSWORD}
```

## 2.2 OAuth2 / OIDC 认证中心集群

```java
/**
 * 认证中心水平扩展关键点：
 * 1. Token 存储集中化（Redis/JDBC）
 * 2. 授权码存储集中化
 * 3. Session 外部化
 * 4. 无文件系统依赖
 */

// Token 存储使用 Redis
@Configuration
public class TokenStoreConfig {

    @Bean
    public TokenStore tokenStore(RedisConnectionFactory connectionFactory) {
        return new RedisTokenStore(connectionFactory);
        // 或者使用 JdbcTokenStore 存储到数据库
    }
}

// 授权码存储使用 Redis
@Bean
public AuthorizationCodeServices authorizationCodeServices(
        RedisConnectionFactory connectionFactory) {
    return new RedisAuthorizationCodeServices(connectionFactory);
}

// OAuth2 客户端信息存储
@Bean
public ClientDetailsService clientDetailsService(DataSource dataSource) {
    return new JdbcClientDetailsService(dataSource);
}
```

## 2.3 Keycloak 集群

```yaml
# Keycloak 集群（基于 Infinispan 分布式缓存）
# standalone-ha.xml 配置

# 使用 JDBC_PING 发现（不需要多播）
<subsystem xmlns="urn:jboss:domain:jgroups:8.0">
    <channels default="ee">
        <channel name="ee" stack="tcp"/>
    </channels>
    <stacks>
        <stack name="tcp">
            <transport type="TCP" socket-binding="jgroups-tcp"/>
            <protocol type="JDBC_PING">
                <property name="datasource_jndi_name">java:jboss/datasources/KeycloakDS</property>
            </protocol>
            <protocol type="MERGE3"/>
            <protocol type="FD_SOCK"/>
            <protocol type="FD_ALL"/>
            <protocol type="VERIFY_SUSPECT"/>
            <protocol type="pbcast.NAKACK2"/>
            <protocol type="UNICAST3"/>
            <protocol type="pbcast.STABLE"/>
            <protocol type="pbcast.GMS"/>
            <protocol type="MFC"/>
            <protocol type="FRAG3"/>
        </stack>
    </stacks>
</subsystem>
```

# 3. 负载均衡策略

## 3.1 Nginx 配置

```nginx
upstream sso_backend {
    # 负载均衡算法：ip_hash（Session 亲和性）
    # 注意：如果使用 Redis 共享 Session，可使用 least_conn
    least_conn;

    server sso-node1.example.com:8443 weight=5 max_fails=3 fail_timeout=30s;
    server sso-node2.example.com:8443 weight=5 max_fails=3 fail_timeout=30s;
    server sso-node3.example.com:8443 weight=3 max_fails=3 fail_timeout=30s;

    # 健康检查（需要 nginx-plus 或 nginx-module）
    # 或使用 DNS 健康检查
    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name sso.example.com;

    ssl_certificate     /etc/ssl/certs/sso.example.com.pem;
    ssl_certificate_key /etc/ssl/private/sso.example.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    # 客户端上传限制
    client_max_body_size 10m;

    # 超时设置（SSO 请求通常很快）
    proxy_connect_timeout 5s;
    proxy_send_timeout    10s;
    proxy_read_timeout    10s;

    location / {
        proxy_pass https://sso_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 支持（Keycloak 需要）
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # 健康检查端点
    location /health {
        proxy_pass https://sso_backend/actuator/health;
        access_log off;
    }
}
```

## 3.2 Spring Cloud Gateway 负载均衡

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: sso-auth-center
          uri: lb://sso-auth-center  # 负载均衡
          predicates:
            - Path=/sso/**
          filters:
            - StripPrefix=1
    loadbalancer:
      health-check:
        sso-auth-center:
          path: /actuator/health
          interval: 15s
```

# 4. 数据层高可用

## 4.1 Redis 高可用

```
方案 1: Redis Sentinel（主从 + 自动故障转移）
  Sentinel 1 ───┐
  Sentinel 2 ───┼── 监控 Master
  Sentinel 3 ───┘
  Master ←→ Slave 1, Slave 2
  
  优点：配置简单，社区成熟
  场景：中小规模集群

方案 2: Redis Cluster（分片 + 故障转移）
  Slot 0-5460    → Master 1 + Slave 1
  Slot 5461-10922 → Master 2 + Slave 2
  Slot 10923-16383→ Master 3 + Slave 3
  
  优点：水平扩展，自动分片
  场景：大规模集群
```

```yaml
# Spring Boot Redis Sentinel 配置
spring:
  redis:
    sentinel:
      master: mymaster
      nodes:
        - redis-sentinel-1:26379
        - redis-sentinel-2:26379
        - redis-sentinel-3:26379
    password: ${REDIS_PASSWORD}
    timeout: 3000ms
```

## 4.2 数据库高可用

```yaml
# MySQL 主从 + 故障切换（通过 Orchestrator / ProxySQL）
spring:
  datasource:
    # 使用 ProxySQL 作为数据库代理
    url: jdbc:mysql://proxysql:6033/cas
    hikari:
      maximum-pool-size: 50
      connection-test-query: SELECT 1
```

# 5. 多活部署

## 5.1 同城双活

```
┌─────────────────────────────────────────────────────────────┐
│                    同城双活架构                              │
│                                                              │
│  用户请求 ──► DNS (GSLB) ── 就近路由                         │
│                 │         │                                  │
│          ┌──────┘         └──────┐                           │
│          ▼                       ▼                          │
│   ┌─────────────┐         ┌─────────────┐                  │
│   │  机房 A      │         │  机房 B      │                  │
│   │  ┌─────────┐│         │  ┌─────────┐│                  │
│   │  │CAS Svr  ││◄────────┤  │CAS Svr  ││                  │
│   │  │x2       ││  专线   │  │x2       ││                  │
│   │  └─────────┘│         │  └─────────┘│                  │
│   │  ┌─────────┐│         │  ┌─────────┐│                  │
│   │  │Redis    ││◄────────┤  │Redis    ││  ← 双向同步     │
│   │  │(单机房) ││   专线   │  │(单机房) ││                  │
│   │  └─────────┘│         │  └─────────┘│                  │
│   └─────────────┘         └─────────────┘                  │
│                                                              │
│  特点：                                                      │
│  - 两个机房间通过高速专线连接（延迟 < 2ms）                   │
│  - 可采用 Redis 双向同步或共享 Redis 集群                     │
│  - 任意机房故障，流量立即切换到另一机房                       │
└─────────────────────────────────────────────────────────────┘
```

## 5.2 两地三中心

```
┌──────────────────────────────────────────────────────────────┐
│                   两地三中心                                   │
│                                                               │
│  同城双活（A+B） ─────+────── 异地灾备（C）                   │
│       ▲                          ▲                            │
│       │                          │                            │
│  正常流量                   仅灾备切换                         │
│                                                               │
│  数据同步策略：                                                │
│  A ↔ B: 实时同步（Redis 双向 / MySQL 双主）                   │
│  A → C: 异步复制（延迟 < 1s）                                 │
└──────────────────────────────────────────────────────────────┘
```

# 6. 故障降级方案

## 6.1 认证中心不可用时的降级

```java
/**
 * 降级策略：认证中心不可用时，业务系统使用本地缓存认证
 */
@Service
public class AuthDegradationService {

    // 本地缓存最近认证成功的用户信息
    private final Cache<String, AuthInfo> fallbackCache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(30))
        .build();

    /**
     * 正常认证（调用认证中心）
     */
    public AuthInfo authenticate(String token) {
        try {
            return authCenterClient.validate(token);
        } catch (AuthCenterUnavailableException e) {
            // 降级：使用本地缓存
            AuthInfo cached = fallbackCache.getIfPresent(token);
            if (cached != null) {
                log.warn("认证中心不可用，使用本地缓存降级: {}", cached.getUsername());
                return cached;
            }
            throw new ServiceUnavailableException("认证服务暂时不可用");
        }
    }

    /**
     * 认证成功后，更新本地缓存
     */
    public void onAuthSuccess(String token, AuthInfo info) {
        fallbackCache.put(token, info);
    }
}
```

## 6.2 业务系统降级开关

```java
/**
 * 降级配置中心（Apollo/Nacos）
 */
@Configuration
@RefreshScope
public class DegradationConfig {

    @Value("${sso.degradation.enabled:false}")
    private boolean degradationEnabled;

    @Value("${sso.degradation.cache-ttl-minutes:30}")
    private int cacheTtlMinutes;

    /**
     * 降级模式：
     * 1. 正常模式：所有请求都验证 Token
     * 2. 弱化模式：已登录用户用缓存，未登录用户拒绝
     * 3. 完全开放：不验证（极度危险，仅紧急情况）
     */
    @Value("${sso.degradation.mode:strict}")
    private String degradationMode; // strict | cache | open
}
```

# 7. 监控与自动恢复

## 7.1 健康检查端点

```java
/**
 * 扩展健康检查端点
 */
@Component
public class AuthHealthIndicator implements HealthIndicator {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    @Autowired
    private DataSource dataSource;

    @Override
    public Health health() {
        Health.Builder builder = new Health.Builder();

        // 1. 检查 Redis 连接
        boolean redisOk = checkRedis();
        builder.withDetail("redis", redisOk ? "UP" : "DOWN");

        // 2. 检查数据库连接
        boolean dbOk = checkDatabase();
        builder.withDetail("database", dbOk ? "UP" : "DOWN");

        // 3. 整体状态
        if (redisOk && dbOk) {
            return builder.up().build();
        } else if (redisOk || dbOk) {
            return builder.status(new Status("DEGRADED",
                "部分依赖不可用，服务降级运行")).build();
        } else {
            return builder.down().build();
        }
    }

    private boolean checkRedis() {
        try {
            redisTemplate.opsForValue().get("health:check");
            return true;
        } catch (Exception e) {
            return false;
        }
    }

    private boolean checkDatabase() {
        try (Connection conn = dataSource.getConnection()) {
            return conn.isValid(3);
        } catch (Exception e) {
            return false;
        }
    }
}
```

## 7.2 故障自愈

```yaml
# Kubernetes 部署配置（自动恢复）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sso-auth-center
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sso-auth-center
  template:
    spec:
      containers:
      - name: auth-center
        image: auth-center:latest
        ports:
        - containerPort: 8443

        # 存活探针（故障重启）
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          failureThreshold: 3

        # 就绪探针（流量切换）
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
          failureThreshold: 2

        # 启动探针
        startupProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 30

        # 资源限制
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"

      # Pod 反亲和性（分散到不同节点）
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: sso-auth-center
            topologyKey: kubernetes.io/hostname
```

# 8. 总结

| 层面 | 高可用措施 |
|------|----------|
| **应用层** | 多实例部署（>= 3），无状态设计 |
| **负载均衡** | Nginx/HAProxy + 健康检查 + 自动摘除 |
| **Session/Ticket** | Redis Cluster / Sentinel 集中存储 |
| **数据库** | 主从 + 读写分离 + 自动故障转移 |
| **配置** | 配置中心（Nacos/Apollo）动态下发改 |
| **降级** | 应用层缓存 + 降级开关 |
| **多活** | 同城双活 + 异地灾备 |
| **容器化** | Kubernetes 自动扩缩 + 自愈 |
| **监控** | 健康检查 + 告警 + 自动恢复 |
