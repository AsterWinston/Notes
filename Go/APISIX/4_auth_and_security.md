网关是流量的第一道门，认证和访问控制几乎是最先要配的东西。APISIX 把"调用方是谁"抽象成 Consumer，把认证方式做成可插拔的插件；理解这套模型，比记住某个插件的参数更重要。

# 1. 鉴权模型

## 1.1 Consumer 与凭据

**Consumer（消费者）** 代表一个调用方，可以是应用、租户、开发者，也可以是终端用户。它承载两类信息：

- 身份标识：`username`，全局唯一。
- 认证凭据：挂在 Consumer 的 `plugins` 里，比如 `key-auth` 的 key、`jwt-auth` 的 secret。

```json
{
  "username": "order-service",
  "plugins": { "key-auth": { "key": "order-service-key-2026" } }
}
```

★ **关键认知**：凭据属于 Consumer，不属于 Route。Route 上挂 `key-auth: {}` 只是"开启这种认证方式"，具体 key 长什么样、对应哪个调用方，全部在 Consumer 里定义。排查 401 时，先看 Consumer 是否存在，再看 Route 是否开启了对应插件。

## 1.2 鉴权在链路中的位置

```text
cors (4000) → ip-restriction (3000) → 认证插件 (2500 左右)
→ consumer-restriction (2400) → 转发上游
```

认证成功后，Consumer 信息写进 `ctx`：`ctx.consumer_name` 是 Consumer 名，上游默认能收到 `X-Consumer-Username` 和 `X-Credential-Identifier` 两个请求头。

# 2. Key Auth

## 2.1 配置

```bash
# 1. 创建 Consumer，凭据写在 Consumer 上
curl -X PUT "http://127.0.0.1:9180/apisix/admin/consumers" \
  -H "X-API-KEY: $ADMIN_KEY" -d '{
  "username": "order-service",
  "plugins": { "key-auth": { "key": "order-service-key-2026" } }
}'

# 2. 在路由上开启插件
curl -X PUT "http://127.0.0.1:9180/apisix/admin/routes/10" \
  -H "X-API-KEY: $ADMIN_KEY" -d '{
  "uri": "/api/orders/*",
  "plugins": { "key-auth": {} },
  "upstream": { "nodes": { "10.0.0.11:8080": 1 } }
}'
```

```bash
curl -H "apikey: order-service-key-2026" http://127.0.0.1:9080/api/orders/1
```

## 2.2 参数

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `header` | `apikey` | 从哪个请求头读取 key |
| `query` | `apikey` | 从哪个查询参数读取 key |
| `hide_credentials` | `false` | 是否在转发前删掉凭据 |

`hide_credentials: true` 会同时删掉请求头和查询参数里的凭据，避免上游日志把 key 打出来。

⚠ **关键认知**：把 key 放在查询参数里，会被访问日志、浏览器历史、代理日志多处记录。生产优先用请求头并配合 HTTPS；查询参数只作兼容。

# 3. Basic Auth

Basic Auth 使用标准头 `Authorization: Basic base64(username:password)`。

```bash
curl -X PUT "http://127.0.0.1:9180/apisix/admin/consumers" \
  -H "X-API-KEY: $ADMIN_KEY" -d '{
  "username": "report-user",
  "plugins": { "basic-auth": { "username": "report", "password": "s3cret-pass" } }
}'
# 路由挂 "basic-auth": {}，调用时用 curl -u report:s3cret-pass
```

Basic Auth 只是把用户名密码做 Base64，**不是加密**。它只适合内网服务、临时调试、已有 mTLS 或专线保护的链路；不适合公网 API 和浏览器前端。

★ **关键认知**：Basic Auth 的安全性完全依赖 TLS。没有 HTTPS 时，Base64 等同于明文。APISIX 会对密码做哈希保存，但不要在脚本里"读出来再改"，正确做法是直接覆盖写入新密码。

# 4. JWT

## 4.1 对称与非对称

JWT 由 Header、Payload、Signature 三部分组成，用 `.` 连接。`jwt-auth` 支持：

| 算法 | 密钥 | 特点 |
|------|------|------|
| HS256 / HS384 / HS512 | 共享 secret | 配置简单，适合内部服务 |
| RS256 / ES256 | 私钥签名、公钥验签 | 公钥可公开，适合多方协作 |

对称算法要求网关和签发方共享 secret，任何一方泄露都能伪造令牌。非对称只把公钥给网关，私钥留在认证服务，更安全但配置和轮换更复杂。

## 4.2 配置与签发

```bash
curl -X PUT "http://127.0.0.1:9180/apisix/admin/consumers" \
  -H "X-API-KEY: $ADMIN_KEY" -d '{
  "username": "mobile-app",
  "plugins": {
    "jwt-auth": {
      "key": "mobile-app",
      "secret": "replace-with-a-long-random-secret",
      "algorithm": "HS256",
      "exp": 7200
    }
  }
}'
```

路由上开启 `"jwt-auth": {}`，请求带 `Authorization: Bearer <token>`。Payload 里的 `key` 必须与 Consumer 的 `key` 一致，否则签名正确也会被拒绝。

```go
package main

import (
    "fmt"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

func main() {
    secret := []byte("replace-with-a-long-random-secret") // 与 Consumer 的 secret 一致

    claims := jwt.MapClaims{
        "key": "mobile-app",                         // 对应 Consumer 的 key
        "sub": "user-1001",                          // 业务用户标识
        "iat": time.Now().Unix(),                    // 签发时间
        "exp": time.Now().Add(2 * time.Hour).Unix(), // 过期时间
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, err := token.SignedString(secret)
    if err != nil {
        panic(err)
    }
    fmt.Println(signed)
}
```

```bash
TOKEN=$(go run main.go)
curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:9080/api/me
```

`exp`、`nbf` 会被严格校验，多节点部署必须校时。`lifetime_grace_period` 可以给跨机房场景留 5-30 秒宽限。

⚠ **关键认知**：JWT 一旦签发，在过期前无法单独撤销。需要"立即下线某个令牌"时，要么把有效期设短并配合刷新令牌，要么在网关外维护黑名单，要么改用 OIDC 的 introspection。

# 5. OIDC

## 5.1 两种模式

| 模式 | `bearer_only` | 适用 | 行为 |
|------|---------------|------|------|
| 浏览器模式 | `false` | 管理后台、需要登录页 | 无会话时 302 到 IdP，回调后建会话 |
| API 模式 | `true` | 移动端、服务间调用 | 只校验 Bearer，不跳转 |

```text
Browser                APISIX                     IdP
   │ GET /admin           │  无会话                 │
   │─────────────────────►│  302 /authorize         │
   │◄─────────────────────│                        │
   │ 登录并授权 ──────────────────────────────────►│
   │  回调 ?code=xxx ─────►│                        │
   │                      │  用 code 换 token       │
   │                      │───────────────────────►│
   │                      │◄──── id_token ─────────│
   │  Set-Cookie session  │  注入 X-Userinfo        │
   │◄─────────────────────│───────────────────────►│ 上游
```

## 5.2 配置

```json
{
  "openid-connect": {
    "client_id": "apisix",
    "client_secret": "$ENV://OIDC_CLIENT_SECRET",
    "discovery": "https://keycloak.example.com/realms/demo/.well-known/openid-configuration",
    "scope": "openid profile email",
    "redirect_uri": "https://api.example.com/callback",
    "bearer_only": false,
    "ssl_verify": true,
    "session": { "secret": "replace-with-a-32-byte-random-string" },
    "set_userinfo_header": true,
    "access_token_in_authorization_header": true
  }
}
```

- `redirect_uri` 必须与 IdP 登记的回调地址完全一致，包括协议、域名、路径和末尾斜杠。
- `session.secret` 用于加密会话 Cookie，必须足够长且各环境不同。
- `discovery` 必须能被 APISIX 访问，内网要确认 DNS 和出网策略。

★ **关键认知**：浏览器模式下，APISIX 会向上游注入用户信息头，但上游不能无条件信任。必须保证上游只接受来自网关的流量，否则攻击者可以绕过网关伪造 `X-Userinfo`。

# 6. 网络层防护

## 6.1 IP 黑白名单

```json
{
  "ip-restriction": {
    "whitelist": ["10.0.0.0/8", "192.168.1.0/24", "2001:db8::/32"]
  }
}
```

`whitelist` 和 `blacklist` 只能二选一，都支持 IPv4、IPv6 和 CIDR。拒绝时返回 403。

⚠ **关键认知**：IP 限制依赖 `remote_addr`。前面还有负载均衡或 CDN 时，`remote_addr` 是代理 IP，黑白名单会失效。要先配 `real-ip` 或 Nginx 的 `set_real_ip_from` 还原真实 IP。

## 6.2 CORS

```json
{
  "cors": {
    "allow_origins": "https://app.example.com,https://admin.example.com",
    "allow_methods": "GET,POST,PUT,DELETE,OPTIONS",
    "allow_headers": "Content-Type,Authorization,X-Request-Id",
    "expose_headers": "X-Request-Id",
    "max_age": 3600,
    "allow_credential": true
  }
}
```

| 字段 | 说明 |
|------|------|
| `allow_origins` | 允许的来源，多个用逗号分隔，`*` 表示全部 |
| `allow_methods` / `allow_headers` | 允许的方法 / 请求头 |
| `expose_headers` | 允许前端读取的响应头 |
| `allow_credential` | 是否允许携带 Cookie |
| `max_age` | 预检结果缓存秒数 |

★ **关键认知**：`allow_origins: "*"` 与 `allow_credential: true` 在浏览器规范下互斥。带 Cookie 的跨域必须返回具体 Origin。支持大量子域名时用 `allow_origins_by_regex`，不要图省事写 `*`。

⚠ **关键认知**：CORS 是浏览器行为，不是服务端安全边界。`curl` 和任意后端程序都不受限制，它只能防止浏览器页面读取响应。

## 6.3 请求体限制

```json
{
  "client-control": { "max_body_size": 1048576 }
}
```

`max_body_size` 单位是字节，0 表示不限制，超限返回 413。也可以在 `nginx_config.http.client_max_body_size` 里设全局默认值。

⚠ **关键认知**：只在网关限制请求体，不能阻止客户端继续发送剩余数据。大文件上传要同时配置 `client_body_buffer_size`、临时目录和上游超时。

# 7. 密钥管理

## 7.1 不写死在配置里

最危险的做法是把 secret、密码、私钥写进 Route 的 JSON，然后随导出、备份、截图一起扩散。APISIX 支持从环境变量和外部密钥服务读取：

```json
{
  "jwt-auth": {
    "key": "mobile-app",
    "secret": "$secret://vault/secret/data/apisix/jwt"
  }
}
```

环境变量用 `$ENV://变量名`，例如 `"key": "$ENV://ORDER_SERVICE_KEY"`；Vault 等 provider 要在 `config.yaml` 的 `secret` 段配置地址和 token。具体 URI 取决于 provider 的 KV 版本。原则只有一条：敏感值不进入 etcd 明文、不进入 Git、不进入日志。

## 7.2 轮换策略

| 凭据 | 轮换方式 | 注意 |
|------|----------|------|
| Key Auth | 新增 Consumer，切客户端，再删旧的 | 两个 key 并存期间权限要一致 |
| Basic Auth | 直接覆盖 Consumer 密码 | 客户端要同步更新 |
| JWT HS256 | 双 secret 并存期，或改 RS256 + JWKS | 单 Consumer 只有一个 secret |
| OIDC client secret | IdP 建新 secret，更新网关，再删旧的 | 确认 IdP 支持双 secret |
| Admin API key | 先加新 key 再删旧 key | 避免 CI 中断 |

★ **关键认知**：轮换的核心是"新旧并存 + 可回滚"，不是"找个夜里直接改"。一次性替换会留下无法回退的窗口。

# 8. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 401 `Missing API key` | 请求头名不对 | 对照 `key-auth.header` |
| 401 `Invalid API key` | Consumer 没建或 key 不匹配 | Admin API 读回 Consumer |
| Basic Auth 一直 401 | `curl -u` 拼错或密码含特殊字符 | 手动构造 Base64 验证 |
| JWT `signature mismatch` | secret 不一致 | 确认签发方与 Consumer 一致 |
| JWT `token expired` | `exp` 已过或时钟不同步 | 校时，必要时设宽限期 |
| JWT `key not found` | Payload 缺 `key` 或值不一致 | 补 `key` claim |
| OIDC 回调后无限跳转 | `redirect_uri` 不匹配、Cookie 域不对 | 对照 IdP 登记和浏览器 Cookie |
| 预检请求 401 | 路由没挂 `cors` | 同一路由挂 `cors` |
| 403 但 IP 在白名单 | 前面有代理，IP 不是真实客户端 | 配 `real-ip` |
| 413 | 请求体超过限制 | 调大 `max_body_size` |

# 9. 练习

1. 创建两个 Consumer，分别用 Key Auth 和 JWT 访问同一路由，验证 `X-Consumer-Username` 是否正确传给上游。
2. 给一条路由同时挂 `cors` 和 `key-auth`，用 `OPTIONS` 验证预检不需要 key，普通请求仍需要。
3. 用 `$ENV://` 引用 JWT secret，重启 APISIX 后验证配置有效，并观察 Admin API 读回时是否暴露明文。

# 10. 小结

| 主题 | 要点 |
|------|------|
| Consumer | 承载身份与凭据，Route 只负责开启认证方式 |
| Key Auth | 最简单，凭据优先放请求头 |
| Basic Auth | Base64 不是加密，必须配 HTTPS，只适合内网 |
| JWT | Payload 必须带 `key`，注意过期与时钟同步 |
| OIDC | 浏览器模式跳转建会话，API 模式用 `bearer_only` |
| IP 限制 | 依赖真实 IP，前面有代理要先还原 |
| CORS | 浏览器行为，`*` 与凭据互斥 |
| 密钥 | 用环境变量或 secret provider，轮换要新旧并存 |

下一篇进入限流与熔断，讲清固定窗口、漏桶、令牌桶的差异，`limit-req`、`limit-count`、`limit-conn` 的配置边界，以及 `api-breaker` 的状态机、重试预算和降级手段。
