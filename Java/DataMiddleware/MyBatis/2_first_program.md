# 1. 版本
```
软件版本：
IntelliJ IDEA：2022.1.4
Navicat for MySQL：16.0.14
MySQL数据库：8.0.30
组件版本：
MySQL驱动：8.0.30
MyBatis：3.5.10
JDK：Java17
JUnit：4.13.2
Logback：1.2.11
```

# 2. MyBatis下载
```
从github上下载，地址：https://github.com/mybatis/mybatis-3
maven就不用这么麻烦了
maven导入依赖
<dependency>
	<groupId>org.mybatis</groupId>
	<artifactId>mybatis</artifactId>
	<version>3.5.19</version>
</dependency>
```


# 3. MyBatis入门程序开发步骤
```
写代码前准备：
准备数据库表：汽车表t_car，字段包括：
id：主键（自增）【bigint】
car_num：汽车编号【varchar】
brand：品牌【varchar】
guide_price：厂家指导价【decimal类型，专门为财务数据准备的类型】
produce_time：生产时间【char，年月日即可，10个长度，'2022-10-11'】
car_type：汽车类型（燃油车、电车、氢能源）【varchar】
使用navicat for mysql工具建表
使用navicat for mysql工具向t_car表中插入两条数据
创建Project：建议创建Empty Project，设置Java版本以及编译版本等。
```
## 3.1 步骤1：打包方式：jar（不需要war，因为mybatis封装的是jdbc。）
```
<groupId>com.powernode</groupId>
<artifactId>mybatis-001-introduction</artifactId>
<version>1.0-SNAPSHOT</version>
<packaging>jar</packaging>
```
## 3.2 步骤2：引入依赖（mybatis依赖 + mysql驱动依赖）
```
<!--mybatis核心依赖-->
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis</artifactId>
  <version>3.5.10</version>
</dependency>
<!--mysql驱动依赖-->
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <version>8.0.30</version>
</dependency>
```
## 3.3 步骤3：在resources根目录下新建mybatis-config.xml配置文件（可以参考mybatis手册拷贝）
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/powernode"/>
                <property name="username" value="root"/>
                <property name="password" value="root"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <!--sql映射文件创建好之后，需要将该文件路径配置到这里-->
        <mapper resource=""/>
    </mappers>
</configuration>
注意1：mybatis核心配置文件的文件名不一定是mybatis-config.xml，可以是其它名字。
注意2：mybatis核心配置文件存放的位置也可以随意。这里选择放在resources根下，相当于放到了类的根路径下。
```
## 3.4 步骤4：在resources根目录下新建CarMapper.xml配置文件（可以参考mybatis手册拷贝）
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!--namespace先随意写一个-->
<mapper namespace="car">
    <!--insert sql：保存一个汽车信息-->
    <insert id="insertCar">
        insert into t_car
            (id,car_num,brand,guide_price,produce_time,car_type) 
        values
            (null,'102','丰田mirai',40.30,'2014-10-05','氢能源')
    </insert>
</mapper>
注意1：sql语句最后结尾可以不写“;”
注意2：CarMapper.xml文件的名字不是固定的。可以使用其它名字。
注意3：CarMapper.xml文件的位置也是随意的。这里选择放在resources根下，相当于放到了类的根路径下。
注意4：将CarMapper.xml文件路径配置到mybatis-config.xml：
<mapper resource="CarMapper.xml"/>
```
## 3.5 步骤5：编写MyBatisIntroductionTest代码
```
package com.powernode.mybatis;

import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import java.io.InputStream;

/**
 * MyBatis入门程序
 * @author 老杜
 * @since 1.0
 * @version 1.0
 */
public class MyBatisIntroductionTest {
    public static void main(String[] args) {
        // 1. 创建SqlSessionFactoryBuilder对象
        SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
        // 2. 创建SqlSessionFactory对象
        InputStream is = Thread.currentThread().getContextClassLoader().getResourceAsStream("mybatis-config.xml");
        SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(is);
        // 3. 创建SqlSession对象
        SqlSession sqlSession = sqlSessionFactory.openSession();
        // 4. 执行sql
        int count = sqlSession.insert("insertCar"); // 这个"insertCar"必须是sql的id
        System.out.println("插入几条数据：" + count);
        // 5. 提交（mybatis默认采用的事务管理器是JDBC，默认是不提交的，需要手动提交。）
        sqlSession.commit();
        // 6. 关闭资源（只关闭是不会提交的）
        sqlSession.close();
    }
}
注意1：默认采用的事务管理器是：JDBC。JDBC事务默认是不提交的，需要手动提交。
```
## 3.6 步骤6：运行程序，查看运行结果，以及数据库表中的数据
```
```

# 4. 关于Mybatis核心配置文件的名字和路径
```

核心配置文件的名字是随意的，因为以下的代码：
// 文件名是出现在程序中的，文件名如果修改了，对应这里的java程序也改一下就行了。
InputStream is = Thread.currentThread().getContextClassLoader().getResourceAsStream("mybatis-config.xml");
将mybatis-config.xml文件拷贝一份放到D盘根下，然后编写以下程序：
package com.powernode.mybatis;

import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import java.io.FileInputStream;
import java.io.InputStream;

/**
 * 测试mybatis核心配置文件路径问题
 * @author 老杜
 * @since 1.0
 * @version 1.0
 */
 public class MyBatisConfigFilePath {
    public static void main(String[] args) throws Exception{
        // 1. 创建SqlSessionFactoryBuilder对象
        SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
        // 2. 创建SqlSessionFactory对象
        // 这只是一个输入流，可以自己new。
        InputStream is = new FileInputStream("D:/mybatis-config.xml"); // 这个不用手动关闭，因为SqlSessionFactoryBuilder.build(InputStream) 方法内部，会负责关闭传入的 InputStream。
        SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(is);
        // 3. 创建SqlSession对象
        SqlSession sqlSession = sqlSessionFactory.openSession();
        // 4. 执行sql
        int count = sqlSession.insert("insertCar");
        System.out.println("插入几条数据：" + count);
        // 5. 提交（mybatis默认采用的事务管理器是JDBC，默认是不提交的，需要手动提交。）
        sqlSession.commit();
        // 6. 关闭资源（只关闭是不会提交的）
        sqlSession.close();
    }
 }
 以上程序运行后，看到数据库表t_car中又新增一条数据，如下（成功了）

经过测试说明mybatis核心配置文件的名字是随意的，存放路径也是随意的。
虽然mybatis核心配置文件的名字不是固定的，但通常该文件的名字叫做：mybatis-config.xml
虽然mybatis核心配置文件的路径不是固定的，但通常该文件会存放到类路径当中，这样让项目的移植更加健壮。

在mybatis中提供了一个类：Resources【org.apache.ibatis.io.Resources】，该类可以从类路径当中获取资源，我们通常使用它来获取输入流InputStream，代码如下
// 这种方式只能从类路径当中获取资源，也就是说mybatis-config.xml文件需要在类路径下。
InputStream is = Resources.getResourceAsStream("mybatis-config.xml");
```

# 5. Mybatis第一个比较完整的代码写法
```
package com.powernode.mybatis;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import java.io.IOException;

/**
 * 比较完整的第一个mybatis程序写法
 * @author 老杜
 * @since 1.0
 * @version 1.0
 */
 public class MyBatisCompleteCodeTest {
    public static void main(String[] args) {
        SqlSession sqlSession = null;
        try {
            // 1.创建SqlSessionFactoryBuilder对象
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            // 2.创建SqlSessionFactory对象
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(Resources.getResourceAsStream("mybatis-config.xml"));
            // 3.创建SqlSession对象
            sqlSession = sqlSessionFactory.openSession();
            // 4.执行SQL
            int count = sqlSession.insert("insertCar");
            System.out.println("更新了几条记录：" + count);
            // 5.提交
            sqlSession.commit();
        } catch (Exception e) {
            // 回滚
            if (sqlSession != null) {
                sqlSession.rollback();
            }
            e.printStackTrace();
        } finally {
            // 6.关闭
            if (sqlSession != null) {
                sqlSession.close();
            }
        }
    }
 }

```

# 6. 引入JUnit
```
JUnit是专门做单元测试的组件。
在实际开发中，单元测试一般是由我们Java程序员来完成的。
我们要对我们自己写的每一个业务方法负责任，要保证每个业务方法在进行测试的时候都能通过。
测试的过程中涉及到两个概念：
期望值
实际值
期望值和实际值相同表示测试通过，期望值和实际值不同则单元测试执行时会报错。
这里引入JUnit是为了代替main方法。
使用JUnit步骤：
第一步：引入依赖
<!-- junit依赖 -->
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13.2</version>
    <scope>test</scope>
</dependency>
第二步：编写单元测试类【测试用例】，测试用例中每一个测试方法上使用@Test注解进行标注。
测试用例的名字以及每个测试方法的定义都是有规范的：
测试用例的名字：XxxTest
测试方法声明格式：public void test业务方法名(){}
// 测试用例
public class CarMapperTest{
    
    // 测试方法
    @Test
    public void testInsert(){}
    
    @Test
    public void testUpdate(){}

}
第三步：可以在类上执行，也可以在方法上执行
在类上执行时，该类中所有的测试方法都会执行。
在方法上执行时，只执行当前的测试方法。
编写一个测试用例，来测试insertCar业务
package com.powernode.mybatis;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;
import org.junit.Test;

public class CarMapperTest {
    
    @Test
    public void testInsertCar(){
        SqlSession sqlSession = null;
        try {
            // 1.创建SqlSessionFactoryBuilder对象
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            // 2.创建SqlSessionFactory对象
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(Resources.getResourceAsStream("mybatis-config.xml"));
            // 3.创建SqlSession对象
            sqlSession = sqlSessionFactory.openSession();
            // 4.执行SQL
            int count = sqlSession.insert("insertCar");
            System.out.println("更新了几条记录：" + count);
            // 5.提交
            sqlSession.commit();
        } catch (Exception e) {
            // 回滚
            if (sqlSession != null) {
                sqlSession.rollback();
            }
            e.printStackTrace();
        } finally {
            // 6.关闭
            if (sqlSession != null) {
                sqlSession.close();
            }aa
        }
    }
}

```

# 7. 引入日志框架logback
```
引入日志框架的目的是为了看清楚mybatis执行的具体sql。
启用标准日志组件，只需要在mybatis-config.xml文件中添加以下配置：【可参考mybatis手册】
<settings>
  <setting name="logImpl" value="STDOUT_LOGGING" />
</settings>
这个配置要放在<environments></environments>结点之前

标准日志也可以用，但是配置不够灵活，可以集成其他的日志组件，例如：log4j，logback等。

logback是目前日志框架中性能较好的，较流行的，所以我们选它。
引入logback的步骤：
第一步：引入logback相关依赖
<dependency>
  <groupId>ch.qos.logback</groupId>
  <artifactId>logback-classic</artifactId>
  <version>1.2.11</version>
  <scope>test</scope>
</dependency>
第二步：引入logback相关配置文件（文件名叫做logback.xml或logback-test.xml（其中logback-test.xml优先级大于logback.xml），放到类路径当中，当然，位置是可以配置的，不过这里不再阐述）
<?xml version="1.0" encoding="UTF-8"?>

<configuration debug="false">
    <!-- 控制台输出 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>
    <!-- 按照每天生成日志文件 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--日志文件输出的文件名-->
            <FileNamePattern>${LOG_HOME}/TestWeb.log.%d{yyyy-MM-dd}.log</FileNamePattern>
            <!--日志文件保留天数-->
            <MaxHistory>30</MaxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
        <!--日志文件最大的大小-->
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <MaxFileSize>100MB</MaxFileSize>
        </triggeringPolicy>
    </appender>

    <!--mybatis log configure-->
    <logger name="com.apache.ibatis" level="TRACE"/>
    <logger name="java.sql.Connection" level="DEBUG"/>
    <logger name="java.sql.Statement" level="DEBUG"/>
    <logger name="java.sql.PreparedStatement" level="DEBUG"/>
    
    <!-- 日志输出级别,logback日志级别包括五个：TRACE < DEBUG < INFO < WARN < ERROR -->
    <root level="DEBUG">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="FILE"/>
    </root>

</configuration>
再次执行单元测试方法testInsertCar，查看控制台是否有sql语句输出

这个logback.xml/logback-test.xml必须放在类的根路径下，入果不这样做，需要
-Dlogback.configurationFile=文件的绝对路径/相对路径在运行是加入这个参数

日志接口（日志门面）有
SLF4J（Simple Logging Facade for Java）：目前最主流、推荐优先使用，MyBatis、Spring 等框架原生支持；
JCL（Apache Commons Logging）：早期主流接口，现在基本淘汰，仅用于老项目兼容。

日志实现
Logback：SLF4J 原生实现，性能最优、功能最全，支持控制台 / 文件输出、按日期 / 大小滚动日志，是开发 / 生产环境首选。原生支持 SLF4J（无需任何适配包，直接对接）、可通过 logback-classic 内置适配包兼容 JCL。
Log4j2：高性能实现，支持异步日志、日志聚合（如输出到 ELK），适合高并发、日志量大的生产环境。原生支持 SLF4J（无需额外依赖）、原生支持 JCL（无需额外依赖）。
JUL（Java Util Logging）：JDK 内置，零依赖，功能简单（无日志滚动），适合简单 Demo 或小型工具。不原生支持，需通过「适配包」对接 SLF4J（如 jul-to-slf4j）、不原生支持，需通过「适配包」对接 JCL（如 commons-logging-jul）。
Log4j 1.x：早期经典实现，已停止维护、有安全漏洞，仅兼容老项目，新项目不推荐。不原生支持，需通过「适配包」对接 SLF4J（如 slf4j-log4j12）、不原生支持，需通过「适配包」对接 JCL（如 commons-logging-log4j12）。
STDOUT_LOGGING：MyBatis 内置专用实现，仅支持控制台输出，无额外依赖，仅用于 MyBatis 开发调试（非通用日志实现）。不支持任何通用接口（SLF4J/JCL 均不兼容）。


```

# 8. MyBatis工具类SqlSessionUtil的封装
```
每一次获取SqlSession对象代码太繁琐，封装一个工具类
package com.powernode.mybatis.utils;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

/**
 * MyBatis工具类
 *
 * @author 老杜
 * @version 1.0
 * @since 1.0
 */
 public class SqlSessionUtil {
    private static SqlSessionFactory sqlSessionFactory;

    /**
     * 类加载时初始化sqlSessionFactory对象
     */
     static {
        try {
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            sqlSessionFactory = sqlSessionFactoryBuilder.build(Resources.getResourceAsStream("mybatis-config.xml"));
        } catch (Exception e) {
            e.printStckTrace();
        }
     }

    /**
     * 每调用一次openSession()可获取一个新的会话，该会话支持自动提交。
     *
     * @return 新的会话对象
     */
     public static SqlSession openSession() {
        return sqlSessionFactory.openSession(true);
     }
  }

测试工具类，将testInsertCar()改造
@Test
public void testInsertCar(){
    SqlSession sqlSession = SqlSessionUtil.openSession();
    // 执行SQL
    int count = sqlSession.insert("insertCar");
    System.out.println("插入了几条记录:" + count);
     sqlSession.close();
}
```

# 9. 启动流程源码追踪

```
下面追踪 SqlSessionFactoryBuilder.build() 到执行 SQL 的完整源码链路。
这是理解 MyBatis 内部机制的最佳入口。
```

## 9.1 SqlSessionFactoryBuilder.build() 源码

```java
/**
 * SqlSessionFactoryBuilder — 构建 SqlSessionFactory
 * 源码：org.apache.ibatis.session.SqlSessionFactoryBuilder
 *
 * build() 方法的调用链：
 *   build(InputStream)
 *     → build(InputStream, null, null)
 *       → new XMLConfigBuilder(inputStream, environment, props)
 *       → parser.parse()  ← ★ 核心：解析配置文件，构建 Configuration
 *       → new DefaultSqlSessionFactory(config)  ← 创建 SqlSessionFactory
 */
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
        // 1. 创建 XMLConfigBuilder 解析器
        XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
        // 2. 解析配置文件 → Configuration
        Configuration config = parser.parse();
        // 3. 创建 DefaultSqlSessionFactory
        return build(config);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
        ErrorContext.instance().reset();
        try {
            // 4. ★ 内部会关闭传入的 InputStream ★
            inputStream.close();
        } catch (IOException e) {
            // Intentionally ignore.
        }
    }
}

public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
}
```

## 9.2 XMLConfigBuilder.parse() 源码

```java
/**
 * parse() → parseConfiguration() → 逐个解析各节点
 */
public Configuration parse() {
    if (parsed) {
        throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    // 解析 <configuration> 根节点
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
}

private void parseConfiguration(XNode root) {
    try {
        // 解析顺序是严格固定的：
        propertiesElement(root.evalNode("properties"));     // 1. 属性
        Properties settings = settingsAsProperties(root.evalNode("settings"));
        loadCustomVfs(settings);                            // 2. VFS
        loadCustomLogImpl(settings);                        // 3. 日志实现
        settingsElement(settings);                          //    settings
        typeAliasesElement(root.evalNode("typeAliases"));   // 4. 别名
        pluginElement(root.evalNode("plugins"));            // 5. 插件
        objectFactoryElement(root.evalNode("objectFactory"));// 6. 对象工厂
        objectWrapperElement(root.evalNode("objectWrapperFactory"));
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
        settingsElement(settings);                          // 二次回填 settings
        environmentsElement(root.evalNode("environments"));  // 9. 环境配置
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        typeHandlerElement(root.evalNode("typeHandlers"));   // 11. 类型处理器
        mapperElement(root.evalNode("mappers"));             // 12. Mapper 映射
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```

## 9.3 DefaultSqlSessionFactory.openSession() 源码

```java
/**
 * DefaultSqlSessionFactory — 创建 SqlSession
 * 源码：org.apache.ibatis.session.defaults.DefaultSqlSessionFactory
 *
 * openSession() 的核心逻辑：
 *   1. 从 Configuration 获取 Environment → TransactionFactory
 *   2. 创建 Transaction 对象
 *   3. 通过 Configuration.newExecutor() 创建 Executor
 *   4. new DefaultSqlSession(config, executor, autoCommit)
 */
public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}

private SqlSession openSessionFromDataSource(ExecutorType execType,
        TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
        final Environment environment = configuration.getEnvironment();
        final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);

        // ★ 1. 创建事务对象（JdbcTransaction / ManagedTransaction）★
        tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);

        // ★ 2. 创建 Executor ★
        //    内部逻辑：
        //      - 先创建 SimpleExecutor / ReuseExecutor / BatchExecutor
        //      - 如果有二级缓存 → 用 CachingExecutor 包装
        //      - 应用拦截器链 → interceptorChain.pluginAll(executor)
        final Executor executor = configuration.newExecutor(tx, execType);

        // ★ 3. 创建 DefaultSqlSession ★
        return new DefaultSqlSession(configuration, executor, autoCommit);

    } catch (Exception e) {
        closeTransaction(tx);
        throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    }
}
```

## 9.4 Configuration.newExecutor() 源码

```java
/**
 * Configuration.newExecutor() — 创建 Executor 的核心
 *
 * 这是 Executor 创建的"三部曲"：
 *   1. 创建基础 Executor
 *   2. 装饰二级缓存
 *   3. 应用插件链
 */
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    Executor executor;

    // 1. 创建基础 Executor
    if (ExecutorType.BATCH == executorType) {
        executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
        executor = new ReuseExecutor(this, transaction);
    } else {
        executor = new SimpleExecutor(this, transaction);  // ★ 默认
    }

    // 2. 如果开启了二级缓存 → CachingExecutor 装饰
    if (cacheEnabled) {
        executor = new CachingExecutor(executor);
    }

    // 3. ★ 应用插件拦截器链 ★
    //    如果有配置插件（PageHelper 等），在这里对 Executor 生成代理
    executor = (Executor) interceptorChain.pluginAll(executor);

    return executor;
}
```

## 9.5 sqlSession.insert() 源码

```java
/**
 * DefaultSqlSession.insert() — INSERT 操作的入口
 *
 * 调用链：
 *   sqlSession.insert("insertCar", parameter)
 *     → update("insertCar", parameter)
 *       → executor.update(ms, parameter)
 *         → BaseExecutor.update(ms, parameter)
 *           → 清空一级缓存
 *           → doUpdate(ms, parameter)  ← 子类实现
 *             → SimpleExecutor.doUpdate()
 *               → 创建 StatementHandler
 *               → 获取 Connection
 *               → 创建 PreparedStatement
 *               → ParameterHandler.setParameters() 设置参数
 *               → handler.update(stmt) 执行
 *               → 关闭 Statement
 *               → 返回影响行数
 */
@Override
public int insert(String statement, Object parameter) {
    return update(statement, parameter);
}

@Override
public int update(String statement, Object parameter) {
    try {
        dirty = true;
        MappedStatement ms = configuration.getMappedStatement(statement);
        return executor.update(ms, wrapCollection(parameter));
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
    }
}
```

## 9.6 总结：从入口到 JDBC 执行的完整时序

```
SqlSessionFactoryBuilder.build(is)
  → XMLConfigBuilder.parse()
    → 解析 properties / settings / typeAliases / plugins / environments / mappers
    → 构建 Configuration 对象
  → new DefaultSqlSessionFactory(configuration)

factory.openSession()
  → openSessionFromDataSource()
    → TransactionFactory.newTransaction() → JdbcTransaction
    → Configuration.newExecutor()
      → new SimpleExecutor(transaction)
      → if cacheEnabled → new CachingExecutor(simpleExecutor)
      → interceptorChain.pluginAll(executor) → 代理包装
    → new DefaultSqlSession(configuration, executor)

sqlSession.insert("insertCar", parameter)
  → sqlSession.update()
    → executor.update(ms, parameter)
      → BaseExecutor.update()
        → clearLocalCache()
        → SimpleExecutor.doUpdate()
          → StatementHandler.prepare() → Connection.prepareStatement(sql)
          → ParameterHandler.setParameters() → ps.setXxx(index, value)
          → StatementHandler.update() → ps.executeUpdate()
          → 返回影响行数
```