
# 1. 安全体系概览

```
┌─────────────────────────────────────────────────┐
│                Dubbo 安全体系                      │
├──────────────┬──────────────┬───────────────────┤
│  传输安全     │  认证鉴权     │   访问控制          │
├──────────────┼──────────────┼───────────────────┤
│ TLS/SSL 加密  │ Token 令牌    │ IP 黑/白名单       │
│ mTLS 双向认证 │ 扩展认证机制   │ 路由规则控制         │
│ 证书管理      │ OAuth2 集成   │ 限流/降级           │
└──────────────┴──────────────┴───────────────────┘
```

# 2. TLS/SSL 传输加密

## 2.1 配置

```yaml
dubbo:
  ssl:
    enable: true
    server-key-cert-chain-path: /certs/server.crt
    server-private-key-path: /certs/server.key
    server-trust-cert-collection-path: /certs/ca.crt
    client-key-cert-chain-path: /certs/client.crt
    client-private-key-path: /certs/client.key
    client-trust-cert-collection-path: /certs/ca.crt
```

## 2.2 SSL 配置源码

```java
public class SslContexts {

    public static SSLContext buildServerSslContext(URL url) {
        try {
            SSLContext sslContext = SSLContext.getInstance("TLSv1.2");
            
            // 加载服务端证书
            KeyManager[] keyManagers = buildKeyManagers(
                url.getParameter("server-key-cert-chain-path"),
                url.getParameter("server-private-key-path")
            );
            
            // 加载信任证书（mTLS 验证客户端证书）
            TrustManager[] trustManagers = buildTrustManagers(
                url.getParameter("server-trust-cert-collection-path")
            );
            
            sslContext.init(keyManagers, trustManagers, new SecureRandom());
            return sslContext;
        } catch (Exception e) {
            throw new RpcException("Failed to build SSL context", e);
        }
    }
}
```

## 2.3 Netty SSL 集成

```java
public class NettyServer extends AbstractServer {

    @Override
    protected void doOpen() {
        // ... bootstrap 配置
        
        if (url.getParameter("ssl-enabled", false)) {
            SSLContext sslContext = SslContexts.buildServerSslContext(url);
            SSLEngine sslEngine = sslContext.createSSLEngine();
            sslEngine.setUseClientMode(false);
            sslEngine.setNeedClientAuth(true);  // mTLS
            
            // 将 SSL Handler 添加到 Pipeline
            ch.pipeline().addFirst("ssl", 
                new SslHandler(sslEngine));
        }
    }
}
```

# 3. Token 令牌验证

## 3.1 配置

```yaml
dubbo:
  provider:
    token: true    # 或自定义 token: "my-secret-token"
```

```java
@DubboService(token = "my-custom-token")
public class UserServiceImpl implements UserService {}
```

## 3.2 Consumer 端携带 Token

```java
@DubboReference(token = "my-custom-token")
private UserService userService;
```

或通过代码动态设置：

```java
RpcContext.getClientAttachment().setAttachment("token", "my-custom-token");
userService.getUser(1L);
```

## 3.3 TokenFilter 源码

```java
@Activate(group = PROVIDER, value = TOKEN_KEY)
public class TokenFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation inv) {
        String token = invoker.getUrl().getParameter(TOKEN_KEY);
        
        if (StringUtils.isNotEmpty(token)) {
            String remoteToken = inv.getAttachment(TOKEN_KEY);
            
            if (!token.equals(remoteToken)) {
                throw new RpcException(
                    "Invalid token! Forbid invoke remote service " +
                    invoker.getInterface() + " method " +
                    inv.getMethodName() + " from consumer " +
                    RpcContext.getServiceContext().getRemoteHost());
            }
        }
        
        return invoker.invoke(inv);
    }
}
```

# 4. 访问控制（IP 黑白名单）

## 4.1 通过路由规则实现

```yaml
# 白名单：只允许 192.168.1.* 网段访问
enabled: true
force: true           # 强制：不匹配时直接拒绝
rule: "host=192.168.1.*"

# 黑名单：禁止 10.0.1.100 访问
enabled: true
force: true
rule: "host!=10.0.1.100"
```

## 4.2 自定义访问控制 Filter

```java
@Activate(group = PROVIDER, order = -10000)
public class AccessControlFilter implements Filter {
    private final Set<String> whitelist = Set.of(
        "192.168.1.0/24", 
        "10.0.0.0/8"
    );
    private final Set<String> blacklist = new HashSet<>();

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) {
        String remoteIp = RpcContext.getServiceContext().getRemoteHost();
        
        // 1. 黑名单检查
        if (isInBlacklist(remoteIp)) {
            throw new RpcException("Access denied for IP: " + remoteIp);
        }
        
        // 2. 白名单检查
        if (!isInWhitelist(remoteIp)) {
            throw new RpcException("Access forbidden for IP: " + remoteIp);
        }
        
        return invoker.invoke(invocation);
    }

    private boolean isInWhitelist(String ip) {
        if (whitelist.isEmpty()) return true;
        return whitelist.stream().anyMatch(range -> ipInRange(ip, range));
    }
}
```

# 5. Spring Security 集成

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class DubboSecurityConfig {
    
    @Bean
    public Filter dubboAuthFilter() {
        return new SecureInvokeFilter();
    }
}

@Activate(group = PROVIDER)
public class SecureInvokeFilter implements Filter {
    
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) {
        // 从 attachments 获取认证信息
        String authToken = invocation.getAttachment("authorization");
        
        if (StringUtils.isEmpty(authToken)) {
            throw new RpcException("Authentication required");
        }
        
        // 验证 Token
        if (!JwtUtils.validate(authToken)) {
            throw new RpcException("Invalid authentication token");
        }
        
        // 权限检查
        String service = invoker.getInterface().getName();
        String method = invocation.getMethodName();
        String principal = JwtUtils.getPrincipal(authToken);
        
        if (!hasPermission(principal, service, method)) {
            throw new RpcException("Permission denied: " + principal);
        }
        
        return invoker.invoke(invocation);
    }
}
```

# 6. 序列化安全

## 6.1 问题

Java 反序列化漏洞是最常见的安全威胁之一。攻击者可能通过构造特殊的序列化数据执行任意代码。

## 6.2 Dubbo 安全措施

```yaml
dubbo:
  protocol:
    serialization-security:
      check-serializable: true
      trusted-classes:
        - com.example.api.*
        - java.util.*
        - java.lang.*
```

源码：

```java
public class SerializeSecurityManager {
    private final Set<String> allowedPrefixes = ConcurrentHashMap.newKeySet();
    private final Set<String> disallowedPrefixes = ConcurrentHashMap.newKeySet();
    
    {
        // 内置黑名单
        disallowedPrefixes.add("com.sun.");
        disallowedPrefixes.add("java.lang.Runtime");
        disallowedPrefixes.add("java.lang.ProcessBuilder");
        disallowedPrefixes.add("javax.management.");
    }

    public void checkSerialization(Class<?> clazz) {
        String name = clazz.getName();
        
        for (String prefix : disallowedPrefixes) {
            if (name.startsWith(prefix)) {
                throw new RpcException(
                    "Serialization of " + name + " is disallowed");
            }
        }
        
        if (!allowedPrefixes.isEmpty()) {
            boolean allowed = false;
            for (String prefix : allowedPrefixes) {
                if (name.startsWith(prefix)) {
                    allowed = true;
                    break;
                }
            }
            if (!allowed) {
                throw new RpcException(
                    "Serialization of " + name + " is not trusted");
            }
        }
    }
}
```

# 7. 注册中心安全

## 7.1 Nacos 认证

```yaml
dubbo:
  registry:
    address: nacos://127.0.0.1:8848?username=nacos&password=xxxxx
```

## 7.2 Zookeeper 认证

```yaml
dubbo:
  registry:
    address: zookeeper://127.0.0.1:2181?digest=user:pass
```

# 8. 推荐的安全实践

1. **内网部署**：Dubbo 服务部署在内网环境，不对外暴露
2. **启用 TLS**：跨机房/跨数据中心时启用 TLS 加密
3. **启用 Token**：所有 Dubbo 服务启用 Token 认证
4. **使用 Kryo 强制注册**：避免序列化未知类

```java
Kryo kryo = new Kryo();
kryo.setRegistrationRequired(true); // 强制注册
kryo.register(User.class);
```

5. **IP 白名单**：关键服务配置 IP 白名单路由
6. **最小权限**：只开放必要的接口和端口
7. **定期审计**：审计访问日志，检查异常调用

```bash
# 检查 Dubbo QOS 是否对外暴露
curl http://localhost:22222/ls

# 生产环境关闭 QOS 或设置复杂 Token
dubbo.application.qos-enable=false
```
