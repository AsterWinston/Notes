# 1. limit分页
```
mysql的limit后面两个数字：

第一个数字：startIndex（起始下标。下标从0开始。）
第二个数字：pageSize（每页显示的记录条数）
假设已知页码pageNum，还有每页显示的记录条数pageSize，第一个数字可以动态的获取吗？
startIndex = (pageNum - 1) * pageSize
所以，标准通用的mysql分页SQL：

select * from tableName ...
limit (pageNum - 1) * pageSize, pageSize

```
# 2. 使用mybatis分页
```
CarMapper接口
List<Car> selectAllByPage(@Param("startIndex") Integer startIndex, @Param("pageSize") Integer pageSize);
CarMapper.xml
<select id="selectAllByPage" resultType="Car">
    select * from t_car limit #{startIndex},#{pageSize}
</select>

```
# 3. PageHelper插件
```
第一步：引入依赖
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper</artifactId>
    <version>5.3.2</version>
</dependency>

第二步：在mybatis-config.xml中配置插件（typeAliases配置之后）
<plugins>
    <plugin interceptor="com.github.pagehelper.PageInterceptor"></plugin>
</plugins>

第三步：编写Java代码
@Test
public void testPageHelper() throws Exception {
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(Resources.getResourceAsStream("mybatis-config.xml"));
    SqlSession sqlSession = sqlSessionFactory.openSession();
    CarMapper mapper = sqlSession.getMapper(CarMapper.class);

    // 开启分页
    PageHelper.startPage(2, 2);

    // 执行查询语句
    List<Car> cars = mapper.selectAll();

    // 获取分页信息对象
    PageInfo<Car> pageInfo = new PageInfo<>(cars, 5);

    System.out.println(pageInfo);
    
    /*
    PageInfo{pageNum=2, pageSize=2, size=2, startRow=3, endRow=4, total=15, pages=8,
    list=Page{count=true, pageNum=2, pageSize=2, startRow=2, endRow=4, total=15, pages=8, reasonable=false, pageSizeZero=false}
    [Car{id=4, carNum='1003', brand='丰田霸道', guidePrice=32.00, produceTime='2001-10-11', carType='燃油车'},
    Car{id=5, carNum='1003', brand='丰田霸道', guidePrice=33.00, produceTime='2002-10-11', carType='燃油车'}],
    prePage=1, nextPage=3, isFirstPage=false, isLastPage=false, hasPreviousPage=true, hasNextPage=true,
    navigatePages=5, navigateFirstPage=1, navigateLastPage=5, navigatepageNums=[1, 2, 3, 4, 5]}
    */
}

```
# 4. PageHelper 插件原理源码分析

PageHelper 是基于 MyBatis 拦截器（Interceptor）机制实现的分页插件。
它的核心逻辑在 PageInterceptor 中。

## 4.1 PageHelper 工作流程

1. 用户调用 PageHelper.startPage(pageNum, pageSize)
   → 将分页参数存入 ThreadLocal

2. 用户执行 mapper.selectAll()
   → 经过 MyBatis 的拦截器链
   → PageInterceptor.intercept() 被调用

3. PageInterceptor 拦截 Executor.query() 方法：
   a. 从 ThreadLocal 获取分页参数
   b. 获取 BoundSql 中的原始 SQL
   c. 执行 count 查询（select count(0) from (原SQL) tmp_count）
   d. 在原 SQL 后面拼接 LIMIT 子句
   e. 通过反射修改 BoundSql 中的 SQL
   f. 执行分页后的 SQL
   g. 将结果封装为 Page 对象

## 4.2 核心拦截链路

```java
/**
 * PageInterceptor 的核心拦截逻辑（简化版）
 *
 * 源码位置：com.github.pagehelper.PageInterceptor
 *
 * 这是理解所有 MyBatis 分页插件的基础
 */
@Intercepts({
    @Signature(
        type = Executor.class,
        method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
    ),
    @Signature(
        type = Executor.class,
        method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class}
    )
})
public class PageInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();
        MappedStatement ms = (MappedStatement) args[0];
        Object parameter = args[1];
        RowBounds rowBounds = (RowBounds) args[2];
        ResultHandler resultHandler = (ResultHandler) args[3];
        Executor executor = (Executor) invocation.getTarget();

        // ★ 1. 检查是否需要分页（从 ThreadLocal 获取分页参数）★
        Page page = getPageFromThreadLocal();
        if (page == null || page.getPageSize() <= 0) {
            // 不需要分页 → 直接执行原方法
            return invocation.proceed();
        }

        // ★ 2. 获取 BoundSql（最终可执行 SQL）★
        BoundSql boundSql = ms.getBoundSql(parameter);
        String originalSql = boundSql.getSql().trim();

        // ★ 3. 执行 count 查询获取总记录数 ★
        String countSql = "select count(0) from (" + originalSql + ") tmp_count";
        // 创建临时的 MappedStatement 执行 count 查询
        // ... 执行 countSql，获取 total

        // ★ 4. 构建分页 SQL ★
        String pageSql = buildPageSql(originalSql, page);

        // ★ 5. 通过反射修改 BoundSql 中的 SQL ★
        // 因为 BoundSql 中的 sql 字段是 final，需要通过反射修改
        MetaObject metaObject = SystemMetaObject.forObject(boundSql);
        metaObject.setValue("sql", pageSql);

        // ★ 6. 执行分页查询 ★
        List result = (List) invocation.proceed();

        // ★ 7. 封装分页结果 ★
        page.setTotal(total);
        page.addAll(result);
        return result;
    }

    /**
     * 构建分页 SQL（以 MySQL 为例）
     */
    private String buildPageSql(String originalSql, Page page) {
        StringBuilder sql = new StringBuilder(originalSql);
        sql.append(" LIMIT ");
        if (page.getStartRow() > 0) {
            sql.append(page.getStartRow()).append(", ");
        }
        sql.append(page.getPageSize());
        return sql.toString();
    }
}
```

## 4.3 ThreadLocal 在 PageHelper 中的使用

```java
/**
 * PageHelper 使用 ThreadLocal 存储当前线程的分页参数
 *
 * startPage() 方法将分页参数存入 ThreadLocal
 * PageInterceptor 从 ThreadLocal 中获取参数
 * 使用后必须清除（否则影响同一线程的下一次查询）
 */
public class PageHelper {

    // ★ 分页参数的 ThreadLocal ★
    protected static final ThreadLocal<Page> LOCAL_PAGE = new ThreadLocal<>();

    /**
     * 开始分页
     */
    public static <E> Page<E> startPage(int pageNum, int pageSize) {
        Page<E> page = new Page<>(pageNum, pageSize);
        LOCAL_PAGE.set(page);   // ★ 存入 ThreadLocal ★
        return page;
    }

    /**
     * 获取当前线程的分页参数（由 PageInterceptor 调用）
     */
    public static <T> Page<T> getLocalPage() {
        return LOCAL_PAGE.get();
    }

    /**
     * 清除分页参数
     * 必须在 PageInterceptor 中调用，否则会污染同一线程的后续查询
     */
    public static void clearPage() {
        LOCAL_PAGE.remove();
    }
}
```

## 4.4 PageHelper 核心要点总结

```
1. PageHelper 本质是一个 MyBatis 插件（Interceptor）
2. 拦截 Executor.query() 方法
3. 分页参数通过 ThreadLocal 传递（startPage → interceptor）
4. 通过反射修改 BoundSql 的 SQL 实现分页
5. 先执行 count 查询获取 total，再执行分页查询
6. 需要配合 PageInfo 使用获取完整的分页信息

对比手写 LIMIT 分页：
  手写：每次都要传 startIndex 和 pageSize 参数
  PageHelper：一行 startPage() 搞定，对原有代码零侵入
```
