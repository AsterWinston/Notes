# 1. 消息转换概述

## 1.1 为什么需要MessageConverter
在AMQP协议中，消息体是**二进制的字节数组**（`byte[]`），而Java应用程序中使用的是**对象**。`MessageConverter`的职责就是在这两者之间进行转换：

```
Java对象  ←→  MessageConverter  ←→  byte[]（AMQP消息体）
```

## 1.2 Spring AMQP的转换体系
```
MessageConverter（接口）
  ├── SimpleMessageConverter（默认实现）
  ├── Jackson2JsonMessageConverter（JSON转换，最常用）
  ├── Jackson2XmlMessageConverter（XML转换）
  ├── MarshallingMessageConverter（Spring OXM绑定）
  ├── SerializerMessageConverter（Kryo/Protobuf等外部序列化）
  ├── ContentTypeDelegatingMessageConverter（根据ContentType选择转换器）
  └── CompositeMessageConverter（组合多个转换器尝试）
```

# 2. SimpleMessageConverter（默认实现）

## 2.1 工作原理
`SimpleMessageConverter`是Spring AMQP的**默认**消息转换器，它按以下规则进行转换：

### 2.1.1 发送时（toMessage）
```java
SimpleMessageConverter converter = new SimpleMessageConverter();

// 1. 如果对象是 byte[] → 直接使用
byte[] data = "hello".getBytes();
Message msg1 = converter.toMessage(data, new MessageProperties());

// 2. 如果对象是 String → 转换为 byte[]（默认UTF-8）
String text = "hello";
Message msg2 = converter.toMessage(text, new MessageProperties());

// 3. 如果对象是 Serializable → 使用Java序列化
Order order = new Order();
Message msg3 = converter.toMessage(order, new MessageProperties());
```

### 2.1.2 接收时（fromMessage）
```java
// 根据MessageProperties中的contentType决定：
// 1. contentType 以 "text" 开头 → 返回 String
// 2. contentType = "application/x-java-serialized-object" → 反序列化
// 3. 其他 → 直接返回 byte[]
```

## 2.2 SimpleMessageConverter的问题
默认的`SimpleMessageConverter`有以下不足：

1. **Java序列化问题**：
   - 可读性差（二进制数据，无法直接在RabbitMQ管理界面查看）
   - 跨语言不友好（其他语言无法反序列化Java序列化的数据）
   - 安全性问题（反序列化漏洞）
   - 性能较差

2. **contentType判断不灵活**：默认只区分text和Java序列化

因此，生产环境强烈建议替换为JSON转换器。

# 3. Jackson2JsonMessageConverter（推荐）

## 3.1 基本配置
```java
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    @Bean
    public Jackson2JsonMessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

配置后，`Jackson2JsonMessageConverter`会自动注入到`RabbitTemplate`和`MessageListenerContainer`中。

## 3.2 工作原理

### 3.2.1 发送时
```java
// 自动将Order对象转换为JSON，设置contentType=application/json
rabbitTemplate.convertAndSend("order.exchange", "order.create", order);

// 实际发送的Message结构：
// body: {"id":1,"name":"测试订单","amount":99.99}
// contentType: application/json
// contentEncoding: UTF-8
```

### 3.2.2 接收时
```java
@RabbitListener(queues = "order.queue")
public void handleOrder(Order order) {
    // 自动将JSON反序列化为Order对象
    // 根据方法的参数类型决定目标类型
}
```

## 3.3 自定义ObjectMapper
```java
@Bean
public Jackson2JsonMessageConverter messageConverter(ObjectMapper objectMapper) {
    // 使用自定义的ObjectMapper配置
    Jackson2JsonMessageConverter converter = new Jackson2JsonMessageConverter(objectMapper);
    return converter;
}

@Bean
public ObjectMapper objectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    
    // 启用/禁用特定功能
    mapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
    mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    mapper.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);
    
    // 注册JavaTimeModule（支持Java 8日期时间）
    mapper.registerModule(new JavaTimeModule());
    mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    
    // 设置时区
    mapper.setTimeZone(TimeZone.getTimeZone("Asia/Shanghai"));
    
    return mapper;
}
```

## 3.4 信任包配置（安全相关）
JSON反序列化时需要指定信任的包，防止反序列化漏洞：

```java
@Bean
public Jackson2JsonMessageConverter messageConverter() {
    Jackson2JsonMessageConverter converter = new Jackson2JsonMessageConverter();
    
    // 方式一：信任所有包（不推荐）
    // converter.getObjectMapper().enableDefaultTyping();
    
    // 方式二：信任指定包（推荐）
    DefaultJackson2JavaTypeMapper typeMapper = new DefaultJackson2JavaTypeMapper();
    typeMapper.setTypePrecedence(Jackson2JavaTypeMapper.TypePrecedence.TYPE_ID);
    converter.setJavaTypeMapper(typeMapper);
    
    return converter;
}
```

## 3.5 类型ID与类型头
当消息需要在多个服务之间传递时，接收端需要知道消息的具体类型。Jackson2JsonMessageConverter通过在消息头中添加`__TypeId__`来传递类型信息：

```java
// 发送端：在MessageProperties的headers中自动添加
// __TypeId__: com.example.entity.Order

// 接收端：根据__TypeId__自动反序列化为对应类型
@RabbitListener(queues = "order.queue")
public void handleOrder(Order order) {
    // 如果方法参数类型与__TypeId__一致，直接使用
    // 如果不一致，需要配置ClassMapper来处理映射关系
}
```

## 3.6 ClassMapper配置
在微服务架构中，发送端和接收端可能使用不同的包名，需要配置ClassMapper进行映射：

```java
@Bean
public Jackson2JsonMessageConverter messageConverter() {
    Jackson2JsonMessageConverter converter = new Jackson2JsonMessageConverter();
    
    DefaultJackson2JavaTypeMapper typeMapper = new DefaultJackson2JavaTypeMapper();
    
    // 配置ID-Class映射
    Map<String, Class<?>> idClassMapping = new HashMap<>();
    idClassMapping.put("order", Order.class);
    idClassMapping.put("orderCreated", OrderCreatedEvent.class);
    idClassMapping.put("user", User.class);
    typeMapper.setIdClassMapping(idClassMapping);
    
    // 配置Class-ID映射
    Map<Class<?>, String> classIdMapping = new HashMap<>();
    classIdMapping.put(Order.class, "order");
    classIdMapping.put(OrderCreatedEvent.class, "orderCreated");
    typeMapper.setClassIdMapping(classIdMapping);
    
    converter.setJavaTypeMapper(typeMapper);
    return converter;
}
```

发送消息时可以指定类型ID：
```java
rabbitTemplate.convertAndSend("order.exchange", "order.create", order,
    message -> {
        message.getMessageProperties().setHeader("__TypeId__", "order");
        return message;
    });
```

# 4. 其他MessageConverter

## 4.1 Jackson2XmlMessageConverter
使用Jackson将消息转换为XML格式：

```java
@Bean
public Jackson2XmlMessageConverter xmlMessageConverter() {
    return new Jackson2XmlMessageConverter();
}

// 发送的消息体：
// <Order><id>1</id><name>测试订单</name></Order>
```

## 4.2 ContentTypeDelegatingMessageConverter
根据消息的`contentType`属性选择不同的转换器：

```java
@Bean
public ContentTypeDelegatingMessageConverter messageConverter() {
    ContentTypeDelegatingMessageConverter converter = new ContentTypeDelegatingMessageConverter();
    
    // 注册不同ContentType对应的转换器
    converter.addDelegate("application/json", new Jackson2JsonMessageConverter());
    converter.addDelegate("application/xml", new Jackson2XmlMessageConverter());
    converter.addDelegate("text/plain", new SimpleMessageConverter());
    
    return converter;
}
```

## 4.3 CompositeMessageConverter
组合多个转换器，逐个尝试直到成功：

```java
@Bean
public CompositeMessageConverter messageConverter() {
    List<MessageConverter> converters = new ArrayList<>();
    converters.add(new Jackson2JsonMessageConverter());
    converters.add(new Jackson2XmlMessageConverter());
    converters.add(new SimpleMessageConverter());
    
    CompositeMessageConverter converter = new CompositeMessageConverter();
    converter.setDelegates(converters);
    return converter;
}
```

# 5. 自定义MessageConverter

## 5.1 实现MessageConverter接口
```java
import org.springframework.amqp.core.Message;
import org.springframework.amqp.core.MessageProperties;
import org.springframework.amqp.support.converter.MessageConverter;

public class CustomMessageConverter implements MessageConverter {

    @Override
    public Message toMessage(Object object, MessageProperties messageProperties) {
        // Java对象 → AMQP Message
        if (object instanceof MyMessage) {
            MyMessage msg = (MyMessage) object;
            byte[] body = serialize(msg);
            messageProperties.setContentType("application/x-custom");
            return new Message(body, messageProperties);
        }
        throw new IllegalArgumentException("Unsupported type: " + object.getClass());
    }

    @Override
    public Object fromMessage(Message message) {
        // AMQP Message → Java对象
        String contentType = message.getMessageProperties().getContentType();
        if ("application/x-custom".equals(contentType)) {
            return deserialize(message.getBody(), MyMessage.class);
        }
        return message.getBody(); // fallback
    }
    
    private byte[] serialize(MyMessage msg) {
        // 自定义序列化逻辑
    }
    
    private MyMessage deserialize(byte[] body, Class<MyMessage> clazz) {
        // 自定义反序列化逻辑
    }
}
```

## 5.2 AbstractMessageConverter
推荐继承`AbstractMessageConverter`，它提供了一些基础实现：

```java
public class ProtobufMessageConverter extends AbstractMessageConverter {

    public ProtobufMessageConverter() {
        super(new MessageProperties());
    }

    @Override
    protected Message createMessage(Object object, MessageProperties messageProperties) {
        byte[] bytes = ((GeneratedMessageV3) object).toByteArray();
        messageProperties.setContentType("application/x-protobuf");
        return new Message(bytes, messageProperties);
    }

    @Override
    public Object fromMessage(Message message) throws MessageConversionException {
        // Protobuf反序列化逻辑
        return YourProto.parseFrom(message.getBody());
    }
}
```

# 6. 生产环境推荐配置

## 6.1 最简配置（推荐）
```java
@Configuration
public class RabbitMQConfig {
    
    @Bean
    public Jackson2JsonMessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

## 6.2 安全配置
```java
@Configuration
public class RabbitMQConfig {
    
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory,
                                          Jackson2JsonMessageConverter converter) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(converter);
        return template;
    }
    
    @Bean
    public Jackson2JsonMessageConverter messageConverter() {
        Jackson2JsonMessageConverter converter = new Jackson2JsonMessageConverter();
        
        // 禁用默认类型以增强安全性
        // 生产环境根据实际情况决定是否启用__TypeId__
        DefaultJackson2JavaTypeMapper typeMapper = new DefaultJackson2JavaTypeMapper();
        typeMapper.setTypePrecedence(Jackson2JavaTypeMapper.TypePrecedence.INFERRED);
        converter.setJavaTypeMapper(typeMapper);
        
        return converter;
    }
}
```

# 7. 消息转换的常见问题

## 7.1 "__TypeId__"头部的问题
当使用Jackson2JsonMessageConverter且配置了类型信息时，消息头部会有`__TypeId__`。如果接收端没有配置对应的类型映射，会出现`ClassNotFoundException`或`ClassNotTrustedException`。

解决方案：
1. 发送时清除`__TypeId__`（如果不需要类型推导）：直接在Listener方法参数中指定具体类型即可
2. 配置信任包（参考上面3.4节）
3. 使用ClassMapper进行类型映射（参考上面3.6节）

## 7.2 消息体与参数类型不匹配
```java
// 消息体: {"name": "张三", "age": 25}
// 但Listener参数类型是String

@RabbitListener(queues = "user.queue")
public void handleUser(String jsonString) {
    // 会收到JSON字符串还是User对象取决于MessageConverter配置
    // SimpleMessageConverter: 收到JSON字符串
    // Jackson2JsonMessageConverter: 尝试反序列化但解析失败
}
```

确保MessageConverter的配置与接收方法的参数类型匹配。

## 7.3 Null值的序列化
```java
// 如果发送的对象中有null字段
Order order = new Order();  // name为null

// 默认Jackson会包含null字段
// {"id": 1, "name": null, "amount": 99.99}

// 可以通过ObjectMapper配置排除null
mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
// 结果: {"id": 1, "amount": 99.99}
```

## 7.4 循环引用问题
当对象之间存在双向引用时（如Order引用User，User引用Order列表），Jackson默认会抛出`JsonMappingException`。

解决方案：
```java
// 使用@JsonIgnore忽略反向引用
public class User {
    @JsonIgnore
    private List<Order> orders;
}

// 或使用@JsonManagedReference / @JsonBackReference
public class User {
    @JsonManagedReference
    private List<Order> orders;
}
public class Order {
    @JsonBackReference
    private User user;
}
```
