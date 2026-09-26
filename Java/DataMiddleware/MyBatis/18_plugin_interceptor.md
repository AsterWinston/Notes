# 1.MyBatis 插件/拦截器机制概述

```
MyBatis 的插件（Plugin）机制基于 JDK 动态代理和责任链模式，
允许开发者在不修改 MyBatis 源码的情况下，拦截核心组件的执行过程。

可拦截的四个对象（四大金刚）：
  1. Executor           — SQL 执行器（update、query、commit、rollback 等方法）
  2. StatementHandler   — 语句处理器（prepare、parameterize、update、query 等方法）
  3. ParameterHandler   — 参数处理器（setParameters 方法）
  4. ResultSetHandler   — 结果集处理器（handleResultSets、handleCursorResultSets 等方法）

工作原理：
  对目标对象生成 JDK 动态代理，在方法调用前后插入自定义逻辑。

架构概览：
  ┌──────────────────────────────────────────────────────────────┐
  │                     InterceptorChain                          │
  │   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
  │   │Plugin A │→ │Plugin B │→ │Plugin C │→ │  Target  │        │
  │   └─────────┘  └─────────┘  └─────────┘  └─────────┘        │
  │                                                              │
  │   每个 Plugin 是一个动态代理，层层包装目标对象                    │
  │   A 先拦截 → B 拦截 → C 拦截 → 目标方法执行 → C 后处理 → ...     │
  └──────────────────────────────────────────────────────────────┘
```

# 2.Interceptor 接口

```java
/**
 * Interceptor 接口 — 所有 MyBatis 插件必须实现的接口
 * 源码：org.apache.ibatis.plugin.Interceptor
 *
 * 三个需要实现的方法：
 *   - intercept(Invocation): 拦截逻辑的核心
 *   - plugin(Object target): 决定是否对该目标生成代理
 *   - setProperties(Properties): 接收配置属性
 */
public interface Interceptor {

    /**
     * 核心拦截方法
     *
     * @param invocation 包含被拦截方法的所有信息
     *   - getTarget(): 被代理的目标对象
     *   - getMethod(): 被拦截的方法
     *   - getArgs():   方法的参数
     *   - proceed():   调用目标方法（类似责任链的 next）
     */
    Object intercept(Invocation invocation) throws Throwable;

    /**
     * 对目标对象进行包装（生成代理）
     * 通常直接调用 Plugin.wrap(target, this)
     *
     * @param target 要包装的目标对象
     * @return 包装后的代理对象（或原对象，如果不拦截）
     */
    default Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    /**
     * 接收 XML 配置中的 <property> 属性
     */
    default void setProperties(Properties properties) { }
}
```

# 3.Invocation — 方法调用封装

```java
/**
 * Invocation 封装了一次方法调用的所有信息
 * 源码：org.apache.ibatis.plugin.Invocation
 */
public class Invocation {

    private final Object target;    // 被代理的目标对象
    private final Method method;    // 被调用的方法
    private final Object[] args;    // 方法的参数

    public Invocation(Object target, Method method, Object[] args) {
        this.target = target;
        this.method = method;
        this.args = args;
    }

    /**
     * 执行目标方法（让拦截链继续向下传播）
     *
     * 在自定义拦截器中：
     *   调用 invocation.proceed() → 执行被拦截的方法（或下一个拦截器）
     *   不调用 invocation.proceed() → 中断执行链
     */
    public Object proceed() throws InvocationTargetException, IllegalAccessException {
        return method.invoke(target, args);
    }
}
```

# 4.Plugin — 动态代理包装器

```java
/**
 * Plugin 是 JDK 动态代理的 InvocationHandler 实现
 * 源码：org.apache.ibatis.plugin.Plugin
 *
 * 核心逻辑：
 *   1. 根据 @Intercepts 注解判断是否拦截当前方法
 *   2. 如果拦截 → 调用 Interceptor.intercept(invocation)
 *   3. 如果不拦截 → 直接调用目标方法
 *
 * wrap() 方法：
 *   收集 @Intercepts 注解中声明的拦截信息
 *   使用 Proxy.newProxyInstance() 生成代理对象
 */
public class Plugin implements InvocationHandler {

    private final Object target;                  // 被代理的目标对象
    private final Interceptor interceptor;        // 拦截器
    private final Map<Class<?>, Set<Method>> signatureMap;  // 拦截签名映射

    public Plugin(Object target, Interceptor interceptor, Map<Class<?>, Set<Method>> signatureMap) {
        this.target = target;
        this.interceptor = interceptor;
        this.signatureMap = signatureMap;
    }

    /**
     * 对目标对象生成代理
     *
     * @param target 目标对象
     * @param interceptor 拦截器实例
     * @return 如果目标被拦截 → 返回代理对象; 否则 → 返回原对象
     */
    public static Object wrap(Object target, Interceptor interceptor) {
        // 1. 解析 @Intercepts 注解，获取签名映射
        Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);

        // 2. 获取目标对象的类型
        Class<?> type = target.getClass();

        // 3. 获取目标对象实现的所有接口 + 所有要拦截的接口的交集
        Class<?>[] interfaces = getAllInterfaces(type, signatureMap);

        // 4. 如果有交集的接口 → 生成代理; 否则 → 返回原对象
        if (interfaces.length > 0) {
            return Proxy.newProxyInstance(
                type.getClassLoader(),
                interfaces,
                new Plugin(target, interceptor, signatureMap)
            );
        }
        return target;
    }

    /**
     * JDK 动态代理的核心方法
     *
     * 执行流程：
     *   1. 检查方法是否在拦截签名中
     *   2. 如果在 → 创建 Invocation，调用 interceptor.intercept()
     *   3. 如果不在 → 直接反射调用目标方法
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            // ★ 判断当前方法是否在拦截签名中 ★
            Set<Method> methods = signatureMap.get(method.getDeclaringClass());
            if (methods != null && methods.contains(method)) {
                // 在拦截签名中 → 调用拦截器
                return interceptor.intercept(new Invocation(target, method, args));
            }
            // 不在拦截签名中 → 直接调用目标方法
            return method.invoke(target, args);
        } catch (Exception e) {
            throw ExceptionUtil.unwrapThrowable(e);
        }
    }

    /**
     * 从 @Intercepts 注解中解析拦截签名
     */
    private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
        Intercepts interceptsAnnotation = interceptor.getClass().getAnnotation(Intercepts.class);
        if (interceptsAnnotation == null) {
            throw new PluginException("No @Intercepts annotation was found in interceptor "
                + interceptor.getClass().getName());
        }
        Signature[] sigs = interceptsAnnotation.value();
        Map<Class<?>, Set<Method>> signatureMap = new HashMap<>();
        for (Signature sig : sigs) {
            Set<Method> methods = MapUtil.computeIfAbsent(signatureMap, sig.type(), k -> new HashSet<>());
            try {
                Method method = sig.type().getMethod(sig.method(), sig.args());
                methods.add(method);
            } catch (NoSuchMethodException e) {
                throw new PluginException("Could not find method on " + sig.type() + " named " + sig.method() + ". Cause: " + e, e);
            }
        }
        return signatureMap;
    }
}
```

# 5.@Intercepts 和 @Signature 注解

```java
/**
 * @Intercepts — 声明一个拦截器的拦截签名集合
 * 源码：org.apache.ibatis.plugin.Intercepts
 *
 * @Signature — 描述要拦截的方法
 * 源码：org.apache.ibatis.plugin.Signature
 *
 * 属性：
 *   - type:    要拦截的接口类型（Executor/StatementHandler/ParameterHandler/ResultSetHandler）
 *   - method:  要拦截的方法名
 *   - args:    方法的参数类型列表（用于精确匹配重载方法）
 */

// 示例：拦截 Executor 的 update 方法
@Intercepts({
    @Signature(
        type = Executor.class,                    // 拦截 Executor 接口
        method = "update",                        // update 方法
        args = {MappedStatement.class, Object.class}  // 参数类型
    )
})
public class ExamplePlugin implements Interceptor {
    // ...
}

// 多个拦截签名（拦截多个方法）
@Intercepts({
    @Signature(type = Executor.class, method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}),
    @Signature(type = Executor.class, method = "update",
        args = {MappedStatement.class, Object.class})
})
public class MultiInterceptor implements Interceptor {
    // ...
}
```

# 6.InterceptorChain — 拦截器链

```java
/**
 * InterceptorChain 管理所有注册的插件
 * 源码：org.apache.ibatis.plugin.InterceptorChain
 *
 * 在 Configuration 中作为一个字段存在：
 *   protected final InterceptorChain interceptorChain = new InterceptorChain();
 *
 * 当创建 Executor / StatementHandler / ParameterHandler / ResultSetHandler 时，
 * Configuration 会调用 interceptorChain.pluginAll(target) 进行包装。
 */
public class InterceptorChain {

    private final List<Interceptor> interceptors = new ArrayList<>();

    /**
     * 对所有目标应用所有拦截器
     *
     * 调用链：
     *   target (原始对象)
     *     → interceptor1.plugin(target)    → 代理1
     *       → interceptor2.plugin(代理1)   → 代理2
     *         → interceptor3.plugin(代理2) → 代理3 (最终返回)
     *
     * 执行时：
     *   调用 → 代理3.invoke() → interceptor3.intercept()
     *     → invocation.proceed() → 代理2.invoke() → interceptor2.intercept()
     *       → invocation.proceed() → 代理1.invoke() → interceptor1.intercept()
     *         → invocation.proceed() → target.method() (真正执行)
     */
    public Object pluginAll(Object target) {
        for (Interceptor interceptor : interceptors) {
            target = interceptor.plugin(target);  // 层层包装
        }
        return target;
    }

    public void addInterceptor(Interceptor interceptor) {
        interceptors.add(interceptor);
    }

    public List<Interceptor> getInterceptors() {
        return Collections.unmodifiableList(interceptors);
    }
}
```

# 7.四大对象的拦截时机

```java
/**
 * Configuration 中四大对象的创建位置：
 *
 * 1. Executor 的创建和拦截
 *    Configuration.newExecutor()
 *      → executor = new SimpleExecutor / ReuseExecutor / BatchExecutor
 *      → if (cacheEnabled) executor = new CachingExecutor(executor)
 *      → executor = (Executor) interceptorChain.pluginAll(executor)  ★ 拦截点
 *
 * 2. StatementHandler 的创建和拦截
 *    Configuration.newStatementHandler()
 *      → handler = new RoutingStatementHandler(...)
 *      → handler = (StatementHandler) interceptorChain.pluginAll(handler)  ★ 拦截点
 *
 * 3. ParameterHandler 的创建和拦截
 *    Configuration.newParameterHandler()
 *      → handler = new DefaultParameterHandler(...)
 *      → handler = (ParameterHandler) interceptorChain.pluginAll(handler)  ★ 拦截点
 *
 * 4. ResultSetHandler 的创建和拦截
 *    Configuration.newResultSetHandler()
 *      → handler = new DefaultResultSetHandler(...)
 *      → handler = (ResultSetHandler) interceptorChain.pluginAll(handler)  ★ 拦截点
 */
```

# 8.自定义插件完整示例

## 8.1 SQL 执行时间统计插件

```java
/**
 * 统计每条 SQL 的执行时间，超过阈值时打印警告
 */
@Intercepts({
    @Signature(
        type = StatementHandler.class,
        method = "query",
        args = {Statement.class, ResultHandler.class}
    ),
    @Signature(
        type = StatementHandler.class,
        method = "update",
        args = {Statement.class}
    )
})
public class SqlCostInterceptor implements Interceptor {

    // 慢 SQL 阈值（毫秒），可在 XML 中配置
    private long slowThreshold = 1000;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 1. 获取目标对象
        StatementHandler handler = (StatementHandler) invocation.getTarget();

        // 2. 通过反射获取 BoundSql（因为 BoundSql 不是 StatementHandler 接口方法的参数）
        MetaObject metaObject = SystemMetaObject.forObject(handler);
        String sql = metaObject.getValue("delegate.boundSql.sql").toString();

        // 3. 记录开始时间
        long start = System.currentTimeMillis();

        // 4. 执行原方法
        Object result;
        try {
            result = invocation.proceed();
        } finally {
            // 5. 记录结束后计算耗时
            long cost = System.currentTimeMillis() - start;
            if (cost >= slowThreshold) {
                System.err.println("========== 慢 SQL 警告 ==========");
                System.err.println("耗时: " + cost + "ms");
                System.err.println("SQL: " + sql);
                System.err.println("================================");
            } else {
                System.out.println("SQL 执行耗时: " + cost + "ms");
            }
        }
        return result;
    }

    @Override
    public void setProperties(Properties properties) {
        String threshold = properties.getProperty("slowThreshold");
        if (threshold != null) {
            this.slowThreshold = Long.parseLong(threshold);
        }
    }
}
```

## 8.2 自动分页插件（PageHelper 原理简化版）

```java
/**
 * 简化的分页插件 — 演示 PageHelper 的核心原理
 *
 * PageHelper 本质上就是通过拦截 Executor.query() 方法：
 *   1. 获取原 SQL 和参数
 *   2. 判断是否需要分页（从 ThreadLocal 获取分页参数）
 *   3. 如果需要分页：
 *      a. 执行 count 查询获取总记录数
 *      b. 在原 SQL 后面拼接 LIMIT 子句
 *      c. 执行分页 SQL
 *   4. 返回分页结果
 */
@Intercepts({
    @Signature(
        type = Executor.class,
        method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
    )
})
public class SimplePageInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();
        MappedStatement ms = (MappedStatement) args[0];
        Object parameter = args[1];
        RowBounds rowBounds = (RowBounds) args[2];

        // 如果不需要分页 → 直接执行原方法
        if (rowBounds == RowBounds.DEFAULT) {
            return invocation.proceed();
        }

        // 获取 BoundSql
        BoundSql boundSql = ms.getBoundSql(parameter);
        String originalSql = boundSql.getSql();

        // 1. 先查总记录数
        String countSql = "select count(1) from (" + originalSql + ") _temp";
        // ... 执行 count 查询

        // 2. 修改原 SQL，追加 LIMIT 子句
        String pageSql = originalSql + " limit " + rowBounds.getOffset() + ", " + rowBounds.getLimit();

        // 3. 通过反射修改 BoundSql 中的 SQL
        MetaObject metaObject = SystemMetaObject.forObject(boundSql);
        metaObject.setValue("sql", pageSql);

        // 4. 执行原方法
        return invocation.proceed();
    }
}
```

## 8.3 敏感数据加密插件

```java
/**
 * 对数据库中的敏感字段进行自动加解密
 *
 * 拦截 StatementHandler 的参数设置（加密）和结果集处理（解密）
 */
@Intercepts({
    @Signature(type = ParameterHandler.class, method = "setParameters", args = {PreparedStatement.class})
})
public class EncryptInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        ParameterHandler handler = (ParameterHandler) invocation.getTarget();

        // 通过 MetaObject 获取 boundSql
        MetaObject metaObject = SystemMetaObject.forObject(handler);
        Object parameterObject = metaObject.getValue("parameterObject");

        // 判断是否需要加密
        if (parameterObject != null && parameterObject.getClass().isAnnotationPresent(SensitiveData.class)) {
            // 对敏感字段加密
            encryptFields(parameterObject);
        }

        // 执行原方法（设置参数）
        return invocation.proceed();
    }

    private void encryptFields(Object obj) {
        // 通过反射找到标记了 @Encrypt 注解的字段，进行加密
        Field[] fields = obj.getClass().getDeclaredFields();
        for (Field field : fields) {
            if (field.isAnnotationPresent(Encrypt.class)) {
                field.setAccessible(true);
                try {
                    Object value = field.get(obj);
                    if (value instanceof String && !((String) value).isEmpty()) {
                        // 加密处理（这里简化为演示）
                        field.set(obj, "ENCRYPTED:" + value);
                    }
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

# 9.插件在 mybatis-config.xml 中的配置

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <!-- ★ 插件配置在 typeAliases 和 environments 之间 ★ -->
    <plugins>
        <plugin interceptor="com.test.plugin.SqlCostInterceptor">
            <!-- 可以在 plugin 标签内配置属性 -->
            <property name="slowThreshold" value="500"/>
        </plugin>

        <plugin interceptor="com.test.plugin.SimplePageInterceptor"/>

        <plugin interceptor="com.test.plugin.EncryptInterceptor"/>
    </plugins>

    <environments default="development">
        <!-- ... -->
    </environments>

</configuration>
```

# 10.MetaObject — MyBatis 的反射工具

```java
/**
 * MetaObject 是 MyBatis 中非常重要的反射工具类
 * 源码：org.apache.ibatis.reflection.MetaObject
 *
 * 在插件开发中非常常用，因为它可以方便地访问和修改对象的私有属性。
 *
 * 主要功能：
 *   - getValue(String name)       — 获取属性值（支持嵌套路径）
 *   - setValue(String name, Object value) — 设置属性值
 *   - getGetterType(String name)  — 获取属性类型
 *   - hasGetter(String name)      — 是否有 getter
 *   - hasSetter(String name)      — 是否有 setter
 *
 * 属性表达式支持：
 *   "property"           — 直接属性
 *   "obj1.obj2.property" — 嵌套属性（obj1.getObj2().getProperty()）
 *   "list[0]"           — 集合/数组元素
 *   "map['key']"        — Map 元素
 *
 * 插件开发中常用场景：
 *   1. 获取 StatementHandler 的 boundSql：
 *      MetaObject.forObject(handler).getValue("delegate.boundSql.sql")
 *
 *   2. 修改 BoundSql 的 SQL（如分页插件）：
 *      MetaObject.forObject(boundSql).setValue("sql", newSql);
 *
 *   3. 获取 MappedStatement 的 id：
 *      MetaObject.forObject(ms).getValue("id");
 */
public class MetaObject {
    // 获取代理对象
    public static MetaObject forObject(Object object, ObjectFactory objectFactory,
                                       ObjectWrapperFactory objectWrapperFactory,
                                       ReflectorFactory reflectorFactory) {
        if (object == null) {
            return SystemMetaObject.NULL_META_OBJECT;
        }
        return new MetaObject(object, objectFactory, objectWrapperFactory, reflectorFactory);
    }
}
```

# 11.插件开发注意事项

```
  1. 插件会对所有符合条件的对象生效
     → 在 intercept() 方法中最好加判断逻辑，只处理需要拦截的情况
     → 例如：只拦截特定 MappedStatement 的 SQL

  2. 插件中不能调用会导致递归的方法
     → 例如：在 Executor.query() 拦截器内部再次调用 sqlSession.selectList()
     → 这会导致无限递归

  3. 多个插件的执行顺序
     → 按照 XML 中声明的顺序，先声明的先执行
     → 类似洋葱模型：插件1(前) → 插件2(前) → 目标方法 → 插件2(后) → 插件1(后)

  4. 性能影响
     → 每个方法调用都会经过代理的 invoke() 和 intercept()
     → 频繁调用的方法（如 setParameters）要特别注意性能
     → 建议使用轻量级的判断逻辑

  5. 线程安全
     → 插件本身是无状态的（由 Configuration 持有，全局唯一）
     → 不要插件中存储与请求相关的状态
     → 需要传递状态时使用 ThreadLocal

  6. 如何获取被拦截方法的更多上下文
     → 例如拦截 ParameterHandler.setParameters() 时，
       可以通过 MetaObject 获取 boundSql、MappedStatement 等信息
       （因为这些信息在 StatementHandler 中，而不是 ParameterHandler 的方法参数中）
```

# 12.插件 vs 拦截器 vs AOP

```
┌─────────────────┬─────────────────────┬──────────────────────┬────────────────────┐
│      维度        │   MyBatis Plugin    │   Spring AOP          │   Java Interceptor  │
├─────────────────┼─────────────────────┼──────────────────────┼────────────────────┤
│  拦截范围        │ MyBatis 四大组件     │ Spring Bean 的方法    │ Servlet/Spring MVC │
│  实现方式        │ JDK 动态代理         │ JDK/CGLIB 代理        │ 回调机制            │
│  配置方式        │ XML + @Intercepts   │ @Aspect + @Around     │ 实现接口            │
│  粒度            │ 方法级别             │ 方法/注解级别          │ 请求级别            │
│  适用场景        │ SQL 改写/分页/监控   │ 事务/日志/权限控制     │ 请求拦截/过滤器     │
└─────────────────┴─────────────────────┴──────────────────────┴────────────────────┘

  区别本质：
  - MyBatis Plugin 是 MyBatis 框架内部的一种扩展机制
  - Spring AOP 是 Spring 容器管理的、面向业务层的切面编程
  - Java Interceptor 是 Web 层面的请求拦截
```
