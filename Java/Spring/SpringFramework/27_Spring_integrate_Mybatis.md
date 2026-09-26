# 1. 实现步骤
第一步：准备数据库表
    使用t_act表（账户表）
第二步：IDEA中创建一个模块，并引入依赖
    spring-context
        会关联引入aop依赖，spring-aop
    spring-jdbc
        会关联引入事务依赖，spring-tx
    mysql驱动
    mybatis
    mybatis-spring：mybatis提供的与spring框架集成的依赖
    德鲁伊连接池
    junit
第三步：基于三层架构实现，所以提前创建好所有的包
    com.powernode.bank.mapper
		在Mybatis中习惯将dao写成mapper，并且不需要写实现类，因为Mybatis会通过动态代理机制自动给我们生成实现类。
    com.powernode.bank.service
    com.powernode.bank.service.impl
    com.powernode.bank.pojo
第四步：编写pojo
    Account，属性私有化，提供公开的setter getter和toString。
第五步：编写mapper接口
    AccountMapper接口，定义方法
第六步：编写mapper配置文件
	在配置文件中配置命名空间，以及每一个方法对应的sql。
第七步：编写service接口和service接口实现类
    AccountService
    AccountServiceImpl
第八步：编写jdbc.properties配置文件
    数据库连接池相关信息
第九步：编写mybatis-config.xml配置文件
	该文件可以没有，大部分的配置可以转移到spring配置文件中。
	如果遇到mybatis相关的系统级配置，还是需要这个文件。如：缓存
	Spring集成Mybatis之后，可以将Mybatis的核心配置文件中的大部分的配置转移到Spring配置文件中，所以可以没有Mybatis的核心配置文件，如果遇到mybatis相关的系统级配置，还是需要这个文件。所以只要不进行系统级参数的配置，Mybatis配置文件可以省略。
第十步：编写spring.xml配置文件
    组件扫描
    引入外部的属性文件
    数据源（德鲁伊连接池）
    SqlSessionFactoryBean配置
        注入mybatis核心配置文件路径
        指定别名包
        注入数据源
    Mapper扫描配置器
        指定扫描的包
    事务管理器DataSourceTransactionManager
        注入数据源
    启用事务注解
        注入事务管理器
第十一步：编写测试程序，并添加事务，进行测试

# 2. 数据库表
```
USE spring6;

DROP TABLE IF EXISTS t_act;

CREATE TABLE t_act (
    id int PRIMARY KEY AUTO_INCREMENT,
    actno VARCHAR(255),
    balance DECIMAL(10, 2)
);

INSERT INTO t_act(actno, balance) VALUES ('act001', 50000);
INSERT INTO t_act(actno, balance) VALUES ('act002', 0);

```

# 3. 创建项目引入依赖
```
spring-context：Spring 基础核心依赖，会关联引入 aop 依赖
spring-jdbc：Spring jdbc 依赖，会关联引入事务的依赖
mysql驱动
mybatis
mybatis-spring：mybatis提供的与spring框架集成的依赖
德鲁伊连接池
junit

<dependencies>
  <!-- spring-context：Spring 基础核心依赖，会关联引入 aop 依赖 spring-aop -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.0.8</version>
  </dependency>
  <!-- spring-jdbc：Spring jdbc 依赖，会关联引入事务的依赖 spring-tx，JdbcTemplate和事务管理器DataSourceTransactionManager都在这个包里 -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>6.0.8</version>
  </dependency>
  <!-- mysql驱动 -->
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.30</version>
  </dependency>
  <!-- mybatis -->
  <dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.10</version>
  </dependency>
  <!-- mybatis-spring：mybatis提供的与spring框架集成的依赖 -->
  <dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis-spring</artifactId>
    <version>2.0.7</version>
  </dependency>
  <!-- 德鲁伊连接池 -->
  <dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.12</version>
  </dependency>
  <!-- junit -->
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13.2</version>
    <scope>test</scope>
  </dependency>
</dependencies>

```

# 4. 创建软件包

# 5. 实体类
```
package cw.study.spring.pojo;

/**
 * ClassName: Account
 * Package: cw.study.spring.pojo
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-28 9:27
 * @Version 1.0
 */
public class Account {
    private String actno;
    private Double balance;
    
    public Account() {
    }
    
    public Account(String actno, Double balance) {
        this.actno = actno;
        this.balance = balance;
    }
    
    @Override
    public String toString() {
        return "Account{" + "actno='" + actno + '\'' + ", balance=" + balance + '}';
    }
    
    public String getActno() {
        return actno;
    }
    
    public void setActno(String actno) {
        this.actno = actno;
    }
    
    public Double getBalance() {
        return balance;
    }
    
    public void setBalance(Double balance) {
        this.balance = balance;
    }
}
```

# 6. 编写Mapper接口
```
package cw.study.spring.mapper;

import cw.study.spring.pojo.Account;

import java.util.List;

/**
 * ClassName: AccountMapper
 * Package: cw.study.spring.mapper
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-05 12:50
 * @Version 1.0
 */
public interface AccountMapper {
    
    /**
     * 新增账户信息
     * 
     * @param account 账户信息
     * @return 影响数据库条数
     */
    int insert(Account account);
    
    /**
     * 根据账号删除账户信息
     * 
     * @param actno 账号
     * @return 影响数据库条数
     */
    int deleteByActno(String actno);
    
    /**
     * 更新账户信息
     * 
     * @param account 账户信息
     * @return 影响数据库条数
     */
    int update(Account account);
    
    /**
     * 根据账号查询账户信息
     * 
     * @param actno 账号
     * @return 账户信息
     */
    Account selectByActno(String actno);
    
    /**
     * 查询所有账户信息
     *  
     * @return 所有账户信息
     */
    List<Account> selectAll();
    
}
```
# 7. 编写Mapper配置文件
```
Mapper 配置文件的所在目录位置和 Mapper 接口的所在目录位置要一致，并且配置文件的文件名要接口的名称一样
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
		PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
		"http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cw.study.spring.mapper.AccountMapper">
	<insert id="insert">
		insert into t_act values(#{actno}, #{balance})
	</insert>
	<delete id="deleteByActno">
		delete from t_act where actno = #{actno}
	</delete>
	<update id="update">
		update t_act set balance = #{balance} where actno = #{actno}
	</update>
	<select id="selectByActno" resultType="Account">
		select * from t_act where actno = #{actno}
	</select>
	<select id="selectAll" resultType="Account">
		select * from t_act
	</select>
</mapper>
```
# 8. 编写service接口和service接口实现类
```
package cw.study.spring.service;

import cw.study.spring.pojo.Account;

import java.util.List;

/**
 * ClassName: AccountService
 * Package: cw.study.spring.service
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-05 13:00
 * @Version 1.0
 */
public interface AccountService {
    /**
     * 开户，新增账户
     * @param act 账户信息
     * @return
     */
    int save(Account act);
    
    /**
     * 根据账号销户
     * @param actno 账户信息
     * @return
     */
    int deleteByActno(String actno);
    
    /**
     * 修改账户
     * @param act 账户信息
     * @return
     */
    int update(Account act);
    
    /**
     * 根据账号获取账户
     * @param actno 账号
     * @return 账户信息
     */
    Account getByActno(String actno);
    
    /**
     * 获取所有账户
     * @return 账户信息组成的集合
     */
    List<Account> getAll();
    
    /**
     * 转账
     * @param fromActno 转出账号
     * @param toActno 转入账号
     * @param money 转账金额
     */
    void transfer(String fromActno, String toActno, double money);
}

package cw.study.spring.service.impl;

import cw.study.spring.mapper.AccountMapper;
import cw.study.spring.pojo.Account;
import cw.study.spring.service.AccountService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;

/**
 * ClassName: AccountServiceImpl
 * Package: cw.study.spring.service.impl
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-05 13:03
 * @Version 1.0
 */
@Service("accountService") // 纳入spring容器管理
public class AccountServiceImpl implements AccountService {
    
    @Autowired // 根据类型自动注入
    private AccountMapper accountMapper;
    
    @Override
    public int save(Account act) {
        return accountMapper.insert(act);
    }
    
    @Override
    public int deleteByActno(String actno) {
        return accountMapper.deleteByActno(actno);
    }
    
    @Override
    public int update(Account act) {
        return accountMapper.update(act);
    }
    
    @Override
    public Account getByActno(String actno) {
        return accountMapper.selectByActno(actno);
    }
    
    @Override
    public List<Account> getAll() {
        return accountMapper.selectAll();
    }
    
    @Override
    public void transfer(String fromActno, String toActno, double money) {
        Account fromAct = accountMapper.selectByActno(fromActno);
        if (fromAct.getBalance() < money) {
            throw new RuntimeException("余额不足");
        }
        // 内存中更新账户
        Account toAct = accountMapper.selectByActno(toActno);
        fromAct.setBalance(fromAct.getBalance() - money);
        toAct.setBalance(toAct.getBalance() + money);
        // 更新数据库
        int count = accountMapper.update(fromAct);
        count += accountMapper.update(toAct);
        if (count != 2) {
            throw new RuntimeException("转账失败");
        }
    }
}
```

# 9. 编写jdbc.properties
```
放在类的根路径下
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/spring6
jdbc.username=root
jdbc.password=123123
```
# 10. 编写 mybatis-config.xml 配置文件
```
放在类的根路径下，只开启日志，其他配置到spring.xml中。
连接数据库的信息、包扫描等在spring配置文件中进行配置，在mybatis核心配置文件中进行mybatis相关的系统级配置

<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
		PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
		"http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
	<settings>
		<!-- 使用Mybatis的标准日志，进行日志信息，SQL语句的打印 -->
		<setting name="logImpl" value="STDOUT_LOGGING"/>
	</settings>
</configuration>

因为在spring配置文件中SqlSessionFactoryBean的配置中没办法整合懒加载，我们可以继续在mybatis核心配置文件中配置懒加载。
<!--懒加载配置-->
<settings>
    <setting name="lazyLoadingEnabled" value="true"/> <!--总开关-->
    <setting name="aggressiveLazyLoading" value="false"/> <!--不然任何方法都触发-->
    <setting name="lazyLoadTriggerMethods" value="clone"/> <!--clone方法触发-->
</settings>
然后在SqlSessionFactoryBean中添加配置引入mybatis的核心配置文件
<!--方案一: 通过加载mybatis 主配置文件, 达到配置懒加载的目的-->
<property name="configLocation" value="classpath:mybatis-config.xml"/>（spring.xml的SqlSessionFactoryBean中）

当然还有方案二：不在mybatis核心文件中配置。在匹配Spring配置文件中先创建一个配置对象。
<bean id="configuration" class="org.apache.ibatis.session.Configuration">
    <property name="lazyLoadingEnabled" value="true"/>
    <property name="aggressiveLazyLoading" value="false"/>
    <property name="lazyLoadTriggerMethods" value="clone"/>
</bean>
然后在SqlSessionFactoryBean中引用该配置对象。
<!--方案二: 通过配置对象, 达到配置懒加载的目的-->
<property name="configuration" ref="configuration"/>（spring.xml的SqlSessionFactoryBean中）

注意：两种方案只能二选一

整合时如何如使用分页插件？除了在mybatis核心配置文件中配置，还可以在SqlSessionFactoryBean中配置。
<property name="plugins">
    <array>
        <bean class="com.github.pagehelper.PageInterceptor">
            <property name="properties">
                <!--使用下面的方式配置参数, 一行配置一个 -->
                <value>
                    helperDialect=mysql
                    pageSizeZero=true
                    reasonable=true
                </value>
            </property>
        </bean>
    </array>
</property>
```

# 11. 编写spring.xml配置文件
```
注意：当你在spring.xml文件中直接写标签内容时，IDEA会自动给你添加命名空间
组件扫描
引入外部的属性文件
    数据源（德鲁伊连接池）
    SqlSessionFactoryBean配置
    注入mybatis核心配置文件路径
    指定别名包
    注入数据源
    配置 SqlSessionFactoryBean相当于mybatis的核心配置文件，那为啥里面没有指定mapper.xml配置文件的位置呢，其实是可以指定的。但是为因为我们Spring中要管理自动给我们生成mapper接口的代理对象（不用我们自己写实现类），就需要配置mapper扫描器，然后因为我们又遵循了规范，mapper.xml文件跟mapper接口就是在同一目录下的，所以spring在扫描mapper接口的时候，mapper.xml文件也连同被一起扫描到了。因此在配置 SqlSessionFactoryBean时就不需要指定mapper.xml配置文件的位置了(老杜原版笔记评论)
Mapper扫描配置器
	指定扫描的包
事务管理器DataSourceTransactionManager
	注入数据源
启用事务注解
	注入事务管理器

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context" xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd">

	<!-- 组件扫描 -->
	<context:component-scan base-package="cw.study.spring"/>
	
	<!-- 引入外部的属性配置文件，加载连接数据库的配置信息 -->
	<context:property-placeholder location="jdbc.properties"/>
	
	<!-- 数据库源 -->
	<bean id="druidDataSource" class="com.alibaba.druid.pool.DruidDataSource">
		<property name="driverClassName" value="${jdbc.driver}"/>
		<property name="url" value="${jdbc.url}"/>
		<property name="username" value="${jdbc.username}"/>
		<property name="password" value="${jdbc.password}"/>
	</bean>
	
	<!-- 配置SqlSessionFactoryBean，获取SqlSession的对象 -->
	<!--
		相当于Mybatis中配置数据库环境的配置转变为该配置，
		在spring容器中注册SqlSessionFactory对象
		一个数据库环境对应一个SqlSessionFactory对象
		由于使用Mapper接口自动代理，底层生成的代理类中会需要SqlSessionFactory对象
		获取SqlSession对象执行数据库操作
		我们不需要手动使用SqlSessionFactory对象
	-->
	<bean class="org.mybatis.spring.SqlSessionFactoryBean">
		<!-- 数据库源 -->
		<property name="dataSource" ref="druidDataSource"/>
		<!-- Mybatis核心配置文件 -->
		<property name="configLocation" value="mybatis-config.xml"/>
		<!-- 指定实体类的别名 -->
		<property name="typeAliasesPackage" value="cw.study.spring.pojo"/>
	</bean>
	
	<!--
		Mapper扫描配置器
		扫描Mapper接口，生成代理类（原先代理类对象通过调用SqlSession的getMapper获取）
		spring中根据类中自动注入，会将底层自动生成的代理类对象主动注入到需要的位置
		我们直接使用底层自动生成的代理类对象即可
	-->
	<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
		<!-- 扫描的包 -->
		<property name="basePackage" value="cw.study.spring.mapper"/>
	</bean>
	
	<!-- 事务管理器 -->
	<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<!-- 数据源 -->
		<property name="dataSource" ref="druidDataSource"/>
	</bean>
	
	<!-- 启用事务注解 -->
	<!-- 指定管理事务的事务管理器 -->
	<!-- 使用注解不用手动配置切面、通知、切点等 -->
	<tx:annotation-driven transaction-manager="txManager"/>
	
</beans>
```
# 12. 添加事务
```
@Transactional // 添加事务
@Service("accountService") // 纳入spring容器管理
public class AccountServiceImpl implements AccountService {
    ...
}
```
# 13. 测试程序
```
import cw.study.spring.service.AccountService;
import org.junit.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * ClassName: SMTest
 * Package: PACKAGE_NAME
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-05 13:47
 * @Version 1.0
 */
public class SMTest {
    @Test
    public void testSM(){
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
        AccountService accountService = applicationContext.getBean("accountService", AccountService.class);
        try {
            accountService.transfer("act001", "act002", 10000.0);
            System.out.println("转账成功");
        } catch (Exception e) {
            e.printStackTrace();
            System.out.println("转账失败");
        }
    }
}
```

# 14. 源码解析：MyBatis-Spring 集成机制

## 14.1. SqlSessionFactoryBean — SqlSessionFactory 的 Spring 管理

```
SqlSessionFactoryBean 实现了 FactoryBean<SqlSessionFactory>，因此：
  - getBean("&sqlSessionFactory") → 返回 SqlSessionFactoryBean 本身
  - getBean("sqlSessionFactory") → 返回 SqlSessionFactory 实例
  - SqlSessionFactoryBean 是单例的，创建的 SqlSessionFactory 也是单例的

public class SqlSessionFactoryBean
        implements FactoryBean<SqlSessionFactory>, InitializingBean, ApplicationListener<ApplicationEvent> {

    // 依赖注入的属性
    private DataSource dataSource;             // 数据源
    private Resource configLocation;           // mybatis-config.xml
    private String typeAliasesPackage;         // 别名包
    private Resource[] mapperLocations;        // mapper.xml 位置
    private Interceptor[] plugins;             // MyBatis 插件（如 PageHelper）
    private Configuration configuration;       // MyBatis Configuration 对象

    @Override
    public void afterPropertiesSet() throws Exception {
        // ★ InitializingBean 回调：在属性注入完成后，真正构建 SqlSessionFactory
        this.sqlSessionFactory = buildSqlSessionFactory();
    }

    protected SqlSessionFactory buildSqlSessionFactory() throws Exception {
        // 1. ★ 创建 Configuration（如果注册了，则使用注册的；否则创建新的）
        Configuration targetConfiguration;
        if (this.configuration != null) {
            targetConfiguration = this.configuration;
        } else {
            targetConfiguration = new Configuration();
        }

        // 2. ★ 创建 Environment（环境 = 数据源 + 事务工厂）
        //    使用 SpringManagedTransactionFactory —— 让 MyBatis 事务由 Spring 管理
        targetConfiguration.setEnvironment(new Environment(
                SqlSessionFactoryBean.class.getSimpleName(),
                new SpringManagedTransactionFactory(),  // ★ 关键：Spring 事务管理
                this.dataSource));

        // 3. ★ 扫描 typeAliasesPackage 下的实体类，注册别名
        if (this.typeAliasesPackage != null) {
            // 扫描包下所有类，注册为 MyBatis 别名（类名首字母小写）
            targetConfiguration.getTypeAliasRegistry()
                    .registerAliases(this.typeAliasesPackage);
        }

        // 4. ★ 注册 MyBatis 插件（如 PageHelper 分页插件）
        if (this.plugins != null) {
            for (Interceptor plugin : this.plugins) {
                targetConfiguration.addInterceptor(plugin);
            }
        }

        // 5. ★ 加载 MyBatis 核心配置文件（如果有，覆盖前面的默认配置）
        if (this.configLocation != null) {
            XMLConfigBuilder xmlConfigBuilder = new XMLConfigBuilder(
                    this.configLocation.getInputStream(), null, null);
            xmlConfigBuilder.parse();  // 解析 settings、typeHandlers 等
        }

        // 6. ★ 扫描 mapperLocations 下的所有 mapper.xml
        if (this.mapperLocations != null) {
            for (Resource mapperLocation : this.mapperLocations) {
                XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(
                        mapperLocation.getInputStream(),
                        targetConfiguration,
                        mapperLocation.toString(),
                        targetConfiguration.getSqlFragments());
                xmlMapperBuilder.parse();  // 解析 namespace、resultMap、SQL 语句
            }
        }

        // 7. ★ 构建 SqlSessionFactory
        return new SqlSessionFactoryBuilder().build(targetConfiguration);
    }

    @Override
    public SqlSessionFactory getObject() throws Exception {
        // ★ FactoryBean.getObject()：返回构建好的 SqlSessionFactory
        return this.sqlSessionFactory;
    }

    @Override
    public Class<?> getObjectType() {
        return SqlSessionFactory.class;
    }

    @Override
    public boolean isSingleton() {
        return true;  // SqlSessionFactory 是单例
    }
}
```

## 14.2. SpringManagedTransactionFactory — 事务桥接

```
SqlSessionFactoryBean 中设置的 SpringManagedTransactionFactory 是关键桥梁：

public class SpringManagedTransactionFactory implements TransactionFactory {

    @Override
    public Transaction newTransaction(DataSource dataSource,
            TransactionIsolationLevel level, boolean autoCommit) {

        // ★ 返回 SpringManagedTransaction 而不是 MyBatis 默认的 JdbcTransaction
        return new SpringManagedTransaction(dataSource);
    }
}

SpringManagedTransaction.getConnection() 的实现：

@Override
public Connection getConnection() throws SQLException {
    if (this.connection == null) {
        openConnection();
    }
    return this.connection;
}

private void openConnection() throws SQLException {
    // ★★★ 使用 DataSourceUtils.getConnection() ★★★
    //    如果当前有 Spring 事务，返回事务绑定的 Connection
    //    如果没有事务，从连接池获取
    this.connection = DataSourceUtils.getConnection(this.dataSource);
}

// commit/rollback 被委托给 Spring 事务管理器，而不是 MyBatis 自行管理：
@Override
public void commit() throws SQLException {
    if (this.connection != null && !this.isConnectionTransactional
            && !this.autoCommit) {
        this.connection.commit();  // 只有非 Spring 事务时才自己 commit
    }
}

@Override
public void rollback() throws SQLException {
    if (this.connection != null && !this.isConnectionTransactional
            && !this.autoCommit) {
        this.connection.rollback();  // 只有非 Spring 事务时才自己 rollback
    }
}

★ 核心设计：
   - 当有 Spring 事务时（isConnectionTransactional = true），
     事务的提交和回滚由 TransactionInterceptor 控制
   - 当没有 Spring 事务时，MyBatis 自行管理事务
```

## 14.3. SqlSessionTemplate — SqlSession 的线程安全封装

```
SqlSessionTemplate 是 MyBatis-Spring 的核心类，封装了 SqlSession 的使用：

public class SqlSessionTemplate implements SqlSession, DisposableBean {

    private final SqlSessionFactory sqlSessionFactory;
    private final ExecutorType executorType;

    // ★ 使用 JDK 动态代理创建 SqlSession 的代理
    private final SqlSession sqlSessionProxy;

    public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType) {
        this.sqlSessionFactory = sqlSessionFactory;
        this.executorType = executorType;

        // ★ 创建动态代理
        this.sqlSessionProxy = (SqlSession) Proxy.newProxyInstance(
                SqlSessionFactory.class.getClassLoader(),
                new Class[] { SqlSession.class },
                new SqlSessionInterceptor());  // ★ 核心拦截器
    }

    // 所有 SqlSession 方法调用都被代理拦截
    // getConnection → close → commit → selectOne → selectList → insert → update → delete

    private class SqlSessionInterceptor implements InvocationHandler {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            // ★ 获取 SqlSession
            //   如果当前有 Spring 事务，返回事务绑定的 SqlSession
            //   如果没有事务，创建新的 SqlSession
            SqlSession sqlSession = getSqlSession(
                    SqlSessionTemplate.this.sqlSessionFactory,
                    SqlSessionTemplate.this.executorType,
                    SqlSessionTemplate.this.exceptionTranslator);

            try {
                // ★ 执行实际的 MyBatis 操作
                Object result = method.invoke(sqlSession, args);

                // ★ 如果当前 session 不在 Spring 事务中，立即 commit
                if (!isSqlSessionTransactional(sqlSession,
                        SqlSessionTemplate.this.sqlSessionFactory)) {
                    sqlSession.commit(true);
                }
                return result;
            } catch (Throwable t) {
                // 异常处理 + 通过 PersistenceExceptionTranslator 转换异常
                throw ExceptionUtil.unwrapThrowable(t);
            } finally {
                if (sqlSession != null) {
                    // ★ 关闭 SqlSession（如果不在事务中）
                    //    如果在事务中，只是标记为"需要关闭"，实际由事务管理器处理
                    closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                }
            }
        }
    }

    // getSqlSession() 的事务感知逻辑：
    static SqlSession getSqlSession(SqlSessionFactory sessionFactory,
            ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {

        // ★ 从 TransactionSynchronizationManager 中获取当前线程的事务资源
        SqlSessionHolder holder = (SqlSessionHolder)
                TransactionSynchronizationManager.getResource(sessionFactory);

        if (holder != null && holder.isSynchronizedWithTransaction()) {
            // ★ 如果当前有事务，且需要的是 ExecutorType.BATCH，升级 Executor
            if (holder.getExecutorType() != executorType) { ... }
            // ★ 返回事务绑定的 SqlSession
            return holder.getSqlSession();
        }

        // ★ 没有事务：创建新的 SqlSession
        SqlSession session = sessionFactory.openSession(executorType);

        // ★ 如果当前线程有事务同步（Spring 事务激活），将 SqlSession 绑定到线程
        if (TransactionSynchronizationManager.isSynchronizationActive()) {
            // 注册 TransactionSynchronization：在事务提交/回滚时自动 close SqlSession
            registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);
        }
        return session;
    }
}

★ 设计精髓：
   SqlSessionTemplate + SqlSessionInterceptor 实现了：
   - 线程安全（每个线程/事务获取独立的 SqlSession）
   - 事务感知（有事务时复用，无事务时新建）
   - 自动资源管理（自动关闭，自动提交/回滚）
```

## 14.4. MapperScannerConfigurer — Mapper 接口自动代理

```
MapperScannerConfigurer 实现了 BeanDefinitionRegistryPostProcessor，
在容器启动时自动扫描指定包下的所有 Mapper 接口，为每个接口创建代理 Bean：

public class MapperScannerConfigurer
        implements BeanDefinitionRegistryPostProcessor, InitializingBean,
        ApplicationContextAware, BeanNameAware {

    private String basePackage;  // 扫描的包名

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        if (this.processPropertyPlaceHolders) {
            processPropertyPlaceHolders();
        }

        // ★ 创建 ClassPathMapperScanner
        ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
        scanner.setAddToConfig(true);
        scanner.setAnnotationClass(this.annotationClass);
        scanner.setMarkerInterface(this.markerInterface);
        scanner.setSqlSessionFactory(this.sqlSessionFactory);
        scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
        scanner.setResourceLoader(this.applicationContext);
        // ...

        // ★ 扫描指定包
        scanner.scan(StringUtils.tokenizeToStringArray(
                this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
    }
}

ClassPathMapperScanner 继承自 Spring 的 ClassPathBeanDefinitionScanner，
重写了 doScan() 方法：

@Override
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    // 调用父类的扫描逻辑，找到所有符合 filter 的接口
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
        logger.warn("No MyBatis mapper was found...");
    } else {
        // ★ 对每个扫描到的 Mapper 接口进行处理
        processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
}

private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    for (BeanDefinitionHolder holder : beanDefinitions) {
        GenericBeanDefinition definition =
                (GenericBeanDefinition) holder.getBeanDefinition();

        // ★★★ 关键：修改 BeanDefinition 的 BeanClass 为 MapperFactoryBean ★★★
        // 原本 BeanClass = com.example.mapper.AccountMapper（接口）
        // 修改后 BeanClass = org.mybatis.spring.mapper.MapperFactoryBean
        definition.getConstructorArgumentValues().addGenericArgumentValue(
                definition.getBeanClassName());  // 保留原始 Mapper 接口类名
        definition.setBeanClass(this.mapperFactoryBeanClass);  // 设置为 MapperFactoryBean ★
        // ...
    }
}
```

### 14.5. MapperFactoryBean — 单个 Mapper 接口的 FactoryBean

```
每个 Mapper 接口最终由一个 MapperFactoryBean 来创建代理对象：

public class MapperFactoryBean<T> extends SqlSessionDaoSupport
        implements FactoryBean<T> {

    private Class<T> mapperInterface;

    @Override
    public T getObject() throws Exception {
        // ★ 通过 SqlSession.getMapper(mapperInterface) 获取 MyBatis 动态代理
        return getSqlSession().getMapper(this.mapperInterface);
    }

    @Override
    public Class<T> getObjectType() {
        return this.mapperInterface;
    }

    @Override
    public boolean isSingleton() {
        return true;  // Mapper 代理对象是单例的
    }
}

MapperFactoryBean 继承自 SqlSessionDaoSupport：
  → 依赖注入 SqlSessionTemplate 或 SqlSessionFactory
  → getSqlSession() 返回 SqlSessionTemplate
  → 所有 Mapper 操作都在 SqlSessionTemplate 中执行
  → 达到事务感知 + 线程安全的效果 ★
```

## 14.5. MyBatis-Spring 集成架构总图

```
┌──────────────────────────────────────────────────────────────────┐
│                     MyBatis-Spring 集成架构                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Spring 容器层                                                    │
│  ┌──────────────────────┐                                        │
│  │ MapperScannerConfigurer│  ← BeanDefinitionRegistryPostProcessor│
│  │ 扫描 Mapper 接口        │                                      │
│  └──────────┬───────────┘                                        │
│             │ 修改 BeanClass                                      │
│             ▼                                                    │
│  ┌──────────────────────┐                                        │
│  │  MapperFactoryBean   │  ← FactoryBean<AccountMapper>          │
│  │  返回 AccountMapper 代理│                                      │
│  └──────────┬───────────┘                                        │
│             │ getSqlSession()                                     │
│             ▼                                                    │
│  ┌──────────────────────┐                                        │
│  │  SqlSessionTemplate  │  ← 线程安全 + 事务感知 + 代理           │
│  └──────────┬───────────┘                                        │
│             │ 使用                                                │
│             ▼                                                    │
│  ┌──────────────────────┐     ┌─────────────────────────┐        │
│  │  SqlSessionFactoryBean│────→│  SqlSessionFactory      │        │
│  └──────────┬───────────┘     └──────────┬──────────────┘        │
│             │                            │                        │
│             │ buildSqlSessionFactory()   │                        │
│             ▼                            │                        │
│  ┌──────────────────────────────────────┐│                        │
│  │           Configuration              ││                        │
│  │  ┌────────────────────────────────┐  ││                        │
│  │  │ Environment                    │  ││                        │
│  │  │  - DataSource (Druid/... )     │  ││                        │
│  │  │  - TransactionFactory          │◀─┘│                        │
│  │  │    ★ SpringManagedTransaction  │   │                        │
│  │  │      Factory                   │   │                        │
│  │  └────────────────────────────────┘   │                        │
│  │  - MappedStatements (解析的 SQL)      │                        │
│  │  - Mapper 接口注册                     │                        │
│  └──────────────────────────────────────┘                        │
│                                                                  │
│  事务整合层                                                       │
│  ┌─────────────────────────────────────────┐                     │
│  │ DataSourceTransactionManager            │                     │
│  │  ├─ doBegin() → TransactionSynchronization                  │
│  │  │             Manager.bindResource(ds, conn)                │
│  │  │   ★ 连接绑定到线程                                        │
│  │  ├─ doCommit() → conn.commit()                               │
│  │  └─ doRollback() → conn.rollback()                           │
│  └─────────────────────────────────────────┘                     │
│                      │                                           │
│  ┌───────────────────┼──────────────────────┐                    │
│  │ TransactionSynchronizationManager       │                    │
│  │  ThreadLocal<Map<Object, Object>>       │                    │
│  │  存储: resources[druidDS] = ConnectionHolder                  │
│  │        resources[sqlSessionFactory] = SqlSessionHolder        │
│  └─────────────────────────────────────────┘                    │
└──────────────────────────────────────────────────────────────────┘

执行流程（以 Service.transfer() @Transactional 为例）：
  1. TransactionInterceptor → doBegin() → con.setAutoCommit(false)
  2. Service → accountMapper.selectByActno("act001")
     → MapperFactoryBean.getObject() → Mapper 代理
     → SqlSessionTemplate.selectOne()
     → getSqlSession() → TransactionSynchronizationManager.getResource(sf)
     → 返回事务绑定的 SqlSession（内部使用同一个 Connection）
  3. accountMapper.update(act1) → 同一个 SqlSession、同一个 Connection
  4. accountMapper.update(act2) → 同一个 SqlSession、同一个 Connection
  5. TransactionInterceptor → doCommit() → con.commit()
  6. TransactionSynchronizationManager 清理绑定资源
```

## 14.6. 设计思想总结

```
1. FactoryBean 的"生产性"运用：
   SqlSessionFactoryBean 和 MapperFactoryBean 都是 FactoryBean 的实际运用：
   - SqlSessionFactoryBean：复杂的构建过程（配置、扫描、组装）
   - MapperFactoryBean：创建动态代理对象（隐藏 MyBatis 的 Proxy 机制）

2. 适配器模式（SpringManagedTransactionFactory）：
   将 MyBatis 的 TransactionFactory 接口适配到 Spring 事务管理体系，
   使 MyBatis 在"不知不觉"中参与了 Spring 的声明式事务。

3. 代理的代理：
   SqlSessionTemplate 通过 JDK 动态代理封装了 SqlSession，
   MapperFactoryBean.getObject() 返回的是 MyBatis 的 Mapper 动态代理，
   形成了"Spring 代理 → SqlSessionTemplate 代理 → MyBatis Mapper 代理"的三层代理结构。

4. BeanDefinitionRegistryPostProcessor 的扩展点运用：
   MapperScannerConfigurer 利用这个扩展点在运行时动态注册 BeanDefinition，
   使用户只需要声明一个 Mapper 接口就能自动获得可注入的 Bean。

5. ThreadLocal 资源绑定：
   与 DataSourceUtils 的事务连接绑定类似，
   SqlSessionHolder 也通过 TransactionSynchronizationManager 实现线程级绑定，
   确保同一事务内的所有 Mapper 操作共享同一个 SqlSession。
```
