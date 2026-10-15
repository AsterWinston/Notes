# 1. MyBatis 最佳实践与性能调优

```
以下是基于源码理解和实战经验总结的 MyBatis 最佳实践和常见问题解决方案。
```

# 2. N+1 查询问题

## 2.1 问题描述

```java
// 典型的 N+1 问题场景
public interface ClazzMapper {
    @Select("select cid,cname from t_clazz")
    List<Clazz> selectAll();  // 1 次查询（获取所有班级）
}

@Results({
    @Result(property = "stus",
            column = "cid",
            many = @Many(select = "selectByCid",
                         fetchType = FetchType.EAGER))
})
// 每个班级都会触发一次查询学生（N 次查询）
// 总共：1 + N 次 SQL！
```

## 2.2 解决方案

```
方案1：使用嵌套结果映射（推荐）
  一次性 JOIN 查询出所有数据，通过 ResultMap 的嵌套映射处理

方案2：使用延迟加载
  fetchType = FetchType.LAZY
  只在真正访问关联对象时才执行查询

方案3：使用 @Batch 批量查询（MyBatis 3.5+）
  在 <collection> 中配置 column="{cid=cid}" 
  一次性传入所有 cid 进行批量查询

方案4：使用 MyBatis-Plus 的关联查询
  或者手动编写优化的 SQL
```

```xml
<!-- 方案1：嵌套结果映射（一条 SQL） -->
<resultMap id="clazzResultMap" type="Clazz">
    <id property="cid" column="cid"/>
    <result property="cname" column="cname"/>
    <collection property="stus" ofType="Student">
        <id property="sid" column="sid"/>
        <result property="sname" column="sname"/>
    </collection>
</resultMap>
<select id="selectAllWithStudents" resultMap="clazzResultMap">
    select c.cid, c.cname, s.sid, s.sname
    from t_clazz c left join t_stu s on c.cid = s.cid
</select>
```

# 3. 批量操作优化

## 3.1 批量插入

```java
// ====== 错误做法：循环单条插入 ======
for (Car car : carList) {
    mapper.insert(car); // 每次都是独立的 SQL + 事务
}
// 对 10000 条数据，就是 10000 次网络 IO

// ====== 推荐做法 1：使用 foreach 拼接 ======
// Mapper.xml
<insert id="insertBatch">
    insert into t_car(car_num,brand,guide_price,produce_time,car_type)
    values
    <foreach collection="cars" item="car" separator=",">
        (#{car.carNum},#{car.brand},#{car.guidePrice},#{car.produceTime},#{car.carType})
    </foreach>
</insert>

// Java 代码
mapper.insertBatch(carList);  // 一次 SQL，一条网络 IO
```

```
注意：MySQL 默认限制单条 SQL 的最大长度（max_allowed_packet）
    大批量时建议分批，每批 500-1000 条：
    
    int batchSize = 500;
    for (int i = 0; i < carList.size(); i += batchSize) {
        int end = Math.min(i + batchSize, carList.size());
        mapper.insertBatch(carList.subList(i, end));
    }
```

## 3.2 批量更新/删除

```java
// 推荐：使用 CASE WHEN 或 foreach
<update id="updateBatch">
    update t_car set
    brand = case id
        <foreach collection="cars" item="car">
            when #{car.id} then #{car.brand}
        </foreach>
    end,
    guide_price = case id
        <foreach collection="cars" item="car">
            when #{car.id} then #{car.guidePrice}
        </foreach>
    end
    where id in
    <foreach collection="cars" item="car" open="(" separator="," close=")">
        #{car.id}
    </foreach>
</update>
```

## 3.3 使用 BATCH 执行器

```java
// 使用 BatchExecutor 进行批量操作
SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
try {
    CarMapper mapper = sqlSession.getMapper(CarMapper.class);
    for (Car car : carList) {
        mapper.insert(car);  // 只 addBatch，不执行
    }
    sqlSession.flushStatements();  // 统一执行 batch
    sqlSession.commit();
} finally {
    sqlSession.close();
}

// Spring 中使用
@Configuration
public class MyBatisConfig {
    @Bean
    public SqlSessionTemplate batchSqlSessionTemplate(SqlSessionFactory factory) {
        return new SqlSessionTemplate(factory, ExecutorType.BATCH);
    }
}
```

# 4. 大字段处理

```java
// 问题：查询包含 BLOB/CLOB/TEXT 大字段的表时，即使不需要大字段也会加载

// ====== 解决方案 1：按需查询（不用 select *）======
<select id="selectSummaryList" resultType="Car">
    select id, car_num, brand, guide_price from t_car  -- 不查大字段
</select>

// ====== 解决方案 2：懒加载大字段 ======
<resultMap id="articleLazyMap" type="Article">
    <id property="id" column="id"/>
    <result property="title" column="title"/>
    <result property="content" column="id"
            select="selectContentById"          ← 延迟加载内容
            fetchType="lazy"/>
</resultMap>

<select id="selectContentById" resultType="string">
    select content from t_article where id = #{id}
</select>

// ====== 解决方案 3：使用流式查询（Cursor）======
try (Cursor<Article> cursor = mapper.selectAllArticleCursor()) {
    cursor.forEach(article -> {
        // 逐条处理，逐条释放，不一次性加载全部到内存
        processArticle(article);
    });
}
```

# 5. SQL 注入防范

```java
// ====== 错误：${} 拼接导致 SQL 注入 ======
<select id="findByOrder" resultType="Car">
    select * from t_car order by ${orderField} ${orderDirection}
</select>
// 用户传入 orderField = "id; drop table t_car; --" → 灾难！

// ====== 正确：白名单校验 ======
public List<Car> findByOrder(String orderField, String orderDirection) {
    // 白名单校验
    Set<String> allowedFields = Set.of("id", "brand", "guide_price", "produce_time");
    Set<String> allowedDirections = Set.of("ASC", "DESC");

    if (orderField == null || !allowedFields.contains(orderField)) {
        throw new IllegalArgumentException("Invalid order field: " + orderField);
    }
    if (orderDirection == null || !allowedDirections.contains(orderDirection.toUpperCase())) {
        throw new IllegalArgumentException("Invalid order direction: " + orderDirection);
    }

    return mapper.findByOrder(orderField, orderDirection.toUpperCase());
}

// ====== 能用 #{} 的地方绝不用 ${} ======
// #{} 使用 PreparedStatement，天然防注入
// ${} 直接拼接字符串，必须校验

// ====== 模糊查询：使用 concat 而非 ${} ======
// 错误：name like '%${name}%'
// 正确：
name like concat('%', #{name}, '%')
// 或
name like "%"#{name}"%"
```

# 6. 连接池配置（以 Druid 为例）

```yaml
# application.yml
spring:
  datasource:
    druid:
      # 基础配置
      url: jdbc:mysql://localhost:3306/test?useSSL=false&serverTimezone=Asia/Shanghai
      username: root
      password: 123456

      # 连接池配置
      initial-size: 5                    # 初始连接数
      min-idle: 5                        # 最小空闲连接
      max-active: 20                     # 最大活跃连接（根据业务 QPS 调整）
      max-wait: 60000                    # 获取连接超时（ms）

      # 连接有效性检测
      test-on-borrow: false              # 获取连接时不检测（性能考虑）
      test-while-idle: true              # 空闲时检测
      validation-query: select 1         # 检测 SQL
      time-between-eviction-runs-millis: 60000  # 检测间隔

      # 连接泄露检测
      remove-abandoned: true
      remove-abandoned-timeout: 180      # 超时 180 秒视为泄露

      # 慢 SQL 监控
      filter:
        stat:
          enabled: true
          slow-sql-millis: 1000          # 慢 SQL 阈值
          log-slow-sql: true
```

# 7. 常见坑点

## 7.1 insert 后自增主键获取不到

```java
// 问题：直接调用 mapper.insert(car) 后 car.getId() 为 null
// 解决：

// XML 方式
<insert id="insert" useGeneratedKeys="true" keyProperty="id" keyColumn="id">
    insert into t_car values(null, #{carNum}, #{brand}, ...)
</insert>

// 注解方式
@Insert("insert into t_car values(null, #{carNum}, #{brand}, ...)")
@Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "id")
int insert(Car car);
```

## 7.2 参数为空时 null 处理

```java
// 问题：参数为 null 或空字符串时 SQL 异常
// Java 传入 name = null
// SQL: select * from t_user where name = #{name}
// → select * from t_user where name = null  ← 查不到数据

// 解决1：动态 SQL
<select id="findByName" resultType="User">
    select * from t_user
    <where>
        <if test="name != null and name != ''">
            name = #{name}
        </if>
    </where>
</select>

// 解决2：使用 @SelectProvider 构建动态 SQL
// 或使用 MyBatis-Plus 的条件构造器
```

## 7.3 一级缓存导致的数据不一致

```java
// 场景：两个请求通过同一个 SqlSession 访问同一数据
// 请求 A 查询得到旧数据
// 请求 B 修改了数据
// 请求 A 再次查询 → 一级缓存命中，拿到旧数据

// 解决：
// 1. Spring+MyBatis 环境下，每次请求使用独立的 SqlSession（推荐）
// 2. 设置 localCacheScope = STATEMENT
<setting name="localCacheScope" value="STATEMENT"/>
// 3. 关键查询前手动清空缓存
sqlSession.clearCache();
```

## 7.4 @Transactional 与 MyBatis 事务

```java
// 问题：@Transactional 不生效，数据没有回滚

// 常见原因：
// 1. 方法不是 public
// 2. 同类内部调用（this.method() 不走代理）
// 3. 异常被 catch 了没抛出去
// 4. 异常不是 RuntimeException（默认只回滚 RuntimeException）

// 解决：指定回滚的异常类型
@Transactional(rollbackFor = Exception.class)
public void transfer(String from, String to, Double money) {
    // ...
}
```

## 7.5 大量 in 查询的性能问题

```java
// 问题：where id in (1,2,3,...,10000) 导致 SQL 过大

// 解决：分批查询
public List<Car> findByIds(List<Long> ids) {
    List<Car> result = new ArrayList<>();
    int batchSize = 200;
    for (int i = 0; i < ids.size(); i += batchSize) {
        List<Long> batch = ids.subList(i, Math.min(i + batchSize, ids.size()));
        result.addAll(mapper.selectByIds(batch));
    }
    return result;
}
```

# 8. 性能调优清单

```
┌────────────────────────────────────────────────────────────────┐
│                    MyBatis 性能调优 Checklist                    │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  [ ] SQL 层面                                                   │
│      [ ] 使用 select 列名 代替 select *                         │
│      [ ] 合理使用索引（通过 EXPLAIN 检查执行计划）                 │
│      [ ] 避免大表 JOIN（考虑分步查询 + 延迟加载）                   │
│      [ ] 使用 limit 限制返回行数                                 │
│      [ ] 批量操作用 foreach 代替循环调用                          │
│                                                                │
│  [ ] MyBatis 层面                                              │
│      [ ] 使用 RawSqlSource 代替 DynamicSqlSource（静态 SQL）     │
│           → 含有动态标签的 SQL 每次都要遍历 SqlNode 树             │
│      [ ] 合理配置 fetchSize（大结果集时避免一次加载全部）           │
│      [ ] 合理配置 defaultStatementTimeout                       │
│      [ ] 对不变的查询启用二级缓存（需要小心一致性）                  │
│      [ ] 使用 @Param 传递参数（性能略优于默认的 arg0/arg1）        │
│                                                                │
│  [ ] 代理与反射层面                                              │
│      [ ] getMapper() 只需调用一次，缓存到成员变量                  │
│      [ ] 避免在循环中重复 openSession（复用 SqlSession）          │
│                                                                │
│  [ ] 数据库连接层面                                              │
│      [ ] 合理配置连接池大小                                       │
│      [ ] 使用连接池（不要用 UNPOOLED）                            │
│      [ ] 确保连接正确关闭（try-with-resources）                    │
│                                                                │
│  [ ] 映射层面                                                   │
│      [ ] 使用 mapUnderscoreToCamelCase 减少 resultMap 配置       │
│      [ ] 列别名严格对应属性名，避免不必要的 resultMap              │
│      [ ] 复杂映射优先使用嵌套结果映射，避免 N+1                    │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

# 9. XML vs 注解选型

```
┌──────────────────────────────┬──────────────────────────────────┐
│           使用 XML            │            使用注解               │
├──────────────────────────────┼──────────────────────────────────┤
│  SQL 复杂程度：中/高           │  SQL 复杂程度：低                  │
│  包含动态 SQL                 │  简单的单表 CRUD                  │
│  多表关联                     │  SELECT 单条记录                  │
│  需要 SQL 片段复用             │  快速原型开发                     │
│  需要 DBA/SQL 专家评审         │  SQL 较少（< 10 条/表）           │
│  SQL 可能经常调整              │  SQL 基本固定                     │
│  团队对 SQL 水平要求高         │  团队偏向"全注解"              │
└──────────────────────────────┴──────────────────────────────────┘

推荐策略：
  1. 简单的单表 CRUD → 注解（减少 XML 文件数）
  2. 复杂查询/关联/动态SQL → XML（表达能力更强）
  3. 同一接口可混合使用（XML 优先，XML 中有的语句不再用注解）
```

# 10. MyBatis 日志分析

```java
// MyBatis 日志的输出格式很有帮助，可以用于调试和性能分析

// ====== 开启日志 ======
// mybatis-config.xml
<settings>
    <setting name="logImpl" value="SLF4J"/>
</settings>

// logback.xml
<logger name="com.test.mapper" level="DEBUG"/>

// ====== 输出示例 ======
// 1. 获取连接
// ==> Opening JDBC Connection

// 2. 创建 PreparedStatement
// ==> Preparing: select id, brand, guide_price from t_car where id = ?

// 3. 设置参数
// ==> Parameters: 1(Long)

// 4. 返回结果行数和耗时
// <==        Row: 1
// <==      Total: 1

// 通过日志可以：
//   - 确认最终执行的 SQL（#{} 已替换为 ?）  
//   - 确认参数值是否正确
//   - 确认返回行数是否符合预期
//   - 定位慢 SQL
```

# 11. 线程安全的 SqlSession 封装

```java
/**
 * 在非 Spring 环境中手动管理 SqlSession 的线程安全
 */
public class ThreadSafeSqlSessionManager {

    private static final SqlSessionFactory sqlSessionFactory = buildFactory();

    private static final ThreadLocal<SqlSession> sessionHolder = new ThreadLocal<>();

    public static SqlSession getSession() {
        SqlSession session = sessionHolder.get();
        if (session == null) {
            session = sqlSessionFactory.openSession();
            sessionHolder.set(session);
        }
        return session;
    }

    public static void closeSession() {
        SqlSession session = sessionHolder.get();
        if (session != null) {
            session.close();
            sessionHolder.remove();
        }
    }

    /**
     * 获取 Mapper（封装了线程安全的 SqlSession 获取）
     */
    public static <T> T getMapper(Class<T> mapperClass) {
        return getSession().getMapper(mapperClass);
    }
}
```

# 12. 总结：MyBatis 实战十大黄金法则

```
1.  SQL 写在 XML 中，保持 Java 代码纯粹

2.  能用 #{} 的地方绝不用 ${}，${} 必须做白名单校验

3.  insert 操作始终配置 useGeneratedKeys

4.  避免 select *，只查需要的列

5.  批量操作使用 foreach 或 BatchExecutor

6.  大结果集必须分页（PageHelper 或手动 LIMIT）

7.  XML 的 namespace 和 id 保持与 Mapper 接口一致

8.  使用 Mapper 接口代理（getMapper），不要手写 DaoImpl

9.  Spring 环境下用 @MapperScan 自动注入 Mapper

10. 理解一级缓存和二级缓存的作用域，避免缓存陷阱
```
