
# 1. 动态路由概述

动态路由是指在不重启网关的情况下，动态地添加、修改、删除路由规则。Spring Cloud Gateway 通过事件驱动机制实现路由的动态刷新。

```
路由变更来源                   网关缓存
    │                            │
    ├── Actuator API ────→ RefreshRoutesEvent
    ├── 数据库变更 ──────→  ↑
    ├── Redis 变更 ──────→  │
    ├── Nacos 配置变更 ────→ │
    └── 自定义事件 ──────→  │
                             ▼
                     CachingRouteLocator.refresh()
```

# 2. 路由刷新机制源码解析

## 2.1 CachingRouteLocator 缓存

```java
public class CachingRouteLocator implements RouteLocator {

    private final RouteLocator delegate;
    private volatile Flux<Route> routes;

    public CachingRouteLocator(RouteLocator delegate) {
        this.delegate = delegate;
    }

    @Override
    public Flux<Route> getRoutes() {
        // 双重检查锁定实现缓存
        Flux<Route> localRoutes = this.routes;
        if (localRoutes == null) {
            synchronized (this) {
                localRoutes = this.routes;
                if (localRoutes == null) {
                    this.routes = delegate.getRoutes()
                        .doOnNext(route -> log.debug("Route cached: {}", route.getId()))
                        .cache();  // 缓存路由列表
                    localRoutes = this.routes;
                }
            }
        }
        return localRoutes;
    }

    public void refresh() {
        synchronized (this) {
            this.routes = null;  // 清空缓存，下次访问时重新加载
            log.info("Routes cache cleared, will reload on next request");
        }
    }
}
```

## 2.2 RefreshRoutesEvent 事件

```java
public class RefreshRoutesEvent extends ApplicationEvent {
    public RefreshRoutesEvent(Object source) {
        super(source);
    }
}
```

**事件监听与处理**：

```java
@Component
public class RouteRefreshListener implements ApplicationListener<RefreshRoutesEvent> {

    private final CachingRouteLocator cachingRouteLocator;

    public RouteRefreshListener(CachingRouteLocator cachingRouteLocator) {
        this.cachingRouteLocator = cachingRouteLocator;
    }

    @Override
    public void onApplicationEvent(RefreshRoutesEvent event) {
        log.info("Received RefreshRoutesEvent, refreshing routes...");
        cachingRouteLocator.refresh();
    }
}
```

## 2.3 RouteDefinitionWriter 接口

```java
public interface RouteDefinitionWriter {
    Mono<Void> save(Mono<RouteDefinition> route);       // 新增/更新
    Mono<Void> delete(Mono<String> routeId);            // 删除
}
```

默认实现：`InMemoryRouteDefinitionWriter`

```java
public class InMemoryRouteDefinitionWriter implements RouteDefinitionWriter {

    private final Map<String, RouteDefinition> routes = new ConcurrentHashMap<>();

    @Override
    public Mono<Void> save(Mono<RouteDefinition> route) {
        return route.flatMap(r -> {
            routes.put(r.getId(), r);
            return Mono.empty();
        });
    }

    @Override
    public Mono<Void> delete(Mono<String> routeId) {
        return routeId.flatMap(id -> {
            routes.remove(id);
            return Mono.empty();
        });
    }

    public Flux<RouteDefinition> getRouteDefinitions() {
        return Flux.fromIterable(routes.values());
    }
}
```

# 3. Actuator 动态路由

## 3.1 配置

```yaml
management:
  endpoints:
    web:
      exposure:
        include: gateway
  endpoint:
    gateway:
      enabled: true
```

## 3.2 路由 CRUD 操作

```bash
# 获取所有路由
curl http://localhost:8080/actuator/gateway/routes

# 获取单个路由详情
curl http://localhost:8080/actuator/gateway/routes/user_service

# 新增路由
curl -X POST http://localhost:8080/actuator/gateway/routes/new_route \
  -H "Content-Type: application/json" \
  -d '{
    "id": "new_route",
    "predicates": [{
      "name": "Path",
      "args": {"pattern": "/new/**"}
    }],
    "filters": [{
      "name": "StripPrefix",
      "args": {"parts": "1"}
    }],
    "uri": "http://localhost:8081",
    "order": 0
  }'

# 删除路由
curl -X DELETE http://localhost:8080/actuator/gateway/routes/new_route

# 刷新路由（清除缓存）
curl -X POST http://localhost:8080/actuator/gateway/refresh
```

# 4. 基于数据库的动态路由

## 4.1 数据模型

```java
@Entity
@Table(name = "gateway_route")
public class GatewayRouteEntity {

    @Id
    private String id;

    @Column(name = "uri", nullable = false)
    private String uri;

    @Column(name = "predicates", columnDefinition = "TEXT")
    private String predicates;    // JSON 格式

    @Column(name = "filters", columnDefinition = "TEXT")
    private String filters;       // JSON 格式

    @Column(name = "route_order")
    private int order;

    @Column(name = "metadata", columnDefinition = "TEXT")
    private String metadata;      // JSON 格式

    @Column(name = "enabled")
    private boolean enabled = true;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

## 4.2 自定义 RouteDefinitionLocator

```java
@Component
public class DatabaseRouteDefinitionLocator implements RouteDefinitionLocator {

    private final RouteRepository routeRepository;

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return Flux.fromIterable(routeRepository.findByEnabledTrue())
            .map(this::convertToRouteDefinition);
    }

    private RouteDefinition convertToRouteDefinition(GatewayRouteEntity entity) {
        RouteDefinition definition = new RouteDefinition();
        definition.setId(entity.getId());
        definition.setUri(URI.create(entity.getUri()));
        definition.setOrder(entity.getOrder());

        // 解析 predicates JSON
        if (entity.getPredicates() != null) {
            List<PredicateDefinition> predicates = parsePredicateDefinitions(
                entity.getPredicates());
            definition.setPredicates(predicates);
        }

        // 解析 filters JSON
        if (entity.getFilters() != null) {
            List<FilterDefinition> filters = parseFilterDefinitions(
                entity.getFilters());
            definition.setFilters(filters);
        }

        // 解析 metadata
        if (entity.getMetadata() != null) {
            definition.setMetadata(parseMetadata(entity.getMetadata()));
        }

        return definition;
    }

    private List<PredicateDefinition> parsePredicateDefinitions(String json) {
        ObjectMapper mapper = new ObjectMapper();
        try {
            JsonNode root = mapper.readTree(json);
            List<PredicateDefinition> predicates = new ArrayList<>();
            if (root.isArray()) {
                for (JsonNode node : root) {
                    PredicateDefinition def = new PredicateDefinition();
                    def.setName(node.get("name").asText());
                    Map<String, String> args = new HashMap<>();
                    JsonNode argsNode = node.get("args");
                    if (argsNode != null) {
                        argsNode.fieldNames().forEachRemaining(
                            key -> args.put(key, argsNode.get(key).asText()));
                    }
                    def.setArgs(args);
                    predicates.add(def);
                }
            }
            return predicates;
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to parse predicates", e);
        }
    }

    private List<FilterDefinition> parseFilterDefinitions(String json) {
        // 类似 predicates 解析逻辑
        return new ArrayList<>();
    }

    private Map<String, Object> parseMetadata(String json) {
        try {
            return new ObjectMapper().readValue(json,
                new TypeReference<Map<String, Object>>() {});
        } catch (JsonProcessingException e) {
            return new HashMap<>();
        }
    }
}
```

## 4.3 路由变更监听

```java
@Component
public class RouteChangeListener {

    private final RouteDefinitionWriter routeDefinitionWriter;
    private final ApplicationEventPublisher publisher;

    public RouteChangeListener(
            RouteDefinitionWriter routeDefinitionWriter,
            ApplicationEventPublisher publisher) {
        this.routeDefinitionWriter = routeDefinitionWriter;
        this.publisher = publisher;
    }

    // 新增/更新路由
    public void saveRoute(GatewayRouteEntity entity) {
        RouteDefinition definition = convertToRouteDefinition(entity);
        routeDefinitionWriter.save(Mono.just(definition)).subscribe();
        publisher.publishEvent(new RefreshRoutesEvent(this));
    }

    // 删除路由
    public void deleteRoute(String routeId) {
        routeDefinitionWriter.delete(Mono.just(routeId)).subscribe();
        publisher.publishEvent(new RefreshRoutesEvent(this));
    }
}
```

## 4.4 路由管理 Controller

```java
@RestController
@RequestMapping("/admin/routes")
public class RouteAdminController {

    private final RouteRepository routeRepository;
    private final RouteChangeListener routeChangeListener;

    @PostMapping
    public ResponseEntity<Void> createRoute(@RequestBody GatewayRouteEntity entity) {
        entity.setCreatedAt(LocalDateTime.now());
        entity.setUpdatedAt(LocalDateTime.now());
        routeRepository.save(entity);
        routeChangeListener.saveRoute(entity);
        return ResponseEntity.ok().build();
    }

    @PutMapping("/{id}")
    public ResponseEntity<Void> updateRoute(
            @PathVariable String id,
            @RequestBody GatewayRouteEntity entity) {
        entity.setId(id);
        entity.setUpdatedAt(LocalDateTime.now());
        routeRepository.save(entity);
        routeChangeListener.saveRoute(entity);
        return ResponseEntity.ok().build();
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteRoute(@PathVariable String id) {
        routeRepository.deleteById(id);
        routeChangeListener.deleteRoute(id);
        return ResponseEntity.ok().build();
    }

    @PostMapping("/refresh")
    public ResponseEntity<Void> refresh() {
        routeChangeListener.refreshAll();
        return ResponseEntity.ok().build();
    }
}
```

# 5. 基于 Redis 的动态路由

## 5.1 RedisRouteDefinitionRepository

```java
@Component
public class RedisRouteDefinitionRepository
        implements RouteDefinitionLocator, RouteDefinitionWriter {

    private static final String ROUTE_KEY = "gateway:routes";
    private final ReactiveRedisTemplate<String, RouteDefinition> redisTemplate;

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return redisTemplate.opsForHash()
            .values(ROUTE_KEY)
            .cast(RouteDefinition.class);
    }

    @Override
    public Mono<Void> save(Mono<RouteDefinition> route) {
        return route.flatMap(r ->
            redisTemplate.opsForHash()
                .put(ROUTE_KEY, r.getId(), r)
                .then()
        );
    }

    @Override
    public Mono<Void> delete(Mono<String> routeId) {
        return routeId.flatMap(id ->
            redisTemplate.opsForHash()
                .remove(ROUTE_KEY, id)
                .then()
        );
    }

    @Bean
    public ReactiveRedisTemplate<String, RouteDefinition> routeRedisTemplate(
            ReactiveRedisConnectionFactory factory) {
        RedisSerializationContext<String, RouteDefinition> context =
            RedisSerializationContext
                .<String, RouteDefinition>newSerializationContext(
                    RedisSerializer.string())
                .hashKey(RedisSerializer.string())
                .hashValue(RedisSerializer.json())
                .build();
        return new ReactiveRedisTemplate<>(factory, context);
    }
}
```

# 6. 基于 Nacos 的动态路由

## 6.1 Nacos 监听器

```java
@Component
public class NacosRouteRefreshListener
        implements ApplicationEventPublisherAware, InitializingBean {

    private final ConfigService configService;
    private final RouteDefinitionLocator routeDefinitionLocator;
    private ApplicationEventPublisher publisher;

    private static final String DATA_ID = "gateway-routes.json";
    private static final String GROUP = "GATEWAY_GROUP";

    @Override
    public void afterPropertiesSet() throws Exception {
        // 添加配置监听器
        configService.addListener(DATA_ID, GROUP, new Listener() {
            @Override
            public Executor getExecutor() {
                return Executors.newSingleThreadExecutor();
            }

            @Override
            public void receiveConfigInfo(String configInfo) {
                // 配置变更时刷新路由
                log.info("Nacos config changed, refreshing routes...");
                publisher.publishEvent(new RefreshRoutesEvent(this));
            }
        });
    }
}
```

## 6.2 Nacos 配置示例

```json
// Nacos 配置: gateway-routes.json
[
  {
    "id": "user_service",
    "uri": "lb://user-service",
    "predicates": [{"name": "Path", "args": {"pattern": "/api/user/**"}}],
    "filters": [{"name": "StripPrefix", "args": {"parts": "2"}}],
    "order": 0,
    "metadata": {"response-timeout": 5000}
  },
  {
    "id": "order_service",
    "uri": "lb://order-service",
    "predicates": [{"name": "Path", "args": {"pattern": "/api/order/**"}}],
    "filters": [
      {"name": "StripPrefix", "args": {"parts": "2"}},
      {"name": "Retry", "args": {"retries": "2", "statuses": "BAD_GATEWAY"}}
    ],
    "order": 1
  }
]
```

# 7. 灰度发布路由

## 7.1 基于权重的动态灰度路由

```java
@Component
public class GrayReleaseRouteDefinitionLocator implements RouteDefinitionLocator {

    private final RouteDefinitionLocator delegate;
    private final DynamicGrayConfig grayConfig;

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return delegate.getRouteDefinitions()
            .concatWith(getGrayRouteDefinitions());
    }

    private Flux<RouteDefinition> getGrayRouteDefinitions() {
        // 动态读取灰度配置
        List<GrayRule> rules = grayConfig.getActiveGrayRules();
        if (rules.isEmpty()) return Flux.empty();

        return Flux.fromIterable(rules).map(this::buildGrayRoute);
    }

    private RouteDefinition buildGrayRoute(GrayRule rule) {
        RouteDefinition def = new RouteDefinition();
        def.setId("gray_" + rule.getServiceId());

        // 权重断言（虚拟分组，权重之和为 100）
        PredicateDefinition weightPredicate = new PredicateDefinition();
        weightPredicate.setName("Weight");
        weightPredicate.addArg("group", "gray_" + rule.getServiceId());
        weightPredicate.addArg("weight", String.valueOf(rule.getWeight()));
        def.getPredicates().add(weightPredicate);

        // 路径断言
        PredicateDefinition pathPredicate = new PredicateDefinition();
        pathPredicate.setName("Path");
        pathPredicate.addArg("pattern", "/" + rule.getServiceId() + "/**");
        def.getPredicates().add(pathPredicate);

        // 灰度版本的后端地址
        def.setUri(URI.create(rule.getGrayUri()));

        return def;
    }
}
```

# 8. 路由刷新安全策略

```java
@Configuration
@EnableWebFluxSecurity
public class RouteManagementSecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .authorizeExchange(exchanges -> exchanges
                // 路由管理端点需要 ADMIN 角色
                .pathMatchers("/actuator/gateway/routes/**")
                    .hasAuthority("SCOPE_admin")
                .pathMatchers("/admin/routes/**")
                    .hasAuthority("SCOPE_admin")
                .anyExchange().permitAll()
            )
            .httpBasic(Customizer.withDefaults())
            .build();
    }
}
```

# 9. 最佳实践

## 9.1 路由存储方案选择

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| InMemory | 简单、无需额外依赖 | 重启丢失、不支持多实例 | 开发测试 |
| Redis | 高性能、支持集群 | 需维护 Redis | 生产环境、多实例 |
| 数据库 | 持久化、可查询 | 性能较低 | 需要持久化和管理的生产环境 |
| Nacos/Apollo | 配置中心原生支持 | 增加配置中心依赖 | 已使用配置中心的团队 |

## 9.2 路由变更的原子性

```java
public class AtomicRouteUpdater {

    private final RouteDefinitionWriter writer;
    private final ApplicationEventPublisher publisher;

    /**
     * 批量更新路由（原子操作）
     */
    public Mono<Void> batchUpdate(List<RouteDefinition> newRoutes) {
        return Flux.fromIterable(newRoutes)
            .flatMap(route -> writer.save(Mono.just(route)))
            .then(Mono.fromRunnable(() ->
                publisher.publishEvent(new RefreshRoutesEvent(this))));
    }
}
```

## 9.3 注意事项

1. **缓存一致**：一定要发送 `RefreshRoutesEvent` 事件才能清除缓存
2. **路由验证**：动态添加路由前验证配置合法性，避免注入非法配置
3. **并发安全**：`RouteDefinitionWriter.save` 和 `refresh` 注意并发问题
4. **事务性**：批量更新路由时确保原子性
5. **权限控制**：路由管理端点需要严格认证和授权
