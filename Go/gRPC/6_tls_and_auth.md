gRPC 默认跑在明文 HTTP/2 上，本地调试很方便，上线就是事故。更麻烦的是，很多人以为"加了 TLS 就安全了"，却把 token 明文塞进 metadata、把私钥写进代码、用 `InsecureSkipVerify` 跳过校验。本篇把传输层加密（TLS/mTLS）和应用层鉴权（Token/JWT）分开讲清楚。

# 1. 先分清两件事

## 1.1 传输安全与身份认证

| 层次 | 解决的问题 | 手段 |
|------|-----------|------|
| 传输层 | 链路是否加密、服务端是不是真的 | TLS、mTLS |
| 应用层 | 调用者是谁、有没有权限 | Token、JWT、API Key、RBAC |

```text
客户端
  │  ① TLS 握手：验证服务端证书，建立加密通道
  │  ② metadata 携带 token：证明"我是谁"
  ▼
服务端
  ├── TLS 层：确认对端身份（mTLS 时连客户端也验）
  └── 拦截器：校验 token、检查权限
```

★ **关键认知**：TLS 保证的是"这条管道是加密的、对端是可信的"，**不保证调用者有权限**。mTLS 解决了"服务间互相认证"，但用户级权限仍然要靠 token。两层都要有，不能互相替代。

## 1.2 gRPC 与 HTTP/2 的加密要求

gRPC 使用 HTTP/2。浏览器要求 HTTP/2 必须基于 TLS，但服务之间的 gRPC 允许 **h2c（HTTP/2 cleartext）**，也就是明文。Go 客户端用 `insecure.NewCredentials()` 就是走 h2c。

⚠ **关键认知**：`insecure.NewCredentials()` 只适合本地开发和受信任的内网调试。它会明文传输所有 metadata，包括 token。生产环境必须换成 TLS，或者至少用服务网格做链路加密。

# 2. 服务端 TLS

## 2.1 最小配置

```go
import "google.golang.org/grpc/credentials"

creds, err := credentials.NewServerTLSFromFile("certs/server.crt", "certs/server.key")
if err != nil {
    log.Fatalf("load server cert: %v", err)
}

s := grpc.NewServer(grpc.Creds(creds))
```

`NewServerTLSFromFile` 是最简写法，适合证书路径固定的场景。它内部会构造 `tls.Config`，但默认参数不一定符合安全要求。

## 2.2 显式构造 tls.Config

```go
func serverCreds(certFile, keyFile string) (credentials.TransportCredentials, error) {
    cert, err := tls.LoadX509KeyPair(certFile, keyFile)
    if err != nil {
        return nil, fmt.Errorf("load key pair: %w", err)
    }

    cfg := &tls.Config{
        Certificates: []tls.Certificate{cert},
        MinVersion:   tls.VersionTLS13, // 至少 TLS 1.2，新系统直接 1.3
        NextProtos:   []string{"h2"},   // 只协商 HTTP/2
    }
    return credentials.NewTLS(cfg), nil
}
```

★ **关键认知**：`NextProtos: []string{"h2"}` 是 gRPC over TLS 的关键。如果 ALPN 协商不到 `h2`，客户端可能退回 HTTP/1.1，gRPC 握手直接失败。用 `grpc.Creds` 时 gRPC 通常会帮你设置，但手写 `tls.Config` 时别漏。

## 2.3 证书生成

本地用 OpenSSL 造一套 CA + 服务端证书：

```bash
# 1. 自建 CA
openssl req -x509 -newkey rsa:4096 -nodes \
  -keyout ca.key -out ca.crt -days 3650 \
  -subj "/CN=Demo CA"

# 2. 服务端私钥与 CSR
openssl req -newkey rsa:4096 -nodes \
  -keyout server.key -out server.csr \
  -subj "/CN=localhost"

# 3. SAN 配置：现代客户端只认 SAN，不认 CN
cat > server.ext <<'EOF'
subjectAltName=DNS:localhost,IP:127.0.0.1
extendedKeyUsage=serverAuth
EOF

# 4. 用 CA 签发
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt -days 825 -extfile server.ext
```

⚠ **关键认知**：证书的 `CN` 字段早已被主流客户端忽略，**身份校验只看 SAN**。只写 `CN=localhost` 不写 `subjectAltName`，Go 客户端会报 `x509: certificate relies on legacy Common Name field`。这是自签证书最常见的坑。

# 3. 客户端 TLS

## 3.1 校验服务端

```go
func clientCreds(caFile, serverName string) (credentials.TransportCredentials, error) {
    caPEM, err := os.ReadFile(caFile)
    if err != nil {
        return nil, err
    }

    pool := x509.NewCertPool()
    if !pool.AppendCertsFromPEM(caPEM) {
        return nil, errors.New("invalid CA pem")
    }

    cfg := &tls.Config{
        RootCAs:    pool,
        ServerName: serverName, // 必须与证书 SAN 匹配
        MinVersion: tls.VersionTLS13,
    }
    return credentials.NewTLS(cfg), nil
}
```

```go
conn, err := grpc.NewClient("localhost:50051",
    grpc.WithTransportCredentials(creds),
)
```

★ **关键认知**：`ServerName` 是**校验目标**，不是"连接地址"。你连的是 `10.0.0.5:50051`，但证书 SAN 写的是 `user.internal`，就要把 `ServerName` 设成 `user.internal`。否则校验必然失败。

## 3.2 对比 `InsecureSkipVerify`

```go
// 错误示范：跳过所有校验，等于把 TLS 降级成"加密但不认证"
cfg := &tls.Config{InsecureSkipVerify: true}
```

⚠ **关键认知**：`InsecureSkipVerify: true` 只是不校验证书，中间人仍然可以解密并转发流量。它经常被用来"先把联调跑通"，然后忘了删。代码评审时应该把它当红线。

# 4. mTLS：双向认证

## 4.1 服务端要求客户端证书

```go
func mtlsServerCreds(caFile, certFile, keyFile string) (credentials.TransportCredentials, error) {
    cert, err := tls.LoadX509KeyPair(certFile, keyFile)
    if err != nil {
        return nil, err
    }

    caPEM, err := os.ReadFile(caFile)
    if err != nil {
        return nil, err
    }
    pool := x509.NewCertPool()
    if !pool.AppendCertsFromPEM(caPEM) {
        return nil, errors.New("invalid client CA")
    }

    cfg := &tls.Config{
        Certificates: []tls.Certificate{cert},
        ClientAuth:   tls.RequireAndVerifyClientCert, // 必须提供并校验客户端证书
        ClientCAs:    pool,
        MinVersion:   tls.VersionTLS13,
    }
    return credentials.NewTLS(cfg), nil
}
```

## 4.2 客户端出示证书

```go
func mtlsClientCreds(caFile, certFile, keyFile, serverName string) (credentials.TransportCredentials, error) {
    cert, err := tls.LoadX509KeyPair(certFile, keyFile)
    if err != nil {
        return nil, err
    }

    caPEM, _ := os.ReadFile(caFile)
    pool := x509.NewCertPool()
    pool.AppendCertsFromPEM(caPEM)

    cfg := &tls.Config{
        Certificates: []tls.Certificate{cert}, // 客户端身份
        RootCAs:      pool,
        ServerName:   serverName,
        MinVersion:   tls.VersionTLS13,
    }
    return credentials.NewTLS(cfg), nil
}
```

## 4.3 在业务里取客户端身份

TLS 握手完成后，客户端证书信息会挂在 ctx 上：

```go
import "google.golang.org/grpc/peer"

func whoAmI(ctx context.Context) string {
    p, ok := peer.FromContext(ctx)
    if !ok {
        return ""
    }

    tlsInfo, ok := p.AuthInfo.(credentials.TLSInfo)
    if !ok || len(tlsInfo.State.PeerCertificates) == 0 {
        return ""
    }

    cert := tlsInfo.State.PeerCertificates[0]
    return cert.Subject.CommonName // 或用 SAN / SPIFFE ID
}
```

★ **关键认知**：mTLS 适合**服务到服务**的认证，尤其是零信任网络。它比共享密钥强的地方在于：每个服务持有自己的私钥，泄露一个不影响其他服务，而且证书可以短期轮换。用户身份仍然用 JWT 这类 token 表达。

# 5. Token 与 JWT 鉴权

## 5.1 通过 metadata 传 token

```go
ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+token)
```

服务端在拦截器里取：

```go
md, _ := metadata.FromIncomingContext(ctx)
values := md.Get("authorization")
```

## 5.2 完整的 JWT 校验拦截器

```go
import (
    "github.com/golang-jwt/jwt/v5"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

type Claims struct {
    UserID string `json:"uid"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func JWTAuth(secret []byte) grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler) (any, error) {

        md, ok := metadata.FromIncomingContext(ctx)
        if !ok {
            return nil, status.Error(codes.Unauthenticated, "missing metadata")
        }

        raw := md.Get("authorization")
        if len(raw) == 0 {
            return nil, status.Error(codes.Unauthenticated, "missing token")
        }
        tokenStr := strings.TrimPrefix(raw[0], "Bearer ")

        claims := &Claims{}
        token, err := jwt.ParseWithClaims(tokenStr, claims,
            func(t *jwt.Token) (any, error) {
                // 必须校验签名算法，防止 alg=none 攻击
                if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
                    return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
                }
                return secret, nil
            },
            jwt.WithIssuer("demo-auth"),
            jwt.WithAudience("demo-api"),
            jwt.WithExpirationRequired(),
        )
        if err != nil || !token.Valid {
            return nil, status.Error(codes.Unauthenticated, "invalid token")
        }

        ctx = context.WithValue(ctx, claimsKey{}, claims)
        return handler(ctx, req)
    }
}
```

★ **关键认知**：JWT 校验有三件事缺一不可——**验签、验 `exp`、校验 `iss`/`aud`**。只验签不校验 `iss`，别的系统签发的 token 也能进来；不校验算法，攻击者可以把 `alg` 改成 `none` 或换成非对称算法绕过。

## 5.3 权限检查放在业务还是拦截器

```go
func (s *server) DeleteUser(ctx context.Context, req *pb.DeleteUserRequest) (*pb.DeleteUserReply, error) {
    claims, ok := ctx.Value(claimsKey{}).(*Claims)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "no claims")
    }

    if claims.Role != "admin" {
        return nil, status.Error(codes.PermissionDenied, "admin required")
    }

    return &pb.DeleteUserReply{}, s.repo.Delete(ctx, req.GetId())
}
```

| 放在哪 | 适合 | 不适合 |
|--------|------|--------|
| 拦截器 | 全局规则：必须登录、租户隔离 | 细粒度资源权限 |
| 业务方法 | 资源级判断：只能删自己的订单 | 重复的通用逻辑 |

⚠ **关键认知**：JWT 是**自包含**的，签发后无法立即吊销。用户被封禁、token 被盗，在过期前仍然有效。需要即时吊销时，要么用短过期时间 + refresh token，要么在服务端维护黑名单/版本号。

# 6. 密钥与证书管理

## 6.1 不要做的事

| 错误做法 | 风险 |
|----------|------|
| 私钥硬编码进代码 | 一旦仓库泄露，全部失守 |
| 私钥提交到 Git | 历史记录永久保留，无法真正删除 |
| 所有服务共用一个证书 | 泄露一个等于泄露全部 |
| 证书有效期 10 年 | 轮换困难，吊销形同虚设 |
| token 密钥写死在配置里 | 无法轮换，泄露后无法补救 |

## 6.2 推荐做法

- 私钥放密钥管理系统（Vault、KMS、云厂商 Secret Manager），启动时注入。
- 证书有效期缩短（如 90 天），自动化轮换（cert-manager、SPIFFE/SPIRE）。
- 每个服务独立证书，SAN 里写服务身份。
- 支持**双证书过渡**：轮换期间新旧证书并存，避免重启时中断。
- 用 `tls.Config.GetCertificate` 支持热加载：

```go
cfg := &tls.Config{
    GetCertificate: func(*tls.ClientHelloInfo) (*tls.Certificate, error) {
        return atomicLoadCert(), nil // 从内存中的原子变量读取，定期刷新
    },
    MinVersion: tls.VersionTLS13,
}
```

# 7. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| `certificate relies on legacy Common Name` | 证书没有 SAN | 重新签发，加 `subjectAltName` |
| `x509: certificate signed by unknown authority` | 客户端没配 CA | 加载正确 CA |
| `tls: bad certificate` | 客户端证书不被服务端信任 | 服务端 `ClientCAs` 配客户端 CA |
| `handshake failed: EOF` | 一方 TLS、一方明文 | 两边凭据配置一致 |
| `transport: Error while dialing` | 端口/证书/ALPN 问题 | 用 `openssl s_client` 验证 |
| `Unauthenticated` | token 没带/过期/iss 不匹配 | 检查 metadata 与 claims |
| 轮换后全挂 | 旧证书立即失效 | 双证书过渡 + 灰度 |

排查 TLS 的第一步永远是：

```bash
openssl s_client -connect localhost:50051 -servername localhost -CAfile ca.crt
```

能看到证书链和 `ALPN protocol: h2`，才说明 TLS 层没问题。

# 8. 练习与自检

1. 用上面的 OpenSSL 命令生成 CA 和服务端证书，跑通 TLS 版 SayHello。
2. 配置 mTLS，客户端不带证书时观察错误信息，再带证书重试。
3. 实现 JWT 拦截器，分别测试：无 token、过期 token、`iss` 不匹配、`alg` 被篡改。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 两层安全 | TLS 管链路，token 管身份，不能互相替代 |
| 服务端 TLS | 证书 + `MinVersion` + `h2` ALPN |
| 客户端 TLS | `RootCAs` + `ServerName`，禁用 `InsecureSkipVerify` |
| mTLS | `RequireAndVerifyClientCert` + `ClientCAs` |
| JWT | 验签、验 exp、验 iss/aud、锁定算法 |
| 密钥管理 | 不落代码、短期证书、自动轮换、双证过渡 |

下一篇进入网关与负载均衡，讲清 grpc-gateway 的 HTTP 映射、服务发现、客户端负载均衡、健康检查、keepalive 与代理兼容问题。
