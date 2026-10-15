# 1. Servlet中的三个域对象
```
请求域：request
会话域：session
应用域：application
三个域都有以下三个方法：
// 向域中存储数据
void setAttribute(String name, Object obj);

// 从域中读取数据
Object getAttribute(String name);

// 删除域中的数据
void removeAttribute(String name);

主要是通过：setAttribute + getAttribute方法来完成在域中数据的传递和共享。
```
## 1.1 request
```
接口名：HttpServletRequest
简称：request
request对象代表了一次请求。一次请求一个request。
使用请求域的业务场景：在A资源中通过转发的方式跳转到B资源，因为是转发，因此从A到B是一次请求，如果想让A资源和B资源共享同一个数据，可以将数据存储到request域中。
```
## 1.2 session
```
接口名：HttpSession
简称：session
session对象代表了一次会话。从打开浏览器开始访问，到最终浏览器关闭，这是一次完整的会话。每个会话session对象都对应一个JSESSIONID，而JSESSIONID生成后以cookie的方式存储在浏览器客户端。浏览器关闭，JSESSIONID失效，会话结束。
使用会话域的业务场景：
1. 在A资源中通过重定向的方式跳转到B资源，因为是重定向，因此从A到B是两次请求，如果想让A资源和B资源共享同一个数据，可以将数据存储到session域中。
2. 登录成功后保存用户的登录状态。
```
## 1.3 application
```
接口名：ServletContext
简称：application
application对象代表了整个web应用，服务器启动时创建，服务器关闭时销毁。对于一个web应用来说，application对象只有一个。
使用应用域的业务场景：记录网站的在线人数。
```

# 2. request域对象
```
在SpringMVC中，在request域中共享数据有以下几种方式：
1. 使用原生Servlet API方式。
2. 使用Model接口。
3. 使用Map接口。
4. 使用ModelMap类。
5. 使用ModelAndView类。
```

## 2.1 使用原生Servlet API方式
```
在Controller的方法上使用HttpServletRequest：
package com.powernode.springmvc.controller;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

/**
 * ClassName: RequestScopeTestController
 * Description:
 * Datetime: 2024/3/18 15:20
 * Author: 老杜@动力节点
 * Version: 1.0
 */
@Controller
public class RequestScopeTestController {

    @RequestMapping("/testServletAPI")
    public String testServletAPI(HttpServletRequest request){
        // 向request域中存储数据
        request.setAttribute("testRequestScope", "在SpringMVC中使用原生Servlet API实现request域数据共享");
        return "view";
    }
}

页面：view.html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>view</title>
</head>
<body>
<div th:text="${testRequestScope}"></div>
</body>
</html>

超链接：
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>index</title>
</head>
<body>
<h1>Index Page</h1>
<a th:href="@{/testServletAPI}">在SpringMVC中使用原生Servlet API实现request域数据共享</a><br>
</body>
</html>

这种方式当然可以，用SpringMVC框架，不建议使用原生Servlet API。
```
## 2.2 使用Model接口
```
@RequestMapping("/testModel")
public String testModel(Model model){
// 向request域中存储数据
model.addAttribute("testRequestScope", "在SpringMVC中使用Model接口实现request域数据共享");
return "view";
}
```
## 2.3 使用Map接口
```
@RequestMapping("/testMap")
public String testMap(Map<String, Object> map){
    // 向request域中存储数据
    map.put("testRequestScope", "在SpringMVC中使用Map接口实现request域数据共享");
    return "view";
}
```

## 2.4 使用ModelMap类
```
@RequestMapping("/testModelMap")
public String testModelMap(ModelMap modelMap){
// 向request域中存储数据
modelMap.addAttribute("testRequestScope", "在SpringMVC中使用ModelMap实现request域数据共享");
return "view";
}
```
### 2.4.1 Model、Map、ModelMap的关系
```
可以在以上Model、Map、ModelMap的测试程序中将其输出，看看输出什么：
通过输出结果可以看出，无论是Model、Map还是ModelMap，底层实例化的对象都是：BindingAwareModelMap。
可以查看BindingAwareModelMap的继承结构：
...
通过继承结构可以看出：BindingAwareModelMap继承了ModelMap，而ModelMap又实现了Map接口。
另外，请看ExtendedModelMap源码
可以看出ModelMap又实现了Model接口。因此表面上是采用了不同方式，底层本质上是相同的。
SpringMVC之所以提供了这些方式，目的就是方便程序员的使用，提供了多样化的方式，可见它的重要性。
```
## 2.5 使用ModelAndView类
```
在SpringMVC框架中为了更好的体现MVC架构模式，提供了一个类：ModelAndView。这个类的实例封装了Model和View。也就是说这个类既封装业务处理之后的数据，也体现了跳转到哪个视图。使用它也可以完成request域数据共享。
@RequestMapping("/testModelAndView")
public ModelAndView testModelAndView(){
    // 创建“模型与视图对象”
    ModelAndView modelAndView = new ModelAndView();
    // 绑定数据
    modelAndView.addObject("testRequestScope", "在SpringMVC中使用ModelAndView实现request域数据共享");
    // 绑定视图
    modelAndView.setViewName("view");
    // 返回
    return modelAndView;
}

这种方式需要注意的是：
1. 方法的返回值类型不是String，而是ModelAndView对象。
2. ModelAndView不是出现在方法的参数位置，而是在方法体中new的。
3. 需要调用addObject向域中存储数据。
4. 需要调用setViewName设置视图的名字。
```
### 2.5.1 ModelAndView源码分析
```
以上我们通过了五种方式完成了request域数据共享，包括：原生Servlet API，Model、Map、ModelMap、ModelAndView

其中后四种：Model、Map、ModelMap、ModelAndView。这四种方式在底层DispatcherServlet调用我们的Controller之后，返回的对象都是ModelAndView，这个可以通过源码进行分析。

在以上四种方式中，拿Model举例，添加断点进行调试

启动服务器，发送请求，走到断点

查看DispatcherServlet的1089行，源码如下

可以看到这里，无论你使用哪种方式，最终都要返回一个ModelAndView对象。
提醒：大家可以通过以下断点调试方式，采用一级一级返回，最终可以看到都会返回ModelAndView对象。
```
## 2.6 @RequestAttribute注解读取
```
@RequestMapping("/test3")
public String test3(@RequestAttribute("msg") String msg) {
    System.out.println(msg);
    return "success";
}
```
# 3. session域对象
```
在SpringMVC中使用session域共享数据，实现方式有多种，其中比较常见的两种方式：
1. 使用原生Servlet API
2. 使用SessionAttributes注解
```

## 3.1 使用原生Servlet API
```
package com.powernode.springmvc.controller;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

/**
 * ClassName: SessionScopeTestController
 * Description:
 * Datetime: 2024/3/18 17:18
 * Author: 老杜@动力节点
 * Version: 1.0
 */
@Controller
public class SessionScopeTestController {

    @RequestMapping("/testSessionScope1")
    public String testServletAPI(HttpSession session) {
        // 向会话域中存储数据
        session.setAttribute("testSessionScope1", "使用原生Servlet API实现session域共享数据");
        return "view";
    }

}

视图页面：view.html
<div th:text="${session.testSessionScope1}"></div>

超链接：index.html
<a th:href="@{/testSessionScope1}">在SpringMVC中使用原生Servlet API实现session域共享数据</a><br>

同时也可以先获取request再获取session
```
## 3.2 使用SessionAttributes注解（只可以放到类型上）
```
使用SessionAttributes注解标注Controller：
package com.powernode.springmvc.controller;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.ModelMap;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.SessionAttributes;

/**
 * ClassName: SessionScopeTestController
 * Description:
 * Datetime: 2024/3/18 17:18
 * Author: 老杜@动力节点
 * Version: 1.0
 */
@Controller
@SessionAttributes(value = {"x", "y"})
public class SessionScopeTestController {

    @RequestMapping("/testSessionScope2")
    public String testSessionAttributes(ModelMap modelMap){
        // 向session域中存储数据
        modelMap.addAttribute("x", "我是埃克斯");
        modelMap.addAttribute("y", "我是歪");

        return "view";
    }
}
注意：SessionAttributes注解使用在Controller类上。标注了当key是 x 或者 y 时，数据将被存储到会话session中。如果没有 SessionAttributes注解，默认存储到request域中。
形参中不仅仅可以用ModelMap，还可以是Map、Model、ModelAndView
```
## 3.3 使用@SessionAttribute注解读取（只可以放到形参上）
```
@RequestMapping("/session3")
public String session3(@SessionAttribute("user") String user) {
    System.out.println(user);
    return "success";
}

```

# 5. RedirectAttributes（闪存属性 — Flash Scope）
```
在Web开发中，Post-Redirect-Get（PRG）是一种防止表单重复提交的重要模式：
用户提交POST表单 → 重定向到结果页面 → 用户刷新不会重复提交。

问题：重定向后，如何在下一个页面展示"操作成功"等提示信息？
方案：使用RedirectAttributes的addFlashAttribute()方法。

5.1 基本用法：
@PostMapping("/user/save")
public String save(User user, RedirectAttributes redirectAttributes) {
    // 模拟保存用户
    System.out.println("保存用户：" + user.getName());

    // flash属性：数据存储在Session中，在下一个请求中自动获取，然后立即删除
    redirectAttributes.addFlashAttribute("msg", "保存成功！");
    redirectAttributes.addFlashAttribute("status", "success");

    // 重定向到列表页
    return "redirect:/user/list";
}

@GetMapping("/user/list")
public String list(Model model) {
    // 无需手动从Session获取，SpringMVC会自动将flash属性放入Model
    // model中已包含 msg 和 status
    return "user_list";
}

视图页面 user_list.html：
<div th:if="${msg}" th:text="${msg}" style="color: green;">保存成功！</div>

5.2 Flash属性的生命周期：
1. 调用 addFlashAttribute() 将数据暂存到Session
2. 浏览器收到302重定向并请求新URL
3. SpringMVC从Session取出flash数据，放入新请求的Model中
4. 从Session中删除flash数据（生命周期结束）

5.3 RedirectAttributes vs addAttribute：
@PostMapping("/user/save")
public String save(User user, RedirectAttributes redirectAttributes) {
    // addFlashAttribute：数据走Session，URL中看不到，适合提示消息
    redirectAttributes.addFlashAttribute("msg", "保存成功");

    // addAttribute：数据拼接到URL上（?name=张三），适合查询参数
    redirectAttributes.addAttribute("name", user.getName());

    return "redirect:/user/list";
    // 最终重定向URL：/user/list?name=张三
    // Flash属性 msg 不会出现在URL中
}

5.4 原理简述：
Flash属性的底层实现：
● FlashMapManager 负责管理flash属性
● 默认使用 SessionFlashMapManager（存储在Session中）
● addFlashAttribute() → 存入Session的FlashMap
● 下一个请求 → FlashMapManager取出并清理
```

# 6. @SessionAttributes的陷阱与最佳实践
```
@SessionAttributes看似方便，但使用不当会造成数据污染和内存泄漏。

6.1 陷阱一：忘记清理Session数据
@Controller
@SessionAttributes("user")  // 将user对象放到Session中
public class UserController {

    @RequestMapping("/user/edit")
    public String edit(Model model) {
        model.addAttribute("user", userService.getCurrentUser());  // ↓ 进了Session
        return "user_edit";
    }

    @RequestMapping("/user/update")
    public String update(@ModelAttribute("user") User user) {
        userService.update(user);
        // 更新完成后，user还在Session中！如果用户再访问其他页面修改其他用户...
        return "ok";
    }
}

正确做法：使用SessionStatus.setComplete()清理
@RequestMapping("/user/update")
public String update(@ModelAttribute("user") User user, SessionStatus sessionStatus) {
    userService.update(user);
    sessionStatus.setComplete();  // 清除该Controller关联的所有Session属性
    return "redirect:/user/list";
}

6.2 陷阱二：@SessionAttributes + redirect导致数据丢失
@Controller
@SessionAttributes("user")
public class UserController {

    @RequestMapping("/user/save")
    public String save(Model model) {
        model.addAttribute("user", new User());  // 放入Session
        return "redirect:/user/confirm";  // ← 重定向后@SessionAttributes绑定可能失效
    }
}

解释：@SessionAttributes只在同一个Controller的处理方法间有效。
重定向后的请求可能由不同的Controller处理，导致@SessionAttributes找不到匹配的属性名。

建议：
● 尽量使用原生 HttpSession 或 RedirectAttributes 代替 @SessionAttributes
● 如果必须使用 @SessionAttributes，时刻记住在适当的时候调用 SessionStatus.setComplete()
● @SessionAttributes 适合向导式表单（多步骤表单），普通场景慎用
```

# 7. @ModelAttribute深入使用
```
@ModelAttribute既可以标注在方法参数上，也可以标注在方法上，功能细致且强大。

7.1 标注在方法参数上（已知用法）：
@RequestMapping("/save")
public String save(@ModelAttribute("user") User user) {
    // 从Model中获取名为"user"的属性，并用请求参数覆盖其值
    userService.save(user);
    return "ok";
}

7.2 标注在方法上 — 预加载引用数据：
在每个请求方法执行前自动执行，将数据放入Model供视图使用：

@ModelAttribute("cityList")
public List<String> loadCities() {
    // 该数据在每个请求中都会自动放入Model
    return Arrays.asList("北京", "上海", "广州", "深圳");
}

@ModelAttribute("genderMap")
public Map<Integer, String> loadGenders() {
    Map<Integer, String> map = new LinkedHashMap<>();
    map.put(1, "男");
    map.put(2, "女");
    map.put(3, "其他");
    return map;
}

// 在视图表单中可以直接使用
<select name="city">
    <option th:each="city : ${cityList}" th:value="${city}" th:text="${city}"></option>
</select>

7.3 @ModelAttribute + 方法参数（不指定value）：
@Controller
@RequestMapping("/abc")
public class SomeController {

    @ModelAttribute
    public User preloadUser(@RequestParam(required = false) Long id) {
        if (id != null) {
            return userService.findById(id);  // 编辑时从数据库加载
        }
        return new User();  // 新增时返回空对象
    }

    @RequestMapping("/edit")
    public String edit(@ModelAttribute User user) {
        // user已被preloadUser预加载，同时还合并了请求参数
        return "user_form";
    }
}

7.4 默认命名规则：
● 返回值为User类型 → Model key = "user"（类名首字母小写）
● 返回值为List<String> → Model key = "stringList"
● 建议始终显式指定名称：@ModelAttribute("myKey")
```

# 8. Application作用域的线程安全问题
```
ServletContext（Application域）是在整个Web应用范围内共享的，所有用户的所有请求共享同一个空间，
因此存在并发修改的安全隐患。

8.1 线程不安全示例：
@Controller
public class AppController {

    @Resource
    private ServletContext servletContext;

    // 危险：多个用户同时访问 → 竞态条件
    @RequestMapping("/dangerous")
    public String incrementCount() {
        Integer count = (Integer) servletContext.getAttribute("visitCount");
        if (count == null) {
            count = 0;
        }
        count = count + 1;  // 读-改-写操作不是原子性的
        servletContext.setAttribute("visitCount", count);
        return "ok";
    }
}

8.2 线程安全解决方案一：使用synchronized关键字
@RequestMapping("/sync1")
public String incrementCountSafe() {
    synchronized (servletContext) {
        Integer count = (Integer) servletContext.getAttribute("visitCount");
        if (count == null) count = 0;
        servletContext.setAttribute("visitCount", count + 1);
    }
    return "ok";
}

8.3 线程安全解决方案二：使用AtomicInteger
@Controller
public class AppController {

    @Resource
    private ServletContext servletContext;

    @PostConstruct
    public void init() {
        servletContext.setAttribute("atomicCount", new AtomicInteger(0));
    }

    @RequestMapping("/atomic")
    public String atomicCount() {
        AtomicInteger counter = (AtomicInteger) servletContext.getAttribute("atomicCount");
        int newValue = counter.incrementAndGet();  // 原子操作
        return "ok";
    }
}

8.4 建议：
● Application域中尽量存储只读数据（如：字典数据、配置信息等）
● 如果需要存储可变共享数据，要保证线程安全
● 集群/分布式部署时，Application域只存在于当前JVM实例中，不适用于跨实例共享
  → 考虑使用Redis等分布式缓存代替Application域
```

# 9. Thymeleaf作用域工具对象
```
Thymeleaf提供了便捷的作用域访问方式，无需在Controller中将数据放入Model。

9.1 直接访问Request域：
视图页面中直接使用 #request 对象：
<div th:text="${#request.getAttribute('user')}"></div>
<div th:text="${#request.getParameter('id')}"></div>
<div th:text="${#request.getContextPath()}"></div>
<div th:text="${#request.getRequestURI()}"></div>

9.2 直接访问Session域：
<div th:text="${session.user}"></div>
<div th:text="${session.loginTime}"></div>
<div th:if="${session.user == null}">
    <a th:href="@{/login}">请登录</a>
</div>

9.3 直接访问ServletContext域：
<div th:text="${application.visitCount}"></div>
<div th:text="${application.serverInfo}"></div>

9.4 Thymeleaf内置对象速查：
| 对象 | 作用 | 示例 |
|------|------|------|
| #request | 当前HttpServletRequest | ${#request.getAttribute('user')} |
| #response | 当前HttpServletResponse | |
| #session | 当前HttpSession | ${session.user} |
| #servletContext | ServletContext | ${application.count} |
| #locale | 当前Locale | ${#locale.country} |

注意：
● ${session.xxx} 直接访问Session属性，比在Controller中通过Model传递更加灵活
● ${application.xxx} 直接访问ServletContext属性
● 过度使用这些对象会降低代码可维护性，建议主要的数据传递仍通过Model完成
```

# 4. application域对象
## 4.1 使用Servlet API
```
在SpringMVC实现application域数据共享，最常见的方案就是直接使用Servlet API了：
package com.powernode.springmvc.controller;

import jakarta.servlet.ServletContext;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

/**
 * ClassName: ApplicationScopeTestController
 * Description:
 * Datetime: 2024/3/18 17:37
 * Author: 老杜@动力节点
 * Version: 1.0
 */
@Controller
public class ApplicationScopeTestController {

    @RequestMapping("/testApplicationScope")
    public String testApplicationScope(HttpServletRequest request){

        // 获取ServletContext对象
        ServletContext application = request.getServletContext();

        // 向应用域中存储数据
        application.setAttribute("applicationScope", "我是应用域当中的一条数据");

        return "view";
    }
}

视图页面：view.html
<div th:text="${application.applicationScope}"></div>

超链接：index.html
<a th:href="@{/testApplicationScope}">在SpringMVC中使用ServletAPI实现application域共享数据</a><br>
```
## 4.2 使用注入
```
@Controller
public class AppController {

    @Resource
    private ServletContext servletContext;

    @RequestMapping("/app2")
    public String app2() {
        servletContext.setAttribute("count", 100);
        return "success";
    }
}

```