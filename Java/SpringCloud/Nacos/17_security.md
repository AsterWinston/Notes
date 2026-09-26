
# 1. 安全体系概述

Nacos 从 1.2.0 版本开始引入了鉴权体系，通过多层次的安全机制保障服务和配置的安全。

```
            Nacos 安全体系架构

  ┌──────────────────────────────────────────────┐
  │            传输安全 (TLS/SSL)                 │
  │  ┌────────────────────────────────────────┐  │
  │  │         认证层 (Authentication)         │  │
  │  │  ┌──────────┐    ┌──────────────────┐  │  │
  │  │  │ Token    │    │  LDAP / OAuth2   │  │  │
  │  │  │ (默认)   │    │  (扩展)          │  │  │
  │  │  └──────────┘    └──────────────────┘  │  │
  │  └────────────────────────────────────────┘  │
  │  ┌────────────────────────────────────────┐  │
  │  │         授权层 (Authorization)           │  │
  │  │  ┌──────────┐    ┌──────────────────┐  │  │
  │  │  │ RBAC     │    │  HTTP 过滤器      │  │  │
  │  │  │ 角色授权  │    │  @Secured 注解   │  │  │
  │  │  └──────────┘    └──────────────────┘  │  │
  │  └────────────────────────────────────────┘  │
  │  ┌────────────────────────────────────────┐  │
  │  │         数据安全                        │  │
  │  │  ┌──────────┐    ┌──────────────────┐  │  │
  │  │  │ 配置加密  │    │  审计日志         │  │  │
  │  │  └──────────┘    └──────────────────┘  │  │
  │  └────────────────────────────────────────┘  │
  └──────────────────────────────────────────────┘
```

# 2. 鉴权开启配置

## 2.1 核心配置

```properties
# conf/application.properties

# 开启鉴权
nacos.core.auth.enabled=true

# 鉴权系统类型（nacos / ldap / custom）
nacos.core.auth.system.type=nacos

# Token Secret Key（Base64 编码，长度 >= 32 字符）
# ⚠️ 建议使用 32 位以上的随机字符串
nacos.core.auth.plugin.nacos.token.secret.key=VGhpc0lzTXlDdXN0b21TZWNyZXRLZXkwMTIzNDU2Nzg=

# Token 过期时间（秒），默认 18000（5小时）
nacos.core.auth.plugin.nacos.token.expire.seconds=18000

# 服务端身份标识
nacos.core.auth.server.identity.key=serverIdentity
nacos.core.auth.server.identity.value=security

# 是否开启 SQL 注入检查（默认 true）
nacos.core.auth.enable.userAgentAuthWhite=false

# 关闭 User-Agent 白名单（生产环境建议关闭）
nacos.core.auth.enable.userAgentAuthWhite=false
```

## 2.2 默认账号

开启鉴权后，默认会创建管理员账号：

| 用户名 | 密码 | 角色 |
|--------|------|------|
| nacos | nacos | ROLE_ADMIN (全局管理员) |

**生产环境务必修改默认密码**。

# 3. RBAC 权限模型

## 3.1 角色体系

```
┌─────────────────────────────────────────────┐
│               Nacos RBAC 模型                │
│                                              │
│  ROLE_ADMIN                                  │
│  ├── 拥有所有权限                            │
│  └── 可管理用户、角色、权限                   │
│                                              │
│  ROLE_USER (自定义角色)                      │
│  ├── Namespace-A: rw  (读写)                 │
│  ├── Namespace-B: r   (只读)                 │
│  └── Namespace-C: 无权限                     │
│                                              │
│  权限粒度：                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ 资源类型  │  │ 操作类型  │  │ 命名空间  │  │
│  │ config   │  │ r (读)   │  │ dev      │  │
│  │ naming   │  │ w (写)   │  │ prod     │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────┘
```

## 3.2 权限管理 API

```bash
# 管理员登录
curl -X POST 'http://127.0.0.1:8848/nacos/v1/auth/login' \
  -d 'username=nacos&password=nacos'

# 创建用户
curl -X POST 'http://127.0.0.1:8848/nacos/v1/auth/users' \
  -d 'username=dev-user&password=dev123'

# 创建角色
curl -X POST 'http://127.0.0.1:8848/nacos/v1/auth/roles' \
  -d 'username=dev-user&role=dev-role'

# 授予权限
curl -X POST 'http://127.0.0.1:8848/nacos/v1/auth/permissions' \
  -d 'role=dev-role&resource=dev:*:*&action=rw'

# 权限格式: namespaceId:group:resource
# 操作:
#   r  - 读
#   w  - 写
#   rw - 读写
#   '' - 无权限
```

## 3.3 权限配置示例

```bash
# 只读用户（只能查看 dev 命名空间的配置）
curl -X POST 'http://127.0.0.1:8848/nacos/v1/auth/permissions' \
  -d 'role=viewer&resource=dev:*:*&action=r'

# 开发者（dev 命名空间读写权限）
curl -X POST 'http://127.0.0.1:8848/nacos/v1/auth/permissions' \
  -d 'role=developer&resource=dev:*:*&action=rw'

# 运维者（所有命名空间读权限，prod 命名空间写权限）
curl -X POST 'http://127.0.0.1:8848/nacos/v1/auth/permissions' \
  -d 'role=operator&resource=*:*:*&action=r'
curl -X POST 'http://127.0.0.1:8848/nacos/v1/auth/permissions' \
  -d 'role=operator&resource=prod:*:*&action=rw'
```

# 4. Token 认证机制

## 4.1 Token 生成流程

```java
// JwtTokenManager.java
@Component
public class JwtTokenManager {
    
    @Value("${nacos.core.auth.plugin.nacos.token.secret.key}")
    private String secretKey;
    
    @Value("${nacos.core.auth.plugin.nacos.token.expire.seconds:18000}")
    private long tokenValidityInSeconds;
    
    // 创建 Token
    public String createToken(String userName) {
        JwtBuilder builder = Jwts.builder()
            .setHeaderParam("alg", "HS256")
            .setHeaderParam("typ", "JWT")
            .setSubject(userName)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 
                tokenValidityInSeconds * 1000))
            .signWith(SignatureAlgorithm.HS256, 
                new SecretKeySpec(Base64.getDecoder().decode(secretKey), "HmacSHA256"));
        
        return builder.compact();
    }
    
    // 解析 Token
    public Authentication parseToken(String token) {
        try {
            Claims claims = Jwts.parser()
                .setSigningKey(Base64.getDecoder().decode(secretKey))
                .parseClaimsJws(token)
                .getBody();
            
            String userName = claims.getSubject();
            return new NacosAuthenticationToken(userName);
        } catch (ExpiredJwtException e) {
            throw new AccessException("Token expired");
        } catch (Exception e) {
            throw new AccessException("Token invalid");
        }
    }
}
```

## 4.2 客户端 Token 使用

```java
// 服务端请求携带 Token
// 1. Java SDK 配置
Properties properties = new Properties();
properties.setProperty(PropertyKeyConst.SERVER_ADDR, "127.0.0.1:8848");
properties.setProperty(PropertyKeyConst.USERNAME, "nacos");
properties.setProperty(PropertyKeyConst.PASSWORD, "nacos");
// SDK 自动管理 Token 的获取和刷新

ConfigService configService = NacosFactory.createConfigService(properties);

// 2. HTTP 请求携带 Token
curl -X GET 'http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=test.yaml&group=DEFAULT_GROUP&accessToken=eyJhbGci...'
```

## 4.3 Token 刷新机制

```java
// SecurityProxy.java (Client 端)
public class SecurityProxy {
    
    private String accessToken;
    private long tokenTtl;
    private long lastRefreshTime;
    
    // 确保 Token 有效
    public String getAccessToken() {
        if (needRefresh()) {
            login();
        }
        return accessToken;
    }
    
    private boolean needRefresh() {
        // 提前 10s 刷新 Token
        return System.currentTimeMillis() - lastRefreshTime > (tokenTtl - 10000);
    }
    
    private void login() {
        // POST /nacos/v1/auth/login
        // 获取新 Token
        Map<String, String> result = client.login(username, password);
        this.accessToken = result.get("accessToken");
        this.tokenTtl = Long.parseLong(result.get("tokenTtl")) * 1000;
        this.lastRefreshTime = System.currentTimeMillis();
    }
}
```

# 5. 服务端认证流程

## 5.1 AuthFilter 过滤器链

```java
// AuthFilter.java
@Component
public class AuthFilter implements Filter {
    
    @Autowired
    private AuthConfigs authConfigs;
    
    @Autowired
    private AuthManager authManager;
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                          FilterChain chain) {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        // 1. 检查是否需要鉴权
        if (!authConfigs.isAuthEnabled()) {
            chain.doFilter(request, response);
            return;
        }
        
        // 2. 检查是否为请求白名单
        if (isWhiteList(httpRequest.getRequestURI())) {
            chain.doFilter(request, response);
            return;
        }
        
        // 3. 提取 Token
        String token = extractToken(httpRequest);
        
        // 4. 验证 Token
        try {
            Authentication auth = authManager.authenticate(token);
            SecurityContextHolder.setAuthentication(auth);
        } catch (AccessException e) {
            sendUnauthorizedResponse(response, e.getMessage());
            return;
        }
        
        // 5. 权限检查
        if (needAuthorization(httpRequest)) {
            Resource resource = parseResource(httpRequest);
            ActionType action = parseAction(httpRequest);
            
            if (!authManager.authorize(
                    SecurityContextHolder.getAuthentication(), 
                    resource, action)) {
                sendForbiddenResponse(response, "Permission denied");
                return;
            }
        }
        
        chain.doFilter(request, response);
    }
}
```

## 5.2 @Secured 注解

```java
// @Secured 注解定义
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Secured {
    ActionTypes action() default ActionTypes.READ;
    String resource() default "";
    String[] tags() default {};
}

// 在 Controller 中使用
@RestController
@RequestMapping("/nacos/v1/ns/instance")
public class InstanceController {
    
    @PostMapping
    @Secured(action = ActionTypes.WRITE)
    public String register(HttpServletRequest request) {
        // 注册实例操作需要 WRITE 权限
        // ...
    }
    
    @GetMapping("/list")
    @Secured(action = ActionTypes.READ)
    public ObjectNode list(HttpServletRequest request) {
        // 查询实例操作只需要 READ 权限
        // ...
    }
}
```

# 6. 配置加密

## 6.1 Nacos 原生加密插件（2.0+）

Nacos 2.x 支持配置内容加密存储：

```properties
# 开启配置加密
nacos.config.encrypt.enabled=true
nacos.config.encrypt.data-key=your-secret-encryption-key
```

## 6.2 使用 AES 加密敏感配置

```java
// 自定义加密处理器
@Component
public class AesEncryptionHandler {
    
    private static final String AES_KEY = "your-aes-key-256";  // 256 bit key
    private static final String TRANSFORMATION = "AES/GCM/NoPadding";
    
    public String encrypt(String plainText) throws Exception {
        SecretKeySpec keySpec = new SecretKeySpec(AES_KEY.getBytes(), "AES");
        Cipher cipher = Cipher.getInstance(TRANSFORMATION);
        cipher.init(Cipher.ENCRYPT_MODE, keySpec);
        
        byte[] encrypted = cipher.doFinal(plainText.getBytes());
        byte[] iv = cipher.getIV();
        
        // 返回 Base64(iv + encrypted)
        ByteBuffer byteBuffer = ByteBuffer.allocate(iv.length + encrypted.length);
        byteBuffer.put(iv);
        byteBuffer.put(encrypted);
        return Base64.getEncoder().encodeToString(byteBuffer.array());
    }
    
    public String decrypt(String cipherText) throws Exception {
        byte[] decoded = Base64.getDecoder().decode(cipherText);
        
        // 提取 IV
        byte[] iv = Arrays.copyOfRange(decoded, 0, 12);
        byte[] encrypted = Arrays.copyOfRange(decoded, 12, decoded.length);
        
        SecretKeySpec keySpec = new SecretKeySpec(AES_KEY.getBytes(), "AES");
        Cipher cipher = Cipher.getInstance(TRANSFORMATION);
        cipher.init(Cipher.DECRYPT_MODE, keySpec, new GCMParameterSpec(128, iv));
        
        return new String(cipher.doFinal(encrypted));
    }
}
```

# 7. 服务间安全通信

## 7.1 服务端身份认证

```properties
# 服务端身份标识（防止伪造成其他 Server）
nacos.core.auth.server.identity.key=serverIdentity
nacos.core.auth.server.identity.value=your-custom-secret

# 所有服务端节点必须使用相同的 identity value
# 客户端无需配置此项
```

## 7.2 Spring Cloud 客户端配置

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        username: nacos
        password: nacos
      config:
        server-addr: 127.0.0.1:8848
        username: nacos
        password: nacos
```

# 8. 安全最佳实践

## 8.1 生产环境安全清单

| 安全项 | 建议 | 优先级 |
|--------|------|--------|
| 修改默认密码 | 修改 nacos 默认密码 | 紧急 |
| 开启鉴权 | `nacos.core.auth.enabled=true` | 高 |
| 自定义 Token 密钥 | 生成 32+ 字符随机字符串并 Base64 编码 | 高 |
| 关闭 UA 白名单 | `enable.userAgentAuthWhite=false` | 高 |
| 配置加密 | 敏感配置加密存储 | 中 |
| 网络隔离 | 防火墙/Nginx 限制访问来源 IP | 高 |
| TLS 通信 | 启用 HTTPS（Nginx 反向代理） | 中 |
| 权限最小化 | 按角色分配最小权限 | 中 |
| 审计日志 | 记录所有配置变更操作 | 中 |
| 定期轮换密钥 | 定期更新 Token Secret Key | 低 |

## 8.2 密码安全建议

```bash
# 生成安全的 Token Secret Key
# Linux
openssl rand -base64 32

# 输出示例: VGhpc0lzTXlDdXN0b21TZWNyZXRLZXkwMTIzNDU2Nzg=

# 在 application.properties 中使用
nacos.core.auth.plugin.nacos.token.secret.key=VGhpc0lzTXlDdXN0b21TZWNyZXRLZXkwMTIzNDU2Nzg=
```

## 8.3 网络安全建议

```nginx
# Nginx 仅暴露必要的 Location
location /nacos/ {
    # IP 白名单
    allow 10.0.0.0/8;
    allow 172.16.0.0/12;
    deny all;
    
    proxy_pass http://nacos-cluster;
}

# 不对外暴露 gRPC 端口（仅内部服务访问）
# 防火墙规则:
# - 8848: 仅允许内部 IP 段
# - 7848/9848/9849: 仅允许 Nacos 集群节点间通信
```
