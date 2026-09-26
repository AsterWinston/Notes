
# 1. ACL 概述

RocketMQ 从 **4.4.0** 版本开始支持 ACL（Access Control List），提供认证和授权机制。

```
安全机制层次：
┌──────────────────────────────────────┐
│  1. 认证 (Authentication)            │
│     - 确认客户端身份                  │
│     - AccessKey / SecretKey          │
│                                      │
│  2. 授权 (Authorization)             │
│     - 控制客户端对资源的访问          │
│     - Topic 级别 / Group 级别         │
│     - PUB / SUB / DENY               │
│                                      │
│  3. 传输加密 (TLS)                   │
│     - 防止数据在传输中被窃听          │
└──────────────────────────────────────┘
```

# 2. ACL 启用配置

## 2.1 Broker 端配置

```properties
# broker.conf
# 开启 ACL
aclEnable = true

# 自定义 ACL 配置文件路径（若为默认可省略）
# rocketmq.home.dir = /usr/local/rocketmq
```

## 2.2 ACL 配置文件 (plain_acl.yml)

```yaml
# conf/plain_acl.yml
globalWhiteRemoteAddresses:
  - 10.10.103.*
  - 192.168.0.*
  - 127.0.0.1      # 白名单地址不进行 ACL 检查

accounts:
  # 管理员账户（所有权限）
  - accessKey: RocketMQ
    secretKey: 12345678
    whiteRemoteAddress: 192.168.1.*
    admin: true

  # 生产者账户
  - accessKey: order-producer
    secretKey: producer123
    whiteRemoteAddress: 192.168.2.*
    group:
      - order-producer-group
    topicPerms:
      - orderTopic=PUB
      - paymentTopic=PUB|SUB   # 可发可收

  # 消费者账户
  - accessKey: order-consumer
    secretKey: consumer123
    whiteRemoteAddress: 192.168.3.*
    group:
      - order-consumer-group
    topicPerms:
      - orderTopic=SUB
      - paymentTopic=SUB

  # 只读账户
  - accessKey: monitor
    secretKey: monitor123
    whiteRemoteAddress: 10.10.0.*
    admin: true
    defaultTopicPerm: DENY     # 默认拒绝所有 Topic
    defaultGroupPerm: DENY
    topicPerms:
      - orderTopic=SUB
```

## 2.3 权限类型

| 权限 | 含义 |
|------|------|
| `PUB` | 允许向该 Topic 发送消息 |
| `SUB` | 允许从该 Topic 消费消息 |
| `PUB\|SUB` | 允许发送和消费 |
| `DENY` | 拒绝所有操作 |

# 3. 客户端使用 ACL

## 3.1 Producer

```java
// 创建 ACL 凭证
RPCHook rpcHook = new AclClientRPCHook(
    new SessionCredentials("order-producer", "producer123")
);

DefaultMQProducer producer = new DefaultMQProducer(
    "order-producer-group", rpcHook);
producer.setNamesrvAddr("127.0.0.1:9876");
producer.start();
```

## 3.2 Consumer

```java
RPCHook rpcHook = new AclClientRPCHook(
    new SessionCredentials("order-consumer", "consumer123")
);

DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(
    "order-consumer-group", rpcHook, 
    new AllocateMessageQueueAveragely());
consumer.setNamesrvAddr("127.0.0.1:9876");
consumer.subscribe("orderTopic", "*");
consumer.start();
```

## 3.3 Spring Boot 集成

```yaml
rocketmq:
  name-server: 127.0.0.1:9876
  producer:
    group: order-producer-group
    access-key: order-producer
    secret-key: producer123
```

# 4. ACL 源码解析

## 4.1 核心接口

```java
// AccessValidator 接口
public interface AccessValidator {
    AccessValidator parse(NettyServerConfig nettyServerConfig);
    AccessValidator parse(Map<String, Object> initParams);
    AccessValidator validate(ChannelHandlerContext context, RemotingCommand request);
    boolean validate(String resource, String necessaryPermForResource);
}
```

## 4.2 PlainAccessValidator 实现

```java
// 默认的 ACL 实现
public class PlainAccessValidator implements AccessValidator {
    private PlainPermissionManager permissionManager;

    @Override
    public AccessValidator parse(NettyServerConfig nettyServerConfig) {
        // 加载 conf/plain_acl.yml
        this.permissionManager.load();
        return this;
    }

    @Override
    public AccessValidator validate(ChannelHandlerContext ctx, RemotingCommand cmd) {
        // 1. 获取请求中的 AccessKey 和 Signature
        String accessKey = getAccessKeyFromRequest(cmd);
        String signature = getSignatureFromRequest(cmd);

        // 2. 本地计算签名
        PlainAccessResource accessResource = parse(accessKey, signature, cmd);

        // 3. 验证签名
        if (accessResource.getSignature().equals(signature)) {
            // 4. 验证权限
            // 验证是否有操作资源（Topic/PUB/SUB）的权限
            boolean authorized = this.permissionManager.checkPermission(accessResource);
            if (!authorized) {
                // 无权限 → 抛出 ACL 异常
                throw new AclException("Not authorized to access resource");
            }
        } else {
            throw new AclException("Invalid signature");
        }
        return this;
    }
}
```

## 4.3 签名算法

```java
// AclClientRPCHook — 客户端生成签名
public class AclClientRPCHook implements RPCHook {
    @Override
    public void doBeforeRequest(String remoteAddr, RemotingCommand request) {
        // 1. 获取当前时间戳
        // 2. 计算签名：HmacSHA1(secretKey, content)
        byte[] signature = AclSigner.calSignature(content, secretKey);
        // 3. 将签名和 accessKey 放入请求头
        request.addExtField(SessionCredentials.ACCESS_KEY, accessKey);
        request.addExtField(SessionCredentials.SECURITY_TOKEN, 
            Base64.encodeBase64String(signature));
        // 4. 添加时间戳防重放
        request.addExtField(SessionCredentials.SIGNATURE, 
            Base64.encodeBase64String(signature));
    }
}
```

# 5. ACL 在 Broker 请求处理中的位置

```
请求进入 Broker
    │
    ▼
RemotingServer 接收
    │
    ▼
RPCHook 执行 (before)
    │   ← ACLClientRPCHook.doBeforeRequest() 在这里被调用
    │   ← 如果验证失败 → 返回 ACL 异常 (403)
    │
    ▼
SendMessageProcessor / PullMessageProcessor...
    │
    ▼
RPCHook 执行 (after)
    │
    ▼
返回响应
```

# 6. TLS/SSL 配置

## 6.1 Broker 端

```properties
# broker.conf
tlsEnable = true
tlsClientAuthServer = true   # 双向认证
# 证书路径
tlsServerKeyPath = /etc/rocketmq/tls/server.key
tlsServerCertPath = /etc/rocketmq/tls/server.crt
tlsClientKeyPath = /etc/rocketmq/tls/client.key
tlsClientCertPath = /etc/rocketmq/tls/client.crt
```

## 6.2 Producer 端

```java
// 开启 TLS
System.setProperty("tlsEnable", "true");
System.setProperty("tlsClientKeyPath", "/path/to/client.key");
System.setProperty("tlsClientCertPath", "/path/to/client.crt");
```

# 7. 安全最佳实践

1. **生产环境开启 ACL**：至少配置 `accessKey/secretKey` 认证
2. **禁止公网直接暴露 Broker**：通过 Proxy 或防火墙
3. **最小权限原则**：每个应用只授予必要的 Topic 权限
4. **定期更换 SecretKey**：降低密钥泄露风险
5. **内网开启 TLS**：即使是内网也建议加密通信
6. **AD/LDAP 集成**：大规模部署时接入企业认证
7. **审计日志**：记录所有 ACL 拒绝事件

# 8. 参考

- [RocketMQ ACL 文档](https://github.com/apache/rocketmq/blob/master/docs/cn/acl/user_guide.md)
