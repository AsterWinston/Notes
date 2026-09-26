# 1.TypeHandler 概述

```
TypeHandler 是 MyBatis 中负责 Java 类型与 JDBC 类型之间相互转换的组件。

当 MyBatis 执行以下操作时，都需要 TypeHandler 参与：
  1. 设置 PreparedStatement 参数时：Java 对象 → JDBC 类型 → PreparedStatement
  2. 获取 ResultSet 结果时：    ResultSet → JDBC 类型 → Java 对象

TypeHandler 的调用链路：
  ParameterHandler.setParameters()
    → typeHandler.setParameter(ps, i, value, jdbcType)     ← 写入参数
  
  DefaultResultSetHandler.getPropertyMappingValue()
    → typeHandler.getResult(rs, columnName)                ← 读出结果
```

# 2.TypeHandler 接口

```java
/**
 * TypeHandler 接口
 * 源码：org.apache.ibatis.type.TypeHandler
 */
public interface TypeHandler<T> {

    /**
     * 设置 PreparedStatement 参数
     * @param ps       PreparedStatement
     * @param i        参数位置（从 1 开始）
     * @param parameter Java 参数值
     * @param jdbcType JDBC 类型
     */
    void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;

    /**
     * 从 ResultSet 中按列名获取值
     */
    T getResult(ResultSet rs, String columnName) throws SQLException;

    /**
     * 从 ResultSet 中按列索引获取值
     */
    T getResult(ResultSet rs, int columnIndex) throws SQLException;

    /**
     * 从 CallableStatement 中获取存储过程输出参数
     */
    T getResult(CallableStatement cs, int columnIndex) throws SQLException;
}
```

# 3.BaseTypeHandler — TypeHandler 的抽象基类

```java
/**
 * BaseTypeHandler 实现了 TypeHandler 接口的模板方法
 * 源码：org.apache.ibatis.type.BaseTypeHandler
 *
 * 它处理了 null 值和异常捕获，子类只需实现具体的类型转换逻辑。
 */
public abstract class BaseTypeHandler<T> extends TypeReference<T> implements TypeHandler<T> {

    @Override
    public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
        if (parameter == null) {
            if (jdbcType == null) {
                throw new TypeException("JDBC requires that the JdbcType must be specified for all nullable parameters.");
            }
            try {
                // null 值统一设置为 JDBC NULL
                ps.setNull(i, jdbcType.TYPE_CODE);
            } catch (SQLException e) {
                throw new TypeException("Error setting null for parameter #" + i + " with JdbcType " + jdbcType + ". " + e.getMessage(), e);
            }
        } else {
            try {
                // ★ 非 null 值委托给子类的 setNonNullParameter() ★
                setNonNullParameter(ps, i, parameter, jdbcType);
            } catch (Exception e) {
                throw new TypeException("Error setting non null for parameter #" + i + " with JdbcType " + jdbcType + ". " + e.getMessage(), e);
            }
        }
    }

    // 子类需实现的具体方法
    public abstract void setNonNullParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;

    public abstract T getNullableResult(ResultSet rs, String columnName) throws SQLException;
    public abstract T getNullableResult(ResultSet rs, int columnIndex) throws SQLException;
    public abstract T getNullableResult(CallableStatement cs, int columnIndex) throws SQLException;
}
```

# 4.内置 TypeHandler

```
MyBatis 内置了大量的 TypeHandler，覆盖了常见类型：

┌────────────────────────┬──────────────────────────────────────┐
│   TypeHandler           │  处理的类型                           │
├────────────────────────┼──────────────────────────────────────┤
│  StringTypeHandler      │  String ↔ VARCHAR / CHAR             │
│  IntegerTypeHandler     │  Integer / int ↔ INTEGER / NUMERIC   │
│  LongTypeHandler        │  Long / long ↔ BIGINT               │
│  DoubleTypeHandler      │  Double / double ↔ DOUBLE            │
│  FloatTypeHandler       │  Float / float ↔ FLOAT              │
│  BooleanTypeHandler     │  Boolean / boolean ↔ BOOLEAN         │
│  ByteTypeHandler        │  Byte / byte ↔ TINYINT              │
│  ShortTypeHandler       │  Short / short ↔ SMALLINT           │
│  BigDecimalTypeHandler  │  BigDecimal ↔ DECIMAL / NUMERIC      │
│  BigIntegerTypeHandler  │  BigInteger ↔ NUMERIC                │
│  DateTypeHandler        │  java.util.Date ↔ TIMESTAMP          │
│  DateOnlyTypeHandler    │  java.util.Date ↔ DATE               │
│  TimeOnlyTypeHandler    │  java.util.Date ↔ TIME               │
│  SqlDateTypeHandler     │  java.sql.Date ↔ DATE                │
│  SqlTimeTypeHandler     │  java.sql.Time ↔ TIME                │
│  SqlTimestampTypeHandler│  java.sql.Timestamp ↔ TIMESTAMP       │
│  EnumTypeHandler        │  Enum ↔ 枚举名称 (VARCHAR)            │
│  EnumOrdinalTypeHandler │  Enum ↔ 枚举序号 (INTEGER)            │
│  ClobTypeHandler        │  String ↔ CLOB                       │
│  BlobTypeHandler        │  byte[] ↔ BLOB                       │
│  ArrayTypeHandler       │  Object ↔ ARRAY                      │
│  NStringTypeHandler     │  String ↔ NVARCHAR / NCHAR           │
│  NClobTypeHandler       │  String ↔ NCLOB                      │
│  ByteArrayTypeHandler   │  byte[] ↔ 兼容多种 JDBC 类型           │
│  ObjectTypeHandler      │ Object ↔ OTHER (序列化)               │
│  UnknownTypeHandler     │ 未知类型处理器（兜底）                   │
│  InstantTypeHandler     │ java.time.Instant (Java 8)           │
│  LocalDateTimeTypeHandler│ java.time.LocalDateTime (Java 8)    │
│  LocalDateTypeHandler   │ java.time.LocalDate (Java 8)         │
│  LocalTimeTypeHandler   │ java.time.LocalTime (Java 8)         │
│  YearTypeHandler        │ java.time.Year (Java 8)              │
│  MonthTypeHandler       │ java.time.Month (Java 8)             │
│  YearMonthTypeHandler   │ java.time.YearMonth (Java 8)         │
│  OffsetDateTimeTypeHandler│ java.time.OffsetDateTime (Java 8)  │
│  OffsetTimeTypeHandler  │ java.time.OffsetTime (Java 8)        │
│  ZonedDateTimeTypeHandler│ java.time.ZonedDateTime (Java 8)    │
│  JapaneseDateTypeHandler│ java.time.chrono.JapaneseDate (Java 8)│
└────────────────────────┴──────────────────────────────────────┘
```

# 5.TypeHandlerRegistry — 类型处理器注册中心

```java
/**
 * TypeHandlerRegistry 管理所有的 TypeHandler
 * 源码：org.apache.ibatis.type.TypeHandlerRegistry
 *
 * 核心功能：
 *   1. 注册内置的 TypeHandler（在构造函数中完成）
 *   2. 根据 Java 类型查找对应的 TypeHandler
 *   3. 根据 JdbcType 查找对应的 TypeHandler
 *   4. 处理泛型类型的 TypeHandler 注册
 *
 * TypeHandler 的查找策略（getTypeHandler）：
 *   1. 先在 TYPE_HANDLER_MAP（精确 JavaType → TypeHandler）中查找
 *   2. 未找到 → 在 ALL_TYPE_HANDLERS_MAP（父类映射）中查找
 *   3. 未找到 → 在 NULL_TYPE_HANDLER_MAP（null 类型映射）中查找
 *   4. 仍未找到 → 使用 UnknownTypeHandler 作为兜底
 */
public class TypeHandlerRegistry {

    // JDBC 类型 → TypeHandler 的映射
    private final Map<JdbcType, TypeHandler<?>> JDBC_TYPE_HANDLER_MAP = new EnumMap<>(JdbcType.class);

    // Java 类型 → TypeHandler 的映射（用于已知 Java 类型的注册）
    private final Map<Type, Map<JdbcType, TypeHandler<?>>> TYPE_HANDLER_MAP = new ConcurrentHashMap<>();

    // Java 类型 → TypeHandler（不考虑 JdbcType）
    private final Map<Class<?>, TypeHandler<?>> ALL_TYPE_HANDLERS_MAP = new HashMap<>();

    // null 值的 TypeHandler
    private final Map<JdbcType, TypeHandler<?>> NULL_TYPE_HANDLER_MAP = new HashMap<>();

    // UnknownTypeHandler (兜底)
    private final TypeHandler<Object> UNKNOWN_TYPE_HANDLER = new UnknownTypeHandler(this);

    /**
     * 构造函数中注册所有内置 TypeHandler
     */
    public TypeHandlerRegistry(Configuration configuration) {
        // String 类型
        register(String.class, new StringTypeHandler());
        register(String.class, JdbcType.CHAR, new StringTypeHandler());
        register(String.class, JdbcType.CLOB, new ClobTypeHandler());
        register(String.class, JdbcType.NCHAR, new NStringTypeHandler());
        register(String.class, JdbcType.NCLOB, new NClobTypeHandler());

        // Integer 类型
        register(Integer.class, new IntegerTypeHandler());
        register(int.class, new IntegerTypeHandler());
        register(Integer.class, JdbcType.INTEGER, new IntegerTypeHandler());

        // Long 类型
        register(Long.class, new LongTypeHandler());
        register(long.class, new LongTypeHandler());

        // ... 大量内置注册

        // 枚举类型（需要在具体使用时注册）
        register(Enum.class, new EnumTypeHandler());

        // Java 8 时间类型
        register(Instant.class, InstantTypeHandler.class);
        register(LocalDateTime.class, LocalDateTimeTypeHandler.class);
        register(LocalDate.class, LocalDateTypeHandler.class);
        register(LocalTime.class, LocalTimeTypeHandler.class);
        // ...
    }

    /**
     * 根据 Java 类型和 JdbcType 查找 TypeHandler
     */
    @SuppressWarnings("unchecked")
    public <T> TypeHandler<T> getTypeHandler(Class<T> type, JdbcType jdbcType) {
        if (type == null) {
            return null;
        }

        // 1. 精确匹配
        Map<JdbcType, TypeHandler<?>> jdbcHandlerMap = getJdbcHandlerMap(type);
        TypeHandler<?> handler = null;
        if (jdbcHandlerMap != null) {
            handler = jdbcHandlerMap.get(jdbcType);
            if (handler == null) {
                handler = jdbcHandlerMap.get(null); // 尝试无 JdbcType 的注册
            }
        }

        // 2. 泛型匹配（例如 Enum）
        if (handler == null && type.isEnum()) {
            handler = jdbcHandlerMap.get(null);
        }

        // 3. 兜底：UnknownTypeHandler
        if (handler == null) {
            handler = getInstance(type, null);
            if (handler == null) {
                handler = UNKNOWN_TYPE_HANDLER;
            }
        }

        return (TypeHandler<T>) handler;
    }
}
```

# 6.自定义 TypeHandler

## 6.1 示例：JSON 字段 TypeHandler

```java
/**
 * 将数据库中的 JSON 字符串自动转换为 Java 对象
 *
 * 使用场景：
 *   数据库中有一个 VARCHAR 字段存储 JSON 数据
 *   Java 实体类中对应的是一个 POJO 对象或 Map
 *
 * 例如数据库存储：
 *   "{"name":"张三","age":25,"hobbies":["读书","游泳"]}"
 * Java 对象中：
 *   private UserProfile profile; // UserProfile 对象
 */
@MappedTypes(UserProfile.class)   // 声明处理的 Java 类型
@MappedJdbcTypes(JdbcType.VARCHAR) // 声明处理的 JDBC 类型
public class JsonTypeHandler<T> extends BaseTypeHandler<T> {

    private final Class<T> type;

    public JsonTypeHandler(Class<T> type) {
        if (type == null) {
            throw new IllegalArgumentException("Type argument cannot be null");
        }
        this.type = type;
    }

    // 写：Java 对象 → JSON 字符串 → PreparedStatement
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i,
                                    T parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, JSON.toJSONString(parameter));
    }

    // 读：ResultSet → JSON 字符串 → Java 对象
    @Override
    public T getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String json = rs.getString(columnName);
        return parseJson(json);
    }

    @Override
    public T getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        String json = rs.getString(columnIndex);
        return parseJson(json);
    }

    @Override
    public T getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        String json = cs.getString(columnIndex);
        return parseJson(json);
    }

    private T parseJson(String json) {
        if (json == null || json.isEmpty()) {
            return null;
        }
        return JSON.parseObject(json, type);
    }
}
```

## 6.2 示例：枚举 TypeHandler（自定义）

```java
/**
 * 自定义枚举 TypeHandler — 按自定义字段值存储（非序号或名称）
 *
 * 例如有一个状态枚举：
 *   public enum OrderStatus {
 *       PENDING(0, "待处理"),
 *       PROCESSING(1, "处理中"),
 *       COMPLETED(2, "已完成"),
 *       CANCELLED(3, "已取消");
 *
 *       private final int code;
 *       private final String desc;
 *   }
 *
 * 希望在数据库中存储 code 字段的值（整数）
 */
@MappedTypes(OrderStatus.class)
@MappedJdbcTypes(JdbcType.INTEGER)
public class OrderStatusTypeHandler extends BaseTypeHandler<OrderStatus> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i,
                                    OrderStatus parameter, JdbcType jdbcType) throws SQLException {
        ps.setInt(i, parameter.getCode());
    }

    @Override
    public OrderStatus getNullableResult(ResultSet rs, String columnName) throws SQLException {
        int code = rs.getInt(columnName);
        return rs.wasNull() ? null : OrderStatus.fromCode(code);
    }

    @Override
    public OrderStatus getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        int code = rs.getInt(columnIndex);
        return rs.wasNull() ? null : OrderStatus.fromCode(code);
    }

    @Override
    public OrderStatus getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        int code = cs.getInt(columnIndex);
        return cs.wasNull() ? null : OrderStatus.fromCode(code);
    }
}
```

## 6.3 示例：加密 TypeHandler

```java
/**
 * 透明加/解密 TypeHandler
 * 数据库存储加密数据，Java 中使用明文
 */
@MappedJdbcTypes(JdbcType.VARCHAR)
public class EncryptTypeHandler extends BaseTypeHandler<String> {

    private static final String AES_KEY = "your-16-byte-key!";

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i,
                                    String parameter, JdbcType jdbcType) throws SQLException {
        // 写入数据库前加密
        ps.setString(i, AESUtils.encrypt(parameter, AES_KEY));
    }

    @Override
    public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String encrypted = rs.getString(columnName);
        return decryptIfNeeded(encrypted);
    }

    @Override
    public String getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        String encrypted = rs.getString(columnIndex);
        return decryptIfNeeded(encrypted);
    }

    @Override
    public String getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        String encrypted = cs.getString(columnIndex);
        return decryptIfNeeded(encrypted);
    }

    private String decryptIfNeeded(String encrypted) {
        if (encrypted == null || encrypted.isEmpty()) {
            return null;
        }
        return AESUtils.decrypt(encrypted, AES_KEY);
    }
}
```

# 7.TypeHandler 的配置方式

## 7.1 在 mybatis-config.xml 中配置

```xml
<configuration>
    <typeHandlers>
        <!-- 方式1：逐个注册（指定 alias 或 jdbcType） -->
        <typeHandler handler="com.test.handler.JsonTypeHandler"
                     javaType="com.test.pojo.UserProfile"
                     jdbcType="VARCHAR"/>

        <!-- 方式2：批量扫描 -->
        <package name="com.test.handler"/>
    </typeHandlers>
</configuration>
```

## 7.2 在 Spring Boot 中配置

```yaml
# application.yml
mybatis:
  type-handlers-package: com.test.handler
```

## 7.3 在 Mapper.xml 中局部指定

```xml
<!-- 方式1：在 resultMap 中指定 -->
<resultMap id="userResultMap" type="User">
    <result column="profile" property="profile"
            typeHandler="com.test.handler.JsonTypeHandler"/>
</resultMap>

<!-- 方式2：在 SQL 语句中通过 #{xxx, typeHandler=xxx} 指定 -->
<insert id="insert">
    insert into t_user(name, profile)
    values(#{name}, #{profile, typeHandler=com.test.handler.JsonTypeHandler})
</insert>
```

## 7.4 在 @Result 注解中指定

```java
@Select("select * from t_user where id = #{id}")
@Results({
    @Result(column = "id", property = "id", id = true),
    @Result(column = "name", property = "name"),
    @Result(column = "profile", property = "profile",
            typeHandler = JsonTypeHandler.class)
})
User selectById(Long id);
```

# 8.TypeHandler 的执行时机

```
写操作（INSERT/UPDATE）：
  ParameterHandler.setParameters()
    → 遍历 parameterMappings
    → 获取参数值
    → 获取 TypeHandler
    → typeHandler.setParameter(ps, index, value, jdbcType)
    → ps.setXxx(index, convertedValue)

读操作（SELECT）：
  DefaultResultSetHandler.handleResultSets()
    → 遍历每一行
    → ColumnHandler → TypeHandler.getResult(rs, columnName)
    → 获取转换后的值
    → 通过 MetaObject.setValue() 设置到结果对象
```

# 9.默认枚举处理器选择

```java
/**
 * MyBatis 3.4.5+ 改变了枚举的默认处理方式
 *
 * 之前：默认使用 EnumTypeHandler（存储枚举名称）
 * 现在：默认使用 EnumOrdinalTypeHandler（存储枚举序号）
 *
 * 可以通过 settings 修改：
 * <setting name="defaultEnumTypeHandler"
 *          value="org.apache.ibatis.type.EnumTypeHandler"/>
 *
 * EnumTypeHandler：    枚举.name()   ↔ VARCHAR
 * EnumOrdinalTypeHandler：枚举.ordinal() ↔ INTEGER
 *
 * 推荐：自定义枚举 TypeHandler（按自定义字段存储）
 */
```
