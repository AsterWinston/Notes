
# 1. 概述

当企业的多个系统分布在不同的域名下时（如 `app1.example.com`、`app2.other.com`、`external.partner.com`），Cookie 的 Domain 机制失效，需要更复杂的跨域 SSO 方案。

```
┌──────────────────────────────────────────────────────────────────┐
│                    跨域 SSO 场景                                  │
│                                                                   │
│  sso.example.com（认证中心）                                      │
│         │                                                         │
│         ├──── app1.example.com（子域，同域 SSO 可行）              │
│         ├──── app2.othercorp.com（完全不同的域）                   │
│         ├──── partner.external.com（合作伙伴域）                   │
│         └──── app.mobile（移动端无 Cookie 概念）                   │
│                                                                   │
│  问题：浏览器不允许 a.com 读取 b.com 的 Cookie                    │
│  解决：通过重定向、Token 传递、父域共享等机制                      │
└──────────────────────────────────────────────────────────────────┘
```

# 2. 方案对比

| 方案 | 原理 | 适用场景 | 复杂度 | 安全性 |
|------|------|---------|--------|--------|
| **CAS Proxy** | 302 重定向 + ST 参数传递 | Web 应用跨域 | 中 | 高 |
| **SAML/OIDC** | 标准协议通过 Assertion/Token 跨域 | 企业/跨组织 | 高 | 高 |
| **Token 回调** | 302 携带 code/token 到各应用 | 自建系统 | 低 | 中 |
| **PostMessage** | 跨域 iframe + postMessage 同步 | 前端 SPA | 低 | 中 |
| **共享父域** | 统一使用 `.parent.com` 子域名 | 同一组织的多子域 | 低 | 高 |
| **JWT Token 传递** | 重定向 URL 携带 JWT Token | 跨域 REST API | 中 | 中 |
| **代理中转** | 统一网关层处理认证 | 微服务 | 中 | 高 |

# 3. 方案一：302 重定向 + Token 传递

## 3.1 流程

```
用户  App A (a.com)    认证中心 (sso.com)      App B (b.com)
 │       │                    │                    │
 │ ① GET a.com              │                    │
 │──────►│                    │                    │
 │       │ ② 未登录→302       │                    │
 │◄──────│ sso.com/login?service=a.com/callback   │
 │       │                    │                    │
 │ ③ GET sso.com/login      │                    │
 │──────────────────────────►│                    │
 │       │                    │ ④ 验证TGC→已登录   │
 │       │                    │    签发ST           │
 │       │                    │                    │
 │ ⑤ 302 a.com/callback?ticket=ST-xxx            │
 │◄──────────────────────────│                    │
 │       │                    │                    │
 │ ⑥ GET a.com/callback?ST   │                    │
 │──────►│                    │                    │
 │       │ ⑦ 后端验证ST       │                    │
 │       │───────────────────►│                    │
 │       │◄───────────────────│                    │
 │       │ ⑧ 创建a.com局部会话│                    │
 │◄──────│                    │                    │
 │       │                    │                    │
 │ ⑨ GET b.com              │                    │
 │───────────────────────────────────────────────►│
 │       │                    │                    │ ⑩ 未登录→302
 │◄───────────────────────────────────────────────│
 │       │                    │   sso.com/login?service=b.com/callback
 │       │                    │                    │
 │ ⑪ GET sso.com/login (带TGC Cookie)            │
 │──────────────────────────►│                    │
 │       │                    │ ⑫ 已登录→签发ST    │
 │       │                    │                    │
 │ ⑬ 302 b.com/callback?ticket=ST-yyy            │
 │◄───────────────────────────────────────────────│
 │       │                    │                    │
 │ ⑭ GET b.com/callback?ST   │                    │
 │───────────────────────────────────────────────►│
 │       │                    │ ⑮ 验证ST          │
 │       │                    │◄───────────────────│
 │       │                    │───────────────────►│
 │       │                    │                    │ ⑯ 创建b.com局部会话
 │◄───────────────────────────────────────────────│

关键点：每个域有独立的 Cookie/Session，通过 302 跳转到 SSO 中心做票据交换
```

## 3.2 实现

```java
/**
 * 通用跨域 SSO 客户端过滤器
 * 适用于任何域名下的应用
 */
@Component
public class CrossDomainSsoFilter extends OncePerRequestFilter {

    @Value("${sso.server-url}")
    private String ssoServerUrl;

    @Value("${sso.client-id}")
    private String clientId;

    private static final String SSO_TOKEN_PARAM = "sso_token";
    private static final String SSO_SESSION_ATTR = "SSO_USER";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        HttpSession session = request.getSession(false);
        String ticket = request.getParameter(SSO_TOKEN_PARAM);

        // Case 1: 已登录 → 放行
        if (session != null && session.getAttribute(SSO_SESSION_ATTR) != null) {
            chain.doFilter(request, response);
            return;
        }

        // Case 2: 有 token 参数 → 验证并创建会话
        if (ticket != null) {
            try {
                SsoUser user = validateToken(ticket);
                session = request.getSession(true);
                session.setAttribute(SSO_SESSION_ATTR, user);
                session.setAttribute("USER_NAME", user.getUsername());

                // 重定向去掉 token 参数（安全 + URL 美观）
                String originalUrl = request.getRequestURL().toString();
                if (request.getQueryString() != null) {
                    originalUrl = originalUrl.replace(
                        SSO_TOKEN_PARAM + "=" + ticket, "");
                }
                response.sendRedirect(originalUrl);
                return;
            } catch (Exception e) {
                response.sendRedirect(ssoServerUrl + "/login");
                return;
            }
        }

        // Case 3: 未登录 → 跳转 SSO 登录
        String callbackUrl = request.getRequestURL().toString();
        String loginUrl = ssoServerUrl + "/login"
            + "?client_id=" + clientId
            + "&redirect_uri=" + URLEncoder.encode(callbackUrl, "UTF-8");
        response.sendRedirect(loginUrl);
    }

    private SsoUser validateToken(String ticket) {
        // 后端调用 SSO Server 验证 Token
        RestTemplate restTemplate = new RestTemplate();
        return restTemplate.postForObject(
            ssoServerUrl + "/validate",
            new TokenValidationRequest(ticket, clientId),
            SsoUser.class);
    }
}
```

# 4. 方案二：跨域 iframe + postMessage

## 4.1 原理

利用一个隐蔽的 iframe 加载 SSO 域页面，通过 `postMessage` API 跨域传递登录状态。

```
主域名页面 (app1.example.com)
    │
    ├── <iframe src="https://sso.example.com/check" style="display:none">
    │
    │    SSO iframe 加载后：
    │    1. 检查 sso.example.com 的 Cookie（是否有 TGC）
    │    2. 通过 postMessage 发送结果到父页面
    │
    │    window.parent.postMessage({
    │        type: 'SSO_STATUS',
    │        loggedIn: true,
    │        user: 'zhangsan',
    │        token: 'eyJ...'
    │    }, 'https://app1.example.com');
    │
    └── 父页面监听 message 事件
```

## 4.2 实现

```html
<!-- SSO 认证中心提供的检测页面 (sso.example.com/check.html) -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
</head>
<body>
<script>
(function() {
    // 读取 SSO 域的 Cookie
    var cookies = document.cookie;
    var ssoToken = getCookie('SSO_TOKEN');

    if (ssoToken) {
        // 用户已在 SSO 域登录
        window.parent.postMessage({
            type: 'SSO_STATUS',
            loggedIn: true,
            token: ssoToken,
            origin: 'https://sso.example.com'
        }, '*'); // 生产环境应限制 origin
    } else {
        window.parent.postMessage({
            type: 'SSO_STATUS',
            loggedIn: false
        }, '*');
    }

    function getCookie(name) {
        var match = document.cookie.match(
            new RegExp('(^| )' + name + '=([^;]+)'));
        return match ? match[2] : null;
    }
})();
</script>
</body>
</html>
```

```javascript
// 各应用的页面中监听 SSO 状态
(function() {
    var iframe = document.createElement('iframe');
    iframe.src = 'https://sso.example.com/check.html';
    iframe.style.display = 'none';
    document.body.appendChild(iframe);

    window.addEventListener('message', function(event) {
        // 生产环境验证 origin:
        // if (event.origin !== 'https://sso.example.com') return;

        if (event.data.type === 'SSO_STATUS') {
            if (event.data.loggedIn) {
                // 用户已在 SSO 登录，自动在本应用登录
                autoLogin(event.data.token);
            } else {
                // 用户未登录，显示登录按钮
                showLoginButton();
            }
        }
    });

    function autoLogin(token) {
        // POST token 到本应用的登录接口
        fetch('/sso/callback', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ token: token })
        }).then(r => r.json())
          .then(data => {
              if (data.success) {
                  window.location.reload();
              }
          });
    }
})();
```

# 5. 方案三：共享父域 Cookie（同组织跨子域）

## 5.1 原理

如果所有系统共享同一个父域名，可以在父域设置 Cookie：

```
sso.example.com 写入 Cookie:
  Set-Cookie: SSO_TOKEN=xxx; Domain=.example.com; Path=/

所有 *.example.com 的下请求自动携带这个 Cookie:
  app1.example.com  → Cookie: SSO_TOKEN=xxx ✓
  app2.example.com  → Cookie: SSO_TOKEN=xxx ✓
  api.example.com   → Cookie: SSO_TOKEN=xxx ✓

但 othercorp.com 不会收到 ✓  ← 这正是我们需要的行为
```

## 5.2 配置

```java
// 认证中心写入父域 Cookie
@PostMapping("/login")
public String login(String username, String password,
                    HttpServletResponse response) {
    User user = userService.authenticate(username, password);
    String token = tokenService.createToken(user);

    Cookie cookie = new Cookie("SSO_TOKEN", token);
    cookie.setDomain(".example.com");     // 父域名
    cookie.setPath("/");
    cookie.setHttpOnly(true);
    cookie.setSecure(true);
    cookie.setMaxAge(7200);               // 2 小时
    response.addCookie(cookie);

    return "redirect:/home";
}

// 各子应用继承父域 Cookie
@Bean
public CookieSerializer cookieSerializer() {
    DefaultCookieSerializer serializer = new DefaultCookieSerializer();
    serializer.setDomainNamePattern("^.+?\\.(\\w+\\.[a-z]+)$");
    return serializer;
}
```

# 6. 方案四：SAML/OIDC 跨域联合

```
组织 A (company-a.com)                    组织 B (partner.com)
┌────────────────────┐                   ┌────────────────────┐
│  IdP (Azure AD)     │                   │  IdP (Okta)         │
│  用户: alice@a.com  │                   │  用户: bob@b.com    │
└─────────┬──────────┘                   └─────────┬──────────┘
          │                                        │
          │      ┌────────────────────┐            │
          └──────┤  SaaS 平台 (SP)    ├────────────┘
                 │  saas.example.com  │
                 └────────────────────┘

用户 alice@a.com 访问 SaaS 平台：
  → SaaS 平台 redirect 到 Azure AD
  → Alice 在 Azure AD 认证
  → Azure AD 签发 SAML Assertion
  → POST 到 SaaS 平台
  → SaaS 平台验证 Assertion，创建 Session

用户 bob@b.com 访问同一个 SaaS 平台：
  → SaaS 平台 redirect 到 Okta
  → Bob 在 Okta 认证
  → Okta 签发 SAML Assertion
  → POST 到 SaaS 平台
  → SaaS 平台验证 Assertion，创建 Session
```

# 7. 移动端跨域 SSO

## 7.1 移动端 SSO 特有方案

```java
/**
 * 移动端 SSO 方案
 * 与 Web 端不同，移动端没有稳定的 Cookie 机制
 */

// 方案 1: Chrome Custom Tabs / SFSafariViewController
// App 内打开浏览器 → 在浏览器中完成认证 → Cookie 在系统和 App 间共享
// 仅限同一设备、同一浏览器引擎

// 方案 2: App Links / Universal Links
// 认证回调 URL 通过操作系统调度到特定 App
// Android: https://app.example.com/callback → 如果 App 已安装则打开 App
// iOS: 类似机制

// 方案 3: 设备级 SSO（Keychain/Keystore 共享）
// 同一开发者签名的 App 可以共享 Keychain（iOS）/ SharedUserId（Android）
// App A 写入 Token 到共享存储
// App B 读取共享存储中的 Token
```

# 8. 安全注意事项

| 风险 | 说明 | 防御 |
|------|------|------|
| **Token 在 URL 泄露** | 302 重定向携带 Token，URL 可被截获 | 使用 code 而非直接传 Token；启用 PKCE |
| **redirect_uri 劫持** | 攻击者注册恶意回调地址 | 白名单精确校验 redirect_uri |
| **PostMessage 劫持** | iframe 的 postMessage 被恶意页面监听 | 验证 `event.origin`，不要使用 `*` |
| **CSRF** | 跨域伪造登出请求 | state 参数 + 同站 Cookie（SameSite） |
| **第三方 Cookie 限制** | 浏览器逐步禁用第三方 Cookie | 使用 OIDC 等标准协议（不依赖第三方 Cookie） |

# 9. 总结

| 场景 | 推荐方案 |
|------|---------|
| 同一组织的多个子域 | 共享父域 Cookie（方案三） |
| 完全不同的域名 | 302 重定向 + Code/Token 传递（方案一） |
| 跨组织联合 SSO | SAML 2.0 / OIDC（方案四） |
| 前端 SPA 静默检测 | iframe + postMessage（方案二） |
| 微服务网关 | 网关层统一认证 |
| 移动端 | App Links + 共享 Keychain/Keystore |
