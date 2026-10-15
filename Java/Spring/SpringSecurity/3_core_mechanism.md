# 1. 安全过滤器链（SecurityFilterChain）

Spring Security 的本质是一条 **Servlet Filter 链**。每个 HTTP 请求都依次经过下面的 Filters：

## 完整过滤器执行顺序

| 序号 | Filter | 作用 |
|------|--------|------|
| 1 | `DisableEncodeUrlFilter` | 禁止 URL 中携带 JSESSIONID |
| 2 | `WebAsyncManagerIntegrationFilter` | 跨线程传递 SecurityContext |
| 3 | `SecurityContextPersistenceFilter` | 请求前加载/请求后保存 SecurityContext |
| 4 | `HeaderWriterFilter` | 添加安全响应头 |
| 5 | `CorsFilter` | 跨域处理 |
| 6 | `CsrfFilter` | CSRF 攻击防护 |
| 7 | `LogoutFilter` | 处理 `/logout` 请求 |
| 8 | `UsernamePasswordAuthenticationFilter` | 表单登录认证 |
| 9 | `BasicAuthenticationFilter` | HTTP Basic 认证 |
| 10 | `RequestCacheAwareFilter` | 恢复登录前的请求 |
| 11 | `SecurityContextHolderAwareRequestFilter` | 包装 HttpServletRequest |
| 12 | `RememberMeAuthenticationFilter` | 记住我自动登录 |
| 13 | `AnonymousAuthenticationFilter` | 未登录用户授予匿名身份 |
| 14 | `ExceptionTranslationFilter` | 异常转换（401/403） |
| 15 | `AuthorizationFilter` | 授权（替代 FilterSecurityInterceptor） |
| 16 | `FilterSecurityInterceptor` | 旧版授权拦截器（6.x 仍可用） |

> **注意：** 6.x 中 `AuthorizationFilter` 是新的授权过滤器，`FilterSecurityInterceptor` 仍可用但非默认。

### 1.1 UsernamePasswordAuthenticationFilter

处理 **表单登录** (`POST /login`) 的过滤器：

```java
public class UsernamePasswordAuthenticationFilter 
        extends AbstractAuthenticationProcessingFilter {
    
    // 默认拦截 POST /login，参数名为 username、password
    public static final String SPRING_SECURITY_FORM_USERNAME_KEY = "username";
    public static final String SPRING_SECURITY_FORM_PASSWORD_KEY = "password";
    
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, 
            HttpServletResponse response) {
        String username = obtainUsername(request);
        String password = obtainPassword(request);
        UsernamePasswordAuthenticationToken authRequest = 
            UsernamePasswordAuthenticationToken.unauthenticated(username, password);
        return this.getAuthenticationManager().authenticate(authRequest);
    }
}
```

**认证成功后流程：**
1. `SessionAuthenticationStrategy` 处理 Session（如更换 Session ID）
2. 将 Authentication 存入 SecurityContextHolder
3. 调用 `RememberMeServices.loginSuccess()`
4. 调用 `AuthenticationSuccessHandler`
5. 发布 `AuthenticationSuccessEvent`

## 1.2 BasicAuthenticationFilter

处理 **HTTP Basic 认证**，从请求头 `Authorization: Basic base64(user:pass)` 提取用户名密码：

```
Authorization: Basic YWRtaW46MTIzNDU2
```

```java
// 前端调用示例
fetch('/api/users', {
    headers: { 'Authorization': 'Basic ' + btoa('admin:123456') }
})
```

## 1.3 CsrfFilter

防止**跨站请求伪造**（Cross-Site Request Forgery）攻击。

**攻击原理：** 攻击者诱导用户点击一个链接，利用用户已登录的身份，向目标站点发送恶意请求。

**防护机制：** 服务端生成随机 Token，前端请求必须携带：
```html
<!-- 表单中 -->
<input type="hidden" name="_csrf" value="token-value" />

<!-- AJAX 请求头中 -->
<meta name="_csrf" content="{{_csrf.token}}">
<meta name="_csrf_header" content="{{_csrf.headerName}}">
```

## 1.4 其他重要过滤器

**ExceptionTranslationFilter：**
```java
// 它捕获 AuthenticationException → 重定向到登录页
// 它捕获 AccessDeniedException → 返回 403 Forbidden
```

**SecurityContextPersistenceFilter：**
```java
// 6.x 中默认使用 SecurityContextHolderFilter
// 请求前：从 SecurityContextRepository 加载到 SecurityContextHolder
// 请求后：清空 SecurityContextHolder（ThreadLocal 防泄漏）
```

---

# 2. AuthenticationManager 详解

## 2.1 ProviderManager 工作原理

```java
public class ProviderManager implements AuthenticationManager {
    private List<AuthenticationProvider> providers;

    @Override
    public Authentication authenticate(Authentication authentication) 
            throws AuthenticationException {
        for (AuthenticationProvider provider : getProviders()) {
            if (!provider.supports(authentication.getClass())) {
                continue;  // 跳过不支持的 Provider
            }
            try {
                result = provider.authenticate(authentication);
                // 认证成功后，拷贝并擦除密码
                copyDetails(authentication, result);
                break;
            } catch (AccountStatusException | InternalAuthenticationServiceException e) {
                // 账户锁定、禁用等——直接抛出不尝试其他 Provider
                throw e;
            } catch (AuthenticationException e) {
                // BadCredentials——继续尝试下一个 Provider
                lastException = e;
            }
        }
        throw lastException;
    }
}
```

**关键行为：**
- 遍历所有 Provider，找到第一个 `supports()` 且成功认证的
- **AccountStatusException** 直接抛出（账户锁定/禁用不应该尝试其他方式）
- **BadCredentialsException** 累积到最后才抛出（可能还有其他 Provider）

## 2.2 DaoAuthenticationProvider

最常用的认证 Provider，负责从 **UserDetailsService** 加载用户并验证密码：

```java
public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {
    
    @Override
    protected void additionalAuthenticationChecks(UserDetails userDetails,
            UsernamePasswordAuthenticationToken authentication) 
            throws AuthenticationException {
        String presentedPassword = authentication.getCredentials().toString();
        if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
            throw new BadCredentialsException("密码错误");
        }
    }
}
```

## 2.3 自定义 AuthenticationProvider

**场景：** 手机验证码登录、微信扫码登录等自定义认证方式：

```java
@Component
public class SmsCodeAuthenticationProvider implements AuthenticationProvider {

    @Autowired
    private RedisTemplate redisTemplate;

    @Override
    public Authentication authenticate(Authentication auth) 
            throws AuthenticationException {
        String phone = auth.getName();
        String code = auth.getCredentials().toString();

        // 从 Redis 校验验证码
        String cachedCode = (String) redisTemplate.opsForValue().get("sms:" + phone);

        if (cachedCode == null || !cachedCode.equals(code)) {
            throw new BadCredentialsException("验证码错误或已过期");
        }

        // 查数据库加载用户
        List<GrantedAuthority> authorities = loadUserAuthorities(phone);
        return new SmsCodeAuthenticationToken(phone, null, authorities);
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return SmsCodeAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

---

# 3. Session 管理

## 3.1 并发控制

限制同一账号同时登录的客户端数量：

```java
http.sessionManagement(session -> session
    .maximumSessions(1)           // 最多同时1个会话
    .maxSessionsPreventsLogin(false)  // false: 踢掉旧会话; true: 阻止新登录
    .expiredSessionStrategy(event -> {
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("{\"code\":401,\"msg\":\"会话已过期\"}");
    })
);
```

## 3.2 Session Fixation 防护

**攻击原理：** 攻击者先获取一个 Session ID，诱骗用户使用该 Session ID 登录，之后攻击者可用同一 Session ID 冒充用户。

**防护机制（Spring Security 默认启用）：**

```java
http.sessionManagement(session -> session
    .sessionFixation(newSessionStrategy) // 登录后更换新 Session ID
);
```

| 策略 | 说明 |
|------|------|
| `changeSessionId()` | 仅换 Session ID，保留内容（默认，推荐） |
| `migrateSession()` | 创建新 Session，拷贝旧内容 |
| `newSession()` | 全新 Session，不拷贝任何内容 |
| `none()` | 不防护（不安全） |

## 3.3 记住我（Remember-Me）

**两种实现方式：**

**① 基于 Hash 的简单方式（不推荐生产环境）：**
```java
http.rememberMe(remember -> remember
    .key("uniqueAndSecret")
    .tokenValiditySeconds(86400) // 1天
);
```

Cookie 内容：`base64(username + ":" + expiryTime + ":" + MD5(username + expiryTime + password + key))`

**② 基于持久化 Token（推荐）：**
```sql
CREATE TABLE persistent_logins (
    username  VARCHAR(64) NOT NULL,
    series    VARCHAR(64) PRIMARY KEY,
    token     VARCHAR(64) NOT NULL,
    last_used TIMESTAMP NOT NULL
);
```

```java
http.rememberMe(remember -> remember
    .tokenRepository(persistentTokenRepository())
    .tokenValiditySeconds(604800) // 7天
);
```

---

# 4. CSRF 防护

## 4.1 原理与作用

**CSRF 攻击流程：**
```
1. 用户登录银行网站 bank.com
2. 用户访问恶意网站 evil.com
3. evil.com 暗藏 <form action="bank.com/transfer" ...>
4. 浏览器自动携带 bank.com 的 Cookie 提交表单
5. 转账成功（用户不知情）
```

**防护原理：** 服务端生成随机 Token，只有合法页面能获取到，非法的跨域请求无法读取该 Token。

## 4.2 默认启用

Spring Security 默认启用 CSRF 防护，自动生成 `XSRF-TOKEN` Cookie 和对应校验逻辑。

## 4.3 自定义配置与禁用

**REST API（无状态的，用 JWT）通常禁用 CSRF：**
```java
http.csrf(csrf -> csrf.disable());
```

**前后端分离保留 CSRF 的方案：**
```java
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())
);
```

> **何时禁用 CSRF？** 前后端分离 + JWT 无状态认证（无 Session → 无 CSRF 攻击面），可以安全禁用。
