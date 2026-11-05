跨域和安全问题往往在前后端联调时才暴露，但根因在浏览器同源策略和服务端响应头。本篇解释同源、CORS 预检和常见 Web 安全风险，并给出 Spring Boot 与网关层的配置思路。

# 1. 同源策略

## 1.1 什么是同源

同源要求**协议、域名、端口**三者完全相同：

| 页面地址 | 请求地址 | 是否同源 | 原因 |
|----------|----------|----------|------|
| `https://example.com/a` | `https://example.com/b` | 是 | 协议、域名、端口一致 |
| `https://example.com` | `http://example.com` | 否 | 协议不同 |
| `https://example.com` | `https://api.example.com` | 否 | 域名不同 |
| `https://example.com:443` | `https://example.com:8443` | 否 | 端口不同 |

同源策略限制的是浏览器中的脚本读取跨源响应，不是限制请求发出。很多请求实际上已经到达服务端，只是浏览器不允许前端读取响应。

## 1.2 跨域请求的分类

| 类型 | 条件 | 是否预检 |
|------|------|----------|
| 简单请求 | GET/HEAD/POST + 安全头 + 简单 Content-Type | 否 |
| 预检请求 | PUT/DELETE、自定义头、`application/json` 等 | 是 |
| 带凭证请求 | 携带 Cookie 或 `Authorization` 且需要跨域 | 需配合 `credentials` |

简单请求的 `Content-Type` 只允许：

- `application/x-www-form-urlencoded`
- `multipart/form-data`
- `text/plain`

因此前后端分离项目发送 JSON 时，通常会触发 OPTIONS 预检。

# 2. CORS

## 2.1 简单请求

浏览器自动带上 `Origin`：

```http
GET /api/users HTTP/1.1
Origin: https://app.example.com
```

服务端允许时返回：

```http
Access-Control-Allow-Origin: https://app.example.com
```

如果响应头不匹配，浏览器会阻止前端读取响应，控制台报 CORS 错误。

## 2.2 预检请求

预检请求先询问服务端是否允许实际请求：

```http
OPTIONS /api/users HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type, authorization
```

服务端响应：

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 3600
```

`Access-Control-Max-Age` 让浏览器缓存预检结果，减少 OPTIONS 请求次数。

## 2.3 响应头

| 响应头 | 作用 |
|--------|------|
| `Access-Control-Allow-Origin` | 允许的来源，不能同时用 `*` 和凭证 |
| `Access-Control-Allow-Methods` | 允许的方法 |
| `Access-Control-Allow-Headers` | 允许的请求头 |
| `Access-Control-Allow-Credentials` | 是否允许携带 Cookie |
| `Access-Control-Expose-Headers` | 允许前端读取的响应头 |
| `Access-Control-Max-Age` | 预检缓存时间 |

Spring Boot 配置示例：

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("Content-Type", "Authorization")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

⚠ 使用 `allowCredentials(true)` 时，`allowedOrigins` 不能写 `*`，必须列出来源。否则浏览器会拒绝。

# 3. 常见 Web 安全

## 3.1 XSS

XSS（跨站脚本）是攻击者把脚本注入页面，在受害者浏览器中执行。

| 类型 | 触发方式 | 例子 |
|------|----------|------|
| 存储型 | 恶意内容存入数据库，其他用户访问时执行 | 评论区注入脚本 |
| 反射型 | 恶意参数随响应立即返回 | 搜索结果页直接回显参数 |
| DOM 型 | 前端 JavaScript 把不可信数据写入 DOM | `innerHTML = location.hash` |

防御：

- 输出编码：根据 HTML、JS、URL、CSS 上下文选择编码方式。
- 使用框架的自动转义，避免直接拼接 HTML。
- 对富文本使用白名单过滤。
- 设置 CSP，限制脚本来源。

## 3.2 CSRF

CSRF（跨站请求伪造）利用用户已登录的 Cookie，诱导浏览器向目标站点发送请求。

```text
1. 用户登录 bank.com，浏览器保存 Cookie
2. 用户访问恶意网站
3. 恶意网站自动提交表单到 bank.com/transfer
4. 浏览器自动带上 bank.com 的 Cookie
5. 服务端认为是用户本人操作
```

防御：

- `SameSite=Lax` 或 `Strict`。
- CSRF Token：表单或请求头中携带服务端签发的随机值。
- 校验 `Origin` / `Referer`。
- 敏感操作二次验证。

⚠ CORS 不能防 CSRF。CORS 限制的是跨源读取响应，而 CSRF 只要求请求被发送。

## 3.3 SSRF

SSRF（服务端请求伪造）是攻击者让服务端去请求任意地址：

```text
用户提交 URL → 服务端发起请求 → 访问内网、云元数据、localhost
```

高风险目标：

- `http://127.0.0.1`
- `http://169.254.169.254`（云元数据）
- 内网网段地址
- `file://`、`gopher://` 等协议

防御：

- URL 白名单，只允许访问明确的域名。
- 解析目标 IP，拒绝内网、回环、链路本地地址。
- 禁止危险协议，只允许 HTTP/HTTPS。
- 关闭不必要的重定向，或对重定向目标重新校验。

# 4. 安全响应头

| 响应头 | 作用 | 示例 |
|--------|------|------|
| `Content-Security-Policy` | 限制脚本、样式、图片等来源 | `default-src 'self'` |
| `X-Content-Type-Options` | 禁止 MIME 嗅探 | `nosniff` |
| `X-Frame-Options` | 防止页面被嵌入 iframe | `DENY` |
| `Strict-Transport-Security` | 强制后续使用 HTTPS | `max-age=31536000` |
| `Referrer-Policy` | 控制 Referer 泄露 | `strict-origin-when-cross-origin` |

Spring Security 可以统一配置这些头，不要在业务代码里逐个接口添加。

# 5. 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|----------|
| `Allow-Origin: *` 又带凭证 | 浏览器拒绝请求 | 明确列出来源 |
| 只处理实际请求，不处理 OPTIONS | 预检失败 | 正确响应预检 |
| 把 CORS 当鉴权 | 攻击者仍可直接请求服务端 | 单独做认证和授权 |
| 只过滤 `<script>` | XSS 绕过 | 上下文输出编码 + CSP |
| 只靠 SameSite 防 CSRF | 老浏览器或配置不当失效 | 组合 CSRF Token |
| 允许任意 URL 访问 | SSRF | 白名单 + 内网地址拦截 |

# 6. 小结

| 主题 | 要点 |
|------|------|
| 同源 | 协议、域名、端口完全一致 |
| CORS | 简单请求、预检请求、带凭证请求三种情况 |
| 配置 | 允许来源要明确，凭证与 `*` 不能共存 |
| 安全 | XSS 靠输出编码，CSRF 靠 SameSite/Token，SSRF 靠白名单 |
| 响应头 | CSP、HSTS、nosniff、X-Frame-Options 组合使用 |

下一篇是模块总结，用知识地图、自检问题和排查工具帮助复习 HTTP 全模块。
