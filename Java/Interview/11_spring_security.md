本篇覆盖 Spring Security 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：总体架构与认证主线、授权模型与配置演进、凭证与登录安全、令牌化与 OAuth2、Web 安全细节与会话。Cookie/Session/CSRF 协议基础见 4_java_web.md，Filter 体系与 CORS 见 8_spring_mvc.md，Boot 整合见 10_springboot.md。

# 1. 总体架构与认证主线

## 1.1 框架定位与核心模型

### 1.1.1 为什么要用 Spring Security？和自研拦截器、和 Shiro 怎么选？

**结论**：Spring Security = 认证 + 授权为核心的**全栈安全框架**——自研拦截器通常只解决了"验证登录态"，而 Security 交付的是一整套容易做错、做错就有安全后果的基建：密码存储标准、会话安全（固定攻击防护）、CSRF/安全响应头、多认证入口编排、RBAC/SpEL 授权表达式、OAuth2/OIDC/SAML 标准协议实现。

**对比自研拦截器**（面试先讲这个才有铺垫）：

| 能力 | 拦截器 + session 自研 | Spring Security |
|------|----------------------|-----------------|
| 登录校验 | ✓（自己写） | 认证过滤器族开箱即用 |
| 密码存储 | 容易写成 MD5/可逆加密（3.1.1） | PasswordEncoder 体系默认 BCrypt |
| CSRF/安全头 | 基本缺位 | CsrfFilter + HeaderWriterFilter 默认开 |
| 会话固定攻击 | 容易漏 | 登录后自动 changeSessionId（3.2.2） |
| 多认证方式共存 | 越写越乱 | Filter + Token + Provider 三件套范式（1.2.3） |
| 标准协议（OAuth2/OIDC/SAML） | 自研 ≈ 造轮子踩坑 | oauth2Login/oauth2ResourceServer/Authorization Server 官方实现 |
| 方法级/动态授权 | if-else 散落业务 | 注解 + SpEL + AuthorizationManager 策略点 |

**与 Shiro 的差异**：Shiro 以 **Session + Realm** 为中心——API 直观（`subject.hasRole`）、上手快、Web 层拦截粒度粗；Security 以 **SecurityContext + Filter 链 + Authentication 对象模型**为中心——抽象多、学习曲线陡，但与 Spring/Spring Boot 生态深度一体化（自动配置、方法安全、OAuth2 全家桶），协议实现与迭代速度明显占优。**趋势判断**：新项目单边倒向 Security（尤其需要对接 OAuth2/OIDC 体系时）；Shiro 常见于存量项目与轻量后台。

**易错**："Security 太重"是 Adapter/XML 时代的历史印象——6.x 组件化 + lambda DSL 下最小配置十行内，Boot 还有默认链（10_springboot.md）。

### 1.1.2 Spring Security 的过滤器链是如何组织的？顺序要点？

**结论**：Web 层安全的本质就是**一条特殊排序的 Filter 链 + 线程上下文**。三级结构：

```text
Servlet 容器（Tomcat FilterChain）
  └ DelegatingFilterProxy（"桥"：从 Servlet 容器接到 Spring 容器里的同名 Bean；war 部署形态）
      └ FilterChainProxy（持有 N 条 SecurityFilterChain，按 request matcher 选**第一条**匹配的链执行）
          └ SecurityFilterChain（一组有序 Security Filter + 该组的授权规则）
```

Boot 内嵌场景由 `springSecurityFilterChain` Bean 注册进内嵌容器（见 10_springboot.md），`DelegatingFilterProxy` 的角色由 `DelegatingFilterProxyRegistrationBean` 承担。

**6.x 关键过滤器顺序**（不必背全，主干记牢）：

| 序 | Filter | 职责 |
|----|--------|------|
| 1 | SecurityContextHolderFilter | 从仓库（session 等）**懒加载** SecurityContext（5.x 的 SecurityContextPersistenceFilter 在 6.0 被替换），请求结束清线程 |
| 2 | HeaderWriterFilter | 安全响应头（X-Content-Type-Options、X-Frame-Options…） |
| 3 | CorsFilter / CsrfFilter | 预检短路、CSRF 令牌校验（都在**认证之前**拦截） |
| 4 | LogoutFilter | 退出处理（默认 POST /logout） |
| 5 | 各认证入口 | UsernamePasswordAuthenticationFilter、BearerTokenAuthenticationFilter、oauth2Login 回调 filter…… |
| 6 | RememberMeAuthenticationFilter → AnonymousAuthenticationFilter | 兜底认证：免登录令牌、匿名用户 |
| 7 | SessionManagementFilter / ConcurrentSessionFilter | 固定攻击策略、并发会话控制 |
| 8 | ExceptionTranslationFilter | **包住下面的授权决策**：捕获认证/授权异常翻译成 401/403（1.2.4） |
| 9 | AuthorizationFilter（5.x 的 FilterSecurityInterceptor 改名换芯） | 授权决策终点，命中 permitAll 才放行到业务 Filter 链与 Servlet |

**三个顺序考点**：① **上下文加载在最前、清理在最后**——整条链都建立在"当前 Authentication 已就位"之上；② **匿名认证是兜底**——走到授权决策时没人是"无 Authentication"，都是真实或匿名 token；③ **ExceptionTranslationFilter 在 AuthorizationFilter 之前**——决策点抛出的异常由外圈翻译，这就是 401/403 统一出口的实现。

**易错**：多链时**一个请求只走一条链**（first match wins），配了第二条链没生效九成是 matcher 顺序问题（@Bean + @Order）；"放行 URL 也会走完全部 filter"——permitAll 只是**授权决策**放行，认证 filter 依旧照常工作（想跳过某些 filter 只能拆链）。

### 1.1.3 Authentication 是什么模型？登录状态到底存在哪？

**对象模型三件套**：
- `Authentication`（接口）：`principal`（身份，登录成功后常是 UserDetails）、`credentials`（凭证，**认证成功后框架默认 eraseCredentials 清掉密码**，不留进会话）、`authorities`（权限集合 `GrantedAuthority`）、`authenticated`（布尔）——未认证请求是一个 `authenticated=false` 的 token（表单 filter 刚构造时），认证成功即换**新的已认证实例**（不可变风格）；
- `SecurityContext`：只装 Authentication 的盒子；
- `SecurityContextHolder`：静态门面，默认 **ThreadLocal** 存放（策略 MODE_INHERITABLETHREADLOCAL/MODE_GLOBAL 可选，异步场景的坑与对策见 5.2.2，ThreadLocal 机制见 1_java.md 3.4）。

**"登录状态存哪"的两层答案**（高频追问的本质）：
1. **请求内**：ThreadLocal——业务代码 `SecurityContextHolder.getContext().getAuthentication()` 随手可取；
2. **跨请求**：由 `SecurityContextRepository` 决定——默认 `HttpSessionSecurityContextRepository`：认证成功的 context 写入 HttpSession（session 又是 JSESSIONID 机制，4_java_web.md 2.1.2）；STATELESS/JWT 场景仓库为 Null，**每请求由认证 filter 重新解析 token 构造 context**，服务端零存储——这就是"有状态 vs 无状态"在 Security 里的准确落点（不是"用不用 session"，而是 **SecurityContextRepository 用不用 HttpSession**）。

**Principal 定制**：要 `authentication.getPrincipal()` 拿到自己的 User 对象——UserDetails 实现里放引用、或自定义 AuthenticationConverter/Provider 决定 principal 内容即可。

## 1.2 认证主线流程

### 1.2.1 表单登录的完整认证流程是怎样的？

**九步主线**：

1. 用户 POST `/login`（默认处理 URL），`UsernamePasswordAuthenticationFilter` 捕获参数（`username`/`password`，可配）；
2. 构造**未认证**的 `UsernamePasswordAuthenticationToken(username, password)`；
3. 调 `AuthenticationManager.authenticate(token)`（实际是 ProviderManager，1.2.2）；
4. 路由到 `DaoAuthenticationProvider`：`retrieveUser()` → **`UserDetailsService.loadUserByUsername()`** 从库/服务加载 `UserDetails`（含密码哈希与 authorities）；
5. 预检 `UserDetailsChecker`：账号是否 enabled / 未锁定 / 未过期；
6. `PasswordEncoder.matches(rawPassword, storedHash)` 比对（3.1.1）；
7. 通过 → 返回**已认证** token（principal=UserDetails、authorities 就位）；同时 RememberMeServices/BadCredentials 计数等事件钩子触发（`AuthenticationSuccessEvent` 可监听做登录日志/次数锁定）；
8. 后处理：session 固定攻击防护（登录后 `changeSessionId()`，3.2.2）→ context 写入仓库 → `SavedRequest` 回跳（登录前想访问的原始 URL 从 RequestCache 取出重放，5.2.1）；
9. `AuthenticationSuccessHandler` 收尾：默认 302 回跳；前后端分离场景换成返回 JSON/token 的自定义 handler。

**失败分支**：抛 `AuthenticationException` → `AuthenticationFailureHandler`；注意默认把"用户不存在"与"密码错"统一成 `BadCredentialsException` 文案（防**用户名枚举**；早期版本 `hideUserNotFoundExceptions` 默认 true，配 false 会泄露账号存在性——面试官爱挖这锹土）。

### 1.2.2 AuthenticationManager 和 AuthenticationProvider 链怎么工作？

**结构**：`ProviderManager`（AuthenticationManager 默认实现）持有 `List<AuthenticationProvider>` + 可选 parent manager。`authenticate()` 遍历：

1. `provider.supports(token.getClass())` 找认领该 token 类型的 Provider；
2. 调用 `authenticate()`——成功立即**短路返回**；
3. 抛 `AuthenticationException` → 记下异常继续下一个 Provider（Provider 类型不匹配抛的 InternalAuthenticationServiceException/ProviderNotFoundException 被忽略）；
4. 全部失败 → 抛最后一个异常（所以"账号密码错"与"短信码错"共存时最终报哪个，取决于 Provider 注册顺序——排障考点）。

**DaoAuthenticationProvider** = UserDetailsService + PasswordEncoder + UserDetailsChecker 的组合，密码认证的标准件。**自定义认证三件套范式**（短信码/邮箱链接/指纹全部套这个模板，追问"怎么加一种登录"就答它）：

```java
// ① 自定义 Token：承载本方式的凭证
class SmsCodeAuthenticationToken extends AbstractAuthenticationToken { /* phone+code */ }
// ② 自定义 Provider：校验逻辑 + supports 认领
class SmsCodeAuthenticationProvider implements AuthenticationProvider { ... }
// ③ 自定义 Filter：从请求提取参数、构造未认证 token、调 manager
class SmsCodeAuthenticationFilter extends AbstractAuthenticationProcessingFilter { ... }
```

**Boot/6.x 接线**：自动配置聚合所有 Provider Bean/AuthenticationProviderConfigurer 进一个 ProviderManager；手工装配就是 `new ProviderManager(providers)` 注册 Bean。

### 1.2.3 密码登录、短信登录、第三方登录如何共存？

**关键洞察：一种登录方式 = 一个入口 Filter + 一种 Token + 一个 Provider；入口在 URL，汇合在同一个 ProviderManager，出口共用同一套成功处理。**

```java
@Bean SecurityFilterChain chain(HttpSecurity http, SmsCodeAuthFilter smsFilter) {
    return http
        .formLogin(f -> f.loginProcessingUrl("/login/password").successHandler(authOut))
        .oauth2Login(o -> o.successHandler(authOut))          // 第三方登录（社交登录）
        .addFilterBefore(smsFilter, UsernamePasswordAuthenticationFilter.class) // 短信三件套的入口
        .rememberMe(r -> r.tokenRepository(persistentTokenRepository()))
        .build();
}
// authOut：AuthenticationSuccessHandler——无论哪种方式成功，统一签发 JWT/建会话、写用户日志
```

- 各入口 filter 监听**各自 URL**（`/login/password`、`/login/oauth2/code/{registrationId}`、`/login/sms`），互不干扰；OAuth2 第三方登录注意与"OAuth2 客户端调用外 API"（oauth2Client，取 token 调外部资源）区分——`oauth2Login` 是"用微信账号登录我"，`oauth2Client` 是"我拿 token 调微信 API"，面试连问必答出这层区别；
- **统一出口**：`AuthenticationSuccessHandler` 一处处理 token 签发/会话建立/审计日志，避免每种登录各写一遍；失败同理共享 FailureHandler；
- **Provider 注册顺序**即失败异常优先级（1.2.2 第 4 条）。

**易错**：给自定义短信 filter 忘了 `.authenticated()`/permitAll 规则——短信发送接口 `/sms/code` 本身要匿名可达，登录接口 `/login/sms` 要 permitAll（filter 处理后不再经过授权规则，但路径仍需匹配链与 CSRF 豁免一致）。

### 1.2.4 认证失败和授权失败如何统一返回 401/403？

**机制在 ExceptionTranslationFilter**（1.1.2 表第 8 行）：它包住 AuthorizationFilter 的执行，catch 两类异常分流：

- `AuthenticationException` → `AuthenticationEntryPoint.commence()`——401 + `WWW-Authenticate` 头（HTTP 语义：请出示凭证，见 28_http.md）；
- `AccessDeniedException` → 看当前身份：**匿名/remember-me 视为未完全认证 → 仍走 EntryPoint（401）**；已认证用户 → `AccessDeniedHandler`（403）。"登录了但没权限"才是 403——这个分流规则就是 401/403 之乱的根源。

**前后端分离标准写法**：

```java
http.exceptionHandling(ex -> ex
    .authenticationEntryPoint((req, res, e) -> write(res, 401, "未登录或凭证已过期"))
    .accessDeniedHandler((req, res, e) -> write(res, 403, "权限不足")));
```

**三个高频事故**：
1. **全部 403**：STATELESS 自建链没配 EntryPoint 时默认 `Http403ForbiddenEntryPoint`——token 过期返回 403，前端误判"已登录"，应显式配 401 EntryPoint；
2. **CSRF 失败 = 403**（CsrfFilter 抛 AccessDeniedException）——别奇怪为什么登录接口 CSRF 校验失败不返回 401；
3. **方法安全抛的 AccessDeniedException** 发生在 Controller/Service 层——沿调用栈穿出 filter 链，照样被 ExceptionTranslationFilter 兜住统一翻译（2.2.1 联动），不要在业务层各自 try-catch 造第二套错误语义。

# 2. 授权模型与配置演进

## 2.1 RBAC 与授权决策

### 2.1.1 RBAC 权限模型如何设计？权限怎么变成 Security 的 Authority？

**三层模型**：用户 →（多对多）角色 →（多对多）权限。经典五表：

```text
sys_user ─ user_role ─ sys_role ─ role_permission ─ sys_permission
                                    （菜单/按钮/API 三类资源同表，resource:action 编码如 sys:user:delete）
```

**接入 Security 的映射**：`loadUserByUsername` 一次把该用户的**角色码与权限码都查出来**转成 `List<GrantedAuthority>`（角色带前缀 `ROLE_ADMIN`，权限码原样 `sys:user:delete`）——之后 URL 规则（2.1.2）与方法表达式（2.2.1）消费的就是这份集合。

**设计纪律**：
1. **权限只挂角色、不直挂用户**（用户-角色、角色-权限两层解耦，直挂会退化成 ACL 乱账；确有例外也要单独审计表）；
2. 角色是"岗位/职责"粒度，权限是"资源:操作"粒度——前端菜单树与 permission 表同源渲染，避免"页面有按钮、接口没权限"的双头维护；
3. **数据权限**（本人/本部门/全公司）不塞进 RBAC 表结构硬做，用方法级表达式带参数判断（2.2.1 的 `#username == principal.username` 形态）；
4. 权限集合过大时**别整个塞进 JWT payload**（膨胀 + 改权限要重签 token——4.1.1），放服务端缓存按用户加载。

### 2.1.2 hasRole 和 hasAuthority 区别？URL 授权规则怎么写？

**一对老陷阱**：`hasRole('ADMIN')` 匹配的是 authority 字符串 **`ROLE_ADMIN`**（表达式内部自动补前缀）；`hasAuthority('ROLE_ADMIN')` 原样比较。所以"角色明明有、hasRole 却不通过"九成是入库时前缀/大小写没对齐约定。规范：角色统一 `ROLE_` 前缀入库，权限码不带前缀，两者都进 authorities 集合。

**6.x 规则写法**：

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/login/**", "/sms/**", "/doc.html").permitAll()
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/orders/**").hasAnyAuthority("order:read", "order:write")
    .requestMatchers(HttpMethod.GET, "/products/**").permitAll()   // 可叠加方法条件
    .anyRequest().authenticated());
```

要点：**自上而下、first match wins**，`anyRequest()` 永远垫底；`requestMatchers` 在 Boot/MVC 存在时用 MvcRequestMatcher 对齐 Spring MVC 的路径解析（6.x 推荐统一用它，替代 5.x 的 antMatchers）。

**追问三件套**：
1. `authenticated()` vs `isFullyAuthenticated()`：匿名 filter 给所有人都造了"anonymous"token——`authenticated()` 对匿名**也成立**吗？不（anonymous 不算 fully，但坑在别处）：真正要防 remember-me 自动登录访问敏感接口时用 `isFullyAuthenticated()`（3.2.1），`rememberMe()`/`isAnonymous()` 是配套谓词；
2. 路径匹配安全边界：`;jsessionid=` 分号参数、大小写、`/admin` 与 `/admin/`（MVC 的尾斜杠匹配在 6 被收紧）——历史上大量"URL 绕过"漏洞都出在**授权匹配语义 ≠ 路由解析语义**，requestMatchers 用 MVC matcher 就是在堵这个洞（延伸 8_spring_mvc.md）；
3. permitAll 与 filter 执行：permitAll 只是跳过授权决策，认证 filter 照样跑（1.1.2 易错）。

### 2.1.3 动态权限（URL-角色关系放数据库、运行时生效）如何实现？

**先泼冷水再给方案（答题姿态）**：静态规则 + 方法级注解能覆盖 80% 需求；把所有接口都搬进数据库规则表，换来的是审计难、匹配语义漂移、运维事故面扩大——"要不要动态"本身是考点。

**6.x 正解 = 自定义 AuthorizationManager**（2.3.1 的机制兑现）：

```java
class DynamicAuthorizationManager implements AuthorizationManager<RequestAuthorizationContext> {
    volatile Map<String, List<String>> rules;            // URL 模式 → 所需权限，本地缓存
    public AuthorizationDecision check(Supplier<Authentication> auth, RequestAuthorizationContext ctx) {
        String url = ctx.getRequest().getRequestURI();
        List<String> need = match(rules, url);            // 最长/有序匹配
        if (need == null) return new AuthorizationDecision(true);   // 无规则=放行（或反向白名单策略）
        return new AuthorizationDecision(hasAny(auth.get(), need));
    }
    public void refresh() { this.rules = loadFromDb(); }  // 管理端变更事件/定时轮询触发
}
// 接线：.anyRequest().access(dynamicAuthorizationManager)
```

旧 5.x 写法（自定义 `FilterInvocationSecurityMetadataSource` + `AccessDecisionManager`）在存量项目仍会见到——概念对应：metadata source ≈ 规则缓存，access decision manager ≈ AuthorizationManager。

**一致性细节**：多实例下规则变更要广播刷新（消息/版本号比对，MQ 见 13_rocketmq.md）；刷新窗口内新旧规则并存是可接受语义（权限收紧场景则反过来——先发"拒绝"再发"放行"）。

## 2.2 方法级安全

### 2.2.1 @PreAuthorize/@PostAuthorize 怎么用？底层机制与失效场景？

**启用**：6.x `@EnableMethodSecurity`（默认开启 prePost/secured/jsr253，替代旧 `@EnableGlobalMethodSecurity(prePostEnabled=true)`）。

**四种注解**：

```java
@PreAuthorize("hasRole('ADMIN')")                                   // 进方法前：角色
@PreAuthorize("#username == authentication.name")                   // 参数与当前用户比对
@PreAuthorize("@orderService.canEdit(#id, authentication.principal.id)") // 调 Spring Bean 判断
@PostFilter("filterObject.ownerId == authentication.principal.id")  // 返回值集合逐元素过滤
@PostAuthorize("returnObject.ownerId == authentication.principal.id")   // 返回值整体校验
@Secured("ROLE_ADMIN")  /  @RolesAllowed("ADMIN")                   // 纯角色串，无 SpEL
```

**底层机制**：方法调用被 **AOP 拦截器**（`AuthorizationManagerBeforeMethodInterceptor`/`After`，6.x 重构后的形态）截住，SpEL 在参数上下文求值（#param、#root、`@beanName`），决策交给 AuthorizationManager——表达式返回 false 抛 `AccessDeniedException`，穿出业务栈由 ExceptionTranslationFilter 统一翻译 403（1.2.4 第 3 条）。理解到"方法安全 = AOP + SpEL + AuthorizationManager"，三个失效场景就能自己推出来：

1. **自调用失效**：同类内部 `this.method()` 不走代理——与 Spring AOP 自调用同一根因（7_spring.md）；解法：拆分 Bean 或经 AopContext；
2. **非 public/接口代理**：JDK 动态代理要求接口方法注解放实现侧的对应位置——注解放错边等于没放；
3. **SpEL 里调的 Bean 又依赖参数外的状态**——表达式求值时机在方法**前**，要"执行后才能判断"（如新建记录归属）用 @PostAuthorize；批量数据裁剪用 @PostFilter（注意大列表的过滤成本在返回侧）。

**性能/安全提醒**：`@PreAuthorize` 每调用一次求值一次，重表达式（查库的 `@service`）是热点；数据行级过滤优先在 SQL 带条件，@PostFilter 只兜小结果集。

## 2.3 决策与配置演进

### 2.3.1 AccessDecisionManager/Voter 是什么？为什么被 AuthorizationManager 取代？

**旧世界**：授权决策 = Manager 收票 + Voter 投票：`RoleVoter`（ROLE_ 前缀匹配）、`AuthenticatedVoter`（IS_AUTHENTICATED_* 谓词）等各自投 grant/deny/**abstain**，Manager 按策略汇总——`AffirmativeBased`（一票同意即过，默认）、`ConsensusBased`（多数）、`UnanimousBased`（一票反对即否）。URL（FilterSecurityInterceptor）与方法安全（拦截器）共用这套。

**问题**：决策逻辑被拆在 Voter 集合与策略组合两层，测试难、扩展点晦涩；投票语义对多数业务是过度设计。

**新世界（5.8 引入、6.0 全面接管）**：单接口 `AuthorizationManager<S>`——`check(Supplier<Authentication>, S object)` 返回 `AuthorizationDecision(布尔)`；URL 侧 `AuthorizationFilter`、方法侧拦截器都改为消费它，**一个抽象贯穿两种入口**；配置直接 lambda：

```java
.requestMatchers("/internal/**").access((auth, ctx) ->
        new AuthorizationDecision(hasRole(auth.get(), "SERVICE")))
```

迁移映射：自写 AccessDecisionVoter → 改写为 AuthorizationManager（动态权限即 2.1.3）；`access()` 表达式字符串配置 → 仍走 SpEL-based 实现。答题收口在"**架构简化 + 决策点统一 + 可测性**"三个词。

### 2.3.2 WebSecurityConfigurerAdapter 为什么废弃？6.x 配置写法发生了什么变化？

**配置形态三时代**：XML → **Adapter 继承**（5.x：`extends WebSecurityConfigurerAdapter` 覆写 configure）→ **组件化 Bean**（5.7 引入、6.0 移除 Adapter）：

```java
@Configuration @EnableWebSecurity            // Boot 下连这个都可省（自动配置）
class SecurityConfig {
    @Bean SecurityFilterChain filterChain(HttpSecurity http) throws Exception { ... return http.build(); }
    @Bean AuthenticationProvider provider(...) { ... }      // 依赖全部走 Bean 注入
    @Bean PasswordEncoder passwordEncoder() { return PasswordEncoderFactories.createDelegatingPasswordEncoder(); }
}
```

**废弃原因**：Adapter 是"一个配置类 = 一条链"的模板方法——链不该靠类继承组装；组件化后多个 `SecurityFilterChain` Bean 用 `@Order` 排优先级（不同 URL 分组不同策略成为一等公民），`HttpSecurity` 的构造依赖（AuthenticationManager 等）全部容器内拼装，Boot 自动配置也能无缝参与。

**6.x 的 API 变化清单（存量迁移题，报得出五条即高分）**：
1. Adapter → `SecurityFilterChain` Bean（上）；
2. `authorizeRequests()` → `authorizeHttpRequests()`（默认走 MVC matcher，语义与路由对齐），`antMatchers` → `requestMatchers`；
3. `.and()` 拼接的流式 → **lambda DSL**（`http.csrf(csrf -> csrf.disable())`），配置树不再"写着写着忘了自己在哪层"；
4. `FilterSecurityInterceptor` → `AuthorizationFilter`，`AccessDecisionManager` → `AuthorizationManager`（2.3.1）；
5. `SecurityContextPersistenceFilter` → `SecurityContextHolderFilter`（懒加载 context，5.2.1）；`@EnableGlobalMethodSecurity` → `@EnableMethodSecurity`；
6. 小项：rememberMe 静态 key 取消（3.2.1）、`logout()` 默认仅 POST、`httpBasic` 与表单登录 Boot 默认链并存（10_springboot.md）。

**易错**：5.x 老教程代码贴进 6.x 工程编译不过——面试被问"接到一个 Spring Security 老项目怎么办"，答"先按上面映射表升级到组件化写法，业务规则不动"。

# 3. 凭证与登录安全

## 3.1 密码存储

### 3.1.1 用户密码应该如何存储？BCrypt 的原理是什么？算法怎么平滑升级？

**结论**：密码只能**单向、加盐、慢速哈希**存储——需求三件：不可逆（拖库看不出原文）、抗彩虹表（每用户独立盐）、抗算力（高成本慢哈希让 GPU 批量爆破贵到放弃）。

**为什么 MD5/SHA 系列不行**：它们是为"快"设计的通用摘要——现代 GPU 单卡每秒百亿次量级；无盐则同密码同密文、彩虹表直接查表；加盐也只是挡住彩虹表，挡不住慢速穷举的降维打击。"MD5+盐"只是及格线的历史方案，现在答"用 SHA-256 存密码"在面试里等于不会。

**BCrypt 机制**（Spring Security 默认推荐）：基于 Blowfish 的口令哈希——`cost` 参数决定 2^cost 轮密钥编排迭代（**计算成本指数可调**，硬件变快就加 cost 不变格式）；随机盐内置生成；输出自描述格式：

```text
$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
 └版本 └cost └────22 位盐────────┘└────────31 位哈希──────────┘
```

`matches(raw, encoded)` 无需额外传盐：从 encoded 提取版本/cost/盐重算再比——**同一密码每次 encode 结果都不同**（盐随机），验证靠库里存的这份自描述串。已知边界：输入超 72 字节截断（超长口令先 SHA-256 预哈希再喂 BCrypt）；cost 10~12 之间取吞吐平衡（一次登录几十到几百毫秒是正常且应有限流兜底的，见下）。

**同类与替代**：SCrypt/Argon2（内存硬，抗 GPU 更强，Spring Security 提供 `Argon2PasswordEncoder`；Argon2 是口令哈希竞赛冠军、新项目首选之一）、PBKDF2（NIST 也认可，但同算力下防御性弱于前两者）。

**DelegatingPasswordEncoder——算法升级正解**（追问"老项目一堆 MD5 怎么办"）：Boot 默认 `PasswordEncoderFactories.createDelegatingPasswordEncoder()` 返回**代理编码器**：存储格式带算法前缀 `{bcrypt}$2a$10$...`，`{noop}` 表明文、`{md5}` 表历史哈希——matches 按前缀路由到对应编码器，**新旧算法密文共存**；encode 一律用默认算法（bcrypt）；配合"登录成功即重哈希回写"（upgradeEncoding 判断 + 更新库），存量 MD5 随用户登录自然清零，不需要强制全员改密。

**配套纪律**：登录/注册/改密接口限速 + 失败计数锁定（防撞库）、密码策略（长度下限重于复杂度花活）、禁止把密码写进任何日志（`eraseCredentials` 的另一面，1.1.3）。

## 3.2 登录会话安全

### 3.2.1 Remember Me（免登录）如何实现？两种方案差在哪？

**场景**：勾选"记住我"后关闭浏览器再访仍登录——本质是**凭证持久化在客户端**、每次请求自动重建 Authentication，与会话超时无关。链路：`RememberMeAuthenticationFilter`（1.1.2 表）在无认证上下文时读 cookie → `RememberMeServices.autoLogin()` 解析出 principal 与 authorities → 构造 `RememberMeAuthenticationToken` 交给 AuthenticationManager 的 `RememberMeAuthenticationProvider` 认领——之后与正常登录同一条上下文通路。

**两档实现**：

| 方案 | 机制 | 弱点 |
|------|------|------|
| TokenBasedRememberMeServices（默认） | cookie 内容 `用户名\|过期秒\|签名(HMAC(secretKey,...))`，**服务端零存储** | 无法撤销（注销/改密后 cookie 仍有效直到过期）；泄露即冒用；密钥硬编码是事故源 |
| **PersistentTokenBasedRememberMeServices（推荐）** | cookie 只放 `series\|token` 随机对；服务端 `persistent_logins` 表存系列号→token；每次自动登录**换发新 token**（rolling），校验失败（token 不匹配但系列号在）判定 cookie 被复制 → 删除该系列全部记录 | 可撤销、被盗可检出——多一次 DB/缓存查询 |

**细节考点**：
1. 有效期默认两周（`tokenValiditySeconds`）；cookie 是 HttpOnly（4_java_web.md 2.2.1）；
2. **Remember-me ≠ 完全认证**：自动登进来看敏感页面是风险——`isFullyAuthenticated()`/`@PreAuthorize("isFullyAuthenticated()")` 要求真实登录（3.2.2 的 step-up 思路）；
3. 6.x 起不再需要手工配 key（自动随机生成并持久化在 token repository——旧教程"固定 key"写法已过时且危险）；
4. remember-me 的 authorities 由 userDetailsService 现查（改权限下次自动登录即生效）还是旧快照，取决于服务实现——答"权限改了为什么免登录还带着旧权限"时的解释点。

### 3.2.2 什么是会话固定攻击？Spring Security 如何防护？并发会话怎么控制？

**会话固定攻击**（Session Fixation，先讲攻击链）：攻击者先拿到一个合法 JSESSIONID（诱导点击带 `;jsessionid=xxx` 的链接/子域种 cookie）→ 受害者用**这个 id** 登录成功 → 攻击者手里同一 id 的会话已变成"已登录"，直接冒用身份（4_java_web.md 2.1.2 的传输机制被反向利用）。

**Security 默认防御**：登录成功瞬间由 SessionManagementFilter 执行 `SessionFixationProtectionStrategy`——6.x 默认策略 **changeSessionId()**：调用 Servlet 3.1 的容器 API 原地换新会话 id 并保住会话属性（Tomcat 实现见 5_tomcat.md 会话章节）；历史上有 `migrateSession`（新建 session 复制属性——旧容器时代的方案，属性序列化不全会丢数据）与 `none`（禁用，等于裸奔）。**无状态 JWT 场景没有会话，天然无此问题**——STATELESS 配置下这套策略不生效。

**并发会话控制**（"一个账号最多几处登录"）：

```java
http.sessionManagement(sm -> sm.maximumSessions(1)
        .maxSessionsPreventsLogin(false)      // false=新登录踢旧的；true=拒绝新登录
        .expiredSessionStrategy(event -> ...)); // 被踢出的响应（401/跳转）
```

机制：`SessionRegistry` 记录 principal→session 映射，登录时计数，超限按策略处理；`ConcurrentSessionFilter` 每次请求检查会话是否已被标记过期。两个追问必答点：① **需要容器发布 HttpSessionEvent**（Boot 内嵌注册 `HttpSessionEventPublisher` Bean），否则注销感知不到；② SessionRegistry 是**单实例内存**——多实例集群下"每个节点各管各的"，全局并发控制要么交给 Spring Session 的会话仓库（Redis 支持按 principal 查会话，见 27_redis.md），要么接受单节点语义。

# 4. 令牌化与 OAuth2

## 4.1 JWT

### 4.1.1 JWT 的结构与签名原理？HS256 和 RS256 怎么选？

**结构**：`Header.Payload.Signature` 三段 **Base64URL** 拼接——

- Header：`{"alg":"HS256","typ":"JWT","kid":"..."}`（kid 供验签方挑公钥，4.2.4 轮换用）；
- Payload：claims 三档——注册字段 `iss`(签发者)/`sub`(主体)/`aud`(受众)/`exp`/`nbf`/`iat` + 公共字段（userId、roles 等业务声明）+ 私有字段；
- Signature：`HMAC-SHA256(header + "." + payload, secret)`（HS 系）或非对称私钥签名（RS/ES 系）。

**Base64 不是加密**——payload 任何人可解码阅读（**别放敏感信息**；确需保密用 JWE 加密套娃，很少用），签名只保证**防篡改**，两者常被混为一谈是面试第一送分点。

**HS256 vs RS256（分布式下的选型是核心追问）**：

| 维度 | HS256（对称） | RS256/ES256（非对称） |
|------|--------------|----------------------|
| 验证 | 验签方持有同一 secret | 验签方只需公钥 |
| 适用 | 签发=验证同一系统 | 签发方（认证中心）独占私钥，**N 个资源服务各自拿公钥验签**，secret 不用到处发 |
| 泄露半径 | 任一验证方泄露 = 全局可伪造 | 公钥泄露无损（只能验不能造） |
| 成本 | 更快 | 签名稍慢、需 JWKS 端点与轮换 |

微服务/多方验证场景选 RS256（公钥经 `/oauth2/jwks` 分发，4.2.4）；单体自用 HS256 足够。

**攻击面清单**（"JWT 有什么风险"必答）：`alg:none` 降级与 HS/RS 混淆攻击（老库才中招——jjwt/nimbus 显式钉死允许的算法即可防）、payload 泄敏、token 外泄后的有效期内重放（对策=短时效+吊销，4.1.2）、**存放位置之争**：localStorage 可被 XSS 读走 vs HttpOnly cookie 免疫 XSS 窃取但把 CSRF 防线拉回来（5.1.1 联动，配合 SameSite，4_java_web.md 2.2.1）——没有银弹，按威胁模型选。

### 4.1.2 JWT 无状态怎么实现"退出登录"？改密后旧 token 怎么立刻失效？

**矛盾本质**：无状态 = 服务端不存 token → 默认"注销"只是客户端删 token，泄露的 token 活到过期。主动吊销都要**在服务端加回一点状态**，四种方案按代价从低到高：

1. **客户端清理 + 短 TTL 接受窗口**：access token 只给 5~15 分钟——注销后最坏被冒用几分钟，多数业务直接接受（4.1.3 双 token 的基石）；
2. **jti 黑名单**：登出时把 token id（或整串哈希）写 Redis，TTL=剩余有效期；每请求验签后多查一次黑名单——精确但每次注销一写、每请求一读；
3. **用户级失效戳/版本号**：`user:tokenVersion` 或"最后登录时间"存 Redis/DB，token 里的 `iat` 早于它即全部作废——**改密/发现盗号时一键踢下线所有设备**；代价是粗粒度（同用户旧新 token 一起死，可用 iat+版本号区分）；
4. **干脆改回有状态**：token 只当索引，会话存 Redis（Spring Session 路线，5.2.1）——放弃了无状态的好处，换来完全控制。

组合拳常态：**短 access + 可吊销 refresh（4.1.3）+ 黑名单兜底高敏操作二次校验**。Redis 实现细节（key 设计、TTL）见 27_redis.md；"黑名单每次查询会不会成瓶颈"——pipeline/本地短缓存（剩余有效期很长的才需要实时查）缓解，答出权衡即可。

### 4.1.3 双 Token（Access + Refresh）机制如何设计？Refresh Token Rotation 是什么？

**分工**：AccessToken 短期（分钟级）随请求携带、验签即用、不进库；RefreshToken 长期（周级）**只用于换新 token**、不出现在业务请求头、服务端存储可撤销。

**流程**：登录发对 → access 过期，客户端拿 refresh POST `/oauth2/token`（或自定义 `/auth/refresh`）→ 服务端查 refresh 存储校验（存在？未撤销？未过期？绑定设备指纹？）→ 发新 access（+**新 refresh，旧的立即作废 = Rotation**）→ 客户端原子替换。

**RTR 的意义与重用检测**（OAuth2 安全 BCP 的核心条款）：refresh 只给一次用——若旧 refresh 被再次使用，要么客户端 bug 要么**被偷**：攻击者用副本时真客户端已轮换掉它 → 服务端检测到"已消费 token 的复用" → 判定整条 token 家族泄露 → 撤销该用户全部 refresh 并强制重登。没有 rotation，被盗 refresh 可以在有效期内无声续命。

**工程细节**：
1. refresh 存储：DB 表或 Redis（key=用户/设备），**永不写日志**；前端存放：能放 HttpOnly cookie 最好，SPA 存内存+静默刷新次之，localStorage 最弱（4.1.1 存放之争）；
2. **并发刷新竞态**：多个请求同时 401 同时刷 → 客户端单飞（一把锁一个刷新 promise），服务端容忍"刷新请求重放窗口"或配合 rotation 的重用检测（实现时把"自己刚轮换掉的极短时间窗"列为豁免，否则误杀全端）；
3. 登出/改密联动 refresh 撤销——refresh 一死，access 最多活几分钟（4.1.2 方案 1 兜底）；
4. 时钟偏移：exp 校验留 30~60s leeway；分布式环境统一 NTP。

## 4.2 OAuth2 与单点登录

### 4.2.1 OAuth2 有哪些授权模式？授权码模式的完整流程？

**四角色**：资源所有者（用户）、客户端、授权服务器、资源服务器——OAuth2 管的是"**第三方 App 能以多大权限、多长期限访问用户在资源服务器的数据**"（授权），身份认证是 OIDC 的事（4.2.2）。

**模式清单（2026 口径）**：

| 模式 | 场景 | 现状 |
|------|------|------|
| 授权码（Authorization Code） | 有后端的 Web 应用；**+PKCE 后公开客户端也用它** | 首选 |
| 客户端凭证（Client Credentials） | 服务对服务，无用户参与 | 常用 |
| 设备码（Device Code） | TV/CLI 等无浏览器输入 | 特定场景 |
| 隐式（Implicit） | SPA 直出 token | **OAuth 2.1 移除**（token 暴露在 URL/历史） |
| 密码（Resource Owner Password） | 客户端直接收用户名密码 | **OAuth 2.1 移除**（无法推行 MFA、扩大凭证暴露面） |

**授权码流程**（背五步 + 每步的防御点）：

```text
① 跳转：/oauth2/authorize?response_type=code&client_id=..&redirect_uri=..&scope=read&state=xyz&code_challenge=..&code_challenge_method=S256
② 用户在授权服务器登录并同意授权页
③ 回调：redirect_uri?code=AUTH_CODE&state=xyz   ← 授权码经浏览器，一次性、分钟级短命
④ 后端换token：POST /oauth2/token（Basic client_id:client_secret 或私钥），带 code + code_verifier + redirect_uri
⑤ 返回 {access_token, refresh_token, expires_in, scope}；此后带 access 调资源 API
```

**安全设计逐点**：授权码走前端但**换 token 走后端直连**（secret 不过浏览器）；`state` 防授权流程 CSRF（回调比对）；`redirect_uri` 精确白名单匹配（防码被劫持到攻击者域）；**PKCE**：生成随机 verifier、`challenge=SHA256(verifier)` 随授权提交、换 token 时出示 verifier——攻击者即使截到授权码，没有 verifier 也换不了 token，**公开客户端（SPA/移动端存不住 secret）的安全基石**，OAuth 2.1 对所有客户端强制。

**易错**："授权码模式为什么安全"要落到"码短命+一次性+后端交换+state/PKCE 两把锁"，只答"比隐式多一步"不及格；client_credentials 的 token 拿不到任何"用户"身份——问"服务间调用带不带用户信息"要答"要透传最终用户就得走授权码/on-behalf-of 或内部身份头方案"。

### 4.2.2 OAuth2 和 OIDC 是什么关系？单点登录（SSO）怎么实现？

**一句话定位**：OAuth2 是**授权框架协议**（access token 回答"你能调哪些 API"），OIDC = 在其上补一层**身份认证协议**——id_token（JWT：`iss/sub/aud/exp/iat + nonce`，声明"授权服务器证明此刻这个用户是谁"）+ UserInfo 端点 + Discovery（`/.well-known/openid-configuration` 自描述端点）。"用 GitHub 登录"用的是 OIDC（或类 OIDC 扩展），"允许我的 App 读你的 GitHub Issue"用的是 OAuth2——面试把两者混称"OAuth2 登录"时要纠出来。nonce 的作用：防 id_token 注入/重放（客户端生成、回调比对）。

**SSO 是目标不是协议**：一次登录，全域互认。三条实现路线：
1. **同域共享会话**：cookie 域写父域（`.example.com`，4_java_web.md 2.1.1）+ 集中会话存储（Redis，27_redis.md）——内部系统最省事；
2. **标准协议中心**：OIDC（授权码模式充当"认证中心"：各应用跳认证中心 authorize，认证中心有自己的会话则直接发码免密，回跳即登录）/SAML（企业老仓库）/CAS（国内教育内网常见）——跨应用、跨团队的正解；
3. **网关统一**：入口层校验 token 后向下游透传身份头（内部信任签名头/双向 TLS），业务服务不再各自处理登录（12_nginx.md、19_spring_cloud_gateway.md）——微服务内部与 OIDC 对外常常混搭。

**Spring 生态落点**：接入方 `oauth2Login`（自己是 OIDC 客户端，对接微信/钉钉/GitHub），建中心方 Spring Authorization Server（4.2.3），资源侧 Resource Server 验 token（4.2.4）。

### 4.2.3 Spring Authorization Server 是什么？和旧的 spring-security-oauth2 什么区别？

**历史**：`spring-security-oauth2`（`@EnableAuthorizationServer/@EnableResourceServer/@EnableOAuth2Client` 三注解时代）已废弃停更——它是 OAuth 2.0 草案年代的产物，配置魔法多、与新版 Security 链难以共存。**替代品 = Spring Authorization Server**（2020 立项、现已稳定）：完全基于 Security 组件化模型（它自己就是一条 SecurityFilterChain + 一组 OAuth2 端点），合规 OAuth 2.1（强制 PKCE、禁密码模式等，4.2.1）、OIDC 1.0 内置。

**核心端点**：`/oauth2/authorize`、`/oauth2/token`、`/oauth2/jwks`、`/oauth2/revoke`、`/oauth2/introspect`、OIDC 的 discovery/logout/userinfo。**核心模型**：`RegisteredClient`（client_id/secret/授权类型/scope/redirect_uri 的注册表，内存/JDBC 仓库）、`JWKSource`（签名密钥与轮换）、`OAuth2TokenGenerator`（JWT 或 opaque，可选）。

**选型题**："要不要自建认证中心？"——需要统一登录 + 开放平台（三方接入）+ 多应用授权治理 → SAS 自建；只是接现有 IdP（Keycloak/Auth0/企业 AD/微信）→ `oauth2Login` + Resource Server 就够，别造中心。存量 `@EnableAuthorizationServer` 项目的迁移按"重写端点配置 + RegisteredClient 数据搬迁"规划，不是一键开关。

### 4.2.4 OAuth2 Resource Server 如何验证 JWT？Claims 怎么变成权限？

**配置即验证**（Boot 属性驱动）：

```yaml
spring.security.oauth2.resourceserver.jwt.issuer-uri: http://auth-server:9000
# 或直接 jwk-set-uri: http://auth-server:9000/oauth2/jwks 跳过发现元数据
```

自动配置产出 `NimbusJwtDecoder` + 默认 `oauth2ResourceServer` 链（10_springboot.md）——**自研 JwtAuthenticationFilter 的大多数场景可以退役了**（4.3.1 对比）。

**验证清单**（回答"验 token 验什么"）：① **签名**：按 header.kid 从 JWKS 取公钥（issuer-uri 走 OIDC Discovery 拿 jwks_uri；密钥轮换/预取有缓存策略，RS256 场景 4.1.1）；② `exp/nbf` 时效；③ `iss` 签发者；④ `aud` 受众（防"别的资源服务的 token 来串门"）；⑤ 可选黑名单（4.1.2）。

**Claims → Authorities**：默认 `JwtAuthenticationConverter` 把 `scope` 声明逐条转成 `SCOPE_read` 形式的权限 → 规则写 `hasAuthority("SCOPE_read")`；自定义 converter 从业务 claims（roles/perm）映射成 `GrantedAuthority`；principal 默认是 `Jwt` 对象（`@AuthenticationPrincipal Jwt jwt` 或自定义 claim set 映射成用户 DTO）。

**opaque token（不透明引用令牌）**：token 是随机串，资源服务器调授权服务器 `/oauth2/introspect` 实时校验并取声明——**天生可吊销**（无状态优势反过来变成"每请求一次远程调用"）；配 introspection 结果缓存折中。**选型对照**：自包含 JWT = 验证零依赖/吊销难；opaque = 吊销即时/依赖授权服务器可用性——高敏低频资源用 opaque 或短命 JWT + 黑名单。

## 4.3 整合路径

### 4.3.1 前后端分离项目集成 JWT，两条路线怎么选？

**路线 A：OAuth2 Resource Server（标准姿势，优先）**：认证中心出 RS256 JWT → 各服务 `spring.security.oauth2.resourceserver.jwt.*` 三行配置 + 权限 converter（4.2.4），链上 `STATELESS + csrf 关闭（token 在 header 才关，5.1.1）+ 401 EntryPoint（1.2.4）`。得到：端点/密钥轮换/claims 转换/异常翻译全部框架托管。

**路线 B：自研认证 Filter（内部签发 JWT、无认证中心时）**：

```java
class JwtAuthFilter extends OncePerRequestFilter {          // 只翻译、不决策
    protected void doFilterInternal(req, res, chain) {
        String token = bearer(req);                          // 提取 Authorization 头
        if (token != null && jwtService.verify(token)) {     // 验签+时效
            var auth = new UsernamePasswordAuthenticationToken(
                    jwtService.toPrincipal(token), null, jwtService.toAuthorities(token));
            SecurityContextHolder.getContext().setAuthentication(auth);  // 只负责塞 context
        }
        chain.doFilter(req, res);                            // 授权交给 AuthorizationFilter
    }
}
```

配置骨架见 1.2.3/5.1.1 各题。**关键原则：filter 只做"身份翻译"，鉴权一律留在授权层**——在 filter 里顺手判权限是第二套授权体系萌芽，规则碎片化的起点。

**路线 B 的五个经典坑**：① 忘设 authorities（token 有效、`hasRole` 全挂——principal 对了权限空）；② 解析异常被吞（应放行给 EntryPoint 统一 401，而非自己 catch 返回）；③ token 在 Cookie 却关了 CSRF（5.1.1 前提不成立）；④ OncePerRequestFilter 与链顺序（addFilterBefore 到 UsernamePasswordAuthenticationFilter.class 之前）；⑤ 微服务下每服务各自验签的成本 vs 网关统一验签 + 内部可信身份头（19_spring_cloud_gateway.md、15_spring_cloud.md）。

**答题收口**："能走标准走标准（路线 A），Resource Server 的 JWT Decoder 甚至不要求你有 Authorization Server（给个 JWKS/secret 就能用）；自研 filter 留给'认证中心是外购魔改/claims 结构怪异'的兜底场景。"

# 5. Web 安全细节与会话

## 5.1 CSRF 与 CORS

### 5.1.1 Spring Security 如何防 CSRF？什么条件下才能 `csrf().disable()`？

**攻击模型回顾**（原理与通用防御见 4_java_web.md 2.2.2）：浏览器对"自动携带凭证"（Cookie）的请求不设防，第三方页面可诱发带身份的写操作。**Security 的答案 = 同步令牌模式（Synchronizer Token Pattern）**：服务端把一次性随机 token 存进会话，表单/AJAX 提交必须原样带回，攻击者跨源读不到 token（同源策略反向保护了你）。

**机制拆解**：
- `CsrfTokenRequestAttributeHandler` + `DeferredCsrfToken`：GET 渲染页面时**惰性生成** token 存 HttpSession；Thymeleaf 模板自动注入 `_csrf` 隐藏域；
- 校验在 **CsrfFilter**（1.1.2 表第 3 行，认证之前）：比对请求参数 `_csrf` 或请求头 `X-CSRF-TOKEN` 与会话内值；**安全方法（GET/HEAD/OPTIONS/TRACE）不校验**，写方法必查；失败抛 AccessDeniedException → 403（1.2.4 事故 2）；
- **SPA 变体 = Cookie 令牌模式**：`CookieCsrfTokenRepository.withHttpOnlyFalse()` 把 token 写进 `XSRF-TOKEN` cookie（前端 JS 可读），axios/fetch 封装回带 `X-XSRF-TOKEN` 头——cookie 里是"钥匙"、会话里是"锁"，攻击者页面能诱发携带 cookie 却**读不到**别的域下的 cookie 内容，闭环成立。

**`csrf.disable()` 的前提核对清单**（"JWT 就能关 CSRF"是半句对的，必须补全）：
1. 鉴权凭证**只走 `Authorization` 头**，浏览器不会自动携带 → 跨站请求带不上身份，攻击无从谈起；
2. **反例必举**：JWT 放 **Cookie**（哪怕 HttpOnly）= 自动携带凭证 = CSRF 暴露面原样存在，关了就是裸奔——token 放哪与要不要关 CSRF 是联动决策（4.1.1 存放之争）；
3. 浏览器 Basic/Digest 认证同样自动携带——不能关；
4. 纯 OAuth2/OIDC 登录流程的 CSRF 风险由 `state`/nonce 参数承担（4.2.1/4.2.2），与此开关无关；
5. 关掉后登录表单本身失去保护——session 方案下登录前 CSRF（攻击者预埋会话再钓账号）需要会话固定防护兜着（3.2.2）。

### 5.1.2 Spring Security 下 CORS 如何配置？和 MVC 的 CORS 怎么分工？

**协议基础**（简单/预检、preflight 规则见 8_spring_mvc.md 4.2.2、28_http.md）：跨源写请求或带自定义头的请求，浏览器先发 `OPTIONS` 预检，按响应头 `Access-Control-Allow-Origin/Credentials/Headers/Methods` 放行真实请求。

**Security 接法**：

```java
http.cors(Customizer.withDefaults());   // 自动发现容器里的 CorsConfigurationSource Bean
@Bean CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration c = new CorsConfiguration();
    c.setAllowedOriginPatterns(List.of("https://*.example.com")); // credentials=true 时禁用 "*" 字面量
    c.setAllowCredentials(true);
    c.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));
    c.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-XSRF-TOKEN"));
    c.setExposedHeaders(List.of("Content-Disposition"));          // 前端要读下载文件名必须 expose
    c.setMaxAge(3600L);                                           // 预检缓存，减 OPTIONS 量
    return new UrlBasedCorsConfigurationSource() {{ registerCorsConfiguration("/**", c); }};
}
```

**分工与顺序考点**（这题的深度所在）：`.cors()` 会把 CorsFilter 插到链**前部**——预检请求（不带任何身份凭证的 OPTIONS）**由 CorsFilter 短路应答，不进认证/授权**；如果把 CORS 只配在 MVC 层（`@CrossOrigin`/WebMvcConfigurer，8_spring_mvc.md 4.2.3），预检 OPTIONS 会先撞上 Security 的授权规则被 401/403 拒掉——**"GET 能跨域、POST 预检就挂"九成是这个错位**。规则：**Security 链上必须有一个能应答预检的 CorsFilter**，MVC 层的配置可作为补充（两处并存时注意响应头不重复添加）。

**易错三件**：① `allowCredentials(true)` + `allowedOrigins("*")` 直接抛异常（规范禁止），通配要用 `allowedOriginPatterns`；② 前端 fetch 忘配 `credentials: 'include'`/`mode: 'cors'` 表现"服务端明明配了还是不携带 cookie"；③ 生产更优解常是**入口网关统一管 CORS**（一处配置、下游全免，12_nginx.md、19_spring_cloud_gateway.md），服务层配置只兜内网直连场景。

## 5.2 会话策略与上下文传递

### 5.2.1 SessionCreationPolicy 四种策略各是什么？SecurityContextPersistenceFilter 和 RequestCache 管什么？

**四策略**（决定 Security 对 HttpSession 的态度）：

| 策略 | 语义 | 场景 |
|------|------|------|
| ALWAYS | 总是创建会话 | 传统 JSP 应用 |
| IF_REQUIRED（默认） | 需要时才建 | 表单登录 |
| NEVER | 自己不建，但别人建了就用 | 混合：会话由 Spring Session 等外管 |
| **STATELESS** | 完全不用会话 | JWT/每请求认证 |

**实现层真相**：策略落到 `SecurityContextRepository` 的选择——STATELESS ≈ `NullSecurityContextRepository`（不读不写会话，认证完全靠请求内 token 重放），其余用 `HttpSessionSecurityContextRepository`。**6.x 的懒加载改进**：旧 `SecurityContextPersistenceFilter` 每请求**无条件**触碰会话（哪怕只读，也导致响应 Set-Cookie、会话创建计数虚高），新 `SecurityContextHolderFilter` 把 context 包成 `Supplier`，**没人真正需要才接触会话**——"配了 JWT 为什么还发 JSESSIONID"的老事故由此根治（追问"6.x 为什么换 filter"就答这个）。

**RequestCache 配套**（"登录后回跳原页面"的机制，5.2.1 与 1.2.1 第 8 步是一件事的两半）：未认证请求被 EntryPoint 打断前，`HttpSessionRequestCache` 把原始 URL 存进会话；登录成功 `SavedRequestAwareAuthenticationSuccessHandler` 取出并 302 回跳。STATELESS 下缓存没处存（无会话），回跳语义自然消失——API 前端靠自行保留"登录后要去的页面"状态。

**易错**：STATELESS 配 `formLogin()` 自相矛盾（表单登录的会话与 CSRF 都落空）；`NEVER` 不等于无状态——会话若被别处创建照样读写；Spring Session 共享会话时 Security 仍用 HttpSession API，只是底层换 Redis（27_redis.md），策略语义不变。

### 5.2.2 @Async/线程池下 SecurityContext 怎么传递？ThreadLocal 不清理会出什么事？

**问题根源**：context 存 ThreadLocal（1.1.3），Worker 线程看不见请求线程的值——`@Async`/自投线程池里 `authentication = null`。

**方案对比（按推荐度）**：
1. **装饰执行器（正解）**：`DelegatingSecurityContextAsyncTaskExecutor` 包装线程池 Bean（Boot 下配成 `applicationTaskExecutor` 即接管 @Async），任务提交时快照 context、执行线程内 set/finally clear——**复用线程池也安全**；变体 `DelegatingSecurityContextRunnable/Callable` 单任务包装、`TaskDecorator` 挂进 ThreadPoolTaskExecutor（10_springboot.md）；
2. 手工透传：提交前 `SecurityContextHolder.getContext()`，任务内 set + **finally clear**（漏 clear = 事故，见下）；
3. `MODE_INHERITABLETHREADLOCAL` 全局策略——**线程池场景禁用**：InheritableThreadLocal 只在**线程创建时**继承一次，池里线程被复用时携带的还是"出生那个请求"的上下文——用户 A 的身份在用户 B 的请求线程里苏醒，越权事故的教科书样本（机制见 1_java.md 3.4）。

**泄漏与脏数据两面**：
- **脏**：线程池跨请求复用，残留 context 被下一个请求读到——Web 请求线程有 SecurityContextHolderFilter 的收尾 clear 兜底（1.1.2 表第 1 行），但**手工 set 的异步线程没人兜底**，必须 finally clear；
- **漏**：context 里挂着的应用类对象若驻在容器长命线程上，热部署/卸载时钉住旧类加载器（5_tomcat.md 2.2.2 的三大泄漏路径之一，Boot 独立进程下影响弱）。

**延伸追问**：虚拟线程与 ThreadLocal 的规模代价（JEP 425 后仍要控制量级）、Reactive/WebFlux 下 context 根本不放 ThreadLocal 而走 Reactor Context（SecurityReactorContextConfiguration）——答到"响应式体系是另一条传递通道"即可止盈。