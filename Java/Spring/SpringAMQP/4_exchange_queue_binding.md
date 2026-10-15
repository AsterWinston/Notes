# 1. Exchange交换机详解

## 1.1 Exchange的职责
Exchange是AMQP协议中的消息路由组件，它的核心职责是**接收生产者发来的消息，根据Routing Key和Binding规则，将消息路由到一个或多个Queue中**。

重要的设计原则：
- **消息永远不会直接发送到Queue**，必须先经过Exchange
- **Exchange不存储消息**，它只是一个路由表，消息存储由Queue负责
- 如果Exchange无法将消息路由到任何Queue（没有匹配的Binding），消息默认会被**丢弃**

## 1.2 Direct Exchange深入

### 1.2.1 工作原理
Direct Exchange采用**精确匹配**策略，将Routing Key与Binding Key进行完全相等的比较。

```
生产者: exchange = "order.exchange", routingKey = "order.create"
                    ↓
Direct Exchange: 检查所有Binding
  ├── Binding: "order.create" → queue.order.creation   ✓ 匹配
  └── Binding: "order.pay"    → queue.order.payment    ✗ 不匹配
                    ↓
消息投递到: queue.order.creation
```

### 1.2.2 Spring声明
```java
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DirectExchangeConfig {

    @Bean
    public DirectExchange orderDirectExchange() {
        // 构造参数：名称、是否持久化、是否自动删除、其他参数
        return new DirectExchange("order.direct.exchange", true, false);
    }

    @Bean
    public Queue orderCreateQueue() {
        return new Queue("order.create.queue", true);
    }

    @Bean
    public Queue orderCancelQueue() {
        return new Queue("order.cancel.queue", true);
    }

    @Bean
    public Binding orderCreateBinding() {
        return BindingBuilder.bind(orderCreateQueue())
                .to(orderDirectExchange())
                .with("order.create");
    }

    @Bean
    public Binding orderCancelBinding() {
        return BindingBuilder.bind(orderCancelQueue())
                .to(orderDirectExchange())
                .with("order.cancel");
    }
}
```

### 1.2.3 使用场景
- 明确的业务路由：不同的业务操作路由到不同队列
- 精确的消息分发：每个Routing Key对应唯一的目标队列
- 点对点通信：配合默认交换机实现精确的点对点投递

## 1.3 Topic Exchange深入

### 1.3.1 工作原理
Topic Exchange采用**模式匹配**策略，Binding Key使用`.`分隔的单词和通配符：
- `*`（星号）：匹配**有且仅有**一个单词
- `#`（井号）：匹配**零个或多个**单词

匹配示例：
```
Binding Key        | Routing Key              | 匹配?
-------------------+--------------------------+------
order.*            | order.create             | ✓ (一个单词)
order.*            | order.create.payment     | ✗ (两个单词)
order.#            | order.create             | ✓
order.#            | order.create.payment     | ✓
order.#            | order                    | ✓ (零个单词)
#.create           | order.create             | ✓
#.create           | payment.order.create     | ✓
#.create.#         | order.create.cancel      | ✓
*.create           | order.create             | ✓
*.create           | payment.order.create     | ✗ (两个单词)
```

### 1.3.2 Spring声明
```java
@Configuration
public class TopicExchangeConfig {

    @Bean
    public TopicExchange logTopicExchange() {
        return new TopicExchange("log.topic.exchange", true, false);
    }

    @Bean
    public Queue logErrorQueue() {
        return new Queue("log.error.queue", true);
    }

    @Bean
    public Queue logAllQueue() {
        return new Queue("log.all.queue", true);
    }

    @Bean
    public Queue orderLogQueue() {
        return new Queue("order.log.queue", true);
    }

    @Bean
    public Binding errorLogBinding() {
        return BindingBuilder.bind(logErrorQueue())
                .to(logTopicExchange())
                .with("log.#.error");  // 匹配 log.xxx.error
    }

    @Bean
    public Binding allLogBinding() {
        return BindingBuilder.bind(logAllQueue())
                .to(logTopicExchange())
                .with("log.#");  // 匹配所有 log 开头的消息
    }

    @Bean
    public Binding orderLogBinding() {
        return BindingBuilder.bind(orderLogQueue())
                .to(logTopicExchange())
                .with("order.*");  // 匹配 order.create、order.pay 等
    }
}
```

### 1.3.3 使用场景
- 日志分级处理：`log.info`、`log.warn`、`log.error`分别路由到不同队列
- 事件广播与筛选：`order.#`捕获所有订单事件，`order.create`只关注创建事件
- 多维度路由：`region.us.store.ny`可以有`region.#`、`#.ny`等多种匹配
- 微服务事件驱动：`user.created`、`user.updated`、`user.deleted`区分处理

## 1.4 Fanout Exchange深入

### 1.4.1 工作原理
Fanout Exchange是最简单的交换机类型，它将接收到的消息**广播到所有绑定的Queue**，完全**忽略Routing Key**。

```
生产者: exchange = "notify.fanout.exchange", routingKey = "任意值(忽略)"
                    ↓
Fanout Exchange: 广播到所有绑定的Queue
  ├── queue.email  ← 收到消息
  ├── queue.sms    ← 收到消息
  └── queue.push   ← 收到消息
```

### 1.4.2 Spring声明
```java
@Configuration
public class FanoutExchangeConfig {

    @Bean
    public FanoutExchange notifyFanoutExchange() {
        return new FanoutExchange("notify.fanout.exchange", true, false);
    }

    @Bean
    public Queue emailQueue() {
        return new Queue("notify.email.queue", true);
    }

    @Bean
    public Queue smsQueue() {
        return new Queue("notify.sms.queue", true);
    }

    @Bean
    public Binding emailBinding() {
        // Fanout Binding不需要指定Routing Key
        return BindingBuilder.bind(emailQueue())
                .to(notifyFanoutExchange());
    }

    @Bean
    public Binding smsBinding() {
        return BindingBuilder.bind(smsQueue())
                .to(notifyFanoutExchange());
    }
}
```
注意：`BindingBuilder.bind(queue).to(fanoutExchange)`在绑定到Fanout Exchange时不需要（也不能）指定Routing Key，因为Fanout Exchange会忽略它。

### 1.4.3 使用场景
- 发布/订阅模式：一条消息需要被多个消费者同时处理
- 数据广播：配置变更、缓存刷新通知等
- 事件通知：用户注册后同时触发邮件、短信、推送等

## 1.5 Headers Exchange深入

### 1.5.1 工作原理
Headers Exchange通过**消息的Headers属性**进行路由匹配，完全忽略Routing Key。

```java
// 声明Headers Exchange的Binding
@Bean
public HeadersExchange headersExchange() {
    return new HeadersExchange("headers.exchange", true, false);
}

@Bean
public Queue eastVipQueue() {
    return new Queue("east.vip.queue", true);
}

@Bean
public Binding eastVipBinding() {
    return BindingBuilder.bind(eastVipQueue())
            .to(headersExchange())
            .where("region").matches("east")    // 匹配条件1
            .and("level").matches("vip");       // 匹配条件2
}
```

Headers匹配规则：
- `.where(key).matches(value)` - 精确匹配
- `.whereAll(headers).match()` - 所有头部都匹配
- `.whereAny(headers).match()` - 任意头部匹配

注意：`BindingBuilder`对Headers Exchange的支持有类型限制（需要`HeadersExchange`类型），如果是抽象的`Exchange`接口，需要向下转型。

### 1.5.2 使用场景
Headers Exchange的使用场景相对较少，推荐优先使用Topic Exchange。

## 1.6 Default Exchange（默认交换机）

### 1.6.1 特性
- 名称：`""`（空字符串）
- 类型：Direct
- **自动将所有Queue名称作为Binding Key**
- 无法显式绑定和解绑

```java
// 发送到默认交换机
rabbitTemplate.convertAndSend("my.queue", message);
// 等价于（显式指定Exchange为空字符串）
rabbitTemplate.convertAndSend("", "my.queue", message);
```

### 1.6.2 使用场景
- 快速开发和测试
- 简单的点对点消息传递
- 不需要复杂路由规则的场景

# 2. Queue队列详解

## 2.1 Queue的属性详解
```java
new Queue(
    "queue.name",      // 名称
    true,              // durable：是否持久化
    false,             // exclusive：是否排他
    false,             // autoDelete：是否自动删除
    new HashMap<>()    // arguments：额外参数
);
```

### 2.1.1 durable（持久化）
- `true`：队列的元数据持久化到磁盘，Broker重启后队列依然存在
- `false`：临时队列，Broker重启后队列消失

> 注意：**durable队列不代表消息持久化**。消息的持久化由`deliveryMode`属性控制。如果队列是非持久化的，即使消息设置了持久化，Broker重启后队列消失，消息也会丢失。

### 2.1.2 exclusive（排他性）
- `true`：该队列只能被创建它的Connection使用，Connection断开后队列自动删除
- `false`：多个Connection可以共享该队列

适用场景：
- 临时队列，如RPC回复队列
- 单消费者场景，确保队列不被其他消费者监听

### 2.1.3 autoDelete（自动删除）
- `true`：当最后一个消费者断开连接后，队列自动删除
- `false`：即使没有消费者，队列也继续存在

### 2.1.4 arguments（额外参数）
通过`arguments`可以设置队列的高级特性：

```java
Map<String, Object> args = new HashMap<>();
args.put("x-message-ttl", 60000);              // 消息TTL（毫秒）
args.put("x-max-length", 10000);                // 最大消息数量
args.put("x-max-length-bytes", 104857600);      // 最大队列字节数（100MB）
args.put("x-overflow", "reject-publish");       // 溢出行为：drop-head / reject-publish
args.put("x-dead-letter-exchange", "dlx.exchange"); // 死信交换机
args.put("x-dead-letter-routing-key", "dlx.routing"); // 死信路由键
args.put("x-queue-mode", "lazy");               // 队列模式：default / lazy
args.put("x-queue-type", "quorum");             // 队列类型：classic / quorum（RabbitMQ 3.8+）

Queue queue = new Queue("my.queue", true, false, false, args);
```

## 2.2 临时队列与匿名队列
```java
// 创建匿名排他自动删除队列（名称由Broker生成，形如 amq.gen-xxx）
@Bean
public Queue anonymousQueue() {
    return new AnonymousQueue();
}

// 或者
@Bean
public Queue anonymousQueue() {
    return QueueBuilder.nonDurable().exclusive().autoDelete().build();
}
```
匿名队列适用于RPC模式中的回复队列，因为名称不需要提前约定。

## 2.3 Queue Builder
Spring AMQP提供了`QueueBuilder`，使Queue的创建更加直观：

```java
// 基础队列
Queue queue1 = QueueBuilder.durable("my.queue").build();

// 带参数
Queue queue2 = QueueBuilder.durable("my.queue")
        .withArgument("x-message-ttl", 60000)
        .withArgument("x-dead-letter-exchange", "dlx.exchange")
        .build();

// 排他队列
Queue queue3 = QueueBuilder.durable("my.queue")
        .exclusive()
        .build();

// 自动删除
Queue queue4 = QueueBuilder.durable("my.queue")
        .autoDelete()
        .build();

// 仲裁队列（RabbitMQ 3.8+）
Queue queue5 = QueueBuilder.durable("my.queue")
        .quorum()
        .build();

// 惰性队列
Queue queue6 = QueueBuilder.durable("my.queue")
        .lazy()
        .build();

// TTL队列
Queue queue7 = QueueBuilder.durable("my.queue")
        .ttl(60000)  // 队列级别消息TTL
        .build();
```

## 2.4 Queue参数的底层实现
当Spring AMQP声明Queue时，`QueueBuilder`会将配置转换为AMQP协议的arguments参数表：

| QueueBuilder方法 | AMQP argument | 说明 |
|-----------------|---------------|------|
| `.ttl(n)` | `x-message-ttl` | 队列中消息的默认TTL |
| `.maxLength(n)` | `x-max-length` | 队列最大消息数量 |
| `.maxLengthBytes(n)` | `x-max-length-bytes` | 队列最大字节数 |
| `.overflow()` | `x-overflow` | 溢出行为 |
| `.deadLetterExchange()` | `x-dead-letter-exchange` | 死信交换机 |
| `.deadLetterRoutingKey()` | `x-dead-letter-routing-key` | 死信路由键 |
| `.lazy()` | `x-queue-mode: lazy` | 惰性队列模式 |
| `.quorum()` | `x-queue-type: quorum` | 仲裁队列模式 |
| `.singleActiveConsumer()` | `x-single-active-consumer` | 单活消费者 |
| `.autoDelete()` | `auto_delete` | 自动删除 |
| `.exclusive()` | `exclusive` | 排他 |
| `.durable()` | `durable` | 持久化 |

## 2.5 惰性队列（Lazy Queue）
从RabbitMQ 3.6开始支持惰性队列，**将消息尽可能存放到磁盘**，仅在消费者请求时才加载到内存。

```java
Queue lazyQueue = QueueBuilder.durable("lazy.queue")
        .lazy()
        .build();
```

适用场景：
- 消息量很大，消费者处理速度跟不上
- 需要处理突发流量，允许消息堆积在磁盘上
- 内存有限，不想消息全部驻留在内存

## 2.6 仲裁队列（Quorum Queue）
从RabbitMQ 3.8开始引入的一种新队列类型，基于Raft共识算法实现的持久化、复制式队列。旨在替代传统镜像队列（Mirrored Queue）。

```java
Queue quorumQueue = QueueBuilder.durable("quorum.queue")
        .quorum()
        .build();
```

仲裁队列的特性：
- **高可用**：基于Raft协议，自动故障转移
- **数据安全**：消息持久化到多个节点，确保不丢失
- **一致性**：强一致性保证
- **自动恢复**：节点故障后自动恢复

仲裁队列的限制：
- 不支持消息优先级
- 不支持消息TTL（每个消息独立TTL）
- 不支持排他队列
- 仅支持`drop-head`溢出策略（不支持`reject-publish`）

# 3. Binding绑定详解

## 3.1 Binding的本质
Binding是Exchange和Queue之间的关联关系，定义了消息从Exchange流向Queue的规则：

```
Exchange --[Binding]--> Queue
  ↑                        ↑
接收消息                 存储并投递给消费者
```

## 3.2 Binding Builder的使用
Spring AMQP提供了`BindingBuilder`链式API来创建Binding：

```java
// Direct Exchange 绑定
BindingBuilder.bind(queue).to(directExchange).with("routing.key");

// Topic Exchange 绑定
BindingBuilder.bind(queue).to(topicExchange).with("order.#");

// Fanout Exchange 绑定（不需要Routing Key）
BindingBuilder.bind(queue).to(fanoutExchange);

// Headers Exchange 绑定
BindingBuilder.bind(queue).to(headersExchange)  // 需要HeadersExchange类型
        .where("headerKey").matches("value");

// 多个Queue绑定到同一个Exchange
BindingBuilder.bind(queue1).to(exchange).with("routing.key1");
BindingBuilder.bind(queue2).to(exchange).with("routing.key2");

// 一个Queue绑定到多个Exchange
BindingBuilder.bind(queue).to(exchange1).with("key1");
BindingBuilder.bind(queue).to(exchange2).with("key2");
```

## 3.3 多Binding与路由策略
一个Exchange可以有多个Binding，一个Queue也可以绑定到多个Exchange：

```
        Queue1 ←--[key1]-- ExchangeA --[key2]--> Queue2
           ↑                                          ↓
        [broadcast]                            Queue3
           ↑
      FanoutExchange
           ↑
        [任意key]
```

这种灵活性可以实现复杂的消息路由拓扑。

# 4. 交换机与队列的高级配置

## 4.1 交换机的其他属性

### 4.1.1 内部交换机（Internal）
内部交换机不能直接被生产者发布消息，只能被其他Exchange通过Exchange-to-Exchange绑定来路由消息：

```java
Exchange internalExchange = ExchangeBuilder
    .directExchange("internal.exchange")
    .internal()   // 标记为内部交换机
    .build();
```

### 4.1.2 备用交换机（Alternate Exchange, AE）
当消息无法被路由到任何Queue时，会自动转发到备用交换机：

```java
// 第一种方式：在声明Exchange时通过arguments指定
Map<String, Object> args = new HashMap<>();
args.put("alternate-exchange", "unroutable.exchange");
Exchange exchange = new DirectExchange("order.exchange", true, false, args);

// 第二种方式：使用ExchangeBuilder
Exchange exchange = ExchangeBuilder.directExchange("order.exchange")
        .withArgument("alternate-exchange", "unroutable.exchange")
        .build();
```

备用交换机的工作流程：
```
消息 → order.exchange → 无匹配Binding
  ↓
alternate-exchange → unroutable.exchange → unroutable.queue（捕获无法路由的消息）
```

适用场景：
- 捕获无法路由的消息，避免消息丢失（而不是默默丢弃）
- 便于排查问题（通过监控备用队列中的消息，发现路由问题）
- 实现统一的消息兜底处理

### 4.1.3 Exchange-to-Exchange 绑定
RabbitMQ支持交换机与交换机之间的绑定（Exchange-to-Exchange Binding），允许将消息从一个Exchange转发到另一个Exchange。

```java
// 将 topic.exchange 绑定到 fanout.exchange
// 发送到 topic.exchange 的消息会同时广播到 fanout.exchange 的所有绑定队列
@Bean
public Exchange topicExchange() {
    return new TopicExchange("topic.exchange", true, false);
}

@Bean
public Exchange fanoutExchange() {
    return new FanoutExchange("fanout.exchange", true, false);
}

@Bean
public Binding exchangeToExchangeBinding() {
    return new Binding(
        "fanout.exchange",              // 目标交换机
        Binding.DestinationType.EXCHANGE,
        "topic.exchange",               // 源交换机
        "routing.key",                  // 路由键
        null
    );
}
```

## 4.2 ExchangeBuilder工具类
```java
// Direct
Exchange directExchange = ExchangeBuilder
    .directExchange("direct.exchange")
    .durable(true)
    .build();

// Topic
Exchange topicExchange = ExchangeBuilder
    .topicExchange("topic.exchange")
    .durable(true)
    .autoDelete()
    .build();

// Fanout
Exchange fanoutExchange = ExchangeBuilder
    .fanoutExchange("fanout.exchange")
    .durable(true)
    .build();

// Headers
Exchange headersExchange = ExchangeBuilder
    .headersExchange("headers.exchange")
    .durable(true)
    .build();

// 带参数的Exchange
Exchange exchange = ExchangeBuilder
    .topicExchange("my.topic.exchange")
    .durable(true)
    .delayed()                                    // 延迟交换机（需安装插件）
    .withArgument("alternate-exchange", "unroutable.exchange")
    .build();
```

# 5. 声明式与编程式管理

## 5.1 声明式（通过@Bean）
声明式是Spring AMQP的常规用法，通过配置类中的@Bean方法定义Exchange、Queue、Binding，RabbitAdmin会自动声明它们到Broker。

优点：
- 配置集中管理，便于维护
- 与IoC容器结合，可以注入使用
- 应用启动时自动声明，确保组件存在

## 5.2 编程式（通过AmqpAdmin）
在某些动态场景下，需要在运行时动态创建或删除组件：

```java
@Autowired
private AmqpAdmin amqpAdmin;

// 运行时创建临时队列
public void createTempQueue(String queueName) {
    Queue queue = QueueBuilder.durable(queueName).build();
    amqpAdmin.declareQueue(queue);
}

// 运行时创建临时Exchange和Binding
public void setupTempRouting(String exchangeName, String queueName, String routingKey) {
    Exchange exchange = ExchangeBuilder.directExchange(exchangeName).build();
    Queue queue = new Queue(queueName, true);
    Binding binding = new Binding(queueName, Binding.DestinationType.QUEUE,
            exchangeName, routingKey, null);

    amqpAdmin.declareExchange(exchange);
    amqpAdmin.declareQueue(queue);
    amqpAdmin.declareBinding(binding);
}

// 运行时删除
public void cleanup(String queueName) {
    amqpAdmin.deleteQueue(queueName);
}

// 运行时获取队列信息
public int getPendingMessageCount(String queueName) {
    QueueInformation info = amqpAdmin.getQueueInfo(queueName);
    return info != null ? info.getMessageCount() : 0;
}
```

# 6. 声明错误处理

## 6.1 属性不匹配的处理
当应用尝试声明的组件与Broker上已存在的同名组件属性不一致时，`RabbitAdmin`默认会失败并抛出异常。

解决方案：
1. **忽略声明异常**：`RabbitAdmin.setIgnoreDeclarationExceptions(true)`（不推荐，可能隐藏真正的问题）
2. **删除后重建**：在声明前先删除旧组件（仅开发环境推荐）
3. **使用Policy**：通过RabbitMQ Server端的Policy管理队列参数，应用端只声明队列名称和基本类型

```java
@Bean
public RabbitAdmin rabbitAdmin(ConnectionFactory connectionFactory) {
    RabbitAdmin admin = new RabbitAdmin(connectionFactory);
    // 忽略声明异常（开发环境可用，生产环境不推荐）
    // admin.setIgnoreDeclarationExceptions(true);
    return admin;
}
```

# 7. 开发建议与最佳实践

## 7.1 命名规范
- Exchange命名：`{domain}.{type}.exchange`，如`order.topic.exchange`
- Queue命名：`{domain}.{action}.queue`，如`order.create.queue`
- Routing Key命名：`{domain}.{action}`，如`order.create`
- Dead Letter相关：加前缀`dlx.` / `dlq.`，如`dlx.order.exchange`、`dlq.order.queue`

## 7.2 设计原则
- **一个Queue一个职责**：一个Queue只处理一种类型的业务消息，便于监控和维护
- **Exchange优先Topic类型**：Topic Exchange是最灵活的，能满足大部分路由需求
- **合理使用备用交换机**：在关键业务Exchange上配置备用交换机，捕获无法路由的消息便于排查
- **避免环形路由**：不要在交换机之间建立循环的路由关系
- **最小权限原则**：为不同服务配置不同的Virtual Host和用户权限
