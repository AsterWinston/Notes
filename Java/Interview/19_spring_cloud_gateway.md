本篇覆盖 Spring Cloud Gateway 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。四个主题块：定位与核心模型、路由与过滤器、流量治理、生产与演进。微服务全景与网关在体系中的位置见 15_spring_cloud.md（1.2.2、3.2 各题），入口层 Nginx 对照见 12_nginx.md（3.1、5.3.1），Sentinel 深水区见 20_sentinel.md。

# 1. 定位与核心模型

## 1.1 产品定位

### 1.1.1 API 网关承担哪些职责？为什么微服务必须有统一入口？

**一句话**：网关是集群的**唯一对外门牌**——把"每个服务都要做、且做不好会出大事"的横切关注点收编到一处：路由转发（服务名→实例）、统一认证鉴权、限流熔断、灰度分流、跨域处理、请求/响应改写、日志与链路起点、协议适配（HTTP/WebSocket/gRPC）。

**没有网关时各方案的病**：客户端直连各服务——证书/域名/安全策略散养 N 份；前端要记 N 个地址、发 N 次跨域请求；鉴权在每个服务重复实现（口径漂移=漏洞面）；限流各做各的挡不住打网关口的流量；灰度要改所有上游。集中到网关后：**对外一张脸、对内一张网**——横切能力一次建设、全量生效，服务可以安心"内向"（只信任网关传来的身份头，11_spring_security.md 4.3.1）。

**边界也要划清（这题的成熟度标志）**：网关不是万能胶水——**业务聚合（BFF）放不放网关**要掂量：聚合逻辑进网关=网关变胖、发布风险集中（它全站单点，4.1.1），大流量场景常见"轻网关（路由+安全+限流）+ 独立 BFF 服务（聚合裁剪）"两层；**重逻辑、懂业务数据的决策**留给应用，网关只做"不懂业务也能做对"的事。与更外层 Nginx/SLB 的分工：TLS 终结、静态资源、四层防护归 Nginx/云 LB，服务语义路由（按注册元数据、灰度标签）归 SCG（12_nginx.md 5.3.1 同款口径）。

**旧题新答**：Spring Cloud Gateway 的官方定位词就是 "routing + cross-cutting concerns (security, monitoring/metrics, resiliency)"——路由、安全、观测、韧性四件事。

### 1.1.2 Spring Cloud Gateway 和 Zuul、Nginx 的区别？

**对 Zuul 1.x（面试原题，本质是线程模型题）**：

| 维度 | SCG | Zuul 1.x |
|------|-----|----------|
| 栈 | WebFlux + Reactor + **Netty**，异步非阻塞 | Servlet 同步阻塞，一请求一线程 |
| 线程模型 | 少量 event loop 撑高并发长连接 | 并发≈线程池上限，慢后端直接抽干池（5_tomcat.md 4.1.2 的账） |
| 长连接/WebSocket | 原生支持 | 勉强（1.x 不支持 WS，Zuul 2 难产夭折） |
| 流控语义 | 背压（Reactor） | 无 |
| 现状 | 现役，双形态（4.2.1） | Netflix 转维护，2026 无人新增使用 |

**答题深化点**：非阻塞的收益不是"单机 QPS 数字大"，而是**并发挂起请求的成本**——网关场景大量时间在后端 RT 上等待，阻塞模型=线程在等待中被浪费，"慢服务拖垮网关"从结构性必然变可避免；但代价是**代码一旦被阻塞就全 loop 中毒**（2.2.3 的反应式纪律）——选型换来的是纪律要求。

**对 Nginx（真实分工题）**：Nginx 是**通用 L4/L7 代理**——不懂服务注册表（upstream 静态或靠外部脚本灌），但作为 TLS 终结/静态资源/抗 D 入口久经考验；SCG 是**微服务原生网关**——`lb://order-service` 直连 Nacos 注册表（实例上下线秒级感知，17_nacos.md 2.1.3 推送链）、过滤器与 Spring 生态同构（Security/Sentinel/LoadBalancer 即插即用）、Java 团队可写代码级策略。**生产常态是串联不是二选一**：`DNS → Nginx/SLB → SCG → 服务`——各层职责见 12_nginx.md 3.2.3 与 4.2.2（两层都配 CORS 的翻车现场）。与 Kong/APISIX 的"Java 网关 vs 云原生网关"取舍，口径同 12_nginx.md 5.3.1（南北向/东西向分工）。

## 1.2 核心模型

### 1.2.1 SCG 的核心组件是什么？一条路由的解剖？

**三件套**：**Route（路由）= id + uri + Predicate 集合 + Filter 集合**；**Predicate（断言）= "这个请求归不归我管"的布尔判断**（可多个，全部满足才命中）；**Filter（过滤器）= "命中之后怎么加工请求/响应"**（可改头、重写路径、限流、熔断……）。配置即路由，yml/代码/动态源等价（2.1.3）：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-api                        # 全局唯一，重名静默覆盖（排查看 /actuator/gateway/routes）
          uri: lb://order-service              # lb:// 走注册中心+LoadBalancer；http:// 直连；ws:// WebSocket
          order: 10000                         # 路由匹配优先级（小者先）
          predicates:
            - Path=/api/orders/**
            - Header=X-Gray, on                # "与"逻辑：全部断言同时满足
          filters:
            - StripPrefix=1                    # /api/orders/1 → /orders/1
            - AddRequestHeader=X-Gateway, scg
          metadata:
            response-timeout: 5000             # 路由级超时（毫秒，3.2.2）
```

**三个组件各自的工厂机制（追问拆深度）**：断言与过滤器都由**命名工厂**创建（`PathRoutePredicateFactory`、`StripPrefixGatewayFilterFactory`）——yml 里 `Path=/api/orders/**` 实际是"约定名 + args 绑定"，工厂反射装配；**自定义扩展点就是再写一个 Factory**（`XxxRoutePredicateFactory` bean 入容器即可在 yml 按名引用）——"配置驱动 + 工厂可插拔"是 SCG 的骨架设计，报得出这层才算读过源码而不只是背过配置。

**易错**：① `lb://` 只声明"要走负载均衡"，真正解析发生在 `ReactiveLoadBalancerClientFilter`（全局过滤器，内置链一员，2.2.1）——把 LB 说成"网关启动时解析好"就错了；② 路由间 Filter 互不可见、GlobalFilter 人人有份；③ `order` 决定**匹配顺序**（第一个全断言命中的路由胜出），不是执行优先级。

### 1.2.2 一条请求在 SCG 里的完整执行流程？

**主线两跳 + 过滤器链**：

```text
① Netty HTTP 服务器收到请求（WebFlux 栈，reactor-http-nio 线程）
② RoutePredicateHandlerMapping：按 order 遍历路由定义，
   每条路由执行其全部 Predicate——第一条全命中的胜出，绑定 exchange 属性；无命中 → 404
③ FilteringWebHandler：组装本次执行的过滤器链 =
   该路由的 GatewayFilter（按 order）+ 全部 GlobalFilter（按 order 合并排序）
④ 链式执行 pre 阶段：鉴权 → 限流 → 改写头/路径……（自定义 GlobalFilter 都在这层）
⑤ 内置链尾段三兄弟接力：RouteToRequestUrlFilter（拼 lb:// 目标）
   → ReactiveLoadBalancerClientFilter（服务名→实例，改写真实 uri）
   → NettyRoutingFilter（Netty HttpClient 转发下游，order 最大）
⑥ 响应回来：NettyWriteResponseFilter 写回客户端，链的 post 阶段逆序执行
   （filter(exchange, chain).then(...) 里挂的后置逻辑：日志、改响应头）
```

**三个流程考点**：
1. **"命中即绑定"**：路由匹配发生在任何过滤器之前——所以"鉴权过滤器对哪条路由生效"早在 ② 就定了（白名单放行靠路径判断是另一层事，2.2.2）；
2. **pre/post 的写法差异**：后置逻辑不是独立回调，而是 `chain.filter(exchange).then(Mono.fromRunnable(...))`——**响应已提交后改响应头无效**，这是新手写日志/改响应过滤器最常见的坑（2.2.3 展开）；
3. **异步链上人人可短路**：任何过滤器返回 `response.setComplete()` 即终止转发（401/429 的快速失败都这么实现）——熔断/限流/鉴权"挂在一起"的架构可行性就来自这条链式契约（15_spring_cloud.md 3.2.3 的多层治理在此落到实现层）。

**追问预备**："路由匹配的缓存"——路由定义变更（RefreshRoutesEvent）时 `CachingRouteLocator` 整表重建，匹配本身每请求执行（断言函数逐个跑，Path 匹配已做编译优化）；"404 了怎么查"——actuator 看路由表 + 断言逐条排除（4.1.2 排障清单）。

# 2. 路由与过滤器

## 2.1 路由能力

### 2.1.1 内置断言有哪些？工程上怎么用组合打法定灰度、多租户、定时切流？

**按维度归组记**（零散背指令名不如按"请求的哪个部位"记）：

| 请求部位 | 断言 | 典型工程用法 |
|----------|------|-------------|
| 路径 | `Path=/api/**`（支持变量 `{id}` 捕获） | 一切路由的主谓词 |
| 方法 | `Method=GET,POST` | 读写分离路由（GET 走缓存集群） |
| 头 | `Header=X-Canary, ^on$`（正则） | 灰度分流、按租户/客户端版本路由 |
| 查询参数 | `Query=source, app` | 渠道差异化路由 |
| Host | `Host=**.example.com` | 多域名单网关 |
| Cookie | `Cookie=region, east` | 会话特征分流 |
| 来源 IP | `RemoteAddr=10.0.0.0/8` | 内网专属路由（admin 接口） |
| 时间 | `After/Before/Between` | **限时切流/活动开关**（到点自动生效） |
| 权重 | `Weight=order-group, 90/10` | 按比例金丝雀（3.2.1） |

**组合纪律（这题的价值段落）**：同路由多断言是"**与**"；"**或**"只能拆成两条路由（同 uri 不同谓词，注意 order 与 id 唯一）；**"非"靠 order**——`/api/orders/**`+Header 的灰度路由 order 小于全量兜底路由，天然形成"特例在前、通用在后"的匹配表——**路由表的排序即规则优先级**，百条路由的大厂网关必须用 order 分段（鉴权类 1 万段、灰度 2 万段、默认路由最后），否则"新加一条路由悄悄截胡了老流量"是经典事故（4.1.2）。

**实战组合样板**：灰度=`Path + Header`；多租户=`Path + Host（tenant.example.com）` 或 `Path + Header X-Tenant` 改写 `RequestHeaderToRequestUri` 定向到租户集群；内网审计=`RemoteAddr + Path`。自定义断言走工厂扩展（1.2.1）。

**易错**：`Path` 变量捕获 `{seg}` 与 MVC 的 `@PathVariable` 不是一回事（不注入任何地方，仅供 RewritePath 等过滤器以 `\{seg}` 引用）；Header 断言的值是**正则不是字面相等**（含特殊字符要转义）。

### 2.1.2 StripPrefix 和 RewritePath 怎么用才不掉坑？内置过滤器还知道哪些？

**前缀改写的两条路**：
- `StripPrefix=1`：**按段删**——`/api/orders/1` 删 1 段成 `/orders/1`（数 "/" 分段，不是删固定字符串）；与 Path 断言的对应关系要口算清楚：`Path=/api/** + StripPrefix=1` → 后端 context-path 恰好等于 `/api` 时**不要**加 StripPrefix（后端自己会再匹配一次 `/api/orders`）——**"404/多一截路径"排障先画路径三段账**：入口路径 → 断言匹配路径 → 转发路径；
- `RewritePath=/api/(?<seg>.*), /$\{seg}`：**正则重写**，支持捕获组替换——前缀非整段、加前缀、按版本号改写都靠它；`PrefixPath` 反向加前缀。

**高频内置过滤器清单**（按用途报菜名，每个一句话）：头类 `AddRequestHeader/Set/Remove`、`AddResponseHeader`、`MapRequestHeader`（把请求头映射成参数）、`DedupeResponseHeader`（**两层网关/CORS 头去重**——12_nginx.md 4.2.2"响应头重复浏览器报错"的标准解件）、`SecureHeaders`；路径参数类 `AddRequestParameter/RewriteRequestParameter`、`SetPath`、`StripPrefix/PrefixPath/RewritePath`；转发控制 `PreserveHostHeader`（后端要拿原始 Host 时必开）、`SetRequestHostHeader`、`RequestHeaderSize/RequestSize`（413/431 式防护）、`RedirectTo`；**韧性类 `Retry`、`CircuitBreaker`、`RequestRateLimiter`（各占一题，3.1/3.2.2）**；加工类 `CacheRequestBody/ModifyRequestBody/ModifyResponseBody`、`RemoveJsonAttributesResponseBody`（响应裁剪脱敏）、`LocalResponseCache`（较新版本内置响应缓存）、`TokenRelay`（OAuth2 令牌透传）、`JsonToGrpc`（gRPC 转码对外 REST）——"知道清单"比"背过参数"更被看重的是**每个解决的真实痛点**（答时挑 2~3 个带例子）。

**default-filters**：路由级复用的过滤器声明在 `spring.cloud.gateway.default-filters`——全站统一加 `X-Gateway` 头、统一 Dedupe CORS 的去处；与 GlobalFilter 的分工（过滤器工厂 vs 代码级组件）见 2.2.1。

### 2.1.3 动态路由怎么做？改了路由为什么不生效/全量重启代价？

**路由的三种来源**（可并存）：yml/代码 DSL（**重启才生效**）、`RouteDefinitionLocator` 自定义读取、运行时 `RouteDefinitionWriter` 写入 + **`RefreshRoutesEvent` 触发重建**（actuator `/actuator/gateway/routes` 的 GET 列表/POST 增删 + `/refresh` 就是这个的 HTTP 皮）。

**生产标准姿势（Nacos 版）**：路由 JSON 存 Nacos 配置（dataId=`gateway-routes.json`）→ 应用 `@ConfigurationProperties("spring.cloud.gateway")` 天然支持 routes 的 rebind（15_spring_cloud.md 4.1.2），**改配置推送即热更，无需重启**——但注意：路由列表 rebind 后仍要发 `RefreshRoutesEvent`，SCG 对 properties 刷新已内置监听（较新版本自动；自存 JSON 则自写监听器里 `publisher.publishEvent(new RefreshRoutesEvent(this))`）——"改了配置没生效"多半卡在这最后一跳。持久化诉求更重（DB 存路由、管理台 CRUD）：自建 `RouteDefinitionRepository` 实现（save/delete 落库、getRouteDefinitions 读库）+ 变更后**逐实例刷新广播**——配置中心推送通道天然是"每实例各听各的"，DB 方案则要么轮询要么 MQ 广播（对比 15_spring_cloud.md 4.1.2 的 Bus 语义）。

**两个必讲的边界**：
1. **内存仓储的脆弱**：内置默认实现进程内存储，**网关重启即回到 yml 基线**——动态路由必须外部持久化（Nacos/DB），否则"昨晚加的路由今早消失"（与 17_nacos.md 3.1.3 的"derby 随容器消失"同款事故）；
2. **actuator 写端点的安全**：`/actuator/gateway/routes` 可 POST 即"往网关注入路由"= **SSRF/RCE 级风险面**（CVE 史上的网关重灾区）——生产要么不暴露写端点要么网络隔离 + 鉴权（4.2.1 安全收口）。

**多实例一致性追问**：配置中心/广播存在到达时差，**秒级窗口内不同网关实例路由表不一致**——LB 轮询下请求可能命中"旧表实例"404；关键切流操作要在低峰 + 变更后立即用多 IP 探活核对各实例生效（排障动作 4.1.2），这是"最终一致"在网关场景的具体账单。

## 2.2 过滤器体系

### 2.2.1 GatewayFilter 和 GlobalFilter 有什么区别？order 怎么编排？

**生效范围**：`GatewayFilter`（工厂产物）**绑路由**——写在该路由 filters 或 default-filters 里，只有命中的请求执行；`GlobalFilter` **全量生效**——`implements GlobalFilter + Ordered` 的 Bean 入容器即挂进每条链（鉴权、日志、限流这类"人人都要过"的逻辑归它）。**运行期两者合流**：`FilteringWebHandler` 把"该路由的 GatewayFilter + 所有 GlobalFilter"合并、按 order 升序成一条链——**本质都是 GatewayFilter 接口的 filter(exchange, chain) 方法**（内置全局件也都实现了它），区别只在声明与装配方式；`GatewayFilterAdapter` 把 MVC/Servlet 风格件也包进来是后话。

**内置 GlobalFilter 记几个关键的**（2.2.1 流程 ⑤ 的具名）：`RouteToRequestUrlFilter`（拼 uri）、`ReactiveLoadBalancerClientFilter`（lb:// 解析）、`NettyRoutingFilter`（真实转发，order = MAX 附近）、`NettyWriteResponseFilter`（写响应）、`AdaptCachedBodyGlobalFilter/RemoveCachedBodyFilter`（body 缓存治理，2.2.3）——**自定义过滤器排在链首尾的位置由 order 决定**：鉴权要早（省下游也省转发），日志后置统计要晚（拿到最终状态）。

**order 编排纪律**（这题的工程增量）：负数段留给"必须最先"（安全、全局限流：-1000~-100），自定义业务件用正数段（100+ 常规、MAX-200 前写响应观测）——**与内置件的 order 区间错开**，内置参考值随版本有变化，自定义一律**显式实现 Ordered 或 @Order，不依赖默认 0**；一张"order 分配表"进团队规范（类比 12_nginx.md 的 location 优先级纪律）——同 order 的执行次序按 Spring 注入序，**不写 order 等于埋随机数**。

**追问**："能不能让全局过滤器只对部分路由生效"——标准做法：**路由级引用同名过滤器工厂**（GatewayFilter 版），或 GlobalFilter 内部按 exchange 属性（`exchange.getAttribute(GATEWAY_ROUTE_ATTR)` 拿 Route id）短路放行——"全量挂载 + 逻辑白名单"是小件常态（2.2.2 鉴权同款写法）。

### 2.2.2 网关统一鉴权怎么写？验 JWT 时应该查库吗？

**姿势（代码骨架）**：

```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {
    private static final List<String> WHITE_LIST = List.of("/auth/login", "/auth/captcha");
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();
        if (WHITE_LIST.stream().anyMatch(p -> pathMatcher.match(p, path))) {
            return chain.filter(exchange);                       // 白名单直通
        }
        String token = exchange.getRequest().getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
        if (token == null) { return unauthorized(exchange, 401); }
        try {
            Claims claims = jwtVerifier.verify(token);            // 本地验签，不发网络请求
            ServerHttpRequest mutated = exchange.getRequest().mutate()
                .headers(h -> {
                    h.remove("Authorization-Internal");
                    h.set("X-User-Id", claims.getSubject());      // 身份下传（下游只信这个头，
                    h.set("X-User-Roles", claims.get("roles"));   //  并剥离外部伪造的同名头！）
                }).build();
            return chain.filter(exchange.mutate().request(mutated).build());
        } catch (JwtException e) { return unauthorized(exchange, 403); }
    }
    @Override public int getOrder() { return -100; }              // 链首段（2.2.1 编排）
}
```

**三条设计论证（这题的分水岭）**：
1. **验 JWT 不查库**：网关全站流量都过这里，**每请求一次 DB/远程调用=网关自己制造瓶颈**——JWT 自包含验签即可断言真伪（11_spring_security.md 4.1）；要"吊销即时生效"再谈黑名单（Redis 位图/版本号，性能与实时性的折中，27_redis.md），或走短时 token + 刷新（过期即吊销的自然解）；
2. **信任边界收口**：网关验完身份把结论塞进**内部头**并**剥离外部同名头**——下游"只信网关头"的前提是网关做过清洗，否则客户端自己带 `X-User-Id: admin` 直接伪装（服务间信任链的入口段，11_spring_security.md 4.3.1；18_feign.md 4.2.1 是内网续段）；
3. **短路返回的正确写法**：`setStatusCode + setComplete()` 之外，统一响应体要**手写 JSON 进 buffer**（`response.writeWith(Mono.just(buffer.wrap(...)))`）并带 `Content-Type`——网关 401 返回 HTML 默认页、前端解析崩的现场很常见。

**替代方案对比**：不手写——接 **Spring Security WebFlux**（网关也是 reactive 应用，oauth2ResourceServer 一行配置干同样的事，`TokenRelay` 过滤器补 OAuth2 透传）；接 **OPA/认证服务**（网关外置鉴权中心，每跳 RPC 换决策灵活性——账在延迟）。收口句："**网关鉴权的目标是把身份验做一次、把结论传得可信**——验签在入口、清洗在入口、下游不再重复做，三件事缺一个都算没做完。"

### 2.2.3 网关代码里的"响应式纪律"有哪些？哪些写法会把整台网关打死？

**先立原理**：SCG(WebFlux) 跑在 Netty event loop 上，**每个 loop 线程串行服务成百上千连接**——`mono.block()/阻塞 IO/JDBC/synchronized 长等`任何一段阻塞代码，毒死的不是当前请求而是**该 loop 上的全部在途请求**（1_java.md 线程模型的反面教材在网关的具体化；WebClient 用对姿势见 15_spring_cloud.md 2.2.2）。

**五条纪律（每条配症状）**：
1. **禁阻塞调用**：鉴权查 Redis 用 ReactiveRedisTemplate、查 DB 用 R2DBC；**必须用阻塞 SDK 时**（遗留 JDBC）包 `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())` 把等待挪出 loop——症状：偶发"全站所有路由一起变慢"，线程栈里 reactor-http-nio 在 JDBC 上睡着；
2. **body 只读一次**：`flux.bodyToMono(String.class)` 消费后链尾拿不到——要在过滤器里加工/校验 body 用 **`CacheRequestBody` 过滤器或 AdaptCachedBodyGlobalFilter 机制**（把 body 缓存进 exchange 属性再重放，Retry 带 body 重试同受此约束，3.2.2）；症状：下游报"请求体空"；**内存账**：缓存的是 DataBuffer，大上传/长流过这种过滤器=堆内存被 body 占爆——**流式转发（SSE/WebSocket/大文件）的路由严禁挂 body 加工件**（12_nginx.md 3.2.2 `proxy_buffering off` 同款意识，36_websocket.md）；
3. **改响应要趁提交前**：post 阶段（`.then()`）改响应头**无效**（头已写出）；改写响应体要 `ServerHttpResponseDecorator` 截 writeWith 换 body——且改完 Content-Length 要重算；
4. **异常也走链**：过滤器里 try-catch 后返回错误响应，别让裸异常炸出默认 HTML（2.2.2）；`onErrorResume` 是网关版的 @ControllerAdvice；
5. **别在 filter 里做启动期才该做的事**：白名单/正则编译等静态数据放字段（构造期编译），不是每请求 new PathMatcher——"网关每秒十万次执行的代码值得抠"。

**追问"那 WebMVC 版呢"**——正是给吃不下响应式纪律的团队准备的逃生门（4.2.1）：Servlet 线程模型按 5_tomcat.md 4.1.2 的并发账重新规划（线程池成为天花板），纪律 2/3 依旧、1 自然消失。**"选型即选纪律"**——这是这题的满分收口。

# 3. 流量治理

## 3.1 限流

### 3.1.1 RequestRateLimiter 网关限流怎么配？和 Nginx、Sentinel 的限流有什么本质区别？

**最小配置**（Redis 令牌桶，内置过滤器工厂）：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-api
          uri: lb://order-service
          predicates: [ Path=/api/orders/** ]
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100    # 令牌生成速率（个/秒）
                redis-rate-limiter.burstCapacity: 200    # 桶容量（允许的突发）
                redis-rate-limiter.requestedTokens: 1    # 每请求消耗令牌数
                key-resolver: "#{@userKeyResolver}"      # SpEL 指向 KeyResolver Bean
                status-code: 429                          # 超限响应码（默认 429，别改 503）
    data:
      redis:                                             # Boot 3 前缀（旧文 spring.redis 已过时）
        host: redis.internal
```

```java
@Bean
public KeyResolver userKeyResolver() {   // 限流维度：这个 key 是谁
    return exchange -> Mono.justOrEmpty(
        exchange.getRequest().getHeaders().getFirst("X-User-Id"))
        .defaultIfEmpty(Objects.requireNonNull(
            exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress());
}
```

**机制拆解**：限流状态放 **Redis**，Lua 脚本原子执行"取两键（`request_rate_limiter.{key}.tokens` / `.timestamp`）→ 按时间差补令牌 → 扣减判放行"——**令牌桶语义与 12_nginx.md 3.2.1 同款**（replenishRate=rate、burstCapacity=桶深；无 nodelay 对应物，超限直接拒）。响应侧附带 `X-RateLimit-Remaining` 等头可供前端退避。

**这题的本质答案是"状态放在哪"**（三层限流对照，面试拉层次的关键）：

| | Nginx limit_req | SCG RequestRateLimiter | Sentinel 集群限流 |
|--|----------------|------------------------|-------------------|
| 计数状态 | **各实例本地内存**（多节点=N 倍实际阈值，12_nginx.md 3.2.1 的老账） | **共享 Redis**（全局精确） | 本地滑动窗口；集群模式 token server（20_sentinel.md） |
| 算法 | 漏桶（延迟放行可选 nodelay） | 令牌桶 | 滑动窗口 QPS + 预热/排队 |
| 维度 | IP/URI 等变量 | KeyResolver 任意（用户/IP/路由组合） | 资源（路由/API 组）+ 参数热点 |
| 新增依赖 | 无 | **Redis 成为限流数据面**（挂了=全放行或全拒，取决于降级策略） | Dashboard+规则源 |

**工程要点三条**：① **网关层限流天然全站视角**，挡在转发之前（省下游也省 LB 带宽）——粒度到"每用户每接口"就够业务用；更细的业务级/热点参数限流留应用侧（20_sentinel.md）；② Redis 单点与热点 key 要算账（超高频 key 分片或本地+Redis 两级）；③ 超限响应要**可判别**（429 + JSON 错误体，前端退避协议提前对齐，2.2.2 短路写法复用）——"限流返回 200 空体"是把保护做成事故的最后一公里。

### 3.1.2 Sentinel 网关流控怎么接？和 RequestRateLimiter 怎么分工？

**接法**：依赖 `spring-cloud-alibaba-sentinel-gateway`（内含 `sentinel-spring-cloud-gateway-adapter`）后，适配器自动把**每条路由的 Route id 注册为资源**、`spring.cloud.sentinel.scg.fallback.*` 配拦截响应（redirect 重定向 / response 直写自定义 429 JSON，响应文案与状态码可配——具体键位随 SCA 版本对表）；控制台出现"网关应用"分组：**API 分组**把多条路由聚成一个逻辑 API 统配规则，**流控规则**按路由/API 组设 QPS 阈值（快速失败/Warm-up/排队三种效果），**降级规则**按慢调用比例/异常比例熔断该路由——拦截发生在过滤器链高位，被拒流量**不出网关**。

**与 RequestRateLimiter 的分工**（不是二选一，是"两套状态机"）：

| 维度 | RequestRateLimiter | Sentinel SCG |
|------|--------------------|--------------|
| 计数 | Redis 全局精确 | 进程内滑动窗口（**每网关实例各算各的**——多实例全局阈值≈N×配置值，同 12_nginx.md 3.2.1 的账，除非上集群限流 token server，20_sentinel.md） |
| 规则变更 | 改配置重启/刷新 | **控制台动态改 + 规则持久化 Nacos** |
| 能力面 | 只有限流 | 流控 + 熔断降级 + 热点参数 + 系统自适应 + 实时监控曲线 |
| 依赖 | Redis 数据面 | Dashboard + 规则数据源 |

**实践口径**：**要"动态规则 + 熔断 + 观测"全家桶 → Sentinel 接管网关流控**（国内 SCA 栈默认，与 18_feign.md 4.1.2 的应用侧熔断同一控制台、同一规则体系——"一份治理心智覆盖入口到叶子"是它的真卖点）；**只要一条"每用户 QPS 硬顶"且已信 Redis → RateLimiter 更轻**。同层叠加要节制（两套限流串在一条链=调参灾难），选一套为主、另一套最多留"总闸"角色。追问"为什么不用 Redis 也做熔断"——熔断的核心输入是**调用结果的实时统计**（异常比例/RT 分布），放 Redis 等于把统计流量再导一遍；限流只需计数器、熔断需要统计窗口——**数据形态决定组件位置**，这句是治理类选型题的通用答案。

## 3.2 灰度、超时与跨域

### 3.2.1 网关灰度发布怎么做？Weight 断言的"比例"是怎么稳定的？

**三层灰度工具箱**（呼应 12_nginx.md 3.2.3 的 Nginx 三把刀，SCG 版）：

```yaml
routes:
  # ① 按特征：白名单/内部员工先行（Header 由入口或客户端带）
  - id: order-gray
    uri: lb://order-service-canary
    order: 100                       # 特例在前（1.2.1 order 纪律）
    predicates: [ Path=/api/orders/**, Weight=order-group, 10 ]   # ② 按权重
  - id: order-stable
    uri: lb://order-service
    order: 200
    predicates: [ Path=/api/orders/**, Weight=order-group, 90 ]
```

③ **按元数据（进阶）**：路由目标不变，靠自定义 Predicate/LoadBalancer 读 Nacos 实例标签选版本组——网关只认请求特征，"谁该进灰度"的决策与实例注册体系合体（17_nacos.md 1.2.2 权重/元数据/cluster 三原语在网关侧的消费点；15_spring_cloud.md 3.2.2 的 supplier 装饰链）。

**Weight 断言机制追问（区分度所在）**：同 group 的多条路由按权重生成随机命中——且**命中结果会写进 WebSession**（session 粘滞：同一浏览器会话后续请求倾向留在同一版本），避免"同一用户一半页面新样式一半旧"的撕裂体验；**匿名流量按会话、登录流量按用户标签**才是生产姿势——粘滞粒度错了灰度观察数据就废（对比 Nginx split_clients 以 `$remote_addr` 为哈希键的"同 IP 稳定"语义，12_nginx.md 3.2.3 同一道题）。

**放量闭环（把"怎么切"升级成"怎么安全地切"）**：5% → 观察错误率/RT/业务指标（4.1.2 监控面板按 routeId 分维度）→ 25% → 50% → 100%，**每一步可秒级回滚=改权重归零**（路由配置动态下发，2.1.3）；灰度流量的**数据兼容性**是比分流更硬的门槛（新旧版本共写一个库——字段只加不改不删的发布纪律，21_seata.md 分布式事务同理的"过渡态思维"）；全链路灰度别忘下游也要会路由（网关打 `X-Gray` 标只是开头，标签透传与逐跳过滤在 18_feign.md 4.2.1、17_nacos.md 1.2.2——**入口灰 ≠ 全链路灰**，答题主动补这句即超纲）。

### 3.2.2 超时、重试、CORS 这三件"配置题"各有什么必错点？

**超时（预算链条的枢纽位）**：全局 `spring.cloud.gateway.httpclient.connect-timeout`（默认 45s——**生产必改**，连接建不起来 45 秒才放手纯属等死）与 `response-timeout`（全局可配 Duration）；路由级 `metadata: connect-timeout / response-timeout`（毫秒）**单独放宽/收紧**（报表接口 30s、核心链路 2s——慢接口专用超时挂 metadata 是 12_nginx.md "单独 location 放宽"同款思想）。**预算不等式**（15_spring_cloud.md 3.2.3 在网关的落地）：`入口(Nginx/客户端) > 网关全局 > 网关慢路由 > Σ下游服务超时+重试`（18_feign.md 2.2.3 的乘法账）——配反的典型症状：**用户看到 504 而网关日志"下游正常返回"**（外层先断，内层白干；12_nginx.md 5.2.2 的 504 排查在网关侧原样重演）。连接池维度（elastic/fixed、`pending-acquire-timeout`、max-life-time）：池耗尽的"排队超时"与下游慢的"响应超时"症状相同根因不同——`pending acquire` 类报错先查池参数（与 18_feign.md 3.1.2 perRoute 账互为镜像）。

**重试（Retry 过滤器，默认值先记对）**：默认 **retries=3、series=5XX、methods=GET、exceptions=IOException/TimeoutException**，backoff/jitter 默认关。三个必错点：① **把 POST 加进重试 methods**=重复下单风险——非幂等写接口永不网关重试（13_rocketmq.md 3.2.3 幂等论证的服务端版）；② **重试会重放它之后的所有过滤器**（官方文档明说）——挂重试点位的下游过滤器若带副作用（审计打标、计数）会被执行多次，顺序设计（2.2.1）要一起想；带 body 的重试会**缓存整个请求体进内存**（DataBuffer）——大 body 路由开重试=内存炸（2.2.3 纪律 2 同款）；③ backoff 不配=毫秒内连打三次，下游过载时等于补刀——`firstBackoff/factor/maxBackoff + jitter` 是标配（退避+抖动防重试共振，20_sentinel.md 风暴论证）。**重试该在哪层**：网关重试只保"连接级失败换机"这种安全场景，业务级补偿交给 MQ/任务（13_rocketmq.md）——"网关越勤快，下游越惨"。

**CORS**：`spring.cloud.gateway.globalcors.cors-configurations` 全局一份（允许的 origin/methods/headers/credentials），路由级可用 `default-filters` 里的 AddResponseHeader 补——但**正解永远是配置化 CORS 而非手工拼头**；三个高频事故：① **Nginx 与网关两层都加 CORS 头** → 响应里出现两个 `Access-Control-Allow-Origin`，浏览器直接报 "not a single origin"（12_nginx.md 4.2.2 点过名的现场，解法=DedupeResponseHeader 或删一层，2.1.2）；② **鉴权过滤器把 OPTIONS 预检拦成 401**——预检不带 Authorization 头是规范行为（28_http.md），白名单必须放行 OPTIONS；③ 带凭证时 origin 写 `*`（规范禁止，credentials 模式必须精确回显 origin + `Vary: Origin`）。分工结论同 12_nginx.md 4.2.2：**CORS 只在一层管**（推荐网关，入口 Nginx 透传），同源部署仍是终极解。

# 4. 生产与演进

## 4.1 部署与可观测

### 4.1.1 网关自己怎么做高可用？"全站入口"的容量与故障域怎么规划？

**形态前提**：SCG 进程**本身无状态**（路由表来自配置/注册中心、限流计数在 Redis、会话粘滞在客户端 cookie/Redis Session）——**横向多实例 + 前置 LB 即 HA 骨架**：`DNS → 云 SLB/Nginx（12_nginx.md）→ SCG × N（≥2，跨机架/可用区）→ 服务`。网关层没有"主备"概念，全活互备，单实例挂 = LB 健康检查摘除（Nginx 被动/主动检查语义 12_nginx.md 3.1.2 在这里反过来用在网关身上）。

**四类状态点的故障域清单**（"无状态"答完要接这句才算完整）：
1. **Redis（限流/会话）**：数据面依赖——Redis 挂：RateLimiter 拒绝或放行取决于异常分支写法（**必须显式设计降级=放行保可用还是拒绝保守**，默认抛异常打挂全站是事故写法，27_redis.md）；
2. **Nacos（路由/注册）**：管控面依赖——挂时网关按本地缓存继续转发、注册表停止更新（17_nacos.md 4.2.1 的"能用不能变"对网关同样成立，且网关是受害最深的客户端：它的下游列表也来自 Nacos）；
3. **配置中心**：动态路由通道——发布窗口内的多实例视图差（2.1.3）；
4. **自身资源**：event loop 线程数≈CPU 核数（Netty 默认）——**容量模型=核数 × 单 loop 吞吐**，压测定标（连接数、在途请求、body 缓存水位三线告警，2.2.3 的内存账在这里变成容量账）。

**发布与容灾动作**：滚动发布 + 优雅停机（在途请求排空再从 LB 摘除，10_springboot.md 4.1.2 机制同态；K8s 场景配 preReadiness/terminationGrace，32_kubernetes.md）；**网关变更是全站变更**——路由/过滤器发布走灰度（先放一台实例验，3.2.1 的方法论反身用于网关自身）；跨 AZ 部署 + LB 跨区探测；**熔断兜底配到位**（CircuitBreaker 过滤器/fallbackUri `forward:` 到本地降级端点——注意与 Retry 同挂时目标端点不能提前提交响应，官方文档点名的组合坑）：下游全挂时网关还能吐出静态降级页而不是 5xx 雪崩——"网关的最后职责是让失败也体面"。

**追问"要不要多活"**：网关实例无状态 → 天然适合多区多活；真正约束在它的依赖（Redis/Nacos 的跨区形态）——答"网关多活的前提是把数据面依赖也做成多活"即闭环。

### 4.1.2 网关的观测体系怎么建？常见故障各给一条排查路径？

**指标（第一公民）**：SCG 内置 Micrometer 计时器 **`spring.cloud.gateway.requests`**（tag：routeId/status/outcome——RED 三件套按路由维度天然就绪）+ WebFlux 服务器指标；连接池指标（reactor netty pool）与限流拒绝数（RateLimiter/Sentinel block）补上"入口层自己的 USE"——Grafana 面板按 **routeId 分维**是灰度观察（3.2.1 门禁）与故障定位的共同底座（15_spring_cloud.md 4.2.2 三支柱在此的实例化）。

**链路**：SCG 是分布式追踪的**天然起点**——Micrometer Tracing 自动为网关生成根 span，`X-Request-Id/traceparent` 向下游贯穿（15_spring_cloud.md 4.2.1）；访问日志开 `reactor.netty.http.server.accessLogEnabled`（或结构化自建 GlobalFilter 打 JSON，字段学 12_nginx.md 5.2.1 的 request_time/upstream_response_time 拆法：网关版对应"总耗时/等下游首字节耗时"）。**traceId 进日志 pattern**，5xx 从入口一路追到叶子服务（12_nginx.md 5.2.1 同款工程，网关侧多做一件：把 traceId **写进响应头回给前端**——用户报障带 ID，客服能直接给研发线索）。

**故障排查速查表**（面试按"症状→第一段查什么"组织）：

| 症状 | 首查 | 常见根因 |
|------|------|---------|
| 某路径 404 | `/actuator/gateway/routes` 比对期望路由表 | 路由没注册/被覆盖（id 重名）、Path 与 StripPrefix 三段账算错（2.1.2）、order 截胡（1.2.1） |
| 502/504 | 网关 error log + 对应 routeId 指标 | 与 Nginx 版同构（12_nginx.md 5.2.2）：实例列表脏（Nacos 摘除滞后）、连接竞态（18_feign.md 3.1.2 keepalive 对齐账在网关-服务之间重演）、response-timeout 到点 |
| 全站所有路由一起变慢 | 线程栈（jstack 看 reactor-http-nio 在干嘛） | **event loop 被阻塞**（2.2.3 纪律 1）、连接池 pending 队列爆 |
| 内存持续涨 | 是否某路由挂了 body 缓存/大流量流式 | CacheRequestBody/Retry 带 body（3.2.2）、流式路由误挂缓冲件（2.2.3） |
| 限流失效/全员被拒 | Redis 连通与异常分支 | Redis 挂（4.1.1 状态点 1）、KeyResolver 返回 null（全员共享一个 key 的"意外全局限流"） |
| CORS 报错 | 数响应头里 Allow-Origin 出现几次 | 双层各加一遍（3.2.2） |
| 改了路由部分实例没生效 | 逐实例 curl actuator 比对路由表 version | RefreshRoutesEvent 未发/配置监听器没跑（2.1.3） |

**告警设计**（区别于"有面板"）：入口层看**错误率与 P99 的环比突变**（按 routeId），连接池 pending、Redis 拒绝异常、路由表各实例 diff（一致性巡检脚本）三类"静默故障"专告——"网关故障的特征是'全都像没事但都慢'"，把这句答出来即实战派。

## 4.2 形态演进

### 4.2.1 2026 年的 Spring Cloud Gateway：双形态、新属性体系与平台化挤压下怎么定位？

**形态全景（先把"SCG 是一个东西"的旧印象掰正）**：官方文档现在把 Gateway 分成**两个 flavor × 两套栈**——**Server**（独立网关应用，主打）与 **Proxy Exchange**（嵌在普通 WebFlux/WebMVC 应用里当转发器用，`ProxyExchange` 参数式写法，BFF 轻转发场景）；Server 又分 **WebFlux 版与 WebMVC 版**——WebMVC 版自 2023.0 时代补入（starter `spring-cloud-starter-gateway-server-webmvc`，路由/谓词/过滤器模型对齐，2.2.3 的响应式纪律换成 Servlet 线程池账），吃不下 Reactor 的团队从此有退路。

**代际与属性口径（2026）**：4.x 线（Boot 3 列车）属性在 `spring.cloud.gateway.*`；**5.x 线（Boot 4/Framework 7，2025.1 列车，15_spring_cloud.md 2.1.2）属性重组进 `spring.cloud.gateway.server.webflux.*` 命名空间**、并持续吸收内置件（`LocalResponseCache` 响应缓存、`JsonToGrpc` 转码、`HttpHeadersFilter` 体系、`SecureHeaders`）——跨代升级时**配置键全表对文档**（本篇所有示例按 4.x 惯用键书写，报得出"5.x 改了前缀"即时效加分项）。

**平台化挤压下的定位（这题真正的考法）**：K8s **Gateway API**/Ingress 与 Envoy/Istio 正在收编"集群边界"的路由与 TLS（15_spring_cloud.md 5.2.1：入口归平台是趋势），云厂商 ALB/API 网关同理——SCG 的生存线随之后移：**从"唯一入口"退居"业务语义网关"**——懂注册中心元数据的路由与灰度（3.2.1 ③）、与 Spring Security/Security 生态一体的鉴权（2.2.2）、懂业务的动态路由控制台（2.1.3）——**平台层做 L4-L7 通用治理，SCG 做 Java 域的业务治理**，与 Nginx/Kong 的分层共存口径一致（12_nginx.md 5.3.1：Kong 偏南北向 API 产品化、SCG 偏 Spring 体系内）。

**选型姿态收口**：纯 K8s、多语言、入口治理诉求轻 → Gateway API/Envoy 直进，可不部署 SCG；Spring Cloud 体系（Nacos/Sentinel/SCS）→ SCG 仍是默认（一体化心智）；两者同存时**别让两层都做灰度/限流**（规则双头漂移，3.2.2 的 CORS 教训放大版）——"入口层只许一个大脑做决策，另一层透传"。**答题最后一句留给演进判断**："网关组件之争在收敛，'横切能力放哪一层'的架构题永存——SCG、Kong、Gateway API 只是同一个答案在不同平台战略下的投影。"