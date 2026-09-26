# 1. 概述与简介

Jackson 是 Java 生态中最流行的高性能 JSON 处理库之一，由 FasterXML 公司维护。它提供了三种主要的 JSON 处理模型：

- **Streaming API**（流式 API）：以事件驱动方式读写 JSON，性能最高、内存开销最小
- **Tree Model**（树模型）：将 JSON 解析为一棵 JsonNode 树，支持随机访问
- **Data Binding**（数据绑定）：通过注解和类型映射，直接在 JSON 和 Java POJO 之间转换

Jackson 的核心模块包括：
- `jackson-core`：提供 Streaming API（JsonParser、JsonGenerator）
- `jackson-databind`：提供数据绑定和树模型（ObjectMapper、JsonNode）
- `jackson-annotations`：提供 Jackson 注解

## 1.1 为什么选择 Jackson

- **性能优异**：在大多数场景下性能优于 Fastjson 和 Gson
- **功能全面**：支持几乎所有 JSON 处理需求
- **社区活跃**：Spring Boot 默认使用 Jackson
- **扩展性强**：可以通过 Module 机制扩展（如 JSR310、Hibernate 等）
- **成熟稳定**：经过大规模生产环境验证

## 1.2 Maven 依赖

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.15.2</version>
</dependency>
```

`jackson-databind` 会自动依赖 `jackson-core` 和 `jackson-annotations`。

如需支持 Java 8 日期时间（JSR310）：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
    <version>2.15.2</version>
</dependency>
```

如需对 XML 格式的支持（可选）：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
    <version>2.15.2</version>
</dependency>
```

---

# 2. 核心功能与 API

## 2.1 ObjectMapper 核心配置

ObjectMapper 是 Jackson 的核心入口，所有序列化和反序列化操作都通过它完成。

```java
public class ObjectMapperConfig {

    public static ObjectMapper createObjectMapper() {
        ObjectMapper mapper = new ObjectMapper();

        // 缩进输出（美化）
        mapper.enable(SerializationFeature.INDENT_OUTPUT);

        // 日期格式
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        mapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

        // 忽略未知属性（防止反序列化时抛异常）
        mapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);

        // 空对象不抛异常
        mapper.disable(SerializationFeature.FAIL_ON_EMPTY_BEANS);

        // 允许字段名不带引号（宽松解析）
        mapper.configure(JsonParser.Feature.ALLOW_UNQUOTED_FIELD_NAMES, true);

        // 允许单引号
        mapper.configure(JsonParser.Feature.ALLOW_SINGLE_QUOTES, true);

        // 将空字符串转为 null
        mapper.enable(DeserializationFeature.ACCEPT_EMPTY_STRING_AS_NULL_OBJECT);

        // 属性命名策略：下划线转驼峰
        mapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);

        // 注册 JSR310 模块（支持 Java 8 日期时间）
        mapper.registerModule(new JavaTimeModule());

        // 只包含非 null 的属性
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);

        return mapper;
    }
}
```

### 常用配置项详解

| 配置项 | 作用 |
|--------|------|
| `FAIL_ON_UNKNOWN_PROPERTIES` | 遇到未知属性时是否抛异常，建议关闭 |
| `FAIL_ON_EMPTY_BEANS` | 空对象是否抛异常，建议关闭 |
| `INDENT_OUTPUT` | 输出缩进 JSON，调试时使用 |
| `WRITE_DATES_AS_TIMESTAMPS` | 日期是否写为时间戳，建议关闭 |
| `ACCEPT_EMPTY_STRING_AS_NULL_OBJECT` | 空字符串转为 null |
| `ALLOW_UNQUOTED_FIELD_NAMES` | 允许字段名不带引号 |
| `ALLOW_SINGLE_QUOTES` | 允许单引号 |

## 2.2 基本序列化与反序列化

```java
public class BasicUsage {

    public static void main(String[] args) throws Exception {
        ObjectMapper mapper = new ObjectMapper();

        // ========== 序列化：Java 对象 -> JSON ==========
        User user = new User("张三", 25, "zhangsan@example.com");

        // 序列化为字符串
        String json = mapper.writeValueAsString(user);
        System.out.println(json);

        // 序列化为字节数组
        byte[] bytes = mapper.writeValueAsBytes(user);

        // 写入文件
        mapper.writeValue(new File("user.json"), user);

        // ========== 反序列化：JSON -> Java 对象 ==========
        String jsonStr = "{\"name\":\"李四\",\"age\":30,\"email\":\"lisi@example.com\"}";

        // 从字符串反序列化
        User user2 = mapper.readValue(jsonStr, User.class);

        // 从字节数组反序列化
        User user3 = mapper.readValue(bytes, User.class);

        // 从文件反序列化
        User user4 = mapper.readValue(new File("user.json"), User.class);

        // 从 InputStream 反序列化
        User user5 = mapper.readValue(new ByteArrayInputStream(bytes), User.class);
    }
}

class User {
    private String name;
    private int age;
    private String email;

    // 必须有无参构造器
    public User() {}

    public User(String name, int age, String email) {
        this.name = name;
        this.age = age;
        this.email = email;
    }

    // getter/setter 省略
}
```

## 2.3 泛型与 TypeReference

由于 Java 泛型擦除，反序列化泛型类型时需要 TypeReference：

```java
public class TypeReferenceExample {

    public static void main(String[] args) throws Exception {
        ObjectMapper mapper = new ObjectMapper();

        // 反序列化 List<User>
        String jsonArray = "[{\"name\":\"张三\",\"age\":25},{\"name\":\"李四\",\"age\":30}]";
        List<User> users = mapper.readValue(jsonArray, new TypeReference<List<User>>() {});

        // 反序列化 Map<String, User>
        String jsonMap = "{\"user1\":{\"name\":\"张三\",\"age\":25}}";
        Map<String, User> userMap = mapper.readValue(jsonMap, new TypeReference<Map<String, User>>() {});

        // 反序列化复杂嵌套泛型
        String jsonResult = "{\"code\":200,\"data\":[{\"name\":\"张三\"}]}";
        ApiResponse<List<User>> response = mapper.readValue(jsonResult,
                new TypeReference<ApiResponse<List<User>>>() {});
    }
}

class ApiResponse<T> {
    private int code;
    private T data;
    // getter/setter
}
```

### TypeReference 的原理

TypeReference 通过创建匿名内部类来捕获泛型参数的类型信息。由于匿名内部类会保留父类的泛型信息，Jackson 可以通过 `TypeFactory` 从中解析出完整的参数化类型。

```java
// 等价实现
TypeReference<List<User>> typeRef = new TypeReference<List<User>>() {};
// typeRef.getType() 可以得到 List<User> 的完整类型信息
```

## 2.4 常用注解详解

### @JsonProperty

用于指定 JSON 字段名，或标记 getter/setter 对应的属性：

```java
public class Product {

    @JsonProperty("product_id")
    private String id;

    @JsonProperty("product_name")
    private String name;

    @JsonProperty("created_at")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createdAt;

    // 用于 getter 方法
    @JsonProperty("full_description")
    public String getFullDesc() {
        return name + " [" + id + "]";
    }
}
```

### @JsonIgnore 和 @JsonIgnoreProperties

```java
public class Employee {

    private String name;
    private String password;

    @JsonIgnore  // 序列化和反序列化时忽略此属性
    private String internalCode;

    // 类级别忽略多个属性
    @JsonIgnoreProperties({"internalId", "internalFlag"})
    private String internalId;
}

// 或者在类上使用
@JsonIgnoreProperties(ignoreUnknown = true)
public class Config {
    private String key;
    private String value;
    // Jackson 会忽略 JSON 中多余的字段，不会抛异常
}
```

### @JsonInclude

控制属性何时包含在 JSON 中：

```java
public class Article {

    @JsonInclude(JsonInclude.Include.NON_NULL)      // 非 null 时包含
    private String title;

    @JsonInclude(JsonInclude.Include.NON_EMPTY)      // 非空时包含（集合、字符串等）
    private List<String> tags;

    @JsonInclude(JsonInclude.Include.NON_DEFAULT)     // 非默认值时包含
    private int version = 1;

    @JsonInclude(JsonInclude.Include.ALWAYS)          // 始终包含（默认）
    private String content;
}
```

### @JsonFormat

格式化日期、枚举等：

```java
public class Order {

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")
    private LocalDateTime createTime;

    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate orderDate;

    @JsonFormat(shape = JsonFormat.Shape.STRING)  // 枚举输出为字符串
    private OrderStatus status;
}

enum OrderStatus {
    PENDING, PROCESSING, COMPLETED, CANCELLED
}
```

### @JsonUnwrapped

展平嵌套对象：

```java
public class Address {
    private String city;
    private String street;
}

public class Person {
    private String name;

    @JsonUnwrapped  // 将 Address 的字段展平到 Person 层级
    private Address address;
}

// 序列化结果：
// {"name":"张三","city":"北京","street":"长安街"}
```

### @JsonCreator 和 @JsonPropertyOrder

```java
public class ImmutableConfig {

    private final String key;
    private final String value;

    @JsonCreator  // 标记反序列化时使用的构造器或工厂方法
    public ImmutableConfig(
            @JsonProperty("key") String key,
            @JsonProperty("value") String value) {
        this.key = key;
        this.value = value;
    }
    // 只有 getter，没有 setter（不可变对象）
}

@JsonPropertyOrder({"name", "age", "email"})  // 指定序列化时的字段顺序
public class UserInfo {
    // ...
}
```

---

# 3. 高级特性

## 3.1 树模型（JsonNode）

当不需要预先定义 Java 类型时，可以使用树模型：

```java
public class TreeModelExample {

    public static void main(String[] args) throws Exception {
        ObjectMapper mapper = new ObjectMapper();

        String json = "{\"name\":\"张三\",\"age\":25,\"addresses\":[{\"city\":\"北京\"},{\"city\":\"上海\"}]}";

        // 解析为树
        JsonNode root = mapper.readTree(json);

        // 访问节点
        String name = root.get("name").asText();
        int age = root.get("age").asInt();

        // 访问嵌套数组
        JsonNode addresses = root.get("addresses");
        if (addresses.isArray()) {
            for (JsonNode addr : addresses) {
                String city = addr.get("city").asText();
                System.out.println(city);
            }
        }

        // 路径表达式访问
        JsonNode firstCity = root.at("/addresses/0/city");
        System.out.println(firstCity.asText());  // 北京

        // ========== 构建树 ==========
        ObjectNode rootNode = mapper.createObjectNode();
        rootNode.put("name", "李四");
        rootNode.put("age", 30);

        ArrayNode addrArray = rootNode.putArray("addresses");
        ObjectNode addr1 = addrArray.addObject();
        addr1.put("city", "广州");
        ObjectNode addr2 = addrArray.addObject();
        addr2.put("city", "深圳");

        String output = mapper.writeValueAsString(rootNode);
        System.out.println(output);
    }
}
```

### 常用 JsonNode 方法

| 方法 | 说明 |
|------|------|
| `get(String field)` | 获取子节点 |
| `at(String path)` | JSON 路径表达式访问 |
| `asText()` / `asInt()` / `asBoolean()` | 类型转换 |
| `isArray()` / `isObject()` / `isValueNode()` | 类型判断 |
| `path(String field)` | 类似 get，但缺失时返回 MissingNode |
| `findValue(String field)` | 深度优先搜索字段 |
| `toString()` | 输出 JSON 字符串 |

## 3.2 自定义序列化器和反序列化器

### 自定义序列化器

```java
public class CustomSerializer extends StdSerializer<User> {

    public CustomSerializer() {
        super(User.class);
    }

    @Override
    public void serialize(User user, JsonGenerator gen, SerializerProvider provider)
            throws IOException {
        gen.writeStartObject();
        gen.writeStringField("username", user.getName());
        gen.writeNumberField("age", user.getAge());
        // 对 email 进行脱敏处理
        String email = user.getEmail();
        if (email != null && email.contains("@")) {
            String[] parts = email.split("@");
            gen.writeStringField("email", parts[0].charAt(0) + "***@" + parts[1]);
        }
        gen.writeEndObject();
    }
}
```

### 自定义反序列化器

```java
public class CustomDeserializer extends StdDeserializer<User> {

    public CustomDeserializer() {
        super(User.class);
    }

    @Override
    public User deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        JsonNode node = p.getCodec().readTree(p);

        String name = node.get("username") != null ?
                node.get("username").asText() : node.get("name").asText();
        int age = node.get("age").asInt();
        String email = node.get("email") != null ? node.get("email").asText() : null;

        return new User(name, age, email);
    }
}
```

### 注册自定义序列化器

```java
@JsonSerialize(using = CustomSerializer.class)
@JsonDeserialize(using = CustomDeserializer.class)
public class User {
    // ...
}

// 或者全局注册
SimpleModule module = new SimpleModule();
module.addSerializer(User.class, new CustomSerializer());
module.addDeserializer(User.class, new CustomDeserializer());
mapper.registerModule(module);
```

## 3.3 多态类型处理（Polymorphism）

当需要序列化/反序列化接口或抽象类时，需要使用多态处理：

```java
// 基类
@JsonTypeInfo(
    use = JsonTypeInfo.Id.NAME,          // 使用名称标识类型
    include = JsonTypeInfo.As.PROPERTY,   // 作为属性包含
    property = "type"                     // 类型标识属性名
)
@JsonSubTypes({
    @JsonSubTypes.Type(value = Circle.class, name = "circle"),
    @JsonSubTypes.Type(value = Rectangle.class, name = "rectangle")
})
public abstract class Shape {
    public abstract double area();
}

public class Circle extends Shape {
    private double radius;

    public Circle(@JsonProperty("radius") double radius) {
        this.radius = radius;
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

public class Rectangle extends Shape {
    private double width;
    private double height;

    public Rectangle(@JsonProperty("width") double width,
                     @JsonProperty("height") double height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public double area() {
        return width * height;
    }
}

// 使用
public class PolymorphismExample {

    public static void main(String[] args) throws Exception {
        ObjectMapper mapper = new ObjectMapper();

        // 序列化
        Shape shape = new Circle(5.0);
        String json = mapper.writeValueAsString(shape);
        // 输出: {"type":"circle","radius":5.0}

        // 反序列化
        String input = "{\"type\":\"rectangle\",\"width\":3,\"height\":4}";
        Shape result = mapper.readValue(input, Shape.class);
        System.out.println(result.area());  // 12.0
    }
}
```

## 3.4 MixIn 注解

当无法修改源码时（如第三方类），可以使用 MixIn：

```java
// 假设第三方类
public class ThirdPartyClass {
    private String secret;
    private String name;
    // getter/setter
}

// 定义 MixIn 接口
public abstract class ThirdPartyMixIn {
    @JsonIgnore
    abstract String getSecret();  // 忽略 getSecret
}

// 使用
ObjectMapper mapper = new ObjectMapper();
mapper.addMixIn(ThirdPartyClass.class, ThirdPartyMixIn.class);
```

## 3.5 Streaming API

流式 API 是 Jackson 性能最高的处理方式，适合处理超大 JSON：

```java
public class StreamingExample {

    // 使用 JsonGenerator 生成 JSON
    public static void writeJson(OutputStream out) throws Exception {
        JsonFactory factory = new JsonFactory();
        try (JsonGenerator gen = factory.createGenerator(out, JsonEncoding.UTF8)) {
            gen.writeStartObject();
            gen.writeStringField("name", "张三");
            gen.writeNumberField("age", 25);

            gen.writeArrayFieldStart("hobbies");
            gen.writeString("读书");
            gen.writeString("编程");
            gen.writeEndArray();

            gen.writeObjectFieldStart("address");
            gen.writeStringField("city", "北京");
            gen.writeEndObject();

            gen.writeEndObject();
        }
    }

    // 使用 JsonParser 解析 JSON（流式，不构建完整树）
    public static void parseJson(InputStream in) throws Exception {
        JsonFactory factory = new JsonFactory();
        try (JsonParser parser = factory.createParser(in)) {
            while (!parser.isClosed()) {
                JsonToken token = parser.nextToken();
                if (token == null) break;

                switch (token) {
                    case FIELD_NAME:
                        String fieldName = parser.getCurrentName();
                        parser.nextToken(); // 移到值
                        switch (fieldName) {
                            case "name":
                                System.out.println("Name: " + parser.getText());
                                break;
                            case "age":
                                System.out.println("Age: " + parser.getIntValue());
                                break;
                        }
                        break;
                }
            }
        }
    }
}
```

### Streaming API vs DataBinding 性能对比

| 特性 | Streaming API | Data Binding |
|------|-------------|--------------|
| 性能 | 最高 | 中等 |
| 内存占用 | 最低 | 中等 |
| 易用性 | 低 | 高 |
| 适用场景 | 超大 JSON、高吞吐 | 日常开发 |
| 类型安全 | 无 | 有 |

## 3.6 过滤器与视图（JsonView）

### @JsonView

```java
public class Views {
    public static class Public {}
    public static class Internal extends Public {}
}

public class Document {

    @JsonView(Views.Public.class)
    private String title;

    @JsonView(Views.Public.class)
    private String summary;

    @JsonView(Views.Internal.class)
    private String internalNotes;

    @JsonView(Views.Internal.class)
    private BigDecimal cost;
}

// 使用视图序列化
ObjectMapper mapper = new ObjectMapper();
String publicView = mapper
    .writerWithView(Views.Public.class)
    .writeValueAsString(doc);
// 只包含 title 和 summary
```

### SimpleFilterProvider

```java
@JsonFilter("userFilter")
public class SensitiveUser {
    private String name;
    private String phone;
    private String idCard;
}

// 动态过滤
SimpleBeanPropertyFilter filter = SimpleBeanPropertyFilter
        .filterOutAllExcept("name");  // 只保留 name

FilterProvider filters = new SimpleFilterProvider()
        .addFilter("userFilter", filter);

String json = mapper.writer(filters).writeValueAsString(user);
// 输出: {"name":"张三"}
```

---

# 4. 底层原理

## 4.1 模块体系

Jackson 采用模块化设计，核心是 `jackson-core`，上层通过 `ObjectMapper` 整合所有功能：

```
+------------------------------------------+
|          jackson-databind                 |
|   ObjectMapper, ObjectReader,            |
|   ObjectWriter, JsonNode, BeanSerializer |
+------------------------------------------+
|          jackson-annotations              |
|   @JsonProperty, @JsonIgnore ...         |
+------------------------------------------+
|          jackson-core                     |
|   JsonParser, JsonGenerator, JsonFactory |
+------------------------------------------+
```

## 4.2 ObjectMapper 工作原理

1. **序列化流程**：
   - `writeValue()` -> `_serializerProvider()` 获取序列化器
   - `BeanSerializerFactory` 创建 `BeanSerializer`
   - `BeanSerializer` 遍历所有 getter 方法
   - 对每个属性调用对应的 `JsonSerializer`
   - 递归处理嵌套对象

2. **反序列化流程**：
   - `readValue()` -> `_deserializerProvider()` 获取反序列化器
   - `BeanDeserializerFactory` 创建 `BeanDeserializer`
   - 解析 JSON token 流
   - 通过 setter 或构造器注入属性值
   - 递归处理嵌套对象

```java
// 简化的序列化流程
public class SerializationFlow {

    public static void main(String[] args) throws Exception {
        ObjectMapper mapper = new ObjectMapper();

        // 1. 查找序列化器
        JavaType type = mapper.constructType(User.class);
        JsonSerializer<Object> serializer = mapper.getSerializerProvider()
                .findTypedValueSerializer(type, true, null);

        // 2. 执行序列化（内部实现类似）
        // serializer.serialize(value, jsonGenerator, provider);
    }
}
```

## 4.3 缓存机制

Jackson 对序列化器和反序列化器有内部缓存：

```java
// 序列化器缓存（在 SerializerProvider 中）
// - _knownSerializers: ConcurrentHashMap
// - 键为 JavaType，值为 JsonSerializer
// - 线程安全，全局共享

// Bean 属性序列化器缓存
// - _beanSerializerCache: 每个 BeanSerializer 缓存属性处理器
```

## 4.4 类型解析机制

```java
ObjectMapper mapper = new ObjectMapper();
TypeFactory typeFactory = mapper.getTypeFactory();

// 构造复杂类型
JavaType listType = typeFactory.constructCollectionType(List.class, User.class);
JavaType mapType = typeFactory.constructMapType(HashMap.class, String.class, User.class);
JavaType paramType = typeFactory.constructParametricType(ApiResponse.class, User.class);
```

---

# 5. 性能优化

## 5.1 ObjectMapper 重用

**最重要的性能优化**：ObjectMapper 是线程安全的，应当作为单例重用。

```java
@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        // 配置...
        return mapper;
    }
}

// 错误做法：每次创建新实例
public class WrongWay {
    public String toJson(Object obj) throws Exception {
        return new ObjectMapper().writeValueAsString(obj);  // 性能差！
    }
}
```

## 5.2 预配置 ObjectMapper 属性

```java
// 开启以下特性可提升性能
ObjectMapper mapper = new ObjectMapper();

// 使用字节码生成（默认开启）- 比反射快
// 对应 jackson-module-afterburner（第三方模块，性能提升约 30%）
mapper.registerModule(new AfterburnerModule());

// 单次创建，重复使用
// Spring Boot 中默认是单例
```

## 5.3 使用 Reader/Writer 缓存

```java
public class ReaderWriterCache {

    private final ObjectMapper mapper = new ObjectMapper();

    // 缓存特定类型的 ObjectReader
    private final Map<JavaType, ObjectReader> readerCache = new ConcurrentHashMap<>();
    private final Map<JavaType, ObjectWriter> writerCache = new ConcurrentHashMap<>();

    public <T> T readValue(String json, Class<T> clazz) throws Exception {
        JavaType type = mapper.constructType(clazz);
        ObjectReader reader = readerCache.computeIfAbsent(type, mapper::readerFor);
        return reader.readValue(json);
    }
}
```

## 5.4 大数据量处理技巧

```java
public class LargeJsonProcessing {

    public static void processLargeArray(InputStream input, ObjectMapper mapper) throws Exception {
        JsonParser parser = mapper.getFactory().createParser(input);

        if (parser.nextToken() != JsonToken.START_ARRAY) {
            throw new IllegalStateException("Expected array");
        }

        // 逐个元素处理，不加载全部到内存
        while (parser.nextToken() == JsonToken.START_OBJECT) {
            User user = mapper.readValue(parser, User.class);
            processUser(user);  // 逐条处理
        }

        parser.close();
    }

    private static void processUser(User user) {
        // 处理单个用户
    }
}
```

## 5.5 序列化性能对比数据

| 操作 | Jackson | Fastjson | Gson |
|------|---------|----------|------|
| 简单对象序列化 | ~800K ops/s | ~700K ops/s | ~500K ops/s |
| 简单对象反序列化 | ~600K ops/s | ~500K ops/s | ~400K ops/s |
| 大 JSON 解析 | 优 | 优 | 中 |
| 内存占用 | 中 | 低 | 高 |

---

# 6. 最佳实践与常见坑

## 6.1 最佳实践

### 1. 全局配置 ObjectMapper

```java
@Configuration
public class JacksonGlobalConfig {

    @Bean
    @Primary
    public ObjectMapper objectMapper() {
        return Jackson2ObjectMapperBuilder.json()
                .featuresToDisable(
                    SerializationFeature.FAIL_ON_EMPTY_BEANS,
                    DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES,
                    SerializationFeature.WRITE_DATES_AS_TIMESTAMPS
                )
                .featuresToEnable(SerializationFeature.INDENT_OUTPUT)
                .serializationInclusion(JsonInclude.Include.NON_NULL)
                .dateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"))
                .modules(new JavaTimeModule())
                .build();
    }
}
```

### 2. 统一处理 null 值

```java
// 类级别
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResult {
    private int code;
    private String message;
    private Object data;
}

// 全局级别
mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
```

### 3. 使用 @JsonIgnore 保护敏感字段

```java
public class UserAccount {
    @JsonIgnore
    private String password;

    @JsonIgnore
    private String idCard;
}
```

### 4. 枚举序列化统一处理

```java
public enum Status {
    @JsonProperty("pending")   PENDING,
    @JsonProperty("active")    ACTIVE,
    @JsonProperty("disabled")  DISABLED;
}

// 或全局配置
mapper.enable(SerializationFeature.WRITE_ENUMS_USING_TO_STRING);
mapper.enable(DeserializationFeature.READ_ENUMS_USING_TO_STRING);
```

## 6.2 常见坑

### 坑 1：循环引用

```java
// 双向引用导致 StackOverflowError
class Parent {
    private String name;
    private List<Child> children;
}

class Child {
    private String name;
    private Parent parent;  // 循环引用
}

// 解决方案：
// 1. 使用 @JsonIgnore 在一端忽略
// 2. 使用 @JsonManagedReference / @JsonBackReference
// 3. 使用 @JsonIdentityInfo
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")
class Parent {
    // ...
}
```

### 坑 2：多态反序列化漏洞

```java
// 安全问题：默认的多态反序列化可能导致远程代码执行
// 解决方案：限制允许的类型
ObjectMapper mapper = new ObjectMapper();
mapper.activateDefaultTyping(
    BasicPolymorphicTypeValidator.builder()
        .allowIfSubType("com.example.")
        .allowIfSubType("java.util.")
        .build(),
    ObjectMapper.DefaultTyping.NON_FINAL
);
```

### 坑 3：日期反序列化失败

```java
// 问题：默认日期格式不匹配
// 解决方案 1：注解指定格式
@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
private LocalDateTime createTime;

// 解决方案 2：注册 JavaTimeModule
mapper.registerModule(new JavaTimeModule());
mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
```

### 坑 4：未知属性抛异常

```java
// 后端一个字段没加 getter，或前端多传了字段都会抛 UnrecognizedPropertyException
// 解决方案：全局关闭
mapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
```

### 坑 5：泛型擦除问题

```java
// 错误：丢失泛型信息
List<User> users = mapper.readValue(json, List.class);  // 得到 List<LinkedHashMap>

// 正确：使用 TypeReference
List<User> users = mapper.readValue(json, new TypeReference<List<User>>() {});
```

### 坑 6：getter 误识别

```java
public class BadBean {
    private boolean active;

    // 被错误识别为字段 "active"
    public boolean isActive() { return active; }

    // 被错误识别为字段 "empty"
    public boolean isEmpty() { return false; }
}
// 解决方案：使用 @JsonIgnore 标记不应该序列化的 getter
```

---

# 7. 与其他技术对比

## 7.1 Jackson vs Fastjson

| 对比维度 | Jackson | Fastjson |
|---------|---------|----------|
| 性能 | 优秀，大数据量场景稳定 | 早期较快，近年代码质量问题 |
| 安全性 | 高，需要有意识开启多态 | 历史漏洞多（AutoType RCE） |
| 规范性 | 严格遵守 JSON 规范 | 宽松，支持部分非标准格式 |
| 易用性 | 配置较多但灵活 | API 简洁易上手 |
| Spring Boot 集成 | 默认支持 | 需要额外配置 |
| 社区活跃度 | 高 | 中 |
| 文档质量 | 好 | 一般 |
| 扩展性 | 极强（模块化设计） | 一般 |

## 7.2 Jackson vs Gson

| 对比维度 | Jackson | Gson |
|---------|---------|------|
| 性能 | 更优 | 中等 |
| 体积 | 较大（~3MB） | 较小（~300KB） |
| 注解 | 丰富 | 较少 |
| 泛型支持 | TypeReference | TypeToken |
| 流式处理 | 强大 | 有限 |
| Android | 可以（但有更大库） | 更常用 |

## 7.3 选型建议

- **新 Spring Boot 项目**：默认使用 Jackson，无需额外引入
- **Android 项目**：推荐使用 Gson（体积小）或 Moshi
- **高性能要求**：Jackson + Afterburner 模块
- **简单场景**：任何库都可满足，根据团队技术栈决定
- **安全性优先**：Jackson > Gson > Fastjson

---

# 8. 面试要点

## 8.1 基础问题

**Q1: Jackson 有哪些核心模块？**

A: `jackson-core`（流式 API）、`jackson-databind`（数据绑定和树模型）、`jackson-annotations`（注解）。

**Q2: ObjectMapper 是线程安全的吗？如何使用？**

A: 是线程安全的，配置完成后可作为单例使用。可声明为 Spring Bean 全局注入。配置和模块注册应在初始化阶段完成，后续共享使用。

**Q3: @JsonIgnore 和 @JsonIgnoreProperties 的区别？**

A: `@JsonIgnore` 作用于字段/getter/setter 级别；`@JsonIgnoreProperties` 作用于类级别，可以指定多个要忽略的属性。后者还可以设置 `ignoreUnknown=true` 忽略所有未知属性。

**Q4: 如何处理泛型反序列化？**

A: 使用 `TypeReference` 匿名内部类，它通过子类保留泛型参数信息。如 `new TypeReference<List<User>>() {}`。

**Q5: Jackson 的三种处理模型及适用场景？**

A: Streaming API（超大 JSON、高性能）、Tree Model（JSON 结构不固定、需要随机访问）、Data Binding（日常开发，POJO 映射）。

## 8.2 进阶问题

**Q6: Jackson 如何处理循环引用？**

A: 使用 `@JsonIgnore` 忽略一端，`@JsonManagedReference`/`@JsonBackReference` 标注父子关系，或 `@JsonIdentityInfo` 使用 ID 引用序列化。

**Q7: 什么是 Jackson MixIn？适用场景？**

A: MixIn 允许在不修改源码的情况下为第三方类添加 Jackson 注解。通过 `addMixIn(Target.class, MixInClass.class)` 绑定。

**Q8: 如何自定义序列化逻辑？**

A: 继承 `StdSerializer<T>` 并重写 `serialize()` 方法，或继承 `StdDeserializer<T>` 重写 `deserialize()` 方法。然后注册到 `SimpleModule` 或使用 `@JsonSerialize`/`@JsonDeserialize` 注解。

**Q9: @JsonInclude 有哪些常用策略？**

A: `ALWAYS`（总是）、`NON_NULL`（非 null）、`NON_EMPTY`（非空）、`NON_DEFAULT`（非默认值）、`NON_ABSENT`（非 Optional.absent）。

**Q10: Jackson 如何防止反序列化漏洞？**

A: 使用 `BasicPolymorphicTypeValidator` 限制允许的类型，或关闭 `enableDefaultTyping()`。

## 8.3 源码分析题

**Q11: ObjectMapper 的序列化流程是怎样的？**

A: `writeValue()` -> 获取 `SerializerFactory` -> 查找或创建 `JsonSerializer`（有缓存）-> 遍历 Bean 属性（通过 `BeanPropertyWriter`）-> 调用属性序列化器 -> 输出 JSON。

**Q12: Jackson 是如何处理注解的？**

A: 通过 `AnnotationIntrospector`（默认是 `JacksonAnnotationIntrospector`）解析注解，提取序列化配置（包含策略、命名策略、自定义序列化器等）。

**Q13: TypeReference 是如何工作的？**

A: `TypeReference<T>` 是一个抽象类，构造时通过 `(Type)((ParameterizedType)getClass().getGenericSuperclass()).getActualTypeArguments()[0]` 获取泛型参数的实际类型。

## 8.4 实战题

**Q14: Spring Boot 中如何统一配置 Jackson？**

A: 通过 `Jackson2ObjectMapperBuilder` 或 `@Bean` 返回自定义的 `ObjectMapper`，配合 `@Primary` 确保全局生效。

**Q15: 大 JSON 文件（1GB+）如何处理？**

A: 使用 Streaming API（`JsonParser`）逐 token 读取；或分批读取数组元素，每处理一个元素就释放引用，避免 OOM。

**Q16: 枚举序列化有哪些方式？**

A: 默认使用 `name()`；`@JsonFormat(shape = Shape.STRING)` 同上；`@JsonValue` 指定方法返回值；`WRITE_ENUMS_USING_TO_STRING` 配置；自定义序列化器。

**Q17: 如何处理 JSON 中蛇形命名转驼峰？**

A: `mapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE)`（2.7+）或 `PropertyNamingStrategy.SNAKE_CASE`（旧版本）。或用 `@JsonProperty` 逐个指定。
