```
mybatis中也提供了注解式开发方式，采用注解可以减少Sql映射文件的配置。
当然，使用注解式开发的话，sql语句是写在java程序中的，这种方式也会给sql语句的维护带来成本。
原则：简单sql可以注解。复杂sql使用xml。
可以减少配置，但是有的配置无法用注释完成，如sql和include
```

# 1.@Insert

```java
public interface CarMapper {

    @Insert("insert into t_car values(null,#{carNum},#{brand},#{guidePrice},#{produceTime},#{carType})")
    int insert(Car car);

    // 使用 useGeneratedKeys 获取自增主键
    @Insert("insert into t_car(car_num,brand,guide_price,produce_time,car_type) " +
            "values(#{carNum},#{brand},#{guidePrice},#{produceTime},#{carType})")
    @Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "id")
    int insertWithGeneratedKey(Car car);

    // 使用 selectKey 获取主键（适用于不支持自增的数据库）
    @Insert("insert into t_car values(null,#{carNum},#{brand},#{guidePrice},#{produceTime},#{carType})")
    @SelectKey(
        statement = "select last_insert_id()",
        keyProperty = "id",
        before = false,
        resultType = Long.class
    )
    int insertWithSelectKey(Car car);
}
```

# 2.@Delete

```java
@Delete("delete from t_car where id = #{id}")
int deleteById(Long id);
```

# 3.@Update

```java
@Update("update t_car set car_num=#{carNum},brand=#{brand}," +
        "guide_price=#{guidePrice},produce_time=#{produceTime}," +
        "car_type=#{carType} where id=#{id}")
int update(Car car);
```

# 4.@Select

```java
public interface CarMapper {

    // 简单查询
    @Select("select * from t_car where id = #{id}")
    @Results({
        @Result(column = "id", property = "id", id = true),
        @Result(column = "car_num", property = "carNum"),
        @Result(column = "brand", property = "brand"),
        @Result(column = "guide_price", property = "guidePrice"),
        @Result(column = "produce_time", property = "produceTime"),
        @Result(column = "car_type", property = "carType")
    })
    Car selectById(Long id);

    // 查询所有
    @Select("select * from t_car")
    @Results(id = "carResultMap", value = {
        @Result(column = "id", property = "id", id = true),
        @Result(column = "car_num", property = "carNum"),
        @Result(column = "brand", property = "brand"),
        @Result(column = "guide_price", property = "guidePrice"),
        @Result(column = "produce_time", property = "produceTime"),
        @Result(column = "car_type", property = "carType")
    })
    List<Car> selectAll();

    // 复用 @Results（通过 id 引用）
    @Select("select * from t_car where brand = #{brand}")
    @ResultMap("carResultMap")  // 引用上面定义的 ID
    List<Car> selectByBrand(String brand);
}
```

# 5.@Results 和 @Result

```java
/**
 * @Results 注解详解
 *
 * 相当于 XML 中的 <resultMap> 标签
 *
 * 关键属性：
 *   - id: 给这个 Results 起一个标识，方便在其他方法中通过 @ResultMap 引用
 *   - value: @Result 数组，定义每个字段的映射
 *
 * @Result 注解详解
 *
 * 相当于 XML 中的 <result> 或 <id> 标签
 *
 * 关键属性：
 *   - id: 是否是主键（true 对应 <id>，false 对应 <result>）
 *   - column: 数据库列名
 *   - property: Java 对象属性名
 *   - javaType: Java 类型（可省略，MyBatis 会自动推断）
 *   - jdbcType: JDBC 类型（可省略）
 *   - one: @One 注解，一对一/多对一关联
 *   - many: @Many 注解，一对多关联
 *   - typeHandler: 自定义类型处理器
 */
public @interface Results {
    String id() default "";
    Result[] value() default {};
}

public @interface Result {
    boolean id() default false;
    String column() default "";
    String property() default "";
    Class<?> javaType() default void.class;
    JdbcType jdbcType() default JdbcType.UNDEFINED;
    Class<? extends TypeHandler> typeHandler() default UnknownTypeHandler.class;
    One one() default @One;
    Many many() default @Many;
}
```

# 6.@One 和 @Many 替代 association、collection

## 6.1 @One — 一对一 / 多对一关联

```java
/**
 * @One 对应 XML 中的 <association> 标签（一对一或多对一）
 *
 * 属性：
 *   - select: 关联查询的 MappedStatement id（全限定方法名）
 *   - fetchType: 加载方式（FetchType.LAZY: 懒加载, FetchType.EAGER: 立即加载, DEFAULT: 全局配置）
 */

// ========== 学生表查询，关联班级信息（多对一） ==========
public interface StudentMapper {

    @Select("select sid,sname,cid from t_stu where sid = #{sid}")
    @Results({
        @Result(column = "sid", property = "sid", id = true),
        @Result(column = "sname", property = "sname"),
        @Result(
            column = "cid",           // 传递给 select 方法的参数
            property = "clazz",       // Student 对象中的 Clazz 属性
            javaType = Clazz.class,   // 关联对象的类型
            one = @One(
                select = "com.test.mapper.ClazzMapper.selectByIdStepTwo",
                fetchType = FetchType.LAZY  // 开启懒加载
            )
        )
    })
    Student selectById(Integer sid);
}

// ========== 第二步：班级查询 ==========
public interface ClazzMapper {

    @Select("select cid,cname from t_clazz where cid = #{cid}")
    @Results({
        @Result(column = "cid", property = "cid", id = true),
        @Result(column = "cname", property = "cname")
    })
    Clazz selectByIdStepTwo(Integer cid);
}
```

```xml
<!-- XML 方式等价写法 -->
<resultMap id="studentResultMap" type="Student">
    <id property="sid" column="sid"/>
    <result property="sname" column="sname"/>
    <association property="clazz"
                 select="com.test.mapper.ClazzMapper.selectByIdStepTwo"
                 column="cid"
                 fetchType="lazy"/>
</resultMap>
<select id="selectById" resultMap="studentResultMap">
    select sid,sname,cid from t_stu where sid = #{sid}
</select>
```

## 6.2 @Many — 一对多关联

```java
/**
 * @Many 对应 XML 中的 <collection> 标签（一对多）
 *
 * 属性同 @One
 */

// ========== 班级查询，关联学生列表（一对多） ==========
public interface ClazzMapper {

    // 第一步：查询班级
    @Select("select cid,cname from t_clazz where cid = #{cid}")
    @Results({
        @Result(column = "cid", property = "cid", id = true),
        @Result(column = "cname", property = "cname"),
        @Result(
            column = "cid",             // 传给 select 的参数
            property = "stus",          // Clazz 对象中的 List<Student> 属性
            javaType = List.class,      // 集合类型
            many = @Many(
                select = "com.test.mapper.StudentMapper.selectByCidStepTwo",
                fetchType = FetchType.LAZY
            )
        )
    })
    Clazz selectByCollection(Integer cid);
}

// ========== 第二步：根据班级 id 查询学生列表 ==========
public interface StudentMapper {

    @Select("select sid,sname from t_stu where cid = #{cid}")
    @Results({
        @Result(column = "sid", property = "sid", id = true),
        @Result(column = "sname", property = "sname")
    })
    List<Student> selectByCidStepTwo(Integer cid);
}
```

```xml
<!-- XML 方式等价写法 -->
<resultMap id="clazzResultMap" type="Clazz">
    <id property="cid" column="cid"/>
    <result property="cname" column="cname"/>
    <collection property="stus"
                select="com.test.mapper.StudentMapper.selectByCidStepTwo"
                column="cid"
                fetchType="lazy"/>
</resultMap>
<select id="selectByCollection" resultMap="clazzResultMap">
    select cid,cname from t_clazz where cid = #{cid}
</select>
```

## 6.3 @One/@Many 的延迟加载原理

```
当使用 fetchType = FetchType.LAZY 时：

  1. MyBatis 创建代理对象（通过 Javassist 或 CGLIB）
     → Student 对象实际上是一个代理对象

  2. 当调用 student.getClazz() 时：
     → 代理对象拦截了 getter 方法调用
     → 检测到 clazz 属性还未加载
     → 执行关联的 SELECT 语句加载数据
     → 将加载的数据设置到 clazz 属性
     → 返回 clazz 对象

  3. 当调用 student.getSname() 时：
     → 直接返回值（sname 已经在第一步查询中加载）

  【关键源码位置】
  org.apache.ibatis.executor.loader.javassist.JavassistProxyFactory
  org.apache.ibatis.executor.loader.cglib.CglibProxyFactory

  代理工厂在 Configuration 中通过 ProxyFactory 配置：
    - JavassistProxyFactory（默认）
    - CglibProxyFactory
```

# 7.@SelectProvider — Provider 类替代动态 SQL

```
@SelectProvider/@InsertProvider/@UpdateProvider/@DeleteProvider 这四个注解，
允许将 SQL 构建逻辑写在单独的 Provider 类中。

这解决了两个问题：
  1. 注解中无法写复杂的动态 SQL（if/where/trim/foreach 等）
  2. 注解字符串过长不便维护

Provider 方法的签名必须严格遵守以下约定：
  方法名可以任意，但：
  - 返回值必须是 String（SQL 语句）
  - 参数可以是：无参、或者带与 Mapper 方法相同类型的参数
```

## 7.1 基本用法

```java
// ========== Mapper 接口 ==========
public interface CarMapper {

    @SelectProvider(type = CarSqlProvider.class, method = "selectByCondition")
    @ResultMap("carResultMap")
    List<Car> selectByCondition(@Param("brand") String brand,
                                @Param("guidePrice") Double guidePrice,
                                @Param("carType") String carType);

    @InsertProvider(type = CarSqlProvider.class, method = "insert")
    int insert(Car car);

    @UpdateProvider(type = CarSqlProvider.class, method = "update")
    int update(Car car);

    @DeleteProvider(type = CarSqlProvider.class, method = "deleteById")
    int deleteById(Long id);
}

// ========== SQL Provider 类 ==========
public class CarSqlProvider {

    /**
     * 动态构建查询 SQL
     *
     * 使用 SQL 工具类（org.apache.ibatis.jdbc.SQL）可以帮助构建 SQL
     * 避免字符串拼接的繁琐和 SQL 注入
     */
    public String selectByCondition(Map<String, Object> params) {
        // 获取参数
        String brand = (String) params.get("brand");
        Double guidePrice = (Double) params.get("guidePrice");
        String carType = (String) params.get("carType");

        // 方式一：使用 MyBatis 内置的 SQL 构建工具类
        return new SQL() {{
            SELECT("*");
            FROM("t_car");
            if (brand != null && !brand.isEmpty()) {
                WHERE("brand like concat('%',#{brand},'%')");
            }
            if (guidePrice != null) {
                WHERE("guide_price > #{guidePrice}");
            }
            if (carType != null && !carType.isEmpty()) {
                WHERE("car_type = #{carType}");
            }
        }}.toString();
    }

    public String insert(Car car) {
        return new SQL() {{
            INSERT_INTO("t_car");
            VALUES("car_num", "#{carNum}");
            VALUES("brand", "#{brand}");
            VALUES("guide_price", "#{guidePrice}");
            VALUES("produce_time", "#{produceTime}");
            VALUES("car_type", "#{carType}");
        }}.toString();
    }

    public String update(Car car) {
        return new SQL() {{
            UPDATE("t_car");
            SET("car_num = #{carNum}");
            SET("brand = #{brand}");
            SET("guide_price = #{guidePrice}");
            SET("produce_time = #{produceTime}");
            SET("car_type = #{carType}");
            WHERE("id = #{id}");
        }}.toString();
    }

    public String deleteById() {
        return new SQL() {{
            DELETE_FROM("t_car");
            WHERE("id = #{id}");
        }}.toString();
    }
}
```

## 7.2 SQL 工具类详解

```java
/**
 * org.apache.ibatis.jdbc.SQL 是 MyBatis 提供的一个 DSL 风格的 SQL 构建工具
 *
 * 核心方法：
 *   SELECT(String... columns)
 *   FROM(String table)
 *   WHERE(String condition)
 *   AND() / OR() — 追加条件
 *
 *   INSERT_INTO(String table)
 *   VALUES(String column, String value)
 *   INTO_COLUMNS(String... columns)    — 批量列
 *   INTO_VALUES(String... values)      — 批量值
 *
 *   UPDATE(String table)
 *   SET(String sets)
 *
 *   DELETE_FROM(String table)
 *
 *   INNER_JOIN(String join) / LEFT_OUTER_JOIN / RIGHT_OUTER_JOIN
 *   ORDER_BY(String orderBy)
 *   GROUP_BY(String groupBy)
 *   HAVING(String having)
 *   LIMIT(String limit) / OFFSET(String offset)
 *
 * toString() 返回构建好的 SQL 语句
 */
```

## 7.3 更强大的 Provider 方式

```java
// ========== Mapper 接口 ==========
public interface CarMapper {

    // 批量插入（使用 foreach 的替代方案）
    @InsertProvider(type = CarSqlProvider.class, method = "insertBatch")
    int insertBatch(@Param("cars") List<Car> cars);
}

// ========== Provider 类 ==========
public class CarSqlProvider {

    /**
     * 批量插入 SQL 构建
     *
     * Provider 方法参数类型必须与 Mapper 方法参数一致，
     * 但可以使用 Map 类型统一接收（MyBatis 会自动将参数包装）
     */
    public String insertBatch(Map<String, Object> params) {
        @SuppressWarnings("unchecked")
        List<Car> cars = (List<Car>) params.get("cars");

        StringBuilder sql = new StringBuilder();
        sql.append("insert into t_car(car_num,brand,guide_price,produce_time,car_type) values ");

        for (int i = 0; i < cars.size(); i++) {
            sql.append("(");
            sql.append("#{cars[").append(i).append("].carNum},");
            sql.append("#{cars[").append(i).append("].brand},");
            sql.append("#{cars[").append(i).append("].guidePrice},");
            sql.append("#{cars[").append(i).append("].produceTime},");
            sql.append("#{cars[").append(i).append("].carType}");
            sql.append(")");
            if (i < cars.size() - 1) {
                sql.append(",");
            }
        }

        return sql.toString();
    }
}
```

## 7.4 多 Provider 参数

```java
// MyBatis 3.5.5+ 支持给 Provider 方法传递额外参数

// ========== Mapper 接口 ==========
public interface UserMapper {

    // 传统方式：参数通过 Map 方式传给 Provider
    @SelectProvider(type = UserSqlProvider.class, method = "getUsersByName")
    List<User> getUsersByName(String name);

    // 新增方式：ProviderMethodResolver 自动匹配方法
    // （接口名 + 方法名 → Provider 中的同名方法）
}
```

# 8.@CacheNamespace — 注解配置二级缓存

```java
/**
 * @CacheNamespace 对应 XML 中的 <cache/> 标签
 * 标注在 Mapper 接口上，启用该接口的二级缓存
 *
 * 属性：
 *   - implementation:  缓存实现类，默认 PerpetualCache
 *   - eviction:        淘汰策略，默认 LRU（LRU/FIFO/SOFT/WEAK）
 *   - flushInterval:   刷新间隔（毫秒），默认 0（不刷新）
 *   - size:            缓存最大对象数，默认 1024
 *   - readWrite:       是否可读写，默认 true
 *                        true: 返回缓存对象的副本（安全）
 *                        false: 返回缓存对象的引用（高性能，但要注意线程安全）
 *   - blocking:        是否阻塞，默认 false
 *                        true: 缓存未命中时阻塞其他线程（防止缓存击穿）
 *   - properties:      自定义属性
 */

// 使用默认配置开启二级缓存
@CacheNamespace
public interface CarMapper {
    @Select("select * from t_car where id = #{id}")
    Car selectById(Long id);
}

// 自定义缓存配置
@CacheNamespace(
    implementation = PerpetualCache.class,
    eviction = LruCache.class,
    flushInterval = 600000,  // 10分钟
    size = 512,
    readWrite = true,
    blocking = true
)
public interface StudentMapper {
    @Select("select * from t_stu where sid = #{sid}")
    Student selectById(Integer sid);
}

// 使用 EhCache 作为二级缓存实现
@CacheNamespace(implementation = org.mybatis.caches.ehcache.EhcacheCache.class)
public interface BookMapper {
    @Select("select * from t_book where id = #{id}")
    Book selectById(Long id);
}
```

```xml
<!-- XML 方式等价写法 -->
<cache
    eviction="LRU"
    flushInterval="600000"
    size="512"
    readOnly="false"
    blocking="true"/>

<!-- 使用 EhCache -->
<cache type="org.mybatis.caches.ehcache.EhcacheCache"/>
```

## 8.1 二级缓存的注意事项

```
1. 所有 select 语句都会被缓存
2. 所有 insert/update/delete 语句都会刷新缓存
3. 二级缓存是 namespace 级别的（一个 XxxMapper.xml 或一个 Mapper 接口）
4. 缓存的实体类必须实现 Serializable 接口
5. 在多个 namespace 中操作同一张表时要小心缓存不一致问题

缓存的刷新时机：
  - 当同一个 namespace 下执行了 insert/update/delete 语句
  - 或者 flushInterval 时间到达
  - 或者手动调用 sqlSession.clearCache()

使用 <cache-ref namespace="com.test.mapper.OtherMapper"/>
可以在不同 namespace 间共享同一个缓存
```

# 9.@Options — 配置 SQL 执行选项

```java
public interface CarMapper {

    @Insert("insert into t_car values(null,#{carNum},#{brand},#{guidePrice},#{produceTime},#{carType})")
    @Options(
        useGeneratedKeys = true,   // 使用自动生成的主键
        keyProperty = "id",        // 主键对应的属性
        keyColumn = "id",          // 主键列名
        flushCache = Options.FlushCachePolicy.TRUE,  // 刷新缓存
        timeout = 5,               // 超时时间（秒）
        useCache = true            // 是否使用二级缓存
    )
    int insert(Car car);

    @Select("select * from t_car where id = #{id}")
    @Options(
        flushCache = Options.FlushCachePolicy.FALSE,  // 不刷新缓存
        useCache = false         // 不使用二级缓存（即使全局启用了）
    )
    Car selectById(Long id);
}
```

# 10.@Flush — 刷新批处理语句

```java
/**
 * @Flush 注解标记的方法用于刷新批处理中的语句
 * 通常与 BatchExecutor 配合使用
 */
public interface CarMapper {

    @Flush
    List<BatchResult> flush();
}
```

# 11.@Lang — 指定语言驱动

```java
/**
 * @Lang 用于指定 SQL 解析的语言驱动
 * 默认是 XMLLanguageDriver，也可以自定义
 *
 * 例如，使用自定义的 SimpleLanguageDriver 来简化 SQL 编写
 */
public interface CarMapper {

    @Lang(XMLLanguageDriver.class)  // 默认，可省略
    @Select("select * from t_car where id = #{id}")
    Car selectById(Long id);
}
```

# 12.@ConstructorArgs — 构造器参数映射

```java
/**
 * @ConstructorArgs 和 @Arg 用于映射构造器参数
 * 当 POJO 没有无参构造器时使用
 */
public interface UserMapper {

    @Select("select id,name,age from t_user where id = #{id}")
    @ConstructorArgs({
        @Arg(column = "id", javaType = Long.class, id = true),
        @Arg(column = "name", javaType = String.class),
        @Arg(column = "age", javaType = Integer.class)
    })
    User selectById(Long id);  // User 必须有一个 User(Long id, String name, Integer age) 构造器
}
```

# 13.@MapKey — 返回 Map 时的 key 指定

```java
public interface CarMapper {

    @Select("select * from t_car")
    @Results(id = "carResultMap", value = {
        @Result(column = "id", property = "id", id = true),
        @Result(column = "car_num", property = "carNum"),
        @Result(column = "brand", property = "brand"),
        @Result(column = "guide_price", property = "guidePrice"),
        @Result(column = "produce_time", property = "produceTime"),
        @Result(column = "car_type", property = "carType")
    })
    @MapKey("id")  // 用 id 作为 Map 的 key
    Map<Long, Car> selectAllAsMap();
    // 返回: {1: Car{id=1,...}, 2: Car{id=2,...}, ...}
}
```

# 14.@TypeDiscriminator — 鉴别器

```java
/**
 * @TypeDiscriminator 用于根据某一列的值决定使用哪种结果映射
 * 相当于 XML 中的 <discriminator> 标签
 */
public interface VehicleMapper {

    @Select("select * from t_vehicle where id = #{id}")
    @Results(id = "vehicleResultMap", value = {
        @Result(column = "id", property = "id", id = true),
        @Result(column = "name", property = "name"),
        // 根据 vehicle_type 列判断是 Car 还是 Truck
        @TypeDiscriminator(
            column = "vehicle_type",    // 鉴别列
            javaType = String.class,    // 鉴别列的 Java 类型
            cases = {
                @Case(value = "CAR", type = Car.class, results = {
                    @Result(column = "seat_count", property = "seatCount")
                }),
                @Case(value = "TRUCK", type = Truck.class, results = {
                    @Result(column = "load_capacity", property = "loadCapacity")
                })
            }
        )
    })
    Vehicle selectById(Long id);
}
```

# 15.注解 vs XML 选型指南

```
┌────────────────────┬──────────────────────────┬──────────────────────────┐
│       维度          │        注解方式            │        XML 方式           │
├────────────────────┼──────────────────────────┼──────────────────────────┤
│  SQL 可见性         │ SQL 和 Java 代码混在一起    │ SQL 集中在 XML 中，清晰    │
│  动态 SQL 支持       │ 需要 Provider 类辅助       │ 原生 if/where/trim/foreach│
│  SQL 可维护性       │ 修改需重新编译              │ 修改 XML 即可，无需编译     │
│  类型安全           │ 编译期检查                 │ 运行时检查                 │
│  开发效率           │ 简单 SQL 开发快            │ 复杂 SQL 表达能力强         │
│  代码可读性         │ Java 代码被 SQL 打断       │ Java 代码保持纯净          │
│  IDE 支持           │ 完整（重构、导航）          │ 需要 MyBatis 插件辅助       │
│  sql/include 复用   │ 不支持                     │ 完全支持                   │
│  适用场景           │ 简单的单表 CRUD            │ 复杂的多表关联、动态查询     │
└────────────────────┴──────────────────────────┴──────────────────────────┘

最佳实践：
  1. 简单 CRUD：使用注解（减少 XML 文件数量）
  2. 复杂查询 / 动态 SQL / 多表关联：使用 XML
  3. 可以在同一个接口中混合使用注解和 XML
     （XML 的 id 优先，如果有同名 XML 和注解，XML 覆盖注解）
```

# 16.注解方式执行原理

```
注解方式的 SQL 是如何被加载的？

1. 在 MapperRegistry.addMapper() 时：
   → 创建 MapperAnnotationBuilder
   → 解析接口上的所有 MyBatis 注解（@Select、@Insert 等）
   → 将注解中的 SQL 构建为 MappedStatement
   → 同时尝试加载同名的 XML 配置文件（如果存在）

2. MapperAnnotationBuilder.parse() 执行过程：
   → parseCache()          — 解析 @CacheNamespace
   → parseCacheRef()       — 解析 @CacheNamespaceRef
   → method.getAnnotations() — 遍历每个方法
     → 解析 @Select / @Insert / @Update / @Delete
     → 解析 @SelectKey
     → 解析 @Options
     → 解析 @ResultMap / @Results
     → 解析 @ConstructorArgs
     → 通过 LanguageDriver 创建 SqlSource
     → 构建 MappedStatement 并注册到 Configuration

3. 关键源码：org.apache.ibatis.builder.annotation.MapperAnnotationBuilder
   - getSqlSourceFromAnnotations() 方法负责从注解中提取 SQL
   - 内部使用 ProviderSqlSource（如果是 Provider 方式）
     或 DynamicSqlSource / RawSqlSource（普通注解）

4. 与 XML 的优先级：
   - 如果 XML 文件中定义了相同的 statementId：
     XML 中的配置会优先使用
   - 所以在混合使用时，XML 可以覆盖注解中的 SQL
```
