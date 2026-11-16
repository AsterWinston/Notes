gRPC 默认是明文传输。只要服务跨主机、跨网络，就必须上 TLS；如果还要确认"客户端是谁"，就得上 mTLS。本篇从证书准备开始，讲清 gRPC Java 的 TLS API、Spring Boot 的 SSL bundle、JWT 鉴权拦截器，以及握手失败时怎么定位。

# 1. TLS 解决什么问题

TLS 提供三件事：

- **机密性**：链路上的数据加密，抓包看不到明文。
- **完整性**：数据被篡改能被发现。
- **身份认证**：客户端可以确认服务端证书由受信任的 CA 签发。

gRPC 的四种调用都跑在 HTTP/2 上，TLS 在 HTTP/2 之下，对应用层透明。开启后，protobuf 消息、Metadata、Trailers 全部加密。

| 模式 | 服务端证书 | 客户端证书 | 能确认什么 |
|------|-----------|-----------|-----------|
| 明文 | 无 | 无 | 什么都不确认 |
| TLS（单向） | 有 | 无 | 客户端确认服务端身份 |
| mTLS（双向） | 有 | 有 | 双方互相确认身份 |

**设计考量**：单向 TLS 适合"客户端是匿名用户"的场景，例如浏览器访问公网 API。mTLS 适合服务网格、内部服务间调用，因为它把"谁在调用"绑定到证书身份，不依赖应用层 token。但 mTLS 的证书签发、轮换、吊销成本更高，通常由 Service Mesh 或内部 PKI 统一管理。

# 2. 准备证书

## 2.1 用 OpenSSL 生成测试证书

本地联调可以自建 CA。生产环境应使用公司 PKI 或公共 CA，不要用自签证书。

```bash
# 1. 生成 CA 私钥与自签根证书
openssl genrsa -out ca.key 2048
openssl req -x509 -new -key ca.key -days 3650 -out ca.crt \
  -subj "/CN=demo-ca"
# 2. 生成服务端私钥与 CSR
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr \
  -subj "/CN=localhost"
# 3. 用 CA 签发服务端证书，必须带 SAN
cat > server.ext <<'EOF'
subjectAltName=DNS:localhost,IP:127.0.0.1
EOF
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt -days 3650 -extfile server.ext
# 4. 生成客户端证书（mTLS 用）
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr \
  -subj "/CN=user-service"
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out client.crt -days 3650
```

几个硬性要求：

- 服务端证书的 SAN 必须包含客户端实际访问的域名或 IP。只写 CN 已经不被现代客户端接受。
- gRPC Java 的 `keyManager` 要求私钥是 **PKCS#8** 格式。如果看到 `BEGIN RSA PRIVATE KEY`，先转换：

```bash
openssl pkcs8 -topk8 -nocrypt -in server.key -out server-pkcs8.key
```

- 私钥权限设为仅属主可读，例如 `chmod 600 server.key`。

# 3. gRPC Java 的 TLS API

## 3.1 服务端

```java
import io.grpc.Grpc;
import io.grpc.Server;
import io.grpc.TlsServerCredentials;
import java.io.File;
public class TlsServer {
    public static void main(String[] args) throws Exception {
        TlsServerCredentials credentials = TlsServerCredentials.newBuilder()
                .keyManager(new File("certs/server.crt"),
                            new File("certs/server-pkcs8.key"))
                .trustManager(new File("certs/ca.crt")) // mTLS 校验客户端证书
                .clientAuth(TlsServerCredentials.ClientAuth.REQUIRE)
                .build();
        Server server = Grpc.newServerBuilderForPort(8443, credentials)
                .addService(new UserServiceImpl())
                .build()
                .start();
        System.out.println("TLS gRPC server started on 8443");
        server.awaitTermination();
    }
}
```

`clientAuth` 的三个取值：

| 值 | 行为 |
|----|------|
| `NONE` | 不要求客户端证书，等同单向 TLS |
| `OPTIONAL` | 客户端给证书就校验，不给也放行 |
| `REQUIRE` | 必须提供并通过校验，即 mTLS |

## 3.2 客户端

```java
import io.grpc.ChannelCredentials;
import io.grpc.Grpc;
import io.grpc.ManagedChannel;
import io.grpc.TlsChannelCredentials;
import java.io.File;
public class TlsClient {
    public static void main(String[] args) throws Exception {
        ChannelCredentials credentials = TlsChannelCredentials.newBuilder()
                .trustManager(new File("certs/ca.crt"))
                // mTLS 时提供客户端证书；单向 TLS 可去掉这两行
                .keyManager(new File("certs/client.crt"),
                            new File("certs/client-pkcs8.key"))
                .build();
        ManagedChannel channel = Grpc.newChannelBuilderForAddress(
                "localhost", 8443, credentials).build();
        try {
            UserServiceGrpc.UserServiceBlockingStub stub =
                    UserServiceGrpc.newBlockingStub(channel);
            System.out.println(stub.getUser(
                    GetUserRequest.newBuilder().setId(1L).build()));
        } finally {
            channel.shutdown();
        }
    }
}
```

**注意**：客户端 `trustManager` 传入的必须是签发服务端证书的 CA，而不是服务端证书本身。把 `server.crt` 当 trust store 用，在某些场景能"碰巧"成功，但证书轮换后会立刻失效。

# 4. Spring Boot 中的 TLS

## 4.1 服务端配置

Spring Boot 3.x 推荐使用 **SSL bundle** 统一管理证书。`application.yml`：

```yaml
spring:
  ssl:
    bundle:
      pem:
        grpc-server:
          keystore:
            certificate: classpath:certs/server.crt
            private-key: classpath:certs/server-pkcs8.key
          truststore:
            certificate: classpath:certs/ca.crt
  grpc:
    server:
      port: 8443
      ssl:
        bundle: grpc-server
        client-auth: require
```

`client-auth` 设为 `require` 即开启 mTLS。证书放在 `src/main/resources/certs/` 下，生产环境更推荐从挂载卷或密钥管理服务读取，而不是打进 jar。

## 4.2 客户端配置

```yaml
spring:
  ssl:
    bundle:
      pem:
        grpc-client:
          truststore:
            certificate: classpath:certs/ca.crt
          keystore:
            certificate: classpath:certs/client.crt
            private-key: classpath:certs/client-pkcs8.key
  grpc:
    client:
      channel:
        user:
          target: static://localhost:8443
          ssl:
            bundle: grpc-client
```

客户端 channel 名 `user` 与 `GrpcChannelFactory.createChannel("user")` 对应。单向 TLS 时去掉 `keystore` 即可。

## 4.3 JKS 与 PEM

企业环境常见 JKS/P12：

- `spring.ssl.bundle.jks.<name>` 配置 `keystore.location`、`keystore.password`、`truststore.location`。
- PEM 更适合容器和 Kubernetes Secret，文本格式便于挂载和审计。

两种方式不要混用同一个 bundle。迁移时先确认私钥格式和别名，否则会出现"证书加载成功但握手失败"。

# 5. Token 与 JWT 鉴权

## 5.1 为什么 TLS 之外还需要认证

TLS 解决"通道安全"，不解决"这个请求代表谁"。单向 TLS 下，任何能连上服务端的人都可以调用接口。因此还需要应用层认证：

- **Bearer Token / JWT**：客户端在 Metadata 里带 `authorization: Bearer <token>`。
- **mTLS**：证书身份即服务身份，适合服务间调用。
- **两者结合**：mTLS 确认服务身份，JWT 确认终端用户身份。

## 5.2 JWT 校验拦截器

先定义校验接口，隔离具体 JWT 库：

```java
public interface TokenVerifier {
    /**
     * 校验签名、过期时间、签发者，返回用户 id。
     * 校验失败抛出异常。
     */
    String verify(String token);
}
```

用 jjwt 0.12.x 实现 HMAC 校验：

```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
public class HmacTokenVerifier implements TokenVerifier {
    private final SecretKey key;
    public HmacTokenVerifier(String secret) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }
    @Override
    public String verify(String token) {
        Claims claims = Jwts.parser()
                .verifyWith(key)
                .requireIssuer("https://issuer.example.com")
                .build()
                .parseSignedClaims(token)
                .getPayload();
        // aud 校验根据业务需要开启
        return claims.getSubject();
    }
}
```

生产环境更常见的是 RS256/ES256：

- 用公钥验签，私钥只留在授权服务器。
- 通过 JWKS 端点获取公钥，并缓存。
- 校验 `iss`、`aud`、`exp`、`nbf`，允许少量时钟偏移。
- 不要把 `alg` 交给 token 自己决定，避免算法混淆攻击。

## 5.3 服务端拦截器

```java
import io.grpc.*;
public class JwtAuthInterceptor implements ServerInterceptor {
    public static final Context.Key<String> USER_ID = Context.key("user-id");
    private static final Metadata.Key<String> AUTHORIZATION =
            Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);
    private final TokenVerifier verifier;
    public JwtAuthInterceptor(TokenVerifier verifier) {
        this.verifier = verifier;
    }
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        String header = headers.get(AUTHORIZATION);
        if (header == null || !header.startsWith("Bearer ")) {
            call.close(Status.UNAUTHENTICATED
                    .withDescription("缺少 Bearer token"), new Metadata());
            return new ServerCall.Listener<>() {};
        }
        try {
            String userId = verifier.verify(header.substring(7));
            Context ctx = Context.current().withValue(USER_ID, userId);
            return Contexts.interceptCall(ctx, call, headers, next);
        } catch (Exception e) {
            call.close(Status.UNAUTHENTICATED
                    .withDescription("token 无效或已过期"), new Metadata());
            return new ServerCall.Listener<>() {};
        }
    }
}
```

业务层读取：

```java
String userId = JwtAuthInterceptor.USER_ID.get();
```

客户端注入 token 的拦截器在上一篇已经写过：在 `ClientCall.start` 里把 `authorization` 写入 Metadata。

安全细节：token 必须走 TLS，日志要脱敏，access token 短有效期，校验 `aud`，并用 `kid` 支持密钥轮换。
# 6. 密钥管理

| 做法 | 是否推荐 | 说明 |
|------|----------|------|
| 私钥提交到 Git | 禁止 | 一旦泄露无法补救 |
| 打进 jar | 不推荐 | 镜像泄露即私钥泄露 |
| 环境变量 | 一般 | 容易泄漏到进程列表和日志 |
| K8s Secret 挂载 | 推荐 | 注意 RBAC 与 etcd 加密 |
| Vault/KMS | 推荐 | 支持审计、轮换、短期凭据 |
| Service Mesh 自动 mTLS | 推荐 | 证书生命周期由控制面管理 |

**行业实践**：证书与私钥的轮换要提前演练。很多故障不是"没配 TLS"，而是"轮换时只更新了服务端，客户端还信任旧 CA"。双证书并存、灰度发布、监控握手失败率，是比一次性切换更安全的做法。

# 7. 常见握手错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `PKIX path building failed` | 客户端不信任服务端 CA | 把签发 CA 加入 trust store |
| `No subject alternative names present` | 证书缺少 SAN 或访问地址不匹配 | 重新签发带 SAN 的证书 |
| `UNAVAILABLE: io exception` | 明文客户端连 TLS 端口，或反之 | 两端 TLS 配置保持一致 |
| `bad_certificate` / `certificate_unknown` | mTLS 客户端证书不被服务端信任 | 服务端 trust store 加入客户端 CA |
| `SSLHandshakeException: Received fatal alert: handshake_failure` | 协议/套件不兼容，或客户端未提供证书 | 检查 TLS 版本、`clientAuth` |
| `HTTP/2 error code: PROTOCOL_ERROR` | 中间代理不支持 HTTP/2 或 ALPN 失败 | 检查网关与 ALPN 配置 |
| 证书过期 | 未监控有效期 | 加到期告警与自动轮换 |
| 只有部分实例握手失败 | 证书链不完整或实例配置不一致 | 对比各实例证书与配置 |

排查命令：

```bash
# 检查服务端证书链与 ALPN 是否协商到 h2
openssl s_client -connect localhost:8443 -servername localhost \
  -CAfile certs/ca.crt -alpn h2
# 校验证书是否由指定 CA 签发
openssl verify -CAfile certs/ca.crt certs/server.crt
# 用 grpcurl 验证 mTLS 调用
grpcurl -cacert certs/ca.crt \
  -cert certs/client.crt -key certs/client.key \
  localhost:8443 list
```

**注意**：`openssl s_client` 能握手成功，不代表 gRPC 一定可用。还要确认 ALPN 协商到了 `h2`，以及网关是否把 HTTP/2 帧正确转发。

# 8. 练习与自检

1. 用 OpenSSL 生成 CA、服务端证书和客户端证书，启动 mTLS 服务端，分别用带证书和不带证书的客户端连接，观察错误信息。
2. 把服务端证书的 SAN 改成 `example.com`，客户端仍访问 `localhost`，记录并解释报错。
3. 用 jjwt 实现一个校验 `iss` 与 `exp` 的 `TokenVerifier`，写一个不带 token 的客户端，确认返回 `UNAUTHENTICATED`。

# 9. 小结

| 主题 | 要点 |
|------|------|
| TLS | 提供加密、完整性、服务端身份认证 |
| mTLS | 双向证书认证，适合服务间调用 |
| Java API | `TlsServerCredentials` / `TlsChannelCredentials` |
| Spring | SSL bundle 统一管理 PEM/JKS |
| JWT | 校验签名、`exp`、`iss`、`aud`，必须走 TLS |
| 密钥 | 不进 Git、不打进镜像，支持轮换 |
| 排错 | 先看 CA 信任链，再看 SAN 和 ALPN |

下一篇进入性能优化，讲清消息大小、连接复用、压缩、线程池、背压、压测与监控这些影响吞吐和延迟的关键因素。
