# 1. 框架
	Java常⽤框架：
	
	SSM三⼤框架：Spring + SpringMVC + MyBatis
	SpringBoot
	SpringCloud 等。。
	
	SSM三⼤框架的学习顺序：MyBatis、Spring、SpringMVC（建议）

# 2. 三层架构
	表现层（UI）：直接跟前端打交互（⼀是接收前端ajax请求，⼆是返回json数据给前端）
	业务逻辑层（BLL）：⼀是处理表现层转发过来的前端请求（也就是具体业务），⼆是将从持久层获取的数据返回到表现层。
	数据访问层（DAL）：直接操作数据库完成CRUD，并将获得的数据返回到上⼀层（也就是业务逻辑层）。

# 3. JDBC的不足
	SQL语句写死在Java程序中，不灵活。改SQL的话就要改Java代码。违背开闭原则OCP。
	给占位符 ? 传值是繁琐的。能不能⾃动化？？？
	将结果集封装成Java对象是繁琐的。能不能⾃动化？？？

# 4. 了解MyBatis
	MyBatis本质上就是对JDBC的封装，通过MyBatis完成CRUD。
	MyBatis在三层架构中负责持久层的，属于持久层框架。
	ORM：对象关系映射
	O（Object）：Java虚拟机中的Java对象
	R（Relational）：关系型数据库
	M（Mapping）：将Java虚拟机中的Java对象映射到数据库表中⼀⾏记录，或是将数据库表中⼀⾏记录映射成Java虚拟机中的⼀个Java对象。
	MyBatis框架特点：
	
	⽀持定制化 SQL、存储过程、基本映射以及⾼级映射
	
	避免了⼏乎所有的 JDBC 代码中⼿动设置参数以及获取结果集
	
	⽀持XML开发，也⽀持注解式开发。【为了保证sql语句的灵活，所以mybatis⼤部分是采⽤
	
	XML⽅式开发。】
	
	将接⼝和 Java 的 POJOs(Plain Ordinary Java Object，简单普通的Java对象)映射成数据库中的
	
	记录
	
	体积⼩好学：两个jar包，两个XML配置⽂件。
	
	完全做到sql解耦合。
	
	提供了基本映射标签。
	
	提供了⾼级映射标签。
	
	提供了XML标签，⽀持动态SQL的编写。

# 5. MyBatis 与 Hibernate 的对比

```
┌───────────────┬──────────────────────────┬───────────────────────────┐
│     维度       │         MyBatis           │        Hibernate           │
├───────────────┼──────────────────────────┼───────────────────────────┤
│  ORM 类型      │ 半自动 ORM                │ 全自动 ORM                │
│  SQL 控制      │ 开发者完全掌控 SQL         │ 框架自动生成 SQL           │
│  灵活性        │ 极高（手写 SQL，可优化）     │ 中等（HQL/JPQL 抽象）      │
│  学习曲线       │ 较低（SQL 熟悉即可）        │ 较高（需学习 HQL、映射配置） │
│  数据库移植性   │ 较低（SQL 与数据库耦合）     │ 较高（方言 Dialect 机制）    │
│  复杂查询       │ 优势（直接写 SQL）          │ 较难（HQL 表达能力有限）     │
│  缓存机制       │ 一级+二级缓存（较简单）      │ 一级+二级+查询缓存（更完善） │
│  动态 SQL       │ 强大的 XML 标签支持         │ Criteria API / JPQL       │
│  适用场景       │ 复杂业务、高频优化、报表     │ 标准 CRUD、快速开发         │
│  代表产品       │ MyBatis / MyBatis-Plus    │ Hibernate / JPA / Spring Data JPA │
└───────────────┴──────────────────────────┴───────────────────────────┘

结论：
  - MyBatis 适合：互联网项目、SQL 需要精细优化、复杂多表关联查询
  - Hibernate 适合：企业级标准化项目、快速 CRUD、对数据库移植性要求高
  - 实际趋势：国内 MyBatis 占据绝对主导地位（灵活 > 自动化）
```

# 6. MyBatis 核心架构全景

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          MyBatis 核心架构图                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │                         接口层 (API Layer)                            ││
│  │                                                                     ││
│  │   SqlSession (用户操作入口)          Mapper 接口 (@Select 等注解)      ││
│  │   ├── selectOne / selectList        ├── getMapper() 获取代理          ││
│  │   ├── insert / update / delete      └── 直接调用接口方法              ││
│  │   └── commit / rollback / close                                      ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                    │                                     │
│                                    ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │                        数据处理层 (Data Processing)                    ││
│  │                                                                     ││
│  │   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  ││
│  │   │  ParameterHandler │  │  ResultSetHandler │  │  StatementHandler│  ││
│  │   │  (参数设置)        │  │  (结果映射)        │  │  (语句创建/执行)   │  ││
│  │   └──────────────────┘  └──────────────────┘  └──────────────────┘  ││
│  │                                                                     ││
│  │   ┌──────────────────────────────────────────────────────────────┐  ││
│  │   │                    Executor (SQL 执行器)                       │  ││
│  │   │   ┌─────────────┐   ┌──────────────┐   ┌──────────────┐      │  ││
│  │   │   │SimpleExecutor│   │ReuseExecutor │   │BatchExecutor │      │  ││
│  │   │   │(简单执行器)   │   │(复用Statement)│   │(批量执行器)   │      │  ││
│  │   │   └─────────────┘   └──────────────┘   └──────────────┘      │  ││
│  │   │   ┌──────────────────────────────────────────────────────┐   │  ││
│  │   │   │            CachingExecutor (二级缓存装饰器)             │   │  ││
│  │   │   └──────────────────────────────────────────────────────┘   │  ││
│  │   └──────────────────────────────────────────────────────────────┘  ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                    │                                     │
│                                    ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │                        基础支撑层 (Foundation)                         ││
│  │                                                                     ││
│  │   ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────────┐  ││
│  │   │ 数据源      │ │ 事务管理    │ │ 缓存         │ │ 连接池          │  ││
│  │   │ DataSource │ │ Transaction│ │ Cache       │ │ PooledDataSource│  ││
│  │   └────────────┘ └────────────┘ └────────────┘ └────────────────┘  ││
│  │                                                                     ││
│  │   ┌──────────────────────────────────────────────────────────────┐  ││
│  │   │   配置解析层                                                    │  ││
│  │   │   XMLConfigBuilder → XMLMapperBuilder → MapperAnnotationBuilder│  ││
│  │   │   解析 mybatis-config.xml       解析 XxxMapper.xml     解析注解  │  ││
│  │   └──────────────────────────────────────────────────────────────┘  ││
│  │                                                                     ││
│  │   ┌──────────────────────────────────────────────────────────────┐  ││
│  │   │   SQL 解析层                                                    │  ││
│  │   │   SqlSource → SqlNode → BoundSql                               │  ││
│  │   │   (原始SQL)   (动态SQL节点树)  (最终可执行SQL + 参数映射)          │  ││
│  │   └──────────────────────────────────────────────────────────────┘  ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

# 7. MyBatis 核心组件详解

```
下面按照一次 SQL 执行的顺序，介绍 MyBatis 的核心组件：

1. Configuration（全局配置对象）
   - 整个 MyBatis 的核心，存储了所有配置信息
   - 持有 Environment、MapperRegistry、TypeHandlerRegistry 等
   - 持有所有 MappedStatement（包含每条 SQL 的完整信息）
   - 源码位置：org.apache.ibatis.session.Configuration

2. SqlSessionFactory（会话工厂）
   - 负责创建 SqlSession，全局唯一
   - 由 SqlSessionFactoryBuilder 构建
   - 实现类：DefaultSqlSessionFactory
   - 源码位置：org.apache.ibatis.session.SqlSessionFactory

3. SqlSession（会话）
   - 与数据库交互的入口，非线程安全
   - 提供 CRUD 和 getMapper 方法
   - 实现类：DefaultSqlSession
   - 生命周期：方法级别（用完即关）
   - 源码位置：org.apache.ibatis.session.SqlSession

4. MapperProxy（Mapper 代理）
   - JDK 动态代理，将 Mapper 接口方法调用转换为 SQL 执行
   - 在 invoke() 中调用 MapperMethod.execute()
   - 源码位置：org.apache.ibatis.binding.MapperProxy

5. Executor（执行器）
   - 执行 SQL 的核心组件
   - 管理一级缓存（localCache）
   - 三个子类实现：SimpleExecutor / ReuseExecutor / BatchExecutor
   - CachingExecutor 作为装饰器提供二级缓存
   - 源码位置：org.apache.ibatis.executor.Executor

6. StatementHandler（语句处理器）
   - 封装 JDBC Statement 的操作
   - 负责创建 Statement、设置参数、执行 SQL
   - 实现类：RoutingStatementHandler → PreparedStatementHandler
   - 源码位置：org.apache.ibatis.executor.statement.StatementHandler

7. ParameterHandler（参数处理器）
   - 负责将 Java 参数设置到 PreparedStatement
   - 实现类：DefaultParameterHandler
   - 源码位置：org.apache.ibatis.executor.parameter.ParameterHandler

8. ResultSetHandler（结果集处理器）
   - 负责将 JDBC ResultSet 映射为 Java 对象
   - 处理嵌套映射、延迟加载等高级特性
   - 实现类：DefaultResultSetHandler
   - 源码位置：org.apache.ibatis.executor.resultset.ResultSetHandler

9. MappedStatement（映射语句）
   - 封装一条 SQL 语句的完整信息（SQL、参数映射、结果映射等）
   - 存储在 Configuration 中的 Map<String, MappedStatement>
   - 源码位置：org.apache.ibatis.mapping.MappedStatement

10. SqlSource / BoundSql（SQL 源）
    - SqlSource：表示 SQL 语句的抽象，可能是动态或静态
      - DynamicSqlSource（动态 SQL，每次执行时重新解析）
      - RawSqlSource（静态 SQL，构建时一次性解析）
    - BoundSql：持有最终可执行的 SQL（#{} 已替换为 ?）和参数映射列表
    - 源码位置：org.apache.ibatis.mapping.SqlSource
```

# 8. MyBatis 一次查询的完整执行流程

```
下面是 mapper.selectById(1) 从调用到返回结果的完整执行链路：

Step 1: 代理拦截
  carMapper.selectById(1)
    → MapperProxy.invoke()                           ← JDK 动态代理
    → 根据方法签名构建 MapperMethod

Step 2: 方法执行
  mapperMethod.execute(sqlSession, args)
    → command.getType() == SELECT
    → sqlSession.selectOne("com.xxx.CarMapper.selectById", 1)

Step 3: 会话层转发
  DefaultSqlSession.selectOne()
    → 从 Configuration 中获取 MappedStatement
    → executor.query(ms, parameter, rowBounds, resultHandler)

Step 4: 二级缓存检查（如果有 CachingExecutor）
  CachingExecutor.query()
    → 构建 BoundSql (将 #{} 替换为 ?)
    → 创建 CacheKey (用于缓存查找)
    → 查询二级缓存
      → 命中：直接返回
      → 未命中：delegate.query()  ← 委托给 BaseExecutor

Step 5: 一级缓存检查
  BaseExecutor.query()
    → 创建 CacheKey
    → 查询一级缓存 (localCache)
      → 命中：直接返回
      → 未命中：queryFromDatabase()

Step 6: 数据库查询
  BaseExecutor.queryFromDatabase()
    → 执行 doQuery()             ← 模板方法，由子类实现

Step 7: StatementHandler 创建和执行
  SimpleExecutor.doQuery()
    → configuration.newStatementHandler()  ← 创建 StatementHandler
    → prepareStatement(handler)            ← 获取 Connection，创建 PreparedStatement
    → parameterHandler.setParameters()      ← 设置 SQL 参数
    → handler.query(stmt, resultHandler)    ← 执行 stmt.execute()

Step 8: 结果映射
  PreparedStatementHandler.query()
    → ps.execute()                           ← JDBC 执行
    → resultSetHandler.handleResultSets()     ← 结果集映射

Step 9: 结果封装返回
  DefaultResultSetHandler.handleResultSets()
    → 遍历 ResultSet 中的每一行
    → 根据 ResultMap 进行自动映射
    → 处理嵌套查询、延迟加载等
    → 返回最终的 Java 对象

Step 10: 缓存写入
  → 查询结果写入一级缓存
  → 如果有 CachingExecutor，事务提交后写入二级缓存

Step 11: 返回用户
  → Car 对象返回到调用方


                        调用链总览

   carMapper.selectById(1)
        │
        ▼
   MapperProxy.invoke() ───────▶ MapperMethod.execute()
                                      │
                                      ▼
                                 DefaultSqlSession.selectOne()
                                      │
                                      ▼
                                 CachingExecutor.query()
                                      │
                                      ▼
                                 BaseExecutor.query()
                                      │
                                      ▼
                                 SimpleExecutor.doQuery()
                                      │
                                      ▼
                                 PreparedStatementHandler.query()
                                      │
                          ┌───────────┼───────────┐
                          ▼           ▼           ▼
                     ParameterHandler  │   ResultSetHandler
                     (设置 ? 参数)      │   (映射结果集)
                                       ▼
                               JDBC Statement.execute()
```

# 9. MyBatis 发展历史

```
  2010年 ─── MyBatis 3.0 发布（从 Apache iBATIS 迁移到 Google Code）
           • 主要改进：引入 Mapper 接口 + XML 映射方式取代 DAO 实现类
           • 支持注解式 SQL（@Select @Insert 等）
  
  2013年 ─── MyBatis 3.2 发布
           • 支持 Java 7 的 try-with-resources
           • 引入 SQL Builder（org.apache.ibatis.jdbc.SQL）
  
  2015年 ─── MyBatis 3.3 发布
           • 支持 @ConstructorArgs、@Arg 构造器映射
           • 支持 OGNL 表达式在动态 SQL 中使用
  
  2016年 ─── MyBatis-Spring 1.3（Spring 整合模块成熟）
           • MapperScannerConfigurer 自动扫描
  
  2017年 ─── MyBatis 3.4 发布
           • 支持 Java 8 Optional
           • 支持 default 方法（Mapper 接口中可以使用 default 方法）
  
  2019年 ─── MyBatis 3.5 发布
           • JDK 11 支持
           • 增强的注解支持
  
  2020年 ─── MyBatis-Spring 2.0（支持 Spring 5 + Java 8+）
  
  2023年 ─── MyBatis 3.5.13+ 和 MyBatis-Spring 3.0（Spring 6 + Jakarta EE）
  
  备注：
  - iBATIS 是 MyBatis 的前身，2001年由 Clinton Begin 创建
  - 2010年从 Apache 基金会迁移，更名为 MyBatis
  - MyBatis-Plus 是国人开发的增强工具（2016年~），在国内使用极广
```

# 10. MyBatis 与其他框架的关系

```
┌──────────────────────────────────────────────────────────────┐
│                      MyBatis 生态                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   MyBatis（核心框架）                                          │
│   ├── SQL 映射、动态 SQL、缓存、插件机制                        │
│   │                                                          │
│   ├── MyBatis-Spring（Spring 整合）                            │
│   │   └── SqlSessionFactoryBean / MapperScannerConfigurer     │
│   │                                                          │
│   ├── MyBatis-Spring-Boot-Starter（Spring Boot 自动配置）       │
│   │                                                          │
│   ├── MyBatis-Plus（增强工具，国内主流）                         │
│   │   └── 无侵入：只做增强不做改变                               │
│   │   └── 内置：通用 CRUD、分页、逻辑删除、乐观锁、代码生成器      │
│   │                                                          │
│   ├── MyBatis Generator（MBG，代码生成器）                      │
│   │   └── 从数据库表生成 POJO + Mapper.xml + Mapper 接口        │
│   │                                                          │
│   ├── PageHelper（分页插件，国内主流）                           │
│   │   └── 基于 MyBatis 拦截器实现，物理分页                      │
│   │                                                          │
│   ├── tk.mybatis / Mapper（通用 Mapper）                       │
│   │   └── 基于 MyBatis 的通用 CRUD 封装                        │
│   │                                                          │
│   └── MyBatis Dynamic SQL（动态 SQL 的 Java DSL 写法）         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```