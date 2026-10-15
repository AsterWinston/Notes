# 1. MVC设计模式

```
1.1 M: Model（模型层）
  职责：业务逻辑处理 + 数据访问
  包含：Service（业务逻辑）、DAO（数据访问）、实体类（Entity/Bean/Domain）
  特征：不依赖View和Controller，可独立测试

1.2 V: View（视图层）
  职责：数据展示，用户交互
  包含：JSP、HTML、CSS、JavaScript、EL、JSTL
  特征：只负责展示，不包含业务逻辑

1.3 C: Controller（控制器层）
  职责：接收请求、调用Model处理业务、选择View展示结果
  包含：Servlet（传统）、SpringMVC的@Controller（现代）
  特征：只是"调度员"，不做具体业务

MVC的核心思想：解耦合
- View和Model之间不直接通信，通过Controller衔接
- 每一层职责明确，修改某一层不影响其他层
- 提高代码的可维护性和复用性

MVC的执行流程：
  用户请求 → Controller接收请求 → 调用Model处理业务
  → Model返回处理结果 → Controller选择View → View展示结果 → 响应给用户

用oa项目举例：
  Controller: DeptServlet（接收/list请求）
  Model:      DeptService + DeptDao（查询数据库）
  View:       list.jsp（展示部门列表）
```

# 2. 三层架构

```
三层架构是JavaWeb项目中最经典的分层设计，将整个应用划分为三层：

  展示层（Web层 / Controller层）
       ↓ 调用
  业务逻辑层（Service层）
       ↓ 调用
  数据访问层（DAO层 / 持久层）
       ↓ 操作
  数据库（MySQL / Oracle...）

2.1 数据访问层（DAO：Data Access Object）
  职责：对数据库进行增删改查（CRUD）
  命名：XxxDao / XxxMapper（MyBatis习惯）
  示例方法：selectById()、insert()、delete()、selectAll()
  特点：每个方法只做一件事，方法粒度细

2.2 业务逻辑层（Service）
  职责：处理具体的业务逻辑
  命名：XxxService / XxxServiceImpl（面向接口编程）
  示例方法：login()（需要验证用户名、密码、状态等多个步骤）
  特点：
    - 调用多个DAO方法来完成一个业务操作
    - 负责事务控制
    - 核心业务规则都在这一层

2.3 展示层（Web / Controller）
  职责：接收请求、参数校验、调用Service、跳转页面
  命名：XxxServlet / XxxController
  示例方法：doGet()、doPost()
  特点：
    - 不应该包含任何业务逻辑
    - 只做"转发"和"调度"

依赖原则：
  展示层 → 依赖 → 业务层 → 依赖 → 数据访问层
  （单向依赖，不能反向。上层可以调用下层，下层不能调用上层。）

接口隔离：
  每一层都应该定义接口（Interface），层与层之间通过接口交互。
  例如：
    DeptService接口（在Service层）
    DeptServiceImpl实现类
    DeptServlet调用 DeptService接口，而不是直接调用DeptServiceImpl

这样做的好处：方便单元测试（Mock）、方便切换实现。
```

# 3. MVC与三层架构的关系

```
MVC 和 三层架构 是两个不同维度的概念：

  MVC（设计模式）               三层架构（分层架构）
  ┌────────────────────┐      ┌─────────────────────┐
  │ Controller         │──对应──│ Web层（展示层）      │
  ├────────────────────┤      ├─────────────────────┤
  │ Model              │──对应──│ Service层 + DAO层    │
  ├────────────────────┤      ├─────────────────────┤
  │ View               │      └─────────────────────┘
  └────────────────────┘

  MVC 的 Model = 三层架构中的 Service + DAO
  MVC 的 Controller + View = 三层架构中的 Web层

  通俗理解：
    三层架构是"纵向"的划分（从上到下）
    MVC是"横向"的划分（对Web层的再细分）
```

# 4. 实体类命名规范（PO / VO / DTO / POJO / BO）

```
在JavaWeb项目中，实体类有多种叫法，每种有不同的用途：

4.1 PO（Persistent Object 持久化对象）
  与数据库表结构一一对应的对象，一个PO对象对应表中的一条记录。
  字段名和数据库字段名对应，用于DAO层的数据传递。

4.2 VO（View Object 视图对象 / 值对象）
  用于展示层的对象，通常是多个PO的属性组合。
  例如：DeptVO可能包含部门信息和部门人数统计。

4.3 DTO（Data Transfer Object 数据传输对象）
  用于层与层之间的数据传输，比如Service层返回给Controller层。
  可以减少方法参数个数，将多个参数封装成一个对象。

4.4 POJO（Plain Ordinary Java Object 简单Java对象）
  最普通的Java对象，没有继承任何类，没有实现任何接口。
  可以理解为PO/VO/DTO的统称。

4.5 BO（Business Object 业务对象）
  封装业务逻辑的对象（通常和Service对应）。

4.6 Domain（领域对象）
  领域驱动设计（DDD）中的概念，包含了属性和业务行为。

常见的分层使用规范：
  Controller 层 → 接收/返回 VO 或 DTO
  Service 层  → 使用 BO / Domain 进行业务处理
  DAO 层      → 使用 PO 与数据库交互

在实际项目中，简单场景下（比如增删改查）不必严格区分，通常使用一个"实体类"贯穿所有层。
复杂场景下（比如查询多个表的关联结果），需要额外定义VO或DTO。
```

# 5. 事务控制

```
5.1 为什么事务控制要在Service层？

原因：一个业务操作可能涉及多个DAO操作，这些DAO操作必须在同一个事务中。

举例：转账业务
  张三的账户减1000元  →  AccountDao.update(张三)
  李四的账户加1000元  →  AccountDao.update(李四)
  这两个操作必须同时成功或同时失败，不能只成功一个。

  如果在DAO层控制事务（每个DAO方法独立提交）：
    张三的钱扣了，但李四的钱没加上 —— 数据不一致！

  正确的做法：
    在Service层开启事务 → 调用多个DAO方法 → 全部成功后提交 → 任何一步失败则回滚。

5.2 ThreadLocal实现事务控制

ThreadLocal 是Java提供的一个线程局部变量工具类。
特点：同一个ThreadLocal对象，在不同的线程中存储的数据是隔离的，互不影响。

为什么需要ThreadLocal？
  - 每个线程（每个用户请求）需要独立的数据库连接（Connection）
  - Connection的提交/回滚应该在线程内完成
  - ThreadLocal可以保证同一个线程使用的是同一个Connection

ThreadLocal核心方法：
  void set(T value)     // 存储数据到当前线程
  T get()               // 获取当前线程的数据
  void remove()         // 移除当前线程的数据（防止内存泄漏！）

5.3 使用ThreadLocal + Filter实现事务控制

实现思路：
  1. 通过Filter拦截所有请求
  2. 在Filter中：获取Connection → 关闭自动提交 → 存入ThreadLocal
  3. Service层从ThreadLocal中获取Connection，执行多个DAO操作
  4. 所有DAO使用同一个Connection（从ThreadLocal获取）
  5. Service执行成功 → Filter提交事务
  6. Service执行失败 → Filter回滚事务
  7. 最终在Filter中关闭Connection，并从ThreadLocal中移除

完整代码实现：

1）工具类：ConnectionUtil —— 管理ThreadLocal

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class ConnectionUtil {
    // ThreadLocal：每个线程存储自己的Connection
    private static ThreadLocal<Connection> threadLocal = new ThreadLocal<>();
    
    // 获取当前线程的Connection
    public static Connection getConnection() throws SQLException {
        Connection conn = threadLocal.get();
        if (conn == null) {
            conn = DriverManager.getConnection(url, username, password);
            threadLocal.set(conn);
        }
        return conn;
    }
    
    // 从ThreadLocal中移除Connection（防止内存泄漏）
    public static void removeConnection() {
        Connection conn = threadLocal.get();
        if (conn != null) {
            try {
                conn.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
            threadLocal.remove();
        }
    }
}

2）过滤器：TransactionFilter —— 事务管理

@WebFilter("/*")
public class TransactionFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) resp;
        
        try {
            // 获取Connection，关闭自动提交（开启事务）
            Connection conn = ConnectionUtil.getConnection();
            conn.setAutoCommit(false);
            
            // 放行，执行Servlet（Service → DAO）
            chain.doFilter(request, response);
            
            // 如果没有异常，提交事务
            conn.commit();
            
        } catch (Exception e) {
            // 发生异常，回滚事务
            try {
                ConnectionUtil.getConnection().rollback();
            } catch (SQLException ex) {
                ex.printStackTrace();
            }
            // 跳转到错误页面
            response.sendRedirect(request.getContextPath() + "/error.jsp");
            
        } finally {
            // 无论成功与否，最终都要关闭Connection并移除
            ConnectionUtil.removeConnection();
        }
    }
}

3）DAO层 —— 使用统一的Connection

public class AccountDao {
    
    public void update(Account account) throws SQLException {
        // 从ThreadLocal中获取当前线程的Connection，保证和Service用的是同一个
        Connection conn = ConnectionUtil.getConnection();
        String sql = "update t_account set balance = ? where id = ?";
        PreparedStatement ps = conn.prepareStatement(sql);
        ps.setDouble(1, account.getBalance());
        ps.setInt(2, account.getId());
        ps.executeUpdate();
    }
}

4）Service层 —— 组合多个DAO操作

public class AccountService {
    private AccountDao accountDao = new AccountDao();
    
    public void transfer(String fromName, String toName, double money) throws SQLException {
        // 这些DAO操作使用的是同一个Connection，在同一个事务中
        Account from = accountDao.selectByName(fromName);
        Account to = accountDao.selectByName(toName);
        
        from.setBalance(from.getBalance() - money);
        to.setBalance(to.getBalance() + money);
        
        accountDao.update(from);  // 扣钱
        accountDao.update(to);    // 加钱
    }
}

5）Servlet层 —— 调用Service

@WebServlet("/transfer")
public class TransferServlet extends HttpServlet {
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) {
        AccountService service = new AccountService();
        service.transfer("张三", "李四", 1000);
        // 事务由TransactionFilter统一管理，Servlet无需关心
    }
}

5.4 事务的四大特性（ACID）

  原子性（Atomicity）：    事务中的操作要么全部成功，要么全部失败
  一致性（Consistency）：   事务执行前后，数据库从一个一致性状态变到另一个一致性状态
  隔离性（Isolation）：     多个事务并发执行时互不干扰
  持久性（Durability）：    事务一旦提交，对数据库的修改是永久性的

5.5 事务的传播行为

  如果ServiceA调用ServiceB，两者都有事务，该怎么处理？
  - ServiceB使用ServiceA的事务？还是新开一个事务？

  常见场景：
    REQUIRED：         如果当前有事务，就加入；没有就新建（最常用）
    REQUIRES_NEW：     无论如何都新建一个事务（当前事务挂起）
    SUPPORTS：         有事务就加入，没有就算了（非事务执行）
    NOT_SUPPORTED：    以非事务方式执行（当前事务挂起）
    MANDATORY：        必须在事务中执行，否则抛异常
    NEVER：            不能在事务中执行，否则抛异常
    NESTED：           嵌套事务（保存点机制，可部分回滚）

  这些概念在Spring的声明式事务中会深入使用，在纯Servlet阶段理解即可。
```

# 6. 前端控制器模式（Front Controller Pattern）

```
为什么需要前端控制器？

在纯Servlet开发中，一个请求对应一个Servlet，导致：
  - 100个请求 → 100个Servlet类 → 类爆炸
  - 每个Servlet中都有重复的公共代码（设编码、获取参数等）

前端控制器模式解决这个问题：
  - 所有请求都先经过一个统一的入口（前端控制器）
  - 前端控制器负责公共处理（编码、权限校验等）
  - 然后根据请求路径，分发给具体的处理器（方法/类）

实现方式（在Servlet中的两种方案）：

方案一：一个业务一个Servlet（模板方法设计模式）
  部门操作 → DeptServlet
    /dept/list   → list方法
    /dept/add    → add方法
    /dept/delete → delete方法
    /dept/edit   → edit方法

方案二：一个Servlet处理所有请求（简易版前端控制器）
  AllServlet：
    /dept/list   → 转发给 DeptController.list()
    /emp/list    → 转发给 EmpController.list()
    /order/save  → 转发给 OrderController.save()

SpringMVC的DispatcherServlet就是前端控制器模式的经典实现。
```

# 7. 总结：标准JavaWeb项目目录结构

```
项目名/
  ├─ src/main/java/
  │    ├─ com.xxx.controller/    --- Servlet（控制器）
  │    │    ├─ DeptServlet.java
  │    │    └─ UserServlet.java
  │    ├─ com.xxx.service/       --- 业务逻辑接口
  │    │    ├─ DeptService.java
  │    │    └─ UserService.java
  │    ├─ com.xxx.service.impl/  --- 业务逻辑实现
  │    │    ├─ DeptServiceImpl.java
  │    │    └─ UserServiceImpl.java
  │    ├─ com.xxx.dao/           --- 数据访问层接口
  │    │    ├─ DeptDao.java
  │    │    └─ UserDao.java
  │    ├─ com.xxx.dao.impl/      --- 数据访问层实现
  │    │    ├─ DeptDaoImpl.java
  │    │    └─ UserDaoImpl.java
  │    ├─ com.xxx.entity/        --- 实体类（PO/VO/DTO）
  │    │    ├─ Dept.java
  │    │    └─ User.java
  │    └─ com.xxx.util/          --- 工具类
  │         ├─ DBUtil.java       --- 数据库连接工具
  │         └─ ConnectionUtil.java -- ThreadLocal事务工具
  │
  ├─ src/main/webapp/            --- Web资源（webapp根）
  │    ├─ WEB-INF/
  │    │    ├─ web.xml           --- 部署描述符
  │    │    ├─ classes/          --- 编译后的class文件
  │    │    └─ lib/              --- 第三方jar包
  │    ├─ index.jsp              --- 首页
  │    ├─ css/                   --- 样式文件
  │    ├─ js/                    --- JavaScript文件
  │    └─ images/                --- 图片资源
  │
  └─ pom.xml（Maven项目） 或 lib目录（传统项目）
```
