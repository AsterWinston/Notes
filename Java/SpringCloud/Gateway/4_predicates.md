
# 1. 断言体系架构

## 1.1 核心接口

```java
// 异步断言接口
public interface AsyncPredicate<T> {
    Publisher<Boolean> apply(T t);

    default AsyncPredicate<T> and(AsyncPredicate<? super T> other) {
        return t -> Mono.from(apply(t)).flatMap(b -> {
            if (!b) return Mono.just(false);
            return Mono.from(other.apply(t));
        });
    }

    default AsyncPredicate<T> or(AsyncPredicate<? super T> other) {
        return t -> Mono.from(apply(t)).flatMap(b -> {
            if (b) return Mono.just(true);
            return Mono.from(other.apply(t));
        });
    }

    default AsyncPredicate<T> negate() {
        return t -> Mono.from(apply(t)).map(b -> !b);
    }
}

// 断言工厂接口
public interface RoutePredicateFactory<C> extends ShortcutConfigurable, Configurable<C> {
    String NAME_KEY = "name";
    String PATTERN_KEY = "pattern";

    default AsyncPredicate<ServerWebExchange> applyAsync(C config) {
        return AsyncPredicate.from(apply(config));
    }

    Predicate<ServerWebExchange> apply(C config);
}

// 抽象基类
public abstract class AbstractRoutePredicateFactory<C> implements RoutePredicateFactory<C> {
    private final Class<C> configClass;

    public AbstractRoutePredicateFactory(Class<C> configClass) {
        this.configClass = configClass;
    }

    @Override
    public C newConfig() {
        return BeanUtils.instantiateClass(this.configClass);
    }
}
```

## 1.2 工厂命名规范

断言工厂的命名遵循约定：`XxxRoutePredicateFactory` → 配置时使用 `Xxx`（去掉 `RoutePredicateFactory` 后缀）。

```yaml
predicates:
  - Path=/api/**                          # PathRoutePredicateFactory
  - Method=GET                            # MethodRoutePredicateFactory
  - Header=X-Id, \d+                      # HeaderRoutePredicateFactory
  - Cookie=sessionId, \w+                 # CookieRoutePredicateFactory
```

# 2. 内置断言工厂详解

## 2.1 PathRoutePredicateFactory

**作用**：根据请求路径匹配，支持 Ant 风格路径模式和 URI 模板变量提取。

```yaml
predicates:
  - Path=/api/{segment}/user/{userId}
```

```java
public class PathRoutePredicateFactory
        extends AbstractRoutePredicateFactory<PathRoutePredicateFactory.Config> {

    public static class Config {
        private List<String> patterns = new ArrayList<>(); // 路径模式列表
        private boolean matchTrailingSlash = true;         // 是否匹配尾部斜杠
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        // 使用 Spring PathMatcher 匹配
        PathPatternParser parser = new PathPatternParser();
        parser.setMatchOptionalTrailingSeparator(config.isMatchTrailingSlash());
        List<PathPattern> patterns = config.getPatterns().stream()
            .map(parser::parse)
            .collect(Collectors.toList());

        return exchange -> {
            PathContainer path = exchange.getRequest().getPath().pathWithinApplication();
            // 优先匹配精确路径
            Optional<PathPattern> match = patterns.stream()
                .filter(pattern -> pattern.matches(path))
                .findFirst();

            if (match.isPresent()) {
                // 将 URI 模板变量放入 exchange 属性
                PathPattern.PathMatchInfo pathMatchInfo = match.get().matchAndExtract(path);
                if (pathMatchInfo != null) {
                    Map<String, String> uriVariables = pathMatchInfo.getUriVariables();
                    exchange.getAttributes().put(
                        ServerWebExchangeUtils.URI_TEMPLATE_VARIABLES_ATTRIBUTE,
                        uriVariables);
                }
                return true;
            }
            return false;
        };
    }
}
```

**提取路径变量**：

```java
@GetMapping("/users/{id}")
public Mono<ServerResponse> getUser(ServerWebExchange exchange) {
    // 在路由过滤器中读取路径变量
    Map<String, String> pathVariables = exchange.getAttribute(
        ServerWebExchangeUtils.URI_TEMPLATE_VARIABLES_ATTRIBUTE);
    String userId = pathVariables.get("userId");
    // ...
}
```

## 2.2 MethodRoutePredicateFactory

**作用**：根据 HTTP 方法匹配。

```yaml
predicates:
  - Method=GET,POST,PUT,DELETE
```

```java
public class MethodRoutePredicateFactory
        extends AbstractRoutePredicateFactory<MethodRoutePredicateFactory.Config> {

    public static class Config {
        private List<HttpMethod> methods = new ArrayList<>();
        // getMethods/setMethods
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> config.getMethods().contains(
            exchange.getRequest().getMethod());
    }
}
```

## 2.3 HeaderRoutePredicateFactory

**作用**：根据请求 Header 的名称和值（支持正则）匹配。

```yaml
predicates:
  - Header=X-Request-Id, \d+                        # 数字格式的 ID
  - Header=X-Version, v(1|2)                        # 只匹配 v1 或 v2
  - Header=Authorization, Bearer\s\w+\.\w+\.\w+\s*$ # JWT 格式
```

```java
public class HeaderRoutePredicateFactory
        extends AbstractRoutePredicateFactory<HeaderRoutePredicateFactory.Config> {

    public static class Config {
        private String header;    // Header 名称
        private String regexp;    // 值的正则表达式
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> {
            String value = exchange.getRequest().getHeaders()
                .getFirst(config.getHeader());
            if (value != null) {
                return Pattern.matches(config.getRegexp(), value);
            }
            return false;
        };
    }
}
```

## 2.4 CookieRoutePredicateFactory

**作用**：根据 Cookie 名称和值（支持正则）匹配。

```yaml
predicates:
  - Cookie=sessionId, [a-f0-9]{32}       # 32 位十六进制 session ID
  - Cookie=language, zh-CN|en-US         # 语言偏好
```

```java
public class CookieRoutePredicateFactory
        extends AbstractRoutePredicateFactory<CookieRoutePredicateFactory.Config> {

    public static class Config {
        private String name;    // Cookie 名称
        private String regexp;  // Cookie 值的正则
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> {
            return exchange.getRequest().getCookies()
                .getFirst(config.getName())
                .map(cookie -> Pattern.matches(config.getRegexp(), cookie.getValue()))
                .orElse(false);
        };
    }
}
```

## 2.5 QueryRoutePredicateFactory

**作用**：根据查询参数名称和值匹配。

```yaml
predicates:
  - Query=page                          # 只要存在 page 参数
  - Query=size, \d+                     # size 参数存在且为数字
```

```java
public class QueryRoutePredicateFactory
        extends AbstractRoutePredicateFactory<QueryRoutePredicateFactory.Config> {

    public static class Config {
        private String param;    // 参数名
        private String regexp;  // 参数值的正则（可选）
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> {
            MultiValueMap<String, String> queryParams =
                exchange.getRequest().getQueryParams();
            String value = queryParams.getFirst(config.getParam());

            if (value == null) return false;
            if (config.getRegexp() == null) return true;
            return Pattern.matches(config.getRegexp(), value);
        };
    }
}
```

## 2.6 HostRoutePredicateFactory

**作用**：根据请求的 `Host` Header 匹配，支持 Ant 风格模式。

```yaml
predicates:
  - Host=**.example.com                 # 匹配所有 example.com 子域名
  - Host=api.example.com                # 精确匹配
  - Host=*.example.com:8080             # 匹配指定端口
```

```java
public class HostRoutePredicateFactory
        extends AbstractRoutePredicateFactory<HostRoutePredicateFactory.Config> {

    public static class Config {
        private List<String> patterns = new ArrayList<>();
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> {
            String host = exchange.getRequest().getHeaders()
                .getFirst(HttpHeaders.HOST);
            if (host != null) {
                for (String pattern : config.getPatterns()) {
                    if (PathMatcher.match(pattern, host)) {
                        // 提取模式变量
                        Map<String, String> variables =
                            PathMatcher.extractUriTemplateVariables(pattern, host);
                        exchange.getAttributes().put(
                            ServerWebExchangeUtils.URI_TEMPLATE_VARIABLES_ATTRIBUTE,
                            variables);
                        return true;
                    }
                }
            }
            return false;
        };
    }
}
```

## 2.7 RemoteAddrRoutePredicateFactory

**作用**：根据客户端 IP 地址（CIDR 格式）匹配。

```yaml
predicates:
  - RemoteAddr=192.168.1.0/24           # 内网段
  - RemoteAddr=10.0.0.0/8               # 大 A 类网段
  - RemoteAddr=203.0.113.0/24,198.51.100.0/24  # 多个网段
```

```java
public class RemoteAddrRoutePredicateFactory
        extends AbstractRoutePredicateFactory<RemoteAddrRoutePredicateFactory.Config> {

    public static class Config {
        private List<String> sources = new ArrayList<>();
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        List<IpSubnetFilterRule> ipSubnetFilterRules = config.getSources().stream()
            .map(source -> {
                String[] parts = source.split("/", 2);
                return new IpSubnetFilterRule(parts[0],
                    Integer.parseInt(parts[1]),
                    IpFilterRuleType.ACCEPT);
            }).collect(Collectors.toList());

        return exchange -> {
            InetSocketAddress remoteAddr =
                exchange.getRequest().getRemoteAddress();
            if (remoteAddr != null) {
                InetAddress address = remoteAddr.getAddress();
                for (IpSubnetFilterRule rule : ipSubnetFilterRules) {
                    if (rule.matches(address)) {
                        return true;
                    }
                }
            }
            return false;
        };
    }
}
```

## 2.8 WeightRoutePredicateFactory

**作用**：按权重分配流量到不同的路由，用于灰度发布、A/B 测试。

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: canary_v1
          uri: http://v1.example.com
          predicates:
            - Path=/api/**
            - Weight=canary, 80
        - id: canary_v2
          uri: http://v2.example.com
          predicates:
            - Path=/api/**
            - Weight=canary, 20
```

**实现原理**：

```java
public class WeightRoutePredicateFactory
        extends AbstractRoutePredicateFactory<WeightRoutePredicateFactory.Config> {

    // 维护一个全局的 Weight 组状态
    private static final Map<String, AtomicInteger> GROUP_COUNTER = new HashMap<>();

    public static class Config {
        private String group;   // 分组名称
        private int weight;     // 权重（百分比）
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        String group = config.getGroup();
        int weight = config.getWeight();

        return exchange -> {
            // 原子递增计数器
            AtomicInteger counter = GROUP_COUNTER
                .computeIfAbsent(group, k -> new AtomicInteger(0));
            int current = counter.getAndIncrement() % 100;
            return current < weight;
        };
    }
}
```

**注意事项**：
- 同一组的路由权重之和应为 100
- 权重是软路由，依赖计数器实现近似分配
- 适合灰度发布场景，不适合精确流量控制

## 2.9 AfterRoutePredicateFactory / BeforeRoutePredicateFactory / BetweenRoutePredicateFactory

**作用**：根据时间范围匹配请求，适用于定时切换、蓝绿部署。

```yaml
predicates:
  # 2024-01-01 08:00:00 之后生效（美国丹佛时区）
  - After=2024-01-01T08:00:00-07:00[America/Denver]

  # 2024-06-30 18:00:00 之前生效
  - Before=2024-06-30T18:00:00+08:00[Asia/Shanghai]

  # 在指定时间段内生效
  - Between=2024-03-01T00:00:00+08:00[Asia/Shanghai],
            2024-03-31T23:59:59+08:00[Asia/Shanghai]
```

```java
public class AfterRoutePredicateFactory
        extends AbstractRoutePredicateFactory<AfterRoutePredicateFactory.Config>
        implements ZonedDateTimeRoutePredicateFactory {

    public static class Config {
        private ZonedDateTime datetime;
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        ZonedDateTime cutoff = config.getDatetime();
        return exchange -> ZonedDateTime.now(cutoff.getZone())
            .isAfter(cutoff);
    }
}
```

**实用场景**：
- 新版本上线后的灰度观察期
- 大促活动期间的流量切换
- 定时降级非核心服务

## 2.10 ReadBodyRoutePredicateFactory

**作用**：读取请求体内容并进行断言判断。注意这是一次性读取，消费后需重新缓存。

```yaml
predicates:
  - name: ReadBody
    args:
      inClass: java.lang.String
      predicate: "#{@bodyPredicate}"
```

```java
@Bean
public Predicate<String> bodyPredicate() {
    return body -> {
        // 解析 body 并判断
        return body.contains("expectedValue");
    };
}
```

**源码要点**：

```java
public class ReadBodyRoutePredicateFactory
        extends AbstractRoutePredicateFactory<ReadBodyRoutePredicateFactory.Config> {

    public static class Config {
        private Class<?> inClass;           // 请求体类型
        private Predicate<Object> predicate; // 断言逻辑
    }

    @Override
    public AsyncPredicate<ServerWebExchange> applyAsync(Config config) {
        return exchange -> {
            // 从 exchange 获取或读取 body
            return ServerWebExchangeUtils
                .cacheRequestBodyAndRequest(exchange, cachedBody -> {
                    // 反序列化 body
                    Object body = deserialize(cachedBody, config.getInClass());
                    // 执行断言
                    return Mono.just(config.getPredicate().test(body));
                });
        };
    }
}
```

## 2.11 XForwardedRemoteAddrRoutePredicateFactory

**作用**：基于 `X-Forwarded-For` 头中的客户端真实 IP（需在反向代理场景下使用）。

```yaml
predicates:
  - XForwardedRemoteAddr=10.0.0.0/8,192.168.0.0/16
```

# 3. 断言组合策略

## 3.1 AND 组合（默认）

所有断言同时满足：

```yaml
predicates:
  - Path=/api/**
  - Method=GET
  - Header=X-Version, v1
```

等价于：`Path` AND `Method` AND `Header`

## 3.2 Java DSL 中的组合

```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("complex_route", r -> r
            // (Path=/api/** AND Method=GET) OR (Path=/health/**)
            .path("/api/**").and().method("GET")
            .or()
            .path("/health/**")
            .uri("http://backend"))
        .build();
}
```

**可用的逻辑操作**：

```java
.and()      // 逻辑 AND
.or()       // 逻辑 OR
.negate()   // 逻辑 NOT（对当前断言取反）
```

## 3.3 断言优先级

同一条路由内的多个断言按定义顺序从左到右短路执行，所以将区分度高的断言放在前面可以提高匹配效率：

```yaml
predicates:
  - Method=POST        # 先判断方法（开销小，区分度高）
  - Path=/api/order/** # 再判断路径
  - Header=X-Auth, .+  # 最后判断 Header
```

# 4. 自定义断言工厂

## 4.1 自定义请求头版本断言

```java
@Component
public class VersionRoutePredicateFactory
        extends AbstractRoutePredicateFactory<VersionRoutePredicateFactory.Config> {

    public VersionRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> {
            String version = exchange.getRequest().getHeaders()
                .getFirst("X-API-Version");
            return config.getVersions().contains(version);
        };
    }

    @Override
    public ShortcutType shortcutType() {
        // 支持快捷配置: Version=v1,v2
        return ShortcutType.GATHER_LIST;
    }

    public static class Config {
        private List<String> versions = new ArrayList<>();

        public List<String> getVersions() { return versions; }
        public void setVersions(List<String> versions) { this.versions = versions; }
    }
}
```

**使用**：

```yaml
predicates:
  - Version=v1,v2
```

## 4.2 自定义时间范围断言

```java
@Component
public class TimeRangeRoutePredicateFactory
        extends AbstractRoutePredicateFactory<TimeRangeRoutePredicateFactory.Config> {

    public TimeRangeRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> {
            LocalTime now = LocalTime.now();
            return !now.isBefore(config.getStart()) && !now.isAfter(config.getEnd());
        };
    }

    public static class Config {
        private LocalTime start = LocalTime.of(0, 0);
        private LocalTime end = LocalTime.of(23, 59);

        // getters & setters
    }
}
```

**使用**：

```yaml
predicates:
  - name: TimeRange
    args:
      start: "09:00"
      end: "18:00"
```

## 4.3 自定义 ip 范围断言

```java
@Component
public class IpRangeRoutePredicateFactory
        extends AbstractRoutePredicateFactory<IpRangeRoutePredicateFactory.Config> {

    public IpRangeRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> {
            String ip = exchange.getRequest().getRemoteAddress()
                .getAddress().getHostAddress();
            return config.getIps().contains(ip);
        };
    }

    public static class Config {
        private List<String> ips = new ArrayList<>();
        // getters & setters
    }
}
```

**使用**：

```yaml
predicates:
  - name: IpRange
    args:
      ips: 192.168.1.100,10.0.0.50
```

# 5. 断言的最佳实践

## 5.1 断言选择策略

| 场景 | 推荐的断言 | 原因 |
|------|-----------|------|
| RESTful API 路由 | `Path` + `Method` | 精准区分不同资源 |
| 多版本共存 | `Header`（X-Version） | 版本号通过 Header 传递 |
| 灰度发布 | `Weight` | 按百分比分配流量 |
| 定时任务 | `After`/`Before`/`Between` | 精确时间控制 |
| 内部系统 | `RemoteAddr`/`X-Forwarded-For` | IP 范围限制 |
| A/B 测试 | `Cookie`/`Header` | 基于用户标识分流 |

## 5.2 性能优化

1. **简单断言在前**：将执行开销小的断言放在前面（如 `Method` 优于 `ReadBody`）
2. **避免 ReadBody 过度使用**：读取 Body 会消费流，如需读取 Body 的断言应配合 `CacheRequestBodyFilter`
3. **Weight 断言的原子性**：多实例部署时，Weight 路由的计数器需要跨进程处理，否则权重分配不精确

## 5.3 调试建议

```yaml
logging:
  level:
    org.springframework.cloud.gateway.handler.predicate: TRACE
```

开启 TRACE 日志后，可以看到每个断言的匹配结果，便于排查路由不生效的问题。
