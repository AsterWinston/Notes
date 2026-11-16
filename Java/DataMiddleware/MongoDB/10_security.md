
# 1. MongoDB安全体系总览

MongoDB的安全机制是一个多层次防护体系，包含以下层面：

| 层级 | 机制 | 说明 |
|------|------|------|
| 网络层 | TLS/SSL加密、绑定IP、防火墙 | 传输加密与网络隔离 |
| 认证层 | SCRAM、X.509、LDAP、Kerberos | 身份验证 |
| 授权层 | RBAC（基于角色的访问控制） | 权限管控 |
| 审计层 | Audit Log | 操作审计 |
| 数据层 | 客户端字段级加密、静态加密 | 数据加密 |
| 应用层 | 输入校验、Schema Validation | 防止注入 |

# 2. 认证（Authentication）

## 2.1 开启认证

默认安装的MongoDB没有开启认证，生产环境必须开启。

```yaml
# mongod.conf
security:
  authorization: enabled
```

或在启动时指定：

```bash
mongod --auth --port 27017 --dbpath /data/db
```

## 2.2 认证机制

| 机制 | 适用场景 | 安全级别 |
|------|---------|---------|
| SCRAM-SHA-256 | 用户名密码认证（默认） | 中 |
| SCRAM-SHA-1 | 旧版本兼容 | 中低 |
| X.509证书 | 服务间认证、高安全要求 | 高 |
| LDAP | 企业统一认证 | 高 |
| Kerberos | Windows域环境 | 高 |

## 2.3 创建用户

```javascript
// 切换到admin数据库（认证用户必须在admin库创建）
use admin

// 创建超级管理员
db.createUser({
    user: "admin",
    pwd: "SecurePassword123!",
    roles: [
        { role: "root", db: "admin" }
    ]
})

// 创建数据库管理员
db.createUser({
    user: "dbAdmin",
    pwd: "AdminPass456!",
    roles: [
        { role: "dbAdmin", db: "mydb" },
        { role: "readWrite", db: "mydb" }
    ]
})

// 创建只读用户
db.createUser({
    user: "readOnlyUser",
    pwd: "ReadPass789!",
    roles: [
        { role: "read", db: "mydb" }
    ]
})

// 创建集群管理用户
db.createUser({
    user: "clusterAdmin",
    pwd: "ClusterPass!",
    roles: [
        { role: "clusterAdmin", db: "admin" },
        { role: "readAnyDatabase", db: "admin" }
    ]
})
```

## 2.4 用户管理操作

```javascript
// 认证登录
db.auth("admin", "password")

// 查看当前用户
db.runCommand({ connectionStatus: 1 })

// 查看所有用户
db.getUsers()
show users

// 修改用户密码
db.changeUserPassword("readOnlyUser", "NewPassword!")

// 更新用户角色
db.updateUser("readOnlyUser", {
    roles: [
        { role: "readWrite", db: "mydb" }
    ]
})

// 删除用户
db.dropUser("readOnlyUser")

// 删除所有用户
db.dropAllUsers()
```

# 3. 授权（Authorization）

## 3.1 内置角色

MongoDB提供了丰富的内置角色，按权限范围分为：

**数据库用户角色**：

| 角色 | 权限 |
|------|------|
| read | 读取指定数据库中所有非系统集合的数据 |
| readWrite | read权限 + 写入指定数据库中所有非系统集合的数据 |

**数据库管理角色**：

| 角色 | 权限 |
|------|------|
| dbAdmin | 执行管理任务（索引管理、统计信息等） |
| userAdmin | 创建和管理用户 |
| dbOwner | 数据库所有者（readWrite + dbAdmin + userAdmin） |

**集群管理角色**：

| 角色 | 权限 |
|------|------|
| clusterManager | 管理和监控集群 |
| clusterMonitor | 只读监控集群 |
| hostManager | 管理服务器 |
| clusterAdmin | 最高集群管理权限 |

**备份恢复角色**：

| 角色 | 权限 |
|------|------|
| backup | 备份数据 |
| restore | 恢复数据 |

**超级角色**：

| 角色 | 权限 |
|------|------|
| root | 超级管理员，所有权限 |
| __system | 超级特权，类似root但有更多内部权限 |

**全数据库角色**（作用于所有数据库，只在admin库授予）：

| 角色 | 权限 |
|------|------|
| readAnyDatabase | 读取所有数据库 |
| readWriteAnyDatabase | 读写所有数据库 |
| userAdminAnyDatabase | 管理所有数据库用户 |
| dbAdminAnyDatabase | 管理所有数据库 |

## 3.2 自定义角色

```javascript
// 创建自定义角色
db.createRole({
    role: "orderManager",
    privileges: [
        {
            resource: { db: "shop", collection: "orders" },
            actions: ["find", "insert", "update", "remove"]
        },
        {
            resource: { db: "shop", collection: "" },
            actions: ["find"]
        }
    ],
    roles: []   // 继承其他角色
})

// 查看角色
db.getRole("orderManager")

// 查看角色权限
db.getRole("orderManager", { showPrivileges: true })

// 删除角色
db.dropRole("orderManager")
```

## 3.3 权限动作参考

常用actions：

| action | 说明 |
|--------|------|
| find | 查询 |
| insert | 插入 |
| update | 更新 |
| remove | 删除 |
| createCollection | 创建集合 |
| createIndex | 创建索引 |
| dropCollection | 删除集合 |
| dropIndex | 删除索引 |
| collStats | 查看集合统计 |
| findIndex | 查看索引 |
| killCursors | 关闭游标 |
| planCacheWrite | 修改计划缓存 |

## 3.4 最小权限原则

```javascript
// 实践：按实际需求创建精确权限的角色

// 微服务订单服务的数据库用户
db.createUser({
    user: "orderService",
    pwd: "ServicePass123!",
    roles: [
        {
            role: "orderServiceRole",
            db: "orderdb"
        }
    ]
})

// 相应的自定义角色
db.createRole({
    role: "orderServiceRole",
    privileges: [
        { resource: { db: "orderdb", collection: "orders" },
          actions: ["find", "insert", "update"] },
        { resource: { db: "orderdb", collection: "orderCounters" },
          actions: ["find", "update"] }
    ],
    roles: []
})

// 报表系统的只读用户
db.createUser({
    user: "reportService",
    pwd: "ReportPass!",
    roles: [
        { role: "read", db: "orderdb" },
        { role: "read", db: "userdb" }
    ]
})
```

# 4. TLS/SSL传输加密

## 4.1 生成证书

```bash
# 生成自签名CA证书
openssl req -new -x509 -days 3650 -nodes \
    -out ca.pem -keyout ca.key \
    -subj "/CN=MongoDB-CA"

# 生成服务器证书
openssl req -new -nodes \
    -out server.csr -keyout server.key \
    -subj "/CN=mongo-server"

# CA签名
openssl x509 -req -days 3650 \
    -in server.csr -out server.pem \
    -CA ca.pem -CAkey ca.key -CAcreateserial

# 合并为PEM格式
cat server.key server.pem > mongo-server.pem
```

## 4.2 配置TLS

```yaml
# mongod.conf - TLS配置
net:
  tls:
    mode: requireTLS                # 强制TLS
    certificateKeyFile: /etc/ssl/mongo-server.pem
    CAFile: /etc/ssl/ca.pem
    allowConnectionsWithoutCertificates: true  # 客户端TLS可选
    allowInvalidHostnames: false
```

```yaml
# 客户端连接（副本集 + TLS）
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongo-server.pem
    CAFile: /etc/ssl/ca.pem
    clusterFile: /etc/ssl/mongo-cluster.pem    # 内部通信证书
    clusterCAFile: /etc/ssl/ca.pem
```

## 4.3 客户端TLS连接

```bash
# mongosh TLS连接
mongosh --tls --tlsCAFile ca.pem \
    --host mongo-server \
    "mongodb://user:pass@mongo-server:27017/admin?tls=true&tlsCAFile=ca.pem"

# Java连接
MongoClientSettings settings = MongoClientSettings.builder()
    .applyToSslSettings(builder ->
        builder.enabled(true)
               .context(sslContext))  // 配置好的SSLContext
    .build();
```

# 5. 副本集认证

副本集节点之间通过**密钥文件（KeyFile）**或X.509证书进行内部认证。

## 5.1 密钥文件认证

```bash
# 生成密钥文件（内容为随机字符串）
openssl rand -base64 756 > /etc/mongod-keyfile

# 权限设置（必须为400或600）
chmod 600 /etc/mongod-keyfile
```

```yaml
# mongod.conf
security:
  keyFile: /etc/mongod-keyfile
  authorization: enabled
```

密钥文件启用后自动开启内部认证，同时外部认证机制也变为SCRAM-SHA-256。

## 5.2 X.509内部认证

```yaml
security:
  clusterAuthMode: x509
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongo-server.pem
    CAFile: /etc/ssl/ca.pem
    clusterFile: /etc/ssl/mongo-cluster.pem
```

# 6. 审计日志（Audit Log）

审计日志记录所有数据库操作，企业版功能。

```yaml
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.log
  filter: '{ atype: { $in: ["authenticate", "dropDatabase", "createUser", "dropUser"] } }'
```

审计事件类型：

| atype | 说明 |
|-------|------|
| authenticate | 认证事件 |
| createUser / dropUser | 用户管理 |
| createRole / dropRole | 角色管理 |
| createCollection / dropCollection | 集合操作 |
| createDatabase / dropDatabase | 数据库操作 |
| createIndex / dropIndex | 索引操作 |
| authCheck | 权限检查 |

# 7. 客户端字段级加密（FLE / QE）

MongoDB 4.2引入字段级加密，6.0引入可查询加密：

```javascript
// 创建加密集合（使用MongoDB驱动配置）
// 在应用层加密敏感字段，数据库服务器只看到密文

// 自动加密配置示例（Java Driver）
Map<String, Object> kmsProviders = new HashMap<>();
kmsProviders.put("local", new HashMap<String, Object>() {{
    put("key", localMasterKey);   // 本地主密钥
}});

Map<String, BsonDocument> schemaMap = new HashMap<>();
// schemaMap定义哪些字段需要加密...

AutoEncryptionSettings autoEncryptionSettings = AutoEncryptionSettings.builder()
    .keyVaultNamespace("encryption.__keyVault")
    .kmsProviders(kmsProviders)
    .schemaMap(schemaMap)
    .build();

MongoClientSettings settings = MongoClientSettings.builder()
    .autoEncryptionSettings(autoEncryptionSettings)
    .build();
```

# 8. 安全最佳实践清单

## 8.1 部署安全

- [ ] 始终开启认证（`authorization: enabled`）
- [ ] 使用TLS/SSL加密所有通信
- [ ] 关闭HTTP状态接口（`net.http.enabled: false` — 3.6已移除）
- [ ] 绑定特定IP（不要`bindIp: 0.0.0.0`除非必要）
- [ ] 使用防火墙限制MongoDB端口访问
- [ ] 使用非默认端口（不推荐作为唯一安全措施）

## 8.2 访问控制

- [ ] 为每个应用创建独立数据库用户
- [ ] 遵循最小权限原则
- [ ] 定期轮换密码
- [ ] 使用自定义角色而非全部给readWrite
- [ ] 删除或禁用默认的测试用户
- [ ] 避免在代码中硬编码密码（使用环境变量或密钥管理服务）

## 8.3 运行时安全

- [ ] 保持MongoDB版本更新（及时安装安全补丁）
- [ ] 开启审计日志
- [ ] 监控异常登录和操作
- [ ] 定期审查用户权限
- [ ] 禁用服务端JavaScript执行（如果不需要）
  ```yaml
  security:
    javascriptEnabled: false
  ```

## 8.4 数据安全

- [ ] 定期备份数据
- [ ] 对敏感字段使用客户端字段级加密
- [ ] 考虑静态数据加密（配合文件系统/LUKS加密）
- [ ] 制定并演练灾难恢复计划

## 8.5 应用安全

- [ ] 使用参数化查询（MongoDB Driver原生防注入）
- [ ] 不要直接在查询中拼接用户输入
- [ ] 校验输入数据（使用Schema Validation）
- [ ] 限制用户可执行的查询复杂度

## 8.6 Java应用安全配置

```java
// 生产环境配置示例
@Configuration
public class SecureMongoConfig {
    
    @Value("${mongodb.username}")
    private String username;
    
    @Value("${mongodb.password}")
    private String password;
    
    @Bean
    public MongoClient mongoClient() throws Exception {
        // 从环境变量获取凭据（不要硬编码）
        MongoCredential credential = MongoCredential.createScramSha256Credential(
            username, "admin", password.toCharArray());
        
        // TLS配置
        SSLContext sslContext = SSLContext.getInstance("TLSv1.2");
        // ... 加载证书配置
        
        MongoClientSettings settings = MongoClientSettings.builder()
            .credential(credential)
            .applyToSslSettings(builder -> 
                builder.enabled(true).context(sslContext))
            .applyToClusterSettings(builder ->
                builder.hosts(Collections.singletonList(
                    new ServerAddress("mongo-prod.internal", 27017))))
            .build();
        
        return MongoClients.create(settings);
    }
}
```

```yaml
# application-prod.yml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI}  # 从环境变量读取
      # mongodb://user:pass@host:27017/db?tls=true&authSource=admin
```
