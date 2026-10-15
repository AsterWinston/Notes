# 1. 当前的OA项目存在什么缺陷？

```
DeptServlet、EmpServlet、OrderServlet。每一个Servlet都是处理自己相关的业务。在这些Servlet执行之前都是需要判断用户是否登录了。如果用户登录了，可以继续操作，如果没有登录，需要用户登录。这段判断用户是否登录的代码是固定的，并且在每一个Servlet类当中都需要编写，显然代码没有得到重复利用。包括每一个Servlet都要解决中文乱码问题，也有公共的代码。这些代码目前都是重复编写，并没有达到复用。怎么解决这个问题?
可以使用Servlet规范中的Filter过滤器来解决这个问题。
```

# 2. Filter是什么，有什么用，执行原理是什么？

```
Filter是过滤器。
Filter可以在Servlet这个目标程序执行之前添加代码。也可以在目标Servlet执行之后添加代码。之前之后都可以添加过滤规则。
一般情况下，都是在过滤器当中编写公共代码。
一个过滤器怎么写呢？

第一步：编写一个Java类实现一个接口：jarkata.servlet.Filter。并且实现这个接口当中所有的方法。

init方法：在Filter对象第一次被创建之后调用，并且只调用一次。
doFilter方法：只要用户发送一次请求，则执行一次。发送N次请求，则执行N次。在这个方法中编写过滤规则。
destroy方法：在Filter对象被释放/销毁之前调用，并且只调用一次。
第二步：在web.xml文件中对Filter进行配置。这个配置和Servlet很像。

<filter>
    <filter-name>filter2</filter-name>
    <filter-class>com.bjpowernode.javaweb.servlet.Filter2</filter-class>
</filter>
<filter-mapping>
    <filter-name>filter2</filter-name>
    <url-pattern>*.do</url-pattern>
</filter-mapping>

或者使用注解：@WebFilter({"*.do"})

注意：

Servlet对象默认情况下，在服务器启动的时候是不会新建对象的。
Filter对象默认情况下，在服务器启动的时候会新建对象。
Servlet是单例的。Filter也是单例的。（单实例。）
目标Servlet是否执行，取决于两个条件：

第一：在过滤器当中是否编写了：chain.doFilter(request, response); 代码。
第二：用户发送的请求路径是否和Servlet的请求路径一致。
chain.doFilter(request, response); 这行代码的作用：

执行下一个过滤器，如果下面没有过滤器了，执行最终的Servlet。
注意：Filter的优先级，天生的就比Servlet优先级高。

/a.do 对应一个Filter，也对应一个Servlet。那么一定是先执行Filter，然后再执行Servlet。
```

# 3. 关于Filter的配置路径

```
/a.do、/b.do、/dept/save。这些配置方式都是精确匹配。
/* 匹配所有路径。
*.do 后缀匹配。不要以 / 开始
/dept/* 前缀匹配。
在web.xml文件中进行配置的时候，Filter的执行顺序是什么？

依靠filter-mapping标签的配置位置，越靠上优先级越高。
过滤器的调用顺序，遵循栈数据结构。

使用@WebFilter的时候，Filter的执行顺序是怎样的呢？

执行顺序是：比较Filter这个类名。
比如：FilterA和FilterB，则先执行FilterA。
比如：Filter1和Filter2，则先执行Filter1.
Filter的生命周期？

和Servlet对象生命周期一致。
唯一的区别：Filter默认情况下，在服务器启动阶段就实例化。Servlet不会。
Filter过滤器这里有一个设计模式：

责任链设计模式。
过滤器最大的优点：
在程序编译阶段不会确定调用顺序。因为Filter的调用顺序是配置到web.xml文件中的，只要修改web.xml配置文件中filter-mapping的顺序就可以调整Filter的执行顺序。显然Filter的执行顺序是在程序运行阶段动态组合的。那么这种设计模式被称为责任链设计模式。
责任链设计模式最大的核心思想：
在程序运行阶段，动态的组合程序的调用顺序。
```

# 4. @WebFilter

```
属性	类型	说明
filterName	String	过滤器名称，默认类名
urlPatterns / value	String[]	匹配 URL 模式 (value 是别名)
dispatcherTypes	DispatcherType[]	过滤器拦截类型：REQUEST, FORWARD, INCLUDE, ERROR, ASYNC
asyncSupported	boolean	是否支持异步请求
initParams	@WebInitParam[]	初始化参数（替代 web.xml <init-param>）
```

# 5. Filter实战示例

```
示例1：字符编码过滤器（解决全站中文乱码）

@WebFilter("/*")
public class CharacterEncodingFilter implements Filter {
    
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        // 设置请求体字符集（处理POST请求乱码）
        request.setCharacterEncoding("UTF-8");
        // 设置响应字符集
        response.setContentType("text/html;charset=UTF-8");
        // 放行
        chain.doFilter(request, response);
    }
    
    @Override
    public void destroy() {
    }
}

示例2：登录验证过滤器（拦截未登录用户）

@WebFilter("/*")
public class LoginFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) resp;
        
        // 获取当前请求路径
        String uri = request.getRequestURI();
        
        // 放行登录相关的页面和请求（白名单）
        if (uri.endsWith("login.jsp") || uri.endsWith("/login") 
                || uri.endsWith(".css") || uri.endsWith(".js") 
                || uri.endsWith(".png") || uri.endsWith(".jpg")) {
            chain.doFilter(request, response);
            return;
        }
        
        // 从session中获取用户信息
        Object user = request.getSession().getAttribute("user");
        
        if (user != null) {
            // 已登录，放行
            chain.doFilter(request, response);
        } else {
            // 未登录，跳转到登录页面
            response.sendRedirect(request.getContextPath() + "/login.jsp");
        }
    }
    
    @Override
    public void init(FilterConfig config) {}
    @Override
    public void destroy() {}
}

示例3：敏感词过滤（通过装饰者模式包装request实现）

原理：自定义一个RequestWrapper继承HttpServletRequestWrapper，重写getParameter方法，
在getParameter返回值中对敏感词进行替换。

public class SensitiveWordsFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        // 使用自定义的包装类替换原request
        HttpServletRequest wrapper = new MyRequestWrapper(request);
        chain.doFilter(wrapper, resp);
    }
}

class MyRequestWrapper extends HttpServletRequestWrapper {
    public MyRequestWrapper(HttpServletRequest request) {
        super(request);
    }
    
    @Override
    public String getParameter(String name) {
        String value = super.getParameter(name);
        if (value != null) {
            value = value.replaceAll("笨蛋", "***");
        }
        return value;
    }
}

说明：HttpServletRequestWrapper是Servlet规范提供的请求包装类（装饰者设计模式）。
我们继承它，只重写需要修改的方法即可。这样下层Servlet调用getParameter时
获取到的就是已经过滤后的数据了。
```

# 6. Filter执行顺序总结

```
1. web.xml配置方式：按<filter-mapping>标签出现的先后顺序执行。
2. @WebFilter注解方式：按Filter类名的字典序执行（FilterA在FilterB之前）。

3. 执行顺序遵循栈数据结构（先进后出 FILO）：
   请求进来：Filter1 → Filter2 → Filter3 → Servlet
   响应出去：Servlet → Filter3 → Filter2 → Filter1

4. 多个Filter的doFilter方法执行过程示意：
   public void doFilter(req, resp, chain) {
       // ========= 请求前置处理代码 =========
       System.out.println("Filter1: 请求进来了");
       
       chain.doFilter(req, resp);  // 放行，执行下一个Filter或Servlet
       
       // ========= 响应后置处理代码 =========
       System.out.println("Filter1: 响应回去了");
   }
```