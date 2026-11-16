本篇覆盖 MyBatis 持久层框架的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。三个主题块：总览与核心组件、SQL 映射与动态 SQL、缓存/插件与生态整合。MyBatis-Plus 的增强特性见 9_mybatis_plus.md。

# 1. 总览与核心组件

## 1.1 框架认知

### 1.1.1 MyBatis 是什么？有哪些优缺点？

**结论**：MyBatis 是**半自动化的 ORM 持久层框架**——SQL 由开发者手写，框架负责"SQL 与 Java 对象之间"的双向映射：入参把 POJO/Map 填充到预编译占位符，出参把 ResultSet 映射回 POJO。它封装了 JDBC 的样板代码（加载驱动、拿连接、拼参数、设参数、处理结果集、关资源、异常处理），但**不帮你生成 SQL**——这与 Hibernate 的"全自动 SQL 生成"相对。

**优点**：

1. SQL 完全掌控——复杂查询、多表连接、执行计划调优都在自己手里，适合对 SQL 精细化要求高的互联网业务；
2. SQL 与代码解耦（XML/注解），改 SQL 不用重编译 Java；
3. 映射灵活：resultMap、动态 SQL、TypeHandler 覆盖各种脏场景；
4. 轻量、学习曲线平缓，与 Spring 整合成熟；
5. 缓存体系（一级/二级）与插件机制可扩展。

**缺点**：

1. 需要手写大量 SQL/映射配置，开发效率低于全自动 ORM；
2. 数据库移植性差——SQL 方言写死在映射文件里（换库要改）；
3. 相比 JDBC 原生性能提升有限，框架自身有反射/解析开销（量级很小，通常忽略）。

**与原生 JDBC 对比**：JDBC 的"硬编码 SQL、手动 setParameter、手动 ResultSet 解析、频繁开关连接"全部由 MyBatis 代办，SQL 还能进 XML 与配置化管理。

### 1.1.2 MyBatis 和 Hibernate 的区别？如何选择？

| 维度 | MyBatis | Hibernate/JPA |
|------|---------|---------------|
| 定位 | 半自动 ORM，SQL 自己写 | 全自动 ORM，面向对象编程 |
| SQL 掌控 | 手写，精细调优容易 | 框架生成，复杂 SQL 需 HQL/NativeQuery |
| 学习成本 | 低（会 SQL 就行） | 高（缓存、懒加载、脏检查体系复杂） |
| 移植性 | 差（方言写死） | 好（dialect 适配） |
| 开发效率 | 单表 CRUD 要写 SQL；配 MBG/通用 Mapper/MP 可追平 | 领域模型建好后 CRUD 极快 |
| 适用 | 复杂查询多、SQL 优化诉求强、迭代快的互联网业务 | 模型稳定、以增删改为主的企业级管理系统 |

**行业实践**：国内互联网以 MyBatis（及 MyBatis-Plus）为绝对主流——SQL 可控、DBA 可审查、方便做分库分表与慢 SQL 治理；Hibernate 多见于欧美企业软件。选型本质是"SQL 优先还是对象优先"的工程权衡。

### 1.1.3 MyBatis 的工作流程？核心组件各司其职是什么？

**结论**：一次查询的完整链路——

1. **SqlSessionFactoryBuilder** 读取 `mybatis-config.xml`（或 Spring 的 `SqlSessionFactoryBean`）构建全局 **Configuration**（环境、数据源、事务工厂、MappedStatement 注册表、别名、拦截器链、TypeHandler 注册表），拿到 **SqlSessionFactory** 后 Builder 即可丢弃；
2. **SqlSessionFactory**（全局单例）`openSession()` 创建 **SqlSession**——面向应用的 API 门面；
3. `session.getMapper(UserMapper.class)` 返回 **MapperProxy** 动态代理，按"接口全限定名 + 方法名"到 Configuration 查 **MappedStatement**；
4. SqlSession 委托 **Executor**（先查二级缓存 → 再查一级缓存 → 未命中才落库）；
5. **StatementHandler** 创建/复用 Statement → **ParameterHandler** 给 `?` 设参 → 执行 SQL；
6. **ResultSetHandler** 按 ResultMap + TypeHandler 把 ResultSet 映射为 Java 对象 → 写一级缓存并返回。

**组件速记**（四大对象是骨架）：

| 组件 | 职责 | 线程安全/生命周期 |
|------|------|------------------|
| SqlSessionFactoryBuilder | 构建工厂 | 一次性，用完即弃 |
| SqlSessionFactory | 创建 SqlSession | 全局唯一、线程安全、重量级 |
| SqlSession | 会话 API 门面 | **非线程安全**，每请求/事务一个，用完必关 |
| Configuration | 全局配置中枢 | 随工厂单例 |
| MappedStatement | 一条 SQL 的静态描述 | 注册在 Configuration |
| Executor | SQL 执行 + 缓存 + 事务 | 随 SqlSession |
| StatementHandler | 封装 JDBC Statement | 每次执行创建 |
| ParameterHandler | 入参映射 | 每次执行创建 |
| ResultSetHandler | 出参映射 | 每次执行创建 |
| MapperProxy | 接口 → SQL 的代理 | 随 Mapper Bean |

> **面试追问**：配置一个 Mapper 插件要动几个对象？——见 3.2.1，四大对象都可被拦截。

## 1.2 会话与执行机制

### 1.2.1 SqlSessionFactoryBuilder、SqlSessionFactory、SqlSession 的生命周期区别？

**结论**：Builder 方法级、Factory 应用级、Session 请求级——按"重量级复用、轻量级短命"设计。

- **SqlSessionFactoryBuilder**：只在启动时用一次，`build()` 后使命完成，作用域最好是方法内的局部变量；
- **SqlSessionFactory**：一旦创建就应存在整个应用生命周期（相当于连接池的配置），最佳实践是单例（Spring 整合后由容器管理）；创建成本很高（解析所有映射文件），绝不该每次操作都建；
- **SqlSession**：包含连接、事务、一级缓存等**有状态**内容，每次请求/事务新建，用完必须 close（try-with-resources 或 finally），否则连接不还池会耗尽连接池。

**易错**：SqlSession 打开后不关闭是 MyBatis 新手第一坑；Spring 的 `SqlSessionTemplate` 已经代理了开关逻辑（见 3.3.1），不要再手工管理。

### 1.2.2 SqlSession 是线程安全的吗？Spring 整合后呢？

**结论**：原生 SqlSession **非线程安全**，多线程共享会导致状态混乱。

原因：内部持有 Connection（Connection 本身非线程安全）、Executor 有执行状态、一级缓存的读写无并发保护。

**Spring 整合后为什么能注入成共享 Bean？**——注入的是 **SqlSessionTemplate**：

1. 它本身线程安全（final 方法 + 代理），每次方法调用时经 `SqlSessionUtils` 从 **TransactionSynchronizationManager（ThreadLocal）** 取当前事务绑定的 SqlSession，没有事务就新建并注册；
2. 从而做到"**同线程同事务复用同一个 SqlSession，跨线程互不干扰**"，一级缓存在事务范围内也顺带生效。

**最佳实践**：每个请求/事务一个会话；不要与 Servlet 实例变量绑定；关闭交给框架，手工场景用 try-with-resources。

### 1.2.3 Executor 有哪几种类型？区别和场景？

**结论**：三种基础执行器 + 一个缓存装饰器。

| 类型 | 行为 | 场景 |
|------|------|------|
| **SimpleExecutor**（默认） | 每次执行都 `createNewStatement`，用完即关 | 通用 |
| **ReuseExecutor** | 以 SQL 为 key 缓存 PreparedStatement 复用 | 同 SQL 高频执行（省预编译开销） |
| **BatchExecutor** | addBatch() 累积语句，flushStatements()/commit() 才真正发出 | 批量插入/更新 |
| **CachingExecutor** | 装饰器，包住上面三种，在它们之上加二级缓存查询逻辑 | 开启二级缓存时 |

**BatchExecutor 细节**：只有 **MappedStatement 相同**（同 SQL 模板、同类型操作）才归入同一 batch；SELECT 不进 batch 立即执行；未 flush 前数据只在 Statement 队列里，中途报错整批失败；务必分批 flush + `clearCache()`（见 3.2.4）。

配置方式：`defaultExecutorType=BATCH` 全局，或 `openSession(ExecutorType.BATCH)` 单次。

> **易错**：Batch 模式下 `insert` 返回的影响行数不可信（`-2147482646`，BATCH_UPDATE_RETURN_VALUE，因为还没真正执行）。

### 1.2.4 什么是 MappedStatement？Mapper 接口是怎么和 SQL 绑定的？

**MappedStatement**：Configuration 注册表里"一条 SQL 的完整静态描述"——SQL 文本（含动态 SQL 解析入口）、参数映射（SqlCommandType、ParameterMap）、结果映射（ResultMap 列表）、语句类型（PREPARED/STATEMENT）、超时、缓存引用等。每个 `<select>/<insert>/<update>/<delete>` 标签、每个 `@Select` 注解方法，解析后都是一个 MappedStatement，id 为 `namespace.方法名`。

**接口绑定 SQL 的两条路**：

1. **XML 绑定（主流）**：`mapper.xml` 的 `namespace = 接口全限定名`，`<select id> = 方法名`，方法签名与 parameterType/resultType 匹配——**重载方法走不通**（id 无法区分），这是"MyBatis 接口不能重载同名方法"的根因；
2. **注解绑定**：`@Select/@Insert/@Update/@Delete` 在方法上直接写 SQL，启动时解析注册成 MappedStatement。

**追问**：XML 和注解同时存在？——启动时注解先解析、XML 后加载，冲突会报 `IllegalArgumentException: ... mappedStatement is already` 直接失败，不是"谁覆盖谁"。

### 1.2.5 Mapper 接口没有实现类，为什么能调用？动态代理原理？

**结论**：MyBatis 用 **JDK 动态代理**为每个 Mapper 接口生成代理实例，把"接口方法调用"翻译成"SQL 执行"。

机制拆解：

1. `getMapper(UserMapper.class)` → `MapperRegistry` 返回 `MapperProxyFactory` 创建的 **MapperProxy**（实现了 `InvocationHandler`，注意它**不是**静态内部类的匿名实现，而是通用的 Handler）；
2. 调用任意接口方法进入 `MapperProxy.invoke()`：先跳过 Object 方法（equals/hashCode/toString）与**default 方法**（直接反射调用本体）；
3. 其余方法包装成 `MapperMethod`（从方法名/注解解析出 SqlCommandType + 参数形式），以 `接口全限定名.方法名` 从 Configuration 取 MappedStatement；
4. 委托 `sqlSession.selectList/insert/update/delete(...)` 走 Executor 四大对象链路。

**Spring 整合后的注册时机**：`@MapperScan` 的 `ClassPathMapperScanner` 把包下每个接口注册为 **MapperFactoryBean**（FactoryBean），容器启动时 `getObject()` 调 `sqlSession.getMapper()` 得到代理对象注入 IoC——所以 Mapper 接口"没有实现也能 @Autowired"。

> **追问**：为什么接口不能重载？——见 1.2.4；确实要两个签名，改方法名或用 `@Param` 区分场景。与 JDK 代理/CGLIB 的通用对比见 1_java.md 1.2.4。


# 2. SQL 映射与动态 SQL

## 2.1 参数与结果映射

### 2.1.1 #{} 和 ${} 的区别？为什么 #{} 能防 SQL 注入？

| 对比维度 | #{} | ${} |
|---------|-----|-----|
| 处理方式 | 预编译占位符，替换为 `?` | 字符串直接替换拼接 |
| 底层实现 | `PreparedStatement.setXXX()` 传参 | 拼接进 SQL 文本 |
| SQL 注入 | **安全**：参数作为数据传入 | **不安全**：参数成为 SQL 语法的一部分 |
| 使用场景 | 字段值、WHERE 条件值 | 动态表名/列名/ORDER BY/LIKE 模式 |
| 类型处理 | 自动做 JDBC 类型转换、加引号 | 原样输出，需自己注意引号 |

**#{} 防注入原理**：数据库处理 PreparedStatement 分两步——① SQL 模板先解析编译，**语义结构此时已定**；② 参数值后传入，只作数据填充。`'1' OR '1'='1'` 进入参数位后只是"一个更长的字符串"，永远不可能翻转成布尔表达式。

**${} 的安全使用**（表名、列名、排序方向这类结构位 #{} 替代不了，只能用 ${}）：

```xml
<!-- 反例：orderBy=${orderBy} 直接拼接 = 注入点 -->
<!-- 正例：白名单校验 -->
ORDER BY ${@com.example.SqlSafety@check(orderBy)} DESC
```

```java
public static String check(String col) {
    if (!ALLOWED.contains(col)) throw new IllegalArgumentException("非法排序列: " + col);
    return col;
}
```

> **行业实践**：`${}` 出现处必须可审计（代码评审重点）；LIKE 场景优先 `CONCAT('%', #{kw}, '%')`（MySQL）而非 `${}` 包引号，既防注入又利用预编译。

### 2.1.2 Mapper 方法多个参数怎么传？@Param 的作用？

MyBatis 对"一个方法多个入参"的绑定按版本演进：

1. **单个普通对象/Map 参数**：直接按属性名/键名取值（`#{name}`、`#{mapKey}`）；
2. **多个参数（3.4 以前）**：只能 `#{arg0}/#{param1}` 位置名，可读性差；
3. **@Param 命名（标准做法）**：

```java
User findByNameAndAge(@Param("name") String name, @Param("age") Integer age);
// XML: WHERE name = #{name} AND age = #{age}
```

原理：ParamNameResolver 把参数收集成 `Map<参数名/位置名, 值>`，`@Param` 的值作为显式 key 同时保留 `paramN` 序号 key。

**注意三个坑**：

- `List`/数组参数：XML 里默认 key 是 `list`/`collection`（List 单参数）或 `array`（数组），显式 `@Param("ids")` 最稳；
- `@Param` 与对象参数混用时，对象属性可直接平铺取（`#{user.name}` 可省略写成 `#{name}`——前提是对象参数只有一个且无歧义，建议带前缀更清楚）；
- 传 Wrapper（如 MP 的 QueryWrapper）是特例，见 9_mybatis_plus.md。

### 2.1.3 resultType 和 resultMap 有什么区别？什么时候用 resultMap？

- **resultType**：声明"返回什么类型"，MyBatis 按**列名=属性名**自动实例化并反射赋值。适合字段名一致（或开启驼峰映射后一致）、无嵌套关系的简单查询，简单快捷。
- **resultMap**：手工定义"列 → 属性"的映射规则。必须用的场景：
  1. 列名与属性名不一致且不想改 SQL 别名（`user_name` → `userName`）；
  2. **一对一**关联（association）、**一对多**关联（collection）的嵌套结构；
  3. 指定主键 `<id>`（帮助一对多按主键去重合并行，性能也更好）；
  4. 需要特殊 TypeHandler、`javaType`/`jdbcType` 显式声明；
  5. 构造器注入映射（`<constructor>`）。

`autoMapping="true"` 时二者可混用：resultMap 里只声明不一致的字段，其余仍走自动映射。

> **易错**：`resultType="map"` 返回列名做 key，不同数据库大小写行为不同（Oracle 默认大写列名）；想要稳定 key 就配 resultMap 或别名。

### 2.1.4 从数据库查出一行数据，MyBatis 是如何映射成 Java 对象的？

**ResultSetHandler（DefaultResultSetHandler）**的处理流程：

1. 根据 MappedStatement 拿到 ResultMap（用户定义；没定义则按 resultType 自动构建）；
2. 判断这次映射是**简单结果集**还是带嵌套的**聚合结果**：
   - 简单：遍历 ResultSet 每一行 → `createResultObject` 反射建实例 → 逐列经 **TypeHandler** 转换值（VARCHAR→String、INTEGER→Integer…）→ setter/字段赋值；
   - 嵌套（association/collection join 映射）：用 `<id>` 列组合算 **CacheKey** 对行分组——相同主键的多行合并为同一个父对象，子对象/集合元素分别填充（这就是为什么一对多映射一定要写 `<id>`，否则去重靠全列比较，既慢又可能错）；
3. 未显式映射的列在 `autoMappingBehavior`（默认 PARTIAL）下尝试自动映射；**驼峰开关** `mapUnderscoreToCamelCase=true` 让 `user_name` 自动命中 `userName`；
4. 结果回填：List 场景每行对象加入结果集合，返回前写入一级缓存。

> **面试追问**：为什么查询返回的 Date 时间差 8 小时？——JDBC 时区问题（MySQL 驱动 `serverTimezone` 与 JVM 默认时区不一致），与映射逻辑无关但属高频事故；建议库里统一存 UTC 或显式设驱动时区。

### 2.1.5 如何自定义 TypeHandler？典型使用场景？

TypeHandler 负责**JavaType ↔ JdbcType** 的双向转换：设参时 `setNonNullParameter`，取值时三个 `getNullableResult` 重载（按列名/列序/CallableStatement）。

实现三步：

1. 继承 `BaseTypeHandler<T>`（只需补空值处理）或实现 `TypeHandler<T>`；
2. 标 `@MappedTypes(UserStatus.class)` + `@MappedJdbcTypes(JdbcType.VARCHAR)`（也可在配置里全局指定）；
3. 注册：`mybatis.type-handlers-package=com.example.handler` 扫描，或在字段/参数上临时指定 `#{status, typeHandler=xxx.TypeHandler}`。

```java
public class LocalDateTimeStringHandler extends BaseTypeHandler<LocalDateTime> {
    private static final DateTimeFormatter FMT = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, LocalDateTime p, JdbcType t) throws SQLException {
        ps.setString(i, FMT.format(p));                 // 写入：时间 → 字符串列
    }
    @Override
    public LocalDateTime getNullableResult(ResultSet rs, String col) throws SQLException {
        String s = rs.getString(col);
        return s == null ? null : LocalDateTime.parse(s, FMT);   // 读取：字符串列 → 时间
    }
    // 另两个 getNullableResult 重载同理
}
```

**典型场景**：

- 枚举 ↔ int/String（`EnumOrdinalTypeHandler`/`EnumTypeHandler` 内置，MP 有 `IEnum` 方案）；
- `LocalDateTime/LocalDate` ↔ DATETIME/VARCHAR；
- JSON 字符串 ↔ 对象/List/Map（MySQL JSON 列、PG jsonb 都用它）；
- 敏感字段落库加解密/脱敏（写入加密、读取解密——注意这是**列级**方案，全局查询条件里的密文匹配要单独设计）。

> **注意**：TypeHandler 管"类型转换"，插件（拦截器）管"行为增强"；脱敏既可以做 TypeHandler（字段级、静态规则），也可以做 ResultSetHandler 插件（动态规则/权限相关）——面试问区别就往这答。

### 2.1.6 insert 之后如何获取自增主键？

两种方式：

1. **useGeneratedKeys（推荐，MySQL 等支持 JDBC 4 getGeneratedKeys 的库）**：

```xml
<insert id="add" useGeneratedKeys="true" keyProperty="id" keyColumn="id">
    INSERT INTO user (name, age) VALUES (#{name}, #{age})
</insert>
<!-- 执行后实体对象的 id 属性被自动回填，可直接使用 -->
```

底层是 `connection.prepareStatement(sql, RETURN_GENERATED_KEYS)` + `ps.getGeneratedKeys()` 回填 keyProperty。

2. **selectKey（通用兜底，Oracle 序列等）**：

```xml
<insert id="add">
    <selectKey keyProperty="id" resultType="long" order="BEFORE">
        SELECT user_seq.NEXTVAL FROM dual     <!-- BEFORE：先取号再插入 -->
    </selectKey>
    INSERT INTO user (id, name) VALUES (#{id}, #{name})
</insert>
```

`order` 选择：自增主键用 `AFTER`（等价 `SELECT LAST_INSERT_ID()`）；序列/UUID 用 `BEFORE`（先生成再当参数插入）。

**批量插入拿 id**：MySQL 多值 `INSERT ... VALUES (...),(...)` + useGeneratedKeys 可回填**每个对象**的 id（驱动连续返回）；但**不能跨语句保证"连续号段"**（InnoDB 自增分配并发下可能跳号），依赖"id 连续"做分批回查是错误假设。

## 2.2 关联映射

### 2.2.1 association 和 collection 的区别？两种加载方式怎么选？

- **association**：**一对一**（或引用单个对象）——User 里的 `Department department`；`javaType` 指定属性类型。
- **collection**：**一对多**——Department 里的 `List<User> users`；`ofType` 指定**集合元素**类型（`javaType` 指集合本身，可省略）。

两者都支持两种加载方式：

| 方式 | 做法 | SQL 次数 | 特点 |
|------|------|---------|------|
| **嵌套查询**（分次查询） | `<association property="dept" select="com.x.DeptMapper.selectById" column="dept_id"/>` | **N+1** | 配置简单、能走缓存；N 大时性能灾难 |
| **嵌套结果**（join 一次查） | resultMap 内联子映射 + LEFT JOIN 一条 SQL | 1 | 结果集有冗余但一次往返；**推荐** |

> **行业实践**：生产默认选嵌套结果；真要拆成两次查询，就用 Java 层显式两段式（查主表 → 批量查子表按外键组装）替代隐式嵌套查询——后者脱离 MyBatis 缓存就退化回 N+1。

### 2.2.2 一对一和一对多关联查询怎么写？（嵌套结果示例）

**一对一（association + JOIN）**：

```xml
<resultMap id="userWithDept" type="User">
    <id property="id" column="u_id"/>
    <result property="name" column="u_name"/>
    <association property="department" javaType="Department">
        <id property="deptId" column="dept_id"/>
        <result property="deptName" column="dept_name"/>
    </association>
</resultMap>

<select id="selectUserWithDept" resultMap="userWithDept">
    SELECT u.id u_id, u.name u_name, d.id dept_id, d.name dept_name
    FROM user u LEFT JOIN department d ON u.dept_id = d.id
</select>
```

**一对多（collection + JOIN）**：

```xml
<resultMap id="deptWithUsers" type="Department">
    <id property="id" column="d_id"/>
    <result property="name" column="d_name"/>
    <collection property="users" ofType="User">
        <id property="id" column="u_id"/>
        <result property="name" column="u_name"/>
    </collection>
</resultMap>

<select id="selectDeptWithUsers" resultMap="deptWithUsers">
    SELECT d.id d_id, d.name d_name, u.id u_id, u.name u_name
    FROM department d LEFT JOIN user u ON d.id = u.dept_id
</select>
```

**细节**：

- 列名冲突必须起别名（两个表都有 id），且 `<id>` 分别标在父、子层级——一对多去重、父子行归属全靠 CacheKey；
- `columnPrefix` 可批量映射前缀列（配合自连接、同表多角色关联）；
- 多对多建中间表拆成两个一对多即可，没有第三种标签。

> **易错**：忘写 `<id>` 时一对多也能"看起来正常"，但去重退化为全列比较——宽表下又慢又可能把同主键不同列的行判成两个对象。

### 2.2.3 什么是 N+1 查询问题？MyBatis 下如何解决？

**定义**：主查询拿到 N 条记录后，为每条记录再发 1 次关联查询，共 **1+N** 次 SQL。典型来源：嵌套查询（association/collection 的 select 属性）、延迟加载在循环里首次访问懒属性、ORM"查列表再逐个取关联"。

**解决**（按优先级）：

1. **改嵌套结果为 JOIN 一次查**（2.2.2）——把 N+1 压成 1 条 SQL；
2. **两段式批量查询**：主表查完提取所有外键，`WHERE dept_id IN (...)` 一次取子表，内存按 key 分组组装——join 影响行数爆炸（一对多 × 一对多）时反而更优；
3. MyBatis 层能开缓存就开（嵌套查询命中二级缓存/一级缓存可省后续查询），但缓存不是根治；
4. 延迟加载把"未用到的关联"成本推迟（不访问就不查），循环里"有条件访问"仍会产生随机 N。

**辨析**：JOIN 把复杂度从"往返次数"换成"结果集冗余与去重成本"——1 对几用 JOIN 好，1 对几千或需要分页时用两段式（分页 + JOIN 会出现"一页里行数被关联行放大"的语义坑，分页对象数还是对象行数要自己控制）。

### 2.2.4 MyBatis 延迟加载的原理？怎么配置？

**结论**：给结果对象生成**代理**，首次访问懒属性时才发 SQL 填充。

机制拆解：

1. 开启 `<setting name="lazyLoadingEnabled" value="true"/>` 后，含 association/collection（`fetchType="lazy"` 或全局）的查询，返回的不是目标类而是 **Javassist（默认）/CGLIB 生成的子类代理**；
2. 代理拦截 getter：属性未加载 → 执行配置的 select 填充 → 返回值；已加载 → 直接返回；
3. `aggressiveLazyLoading`（3.4.1+ 默认 false）：true 是"访问**任意**属性加载**所有**懒属性"（旧行为，等于没懒）；false 才是按属性**逐一**加载；
4. 局部控制：association/collection 上 `fetchType="lazy|eager"` 覆盖全局开关。

**前提条件（高频追问）**：懒加载在"取属性时"需要 SqlSession 仍然存活——脱离会话访问会拿不到数据（Spring 下事务结束即关会话，Controller/序列化阶段触发懒属性会失败）。常见对策：VO 组装阶段显式触发、或 JOIN 一次查全、或 `spring.jpa` 式 open-session-in-view 的反面教材（不建议 MyBatis 配过滤器延长会话）。Jackson 序列化代理对象还会遇到"懒属性被反射全部访问"问题，需 `@JsonIgnoreProperties`/VO 隔离。

## 2.3 动态 SQL

### 2.3.1 动态 SQL 有哪些标签？trim 的作用和本质？

| 标签 | 作用 |
|------|------|
| `<if test="">` | 条件成立才拼接片段；OGNL 表达式（`name != null and name != ''`） |
| `<choose>/<when>/<otherwise>` | 多分支互斥（等价 if-else if-else，命中一个即止） |
| `<where>` | 有内容才加 `WHERE`，并**去掉开头多余的 AND/OR** |
| `<set>` | 有内容才加 `SET`，并**去掉结尾多余逗号** |
| `<trim>` | 通用裁剪器：prefix / suffix / prefixOverrides / suffixOverrides |
| `<foreach>` | 遍历集合，IN / 批量插入 |
| `<bind>` | 创建变量供后续使用（`<bind name="pattern" value="'%'+keyword+'%'"/>`） |
| `<sql>` + `<include>` | SQL 片段复用 |
| `<insert>` 内 `<selectKey>` | 取主键 |

**trim 是 where/set 的本质**：

```xml
<!-- <where> 等价于 -->
<trim prefix="WHERE" prefixOverrides="AND |OR "> ... </trim>
<!-- <set> 等价于 -->
<trim prefix="SET" suffixOverrides=","> ... </trim>
```

prefixOverrides/suffixOverrides 按 `|` 分隔多候选，只在片段首/尾**恰好**出现时裁掉；prefix/suffix 只在内容非空时添加。

> **易错**：`<if test="flag == 0">`——OGNL 里 `==` 对包装类型走 equals 语义问题不大，但 `and/or/not` 必须小写；字符串比较写 `test='type == "A"'` 注意外层引号用单引号。

### 2.3.2 foreach 批量 IN 和批量插入怎么用？有什么限制？

```xml
<!-- 批量查询 IN -->
<select id="selectByIds" resultMap="user">
    SELECT * FROM user WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>

<!-- 批量插入：多值一条 SQL -->
<insert id="batchInsert">
    INSERT INTO user (name, age) VALUES
    <foreach collection="list" item="u" separator=",">
        (#{u.name}, #{u.age})
    </foreach>
</insert>
```

**属性**：`collection`（List 单参数默认 `list`/`collection`，数组 `array`，@Param 名最稳）、`item`、`index`（List 下标/Map 键）、`open/close/separator`。

**限制与坑**：

1. 多值 INSERT 受 `max_allowed_packet`（默认 4~64MB 视版本）限制，且单条 SQL 过大影响解析——**分批 500~1000 行**；
2. IN 列表过长同样撑大 SQL、可能废掉索引选择（MySQL 对 IN 的 range 优化在超长列表退化）——超大集合改临时表/两段式；
3. 多值 INSERT 的自增 id 不保证连续（并发分配），别拿"连续段"做逻辑（见 2.1.6）；
4. Oracle 批量插入不支持多值 VALUES，要 `INSERT ALL` 或 `BEGIN ... END;` 包裹 foreach——方言差异是 foreach 方案移植的最大暗礁。

### 2.3.3 注解方式和 XML 方式有什么区别？怎么选？

| 维度 | XML 映射器 | 注解映射 |
|------|-----------|---------|
| SQL 位置 | 独立 mapper 文件（`<select>`/`<insert>`/…） | `@Select/@Insert/@Update/@Delete` |
| 动态 SQL | 全标签体系（if/foreach/choose） | 不支持标签，需 `@SelectProvider` + SQL Builder 或 script 标记 |
| 复杂 resultMap | 完整支持（association/collection/嵌套） | 仅简单映射，复杂结构仍回 XML |
| 可维护性 | SQL 与代码分离，DBA/脚本可审查、可替换 | 代码集中，短 SQL 顺手 |
| 热修改 | 改 XML 重启即可（配合配置化加载甚至免编译） | 改 Java 需重编译 |

**结论**：团队实践以 **XML 为主**（复杂 SQL、resultMap、动态 SQL 都在 XML）；简单固定 SQL 可用注解省一份文件；Provider 是"注解 + 动态"的折中，可读性一般——能用 XML 就别上 Provider。两者可共存（接口方法要么注解要么 XML，不能同时）。


# 3. 缓存、插件与整合生态

## 3.1 缓存机制

### 3.1.1 MyBatis 的一级缓存和二级缓存有什么区别？

| 对比 | 一级缓存（Local Cache） | 二级缓存（namespace 缓存） |
|------|----------------------|--------------------------|
| 作用域 | 同一个 SqlSession 内 | 同一 namespace 下所有 SqlSession 共享 |
| 默认状态 | **默认开启，不能关闭**（可改作用域） | 默认关闭，需 `<cache/>` 显式开启 |
| 存储 | SqlSession 内部 PerpetualCache（HashMap） | 可插拔 Cache 实现（默认 PerpetualCache，可换 Ehcache/Redis/Caffeine） |
| 失效条件 | 会话内任何增删改、`clearCache()`、commit/close | namespace 内增删改**清空整个 namespace 缓存** |
| 序列化 | 不需要 | 对象需实现 Serializable（供第三方缓存存储） |
| 粒度控制 | 语句级：`<select flushCache="true">` 跳过 | `<cache eviction/readWrite/useCache/>`；`<select useCache="false">` 单条排除 |

**命中条件（一级）**：同一 SqlSession + 同一 MappedStatement + **参数相同** + RowBounds 相同 + SQL 相同（CacheKey 由这些组成），且中间无增删改。`localCacheScope=STATEMENT`（默认 SESSION）可把一级缓存降为"仅本次语句内"，每条 SQL 执行完即清空。

**二级缓存读写细节**：查询结果先进一级；**SqlSession commit/close 时才把一级刷入二级**（不是立即）——所以两个并发会话读同一数据不会互相"看见"对方缓存；增删改先清二级再执行。Spring 整合下同一事务共享一个 SqlSession，一级缓存在事务内生效，跨事务共享要靠二级。

### 3.1.2 缓存的脏读/脏数据问题？生产环境该不该用？

**一级缓存脏读**：

```text
Session1 SELECT id=1 → 缓存
Session2 UPDATE id=1 → commit（数据库已新值）
Session1 再 SELECT id=1 → 返回旧缓存值
```

会话生命周期内"别人改了库"无法感知。解决：① 会话尽量短、及时关闭（Spring 事务边界天然是这个粒度）；② `sqlSession.clearCache()`；③ `localCacheScope=STATEMENT`（牺牲缓存换强一致）。

**二级缓存脏数据**（比一级严重）：

1. **跨 namespace 不感知**：A 的查询 JOIN 了 B 表，B 的增删改只清自己 namespace 的缓存，A 的缓存全是脏的。缓解：`<cache-ref namespace="..."/>` 让多方共享一份缓存（谁写都清），本质是"关联表划进同一失效域"；
2. **分布式多节点**：每台 JVM 一份进程内二级缓存，互相不同步——节点 A 更新、节点 B 仍吐旧值；
3. **全表清空粒度粗**：namespace 一条写清掉整个 namespace 查询缓存，命中率实际很低。

**行业结论（答题落点）**：**生产环境不推荐开二级缓存**。缓存的正确归属是业务层——用 Redis 做 Cache-Aside（见 27_redis.md），失效策略、分布式一致性、监控都有成熟方案；MyBatis 二级缓存是"进程内 + 表级失效"的初级方案，与 Redis 方案并存时只会制造两套真相。一级缓存保留默认（作用域=事务）即可，不必刻意干预。

## 3.2 插件、分页与批量

### 3.2.1 MyBatis 插件机制的实现原理？

**结论**：基于 **JDK 动态代理 + 责任链**，对**四大对象**的方法调用做拦截增强。

**可拦截点**（@Intercepts + @Signature 精确到"类 + 方法 + 参数类型"）：

| 对象 | 典型拦截方法 | 应用 |
|------|-------------|------|
| Executor | update / query / flushStatements / commit | 二级缓存（CachingExecutor 本体）、读写分离、慢 SQL 统计 |
| ParameterHandler | setParameters / getParameter | 参数加密、脱敏、默认值填充 |
| StatementHandler | prepare / parameterize / query / update | **SQL 改写（分页插件）**、表名重写 |
| ResultSetHandler | handleResultSets / handleCursor | 结果集脱敏、字段解密 |

**织入机制**：Configuration 的 `newExecutor/newStatementHandler/...` 工厂方法创建对象后，统一过 `interceptorChain.pluginAll(target)`——遍历插件列表逐个调 `plugin()`，内部 `Plugin.wrap()` 判断该对象是否命中插件声明的 Signature，命中则生成**代理对象层层套娃**（插件 A 包 B 包目标）。调用链：

```text
proceed() 顺序: Caller → 外层插件A.intercept → B.intercept → 真实对象 → 返回
```

每个 `intercept(Invocation)` 里 `invocation.proceed()` 才是放行下一层——**忘调 proceed 直接断链**（返回 null 或自行组装返回值）。

> **易错**：一个插件类拦截多个方法时，`invoke` 里要按 method.getName() 分派；`@Signature` 的 parameterTypes 必须写全签名。插件增强常驻字节码代理链，生产只开必要插件。

### 3.2.2 PageHelper 分页插件的原理和注意事项？

**原理（拦截 StatementHandler.prepare 改写 SQL）**：

1. `PageHelper.startPage(pageNum, pageSize)` 把分页参数存入 **ThreadLocal**；
2. 拦截器在 **SQL 预编译（prepare）前**介入：从 ThreadLocal 取分页参数，按数据库方言改写 BoundSql——MySQL 拼 `LIMIT ?, ?`、Oracle 套 `ROWNUM` 子查询；
3. 自动构造 **COUNT SQL**（`SELECT count(0) FROM (原SQL)` 或优化为去掉 ORDER BY 的简化 count）先查总数；
4. 执行分页 SQL，结果包装成 `Page<T>`（ArrayList 子类带 total/pageNum/pages 元数据）；
5. finally 中 **clearPage() 清 ThreadLocal**——防线程池复用线程导致"下一页"参数串号。

**高频注意事项**：

1. `startPage()` 与目标查询之间**不能插入其他查询**——它只作用于 ThreadLocal 里"下一条"，中间多查一条会把分页错扣；
2. try-finally 或紧随其后使用；线程池场景 PageHelper 自身已清理，但自定义分页要自己清；
3. `count=false` 可关总数查询（瀑布流滚动分页不需要总数）；
4. 深分页 `LIMIT 1000000, 10` 依旧要先扫描跳过 100 万行（O(offset)）——PageHelper 解决"写分页 SQL"，**不解决深分页性能**（方案见 16_mysql.md 延迟关联）；
5. 多数据源/方言用 `helperDialect` 显式指定，自动探测可能猜错。

### 3.2.3 MyBatis 原生分页有几种方式？RowBounds 为什么少用？

1. **SQL 物理分页（正解）**：方言的 LIMIT/OFFSET、ROWNUM、FETCH FIRST——数据库只返回目标页，配合 `#{}` 传参或 PageHelper 插件；
2. **RowBounds 逻辑分页**：`selectList(stmt, param, new RowBounds(offset, limit))`——MyBatis 把**全部结果**读进内存、在 ResultSet 层面跳过 offset 行取 limit 行。网络与内存都按全表付成本，大表直接 OOM 级灾难，生产几乎不用（面试常拿来考"你知道逻辑分页吗"）；
3. **游标（Cursor/FetchSize）**：流式逐批取（`fetchSize=Integer.MIN_VALUE` for MySQL），适合超大结果集导出，但仍非"分页语义"。

> **一句话**：分页必须发生在**数据库端**；MyBatis-Plus 的 PaginationInnerInterceptor 是另一种主流实现（拦截 Executor 改写，自带 count 优化与多方言），见 9_mybatis_plus.md。

### 3.2.4 MyBatis 批量操作有哪几种方式？各自注意点？

**方式一：BatchExecutor（真·批量，推荐）**

```java
try (SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH)) {
    UserMapper mapper = session.getMapper(UserMapper.class);
    for (int i = 0; i < list.size(); i++) {
        mapper.insert(list.get(i));
        if (i % 1000 == 999) { session.flushStatements(); session.clearCache(); } // 分批防OOM
    }
    session.flushStatements();
    session.commit();
}
```

JDBC `addBatch/executeBatch` 打包发送，同一条 PreparedStatement 复用。注意：flush 前返回值失真（见 1.2.3）；每千条 flush + clearCache（一级缓存也会涨）；Spring 下可用 `sqlSessionTemplate.flushStatements()` 或 MP 的 saveBatch。

**方式二：foreach 多值 INSERT（一条 SQL 塞 N 组 VALUES）**

单次网络往返、吞吐高，受 `max_allowed_packet` 限制需 500~1000 行/批（见 2.3.2）；自增 id 不连续；Oracle 要特殊语法。

**方式三：循环单条 insert（反面教材）**——N 次网络往返，只适合个位数场景。

**选型**：读多写少的常规系统，方式二最常用；超大批量（数十万行）用方式一分批，或直接走数据库 Loader（MySQL `LOAD DATA INFILE`，见 16_mysql.md）。Spring 整合里方式一要注意**事务内混用 BATCH 与普通会话的可见性**：BATCH 未 flush 前，同事务内后续 SELECT 看不到未提交批次——"批完先 flush 再查"。

## 3.3 Spring 整合与工程工具

### 3.3.1 Spring Boot 整合 MyBatis 做了哪些事？自动配置原理？

**使用步骤**：

1. 依赖 `mybatis-spring-boot-starter` + 驱动；
2. `application.yml` 配 `spring.datasource.*` 与 `mybatis.*`（mapper-locations、type-aliases-package、configuration.map-underscore-to-camel-case）；
3. 启动类 `@MapperScan("com.xxx.mapper")`。

**自动配置链路（MybatisAutoConfiguration）**：

1. 条件：classpath 同时存在 `SqlSessionFactory` 与 `DataSource`（`@ConditionalOnBean(DataSource.class)` 等）才生效；
2. 创建 **SqlSessionFactoryBean**：注入 DataSource（通常 Hikari）、合并 mybatis-config.xml 与 yml 属性、注册 TypeHandler/Interceptor/别名 → `getObject()` 产出 SqlSessionFactory；
3. 创建 **SqlSessionTemplate** 单例（线程安全门面，见 1.2.2）；
4. `@MapperScan` 经 `MapperScannerConfigurer` 扫包，每个接口注册成 **MapperFactoryBean**，注入容器的是其产出的 MapperProxy；
5. 事务：Mapper 执行时 SqlSessionTemplate 从 `TransactionSynchronizationManager` 取**当前事务绑定的 SqlSession**——同事务同会话、一级缓存生效；无事务则用完即关（连接归还池）。

> **易错**：忘了 @MapperScan 与 @Mapper 都没加 → 容器里没有 Mapper Bean，注入报 NoSuchBean；自动配置类位置 2.7+ 用 `AutoConfiguration.imports`（见 10_springboot.md）。

### 3.3.2 @Mapper 和 @MapperScan 有什么区别？

- **@Mapper**（`org.apache.ibatis.annotations.Mapper`）：标在**单个接口**上，配合 `@MapperScan` 或自动配置里的 `AutoConfiguredMapperScannerRegistrar`（扫 `@MapperScan` 未指定时的兜底：自动扫描启动包下带该注解的接口）。逐接口标注，繁琐；
- **@MapperScan("com.x.mapper")**（spring 扩展注解）：标在配置类/启动类上，**按包批量注册**，可配 `annotationClass`（只认某注解）/`markerInterface`（只认某父接口）做二次过滤。**推荐**。

**坑**：

1. 多个模块/多数据源：每个 `@MapperScan` 绑定各自 SqlSessionFactory（`sqlSessionFactoryRef`），扫错包注入到错误库——多数据源事故的常见根因（见 9_mybatis_plus/10_springboot 动态数据源题）；
2. 第三方增强不通用：tk.mybatis 要 `tk.mybatis.spring.annotation.MapperScan`，MyBatis-Plus 用 `com.baomidou.mybatisplus.annotation.MapperScan`（或直接用原生 @MapperScan 也可，MP 靠自己的注入器替换 CRUD 方法实现）；
3. 接口无注解、包路径漏扫——注入失败先查包。

### 3.3.3 MyBatis Generator 和通用 Mapper（tk.mapper）有什么区别？

- **MBG**：**代码生成器**。连库读表结构，一次性生成实体、Mapper 接口、XML（含 Example/Criteria 条件查询类）。产物是普通 MyBatis 代码，后续手工维护；再生成会**覆盖**，需要定制模板或分模块隔离。
- **tk.mybatis 通用 Mapper**：**运行时增强**。接口继承 `Mapper<T>`（组合 BaseSelectMapper/BaseInsertMapper…）即获得全套单表 CRUD 与 Example 查询，**不生成 XML**——方法注入发生在启动时（MapperHelper 按实体注解拼 MappedStatement 塞进 Configuration）。

**组合用法**：项目初期 MBG 生成骨架 + 接口改继承通用 Mapper，复杂 SQL 手写 XML——兼顾生成效率与定制自由。当下新项目更多直接选 **MyBatis-Plus**（Lambda 条件构造器、分页拦截器、代码生成器、逻辑删除/乐观锁全家桶），能力覆盖两者，见 9_mybatis_plus.md。
