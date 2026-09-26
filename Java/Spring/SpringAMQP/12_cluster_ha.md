# 1.RabbitMQ集群概述

## 1.1为什么需要集群
单节点RabbitMQ存在以下问题：
- **单点故障**：节点宕机后所有消息服务不可用
- **性能瓶颈**：单个节点的CPU、内存、网络带宽有限
- **容量限制**：单个节点的存储空间有限

集群可以解决上述问题：
- **高可用**：多节点冗余，单节点故障不影响整体服务
- **负载均衡**：多节点分担消息流量压力
- **水平扩展**：通过增加节点来提升整体处理能力

## 1.2RabbitMQ集群的架构特点
RabbitMQ集群的设计与其他分布式系统有些不同：

1. **元数据复制**：RabbitMQ集群会复制Exchange、Queue、Binding等元数据到所有节点（但不默认复制消息内容）
2. **消息不自动复制**：默认情况下，Queue中的消息只存储在声明的节点上，其他节点只知道消息的元信息（如Queue中有100条消息），但实际消息内容不在本地
3. **节点间消息拉取**：消费者连接到非Queue所在节点时，会通过集群内部通信从Queue所在节点拉取消息
4. **基于Erlang Cookie**：节点间通过共享的Erlang Cookie建立信任关系

# 2.集群模式详解

## 2.1普通集群（默认模式）
```
节点A              节点B              节点C
 ├── Queue1         ├── Queue2         ├── Queue3（消息存储在节点A）
 │   └── 消息1..N  │   └── 消息1..M   │   物理上不存在，从A拉取
 └── 元数据（全部） └── 元数据（全部）  └── 元数据（全部）
```

特点：
- 消息只存储在声明Queue的节点上
- 其他节点通过路由转发访问消息
- 优点：配置简单
- 缺点：Queue所在节点宕机后，该Queue的消息不可用

## 2.2镜像队列（经典队列镜像）
镜像队列通过将Queue的数据复制到集群中的多个节点来提供高可用性。

```
主节点A（Master）       从节点B（Mirror）       从节点C（Mirror）
 ├── Queue1              ├── Queue1（副本）       ├── Queue1（副本）
 │   └── 消息1..N        │   └── 消息1..N         │   └── 消息1..N
└── 生产者/消费者        └── 只读副本             └── 只读副本
```

特点：
- 主节点处理所有读写操作
- 从节点同步复制主节点的消息，但不同步消费者的消费状态
- 主节点宕机，最老的从节点自动升级为主节点

> **注意**：从RabbitMQ 3.12开始，经典镜像队列已被标记为deprecated，推荐使用仲裁队列替代。

## 2.3仲裁队列（Quorum Queue，RabbitMQ 3.8+）
仲裁队列是RabbitMQ 3.8引入的新型队列，基于Raft共识算法实现，旨在替代经典镜像队列。

```
节点A（Leader）        节点B（Follower）        节点C（Follower）
 ├── Queue1              ├── Queue1（副本）        ├── Queue1（副本）
 │   └── 消息1..N        │   └── 消息1..N          │   └── 消息1..N
└── 处理所有读写         └── 参与Raft共识          └── 参与Raft共识
```

仲裁队列的核心特性：
- **Raft共识**：所有写入操作需要在多数节点上确认（quorum）
- **自动故障转移**：Leader宕机后自动选举新Leader
- **数据安全**：消息持久化并复制到多个节点
- **强一致性**：保证已确认的消息不会丢失

仲裁队列的限制：
- 不支持消息优先级
- 不支持每条消息独立TTL
- 不支持排他队列
- 仅支持`drop-head`溢出行为，不支持`reject-publish`

## 2.4三种模式的对比
| 特性 | 普通集群 | 镜像队列 | 仲裁队列 |
|------|---------|---------|---------|
| 消息持久化 | 仅本地 | 主+镜像 | Raft复制（多节点） |
| 高可用 | 否 | 是 | 是 |
| 一致性 | 无保证 | 最终一致 | 强一致 |
| 故障恢复 | 手动 | 自动（最老镜像升主） | 自动（Leader选举） |
| 性能 | 高 | 中（同步开销） | 中高 |
| 复杂度 | 低 | 中 | 中 |
| 推荐度 | 非高可用场景 | 3.12后被废弃，不推荐新项目 | 推荐（高可用场景） |

# 3.Spring AMQP的集群配置

## 3.1配置集群地址
```yaml
spring:
  rabbitmq:
    addresses: 192.168.1.100:5672,192.168.1.101:5672,192.168.1.102:5672
    username: admin
    password: admin
    virtual-host: /
```

或通过代码：
```java
@Bean
public ConnectionFactory connectionFactory() {
    CachingConnectionFactory factory = new CachingConnectionFactory();
    factory.setAddresses("192.168.1.100:5672,192.168.1.101:5672,192.168.1.102:5672");
    factory.setUsername("admin");
    factory.setPassword("admin");
    
    // 启用shuffle（打乱地址顺序，防止多个客户端同时连接同一节点）
    factory.setAddressShuffleMode(CachingConnectionFactory.AddressShuffleMode.RANDOM);
    
    return factory;
}
```

## 3.2连接恢复配置
集群中节点可能宕机，连接恢复机制至关重要：

```java
@Bean
public ConnectionFactory connectionFactory() {
    CachingConnectionFactory factory = new CachingConnectionFactory();
    factory.setAddresses("192.168.1.100:5672,192.168.1.101:5672,192.168.1.102:5672");
    
    // 连接恢复间隔（毫秒）
    factory.setConnectionRecoveryInterval(10000);  // 10秒
    // RecoveryCallbackAuto（自动恢复连接，默认true）
    // factory.setRecoveryListener(...)  // 监听恢复事件
    
    return factory;
}
```

## 3.3实现自动故障转移
```java
@Bean
public ConnectionFactory connectionFactory() {
    CachingConnectionFactory factory = new CachingConnectionFactory();
    
    // 多个Broker地址
    factory.setAddresses("node1:5672,node2:5672,node3:5672");
    
    // 随机打乱地址顺序，避免所有客户端同时连接同一个节点
    factory.setAddressShuffleMode(CachingConnectionFactory.AddressShuffleMode.RANDOM);
    
    // 设置连接重试间隔
    factory.setConnectionRecoveryInterval(5000);
    
    // 添加连接恢复监听器
    factory.addConnectionListener(new ConnectionListener() {
        @Override
        public void onCreate(Connection connection) {
            System.out.println("连接已建立: " + connection);
        }
        
        @Override
        public void onClose(Connection connection) {
            System.out.println("连接已断开: " + connection + "，将自动重连...");
        }
    });
    
    return factory;
}
```

# 4.仲裁队列的配置与使用

## 4.1声明仲裁队列
```java
// 方式一：QueueBuilder
@Bean
public Queue quorumQueue() {
    return QueueBuilder.durable("important.queue")
        .quorum()  // 声明为仲裁队列
        .withArgument("x-quorum-initial-group-size", 3)  // 初始节点数
        .build();
}

// 方式二：通过arguments
@Bean
public Queue quorumQueue2() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-queue-type", "quorum");
    args.put("x-quorum-initial-group-size", 3);
    return new Queue("important.queue", true, false, false, args);
}
```

## 4.2仲裁队列的最佳实践
```java
@Bean
public Queue criticalBusinessQueue() {
    return QueueBuilder.durable("critical.business.queue")
        .quorum()
        // 初始组大小：建议与集群节点数相同
        .withArgument("x-quorum-initial-group-size", 3)
        // 最大内存字节数（仲裁队列的内存管理特殊）
        .withArgument("max-length-bytes", 100 * 1024 * 1024)  // 100MB
        // 死信配置
        .deadLetterExchange("dlx.critical.exchange")
        .deadLetterRoutingKey("dlx.critical")
        .build();
}
```

# 5.高可用相关机制

## 5.1连接恢复（Connection Recovery）
当Broker节点宕机或网络断开时，Spring AMQP会自动尝试重新连接：

```java
@Bean
public CachingConnectionFactory connectionFactory() {
    CachingConnectionFactory factory = new CachingConnectionFactory();
    // ...基本配置...
    
    // 启用自动恢复（默认true）
    // 它会恢复：Connection、Channel、ShutdownListener、ConfirmListener等
    
    // 恢复间隔
    factory.setConnectionRecoveryInterval(5000);
    
    return factory;
}
```

恢复机制的触发条件：
- Broker连接异常断开
- 网络故障
- Broker节点重启

恢复后自动重建的内容：
- Connection
- Channel（包括事务状态）
- ShutdownListener
- ConfirmListener（Publisher Confirm）
- ReturnListener（Return Callback）

## 5.2消费者自动恢复
当Broker节点故障后恢复，消费者会自动重新订阅队列：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        # 以下配置影响消费者自动恢复行为
        auto-startup: true          # 默认自动启动
        missing-queues-fatal: false  # 队列不存在时不阻止启动
```

## 5.3Exchange/Queue的HA配置
虽然Spring AMQP的配置主要在应用端，但Exchange和Queue的HA属性需要在RabbitMQ Server端配置：

```bash
# 通过rabbitmqctl设置策略
# 将匹配 "^order\." 的队列设置为仲裁队列
rabbitmqctl set_policy --priority 1 --apply-to queues quorum-policy "^order\." '{"queue-type":"quorum"}'
```

## 5.4RabbitMQ的Policy管理
Policy是RabbitMQ Server端的配置策略，可以批量配置队列和交换机的属性：

```bash
# 设置HA策略（镜像队列，已废弃）
# rabbitmqctl set_policy ha-all "^important\." \
#   '{"ha-mode":"all","ha-sync-mode":"automatic"}'

# 设置仲裁队列策略（推荐）
rabbitmqctl set_policy quorum-queue "^important\." \
  '{"queue-type":"quorum","x-quorum-initial-group-size":3}'

# 设置消息TTL策略
rabbitmqctl set_policy ttl-policy "^temp\." \
  '{"message-ttl":60000}'

# 设置最大长度策略
rabbitmqctl set_policy max-length-policy "^bounded\." \
  '{"max-length":10000,"overflow":"reject-publish"}'
```

# 6.集群监控与运维

## 6.1Spring Actuator集成
```yaml
management:
  endpoint:
    health:
      show-details: always
  endpoints:
    web:
      exposure:
        include: health,metrics

spring:
  rabbitmq:
    host: localhost
    port: 5672
```

Spring Boot Actuator会自动检测RabbitMQ连接健康状态，通过`/actuator/health`端点查看。

## 6.2关键监控指标
通过RabbitMQ管理API获取的集群关键指标：

| 指标 | 说明 | 告警阈值建议 |
|------|------|------------|
| node_memory_used | 节点内存使用量 | > 80% |
| node_disk_free | 节点磁盘剩余空间 | < 2GB |
| queue_messages_ready | 队列消息堆积数 | > 10000 |
| queue_messages_unacknowledged | 未确认消息数 | > 5000 |
| connection_count | 连接数 | > 1000 |
| channel_count | 信道数 | > 5000 |
| message_rates_publish | 发布速率 | 关注突变 |
| message_rates_deliver | 投递速率 | 关注突变 |

## 6.3编程式获取集群信息
```java
@Component
public class ClusterStatusMonitor {

    @Autowired
    private ConnectionFactory connectionFactory;

    /**
     * 通过RabbitMQ Management HTTP API获取集群状态
     */
    public void checkClusterStatus() {
        // RabbitMQ Management API
        // GET /api/overview        - 集群概览
        // GET /api/nodes           - 节点列表
        // GET /api/queues          - 队列列表
        // GET /api/exchanges       - 交换机列表
        // GET /api/connections     - 连接列表
        
        // 可以使用RestTemplate或WebClient调用这些API
    }

    /**
     * 检查本地连接状态
     */
    public boolean isConnectionActive() {
        try {
            Connection connection = connectionFactory.createConnection();
            boolean isOpen = connection.isOpen();
            connection.close();
            return isOpen;
        } catch (Exception e) {
            return false;
        }
    }
}
```

# 7.集群最佳实践

## 7.1部署建议
- **奇数节点**：使用3、5或7个节点，避免脑裂（split-brain）
- **独立磁盘**：不同节点使用不同的物理磁盘，避免单点故障
- **网络要求**：集群节点间需要低延迟、高带宽的内部网络
- **负载均衡**：使用HAProxy/Nginx/Cloud Load Balancer进行连接负载均衡

## 7.2配置建议
```bash
# 生产环境推荐配置
# 1. 内存限制
vm_memory_high_watermark.relative = 0.4  # 内存使用上限40%

# 2. 磁盘限制
disk_free_limit.absolute = 2GB           # 磁盘剩余少于2GB时阻塞生产者

# 3. 心跳超时
heartbeat = 60                           # 心跳60秒

# 4. 日志与持久化
log.file.level = info                    # 日志级别
queue_index_embed_msgs_below = 4096      # 小于4KB的消息嵌入索引（提升性能）
```

## 7.3Spring AMQP的集群配置最佳实践
```java
@Configuration
public class ClusterConfig {

    @Bean
    public ConnectionFactory connectionFactory() {
        CachingConnectionFactory factory = new CachingConnectionFactory();
        
        // 1. 多个节点地址
        factory.setAddresses("node1:5672,node2:5672,node3:5672");
        
        // 2. 随机连接（负载均衡）
        factory.setAddressShuffleMode(
            CachingConnectionFactory.AddressShuffleMode.RANDOM);
        
        // 3. 连接恢复间隔
        factory.setConnectionRecoveryInterval(5000);
        
        // 4. Channel缓存大小
        factory.setChannelCacheSize(50);
        
        return factory;
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        
        // 5. 发送重试
        RetryTemplate retryTemplate = RetryTemplate.builder()
            .maxAttempts(3)
            .exponentialBackoff(1000, 2, 10000)
            .build();
        template.setRetryTemplate(retryTemplate);
        
        return template;
    }
}
```
