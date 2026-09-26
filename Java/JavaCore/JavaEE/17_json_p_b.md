
# 1. 概述

## 1.1 两个规范

| 规范 | 全称 | 定位 |
|------|------|------|
| **JSON-P** | JSON Processing | **低层 API**：解析、生成 JSON（流式 + 树模型） |
| **JSON-B** | JSON Binding | **高层 API**：Java 对象 ↔ JSON 双向绑定（类似 JAXB for JSON） |

```
JSON-P：JsonParser / JsonGenerator / JsonObject / JsonArray
JSON-B：@JsonbProperty 注解 + Jsonb 对象
```

## 1.2 参考实现

- 两个规范都由 **Eclipse Yasson** 作为参考实现（RI），Maven 坐标：

```xml
<dependency>
    <groupId>jakarta.json</groupId>
    <artifactId>jakarta.json-api</artifactId>
    <version>2.1.3</version>
</dependency>
<dependency>
    <groupId>org.eclipse.parsson</groupId>
    <artifactId>parsson</artifactId>
    <version>1.1.6</version>
</dependency>
<dependency>
    <groupId>jakarta.json.bind</groupId>
    <artifactId>jakarta.json.bind-api</artifactId>
    <version>3.0.1</version>
</dependency>
<dependency>
    <groupId>org.eclipse</groupId>
    <artifactId>yasson</artifactId>
    <version>3.0.4</version>
</dependency>
```

> 实践中 Spring Boot 项目多用 Jackson / Fastjson / Gson，JSON-P/B 主要出现在纯 Jakarta EE 应用。

# 2. JSON-P 流式 API

## 2.1 生成 JSON（JsonGenerator）

```java
JsonGeneratorFactory factory = Json.createGeneratorFactory(Map.of(
        JsonGenerator.PRETTY_PRINTING, true));
JsonGenerator gen = factory.createGenerator(System.out);

gen.writeStartObject()
   .write("id", 1)
   .write("name", "张三")
   .writeStartArray("roles")
       .write("admin")
       .write("user")
   .writeEnd()
   .writeEnd();
gen.close();
```

输出：

```json
{
  "id": 1,
  "name": "张三",
  "roles": ["admin", "user"]
}
```

## 2.2 解析 JSON（JsonParser）

```java
JsonParser parser = Json.createParser(new StringReader(jsonStr));

while (parser.hasNext()) {
    JsonParser.Event event = parser.next();
    switch (event) {
        case KEY_NAME -> System.out.println("key: " + parser.getString());
        case VALUE_STRING -> System.out.println("value: " + parser.getString());
        case VALUE_NUMBER -> System.out.println("number: " + parser.getBigDecimal());
        default -> { }
    }
}
```

# 3. JSON-P 对象模型 API

## 3.1 构建 JsonObject

```java
JsonObject obj = Json.createObjectBuilder()
        .add("id", 1)
        .add("name", "张三")
        .add("roles", Json.createArrayBuilder()
                .add("admin").add("user"))
        .add("address", Json.createObjectBuilder()
                .add("city", "北京"))
        .build();

String json = obj.toString();
```

## 3.2 读取 JsonObject

```java
JsonObject obj = Json.createReader(new StringReader(json)).readObject();

int id = obj.getInt("id");
String name = obj.getString("name");
JsonArray roles = obj.getJsonArray("roles");
String city = obj.getJsonObject("address").getString("city");

for (JsonValue v : roles) {
    System.out.println(v.toString());
}
```

## 3.3 核心接口

| 接口 | 说明 |
|------|------|
| `JsonValue` | 所有 JSON 值的父接口 |
| `JsonObject` | JSON 对象（`{...}`） |
| `JsonArray` | JSON 数组（`[...]`） |
| `JsonString` | JSON 字符串 |
| `JsonNumber` | JSON 数字 |
| `JsonStructure` | 结构类型（Object/Array） |
| `JsonPointer` | JSON 指针（按路径查询/修改） |
| `JsonPatch` | JSON 补丁操作 |

## 3.4 JsonPointer 示例

```java
JsonObject obj = Json.createReader(new StringReader(json)).readObject();
JsonPointer pointer = Json.createPointer("/address/city");
JsonValue city = pointer.getValue(obj);    // "北京"
```

# 4. JSON-B 绑定

## 4.1 注解

| 注解 | 作用 |
|------|------|
| `@JsonbProperty(name)` | 指定 JSON 字段名 |
| `@JsonbPropertyOrder` | 字段顺序 |
| `@JsonbDateFormat` | 日期格式 |
| `@JsonbNumberFormat` | 数字格式 |
| `@JsonbNillable` | 是否序列化 null（默认不序列化） |
| `@JsonbTransient` | 忽略字段 |
| `@JsonbVisibility` | 字段可见性策略 |
| `@JsonbAdapter` | 自定义适配器 |
| `@JsonbCreator` | 反序列化构造器/工厂 |
| `@JsonbPolymorphic` | 多态序列化 |

## 4.2 模型类

```java
public class User {

    @JsonbProperty("id")
    private Long id;

    @JsonbProperty("userName")
    private String name;

    @JsonbDateFormat("yyyy-MM-dd")
    private LocalDate birthday;

    @JsonbTransient
    private String password;

    // 默认构造器、getter/setter（字段私有需配置访问策略）
}
```

## 4.3 序列化与反序列化

```java
Jsonb jsonb = JsonbBuilder.create();

// 对象 → JSON
String json = jsonb.toJson(user);

// JSON → 对象
User user = jsonb.fromJson(json, User.class);
```

## 4.4 集合/泛型

```java
Type type = new ArrayList<User>() { }.getGenericSuperclass();
List<User> users = jsonb.fromJson(jsonArrayStr, type);
```

# 5. 与主流 JSON 库对比

| 对比 | JSON-P/JSON-B | Jackson | Fastjson | Gson |
|------|---------------|---------|----------|------|
| 标准 | Jakarta EE 标准 | 第三方 | 第三方 | Google |
| 性能 | 中 | 高 | 高 | 中 |
| 注解 | `@JsonbProperty` | `@JsonProperty` | `@JSONField` | `@SerializedName` |
| 流式 API | JsonParser/Generator | JsonParser/Generator | JSONReader/Writer | JsonReader/Writer |
| Spring Boot | 需集成 | 默认 | 可集成 | 可集成 |

## 5.1 注解对照速查

| 需求 | JSON-B | Jackson |
|------|--------|---------|
| 字段改名 | `@JsonbProperty("x")` | `@JsonProperty("x")` |
| 忽略字段 | `@JsonbTransient` | `@JsonIgnore` |
| 日期格式 | `@JsonbDateFormat` | `@JsonFormat` |
| 自定义序列化 | `@JsonbAdapter` | `@JsonSerialize` + `JsonSerializer` |
| 忽略 null | `@JsonbNillable(false)` | `@JsonInclude(NON_NULL)` |

# 6. 常见问题排查清单

| 问题 | 排查方向 |
|------|----------|
| 无默认构造器反序列化失败 | 提供无参构造器或用 `@JsonbCreator` |
| 日期解析失败 | 配置 `@JsonbDateFormat` 与实际格式一致 |
| 字段丢失 | 检查 `@JsonbTransient`、null 处理、访问策略 |
| 循环引用栈溢出 | 忽略反向字段或用适配器 |
| 包名找不到 | JDK 11+ 引入 `parsson` + `yasson` 运行时 |
