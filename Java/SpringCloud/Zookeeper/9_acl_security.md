
# 1. ACL 概述

ZooKeeper 提供了基于 ACL（Access Control List，访问控制列表）的权限控制机制，用于控制谁可以对 ZNode 执行什么操作。

```text
ZooKeeper 安全模型：
┌──────────────────────────────────────────────────────┐
│                                                       │
│  ┌──────────────────────────────────────────────┐    │
│  │            ACL 权限模型                        │    │
│  │                                                │    │
│  │  Principal（主体） + Permission（权限）         │    │
│  │                                                │    │
│  │  示例：                                        │    │
│  │    scheme:id:permissions                       │    │
│  │    world:anyone:cdrwa                          │    │
│  │    digest:user:password_hash:cdrwa             │    │
│  │    ip:192.168.1.0/24:r                        │    │
│  └──────────────────────────────────────────────┘    │
│                                                       │
│  每个 ZNode 拥有自己的 ACL 列表                        │
│  ACL 权限检查在服务端执行，不可被客户端绕过             │
│                                                       │
└──────────────────────────────────────────────────────┘
```

# 2. ACL 组成与权限位

## 2.1 权限位定义

ZooKeeper 定义了 5 种权限：

| 权限 | 常量值 | 含义 | 影响的命令 |
|------|--------|------|-----------|
| **CREATE** | 1 (0b00001) | 可以创建子节点 | `create` |
| **READ** | 2 (0b00010) | 可以读取节点数据、查看子节点列表 | `getData`, `getChildren` |
| **WRITE** | 4 (0b00100) | 可以设置节点数据 | `setData` |
| **DELETE** | 8 (0b01000) | 可以删除子节点（注意：删除的是子节点，非自身） | `delete` |
| **ADMIN** | 16 (0b10000) | 可以设置 ACL | `setAcl` |

```text
权限与操作的对应关系：
┌──────────────────────────────────────────────────────┐
│                                                       │
│  create   → 父节点的 CREATE 权限（对所创建节点本身无要求）│
│  delete   → 父节点的 DELETE 权限                      │
│  getData  → 节点本身的 READ 权限                      │
│  setData  → 节点本身的 WRITE 权限                     │
│  getChildren → 节点本身的 READ 权限                   │
│  setAcl   → 节点本身的 ADMIN 权限                     │
│  getAcl   → 任何权限均可（不受 ACL 限制）             │
│                                                       │
└──────────────────────────────────────────────────────┘
```

## 2.2 权限组合（复合权限）

```java
// 常用的权限组合常量
ZooDefs.Perms.READ    // 只读：r (2)
ZooDefs.Perms.WRITE   // 只写：w (4)
ZooDefs.Perms.CREATE  // 创建：c (1)
ZooDefs.Perms.DELETE  // 删除：d (8)
ZooDefs.Perms.ADMIN   // 管理：a (16)
ZooDefs.Perms.ALL     // 全部：cdrwa (31 = 1|2|4|8|16)

// 组合权限示例：
int customPerm = Perms.READ | Perms.WRITE;  // rw = 6 (仅读写)
int fullAccess = Perms.ALL;                  // cdrwa = 31
```

# 3. 内置的 ACL Schemes

ZooKeeper 提供了五种内置的认证方案（Scheme）：

## 3.1 world（全局权限）

```text
world Scheme：
┌──────────────────────────────────────────────────────┐
│  · 默认 Scheme，表示所有人                             │
│  · id 固定为 "anyone"                                │
│  · 所有 Client 都属于 world:anyone                   │
│  · 新创建的 ZNode 默认 ACL：world:anyone:cdrwa       │
│                                                       │
│  使用方式：                                           │
│    world:anyone:r        → 所有人可读                 │
│    world:anyone:cdrwa    → 所有人拥有全部权限（默认）  │
└──────────────────────────────────────────────────────┘
```

## 3.2 auth（已认证用户）

```text
auth Scheme：
┌──────────────────────────────────────────────────────┐
│  · 表示所有已通过认证的用户                           │
│  · id 留空（实际匹配时自动填充）                      │
│  · 只有已调用 addAuthInfo() 的客户端才匹配            │
│  · 不关心具体是哪个用户，只要认证过即可               │
│                                                       │
│  使用方式：                                           │
│    auth::r              → 任何已认证用户可读          │
│                                                       │
│  示例：                                               │
│    Client A 认证为 "user:pass1" → 匹配 auth 权限       │
│    Client B 认证为 "user:pass2" → 匹配 auth 权限       │
│    Client C 未认证              → 不匹配 auth 权限     │
└──────────────────────────────────────────────────────┘
```

## 3.3 digest（用户名:密码摘要）

```text
digest Scheme：
┌──────────────────────────────────────────────────────┐
│  · 基于用户名和密码 SHA-1 摘要的认证                  │
│  · 格式：digest:username:base64(sha1(username:password))│
│  · 密码在传输和存储时都使用 SHA-1 哈希                │
│  · 提供具体的用户级权限控制                            │
│                                                       │
│  生成 digest 字符串：                                  │
│    $ java -cp zookeeper.jar \                         │
│      org.apache.zookeeper.server.auth.DigestAuthenticationProvider \│
│      super:secret123                                  │
│    → super:UdxDQl4f9CvphSA4d9mKz3o9tcY=              │
│                                                       │
│  使用方式：                                           │
│    digest:super:UdxDQl4f9CvphSA4d9mKz3o9tcY=:cdrwa  │
└──────────────────────────────────────────────────────┘
```

## 3.4 ip（IP 地址控制）

```text
ip Scheme：
┌──────────────────────────────────────────────────────┐
│  · 基于客户端 IP 地址的访问控制                        │
│  · 支持 CIDR 格式的子网掩码                           │
│  · 适用于内网环境                                     │
│                                                       │
│  使用方式：                                           │
│    ip:192.168.1.100:r         → 单个 IP              │
│    ip:192.168.1.0/24:rw       → 子网范围             │
│    ip:10.0.0.0/8:cdrwa        → A 类子网             │
└──────────────────────────────────────────────────────┘
```

## 3.5 x509（证书认证，ZooKeeper 3.5+）

```text
x509 Scheme：
┌──────────────────────────────────────────────────────┐
│  · 基于 X.509 客户端证书的认证                         │
│  · TLS 连接建立时自动提取客户端证书 DN                 │
│  · 格式：x509:CN=client1,O=MyOrg:crw                 │
│  · 适用于高安全要求的场景                              │
└──────────────────────────────────────────────────────┘
```

# 4. ACL 操作 API

## 4.1 Java 原生 API

```java
// 创建带 ACL 的节点
List<ACL> acls = new ArrayList<>();
acls.add(new ACL(
    ZooDefs.Perms.READ | ZooDefs.Perms.WRITE,
    new Id("digest", "user:UdxDQl4f9CvphSA4d9mKz3o9tcY=")
));
acls.add(new ACL(
    ZooDefs.Perms.READ,
    new Id("world", "anyone")
));
zk.create("/secure", data, acls, CreateMode.PERSISTENT);

// 获取节点的 ACL
List<ACL> currentAcls = zk.getACL("/secure", stat);

// 设置节点的 ACL
zk.setACL("/secure", newAcls, stat.getAversion());

// 使用预定义的 ACL
import org.apache.zookeeper.ZooDefs.Ids;

Ids.OPEN_ACL_UNSAFE          // world:anyone:cdrwa（默认）
Ids.READ_ACL_UNSAFE          // world:anyone:r
Ids.CREATOR_ALL_ACL          // 创建者拥有全部权限（auth::cdrwa）
```

## 4.2 Curator 的 ACL API

```java
// Curator 的 ACL Provider
CuratorFramework client = CuratorFrameworkFactory.builder()
    .connectString("localhost:2181")
    .authorization("digest", "user:password".getBytes())
    .aclProvider(new ACLProvider() {
        @Override
        public List<ACL> getDefaultAcl() {
            return Collections.singletonList(
                new ACL(Perms.ALL, new Id("digest",
                    DigestAuthenticationProvider
                        .generateDigest("user:password")))
            );
        }
        @Override
        public List<ACL> getAclForPath(String path) {
            if (path.startsWith("/public")) {
                return ZooDefs.Ids.OPEN_ACL_UNSAFE;
            }
            return getDefaultAcl();
        }
    })
    .build();
```

# 5. 客户端认证

```java
// 客户端添加认证信息
ZooKeeper zk = new ZooKeeper("host:2181", 30000, watcher);

// digest 认证
zk.addAuthInfo("digest", "user:password".getBytes());

// 之后该客户端的所有操作都带有 "user" 的身份标识
// 可以访问 ACL 中包含 digest:user:... 的节点
```

```bash
# CLI 认证示例
addauth digest user:password
create /secure "data" digest:user:password:cdrwa
# 另一个客户端需要先认证才能访问
```

# 6. SASL 认证

ZooKeeper 支持使用 SASL（Simple Authentication and Security Layer）进行客户端-服务端和集群节点间的认证：

```properties
# zoo.cfg — 启用 SASL 认证

# 服务端启用 SASL
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
requireClientAuthScheme=sasl
jaasLoginRenew=3600000

# 客户端 JAAS 配置文件（zk_client_jaas.conf）
zookeeper.sasl.client=true
zookeeper.sasl.clientconfig=Client
```

```text
JAAS 配置文件示例（zk_server_jaas.conf）：
┌──────────────────────────────────────────────────────┐
│  Server {                                            │
│    org.apache.zookeeper.server.auth.DigestLoginModule│
│    required                                          │
│    user_super="secret123";  ← 定义用户和密码         │
│  };                                                  │
└──────────────────────────────────────────────────────┘
```

# 7. SSL/TLS 加密通信

ZooKeeper 3.5+ 支持 SSL/TLS 加密客户端-服务端和集群节点间的通信：

```properties
# zoo.cfg — SSL/TLS 配置示例

# 启用 SSL（需要同时开启一个 SSL 端口和一个非 SSL 端口）
ssl.keyStore.location=/path/to/server.jks
ssl.keyStore.password=keystore_password
ssl.trustStore.location=/path/to/truststore.jks
ssl.trustStore.password=truststore_password
ssl.clientAuth=need  # none / want / need

# SSL 客户端端口
secureClientPort=2281

# 集群间通信加密（3.6+）
sslQuorum=true
ssl.quorum.keyStore.location=/path/to/server.jks
ssl.quorum.keyStore.password=keystore_password
ssl.quorum.trustStore.location=/path/to/truststore.jks
ssl.quorum.trustStore.password=truststore_password
```

```text
安全通信架构图：
┌──────────────────────────────────────────────────────────┐
│                                                           │
│   Client (SSL)                                            │
│       │                                                   │
│       │ TLS 加密                                          │
│       ▼                                                   │
│  ┌──────────────────────────────────────────────────┐    │
│  │                  ZooKeeper Cluster                 │    │
│  │                                                   │    │
│  │  ┌─────────┐    TLS 加密    ┌─────────┐          │    │
│  │  │ Server1 │◄───────────────►│ Server2 │          │    │
│  │  └─────────┘                └─────────┘          │    │
│  │       ▲                         ▲                 │    │
│  │       └───────── TLS ──────────┘                 │    │
│  │                       │                           │    │
│  │                  ┌─────────┐                      │    │
│  │                  │ Server3 │                      │    │
│  │                  └─────────┘                      │    │
│  └──────────────────────────────────────────────────┘    │
│                                                           │
│  三处需要加密：                                           │
│    1. Client ↔ Server（SSL 连接）                         │
│    2. Server ↔ Server 通信（选举和同步）                  │
│    3. Server ↔ Server 通信（事务广播）                    │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

# 8. ACL 的继承与传播

```text
重要：ZooKeeper 的 ACL 不具有继承性！

┌──────────────────────────────────────────────────────┐
│                                                       │
│  /app                ACL: world:anyone:cdrwa         │
│    ├── /app/config   ACL: （独立设置）                 │
│    └── /app/data     ACL: （独立设置）                 │
│                                                       │
│  · 父节点的 ACL 不自动传递给子节点                    │
│  · 每个子节点拥有自己独立的 ACL                       │
│  · 创建子节点时需要显式指定 ACL（否则继承客户端的 ACL）│
│                                                       │
│  默认 ACL 行为（Curator）：                            │
│    // 使用全局默认 ACL Provider                       │
│    client.setAclProvider(provider);                   │
│    // 创建节点时若未指定 ACL 则使用 provider 提供的   │
│                                                       │
└──────────────────────────────────────────────────────┘
```

# 9. ACL 最佳实践

```text
ACL 使用建议：

1. 分层权限设计
   ┌──────────────────────────────────────────────────┐
   │  /app                                             │
   │    ├── /public      → world:anyone:r             │
   │    ├── /restricted   → digest:app_user:...:r      │
   │    └── /admin        → digest:admin:...:cdrwa    │
   │                                                   │
   │  按功能的权限分级，遵循最小权限原则                │
   └──────────────────────────────────────────────────┘

2. 环境隔离
   ┌──────────────────────────────────────────────────┐
   │  · 开发环境：使用 world:anyone:cdrwa              │
   │  · 测试环境：使用 digest 基础认证                 │
   │  · 生产环境：启用 SASL + TLS + ip 白名单          │
   └──────────────────────────────────────────────────┘

3. 生产环境安全加固检查清单
   ┌──────────────────────────────────────────────────┐
   │  □ 禁用 world:anyone:cdrwa（默认 ACL）            │
   │  □ 启用 digest 或 SASL 认证                       │
   │  □ 开启 SSL/TLS 加密传输                           │
   │  □ 配置 IP 白名单（防火墙 + ip ACL）              │
   │  □ 使用独立网络平面（隔离客户端和内网流量）        │
   │  □ 定期审计 ACL 配置                              │
   │  □ ZooKeeper 管理端口（8080）仅内网访问           │
   │  □ 禁用四字命令中泄露信息的命令（如 dump/conf）   │
   └──────────────────────────────────────────────────┘

4. 密码管理
   ┌──────────────────────────────────────────────────┐
   │  · digest 密码使用专用工具生成 SHA-1 摘要         │
   │  · 不要在日志中输出明文 ACL                      │
   │  · 定期轮换密码                                   │
   │  · 使用 JAAS 配置文件管理密码（非硬编码）         │
   └──────────────────────────────────────────────────┘
```

# 10. 常见 ACL 问题排查

| 问题 | 错误信息 | 解决方案 |
|------|---------|---------|
| 无权限读取 | `KeeperErrorCode = NoAuth` | 调用 `addAuthInfo()` 添加认证；检查 ACL 权限位 |
| 无权限写入 | `KeeperErrorCode = NoAuth` | 同上 |
| ACL 设置失败 | `KeeperErrorCode = NoAuth` | 需要 ADMIN 权限才能修改 ACL |
| 默认 ACL 风险 | 无报错 | 生产环境禁用 `OPEN_ACL_UNSAFE` |
| digest 认证失败 | `AuthFailedException` | 检查用户名密码是否正确；确认使用 SHA-1 摘要格式 |
| IP 限制不生效 | 无权限但预期有 | 检查客户端实际出口 IP（代理/NAT 后的 IP） |
