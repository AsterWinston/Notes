# 1 Kafka 安全概述

## 1.1 安全威胁与防护维度

```
┌────────────────────────────────────────────────────────┐
│                   Kafka 安全防护体系                      │
│                                                        │
│  ① 认证 (Authentication) - 你是谁？                      │
│     ├── SSL/TLS 双向证书认证                           │
│     ├── SASL/PLAIN (用户名/密码)                       │
│     ├── SASL/SCRAM (加密密码)                          │
│     ├── SASL/GSSAPI (Kerberos)                        │
│     ├── SASL/OAUTHBEARER (OAuth 2.0)                  │
│     └── Delegation Token                              │
│                                                        │
│  ② 授权 (Authorization) - 你能做什么？                    │
│     └── ACL (Access Control List)                      │
│                                                        │
│  ③ 加密 (Encryption) - 数据是否安全传输？                │
│     └── SSL/TLS 传输加密                               │
│                                                        │
│  ④ 审计 (Audit) - 谁做了什么？                           │
│     └── Authorizer 日志 / 外部审计系统                   │
└────────────────────────────────────────────────────────┘
```

# 2 SSL/TLS 传输加密

## 2.1 生成证书

```bash
#!/bin/bash
# 生成 CA 证书
openssl req -new -x509 -keyout ca-key -out ca-cert -days 3650 \
    -subj "/CN=kafka-ca" -nodes

# 生成 Broker 密钥库（Java KeyStore）
keytool -keystore broker.keystore.jks -alias localhost \
    -validity 3650 -genkey -keyalg RSA \
    -dname "CN=broker1.example.com" \
    -storepass changeit -keypass changeit

# 生成 CSR
keytool -keystore broker.keystore.jks -alias localhost \
    -certreq -file broker.csr -storepass changeit

# 用 CA 签署
openssl x509 -req -CA ca-cert -CAkey ca-key \
    -in broker.csr -out broker.crt -days 3650 -CAcreateserial

# 导入 CA 证书和签名证书到 Broker 密钥库
keytool -keystore broker.keystore.jks -alias CARoot \
    -import -file ca-cert -storepass changeit -noprompt
keytool -keystore broker.keystore.jks -alias localhost \
    -import -file broker.crt -storepass changeit -noprompt

# 创建 Truststore（客户端用）
keytool -keystore client.truststore.jks -alias CARoot \
    -import -file ca-cert -storepass changeit -noprompt
```

## 2.2 Broker 配置 SSL

```properties
# server.properties

# 监听器（SSL 协议）
listeners=SSL://0.0.0.0:9093
advertised.listeners=SSL://broker1.example.com:9093

# SSL 密钥库配置
ssl.keystore.location=/var/ssl/private/broker.keystore.jks
ssl.keystore.password=changeit
ssl.key.password=changeit

# SSL Truststore（客户端证书验证，可选）
ssl.truststore.location=/var/ssl/private/broker.truststore.jks
ssl.truststore.password=changeit

# 双向认证（要求客户端也提供证书）
ssl.client.auth=required   # required / requested / none

# 协议和加密套件
ssl.protocol=TLSv1.2
ssl.enabled.protocols=TLSv1.2,TLSv1.3
ssl.cipher.suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256

# 端点识别
ssl.endpoint.identification.algorithm=HTTPS
```

## 2.3 Client 配置 SSL

```java
// Producer / Consumer 配置
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9093");
props.put("security.protocol", "SSL");

// Truststore（验证 Broker 证书）
props.put("ssl.truststore.location", "/path/to/client.truststore.jks");
props.put("ssl.truststore.password", "changeit");

// Keystore（双向认证时客户端证书）
props.put("ssl.keystore.location", "/path/to/client.keystore.jks");
props.put("ssl.keystore.password", "changeit");
props.put("ssl.key.password", "changeit");

// 端点识别
props.put("ssl.endpoint.identification.algorithm", "HTTPS");
```

# 3 SASL 认证

## 3.1 SASL/PLAIN（简单用户名密码）

```properties
# === Broker 配置 (server.properties) ===

# 启用 SASL/PLAIN
listeners=SASL_PLAINTEXT://0.0.0.0:9092

# 安全协议映射
listener.security.protocol.map=
    SASL_PLAINTEXT:SASL_PLAINTEXT,SSL:SSL

# SASL 机制
sasl.enabled.mechanisms=PLAIN

# Broker 间通信协议
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=PLAIN

# === JAAS 配置 (kafka_server_jaas.conf) ===
KafkaServer {
    org.apache.kafka.common.security.plain.PlainLoginModule required
    username="admin"
    password="admin-secret"
    user_admin="admin-secret"
    user_alice="alice-secret"
    user_bob="bob-secret";
};

# 启动时指定 JAAS 文件
export KAFKA_OPTS="-Djava.security.auth.login.config=/path/to/kafka_server_jaas.conf"
```

### 3.1.1 SASL/PLAIN Client 配置

```java
// Producer / Consumer
Properties props = new Properties();
props.put("security.protocol", "SASL_PLAINTEXT");
props.put("sasl.mechanism", "PLAIN");

// JAAS 配置 (通过代码)
props.put("sasl.jaas.config",
    "org.apache.kafka.common.security.plain.PlainLoginModule required " +
    "username=\"alice\" password=\"alice-secret\";");

// 或通过系统属性
// -Djava.security.auth.login.config=/path/to/client_jaas.conf
```

## 3.2 SASL/SCRAM（加密密码，推荐）

```bash
#!/bin/bash
# 1. 创建 SCRAM 凭证
kafka-configs.sh --bootstrap-server localhost:9092 \
    --alter --add-config \
    'SCRAM-SHA-256=[iterations=8192,password=admin-secret],SCRAM-SHA-512=[password=admin-secret]' \
    --entity-type users --entity-name admin

kafka-configs.sh --bootstrap-server localhost:9092 \
    --alter --add-config \
    'SCRAM-SHA-256=[iterations=8192,password=alice-secret],SCRAM-SHA-512=[password=alice-secret]' \
    --entity-type users --entity-name alice

# 2. Broker 配置
# server.properties
listeners=SASL_PLAINTEXT://0.0.0.0:9092
sasl.enabled.mechanisms=SCRAM-SHA-256,SCRAM-SHA-512
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256

# kafka_server_jaas.conf
KafkaServer {
    org.apache.kafka.common.security.scram.ScramLoginModule required;
};
```

## 3.3 SASL/GSSAPI（Kerberos）

```properties
# 适用于企业 AD/LDAP 环境
sasl.enabled.mechanisms=GSSAPI
sasl.kerberos.service.name=kafka

# JAAS 配置
KafkaServer {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/etc/security/keytabs/kafka.keytab"
    principal="kafka/broker1.example.com@EXAMPLE.COM";
};
```

## 3.4 Delegation Token（轻量级授权）

```properties
# Delegation Token 机制允许在不暴露长期凭证的情况下进行认证
# 适用于: 短期任务、跨系统通信

# Broker 配置
delegation.token.enable=true
delegation.token.max.lifetime.ms=604800000    # 7 天
delegation.token.expiry.time.ms=86400000       # 1 天

# 管理命令
# 创建 Token
kafka-delegation-tokens.sh --bootstrap-server localhost:9092 --create \
    --owner-principal alice --renewer-principal admin

# 查看 Token
kafka-delegation-tokens.sh --bootstrap-server localhost:9092 --describe \
    --owner-principal alice

# 续期 Token
kafka-delegation-tokens.sh --bootstrap-server localhost:9092 --renew \
    --hmac <token-hmac>

# 撤销 Token
kafka-delegation-tokens.sh --bootstrap-server localhost:9092 --expire \
    --hmac <token-hmac>
```

# 4 ACL 授权

## 4.1 授权模型

Kafka ACL（Access Control List）使用 **principal → operation → resource** 模型：

```
Principal (身份)     Operation (操作)      Resource (资源)
────────────────     ───────────────      ──────────────
User:alice           READ                 Topic:orders
User:bob             WRITE                Group:order-group
User:*               DESCRIBE             Cluster:kafka-cluster
Host:192.168.1.100  CREATE               Topic:*
```

### 4.1.1 Resource 类型与 Operation 对应关系

| Resource | 支持的 Operation |
|----------|-----------------|
| Topic | READ, WRITE, CREATE, DESCRIBE, DELETE, DESCRIBE_CONFIGS, ALTER_CONFIGS |
| Group | READ, DESCRIBE, DELETE |
| Cluster | CREATE, CLUSTER_ACTION, DESCRIBE_CONFIGS, ALTER_CONFIGS, IDEMPOTENT_WRITE |
| TransactionalId | DESCRIBE, WRITE |
| DelegationToken | DESCRIBE |
| User | CREATE_TOKENS, DESCRIBE_TOKENS |

## 4.2 启用 ACL

```properties
# server.properties

# 启用 ACL Authorizer
authorizer.class.name=kafka.security.authorizer.AclAuthorizer

# 超级用户（豁免 ACL 检查）
super.users=User:admin;User:kafka
```

## 4.3 ACL 管理命令

```bash
# === Topic 权限 ===

# 授予 alice 对 test-topic 的 READ 权限
kafka-acls.sh --bootstrap-server localhost:9092 \
    --add --allow-principal User:alice \
    --operation READ --topic test-topic

# 授予 bob 对 test-topic 的 WRITE 权限
kafka-acls.sh --bootstrap-server localhost:9092 \
    --add --allow-principal User:bob \
    --operation WRITE --topic test-topic

# 授予 producer 角色对所有 topic-* 的 READ + WRITE
kafka-acls.sh --bootstrap-server localhost:9092 \
    --add --allow-principal User:producer1 \
    --operation READ --operation WRITE \
    --topic "topic-" --resource-pattern-type prefixed

# === Consumer Group 权限 ===

# 授予 consumer 角色对 group 的 READ 权限（offset 读取）
kafka-acls.sh --bootstrap-server localhost:9092 \
    --add --allow-principal User:consumer1 \
    --operation READ --group my-group

# === Cluster 权限 ===

# 授予创建 Topic 的权限
kafka-acls.sh --bootstrap-server localhost:9092 \
    --add --allow-principal User:admin \
    --operation CREATE --cluster

# 授予幂等写入（需要 IDEMPOTENT_WRITE 权限）
kafka-acls.sh --bootstrap-server localhost:9092 \
    --add --allow-principal User:producer1 \
    --operation IDEMPOTENT_WRITE --cluster

# === 查看 ACL ===
kafka-acls.sh --bootstrap-server localhost:9092 --list

# 查看指定 Topic 的 ACL
kafka-acls.sh --bootstrap-server localhost:9092 \
    --list --topic test-topic

# === 移除 ACL ===
kafka-acls.sh --bootstrap-server localhost:9092 \
    --remove --allow-principal User:alice \
    --operation READ --topic test-topic
```

## 4.4 通配符与模式匹配

```bash
# 字面量匹配 (Literal): 精确匹配
--topic test-topic --resource-pattern-type literal

# 前缀匹配 (Prefixed): 匹配前缀
--topic "orders-" --resource-pattern-type prefixed
# 匹配: orders-us, orders-eu, orders-asia

# 任意匹配: 匹配所有（但需有前缀才能用 prefixed）
--topic "*" --resource-pattern-type literal

# 注意: 通配符 * 可以表示任意资源
--allow-principal User:*
```

# 5 安全协议组合

## 5.1 常见安全协议堆栈

```
┌─────────────────────────────────────────────────────┐
│         Kafka 安全协议组合 (Security Protocol)        │
│                                                     │
│  Level 0: 无安全                                     │
│    PLAINTEXT → 无认证, 无加密                         │
│                                                     │
│  Level 1: 仅加密                                     │
│    SSL → 传输加密, 可选双向认证                        │
│                                                     │
│  Level 2: 认证 + 加密                                 │
│    SASL_SSL → SASL 认证 + SSL 传输加密                │
│    (最常见的企业级配置)                                │
│                                                     │
│  Level 3: 认证 + 授权 + 加密 + 审计                    │
│    SASL_SSL + ACL Authorizer + Audit Logging         │
│                                                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  listener.security.protocol.map:                    │
│    INTERNAL:SASL_SSL (Broker 间通信, 内网)            │
│    EXTERNAL:SASL_SSL (Client 通信, 外网)              │
│    CONTROLLER:PLAINTEXT (KRaft Controller 通信, 内网) │
└─────────────────────────────────────────────────────┘
```

## 5.2 生产级安全配置模板

```properties
# === server.properties ===

# 多监听器配置
listeners=INTERNAL://0.0.0.0:9092,EXTERNAL://0.0.0.0:9093
advertised.listeners=INTERNAL://broker1.internal:9092,EXTERNAL://broker1.example.com:9093

# 安全协议映射
listener.security.protocol.map=\
    INTERNAL:SASL_SSL,EXTERNAL:SASL_SSL

# SASL 配置
sasl.enabled.mechanisms=SCRAM-SHA-512
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
security.inter.broker.protocol=SASL_SSL

# SSL 配置
ssl.keystore.location=/var/ssl/private/broker.keystore.jks
ssl.keystore.password=${SSL_KEYSTORE_PASSWORD}
ssl.key.password=${SSL_KEY_PASSWORD}
ssl.truststore.location=/var/ssl/private/broker.truststore.jks
ssl.truststore.password=${SSL_TRUSTSTORE_PASSWORD}
ssl.client.auth=required
ssl.endpoint.identification.algorithm=

# ACL
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin

# 强制安全（拒绝 PLAINTEXT 连接）
allow.everyone.if.no.acl.found=false
```

```java
// === Client 配置 (Producer / Consumer) ===

Properties props = new Properties();
props.put("bootstrap.servers", "broker1.example.com:9093");
props.put("security.protocol", "SASL_SSL");
props.put("sasl.mechanism", "SCRAM-SHA-512");

// SASL 凭证
props.put("sasl.jaas.config",
    "org.apache.kafka.common.security.scram.ScramLoginModule required " +
    "username=\"myuser\" password=\"mypassword\";");

// SSL Truststore
props.put("ssl.truststore.location", "/etc/kafka/secrets/client.truststore.jks");
props.put("ssl.truststore.password", "truststore-pass");

// SSL Keystore (如需双向认证)
props.put("ssl.keystore.location", "/etc/kafka/secrets/client.keystore.jks");
props.put("ssl.keystore.password", "keystore-pass");
props.put("ssl.key.password", "key-pass");
```

# 6 安全最佳实践

## 6.1 密码管理

```bash
# 不要在配置文件中明文存储密码
# 使用环境变量或外部 Secret 管理系统

# 示例：通过脚本注入密码
export KAFKA_OPTS="
    -Dssl.keystore.password=${KEYSTORE_PASSWORD}
    -Dssl.key.password=${KEY_PASSWORD}
    -Dssl.truststore.password=${TRUSTSTORE_PASSWORD}
    -Djava.security.auth.login.config=/path/to/jaas.conf
"

# JAAS 使用环境变量
# kafka_server_jaas.conf
KafkaServer {
    org.apache.kafka.common.security.scram.ScramLoginModule required
    username="admin"
    password="${ADMIN_PASSWORD}";
};
```

## 6.2 证书管理

1. **证书有效期**：建议不超过 1 年，定期轮换
2. **密钥库保护**：文件权限 600，目录权限 700
3. **CA 隔离**：内网 CA 和公网 CA 分开
4. **证书监控**：监控证书到期时间，提前预警

## 6.3 网络隔离

```
建议架构:

┌────────────────────────────────────────────────────┐
│                   生产环境网络                        │
│                                                    │
│  ┌──────────────┐   内网 9092      ┌──────────────┐ │
│  │  Producer    │────SASL_SSL────▶│   Broker     │ │
│  │  Consumer    │                 │              │ │
│  └──────────────┘                 │   内部:9092   │ │
│                                    │   INTERNAL   │ │
│  ┌──────────────┐   外网 9093    │              │ │
│  │  外部 Client  │───SASL_SSL───▶│   外部:9093   │ │
│  └──────────────┘                │   EXTERNAL   │ │
│                                   └──────────────┘ │
│                                                    │
│  Broker 间: INTERNAL listener (内网, 高速)          │
│  Client: EXTERNAL listener (外网, 经过防火墙)       │
│  不同监听器可配置不同认证机制                         │
└────────────────────────────────────────────────────┘
```

## 6.4 ACL 管理建议

1. **最小权限原则**：只为用户/服务授予必须的操作权限
2. **命名规范**：使用前缀模式批量为 Topic/Group 授权
3. **定期审计**：定期检查 ACL 列表和授权日志
4. **超级用户限制**：super.users 仅给运维账号，不用于日常操作
5. **全局映射**：建立 Principal → 服务/人员的映射文档
