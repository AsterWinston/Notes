# 1.RabbitMQ环境搭建

## 1.1RabbitMQ简介
RabbitMQ是由Rabbit Technologies开发、后被VMware收购的开源消息中间件，基于Erlang/OTP平台构建。它完整实现了AMQP 0-9-1协议，并支持AMQP 1.0（通过插件）、STOMP、MQTT等多种消息协议。

RabbitMQ本身不提供持久化存储（数据存储委托给了内部集成的Mnesia或外部数据库），其核心功能是**消息的路由和传递**。

RabbitMQ的核心优势：
- **可靠性**：支持消息持久化、发送确认、消费确认等机制，确保消息不丢失
- **灵活路由**：支持Direct、Fanout、Topic、Headers四种Exchange类型
- **高可用**：支持集群部署、镜像队列（Classic Queue Mirroring）、仲裁队列（Quorum Queue）
- **多协议支持**：除了AMQP，还支持STOMP、MQTT、HTTP等协议
- **管理界面**：提供完善的Web管理界面，方便运维管理
- **插件体系**：丰富的插件生态，如延迟队列插件、追踪插件等
- **多语言客户端**：支持Java、Python、Go、.NET、Node.js等多种语言的官方客户端

## 1.2RabbitMQ的核心概念
在学习Spring AMQP之前，需要理解RabbitMQ的几个核心概念：

| 概念 | 说明 |
|------|------|
| Broker | 消息中间件服务实例，即RabbitMQ Server |
| Virtual Host（vhost） | 虚拟主机，逻辑隔离单元。每个vhost拥有独立的Exchange、Queue、Binding、权限。默认vhost为`/` |
| Connection | TCP长连接，由应用程序（客户端）与Broker建立 |
| Channel | 信道，TCP连接中的虚拟子连接，AMQP命令都在Channel上执行 |
| Producer | 消息生产者，发送消息到Exchange |
| Consumer | 消息消费者，从Queue中获取消息 |
| Exchange | 交换机，接收消息并根据Routing Key和Binding来路由 |
| Queue | 队列，消息的存储载体，直到被消费者取走或过期 |
| Routing Key | 路由键，Producer发送消息时指定 |
| Binding Key | 绑定键，绑定Exchange和Queue时指定 |

## 1.3安装RabbitMQ

### 1.3.1下载和安装Erlang
RabbitMQ基于Erlang开发，需要先安装Erlang运行环境：

1. 从Erlang官网下载对应操作系统的安装包：https://www.erlang.org/downloads
2. 按向导完成安装
3. 配置环境变量：`ERLANG_HOME`指向Erlang安装目录
4. 将`%ERLANG_HOME%\bin`添加到PATH中
5. 验证安装：`erl -version`

### 1.3.2下载和安装RabbitMQ
1. 从RabbitMQ官网下载：https://www.rabbitmq.com/download.html
2. 按向导完成安装（Windows下选择exe安装包）
3. 安装完成后，RabbitMQ会作为Windows服务自动启动

### 1.3.3启用管理插件
RabbitMQ提供了一个Web管理界面（Management Plugin），需要手动启用：

```bash
# 进入RabbitMQ的sbin目录
cd "C:\Program Files\RabbitMQ Server\rabbitmq_server-3.13.x\sbin"

# 启用管理插件
rabbitmq-plugins enable rabbitmq_management
```

启用后，访问 http://localhost:15672
- 默认用户名：`guest`
- 默认密码：`guest`

> 注意：`guest`用户默认只允许localhost登录，远程访问需要创建新用户。

### 1.3.4Docker安装（推荐开发环境使用）
使用Docker安装最为便捷：

```bash
# 拉取带管理插件的RabbitMQ镜像
docker pull rabbitmq:3.13-management

# 启动容器
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \        # AMQP协议端口
  -p 15672:15672 \      # Web管理界面端口
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=admin \
  rabbitmq:3.13-management
```

## 1.4RabbitMQ管理界面介绍
登录管理界面后，主要的Tab页面如下：

| Tab | 功能 |
|-----|------|
| Overview | 总览页面，显示消息速率、连接数、队列数、系统资源等 |
| Connections | 查看和管理所有TCP连接 |
| Channels | 查看和管理所有信道 |
| Exchanges | 查看和管理所有交换机，可手动创建Exchange，手动发送消息 |
| Queues | 查看和管理所有队列，可手动创建Queue，查看消息，清空队列 |
| Admin | 用户管理、vhost管理、权限设置、策略管理 |

## 1.5RabbitMQ核心端口
| 端口 | 协议 | 说明 |
|------|------|------|
| 5672 | AMQP 0-9-1（不带TLS） | Java客户端默认连接端口 |
| 5671 | AMQP 0-9-1（带TLS） | 加密连接端口 |
| 15672 | HTTP | Web管理界面端口 |
| 25672 | Erlang分发 | 集群节点间通信端口 |
| 61613 | STOMP | STOMP协议端口 |
| 1883 | MQTT | MQTT协议端口 |

# 2.Spring AMQP快速入门

## 2.1创建Spring Boot项目
使用Spring Initializr或IDEA的脚手架创建一个Spring Boot项目，添加以下依赖：

- **Spring for RabbitMQ**：Spring AMQP的核心依赖

Maven依赖如下（Spring Boot会自动管理版本）：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

`spring-boot-starter-amqp`会自动引入：
- `spring-amqp`：AMQP核心抽象
- `spring-rabbit`：RabbitMQ实现
- `spring-messaging`：Spring消息抽象

## 2.2配置RabbitMQ连接
在`application.yml`中配置RabbitMQ连接信息：

```yaml
spring:
  rabbitmq:
    host: localhost          # RabbitMQ服务器地址
    port: 5672               # AMQP端口
    username: guest          # 用户名
    password: guest          # 密码
    virtual-host: /          # 虚拟主机
    # 更多可选配置
    connection-timeout: 30000      # 连接超时（毫秒）
    requested-heartbeat: 60        # 心跳间隔（秒）
    publisher-confirm-type: correlated   # 发送确认类型
    publisher-returns: true        # 开启发送回退
```

## 2.3编写第一个Spring AMQP应用
完整演示：生产者发送消息，消费者接收消息。

### 2.3.1配置类（声明Queue、Exchange、Binding）
```java
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    public static final String QUEUE_NAME = "hello.queue";
    public static final String EXCHANGE_NAME = "hello.exchange";
    public static final String ROUTING_KEY = "hello.routing";

    // 1. 声明队列
    @Bean
    public Queue helloQueue() {
        // durable: 是否持久化（服务重启后队列依然存在）
        return new Queue(QUEUE_NAME, true);
    }

    // 2. 声明交换机
    @Bean
    public DirectExchange helloExchange() {
        return new DirectExchange(EXCHANGE_NAME, true, false);
    }

    // 3. 绑定
    @Bean
    public Binding helloBinding() {
        return BindingBuilder.bind(helloQueue())
                .to(helloExchange())
                .with(ROUTING_KEY);
    }
}
```

### 2.3.2生产者（发送消息）
```java
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class MessageProducer {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @GetMapping("/send")
    public String sendMessage(@RequestParam String msg) {
        rabbitTemplate.convertAndSend(
                RabbitMQConfig.EXCHANGE_NAME,
                RabbitMQConfig.ROUTING_KEY,
                msg
        );
        return "消息发送成功: " + msg;
    }
}
```

### 2.3.3消费者（接收消息）
```java
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
public class MessageConsumer {

    @RabbitListener(queues = "hello.queue")
    public void receiveMessage(String message) {
        System.out.println("接收到消息: " + message);
        // 实际业务逻辑处理
    }
}
```

### 2.3.4启动类
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class RabbitMQApplication {
    public static void main(String[] args) {
        SpringApplication.run(RabbitMQApplication.class, args);
    }
}
```

### 2.3.5测试
1. 启动Spring Boot应用
2. 访问 http://localhost:8080/send?msg=HelloRabbitMQ
3. 控制台输出：`接收到消息: HelloRabbitMQ`

## 2.4Spring Boot自动配置解析
Spring Boot对RabbitMQ的自动配置由`RabbitAutoConfiguration`类完成，主要做了以下事情：

1. **自动创建ConnectionFactory**：根据`spring.rabbitmq.*`配置自动创建`CachingConnectionFactory`
2. **自动创建RabbitTemplate**：创建一个`RabbitTemplate` Bean并注入容器
3. **自动创建RabbitAdmin**：创建一个`RabbitAdmin` Bean，用于自动声明Exchange、Queue、Binding
4. **启用@RabbitListener**：通过`@EnableRabbit`自动启用注解驱动的消息监听

这意味着，在标准情况下，你只需要：
- 配置`spring.rabbitmq.*`连接信息
- 编写`@RabbitListener`或使用`RabbitTemplate`
- Spring Boot会帮你完成其余所有配置

## 2.5RabbitTemplate的两种获取方式

### 2.5.1使用Spring Boot自动配置的RabbitTemplate（推荐）
```java
@Autowired
private RabbitTemplate rabbitTemplate;
```

### 2.5.2自定义RabbitTemplate
如果需要对RabbitTemplate进行特定配置（如自定义MessageConverter、ConfirmCallback等），可以手动创建：

```java
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
    // 自定义配置...
    return rabbitTemplate;
}
```

## 2.6RabbitAdmin的作用
`RabbitAdmin`的核心职责是**自动声明AMQP实体**（Exchange、Queue、Binding）。

当Spring容器启动时，RabbitAdmin会：
1. 扫描容器中所有Exchange、Queue、Binding类型的Bean
2. 在Broker上自动声明（declare）这些组件
3. 如果组件已存在且属性匹配，则跳过；如果属性不匹配，默认会抛出异常

```java
// 以下Bean会被RabbitAdmin自动声明
@Bean
public Queue myQueue() {
    return new Queue("my.queue", true);
}
// 应用启动后，Broker上会自动创建 my.queue 这个队列
```

RabbitAdmin在底层通过`declareExchanges`、`declareQueues`、`declareBindings`等AMQP命令与Broker交互。

# 3.配置文件详解

## 3.1核心连接配置
```yaml
spring:
  rabbitmq:
    # 基本连接信息
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
    addresses: localhost:5672    # 集群地址，多个用逗号分隔
    
    # 连接超时
    connection-timeout: 30000      # 连接超时，默认30秒
    
    # SSL支持
    ssl:
      enabled: false               # 是否启用SSL
      algorithm: TLSv1.2           # SSL算法
      key-store: classpath:client.jks
      key-store-password: password
      trust-store: classpath:truststore.jks
      trust-store-password: password
```

## 3.2缓存配置
```yaml
spring:
  rabbitmq:
    cache:
      channel:
        size: 25                   # 缓存的Channel数量
        checkout-timeout: 10000    # 等待获取Channel的超时（毫秒）
      connection:
        mode: channel              # 缓存模式：channel（默认）或 connection
        size: 1                    # 缓存的Connection数量（仅connection模式有效）
```

关于缓存模式：
- `channel`模式（默认）：维护一个共享的Connection，缓存多个Channel。适合大多数场景
- `connection`模式：维护多个独立的Connection（每个Connection一个Channel）。适合需要严格隔离的场景（如不同的Virtual Host、不同的SSL证书）

## 3.3消息发送配置
```yaml
spring:
  rabbitmq:
    # 确认与回退
    publisher-confirm-type: correlated   # 发送确认类型
    publisher-returns: true              # 开启发送回退
    
    # 模板配置
    template:
      mandatory: true                    # 强制投递（无匹配Queue时回调ReturnCallback）
      receive-timeout: 10000             # 同步接收超时（毫秒）
      reply-timeout: 5000                # 回复超时（毫秒）
      retry:
        enabled: false                   # 是否启用重试
        initial-interval: 1000           # 初始重试间隔
        max-attempts: 3                  # 最大重试次数
        max-interval: 10000              # 最大重试间隔
        multiplier: 2.0                  # 间隔乘数
```

## 3.4消费者配置
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        concurrency: 1                   # 最小消费者线程数
        max-concurrency: 10              # 最大消费者线程数
        prefetch: 250                    # 每个消费者预取数量
        acknowledge-mode: auto           # 确认模式：none/auto/manual
        default-requeue-rejected: true   # 被拒绝的消息是否重新入队
        retry:
          enabled: false                 # 是否启用消费者重试
          initial-interval: 1000         # 初始间隔
          max-attempts: 3                # 最大重试次数
          multiplier: 2.0                # 间隔乘数
          max-interval: 10000            # 最大间隔
      direct:
        # DirectMessageListenerContainer 的配置
        consumers-per-queue: 5           # 每个队列的消费者数量
```

# 4.Maven依赖详解

## 4.1核心依赖
```xml
<!-- Spring Boot AMQP Starter（推荐，会自动管理版本） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

## 4.2不使用Spring Boot时的依赖管理
如果不使用Spring Boot，需要手动引入Spring AMQP：

```xml
<dependencies>
    <!-- AMQP核心抽象 -->
    <dependency>
        <groupId>org.springframework.amqp</groupId>
        <artifactId>spring-amqp</artifactId>
        <version>3.1.0</version>
    </dependency>
    
    <!-- RabbitMQ实现 -->
    <dependency>
        <groupId>org.springframework.amqp</groupId>
        <artifactId>spring-rabbit</artifactId>
        <version>3.1.0</version>
    </dependency>
</dependencies>
```

# 5.测试环境搭建
Spring AMQP提供了测试支持，可以方便地进行集成测试：

```xml
<dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-rabbit-test</artifactId>
    <scope>test</scope>
</dependency>
```

测试工具类：
- `TestRabbitTemplate`：专门用于测试的RabbitTemplate
- `RabbitListenerTestHarness`：用于测试@RabbitListener方法
- `@RabbitListenerTest`：标记测试配置

# 6.版本兼容性
| Spring Boot | Spring AMQP | RabbitMQ Erlang Client | AMQP Client | Java |
|-------------|-------------|------------------------|-------------|------|
| 3.3.x | 3.1.x | 5.20.x | 5.20.x | 17+ |
| 3.2.x | 3.1.x | 5.18.x | 5.18.x | 17+ |
| 3.1.x | 3.0.x | 5.17.x | 5.17.x | 17+ |
| 2.7.x | 2.4.x | 5.14.x | 5.14.x | 8+ |
| 2.6.x | 2.3.x | 5.13.x | 5.13.x | 8+ |
