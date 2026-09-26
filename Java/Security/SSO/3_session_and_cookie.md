
# 1. 概述

Cookie 和 Session 是 SSO 的基石。理解它们的底层机制，才能深入理解 SSO 中的跨域问题、会话安全、分布式 Session 等进阶主题。本章从 HTTP 无状态特性出发，系统讲解 Cookie/Session 的原理、安全属性以及同域 SSO 的实现。

# 2. HTTP 无状态与 Session 机制

## 2.1 HTTP 的无状态特性

HTTP 协议本身是无状态的：每一个请求都是独立的，服务端不会自动关联两个请求是否来自同一用户。

```
请求 1: GET /index    → 服务端：不知道你是谁
请求 2: POST /login   → 服务端：验证用户名密码，知道了
请求 3: GET /profile  → 服务端：又不知道你是谁了！
```

为了在无状态协议上实现"会话"概念，产生了 Session + Cookie 机制。

## 2.2 Session 的生命周期

```
┌──────────────────────────────────────────────────────────────┐
│                       Session 生命周期                        │
│                                                               │
│  ① 创建：用户第一次访问时（或登录成功后）                      │
│      request.getSession(true)                                │
│      → 服务端生成唯一 SessionID                               │
│      → 创建 HttpSession 对象并存入内存/Redis                  │
│      → Set-Cookie: JSESSIONID=xxx 返回浏览器                  │
│                                                               │
│  ② 使用：每次请求，浏览器自动带上 Cookie                      │
│      Cookie: JSESSIONID=xxx                                  │
│      → 服务端根据 JSESSIONID 查找对应 HttpSession             │
│      → 从 Session 中获取用户信息                              │
│                                                               │
│  ③ 销毁：主动登出 / 超时 / Cookie 清除 / 服务端重启           │
│      session.invalidate()                                    │
│      → 服务端删除 Session                                     │
│      → 浏览器 Cookie 失效（或手动清除）                       │
│                                                               │
│  Session 默认超时时间：30 分钟（Servlet 规范）                 │
│  可在 web.xml 中配置：                                        │
│  <session-config>                                             │
│      <session-timeout>30</session-timeout>  <!-- 分钟 -->    │
│  </session-config>                                            │
└──────────────────────────────────────────────────────────────┘
```

## 2.3 Session 的底层实现

Tomcat 中的 Session 管理：

```java
// 简化版 Tomcat Session 管理核心逻辑

public class StandardManager implements Manager {
    // 所有 Session 存储在此 Map 中
    protected Map<String, Session> sessions = new ConcurrentHashMap<>();

    @Override
    public Session createSession(String sessionId) {
        // 1. 如果 sessionId 为空，生成新的（随机数）
        Session session = new StandardSession(this);
        session.setId(sessionId != null ? sessionId : generateSessionId());
        session.setCreationTime(System.currentTimeMillis());
        session.setMaxInactiveInterval(30 * 60); // 30分钟

        // 2. 存入 sessions Map
        sessions.put(session.getId(), session);

        // 3. 启动后台线程定期清除过期 Session
        return session;
    }

    @Override
    public Session findSession(String id) {
        if (id == null) return null;
        Session session = sessions.get(id);
        if (session != null && !session.isValid()) {
            sessions.remove(id);
            return null;
        }
        return session;
    }

    // SessionID 生成：随机数 + JVM 路由（用于集群）
    protected String generateSessionId() {
        // 格式：32位随机数.JVM实例标识
        // 例如：A1B2C3D4E5F6A7B8C9D0E1F2A3B4C5D6.node1
        return UUID.randomUUID().toString().replace("-", "")
               + "." + getJvmRoute();
    }

    // 后台线程：每 60 秒检查一次过期 Session
    @Override
    public void backgroundProcess() {
        processExpires();
    }

    private void processExpires() {
        long timeNow = System.currentTimeMillis();
        Session[] expiredSessions = sessions.values().stream()
            .filter(s -> !s.isValid())
            .toArray(Session[]::new);
        for (Session session : expiredSessions) {
            sessions.remove(session.getId());
        }
    }
}
```

## 2.4 SessionID 生成机制

| 容器 | SessionID 命名 | 格式 | 生成算法 |
|------|---------------|------|---------|
| **Tomcat** | JSESSIONID | 32位hex.JvmRoute | SecureRandom |
| **Jetty** | JSESSIONID | node0xxxxx | 随机字符串 |
| **WebLogic** | JSESSIONID | 随机 | 随机 + 集群标识 |
| **Undertow** | JSESSIONID | 随机 | SecureRandom |

**安全要求：**
- SessionID 必须具有足够的随机性（至少 128 位熵）
- 每次认证成功后必须更换 SessionID（防会话固定攻击）
- 不应包含可预测的信息（如用户名、序列号）

```java
// Spring Security 的会话固定防护
// SecurityContextHolder 在认证成功后会执行：
protected void onAuthenticationSuccess(HttpServletRequest request,
        HttpServletResponse response, Authentication authentication) {
    // 强制更换 SessionID
    HttpSession session = request.getSession(false);
    if (session != null) {
        session.invalidate();  // 使旧 Session 失效
    }
    // 创建新 Session（新 JSESSIONID）
    session = request.getSession(true);
    super.onAuthenticationSuccess(request, response, authentication);
}
```

# 3. Cookie 机制深度解析

## 3.1 Cookie 的属性详解

```
Set-Cookie: JSESSIONID=A1B2C3D4E5F6A7B8;        ← 键值对
             Path=/app;                            ← 路径范围
             Domain=.example.com;                   ← 域名范围
             Max-Age=3600;                          ← 过期时间（秒）
             Expires=Wed, 21 Oct 2025 07:28:00 GMT; ← 绝对过期时间
             HttpOnly;                              ← 禁止 JavaScript 访问
             Secure;                                ← 仅 HTTPS 传输
             SameSite=Lax                           ← 跨站请求控制
```

| 属性 | 作用 | 默认值 | SSO 中的使用 |
|------|------|--------|-------------|
| **Domain** | 控制 Cookie 的作用域名范围 | 当前域名 | 设 `.example.com` 跨子域共享 |
| **Path** | 控制 Cookie 作用的路径范围 | 当前路径 | `/` 对整个应用可见 |
| **Max-Age** | Cookie 有效期（秒），浏览器自动删除 | -1（会话结束） | SSO TGC 通常设置较长 |
| **Expires** | 绝对过期时间（GMT） | 无 | 与 Max-Age 类似 |
| **HttpOnly** | 禁止 `document.cookie` 读取 | false | **必须设为 true**，防 XSS |
| **Secure** | 仅通过 HTTPS 传输 | false | **生产必须设为 true** |
| **SameSite** | 跨站请求时是否携带 | Lax（Chrome） | SSO 关键属性（见下文） |

## 3.2 SameSite 属性与 SSO

SameSite 是 SSO 实现中最关键也最容易出问题的 Cookie 属性：

```
┌──────────────────────────────────────────────────────────────┐
│               SameSite 属性详解                               │
│                                                               │
│  SameSite=Strict                                              │
│  ├── 最严格：只有同站请求才携带 Cookie                         │
│  ├── 示例：用户在 a.com 点击链接到 b.com，不携带 b.com Cookie │
│  └── SSO 影响：从系统 A 跳转到 CAS → Cookie 不会被携带        │
│       → CAS 无法识别已登录用户 → SSO 失效！                   │
│                                                               │
│  SameSite=Lax（Chrome 默认值）                                │
│  ├── 中等严格：同站请求 + 顶级导航 GET 请求携带 Cookie         │
│  ├── 示例：用户点击链接从 a.com→b.com，携带 Cookie ✓          │
│  │         a.com 页面的 <img> 加载 b.com 图片，不携带 ✗       │
│  └── SSO 影响：302 重定向被归类为顶级导航                     │
│       → GET 302 重定向会携带 Cookie → SSO 正常工作 ✓          │
│                                                               │
│  SameSite=None                                                │
│  ├── 最宽松：所有请求都携带 Cookie                             │
│  ├── 必须同时设置 Secure=true                                 │
│  └── SSO 影响：完全兼容，但安全风险最高                       │
│                                                               │
│  SSO 最佳实践：SameSite=Lax + HttpOnly + Secure               │
│  - 与 CAS 的 302 重定向兼容                                   │
│  - 防御 CSRF（POST/PUT 等非 GET 请求不会携带 Cookie）          │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

```java
// Spring Boot 配置 SameSite Cookie
@Configuration
public class CookieConfiguration {

    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();
        serializer.setSameSite("Lax");          // SSO 推荐 Lax
        serializer.setUseHttpOnlyCookie(true);   // 防 XSS
        serializer.setUseSecureCookie(true);     // HTTPS
        serializer.setCookieName("SESSION");
        serializer.setCookiePath("/");
        return serializer;
    }
}

// 也可以通过 Tomcat 配置
// application.yml
server:
  servlet:
    session:
      cookie:
        http-only: true
        secure: true
        same-site: lax
        name: SESSION
```

## 3.3 Cookie 的大小和数量限制

| 规范 | 单条 Cookie 大小 | 每个域名 Cookie 数量 | 浏览器总数 |
|------|-----------------|---------------------|-----------|
| **RFC 6265** | 至少 4096 字节 | 至少 50 条 | ≥ 3000 条 |
| **Chrome** | ~4096 字节 | ~180 条 | — |
| **Firefox** | ~4096 字节 | ~150 条 | — |
| **Safari** | ~4096 字节 | ~600 条 | — |

**SSO 相关注意事项：**
- SSO TGC Cookie 应该尽可能小（只存 TicketID，不存用户信息）
- 避免在 Cookie 中存储大量用户属性（JWT 更适合放 Header）
- Cookie 太大可能导致请求头过大，Nginx 等反向代理可能拒绝

# 4. 同域 SSO 实现

## 4.1 原理

同一父域名下的不同子域名应用共享 Cookie：

```
app1.example.com  ────┐
app2.example.com  ────┼── 共享 Cookie（Domain=.example.com）
app3.example.com  ────┘
```

## 4.2 基于共享 Cookie 的同域 SSO

```java
// ========== 方案一：共享 Cookie（最简单） ==========

// 1. 登录中心（login.example.com）
@PostMapping("/login")
public String login(String username, String password, HttpServletResponse response) {
    User user = userService.authenticate(username, password);
    String token = jwtService.createToken(user);
    
    // 写入共享域的 Cookie
    Cookie cookie = new Cookie("SSO_TOKEN", token);
    cookie.setDomain(".example.com");  // 所有子域共享
    cookie.setPath("/");
    cookie.setHttpOnly(true);
    cookie.setSecure(true);
    cookie.setMaxAge(3600);
    response.addCookie(cookie);
    
    return "redirect:/home";
}

// 2. 各子系统（app1.example.com / app2.example.com）
// 统一的认证 Filter
@Component
public class SsoCookieFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) {
        
        Cookie[] cookies = request.getCookies();
        if (cookies != null) {
            for (Cookie cookie : cookies) {
                if ("SSO_TOKEN".equals(cookie.getName())) {
                    // 验证 Token
                    String token = cookie.getValue();
                    Authentication auth = jwtService.verifyAndGetAuth(token);
                    SecurityContextHolder.getContext().setAuthentication(auth);
                    break;
                }
            }
        }
        
        if (SecurityContextHolder.getContext().getAuthentication() == null) {
            // 未认证，重定向到登录中心
            response.sendRedirect("https://login.example.com/login?redirect="
                + URLEncoder.encode(request.getRequestURL().toString(), "UTF-8"));
            return;
        }
        
        chain.doFilter(request, response);
    }
}

// ========== 方案二：共享 Session（Redis） ==========

// Spring Session + Redis 实现同域 SSO
// 所有应用共享同一个 Redis 作为 Session 存储

// 依赖：spring-session-data-redis

// application.yml（所有应用一样配置）
spring:
  session:
    store-type: redis
    redis:
      namespace: spring:session
  redis:
    host: redis.example.com
    port: 6379
    password: xxx

// 所有应用的 Cookie 配置
@Bean
public CookieSerializer cookieSerializer() {
    DefaultCookieSerializer serializer = new DefaultCookieSerializer();
    serializer.setDomainNamePattern("^.+?\\.(\\w+\\.[a-z]+)$"); // 提取父域名
    serializer.setCookieName("SSO_SESSION");
    return serializer;
}
```

## 4.3 同域 SSO 流程示意

```
用户登录 login.example.com
         │
         │ Set-Cookie: SSO_TOKEN=xxx; Domain=.example.com
         ▼
用户访问 app1.example.com
         │
         │ Cookie: SSO_TOKEN=xxx（自动携带，因为是 .example.com 子域）
         ▼
app1 Filter 验证 Token ✓ → 已登录，无需跳转
         │
用户访问 app2.example.com
         │
         │ Cookie: SSO_TOKEN=xxx（自动携带）
         ▼
app2 Filter 验证 Token ✓ → 已登录，无需跳转
```

**同域 SSO 的局限性：**
1. 只能同一父域名（如 `.example.com`）
2. 不同域名的系统（如 `a.com` 和 `b.com`）无法共享 Cookie
3. 安全问题：任一子域被 XSS 攻击可能泄露 Cookie
4. 需要所有系统在同一可信域下

# 5. Session 安全实践

## 5.1 会话固定攻击防护

```
攻击流程：
1. 攻击者访问 login.example.com，获得 JSESSIONID=ATTACKER123
2. 攻击者构造链接：https://login.example.com/login?JSESSIONID=ATTACKER123
3. 诱导受害者点击链接
4. 受害者登录成功 → Session 内容变为受害者的，但 SessionID 不变
5. 攻击者用 JSESSIONID=ATTACKER123 访问 → 获得受害者身份！
```

**防御措施：**

```java
// Spring Security 自动防御（默认启用）
// SessionManagementConfigurer 在认证成功后会强制更换 SessionID

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session -> session
                .sessionFixation()              // 会话固定防护
                .migrateSession()               // 认证后更换 SessionID
                // .newSession()               // 创建全新 Session（更安全）
                // .none()                      // 关闭（不推荐）
            );
        return http.build();
    }
}
```

## 5.2 Session 并发控制

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session -> session
                .maximumSessions(1)              // 同一账号最多1个会话
                .maxSessionsPreventsLogin(false) // false: 新登录踢旧
                                                  // true:  禁止新登录
                .expiredUrl("/login?expired")    // 被踢后跳转地址
                .sessionRegistry(sessionRegistry())
            );
        return http.build();
    }

    @Bean
    public SessionRegistry sessionRegistry() {
        return new SessionRegistryImpl();
    }
}

// SessionRegistry 存储所有活跃会话
// 用户在新设备登录 → 检测到超出限制 → 旧 Session 标记过期
// 旧 Session 下次请求 → 检测到已过期 → 重定向到登录页
```

## 5.3 Session 超时与空闲检测

```java
// application.yml
server:
  servlet:
    session:
      timeout: 30m          # 最大会话时长（不活跃也失效）
      # 注意：这是绝对时间，不是空闲时间

// 空闲超时（Idle Timeout）需要自己实现：
@Component
public class IdleSessionFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) {
        
        HttpSession session = request.getSession(false);
        if (session != null) {
            Long lastAccessTime = (Long) session.getAttribute("LAST_ACCESS_TIME");
            long now = System.currentTimeMillis();
            
            if (lastAccessTime != null) {
                long idleTime = now - lastAccessTime;
                long maxIdleTime = 15 * 60 * 1000; // 15分钟空闲超时
                
                if (idleTime > maxIdleTime) {
                    session.invalidate();
                    response.sendRedirect("/login?timeout");
                    return;
                }
            }
            session.setAttribute("LAST_ACCESS_TIME", now);
        }
        chain.doFilter(request, response);
    }
}
```

# 6. Session 共享方案对比

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| **Session 复制** | 各节点互相广播 Session | 无外部依赖 | 网络开销大，节点越多越糟 | 小集群（2-3节点） |
| **粘性路由** | Nginx ip_hash 固定用户→节点 | 简单，无存储成本 | 节点宕机 Session 丢失 | 可容忍 Session 丢失 |
| **集中存储(Redis)** | 所有 Session 存 Redis | 无状态节点，水平扩展 | Redis 单点（需集群） | **主流方案** |
| **JWT 无状态** | Session 信息编码在 Token | 零服务端存储 | Token 变大，吊销困难 | 微服务 |
| **数据库存储** | Session 存 MySQL | 利用现有 DB | 性能差，不推荐 | 一般不推荐 |

## 6.1 Spring Session + Redis 配置

```java
// build.gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'org.springframework.session:spring-session-data-redis'
}

// application.yml
spring:
  session:
    store-type: redis
    timeout: 30m  # Session 过期时间（秒/分/时/天）
    redis:
      namespace: myapp:session    # Redis key 前缀
      flush-mode: on_save         # on_save: 保存时写入，immediate: 立即写入
      save-mode: on_set_attribute # on_set_attribute: 属性变更时保存

  redis:
    host: localhost
    port: 6379
    # 集群模式
    # cluster:
    #   nodes:
    #     - redis1:6379
    #     - redis2:6379
    #     - redis3:6379
```

# 7. 总结

| 概念 | 关键点 |
|------|--------|
| **Cookie Domain** | `.example.com` 实现子域共享，这是同域 SSO 的基础 |
| **SameSite** | SSO 必须设为 `Lax` 才能支持 302 重定向流程 |
| **HttpOnly + Secure** | 生产环境必须设，防 XSS 和中间人攻击 |
| **会话固定** | Spring Security 默认防御，认证后更换 SessionID |
| **分布式 Session** | Redis 集中存储是主流方案 |
| **Cookie 大小** | 控制在 4KB 以下，不存储大量用户数据 |
