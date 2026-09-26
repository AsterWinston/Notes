# 1.分布式部署架构详解

## 1.1集群设计原则

分布式 MinIO 的核心设计原则：

1. **无共享（Shared Nothing）**：每个节点完全对等，没有独立的元数据服务器
2. **最少 4 节点**：生产环境分布式部署最少需要 4 个物理/虚拟节点
3. **每节点最少 4 块磁盘**：生产环境每节点 4~16 块磁盘
4. **EC 默认配置 N/2 校验**：确保至少可容忍一半磁盘故障
5. **节点间必须网络互通**：通过 Gossip 协议保持集群状态同步
6. **所有节点时间同步**：强烈建议配置 NTP，时间偏差过大会导致认证失败
7. **推荐 XFS 文件系统**：XFS 的 O_DIRECT 模式提供最佳性能

### 典型生产集群配置

```text
方案 A：高密度存储（适合大文件、归档场景）
  节点数: 4
  每节点磁盘: 8
  总磁盘: 32
  EC 配置: 16+16
  容错: 16 磁盘或 4 节点故障
  存储效率: 50%
  适用: 冷数据存储、备份归档

方案 B：平衡型（适合通用场景）
  节点数: 8
  每节点磁盘: 4
  总磁盘: 32
  EC 配置: 16+16
  容错: 16 磁盘或 4 节点故障
  存储效率: 50%
  适用: 企业网盘、文件服务

方案 C：高性能（适合小文件、高并发）
  节点数: 16
  每节点磁盘: 4
  总磁盘: 64
  EC 配置: 8+8 (每个 Set)
  Erasure Set 数: 8 个 (每 Set 8 磁盘)
  容错: 每 Set 可容忍 4 磁盘故障
  存储效率: 50%
  适用: 小文件海量、高 QPS 场景
```

## 1.2网络规划

```text
┌──────────────────────────────────────────────────┐
│                    客户端网络                      │
│                (接入层: 10Gbps)                    │
└──────────┬────────┬──────────┬───────────────────┘
           │        │          │
    ┌──────▼──┐ ┌──▼─────┐ ┌──▼─────┐
    │  nginx  │ │ nginx  │ │ nginx  │   负载均衡器 (HA)
    └──────┬──┘ └──┬─────┘ └──┬─────┘
           │       │          │
    ┌──────▼───────▼──────────▼───────────────────┐
    │            内部网络 (25/100Gbps)              │
    │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐│
    │  │Node 1  │ │Node 2  │ │Node 3  │ │Node 4  ││
    │  │8×SSD   │ │8×SSD   │ │8×SSD   │ │8×SSD   ││
    │  └────────┘ └────────┘ └────────┘ └────────┘│
    └─────────────────────────────────────────────┘
                      │
              ┌───────▼────────┐
              │   管理网络      │
              │  (SSH/Console)  │
              └────────────────┘
```

### 网络要求

| 要求 | 说明 |
|------|------|
| 客户端 ↔ MinIO | 10Gbps 起，高吞吐场景 25Gbps+ |
| MinIO ↔ MinIO | 节点间通信（数据复制、EC 分发），强烈建议 25Gbps+ |
| 延迟 | 节点间延迟 < 1ms（同机房），< 10ms（同城） |
| 防火墙规则 | 开放端口 9000 (API)、9001 (Console)、节点间内部 RPC |
| 负载均衡 | 建议使用 Nginx/HAProxy 或硬件 LB |

## 1.3磁盘规划

```bash
# 每个节点上准备磁盘
# 使用 lsblk 查看可用磁盘
lsblk

# 格式化 XFS 文件系统
sudo mkfs.xfs /dev/sdb -f -L DISK1
sudo mkfs.xfs /dev/sdc -f -L DISK2
# ... 格式化所有数据磁盘

# 创建挂载点
sudo mkdir -p /mnt/disk{1..8}

# 挂载
sudo mount LABEL=DISK1 /mnt/disk1
sudo mount LABEL=DISK2 /mnt/disk2

# 配置 /etc/fstab 持久化挂载
# /dev/sdb /mnt/disk1 xfs defaults,noatime 0 2
# /dev/sdc /mnt/disk2 xfs defaults,noatime 0 2
```

### 磁盘性能要求

| 指标 | 最低要求 | 推荐 |
|------|----------|------|
| 类型 | HDD | NVMe SSD |
| 单盘 IOPS | 1000 | 100000+ |
| 单盘吞吐 | 100MB/s | 1000MB/s+ |
| 容量 | 1TB+ | 4TB+ |

## 1.4节点启动

所有 4 个节点执行完全相同的命令：

```bash
export MINIO_ROOT_USER=minioadmin
export MINIO_ROOT_PASSWORD=<强密码>

minio server \
  http://minio-{1...4}.example.com/mnt/disk{1...8}/data \
  --console-address ":9001"
```

```bash
# 或者使用 systemd（推荐生产环境）
sudo systemctl start minio
sudo systemctl enable minio
```

## 1.5集群状态验证

```bash
# 使用 mc 查看集群信息
mc admin info myminio

# 输出示例:
# ●  minio-1.example.com:9000
#    Uptime: 3 days
#    Drives: 8/8 OK
#    Used Capacity: 2.5 TiB
#
# ●  minio-2.example.com:9000
#    Uptime: 3 days
#    Drives: 8/8 OK
#    Used Capacity: 2.5 TiB
# ...

# 健康检查
mc admin info myminio
# 还可以查询特定节点
mc admin info myminio/minio-1.example.com:9000

# 检查集群拓扑
mc admin topology myminio
```

---

# 2.扩容方案

## 2.1添加新 Server Pool

```text
现有集群:
  Pool 0: 4 节点 × 8 磁盘 = 32 磁盘, EC: 16+16

扩容:
  Pool 1: 4 节点 × 8 磁盘 = 32 磁盘, EC: 16+16

扩容后:
  Pool 0 + Pool 1 = 64 磁盘
  总可用容量翻倍
```

### 扩容步骤

```bash
# 1. 准备新节点：安装 MinIO、格式化磁盘、配置 systemd

# 2. 在所有节点上更新启动命令
#    旧命令:
#    minio server http://minio-{1...4}.example.com/mnt/disk{1...8}/data
#    新命令:
#    minio server \
#      http://minio-{1...4}.example.com/mnt/disk{1...8}/data \
#      http://minio-{5...8}.example.com/mnt/disk{1...8}/data

# 3. 在所有节点上重启 MinIO（滚动重启）
#    先重启 Pool 0 的节点（需要重新启动以识别新 Pool）
#    然后启动 Pool 1 的节点

# 4. 验证扩容
mc admin info myminio
```

### 扩容注意事项

- **不能给已有 Pool 添加磁盘**：Pool 的 EC 配置是固定的，添加磁盘到已有 Pool 会破坏 EC 条带
- **新 Pool 应与旧 Pool 具有相同的 EC 条带大小**（即相同数量的节点和磁盘）
- **扩容后数据不会自动重平衡**：已有数据留在旧 Pool，新数据会分布到所有 Pool
- **如果旧 Pool 容量已满，MinIO 会自动将新对象写入新 Pool**

## 2.2退役节点（Decommissioning）

```bash
# 退役指定 Pool
mc admin decommission start myminio http://minio-{5...8}.example.com/mnt/disk{1...8}/data

# 查看退役状态
mc admin decommission status myminio

# 退役完成后，可以移除该 Pool 的节点
# 在剩余节点的启动命令中删除该 Pool 的端点
```

---

# 3.故障恢复机制

## 3.1磁盘故障恢复

```text
场景: Pool 0 中 minio-2 节点的 disk3 故障

MinIO 自动检测 → 标记故障磁盘
     │
     ▼
读操作: 从剩余正常分片 EC 解码恢复数据（对客户端透明）
写操作: 暂时跳过故障磁盘，写入 Quorum 满足即可
     │
     ▼
运维: 更换故障磁盘，MinIO 自动触发 Heal 过程
     │
     ▼
Heal: 从正常分片 EC 解码，重建故障磁盘上的数据
```

### 手动触发 Heal

```bash
# 扫描并修复整个集群
mc admin heal -r myminio

# 修复特定 Bucket
mc admin heal -r myminio/mybucket

# 查看 Heal 状态
mc admin heal myminio --verbose

# 检查挂起的修复任务
mc admin heal myminio --pending
```

## 3.2节点故障处理

```text
如果整个节点故障（如硬件故障、网络中断）：

1. MinIO 自动检测节点离线（通过 Gossip 协议）
2. 读操作：从其他节点的分片恢复数据
3. 写操作：跳过故障节点（Quorum 仍可满足，只要故障节点数 < N/2）
4. 节点恢复后：MinIO 自动触发 Heal，同步缺失的数据

哪些操作受影响:
  - List 操作无影响（Quorum 满足）
  - GET 操作无影响（EC 可从剩余分片恢复）
  - PUT 操作无影响（写入其他节点即可满足 Quorum）
  - 只要故障节点数 ≤ N/2-1（8+8配置下≤7），集群完全可用
```

---

# 4.身份与访问管理（IAM）

## 4.1MinIO 的用户体系

MinIO 的 IAM 系统管理两大类用户：

| 用户类型 | 说明 | 认证方式 |
|----------|------|----------|
| **Root 用户** | 集群安装时创建的唯一超级用户，拥有所有权限，不可删除 | AccessKey + SecretKey |
| **IAM 用户** | 通过 Console 或 mc admin 创建的普通用户 | AccessKey + SecretKey |
| **STS 用户** | 通过 OIDC/LDAP 认证后生成的临时用户 | JWT Token (临时) 或长期凭证 |
| **服务账号** | 绑定到 IAM 用户的长期凭证，用于应用程序 | AccessKey + SecretKey |

## 4.2创建和管理 IAM 用户

```bash
# 添加用户
mc admin user add myminio alice Password123!

# 查看用户列表
mc admin user list myminio

# 查看用户信息
mc admin user info myminio alice

# 启用/禁用用户
mc admin user disable myminio alice
mc admin user enable myminio alice

# 删除用户
mc admin user remove myminio alice
```

## 4.3创建和管理服务账号（Service Accounts）

服务账号推荐用于应用程序，可以绑定特定的权限策略：

```bash
# 创建服务账号（绑定到用户 alice）
mc admin user svcacct add myminio alice \
  --name "my-app-service-account" \
  --description "用于 Spring Boot 应用的文件服务"

# 输出:
# Access Key: SVSRVxxxxxxxxx
# Secret Key: yyyyyyyyyyyyyyy
# (请妥善保管 Secret Key，后续无法再次获取)

# 查看服务账号列表
mc admin user svcacct list myminio alice

# 删除服务账号
mc admin user svcacct remove myminio <access-key>

# 禁用服务账号
mc admin user svcacct disable myminio <access-key>
```

## 4.4创建和管理 Policy（策略）

Policy 是 JSON 格式的权限定义文件，使用 AWS IAM 兼容语法：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket/*",
        "arn:aws:s3:::my-bucket"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket"
      ]
    }
  ]
}
```

### 常用策略示例

**只读策略**（readonly.json）：
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": ["arn:aws:s3:::public-bucket", "arn:aws:s3:::public-bucket/*"]
    }
  ]
}
```

**读写策略**（readwrite.json）：
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject", "s3:PutObject", "s3:DeleteObject",
        "s3:ListBucket", "s3:GetBucketLocation",
        "s3:ListMultipartUploadParts", "s3:AbortMultipartUpload"
      ],
      "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"]
    }
  ]
}
```

**管理员策略**（admin.json）：
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*"],
      "Resource": ["arn:aws:s3:::*"]
    },
    {
      "Effect": "Allow",
      "Action": ["admin:*"],
      "Resource": ["arn:aws:s3:::*"]
    }
  ]
}
```

### 管理 Policy 和关联用户

```bash
# 创建策略
mc admin policy create myminio readonly readonly.json
mc admin policy create myminio readwrite readwrite.json

# 列出策略
mc admin policy list myminio

# 查看策略内容
mc admin policy info myminio readonly

# 关联策略到用户
mc admin policy attach myminio readonly --user alice

# 查看用户关联的策略
mc admin user info myminio alice

# 移除用户策略关联
mc admin policy detach myminio readonly --user alice

# 关联策略到用户组
mc admin policy attach myminio readwrite --group developers

# 删除策略
mc admin policy remove myminio readonly
```

## 4.5用户组管理

```bash
# 创建用户组
mc admin group add myminio developers alice bob

# 查看用户组信息
mc admin group info myminio developers

# 列出所有组
mc admin group list myminio

# 从组中移除用户
mc admin group remove myminio developers bob

# 删除组
mc admin group disable myminio developers
```

---

# 5.Bucket 级别策略

Bucket Policy 是绑定在 Bucket 上的访问控制，支持匿名访问和条件限制。

## 5.1设置公共读 Bucket

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": ["*"]},
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::public-bucket/*"]
    }
  ]
}
```

```bash
# 通过 mc 设置公开读
mc anonymous set public myminio/public-bucket

# 设置私有
mc anonymous set private myminio/public-bucket

# 设置仅允许下载（不允许列举目录）
mc anonymous set download myminio/public-bucket
```

## 5.2条件访问控制

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": ["*"]},
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::web-bucket/*"],
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": ["10.0.0.0/8", "192.168.0.0/16"]
        },
        "StringEquals": {
          "aws:Referer": ["https://myapp.example.com"]
        }
      }
    }
  ]
}
```

### Java SDK 设置 Bucket Policy

```java
String policyJson = """
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": ["*"]},
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::public-bucket/*"],
      "Condition": {
        "IpAddress": {"aws:SourceIp": ["10.0.0.0/8"]}
      }
    }
  ]
}
""";

minioClient.setBucketPolicy(
    SetBucketPolicyArgs.builder()
        .bucket("public-bucket")
        .config(policyJson)
        .build()
);

// 获取 Bucket Policy
String retrievedPolicy = minioClient.getBucketPolicy(
    GetBucketPolicyArgs.builder()
        .bucket("public-bucket")
        .build()
);

// 删除 Bucket Policy
minioClient.deleteBucketPolicy(
    DeleteBucketPolicyArgs.builder()
        .bucket("public-bucket")
        .build()
);
```

---

# 6.TLS/SSL 配置

## 6.1自签名证书配置

```bash
# 生成自签名证书
openssl req -x509 -newkey rsa:4096 -keyout private.key \
  -out public.crt -days 365 -nodes \
  -subj "/CN=minio.example.com"

# 证书存放目录（MinIO 自动检测）
# Linux: ${HOME}/.minio/certs/
# 或: /etc/minio/certs/

# 文件命名规范:
# public.crt  - 证书文件（必须这个名称）
# private.key - 私钥文件（必须这个名称）

sudo mkdir -p /etc/minio/certs
sudo cp public.crt /etc/minio/certs/
sudo cp private.key /etc/minio/certs/
sudo chown -R minio-user:minio-user /etc/minio/certs
sudo chmod 600 /etc/minio/certs/private.key
```

启动 MinIO 时自动启用 HTTPS：

```bash
minio server /data --console-address ":9001" --certs-dir /etc/minio/certs
```

## 6.2使用 Let's Encrypt 自动签发证书

```bash
# 设置环境变量
export MINIO_ROOT_USER=minioadmin
export MINIO_ROOT_PASSWORD=<密码>
export MINIO_DOMAIN=minio.example.com

# MinIO 会自动通过 Let's Encrypt 获取证书
minio server /data --console-address ":9001"
```

## 6.3Java 客户端连接 HTTPS MinIO

```java
// 信任自签名证书
import javax.net.ssl.*;
import java.security.cert.X509Certificate;
import okhttp3.OkHttpClient;

// 仅开发环境使用：信任所有证书
TrustManager[] trustAllCerts = new TrustManager[]{
    new X509TrustManager() {
        public X509Certificate[] getAcceptedIssuers() { return new X509Certificate[0]; }
        public void checkClientTrusted(X509Certificate[] certs, String authType) {}
        public void checkServerTrusted(X509Certificate[] certs, String authType) {}
    }
};

SSLContext sslContext = SSLContext.getInstance("TLSv1.2");
sslContext.init(null, trustAllCerts, new java.security.SecureRandom());

OkHttpClient httpClient = new OkHttpClient.Builder()
    .sslSocketFactory(sslContext.getSocketFactory(), (X509TrustManager) trustAllCerts[0])
    .hostnameVerifier((hostname, session) -> true)
    .build();

MinioClient minioClient = MinioClient.builder()
    .endpoint("https://minio.example.com:9000")
    .credentials("minioadmin", "minioadmin")
    .httpClient(httpClient)
    .build();
```

> **生产环境**：将 CA 证书导入 JVM 信任库，而非信任所有证书。
> ```bash
> keytool -import -alias minio -keystore $JAVA_HOME/lib/security/cacerts \
>   -file /path/to/minio-ca.crt -storepass changeit -noprompt
> ```

---

# 7.集成 LDAP / Active Directory

## 7.1配置 LDAP 认证

通过环境变量配置 LDAP 集成：

```bash
# MinIO 服务端环境变量
export MINIO_IDENTITY_LDAP_SERVER_ADDR="ldap.example.com:389"
export MINIO_IDENTITY_LDAP_LOOKUP_BIND_DN="cn=admin,dc=example,dc=com"
export MINIO_IDENTITY_LDAP_LOOKUP_BIND_PASSWORD="ldap_admin_password"
export MINIO_IDENTITY_LDAP_USER_DN_SEARCH_BASE="ou=users,dc=example,dc=com"
export MINIO_IDENTITY_LDAP_USER_DN_SEARCH_FILTER="(uid=%s)"
export MINIO_IDENTITY_LDAP_GROUP_SEARCH_BASE="ou=groups,dc=example,dc=com"
export MINIO_IDENTITY_LDAP_GROUP_SEARCH_FILTER="(&(objectClass=groupOfNames)(member=%d))"

minio server /data --console-address ":9001"
```

## 7.2配置 OpenID Connect (OIDC)

```bash
# 以 Keycloak 为例配置 OIDC
export MINIO_IDENTITY_OPENID_CONFIG_URL="https://keycloak.example.com/realms/minio/.well-known/openid-configuration"
export MINIO_IDENTITY_OPENID_CLIENT_ID="minio-client"
export MINIO_IDENTITY_OPENID_CLIENT_SECRET="client-secret"
export MINIO_IDENTITY_OPENID_CLAIM_NAME="policy"  # JWT 中的策略字段
export MINIO_IDENTITY_OPENID_DISPLAY_NAME="Keycloak SSO"
export MINIO_IDENTITY_OPENID_SCOPES="openid,profile,email"
export MINIO_IDENTITY_OPENID_REDIRECT_URI="https://minio.example.com:9001/oauth_callback"

minio server /data --console-address ":9001"
```

---

# 8.KES（Key Encryption Service）

## 8.1KES 概述

KES 是 MinIO 官方提供的密钥加密服务，用于 SSE-KMS 场景。它需要配合外部 KMS（如 HashiCorp Vault）使用。

```text
┌─────────┐     ┌─────────┐     ┌─────────────────┐
│  MinIO  │────▶│   KES   │────▶│ HashiCorp Vault │
│  Server │     │  Server │     │  (KMS 后端)      │
└─────────┘     └─────────┘     └─────────────────┘
   SSE-KMS        密钥代理         密钥存储/轮换
```

## 8.2启动 KES 服务

```bash
# 下载 KES
wget https://github.com/minio/kes/releases/latest/download/kes-linux-amd64
chmod +x kes-linux-amd64
sudo mv kes-linux-amd64 /usr/local/bin/kes

# KES 配置文件 (kes-config.yml)
# 配置 Vault 后端
# 启动 KES
kes server --config kes-config.yml --auth off
```

### KES 配置示例（使用 Vault）

```yaml
# kes-config.yml
address: 0.0.0.0:7373
root: disabled  # 生产环境应禁用 root 身份

tls:
  key: /path/to/kes.key
  cert: /path/to/kes.crt

policy:
  my-policy:
    paths:
      - /v1/key/create/minio-*
      - /v1/key/generate/minio-*
      - /v1/key/decrypt/minio-*
    identities:
      - ${MINIO_KES_IDENTITY}

keystore:
  vault:
    endpoint: https://vault.example.com:8200
    prefix: minio-keys
    approle:
      engine: /path/to/approle
      id: ${VAULT_APPROLE_ID}
      secret: ${VAULT_APPROLE_SECRET}
    status:
      ping: 10s
```

## 8.3MinIO 连接 KES

```bash
# MinIO 服务端环境变量
export MINIO_KMS_KES_ENDPOINT="https://kes.example.com:7373"
export MINIO_KMS_KES_KEY_NAME="minio-default-key"
export MINIO_KMS_KES_CERT_FILE="/path/to/client.crt"
export MINIO_KMS_KES_KEY_FILE="/path/to/client.key"
export MINIO_KMS_KES_CA_PATH="/path/to/ca.crt"
```

---

# 9.审计日志

## 9.1配置审计日志

```bash
# 设置审计日志输出目标
# 输出到 Webhook（HTTP POST）
mc admin config set myminio audit_webhook:primary \
  endpoint="http://log-collector:8080/audit" \
  auth_token="your-auth-token"

# 输出到 Kafka
mc admin config set myminio audit_kafka:primary \
  brokers="kafka1:9092,kafka2:9092" \
  topic="minio-audit-logs"

# 重新加载配置
mc admin service restart myminio
```

## 9.2审计日志格式

```json
{
  "time": "2024-01-15T10:30:00.123456Z",
  "api": {
    "name": "PutObject",
    "bucket": "my-bucket",
    "object": "photos/avatar.jpg"
  },
  "remotehost": "192.168.1.100",
  "requestID": "17A3F4B5E6C8D9E0",
  "userAgent": "MinIO (linux; amd64) minio-java/8.5.12",
  "requestHeader": {
    "Authorization": "AWS4-HMAC-SHA256 Credential=***/20240115/...",
    "Content-Type": "image/jpeg",
    "User-Agent": "..."
  },
  "responseHeader": {
    "ETag": "abc123...",
    "X-Amz-Request-Id": "17A3F4B5E6C8D9E0",
    "Content-Length": "0",
    "X-Xss-Protection": "1; mode=block"
  },
  "tags": {
    "name": "minio-1.example.com:9000"
  }
}
```

---

# 10.安全最佳实践清单

```text
认证与访问控制:
  □ Root 用户密码使用强密码（至少 16 位，含大小写+数字+特殊字符）
  □ 不使用 Root 用户进行日常操作
  □ 为应用程序创建专用的 IAM 用户/服务账号
  □ 使用最小权限原则（ReadOnly / ReadWrite）
  □ 定期轮换 AccessKey/SecretKey
  □ 启用 OIDC/LDAP 统一认证（如果有企业 IdP）
  □ 限制拥有 admin 权限的用户数量

网络安全:
  □ 启用 TLS/SSL（生产环境强制 HTTPS）
  □ 使用经过 CA 签名的证书（不要用自签名证书）
  □ 配置防火墙，限制访问来源 IP
  □ 将 MinIO 部署在内网，通过反向代理对外暴露
  □ 启用 VPC/安全组隔离
  □ 限制 Console 访问来源 IP

加密:
  □ 对敏感 Bucket 启用默认加密
  □ 使用 SSE-KMS 实现密钥集中管理和审计
  □ 定期轮换 KMS 主密钥
  □ 确保传输加密和存储加密同时启用

数据保护:
  □ 对关键数据启用对象锁（防勒索）
  □ 开启版本控制（防止误删）
  □ 配置跨集群复制（灾备）
  □ 定期验证备份数据可恢复性
  □ 配置生命周期自动清理过期数据

运维安全:
  □ 启用审计日志并集中存储
  □ 监控异常登录和操作
  □ 定期更新 MinIO 版本
  □ 关注安全公告（CVE）
  □ 定期执行安全扫描
  □ 限制 SSH 访问 MinIO 节点
  □ 保持操作系统和内核更新

磁盘与数据安全:
  □ 使用加密磁盘（LUKS/加密存储）
  □ 磁盘退役前使用安全擦除（shred / blkdiscard）
  □ 备份 format.json（含集群拓扑信息，丢失后无法启动集群）
```
