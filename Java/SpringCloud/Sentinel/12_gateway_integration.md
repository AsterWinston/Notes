
# 1. 概述

Sentinel 提供了专为 Spring Cloud Gateway 设计的适配模块，支持**路由级别**和**API 分组级别**的流控与降级。

```
HTTP 请求 → Gateway
  → SentinelGatewayFilter（全局）
    → 路由匹配
    → Sentinel 检查
      → 通过 → 转发后端
      → 拒绝 → 返回 JSON 错误
```

# 2. 依赖引入

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>

<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

**注意**：Gateway 集成与 Spring MVC 集成**互斥**。Gateway 场景下不要引入 `spring-boot-starter-web`。

# 3. 配置

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
      eager: true
      # Gateway 专用配置
      scg:
        fallback:
          mode: response   # response（直接返回）| redirect（重定向）
          response-body: '{"code":429,"message":"请求过于频繁"}'
          response-status: 429
          content-type: application/json
```

# 4. 路由级限流

Sentinel Gateway 自动为每个路由创建资源，资源名格式为路由 ID：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=1
```

在 Dashboard 中针对资源名 `user-service` 配置流控规则，即可对 `/api/user/**` 路由限流。

# 5. API 分组级限流

API 分组支持跨路由的组合限流：

```java
@Configuration
public class GatewaySentinelConfig {

    @PostConstruct
    public void initApiDefinitions() {
        Set<ApiDefinition> definitions = new HashSet<>();

        // API 分组：包含多个路径
        ApiDefinition userApi = new ApiDefinition("user_api_group")
            .setPredicateItems(new HashSet<>(Arrays.asList(
                new ApiPredicateItem().setPattern("/api/user/**")
                                       .setMatchStrategy(SentinelGatewayConstants.URL_MATCH_STRATEGY_PREFIX),
                new ApiPredicateItem().setPattern("/api/profile/**")
                                       .setMatchStrategy(SentinelGatewayConstants.URL_MATCH_STRATEGY_PREFIX)
            )));
        definitions.add(userApi);

        GatewayApiDefinitionManager.loadApiDefinitions(definitions);

        // 为 API 分组配置流控规则
        Set<GatewayFlowRule> rules = new HashSet<>();
        rules.add(new GatewayFlowRule("user_api_group")
            .setCount(100)
            .setIntervalSec(1)
            .setResourceMode(SentinelGatewayConstants.RESOURCE_MODE_CUSTOM_API_NAME));
        
        GatewayRuleManager.loadRules(rules);
    }
}
```

# 6. 自定义限流参数

```java
@Configuration
public class GatewayParamConfig {

    @PostConstruct
    public void initParamRules() {
        Set<GatewayFlowRule> rules = new HashSet<>();

        // 按 IP 限流
        GatewayFlowRule ipRule = new GatewayFlowRule("user-service")
            .setCount(10)
            .setIntervalSec(1)
            .setParamItem(new GatewayParamFlowItem()
                .setParseStrategy(SentinelGatewayConstants.PARAM_PARSE_STRATEGY_CLIENT_IP));

        // 按 Header 限流
        GatewayFlowRule headerRule = new GatewayFlowRule("user-service")
            .setCount(50)
            .setIntervalSec(1)
            .setParamItem(new GatewayParamFlowItem()
                .setParseStrategy(SentinelGatewayConstants.PARAM_PARSE_STRATEGY_HEADER)
                .setFieldName("X-User-Id"));

        // 按 URL 参数限流
        GatewayFlowRule paramRule = new GatewayFlowRule("user-service")
            .setCount(100)
            .setIntervalSec(1)
            .setParamItem(new GatewayParamFlowItem()
                .setParseStrategy(SentinelGatewayConstants.PARAM_PARSE_STRATEGY_URL_PARAM)
                .setFieldName("categoryId"));

        rules.addAll(Arrays.asList(ipRule, headerRule, paramRule));
        GatewayRuleManager.loadRules(rules);
    }
}
```

# 7. 自定义降级响应

```java
@Configuration
public class GatewayFallbackConfig {

    @PostConstruct
    public void initBlockHandlers() {
        BlockRequestHandler blockHandler = (exchange, ex) -> {
            Map<String, Object> result = new HashMap<>();
            if (ex instanceof FlowException) {
                result.put("code", 429);
                result.put("message", "请求过于频繁");
            } else if (ex instanceof DegradeException) {
                result.put("code", 503);
                result.put("message", "服务已熔断");
            } else {
                result.put("code", 500);
                result.put("message", "系统保护已触发");
            }

            exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);
            
            byte[] bytes = new ObjectMapper().writeValueAsBytes(result);
            return exchange.getResponse()
                .writeWith(Mono.just(exchange.getResponse().bufferFactory().wrap(bytes)));
        };

        GatewayCallbackManager.setBlockHandler(blockHandler);
    }
}
```

# 8. 规则持久化

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        gw-flow:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-gw-flow-rules
            group-id: SENTINEL_GROUP
            rule-type: gw-flow
        gw-api-group:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-gw-api-groups
            group-id: SENTINEL_GROUP
            rule-type: gw-api-group
```

# 9. 与 Spring MVC Sentinel 的区别

| 维度 | Spring MVC | Spring Cloud Gateway |
|------|-----------|---------------------|
| 拦截方式 | `SentinelWebInterceptor` | `SentinelGatewayFilter` |
| 资源粒度 | URL Pattern | Route ID / API Group |
| 参数限流 | `@SentinelResource` | `GatewayParamFlowItem` |
| 降级响应 | `@ExceptionHandler` | `BlockRequestHandler` |
| 编程模型 | 同步阻塞 | 响应式（Reactor） |
| 依赖冲突 | — | 不能与 spring-boot-starter-web 共存 |

# 10. 最佳实践

1. **路由 ID 规范**：用 `{服务名}-{功能}` 命名路由，便于在 Dashboard 识别
2. **API 分组策略**：按业务域（用户、订单、商品）划分 API 分组
3. **多维度限流**：路由级 + API 分组级 + IP/Header/Param 级叠用
4. **降级响应标准化**：统一 JSON 格式，包含 code/message/timestamp
5. **Gateway 监控**：配合 `SentinelGatewayFilter` 的 Metrics 在 Dashboard 查看网关级流量
