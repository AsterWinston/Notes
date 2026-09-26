# 1.使用 SqlSession 的 getMapper 方法

```
通过 SqlSession.getMapper(XXMapper.class) 可以获得 Mapper 接口的动态代理对象。
底层基于 JDK 动态代理（java.lang.reflect.Proxy），不需要手动编写 DAO 实现类。
```

## 1.1 要求

```
1. XxxMapper.xml 的 namespace 必须是 Mapper 接口的全限定类名
2. XxxMapper.xml 中 SQL 语句的 id 必须和 Mapper 接口中方法的名字一致
3. XxxMapper.xml 已经正确配置到 mybatis-config.xml 的 <mappers> 中
```

## 1.2 示例

```java
// 传统方式（需要手动写 DaoImpl）
SqlSession sqlSession = factory.openSession();
List<Car> cars = sqlSession.selectList("com.test.mapper.CarMapper.selectAll");

// getMapper 方式（无需实现类）
SqlSession sqlSession = factory.openSession();
CarMapper mapper = sqlSession.getMapper(CarMapper.class);
List<Car> cars = mapper.selectAll();  // 直接调用接口方法！
```

## 1.3 对比传统方式

```
传统方式的问题：
  1. 需要手写 DaoImpl 实现类，代码冗余
  2. sqlId 是字符串硬编码，容易写错且 IDE 无法检查
  3. 返回值需要强制类型转换
  4. 增加/删除方法需要修改多个地方

getMapper 方式的优势：
  1. 不需要手写实现类，MyBatis 通过动态代理自动生成
  2. 调用的是 Java 接口方法，有编译期类型检查
  3. 返回值自动推断类型
  4. 支持 IDE 代码导航和重构
```

# 2.getMapper 底层实现原理

## 2.1 总体流程

```
SqlSession.getMapper(CarMapper.class)
    │
    ▼
Configuration.getMapper(CarMapper.class, this)
    │
    ▼
MapperRegistry.getMapper(CarMapper.class, sqlSession)
    │
    ▼
从 knownMappers (Map<Class<?>, MapperProxyFactory<?>>) 中取出 factory
    │
    ▼
MapperProxyFactory.newInstance(sqlSession)
    │
    ▼
Proxy.newProxyInstance(...)  ← JDK 动态代理
    │
    ▼
返回代理对象 → mapper.selectAll() 时触发 MapperProxy.invoke()
```

## 2.2 MapperRegistry 源码分析

```java
/**
 * MapperRegistry — Mapper 接口注册中心
 *
 * MyBatis 源码：org.apache.ibatis.binding.MapperRegistry
 *
 * 核心职责：
 *   1. 注册 Mapper 接口与 MapperProxyFactory 的映射关系
 *   2. 提供 getMapper() 方法获取代理对象
 *   3. 在解析配置阶段，XMLMapperBuilder 会调用 addMapper() 注册
 */
public class MapperRegistry {

    private final Configuration config;

    // knownMappers 存储了所有已注册的 Mapper 接口
    // key = Mapper 接口的 Class 对象
    // value = 对应的 MapperProxyFactory（用于创建代理对象）
    private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new ConcurrentHashMap<>();

    public MapperRegistry(Configuration config) {
        this.config = config;
    }

    /**
     * 获取 Mapper 的动态代理对象
     *
     * @param type Mapper 接口的 Class
     * @param sqlSession 当前 SqlSession
     * @return Mapper 代理对象
     */
    @SuppressWarnings("unchecked")
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        // 从 knownMappers 中获取对应的 MapperProxyFactory
        final MapperProxyFactory<T> mapperProxyFactory =
            (MapperProxyFactory<T>) knownMappers.get(type);

        if (mapperProxyFactory == null) {
            throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
        }

        try {
            // 通过工厂创建代理实例
            return mapperProxyFactory.newInstance(sqlSession);
        } catch (Exception e) {
            throw new BindingException("Error getting mapper instance. Cause: " + e, e);
        }
    }

    /**
     * 注册 Mapper 接口
     * 在解析 <mappers> 配置时被调用
     */
    public <T> void addMapper(Class<T> type) {
        if (type.isInterface()) {
            if (hasMapper(type)) {
                throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
            }
            boolean loadCompleted = false;
            try {
                knownMappers.put(type, new MapperProxyFactory<>(type));
                // 解析注解式 SQL（如果使用了注解）
                MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
                parser.parse();
                loadCompleted = true;
            } finally {
                if (!loadCompleted) {
                    knownMappers.remove(type);
                }
            }
        }
    }
}
```

## 2.3 MapperProxyFactory 源码分析

```java
/**
 * MapperProxyFactory — Mapper 代理工厂
 *
 * MyBatis 源码：org.apache.ibatis.binding.MapperProxyFactory
 *
 * 每个 Mapper 接口对应一个 MapperProxyFactory 实例。
 * 通过 newInstance() 创建 JDK 动态代理对象。
 */
public class MapperProxyFactory<T> {

    // Mapper 接口的 Class
    private final Class<T> mapperInterface;

    // 方法缓存：Map<Method, MapperMethodInvoker>
    // 避免每次调用都重新创建 MapperMethod
    private final Map<Method, MapperMethodInvoker> methodCache = new ConcurrentHashMap<>();

    public MapperProxyFactory(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    /**
     * 创建 Mapper 接口的代理实例
     */
    @SuppressWarnings("unchecked")
    protected T newInstance(MapperProxy<T> mapperProxy) {
        return (T) Proxy.newProxyInstance(
            mapperInterface.getClassLoader(),
            new Class[]{mapperInterface},
            mapperProxy
        );
    }

    public T newInstance(SqlSession sqlSession) {
        final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
    }
}
```

## 2.4 MapperProxy.invoke() — 核心执行入口

```java
/**
 * MapperProxy — Mapper 接口的 JDK 动态代理 InvocationHandler
 *
 * MyBatis 源码：org.apache.ibatis.binding.MapperProxy
 *
 * 这是 MyBatis 中最重要的类之一。
 * 每次调用 mapper.xxx() 方法，都会进入 invoke() 方法。
 *
 * invoke() 内部流程：
 *   1. 判断是否为 Object 类的方法（toString、equals、hashCode）
 *      → 直接反射调用
 *   2. 判断是否为接口的 default 方法（Java 8+）
 *      → 使用 MethodHandle 调用
 *   3. 核心：通过 MapperMethod 执行 SQL
 *      → 从缓存获取 MapperMethod
 *      → 调用 MapperMethod.execute(sqlSession, args)
 */
public class MapperProxy<T> implements InvocationHandler, Serializable {

    private final SqlSession sqlSession;
    private final Class<T> mapperInterface;
    private final Map<Method, MapperMethodInvoker> methodCache;

    public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface,
                       Map<Method, MapperMethodInvoker> methodCache) {
        this.sqlSession = sqlSession;
        this.mapperInterface = mapperInterface;
        this.methodCache = methodCache;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            // 1. 如果是 Object 类的方法，直接调用
            if (Object.class.equals(method.getDeclaringClass())) {
                return method.invoke(this, args);
            }
            // 2. 如果是 default 方法，特殊处理（Java 8+）
            if (method.isDefault()) {
                return invokeDefaultMethod(proxy, method, args);
            }
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }

        // 3. 核心：获取或创建 MapperMethod，执行 SQL
        final MapperMethod mapperMethod = cachedMapperMethod(method);
        return mapperMethod.execute(sqlSession, args);
    }

    /**
     * 从缓存获取 MapperMethod，如果不存在则创建并缓存
     */
    private MapperMethod cachedMapperMethod(Method method) {
        return new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
    }
}
```

# 3.MapperMethod — 方法执行的核心逻辑

```java
/**
 * MapperMethod — 封装一个 Mapper 接口方法的完整信息
 *
 * MyBatis 源码：org.apache.ibatis.binding.MapperMethod
 *
 * 内部包含两个核心组件：
 *   1. SqlCommand    — 封装 SQL 命令信息（sqlId + SQL 类型）
 *   2. MethodSignature — 封装方法签名信息（返回值类型、参数处理方式）
 *
 * execute() 方法根据 SqlCommandType 分发到不同的执行逻辑
 */
public class MapperMethod {

    private final SqlCommand command;
    private final MethodSignature method;

    public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
        this.command = new SqlCommand(config, mapperInterface, method);
        this.method = new MethodSignature(config, mapperInterface, method);
    }

    /**
     * 核心执行方法
     *
     * 根据 command.getType() 分发：
     *   INSERT → sqlSession.insert()
     *   UPDATE → sqlSession.update()
     *   DELETE → sqlSession.delete()
     *   SELECT → 根据返回类型选择 selectOne/selectList/selectMap/selectCursor
     */
    public Object execute(SqlSession sqlSession, Object[] args) {
        Object result;
        switch (command.getType()) {
            case INSERT: {
                // 将方法参数转换为 SQL 命令参数
                Object param = method.convertArgsToSqlCommandParam(args);
                // 执行 insert，返回影响行数
                result = rowCountResult(sqlSession.insert(command.getName(), param));
                break;
            }
            case UPDATE: {
                Object param = method.convertArgsToSqlCommandParam(args);
                result = rowCountResult(sqlSession.update(command.getName(), param));
                break;
            }
            case DELETE: {
                Object param = method.convertArgsToSqlCommandParam(args);
                result = rowCountResult(sqlSession.delete(command.getName(), param));
                break;
            }
            case SELECT:
                if (method.returnsVoid() && method.hasResultHandler()) {
                    // 通过 ResultHandler 处理（流式查询）
                    executeWithResultHandler(sqlSession, args);
                    result = null;
                } else if (method.returnsMany()) {
                    // 返回 List
                    result = executeForMany(sqlSession, args);
                } else if (method.returnsMap()) {
                    // 返回 Map
                    result = executeForMap(sqlSession, args);
                } else if (method.returnsCursor()) {
                    // 返回 Cursor（游标）
                    result = executeForCursor(sqlSession, args);
                } else {
                    // 默认：返回单个对象
                    Object param = method.convertArgsToSqlCommandParam(args);
                    result = sqlSession.selectOne(command.getName(), param);

                    // 处理返回值为 Optional 的情况
                    if (method.returnsOptional() &&
                        (result == null || !method.getReturnType().equals(result.getClass()))) {
                        result = Optional.ofNullable(result);
                    }
                }
                break;
            case FLUSH:
                // 刷新缓存（批处理时使用）
                result = sqlSession.flushStatements();
                break;
            default:
                throw new BindingException("Unknown execution method for: " + command.getName());
        }

        // 如果返回值类型是基本类型且结果为 null，抛出异常
        if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
            throw new BindingException("Mapper method '" + command.getName()
                + "' attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
        }
        return result;
    }

    // ==================== 内部类 SqlCommand ====================

    /**
     * SqlCommand — 封装 SQL 命令的信息
     *
     * 核心字段：
     *   - name: sqlId = Mapper 接口全限定名 + "." + 方法名
     *   - type: SqlCommandType (INSERT/UPDATE/DELETE/SELECT/FLUSH/UNKNOWN)
     */
    public static class SqlCommand {

        private final String name;
        private final SqlCommandType type;

        public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
            // sqlId 的构建规则
            final String methodName = method.getName();
            final Class<?> declaringClass = method.getDeclaringClass();
            MappedStatement ms = resolveMappedStatement(mapperInterface, methodName, declaringClass, configuration);

            if (ms == null) {
                // 如果 MappedStatement 不存在，检查是否有 @Flush 注解
                if (method.isAnnotationPresent(Flush.class)) {
                    name = null;
                    type = SqlCommandType.FLUSH;
                } else {
                    throw new BindingException("Invalid bound statement (not found): "
                        + mapperInterface.getName() + "." + methodName);
                }
            } else {
                name = ms.getId();
                type = ms.getSqlCommandType();
                if (type == SqlCommandType.UNKNOWN) {
                    throw new BindingException("Unknown execution method for: " + name);
                }
            }
        }

        /**
         * 解析 MappedStatement
         * 先尝试用 method.getDeclaringClass() 的 MappedStatement
         * 如果找不到，再尝试 mapperInterface（处理继承关系）
         */
        private MappedStatement resolveMappedStatement(Class<?> mapperInterface, String methodName,
            Class<?> declaringClass, Configuration configuration) {
            String statementId = mapperInterface.getName() + "." + methodName;
            if (configuration.hasStatement(statementId)) {
                return configuration.getMappedStatement(statementId);
            }
            if (mapperInterface.equals(declaringClass)) {
                return null;
            }
            for (Class<?> superInterface : mapperInterface.getInterfaces()) {
                if (declaringClass.isAssignableFrom(superInterface)) {
                    MappedStatement ms = resolveMappedStatement(superInterface, methodName, declaringClass, configuration);
                    if (ms != null) return ms;
                }
            }
            return null;
        }

        public String getName() { return name; }
        public SqlCommandType getType() { return type; }
    }

    // ==================== 内部类 MethodSignature ====================

    /**
     * MethodSignature — 封装方法签名信息
     *
     * 核心职责：
     *   1. 判断返回类型（void/List/Map/Cursor/单个对象/Optional）
     *   2. 处理参数（参数命名、参数封装）
     *   3. convertArgsToSqlCommandParam() — 将方法参数转换为 SQL 参数
     */
    public static class MethodSignature {

        private final boolean returnsMany;
        private final boolean returnsMap;
        private final boolean returnsVoid;
        private final boolean returnsCursor;
        private final boolean returnsOptional;
        private final Class<?> returnType;
        private final String mapKey;
        private final Integer resultHandlerIndex;
        private final Integer rowBoundsIndex;
        private final ParamNameResolver paramNameResolver;

        public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
            // 分析返回类型
            Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
            if (resolvedReturnType instanceof Class<?>) {
                this.returnType = (Class<?>) resolvedReturnType;
            } else if (resolvedReturnType instanceof ParameterizedType) {
                this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
            } else {
                this.returnType = method.getReturnType();
            }

            this.returnsVoid = void.class.equals(this.returnType);
            this.returnsMany = configuration.getObjectFactory().isCollection(this.returnType)
                || this.returnType.isArray();
            this.returnsCursor = Cursor.class.equals(this.returnType);
            this.returnsOptional = Optional.class.equals(this.returnType);
            this.mapKey = getMapKey(method);
            this.returnsMap = this.mapKey != null;
            this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
            this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
            this.paramNameResolver = new ParamNameResolver(configuration, method);
        }

        /**
         * 将方法的参数数组转换为 SQL 可用的参数对象
         */
        public Object convertArgsToSqlCommandParam(Object[] args) {
            return paramNameResolver.getNamedParams(args);
        }
    }
}
```

# 4.Mapper 接口代理的完整执行链路

```
用户代码: carMapper.selectById(1);

  ┌─────────────────────────────────────────────────────────────────────────┐
  │ Step 1: MapperProxy.invoke()                                             │
  │   - 判断方法是否 Object 类的方法 → 否                                      │
  │   - 获取 MapperMethod（从缓存或新建）                                      │
  │   - 调用 mapperMethod.execute(sqlSession, args)                          │
  └─────────────────────────────────────────────────────────────────────────┘
      │
      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │ Step 2: MapperMethod.execute()                                           │
  │   - command.getType() == SELECT                                          │
  │   - method.returnsMany() == false, method.returnsMap() == false          │
  │   - 调用 sqlSession.selectOne("com.test.mapper.CarMapper.selectById", 1) │
  └─────────────────────────────────────────────────────────────────────────┘
      │
      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │ Step 3: DefaultSqlSession.selectOne()                                    │
  │   - 从 Configuration 获取 MappedStatement                                │
  │   - 委托给 executor.query(ms, parameter, RowBounds, Executor.NO_RESULT)  │
  └─────────────────────────────────────────────────────────────────────────┘
      │
      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │ Step 4: CachingExecutor.query() (如果有二级缓存)                           │
  │   - 获取 BoundSql（替换 #{} 后的最终 SQL）                                 │
  │   - 创建 CacheKey（计算缓存 key）                                         │
  │   - 查询二级缓存 → 命中则直接返回                                          │
  │   - 未命中 → 委托给 BaseExecutor.query()                                 │
  └─────────────────────────────────────────────────────────────────────────┘
      │
      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │ Step 5: BaseExecutor.query()                                             │
  │   - 查询一级缓存（localCache）→ 命中则直接返回                              │
  │   - 未命中 → 调用 doQuery()（子类实现）                                    │
  │   - doQuery() 返回后，将结果存入一级缓存                                    │
  └─────────────────────────────────────────────────────────────────────────┘
      │
      ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │ Step 6: SimpleExecutor.doQuery()                                         │
  │   - 创建 StatementHandler（RoutingStatementHandler）                      │
  │   - 创建 ParameterHandler（DefaultParameterHandler）                      │
  │   - 创建 ResultSetHandler（DefaultResultSetHandler）                      │
  │   - 获取 JDBC Connection                                                 │
  │   - 创建 PreparedStatement，设置参数                                      │
  │   - 执行 executeQuery()                                                  │
  │   - 调用 ResultSetHandler.handleResultSets() 映射结果                     │
  └─────────────────────────────────────────────────────────────────────────┘
```

# 5.使用 getMapper 的完整示例

## 5.1 Mapper 接口

```java
package com.test.mapper;

import com.test.pojo.Car;
import java.util.List;

public interface CarMapper {
    int insert(Car car);
    int deleteById(Long id);
    int update(Car car);
    Car selectById(Long id);
    List<Car> selectAll();
}
```

## 5.2 Mapper.xml 映射文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!-- namespace 必须是接口的全限定类名 -->
<mapper namespace="com.test.mapper.CarMapper">

    <!-- id 必须和方法名一致 -->
    <insert id="insert">
        insert into t_car(car_num,brand,guide_price,produce_time,car_type)
        values(#{carNum},#{brand},#{guidePrice},#{produceTime},#{carType})
    </insert>

    <delete id="deleteById">
        delete from t_car where id = #{id}
    </delete>

    <update id="update">
        update t_car set
            car_num=#{carNum}, brand=#{brand},
            guide_price=#{guidePrice}, produce_time=#{produceTime},
            car_type=#{carType}
        where id=#{id}
    </update>

    <select id="selectById" resultType="com.test.pojo.Car">
        select id,car_num as carNum,brand,guide_price as guidePrice,
               produce_time as produceTime,car_type as carType
        from t_car where id=#{id}
    </select>

    <select id="selectAll" resultType="com.test.pojo.Car">
        select id,car_num as carNum,brand,guide_price as guidePrice,
               produce_time as produceTime,car_type as carType
        from t_car
    </select>
</mapper>
```

## 5.3 测试代码

```java
public class CarMapperTest {

    @Test
    public void testSelectAll() {
        SqlSession sqlSession = SqlSessionUtil.openSession();
        // 通过 getMapper 获取代理对象 — 不需要写实现类
        CarMapper mapper = sqlSession.getMapper(CarMapper.class);

        List<Car> cars = mapper.selectAll();
        cars.forEach(System.out::println);

        sqlSession.close();
    }

    @Test
    public void testInsert() {
        SqlSession sqlSession = SqlSessionUtil.openSession();
        CarMapper mapper = sqlSession.getMapper(CarMapper.class);

        Car car = new Car();
        car.setCarNum("9000");
        car.setBrand("凯美瑞");
        car.setGuidePrice(35.0);
        car.setProduceTime("2020-10-11");
        car.setCarType("燃油车");

        int count = mapper.insert(car);
        System.out.println("插入 " + count + " 条记录");
        sqlSession.commit();
        sqlSession.close();

        // 此时 car.getId() 可以使用 useGeneratedKeys 获取自动生成的主键
        System.out.println("自动生成的 id: " + car.getId());
    }
}
```

# 6.基于 Mapper 接口的包扫描配置

```
当项目中有大量 Mapper 接口时，一个个配置 <mapper resource=""/> 太繁琐。

方式一：使用 <package name="..."/> 批量扫描
<mappers>
    <package name="com.test.mapper"/>
</mappers>

要求：
  1. XxxMapper.xml 文件必须和 XxxMapper 接口同名
  2. XxxMapper.xml 必须放在和 XxxMapper 接口同一个目录下
     （如果接口在 src/main/java/com/test/mapper/ 下，
       XML 应在 src/main/resources/com/test/mapper/ 下）

方式二：使用 <mapper class="..."/> 逐个声明
<mappers>
    <mapper class="com.test.mapper.CarMapper"/>
    <mapper class="com.test.mapper.StudentMapper"/>
</mappers>

要求同方式一。

原理：当指定 class 属性时，MyBatis 会从类路径中加载同名 XML，
      同时用 MapperAnnotationBuilder 解析接口上的注解式 SQL
```

# 7.MapperProxy 与手写 DaoImpl 的本质区别

```
传统 DaoImpl 方式：
  - 编译期生成字节码（我们手写的 .java → .class）
  - 方法调用：直接调用实现类的方法

MyBatis MapperProxy 方式：
  - 运行时生成字节码（JDK 动态代理在内存中生成）
  - 方法调用：通过 InvocationHandler.invoke() 间接调用
  - 实际执行：MapperMethod → SqlSession → Executor → JDBC

       手写 DaoImpl                         MapperProxy
  ┌──────────────────────┐        ┌──────────────────────────┐
  │ 调用时期：编译期确定     │        │ 调用时期：运行时确定         │
  │ 对象类型：具体实现类     │        │ 对象类型：Proxy 子类        │
  │ 方法分发：JVM 直接调用   │   vs   │ 方法分发：invoke() 反射     │
  │ 代码维护：需手动修改     │        │ 代码维护：只改 Mapper.xml    │
  │ 类型安全：编译期检查     │        │ 类型安全：编译期检查        │
  └──────────────────────┘        └──────────────────────────┘

共同点：都需要 xml 映射文件来定义 SQL（注解方式除外）
不同点：不需要手写实现类，由 JDK 动态代理自动生成
```
