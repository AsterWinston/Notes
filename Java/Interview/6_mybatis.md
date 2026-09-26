# 1. 知识点
- **MyBatis定位**：半自动ORM持久层框架，开发者手写SQL，MyBatis负责SQL与POJO之间的映射（结果集→对象、参数设置），相比Hibernate更灵活，适合复杂查询和SQL精细化控制
- **工作流程**：SqlSessionFactoryBuilder读取配置文件(mybatis-config.xml)→构建Configuration对象→创建SqlSessionFactory（全局一个）→每次操作时openSession()获取SqlSession→SqlSession委托Executor执行SQL→Executor使用StatementHandler封装JDBC操作→ParameterHandler设置SQL参数→执行JDBC→ResultSetHandler将ResultSet映射为Java对象→返回结果
- **SqlSessionFactoryBuilder**：一次性使用，读取mybatis-config.xml（或mybatis-spring的SqlSessionFactoryBean），构建Configuration和SqlSessionFactory后即可丢弃
- **Configuration**：全局配置对象，包含：数据库环境（Environment/DataSource/TransactionFactory）、Mapper注册表（MappedStatement集合）、别名注册表、插件拦截器链、类型处理器、settings等
- **SqlSessionFactory**：全局唯一，线程安全，重量级，应用生命周期内只有一个实例。用于创建SqlSession
- **SqlSession**：每次数据库操作的会话对象，非线程安全（每个请求或事务一个），提供select/insert/update/delete等方法。使用完必须关闭（try-with-resources或finally），归还连接给连接池
- **Executor**：MyBatis的核心执行器，负责SQL执行、缓存处理、事务管理。四大实现：SimpleExecutor（默认，每次执行都创建新的Statement）、ReuseExecutor（复用预编译好的PreparedStatement）、BatchExecutor（JDBC批量操作）、CachingExecutor（二级缓存装饰器，包装上述三种基础Executor）
- **StatementHandler**：封装JDBC Statement操作，负责创建Statement、设置参数、执行SQL。三大实现：SimpleStatementHandler（无参）、PreparedStatementHandler（预编译，默认使用）、CallableStatementHandler（存储过程）
- **ParameterHandler**：将Java方法参数转换为JDBC PreparedStatement的占位符参数（?占位符赋值），处理参数映射和类型转换
- **ResultSetHandler**：将JDBC的ResultSet结果集映射为Java对象（POJO/Map/基本类型），处理复杂映射（嵌套查询/嵌套结果/关联查询/集合映射）
- **MappedStatement**：封装一个完整的SQL映射信息（SQL语句、参数映射、结果映射、statement类型等），每个`<select>`/`<insert>`/`<update>`/`<delete>`标签对应一个MappedStatement对象，存储在Configuration的注册表中
- **MapperProxy动态代理**：MyBatis通过JDK动态代理为Mapper接口生成代理对象。调用Mapper接口方法时，代理对象(MapperProxy)根据接口全限定名+方法名从Configuration中找到对应的MappedStatement，然后委托SqlSession执行
- **XML映射器**：在mapper.xml中定义SQL和结果映射。核心元素：`<select>`（查询）、`<insert>`（插入）、`<update>`（更新）、`<delete>`（删除）、`<resultMap>`（自定义结果映射）、`<sql>`（可复用SQL片段）、`<cache>`（二级缓存配置）
- **注解映射**：直接在Mapper接口方法上使用`@Select/@Insert/@Update/@Delete`注解写SQL。优点是代码集中、简洁；缺点是复杂SQL难以阅读、不支持动态SQL（需用`@SelectProvider`配合SQL Builder）、SQL与代码耦合
- **resultMap**：当数据库字段名（如user_name）与POJO属性名（如userName）不一致时，通过resultMap定义映射关系。支持：`<id>`主键映射、`<result>`普通字段映射、`<association>`一对一关联映射（如User包含一个Department对象）、`<collection>`一对多关联映射（如Department包含List<User>列表）
- **association关联映射**：一对一关系，两种实现方式：嵌套查询（select属性指定另一个查询id，会执行额外SQL，N+1问题）和嵌套结果映射（join一条SQL查出，在resultMap中按层级映射，推荐方式）
- **collection集合映射**：一对多关系，同样有嵌套查询和嵌套结果两种方式。嵌套结果通过columnPrefix或id去重处理
- **动态SQL**：根据条件动态拼接SQL，MyBatis的核心优势之一。标签：`<if test="">`（条件判断）、`<choose><when><otherwise>`（多分支选择，类似switch-case）、`<trim prefix="" suffix="" prefixOverrides="" suffixOverrides="">`（前缀后缀处理）、`<where>`（自动处理WHERE和第一个AND/OR）、`<set>`（自动处理UPDATE SET中的逗号）、`<foreach collection="" item="" separator="" open="" close="">`（遍历集合批量操作）
- **foreach批量操作**：`<foreach collection="list" item="item" separator=","> (#{item}) </foreach>`，collection指定集合（list/array/@Param命名的参数），separator分隔符，open/close包裹符号；常用于批量插入、批量查询（IN条件）等
- **一级缓存（本地缓存）**：Session级别，默认开启且不可关闭。同一个SqlSession内执行相同SQL和参数，第二次从缓存中取结果，不查询数据库。缓存失效条件：执行增删改操作、手动clearCache()、SqlSession关闭。缺点是多个SqlSession不共享缓存
- **二级缓存**：Mapper级别（namespace级别），一个namespace下的所有SqlSession共享。需要显式开启（mapper.xml中加`<cache/>`标签，POJO需实现Serializable）。可用第三方缓存实现（Ehcache、Redis、Caffeine等实现MyBatis的Cache接口）。注意：查询结果存入缓存、增删改会清空相关namespace缓存
- **二级缓存脏数据问题**：多表关联查询时，A namespace的SQL关联了B表，B表被B namespace的增删改修改了，但A namespace的缓存无法感知B表变化，导致脏数据。解决方案：1) 使用`<cache-ref namespace=""/>`引用其他namespace缓存使缓存共享；2) 不使用二级缓存，业务层用Redis统一管理
- **MyBatis缓存使用建议**：一级缓存默认使用但作用有限；二级缓存生产环境**不推荐使用**（脏数据问题、分布式环境不能同步），推荐将缓存功能交给业务层用Redis等外部缓存统一管理
- **插件机制（Interceptor）**：MyBatis提供的四大拦截器接口，通过动态代理实现：Executor（拦截SQL执行全流程，如分页、读写分离）、ParameterHandler（拦截参数设置，如加密）、StatementHandler（拦截预编译和SQL设置）、ResultSetHandler（拦截结果映射，如脱敏）。需使用@Intercepts和@Signature注解指定拦截目标
- **插件执行链**：多个插件按配置顺序包装目标对象（责任链+动态代理），形成多层代理。每个插件的intercept()接收到Invocation后处理，调用proceed()传递给下一个
- **PageHelper分页原理**：基于MyBatis插件机制，在Executor.query()方法前通过ThreadLocal存储分页参数（pageNum, pageSize），拦截prepare方法对原始SQL进行包装，拼接`LIMIT offset, limit`（MySQL方言）或`ROWNUM`（Oracle方言）等，执行后清理ThreadLocal。支持各种数据库方言自动适配
- **PageHelper关键细节**：1) 通过ThreadLocal传递分页参数，线程隔离；2) 拦截Executor的query方法；3) 在SQL被预编译前修改SQL语句；4) 分页后自动清理ThreadLocal避免内存泄漏；5) 支持count查询（自动生成COUNT SQL）；6) 注意：PageHelper.startPage()后必须紧跟一个Mapper查询
- **#{}和${}的区别**：#{}是预编译占位符，用?代替参数，再通过PreparedStatement.setXXX()设置值，SQL注入安全；${}是字符串直接替换（拼接SQL），不防SQL注入，但可用于动态表名/列名/ORDER BY/LIKE模式等
- **#{}防SQL注入原理**：底层使用`PreparedStatement`，参数位置用?占位符，数据库在SQL编译阶段确定SQL结构（语义），参数值作为数据传入不会改变SQL结构，恶意输入不会被执行为SQL命令
- **${}安全处理**：必须使用${}的场景（动态表名、动态排序列名），要对参数做白名单校验，例如校验orderBy参数必须在允许的列名列表中才拼接
- **延迟加载（懒加载）**：当访问关联对象属性时才真正发起SQL查询加载数据。在association和collection中配置`fetchType="lazy"`。MyBatis通过CGLIB/Javassist生成代理对象，当getter被首次调用时触发实际SQL查询
- **aggressiveLazyLoading**：MyBatis 3.4.1+默认为false（非激进懒加载），false表示按需加载（只加载被访问的属性）；true（激进模式，旧版本默认）表示只要访问对象的任意属性就会加载所有懒加载属性
- **延迟加载实现原理**：使用Javassist（默认，3.4+）或CGLIB创建目标对象的代理，代理拦截getter方法调用。当getter被调用时检查是否已加载，未加载则执行SQL查询填充值，然后返回
- **Spring Boot整合MyBatis**：引入mybatis-spring-boot-starter → 自动配置MybatisAutoConfiguration → 创建SqlSessionFactoryBean（读取DataSource、指定mapper位置、typeAliasesPackage等）→ 创建SqlSessionTemplate → 通过@MapperScan或@Mapper扫描Mapper接口注册为Bean
- **@Mapper和@MapperScan**：@Mapper加在单个Mapper接口上让Spring识别；@MapperScan配置在启动类上统一扫描指定包路径，如`@MapperScan("com.example.mapper")`，更推荐后者
- **批量操作**：使用BatchExecutor，通过`SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH)`获取批量模式的SqlSession，多次调用insert/update/delete后统一执行`sqlSession.flushStatements()`批量提交（JDBC addBatch+executeBatch），注意每次批量后`sqlSession.clearCache()`防止OOM
- **批量操作注意事项**：Batch模式下每次操作会维护日志（Statement列表）不立即提交，适合大量同类操作一次性提交；注意控制每次批量大小（如每1000条flush一次+clearCache）防止内存溢出；Spring整合时使用SqlSessionTemplate需每个批量操作获取新的SqlSession
- **MyBatis Generator(MBG)**：官方代码生成器，根据数据库表生成实体类、Mapper接口和XML映射文件。支持CRUD基础操作、Example查询类（条件查询）、主键策略、注释配置。XML/Java代码/命令行多种配置运行方式
- **tk.mybatis（通用Mapper）**：第三方增强，只需继承通用Mapper接口即可获得全部单表增删改查方法，无需写XML，支持Example查询和分页。Spring Boot整合：`tk.mybatis.spring.annotation.MapperScan`替代`org.mybatis.spring.annotation.MapperScan`

# 2. 面试题

## 2.1 MyBatis的工作流程是什么？核心组件有哪些？
**工作流程**：
1. 通过SqlSessionFactoryBuilder读取mybatis-config.xml构建Configuration（包含数据源、事务管理器、映射注册表等）
2. 创建SqlSessionFactory（全局唯一）→ 每次操作调用openSession()获取SqlSession
3. SqlSession通过MapperProxy（JDK动态代理）找到对应的MappedStatement
4. 委托Executor执行：先从二级缓存查 → 到一级缓存查 → 最后通过StatementHandler执行JDBC
5. StatementHandler创建PreparedStatement → ParameterHandler设置参数 → 执行SQL
6. ResultSetHandler将ResultSet映射为Java对象 → 返回结果
7. 提交事务，关闭SqlSession

**核心组件**：
- Configuration：全局配置，存放所有MappedStatement和设置
- SqlSessionFactoryBuilder：构建器，一次性
- SqlSessionFactory：工厂，全局唯一
- SqlSession：会话，非线程安全
- Executor：执行器（Simple/Reuse/Batch/Caching）
- StatementHandler：JDBC Statement封装
- ParameterHandler：参数设置
- ResultSetHandler：结果映射
- MapperProxy：Mapper接口的动态代理

## 2.2 MyBatis中#{}和${}的区别是什么？为什么#{}能防止SQL注入？
| 对比维度 | #{} | ${} |
|---------|-----|-----|
| 处理方式 | 预编译占位符，用?替代参数 | 字符串直接替换拼接 |
| 底层实现 | PreparedStatement.setXXX() | Statement直接拼接 |
| SQL注入 | 安全，参数作为数据处理 | 不安全，参数作为SQL的一部分 |
| 使用场景 | 绝大多数字段值、WHERE条件值 | 动态表名、列名、ORDER BY、LIKE "%"{}"%" |
| 类型转换 | 自动进行JDBC类型转换 | 原样输出，需注意引号 |

**#{}防注入原理**：数据库的预编译过程分两步：1) 解析SQL结构并编译（此时SQL语义已定）；2) 传入参数值执行。由于参数是在SQL编译后才传入，参数中的恶意SQL语句不会被解析为SQL命令，只当作普通字符串数据。例如：`password = '1' OR '1'='1'` 被当作字符串值比较，不会改变WHERE子句的逻辑结构。

## 2.3 MyBatis的一级缓存和二级缓存有什么区别？有哪些问题？
| 对比 | 一级缓存（Local Cache） | 二级缓存 |
|------|---------------------|---------|
| 级别 | SqlSession级别 | Mapper(namespace)级别 |
| 作用范围 | 同一SqlSession内 | 同一namespace下所有SqlSession共享 |
| 默认状态 | 默认开启，不能关闭 | 默认关闭，需`<cache/>`启用 |
| 存储位置 | MyBatis内部HashMap | 可更换存储（默认HashMap，支持Redis/Ehcache/Caffeine等） |
| 清除条件 | update/insert/delete/commit/close | update/insert/delete在对应namespace触发清除 |
| 生命周期 | 随SqlSession创建/关闭 | 随应用生命周期 |
| 序列化要求 | 不需要 | POJO必须实现Serializable |

**问题**：
- 一级缓存：不同SqlSession之间数据不一致；可能读到脏数据（另一个SqlSession修改了数据但一级缓存读旧值）
- 二级缓存：跨namespace关联查询时数据脏读（A namespace查询关联B表，B表变更后A的缓存不感知）；分布式环境多节点缓存不一致

在生产环境，**不建议使用MyBatis二级缓存**，推荐使用Redis等外部缓存由业务层统一管理。

## 2.4 MyBatis如何进行批量操作？有哪几种方式？
**方式一：BatchExecutor**
```java
SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
try {
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);
    for (User user : userList) {
        mapper.insert(user);
    }
    sqlSession.flushStatements(); // 统一批量提交
    sqlSession.commit();
} finally {
    sqlSession.close();
}
```
BatchExecutor将多次操作积累在Statement列表中，flushStatements时通过JDBC的addBatch()+executeBatch()批量执行。注意每处理一定量（如1000条）后flush+clearCache防止OOM。

**方式二：动态SQL foreach**
```xml
<insert id="batchInsert">
    INSERT INTO user (name, age) VALUES
    <foreach collection="list" item="user" separator=",">
        (#{user.name}, #{user.age})
    </foreach>
</insert>
```
将多条数据拼接为`INSERT INTO ... VALUES (...), (...), (...)`一次执行。MySQL默认有max_allowed_packet限制（4MB），数据量太大需分批处理。

**方式三（Spring整合）**：注入SqlSessionTemplate，每次批量获取新SqlSession。使用mybatis-plus的ServiceImpl自带saveBatch()方法简化。

## 2.5 MyBatis插件机制的实现原理是什么？PageHelper是如何工作的？
**插件机制**：MyBatis插件基于**JDK动态代理+责任链模式**实现。四大对象（Executor、ParameterHandler、StatementHandler、ResultSetHandler）在创建时会被pluginAll()方法包装：
1. 遍历配置的Interceptor插件列表
2. 对每个插件调用`plugin(target)`方法
3. 插件内部用`Plugin.wrap(target, this)`创建代理，只拦截@Signature注解指定接口上的方法
4. 多个插件形成多层代理链（层层包装）

**PageHelper原理**：
1. 调用`PageHelper.startPage(pageNum, pageSize)` → 将分页参数存入ThreadLocal（Page对象）
2. PageHelper实现了Executor的拦截器，拦截Executor.query(MappedStatement, parameter, RowBounds, ResultHandler)方法
3. 在执行query之前从ThreadLocal获取Page参数
4. 判断是否需要分页，是则修改原始SQL（通过MyBatis的BoundSql获取原始SQL，拼接方言如MySQL的`LIMIT ?, ?`）
5. 执行分页SQL获取数据，同时构造COUNT SQL获取总数
6. finally中清理ThreadLocal避免内存泄漏

核心点：拦截在SQL执行前修改SQL，利用ThreadLocal实现线程隔离传递参数。

## 2.6 resultMap和resultType有什么区别？什么时候使用resultMap？
- **resultType**：直接指定返回类型的全限定名或别名。适用于数据库字段名与POJO属性名一致（自动映射），或字段名一致但类型兼容的场景。简单快捷
- **resultMap**：自定义映射规则。适用于：
  - 数据库字段名与POJO属性名不一致（如user_name → userName）
  - 一对一关联查询（association），如查询User并包含其所Department
  - 一对多关联查询（collection），如查询Department并包含其下所有User列表
  - 复杂嵌套映射（多层级对象关系）
  - 字段类型需要特殊转换（如数据库中存int表示枚举值）

resultMap的autoMapping="true"时仍会自动映射同名字段，只需手动配置不一致字段即可。

## 2.7 MyBatis中的association和collection有什么区别？
- **association**：一对一关联（或引用单个对象）。如一个User对象中包含一个Department department属性。映射时使用`<association property="department" javaType="Department">`
- **collection**：一对多关联（集合对象）。如一个Department对象中包含`List<User> users`属性。映射时使用`<collection property="users" ofType="User">`

两者都支持两种加载方式：
1. **嵌套查询**（select属性）：执行额外的一次或多次SQL查询。N+1问题（主查询1次+N个关联查询），性能较差
2. **嵌套结果映射**（join查询结果直接映射）：一条SQL用JOIN查出所有数据，在resultMap中按层级关系映射。推荐方式，但复杂join可能影响性能需要权衡

## 2.8 MyBatis延迟加载（懒加载）的原理是什么？
**原理**：使用Javassist（默认）或CGLIB创建目标对象的代理对象。当访问代理对象的getter方法时：
1. 拦截getter调用
2. 检查该属性是否已加载（有结果值则直接返回）
3. 若未加载，触发原始关联查询（从数据库加载关联数据）
4. 将查询结果设置到目标对象中
5. 返回结果值

**配置**：
- mybatis-config.xml中：`<setting name="lazyLoadingEnabled" value="true"/>`（启用延迟加载）
- `<setting name="aggressiveLazyLoading" value="false"/>`（按需加载，3.4.1+默认false）
- 在association/collection中：`fetchType="lazy"`（局部控制，覆盖全局设置）

延迟加载要求：SqlSession在数据加载时必须存活（否则代理触发查询时已无可用Session）。Spring整合时需配置事务确保SqlSession生命周期覆盖到视图层。

## 2.9 MyBatis的动态SQL有哪些标签？trim标签的作用是什么？
动态SQL标签：
- **`<if test="">`**：条件判断，满足条件才拼接SQL片段
- **`<choose><when><otherwise>`**：多分支选择，类似Java的if-else if-else
- **`<where>`**：包裹WHERE条件，自动处理第一个AND/OR（移除前缀多余的AND/OR）
- **`<set>`**：包裹UPDATE的SET部分，自动处理尾部多余的逗号
- **`<trim>`**：万能裁剪标签，比where/set更灵活：
  - prefix：在内容最前面添加的前缀
  - suffix：在内容最后面添加的后缀
  - prefixOverrides：去掉内容最前面的指定字符串
  - suffixOverrides：去掉内容最后面的指定字符串
- **`<foreach>`**：遍历集合，批量操作和IN查询
- **`<bind>`**：创建新变量用于SQL中，如模糊搜索`<bind name="pattern" value="'%' + keyword + '%'" />`
- **`<sql>` + `<include>`**：定义可复用的SQL片段，减少重复代码

`<trim>`是where和set的底层实现，where本质是`<trim prefix="WHERE" prefixOverrides="AND |OR ">`，set本质是`<trim prefix="SET" suffixOverrides=",">`。

## 2.10 MyBatis的SqlSession是线程安全的吗？为什么？
SqlSession**不是线程安全的**。原因：
- SqlSession内部持有数据库连接（Connection），而Connection不是线程安全的
- SqlSession的一级缓存（ConcurrentHashMap存储查询结果）没有同步控制，多线程同时操作会导致数据混乱
- SqlSession的Executor实例也不是线程安全设计的（有执行状态）

最佳实践：
- 每个请求或每个事务创建一个SqlSession，使用完立即关闭
- 不与Servlet的实例变量绑定（会被多线程共享）
- Spring整合后使用SqlSessionTemplate（线程安全，内部通过ThreadLocal为每个线程维护独立的SqlSession）
- 在方法内使用try-with-resources确保关闭

## 2.11 MyBatis如何进行一对一和一对多关联查询？
**一对一（association）**：
```xml
<resultMap id="userResultMap" type="User">
    <id property="id" column="id"/>
    <result property="name" column="name"/>
    <!-- 嵌套结果映射（推荐，一条SQL） -->
    <association property="department" javaType="Department">
        <id property="deptId" column="dept_id"/>
        <result property="deptName" column="dept_name"/>
    </association>
</resultMap>

<select id="selectUserWithDept" resultMap="userResultMap">
    SELECT u.*, d.id as dept_id, d.name as dept_name
    FROM user u LEFT JOIN department d ON u.dept_id = d.id
</select>
```

**一对多（collection）**：
```xml
<resultMap id="deptResultMap" type="Department">
    <id property="id" column="id"/>
    <result property="name" column="name"/>
    <!-- 嵌套结果映射 -->
    <collection property="users" ofType="User">
        <id property="id" column="user_id"/>
        <result property="name" column="user_name"/>
    </collection>
</resultMap>

<select id="selectDeptWithUsers" resultMap="deptResultMap">
    SELECT d.id, d.name, u.id as user_id, u.name as user_name
    FROM department d LEFT JOIN user u ON d.id = u.dept_id
</select>
```

注意：collection中必须用`ofType`指定集合元素的类型。嵌套结果映射时，MyBatis会根据`<id>`标签自动去重（根据主键合并重复行）。

## 2.12 @Mapper和@MapperScan有什么区别？
- **@Mapper**：标注在单个Mapper接口上，告诉Spring该接口需要生成代理Bean。每个Mapper接口都需要加这个注解，比较繁琐
- **@MapperScan("com.example.mapper")**：标注在启动类或配置类上，扫描指定包路径下所有接口，自动为它们生成代理Bean。一次配置，全包生效，推荐使用

注意：如果是tk.mybatis（通用Mapper），需要使用`tk.mybatis.spring.annotation.MapperScan`；如果是MyBatis-Plus，使用`com.baomidou.mybatisplus.annotation.MapperScan`，它们与标准@MapperScan不通用。

## 2.13 MyBatis Generator(MBG)和通用Mapper(tk.mapper)有什么区别？
- **MBG（MyBatis Generator）**：代码生成器，根据数据库表**生成**实体类、Mapper接口、XML映射文件的代码（一次生成，后续手动维护）。生成的文件是完整的，包含Example查询类。适合项目初期生成基础代码，后续手动修改
- **tk.mapper（通用Mapper）**：运行时增强框架，提供通用Mapper接口（如SelectMapper, InsertMapper, UpdateMapper, DeleteMapper），Mapper接口直接继承即可获得**全部单表CRUD方法**，不需要写XML和接口方法。支持Example查询、分页、TypeHandler等。适用于单表操作场景，减少大量重复代码

实际项目可以两者结合：MBG生成初始代码，其中Mapper接口改为继承通用Mapper获得增删改查能力，特定复杂查询手动写。

## 2.14 MyBatis执行insert后如何获取自增主键？
两种方式：
1. **useGeneratedKeys**：配置`<insert useGeneratedKeys="true" keyProperty="id" keyColumn="id">`，自动将数据库生成的自增ID回填到插入对象的指定属性中。底层使用`PreparedStatement.getGeneratedKeys()`
2. **selectKey**：在insert内使用`<selectKey keyProperty="id" resultType="long" order="AFTER">SELECT LAST_INSERT_ID()</selectKey>`，专门执行一条SQL获取最后插入的主键值并设置到对象属性中

order参数：AFTER用于自增主键（先插入再获取），BEFORE用于非自增主键（如UUID，先生成主键再插入）。

注意区分：useGeneratedKeys适合支持JDBC getGeneratedKeys()的数据库（MySQL/SQL Server）；selectKey适用于Oracle等不支持getGeneratedKeys的数据库（用序列sequence获取）。

## 2.15 Spring Boot如何整合MyBatis？自动配置做了什么？
**用法**：
1. 引入依赖`mybatis-spring-boot-starter` + MySQL驱动
2. `application.yml`配置DataSource（spring.datasource.*）
3. 配置mybatis.*（mapper-locations/mapper扫描包/别名包/驼峰命名等）
4. 启动类加@MapperScan("com.xxx.mapper")

**自动配置过程**（MybatisAutoConfiguration）：
1. 检测到classpath中有SqlSessionFactory和DataSource就自动配置
2. 创建SqlSessionFactoryBean → 注入DataSource → 设置mapper位置/别名包等 → 创建SqlSessionFactory
3. 创建SqlSessionTemplate（线程安全的SqlSession实现，包装了SqlSessionFactory）
4. 通过MapperScannerConfigurer扫描@MapperScan指定的包，为每个接口生成MapperProxy代理对象并注册为Spring Bean

核心原理：`mybatis-spring-boot-autoconfigure`模块中的`spring.factories`文件包含了自动配置类的注册信息。

## 2.16 从数据库查出一条数据，MyBatis是如何将ResultSet映射为Java对象的？
**ResultSetHandler处理流程**：
1. 根据MappedStatement获取对应的ResultMap（用户定义或自动构造）
2. 遍历ResultSet的每一行数据
3. 对于每一行，根据ResultMap中的映射关系（列名→属性名）：
   - 类型转换：通过TypeHandler将JDBC类型转换为Java类型（VARCHAR→String、INTEGER→Integer等）
   - 若有嵌套ResultMap（association/collection），递归处理子映射
   - 对当前对象调用setter方法或直接通过反射设置属性值
4. 对于collection映射：根据`<id>`标签定义的唯一标识进行分组，将多行数据中相同主键的行合并为一个对象，子属性和集合分别填充
5. 返回最终的Java对象集合

如果开启了自动映射(autoMapping)，同名字段自动映射；不同名字段通过ResultMap显式指定column到property的对应关系。

## 2.17 MyBatis Executor 有哪三种类型？各自的区别和使用场景？
| 类型 | 特点 | 适用场景 |
|------|------|---------|
| **SimpleExecutor**（默认） | 每次执行 update/query 都创建新的 Statement 对象，用完即关 | 一般场景，未配置时默认使用 |
| **ReuseExecutor** | 执行 update/query 时，复用已存在的 Statement（缓存到 Map 中，以 SQL 为 key） | 同一 SQL 频繁执行的场景（减少 Statement 编译开销） |
| **BatchExecutor** | 连续调用相同 SQL 且 MappedStatement 相同时，通过 `statement.addBatch()` 批量提交。必须手动执行 `executor.flushStatements()` 或调用 commit() 才会发出 SQL | 批量操作（连续执行相同结构的 INSERT/UPDATE/DELETE） |

**配置方式**：
```yaml
mybatis:
  configuration:
    default-executor-type: REUSE   # SIMPLE / REUSE / BATCH
```
或者通过 `SqlSessionFactory.openSession(ExecutorType.BATCH)` 每次创建特定类型的 SqlSession。

**BatchExecutor 注意事项**：
- 只在 MappedStatement 相同时自动加入 batch（相同 SQL 模板 + 相同返回类型）
- 与事务绑定：commit/rollback 时才会 flush
- 当 sequence 序列或 ID 冲突时可能导致批量失效
- select 操作不会进入 batch，单独执行

## 2.18 如何自定义 TypeHandler？典型使用场景有哪些？
TypeHandler 用于在 PreparedStatement 设置参数时和从 ResultSet 获取值时进行**Java 类型与 JDBC 类型的相互转换**。

**实现步骤**：
1. 实现 `TypeHandler<T>` 接口（或继承 `BaseTypeHandler<T>` 更方便）。
2. 重写四个方法：`setNonNullParameter()`、`getNullableResult(ResultSet)`、`getNullableResult(ResultSet, columnName)`、`getNullableResult(CallableStatement, index)`。
3. 在配置中注册。

**示例——LocalDateTime 到 VARCHAR 转换**：
```java
public class LocalDateTimeTypeHandler extends BaseTypeHandler<LocalDateTime> {
    private static final DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, LocalDateTime parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, fmt.format(parameter));
    }

    @Override
    public LocalDateTime getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String s = rs.getString(columnName);
        return s == null ? null : LocalDateTime.parse(s, fmt);
    }

    @Override
    public LocalDateTime getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        String s = rs.getString(columnIndex);
        return s == null ? null : LocalDateTime.parse(s, fmt);
    }

    @Override
    public LocalDateTime getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        String s = cs.getString(columnIndex);
        return s == null ? null : LocalDateTime.parse(s, fmt);
    }
}
```

**注册 TypeHandler**：
```yaml
mybatis:
  type-handlers-package: com.example.handler
```
或在 `@TableField(typeHandler = LocalDateTimeTypeHandler.class)` / `@MappedTypes/@MappedJdbcTypes` 注解。

**典型场景**：
- 枚举与数据库 int/String 互转（MyBatis Plus 已封装）
- LocalDateTime / LocalDate 与 VARCHAR 互转
- JSON 字符串与 Java 对象/List/Map 互转（Jackson/Gson TypeHandler）
- 手机号/身份证脱敏存储（写入时加密，读取时解密）
- 性别/状态码等枚举值的自动转换

## 2.19 MyBatis 一级缓存脏读问题及解决方案
**一级缓存**：SqlSession 级别，默认开启。同一个 SqlSession 内对同一 MappedStatement + 参数执行的相同查询，第二次直接返回缓存结果（不走数据库）。

**脏读场景**：
- SqlSession1 执行 `SELECT * FROM user WHERE id=1` → 结果缓存。
- 在这之后，SqlSession2 执行 `UPDATE user SET name='new' WHERE id=1` 并提交。
- SqlSession1 再次执行 `SELECT * FROM user WHERE id=1` → **返回旧值（缓存数据）**，而不是 SqlSession2 写入的新值。

**解决方案**：
1. **及时关闭 SqlSession**：每次操作后关闭 SqlSession（MyBatis 和 Spring 整合后 `@Transactional` 方法结束自动关闭）。最常用的方案。
2. **手动清空缓存**：`sqlSession.clearCache()` 清除当前 SqlSession 的一级缓存。
3. **关闭一级缓存**：`mybatis.configuration.local-cache-scope=STATEMENT`（5.1+），每条 SQL 后自动清空一级缓存。

**二级缓存的脏读**：类似原理但变化更大——同一 Namespace 多个 SqlSession 同时读写时，数据不一致。解决方案：混用 MyBatis 和数据库外部更新时不建议使用二级缓存；推荐使用 Redis 作为缓存层的 Cache-Aside 模式。
