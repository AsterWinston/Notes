# 1. Spring 事务支持概述

spring 除了提供了对AOP的支持外，使得我们可以通过编程进行事务的管理，spring 还提供了一套API，使得我们可以通过声明实现事务的管理
spring 对于事务的支持，底层还是通过动态代理的，是基于面向切面编程AOP的，spring 对AOP进行了封装

# 2. 事务概述
什么是事务
    在一个业务流程当中，通常需要多条DML（insert delete update）语句共同联合才能完成，这多条DML语句必须同时成功，或者同时失败，这样才能保证数据的安全。
    多条DML要么同时成功，要么同时失败，这叫做事务。
    事务：Transaction（tx）
事务的四个处理过程：
    第一步：开启事务 (start transaction)
    第二步：执行核心业务代码
    第三步：提交事务（如果核心业务处理过程中没有出现异常）(commit transaction)
    第四步：回滚事务（如果核心业务处理过程中出现异常）(rollback transaction)
事务的四个特性：
    A 原子性：事务是最小的工作单元，不可再分。
    C 一致性：事务要求要么同时成功，要么同时失败。事务前和事务后的总量不变。
    I 隔离性：事务和事务之间因为有隔离性，才可以保证互不干扰。
    D 持久性：成功的操作对数据库的影响永久保存，持久性是事务结束的标志。

# 3. 引入事务场景
以银行账户转账为例学习事务。两个账户act-001和act-002。act-001账户向act-002账户转账10000，必须同时成功，或者同时失败。（一个减成功，一个加成功， 这两条update语句必须同时成功，或同时失败。）
连接数据库的技术采用Spring框架的JdbcTemplate。
采用三层架构搭建
## 3.1 依赖
```
<dependencies>
  <!-- spring 核心依赖 -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.0.8</version>
  </dependency>
  <!-- spring jdbc 依赖，要使用JdbcTemplate -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>6.0.8</version>
  </dependency>
  <!-- mysql驱动，连接数据库 -->
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.30</version>
  </dependency>
  <!-- 数据库连接池 -->
  <dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.12</version>
  </dependency>
  <!-- 使用@Resource注解 -->
  <dependency>
    <groupId>jakarta.annotation</groupId>
    <artifactId>jakarta.annotation-api</artifactId>
    <version>1.3.5</version>
  </dependency>
  <!-- 单元测试 -->
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13.2</version>
    <scope>test</scope>
  </dependency>
</dependencies>

```
## 3.2 数据库表
```
USE dbtest;

DROP TABLE IF EXISTS t_act;

CREATE TABLE t_act (
    id int PRIMARY KEY AUTO_INCREMENT,
    actno VARCHAR(255),
    balance DECIMAL(10, 2)
);

INSERT INTO t_act(actno, balance) VALUES ('act001', 50000);
INSERT INTO t_act(actno, balance) VALUES ('act002', 0);

```
## 3.3 创建包结构
## 3.4 实体类
```
package cw.study.spring.bank.pojo;

/**
 * ClassName: Account
 * Package: cw.study.spring.bank.pojo
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-04 21:07
 * @Version 1.0
 */
public class Account {
    private Long id;
    private String actno;
    private Double balance;
    
    @Override
    public String toString() {
        return "Account{" +
                "id=" + id +
                ", actno='" + actno + '\'' +
                ", balance=" + balance +
                '}';
    }
    
    public Account() {
    }
    
    public Account(Long id, String actno, Double balance) {
        this.id = id;
        this.actno = actno;
        this.balance = balance;
    }
    
    public Long getId() {
        return id;
    }
    
    public void setId(Long id) {
        this.id = id;
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
## 3.5 DAO
```
package cw.study.spring.bank.dao;

import cw.study.spring.bank.pojo.Account;

/**
 * ClassName: AccountDao
 * Package: cw.study.spring.bank.dao
 * Description:
 * 定义账户信息的CRUD
 * DAO中只执行SQL语句，没有任何业务逻辑，不和业务挂钩
 *
 * @Author tcw
 * @Create 2023-06-04 21:10
 * @Version 1.0
 */
public interface AccountDao {
    
    /**
     * 根据账号查询账户信息
     * 
     * @param actno 账号
     * @return 账户信息
     */
    Account selectByActno(String actno);
    
    /**
     * 更新账户信息
     * @param account 新的账户信息
     * @return 影响数据库条数
     */
    int updateAccount(Account account);
    
}

package cw.study.spring.bank.dao.impl;

import cw.study.spring.bank.dao.AccountDao;
import cw.study.spring.bank.pojo.Account;
import org.springframework.jdbc.core.BeanPropertyRowMapper;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

/**
 * ClassName: AccountDaoImpl
 * Package: cw.study.spring.bank.dao.impl
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-04 21:14
 * @Version 1.0
 */
@Repository("accountDao")
public class AccountDaoImpl implements AccountDao {
    
    @Resource(name = "jdbcTemplate")
    private JdbcTemplate jdbcTemplate;
    
    @Override
    public Account selectByActno(String actno) {
        String sql = "select * from t_act where actno = ?";
        return jdbcTemplate.queryForObject(sql, new BeanPropertyRowMapper<>(Account.class), actno);
    }
    
    @Override
    public int updateAccount(Account account) {
        String sql = "update t_act set balance = ? where actno = ?";
        return jdbcTemplate.update(sql, account.getBalance(), account.getActno());
    }
}

```
## 3.6 Service
```
package cw.study.spring.bank.service;

/**
 * ClassName: AccountService
 * Package: cw.study.spring.bank.service
 * Description:
 * 业务接口
 * 事务在这个接口下控制
 *
 * @Author tcw
 * @Create 2023-06-04 21:19
 * @Version 1.0
 */
public interface AccountService {
    
    /**
     * 转账业务
     * 
     * @param fromActno 转出账户
     * @param toActno 转入账户
     * @param balance 转账金额
     */
    void transfer(String fromActno, String toActno, Double balance);
    
}

package cw.study.spring.bank.service.impl;

import cw.study.spring.bank.dao.AccountDao;
import cw.study.spring.bank.pojo.Account;
import cw.study.spring.bank.service.AccountService;
import org.springframework.stereotype.Service;

/**
 * ClassName: AccountServiceImpl
 * Package: cw.study.spring.bank.service.impl
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-04 21:21
 * @Version 1.0
 */
@Service("accountService")
public class AccountServiceImpl implements AccountService {
    
    @Resource(name = "accountDao")
    private AccountDao accountDao;
    
    /**
     * 转账业务，需要进行事务的控制
     *
     * @param fromActno 转出账户
     * @param toActno 转入账户
     * @param balance 转账金额
     */
    @Override
    public void transfer(String fromActno, String toActno, Double balance) {
        // 查询转出账户的余额
        Account fromAct = accountDao.selectByActno(fromActno);
        if (fromAct.getBalance() < balance) {
            throw new RuntimeException("余额不足...");
        }
        // 余额充足，进行转账
        // 查询转入账户
        Account toAct = accountDao.selectByActno(toActno);
        // 先在内存中进行账号信息的更新
        fromAct.setBalance(fromAct.getBalance() - balance);
        toAct.setBalance(toAct.getBalance() + balance);
        // 更新数据库
        int count = accountDao.updateAccount(fromAct);
        count += accountDao.updateAccount(toAct);
        if (count != 2) {
            throw new RuntimeException("转账失败...");
        }
    }
}

```
## 3.7 配置文件spring.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
						   http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

	<!-- 组件扫描 -->
	<context:component-scan base-package="cw.study.spring.bank"/>
	
	<!-- 配置JDBCTemplate -->
	<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
		<property name="dataSource" ref="dataSource"/>
	</bean>
	
	<!-- 数据源 -->
	<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
		<property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
		<property name="url" value="jdbc:mysql://127.0.0.1:3306/dbtest"/>
		<property name="username" value="root"/>
		<property name="password" value="123123"/>
	</bean>
	
</beans>
```
## 3.8 客户端
```
@org.junit.Test
public void test() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    AccountService accountService = applicationContext.getBean("accountService", AccountService.class);
    try {
        accountService.transfer("act001", "act002", 10000.0);
        System.out.println("转账成功...");
    } catch (Exception e) {
        e.printStackTrace();
    }
}

```
## 3.9 模拟异常
```
@Service("accountService")
public class AccountServiceImpl implements AccountService {
    
    ...
    
    @Override
    public void transfer(String fromActno, String toActno, Double balance) {
    	...
        // 更新数据库
        int count = accountDao.updateAccount(fromAct);
        
        // 模拟异常
        String s = null;
        s.toLowerCase();
        
        count += accountDao.updateAccount(toAct);
        if (count != 2) {
            throw new RuntimeException("转账失败...");
        }
    }
}

```

# 4. Spring实现事务的两种方式
编程式事务
    通过编写代码的方式来实现事务的管理。
声明式事务
    基于注解方式
    基于XML配置方式

声明式事务，spring框架封装的一套用于处理事务的API

# 5. Spring事务管理API
```
Spring对事务的管理底层实现方式是基于AOP实现的。采用AOP的方式进行了封装。所以Spring专门针对事务开发了一套API，API的核心接口如下：
PlatformTrasactionManager(interface)
  CallbackPerferringPlatformTrasactionManager(interface)
  AbstractPlatformTransactionManager(class)
    DataSourceTransactionManager(class)
      JdbcTrasactionManager(class)
    JtaTrasactionManager(class)
  ResourceTransactionManager(interface)

PlatformTransactionManager接口：spring事务管理器的核心接口。
第三方ORM框架可以通过提供该接口的实现类，让Spring可以帮助第三方ORM框架进行事务的管理
在Spring6中PlatformTransactionManager接口有两个实现：
DataSourceTransactionManager：支持JdbcTemplate、MyBatis、Hibernate等事务管理。
JtaTransactionManager：支持分布式事务管理。
如果要在Spring6中使用JdbcTemplate，就要使用DataSourceTransactionManager来管理事务。（Spring内置写好了，可以直接用。）

```

# 6. 声明式事务之注解实现方式
## 6.1 Spring 配置文件配置事务管理器
```
<!-- 
  配置事务管理器
  由于管理事务需要通过数据库连接对象调用setAutoCommit方法
  所以需要提供数据源
-->
<bean id="dataSourceTransactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
  <property name="dataSource" ref="dataSource"/>
</bean>

```
## 6.2 在spring配置文件中引入tx命名空间
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
						   http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
						   http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd">

```
## 6.3 开启事务注解驱动器
```
在Spring配置文件中配置事务注解驱动器，开启事务注解，告诉Spring采用注解的方式控制事务
<!-- 开启事务注解驱动器，采用注解的方式控制事务 -->
<tx:annotation-driven transaction-manager="dataSourceTransactionManager"/>
//transaction-manager 是指定事务管理器的 Bean 名称，必须和你配置的事务管理器 Bean 的 id 一致
//当且仅当 Spring 容器中「只有一个」PlatformTransactionManager Bean 时，<tx:annotation-driven/> 才可以省略 transaction-manager 属性。
```
## 6.4 在Service类上或方法上添加@Transactional注解控制事务
```
在类上添加该注解，该类中所有的方法都有事务。
在某个方法上添加该注解，表示只有这个方法使用事务。
@Service("accountService")
public class AccountServiceImpl implements AccountService {
    ...
    @Override
    @Transactional // 控制事务
    public void transfer(String fromActno, String toActno, Double balance) {
        ...
    }
}

```
## 6.5 测试
## 6.6 事务实现原理
```
在使用@Transactional注解后

DataSourceTransactionManager：负责事务控制
TransactionSynchronizationManager：负责线程绑定
ConnectionHolder：负责资源封装
```

# 7. 事务的属性
## 7.1 事务的属性
```
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@Reflective
public @interface Transactional {
    @AliasFor("transactionManager")
    String value() default "";

    @AliasFor("value")
    String transactionManager() default "";

    String[] label() default {};
	// 事务的传播行为
    Propagation propagation() default Propagation.REQUIRED;
	// 事务的隔离级别
    Isolation isolation() default Isolation.DEFAULT;
	// 事务的超时时间，默认-1表示不限时
    int timeout() default -1;
	
    String timeoutString() default "";
	// 属性值为true，表示当前事务是一个只读事务
    boolean readOnly() default false;
	// 设置当程序出现什么样的异常的时候进行回滚
    Class<? extends Throwable>[] rollbackFor() default {};

    String[] rollbackForClassName() default {};
	// 设置当程序出现什么样的异常的时候不回滚
    Class<? extends Throwable>[] noRollbackFor() default {};

    String[] noRollbackForClassName() default {};
}
```
## 7.2 事务传播行为
```
事务的传播行为：在service类中有a()方法和b()方法，a()方法上有事务，b()方法上也有事务，当a()方法执行过程中调用了b()方法，事务是如何传递的？合并到一个事务里？还是开启一个新的事务？这就是事务传播行为。
事务传播行为在spring框架中被定义为枚举类型：
public enum Propagation {
    REQUIRED(0),
    SUPPORTS(1),
    MANDATORY(2),
    REQUIRES_NEW(3),
    NOT_SUPPORTED(4),
    NEVER(5),
    NESTED(6);

    private final int value;

    private Propagation(int value) {
        this.value = value;
    }

    public int value() {
        return this.value;
    }
}

一共有七种传播行为：
	REQUIRED：支持当前事务，如果不存在就新建一个(默认行为)【没有就新建，有就加入】
		执行到当前业务方法时，如果原先有事务就加入原来的事务，如果原先没有事务，就自己新建一个事务
	SUPPORTS：支持当前事务，如果当前没有事务，就以非事务方式执行【有就加入，没有就不管了】
		如果原先有事务就加入原来的事务，如果原先没有事务，那就不开启事务，不进行事务控制
	MANDATORY（强制的、义务的）：必须运行在一个事务中，如果当前没有事务正在发生，将抛出一个异常【有就加入，没有就抛异常】
		如果原先有事务就加入原来的事务，如果原先没有事务，那就抛异常
	REQUIRES_NEW：开启一个新的事务，如果一个事务已经存在，则将这个存在的事务挂起【不管有没有，直接开启一个新事务，开启的新事务和之前的事务不存在嵌套关系，之前事务被挂起】
		不管原先是否有事务，都开启一个新的事务，与原先的事务并列，挂起暂停原先的事务，执行控制新的事务
	NOT_SUPPORTED：以非事务方式运行，如果有事务存在，挂起当前事务【不支持事务，存在就挂起】
		不进行事务控制，原先有事务，就暂停挂起事务
	NEVER：以非事务方式运行，如果有事务存在，抛出异常【不支持事务，存在就抛异常】
	NESTED：如果当前正有一个事务在进行中，则该方法应当运行在一个嵌套式事务中。被嵌套的事务可以独立于外层事务进行提交或回滚。如果外层事务不存在，行为就像REQUIRED一样。【有事务的话，就在这个事务里再嵌套一个完全独立的事务，嵌套的事务可以独立的提交和回滚。没有事务就和REQUIRED一样。】（这个功能依赖于数据库的保存点功能，如果数据库不支持，这个注解也就失效）
		原先没有事务就新建一个事务，原先有事务就在原先的事务内新建一个独立的嵌套事务

代码没有执行到业务方法的结尾花括号，那么业务方法就没结束，事务也不会结束，因为没有执行到结尾花括号，永远不知道后面有什么代码
对于在一个事务内捕获出现的异常，依然还会进行当前事务的回滚，捕获异常无效，因为当前事务不知道捕获异常后面还有什么DML语句，为了保证数据安全，出现异常只能回滚
事务1调用事务2，事务1的传播行为为REQUIRED，事务2的传播行为为REQUIRES_NEW，当事务2中抛出异常时，对于事务2会进行事务回滚，如果在事务1中捕获事务2抛出的异常，则事务1还会正常进行执行，因为REQUIRES_NEW是挂起原先的事务，创建一个与原先事务并列的事务，两个事务并没有关系，所以事务1相当于无异常，不会进行回滚
```
|传播行为|核心逻辑|关键特点|适用场景|
| - | - | - | - |
| REQUIRED |有则加入，无则新建（默认）|共用事务，整体回滚|绝大多数核心业务（下单/转账）|
|SUPPORTS|有则加入，无则无事务|灵活适配，查询优先|普通查询方法|
|MANDATORY|有则加入，无则抛异常|强制事务，防止无事务执行|核心修改操作|
|REQUIRES_NEW|必新建事务，有则挂起|事务隔离，主/子事务独立提交/回滚|日志记录、独立操作|
|NOT_SUPPORTED|必无事务，有则挂起|脱离事务，提升性能|纯查询 / 导出报表|
|NEVER|必无事务，有则抛异常|禁止事务，严格限制|异步通知第三方接口|
|NESTED|有则建保存点，无则新建|嵌套子事务，子事务仅回滚自身，不影响主事务；主事务回滚，子事务跟着回滚|子操作失败不影响主操作，主操作影响子操作|
## 7.3 在代码中设置事务的传播行为
```
@Transactional(propagation = Propagation.REQUIRED)
```
## 7.4 事务隔离级别
```
事务隔离级别类似于教室A和教室B之间的那道墙，隔离级别越高表示墙体越厚。隔音效果越好。
事务隔离，防止多事务并发的情况下，多个事务同时操作同一个表时互相干扰的一种机制
事务隔离级别在Spring中定义为枚举类型：
public enum Isolation {
    DEFAULT(-1), // 默认隔离级别
    READ_UNCOMMITTED(1), // 读未提交
    READ_COMMITTED(2), // 读已提交（Oracle默认隔离级别）
    REPEATABLE_READ(4), // 可重复读（MySQL默认隔离级别）
    SERIALIZABLE(8); // 序列化

    private final int value;

    private Isolation(int value) {
        this.value = value;
    }

    public int value() {
        return this.value;
    }
}

```
## 7.5 在Spring代码中如何设置隔离级别
```
设置当前事务的隔离级别
@Transactional(isolation = Isolation.READ_COMMITTED)
```
## 7.6 数据库中读取数据存在的三大问题（三大读问题）
脏读：读取到没有提交到数据库的数据，叫做脏读。
    事务1对数据库表中的数据进行了修改，但是还没有进行提交，此时事务2就可以读到事务1修改后没有提交的数据，这个时候就发生了脏读
不可重复读：在同一个事务当中，第一次和第二次读取的数据不一样。
    对于两个事务Session A、Session B，Session A 读取 了一个字段，然后 Session B 更新 了该字段并且进行了提交。 之后 Session A 再次读取 同一个字段， 值就不同 了。那就意味着发生了不可重复读。
    不可重复读，同一个事务内，两次读取相同字段的数据，获取到的值不一样
    不可重复读，是前后两次从数据库中读取数据的结果不同
幻读：读到的数据是假的。
    对于两个事务Session A、Session B, Session A 从一个表中 读取 了一个字段, 然后 Session B 在该表中 插入 了一些新的行。 之后, 如果 Session A 再次读取 同一个表, 就会多出几行。那就意味着发生了幻读。
    幻读是针对于向数据库中进行数据插入的操作，当前从数据库中读到的数据记录的条数比之前读取到的数据记录条数多了，好像出现了幻觉一样，凭空多了一些记录出来
    幻读强调的是读到了之前没有读到的记录。
## 7.7 事务隔离级别包括四个级别
读未提交：READ_UNCOMMITTED
这种隔离级别，存在脏读问题，所谓的脏读(dirty read)表示能够读取到其它事务未提交的数据。
读提交：READ_COMMITTED
解决了脏读问题，其它事务提交之后才能读到，但存在不可重复读问题。
同一个事务内前后读取到的不一样
可重复读：REPEATABLE_READ
解决了不可重复读，可以达到可重复读效果，只要当前事务不结束，读取到的数据一直都是一样的。但存在幻读问题。
序列化：SERIALIZABLE
解决了幻读问题，事务排队执行。不支持并发，事务排队执行，效率低。
| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| - | - | - | - |
| 读未提交 | 有 | 有 | 有 |
| 读提交 | 无 | 有 | 有 |
| 可重复读 | 无 | 无 | 有 |
| 序列化 | 无 | 无 | 无 |

## 7.8 事务超时
```
在Spring代码中如何设置事务超时
@Transactional(timeout = 10)
    以上代码表示设置事务的超时时间为10秒。
    表示超过10秒如果该事务中所有的DML语句还没有执行完毕的话，最终结果会选择回滚。
    默认值-1，表示没有时间限制。
    这里有个坑，事务的超时时间指的是哪段时间？
    在当前事务当中，最后一条DML语句执行之前的时间。如果最后一条DML语句后面很有很多业务逻辑，这些业务代码执行的时间不被计入超时时间。
    超时时间，是到最后一条DML语句执行完时所耗费的时间，最后一条DML语句之后的代码执行时间不计入超时时间
    如果想把所有代码执行时间都记入超时时间，可以在最后添加一个无关紧要的DML语句。
```
## 7.9 只读事务
```
代码如下：
@Transactional(readOnly = true)
readOnly属性默认值为false
将当前事务设置为只读事务，在该事务执行过程中只允许select语句执行，delete insert update均不可执行。
该特性的作用是：启动spring的优化策略。提高select语句执行效率。
如果该事务中确实没有增删改操作，建议设置为只读事务，提高select语句的执行效率
```
## 7.10 设置哪些异常回滚事务
```
代码如下：
@Transactional(rollbackFor = RuntimeException.class)
    rollbackFor属性为Class数组类型
    表示只有发生RuntimeException异常或该异常的子类异常才回滚。
```
## 7.11 设置哪些异常不回滚事务
```
代码如下：
@Transactional(noRollbackFor = NullPointerException.class)
    noRollbackFor属性为Class数组类型
    表示发生NullPointerException或该异常的子类异常不回滚，其他异常则回滚。
```


# 8. 事务的全注解式开发
## 8.1 编写配置类代替配置文件
```
package cw.study.spring.bank;

import com.alibaba.druid.pool.DruidDataSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.transaction.annotation.EnableTransactionManagement;

import javax.sql.DataSource;

/**
 * ClassName: SpringConfig
 * Package: cw.study.spring.bank
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-05 10:49
 * @Version 1.0
 */
@Configuration // 标注该类为配置类
@ComponentScan({"cw.study.spring.bank"}) // 组件扫描
// 开启事务注解驱动器，采用注解的方式控制事务 ，代替了tx:annotation-driven标签
@EnableTransactionManagement
public class SpringConfig {
    
    // Spring看到这个注解会调用该注解标注的方法
    // 该方法的返回值是一个Java对象，这个Java对象会自动纳入spring容器管理
    @Bean("druidDataSource") // 注册组件
    public DruidDataSource druidDataSource() {
        DruidDataSource druidDataSource = new DruidDataSource();
        druidDataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
        druidDataSource.setUrl("jdbc:mysql://127.0.0.1:3306/dbtest");
        druidDataSource.setUsername("root");
        druidDataSource.setPassword("123123");
        return druidDataSource;
    }
    
    // spring在调用该方法时，会自动为该方法的参数赋值
    @Bean("jdbcTemplate") // 注册组件
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        jdbcTemplate.setDataSource(dataSource);
        return jdbcTemplate;
    }
    
    @Bean("dataSourceTransactionManager")
    public DataSourceTransactionManager dataSourceTransactionManager(DataSource dataSource) {
        DataSourceTransactionManager dataSourceTransactionManager = new DataSourceTransactionManager();
        dataSourceTransactionManager.setDataSource(dataSource);
        return dataSourceTransactionManager;
    }
    
}

```
## 8.2 测试程序

```
使用原先的转账业务进行测试
@org.junit.Test
public void test() {
    ApplicationContext applicationContext = new AnnotationConfigApplicationContext(SpringConfig.class);
    AccountService accountService = applicationContext.getBean("accountService", AccountService.class);
    try {
        accountService.transfer("act001", "act002", 10000.0);
        System.out.println("转账成功...");
    } catch (Exception e) {
        e.printStackTrace();
    }
}

存在模拟异常
...

不存在模拟异常
...


```

# 9. 声明事务之XML实现方式

## 9.1 配置步骤
第一步：配置事务管理器
要让spring进行事务管理，就需要事务管理器
第二步：配置通知
spring的事务管理是基于AOP实现的
第三步：配置切面
spring的事务管理是基于AOP实现的

## 9.2 依赖
```
<dependencies>
  <!-- spring核心依赖 -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.0.8</version>
  </dependency>
  <!--
    spring AspectJ依赖
    由于要使用XML配置的方式让spring进行事务的管理
    spring事务管理是基于AOP的
    需要配置通知和切面，面向切面就需要AspectJ依赖
    将事务管理以切面的方式加入
  -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>6.0.8</version>
  </dependency>
  <!-- spring jdbc -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>6.0.8</version>
  </dependency>
  <!-- 数据库驱动 -->
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.30</version>
  </dependency>
  <!-- 数据库连接池 -->
  <dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.12</version>
  </dependency>
  <!-- 使用@Resource注解，组件自动注入 -->
  <dependency>
    <groupId>jakarta.annotation</groupId>
    <artifactId>jakarta.annotation-api</artifactId>
    <version>2.1.1</version>
  </dependency>
  <!-- 单元测试 -->
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13.2</version>
    <scope>test</scope>
  </dependency>
</dependencies>

```
## 9.3 业务程序
依旧使用之前的转账业务的代码
组件的注入使用注解的形式，事务的管理采用xml文件配置的形式

## 9.4 spring.xml配置文件
```
记得添加context、tx、aop的命名空间
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
						   http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
						   http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd
						   http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

	<!-- 组件扫描 -->
	<context:component-scan base-package="cw.study.spring.bank"/>
	
	<!-- 配置JDBCTemplate -->
	<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
		<property name="dataSource" ref="dataSource"/>
	</bean>
	
	<!-- 数据源 -->
	<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
		<property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
		<property name="url" value="jdbc:mysql://127.0.0.1:3306/dbtest"/>
		<property name="username" value="root"/>
		<property name="password" value="285013"/>
	</bean>
	
	<!--
		配置事务管理器
		由于管理事务需要通过数据库连接对象调用setAutoCommit方法
		所以需要提供数据源
	-->
	<bean id="dataSourceTransactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<property name="dataSource" ref="dataSource"/>
 	</bean>
	
	<!-- 由于使用XML配置事务管理，所以不需要事务注解驱动器 -->
	<!-- 开启事务注解驱动器，采用注解的方式控制事务 -->
	<!-- <tx:annotation-driven transaction-manager="dataSourceTransactionManager"/> -->
	
	<!-- 配置事务的通知，增强代码，spring已经封装好了，配置一下，然后以切面的形式加入业务流程 -->
	<!-- 为方法配置增强代码，和事务管理相关的代码 -->
	<!-- transaction-manager 指定管理事务的事务管理器，在通知中关联事务管理器 -->
	<tx:advice id="txAdvice" transaction-manager="dataSourceTransactionManager">
		<!-- 配置通知的相关属性 -->
		<tx:attributes>
			<!-- 要为哪种业务方法配置事务，这些通知是为哪些方法增强的 -->
			<!-- 之前的事务属性都可以在以下标签中进行配置 -->
			<tx:method name="transfer" propagation="REQUIRED" rollback-for="java.lang.Throwable"/>
  			<!--一般不写具体方法，因为方法太多，采用模糊匹配，以save开头的方法-->
      		<tx:method name="save*" propagation="REQUIRED" rollback-for="java.lang.Throwable"/>
    	</tx:attributes>
	</tx:advice>
	<!-- 配置切面，切面 = 切点 + 通知 -->
	<!-- 将增强代码以切面的形式加入业务流程 -->
	<aop:config>
		<!-- 配置切点，要为哪个方法增强 -->
		<aop:pointcut id="txPointcut" expression="execution(* cw.study.spring.bank.service..*(..))"/>
		<!-- 配置切面，将增强代码以切面的形式加入业务流程 -->
    	<!-- 配置切面使用的通知（增强代码）是哪个，要为哪个切点增强 -->
		<aop:advisor advice-ref="txAdvice" pointcut-ref="txPointcut"/>
	</aop:config>
	
</beans>

```

## 9.5 测试程序
```
@org.junit.Test
public void test() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    AccountService accountService = applicationContext.getBean("accountService", AccountService.class);
    try {
        accountService.transfer("act001", "act002", 10000.0);
        System.out.println("转账成功...");
    } catch (Exception e) {
        e.printStackTrace();
    }
}

存在模拟异常
...

不存在模拟异常
...
```

# 10. @Transactional注解失效的可能原因

## 10.1 没有经过Spring代理（最核心）
本质：事务是通过代理对象生效的，而不是原对象
### 10.1.1 同类内部方法调用（最经典）
```
@Service
public class UserService {

    public void methodA() {
        methodB(); // ❌ 直接调用，事务失效
    }

    @Transactional
    public void methodB() {
        // 事务不会生效
    }
}
```
原因：

- methodA() 调用 methodB() 是 this调用

- 没走代理对象

解决：
```
@Autowired
private UserService userService;

public void methodA() {
    userService.methodB(); // ✅ 走代理
}
```
或者（如果一定要在A这个没有事务的方法里里调用B这个带有事务的方法）
```
import org.springframework.aop.framework.AopContext;

public void methodA() {
    ((UserService) AopContext.currentProxy()).methodB();
}
```
### 10.1.2 方法不是 public
```
@Transactional
private void methodB() {}
```
结果：事务不生效

原因：

- Spring 默认只代理 public方法

### 10.1.3 使用 final / static 方法
```
@Transactional
public final void method() {}
```
原因：

- 动态代理无法重写 final 方法

- static 方法不属于对象

## 10.2 事务没有被Spring管理

### 10.2.1 类没有交给Spring管理
```
public class UserService { ... } // ❌ 没有 @Service
```
原因：

- Spring 根本没创建代理对象

必须：
```
@Service
```
### 10.2.2 new 出来的对象
UserService userService = new UserService();
userService.methodB();

原因：

- 手动 new 的对象不是 Spring Bean

## 10.3 异常处理导致事务不回滚

### 10.3.1 捕获异常但不抛出
```
@Transactional
public void test() {
    try {
        int a = 1 / 0;
    } catch (Exception e) {
        // ❌ 吞了异常
    }
}
```
结果：

- 事务不会回滚

原因：

- Spring 只在 异常抛出到代理层时 才回滚

### 10.3.2 抛出的是受检异常（默认不回滚）
```
@Transactional
public void test() throws Exception {
    throw new Exception();
}
```
结果：不会回滚

原因：

默认只回滚：

- RuntimeException

- Error

解决：
```
@Transactional(rollbackFor = Exception.class)
```

## 10.4 事务传播机制问题

### 10.4.1 事务传播行为设置不当
```
@Transactional(propagation = Propagation.REQUIRES_NEW)
```

或

```
@Transactional(propagation = Propagation.NOT_SUPPORTED)
```

可能导致：

- 新开事务
- 或直接关闭事务

### 10.4.2 外层没有事务
```
public void outer() {
    inner(); // inner有事务，但调用方式不对
}
```
如果 inner 没走代理 → 事务依然不生效

# 11. 源码解析：Spring 声明式事务底层机制

## 11.1 @EnableTransactionManagement 的事件链路

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {
    boolean proxyTargetClass() default false;               // 默认 false：有接口用 JDK，无接口用 CGLIB
    AdviceMode mode() default AdviceMode.PROXY;             // 默认 PROXY（还可以是 ASPECTJ 编译时织入）
    int order() default Ordered.LOWEST_PRECEDENCE;          // 事务切面的优先级，默认最低
}

★ @Import(TransactionManagementConfigurationSelector.class) 是关键！

TransactionManagementConfigurationSelector 是 ImportSelector 的实现类：根据 AdviceMode
的值选择注册 PROXY 模式或 ASPECTJ 模式的配置类。

public class TransactionManagementConfigurationSelector
        extends AdviceModeImportSelector<EnableTransactionManagement> {

    @Override
    protected String[] selectImports(AdviceMode adviceMode) {
        switch (adviceMode) {
            case PROXY:  // ← 默认走的这个分支
                return new String[] {
                    AutoProxyRegistrar.class.getName(),           // 1. 注册 InfrastructureAdvisorAutoProxyCreator
                    ProxyTransactionManagementConfiguration.class.getName()  // 2. 注册事务 BeanFactoryAdvisor
                };
            case ASPECTJ:
                return new String[] {
                    TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME
                };
            default:
                return null;
        }
    }
}
```

## 11.2 ProxyTransactionManagementConfiguration 注册了什么

```
@Configuration(proxyBeanMethods = false)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

    // ★ 注册一个 Advisor：BeanFactoryTransactionAttributeSourceAdvisor
    //   这个 Advisor 决定哪些 Bean 的哪些方法需要事务增强
    @Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(
            TransactionAttributeSource transactionAttributeSource,
            TransactionInterceptor transactionInterceptor) {

        BeanFactoryTransactionAttributeSourceAdvisor advisor =
                new BeanFactoryTransactionAttributeSourceAdvisor();
        advisor.setTransactionAttributeSource(transactionAttributeSource);
        advisor.setAdvice(transactionInterceptor);
        if (this.enableTx != null) {
            advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
        }
        return advisor;
    }

    // ★ 注册 TransactionAttributeSource：解析 @Transactional 注解
    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public TransactionAttributeSource transactionAttributeSource() {
        // AnnotationTransactionAttributeSource 内部组合了多个 TransactionAnnotationParser：
        //   1. SpringTransactionAnnotationParser  ← 解析 @Transactional
        //   2. JtaTransactionAnnotationParser     ← 解析 @Transactional(javax)
        //   3. Ejb3TransactionAnnotationParser    ← 解析 EJB3 事务注解
        return new AnnotationTransactionAttributeSource();
    }

    // ★★★ 注册 TransactionInterceptor ★★★
    //   这是事务管理的核心拦截器！实现了 MethodInterceptor 接口
    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public TransactionInterceptor transactionInterceptor(
            TransactionAttributeSource transactionAttributeSource) {
        TransactionInterceptor interceptor = new TransactionInterceptor();
        interceptor.setTransactionAttributeSource(transactionAttributeSource);
        if (this.txManager != null) {
            interceptor.setTransactionManager(this.txManager);
        }
        return interceptor;
    }
}
```

### 11.2.1 BeanFactoryTransactionAttributeSourceAdvisor 如何匹配方法

```
BeanFactoryTransactionAttributeSourceAdvisor 是一个 PointcutAdvisor，其切入点判断逻辑是：

如果某个类的某个方法上有 @Transactional 注解（或类上有，方法上没有），则匹配成功。

Advisor 内部通过 TransactionAttributeSource.getTransactionAttribute(method, targetClass) 来判断：
  → AnnotationTransactionAttributeSource 内部调用 SpringTransactionAnnotationParser.parseTransactionAnnotation()
  → 反射检查方法/类上是否有 @Transactional 注解
  → 如果有，解析注解属性封装为 TransactionAttribute（包含传播行为、隔离级别等）
  → 如果返回非 null，说明该方法需要事务增强
```

## 11.3 TransactionInterceptor.invoke() — 事务拦截核心源码

```
TransactionInterceptor 实现了 MethodInterceptor（AOP 中的 MethodInterceptor），
它的 invoke() 方法是事务拦截的总入口：

public class TransactionInterceptor extends TransactionAspectSupport
        implements MethodInterceptor, Serializable {

    @Override
    @Nullable
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // 1. 获取目标对象的 Class
        Class<?> targetClass = (invocation.getThis() != null)
                ? AopUtils.getTargetClass(invocation.getThis()) : null;

        // 2. 调用父类 TransactionAspectSupport.invokeWithinTransaction()
        //    这是事务管理的真正核心方法
        return invokeWithinTransaction(invocation.getMethod(), targetClass,
                new CoroutinesInvocationCallback() {
                    @Override
                    @Nullable
                    public Object proceedWithInvocation() throws Throwable {
                        return invocation.proceed();
                    }
                    @Override
                    public Object getTarget() {
                        return invocation.getThis();
                    }
                    @Override
                    public Object[] getArguments() {
                        return invocation.getArguments();
                    }
                });
    }
}
```

## 11.4 TransactionAspectSupport.invokeWithinTransaction() — 事务管理核心

```
这是 Spring 声明式事务最核心的方法！所有事务的创建、提交、回滚逻辑都在这里：

@Nullable
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
        final InvocationCallback invocation) throws Throwable {

    // ★ 步骤 1: 获取 TransactionAttribute
    //    如果当前方法上有 @Transactional，返回封装了注解属性的 TransactionAttribute
    //    如果没有，则返回 null（说明此方法不需要事务管理）
    TransactionAttributeSource tas = getTransactionAttributeSource();
    final TransactionAttribute txAttr = (tas != null
            ? tas.getTransactionAttribute(method, targetClass) : null);

    // ★ 步骤 2: 获取当前线程绑定的 PlatformTransactionManager
    //    默认从 BeanFactory 中按类型获取（因此需要容器中有唯一的 PTM 实例）
    final TransactionManager tm = determineTransactionManager(txAttr);

    // 处理 Reactive 事务（WebFlux 场景），这里不展开
    if (this.reactiveAdapterRegistry != null && tm instanceof ReactiveTransactionManager rtm) {
        // ... 响应式事务处理
    }

    // ★ 步骤 3: 将 tm 强转为 PlatformTransactionManager
    PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
    final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

    if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
        // ===== 标准声明式事务执行路径 =====

        // ★ 步骤 4: 创建事务（或加入已有事务）
        //    根据传播行为决定：新建事务 / 加入已有 / 抛异常
        //    返回 TransactionInfo（封装了事务状态）
        TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

        Object retVal;
        try {
            // ★ 步骤 5: 执行目标方法（进入下一个 AOP 拦截器 → 最终到达业务方法）
            retVal = invocation.proceedWithInvocation();
        } catch (Throwable ex) {
            // ★ 步骤 6: 异常回滚
            //   completeTransactionAfterThrowing(txInfo, ex)
            //   内部判断该异常是否需要回滚（根据 rollbackFor/noRollbackFor）
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        } finally {
            // ★ 步骤 7: 清理线程绑定的 TransactionInfo
            cleanupTransactionInfo(txInfo);
        }

        if (retVal != null && txAttr != null) {
            // 处理 vavr 的 Try 等特殊返回类型（类似 CompletableFuture 的场景）
        }

        // ★ 步骤 8: 提交事务
        //   commitTransactionAfterReturning(txInfo)
        //   内部调用 ptm.commit(txInfo.getTransactionStatus())
        commitTransactionAfterReturning(txInfo);
        return retVal;

    } else {
        // ===== Callback 模式事务执行路径（较少使用）=====
        // 适用于编程式事务场景
        // ...
    }
}
```

## 11.5 createTransactionIfNecessary() — 事务的创建与传播

```
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
        @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

    // 1. 如果 TransactionAttribute 没有名字，设置一个默认的 joinpointIdentification
    if (txAttr != null && txAttr.getName() == null) {
        txAttr = new DelegatingTransactionAttribute(txAttr) {
            @Override
            public String getName() {
                return joinpointIdentification;
            }
        };
    }

    TransactionStatus status = null;
    if (txAttr != null) {
        if (tm != null) {
            // ★ 2. 调用 PlatformTransactionManager.getTransaction(txAttr)
            //    这是事务传播行为的核心处理逻辑！
            status = tm.getTransaction(txAttr);
        } else {
            // ... 没有事务管理器，不打日志
        }
    }

    // 3. 包装为 TransactionInfo，绑定到当前线程
    return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```

## 11.6 AbstractPlatformTransactionManager.getTransaction() — 传播行为处理

```
这是事务传播行为的真正实现！位于 AbstractPlatformTransactionManager：

@Override
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
        throws TransactionException {

    // 1. 尝试获取当前线程绑定的 TransactionSynchronizationManager 中的事务
    //    doGetTransaction() 是抽象方法，由 DataSourceTransactionManager 等子类实现
    //    具体逻辑：从 TransactionSynchronizationManager 中获取线程绑定的 ConnectionHolder
    Object transaction = doGetTransaction();

    // 2. 如果当前方法已经有事务属性和事务对象，根据传播行为判断是否使用已有事务
    boolean debugEnabled = logger.isDebugEnabled();

    if (definition == null) {
        definition = new DefaultTransactionDefinition();
    }

    // ★ 3. 检查是否已经有已经存在的事务（从 DataSource 中获取 Connection）
    if (isExistingTransaction(transaction)) {
        // ★ 已有事务存在，根据传播行为处理
        return handleExistingTransaction(definition, transaction, debugEnabled);
    }

    // ★ 4. 检查事务超时设置
    if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
    }

    // ★ 5. 没有任何已存在的事务，传播行为为 MANDATORY 时需要抛异常
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
                "No existing transaction found for transaction marked with propagation 'mandatory'");
    }

    // ★ 6. 需要创建新事务：REQUIRED / REQUIRES_NEW / NESTED
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED
            || definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW
            || definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {

        // SUSPEND 挂起当前事务（如果有）—— 如果是 REQUIRES_NEW
        SuspendedResourcesHolder suspendedResources = suspend(null);
        try {
            // ★ 开启新事务
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            // ★ 调用子类的 doBegin() 方法 —— DataSourceTransactionManager 中从 DataSource 获取连接
            doBegin(transaction, definition);
            // ★ 准备事务同步（ThreadLocal 绑定连接、事务名称等）
            prepareSynchronization(status, definition);
            return status;
        } catch (RuntimeException | Error ex) {
            resume(null, suspendedResources);
            throw ex;
        }
    } else {
        // ★ 7. 不需要事务（或不需要事务连接）：SUPPORTS / NOT_SUPPORTED / NEVER
        //    创建空的 TransactionStatus，不开启事务
        boolean newSynchronization = (getTransactionSynchronization()
                == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(definition, null, true, newSynchronization,
                debugEnabled, null);
    }
}
```

### 11.6.1 handleExistingTransaction() — 处理已存在事务时的传播

```
private TransactionStatus handleExistingTransaction(
        TransactionDefinition definition, Object transaction, boolean debugEnabled)
        throws TransactionException {

    // ★ PROPAGATION_NEVER: 不允许事务，抛异常
    if (definition.getPropagationBehavior()
            == TransactionDefinition.PROPAGATION_NEVER) {
        throw new IllegalTransactionStateException(
                "Existing transaction found for transaction marked with propagation 'never'");
    }

    // ★ PROPAGATION_NOT_SUPPORTED: 不支持事务，挂起当前事务
    if (definition.getPropagationBehavior()
            == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
        Object suspendedResources = suspend(transaction);
        boolean newSynchronization = (getTransactionSynchronization()
                == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(definition, null, false,
                newSynchronization, debugEnabled, suspendedResources);
    }

    // ★ PROPAGATION_REQUIRES_NEW: 挂起当前事务，开启新事务
    if (definition.getPropagationBehavior()
            == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
        SuspendedResourcesHolder suspendedResources = suspend(transaction);
        try {
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, true, debugEnabled, suspendedResources);
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        } catch (RuntimeException | Error beginEx) {
            resumeAfterBeginException(transaction, suspendedResources, beginEx);
            throw beginEx;
        }
    }

    // ★ PROPAGATION_NESTED: 嵌套事务（需要 JDBC 3.0 Savepoint 支持）
    if (definition.getPropagationBehavior()
            == TransactionDefinition.PROPAGATION_NESTED) {
        // 使用 savepoint，不真正开启新事务
        if (useSavepointForNestedTransaction()) {
            DefaultTransactionStatus status = prepareTransactionStatus(
                    definition, transaction, false, false, debugEnabled, null);
            status.createAndHoldSavepoint();
            return status;
        } else {
            // 如果不支持 savepoint 但允许嵌套，走 REQUIRES_NEW 逻辑
            // ...
        }
    }

    if (debugEnabled) {
        // ... PROPAGATION_REQUIRED / SUPPORTS / MANDATORY 的情况：加入已有事务
    }
    boolean newSynchronization = (getTransactionSynchronization()
            != SYNCHRONIZATION_NEVER);
    return prepareTransactionStatus(definition, transaction, false,
            newSynchronization, debugEnabled, null);
}
```

## 11.7 DataSourceTransactionManager.doBegin() — 真正开启事务

```
这是真正开启数据库事务的地方！从 DataSource 获取 Connection，设置 autoCommit=false：

@Override
protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    Connection con = null;

    try {
        // 1. 如果事务对象中还没有 Connection，从 DataSource 获取一个
        if (!txObject.hasConnectionHolder()
                || txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            // ★ 从 DataSource 获取新的 Connection
            Connection newCon = obtainDataSource().getConnection();
            // ★ 用 ConnectionHolder 包装，并绑定到当前线程
            txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
        }

        txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
        con = txObject.getConnectionHolder().getConnection();

        // 2. ★ 设置事务隔离级别
        //    调用 con.getTransactionIsolation() 读取当前隔离级别
        //    如果 definition 中指定了隔离级别且与当前不同，则更改
        Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(
                con, definition);
        txObject.setPreviousIsolationLevel(previousIsolationLevel);
        txObject.setReadOnly(definition.isReadOnly());

        // 3. ★★★ 关键步骤：将 autoCommit 设置为 false ★★★
        //    这是事务开启的标志！
        //    如果发起的 DML 没有被显式提交，在连接关闭时会被回滚
        if (con.getAutoCommit()) {
            txObject.setMustRestoreAutoCommit(true);
            con.setAutoCommit(false);
        }

        // 4. 准备事务性连接（如果是只读，可能在连接级优化）
        prepareTransactionalConnection(con, definition);

        // 5. ★ 标记 ConnectionHolder 与事务的同步关系
        txObject.getConnectionHolder().setTransactionActive(true);

        // 6. 设置事务超时（转换为数据库级别的 Statement.setQueryTimeout）
        int timeout = determineTimeout(definition);
        if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
            txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
        }

        // 7. ★ 将 ConnectionHolder 绑定到当前线程
        //    同一个线程内的所有数据库操作将共享这个连接
        if (txObject.isNewConnectionHolder()) {
            TransactionSynchronizationManager.bindResource(
                    obtainDataSource(), txObject.getConnectionHolder());
        }
    } catch (Throwable ex) {
        if (txObject.isNewConnectionHolder()) {
            DataSourceUtils.releaseConnection(con, obtainDataSource());
            txObject.setConnectionHolder(null, false);
        }
        throw new CannotCreateTransactionException(
                "Could not open JDBC Connection for transaction", ex);
    }
}
```

## 11.8 事务提交：commit() 和 doCommit()

```
// AbstractPlatformTransactionManager.commit()
@Override
public final void commit(TransactionStatus status) throws TransactionException {
    if (status.isCompleted()) {
        throw new IllegalTransactionStateException(
                "Transaction is already completed - do not call commit or rollback more than once per transaction");
    }

    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;

    // 1. 如果事务被标记为 RollbackOnly（可能是内层事务标记的）
    if (defStatus.isLocalRollbackOnly()) {
        // ★ 不走 commit，走 rollback
        processRollback(defStatus, false);
        return;
    }

    // 2. 全局 RollbackOnly（由全局事务协调器设置，如 JTA）
    if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
        processRollback(defStatus, true);
        return;
    }

    // 3. ★ 执行提交
    processCommit(defStatus);
}

// processCommit
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    try {
        boolean beforeCompletionInvoked = false;

        try {
            boolean unexpectedRollback = false;
            // 1. 事务同步管理器触发：beforeCommit()
            prepareForCommit(status);
            // 2. ★ 事务同步管理器触发：beforeCompletion()
            triggerBeforeCommit(status);
            triggerBeforeCompletion(status);
            beforeCompletionInvoked = true;

            // 3. 处理 savepoint
            if (status.hasSavepoint()) {
                unexpectedRollback = status.isGlobalRollbackOnly();
                status.releaseHeldSavepoint();
            }
            // 4. 处理新事务
            else if (status.isNewTransaction()) {
                unexpectedRollback = status.isGlobalRollbackOnly();
                // ★ 调用子类 doCommit() — DataSourceTransactionManager 中调用 con.commit()
                doCommit(status);
            }
            // 5. 处理并发回滚
            else if (isFailEarlyOnGlobalRollbackOnly()) {
                unexpectedRollback = status.isGlobalRollbackOnly();
            }

            if (unexpectedRollback) {
                throw new UnexpectedRollbackException(
                        "Transaction silently rolled back because it has been marked as rollback-only");
            }
        } catch (UnexpectedRollbackException ex) {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
            throw ex;
        } catch (TransactionException ex) {
            if (isRollbackOnCommitFailure()) {
                doRollbackOnCommitException(status, ex);
            } else {
                triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            }
            throw ex;
        }

        // ★ 事务同步管理器触发：afterCommit()
        triggerAfterCommit(status);
    } finally {
        // ★ 事务同步管理器触发：afterCompletion(status.COMMITTED)
        triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
    }
    finally {
        // 清理线程绑定资源
        cleanupAfterCompletion(status);
    }
}

// DataSourceTransactionManager.doCommit() — 真正的 JDBC 提交
@Override
protected void doCommit(DefaultTransactionStatus status) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
    Connection con = txObject.getConnectionHolder().getConnection();
    try {
        // ★ JDBC 原生提交：connection.commit()
        con.commit();
    } catch (SQLException ex) {
        throw translateException("JDBC commit", ex);
    }
}
```

## 11.9 事务回滚：rollback() 和 doRollback()

```
// DataSourceTransactionManager.doRollback() — 真正的 JDBC 回滚
@Override
protected void doRollback(DefaultTransactionStatus status) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
    Connection con = txObject.getConnectionHolder().getConnection();
    try {
        // ★ JDBC 原生回滚：connection.rollback()
        con.rollback();
    } catch (SQLException ex) {
        throw translateException("JDBC rollback", ex);
    }
}

// completeTransactionAfterThrowing — 异常触发回滚
protected void completeTransactionAfterThrowing(
        @Nullable TransactionInfo txInfo, Throwable ex) {
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        // ★ 判断当前异常的 class 是否匹配 rollbackFor 和 noRollbackFor 配置
        if (txInfo.transactionAttribute != null
                && txInfo.transactionAttribute.rollbackOn(ex)) {
            try {
                // rollbackOn 返回 true：执行回滚
                txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
            } catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            } catch (RuntimeException | Error ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                throw ex2;
            }
        } else {
            // rollbackOn 返回 false：不回滚，仍然提交
            try {
                txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
            } catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by commit exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            } catch (RuntimeException | Error ex2) {
                logger.error("Application exception overridden by commit exception", ex);
                throw ex2;
            }
        }
    }
}
```

## 11.10 TransactionSynchronizationManager — 线程绑定机制

```
TransactionSynchronizationManager 是事务管理的"数据总管"。它使用 ThreadLocal 来存储
每个线程的事务相关资源，保证同一线程内多个数据库操作共享同一个数据库连接。

public abstract class TransactionSynchronizationManager {

    // ★ 核心：存储所有 DataSource 对应的 ConnectionHolder
    //   key: DataSource 实例（如 DruidDataSource）
    //   value: ConnectionHolder（包装了 Connection + TransactionActive 标记）
    //   一个线程可能操作多个数据库，这是 Map 存储的原因
    private static final ThreadLocal<Map<Object, Object>> resources =
            new NamedThreadLocal<>("Transactional resources");

    // 事务同步回调列表（afterCommit、afterCompletion 等回调）
    private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
            new NamedThreadLocal<>("Transaction synchronizations");

    // 当前事务名称
    private static final ThreadLocal<String> currentTransactionName =
            new NamedThreadLocal<>("Current transaction name");

    // 当前事务是否只读
    private static final ThreadLocal<Boolean> currentTransactionReadOnly =
            new NamedThreadLocal<>("Current transaction read-only");

    // 当前事务隔离级别
    private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
            new NamedThreadLocal<>("Current transaction isolation level");

    // 当前事务是否激活
    private static final ThreadLocal<Boolean> actualTransactionActive =
            new NamedThreadLocal<>("Actual transaction active");

    // ★ 绑定资源到当前线程
    public static void bindResource(Object key, Object value) throws IllegalStateException {
        // 如果提供了 actualKey，则用 actualKey 作为绑定键
        Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
        Assert.notNull(value, "Value must not be null");
        // 获取当前线程的 resources Map
        Map<Object, Object> map = resources.get();
        if (map == null) {
            map = new HashMap<>();
            resources.set(map);
        }
        Object oldValue = map.put(actualKey, value);
        // 防止重复绑定
        if (oldValue != null) {
            throw new IllegalStateException("Already value [" + oldValue + "] for key ["
                    + actualKey + "] bound to thread");
        }
    }

    // ★ 获取当前线程绑定的资源
    @Nullable
    public static Object getResource(Object key) {
        Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
        return doGetResource(actualKey);
    }

    @Nullable
    private static Object doGetResource(Object actualKey) {
        Map<Object, Object> map = resources.get();
        if (map == null) {
            return null;
        }
        return map.get(actualKey);
    }
}
```

## 11.11 @Transactional 注解解析：SpringTransactionAnnotationParser

```
@Transactional 注解的属性如何被解析为 TransactionAttribute？

AnnotationTransactionAttributeSource 内部持有多个 TransactionAnnotationParser：

public class AnnotationTransactionAttributeSource
        extends AbstractFallbackTransactionAttributeSource {

    // 多个解析器按优先级排列
    private static final Set<TransactionAnnotationParser> DEFAULT_ANNOTATION_PARSERS =
            new LinkedHashSet<>(3);

    static {
        DEFAULT_ANNOTATION_PARSERS.add(new SpringTransactionAnnotationParser());  // Spring 的
        DEFAULT_ANNOTATION_PARSERS.add(new JtaTransactionAnnotationParser());    // JTA 的
        DEFAULT_ANNOTATION_PARSERS.add(new Ejb3TransactionAnnotationParser());   // EJB3 的
    }
}

SpringTransactionAnnotationParser.parseTransactionAnnotation():

protected TransactionAttribute parseTransactionAnnotation(AnnotationAttributes attributes) {
    RuleBasedTransactionAttribute rbta = new RuleBasedTransactionAttribute();

    // propagation → 解析为 Propagation 枚举
    Propagation propagation = attributes.getEnum("propagation");
    rbta.setPropagationBehavior(propagation.value());

    // isolation → 解析为 Isolation 枚举
    Isolation isolation = attributes.getEnum("isolation");
    rbta.setIsolationLevel(isolation.value());

    // timeout → 超时秒数
    rbta.setTimeout(attributes.getNumber("timeout").intValue());
    String timeoutString = attributes.getString("timeoutString");
    if (StringUtils.hasText(timeoutString)) {
        // 支持占位符，如 @Transactional(timeoutString = "${tx.timeout}")
        rbta.setTimeoutString(timeoutString);
    }

    // readOnly → 是否只读
    rbta.setReadOnly(attributes.getBoolean("readOnly"));

    // rollbackFor / rollbackForClassName → 哪些异常类型需要回滚
    List<RollbackRuleAttribute> rollbackRules = new ArrayList<>();
    for (Class<?> rbRule : attributes.getClassArray("rollbackFor")) {
        rollbackRules.add(new RollbackRuleAttribute(rbRule));
    }
    for (String rbRule : attributes.getStringArray("rollbackForClassName")) {
        rollbackRules.add(new RollbackRuleAttribute(rbRule));
    }

    // noRollbackFor / noRollbackForClassName → 哪些异常不需要回滚
    for (Class<?> rbRule : attributes.getClassArray("noRollbackFor")) {
        rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
    }
    for (String rbRule : attributes.getStringArray("noRollbackForClassName")) {
        rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
    }

    rbta.setRollbackRules(rollbackRules);
    return rbta;
}
```

## 11.12 事务拦截器完整链路图

```
业务方法调用 (orderService.createOrder())
    │
    ▼
JDK/CGLIB AOP 代理
    │
    ▼
TransactionInterceptor.invoke(MethodInvocation)
    │
    ▼
TransactionAspectSupport.invokeWithinTransaction()
    │
    ├─① 解析 @Transactional 注解 → TransactionAttribute
    │   包含：propagation, isolation, timeout, readOnly, rollbackFor...
    │
    ├─② determineTransactionManager() → 获取 PlatformTransactionManager
    │
    ├─③ createTransactionIfNecessary()
    │    │
    │    └→ ptm.getTransaction(txAttr)
    │         │
    │         ├─ doGetTransaction() → TransactionSynchronizationManager.getResource(ds)
    │         │                      获取线程绑定的 ConnectionHolder
    │         │
    │         ├─ isExistingTransaction() → 检查是否已有事务
    │         │    │
    │         │    ├─ YES → handleExistingTransaction()
    │         │    │        根据传播行为决定：加入/挂起/抛异常
    │         │    │
    │         │    └─ NO  → 创建新事务
    │         │              │
    │         │              └→ doBegin() → ds.getConnection()
    │         │                              → con.setAutoCommit(false) ★
    │         │                              → TransactionSynchronizationManager.bindResource() ★
    │         │
    │         └→ prepareTransactionInfo() → 绑定到线程
    │
    ├─④ invocation.proceedWithInvocation()  ← 执行目标方法
    │    │                                    (所有 DML 共享同一个 Connection)
    │    │
    │    └→ 业务方法执行 JdbcTemplate、MyBatis 等
    │          JdbcTemplate → DataSourceUtils.getConnection(ds)
    │                       → TransactionSynchronizationManager.getResource(ds)
    │                       → 返回事务绑定的同一个 Connection ★
    │
    ├─⑤ 如果抛出异常 → completeTransactionAfterThrowing(txInfo, ex)
    │    │
    │    ├─ txAttr.rollbackOn(ex) → true → ptm.rollback(status) → con.rollback() ★
    │    │
    │    └─ txAttr.rollbackOn(ex) → false → ptm.commit(status) → con.commit()
    │                                         (即使抛异常也提交)
    │
    ├─⑥ cleanupTransactionInfo() → 解绑线程的 TransactionInfo
    │
    └─⑦ commitTransactionAfterReturning(txInfo)
         │
         └→ ptm.commit(status)
              │
              ├─ triggerBeforeCommit() / triggerBeforeCompletion()
              ├─ doCommit() → con.commit() ★
              ├─ triggerAfterCommit() / triggerAfterCompletion()
              └─ cleanupAfterCompletion()
                   │
                   ├─ TransactionSynchronizationManager.unbindResource(ds)
                   ├─ con.close() (或归还到连接池)
                   └─ 恢复 autoCommit (如果之前改过)
```

## 11.13 设计思想总结

```
1. AOP + 拦截器 双重组合模式：
   事务管理由两个层次构成：
   - 外层：BeanFactoryTransactionAttributeSourceAdvisor（切面）
           判断哪些方法需要事务增强
   - 内层：TransactionInterceptor（拦截器/MethodInterceptor）
           具体执行事务创建/提交/回滚逻辑

2. 模板方法模式：
   AbstractPlatformTransactionManager 定义了事务处理的模板流程：
   getTransaction() → commit() → rollback()
   具体的数据库操作委托给子类 DataSourceTransactionManager 实现：
   doGetTransaction(), doBegin(), doCommit(), doRollback()

3. ThreadLocal 线程级事务隔离：
   TransactionSynchronizationManager 使用 ThreadLocal 存储 ConnectionHolder，
   保证同一线程内所有数据库操作共享一个 Connection，从而实现事务的 ACID 特性。

4. 策略模式（传播行为）：
   7 种传播行为对应不同的处理策略，在同一个 getTransaction() 方法中通过
   if/else 分支实现策略切换。

5. 与 AOP 的关系：
   TransactionInterceptor 是 AOP 通知链中的一个 MethodInterceptor。
   它的位置由 order 属性控制（默认最低优先级），意味着：
   - 其他自定义 AOP 通知可能在事务之外执行
   - @Async 的事务需要在 @Transactional 之后处理（否则事务丢失）
```
