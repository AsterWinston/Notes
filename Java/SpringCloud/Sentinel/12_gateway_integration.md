# 1. 概述

Sentinel 提供了专为 Spring Cloud Gateway 设计的适配模块，支持**路由级别**和**API 分组级别**的流控与降级。

```text
HTTP 请求 → Gateway
  → SentinelGatewayFilter（全局）
    → 路由匹配
    → Sentinel 检查
      → 通过 → 转发后端
      → 拒绝 → 返回 JSON 错误
```

## 1.1 网关侧防护的独特立场：一层"总闸"+ 两个资源层级

网关是流量的唯一入口，这里的规则天然是**全局语义**（所有下游实例的请求都先过这道闸），单点阈值即总量阈值——这是网关限流比应用内单机限流更"可信"的根本原因，也是它的双刃剑：网关自身成为故障放大器，规则误配全域遭殃。资源模型先立框架：每个请求会同时建立**路由（route id）维度的 Entry** 与命中的**自定义 API 分组维度**的 Entry（§5），`GatewayFlowRule.resourceMode` 决定规则挂哪一个；**熔断降级不走 GatewayFlowRule**——它对 DegradeRule 同样透明，资源名写 route id 或分组名即可（§4.1），这是本篇最常被问倒的一条。

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

## 2.1 互斥的原因与自动装配差异

互斥不是建议而是硬约束：spring-cloud-starter-gateway 基于 WebFlux/Netty，`spring-boot-starter-web` 会把应用拉进 Servlet 栈，Boot 的 `WebApplicationType` 推导冲突后行为不可预期（典型症状：启动报 bean 缺失或网关路由不生效）。自动配置类也不同名代：`SentinelSCGAutoConfiguration` 注册 `SentinelGatewayFilter`（GlobalFilter，order 靠前）与 `SentinelGatewayBlockExceptionHandler`（排在框架默认错误处理之前，保证被拦请求的响应形态由 §7 的 handler 说了算）；`spring-cloud-starter-alibaba-sentinel` 主 starter 仍负责数据源/心跳/Dashboard 接线——**三个依赖各司其职，少 gateway 那个 adapter 则规则配了也没人消费**（`8_hotspot_param.md` §1.1 的"依赖决定进链"同款逻辑）。

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

## 3.1 scg.fallback 与代码级 BlockHandler 的覆盖关系

这组属性只是**默认块响应**的配置化表达，装配时它会构造一个 `DefaultBlockRequestHandler` 注册进 `GatewayCallbackManager`；一旦你按 §7 用 `GatewayCallbackManager.setBlockHandler(...)` 手设处理器，**yml 这组即失效**（后写覆盖，无合并）——两处都配时以代码为准，排障先看注册顺序。redirect 模式的定位是给"被限后跳排队页"的传统 Web 场景，API 网关基本只用 response。

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

## 4.1 路由名规则的两个隐藏点

其一，Dashboard 网关版页面（路由流控/API 分组）与默认控制台 UI 不是同一套菜单——官方 dashboard 需配合 gateway 规则展示页；不少团队直接把网关当普通客户端看，在"簇点链路"里对 `user-service` 这个资源名配**普通 FlowRule** 也能生效（Entry 是真资源，`2_architecture.md` §2.4 通用），两套配法等价于规则类型选择：`GatewayFlowRule`（支持窗口/参数项）vs `FlowRule`（支持熔断联动生态），只配其一即可，双配就是双闸串联。其二，熔断不需要 Gateway 专用规则：对 route id 配**普通 DegradeRule**（异常比例/慢调用比例，`5_circuit_breaker.md` 全套语义）即可实现"下游变慢时网关快速失败"——网关侧 RT 含转发与下游全程，count 阈值要按端到端 P99 定，这比下游各自的熔断更早反映用户体感。

## 4.2 intervalSec：网关规则独有的窗口粒度

`GatewayFlowRule.count` 的语义是"**每 intervalSec 秒** count 个"——普通 FlowRule 固定 1s 窗在这里被参数化（默认 1），适合"每分钟 100 次"这类配额型治理，无需把 1.66 QPS 折算成小数 count。窗口越大响应越钝（`5_circuit_breaker.md` §5.1 同理），配额与止血两种需求别混用一个资源的双窗口规则。

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

## 5.1 分组模型的三个使用账

分组解决的是"路由切分与业务域不重合"的矛盾：一条 Path=/api/** 大路由下想单限"用户域"，或跨多条路由合并限流同一业务面。三笔账：**① 匹配策略**——示例两条 item 都用了 PREFIX（路径前缀匹配原始请求路径），另一枚举值 `URL_MATCH_STRATEGY_TEMPLATE` 走 Spring 路径模板/正则能力，两种可混用；但要清楚 pattern 匹配的是**进入网关的原始 URI**，与路由 StripPrefix/重写后的下游路径无关，写错静默不命中（分组存在、流量为 0，规则"不生效"实为没匹配上——先打请求再看分组统计验证）；**② 一个请求可命中多个分组**，每个分组各建 Entry 各算各的，多分组规则叠加=串联闸门，最紧的那个说话；**③ 分组定义与规则是两份独立数据源**（§8 的 gw-api-group 与 gw-flow），只改分组不改规则、或规则先到分组没到，都会出现"规则指向不存在资源"的空转——发布顺序：分组先行。

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

## 6.1 paramItem 的四个落地细节

网关版热点参数（内部同样经 ParamFlowChecker 体系，`8_hotspot_param.md`）比应用版多了几个现实问题：**① CLIENT_IP 在代理链后失真**——策略取 `X-Forwarded-For` 首段还是 socket 地址受配置影响，多层代理时配 `matchClientIpPattern` 控制 XFF 解析行为，**但 XFF 可伪造**，IP 配额类规则只当"防脚本"的粗闸，别进安全账；**② parseStrategy 还有 SESSION/CONSTANT**——CONSTANT 配 `matchStrategy`（精确/正则匹配 pattern）能把"只对某个 header 值限流"表达出来（如只限 `X-Gray: true` 的流量），比给灰度单独建分组轻；**③ count 语义是"每个参数值 count/窗口"**——Header 限 50 指**每个 X-User-Id** 50，不是全部用户合计，默认值维度没有"例外项"式长尾稀释问题（网关参数项就是按值分桶）；**④ 高基数值防护**：header 取到无界值（traceId 当字段名）会撞热点参数的基数墙（`8_hotspot_param.md` §5.1 的清理策略），paramItem 字段名写错=拿到空值，空值也是一个桶（全部空值请求共享阈值）——上线前故意不发那个头验证空值桶的行为符合预期。

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

## 7.1 示例自身的两处工程修正

**状态码与 body 的 code 不一致**：else-if 分支写了 503/500 语义，响应码却一律 429——要么按异常族 `setStatusCode` 联动，要么全部 429 让 code 字段做区分，语义混用会让客户端重试策略精神分裂（熔断应给 503 触发标准退避）。**ObjectMapper 每请求 new**：响应式网关里这是可见的 CPU/GC 开销，提到 lambda 外做静态单例。再加两条实践注：**被拦路径同样要发 header**（`Retry-After` 对 429/503 是 HTTP 规范义务，给客户端退避依据，§9.4 标准化 JSON 里补 timestamp/requestId 才可对账）；**else 兜底分支的"系统保护"表述要留心**——落到这里的还有 ParamFlowException、AuthorityException 家族（`2_architecture.md` §3.3 的链上全员），统一 500 会把"你没权限/你被配额"错报成服务端故障，按类型细分或以 BlockException 通用文案+code 区分。origin 定制（下游想知道终端来源）走 `WebFluxCallbackManager.setOriginParser(exchange -> ...)` 函数式注册，与 §6.1 的 XFF 语义衔接成"网关统一注入来源头 → 下游读头"的标准链路（`7_authority_rule.md` §6.1 勘误的正规解）。

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

## 8.1 数据源时代的代码删除指南

接了这两段后，§5/§6 的 `@PostConstruct` 手工加载代码**必须退役**——数据源 watch 回调与手工 loadRules 是整表覆盖竞争（`2_architecture.md` §6.1），留着手工版等于每次 Nacos 推送就把本地种子盖掉、每次重启手工版又把 Nacos 态盖掉，规则来源不可追溯是这类混用的最终形态。Nacos JSON 字段与 §5/§6 的 setter 一一对应（`count`/`intervalSec`/`resourceMode`/`paramItem{parseStrategy,fieldName,matchStrategy,matchPattern}`；分组文件是 `apiName/predicateItems[]`）；网关规则对普通 Dashboard 的可见性问题在 `9_dashboard.md` §1.1 的"控制台形态选择"账里一并处理（很多团队网关规则根本不走官方 UI，直接 Nacos 模板+流水线，§5.1 ③ 的发布顺序反而天然满足）。

# 9. 与 Spring MVC Sentinel 的区别

| 维度 | Spring MVC | Spring Cloud Gateway |
|------|-----------|---------------------|
| 拦截方式 | `SentinelWebInterceptor` | `SentinelGatewayFilter` |
| 资源粒度 | URL Pattern | Route ID / API Group |
| 参数限流 | `@SentinelResource` | `GatewayParamFlowItem` |
| 降级响应 | `@ExceptionHandler` | `BlockRequestHandler` |
| 编程模型 | 同步阻塞 | 响应式（Reactor） |
| 依赖冲突 | — | 不能与 spring-boot-starter-web 共存 |

## 9.1 表的两处补正与一行新维度

**"参数限流"行不对称**：MVC 侧的对应物是热点参数规则（args 维度，`8_hotspot_param.md`），不是 `@SentinelResource` 本身；**"降级响应"行只对了半个通道**：MVC 侧 Filter 阶段被拦走 BlockExceptionHandler、根本进不了 `@ExceptionHandler`（`10_spring_cloud_integration.md` §7.1 的双通道，网关侧反而只有单通道——handler 就是唯一出口，这也是网关"响应形态好统一"的原因）。值得加的新维度是**线程模型后果**：网关 Netty 事件循环上没有"阻塞等待"的选项——匀速排队类 controlBehavior（`4_flow_control.md` §4.5 的 sleep 语义）在网关等于占用 event loop，**网关侧规则一律只用快速失败**；同理熔断/热点的判定都是 O(1) 内存操作才撑得住聚合入口的 QPS。

# 10. 最佳实践

1. **路由 ID 规范**：用 `{服务名}-{功能}` 命名路由，便于在 Dashboard 识别
2. **API 分组策略**：按业务域（用户、订单、商品）划分 API 分组
3. **多维度限流**：路由级 + API 分组级 + IP/Header/Param 级叠用
4. **降级响应标准化**：统一 JSON 格式，包含 code/message/timestamp
5. **Gateway 监控**：配合 `SentinelGatewayFilter` 的 Metrics 在 Dashboard 查看网关级流量

## 10.1 叠用纪律与"两层容量不互抄"

第 3 条"叠用"要带刹车：§5.1② 的串闸效应意味着每个资源上最多一条路由规则+若干分组/参数项，**层级越多、误配放大面越大**——建议配额型（IP/Header 防滥用）放 paramItem，容量型放 route/分组一级，两层各写注释说明预算换算关系。第 5 条的监控账：网关指标是**全域入口视角**，它与下游各应用自己的 QPS 曲线对账时差值 = 网关拦截量 + 路由未命中——大促值班盯网关曲线先看"拒绝 QPS 抬头"再下钻是哪个分组哪个参数值（§7.1 的 requestId 贯穿日志）。最后补一条网关特有纪律：**路由变更与规则联动**——改 route id 等于换资源名，规则全部失联且不报错（静默失明，`3_quick_start.md` §6.2 第 1 名的集成版），发布检查清单里"改路由→grep 网关规则 JSON 资源名"应当是自动化的那一步。

# 11. 相关阅读

网关资源 Entry 的建立细节在 `15_slot_chain_source.md` 的 Context 章节；参数项复用的热点内核在 `18_statistics_engine.md`/`16_flow_control_source.md`；下游应用侧的配套规则在 `13_feign_integration.md`（调用方）与各服务自身（提供方权威，`11_dubbo_integration.md` §1.1 同理）；控制台形态选择在 `9_dashboard.md`。
