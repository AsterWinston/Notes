# 1.dom4j解析XML文件

```
dom4j 是一个高性能的 Java XML 解析库。在 MyBatis 中，核心配置文件 mybatis-config.xml 
和 SQL 映射文件 XxxMapper.xml 都是通过 dom4j（实际上是 XPath + dom4j）来解析的。

关键 API：
  SAXReader：读取 XML 文件
  Document：表示整个 XML 文档
  Element：表示 XML 元素节点
  Attribute：表示 XML 属性
  XPath：通过路径表达式快速定位节点
```

## 1.1 dom4j 基本使用示例

```java
// 1. 创建 SAXReader 对象
SAXReader reader = new SAXReader();
// 2. 读取 XML 文件，获取 Document 对象
Document document = reader.read(new File("mybatis-config.xml"));
// 3. 获取根元素
Element root = document.getRootElement();
// 4. 通过 XPath 获取指定节点
String xpath = "//environments/environment/dataSource/property";
List<Element> propertyList = root.selectNodes(xpath);
propertyList.forEach(element -> {
    String name = element.attributeValue("name");
    String value = element.attributeValue("value");
    System.out.println(name + " = " + value);
});
```

## 1.2 解析 mybatis-config.xml

```java
// 解析 environments 获取数据源配置
Element environments = (Element) root.selectSingleNode("environments");
String defaultEnvId = environments.attributeValue("default");
Element environment = (Element) root.selectSingleNode(
    "//environments/environment[@id='" + defaultEnvId + "']"
);
// 获取事务管理器类型
String transactionType = environment.element("transactionManager").attributeValue("type");
// 获取数据源信息
Element dataSource = environment.element("dataSource");
String dsType = dataSource.attributeValue("type");
List<Element> props = dataSource.elements("property");
String driver = null, url = null, username = null, password = null;
for (Element prop : props) {
    String name = prop.attributeValue("name");
    String value = prop.attributeValue("value");
    switch (name) {
        case "driver":   driver = value;   break;
        case "url":      url = value;      break;
        case "username": username = value;  break;
        case "password": password = value;  break;
    }
}
```

## 1.3 解析 Mapper.xml 映射文件

```java
// 解析 mappers 节点获取 SQL 映射文件路径
Element mappers = (Element) root.selectSingleNode("mappers");
List<Element> mapperList = mappers.elements("mapper");
for (Element mapper : mapperList) {
    String resource = mapper.attributeValue("resource");
    // 解析每个 SQL 映射文件
    parseMapperXml(resource);
}

// 解析单个 Mapper.xml
private static void parseMapperXml(String resource) {
    SAXReader reader = new SAXReader();
    Document doc = reader.read(Resources.getResourceAsStream(resource));
    Element root = doc.getRootElement();
    String namespace = root.attributeValue("namespace");
    
    // 解析 select 标签
    List<Element> selects = root.elements("select");
    for (Element select : selects) {
        String id = select.attributeValue("id");
        String resultType = select.attributeValue("resultType");
        String sql = select.getTextTrim();
        // 将 SQL 语句封装成 MappedStatement 对象
        MappedStatement ms = new MappedStatement();
        ms.setId(namespace + "." + id);
        ms.setResultType(resultType);
        ms.setSql(sql);
        // 存储到 Map 中：key=sqlId, value=MappedStatement
        mappedStatements.put(ms.getId(), ms);
    }
    
    // 同样解析 insert / update / delete 标签...
}
```

# 2.GodBatis 手写框架

```
目标：手写一个极简版 MyBatis 框架，命名为 GodBatis，核心实现以下功能：
  1. 解析 mybatis-config.xml 获取数据源配置
  2. 解析 XxxMapper.xml 获取 SQL 语句
  3. 提供 SqlSession API（insert/delete/update/selectOne/selectList）
  4. 支持 #{} 占位符替换为 JDBC 的 ?
  5. 支持结果集自动映射到 POJO
  6. 支持 getMapper() 动态代理
```

## 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                         GodBatis 核心架构                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   SqlSessionFactoryBuilder                                   │
│        │ build(InputStream)                                  │
│        ▼                                                    │
│   SqlSessionFactory                                          │
│        │ openSession()                                       │
│        ▼                                                    │
│   SqlSession                                                 │
│        │ getMapper() / selectOne() / insert() / ...         │
│        ▼                                                    │
│   Executor (真正执行 JDBC 的组件)                              │
│        │                                                    │
│        ├── Connection (JDBC)                                 │
│        ├── PreparedStatement                                 │
│        └── ResultSet                                         │
│                                                             │
│   辅助组件：                                                   │
│   ├── Configuration (存储所有配置信息)                          │
│   ├── MappedStatement (封装一条 SQL 的完整信息)                  │
│   ├── XMLConfigBuilder (解析 mybatis-config.xml)             │
│   ├── XMLMapperBuilder (解析 XxxMapper.xml)                  │
│   └── MapperProxy (Mapper 接口的动态代理)                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2.2 核心类 MappedStatement

```java
package com.godbatis.core;

/**
 * MappedStatement 封装一条 SQL 语句的完整信息
 * 在 MyBatis 源码中，这是最核心的数据结构之一：
 *   - 包含 SQL 语句、参数映射、结果映射
 *   - 包含 statementType、sqlCommandType 等执行配置
 *   - 它是 Configuration 中存储的最小粒度 SQL 单元
 *
 * MyBatis 源码对应：org.apache.ibatis.mapping.MappedStatement
 */
public class MappedStatement {

    /** SQL 的唯一标识：namespace + id */
    private String id;

    /** 结果类型（全限定类名） */
    private String resultType;

    /** 参数类型（全限定类名） */
    private String parameterType;

    /** 原始的 SQL 语句（可能包含 #{} 占位符） */
    private String sql;

    /**
     * 在 MyBatis 中，一个 MappedStatement 还包含：
     *   - SqlCommandType（INSERT/UPDATE/DELETE/SELECT）
     *   - SqlSource（编译后的 SQL 源）
     *   - ResultMap 列表
     *   - 缓存配置
     *   - statementType（PREPARED/STATEMENT/CALLABLE）
     *   - keyGenerator、keyProperty 等
     * 这里为了简洁，只保留核心字段
     */

    // getter / setter / toString 省略
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public String getResultType() { return resultType; }
    public void setResultType(String resultType) { this.resultType = resultType; }
    public String getParameterType() { return parameterType; }
    public void setParameterType(String parameterType) { this.parameterType = parameterType; }
    public String getSql() { return sql; }
    public void setSql(String sql) { this.sql = sql; }

    /**
     * 获取 SQL 命令类型
     * 注意：普通 Java 反射如果能获取到 Method 对象，可以判断
     * 这里简单的在解析的时候就加上 "type"
     */
    private String sqlCommandType; // SELECT / INSERT / UPDATE / DELETE

    public void setSqlCommandType(String sqlCommandType) {
        this.sqlCommandType = sqlCommandType;
    }
    public String getSqlCommandType() {
        return sqlCommandType;
    }
}
```

## 2.3 核心配置类 Configuration

```java
package com.godbatis.core;

import javax.sql.DataSource;
import java.util.HashMap;
import java.util.Map;

/**
 * Configuration 是整个 GodBatis 的核心配置对象
 * 存储了数据源、所有 MappedStatement、以及运行时的全局配置
 *
 * MyBatis 源码对应：org.apache.ibatis.session.Configuration
 *   - MyBatis 的 Configuration 非常庞大，包含几百个属性
 *   - 我们这里只保留最核心的部分
 */
public class Configuration {

    /** 数据源：提供 Connection 对象 */
    private DataSource dataSource;

    /**
     * SQL 语句映射集合
     * key = namespace + "." + id (sqlId)
     * value = MappedStatement 对象
     */
    private Map<String, MappedStatement> mappedStatements = new HashMap<>();

    public DataSource getDataSource() {
        return dataSource;
    }

    public void setDataSource(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public Map<String, MappedStatement> getMappedStatements() {
        return mappedStatements;
    }

    public void setMappedStatements(Map<String, MappedStatement> mappedStatements) {
        this.mappedStatements = mappedStatements;
    }

    public void addMappedStatement(String id, MappedStatement ms) {
        mappedStatements.put(id, ms);
    }

    public MappedStatement getMappedStatement(String id) {
        return mappedStatements.get(id);
    }
}
```

## 2.4 XMLConfigBuilder — 解析核心配置文件

```java
package com.godbatis.builder;

import com.godbatis.core.Configuration;
import com.godbatis.core.MappedStatement;
import com.godbatis.utils.Resources;
import org.dom4j.Document;
import org.dom4j.Element;
import org.dom4j.io.SAXReader;

import javax.sql.DataSource;
import java.io.InputStream;
import java.util.List;

/**
 * XMLConfigBuilder 负责解析 mybatis-config.xml
 * 将其中的配置信息封装到 Configuration 对象中
 *
 * MyBatis 源码对应：org.apache.ibatis.builder.xml.XMLConfigBuilder
 *   - 真正的 MyBatis XMLConfigBuilder 继承自 BaseBuilder
 *   - 内部解析包括 properties、settings、typeAliases、plugins、
 *     environments、mappers 等十几个节点
 *   - 解析逻辑分布在多个 parseXxx() 私有方法中
 */
public class XMLConfigBuilder {

    private Configuration configuration;

    public XMLConfigBuilder() {
        this.configuration = new Configuration();
    }

    /**
     * 解析核心配置文件，返回 Configuration 对象
     *
     * MyBatis 源码中 parse() 方法的解析顺序：
     *   1. properties
     *   2. settings
     *   3. typeAliases
     *   4. plugins
     *   5. objectFactory
     *   6. objectWrapperFactory
     *   7. reflectorFactory
     *   8. environments
     *   9. databaseIdProvider
     *   10. typeHandlers
     *   11. mappers
     */
    public Configuration parse(InputStream inputStream) {
        try {
            SAXReader reader = new SAXReader();
            Document document = reader.read(inputStream);
            Element root = document.getRootElement();

            // 1. 解析数据源配置
            parseEnvironments(root);

            // 2. 解析 mappers 节点，加载 SQL 映射文件
            parseMappers(root);

        } catch (Exception e) {
            e.printStackTrace();
        }
        return configuration;
    }

    /**
     * 解析 environments 节点
     *   <environments default="development">
     *     <environment id="development">
     *       <transactionManager type="JDBC"/>
     *       <dataSource type="POOLED">
     *         <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
     *         <property name="url" value="jdbc:mysql://localhost:3306/test"/>
     *         <property name="username" value="root"/>
     *         <property name="password" value="123456"/>
     *       </dataSource>
     *     </environment>
     *   </environments>
     */
    private void parseEnvironments(Element root) {
        Element environments = root.element("environments");
        String defaultId = environments.attributeValue("default");

        // 通过 XPath 找到 default 对应的 environment
        Element environment = (Element) environments.selectSingleNode(
            "//environment[@id='" + defaultId + "']"
        );

        Element dataSourceElement = environment.element("dataSource");
        List<Element> props = dataSourceElement.elements("property");

        String driver = null, url = null, username = null, password = null;
        for (Element prop : props) {
            String name = prop.attributeValue("name");
            String value = prop.attributeValue("value");
            switch (name) {
                case "driver":   driver = value;   break;
                case "url":      url = value;      break;
                case "username": username = value;  break;
                case "password": password = value;  break;
            }
        }

        // 使用 Druid 数据源（或直接 DriverManager）
        DruidDataSource ds = new DruidDataSource();
        ds.setDriverClassName(driver);
        ds.setUrl(url);
        ds.setUsername(username);
        ds.setPassword(password);
        configuration.setDataSource(ds);
    }

    /**
     * 解析 mappers 节点
     *   <mappers>
     *     <mapper resource="CarMapper.xml"/>
     *     <mapper resource="StudentMapper.xml"/>
     *   </mappers>
     */
    private void parseMappers(Element root) {
        Element mappers = root.element("mappers");
        List<Element> mapperList = mappers.elements("mapper");
        for (Element mapper : mapperList) {
            String resource = mapper.attributeValue("resource");
            // 委托给 XMLMapperBuilder 解析每个 SQL 映射文件
            new XMLMapperBuilder(configuration).parse(resource);
        }
    }
}
```

## 2.5 XMLMapperBuilder — 解析 SQL 映射文件

```java
package com.godbatis.builder;

import com.godbatis.core.Configuration;
import com.godbatis.core.MappedStatement;
import org.dom4j.Document;
import org.dom4j.Element;
import org.dom4j.io.SAXReader;

import java.io.InputStream;
import java.util.List;

/**
 * XMLMapperBuilder 负责解析 XxxMapper.xml 文件
 * 将 SQL 语句封装成 MappedStatement 存入 Configuration
 *
 * MyBatis 源码对应：org.apache.ibatis.builder.xml.XMLMapperBuilder
 *   - 真正的实现还会解析 resultMap、cache、parameterMap 等
 *   - 内部使用 XMLStatementBuilder 解析每一个 SQL 节点
 *   - 使用 XMLIncludeTransformer 处理 <include> 引用
 *   - 使用 XMLScriptBuilder 解析动态 SQL（if/where/foreach等）
 */
public class XMLMapperBuilder {

    private Configuration configuration;

    public XMLMapperBuilder(Configuration configuration) {
        this.configuration = configuration;
    }

    public void parse(String resource) {
        try {
            // 从类路径加载 XML 文件
            InputStream is = Resources.getResourceAsStream(resource);
            SAXReader reader = new SAXReader();
            Document document = reader.read(is);
            Element root = document.getRootElement();

            // 获取 namespace
            String namespace = root.attributeValue("namespace");

            // 解析 select 标签
            List<Element> selectElements = root.elements("select");
            for (Element element : selectElements) {
                parseStatementElement(element, namespace, "SELECT");
            }

            // 解析 insert 标签
            List<Element> insertElements = root.elements("insert");
            for (Element element : insertElements) {
                parseStatementElement(element, namespace, "INSERT");
            }

            // 解析 update 标签
            List<Element> updateElements = root.elements("update");
            for (Element element : updateElements) {
                parseStatementElement(element, namespace, "UPDATE");
            }

            // 解析 delete 标签
            List<Element> deleteElements = root.elements("delete");
            for (Element element : deleteElements) {
                parseStatementElement(element, namespace, "DELETE");
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 解析单个 SQL 语句节点
     *
     * 在 MyBatis 源码中，这个方法由 XMLStatementBuilder 负责，
     * 核心逻辑包括：
     *   1. 提取 id、parameterType、resultType、resultMap 等属性
     *   2. 解析子节点生成 SqlNode 树（处理动态 SQL）
     *   3. 通过 LanguageDriver 创建 SqlSource
     *   4. 构建 MappedStatement 并通过 MapperBuilderAssistant 注册
     */
    private void parseStatementElement(Element element, String namespace, String commandType) {
        String id = element.attributeValue("id");
        String resultType = element.attributeValue("resultType");
        String parameterType = element.attributeValue("parameterType");

        // 获取 SQL 文本（保留空白，因为 SQL 语句需要空白分隔）
        String sql = element.getTextTrim();

        // 构建 sqlId = namespace + "." + id
        String sqlId = namespace + "." + id;

        MappedStatement ms = new MappedStatement();
        ms.setId(sqlId);
        ms.setResultType(resultType);
        ms.setParameterType(parameterType);
        ms.setSql(sql);
        ms.setSqlCommandType(commandType);

        // 注册到 Configuration
        configuration.addMappedStatement(sqlId, ms);
    }
}
```

## 2.6 SqlSession 接口与实现

```java
package com.godbatis.session;

import java.util.List;

/**
 * SqlSession 接口 — 用户操作的入口
 * 提供 CRUD 操作和 getMapper 方法
 *
 * MyBatis 源码对应：org.apache.ibatis.session.SqlSession
 *   - 真正的 SqlSession 接口有 20+ 个方法
 *   - 包括带 RowBounds 的分页方法
 *   - 以及带 ResultHandler 的流式查询方法
 */
public interface SqlSession {

    /** 插入一条记录，返回受影响的行数 */
    int insert(String sqlId, Object parameter);

    /** 删除记录，返回受影响的行数 */
    int delete(String sqlId, Object parameter);

    /** 更新记录，返回受影响的行数 */
    int update(String sqlId, Object parameter);

    /** 查询一条记录 */
    <T> T selectOne(String sqlId, Object parameter);

    /** 查询多条记录 */
    <T> List<T> selectList(String sqlId, Object parameter);

    /** 获取 Mapper 接口的动态代理对象 */
    <T> T getMapper(Class<T> mapperClass);

    /** 提交事务 */
    void commit();

    /** 回滚事务 */
    void rollback();

    /** 关闭会话 */
    void close();
}
```

```java
package com.godbatis.session.impl;

import com.godbatis.core.Configuration;
import com.godbatis.core.MappedStatement;
import com.godbatis.executor.Executor;
import com.godbatis.executor.SimpleExecutor;
import com.godbatis.proxy.MapperProxyFactory;
import com.godbatis.session.SqlSession;

import java.sql.SQLException;
import java.util.List;

/**
 * DefaultSqlSession — SqlSession 的默认实现
 *
 * MyBatis 源码对应：org.apache.ibatis.session.defaults.DefaultSqlSession
 *   - 真正的 DefaultSqlSession 持有 Configuration 和 Executor 引用
 *   - 所有 CRUD 方法最终都委托给 Executor 执行
 *   - getMapper() 委托给 Configuration.getMapper()
 *     - Configuration.getMapper() 再委托给 MapperRegistry.getMapper()
 *       - MapperRegistry 中维护 Map<Class<?>, MapperProxyFactory<?>>
 *       - 从 Map 中获取 MapperProxyFactory，调用 newInstance() 创建代理
 */
public class DefaultSqlSession implements SqlSession {

    private Configuration configuration;
    private Executor executor;

    public DefaultSqlSession(Configuration configuration) {
        this.configuration = configuration;
        this.executor = new SimpleExecutor(configuration);
    }

    @Override
    public int insert(String sqlId, Object parameter) {
        return executeUpdate(sqlId, parameter);
    }

    @Override
    public int delete(String sqlId, Object parameter) {
        return executeUpdate(sqlId, parameter);
    }

    @Override
    public int update(String sqlId, Object parameter) {
        return executeUpdate(sqlId, parameter);
    }

    /**
     * 统一的更新操作（insert/delete/update）
     *
     * 在 MyBatis 源码中：
     *   DefaultSqlSession.update() → executor.update(ms, parameter)
     *     → BaseExecutor.update() → doUpdate()
     *       → SimpleExecutor.doUpdate() → StatementHandler.update()
     */
    private int executeUpdate(String sqlId, Object parameter) {
        try {
            MappedStatement ms = configuration.getMappedStatement(sqlId);
            return executor.update(ms, parameter);
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException("执行更新操作失败: " + sqlId, e);
        }
    }

    @Override
    public <T> T selectOne(String sqlId, Object parameter) {
        List<T> list = selectList(sqlId, parameter);
        if (list.size() == 1) {
            return list.get(0);
        } else if (list.size() > 1) {
            throw new RuntimeException("期望返回一条记录，但返回了 " + list.size() + " 条");
        }
        return null;
    }

    @Override
    public <T> List<T> selectList(String sqlId, Object parameter) {
        try {
            MappedStatement ms = configuration.getMappedStatement(sqlId);
            return executor.query(ms, parameter);
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException("执行查询操作失败: " + sqlId, e);
        }
    }

    @Override
    public <T> T getMapper(Class<T> mapperClass) {
        // 委托给 MapperProxyFactory 创建动态代理对象
        return MapperProxyFactory.getMapper(mapperClass, this);
    }

    @Override
    public void commit() {
        try {
            executor.commit();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void rollback() {
        try {
            executor.rollback();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void close() {
        executor.close();
    }
}
```

## 2.7 SqlSessionFactory 接口与实现

```java
package com.godbatis.session;

/**
 * SqlSessionFactory — 创建 SqlSession 的工厂
 *
 * MyBatis 源码对应：org.apache.ibatis.session.SqlSessionFactory
 *   - 提供多个重载的 openSession() 方法
 *     - 无参：默认自动提交=false，无特定 ExecutorType
 *     - 带 boolean autoCommit 参数
 *     - 带 ExecutorType 参数（SIMPLE/REUSE/BATCH）
 *     - 带 TransactionIsolationLevel 参数
 */
public interface SqlSessionFactory {
    SqlSession openSession();
}
```

```java
package com.godbatis.session.impl;

import com.godbatis.core.Configuration;
import com.godbatis.session.SqlSession;
import com.godbatis.session.SqlSessionFactory;

/**
 * DefaultSqlSessionFactory
 *
 * MyBatis 源码对应：org.apache.ibatis.session.defaults.DefaultSqlSessionFactory
 *   - openSession() 方法内部：
 *     - 从 Configuration 获取 Environment → TransactionFactory
 *     - 创建 Transaction 对象（JdbcTransaction 或 ManagedTransaction）
 *     - 根据 ExecutorType 创建 Executor（Simple/Reuse/Batch）
 *     - 如果有二级缓存开启，用 CachingExecutor 包装
 *     - 如果有插件，用 InterceptorChain 包装 Executor
 *     - 最后 new DefaultSqlSession(configuration, executor, autoCommit)
 */
public class DefaultSqlSessionFactory implements SqlSessionFactory {

    private Configuration configuration;

    public DefaultSqlSessionFactory(Configuration configuration) {
        this.configuration = configuration;
    }

    @Override
    public SqlSession openSession() {
        return new DefaultSqlSession(configuration);
    }
}
```

## 2.8 SqlSessionFactoryBuilder

```java
package com.godbatis.session;

import com.godbatis.builder.XMLConfigBuilder;
import com.godbatis.core.Configuration;
import com.godbatis.session.impl.DefaultSqlSessionFactory;

import java.io.InputStream;

/**
 * SqlSessionFactoryBuilder — 构建 SqlSessionFactory
 *
 * MyBatis 源码对应：org.apache.ibatis.session.SqlSessionFactoryBuilder
 *   - 提供多个重载的 build() 方法：
 *     - build(InputStream)
 *     - build(InputStream, String environment)
 *     - build(InputStream, Properties)
 *     - build(Configuration)
 *   - build() 核心流程：
 *     1. new XMLConfigBuilder(inputStream, environment, props)
 *     2. XMLConfigBuilder.parse() → Configuration (解析全过程)
 *     3. new DefaultSqlSessionFactory(configuration)
 *   - 注意：build() 方法完成后会关闭 InputStream
 *
 * 【重要源码分析】
 * 在 MyBatis 3.x 的 SqlSessionFactoryBuilder.build() 中：
 *
 *   public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
 *     try {
 *       XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
 *       return build(parser.parse());  // ← parser.parse() 解析出 Configuration
 *     } catch (Exception e) {
 *       throw ExceptionFactory.wrapException("Error building SqlSession.", e);
 *     } finally {
 *       ErrorContext.instance().reset();
 *       try {
 *         inputStream.close();  // ← 内部会关闭 InputStream
 *       } catch (IOException e) { }
 *     }
 *   }
 *
 * 这就是为什么我们不需要手动关闭传入的 InputStream
 */
public class SqlSessionFactoryBuilder {

    public SqlSessionFactory build(InputStream inputStream) {
        // 1. 使用 XMLConfigBuilder 解析配置文件
        XMLConfigBuilder configBuilder = new XMLConfigBuilder();
        Configuration configuration = configBuilder.parse(inputStream);

        // 2. 根据 Configuration 创建 DefaultSqlSessionFactory
        return new DefaultSqlSessionFactory(configuration);
    }
}
```

## 2.9 Executor — SQL 执行器

```java
package com.godbatis.executor;

import com.godbatis.core.MappedStatement;

import java.sql.SQLException;
import java.util.List;

/**
 * Executor 接口 — SQL 执行器
 * 负责实际的 JDBC 操作
 *
 * MyBatis 源码对应：org.apache.ibatis.executor.Executor
 *   - Executor 体系是 MyBatis 执行层的核心：
 *     Executor (接口)
 *       ├── BaseExecutor (抽象类，实现缓存和事务管理)
 *       │     ├── SimpleExecutor (每次执行都创建新的 Statement)
 *       │     ├── ReuseExecutor (复用 Statement)
 *       │     └── BatchExecutor (批量执行)
 *       └── CachingExecutor (装饰器，实现二级缓存)
 *
 *   - Executor 的核心方法：
 *     - update(MappedStatement, Object) → int
 *     - query(MappedStatement, Object, RowBounds, ResultHandler) → List
 *     - queryCursor(MappedStatement, Object, RowBounds) → Cursor
 *     - commit(boolean required)
 *     - rollback(boolean required)
 *     - close(boolean forceRollback)
 *     - createCacheKey(MappedStatement, Object, RowBounds, BoundSql)
 *     - isCached(MappedStatement, CacheKey)
 *     - clearLocalCache()
 */
public interface Executor {
    int update(MappedStatement ms, Object parameter) throws Exception;
    <T> List<T> query(MappedStatement ms, Object parameter) throws Exception;
    void commit() throws SQLException;
    void rollback() throws SQLException;
    void close();
}
```

```java
package com.godbatis.executor;

import com.godbatis.core.Configuration;
import com.godbatis.core.MappedStatement;
import com.godbatis.utils.GenericTokenParser;
import com.godbatis.utils.ParameterMappingTokenHandler;

import javax.sql.DataSource;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

/**
 * SimpleExecutor — 简单执行器
 * 每次执行 SQL 时都创建一个新的 PreparedStatement
 *
 * MyBatis 源码对应：org.apache.ibatis.executor.SimpleExecutor
 *   - doUpdate() 流程：
 *     1. 调用 configuration.newStatementHandler() 创建 StatementHandler
 *     2. 调用 prepareStatement(handler) 获取 PreparedStatement
 *     3. 调用 handler.update(stmt) 执行
 *     4. 返回影响行数
 *
 *   - doQuery() 流程：
 *     1. 调用 configuration.newStatementHandler() 创建 StatementHandler
 *     2. 调用 prepareStatement(handler) 获取 PreparedStatement
 *     3. 调用 handler.query(stmt, resultHandler) 执行查询
 *     4. 返回结果列表
 */
public class SimpleExecutor implements Executor {

    private Configuration configuration;
    private Connection connection;

    public SimpleExecutor(Configuration configuration) {
        this.configuration = configuration;
        try {
            DataSource dataSource = configuration.getDataSource();
            this.connection = dataSource.getConnection();
            // 关闭自动提交（手动管理事务）
            this.connection.setAutoCommit(false);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    /**
     * 执行更新操作（insert/delete/update）
     *
     * 核心流程：
     *   1. 获取 MappedStatement 中的原始 SQL
     *   2. 解析 #{} 占位符，替换为 ?
     *   3. 创建 PreparedStatement
     *   4. 给 ? 传值（通过反射获取参数对象的属性值）
     *   5. 执行 executeUpdate()
     *   6. 返回影响行数
     */
    @Override
    public int update(MappedStatement ms, Object parameter) throws Exception {
        // 1. 获取原始 SQL
        String originalSql = ms.getSql();

        // 2. 解析 #{} 占位符 → 替换为 ?
        //    "insert into t_car values(null,#{carNum},#{brand})"
        //    → 变成:
        //    "insert into t_car values(null,?,?)"
        //    同时解析出参数名列表: ["carNum", "brand"]
        ParameterMappingTokenHandler tokenHandler = new ParameterMappingTokenHandler();
        GenericTokenParser tokenParser = new GenericTokenParser("#{", "}", tokenHandler);
        String parsedSql = tokenParser.parse(originalSql);
        List<String> parameterNames = tokenHandler.getParameterNames();

        System.out.println("解析后的 SQL: " + parsedSql);

        // 3. 创建 PreparedStatement
        PreparedStatement ps = connection.prepareStatement(parsedSql);

        // 4. 给 ? 占位符传值
        if (parameter != null && parameterNames.size() > 0) {
            setParameters(ps, parameter, parameterNames);
        }

        // 5. 执行更新
        int count = ps.executeUpdate();
        ps.close();

        return count;
    }

    /**
     * 给 PreparedStatement 的占位符设值
     *
     * 如果是 Map 类型的参数：
     *   通过 map.get(parameterName) 获取值
     *
     * 如果是 POJO 类型的参数：
     *   通过反射调用 getter 方法获取值
     *
     * MyBatis 源码中这个逻辑在 DefaultParameterHandler 中：
     *   - 它通过 TypeHandler 进行类型处理
     *   - 获取值的逻辑在参数映射的 ParameterMapping 中
     */
    private void setParameters(PreparedStatement ps, Object parameter, List<String> parameterNames) throws Exception {
        if (parameter instanceof java.util.Map) {
            // Map 参数：#{key} → map.get(key)
            @SuppressWarnings("unchecked")
            java.util.Map<String, Object> map = (java.util.Map<String, Object>) parameter;
            for (int i = 0; i < parameterNames.size(); i++) {
                String name = parameterNames.get(i);
                Object value = map.get(name);
                ps.setObject(i + 1, value); // JDBC 索引从 1 开始
            }
        } else {
            // POJO 参数：#{propertyName} → 反射调用 getPropertyName()
            Class<?> clazz = parameter.getClass();
            for (int i = 0; i < parameterNames.size(); i++) {
                String propertyName = parameterNames.get(i);
                // getter 方法名: get + 首字母大写
                String methodName = "get" + propertyName.substring(0, 1).toUpperCase() + propertyName.substring(1);
                Method getter = clazz.getMethod(methodName);
                Object value = getter.invoke(parameter);
                ps.setObject(i + 1, value);
            }
        }
    }

    /**
     * 执行查询操作
     *
     * 与 update 类似，区别在于：
     *   - 调用 executeQuery() 而非 executeUpdate()
     *   - 需要将 ResultSet 结果集映射为 Java 对象
     */
    @Override
    public <T> List<T> query(MappedStatement ms, Object parameter) throws Exception {
        String originalSql = ms.getSql();

        // 解析 #{} 占位符
        ParameterMappingTokenHandler tokenHandler = new ParameterMappingTokenHandler();
        GenericTokenParser tokenParser = new GenericTokenParser("#{", "}", tokenHandler);
        String parsedSql = tokenParser.parse(originalSql);
        List<String> parameterNames = tokenHandler.getParameterNames();

        PreparedStatement ps = connection.prepareStatement(parsedSql);

        // 设置参数
        if (parameter != null && parameterNames.size() > 0) {
            setParameters(ps, parameter, parameterNames);
        }

        // 执行查询
        ResultSet rs = ps.executeQuery();

        // 结果集 → Java 对象映射
        String resultType = ms.getResultType();
        Class<?> resultClass = Class.forName(resultType);
        List<T> resultList = new ArrayList<>();

        while (rs.next()) {
            @SuppressWarnings("unchecked")
            T obj = (T) resultClass.getDeclaredConstructor().newInstance();

            // 获取结果集元数据
            ResultSetMetaData metaData = rs.getMetaData();
            int columnCount = metaData.getColumnCount();

            for (int i = 1; i <= columnCount; i++) {
                // 获取列名（数据库列名可能是下划线风格）
                String columnName = metaData.getColumnName(i);
                // 列名转驼峰: car_num → carNum
                String propertyName = underlineToCamel(columnName);
                Object value = rs.getObject(i);

                // 通过反射设置属性值
                try {
                    Field field = resultClass.getDeclaredField(propertyName);
                    field.setAccessible(true);
                    field.set(obj, value);
                } catch (NoSuchFieldException e) {
                    // 忽略没有对应字段的列
                }
            }

            resultList.add(obj);
        }

        rs.close();
        ps.close();

        return resultList;
    }

    /**
     * 下划线转驼峰: car_num → carNum
     */
    private String underlineToCamel(String str) {
        StringBuilder result = new StringBuilder();
        boolean nextUpperCase = false;
        for (char c : str.toCharArray()) {
            if (c == '_') {
                nextUpperCase = true;
            } else {
                if (nextUpperCase) {
                    result.append(Character.toUpperCase(c));
                    nextUpperCase = false;
                } else {
                    result.append(Character.toLowerCase(c));
                }
            }
        }
        return result.toString();
    }

    @Override
    public void commit() throws SQLException {
        connection.commit();
    }

    @Override
    public void rollback() throws SQLException {
        connection.rollback();
    }

    @Override
    public void close() {
        try {
            if (connection != null && !connection.isClosed()) {
                connection.close();
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

## 2.10 #{} 占位符解析器

```java
package com.godbatis.utils;

import java.util.ArrayList;
import java.util.List;

/**
 * GenericTokenParser — 通用的占位符解析器
 *
 * 功能：将字符串中的 #{xxx} 替换为 ?，并记录 xxx
 *
 * MyBatis 源码对应：org.apache.ibatis.parsing.GenericTokenParser
 *   - 这是 MyBatis 中非常经典的解析器设计
 *   - 采用"策略模式"：GenericTokenParser 负责解析占位符
 *     TokenHandler 负责处理解析到的内容
 *   - 在 MyBatis 中有两个主要的 TokenHandler：
 *     1. ParameterMappingTokenHandler — 处理 #{}
 *        - 将 #{} 替换为 ?
 *        - 记录参数名称，构建 ParameterMapping 列表
 *     2. DynamicCheckerTokenParser — 检查是否是动态 SQL
 *        - 判断 SQL 中是否含有 #{} 或 ${}
 *   - 解析算法：
 *     遍历字符串，找到 openToken（"#{" 或 "${"）
 *     再找到 closeToken（"}"）
 *     中间的内容（"xxx"）交给 TokenHandler 处理
 *     返回处理后的完整字符串
 */
public class GenericTokenParser {

    private String openToken;   // 开始标记，如 "#{"
    private String closeToken;  // 结束标记，如 "}"
    private TokenHandler handler; // 处理回调

    public GenericTokenParser(String openToken, String closeToken, TokenHandler handler) {
        this.openToken = openToken;
        this.closeToken = closeToken;
        this.handler = handler;
    }

    /**
     * 解析字符串中的占位符
     * 例如："insert into t_car values(#{carNum}, #{brand})"
     *   → "insert into t_car values(?, ?)"
     */
    public String parse(String text) {
        if (text == null || text.isEmpty()) {
            return "";
        }

        StringBuilder builder = new StringBuilder();
        int start = text.indexOf(openToken);
        if (start == -1) {
            return text; // 没有占位符，直接返回原文
        }

        char[] src = text.toCharArray();
        int offset = 0;

        while (start > -1) {
            // 追加占位符之前的内容
            builder.append(src, offset, start - offset);
            offset = start + openToken.length();

            // 找到结束标记
            int end = text.indexOf(closeToken, offset);
            if (end == -1) {
                // 没有找到结束标记，保留原文
                builder.append(src, start, src.length - start);
                offset = src.length;
                break;
            }

            // 提取占位符中的内容 xxx = #{xxx}
            String expression = new String(src, offset, end - offset).trim();

            // 交给 handler 处理
            String replacement = handler.handleToken(expression);
            builder.append(replacement);

            offset = end + closeToken.length();
            start = text.indexOf(openToken, offset);
        }

        // 追加剩余内容
        if (offset < src.length) {
            builder.append(src, offset, src.length - offset);
        }

        return builder.toString();
    }
}

/**
 * TokenHandler — 占位符处理接口
 */
interface TokenHandler {
    String handleToken(String content);
}

/**
 * ParameterMappingTokenHandler — 参数映射处理器
 *
 * 功能：
 *   1. 将 #{} 替换为 ?
 *   2. 记录 #{xxx} 中的 xxx（参数名）
 */
class ParameterMappingTokenHandler implements TokenHandler {

    private List<String> parameterNames = new ArrayList<>();

    @Override
    public String handleToken(String content) {
        // 记录参数名
        parameterNames.add(content);
        // 替换为 JDBC 占位符 ?
        return "?";
    }

    public List<String> getParameterNames() {
        return parameterNames;
    }
}
```

## 2.11 MapperProxy — 动态代理（getMapper 实现）

```java
package com.godbatis.proxy;

import com.godbatis.session.SqlSession;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.List;

/**
 * MapperProxy — Mapper 接口的 JDK 动态代理
 *
 * 当调用 mapper.selectById(1) 时：
 *   → MapperProxy.invoke() 被调用
 *   → 通过接口全限定名 + 方法名构建 sqlId
 *   → 委托给 SqlSession 执行对应的方法
 *
 * MyBatis 源码对应：org.apache.ibatis.binding.MapperProxy
 *
 * 【重要源码分析】
 * MyBatis 中 MapperProxy.invoke() 的完整流程：
 *
 *   public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
 *     // 1. 如果方法是 Object 类的方法（toString/equals/hashCode），直接调用
 *     if (Object.class.equals(method.getDeclaringClass())) {
 *       return method.invoke(this, args);
 *     }
 *     // 2. 如果是 default 方法（Java 8+），使用 MethodHandle 调用
 *     if (isDefaultMethod(method)) {
 *       return invokeDefaultMethod(proxy, method, args);
 *     }
 *     // 3. 核心：通过 MapperMethod 执行 SQL
 *     //    先从缓存中获取或创建 MapperMethod
 *     MapperMethod mapperMethod = cachedMapperMethod(method);
 *     return mapperMethod.execute(sqlSession, args);
 *   }
 *
 * MapperMethod.execute() 流程：
 *
 *   public Object execute(SqlSession sqlSession, Object[] args) {
 *     switch (command.getType()) {
 *       case INSERT: {
 *         Object param = method.convertArgsToSqlCommandParam(args);
 *         int rowCount = sqlSession.insert(command.getName(), param);
 *         // 如果返回类型是 void/int/long/boolean，做相应转换
 *         return rowCountResult(rowCount);
 *       }
 *       case UPDATE: { ... }
 *       case DELETE: { ... }
 *       case SELECT: {
 *         // 根据返回类型（void/List/Map/Cursor/自定义对象）选择不同方法
 *         if (method.returnsVoid()) {
 *           sqlSession.select(command.getName(), param, resultHandler);
 *           return null;
 *         } else if (method.returnsMany()) {
 *           return sqlSession.selectList(command.getName(), param);
 *         } else if (method.returnsMap()) {
 *           return sqlSession.selectMap(command.getName(), param,
 *                                       method.getMapKey());
 *         } else if (method.returnsCursor()) {
 *           return sqlSession.selectCursor(command.getName(), param);
 *         } else {
 *           return sqlSession.selectOne(command.getName(), param);
 *         }
 *       }
 *       case FLUSH: { ... }
 *     }
 *   }
 */
public class MapperProxy implements InvocationHandler {

    private SqlSession sqlSession;

    public MapperProxy(SqlSession sqlSession) {
        this.sqlSession = sqlSession;
    }

    /**
     * 拦截所有对 Mapper 接口方法的调用
     *
     * sqlId 的构建规则：接口全限定名 + "." + 方法名
     * 例如：com.godbatis.mapper.CarMapper.selectById
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 获取接口的全限定名
        String interfaceName = method.getDeclaringClass().getName();
        // 获取方法名
        String methodName = method.getName();
        // 构建 sqlId
        String sqlId = interfaceName + "." + methodName;

        // 根据方法的返回类型决定调用 SqlSession 的哪个方法
        Class<?> returnType = method.getReturnType();

        // 参数：如果没有参数传 null，如果只有一个参数直接传，多个参数包装为 Map
        Object parameter = (args != null && args.length > 0) ? args[0] : null;

        if (returnType == void.class || returnType == int.class || returnType == Integer.class
            || returnType == long.class || returnType == Long.class
            || returnType == boolean.class || returnType == Boolean.class) {
            // 增删改操作 — 通过方法名判断
            String name = methodName.toLowerCase();
            if (name.startsWith("insert") || name.startsWith("save") || name.startsWith("add")) {
                return sqlSession.insert(sqlId, parameter);
            } else if (name.startsWith("delete") || name.startsWith("remove")) {
                return sqlSession.delete(sqlId, parameter);
            } else if (name.startsWith("update") || name.startsWith("modify")) {
                return sqlSession.update(sqlId, parameter);
            }
        }

        if (returnType == List.class) {
            return sqlSession.selectList(sqlId, parameter);
        }

        // 默认当作查询单条记录
        return sqlSession.selectOne(sqlId, parameter);
    }
}

/**
 * MapperProxyFactory — 创建 MapperProxy 的工厂
 *
 * MyBatis 源码对应：org.apache.ibatis.binding.MapperProxyFactory
 *
 * 在 MyBatis 中，每个 Mapper 接口对应一个 MapperProxyFactory 实例，
 * 存储在 MapperRegistry 的 Map<Class<?>, MapperProxyFactory<?>> 中。
 *
 * getMapper() 调用 newInstance() 创建代理对象：
 *   protected T newInstance(MapperProxy<T> mapperProxy) {
 *     return (T) Proxy.newProxyInstance(
 *         mapperInterface.getClassLoader(),
 *         new Class[] { mapperInterface },
 *         mapperProxy
 *     );
 *   }
 */
class MapperProxyFactory {

    @SuppressWarnings("unchecked")
    public static <T> T getMapper(Class<T> mapperClass, SqlSession sqlSession) {
        return (T) Proxy.newProxyInstance(
            mapperClass.getClassLoader(),
            new Class[]{mapperClass},
            new MapperProxy(sqlSession)
        );
    }
}
```

## 2.12 Resources 工具类

```java
package com.godbatis.utils;

import java.io.InputStream;

/**
 * Resources — 资源加载工具类
 *
 * MyBatis 源码对应：org.apache.ibatis.io.Resources
 *   - 真正的 Resources 提供了多种重载的 getResourceAsStream 方法
 *   - 还提供了 getResourceAsFile、getUrlAsStream 等方法
 *   - 使用 ClassLoader.getResourceAsStream() 从类路径加载资源
 *   - 内部使用 ClassLoaderWrapper 封装了 5 种 ClassLoader 尝试策略：
 *     1. 当前线程的 ContextClassLoader
 *     2. 当前类的 ClassLoader
 *     3. 系统 ClassLoader
 *     4. 等等
 */
public class Resources {
    public static InputStream getResourceAsStream(String resource) {
        return Resources.class.getClassLoader().getResourceAsStream(resource);
    }
}
```

# 3.GodBatis 使用 Maven 打包

```xml
<!-- pom.xml -->
<project>
    <groupId>com.godbatis</groupId>
    <artifactId>godbatis</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <dependencies>
        <!-- dom4j: XML 解析 -->
        <dependency>
            <groupId>org.dom4j</groupId>
            <artifactId>dom4j</artifactId>
            <version>2.1.3</version>
        </dependency>
        <!-- Druid 连接池 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.2.16</version>
        </dependency>
        <!-- MySQL 驱动 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.30</version>
        </dependency>
    </dependencies>
</project>
```

# 4.使用 GodBatis

## 4.1 配置文件 mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/test"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="CarMapper.xml"/>
    </mappers>
</configuration>
```

## 4.2 SQL 映射文件 CarMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<mapper namespace="car">
    <insert id="insertCar">
        insert into t_car values(null,#{carNum},#{brand},#{guidePrice},#{produceTime},#{carType})
    </insert>
    <select id="selectAll" resultType="com.test.pojo.Car">
        select id,car_num as carNum,brand,guide_price as guidePrice,
               produce_time as produceTime,car_type as carType from t_car
    </select>
</mapper>
```

## 4.3 编写测试代码

```java
public class GodBatisTest {
    @Test
    public void testInsert() {
        InputStream is = Resources.getResourceAsStream("mybatis-config.xml");
        SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(is);
        SqlSession sqlSession = factory.openSession();

        // 准备数据
        Map<String, Object> map = new HashMap<>();
        map.put("carNum", "1001");
        map.put("brand", "凯美瑞");
        map.put("guidePrice", 35.0);
        map.put("produceTime", "2020-10-10");
        map.put("carType", "燃油车");

        int count = sqlSession.insert("car.insertCar", map);
        System.out.println("插入 " + count + " 条记录");
        sqlSession.commit();
        sqlSession.close();
    }

    @Test
    public void testSelectAll() {
        InputStream is = Resources.getResourceAsStream("mybatis-config.xml");
        SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(is);
        SqlSession sqlSession = factory.openSession();

        List<Car> cars = sqlSession.selectList("car.selectAll", null);
        cars.forEach(System.out::println);
        sqlSession.close();
    }
}
```

# 5.总结 MyBatis 框架的重要实现原理

```
┌────────────────────────────────────────────────────────────────────┐
│                   MyBatis 核心执行流程总结                              │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. 启动阶段（配置解析）                                              │
│     ┌─────────────────────────────────────────────────────┐        │
│     │ SqlSessionFactoryBuilder.build(InputStream)          │        │
│     │   → XMLConfigBuilder.parse()                        │        │
│     │     → 解析 mybatis-config.xml                        │        │
│     │       ├── properties（属性）                          │        │
│     │       ├── settings（全局设置）                        │        │
│     │       ├── typeAliases（类型别名）                     │        │
│     │       ├── plugins（插件/拦截器）                      │        │
│     │       ├── environments（数据源/事务）                  │        │
│     │       └── mappers（SQL 映射文件）                     │        │
│     │         → XMLMapperBuilder.parse()                  │        │
│     │           → 解析每个 XxxMapper.xml                    │        │
│     │           → 生成 MappedStatement                      │        │
│     │           → 注册到 Configuration.mappedStatements    │        │
│     │     → new DefaultSqlSessionFactory(configuration)   │        │
│     └─────────────────────────────────────────────────────┘        │
│                                                                    │
│  2. 运行阶段（SQL 执行）                                              │
│     ┌─────────────────────────────────────────────────────┐        │
│     │ SqlSession.getMapper(XXMapper.class)                  │        │
│     │   → MapperProxy (JDK 动态代理)                        │        │
│     │   → mapper.selectById(1)                            │        │
│     │     → MapperProxy.invoke()                          │        │
│     │       → MapperMethod.execute()                      │        │
│     │         → sqlSession.selectOne(sqlId, param)        │        │
│     │           → executor.query(ms, param)               │        │
│     │             ├── 一级缓存查询 (BaseExecutor)           │        │
│     │             ├── 二级缓存查询 (CachingExecutor)        │        │
│     │             ├── doQuery()                           │        │
│     │             │   ├── StatementHandler 创建 Statement   │        │
│     │             │   ├── ParameterHandler 设置参数          │        │
│     │             │   ├── Statement.execute() 执行          │        │
│     │             │   └── ResultSetHandler 结果映射           │        │
│     │             └── 将结果放入一级缓存                      │        │
│     └─────────────────────────────────────────────────────┘        │
│                                                                    │
│  3. 核心设计模式                                                      │
│     ├── 建造者模式：SqlSessionFactoryBuilder、XMLConfigBuilder         │
│     ├── 工厂模式：SqlSessionFactory、MapperProxyFactory              │
│     ├── 代理模式：MapperProxy（JDK 动态代理，实现 Mapper 接口代理）      │
│     ├── 装饰器模式：CachingExecutor 装饰 BaseExecutor                  │
│     ├── 模板方法模式：BaseExecutor 定义骨架，子类实现 doXxx()           │
│     ├── 策略模式：不同的 Executor 实现（Simple/Reuse/Batch）           │
│     └── 责任链模式：InterceptorChain（插件拦截器链）                    │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 5.1 关键扩展点

```
MyBatis 提供了多个扩展点，允许用户自定义行为：

1. TypeHandler（类型处理器）
   - 自定义 Java 类型 ↔ JDBC 类型的转换
   - 例如：将 JSON 字符串自动映射为 Java 对象
   - 实现 TypeHandler 接口或继承 BaseTypeHandler

2. Plugin / Interceptor（插件 / 拦截器）
   - 可拦截 Executor、StatementHandler、ParameterHandler、ResultSetHandler
   - 典型应用：PageHelper 分页插件、读写分离插件
   - 实现 Interceptor 接口并使用 @Intercepts 注解

3. ObjectFactory（对象工厂）
   - 自定义实例化结果对象的方式
   - 默认使用构造器反射，可以改为 CGLIB 等方式

4. LanguageDriver（语言驱动）
   - 自定义 SQL 解析方式
   - 默认是 XMLLanguageDriver，可以自定义注解式 SQL 解析
```

### 5.2 与真正 MyBatis 的差距

```
我们的 GodBatis 是一个极简版，与真正的 MyBatis 相比，缺少以下核心特性：

1. 动态 SQL（if/where/trim/foreach/choose/set）
   → 需要实现 SqlNode 树结构，构建 AST 并在运行时动态拼接 SQL
   → 真正的实现在 org.apache.ibatis.scripting.xmltags 包中

2. 标签的语义化解析（selectOne / selectList 自动判断）
   → 需要根据 select 标签的属性或映射方法签名自动判断
   → 真正的实现在 MapperMethod 的 resolveReturnType() 中

3. 完整的 TypeHandler 体系
   → 需要支持各种 Java 类型到 JDBC 类型的自动映射
   → 真正的实现在 org.apache.ibatis.type 包中

4. 一级缓存和二级缓存
   → 需要实现 BaseExecutor 中的 localCache（PerpetualCache）
   → 二级缓存需要 CachingExecutor + TransactionalCache
   → 缓存 Key 的计算需要 CacheKey 类

5. 插件拦截器链
   → 需要实现 InterceptorChain + Plugin 动态代理包装

6. 结果映射的高级特性
   → association（一对一/多对一嵌套映射）
   → collection（一对多嵌套映射）
   → discriminator（鉴别器）
   → 延迟加载（需要生成代理对象，拦截 getter 方法调用）

7. 注解式开发支持
   → @Select、@Insert、@Update、@Delete 注解解析
   → 需要实现注解语言驱动（AnnotationLanguageDriver）

8. 与 Spring 的整合
   → SqlSessionFactoryBean、MapperFactoryBean
   → MapperScannerConfigurer 自动扫描 Mapper 接口
   → SqlSessionTemplate 线程安全封装
   → 事务管理器整合
```

### 5.3 推荐阅读源码的入口

```
如果你想深入阅读 MyBatis 源码，建议按以下顺序：

1. SqlSessionFactoryBuilder.build() → 了解配置加载入口
   - 文件：org.apache.ibatis.session.SqlSessionFactoryBuilder

2. XMLConfigBuilder.parse() → 了解核心配置解析
   - 文件：org.apache.ibatis.builder.xml.XMLConfigBuilder

3. Configuration.newExecutor() → 了解 Executor 创建和拦截器包装
   - 文件：org.apache.ibatis.session.Configuration

4. MapperProxy.invoke() → 了解 Mapper 接口代理机制
   - 文件：org.apache.ibatis.binding.MapperProxy

5. BaseExecutor.query() → 了解一级缓存和查询流程
   - 文件：org.apache.ibatis.executor.BaseExecutor

6. SimpleExecutor.doQuery() → 了解实际 JDBC 执行
   - 文件：org.apache.ibatis.executor.SimpleExecutor

7. DefaultResultSetHandler.handleResultSet() → 了解结果映射
   - 文件：org.apache.ibatis.executor.resultset.DefaultResultSetHandler

8. XMLScriptBuilder.parseScriptNode() → 了解动态 SQL 解析
   - 文件：org.apache.ibatis.scripting.xmltags.XMLScriptBuilder
```
