
# 1. 概述

Web 层是 JavaEE 中最基础、最常用的规范集合。本笔记从**规范视角**总览，动手实操细节见 `JavaCore/JavaWeb` 系列笔记。

```
┌────────────────────────────────────────────┐
│              Web 请求处理栈                 │
│                                            │
│   HTTP Request                             │
│      │                                     │
│      ▼                                     │
│  Filter（过滤器）                           │
│      │                                     │
│      ▼                                     │
│  Servlet（控制器） ──► 业务逻辑 ──► 模型     │
│      │                                     │
│      ▼                                     │
│  JSP/EL/JSTL（视图渲染）                    │
│      │                                     │
│      ▼                                     │
│   HTTP Response                            │
└────────────────────────────────────────────┘
```

## 1.1 各规范定位

| 规范 | 定位 | 关键类/标签 |
|------|------|-------------|
| Servlet | 服务端处理请求/响应的核心组件 | `jakarta.servlet.Servlet` |
| JSP | 服务端模板技术，本质是 Servlet 的编译产物 | `.jsp` 文件 |
| EL | 表达式语言，在 JSP 中取值、运算 | `${user.name}` |
| JSTL | JSP 标准标签库，替代页面中的 Java 代码 | `<c:forEach>` |

# 2. Servlet 规范

## 2.1 核心接口继承体系

```
jakarta.servlet.Servlet（接口）            【爷爷】
    ▲
jakarta.servlet.GenericServlet（抽象类）   【儿子】协议无关
    ▲
jakarta.servlet.http.HttpServlet（抽象类）【孙子】HTTP 专用
```

- 开发者编写的 Servlet 继承 `HttpServlet`，重写 `doGet` / `doPost` 等方法。
- Servlet 生命周期由容器管理：**实例化 → init → service → destroy**。

## 2.2 Servlet 核心 API 一览

| API | 作用 |
|-----|------|
| `HttpServletRequest` | 封装请求：参数、Header、Cookie、Session |
| `HttpServletResponse` | 封装响应：状态码、Header、输出流 |
| `ServletConfig` | 单个 Servlet 的配置（初始化参数） |
| `ServletContext` | 应用上下文（应用域，全局共享） |
| `HttpSession` | 会话对象 |
| `Cookie` | 客户端状态 |
| `DispatcherType` | 请求分发类型：REQUEST/FORWARD/INCLUDE/ERROR/ASYNC |
| `ServletRegistration` | 动态注册 Servlet（编程式配置） |

## 2.3 版本演进

| 版本 | JavaEE 版本 | 关键特性 |
|------|-------------|----------|
| Servlet 3.0 | JavaEE 6 | 注解 `@WebServlet`、可插拔性（ServletContainerInitializer）、异步处理 |
| Servlet 3.1 | JavaEE 7 | HTTP 协议升级、非阻塞 IO（`ReadListener`/`WriteListener`） |
| Servlet 4.0 | JavaEE 8 | HTTP/2 支持、`Server Push` |
| Servlet 5.0 | Jakarta EE 9 | 包名 `javax` → `jakarta` |
| Servlet 6.0 | Jakarta EE 10 | 移除过时 API、要求 Java 17 |
| Servlet 6.1 | Jakarta EE 11 | 支持 Java 21、改进虚拟线程兼容性 |

## 2.4 注解配置 vs XML 配置

```java
@WebServlet(urlPatterns = "/hello", loadOnStartup = 1)
public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        resp.setContentType("text/html;charset=UTF-8");
        resp.getWriter().println("<h1>Hello Servlet</h1>");
    }
}
```

- 等价 XML 配置见 `JavaWeb/1_servlet.md`。
- 注解与描述符可同时存在，**描述符优先级更高**。

## 2.5 过滤器与监听器

| 组件 | 接口/注解 | 用途 |
|------|-----------|------|
| Filter | `jakarta.servlet.Filter` + `@WebFilter` | 请求前/响应后公共逻辑（编码、鉴权、日志） |
| Listener | 8 种 Listener 接口 + `@WebListener` | 监听域对象生命周期与属性变化 |

> 详细内容见 `JavaWeb/5_filter.md` 与 `JavaWeb/6_listener.md`。

## 2.6 异步处理（Async）

Servlet 3.0+ 支持异步请求处理，释放容器线程，适合长轮询、SSE、慢 IO：

```java
@WebServlet(value = "/async", asyncSupported = true)
public class AsyncServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        AsyncContext ctx = req.startAsync();
        ctx.start(() -> {
            // 在容器线程池中异步执行
            try {
                Thread.sleep(2000);
                ctx.getResponse().getWriter().println("done");
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                ctx.complete();
            }
        });
    }
}
```

# 3. JSP 规范

## 3.1 JSP 的本质

- JSP（JavaServer Pages）本质是**Servlet 的模板化封装**：首次访问时被容器翻译成 Java 源文件，再编译成 Servlet。
- 翻译产物存放在 Tomcat 的 `work` 目录。
- JSP 中嵌入的 `<% %>` Java 代码最终进入编译后的 `_jspService` 方法。

## 3.2 JSP 内置对象

| 内置对象 | 类型 | 作用域 |
|----------|------|--------|
| `pageContext` | `PageContext` | 页面作用域 |
| `request` | `HttpServletRequest` | 请求作用域 |
| `session` | `HttpSession` | 会话作用域 |
| `application` | `ServletContext` | 应用作用域 |
| `config` | `ServletConfig` | 页面配置 |
| `out` | `JspWriter` | 输出 |
| `response` | `HttpServletResponse` | 响应 |
| `page` | `Object` | 当前页面对象 |
| `exception` | `Throwable` | 异常（仅在 errorPage 中可用） |

## 3.3 JSP 的三种脚本元素

```
<%! ... %>      声明：定义成员变量/方法
<% ... %>       脚本片段：执行 Java 代码
<%= ... %>      表达式：输出表达式结果（等价于 out.print()）
```

## 3.4 指令与动作

```
<%@ page ... %>         页面指令（导包、编码、errorPage、isELIgnored 等）
<%@ include %>          静态包含（编译期合并）
<%@ taglib %>           引入标签库

<jsp:include>            动态包含（运行时）
<jsp:forward>            请求转发
<jsp:useBean> / <jsp:setProperty> / <jsp:getProperty>  操作 JavaBean
```

## 3.5 页面作用域（四大域）

| 域 | 有效范围 | 常用对象 |
|----|----------|----------|
| page | 当前 JSP 页面 | `pageContext` |
| request | 一次请求（含转发链） | `request` |
| session | 一次会话 | `session` |
| application | 整个应用 | `application` |

## 3.6 JSP 的缺陷与淘汰

- JSP 混合 HTML 与 Java，**前后端耦合严重**、难以维护与测试。
- 现代开发以**前后端分离**为主（JSON + 前端框架），或服务端渲染用模板引擎（Thymeleaf、Freemarker）。
- JSP 在 JavaEE 企业实践中已很少用于新项目，但存量系统仍大量存在。

# 4. EL 表达式语言

## 4.1 基本语法

```jsp
${表达式}
```

- 访问属性：`${user.name}`（等价 `user.getName()`）。
- 访问 Map/List/数组：`${map.key}`、`${list[0]}`。
- 三目运算：`${age >= 18 ? '成年' : '未成年'}`。
- 空值安全：对 `null` 输出空字符串，不抛异常。

## 4.2 隐式对象

| 类别 | 隐式对象 |
|------|----------|
| 作用域 | `pageScope`、`requestScope`、`sessionScope`、`applicationScope` |
| 请求 | `param`、`paramValues`、`header`、`headerValues` |
| Cookie | `cookie` |
| 初始化参数 | `initParam` |
| 其他 | `pageContext` |

## 4.3 取值优先级

EL 默认按 `pageScope → requestScope → sessionScope → applicationScope` 顺序查找。

# 5. JSTL 标准标签库

## 5.1 引入依赖

```xml
<dependency>
    <groupId>jakarta.servlet.jsp.jstl</groupId>
    <artifactId>jakarta.servlet.jsp.jstl-api</artifactId>
    <version>3.0.0</version>
</dependency>
<dependency>
    <groupId>org.glassfish.web</groupId>
    <artifactId>jakarta.servlet.jsp.jstl</artifactId>
    <version>3.0.1</version>
</dependency>
```

## 5.2 常用标签

| 标签库 | URI | 常用标签 |
|--------|-----|----------|
| core | `http://java.sun.com/jsp/jstl/core` | `c:out` `c:if` `c:choose` `c:forEach` `c:set` `c:remove` |
| fmt | `http://java.sun.com/jsp/jstl/fmt` | `fmt:formatDate` `fmt:formatNumber` |
| fn | `http://java.sun.com/jsp/jstl/functions` | `fn:length` `fn:substring` |

```jsp
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<c:forEach items="${users}" var="u" varStatus="s">
    <tr>
        <td>${s.count}</td>
        <td>${u.name}</td>
        <td>${u.age}</td>
    </tr>
</c:forEach>
```

# 6. 请求转发 vs 重定向

| 对比项 | 转发（forward） | 重定向（redirect） |
|--------|-----------------|-------------------|
| 实现 | `request.getRequestDispatcher().forward()` | `response.sendRedirect()` |
| 地址栏 | 不变 | 变化 |
| 请求次数 | 1 次（服务器内部跳转） | 2 次（浏览器发起新请求） |
| 数据共享 | 共享 request/session | 只能通过 URL/session |
| 典型场景 | 业务跳转、JSP 渲染 | 登录后跳转、POST 后重定向(PRG) |

# 7. Web 应用目录结构

```
webapp/
├── WEB-INF/
│   ├── web.xml          # 部署描述符（Servlet 3.0+ 可省略）
│   ├── classes/         # 编译后的 class
│   └── lib/             # 依赖 JAR
├── static 资源（css/js/image）
└── JSP 页面
```

- `WEB-INF` 目录对客户端不可直接访问，受保护。
- JSP 页面可以放在 `WEB-INF` 内，通过转发访问，提高安全性。

# 8. 与 Spring MVC 的对照

| 职责 | Servlet 规范 | Spring MVC |
|------|-------------|------------|
| 核心控制器 | `HttpServlet` | `DispatcherServlet` |
| 请求映射 | `@WebServlet` / web.xml | `@RequestMapping` / `@GetMapping` |
| 参数绑定 | 手动 `getParameter` | `@RequestParam` / `@PathVariable` |
| 视图渲染 | JSP/EL/JSTL | Thymeleaf 等 |
| 拦截器 | Filter | `HandlerInterceptor` + Filter 兼容 |
| 异步 | `AsyncContext` | `Callable` / `DeferredResult` / 响应式 |
| 校验 | 手动 | Bean Validation 集成 |

> 现代 Web 开发中，Servlet 通常作为 Spring MVC 的底层容器存在（`DispatcherServlet` 本身就是一个 `HttpServlet`）。
