
# 1. 概述

**CAS（Central Authentication Service）** 是耶鲁大学开发的开源 SSO 协议，广泛用于 Java Web 应用的企业内部认证。CAS 以票据（Ticket）为核心，提供完整的认证、单点登出（SLO）和代理（Proxy）能力。

```
┌──────────────────────────────────────────────────────────────────┐
│                        CAS 核心架构                               │
│                                                                   │
│  浏览器                         CAS Server                        │
│  ┌─────┐                    ┌──────────────────┐                 │
│  │TGC  │ ← Cookie ────────►│  TGT 存储         │                 │
│  │     │                    │  (内存/Redis)     │                 │
│  └─────┘                    │                  │                 │
│     │                       │  用户身份库       │                 │
│     │                       │  (LDAP/DB)       │                 │
│     │                       └────────┬─────────┘                 │
│     │                                │                            │
│     │     ┌─────────────────┐        │                            │
│     ├────►│ CAS Client (A)  │───────►│  /serviceValidate          │
│     │     └─────────────────┘        │  (验证 ST)                 │
│     │     ┌─────────────────┐        │                            │
│     ├────►│ CAS Client (B)  │───────►│  /p3/serviceValidate       │
│     │     └─────────────────┘        │  (验证 ST + 返回属性)      │
│     │                                │                            │
│     │                                │  /logout (SLO)             │
│     │                                │  /proxy (代理认证)         │
│     │                                └────────────────────────────┘
└──────────────────────────────────────────────────────────────────┘
```

# 2. CAS 核心票据体系

## 2.1 票据类型与关系

```
TGC (Ticket Granting Cookie)
  │  浏览器 Cookie，表示用户已全局认证
  │  存储在客户端，每次访问 CAS Server 时携带
  ▼
TGT (Ticket Granting Ticket)
  │  服务端票据，TGC 对应的服务端实体
  │  存储在 CAS Server（内存/ticket registry）
  │  用来签发 ST 和 PGT
  ▼
ST (Service Ticket)
  │  一次性票据，用于访问特定 Service
  │  通过 URL 参数传递
  │  验证后立即销毁
  ▼
PGT (Proxy Granting Ticket)
  │  代理票据，用于后端服务代表用户请求其他服务
  │  通过 PGTIOU 回调传递
  ▼
PT (Proxy Ticket)
      代理模式下访问其他 Service 的票据
```

## 2.2 票据属性对比

| 票据 | 用途 | 生命周期 | 验证次数 | 传递方式 | 安全要求 |
|------|------|---------|---------|---------|---------|
| **TGC** | 全局登录凭证 | 数小时 | 不限 | Cookie | Secure+HttpOnly |
| **TGT** | 签发票据的凭证 | 与 TGC 同步 | 不限 | 服务端 | 不可暴露 |
| **ST** | 访问单个 Service | 数秒~数分钟 | 1次 | URL 参数 | 一次性 |
| **PGT** | 代理认证凭证 | 与 TGT 同步 | 不限 | PGTIOU 回调 | 不可暴露 |
| **PT** | 代理访问 Service | 数秒~数分钟 | 1次 | HTTP Header | 一次性 |

## 2.3 CAS Server 票据存储

```java
// CAS Server 核心：TicketRegistry 接口

public interface TicketRegistry {
    void addTicket(Ticket ticket);
    Ticket getTicket(String ticketId);
    Ticket getTicket(String ticketId, Class<? extends Ticket> clazz);
    boolean deleteTicket(String ticketId);
    long deleteAll();
    Collection<Ticket> getTickets();
    long sessionCount();
}

// 默认实现：DefaultTicketRegistry（基于 HashMap，仅单机）
public class DefaultTicketRegistry extends AbstractTicketRegistry {
    private final Map<String, Ticket> cache = new ConcurrentHashMap<>();
    // ...
}

// 生产推荐：RedisTicketRegistry / JpaTicketRegistry
// 支持集群环境下 TGT 共享
```

# 3. CAS 认证协议版本

## 3.1 协议版本对比

| 版本 | 协议端点 | 数据格式 | 功能 | 推荐度 |
|------|---------|---------|------|--------|
| **CAS 1.0** | `/validate` | 纯文本（yes/no） | 仅验证 ST | ❌ 过时 |
| **CAS 2.0** | `/serviceValidate` | XML | 验证 ST + 返回用户 principal | ⚠️ 可用 |
| **CAS 3.0** | `/p3/serviceValidate` | XML/JSON | + 返回用户属性（attributes） | ✅ 推荐 |
| **SAML 1.1** | `/samlValidate` | SAML XML | SAML 风格响应 | 特殊场景 |

## 3.2 CAS 1.0 协议（已过时）

```
验证请求：
GET /validate?service=app-url&ticket=ST-xxx

验证响应（纯文本）：
yes   ← 验证通过
no    ← 验证失败
```

**缺陷**：只能知道 ST 是否有效，无法获取用户身份信息。

## 3.3 CAS 2.0 协议

```xml
<!-- 验证请求 -->
GET /serviceValidate?service=https://app.example.com/callback&ticket=ST-xxx

<!-- 验证成功响应 -->
<cas:serviceResponse xmlns:cas="http://www.yale.edu/tp/cas">
    <cas:authenticationSuccess>
        <cas:user>zhangsan</cas:user>
    </cas:authenticationSuccess>
</cas:serviceResponse>

<!-- 验证失败响应 -->
<cas:serviceResponse xmlns:cas="http://www.yale.edu/tp/cas">
    <cas:authenticationFailure code="INVALID_TICKET">
        Ticket 'ST-xxx' not recognized
    </cas:authenticationFailure>
</cas:serviceResponse>
```

## 3.4 CAS 3.0 协议（推荐）

```xml
<!-- 验证请求（多了 format=JSON 参数可返回 JSON） -->
GET /p3/serviceValidate?service=app-url&ticket=ST-xxx&format=JSON

<!-- JSON 响应（更轻量） -->
{
    "serviceResponse": {
        "authenticationSuccess": {
            "user": "zhangsan",
            "attributes": {
                "email": "zhangsan@example.com",
                "displayName": "张三",
                "department": "技术部",
                "roles": ["ROLE_ADMIN", "ROLE_USER"],
                "phone": "13800138000"
            }
        }
    }
}
```

# 4. CAS 认证流程深度解析

## 4.1 首次访问（未登录）

```
浏览器                     App Server              CAS Server
  │                            │                       │
  │ 1. GET /protected          │                       │
  │───────────────────────────►│                       │
  │                            │ 2. 检测无 session     │
  │                            │    无 ticket 参数     │
  │                            │                       │
  │ 3. 302 重定向              │                       │
  │◄───────────────────────────│                       │
  │ Location: cas/login?service=https://app/callback  │
  │                            │                       │
  │ 4. GET /login?service=https://app/callback        │
  │──────────────────────────────────────────────────►│
  │                            │                       │ 5. 检测无 TGC
  │                            │                       │    展示登录页
  │                            │                       │
  │ 6. 登录页（HTML Form）                             │
  │◄──────────────────────────────────────────────────│
  │                            │                       │
  │ 7. POST /login（用户名+密码+service）               │
  │──────────────────────────────────────────────────►│
  │                            │                       │ 8. 验证凭据
  │                            │                       │ 9. 生成 TGT
  │                            │                       │10. 生成 TGC Cookie
  │                            │                       │11. 生成 ST
  │                            │                       │
  │12. 302 重定向                                     │
  │◄──────────────────────────────────────────────────│
  │ Set-Cookie: TGC=xxx; HttpOnly; Secure             │
  │ Location: https://app/callback?ticket=ST-1-xxx    │
  │                            │                       │
  │13. GET /callback?ticket=ST-1-xxx                  │
  │───────────────────────────►│                       │
  │                            │14. 后台验证 ST        │
  │                            │──────────────────────►│
  │                            │ POST /p3/serviceValidate
  │                            │ service=app&ticket=ST │
  │                            │                       │
  │                            │15. 返回用户信息        │
  │                            │◄──────────────────────│
  │                            │ {user, attributes}    │
  │                            │                       │
  │                            │16. 创建 App 局部会话  │
  │                            │17. 302 去掉 ticket   │
  │◄───────────────────────────│                       │
  │ Location: /protected                              │
  │                            │                       │
  │18. GET /protected（带 JSESSIONID）                │
  │───────────────────────────►│                       │
  │◄────────────── 页面内容 ───│                       │
```

## 4.2 访问第二个应用（已登录）

```
浏览器                     App B                  CAS Server
  │                            │                       │
  │ 1. GET /app-b              │                       │
  │───────────────────────────►│                       │
  │                            │ 2. 无 session         │
  │ 3. 302 重定向              │                       │
  │◄───────────────────────────│                       │
  │ Location: cas/login?service=https://app-b/callback│
  │                            │                       │
  │ 4. GET /login?service=callback                    │
  │──────────────────────────────────────────────────►│
  │ Cookie: TGC=xxx           │                       │
  │                            │                       │ 5. 检测 TGC ✓
  │                            │                       │    验证 TGT 有效 ✓
  │                            │                       │    直接签发 ST
  │                            │                       │    （无需用户再输入密码！）
  │                            │                       │
  │ 6. 302 重定向                                     │
  │◄──────────────────────────────────────────────────│
  │ Location: https://app-b/callback?ticket=ST-2-yyy  │
  │                            │                       │
  │ 7. 同首次访问 13-18 步......                     │
  │   （App B 用 ST 向 CAS 验证 → 创建局部 session）  │
```

**关键点**：步骤 5 CAS 检测到 TGC 有效（用户已全局登录），直接签发 ST，不再要求输入密码。

## 4.3 CAS Client 核心 Filter 链

```java
// CAS Client 的核心 Filter 链（web.xml 或 Java Config）

// 1. AuthenticationFilter：拦截未登录请求，跳转到 CAS 登录
// 2. TicketValidationFilter：拦截带 ticket 参数的请求，验证 ST
// 3. HttpServletRequestWrapperFilter：包装 request，暴露用户信息
// 4. AssertionThreadLocalFilter：将 Assertion 存入 ThreadLocal
// 5. SingleSignOutFilter：接收 CAS Server 的 SLO 通知

// ========== AuthenticationFilter 核心逻辑 ==========
public class AuthenticationFilter extends AbstractCasFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) {

        // 1. 检查是否已登录（session 中是否有 Assertion）
        if (isAuthenticated(request)) {
            chain.doFilter(request, response);
            return;
        }

        // 2. 检查请求中是否有 ticket 参数
        String ticket = request.getParameter("ticket");
        if (ticket != null) {
            // 有 ticket，交给 TicketValidationFilter 验证
            chain.doFilter(request, response);
            return;
        }

        // 3. 未登录且无 ticket → 重定向到 CAS 登录页
        String loginUrl = getCasServerLoginUrl()
            + "?service=" + URLEncoder.encode(getServiceUrl(request), "UTF-8");
        response.sendRedirect(loginUrl);
    }
}

// ========== TicketValidationFilter 核心逻辑 ==========
public class TicketValidationFilter extends AbstractTicketValidationFilter {

    @Override
    protected Assertion validateTicket(HttpServletRequest request,
            HttpServletResponse response, String ticket) {

        // 调用 CAS Server /p3/serviceValidate 验证 ST
        String validationUrl = casServerUrlPrefix + "/p3/serviceValidate"
            + "?service=" + URLEncoder.encode(serviceUrl, "UTF-8")
            + "&ticket=" + ticket
            + "&format=JSON";

        // 使用 HTTP Client 发送验证请求
        String responseBody = httpClient.get(validationUrl);

        // 解析 CAS 响应
        Cas30ServiceTicketValidator validator = new Cas30ServiceTicketValidator(casServerUrlPrefix);
        Assertion assertion = validator.validate(ticket, serviceUrl);

        return assertion;
    }
}
```

# 5. 单点登出（Single Logout / SLO）

## 5.1 SLO 流程

```
用户点击登出
     │
     ▼
注销 App A 的局部 Session
     │
     │ 302 重定向
     ▼
CAS Server /logout
     │
     ├── ① 销毁 TGT（全局会话）
     ├── ② 删除 TGC Cookie
     ├── ③ 遍历 TGT 下所有已签发的 ST
     │      找到所有已登录的 App
     │
     ├── ④ 异步向各 App 发送登出请求
     │      POST https://app-a/logout
     │      POST https://app-b/logout
     │
     └── ⑤ 展示登出成功页
          （或重定向到指定的 afterLogoutUrl）
```

## 5.2 CAS Server SLO 核心实现

```java
// CAS Server 登出处理核心
@GetMapping("/logout")
public String logout(HttpServletRequest request,
        HttpServletResponse response,
        @RequestParam(required = false) String service) {

    // 1. 获取 TGT
    String tgtId = getTicketGrantingTicketId(request);
    TicketGrantingTicket tgt = ticketRegistry.getTicket(tgtId);

    if (tgt != null) {
        // 2. 获取该 TGT 下的所有 ST（即所有已登录的 App）
        Collection<ServiceTicket> sts = tgt.getServices().values();

        // 3. 异步通知各 App 登出
        for (ServiceTicket st : sts) {
            String logoutUrl = st.getService().getLogoutUrl();
            if (logoutUrl != null) {
                // 构造 SLO 请求（XML 或 JSON）
                String sloMessage = buildLogoutRequest(tgt, st);
                // POST 到 App 的 logout 端点
                logoutManager.performLogout(logoutUrl, sloMessage);
            }
        }

        // 4. 销毁 TGT
        ticketRegistry.deleteTicket(tgtId);
    }

    // 5. 清除 TGC Cookie
    Cookie cookie = new Cookie("TGC", "");
    cookie.setMaxAge(0);
    cookie.setPath("/cas");
    response.addCookie(cookie);

    return "logout";
}
```

## 5.3 CAS Client SLO 接收

```java
// CAS Client 端的 SingleSignOutFilter
public class SingleSignOutFilter implements Filter {

    // CAS Client 提供一个端点接收 CAS Server 的登出通知
    // Web.xml 配置：
    // <filter>
    //     <filter-name>CAS Single Sign Out Filter</filter-name>
    //     <filter-class>org.jasig.cas.client.session.SingleSignOutFilter</filter-class>
    // </filter>

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) {

        HttpServletRequest httpRequest = (HttpServletRequest) request;

        // 检查是否是 CAS Server 发来的登出请求
        if ("POST".equals(httpRequest.getMethod())
                && isLogoutRequest(httpRequest)) {

            // 解析登出请求中的 SessionID
            String sessionId = getSessionIdFromLogoutRequest(httpRequest);

            // 使对应的 Session 失效
            HttpSession session = sessionMappingStorage.removeSessionByMappingId(sessionId);
            if (session != null) {
                session.invalidate();
            }

            // 返回成功响应
            response.getWriter().write("OK");
            return;
        }

        chain.doFilter(request, response);
    }
}
```

# 6. CAS Proxy（代理认证）

## 6.1 代理模式场景

```
用户 ──→ App A（前端应用，有用户 Session）
              │
              │ 需要以"用户身份"访问 App B（后端 API）
              ▼
         App B（后端 API，需要认证）

问题：App B 没有用户的 Session，怎么认证？
方案：CAS Proxy —— App A 代表用户获取 PT（Proxy Ticket）去访问 App B
```

## 6.2 Proxy 认证流程

```
用户        App A (SP)          CAS Server          App B (Target)
 │             │                    │                    │
 │ 1. 已登录   │                    │                    │
 │────────────►│                    │                    │
 │             │ 2. GET /proxy     │                    │
 │             │───────────────────►│                    │
 │             │ service=app-a     │                    │
 │             │ pgtUrl=回调        │                    │
 │             │                    │ 3. 签发 PGT+PGTIOU │
 │             │ 4. 302 + ST       │                    │
 │             │◄───────────────────│                    │
 │             │                    │                    │
 │             │ 5. 后台回调 PGTIOU │                    │
 │             │   (通过 pgtUrl)    │                    │
 │             │◄───────────────────│                    │
 │             │   获取 PGT         │                    │
 │             │                    │                    │
 │             │ 6. 请求 Proxy Ticket (PT)              │
 │             │───────────────────►│                    │
 │             │ targetService=app-b│                   │
 │             │ pgt=PGT-xxx        │                    │
 │             │                    │ 7. 签发 PT         │
 │             │ 8. 返回 PT         │                    │
 │             │◄───────────────────│                    │
 │             │                    │                    │
 │             │ 9. 用 PT 访问 App B                    │
 │             │───────────────────────────────────────►│
 │             │                    │                    │ 10. 验证 PT
 │             │                    │◄───────────────────│
 │             │                    │ 11. 返回用户信息    │
 │             │                    │───────────────────►│
 │             │                    │                    │ 12. 处理请求
 │             │ 13. 返回结果       │                    │
 │             │◄───────────────────────────────────────│
```

## 6.3 Proxy 票据说明

| 票据 | 作用 | 生命周期 |
|------|------|---------|
| **PGT**（Proxy Granting Ticket） | 后端应用持有，用于签发 PT | 与 TGT 同步 |
| **PGTIOU**（PGT I Owe You） | PGT 的引用凭证，通过回调传递 | 一次性 |
| **PT**（Proxy Ticket） | 后端应用代表用户访问目标服务 | 一次性（同 ST） |

# 7. CAS Server 核心配置

## 7.1 Ticket 过期策略

```properties
# cas.properties

# TGT 超时时间（秒），默认 7200（2小时）
tgt.timeToKillInSeconds=7200

# ST 超时时间（秒），默认 10 秒
st.timeToKillInSeconds=10

# TGT 最大未使用时间（秒），超过此时间未签发新 ST 则过期
tgt.maxTimeToLiveInSeconds=28800

# Ticket 清理周期（秒）
ticket.registry.cleaner.schedule.repeatInterval=60
```

## 7.2 Service 注册（安全白名单）

```json
// services/MyApp-100.json（CAS 6.x JSON Service Registry）
{
  "@class": "org.apereo.cas.services.RegexRegisteredService",
  "serviceId": "^https://app\\.example\\.com/.*",
  "name": "My Application",
  "id": 100,
  "evaluationOrder": 1,
  "attributeReleasePolicy": {
    "@class": "org.apereo.cas.services.ReturnAllAttributeReleasePolicy"
  },
  "accessStrategy": {
    "@class": "org.apereo.cas.services.DefaultRegisteredServiceAccessStrategy",
    "enabled": true,
    "ssoEnabled": true
  },
  "logoutType": "BACK_CHANNEL",
  "logoutUrl": "https://app.example.com/logout"
}
```

# 8. CAS 安全注意事项

| 安全项 | 措施 |
|--------|------|
| **ST 重放攻击** | ST 验证后立即删除，仅使用一次 |
| **ST 劫持** | 全程 HTTPS，ST 生命周期极短（默认10秒） |
| **TGC 窃取** | HttpOnly + Secure + SameSite=Lax |
| **TGT 固定攻击** | 不支持 TGT 通过 URL 传递 |
| **Service 伪造** | 注册白名单（Service Registry），仅允许已注册的 Service |
| **Proxy 滥用** | 仅对受信的应用开放 Proxy 能力 |
| **登出不完备** | 开启 SLO，前后端通道双重通知 |

# 9. CAS 协议优缺点总结

| 优点 | 缺点 |
|------|------|
| 协议简单清晰，易于理解和实现 | 强依赖 Cookie 和浏览器重定向 |
| SSO 和 SLO 支持完整 | 移动端支持较弱 |
| Java 生态成熟（Spring Security 原生支持） | 跨组织认证（联邦）能力不如 SAML |
| 票据生命周期短，安全性较高 | 协议较老，数据格式为 XML |
| CAS Server 功能丰富（MFA、密码管理等） | 生态不如 OAuth2/OIDC 活跃 |
| 企业内部部署成本低 | 不支持 OAuth2 的授权（非认证）场景 |
