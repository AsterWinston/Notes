# 1. 单个简单类型参数

```
简单类型包括：

byte short int long float double char
Byte Short Integer Long Float Double Character
String
java.util.Date
java.sql.Date
mybatis可以自动识别简单类型，底层可以自动推断使用哪一个preparedStatement.setXxx()方法。比如传String的值，就会调用preparedStatement.setString()方法。
SQL映射文件中的标签比较完整的写法是：

<select id="selectStudentByName" resultType="student" parameterType="string">
	select * from t_student where name = #{name, javaType=String, jdbcType=VARCHAR}
</select>

resultType和parameterType里面本来应该填写全限定类名，但是mybatis默认提供了别名（详细查看mybatis文档），所以这里直接填写类简名。另外，sql语句中的javaType，jdbcType，以及select标签中的parameterType属性，都是⽤来帮助mybatis进⾏类型确定的。不过这些配置多数是可以省略的。因为mybatis它有强⼤的⾃动类型推断机制。

jdbcType参考org.apache.ibatis.type.JdbcType
public enum JdbcType {
  /*
   * This is added to enable basic support for the ARRAY data type - but a custom type handler is still required
   */
  ARRAY(Types.ARRAY),

  BIT(Types.BIT),

  TINYINT(Types.TINYINT),

  SMALLINT(Types.SMALLINT),

  INTEGER(Types.INTEGER),

  BIGINT(Types.BIGINT),

  FLOAT(Types.FLOAT),

  REAL(Types.REAL),

  DOUBLE(Types.DOUBLE),

  NUMERIC(Types.NUMERIC),

  DECIMAL(Types.DECIMAL),

  CHAR(Types.CHAR),

  VARCHAR(Types.VARCHAR),

  LONGVARCHAR(Types.LONGVARCHAR),

  DATE(Types.DATE),

  TIME(Types.TIME),

  TIMESTAMP(Types.TIMESTAMP),

  BINARY(Types.BINARY),

  VARBINARY(Types.VARBINARY),

  LONGVARBINARY(Types.LONGVARBINARY),

  NULL(Types.NULL),

  OTHER(Types.OTHER),

  BLOB(Types.BLOB),

  CLOB(Types.CLOB),

  BOOLEAN(Types.BOOLEAN),

  CURSOR(-10), // Oracle

  UNDEFINED(Integer.MIN_VALUE + 1000),

  NVARCHAR(Types.NVARCHAR), // JDK6

  NCHAR(Types.NCHAR), // JDK6

  NCLOB(Types.NCLOB), // JDK6

  STRUCT(Types.STRUCT),

  JAVA_OBJECT(Types.JAVA_OBJECT),

  DISTINCT(Types.DISTINCT),

  REF(Types.REF),

  DATALINK(Types.DATALINK),

  ROWID(Types.ROWID), // JDK6

  LONGNVARCHAR(Types.LONGNVARCHAR), // JDK6

  SQLXML(Types.SQLXML), // JDK6

  DATETIMEOFFSET(-155), // SQL Server 2008

  TIME_WITH_TIMEZONE(Types.TIME_WITH_TIMEZONE), // JDBC 4.2 JDK8

  TIMESTAMP_WITH_TIMEZONE(Types.TIMESTAMP_WITH_TIMEZONE); // JDBC 4.2 JDK8

  public final int TYPE_CODE;
  private static final Map<Integer, JdbcType> codeLookup = new HashMap<>();

  static {
    for (JdbcType type : JdbcType.values()) {
      codeLookup.put(type.TYPE_CODE, type);
    }
  }

  JdbcType(int code) {
    this.TYPE_CODE = code;
  }

  public static JdbcType forCode(int code) {
    return codeLookup.get(code);
  }

}

javaType：可以省略
jdbcType：可以省略
parameterType：可以省略
```

# 2. map参数
```
这种⽅式是⼿动封装Map集合，将每个条件以key和value的形式存放到集合中。然后在使⽤的时候通过#{map集合的key}来取值。
```

# 3. 实体类参数
```
这⾥需要注意的是：#{} ⾥⾯写的是属性名字。这个属性名其本质上是：set/get⽅法名去掉set/get之后的名字。
```

# 4. 多参数
```
对于多个参数，mybatis底层会创建⼀个map集合，以arg0/param1为key，传入的参数为 value
例如：
List<Student> students = mapper.selectByNameAndSex("张三", '⼥')
//底层封装成了这样的map集合，其中arg从0开始，param从1开始

Map<String,Object> map = new HashMap<>();
map.put("arg0", name);
map.put("arg1", sex);
map.put("param1", name);
map.put("param2", sex);

<select id="selectByNameAndSex">
	select * from t_student where name = #{arg0} and sex = #{arg1}
</select>

```

# 5. @Param注解（命名参数）
```
mybatis提供的key可读性太差，可以使用@Param注解自己指定key的名字

List<Student> selectByNameAndAge(@Param(value="name") String name, @Param("age") int age);

<select id="selectByNameAndAge">
	select * from t_student where name = #{name} and sex = #{age}
</select>

注意：@Param替换的是map集合中arg0，arg1… 另外的param1，param2…还是可以继续使用的。
```

# 6. @Param源码分析
```
反射机制可以做到获取方法的形参名字，但是为何Mybatis使用注解标注形参的原因：MyBatis 早期版本发布时，Java 8 的 -parameters 编译选项还不存在，所以它必须依赖于其他机制。即使在 Java 8 普及之后，MyBatis 仍然选择继续使用注解和约定（例如 param1, param2），主要是出于以下原因

历史兼容性与可靠性（Historical Compatibility & Reliability）
诞生时间早于 Java 8: MyBatis (及其前身 iBATIS) 在 Java 8 发布之前就已经成熟并广泛使用了。在 Java 8 之前，获取形参名称的唯一标准方法是使用复杂的字节码操作库或特殊的调试信息，这既不可靠也不跨平台。因此，MyBatis 必须采用基于注解的显式命名方案。

首先，不依赖编译参数: Java 8 的反射机制依赖于开发者在编译时设置 -parameters 标志。如果开发者忘记设置这个标志，那么 MyBatis 运行时将无法获取到正确的参数名，导致参数绑定静默失败或抛出令人困惑的异常。
注解是明确的契约: 使用 @Param("userId") 是一种明确的编程契约，它明确告诉 MyBatis：“请使用 userId 这个名字来绑定 SQL 中的参数。”这比依赖外部的编译设置要可靠得多。

其次，避免代码混淆（Avoid Obfuscation Issues）
在许多生产环境中，代码会经过**混淆（Obfuscation）**处理，以保护知识产权或减小体积。
混淆器的影响: 混淆器通常会更改类、方法和局部变量的名称。虽然 -parameters 标记会保留参数名称，但在某些激进的混淆设置下，这些名称仍有可能被更改或不可用。
注解的保护: 注解作为元数据，通常被保留下来，为 MyBatis 提供了一个稳定且不受混淆影响的命名来源。

```

# 7. ParamNameResolver 源码分析

```java
/**
 * ParamNameResolver 负责解析 Mapper 接口方法的参数名
 * 源码：org.apache.ibatis.reflection.ParamNameResolver
 *
 * 它是 MapperMethod.MethodSignature 中使用的核心组件，
 * 决定了 SQL 中可通过什么名字引用方法参数。
 *
 * 参数名确定优先级（从高到低）：
 *   1. @Param 注解指定的值（最优先，最可靠）
 *   2. Java 8 -parameters 编译选项保留的实际参数名
 *   3. 序号命名：arg0, arg1, arg2... 和 param1, param2, param3...
 *
 * getNamedParams(Object[] args) 方法返回一个 Map：
 *   - 如果只有一个参数且没有 @Param：
 *     → 直接返回参数本身（不包装为 Map）
 *   - 如果有多个参数或使用了 @Param：
 *     → 返回包装的 Map，包含以下键：
 *       - @Param 指定的参数名（如 @Param("name") → "name"）
 *       - param1, param2, param3...（兼容旧版本）
 *       - 如果开启了 useActualParamName，还有实际参数名
 */
public class ParamNameResolver {

    private static final String GENERIC_NAME_PREFIX = "param";

    // 方法参数名列表（按顺序）
    private final SortedMap<Integer, String> names;

    // 是否使用实际参数名（Java 8+）
    private boolean hasParamAnnotation;

    public ParamNameResolver(Configuration config, Method method) {
        // 获取方法参数类型列表
        final Class<?>[] paramTypes = method.getParameterTypes();
        // 获取方法参数上的所有注解
        final Annotation[][] paramAnnotations = method.getParameterAnnotations();
        final SortedMap<Integer, String> map = new TreeMap<>();

        int paramCount = paramAnnotations.length;
        for (int paramIndex = 0; paramIndex < paramCount; paramIndex++) {
            if (isSpecialParameter(paramTypes[paramIndex])) {
                // 跳过特殊类型参数（RowBounds / ResultHandler）
                continue;
            }
            String name = null;
            // ★ 1. 优先使用 @Param 注解指定的值 ★
            for (Annotation annotation : paramAnnotations[paramIndex]) {
                if (annotation instanceof Param) {
                    hasParamAnnotation = true;
                    name = ((Param) annotation).value();
                    break;
                }
            }
            if (name == null) {
                // ★ 2. 如果开启了 useActualParamName（默认 true），获取实际参数名 ★
                if (config.isUseActualParamName()) {
                    name = getActualParamName(method, paramIndex);
                }
                // ★ 3. 如果还是 null → 使用序号 ★
                if (name == null) {
                    name = String.valueOf(map.size());
                }
            }
            map.put(paramIndex, name);
        }
        names = Collections.unmodifiableSortedMap(map);
    }

    /**
     * 将方法参数数组转换为命名参数
     *
     * @return 如果有 @Param → 返回 Map<String, Object>
     *         如果单参数无 @Param → 返回参数本身
     *         如果无参数 → 返回 null
     */
    public Object getNamedParams(Object[] args) {
        final int paramCount = names.size();
        if (args == null || paramCount == 0) {
            return null;
        } else if (!hasParamAnnotation && paramCount == 1) {
            // ★ 单参数无 @Param → 直接返回参数本身 ★
            Object value = args[names.firstKey()];
            return wrapToMapIfCollection(value, names.get(names.firstKey()));
        } else {
            final Map<String, Object> param = new ParamMap<>();
            int i = 0;
            for (Map.Entry<Integer, String> entry : names.entrySet()) {
                // ★ 写入：@Param 名 → 值 ★
                param.put(entry.getValue(), args[entry.getKey()]);
                // ★ 写入：param1, param2, ... → 值（兼容 arg0, arg1 的另一种命名）★
                final String genericParamName = GENERIC_NAME_PREFIX + (i + 1);
                if (!names.containsValue(genericParamName)) {
                    param.put(genericParamName, args[entry.getKey()]);
                }
                i++;
            }
            return param;
        }
    }
}
```

## 7.1 getNamedParams 返回值示例

```java
// 示例 1：单参数无 @Param
User selectById(Long id);
// getNamedParams([1]) → 返回 1L（参数直接传递）

// 示例 2：多参数无 @Param
User selectByNameAndAge(String name, Integer age);
// getNamedParams(["张三", 25]) → 返回 Map {
//     "arg0" → "张三",
//     "param1" → "张三",
//     "arg1" → 25,
//     "param2" → 25
// }

// 示例 3：多参数带 @Param
User selectByNameAndAge(@Param("name") String name, @Param("age") Integer age);
// getNamedParams(["张三", 25]) → 返回 Map {
//     "name" → "张三",
//     "param1" → "张三",
//     "age" → 25,
//     "param2" → 25
// }

// 示例 4：参数为空
List<User> selectAll();
// getNamedParams(null) → 返回 null
```