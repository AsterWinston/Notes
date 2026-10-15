# 1. Listener监听器

```
什么是监听器？

监听器是 Servlet 规范中的一员。就像 Filter 一样。Filter 也是 Servlet 规范中的一员。
在 Servlet 中，所有的监听器接口都是以“Listener”结尾。
监听器有什么用？

监听器实际上是 Servlet 规范留给我们 javaweb 程序员的特殊时机。
特殊的时刻如果想执行这段代码，你需要想到使用对应的监听器。
```

# 2. Servlet规范中提供了哪些监听器？

```
jakarta.servlet 包下：
ServletContextListener
ServletContextAttributeListener
ServletRequestListener
ServletRequestAttributeListener
jakarta.servlet.http 包下：
HttpSessionListener
HttpSessionAttributeListener
该监听器需要使用 @WebListener 注解进行标注。
该监听器监听的是什么？是 session 域中数据的变化。只要数据变化，则执行相应的方法。主要监测点在 session 域对象上。
HttpSessionBindingListener
该监听器不需要使用 @WebListener 进行标注。HttpSessionBindingListener 不是容器扫描的类
假设 User 类实现了该监听器，那么 User 对象在被放入 session 的时候触发 bind 事件，User 对象从 session 中删除的时候，触发 unbind 事件。
假设 Customer 类没有实现该监听器，那么 Customer 对象放入 session 或者从 session 删除的时候，不会触发 bind 和 unbind 事件。
HttpSessionIdListener
session 的 id 发生改变的时候，监听器中的唯一一个方法就会被调用。
HttpSessionActivationListener
监听 session 对象的钝化和活化的。
钝化：session 对象从内存存储到硬盘文件。
活化：从硬盘文件把 session 恢复到内存。
```

# 3. 实现一个监听器的步骤：以 ServletContextListener 为例

```
第一步：编写一个类实现 ServletContextListener 接口。并且实现里面的方法。

void contextInitialized(ServletContextEvent event)
void contextDestroyed(ServletContextEvent event)

第二步：在 web.xml 文件中对 ServletContextListener 进行配置，如下：

<listener>
    <listener-class>com.bjpowernode.javaweb.listener.MyServletContextListener</listener-class>
</listener>

当然，第二步也可以不使用配置文件，也可以用注解，例如：@WebListener

注意：所有监听器中的方法都是不需要 javaweb 程序员调用的，由服务器来负责调用？什么时候被调用呢？

当某个特殊的事件发生（特殊的事件发生其实就是某个时机到了。）之后，被 web 服务器自动调用。
思考一个业务场景：

请编写一个功能，记录该网站实时的在线用户的个数。
我们可以通过服务器端有没有分配 session 对象，因为一个 session 代表了一个用户。有一个 session 就代表有一个用户。如果你采用这种逻辑去实现的话，session 有多少个，在线用户就有多少个。这种方式的话：HttpSessionListener 够用了。session 对象只要新建，则 count++，然后将 count 存储到 ServletContext 域当中，在页面展示在线人数即可。
业务发生改变了，只统计登录的用户的在线数量，这个该怎么办？
session.setAttribute(“user”, userObj);
用户登录的标志是什么？session 中曾经存储过 User 类型的对象。那么这个时候可以让 User 类型的对象实现 HttpSessionBindingListener 监听器，只要 User 类型对象存储到 session 域中，则 count++，然后将 count++ 存储到 ServletContext 对象中。页面展示在线人数即可。
实现 oa 项目中当前登录在线的人数。

什么代表着用户登录了？
session.setAttribute(“user”, userObj); User 类型的对象只要往 session 中存储过，表示有新用户登录。
什么代表着用户退出了？
session.removeAttribute(“user”); User 类型的对象从 session 域中移除了。
或者有可能是 session 销毁了。（session 超时）
```

# 4. 各监听器实战示例

```
示例1：ServletContextListener —— 初始化全局资源

@WebListener
public class AppContextListener implements ServletContextListener {
    
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        System.out.println("应用启动了！");
        // 在这里可以：初始化数据库连接池、加载全局配置、创建定时任务等
        ServletContext context = sce.getServletContext();
        context.setAttribute("appStartTime", System.currentTimeMillis());
    }
    
    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        System.out.println("应用关闭了！");
        // 在这里释放资源：关闭连接池、停止定时任务等
    }
}

示例2：ServletRequestListener —— 监控每个请求

@WebListener
public class RequestMonitorListener implements ServletRequestListener {
    
    @Override
    public void requestInitialized(ServletRequestEvent sre) {
        HttpServletRequest request = (HttpServletRequest) sre.getServletRequest();
        String uri = request.getRequestURI();
        String ip = request.getRemoteAddr();
        System.out.println("请求开始：" + ip + " -> " + uri);
        // 记录请求开始时间
        request.setAttribute("startTime", System.currentTimeMillis());
    }
    
    @Override
    public void requestDestroyed(ServletRequestEvent sre) {
        HttpServletRequest request = (HttpServletRequest) sre.getServletRequest();
        Long startTime = (Long) request.getAttribute("startTime");
        if (startTime != null) {
            long duration = System.currentTimeMillis() - startTime;
            System.out.println("请求结束，耗时：" + duration + "ms");
        }
    }
}

示例3：HttpSessionListener —— 统计在线用户数（包含未登录的访客）

@WebListener
public class OnlineUserListener implements HttpSessionListener {
    
    @Override
    public void sessionCreated(HttpSessionEvent se) {
        // 新用户访问网站，服务器为其创建session
        ServletContext context = se.getSession().getServletContext();
        Integer count = (Integer) context.getAttribute("onlineCount");
        if (count == null) {
            count = 0;
        }
        count++;
        context.setAttribute("onlineCount", count);
        System.out.println("新用户上线，当前在线人数：" + count);
    }
    
    @Override
    public void sessionDestroyed(HttpSessionEvent se) {
        // session销毁（超时30分钟默认，或手动调用invalidate）
        ServletContext context = se.getSession().getServletContext();
        Integer count = (Integer) context.getAttribute("onlineCount");
        if (count != null && count > 0) {
            count--;
            context.setAttribute("onlineCount", count);
            System.out.println("用户下线，当前在线人数：" + count);
        }
    }
}

示例4：HttpSessionAttributeListener —— 监听session域中属性的变化

@WebListener
public class SessionAttrListener implements HttpSessionAttributeListener {
    
    @Override
    public void attributeAdded(HttpSessionBindingEvent se) {
        System.out.println("session中添加了属性：" + se.getName() + "=" + se.getValue());
    }
    
    @Override
    public void attributeRemoved(HttpSessionBindingEvent se) {
        System.out.println("session中移除了属性：" + se.getName());
    }
    
    @Override
    public void attributeReplaced(HttpSessionBindingEvent se) {
        System.out.println("session中替换了属性：" + se.getName());
        // se.getValue() 是旧值
        System.out.println("  旧值：" + se.getValue());
        // 通过session获取新值
        System.out.println("  新值：" + se.getSession().getAttribute(se.getName()));
    }
}

示例5：ServletContextAttributeListener —— 监听应用域属性变化

@WebListener
public class ContextAttrListener implements ServletContextAttributeListener {
    
    @Override
    public void attributeAdded(ServletContextAttributeEvent scae) {
        System.out.println("应用域添加了属性：" + scae.getName());
    }
    
    @Override
    public void attributeRemoved(ServletContextAttributeEvent scae) {
        System.out.println("应用域移除了属性：" + scae.getName());
    }
    
    @Override
    public void attributeReplaced(ServletContextAttributeEvent scae) {
        System.out.println("应用域替换了属性：" + scae.getName());
    }
}

示例6：HttpSessionBindingListener —— 精准统计已登录用户数

// 这个监听器不需要@WebListener注解，由实现该接口的JavaBean自行触发
public class User implements HttpSessionBindingListener {
    private String username;
    
    @Override
    public void valueBound(HttpSessionBindingEvent event) {
        System.out.println(username + " 登录了！");
        // 统计登录用户数
        ServletContext context = event.getSession().getServletContext();
        Integer loginCount = (Integer) context.getAttribute("loginCount");
        if (loginCount == null) loginCount = 0;
        context.setAttribute("loginCount", loginCount + 1);
    }
    
    @Override
    public void valueUnbound(HttpSessionBindingEvent event) {
        System.out.println(username + " 退出了！");
        ServletContext context = event.getSession().getServletContext();
        Integer loginCount = (Integer) context.getAttribute("loginCount");
        if (loginCount != null && loginCount > 0) {
            context.setAttribute("loginCount", loginCount - 1);
        }
    }
    
    // getter/setter省略...
}

// 使用场景：登录时将User对象存入session，User自动感知到登录事件
// session.setAttribute("user", user)    --> 触发 valueBound()
// session.removeAttribute("user");      --> 触发 valueUnbound()
// session.invalidate();                 --> 触发 valueUnbound()

示例7：HttpSessionActivationListener —— Session钝化与活化

// 钝化：服务器正常关闭时，将session对象序列化到硬盘文件保存
// 活化：服务器重新启动后，从硬盘文件反序列化恢复session对象
// 前提：实体类和其关联的所有对象都必须实现Serializable接口

public class Customer implements Serializable, HttpSessionActivationListener {
    
    @Override
    public void sessionWillPassivate(HttpSessionEvent se) {
        System.out.println("session即将钝化到硬盘...");
        // 可以在这里：将未保存的数据持久化到数据库
    }
    
    @Override
    public void sessionDidActivate(HttpSessionEvent se) {
        System.out.println("session已从硬盘活化恢复...");
        // 可以在这里：重新建立数据库连接等资源
    }
}

// 钝化机制配置（在 CATALINA_HOME/conf/context.xml 中）：
// <Manager className="org.apache.catalina.session.PersistentManager">
//     <Store className="org.apache.catalina.session.FileStore" directory="sessions"/>
// </Manager>
```

# 5. 监听器总结

```
监听器总共有 8 种（6+2）：

三类域对象监听器（每个域有2种，共6种）：
  ┌─ ServletContext 域  →  ServletContextListener       +  ServletContextAttributeListener
  ├─ HttpSession    域  →  HttpSessionListener           +  HttpSessionAttributeListener
  └─ ServletRequest 域  →  ServletRequestListener        +  ServletRequestAttributeListener

两种特殊监听器（关注对象本身，共2种）：
  ├─ HttpSessionBindingListener       → JavaBean 感知自己被放入/移出 session（不需要 @WebListener）
  └─ HttpSessionActivationListener    → JavaBean 感知自己被钝化/活化（不需要 @WebListener）

共通特征：
  - 域监听器（6种）：需要 @WebListener 注解 或 web.xml 注册
  - 对象监听器（2种）：由 JavaBean 实现接口即可，不需要额外注册
  - 所有监听器的方法都由 Web 容器自动调用，程序员无需手动调用
```
