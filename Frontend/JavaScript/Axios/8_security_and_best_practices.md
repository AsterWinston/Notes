前端安全有一条容易被忽视的分界线：**Axios 本身不提供安全能力**。它不会帮你防 XSS、不会替你校验 CSRF，也不会因为用了 `withCredentials` 就自动安全。它只是把请求发出去。真正决定安全性的，是 Token 放哪里、Cookie 怎么配、CORS 怎么设、日志记录了什么。

# 1. 威胁模型

## 1.1 攻击面

| 攻击 | 入口 | 后果 |
|------|------|------|
| XSS | 恶意脚本注入页面 | 读取 localStorage、冒充用户发请求 |
| CSRF | 第三方站点诱导浏览器带 Cookie | 以用户身份执行操作 |
| CORS 配置错误 | 服务端放开任意 Origin | 任意站点读取用户数据 |
| Token 泄露 | 日志、URL、错误上报 | 会话被劫持 |
| 重试风暴 | 失败后无上限重试 | 服务端被打挂、费用飙升 |
| 资源耗尽 | 超大响应、无限下载 | 浏览器或 Node 进程崩溃 |

## 1.2 一条原则

★ **关键认知**：**永远不要信任客户端**。前端的校验、隐藏按钮、CORS 配置都只是"提高攻击成本"，不是安全边界。⚠ 真正的权限校验、金额计算、文件类型检查必须在服务端完成；把管理员接口藏在"前端不显示入口"后面，等于没有防护。

# 2. Token 存储

## 2.1 三种位置

`localStorage` 简单、跨标签页，但任何同源 JS 都能读；`httpOnly Cookie` JS 读不到，但浏览器会自动携带；内存变量刷新即丢，XSS 无法存下来带走，但页面被控制期间仍可发请求。

## 2.2 对比

| 方式 | XSS 可读 | CSRF 风险 | 刷新后保留 | 多标签共享 |
|------|---------|----------|-----------|-----------|
| localStorage | 是 | 低 | 是 | 是 |
| sessionStorage | 是 | 低 | 否（仅当前标签） | 否 |
| 内存 | 否（但可被使用） | 低 | 否 | 否 |
| httpOnly Cookie | 否 | 高（需防护） | 是 | 是 |

## 2.3 推荐组合

```text
access token  → 内存，短有效期（5 到 15 分钟）
refresh token → httpOnly + Secure + SameSite Cookie，长有效期
刷新接口      → 只接受 refresh cookie，返回新的 access token
```

★ **关键认知**：**httpOnly 防的是"偷走"，不是"使用"**。XSS 在页面内仍然可以调用你的 `request()` 发请求。因此 httpOnly 不能替代 XSS 防护，只能降低 Token 被带离站点的风险。

## 2.4 存储注意事项

- 不要把 Token 放进 URL 查询串：会进入浏览器历史、Referer、服务器访问日志。
- 登出时同时清理内存、Cookie、localStorage，并通知服务端吊销会话。
- 多标签页登出要同步，可监听 `storage` 事件。

# 3. XSS 与请求层

## 3.1 XSS 如何利用请求层

```text
恶意脚本注入
      ├─ 读取 localStorage 中的 token
      ├─ 直接调用页面里的 http 实例
      ├─ 篡改请求参数（转账账号、金额）
      └─ 把响应数据外传到攻击者服务器
```

即使 Token 存在 httpOnly Cookie 里，攻击者也能用页面自己的请求封装发起操作。所以 XSS 防护是根本。

## 3.2 防护要点

| 措施 | 说明 |
|------|------|
| 输出转义 | React/Vue 默认转义文本，别用 `dangerouslySetInnerHTML` / `v-html` |
| HTML sanitize | 必须渲染富文本时用 DOMPurify 等库过滤 |
| CSP | 限制脚本来源，禁用内联脚本与 `eval` |
| 输入校验 | 服务端校验所有输入，前端只做体验优化 |
| 依赖审计 | 定期 `npm audit`，避免引入带恶意脚本的包 |

```text
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://trusted.cdn.com;
  object-src 'none';
```

## 3.3 URL 拼接注入

用户输入直接拼进路径时，`../` 或 `?` 可能改变请求路径与参数；至少要用 `encodeURIComponent` 编码路径片段，更严格的做法是参数化接口或 ID 格式白名单。

## 3.4 限制可请求的域名

如果封装层允许调用方传完整 URL，攻击者可以通过 XSS 把请求发到自己的域名，带走数据。可以给实例设置：

```javascript
const http = axios.create({
  baseURL: 'https://api.example.com',
  allowAbsoluteUrls: false // Axios 1.8+，禁止绝对 URL 绕过 baseURL
})
```

Node/SSR 场景还要防 SSRF：不要让用户控制请求目标，必须做域名白名单与内网地址拦截。

# 4. CSRF

## 4.1 原理

Cookie 会自动随请求发送。用户在已登录 A 站的情况下访问恶意 B 站，B 站可以构造一个向 A 站发起请求的表单或图片；浏览器会带上 A 站的 Cookie，服务端如果只认 Cookie，就会把这次操作当成用户本人发起。

## 4.2 SameSite

```text
Set-Cookie: session=...; HttpOnly; Secure; SameSite=Lax
```

`Strict` 任何跨站请求都不带 Cookie；`Lax` 顶层导航的 GET 带、跨站 POST 不带（默认）；`None` 跨站也带，必须配合 `Secure`。`Lax` 能挡住大部分跨站 POST，但老浏览器、同站子域、部分 GET 操作仍可能被利用。

## 4.3 CSRF Token

双重提交方案：

```text
1. 服务端设置可被 JS 读取的 Cookie: XSRF-TOKEN=abc
2. 前端从 Cookie 读出，放进请求头: X-XSRF-TOKEN: abc
3. 服务端比对 Cookie 与请求头是否一致
```

攻击者能诱导浏览器带 Cookie，但**读不到 Cookie 的值**（跨域限制），因此无法伪造请求头。Axios 内置了这套逻辑：

同源请求下，Axios 会自动读取 `XSRF-TOKEN` Cookie 并写入 `X-XSRF-TOKEN` 请求头；跨域时还需要 `withCredentials: true` 与 `withXSRFToken: true`，或手工读取 Cookie 设置请求头。⚠ 这套机制要求 `XSRF-TOKEN` Cookie **不是 httpOnly**；真正的会话 Cookie 仍然应该 httpOnly。

## 4.4 哪些接口需要防护

| 接口类型 | 是否需要 |
|---------|---------|
| 使用 Cookie 鉴权的写操作 | 需要 |
| 使用 `Authorization` 头的写操作 | 通常不需要 |
| GET 且无副作用 | 不需要 |
| 登录接口 | 需要防登录 CSRF |

★ **关键认知**：CSRF 的前提是"浏览器会自动带凭证"。如果鉴权完全依赖 `Authorization` 头，且 Token 存在 JS 内存里，CSRF 风险会显著降低；但只要用了 Cookie，就必须认真处理 SameSite 与 CSRF Token。

# 5. CORS

## 5.1 CORS 不是鉴权

CORS 是**浏览器**的同源策略扩展，用来限制"某个源的前端能否读取另一个源的响应"。它不能阻止请求到达服务器，也不能阻止 curl、Postman、服务端脚本访问接口。

★ **关键认知**：不要用 CORS 当权限控制。`Access-Control-Allow-Origin: *` 不等于"公开数据"，只等于"任意网页的 JS 都能读响应"。真正的鉴权必须在服务端校验身份。

## 5.2 预检请求

非简单请求会先发 `OPTIONS` 预检：

```text
OPTIONS /api/users
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: authorization, content-type
```

服务端需要返回具体的 `Access-Control-Allow-Origin`、允许的方法与请求头，带凭证时再加 `Access-Control-Allow-Credentials: true`。触发预检的常见条件是自定义请求头、`Content-Type: application/json`、PUT/DELETE 等非简单方法。

## 5.3 带凭证的坑

跨域携带 Cookie 时，实例要设置 `withCredentials: true`。服务端必须返回**具体 Origin**，不能是 `*`，并设置 `Access-Control-Allow-Credentials: true`。如果响应头需要被 JS 读取（如 `Content-Disposition`、`X-Total-Count`），还要加 `Access-Control-Expose-Headers`。

⚠ 常见报错"The value of the 'Access-Control-Allow-Origin' header must not be the wildcard '*'"就是 `withCredentials: true` 配了 `*`。

## 5.4 开发环境代理

本地开发时前端在 `localhost:5173`、后端在 `localhost:8080`，可以直接用开发服务器代理：

```javascript
// vite.config.js
export default {
  server: {
    proxy: {
      '/api': { target: 'http://localhost:8080', changeOrigin: true }
    }
  }
}
```

代理让浏览器认为请求同源，从而绕开 CORS。⚠ 代理只解决开发期问题，生产环境仍需正确配置 CORS 或同源部署。

# 6. 超时、重试与资源保护

## 6.1 必设超时与体积上限

Axios 默认 `timeout: 0`，表示不限制。生产环境应设置 10 秒左右的超时，并用 `maxContentLength` / `maxBodyLength` 限制请求与响应体积（例如 50 MB）。没有上限时，一个异常接口可能返回几百 MB，直接把标签页或 Node 进程拖垮。

## 6.2 重试风暴

```text
服务端 503 → 所有客户端同时重试 → 服务端更慢
          → 更多超时 → 更多重试
```

防护手段：最大重试 1 到 3 次、指数退避（1s、2s、4s）、随机抖动打散时间、只重试幂等请求、429/503 时尊重 `Retry-After`。

```javascript
function backoff(attempt) {
  const base = Math.min(1000 * 2 ** attempt, 8000)
  return base + Math.random() * 300
}
```

## 6.3 幂等键

对下单、支付这类不能重复的写操作，服务端应支持幂等键：

```javascript
await http.post('/orders', data, {
  headers: { 'Idempotency-Key': crypto.randomUUID() }
})
```

客户端重试时复用同一个 key，服务端据此去重。仅靠前端"不重试"不足以保证安全，网络层的重复投递仍可能发生。

# 7. 日志脱敏

## 7.1 哪些字段不能进日志

| 类别 | 示例 |
|------|------|
| 认证信息 | `Authorization`、`Cookie`、`Set-Cookie`、`X-Api-Key` |
| Token | `access_token`、`refresh_token`、`id_token` |
| 凭据 | `password`、`secret`、`privateKey` |
| 个人信息 | 手机号、身份证号、银行卡号、地址 |
| 业务敏感数据 | 余额、交易明细、聊天内容 |

## 7.2 递归脱敏

```javascript
const SENSITIVE_KEYS = new Set([
  'authorization', 'cookie', 'set-cookie', 'password',
  'secret', 'token', 'access_token', 'refresh_token', 'x-api-key'
])

function redact(value, seen = new WeakSet(), depth = 0) {
  if (depth > 6) return '[MaxDepth]'
  if (value === null || typeof value !== 'object') return value
  if (seen.has(value)) return '[Circular]'
  seen.add(value)

  if (Array.isArray(value)) {
    return value.map((item) => redact(item, seen, depth + 1))
  }
  return Object.fromEntries(
    Object.entries(value).map(([key, item]) => [
      key,
      SENSITIVE_KEYS.has(key.toLowerCase()) ? '[REDACTED]' : redact(item, seen, depth + 1)
    ])
  )
}
```

## 7.3 上报前处理 AxiosError

上报时只挑必要字段（`message`、`code`、`status`、`url`、`method`），参数与请求体先过 `redact()`，不要直接展开整个 config。

★ **关键认知**：`AxiosError.toJSON()` 会包含请求配置，配置里可能有 `Authorization` 头。⚠ 直接把 `JSON.stringify(error)` 丢进监控平台，等于把 Token 主动上传。日志脱敏必须在**上报入口**统一做。

# 8. 最佳实践清单

| 项 | 建议 |
|----|------|
| Token | access 放内存，refresh 放 httpOnly Cookie |
| Cookie | `HttpOnly; Secure; SameSite=Lax/Strict` |
| CSRF | Cookie 鉴权必须配 CSRF Token 或 SameSite |
| XSS | 输出转义、富文本 sanitize、CSP |
| CORS | 白名单 Origin，带凭证时禁用 `*` |
| 超时 | 全局 10 秒左右，特殊接口单独放宽 |
| 体积 | 设置 `maxContentLength` / `maxBodyLength` |
| 重试 | 仅幂等请求、限次数、指数退避、抖动 |
| 取消 | 页面切换时 abort，避免无意义请求 |
| 日志 | 统一脱敏，不上报 Token 与个人信息 |

# 9. 练习

## 9.1 练习一：审计 Token 存储

检查一个现有项目：Token 存在哪里、是否 httpOnly、是否进过 URL 或日志、登出时是否清理干净。列出至少三个风险点并给出改法。

## 9.2 练习二：配置 CSRF

假设后端使用 Cookie 鉴权，接口为 `POST /api/transfer`。写出 Axios 实例配置、服务端需要设置的 Cookie 属性，以及请求头如何携带 CSRF Token。

## 9.3 练习三：实现脱敏

用 `redact()` 处理一个包含 `headers.Authorization`、`data.password`、`params.phone` 的对象，验证敏感值都变成 `[REDACTED]`，并处理循环引用不报错。

# 10. 小结

| 主题 | 要点 |
|------|------|
| Token | 内存 + httpOnly refresh 是较稳的组合 |
| XSS | 根本防线，httpOnly 不能替代 |
| CSRF | Cookie 自动携带是前提，SameSite + Token |
| CORS | 浏览器策略，不是鉴权 |
| 资源 | 超时、体积上限、有限重试 |
| 日志 | 上报前统一脱敏 |

下一篇进入统一封装与测试：如何把实例、拦截器、错误归一化、取消、去重组合成一个可维护的请求层，如何用 Mock 与 Vitest 覆盖成功、失败、取消三条路径，并给出最佳实践总结。
