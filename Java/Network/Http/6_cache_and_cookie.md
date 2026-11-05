缓存和 Cookie 是 HTTP 中最容易被"能用就行"掩盖的细节。本篇讲清强缓存与协商缓存的区别、Cache-Control 的常见组合、Cookie 的安全属性，以及 Session 在分布式环境下的处理方式。

# 1. HTTP 缓存

## 1.1 强缓存

强缓存命中时，浏览器直接使用本地副本，不向服务器发请求。

| 响应头 | 含义 | 优先级 |
|--------|------|--------|
| `Expires` | 绝对时间，例如 `Wed, 21 Oct 2026 07:28:00 GMT` | 低 |
| `Cache-Control: max-age=3600` | 相对时间，单位秒 | 高 |

```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=31536000, immutable
```

`max-age` 从响应生成时开始计算，不受客户端时钟影响，因此比 `Expires` 更可靠。`immutable` 表示资源在有效期内不会变化，浏览器可以跳过重新验证。

★ **关键认知**：强缓存期间，浏览器不会请求服务器。即使服务端更新了文件，只要 URL 不变且缓存未过期，用户看到的仍是旧版本。因此静态资源通常配合文件名哈希使用。

## 1.2 协商缓存

协商缓存需要向服务器确认资源是否变化：

```text
浏览器                                    服务器
  │ ── GET /app.js ──────────────────────> │
  │    If-None-Match: "abc123"             │
  │                                        │
  │ <── 304 Not Modified ───────────────── │  未变化，使用本地副本
```

两种验证方式：

| 响应头 | 请求头 | 依据 |
|--------|--------|------|
| `Last-Modified` | `If-Modified-Since` | 文件最后修改时间 |
| `ETag` | `If-None-Match` | 文件内容标识 |

ETag 比 Last-Modified 更精确：

- 文件内容变了但修改时间没变，Last-Modified 会误判。
- 文件被重新保存但内容没变，Last-Modified 会导致不必要的传输。

304 响应没有消息体。如果服务端在 304 中返回 body，客户端通常会忽略，但这是不规范的。

## 1.3 Cache-Control 实战

```http
# 带哈希的静态资源，长期缓存
Cache-Control: public, max-age=31536000, immutable

# HTML 入口，每次协商
Cache-Control: no-cache

# 用户私有数据，不缓存
Cache-Control: no-store

# API 短缓存，允许 CDN 缓存但需要重新验证
Cache-Control: public, max-age=60, must-revalidate
```

`no-cache` 不是"不缓存"，而是"使用前必须向服务器验证"。`no-store` 才是完全不存储。

# 2. Cookie

## 2.1 Cookie 属性

```http
Set-Cookie: sessionId=abc123; Domain=example.com; Path=/; Max-Age=3600; HttpOnly; Secure; SameSite=Lax
```

| 属性 | 作用 |
|------|------|
| `Domain` | 允许发送 Cookie 的域名 |
| `Path` | 允许发送 Cookie 的路径前缀 |
| `Expires` / `Max-Age` | 过期时间；不设置则为会话 Cookie |
| `HttpOnly` | JavaScript 无法读取，降低 XSS 窃取风险 |
| `Secure` | 只在 HTTPS 下发送 |
| `SameSite` | 控制跨站请求是否携带 Cookie |

## 2.2 安全属性

- **HttpOnly**：防止 `document.cookie` 读取，降低 XSS 窃取会话的风险，但不能阻止 XSS 本身。
- **Secure**：防止 Cookie 在 HTTP 明文连接中传输。
- **SameSite**：
  - `Strict`：完全禁止跨站携带。
  - `Lax`：顶层导航的 GET 请求可携带，POST 跨站不携带。
  - `None`：允许跨站，但必须同时设置 `Secure`。

★ **关键认知**：HttpOnly 防的是"读 Cookie"，不是"利用 Cookie 发请求"。如果存在 XSS，攻击者仍可以在页面内发起请求，因此输出编码、CSP 等措施同样重要。

# 3. Session

## 3.1 服务端会话

传统 Session 流程：

```text
1. 用户登录成功
2. 服务端创建 Session，保存用户信息
3. 返回 Set-Cookie: sessionId=...
4. 后续请求携带 sessionId
5. 服务端根据 sessionId 找到会话
```

Session 数据通常存在内存、Redis 或数据库中。Session ID 必须足够随机，不能包含用户信息。

## 3.2 分布式会话

多实例部署时，用户第一次请求可能落到实例 A，第二次落到实例 B。如果 Session 只存在实例 A 的内存中，实例 B 就找不到会话。

| 方案 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| 粘性会话 | 负载均衡按 Session ID 固定到同一实例 | 实现简单 | 实例故障会丢失会话，扩缩容不均衡 |
| Session 共享 | 所有实例读写同一个 Redis | 无状态实例、易扩展 | 依赖 Redis，增加一次网络调用 |
| Token | 客户端保存签名 Token | 服务端无状态 | 撤销困难，需额外机制 |
| 混合 | 短期 Token + Redis 会话 | 兼顾扩展和撤销 | 架构更复杂 |

生产环境更常见的是 Session 共享或 Token 方案。无论哪种，都要设置合理过期时间，并在登出、改密码等场景使旧凭据失效。

# 4. CDN 缓存

CDN 缓存键通常由 URL、查询参数、部分请求头组成。配置不当会导致：

- 不同用户看到彼此的数据（缓存键未包含用户标识）。
- 缓存命中率低（缓存键包含随机参数）。
- 更新后用户仍看到旧内容（未刷新缓存）。

```text
用户 → CDN 边缘节点 → 源站
         │
         ├── 命中：直接返回缓存
         └── 未命中：回源获取并缓存
```

实践建议：

- 公共静态资源使用长缓存 + 文件名哈希。
- 用户私有接口设置 `private, no-store`。
- 需要快速更新的资源使用 CDN 刷新或版本化 URL。
- 不要把带 `Authorization` 的响应缓存为公共资源。

# 5. 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|----------|
| 缓存了用户私有数据 | 用户 A 看到用户 B 的数据 | `private` 或 `no-store` |
| Cookie 未设 HttpOnly | XSS 可直接窃取会话 | 敏感 Cookie 加 HttpOnly |
| Cookie 未设 Secure | HTTP 请求泄露 Cookie | 生产环境加 Secure |
| SameSite=None 未配 Secure | 浏览器拒绝 Cookie | 同时设置 Secure |
| 304 返回 body | 违反协议 | 304 只返回头部 |
| 静态资源无哈希 | 更新后用户看到旧文件 | 文件名加内容哈希 |

# 6. 小结

| 主题 | 要点 |
|------|------|
| 强缓存 | 命中不发请求，靠 max-age/Expires |
| 协商缓存 | 用 ETag/Last-Modified 验证，命中返回 304 |
| Cache-Control | `no-cache` 要验证，`no-store` 不存储 |
| Cookie 安全 | HttpOnly、Secure、SameSite 三者配合 |
| Session | 分布式环境优先 Session 共享或 Token |
| CDN | 缓存键和私有数据边界要设计清楚 |

下一篇进入 Java HTTP 客户端，对比四种客户端并给出同步、异步、连接池和统一异常封装示例。
