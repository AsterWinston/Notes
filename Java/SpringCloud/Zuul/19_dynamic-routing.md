
# 1. 动态路由概述

Zuul 的路由信息存在 `SimpleRouteLocator` 中，默认情况下路由在应用启动时加载，运行期间无法动态修改。动态路由的需求包括：

- **热加载路由规则**：不重启网关即可增减路由
- **配置中心驱动**：通过 Nacos、Apollo 等配置中心推送路由变更
- **自定义路由存储**：从数据库、Redis 等外部存储读取路由

# 2. RefreshableRouteLocator 接口

Spring Cloud 为动态路由提供了 `RefreshableRouteLocator` 接口：

```java
public interface RefreshableRouteLocator extends RouteLocator {
    // 刷新路由
    void refresh();
}
```

它继承自 `RouteLocator`，增加了 `refresh()` 方法。配合 Spring Cloud 的 `@RefreshScope` 或配置中心的刷新机制，可以实现路由的热更新。

# 3. 基于 Spring Cloud Bus 的路由刷新

## 3.1 配置中心 + Bus 架构

```
配置中心 (Config Server)
    │
    ├── 推送变更
    │
    ▼
Spring Cloud Bus (消息总线)
    │
    ├── RabbitMQ / Kafka
    │
    ▼
Zuul 网关实例 (多实例同时收到刷新通知)
```

## 3.2 依赖引入

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

## 3.3 配置

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/example/config-repo
      bus:
        enabled: true
        refresh:
          enabled: true

management:
  endpoints:
    web:
      exposure:
        include: bus-refresh,refresh,routes
```

## 3.4 手动触发刷新

```bash
# 触发单机路由刷新
curl -X POST http://zuul-gateway:8080/actuator/refresh

# 触发集群路由刷新（通过 Bus）
curl -X POST http://zuul-gateway:8080/actuator/bus-refresh
```

## 3.5 Zuul 监听路由变更源码

```java
// ZuulServerAutoConfiguration 中注册的刷新监听器
@Configuration
public class ZuulRefreshListenerConfiguration {
    @Bean
    @ConditionalOnBean(ZuulServerMarkerConfiguration.Marker.class)
    public ZuulRefreshListener zuulRefreshListener() {
        return new ZuulRefreshListener();
    }
}

// 监听 RefreshScopeRefreshedEvent 事件
public class ZuulRefreshListener implements ApplicationListener<ApplicationEvent> {
    @Autowired
    private ZuulHandlerMapping zuulHandlerMapping;

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof RefreshScopeRefreshedEvent
                || event instanceof RoutesRefreshedEvent) {
            // 重置 ZuulHandlerMapping，强制重新加载路由
            this.zuulHandlerMapping.setDirty(true);
        }
    }
}
```

当配置中心推送变更后，Spring Cloud 框架会发出 `RefreshScopeRefreshedEvent` 事件，`ZuulRefreshListener` 监听到事件后，通知 `ZuulHandlerMapping` 重新加载路由表。

## 3.6 动态路由的局限性

基于 Bus + Config 的动态路由更新的是 `ZuulProperties` 中的属性值（即 `zuul.routes.*` 中的配置）。这意味着被替换的路由规则会丢失，而新增的规则需要另外的机制管理。对于复杂的动态路由场景，更推荐自定义 RouteLocator。

# 4. 自定义 DynamicRouteLocator

## 4.1 从数据库加载路由

```java
@Component
public class DatabaseRouteLocator extends SimpleRouteLocator
        implements RefreshableRouteLocator {

    private final JdbcTemplate jdbcTemplate;

    // 数据库表结构：zuul_route(id, path, url, service_id, strip_prefix, enabled)
    public DatabaseRouteLocator(ServerProperties server,
                                 ZuulProperties properties,
                                 JdbcTemplate jdbcTemplate) {
        super(server.getServlet().getContextPath(), properties);
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public void refresh() {
        // 重新加载路由（从数据库重新读取）
        doRefresh();
    }

    @Override
    protected Map<String, ZuulRoute> locateRoutes() {
        // 合并数据库路由和配置文件路由
        Map<String, ZuulRoute> routes = new LinkedHashMap<>();
        routes.putAll(super.locateRoutes());  // 配置文件中的路由

        // 数据库中的路由（优先级高于配置文件）
        List<ZuulRoute> dbRoutes = jdbcTemplate.query(
            "SELECT * FROM zuul_route WHERE enabled = 1",
            (rs, rowNum) -> {
                ZuulRoute route = new ZuulRoute();
                route.setId(rs.getString("id"));
                route.setPath(rs.getString("path"));
                route.setUrl(rs.getString("url"));
                route.setServiceId(rs.getString("service_id"));
                route.setStripPrefix(rs.getBoolean("strip_prefix"));
                return route;
            }
        );

        for (ZuulRoute route : dbRoutes) {
            routes.put(route.getId(), route);
        }

        return routes;
    }

    // 暴露刷新端点
    public void refreshRoute() {
        this.reset();  // 清空缓存 → 下次请求时重新调用 locateRoutes()
    }
}
```

## 4.2 提供 REST API 管理路由

```java
@RestController
@RequestMapping("/admin/routes")
public class RouteManageController {

    @Autowired
    private JdbcTemplate jdbcTemplate;
    @Autowired
    private DatabaseRouteLocator routeLocator;

    // 查看所有路由
    @GetMapping
    public List<Map<String, Object>> listRoutes() {
        return jdbcTemplate.queryForList(
            "SELECT * FROM zuul_route ORDER BY id");
    }

    // 新增路由
    @PostMapping
    public String addRoute(@RequestBody ZuulRoute route) {
        jdbcTemplate.update(
            "INSERT INTO zuul_route(id, path, url, service_id, " +
            "strip_prefix, enabled) VALUES (?,?,?,?,?,?)",
            route.getId(), route.getPath(), route.getUrl(),
            route.getServiceId(), route.isStripPrefix(), 1
        );
        routeLocator.refresh();
        return "路由已添加并生效";
    }

    // 删除路由
    @DeleteMapping("/{id}")
    public String deleteRoute(@PathVariable String id) {
        jdbcTemplate.update(
            "UPDATE zuul_route SET enabled = 0 WHERE id = ?", id);
        routeLocator.refresh();
        return "路由已禁用";
    }

    // 刷新路由
    @PostMapping("/refresh")
    public String refresh() {
        routeLocator.refresh();
        return "路由已刷新";
    }
}
```

## 4.3 数据库表设计

```sql
CREATE TABLE zuul_route (
    id          VARCHAR(128) PRIMARY KEY COMMENT '路由ID',
    path        VARCHAR(256) NOT NULL COMMENT '请求路径(Ant风格)',
    url         VARCHAR(256) COMMENT '物理URL',
    service_id  VARCHAR(128) COMMENT '服务ID(Eureka)',
    strip_prefix TINYINT DEFAULT 1 COMMENT '是否剥离前缀',
    retryable   TINYINT DEFAULT 0 COMMENT '是否可重试',
    enabled     TINYINT DEFAULT 1 COMMENT '是否启用',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_enabled (enabled)
) COMMENT='Zuul动态路由表';
```

# 5. 基于 Redis 的轻量级动态路由

```java
@Component
public class RedisRouteLocator extends SimpleRouteLocator
        implements RefreshableRouteLocator {

    private final StringRedisTemplate redisTemplate;
    private static final String ROUTE_KEY = "zuul:routes";

    public RedisRouteLocator(ServerProperties server,
                              ZuulProperties properties,
                              StringRedisTemplate redisTemplate) {
        super(server.getServlet().getContextPath(), properties);
        this.redisTemplate = redisTemplate;
    }

    @Override
    public void refresh() {
        this.reset();  // 清空本地缓存
    }

    @Override
    protected Map<String, ZuulRoute> locateRoutes() {
        Map<String, ZuulRoute> routes = new LinkedHashMap<>();

        // 从 Redis 加载路由配置
        Map<Object, Object> redisRoutes = redisTemplate.opsForHash()
            .entries(ROUTE_KEY);

        for (Map.Entry<Object, Object> entry : redisRoutes.entrySet()) {
            String routeJson = (String) entry.getValue();
            try {
                ZuulRoute route = new ObjectMapper()
                    .readValue(routeJson, ZuulRoute.class);
                routes.put(entry.getKey().toString(), route);
            } catch (JsonProcessingException e) {
                log.error("解析路由配置失败: {}", routeJson, e);
            }
        }

        // 合并配置文件中的静态路由
        routes.putAll(super.locateRoutes());

        return routes;
    }
}
```

# 6. 基于 Groovy 脚本的动态过滤器（附加）

除了动态路由，Zuul 还支持动态加载 Groovy 过滤器：

```yaml
zuul:
  filter:
    root: classpath:groovy/filters
    interval: 30  # 每 30 秒扫描一次变更
```

```groovy
// groovy/filters/DynamicFilter.groovy
class DynamicFilter extends com.netflix.zuul.ZuulFilter {
    @Override
    String filterType() { return "pre" }

    @Override
    int filterOrder() { return 20 }

    @Override
    boolean shouldFilter() {
        return com.netflix.zuul.context.RequestContext
            .getCurrentContext().sendZuulResponse()
    }

    @Override
    Object run() {
        println "[Dynamic Groovy Filter] Request: " +
            com.netflix.zuul.context.RequestContext
                .getCurrentContext().request.requestURI
        return null
    }
}
```

Groovy 过滤器热更新的原理是 `FilterLoader` 定期检查文件修改时间，发现变更后重新编译并注册到 `FilterRegistry`。

# 7. 自定义 RouteManager

完整的动态路由管理方案：

```java
@Component
public class RouteManager {

    @Autowired
    private DatabaseRouteLocator routeLocator;
    @Autowired
    private ZuulHandlerMapping zuulHandlerMapping;

    // 原子性更新路由
    @Transactional
    public void updateRoute(ZuulRoute route) {
        // 1. 更新数据库
        jdbcTemplate.update(...);

        // 2. 刷新 RouteLocator
        routeLocator.refresh();

        // 3. 刷新 HandlerMapping
        zuulHandlerMapping.setDirty(true);

        // 4. 发布路由变更事件
        applicationContext.publishEvent(
            new RoutesRefreshedEvent(routeLocator));
    }

    // 批量导入路由
    @Transactional
    public void importRoutes(List<ZuulRoute> routes) {
        // 1. 开启事务
        // 2. 批量写入数据库
        // 3. 一次性刷新路由表
        // 4. 发布事件
    }
}
```

# 8. 注意事项

1. **灰度变更**：生产环境更新路由时，建议先在一台网关实例上验证，确认无误后再全量推送
2. **回滚能力**：每次变更前保存当前路由快照，异常时快速回滚
3. **变更审计**：记录谁在什么时间做了什么路由变更
4. **并发冲突**：多个管理员同时修改路由时，使用分布式锁或乐观锁
5. **路由缓存**：动态路由的 `locateRoutes()` 会在每次匹配时调用，对于数据库方案需自行实现缓存，避免频繁查库
