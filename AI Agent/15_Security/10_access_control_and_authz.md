# 1. 访问控制的全链路

访问控制（Access Control）是 AI 应用安全的基础设施：在用户与资源之间建立一条可验证、可追溯的链路。这条链路包含三个环节——认证（Authentication）、授权（Authorization）与审计（Audit）。

```
用户请求
   │
   ▼
┌──────────────┐
│ 认证          │  Authentication：你是谁？
└──────┬───────┘
       ▼
┌──────────────┐
│ 授权          │  Authorization：你能做什么？
└──────┬───────┘
       ▼
┌──────────────┐
│ 资源访问      │  Resource：模型 / 工具 / 知识库
└──────┬───────┘
       ▼
┌──────────────┐
│ 审计          │  Audit：你做了什么？
└──────────────┘
```

| 概念 | 英文 | 说明 |
|------|------|------|
| 认证 | Authentication | 确认身份（登录、Token） |
| 授权 | Authorization | 确认权限（能访问什么） |
| 审计 | Audit | 记录行为，供追责与合规 |

## 1.1 认证、授权与审计的关系

三者不是孤立的步骤，而是相互依赖的闭环：

- 没有认证，授权无从谈起——不知道“你是谁”，就无法判断“你能做什么”。
- 没有授权，认证只是“证明身份”，任何已登录用户都能访问全部资源。
- 没有审计，前两者出错时无法定位，也无法满足合规要求。

一个常见误解是“做了登录就等于安全”。登录只解决认证；越权（横向越权/纵向越权）恰恰发生在认证之后。

## 1.2 访问控制的核心原则

### 1.2.1 最小权限原则（Least Privilege）

主体只应拥有完成当前任务所必需的最小权限。AI 场景下尤其重要：Agent 可调用的工具、可检索的知识库、可使用的模型都应被裁剪到当前用户所需范围，而不是“默认全开”。

### 1.2.2 默认拒绝（Deny by Default）

未被显式允许的一律拒绝。白名单优于黑名单，因为黑名单永远无法穷举所有攻击面。

### 1.2.3 职责分离（Separation of Duties）

敏感操作需要多个角色或多次确认，避免单点拥有过大权力。例如“导出全量数据”与“审批导出”应由不同角色完成。

## 1.3 与传统 Web 访问控制的差异

| 维度 | 传统 Web | AI 应用 |
|------|----------|---------|
| 资源粒度 | 接口、页面 | 模型、工具、知识库、会话、配额 |
| 授权时机 | 请求前 | 请求前 + 调用中（工具、检索） |
| 上下文 | 较静态 | 会话、记忆、多轮动态 |
| 主要风险 | 越权访问数据 | 越权工具调用、提示注入绕过 |
| 审计对象 | 数据读写 | 数据读写 + 模型/工具调用 |

# 2. 认证方式

认证回答“你是谁”。不同场景下适用的认证方式不同，选择时要在安全性、性能与实现复杂度之间权衡。

## 2.1 常见认证方式对比

| 方式 | 英文 | 适用 | 特点 |
|------|------|------|------|
| JWT | JSON Web Token | 无状态、微服务 | 自包含、可验签、难以主动撤销 |
| Session | Session | 传统 Web | 服务端存储、易撤销、有状态 |
| OAuth2 | OAuth 2.0 | 第三方登录、委托授权 | 标准协议、需授权服务器 |
| API Key | API Key | 服务间调用 | 简单、粒度粗、需妥善保管 |
| mTLS | Mutual TLS | 高安全内部通信 | 双向证书、强身份、运维成本高 |

选择建议：

- 面向用户的 Web/App：OAuth2 + JWT 组合最常见。
- 服务间调用：mTLS 或带短时效令牌的服务身份。
- 对外开放平台：API Key + 配额 + 签名。

## 2.2 JWT 认证实现

JWT 由三部分组成：头部（Header）、载荷（Payload）、签名（Signature），用点号连接。服务端只需验签即可确认身份，无需查询会话存储。

### 2.2.1 JWT 的结构与校验要点

| 部分 | 内容 | 校验要点 |
|------|------|----------|
| Header | 算法、类型 | 只允许白名单算法，禁用 `none` |
| Payload | 声明（claims） | 校验 `iss`、`aud`、`exp`、`nbf` |
| Signature | 签名 | 使用服务端密钥/公钥验签 |

常见错误：

- 信任 `alg` 字段（算法混淆攻击）。
- 不校验 `exp`，导致过期令牌仍可用。
- 把敏感信息（密钥、PII）放进 Payload，因为 Payload 只是 Base64 编码，并非加密。

### 2.2.2 JWT 过滤器完整实现

```java
// JWT 解析并构建用户上下文（响应式 WebFlux）
@Component
public class JwtAuthenticationFilter implements WebFilter {

    private final JwtVerifier jwtVerifier;

    public JwtAuthenticationFilter(JwtVerifier jwtVerifier) {
        this.jwtVerifier = jwtVerifier;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String token = extractBearer(exchange.getRequest());

        if (token == null) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        try {
            Claims claims = jwtVerifier.verify(token);

            UserContext user = new UserContext(
                    claims.get("tenantId", String.class),
                    claims.getSubject(),
                    claims.get("roles", List.class),
                    claims.get("department", String.class));

            // 放入 Reactor Context（响应式场景）
            return chain.filter(exchange)
                    .contextWrite(ctx -> ctx.put(UserContext.class, user));

        } catch (Exception e) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
    }

    private String extractBearer(ServerHttpRequest request) {
        String header = request.getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
        if (header == null || !header.startsWith("Bearer ")) {
            return null;
        }
        return header.substring(7);
    }
}
```

## 2.3 Session 与 OAuth2

Session 适合传统有状态 Web：登录后服务端保存会话，客户端只持有会话 ID。优势是易于主动撤销，劣势是不利于水平扩展与跨域。

OAuth2 适合委托授权（如“使用某平台账号登录”）。典型流程为授权码模式（Authorization Code）：客户端跳转授权服务器，用户同意后回调获取授权码，再用授权码换取令牌。

| 维度 | Session | JWT | OAuth2 |
|------|---------|-----|--------|
| 状态 | 有状态 | 无状态 | 取决于令牌实现 |
| 撤销 | 容易 | 困难 | 支持撤销令牌 |
| 扩展性 | 需共享存储 | 好 | 好 |
| 适用 | 单体内网 | 微服务 | 第三方授权 |

## 2.4 API Key 与 mTLS

API Key 是最简单的服务间认证：请求头携带密钥，服务端查表校验。要点是密钥必须哈希存储、支持轮换、绑定来源与配额，并避免出现在日志与前端代码中。

mTLS 通过双向证书认证双方身份，适合高安全内部通信。它能防中间人攻击，但证书签发与轮换的运维成本较高。

# 3. 用户上下文传递

认证完成后，用户身份需要在调用链中一路传递到业务逻辑。传递方式因编程模型而异。

## 3.1 ThreadLocal（阻塞场景）

### 3.1.1 实现与清理

```java
public class UserContextHolder {

    private static final ThreadLocal<UserContext> CURRENT = new ThreadLocal<>();

    public static void set(UserContext user) { CURRENT.set(user); }
    public static UserContext current() { return CURRENT.get(); }
    public static void clear() { CURRENT.remove(); }
}
```

```java
// 阻塞式 Filter 中设置，必须在 finally 清理
try {
    UserContextHolder.set(user);
    chain.doFilter(request, response);
} finally {
    UserContextHolder.clear();          // 必须清理
}
```

### 3.1.2 线程池串数据问题

Web 容器（如 Tomcat）会复用线程。如果不在 `finally` 中清理，上一个请求的用户身份会残留在 `ThreadLocal` 中，被下一个请求读到，造成**身份串号**——这是严重的安全事故。

## 3.2 Reactor Context（响应式）

响应式编程中，逻辑可能在不同线程上执行，`ThreadLocal` 无法可靠传递。此时必须使用 Reactor Context。

### 3.2.1 Context 的不可变性与传播

Reactor Context 是**不可变**的：每次 `contextWrite` 返回新的 Context，只对**上游**（订阅方向的下游数据流）可见。因此写入位置与读取位置的方向必须正确。

```java
public Mono<String> chat(String question) {
    return Mono.deferContextual(ctx -> {
        UserContext user = ctx.get(UserContext.class);
        return Mono.fromCallable(() -> doChat(question, user));
    });
}
```

### 3.2.2 完整示例

```java
public Mono<ChatResult> handle(String question) {
    return Mono.deferContextual(ctx -> {
        UserContext user = ctx.get(UserContext.class);
        return chatService.chat(question, user)
                .map(reply -> new ChatResult(reply, user.tenantId()));
    });
}
```

## 3.3 上下文传递的边界与反例

| 反例 | 后果 | 正确做法 |
|------|------|----------|
| ThreadLocal 未清理 | 身份串号 | finally 清理 |
| 响应式用 ThreadLocal | 上下文丢失 | Reactor Context |
| 从请求参数取 userId | 可被伪造 | 从认证结果取 |
| 异步线程不传上下文 | 越权或空指针 | 显式传递或包装 |

> **ThreadLocal 在线程池复用时会串数据**，响应式必须用 Reactor Context。

# 4. 授权模型

授权回答“你能做什么”。常见模型有 RBAC、ABAC，以及更细粒度的 ReBAC。

## 4.1 RBAC

### 4.1.1 模型定义

RBAC（Role-Based Access Control，基于角色的访问控制）通过角色桥接用户与权限：

```
用户 → 角色 → 权限
```

### 4.1.2 实现

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteData(String id) { ... }

@PreAuthorize("hasAnyRole('SUPPORT', 'ADMIN')")
public void refund(String orderId) { ... }
```

RBAC 的优点是简单直观，缺点是角色爆炸：当权限维度变多（部门、地区、数据级别）时，角色数量会急剧膨胀。

## 4.2 ABAC

### 4.2.1 属性维度

ABAC（Attribute-Based Access Control，基于属性的访问控制）基于属性组合决策：

```
主体属性 + 资源属性 + 环境属性 → 决策
```

- 主体属性：用户 ID、角色、部门。
- 资源属性：所有者、部门、密级。
- 环境属性：时间、IP、设备。

### 4.2.2 实现

```java
@Component
public class AbacEvaluator {

    public boolean canAccess(UserContext user, Resource resource) {
        return switch (resource.type()) {
            case ORDER -> resource.ownerId().equals(user.userId())
                    || user.hasRole("ADMIN");
            case DOCUMENT -> resource.department().equals(user.department())
                    || user.hasRole("ADMIN");
            default -> false;
        };
    }
}
```

## 4.3 在 AI 场景的应用

AI 应用的资源比传统 Web 更多，授权点也更细：

| 资源 | 授权点 |
|------|--------|
| 模型 | 用户可用哪些模型 |
| 工具 | 用户可调用哪些工具 |
| 知识库 | 用户可检索哪些文档 |
| 会话 | 用户可访问哪些会话 |
| 配额 | 用户的 Token/成本上限 |

# 5. AI 特有的访问控制

## 5.1 资源授权点

AI 应用的授权要覆盖“模型、工具、检索、配额、自主性”五个维度。这些维度不能只靠 RBAC 的单一角色判断，往往需要结合租户、部门、数据级别等属性。

## 5.2 策略解析与应用

### 5.2.1 AiPolicy 构建

```java
public class AiAccessControl {

    public AiPolicy resolve(UserContext user) {
        return AiPolicy.builder()
                .allowedModels(modelPolicy.forUser(user))
                .allowedTools(toolPolicy.forUser(user))
                .retrievalFilter(retrievalPolicy.forUser(user))   // 元数据过滤
                .tokenQuota(quotaService.quota(user))
                .autonomyLevel(autonomyPolicy.forUser(user))
                .build();
    }
}
```

### 5.2.2 策略应用到调用链

```java
// 应用策略：工具白名单、检索过滤、模型选项
String answer = chatClient.prompt()
        .user(question)
        .toolCallbacks(policy.allowedTools())
        .advisors(a -> a.param("filter", policy.retrievalFilter()))
        .options(policy.modelOptions())
        .call()
        .content();
```

## 5.3 检索层的权限过滤

### 5.3.1 元数据过滤

知识库检索必须在**检索阶段**就注入权限过滤，而不是检索后再过滤。检索后再过滤会导致：无关文档被送入模型，造成信息泄露；以及召回结果被截断，答案质量下降。

```java
// 伪代码：把权限条件下推到向量库过滤表达式
Filter filter = Filter.and(
        Filter.eq("tenantId", user.tenantId()),
        Filter.in("department", user.departments()),
        Filter.lte("level", user.clearanceLevel()));

List<Document> docs = vectorStore.similaritySearch(
        SearchRequest.builder()
                .query(query)
                .filterExpression(filter)
                .topK(10)
                .build());
```

### 5.3.2 检索越权反例

| 反例 | 风险 |
|------|------|
| 先全量检索再过滤 | 敏感内容进入 Prompt，可能被模型复述 |
| 只按租户过滤不按部门 | 同租户内跨部门越权 |
| 过滤条件由前端传入 | 攻击者可篡改过滤条件 |
| 缓存未按用户隔离 | 命中他人检索缓存 |

# 6. 服务间鉴权

## 6.1 身份透传

内部服务调用必须透传调用者身份与链路标识，否则下游无法做权限校验与审计。

```java
// 内部服务调用带身份透传
public String callInternalService(String url, String body, UserContext user) {
    return webClient.post()
            .uri(url)
            .header("Authorization", "Bearer " + serviceToken())
            .header("X-User-Id", user.userId())
            .header("X-Tenant-Id", user.tenantId())
            .header("X-Trace-Id", MDC.get("traceId"))
            .bodyValue(body)
            .retrieve()
            .bodyToMono(String.class)
            .block();
}
```

> 服务间调用要透传用户身份与 traceId，否则下游无法做权限与审计。

## 6.2 服务令牌与零信任

零信任（Zero Trust）的核心是“永不信任，始终验证”。服务间通信同样要验证身份，不能因为“在内网”就默认可信。

| 机制 | 说明 |
|------|------|
| 服务令牌 | 短时效、限定受众（audience） |
| mTLS | 双向证书验证服务身份 |
| 网络策略 | 限制服务间可达范围 |
| 请求签名 | 防篡改、防重放 |

## 6.3 常见反例

- 下游服务只信任 `X-User-Id`，攻击者可直接伪造该头绕过认证。
- 服务令牌长期有效，泄露后长期可用。
- 内网服务无鉴权，一旦边界被突破即可横向移动。

# 7. 与 Spring Security 集成

## 7.1 响应式配置

```java
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(
            ServerHttpSecurity http) {
        return http
                .csrf(ServerHttpSecurity.CsrfSpec::disable)
                .authorizeExchange(ex -> ex
                        .pathMatchers("/actuator/health/**").permitAll()
                        .pathMatchers("/api/admin/**").hasRole("ADMIN")
                        .pathMatchers("/api/chat/**").authenticated()
                        .anyExchange().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
                .build();
    }
}
```

## 7.2 方法级鉴权

```java
@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {
}
```

```java
@PreAuthorize("@authz.canAccess(#resourceId, authentication)")
public Resource getResource(String resourceId) {
    return repository.findById(resourceId).orElseThrow();
}
```

# 8. 权限缓存

权限查询往往涉及多次数据库访问，缓存能显著降低延迟，但必须处理一致性问题。

## 8.1 缓存实现

```java
@Component
public class CachedPermissionService {

    private final Cache<String, Set<String>> cache = Caffeine.newBuilder()
            .expireAfterWrite(Duration.ofMinutes(5))
            .maximumSize(10_000)
            .build();

    public Set<String> allowedTools(UserContext user) {
        String key = user.tenantId() + ":" + user.userId();
        return cache.get(key, k -> computeAllowedTools(user));
    }
}
```

## 8.2 失效策略

```java
    // 权限变更时主动失效
    @EventListener
    public void onPermissionChanged(PermissionChangedEvent event) {
        cache.invalidate(event.tenantId() + ":" + event.userId());
    }
```

| 策略 | 说明 | 缺点 |
|------|------|------|
| 主动失效 | 变更时立即清除 | 需事件可靠投递 |
| 短 TTL | 过期自动刷新 | 存在时间窗 |
| 版本号 | 权限版本变化即失效 | 实现复杂 |

# 9. 生产注意事项

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| ThreadLocal 未清理 | finally 清理 / Reactor Context |
| 只认证不授权 | 每个资源都要授权 |
| 权限写死 | 配置化 + 缓存 |
| 服务间无身份透传 | 透传用户与 traceId |
| 无权限变更失效 | 变更时清缓存 |
| 无审计 | 记录访问决策 |
| 检索后再过滤 | 过滤下推到检索层 |
| 伪造请求头 | 内部调用签名或 mTLS |

## 9.2 上线检查清单

- [ ] 所有资源接口均有授权校验。
- [ ] 认证令牌校验算法、过期时间与受众。
- [ ] 用户上下文在阻塞与响应式场景均正确传递。
- [ ] 检索过滤条件在服务端生成，不信任前端。
- [ ] 服务间调用透传身份与 traceId。
- [ ] 权限缓存有失效策略且键含租户。
- [ ] 访问决策（允许/拒绝）均写入审计。

# 10. 小结

## 10.1 核心结论

- 访问控制 = 认证 + 授权 + 审计。
- 用户上下文传递注意 ThreadLocal 清理与 Reactor Context。
- AI 特有的授权点：模型、工具、知识库、会话、配额。
- 服务间调用透传用户身份与 traceId。
- 权限可缓存，但变更时要主动失效。

## 10.2 常见误区回顾

- 认为“登录了就等于安全”，忽视授权。
- 检索后过滤，导致敏感内容进入模型上下文。
- 信任前端传来的用户标识或过滤条件。
- 内部服务默认可信，缺少服务间鉴权。
