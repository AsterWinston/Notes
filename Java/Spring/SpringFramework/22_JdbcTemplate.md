# 1. Spring JdbcTemplate 详解
JdbcTemplate是Spring提供的一个JDBC模板类，是对JDBC的封装，简化JDBC代码。
我们可以使用JdbcTemplate完成与数据库的相关操作
Spring也可以集成其它的ORM框架，例如：MyBatis、Hibernate等。
# 1. 环境准备
## 1.1 创建模块
## 1.2 引入依赖
```
<dependencies>
  <!-- spring 基础依赖 -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.0.8</version>
  </dependency>
  <!-- Spring jdbc 依赖（要使用spring的jdbcTemplate进行数据库操作） -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>6.0.8</version>
  </dependency>
  <!-- mysql驱动依赖（要进行MySQL数据库连接） -->
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.30</version>
  </dependency>
  <!-- 单元测试依赖 -->
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13.2</version>
    <scope>test</scope>
  </dependency>
  
</dependencies>
```
## 1.3 数据库表
```
CREATE DATABASE spring6;

USE spring6;

CREATE TABLE t_user
(
    id        INT PRIMARY KEY AUTO_INCREMENT,
    real_name VARCHAR(255),
    age       INT
);

INSERT INTO t_user
VALUES (NULL, '张三', 12),
       (NULL, '李四', 23);
```
## 1.4 数据库表对应的 Java 类
```
package cw.spring.study.pojo;

/**
 * ClassName: User
 * Package: cw.spring.study.pojo
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-26 18:04
 * @Version 1.0
 */
public class User {
    private Integer id;
    private String realName;
    private Integer age;
    
    public User() {
    }
    
    public User(Integer id, String realName, Integer age) {
        this.id = id;
        this.realName = realName;
        this.age = age;
    }
    
    @Override
    public String toString() {
        return "User{" + "id=" + id + ", realName='" + realName + '\'' + ", age=" + age + '}';
    }
    
    public Integer getId() {
        return id;
    }
    
    public void setId(Integer id) {
        this.id = id;
    }
    
    public String getRealName() {
        return realName;
    }
    
    public void setRealName(String realName) {
        this.realName = realName;
    }
    
    public Integer getAge() {
        return age;
    }
    
    public void setAge(Integer age) {
        this.age = age;
    }
}
```
## 1.5 数据源
```
package cw.spring.study.pojo;

import javax.sql.DataSource;
import java.io.PrintWriter;
import java.lang.management.ManagementFactory;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import java.sql.SQLFeatureNotSupportedException;
import java.util.logging.Logger;

/**
 * ClassName: MyDataSource
 * Package: cw.spring.study.pojo
 * Description:
 * 数据源，用于获取与数据库的连接对象
 * 只要是能够提供与数据库之间的连接对象的，就是数据源
 * 要写自己数据源就必须实现DataSource接口
 *
 * @Author tcw
 * @Create 2023-05-26 18:09
 * @Version 1.0
 */
public class MyDataSource implements DataSource {
    private String driver;
    private String url;
    private String username;
    private String password;
    
    public void setDriver(String driver) {
        this.driver = driver;
    }
    
    public void setUrl(String url) {
        this.url = url;
    }
    
    public void setUsername(String username) {
        this.username = username;
    }
    
    public void setPassword(String password) {
        this.password = password;
    }
    
    /**
     * 获取数据库的连接对象
     *
     * @return 数据库的连接对象
     * @throws SQLException 异常
     */
    @Override
    public Connection getConnection() throws SQLException {
        try {
            // 注册驱动
            Class.forName(driver);
            // 获取数据库连接对象
            return DriverManager.getConnection(url, username, password);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
    
    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        return null;
    }
    
    @Override
    public PrintWriter getLogWriter() throws SQLException {
        return null;
    }
    
    @Override
    public void setLogWriter(PrintWriter out) throws SQLException {
        
    }
    
    @Override
    public void setLoginTimeout(int seconds) throws SQLException {
        
    }
    
    @Override
    public int getLoginTimeout() throws SQLException {
        return 0;
    }
    
    @Override
    public Logger getParentLogger() throws SQLFeatureNotSupportedException {
        return null;
    }
    
    @Override
    public <T> T unwrap(Class<T> iface) throws SQLException {
        return null;
    }
    
    @Override
    public boolean isWrapperFor(Class<?> iface) throws SQLException {
        return false;
    }
}
```
## 1.6 spring 配置文件
```
要使用spring中的JdbcTemplate进行数据库操作，就需要一个JdbcTemplate对象
spring是对象的容器，可以在Spring配置文件中配置JdbcTemplate，由spring管理JdbcTemplate对象
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

	<!-- 数据源 -->
	<bean id="myDataSource" class="cw.spring.study.pojo.MyDataSource">
		<property name="driver" value="com.mysql.cj.jdbc.Driver"/>
		<property name="url" value="jdbc:mysql://127.0.0.1:3306/spring6"/>
		<property name="username" value="root"/>
		<property name="password" value="123123"/>
	</bean>
	
	<!-- JdbcTemplate 要进行数据库操作，就需要一个数据源，用于获取与数据库的连接对象 -->
	<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
		<!-- 配置 JdbcTemplate 的数据源 -->
		<property name="dataSource" ref="myDataSource"/>
	</bean>
	
</beans>
```
## 1.7 测试
```
@org.junit.Test
public void testJDBC() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    JdbcTemplate jdbcTemplate = applicationContext.getBean("jdbcTemplate", JdbcTemplate.class);
    System.out.println(jdbcTemplate);
}

```

# 2. 新增数据
```
@org.junit.Test
public void testInsert() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    JdbcTemplate jdbcTemplate = applicationContext.getBean("jdbcTemplate", JdbcTemplate.class);
    // SQL
    String sql = "insert into t_user (real_name, age) values (?, ?);";
    // 在 JdbcTemplate 中执行增删改，都调用 update 方法
    // 第一个参数为 SQL
    // 第二个参数为 可变形参，向SQL中占位符填充的数据
    // 返回值为 影响数据库表数据的条数
    int count = jdbcTemplate.update(sql, "赵六", 45);
    System.out.println(count);
}

```

# 3. 修改数据
```
@org.junit.Test
public void testUpdate() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    JdbcTemplate jdbcTemplate = applicationContext.getBean("jdbcTemplate", JdbcTemplate.class);
    String sql = "update t_user set real_name = ?, age = ? where id = ?";
    int count = jdbcTemplate.update(sql, "tom", 33, 10);
    System.out.println(count);
}

```

# 4. 删除数据
```
@org.junit.Test
public void testDelete () {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    JdbcTemplate jdbcTemplate = applicationContext.getBean("jdbcTemplate", JdbcTemplate.class);
    String sql = "delete from t_user where id = ?";
    int count = jdbcTemplate.update(sql, 10);
    System.out.println(count);
}

```

# 5. 查询一个对象
```
Spring 中的 JDBC 可以实现数据库表字段名到Java对象属性名的映射（要符合命名规范）
@org.junit.Test
public void testSelectOneObject() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    JdbcTemplate jdbcTemplate = applicationContext.getBean("jdbcTemplate", JdbcTemplate.class);
    // SQL
    String sql = "select id, real_name, age from t_user where id = ?";
    // 执行SQL
    // 参数一：需要执行的SQL语句
    // 参数二：查询出来的数据记录要映射成的对象类型
    // 参数三：可变参数，填充SQL占位符的数据
    User user = jdbcTemplate.queryForObject(sql, new BeanPropertyRowMapper<>(User.class), 1);
    System.out.println(user);
}

```

# 6. 查询多个对象
```
@org.junit.Test
public void testSelectAll() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    JdbcTemplate jdbcTemplate = applicationContext.getBean("jdbcTemplate", JdbcTemplate.class);
    // SQL
    String sql = "select * from t_user;";
    // 执行SQL
    // 参数一：SQL
    // 参数二：查询出来的数据记录要映射成的对象类型
    // 参数三：可变参数，填充SQL占位符的数据
    List<User> users = jdbcTemplate.query(sql, new BeanPropertyRowMapper<>(User.class));
    users.forEach(System.out::println);
}

```

# 7. 查一个值
```
@org.junit.Test
public void testSelectOneValue() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    JdbcTemplate jdbcTemplate = applicationContext.getBean("jdbcTemplate", JdbcTemplate.class);
    String sql = "select count(*) from t_user";
    // 参数一：SQL
    // 参数二：返回值的类型
    Integer count = jdbcTemplate.queryForObject(sql, int.class);
    System.out.println("记录总条数为：" + count);
}

```

# 8. 批量添加
```
@org.junit.Test
public void testBatchInsert() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    JdbcTemplate jdbcTemplate = applicationContext.getBean("jdbcTemplate", JdbcTemplate.class);
    String sql = "insert into t_user(real_name, age) values(?, ?)";

    // 准备数据
    // 每条记录的数据封装在一个Object数组中，然后将Object数组封装在List集合中
    Object[] user1 = {"tom", 23};
    Object[] user2 = {"tom", 23};
    Object[] user3 = {"tom", 23};
    // 封装数据
    ArrayList<Object[]> objects = new ArrayList<>();
    objects.add(user1);
    objects.add(user2);
    objects.add(user3);

    // 进行批量插入，调用batchUpdate方法
    // 返回值为每条记录插入后影响数据库记录条数组成的数组
    int[] count = jdbcTemplate.batchUpdate(sql, objects);
    System.out.println(Arrays.toString(count));
}

```

# 9. 批量更新
```
@org.junit.Test
public void testBatchUpdate() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    JdbcTemplate jdbcTemplate = applicationContext.getBean("jdbcTemplate", JdbcTemplate.class);
    String sql = "update t_user set real_name = ?, age = ? where id = ?";

    // 准备数据
    // real_name age id
    Object[] user1 = {"mary1", 12, 1};
    Object[] user2 = {"mary2", 13, 2};
    Object[] user3 = {"mary3", 14, 3};
    // 封装数据
    ArrayList<Object[]> objects = new ArrayList<>();
    objects.add(user1);
    objects.add(user2);
    objects.add(user3);
    
    // 批量更新调用batchUpdate方法
    // 返回每条记录修改后影响数据库记录条数组成的数组
    int[] counts = jdbcTemplate.batchUpdate(sql, objects);
    System.out.println(Arrays.toString(counts));
}

```

# 10. 批量删除
```
@org.junit.Test
public void testBatchDelete() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    JdbcTemplate jdbcTemplate = applicationContext.getBean("jdbcTemplate", JdbcTemplate.class);
    String sql = "delete from t_user where id = ?";

    // 准备数据
    // id
    // 每次只有一个数据，也要使用数组
    Object[] id1 = {11};
    Object[] id2 = {12};
    Object[] id3 = {13};
    // 封装数据
    ArrayList<Object[]> list = new ArrayList<>();
    list.add(id1);
    list.add(id2);
    list.add(id3);

    int[] counts = jdbcTemplate.batchUpdate(sql, list);
    System.out.println(Arrays.toString(counts));
}

```

# 11. 回调函数
```
如果要写JDBC代码，可以使用回调函数
@org.junit.Test
public void test10() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    JdbcTemplate jdbcTemplate = applicationContext.getBean("jdbcTemplate", JdbcTemplate.class);
    String sql = "select id, real_name, age from t_user where id = ?";
    // 执行SQL语句，使用回调函数执行SQL语句
    // 注册回调函数，当该execute方法执行时，回调函数中的doInPreparedStatement方法会被调用
    // 返回值为回调函数中的返回值
    User user = jdbcTemplate.execute(sql, new PreparedStatementCallback<User>() {
        @Override
        public User doInPreparedStatement(PreparedStatement ps) throws SQLException, DataAccessException {
            // doInPreparedStatement 方法体中的为JDBC代码
            // 填充占位符
            ps.setInt(1, 1);
            // 执行SQL语句获取结果集
            ResultSet resultSet = ps.executeQuery();
            User user = null;
            // 读取结果集中的数据
            if (resultSet.next()) {
                int id = resultSet.getInt(1);
                String realName = resultSet.getString(2);
                int age = resultSet.getInt(3);
                user = new User(id, realName, age);
            }
            return user;
        }
    });
    System.out.println(user);
}

```

# 12. 整合德鲁伊连接池
```
引入德鲁伊连接池的依赖
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>druid</artifactId>
  <version>1.1.8</version>
</dependency>

将德鲁伊连接池数据源配置到spring配置文件中，并更新JdbcTemplate使用的数据源
<bean id="druidDataSource" class="com.alibaba.druid.pool.DruidDataSource">
  <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
  <property name="url" value="jdbc:mysql://127.0.0.1:3306/spring6"/>
  <property name="username" value="root"/>
  <property name="password" value="285013"/>
</bean>
<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
  <property name="dataSource" ref="druidDataSource"/>
</bean>

测试
@org.junit.Test
public void test06() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    JdbcTemplate jdbcTemplate = applicationContext.getBean("jdbcTemplate", JdbcTemplate.class);
    String sql = "select * from t_user";
    // 查询多条记录调用 query 方法
    List<User> userList = jdbcTemplate.query(sql, new BeanPropertyRowMapper<>(User.class));
    userList.forEach(user -> System.out.println(user));
}
```

# 13. 源码解析：JdbcTemplate 执行链路与连接管理

## 13.1 JdbcTemplate 的核心架构

```
JdbcTemplate 是 Spring JDBC 的核心类，它封装了所有样板代码：

JdbcTemplate 的设计模式：
  ┌────────────────────────────────────────┐
  │        JdbcTemplate（门面）            │
  │  query() / update() / execute() / ...  │
  └────────────────┬───────────────────────┘
                   │ 委托
                   ▼
  ┌────────────────────────────────────────┐
  │        JdbcOperations（接口）           │
  │  定义了所有 JDBC 操作的契约            │
  └────────────────┬───────────────────────┘
                   │ 委托
                   ▼
  ┌────────────────────────────────────────┐
  │        execute() — 核心执行引擎         │
  │  所有 query() / update() 最终都委托给  │
  │   execute() 方法执行                   │
  └────────────────┬───────────────────────┘
                   │ 通过
                   ▼
  ┌────────────────────────────────────────┐
  │        DataSource                      │
  │  获取 Connection                       │
  └────────────────────────────────────────┘
```

## 13.2 execute() — 所有操作的统一入口

```
JdbcTemplate.execute() 是所有 JDBC 操作的底层统一入口：

@Nullable
private <T> T execute(PreparedStatementCreator psc, PreparedStatementCallback<T> action,
        boolean closeResources) throws DataAccessException {

    Assert.notNull(psc, "PreparedStatementCreator must not be null");
    Assert.notNull(action, "Callback object must not be null");

    // ★ 步骤 1: 获取 Connection
    //    如果当前线程绑定了事务 Connection，返回事务连接
    //    否则从 DataSource 获取新连接
    Connection con = DataSourceUtils.getConnection(obtainDataSource());

    PreparedStatement ps = null;
    try {
        // ★ 步骤 2: 创建 PreparedStatement（由传入的 psc 创建）
        //    PreparedStatementCreator 是一个函数式接口：Connection → PreparedStatement
        ps = psc.createPreparedStatement(con);

        // ★ 步骤 3: 应用 JDBC 语句设置（fetchSize, maxRows, queryTimeout）
        applyStatementSettings(ps);

        // ★ 步骤 4: 执行回调（核心业务逻辑）
        //    PreparedStatementCallback 是一个函数式接口：PreparedStatement → T
        T result = action.doInPreparedStatement(ps);

        // ★ 步骤 5: 处理可能的警告
        handleWarnings(ps);
        return result;
    } catch (SQLException ex) {
        // ★ 步骤 6: 异常转换
        //    将 JDBC SQLException 转换为 Spring 的 DataAccessException
        //    如果当前连接的 DataSource 有 PersistenceExceptionTranslator
        //    （如 HibernateException、JpaDialect），会进一步转换为 JPA/Hibernate 异常
        if (psc instanceof ParameterDisposer) {
            ((ParameterDisposer) psc).cleanupParameters();
        }
        String sql = getSql(psc);
        psc = null;
        JdbcUtils.closeStatement(ps);
        ps = null;
        DataSourceUtils.releaseConnection(con, getDataSource());
        con = null;
        throw translateException("PreparedStatementCallback", sql, ex);
    } finally {
        // ★ 步骤 7: 资源清理
        if (closeResources) {
            if (psc instanceof ParameterDisposer) {
                ((ParameterDisposer) psc).cleanupParameters();
            }
            JdbcUtils.closeStatement(ps);
            // ★ 注意：如果没有事务，releaseConnection 会关闭连接
            //        如果当前有事务，releaseConnection 会将连接归还给事务管理器管理
            DataSourceUtils.releaseConnection(con, getDataSource());
        }
    }
}
```

## 13.3 query() 方法的执行链路

```
以 query(sql, RowMapper) 为例，追踪完整执行链路：

public <T> List<T> query(String sql, RowMapper<T> rowMapper) throws DataAccessException {
    // 1. 调用内部的 query 方法，将 RowMapper 包装为 RowMapperResultSetExtractor
    return result(query(sql, new RowMapperResultSetExtractor<>(rowMapper)));
}

public <T> T query(String sql, ResultSetExtractor<T> rse) throws DataAccessException {
    // 2. 通过 PreparedStatementCreatorFactory 创建 PreparedStatementCreator
    return query(new SimplePreparedStatementCreator(sql), rse);
}

public <T> T query(
        final PreparedStatementCreator psc, final ResultSetExtractor<T> rse)
        throws DataAccessException {

    // 3. ★ 调用 execute()
    //    psc (PreparedStatementCreator): 创建 PreparedStatement
    //    action (PreparedStatementCallback): 执行 SQL 并处理 ResultSet
    return execute(psc, new PreparedStatementCallback<T>() {
        @Override
        @Nullable
        public T doInPreparedStatement(PreparedStatement ps) throws SQLException {
            ResultSet rs = null;
            try {
                if (ps.execute()) {  // ★ 执行 SQL
                    rs = ps.getResultSet();  // 获取结果集
                } else {
                    rs = null;
                }
                // ★ 调用 ResultSetExtractor 将 ResultSet 转换为目标对象
                return rse.extractData(rs);
            } finally {
                JdbcUtils.closeResultSet(rs); // ★ 关闭 ResultSet
            }
        }
    });
}
```

## 13.4 update() 方法的执行链路

```
public int update(final String sql) throws DataAccessException {
    Assert.notNull(sql, "SQL must not be null");
    // ★ 调用 execute()，使用 SimplePreparedStatementCreator
    return execute(new SimplePreparedStatementCreator(sql),
            (PreparedStatementCallback<Integer>) ps -> ps.executeUpdate());
}

// 带参数的 update:
public int update(String sql, @Nullable Object... args) throws DataAccessException {
    return update(sql, newArgPreparedStatementSetter(args));
}

public int update(String sql, PreparedStatementSetter pss) throws DataAccessException {
    return update(new SimplePreparedStatementCreator(sql), pss);
}

protected int update(
        final PreparedStatementCreator psc, @Nullable final PreparedStatementSetter pss)
        throws DataAccessException {

    // ★ execute() + 先设置参数，再 executeUpdate
    return execute(psc, ps -> {
        try {
            if (pss != null) {
                pss.setValues(ps);  // ★ 设置占位符参数
            }
            int rows = ps.executeUpdate(); // ★ 执行更新
            return rows;  // 返回影响行数
        } finally {
            if (pss instanceof ParameterDisposer) {
                ((ParameterDisposer) pss).cleanupParameters();
            }
        }
    });
}
```

## 13.5 DataSourceUtils.getConnection() — 连接获取策略

```
DataSourceUtils 是 Spring JDBC 中的连接管理工具类，负责处理连接获取、事务感知和释放：

public static Connection getConnection(DataSource dataSource) throws CannotGetJdbcConnectionException {
    try {
        return doGetConnection(dataSource);
    } catch (SQLException ex) {
        throw new CannotGetJdbcConnectionException("Failed to obtain JDBC Connection", ex);
    } catch (IllegalStateException ex) {
        throw new CannotGetJdbcConnectionException(
                "Failed to obtain JDBC Connection: " + ex.getMessage());
    }
}

public static Connection doGetConnection(DataSource dataSource) throws SQLException {
    Assert.notNull(dataSource, "No DataSource specified");

    // ★ 关键：检查当前线程是否绑定了事务连接
    ConnectionHolder conHolder =
            (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);

    if (conHolder != null && conHolder.hasConnection()) {
        // ★ 如果有事务，返回事务绑定的连接（同一事务内所有操作共享这个连接）
        if (conHolder.isSynchronizedWithTransaction()) {
            conHolder.requested();
            return conHolder.getConnection();
        }
        // 如果不是事务同步的，释放旧连接
        releaseConnection(conHolder.getConnection(), dataSource);
        conHolder = null;
    }

    // ★ 没有事务：直接从 DataSource 获取新连接
    Connection con = fetchConnection(dataSource);
    // 包装到 ConnectionHolder
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
        ConnectionHolder holderToUse = conHolder;
        if (holderToUse == null) {
            holderToUse = new ConnectionHolder(con);
        }
        holderToUse.requested();
        TransactionSynchronizationManager.registerSynchronization(
                new ConnectionSynchronization(holderToUse, dataSource));
        holderToUse.setSynchronizedWithTransaction(true);
        if (holderToUse != conHolder) {
            TransactionSynchronizationManager.bindResource(dataSource, holderToUse);
        }
    }
    return con;
}
```

## 13.6 DataSourceUtils.releaseConnection() — 连接释放策略

```
public static void releaseConnection(
        @Nullable Connection con, @Nullable DataSource dataSource) {

    if (con == null || dataSource == null) {
        return;
    }

    try {
        // ★ 关键：如果有事务绑定，不关闭连接，只减少引用计数
        //        连接会在事务提交或回滚时由事务管理器统一处理
        //        如果没有事务绑定，直接关闭连接或归还连接池
        doReleaseConnection(con, dataSource);
    } catch (SQLException ex) {
        logger.debug("Could not close JDBC Connection", ex);
    } catch (RuntimeException ex) {
        logger.debug("Unexpected exception on closing JDBC Connection", ex);
    }
}

public static void doReleaseConnection(
        @Nullable Connection con, @Nullable DataSource dataSource) throws SQLException {

    if (con == null) {
        return;
    }
    if (dataSource != null) {
        ConnectionHolder conHolder =
                (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
        if (conHolder != null && connectionEquals(conHolder, con)) {
            // ★ 事务同步激活中：只是标记为 released，不真正关闭
            conHolder.released();
            return;
        }
    }
    // ★ 没有事务：关闭连接
    doCloseConnection(con, dataSource);
}

public static void doCloseConnection(
        Connection con, @Nullable DataSource dataSource) throws SQLException {

    // 如果有事务但连接已经不在同步状态，先解绑
    if (!(dataSource instanceof TransactionAwareDataSourceProxy)
            && isConnectionTransactional(con, dataSource)) {
        TransactionSynchronizationManager.unbindResource(dataSource);
    }
    // ★ 真正关闭连接（如果使用连接池，会归还到池中）
    JdbcUtils.closeConnection(con);
}
```

## 13.7 BeanPropertyRowMapper — 结果集到 Bean 的映射

```
BeanPropertyRowMapper 负责将 ResultSet 的一行转换为 Bean：

public class BeanPropertyRowMapper<T> implements RowMapper<T> {

    @Override
    public T mapRow(ResultSet rs, int rowNumber) throws SQLException {
        // 1. 创建 Bean 实例（通过无参构造）
        T mappedObject = BeanUtils.instantiateClass(this.mappedClass);
        BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(mappedObject);

        // 2. 遍历 ResultSet 的所有字段
        for (int i = 0; i < rs.getMetaData().getColumnCount(); i++) {
            String column = JdbcUtils.lookupColumnName(rs.getMetaData(), i + 1);
            // 将数据库字段名（下划线）转换为 Java 属性名（驼峰）
            String propertyName = JdbcUtils.convertUnderscoreNameToPropertyName(column);
            // 查找 Bean 中是否有匹配的属性
            PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
            if (pd != null) {
                // ★ 通过 BeanWrapper 设置属性值（内部使用反射调用 setter）
                Object value = getColumnValue(rs, i + 1, pd);
                bw.setPropertyValue(propertyName, value);
            }
        }
        return mappedObject;
    }

    // 列值 → Java 类型转换
    protected Object getColumnValue(ResultSet rs, int index, PropertyDescriptor pd)
            throws SQLException {
        // Spring 内部使用 TypeConverter 进行类型转换
        // 如 VARCHAR → Integer、TIMESTAMP → Date 等
        return JdbcUtils.getResultSetValue(rs, index, pd.getPropertyType());
    }
}
```

## 13.8 JdbcTemplate 事务感知完整流程图

```
业务方法 @Transactional
    │
    └─ TransactionInterceptor（事务开启）
         │
         ├─ doBegin() → DataSource.getConnection()
         │              → con.setAutoCommit(false)
         │              → TransactionSynchronizationManager.bindResource(ds, conHolder)
         │              ★ 连接绑定到当前线程
         │
         └─ 业务方法执行
              │
              ├─ jdbcTemplate.query(sql, rowMapper)
              │    │
              │    ├─ execute()
              │    │    │
              │    │    ├─ DataSourceUtils.getConnection(ds)
              │    │    │   └→ TransactionSynchronizationManager.getResource(ds)
              │    │    │       └→ ★ 返回事务绑定的同一个 Connection
              │    │    │
              │    │    ├─ psc.createPreparedStatement(con)
              │    │    │   └→ con.prepareStatement(sql)
              │    │    │       ★ 使用事务连接创建 Statement
              │    │    │
              │    │    ├─ ps.execute()  ← 执行 SQL
              │    │    └─ DataSourceUtils.releaseConnection(con, ds)
              │    │         └→ ★ 事务上下文中，不关闭连接，只减少引用计数
              │    │
              │    └─ 返回查询结果
              │
              ├─ jdbcTemplate.update(sql, args)
              │    │
              │    ├─ execute()
              │    │    └─ DataSourceUtils.getConnection(ds)
              │    │        └→ ★ 同样返回事务绑定的连接
              │    └─ 返回影响行数
              │
         ─ TransactionInterceptor（提交或回滚）
              │
              ├─ 正常 → doCommit() → con.commit()
              └─ 异常 → doRollback() → con.rollback()
                  │
                  └─ cleanupAfterCompletion()
                       ├─ TransactionSynchronizationManager.unbindResource(ds)
                       └─ DataSourceUtils.releaseConnection(con, ds)
                            └→ con.close() (归还连接池)
```

## 13.9 设计思想总结

1. 模板方法模式（Template Method Pattern）：
   JdbcTemplate 是命名的来源，它将 JDBC 的固定流程（获取连接 → 创建 Statement →
   执行 SQL → 处理 ResultSet → 释放资源）作为模板，变化部分（SQL 语句、参数设置、
   结果映射）通过回调函数（PreparedStatementCallback、RowMapper）传递。
   
   这是 Spring 中最经典的模板方法模式应用，职责清晰：
   - JdbcTemplate: 负责流程控制 + 资源管理 + 异常转换
   - Callback: 负责业务特定的 JDBC 操作
   - RowMapper: 负责结果映射

2. 策略模式 + 函数式接口：
   PreparedStatementCreator: 策略——如何创建 Statement
   PreparedStatementSetter: 策略——如何设置参数
   ResultSetExtractor: 策略——如何从 ResultSet 提取结果
   RowMapper: 策略——如何将一行映射为对象

3. 事务感知连接管理：
   DataSourceUtils 通过 TransactionSynchronizationManager 实现了连接的事务感知：
   - 有事务：共享同一个 Connection（确保 ACID）
   - 无事务：每次操作自动获取/释放连接

4. Lambda 与函数式接口的结合：
   Spring 5+ 的 JdbcTemplate 大量使用函数式接口，支持 Lambda 表达式，
   如 jdbcTemplate.query(sql, (rs, rowNum) -> new User(...))
   这是从传统内部类向现代 Java 编程风格的演进。
