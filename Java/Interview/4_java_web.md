# 1. 知识点
- **Servlet定义**：运行在Web服务器（如Tomcat）中的Java程序，用于处理客户端HTTP请求并生成响应；是Java EE/Jakarta EE的核心组件
- **Servlet生命周期**：init(ServletConfig)（初始化，调用一次）→ service(ServletRequest, ServletResponse)（处理每次请求，多线程调用）→ destroy()（销毁前释放资源，调用一次）；由Servlet容器管理全程
- **单实例多线程**：一个Servlet在容器中只有一个实例，多个请求由多个线程并发调用同一实例的service方法，因此Servlet必须是**线程安全的**，不应在Servlet中定义可变成员变量（除非使用ThreadLocal或加锁）
- **Servlet接口层次**：Servlet接口（顶层）→ GenericServlet（抽象类，实现Servlet，与协议无关）→ HttpServlet（抽象类，继承GenericServlet，专门处理HTTP协议）
- **HttpServlet方法分发**：service()根据请求方法类型（GET/POST/PUT/DELETE等）分发给对应的doGet()、doPost()、doPut()、doDelete()等方法
- **HttpServletRequest**：封装HTTP请求信息，包括请求行（方法/URI/协议）、请求头（User-Agent/Cookie/Accept等）、请求体（POST参数/JSON/XML）、请求参数（URL参数和表单参数，getParameter获取）
- **HttpServletResponse**：封装HTTP响应，包括设置状态码（setStatus/sendError）、响应头（setHeader/setContentType/addCookie）、响应体（getWriter字符流/getOutputStream字节流）
- **请求参数编码问题**：GET请求参数在URL中，Tomcat默认使用ISO-8859-1解码，通过server.xml中URIEncoding="UTF-8"配置解决；POST请求参数编码通过request.setCharacterEncoding("UTF-8")设置（需在第一次getParameter前调用）
- **Cookie机制**：服务器通过response.addCookie(Cookie)向浏览器发送小的文本数据，浏览器保存后每次对该域名的请求自动携带Cookie请求头；常用场景：会话标识(Session ID)、记住密码、购物车、跟踪分析等
- **Cookie关键属性**：name/value（键值对）、maxAge（存活秒数，0立即删除，负数会话结束即删）、domain（生效域名）、path（生效路径）、httpOnly（禁止JavaScript读取防XSS）、secure（仅HTTPS传输）、SameSite（防CSRF，Strict/Lax/None）
- **HttpOnly Cookie**：设`cookie.setHttpOnly(true)`后JavaScript无法通过document.cookie访问，有效防御XSS窃取Cookie的攻击
- **Secure Cookie**：设`cookie.setSecure(true)`后Cookie仅在HTTPS连接中传输，防止中间人攻击窃听
- **SameSite Cookie**：控制跨站请求是否携带Cookie，Strict（严格禁止跨站携带）、Lax（允许GET跨站携带，如链接跳转）、None（始终携带但必须配合Secure）；Chrome 80+默认Lax
- **Session会话机制**：`HttpSession session = request.getSession()`，服务器为每个用户创建Session对象存储会话数据（如登录状态、购物车），通过JSESSIONID Cookie标识用户
- **Session生命周期**：创建（首次request.getSession()或getSession(true)）、销毁（session.invalidate()手动销毁、超时自动销毁web.xml中session-timeout默认30分钟、Tomcat关闭时序化到磁盘SESSIONS.ser）
- **JSESSIONID**：Tomcat使用的默认Session标识Cookie名，值类似32位十六进制字符串，每次请求携带此Cookie，服务器根据JSESSIONID查找对应的HttpSession对象
- **URL重写(URL Rewriting)**：当浏览器禁用Cookie时，通过`response.encodeURL(url)`或`response.encodeRedirectURL(url)`在每个URL末尾追加`;jsessionid=xxx`参数，服务器识别该参数获取Session
- **Filter过滤器**：在请求到达Servlet之前和响应返回客户端之前进行预处理和后处理（如编码设置、权限校验、日志记录）；实现javax.servlet.Filter接口
- **Filter生命周期**：init(FilterConfig)（初始化一次）→ doFilter(ServletRequest, ServletResponse, FilterChain)（拦截每个匹配请求）→ destroy()（销毁前释放资源）
- **FilterChain**：多个Filter组成链，每个Filter的doFilter中调用`chain.doFilter(request, response)`传递给下一个Filter或最终Servlet，不调用则请求被拦截
- **Filter执行顺序**：按web.xml中`<filter-mapping>`的声明顺序执行（或注解`@WebFilter`时按类名字典序）；注意：请求处理顺序为f1（前）→ f2（前）→ Servlet → f2（后）→ f1（后），栈结构
- **Dispatcher类型**：`<dispatcher>REQUEST</dispatcher>`（默认，直接请求）、FORWARD（forward时过滤）、INCLUDE（include时过滤）、ERROR（错误页面时过滤）、ASYNC（异步请求时过滤）；Spring Security中常用FORWARD配合权限控制
- **Listener监听器**：监听Web应用中的事件，如ServletContext创建/销毁、HttpSession创建/销毁/属性变更、ServletRequest创建/销毁/属性变更等，共8种监听器接口
- **ServletContextListener**：监听Web应用启动和关闭，`contextInitialized`中初始化全局资源，`contextDestroyed`中释放资源；Spring的ContextLoaderListener即实现此接口来初始化Spring容器
- **HttpSessionListener**：监听Session创建(sessionCreated)和销毁(sessionDestroyed)，可用于统计在线用户数
- **ServletContext vs HttpSession vs HttpServletRequest**：作用域从大到小，context（全局一个，整个应用共享）、session（每用户一个，跨请求共享）、request（每请求一个，一次请求内共享，常用setAttribute/getAttribute传递数据）
- **转发(forward)**：服务端行为，`request.getRequestDispatcher("/path").forward(request, response)`，同一个请求对象，URL不变，浏览器不知道转发，只发一次请求；常用于MVC中Controller→JSP
- **重定向(redirect)**：客户端行为，`response.sendRedirect("/path")`，服务器返回302状态码+Location头，浏览器自动发起新请求，URL变为新地址，两次请求，Request中的数据丢失；常用于登录成功后跳转（PRG模式防重复提交）
- **302 vs 301**：302临时重定向（浏览器下次继续访问原URL），301永久重定向（浏览器缓存记录直接访问新URL，SEO权重转移）；sendRedirect默认302
- **JSP 9大内置对象**：request（HttpServletRequest）、response（HttpServletResponse）、out（JspWriter，输出到页面）、session（HttpSession）、application（ServletContext）、config（ServletConfig）、pageContext（PageContext，可获取其他8大对象）、page（当前Servlet对象this）、exception（Throwable，仅错误页面可用）
- **JSP三大指令**：`<%@ page %>`（页面属性，编码/错误页/缓冲区/import等）、`<%@ include %>`（静态包含，编译时将两个文件合并为一个Servlet）、`<%@ taglib %>`（引入标签库，如JSTL uri="http://java.sun.com/jsp/jstl/core" prefix="c"）
- **JSP动态包含vs静态包含**：`<%@ include file="%>`静态包含（编译时合并，被包含页修改后需重新编译主页面）；`<jsp:include page="">`动态包含（运行时调用，被包含页独立编译，参数可动态传递）
- **EL表达式(Expression Language)**：`${}`语法，在JSP中访问JavaBean属性、集合、隐式对象等，代替脚本表达式`<%= %>`；支持.和[]操作符、算术/比较/逻辑运算、empty关键字判空
- **JSTL(JSP Standard Tag Library)**：标准标签库，包括core(c:if/c:choose/c:forEach/c:set/c:out)、fmt(格式化日期数字)、sql(数据库操作，不推荐)、xml(XML处理)、fn(字符串函数)；JSP中不写Java代码的最佳实践
- **文件上传**：form的enctype="multipart/form-data"，Servlet 3.0+通过`request.getPart("fileFieldName")`获取Part对象，Part.write()写入磁盘；Spring MVC中使用MultipartFile参数接收
- **文件下载**：设置响应头`Content-Disposition: attachment; filename="xxx"`和`Content-Type: application/octet-stream`，通过response.getOutputStream()写出文件字节流
- **跨域(CORS)**：浏览器的同源策略（协议+域名+端口都相同）限制不同源的AJAX请求；服务器通过CORS响应头（Access-Control-Allow-Origin等）告知浏览器允许跨域
- **CORS关键响应头**：Access-Control-Allow-Origin（允许的来源，设为*或具体域名）、Access-Control-Allow-Methods（允许的HTTP方法）、Access-Control-Allow-Headers（允许的请求头）、Access-Control-Allow-Credentials（是否允许携带Cookie，此时Origin不能为*）
- **预检请求(Preflight OPTIONS)**：浏览器在发起非简单跨域请求前，先发一个OPTIONS方法请求询问服务器是否允许跨域，服务器返回CORS头确认后浏览器才发实际请求；非简单请求指Content-Type非application/x-www-form-urlencoded/multipart/form-data/text/plain，或使用了自定义请求头
- **CSRF跨站请求伪造**：攻击者诱导用户访问恶意网站，利用用户已登录状态伪造请求；防御方式：SameSite Cookie、CSRF Token（服务端生成随机Token，表单隐藏字段携带，提交时校验）、Referer头校验
- **简单请求vs非简单请求**：同时满足请求方法为GET/HEAD/POST、Content-Type为三种之一、无自定义请求头，则为简单请求不触发预检；其他均为非简单请求
- **Servlet 3.0异步Servlet**：`request.startAsync()`获取AsyncContext，业务处理在另一个线程执行，Servlet线程立即释放回线程池，提高并发处理能力；适用于耗时操作（如长轮询、SSE）
- **AsyncContext核心方法**：start(Runnable)在另一个线程执行任务、complete()标记异步完成、dispatch()将请求转发到另一个资源
- **Spring DeferredResult**：Spring MVC对异步请求的封装，Controller返回DeferredResult，业务线程通过setResult()填充结果，与Servlet 3.0异步机制底层一致
- **Servlet 3.0@WebServlet注解**：`@WebServlet(urlPatterns = "/hello")`替代web.xml中的`<servlet>`和`<servlet-mapping>`配置；同理@WebFilter/@WebListener替代filter/listener的xml配置
- **Servlet 4.0 HTTP/2支持**：支持Server Push（服务器主动推送资源给客户端，PushBuilder API），请求复用（multiplexing，多请求共享一个TCP连接），头部压缩(HPACK)
- **Servlet 5.0 Jakarta命名空间变更**：从javax.servlet.*变为jakarta.servlet.*，javax改成jakarta，影响所有Servlet API的import语句；这是Java EE捐赠给Eclipse基金会的商标变更
- **Tomcat与Servlet容器关系**：Tomcat是Servlet容器的一个实现（另有Jetty、Undertow等），它实现了Servlet规范，负责Servlet的生命周期管理、请求路由、多线程处理等；Spring Boot默认嵌入Tomcat
- **Servlet线程安全问题原因和解决方案**：servlet是单实例，多个线程同时调用service方法，若servlet有成员变量可能导致数据错乱。解决方案：避免在servlet中定义可变的成员变量；使用局部变量；使用ThreadLocal（确保每个线程有独立副本）；加synchronized（降低并发性）

# 2. 面试题

## 2.1 Servlet的生命周期是什么？
Servlet生命周期由容器管理，分为三个阶段：
1. **init(ServletConfig config)**：Servlet被创建时调用一次。完成初始化工作（如加载配置、建立数据库连接）。可通过重写init()无参方法进行自定义初始化
2. **service(ServletRequest req, ServletResponse res)**：每次请求被接收时调用，决定调用doGet/doPost/doPut/doDelete等方法。该方法在多线程环境下被并发调用，因此Servlet需要线程安全
3. **destroy()**：Servlet被销毁前调用一次。用于释放资源（关闭数据库连接、停止后台线程等）

底层实现：默认Servlet在第一次被请求时创建和初始化（<load-on-startup>可指定容器启动时创建），Servlet实例在整个容器中只有一个，由多个请求线程共享。

## 2.2 转发(forward)和重定向(redirect)的区别是什么？
| 对比维度 | 转发(forward) | 重定向(redirect) |
|---------|--------------|-----------------|
| 发生位置 | 服务端 | 客户端(浏览器) |
| 请求次数 | 1次 | 2次(第一次返回302，第二次请求新URL) |
| URL变化 | 不变 | 变为新地址 |
| request对象 | 同一个，数据可共享 | 两次不同请求，数据丢失 |
| 访问范围 | 仅内部资源(同Web应用) | 可以跳转到任意URL(外部网站) |
| 状态码 | 无(服务端内部) | 302(临时) / 301(永久) |
| 执行方法 | request.getRequestDispatcher().forward() | response.sendRedirect() |

使用场景：forward适用于MVC中Controller将结果传递给JSP渲染（内部流转）；redirect适用于登录成功后跳转首页（PRG模式，防止刷新重复提交表单）、外部跳转。

## 2.3 Cookie和Session的区别是什么？工作流程是怎样的？
| 区别 | Cookie | Session |
|------|--------|---------|
| 存储位置 | 浏览器客户端 | 服务端 |
| 容量限制 | 单个≤4KB，单域名20条左右 | 无明确限制(受服务器内存) |
| 安全性 | 较低（可见可篡改） | 较高（存在服务端） |
| 性能影响 | 不占服务器资源 | 占用服务器内存 |
| 禁用影响 | 可被禁用 | Cookie被禁用时Session也无法工作 |

工作流程：
1. 用户首次请求服务器，服务器调用`request.getSession()`创建Session对象并生成唯一JSESSIONID
2. 服务器通过`response.addCookie()`将JSESSIONID作为Cookie发送给浏览器
3. 浏览器后续请求自动携带该Cookie
4. 服务器根据JSESSIONID查找对应的HttpSession对象，读取用户的会话数据

若Cookie被禁用，可通过URL重写(`response.encodeURL()`)将会话ID拼接到URL中。

## 2.4 Cookie的安全属性有哪些？分别防御什么攻击？
- **HttpOnly**：设置为true后JavaScript无法通过`document.cookie`读取Cookie，防御XSS攻击窃取Cookie（攻击者注入脚本无法读取认证Cookie）
- **Secure**：设置为true后Cookie仅在HTTPS加密连接中传输，防御中间人攻击(MITM)通过HTTP明文窃听Cookie
- **SameSite**：控制跨站点请求是否携带Cookie：
  - Strict：严格禁止任何跨站请求携带Cookie（如从第三方网站点击链接访问也不会携带，可能需要重新登录）
  - Lax：默认值，允许顶级导航的GET请求携带Cookie（如从搜索引擎点击链接），禁止跨站POST等携带
  - None：始终携带（必须同时设置Secure）
  
  SameSite主要防御CSRF攻击，避免攻击者诱导用户在第三方网站发起伪造请求时携带认证Cookie

## 2.5 Filter过滤器的工作原理是什么？多个Filter的执行顺序是怎样的？
Filter是基于AOP思想在请求处理前后插入逻辑的组件。实现javax.servlet.Filter接口，核心是doFilter()方法：

```java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
    // 前置处理（如：权限检查、编码设置、日志记录）
    chain.doFilter(request, response); // 传递给下一个Filter或Servlet
    // 后置处理（如：响应压缩、耗时统计）
}
```

执行顺序：按web.xml中filter-mapping声明顺序（或@WebFilter注解按类名字母序），形成链式调用：

**请求进入** → Filter1前置 → Filter2前置 → Servlet处理 → Filter2后置 → Filter1后置 → **响应返回**

通过`<dispatcher>`可指定过滤的请求类型（REQUEST/FORWARD/INCLUDE/ERROR/ASYNC）。

## 2.6 什么是跨域问题？为什么要同源策略？如何解决？
**同源策略**：浏览器的核心安全机制，要求AJAX请求的协议、域名、端口号三者必须与当前页面完全相同，否则请求会被浏览器阻止。

**为什么要有同源策略**：防止恶意网站的脚本读取另一个网站的敏感数据。如用户在A银行已登录，若浏览器允许跨域，恶意网站B可通过AJAX请求A银行的接口并获取用户账户信息。

**解决方案（CORS跨来源资源共享）**：
- 服务端设置响应头`Access-Control-Allow-Origin`：允许跨域的来源（*表示任意，但带Cookie时不能为*，需指定具体域名）
- `Access-Control-Allow-Methods`：允许的HTTP方法
- `Access-Control-Allow-Headers`：允许的请求头
- `Access-Control-Allow-Credentials`：是否允许携带Cookie（true时Origin不能为*）
- 预检请求（Preflight）：浏览器对非简单请求先发OPTIONS请求询问，服务器返回上述CORS头确认允许后浏览器才发实际请求

其他方案：JSONP（仅GET，已过时）、反向代理（Nginx配置同域转发）。

## 2.7 简单请求和非简单请求的区别是什么？
**简单请求**必须同时满足：
- 请求方法：GET、HEAD或POST
- Content-Type仅限于：application/x-www-form-urlencoded、multipart/form-data、text/plain
- 不使用自定义请求头（如X-Requested-With、Authorization等）

**简单请求**：浏览器直接发送实际请求，服务器返回响应，若响应头CORS不匹配则浏览器拦截结果但不发预检。

**非简单请求**（不满足条件之一即为非简单请求，如Content-Type为application/json、使用PUT/DELETE方法、带Authorization头）：浏览器先发OPTIONS预检请求询问服务器允许哪些方法和头，服务器返回允许后才发实际请求。预检可以被浏览器缓存（Access-Control-Max-Age指定缓存时间），减少重复预检。

## 2.8 CSRF攻击的原理是什么？如何防御？
**原理**：攻击者诱导用户点击恶意链接或访问恶意网站，该网站自动向目标网站发起请求（如转账、修改密码）。由于用户已在目标网站登录（Cookie有效），请求携带着认证Cookie被服务器认为是合法请求。

**举例**：用户在bank.com登录后，访问攻击者网站evil.com。evil.com的页面中隐藏了`<img src="http://bank.com/transfer?to=attacker&amount=10000">`，浏览器加载图片时自动携带用户在bank.com的Cookie，完成了转账。

**防御手段**：
1. **CSRF Token**：服务器生成随机Token，放在表单隐藏字段中，提交时校验Token是否匹配。攻击者无法获取该Token（同源策略保护）
2. **SameSite Cookie**：设置SameSite=Strict或Lax，跨站请求不携带Cookie
3. **Referer/Origin头校验**：检查请求来源是否在允许的白名单中
4. **验证码**：关键操作（转账、改密）需输入验证码
5. **自定义请求头**：AJAX请求添加自定义头（如X-Requested-With），简单跨域请求无法携带自定义头，触发预检

Spring Security内置了CSRF保护（默认开启），通过CsrfFilter和Thymeleaf的_csrf隐藏域实现Token校验。

## 2.9 Servlet 3.0异步Servlet的原理和使用场景？
**原理**：传统Servlet的service方法处理完请求后线程才归还线程池。如果请求中有耗时操作（如调用远程服务、等待消息），线程会长时间阻塞。异步Servlet通过`request.startAsync()`将请求放入异步模式，Servlet线程立即归还线程池处理其他请求，业务逻辑在另一个线程中执行，完成后通过AsyncContext的complete()或dispatch()返回响应。

```java
AsyncContext asyncContext = request.startAsync();
asyncContext.start(() -> {
    // 异步线程处理耗时业务
    String result = slowService.query();
    asyncContext.getResponse().getWriter().write(result);
    asyncContext.complete(); // 标记完成
});
```

**使用场景**：
- 长轮询（Long Polling）：客户端等待服务端消息，如消息推送
- 服务端推送（SSE，Server-Sent Events）
- 需要整合外部慢速服务，不阻塞容器线程

Spring MVC通过DeferredResult和Callable对异步Servlet进行了封装，使用更简洁。

## 2.10 JSP的9大内置对象分别是什么？
| 对象 | 类型 | 说明 |
|------|------|------|
| request | HttpServletRequest | 封装请求信息 |
| response | HttpServletResponse | 封装响应信息 |
| out | JspWriter | 输出到页面，类似PrintWriter |
| session | HttpSession | 当前用户会话 |
| application | ServletContext | 整个Web应用共享 |
| config | ServletConfig | Servlet配置信息 |
| pageContext | PageContext | 页面的上下文，可通过它获取其他8个对象 |
| page | Object(this) | 当前Servlet对象的引用 |
| exception | Throwable | 异常对象，仅在isErrorPage="true"的页面可用 |

## 2.11 文件上传的multipart/form-data与application/x-www-form-urlencoded有什么区别？
- **application/x-www-form-urlencoded**（默认）：表单数据的键值对以`key1=value1&key2=value2`格式编码在请求体中。**只支持文本数据**，不对二进制文件进行编码（会导致文件损坏）。效率相对较低
- **multipart/form-data**：请求体被分割为多个部分(part)，每个部分包含各自的Content-Disposition和Content-Type，可以独立处理文本字段和二进制文件。**必须用于文件上传**。相比前者数据量更大（因为边界分隔符等元数据）

Servlet 3.0中处理文件上传：`@MultipartConfig`注解+`request.getPart("file")`。Spring MVC中直接用`@RequestParam("file") MultipartFile file`接收。

## 2.12 Servlet规范各版本的重要变化有哪些？
- **Servlet 3.0**（Java EE 6）：注解支持(@WebServlet/@WebFilter/@WebListener)、异步处理(AsyncContext)、动态注册Servlet、文件上传(Part接口)、可插性(web-fragment.xml)
- **Servlet 3.1**（Java EE 7）：非阻塞I/O(ReadListener/WriteListener)、HTTP协议升级机制(Upgrade，支持WebSocket)、异步安全增强
- **Servlet 4.0**（Java EE 8）：HTTP/2支持（Server Push-PushBuilder API）、HttpServletMapping支持运行时获取URL映射信息、HttpFilter抽象类（简化Filter实现）
- **Servlet 5.0**（Jakarta EE 9）：包名从javax.servlet改为jakarta.servlet（商标变更）、语义上无大变化
- **Servlet 6.0**（Jakarta EE 10）：去除已废弃的SecurityManager相关API、全面拥抱Java 17+

实际面试最常问3.0和5.0的变化。注意：javax→jakarta的迁移对所有依赖Servlet API的框架（Spring MVC、Spring Security等）有影响，体现在Maven坐标从spring-webmvc到spring-webmvc（需要jakarta版本）。

## 2.13 JSP的动态包含和静态包含有什么区别？
- **静态包含** `<%@ include file="header.jsp" %>`：编译时合并，JSP引擎将两个文件合并为一个Servlet源文件再编译。被包含页修改后，主页面需要重新编译才能生效（因为已合并编译）。性能更好，但内容固定不变
- **动态包含** `<jsp:include page="header.jsp" />`：运行时处理，主页面和被包含页分别编译为独立Servlet。请求时主Servlet调用被包含Servlet并将输出合并。可以传递参数`<jsp:param>`，被包含页修改后无需重新编译主页面。性能略低，但更灵活

## 2.14 什么是Servlet容器？Tomcat和Servlet容器是什么关系？
Servlet容器（Servlet Container，也称Web容器）是实现了Servlet规范的服务器，负责管理Servlet的生命周期、请求分发、多线程处理、安全等。核心功能包括：通信支持（接收HTTP请求）、Servlet生命周期管理、多线程支持、JSP支持（将JSP编译为Servlet）。

**Tomcat**是Servlet容器的一个具体实现（Apache组织开源），其他实现还有Jetty（Eclipse基金会，轻量级）、Undertow（RedHat，高性能）、GlassFish/Payara（完整Java EE/Jakarta EE服务器）。Tomcat严格来说是**Servlet容器+Web服务器**（内置HTTP服务器能力），不是完整的Java EE应用服务器（不支持EJB等）。

Spring Boot默认内嵌Tomcat，也可以替换为Jetty或Undertow，这些嵌入式容器的核心jar分别是spring-boot-starter-tomcat、spring-boot-starter-jetty、spring-boot-starter-undertow。

## 2.15 Servlet是线程安全的吗？为什么？如何保证线程安全？
Servlet**不是线程安全的**。原因：默认情况下容器对一个Servlet只创建一个实例（单例）。当多个请求同时到达时，容器从线程池分配多个线程并发调用同一个Servlet实例的service()方法。如果Servlet中存在可变实例变量（成员变量），多线程并发修改会导致数据错乱。

**保证线程安全的方案**：
1. **不要在Servlet中定义可变成员变量**（最佳实践）：所有状态放在方法局部变量中
2. **使用ThreadLocal**：为每个线程维护独立的变量副本，适合需要在多个方法间共享线程级数据
3. **同步(synchronized)**：对关键代码块加锁，但会严重降低并发性能，不推荐
4. **实现SingleThreadModel**（已废弃）：为每个请求创建新Servlet实例，性能极差

Spring MVC的Controller默认也是单例的，同样需要注意线程安全问题。Spring管理的Service/Controller一般设计为无状态的，线程安全。
