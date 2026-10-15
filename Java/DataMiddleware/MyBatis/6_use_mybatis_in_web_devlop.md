# 1. 需求描述
# 2. 数据库表的设计和准备数据
# 3. 实现步骤

## 3.1 第一步：创建模块，配置tomcat，引入依赖

### 3.1.1 web.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee 
                             https://jakarta.ee/xml/ns/jakartaee/web-app_6_0.xsd"
         version="6.0">
         
</web-app>
```
### 3.1.2 pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>org.ziv</groupId>
  <artifactId>mybatis-004-web</artifactId>
  <version>1.0</version>
  <packaging>war</packaging>

  <name>mybatis-004-web Maven Webapp</name>
  <!-- FIXME change it to the project's website -->
  <url>http://www.example.com</url>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>1.7</maven.compiler.source>
    <maven.compiler.target>1.7</maven.compiler.target>
  </properties>

  <dependencies>
    <dependency>
      <groupId>jakarta.servlet</groupId>
      <artifactId>jakarta.servlet-api</artifactId>
      <version>5.0.0</version>
    </dependency>
    <dependency>
      <groupId>ch.qos.logback</groupId>
      <artifactId>logback-classic</artifactId>
      <version>1.2.11</version>
    </dependency>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>8.0.30</version>
    </dependency>
    <dependency>
      <groupId>org.mybatis</groupId>
      <artifactId>mybatis</artifactId>
      <version>3.5.1</version>
    </dependency>
  </dependencies>

  <build>
    <finalName>mybatis-004-web</finalName>
    <pluginManagement><!-- lock down plugins versions to avoid using Maven defaults (may be moved to parent pom) -->
      <plugins>
        <plugin>
          <artifactId>maven-clean-plugin</artifactId>
          <version>3.1.0</version>
        </plugin>
        <!-- see http://maven.apache.org/ref/current/maven-core/default-bindings.html#Plugin_bindings_for_war_packaging -->
        <plugin>
          <artifactId>maven-resources-plugin</artifactId>
          <version>3.0.2</version>
        </plugin>
        <plugin>
          <artifactId>maven-compiler-plugin</artifactId>
          <version>3.8.0</version>
        </plugin>
        <plugin>
          <artifactId>maven-surefire-plugin</artifactId>
          <version>2.22.1</version>
        </plugin>
        <plugin>
          <artifactId>maven-war-plugin</artifactId>
          <version>3.2.2</version>
        </plugin>
        <plugin>
          <artifactId>maven-install-plugin</artifactId>
          <version>2.5.2</version>
        </plugin>
        <plugin>
          <artifactId>maven-deploy-plugin</artifactId>
          <version>2.8.2</version>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>
</project>
```
### 3.1.3 mybatis-config.xml
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <properties resource="jdbc.properties"/>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>
<mappers>
    <mapper resource="AccountMapper.xml"/>
</mappers>
</configuration>
```
### 3.1.4 AccountMapper.xml
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="bank">
    <select id="selectByActno" resultType="com.ziv.bank.pojo.Account">
        select id,actno,balance from t_act where #{actno}
    </select>
    <update id="updateByActno">
        update t_act set(balance=#{balance}) where actno = #{actno}
    </update>
</mapper>
```
### 3.1.5 logback.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<!-- 配置文件修改时重新加载，默认true -->
<configuration debug="false">
    <!-- 控制台输出 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder" charset="UTF-8">
            <!-- 输出日志记录格式 -->
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>
    <!--    mybatis log configure-->
    <logger name="com.apache.ibatis" level="TRACE"/>
    <logger name="java.sql.Connection" level="DEBUG"/>
    <logger name="java.sql.Statement" level="DEBUG"/>
    <logger name="java.sql.PreparedStatement" level="DEBUG"/>
    <!-- 日志输出级别,LOGBACK日志级别包括五个：TRACE < DEBUG < INFO < WARN < ERROR-->
    <root level="DEBUG">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```
### 3.1.6 jdbc.properties
```
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/ziv_mybatis2
jdbc.username=root
jdbc.password=123456
```
## 3.2 第二步：前端页面Index.html
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>银行账户转账</title>
</head>
<body>
<form action="/bank/transfer" method="post">
    转出账户：<input type="text" name="fromAct"><br>
    转入账户：<input type="text" name="toAct"><br>
    转账金额：<input type="text" name="money"><br>
    <input type="submit" value="转账">
</form>
</body>
</html>

注意：web.xml中的web-app标签属性metadata-complete=“false”，其值为false表示启用注解编程，其值为true表示不开启注解。
```
## 3.3 第三步：创建包
```
com.ziv.bank.pojo
com.ziv.bank.service
com.ziv.bank.service.impl
com.ziv.bank.dao
com.ziv.bank.dao.impl
com.ziv.bank.web.controller
com.ziv.bank.exception
com.ziv.bank.utils
```
## 3.4 第四步：定义pojo类：Account 和工具类
```
package com.ziv.bank.pojo;
/**
 * 账户类，封装数据
 * @author ziv
 * @version 1.0
 * @since 1.0
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
    public Account(Long id, String actno, Double balance) {
        this.id = id;
        this.actno = actno;
        this.balance = balance;
    }
    public Account() {
    }
}

package com.ziv.bank.utils;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import java.io.IOException;

public class SqlSessionUtil {
    private static SqlSessionFactory sqlSessionFactory;

    static {
        try {
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(Resources.getResourceAsStream("mybatis-config.xml"));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
	//ThreadLocal解决事务问题
    private static ThreadLocal<SqlSession> local = new ThreadLocal<>();

    private SqlSessionUtil(){}

    /**
     * 获取会话对象
     * @return 会话对象
     */
    public static SqlSession openSession() {
        SqlSession sqlSession = local.get();
        if (sqlSession == null) {
            sqlSession = sqlSessionFactory.openSession();
            local.set(sqlSession);
        }
        return sqlSession;
    }

    /**
     * 关闭sqlSession对象
     *
     * @param sqlSession
     */
    public static void close(SqlSession sqlSession) {
        if (sqlSession != null) {
            sqlSession.close();
            local.remove();
        }
    }
}
```
## 3.5 第五步：编写AccountDao接口，以及AccountDaoImpl实现类
```
package com.ziv.bank.dao;

import com.ziv.bank.pojo.Account;
/**
 * 账户的dao对象，负责t_act的CRUD
 *
 * @author ziv
 * @version 1.0
 * @since 1.0
 */
public interface AccountDao {
    /**
     * 通过账号查询账户信息
     * @param actno 账号
     * @return 账户信息
     */
    Account selectByActno(String actno);
    /**
     * 更新账户信息
     * @param act 需要更新的账户
     * @return 1 表示更新成功，其他表示更新失败
     */
    int updateByActno(Account act);
}

package com.ziv.bank.dao.impl;

import com.ziv.bank.dao.AccountDao;
import com.ziv.bank.pojo.Account;
import com.ziv.bank.utils.SqlSessionUtil;
import org.apache.ibatis.session.SqlSession;

public class AccountDaoImpl implements AccountDao {
    @Override
    public Account selectByActno(String actno) {
        SqlSession sqlSession = SqlSessionUtil.openSession();
        Account account = (Account) sqlSession.selectOne("bank.selectByActno", actno);
        return account;
    }

    @Override
    public int updateByActno(Account act) {
        SqlSession sqlSession = SqlSessionUtil.openSession();
        int count = sqlSession.update("bank.updateByActno", act);
        return count;
    }
}
```
## 3.6 第六步：编写SQL映射文件 AccountMapper.xml
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="bank">
    <select id="selectByActno" resultType="com.ziv.bank.pojo.Account">
        select id,actno,balance from t_act where #{actno}
    </select>
    <update id="updateByActno">
        update t_act set(balance=#{balance}) where actno = #{actno}
    </update>
</mapper>
```
## 3.7 第七步：编写AccountService接⼝和AccountServiceImpl，以及异常类
```
public class MoneyNotEnoughException extends Exception {
    public MoneyNotEnoughException(){}
    public MoneyNotEnoughException(String msg){super(msg);}
}

public class TransferException extends Exception {
    public TransferException(){}
    public TransferException(String msg){super(msg);}
}

package com.ziv.bank.service;

import com.ziv.bank.exceptions.MoneyNotEnoughException;
import com.ziv.bank.exceptions.TransferException;

/**
 * 处理账户业务的接口
 *
 * @author ziv
 * @version 1.0
 * @since 1.0
 */
public interface AccountService {
    /**
     * 账户转账业务
     *
     * @param fromAct 转出账户
     * @param toAct   转入账户
     * @param money   转账金额
     */
    void transfer(String fromAct, String toAct, Double money) throws MoneyNotEnoughException, TransferException;
}

package com.ziv.bank.service.impl;

import com.ziv.bank.dao.AccountDao;
import com.ziv.bank.dao.impl.AccountDaoImpl;
import com.ziv.bank.exceptions.MoneyNotEnoughException;
import com.ziv.bank.exceptions.TransferException;
import com.ziv.bank.pojo.Account;
import com.ziv.bank.service.AccountService;
import com.ziv.bank.utils.SqlSessionUtil;
import org.apache.ibatis.session.SqlSession;

public class AccountServiceImpl implements AccountService {

    AccountDao accountDao = new AccountDaoImpl();

    @Override
    public void transfer(String fromAct, String toAct, Double money) throws MoneyNotEnoughException, TransferException {
        //添加事务代码
        SqlSession sqlSession = SqlSessionUtil.openSession();
        //判断转出账户余额是否充足
        Account fromAccount = accountDao.selectByActno(fromAct);
        Account toAccount = accountDao.selectByActno(toAct);
        //余额不足，抛出异常
        if (fromAccount.getBalance() < money) {
            throw new MoneyNotEnoughException("余额不足");
        }
        //余额充足，继续转账
        fromAccount.setBalance(fromAccount.getBalance() - money);
        toAccount.setBalance(toAccount.getBalance() + money);
        int count = accountDao.updateByActno(fromAccount);
        //int i = 12/0;
        count += accountDao.updateByActno(toAccount);
        if (count != 2) {
            throw new TransferException("转账失败");
        }
        //提交事务
        sqlSession.commit();
        //关闭事务
        sqlSession.close();
    }
}
```
## 3.8 第⼋步：编写AccountController
```
package com.ziv.bank.web;

import com.ziv.bank.exceptions.MoneyNotEnoughException;
import com.ziv.bank.exceptions.TransferException;
import com.ziv.bank.service.AccountService;
import com.ziv.bank.service.impl.AccountServiceImpl;
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

//@WebServlet("/transfer")
public class AccountController extends HttpServlet {

    AccountService accountService = new AccountServiceImpl();

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        String fromAct = request.getParameter("fromAct");
        String toAct = request.getParameter("toAct");
        double money = Double.parseDouble(request.getParameter("money"));
        try {
            accountService.transfer(fromAct, toAct, money);
            response.sendRedirect(request.getContextPath()+"/success.html");
        } catch (MoneyNotEnoughException e) {
            e.printStackTrace();
            response.sendRedirect(request.getContextPath()+"/error1.html");
        } catch (TransferException e) {
            response.sendRedirect(request.getContextPath()+"/error2.html");
            e.printStackTrace();
        }
    }
}
```

# 4. MyBatis对象作用域以及事务问题

## 4.1 SqlSessionFactoryBuilder
这个类可以被实例化、使用和丢弃，一旦创建了 SqlSessionFactory，就不再需要它了。 因此 SqlSessionFactoryBuilder 实例的最佳作用域是方法作用域（也就是局部方法变量）。 你可以重用 SqlSessionFactoryBuilder 来创建多个 SqlSessionFactory 实例，但最好还是不要一直保留着它，以保证所有的 XML 解析资源可以被释放给更重要的事情。
## 4.2 SqlSessionFactory
SqlSessionFactory 一旦被创建就应该在应用的运行期间一直存在，没有任何理由丢弃它或重新创建另一个实例。 使用 SqlSessionFactory 的最佳实践是在应用运行期间不要重复创建多次，多次重建 SqlSessionFactory 被视为一种代码“坏习惯”。因此 SqlSessionFactory 的最佳作用域是应用作用域。 有很多方法可以做到，最简单的就是使用单例模式或者静态单例模式。
## 4.3 SqlSession
每个线程都应该有它自己的 SqlSession 实例。SqlSession 的实例不是线程安全的，因此是不能被共享的，所以它的最佳的作用域是请求或方法作用域。 绝对不能将 SqlSession 实例的引用放在一个类的静态域，甚至一个类的实例变量也不行。 也绝不能将 SqlSession 实例的引用放在任何类型的托管作用域中，比如 Servlet 框架中的 HttpSession。 如果你现在正在使用一种 Web 框架，考虑将 SqlSession 放在一个和 HTTP 请求相似的作用域中。 换句话说，每次收到 HTTP 请求，就可以打开一个 SqlSession，返回一个响应后，就关闭它。 这个关闭操作很重要，为了确保每次都能执行关闭操作，你应该把这个关闭操作放到 finally 块中。 下面的示例就是一个确保 SqlSession 关闭的标准模式：
```java
try (SqlSession session = sqlSessionFactory.openSession()) {
  // 你的应用逻辑代码
}
```
```
```

# 5. 分析当前程序存在的问题
我们不难发现，这个dao实现类中的⽅法代码很固定，基本上就是⼀⾏代码，通过SqlSession对象调⽤
insert、delete、update、select等⽅法，这个类中的⽅法没有任何业务逻辑，既然是这样，这个类我们
 能不能动态的⽣成，以后可以不写这个类吗？答案：可以。

# 6. ThreadLocal 事务管理源码分析

## 6.1 为什么需要 ThreadLocal

在 Web 项目中，每个请求都在独立的线程中处理。
如果多个线程共享同一个 SqlSession，会出现线程安全问题：

线程 A：开启事务 → 执行操作 A
线程 B：开启事务 → 执行操作 B → 提交（把 A 的修改也提交了！）

因此需要每个线程拥有独立的 SqlSession 实例。
ThreadLocal 提供了线程隔离的机制。

## 6.2 SqlSessionUtil 中的 ThreadLocal

```java
public class SqlSessionUtil {
    private static SqlSessionFactory sqlSessionFactory;

    // ★ ThreadLocal：每个线程独立的 SqlSession
    private static ThreadLocal<SqlSession> local = new ThreadLocal<>();

    static {
        try {
            sqlSessionFactory = new SqlSessionFactoryBuilder()
                .build(Resources.getResourceAsStream("mybatis-config.xml"));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private SqlSessionUtil() {}

    /**
     * 获取当前线程绑定的 SqlSession
     * 如果不存在则创建并绑定
     */
    public static SqlSession openSession() {
        SqlSession sqlSession = local.get();   // ← 从 ThreadLocal 获取
        if (sqlSession == null) {
            sqlSession = sqlSessionFactory.openSession();
            local.set(sqlSession);             // ← 绑定到当前线程
        }
        return sqlSession;
    }

    /**
     * 关闭并解绑当前线程的 SqlSession
     */
    public static void close(SqlSession sqlSession) {
        if (sqlSession != null) {
            sqlSession.close();
            local.remove();   // ← ★ 必须 remove()，防止内存泄漏！
        }
    }
}
```

## 6.3 ThreadLocal 实现原理

```java
/**
 * ThreadLocal 的 set/get/remove 方法
 *
 * 每个 Thread 对象内部都有一个 ThreadLocalMap：
 *   Thread.threadLocals (类型: ThreadLocal.ThreadLocalMap)
 *
 * ThreadLocalMap 是 ThreadLocal 的静态内部类，
 * 它是一个自定义的哈希表，key 是 ThreadLocal 对象本身，
 * value 是我们存储的值。
 *
 * set(T value) 流程：
 *   1. 获取当前线程 t = Thread.currentThread()
 *   2. 获取 t 的 ThreadLocalMap
 *   3. 以当前 ThreadLocal 对象为 key，value 为值存入 map
 *
 * get() 流程：
 *   1. 获取当前线程 t
 *   2. 获取 t 的 ThreadLocalMap
 *   3. 以当前 ThreadLocal 对象为 key 查找 value
 *
 * remove() 流程：
 *   1. 获取当前线程 t
 *   2. 从 t 的 ThreadLocalMap 中移除当前 ThreadLocal 对应的 entry
 *
 * 内存泄漏问题：
 *   ThreadLocalMap 的 Entry 继承自 WeakReference，
 *   key（ThreadLocal）是弱引用。当 ThreadLocal 对象被 GC 回收后，
 *   key 变为 null，但 value 仍然存在。如果线程长时间存活
 *   （如线程池中的线程），这些 value 会一直占用内存。
 *
 *   因此 ★★★ 每次使用完必须调用 remove() ★★★
 */

// 示意图：
// Thread-1 → ThreadLocalMap → { ThreadLocal-A: SqlSession-1 }
// Thread-2 → ThreadLocalMap → { ThreadLocal-A: SqlSession-2 }
// Thread-3 → ThreadLocalMap → { ThreadLocal-A: SqlSession-3 }
//
// 同一个 ThreadLocal 对象，在不同线程中映射到不同的值
```

## 6.4 Spring 中的 SqlSessionTemplate 也是类似原理

```java
/**
 * Spring 的 SqlSessionTemplate 内部也是通过代理 + TransactionSynchronizationManager
 * （本质也是 ThreadLocal）来实现线程安全
 *
 * 对比：
 *
 *   SqlSessionUtil (手动方案):
 *     - 使用 ThreadLocal 直接存储 SqlSession
 *     - 需要手动管理生命周期
 *
 *   SqlSessionTemplate (Spring 方案):
 *     - 使用 TransactionSynchronizationManager (内部是 ThreadLocal)
 *     - 由 Spring 事务管理器自动管理生命周期
 *     - 通过 JDK 动态代理包装，每次调用自动获取/释放 SqlSession
 */
```

# 7. DAO 接口代理化 — 从 DaoImpl 到 MapperProxy
```
前面分析到 DaoImpl 中的代码非常固定和冗余，这就是 MyBatis 动态代理解决的问题。

演进历程：
  1. 手写 DaoImpl（本文件用的方式）
     → 代码冗余、维护成本高

  2. 使用 Javassist 动态生成 DaoImpl（7_javassist.md 中的方案）
     → 不需要手写实现类，但需要手动调用生成方法

  3. MyBatis 内置的 MapperProxy（JDK 动态代理）
     → SqlSession.getMapper() 一行代码获取代理对象
     → 完全透明，开发者只需要定义接口

对比：
  手写 DaoImpl：   AccountDaoImpl.java (手动维护)
  Javassist生成：  GenerateDaoProxy.generate(...) (运行时生成 .class)
```