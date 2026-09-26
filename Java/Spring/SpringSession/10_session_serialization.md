# 1. 序列化概述

Spring Session 需要将 Session 对象及其属性序列化后存储到外部存储中。序列化的选择和配置直接影响**性能、可读性、存储空间和跨语言兼容性**。

```
Java 对象 (内存)
    │
    ▼ 序列化 (Serializer)
  ┌──────────────────────┐
  │ Redis / JDBC / ...   │
  │ (字节数组或JSON)     │
  └──────────────────────┘
    │
    ▼ 反序列化 (Deserializer)
Java 对象 (内存)
```

---

# 2. JDK 序列化（默认）

## 2.1 实现原理

```java
// JDK 内置序列化
public class JdkSerializationRedisSerializer implements RedisSerializer<Object> {

    @Override
    public byte[] serialize(Object object) {
        if (object == null) return new byte[0];
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bos);
        oos.writeObject(object);
        return bos.toByteArray();
    }

    @Override
    public Object deserialize(byte[] bytes) {
        if (bytes == null || bytes.length == 0) return null;
        ByteArrayInputStream bis = new ByteArrayInputStream(bytes);
        ObjectInputStream ois = new ObjectInputStream(bis);
        return ois.readObject();
    }
}
```

## 2.2 要求与限制

| 要求 | 说明 |
|------|------|
| **实现 Serializable** | 所有存入 Session 的对象必须实现 `java.io.Serializable` |
| **serialVersionUID** | 建议显式声明，避免反序列化版本冲突 |
| **类存在** | 反序列化时 JVM 中必须有对应的类定义 |
| **安全性** | JDK 反序列化存在安全漏洞风险 |

```java
// 正确做法
public class UserInfo implements Serializable {
    private static final long serialVersionUID = 1L;

    private String username;
    private String email;
    // getter/setter
}
```

## 2.3 优缺点

| 优点 | 缺点 |
|------|------|
| 无需额外依赖 | 序列化后数据不可读（二进制） |
| 完整保留对象结构（包括类型、继承） | 跨语言无法解析 |
| JVM 原生支持 | 性能一般 |
| | 类变更可能导致反序列化失败 |

---

# 3. Jackson JSON 序列化

## 3.1 配置方式

```java
@Configuration
public class JsonSerializationConfig {

    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        ObjectMapper mapper = new ObjectMapper();

        // 注册 Java 8 时间模块
        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

        // 关键：启用 default typing（类型安全）
        mapper.activateDefaultTyping(
            mapper.getPolymorphicTypeValidator(),
            ObjectMapper.DefaultTyping.NON_FINAL
        );

        return new GenericJackson2JsonRedisSerializer(mapper);
    }
}
```

## 3.2 序列化结果对比

**JDK 序列化**（十六进制不可读）：

```
\xAC\xED\x00\x05sr\x00\x0ecom.example.User...
```

**JSON 序列化**（可读）：

```json
{
  "@class": "com.example.UserInfo",
  "username": "zhangsan",
  "email": "zhangsan@example.com",
  "roles": ["java.util.Arrays$ArrayList", ["ROLE_USER", "ROLE_ADMIN"]]
}
```

## 3.3 类型安全（Default Typing）

JSON 本身不保留类型信息，需要通过 Jackson 的 Default Typing 机制写入类名：

```java
// 四种 Default Typing 策略
ObjectMapper.DefaultTyping.NON_FINAL       // 非 final 类型写入 @class（推荐）
ObjectMapper.DefaultTyping.EVERYTHING      // 所有类型都写入 @class
ObjectMapper.DefaultTyping.JAVA_LANG_OBJECT  // 仅 Object 类型写入
ObjectMapper.DefaultTyping.OBJECT_AND_NON_CONCRETE  // Object + 非具体类型
```

**安全注意**：Default Typing 可能导致反序列化安全漏洞。应使用 `BasicPolymorphicTypeValidator` 限制可反序列化的类：

```java
PolymorphicTypeValidator ptv = BasicPolymorphicTypeValidator.builder()
    .allowIfBaseType(Object.class)
    .allowIfSubType("com.example.")
    .build();

mapper.activateDefaultTyping(ptv, ObjectMapper.DefaultTyping.NON_FINAL);
```

---

# 4. Kryo 序列化

## 4.1 Kryo 简介

Kryo 是一个高性能的 Java 序列化框架，比 JDK 序列化快 5-10 倍。

## 4.2 集成方式

```xml
<dependency>
    <groupId>com.esotericsoftware</groupId>
    <artifactId>kryo</artifactId>
    <version>5.5.0</version>
</dependency>
```

```java
@Component
public class KryoRedisSerializer implements RedisSerializer<Object> {

    // Kryo 不是线程安全的，使用 ThreadLocal
    private final ThreadLocal<Kryo> kryoThreadLocal = ThreadLocal.withInitial(() -> {
        Kryo kryo = new Kryo();
        kryo.setRegistrationRequired(false); // 不强制注册
        // 注册常用类以提升性能
        kryo.register(HashMap.class);
        kryo.register(ArrayList.class);
        kryo.register(UserInfo.class);
        return kryo;
    });

    @Override
    public byte[] serialize(Object object) {
        if (object == null) return new byte[0];

        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        Output output = new Output(bos);
        kryoThreadLocal.get().writeClassAndObject(output, object);
        output.close();

        return bos.toByteArray();
    }

    @Override
    public Object deserialize(byte[] bytes) {
        if (bytes == null || bytes.length == 0) return null;

        ByteArrayInputStream bis = new ByteArrayInputStream(bytes);
        Input input = new Input(bis);
        Object obj = kryoThreadLocal.get().readClassAndObject(input);
        input.close();

        return obj;
    }
}
```

---

# 5. Protobuf 序列化

## 5.1 适用场景

| 场景 | 说明 |
|------|------|
| 跨语言通信 | Protobuf 天然支持多语言 |
| 极致的空间效率 | 二进制编码，大小显著低于 JSON |
| Schema 先行 | 有明确的 `.proto` 定义，类型安全 |
| gRPC 生态 | 与 gRPC 生态系统无缝集成 |

## 5.2 限制

- 需要预先定义 `.proto` 文件
- 动态类型支持差（Session 属性类型多变）
- 一般不推荐用于通用 Session 序列化

---

# 6. 序列化对比与选择

| 方案 | 性能 | 大小 | 可读性 | 跨语言 | 配置复杂度 | 推荐场景 |
|------|------|------|--------|--------|------------|----------|
| **JDK** | ★★☆ | ★★☆ | ✗ | ✗ | 低 | 快速上手 |
| **Jackson JSON** | ★★★ | ★★★ | ✓ | ✓ | 中 | 通用生产环境 |
| **Kryo** | ★★★★ | ★★★★ | ✗ | ✗ | 中 | 高性能场景 |
| **Protobuf** | ★★★★★ | ★★★★★ | ✗ | ✓ | 高 | gRPC 生态 |

## 6.1 推荐方案

```
大多数应用：Jackson JSON
  - 可读性好，方便排查问题
  - 性能足够
  - 注意 Default Typing 安全配置

高并发场景：Kryo
  - 性能最优
  - 但不可读，调试困难

不要用 JDK 序列化放在生产环境：
  - 存在安全漏洞
  - 可读性差
  - 类变更敏感
```

---

# 7. 序列化常见问题

## 7.1 反序列化失败

**问题**：发布新版本后，旧 Session 反序列化失败

**原因**：Session 中的对象类结构发生了不兼容的变更

**解决方案**：

```java
// 1. 使用 serialVersionUID
public class UserInfo implements Serializable {
    private static final long serialVersionUID = 2L;

    private String username;
    private String email;
    private String phone; // 新增字段
}

// 2. 使用 JSON（容错性更好）
mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

// 3. 发布前清理旧 Session（强制用户重新登录）
// 方式一：清空 Redis 中的 Session 数据
// 方式二：修改 Cookie 名称或域名，使旧 Cookie 无效
```

## 7.2 序列化循环引用

```java
public class TreeNode implements Serializable {
    private String name;
    private TreeNode parent;  // 可能形成循环引用
    private List<TreeNode> children;
}
```

**后果**：JDK 序列化支持循环引用，但 JSON 序列化会导致栈溢出。

**解决**：使用 `@JsonIgnore` 或 `@JsonBackReference` / `@JsonManagedReference`。

## 7.3 大数据量序列化性能

```java
// 不推荐：在 Session 中存储大数据
session.setAttribute("largeData", hugeList); // 可能几 MB

// 推荐：Session 只存标识，大数据放缓存/数据库
session.setAttribute("dataId", "ref:cache:largeData");
// 需要时从 Redis/DB 获取
```

## 7.4 Lambda 表达式序列化

```java
// 问题：Lambda 表达式类不属于常规可序列化类
Runnable task = (Runnable & Serializable) () -> System.out.println("task");
session.setAttribute("task", task); // 可能序列化失败

// 建议：不要在 Session 中存储 Lambda 或匿名类
```
