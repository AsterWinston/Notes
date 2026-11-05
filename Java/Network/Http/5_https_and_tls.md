HTTPS 不是"加个 s"这么简单，它背后是证书、密钥交换和信任链。本篇从明文传输的风险讲起，拆解 TLS 握手、证书校验和 Java 中的 keystore/truststore，并给出证书过期、自签名等常见问题的排查方法。

# 1. 为什么需要 HTTPS

## 1.1 明文传输的风险

HTTP 报文在网络上明文传输，可能遭遇三类攻击：

| 风险 | 表现 | 后果 |
|------|------|------|
| 窃听 | 抓包看到请求头、Cookie、密码 | 账号泄露、数据泄露 |
| 篡改 | 中间人修改响应内容 | 注入广告、篡改转账账号 |
| 冒充 | 攻击者伪装成目标网站 | 钓鱼、凭据窃取 |

HTTPS 在 HTTP 和 TCP 之间加入 TLS 层，解决上述问题。

## 1.2 机密性、完整性与身份认证

HTTPS 同时提供三个安全目标：

- **机密性**：传输内容加密，窃听者看不到明文。
- **完整性**：内容被篡改可以被发现。
- **身份认证**：客户端确认服务端持有对应证书的私钥。

TLS 同时使用对称加密和非对称加密：

```text
非对称加密：用于握手阶段协商密钥、验证身份
对称加密：用于握手后的数据传输
```

原因是对称加密速度快，适合大量数据；非对称加密适合少量密钥交换，但性能较低。

# 2. TLS 握手

## 2.1 证书与 CA

证书包含：

- 域名（Subject Alternative Name）。
- 公钥。
- 有效期。
- 签发机构（CA）。
- CA 的签名。

信任链：

```text
根 CA（浏览器/操作系统内置信任）
  └── 中间 CA
        └── 服务器证书（example.com）
```

浏览器验证证书时，会从服务器证书一路向上验证签名，直到找到一个受信任的根 CA。任何一环失败，都会提示证书不受信任。

★ **关键认知**：证书解决的是"这个公钥确实属于 example.com"，而不是"这个网站是好人"。证书只能证明身份，不能保证网站没有恶意。

## 2.2 TLS 1.2 握手流程

简化流程：

```text
客户端                                     服务端
  │                                          │
  │ ─────── ClientHello ───────────────────> │  支持的版本、密码套件、随机数
  │                                          │
  │ <────── ServerHello ──────────────────── │  选定版本、密码套件、随机数
  │ <────── Certificate ──────────────────── │  服务器证书链
  │ <────── ServerKeyExchange ────────────── │  密钥交换参数
  │ <────── ServerHelloDone ──────────────── │
  │                                          │
  │ ─────── ClientKeyExchange ─────────────> │  客户端生成预主密钥
  │ ─────── ChangeCipherSpec ──────────────> │
  │ ─────── Finished ──────────────────────> │
  │                                          │
  │ <────── ChangeCipherSpec ─────────────── │
  │ <────── Finished ─────────────────────── │
  │                                          │
  │            加密通信开始                   │
```

TLS 1.2 通常需要 2 个往返，TLS 1.3 优化到 1 个往返，恢复连接时甚至支持 0-RTT。

## 2.3 TLS 1.3 优化

TLS 1.3 的主要改进：

- 握手减少到 1-RTT，会话恢复支持 0-RTT。
- 移除不安全算法，例如 RC4、3DES、SHA-1。
- 只保留前向安全的密钥交换方式。
- 加密更多握手字段，减少信息泄露。

⚠ 0-RTT 有重放风险，不应让非幂等请求使用 0-RTT 数据。

# 3. Java 中的 TLS

## 3.1 keystore 与 truststore

Java 使用两个密钥库：

| 名称 | 存什么 | 用途 |
|------|--------|------|
| keystore | 自己的私钥和证书 | 服务端证明身份，客户端双向认证 |
| truststore | 信任的 CA 证书 | 校验对端证书 |

常用 `keytool` 命令：

```bash
# 查看 keystore 内容
keytool -list -v -keystore server.p12 -storetype PKCS12

# 导出证书
keytool -exportcert -alias server -keystore server.p12 -file server.crt

# 导入 CA 证书到 truststore
keytool -importcert -alias my-ca -file ca.crt -keystore truststore.p12 -storetype PKCS12
```

## 3.2 HttpClient 配置

默认情况下，Java HttpClient 使用 JVM 默认的 truststore。需要自定义信任链时，可以构造 `SSLContext`：

```java
KeyStore trustStore = KeyStore.getInstance("PKCS12");
try (InputStream in = Files.newInputStream(Path.of("truststore.p12"))) {
    trustStore.load(in, "changeit".toCharArray());
}

TrustManagerFactory tmf = TrustManagerFactory.getInstance(
    TrustManagerFactory.getDefaultAlgorithm()
);
tmf.init(trustStore);

SSLContext sslContext = SSLContext.getInstance("TLS");
sslContext.init(null, tmf.getTrustManagers(), null);

HttpClient client = HttpClient.newBuilder()
    .sslContext(sslContext)
    .build();
```

⚠ 不要在生产环境使用"信任所有证书"的 `TrustManager`。它会让 HTTPS 失去身份认证能力，等于把连接暴露给中间人。

# 4. 常见问题

## 4.1 证书过期与域名不匹配

常见报错：

```text
javax.net.ssl.SSLHandshakeException: PKIX path validation failed:
java.security.cert.CertPathValidatorException: validity check failed
```

或：

```text
SSLHandshakeException: No subject alternative names matching IP address ...
```

排查步骤：

```bash
# 查看服务端证书有效期和域名
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -dates -subject -ext subjectAltName

# 检查系统时间
date
```

原因通常有三类：

1. 证书确实过期，需要续期。
2. 客户端系统时间不正确，导致证书被判定为未生效或已过期。
3. 访问的域名/IP 与证书 SAN 不匹配。

## 4.2 自签名证书

开发环境可以使用自签名证书，但需要让客户端显式信任：

```bash
# 生成自签名证书（仅用于本地开发）
keytool -genkeypair -alias local -keyalg RSA -keysize 2048 \
  -validity 365 -keystore local.p12 -storetype PKCS12 \
  -dname "CN=localhost" -ext "SAN=dns:localhost,ip:127.0.0.1"
```

生产环境不应关闭证书校验，也不应把自签名证书直接分发给所有客户端。正确做法是使用受信任 CA 签发的证书，并做好自动续期。

# 5. 安全实践

| 实践 | 说明 |
|------|------|
| 使用 TLS 1.2+ | 禁用 TLS 1.0/1.1 和不安全密码套件 |
| 证书自动续期 | 避免因过期导致服务中断 |
| HSTS | 告诉浏览器后续强制使用 HTTPS |
| 双向认证 | 高安全场景校验客户端证书 |
| 密钥保护 | 私钥文件权限最小化，使用密钥管理服务 |
| 定期轮换 | 证书和密钥按策略轮换 |
| 监控到期时间 | 提前告警，不要等故障发生 |

# 6. 小结

| 主题 | 要点 |
|------|------|
| HTTPS 目标 | 机密性、完整性、身份认证 |
| 证书 | 由 CA 签发，通过信任链验证 |
| 握手 | TLS 1.2 两个往返，TLS 1.3 一个往返 |
| Java | keystore 存自己的密钥，truststore 存信任的 CA |
| 常见问题 | 过期、域名不匹配、系统时间错误、自签名未信任 |
| 安全底线 | 不关闭证书校验，不在生产使用自签名证书 |

下一篇进入缓存与 Cookie，讲清强缓存、协商缓存、Cookie 安全属性和分布式 Session。
