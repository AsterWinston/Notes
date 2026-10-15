# 1. environment
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!--可以配置多个environment环境标签-->
    <!--default 属性指明默认使用哪个环境配置-->
    <environments default="development">
        <!--一般一个数据库会对应一个SqlSessionFactory对象-->
        <!--一个环境environment会与对应一个SqlSessionFactory对象-->
        <!--开发环境配置-->
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/ziv_mybatis"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
        <!--生产环境配置-->
        <environment id="production">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/ziv_mybatis2"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="CarMapper.xml"/>
    </mappers>
</configuration>

@Test
public void configTest() throws IOException {
    InputStream is = Resources.getResourceAsStream("mybatis-config.xml");
    //build方法的第二个参数用来指定使用哪个环境配置，如果没有指定使用默认配置。
    SqlSession sqlSession = new SqlSessionFactoryBuilder().build(is).openSession();
    SqlSession sqlSession2 = new SqlSessionFactoryBuilder().build(is,"production").openSession();

}
```

# 2. transactionManager
```
<environment id="development">
    <!--
        transactionManager标签：
            1.作用：配置事务管理器。指定mybatis具体使用什么方式去管理事务。
            2.type属性有两个值(不区分大小写)：
                a) JDBC：使用原生的JDBC代码来管理事务
                    conn.setAutoCommit(false);
                    ...
                    conn.commit();
                b) MANAGED：mybatis不再负责事务的管理，将事务交给其他容器管理，如spring。
						 当mybatis找不到容器的支持时：没有事务
            3.mybatis中提供了一个事务管理器接口：Transaction
                该接口下有两个实现类：
                    JdbcTransaction
                    ManagedTransaction
                type中写哪个，底层机会实例化哪个对象
    -->
    <transactionManager type="JDBC"/>
    <dataSource type="POOLED">
        <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/ziv_mybatis"/>
        <property name="username" value="root"/>
        <property name="password" value="123456"/>
    </dataSource>
</environment>
```
# 3. dataSource
```
<environment id="development">
    <transactionManager type="JDBC"/>
    <!--
        dataSource配置：
            1.dataSource被称为数据源。
            2.dataSource作用：为程序提供Connection对象。(只要是给程序提供Connection对象的，都可以叫做数据源)
            3.数据源实际上是一套规范。JDK中有这套规范：javax.sql.DataSource (这个数据源的规范，实际上就是JDK规定的)
            4.我们也可以自己实现javax.sql.DataSource接口，编写数据源组件。
                比如自己写一个数据库连接池
            5.常见的数据源组件(也称常见的数据库连接池)：
                druid c3p0 dbcp...
            6.type属性用来指定数据源的类型，就是指定用什么方式来获取Connection对象
                type有三个值，必须三选一：
                UNPOOLED:不使用数据库连接池技术。每次都创建新的Connection对象
                POOLED:使用mybatis自己的数据库连接池
                JNDI:继承第三方的数据库连接池

                JNDI是一套规范，大部分web容器都是先了JNDI规范
                    例如：Tomcat、Jetty、WebLogic、WehSphere
                JNDI是Java命名目录接口。Tomcat服务器实现了这个规范。

    -->
    <dataSource type="POOLED">
        <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/ziv_mybatis"/>
        <property name="username" value="root"/>
        <property name="password" value="123456"/>
    </dataSource>
</environment>
```

# 4. properties
```
mybatis提供了更加灵活的配置，连接数据库的信息可以单独写到一个属性资源文件中，假设在类的根路
径下创建jdbc.properties文件，配置如下：
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/ziv_mybatis2
jdbc.username=root
jdbc.password=123456

在mybatis核心配置文件中引入并使用：
<!--引入外部属性资源文件-->
<properties resource="jdbc.properties"/>
这个标签需要在environments之前

<environment id="production">
    <transactionManager type="JDBC"/>
    <dataSource type="POOLED">
        <!--${key}使用-->
        <property name="driver" value="${jdbc.driver}"/>
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.username}"/>
        <property name="password" value="${jdbc.password}"/>
    </dataSource>
</environment>

properties两个属性：
resource：这个属性从类的根路径下开始加载。【常用的。】
url：从指定的url加载，假设文件放在d:/jdbc.properties，这个url可以写成：file:///d:/jdbc.properties。注意是三个斜杠哦。?
```

# 5. mapper
```
mapper标签用来指定SQL映射文件的路径，包含多种指定方式，这里先主要看其中两种：

第一种：resource，从类的根路径下开始加载【比url常用】
<mappers>
    <mapper resource="CarMapper.xml"/>
</mappers>

第二种：url，从指定的url位置加载
<mappers>
	<mapper url="file:///d:/CarMapper.xml"/>
</mappers>

mapper还有其他的指定方式，后面再看！！！

<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!--java.util.Properties类，是一个Map集合。key和value都是String类型-->
    <!--<properties resource="jdbc.properties">
        &lt;!&ndash;在properties标签中可以配置很多属性&ndash;&gt;
        <property name="jdbc.driver" value="com.mysql.cj.jdbc.Driver"/>
        <property name="jdbc.url" value="jdbc:mysql://localhost:3306/ziv_mybatis2"/>
        <property name="jdbc.username" value="root"/>
        <property name="jdbc.password" value="123456"/>
    </properties>-->
    <properties resource="jdbc.properties"/>
    <!--可以配置多个environment环境标签-->
    <!--default 属性指明默认使用哪个环境配置-->
    <environments default="development">
        <!--一般一个数据库会对应一个SqlSessionFactory对象-->
        <!--一个环境environment会与对应一个SqlSessionFactory对象-->
        <!--开发环境配置-->
        <environment id="development">
            <!--
                transactionManager标签：
                    1.作用：配置事务管理器。指定mybatis具体使用什么方式去管理事务。
                    2.type属性有两个值(不区分大小写)：
                        a) JDBC：使用原生的JDBC代码来管理事务
                            conn.setAutoCommit(false);
                            ...
                            conn.commit();
                        b) MANAGED：mybatis不再负责事务的管理，将事务交给其他容器管理，如spring。
                    3.mybatis中提供了一个事务管理器接口：Transaction
                        该接口下有两个实现类：
                            JdbcTransaction
                            ManagedTransaction
                        type中写哪个，底层机会实例化哪个对象
            -->
            <transactionManager type="JDBC"/>
            <!--
                dataSource配置：
                    1.dataSource被称为数据源。
                    2.dataSource作用：为程序提供Connection对象。(只要是给程序提供Connection对象的，都可以叫做数据源)
                    3.数据源实际上是一套规范。JDK中有这套规范：javax.sql.DataSource (这个数据源的规范，实际上就是JDK规定的)
                    4.我们也可以自己实现javax.sql.DataSource接口，编写数据源组件。
                        比如自己写一个数据库连接池
                    5.常见的数据源组件(也称常见的数据库连接池)：
                        druid c3p0 dbcp...
                    6.type属性用来指定数据源的类型，就是指定用什么方式来获取Connection对象
                        type有三个值，必须三选一：
                        UNPOOLED:不使用数据库连接池技术。每次都创建新的Connection对象
                        POOLED:使用mybatis自己的数据库连接池
                        JNDI:继承第三方的数据库连接池

                        JNDI是一套规范，大部分web容器都是先了JNDI规范
                            例如：Tomcat、Jetty、WebLogic、WehSphere
                        JNDI是Java命名目录接口。Tomcat服务器实现了这个规范。
            -->
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/ziv_mybatis"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
        <!--生产环境配置-->
        <environment id="production">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>
<mappers>
    <mapper resource="CarMapper.xml"/>
</mappers>
</configuration>
```
# 6. order of all kinds of config
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "[https://mybatis.org/dtd/mybatis-3-config.dtd](https://mybatis.org/dtd/mybatis-3-config.dtd)">

<!-- 1. 根标签 -->
<configuration>

    <!-- 可选：属性配置 (properties) -->
    <!-- ... -->

    <!-- 2. 核心位置：设置 (settings) -->
    <settings>
        <!-- 你的设置就放在这里 -->
        <setting name="mapUnderscoreToCamelCase" value="true"/>
        
        <!-- 其他常用设置，例如日志级别等 -->
        <!-- <setting name="logImpl" value="STDOUT_LOGGING"/> -->
    </settings>
    
    <!-- 可选：别名配置 (typeAliases) -->
    <!-- ... -->

    <!-- 必选：环境配置 (environments) -->
    <!-- ... -->

    <!-- 必选：Mapper 注册 (mappers) -->
    <!-- ... -->

</configuration>

```
# 7. XMLConfigBuilder 源码解析

XMLConfigBuilder 是 mybatis-config.xml 的解析器，
它是理解 MyBatis 配置加载机制的核心入口。

源码位置：org.apache.ibatis.builder.xml.XMLConfigBuilder

## 7.1 解析入口 parseConfiguration()

```java
/**
 * parseConfiguration() 方法按固定顺序解析 <configuration> 下的各个子节点
 * 顺序不可变，因为后面的节点可能依赖前面的解析结果
 *
 * 解析顺序和原因：
 *   1. properties   — 最先，因为后续配置可能引用 ${xxx} 变量
 *   2. settings     — 修改 Configuration 中的各种 flag 和默认值
 *   3. typeAliases  — 注册别名，后续 resultType/parameterType 需要
 *   4. plugins      — 注册拦截器
 *   5-7. factory相关 — ObjectFactory/ObjectWrapperFactory/ReflectorFactory
 *   8. settings     — 二次回填（确保所有 settings 都生效）
 *   9. environments — 环境配置（数据源 + 事务管理器）
 *   10. databaseIdProvider — 数据库厂商标识
 *   11. typeHandlers — 类型处理器
 *   12. mappers      — 最后，因为前面的配置它都可能用到
 */
private void parseConfiguration(XNode root) {
    try {
        propertiesElement(root.evalNode("properties"));
        Properties settings = settingsAsProperties(root.evalNode("settings"));
        loadCustomVfs(settings);
        loadCustomLogImpl(settings);
        settingsElement(settings);
        typeAliasesElement(root.evalNode("typeAliases"));
        pluginElement(root.evalNode("plugins"));
        objectFactoryElement(root.evalNode("objectFactory"));
        objectWrapperElement(root.evalNode("objectWrapperFactory"));
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
        settingsElement(settings);
        environmentsElement(root.evalNode("environments"));
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        typeHandlerElement(root.evalNode("typeHandlers"));
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. " + e, e);
    }
}
```

## 7.2 mappers 解析源码

```java
/**
 * 解析 <mappers> 节点，支持四种方式：
 *   1. <mapper resource=""/>   → 类路径 XML 文件
 *   2. <mapper url=""/>        → URL 指定 XML 文件
 *   3. <mapper class=""/>      → 指定 Mapper 接口（XML 和注解都解析）
 *   4. <package name=""/>      → 批量扫描包下所有接口
 */
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            if ("package".equals(child.getName())) {
                // 方式4：批量扫描
                String mapperPackage = child.getStringAttribute("name");
                configuration.addMappers(mapperPackage);
            } else {
                String resource = child.getStringAttribute("resource");
                String url = child.getStringAttribute("url");
                String mapperClass = child.getStringAttribute("class");

                // 方式1：resource（最常用）
                if (resource != null && url == null && mapperClass == null) {
                    ErrorContext.instance().resource(resource);
                    try (InputStream inputStream = Resources.getResourceAsStream(resource)) {
                        XMLMapperBuilder mapperParser = new XMLMapperBuilder(
                            inputStream, configuration, resource,
                            configuration.getSqlFragments());
                        mapperParser.parse();   // ★ 解析 SQL 映射文件
                    }
                }
                // 方式3：class → 自动找同名 XML + 解析注解
                else if (resource == null && url == null && mapperClass != null) {
                    Class<?> mapperInterface = Resources.classForName(mapperClass);
                    configuration.addMapper(mapperInterface);
                }
                // ... 方式2同理
            }
        }
    }
}
```

## 7.3 dataSource 数据源解析

```java
/**
 * 解析 <dataSource> 节点
 *
 * type 属性对应三种实现：
 *   POOLED  → PooledDataSource（内置连接池，默认）
 *   UNPOOLED → UnpooledDataSource（每次新建连接）
 *   JNDI    → JndiDataSourceFactory（容器提供的 JNDI 数据源）
 *
 * MyBatis 的数据源都实现了 javax.sql.DataSource 接口
 */
private DataSourceFactory dataSourceElement(XNode context) throws Exception {
    if (context != null) {
        String type = context.getStringAttribute("type");  // POOLED / UNPOOLED / JNDI
        Properties props = context.getChildrenAsProperties();
        DataSourceFactory factory = (DataSourceFactory) resolveClass(type).newInstance();
        factory.setProperties(props);
        return factory;
    }
    throw new BuilderException("Environment declaration requires a DataSourceFactory.");
}
```

## 7.4 Configuration 中 settings 对应的源码字段

```
以下是 settings 中每个配置项在 Configuration 中的对应字段：

┌───────────────────────────────────┬──────────────────────────────────────┐
│  settings 配置项                    │  Configuration 中的字段（默认值）      │
├───────────────────────────────────┼──────────────────────────────────────┤
│  cacheEnabled                     │  boolean cacheEnabled = true          │
│  lazyLoadingEnabled               │  boolean lazyLoadingEnabled = false   │
│  aggressiveLazyLoading            │  boolean aggressiveLazyLoading = false│
│  multipleResultSetsEnabled        │  boolean multipleResultSetsEnabled... │
│  useColumnLabel                   │  boolean useColumnLabel = true        │
│  useGeneratedKeys                 │  boolean useGeneratedKeys = false     │
│  autoMappingBehavior              │  AutoMappingBehavior = PARTIAL        │
│  defaultExecutorType              │  ExecutorType = SIMPLE                │
│  defaultStatementTimeout          │  Integer defaultStatementTimeout       │
│  defaultFetchSize                 │  Integer defaultFetchSize             │
│  mapUnderscoreToCamelCase         │  boolean mapUnderscoreTo... = false   │
│  safeRowBoundsEnabled             │  boolean safeRowBoundsEnabled = false │
│  localCacheScope                  │  LocalCacheScope = SESSION            │
│  jdbcTypeForNull                  │  JdbcType = OTHER                     │
│  lazyLoadTriggerMethods           │  Set<String> = {equals,clone,hash...} │
│  defaultScriptingLanguage         │  Class = XMLLanguageDriver            │
│  defaultEnumTypeHandler           │  Class = EnumTypeHandler              │
│  callSettersOnNulls               │  boolean callSettersOnNulls = false   │
│  returnInstanceForEmptyRow        │  boolean returnInstanceFor... = false │
│  logPrefix                        │  String logPrefix                     │
│  logImpl                          │  Class<? extends Log> logImpl         │
│  proxyFactory                     │  ProxyFactory = JavassistProxyFactory │
│  vfsImpl                          │  Class<? extends VFS> vfsImpl         │
│  useActualParamName               │  boolean useActualParamName = true    │
│  configurationFactory             │  Class<?> configurationFactory        │
│  shrinkWhitespacesInSql           │  boolean = false(3.5.5+)              │
└───────────────────────────────────┴──────────────────────────────────────┘
```