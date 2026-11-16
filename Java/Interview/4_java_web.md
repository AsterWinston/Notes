本篇覆盖 Java Web（Servlet/JSP）的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。四个主题块：Servlet 基础、会话与 Web 安全、过滤器监听器与异步、JSP 上传下载与规范演进。容器内部机制见 5_tomcat.md，Spring MVC 层的应用见 8_spring_mvc.md，HTTP 协议细节见 28_http.md。

# 1. Servlet 基础

## 1.1 生命周期与并发模型

### 1.1.1 Servlet 是什么？接口继承体系是怎样的？

**定义**：Servlet = **运行在 Web 服务器（容器）中的 Java 组件**，负责接收 HTTP 请求、处理业务、生成响应——它是 Java Web 的"底层协议组件"：容器按 **Servlet 规范**（Jakarta Servlet）管理它的生命周期与多线程调度，JSP 编译后也是 Servlet（4.1.1），Spring MVC 的 DispatcherServlet 同样是一个 Servlet（4.4.2）。

**继承体系**：

```text
Servlet（接口：生命周期三方法 init/service/destroy）
  └─ GenericServlet（抽象类：协议无关，提供 config/log 便捷方法）
       └─ HttpServlet（抽象类：HTTP 协议专用，service 按方法分发 doGet/doPost/…）
```

**HttpServlet 的分发机制**：`service(req,res)` 内部按请求方法（GET/POST/PUT/DELETE…）路由到对应 `doXxx`；doXxx 默认实现直接返回 **405**——所以只重写 doGet/doPost 的旧式 Servlet 对其他方法天然拒绝。HTTP 与方法的映射语义（幂等/安全）见 28_http.md。

### 1.1.2 Servlet 的生命周期是怎样的？

容器管理的三阶段（全程容器掌控，应用代码管不了）：

1. **加载与实例化**：首次请求时（默认懒加载）或容器启动时（`<load-on-startup>`/`@WebServlet(loadOnStartup=…)`，值小者优先）——**一个 Servlet 类只有一个实例**；
2. **init(ServletConfig)**：仅一次。读初始化参数、建连接、装资源；`getServletConfig()` 拿的是"本 Servlet 的局部配置"，`getServletContext()` 是全局（1.2.2/3.2.1 区分）；
3. **service(ServletRequest, ServletResponse)**：**每个请求一个线程并发调用同一实例**（1.1.3）；
4. **destroy()**：应用卸载/容器关闭前一次，释放资源；随后实例等待 GC。

**高频追问**：

- "第一次请求慢"——实例化+init+JSP 编译都发生在首请求：生产用 loadOnStartup 预热；
- "init 里抛异常会怎样"——容器标记该 Servlet 不可用，后续请求 500；Boot 嵌入式场景则启动失败；
- "Servlet 实例会被复用吗"——是：**跨请求复用同一实例**，这正是线程安全问题的来源。

### 1.1.3 Servlet 是线程安全的吗？如何保证线程安全？

**结论：规范层面不是线程安全的。** 单实例 + 多线程并发调 service（容器从线程池取线程），任何**可变成员变量**都是共享状态，并发读写即数据错乱。

**方案按优先级**：

1. **无状态化（最佳实践）**：状态全放方法局部变量、请求域属性——Servlet/Controller 天生无状态，这是 Web 层的第一设计戒律；
2. **ThreadLocal**：需要"跨方法共享线程级数据"（用户上下文、连接、格式化器）时每线程一份副本（机制与泄漏注意见 1_java.md 3.4）；注意线程池复用线程必须 finally remove；
3. **synchronized 局部加锁**：能正确但吞吐受损，只保护真正的临界资源，别锁整个 service；
4. SingleThreadModel（已废弃）——容器用实例池串行化，性能灾难，历史上被规范亲手废除。

**同构问题**：Spring MVC 的 Controller/Service 默认单例，同样遵守以上结论（见 8_spring_mvc.md 1.1.2、1_java.md 3.1）。

### 1.1.4 Servlet 有哪几种注册方式？load-on-startup 有什么用？

三种等价途径（最终都注册进容器的 MapperContext）：

1. **web.xml**（Servlet 2.5 时代）：`<servlet>/<servlet-mapping>` 两段配对，靠 `<url-pattern>` 匹配（精确 > 路径 `/dir/*` > 扩展名 `*.do` > 默认 `/`）；
2. **注解**（Servlet 3.0+）：`@WebServlet(urlPatterns="/hello", loadOnStartup=1, initParams=…)`——需禁用 metadata-complete；
3. **编程式**：`ServletContainerInitializer`/`ServletContextInitializer` 在启动回调里 `servletContext.addServlet(...)`——**Spring Boot 内嵌容器注册 DispatcherServlet 的通道**（SpringServletContainerInitializer 扫 SpringBootServletInitializer，见 5_tomcat.md、10_springboot.md 2.2.2）。

**load-on-startup**：正值=容器启动即创建初始化（预热、暴露配置错误）；缺省/负值=首次请求才建。**实践**：生产把核心 Servlet（尤其 DispatcherServlet）配 0~1 预热，避免"上线第一个用户承担全部冷启动"；Boot 对应 `spring.servlet.dispatcher-servlet.load-on-startup=0` 与异步预热 `spring.mvc.async.…`/应用级 Ready 钩子（见 10_springboot.md 2.1.3）。

## 1.2 请求与响应对象

### 1.2.1 HttpServletRequest 封装了什么？参数编码怎么处理？

**请求侧视图**：请求行（方法/URI/Query）→ `getMethod()/getRequestURI()/getQueryString()/getParameterMap()`；请求头 → `getHeader()/getCookies()`；请求体 → 参数（表单）或原始流（JSON/上传，**流只能读一次**，重复读要靠包装缓存——Spring 的 ContentCachingRequestWrapper、Filter 包请求的根因）；属性域 → `setAttribute/getAttribute`（一次请求内传值）。

**参数解码两套链路（经典编码题）**：

- **URL Query 参数**：Tomcat 按 `server.xml Connector` 的 `URIEncoding`（**Tomcat 8+ 默认已 UTF-8**；7.x 默认 ISO-8859-1 产生"GET 中文乱码"的历史名场面）解码；Nginx/容器间转发时 `proxy_pass` 是否改写编码也要对齐；
- **表单 Body（urlencoded）**：`request.setCharacterEncoding("UTF-8")` 必须在**首次 getParameter 之前**调用（通常在编码 Filter 里统一设，见 3.1.1）；
- **路径参数/REST 中文**：同走 URI 解码；`%` 百分号编码规则见 28_http.md；
- **JSON body**：不经 getParameter，字节流由 HttpMessageConverter 按 `Content-Type charset` 解析（见 8_spring_mvc.md 2.1.1），三层编码各管一段是排查"还是乱码"的地图。

### 1.2.2 HttpServletResponse 如何设置响应？两个输出流有什么关系？

- **状态行**：`setStatus(200)`（只设码）与 `sendError(404,"…")`（**容器接管**：跳错误页机制 + 丢弃已写正文）语义不同——Boot 的错误页体系由 BasicErrorController 承接（见 10_springboot.md 4.2.1）；`sendRedirect` = 302 + Location（1.2.3）；
- **响应头**：`setHeader`（覆盖）vs `addHeader`（追加）——Cookie 必须 add（多个 Set-Cookie）；
- **正文**：`getWriter()`（字符流，受 setCharacterEncoding/ContentType 约束）与 `getOutputStream()`（字节流）——**二选一，混用抛 IllegalStateException**；先 `setContentType` 再写，缓冲区提交（`isCommitted()`）后改头全部失效；
- 响应头/正文一旦 flush 到客户端就覆水难收——"异常时想返回 JSON 却输出了一半 HTML"的根因。

### 1.2.3 forward 和 redirect 有什么区别？

| 维度 | forward（转发） | redirect（重定向） |
|------|----------------|--------------------|
| 发起者 | **服务端内部**行为 | **浏览器**行为 |
| API | `request.getRequestDispatcher("/b").forward(req,res)` | `response.sendRedirect("/b")`（302+Location） |
| 请求次数 | 1 次（req/res 对象原样复用） | 2 次（新请求，**request 属性全丢**） |
| 地址栏 | 不变 | 变为新 URL |
| 范围 | 同一 Web 应用内 | 任意 URL（含外站） |
| 数据传递 | `setAttribute` 照拿 | 只能 URL 参数 / Flash 会话（8_spring_mvc.md 5.3.1） |

**追问点**：① 302 vs 301——302 临时（每次仍访问原 URL）、301 永久（浏览器/搜索引擎缓存新址，SEO 权重转移）；sendRedirect 是 302；307/308 是"保留方法与 body"的现代临时/永久码（见 28_http.md）；② forward 时已写入 response 缓冲区的内容默认被丢弃（已 flush 则报错）；③ PRG 模式（Post-Redirect-Get）防重复提交用 redirect，MVC 层的 `redirect:`/`forward:` 前缀写法见 8_spring_mvc.md 5.3.1。

# 2. 会话与 Web 安全

## 2.1 状态管理

### 2.1.1 Cookie 是如何工作的？关键属性有哪些？

**机制**：HTTP 无状态——Cookie 是**服务端通过响应头 `Set-Cookie` 投放、浏览器按域/路径自动存储并在后续请求头 `Cookie` 自动回带**的键值对协议（细节归浏览器，服务端管不了偷改）。

**属性全表**：

| 属性 | 语义 | 注意 |
|------|------|------|
| name/value | 载荷 | value 需 URL 安全；单条约 4KB 上限、每域百来条 |
| maxAge | 存活秒数 | **0=立即删除，负数=会话级（关浏览器没了）** |
| domain | 生效域 | 写 `.example.com` 子域共享（跨主域无解） |
| path | 生效路径前缀 | 常设 `/` 全域携带 |
| HttpOnly | JS 不可读 | 防 XSS 偷 Cookie（2.2.1） |
| Secure | 仅 HTTPS 传输 | 防中间人窃听 |
| SameSite | 跨站携带策略 | Strict/Lax/None（None 强制配 Secure），防 CSRF（2.2.2） |
| Version/Comment | 遗留字段 | 忽略 |

**工程口径**：现代会话/认证只用 Cookie 存**不透明标识**（JSESSIONID/token），敏感数据一律放服务端（2.1.2）；"记住我"用长 maxAge 的 token 而非密码。

### 2.1.2 Session 的工作原理？生命周期如何？

**链路**：首次 `request.getSession()`（不存在则创建）→ 容器生成 **JSESSIONID**（32 位随机串，Tomcat 标准实现）经 Set-Cookie 下发 → 后续请求带回 → 容器按 id 找到 `HttpSession` 对象 → `setAttribute/getAttribute` 读写会话数据。Session 对象存服务端内存（Tomcat `Manager` 组件的会话仓库）。

**销毁三途径**：① `session.invalidate()` 主动；② **超时**——默认 30 分钟（web.xml `<session-timeout>`；Boot `server.servlet.session.timeout`），基于**最后访问时间**的不可达判定；③ 容器正常关闭**序列化到 `SESSIONS.ser`**、异常关闭全丢（持久化 Manager/JDBC Store 可救，见 5_tomcat.md）。

**追问**：① "每次 getSession 是同一个吗"——同一会话内是（id 匹配）；`getSession(false)` 只查不建，鉴权 Filter 常用；② Session 固定攻击（Session Fixation）——登录成功后必须 `changeSessionId()` 换新 id（Spring Security 默认动作，见 11_spring_security.md）；③ 集群/微服务下**内存 Session 不可用**（会话粘滞是权宜），标准答案：token 自包含或 Redis 集中式会话（见 27_redis.md）。

### 2.1.3 Cookie 和 Session 的区别是什么？

| 维度 | Cookie | Session |
|------|--------|---------|
| 存储位置 | **客户端**浏览器 | **服务端**内存（或外置存储） |
| 容量 | 4KB 级、条数受限 | 仅受内存约束 |
| 安全 | 明文可见可篡改 | 数据不出服务端，只传 id |
| 性能成本 | 无服务端开销 | 占服务端资源、集群要共享 |
| 依赖关系 | 可独立用（埋点/偏好） | **会话标识通常靠 Cookie 传递**（id 也可走 URL，2.1.4） |

**关系正解**：二者不是同类竞争——Session 是"服务端数据仓库"，Cookie 是"传 id 的快递员"之一；把"禁用 Cookie 则 Session 失效"说成铁律是错的（URL 重写兜底），说清"默认传输通道是 Cookie"才准确。

### 2.1.4 浏览器禁用 Cookie 后 Session 怎么办？现代应用怎么替代？

**经典兜底：URL 重写**——`response.encodeURL("/list")`：容器若发现客户端未回传 JSESSIONID，自动在 URL 追加 `;jsessionid=xxx`（path 参数形式），后续请求靠它定位会话。代价：URL 泄 id（日志/Referer 泄露面）、分享链接会串会话、重定向必须用 `encodeRedirectURL` 且**首次响应就探测**——工程上属于"证明你知道，生产不再用"。

**现代三条路**（追问必须延伸到这里）：

1. **Token（JWT/自管）**放 Header `Authorization`——完全脱离 Cookie；前后端分离跨域方案（8_spring_mvc.md 4.2.3）；
2. **集中式会话**：会话数据挪进 Redis，key=token——横向扩容/重启不丢（见 27_redis.md）；
3. **同域反向代理**：前端 Nginx 把 `/api` 代理到后端，同源策略与 Cookie 通道一并解决（见 12_nginx.md）。

## 2.2 攻击与防御

### 2.2.1 Cookie 的三个安全属性分别防什么攻击？

| 属性 | 防御 | 机制一句话 |
|------|------|-----------|
| **HttpOnly** | XSS 窃取凭证 | JS 的 `document.cookie` 读不到该 Cookie——脚本注入后拿不到会话 id（挡不住"带 Cookie 发请求"，只挡"读走"） |
| **Secure** | 中间人窃听 | 只在 HTTPS 通道发送——HTTP 明文页面/跳转不带，避免网络嗅探 |
| **SameSite** | CSRF | 控制**跨站请求**是否自动携带 Cookie：`Strict`=任何跨站都不带（外链跳入也可能要重登）、`Lax`=顶级导航 GET 带、表单类跨站 POST 不带（**Chrome 80+ 默认 Lax**）、`None`=都带但必须配 Secure（iframe 嵌入、跨站回调场景才用，第三方 Cookie 治理下正被浏览器逐步淘汰） |

**组合拳常识**：三属性是纵深防御的三块砖，缺 CSRF Token 校验仍可能被绕过（同站脚本、浏览器实现差异）；生产基线 = HttpOnly + Secure + SameSite=Lax **再叠加** 2.2.2 的 Token 校验。

### 2.2.2 CSRF 攻击的原理是什么？如何防御？

**原理**（Cross-Site Request Forgery，跨站请求伪造）：攻击者**借用户已登录的身份、利用浏览器自动携带 Cookie** 伪造请求。经典剧本：用户在 bank.com 保持登录 → 访问 evil.com，页面藏 `<img src="https://bank.com/transfer?to=attacker&amount=10000">` 或自动提交表单 → 浏览器请求 bank 时自动带凭证 Cookie → 服务端视其为合法操作。**信任的是 Cookie，漏洞是"请求可被第三方页面诱发"**。

**防御清单（按有效性排序）**：

1. **CSRF Token（主力）**：服务端在每个表单/页面种随机 Token（与会话绑定），提交时校验——攻击页面跨源拿不到 Token（同源策略反过来保护了你）；Spring Security 的 CsrfFilter + `_csrf` 隐藏域（Thymeleaf 自动注入），前后端分离配 Cookie `XSRF-TOKEN` + Header `X-XSRF-TOKEN`（见 11_spring_security.md）；
2. **SameSite=Lax/Strict**：跨站 POST 默认不带 Cookie，釜底抽薪（2.2.1）；
3. **校验 Origin/Referer 头**：白名单来源才放行——成本低的第二道闸；
4. 二次确认（验证码/密码）：转账改密等最高危操作；
5. 自定义头触发预检：AJAX 带 `X-Requested-With`/`Authorization`，跨站伪造页面靠简单请求带不上这些头——一带头就触发预检（简单/非简单规则见 8_spring_mvc.md 4.2.2、28_http.md）。

**易错**：① CSRF 不需要"看到响应"——盲打即可（img/form 提交），所以 GET 带副作用的接口（老式 `/delete?id=1`）照样中招；**GET 安全（无副作用）是 CSRF 免疫的前提**（REST 语义，8_spring_mvc.md 4.4.1）；② XSS 与 CSRF 区分：XSS 是"注入脚本冒充你执行"，CSRF 是"借你的身份发请求"。

### 2.2.3 什么是 XSS？如何防御？（补充高频题）

**原理**（Cross-Site Scripting，跨站脚本）：攻击者把**脚本塞进会被其他用户浏览器执行的输出**——本质是"数据被当成代码执行"。三种形态：

| 形态 | 剧本 | 例 |
|------|------|-----|
| 存储型 | 恶意脚本入库（评论/私信），每个浏览者中招 | 评论区 `<script>偷 Cookie 再发 evil.com</script>` |
| 反射型 | URL 参数原样回显到页面 | 搜索结果页 `?q=<script>…` |
| DOM 型 | 前端 JS 操作 innerHTML 等 sink 执行 | `location.hash` 内容直接 innerHTML |

**防御（按层）**：

1. **输出编码是第一原则**——按上下文转义：HTML 实体（`<`→`&lt;`）、属性值、JS 字符串、URL 参数各有对应编码器（OWASP Encoder）；JSP 用 `<c:out>`/EL 默认转义（`${}` 不转义的场景要警惕历史版本差异），Thymeleaf 默认转义（`th:utext` 显式豁免）；
2. **输入校验/过滤**：黑名单挡不住花样（`<img onerror>`、编码绕），只做"格式白名单"辅助；富文本用净化库（Jsoup Safelist/ DOMPurify）；
3. **HttpOnly Cookie**（2.2.1）+ CSP（响应头限制脚本来源 `'self'`/nonce）——把"偷到了也执行不了/执行不了什么"兜住；
4. XSS Filter 写进 Filter 链统一做参数预处理（3.1.1 的典型用例，注意别当唯一防线）。

**与 CSRF 一句话区分**：XSS=服务端执行/回显了你的输入（身份被盗用）；CSRF=你的浏览器执行了别人的请求（身份被盗刷）。

# 3. 过滤器、监听器与异步

## 3.1 Filter 过滤器

### 3.1.1 Filter 的工作原理是什么？生命周期如何？

**定位**：Filter = 容器层面、**请求到达目标 Servlet 前后**的横切拦截组件（AOP 思想在 Web 层的鼻祖实现）——编码处理、鉴权、日志、审计、响应包装的标准挂载点。

```java
public class LogFilter implements Filter {
    @Override public void init(FilterConfig cfg) { /* 容器启动/首次装载，一次 */ }
    @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        long t0 = System.currentTimeMillis();
        // 前置：可包装 req/res（XSS 清洗、缓存 body）、可直接拦截（鉴权失败 sendError 后 return）
        chain.doFilter(req, res);                  // 放行：交给下一个 Filter 或目标 Servlet
        // 后置：响应已写出后再执行（统计耗时）
        log.info("{}ms", System.currentTimeMillis() - t0);
    }
    @Override public void destroy() { /* 卸载前一次 */ }
}
```

**生命周期**：init（一次）→ doFilter（每请求）→ destroy（一次）——与 Servlet 同款"单实例多线程"，**doFilter 自身也要线程安全**（1.1.3 结论全适用）。

**链式机制**：`chain.doFilter` 是"责任链推进"的开关——**不调用 = 请求被截流**（鉴权拒绝的标准姿势）；Filter 拿到的 Chain 是容器构造的包装对象，内部持游标，逐个前进到尽头才进 Servlet。

### 3.1.2 多个 Filter 的执行顺序是怎样的？Dispatcher 类型是什么？

**顺序规则**：① web.xml 按 `<filter-mapping>` **声明顺序**排链（`<filter>` 定义顺序不算，mapping 才算）；② `@WebFilter` 注解顺序**不保证**（多数容器按类名字典序——依赖它是事故源），要精确顺序用 web.xml 或 Spring 的 `FilterRegistrationBean` + `@Order`（Boot 推荐，见 8_spring_mvc.md 3.1.1 周边）；③ 执行形态是**洋葱模型/栈**：

```text
f1 前 → f2 前 → Servlet → f2 后 → f1 后
```

——每个 Filter 的 `chain.doFilter` 之后代码按**注册逆序**出栈；后置逻辑异常不会"漏进"前一层的正常流，try/finally 包裹才稳。

**Dispatcher 类型**（`<dispatcher>`/`@WebFilter(dispatchTypes=…)`，决定"哪些调度类型触发过滤"）：

| 类型 | 触发场景 |
|------|---------|
| REQUEST（默认） | 客户端直接发起的请求 |
| FORWARD | `forward:` 转发的目标资源 |
| INCLUDE | `<jsp:include>`/RequestDispatcher.include |
| ERROR | 错误页/错误 Servlet（sendError 之后） |
| ASYNC | 异步派发（startAsync 后的 dispatch 回来，3.3.1） |

**高频坑**："Filter 里设的编码在 forward 后失效/没执行"——因为只匹配了 REQUEST；Spring Security 这类要管转发后资源权限的框架会显式配 FORWARD（其 FilterChainProxy 注册方式见 11_spring_security.md）。

### 3.1.3 Filter 和 Interceptor（Spring MVC 拦截器）有什么区别？

**一句话**：Filter 归 **Servlet 容器规范**，Interceptor 归 **Spring MVC 框架**——层次、粒度、能力三方面都不同：

| 维度 | Filter | HandlerInterceptor |
|------|--------|--------------------|
| 归属 | 容器（jakarta.servlet.Filter） | Spring MVC 组件（DispatcherServlet 流程内） |
| 拦截范围 | **进容器的所有请求**（含静态资源、错误页、非 MVC 资源） | 仅**命中 HandlerMapping 的 MVC 请求** |
| 时点 | Servlet 执行前后 | Handler **执行前后 + afterCompletion 渲染后**三回调 |
| 上下文 | 只有 req/res | 知道 handler（Controller 方法）、mv（ModelAndView） |
| 典型用途 | 编码、CORS、鉴权网关、压缩、审计 | 登录态校验带业务、接口统计、视图数据注入 |

顺序全景：`Filter → DispatcherServlet → 拦截器 pre → Controller → 拦截器 post → 视图渲染 → 拦截器 afterCompletion → Filter 后置`。**面试连招**：三者的第三种——AOP（切方法不切请求）——取舍见 8_spring_mvc.md 3.1.4；"鉴权放哪一层"的答复：**跨域/凭证类放 Filter（要挡一切）、业务权限放拦截器（要拿 handler 元数据）**。

## 3.2 Listener 监听器

### 3.2.1 Listener 能监听什么？ContextLoaderListener 是怎么启动 Spring 容器的？

**规范事件体系（8 接口）**：围绕三大作用域对象的"生命周期 + 属性变更"：

| 事件源 | 生命周期 | 属性变更 |
|--------|---------|---------|
| ServletContext | ServletContextListener（contextInitialized/contextDestroyed） | ServletContextAttributeListener |
| HttpSession | **HttpSessionListener**（在线人数统计）、HttpSessionActivationListener（钝化/活化）、HttpSessionBindingListener | HttpSessionAttributeListener |
| ServletRequest | ServletRequestListener | ServletRequestAttributeListener |

注册：`@WebListener` 或 web.xml `<listener>`；启动顺序上 Listener 先于一切 Filter/Servlet 的 init。

**ContextLoaderListener（经典追问）**：传统 SSM war 项目里它就是 ServletContextListener 实现——`contextInitialized` 中创建**根 XmlWebApplicationContext**（加载 contextConfigLocation 的 Spring 配置），把容器"桥接"给 Web 环境；DispatcherServlet 再建自己的子上下文挂在其上（父子容器，见 7_spring.md 1.1.1 语境、8_spring_mvc.md 1.1.2）。Boot 时代被 `SpringBootServletInitializer`/main 方法取代，但"用监听器钩应用启停"的思想活在所有 `ApplicationListener`/`ServletContextInitializer` 里（见 10_springboot.md 2.1.2）。

## 3.3 异步处理

### 3.3.1 Servlet 3.0 异步处理的原理是什么？适用哪些场景？

**动机**：同步模型下容器线程"陪着慢调用干等"——Tomcat 200 线程被一个 2s 的 RPC 占满，吞吐雪崩（线程池模型局限见 5_tomcat.md）。

**机制**：`AsyncContext ctx = req.startAsync()` → 容器**立即归还当前 Servlet 线程**，但**不提交响应**（请求挂起）→ 业务在别的线程（`ctx.start(runnable)` 交给容器线程或自有池）跑完 → `ctx.getResponse().getWriter().write(...)` + `ctx.complete()`（或 `ctx.dispatch()` 回 MVC 渲染）。超时：`ctx.setTimeout(30_000)` + `onTimeout` 兜底。

```java
AsyncContext ctx = req.startAsync();
ctx.start(() -> {
    String r = slowService.query();
    ctx.getResponse().getWriter().write(r);
    ctx.complete();
});
```

**场景**：长轮询（消息推送）、SSE、聚合慢下游。**Spring 封装**：Controller 返回 `Callable`/`DeferredResult` 即底层走这套 AsyncContext（封装差异与线程池坑见 8_spring_mvc.md 5.1.1）；再往上 Reactive/WebFlux 是"全链路非阻塞"的彻底版。注意：异步链路**ThreadLocal 上下文会丢**（Filter 的鉴权信息不会自动跟过去），`dispatcher` 要含 ASYNC 类型，SecurityContextHolder/租户上下文要显式传递（1_java.md 3.4）。

# 4. JSP、上传下载与规范演进

## 4.1 JSP

### 4.1.1 JSP 的工作原理？它和 Servlet 什么关系？

**本质**：JSP 是**"写成了页面形态的 Servlet"**——HTML 为主、嵌 Java 逻辑，降低模板可读性。

**运行链路**：首次请求 `a.jsp` → Jasper 引擎**翻译成** `a_jsp.java`（一个继承 HttpJspBase→HttpServlet 的类：模板静态部分进 out.write、`<%= %>` 表达式进 out.print、声明变成员变量、脚本片段进 `_jspService`）→ 编译成 class → 之后按标准 Servlet 生命周期服务（init 一次、service 多线程）。

**工程含义**：① JSP 首请求慢=翻译+编译成本，生产 precompile 或已被 Thymeleaf/Freemarker/前后端分离淘汰；② JSP 的"脚本可维护性灾难"是前后端分离兴起的直接动因之一；③ 修改 JSP 需重新翻译（开发期容器自动做，生产常关开发检查）。

### 4.1.2 JSP 的 9 大内置对象和 4 种作用域？

`_jspService` 方法开头就声明好的 9 个对象（**其实就是形参与局部变量**，并非魔法）：

| 对象 | 真实类型 | 作用域 |
|------|---------|--------|
| request | HttpServletRequest | request |
| response | HttpServletResponse | page（对象本身跨请求） |
| out | JspWriter（带缓冲的 Writer） | page |
| session | HttpSession | session |
| application | ServletContext | application |
| config | ServletConfig | page |
| pageContext | PageContext | **page，但能取到其他 8 个** |
| page | this（当前 Servlet） | page |
| exception | Throwable | 仅 `isErrorPage=true` 的页面 |

**四种作用域（必背，从大到小）**：`page`（一次请求内当前页）→ `request`（一次请求，forward 共享）→ `session`（一次会话，跨请求）→ `application`（全局，全应用共享）。`setAttribute/getAttribute` 逐级可见——数据"该放哪层"就是这道题；pageContext 是跨层取值的统一入口。

### 4.1.3 静态包含和动态包含有什么区别？

| 维度 | 静态 `<%@ include file="h.jsp" %>` | 动态 `<jsp:include page="h.jsp"/>` |
|------|-----------------------------------|-------------------------------------|
| 时机 | **翻译期**文本合并 | **运行期**调用（RequestDispatcher.include） |
| 产物 | 一个 Servlet | 各编译各的，输出拼接 |
| 传参 | 不能 | 可 `<jsp:param>` 且被包含页可区分来源 |
| 被包含页变更 | 主页面需重编译 | 独立生效 |
| 典型用途 | 头部/页脚固定模板 | 可配置组件、带参局部刷新 |

**辨析追问**：静态 include 合并后两个页面的指令/声明会互相影响（重复 import 报错）；`<jsp:forward>` 同属运行期机制（与 1.2.3 forward 一条线）。

### 4.1.4 EL 和 JSTL 是什么？解决什么问题？

**动机**：把 Java 代码赶出页面。JSP 脚本 `<% %>`/`<%= %>` 让模板变代码坟场——EL+JSTL 是"模板语言化"的答案。

- **EL** `${user.name}`：按 pageContext→request→session→application **顺序搜索作用域**取值（支持 `.`/`[]`、运算、`empty` 判空、隐式对象 param/header/cookie）；本质是属性访问表达式，取值走 JavaBean getter；
- **JSTL**：`<c:if>/<c:forEach>/<c:out>/<c:set>` + fmt 格式化 + fn 函数库——循环/判空/格式化都不再写脚本；`<c:out value="${x}" escapeXml="true">` 默认转义即 XSS 防御一环（2.2.3）。

**现状**：JSP/JSTL 属"维护存量、新课不教"——新前端要么 Thymeleaf（服务端模板现代化：命名空间即属性、无脚本、与 Spring 绑定生态），要么前后端分离 REST+SPA（8_spring_mvc.md 4.4.1）。面试会问"为什么不用 JSP 了"：脚本混乱、翻译编译耦合、无法组件化协作、首请求慢四条即可。

## 4.2 文件上传与下载

### 4.2.1 multipart/form-data 与 urlencoded 有什么区别？Servlet 如何处理上传？

| 维度 | application/x-www-form-urlencoded | multipart/form-data |
|------|-----------------------------------|---------------------|
| 形态 | `k1=v1&k2=v2` URL 编码单串 | 按 `--boundary` 切段，每段独立头（Content-Disposition，可再带 Content-Type） |
| 二进制 | **不友好**（全量百分号编码，体积膨胀） | **原生支持**——每段原样字节 |
| 用途 | 普通表单 | **文件上传必选**（enctype 指定） |

**Servlet 3.0 原生接收**：`@MultipartConfig`（声明大小阈值、临时目录、文件命名策略）+ `request.getPart("file")` → `Part` API：`write(name)` 落盘、`getInputStream()` 流式读、`getSubmittedFileName()`。**Boot 内嵌默认已启用**（spring.servlet.multipart），Spring MVC 侧的 `MultipartFile/@RequestPart` 绑定与大小配置、Nginx 413 链路见 8_spring_mvc.md 4.3.1。安全三件套：**扩展名白名单 + Magic Number 校验 + 随机化存储名**（防上传 webshell 与同名覆盖），文件尽量进对象存储（见 25_minio.md）。

### 4.2.2 文件下载如何实现？

```java
resp.setContentType("application/octet-stream");   // 未知类型一律 octet-stream
resp.setHeader("Content-Disposition",
        "attachment; filename=\"report.pdf\"; filename*=UTF-8''%E6%8A%A5%E5%91%8A.pdf");
try (InputStream in = fileService.stream(id); OutputStream out = resp.getOutputStream()) {
    in.transferTo(out);                             // JDK9+，替代手写拷贝循环
}
```

**要点**：① `Content-Disposition: attachment` 触发另存、`inline` 直接预览（浏览器类型支持时）；② **中文文件名**：`filename` 老兼容 + `filename*`（RFC 5987/`encodeURIComponent` 式编码）现代标准；③ `Content-Length` 给得出就给（进度条），chunked 传输也行；④ 图片/视频等静态直出交给容器默认 Servlet/资源 handler，支持 Range 断点续传（MVC 的 `ResponseEntity<Resource>` 自动处理，见 8_spring_mvc.md 4.3.2）；⑤ 权限校验在下载入口做——id 可枚举接口要鉴权 + 限流。

## 4.3 规范演进

### 4.3.1 Servlet 规范各版本有哪些重要变化？

| 版本 | 归属 | 关键点 |
|------|------|--------|
| 2.5 | Java EE 5 | web.xml 时代的成熟形态；注解缺席 |
| **3.0** | Java EE 6 | **去 XML 化三件套**：`@WebServlet/@WebFilter/@WebListener` + 编程式 `ServletContainerInitializer`（Boot SPI 的根）+ **异步处理 AsyncContext** + 文件上传 Part/MultipartConfig + web-fragment 可插拔 |
| 3.1 | Java EE 7 | **非阻塞 IO**（ReadListener/WriteListener）、**协议升级 Upgrade（WebSocket 握手地基**，36_websocket.md）、HTTP 客户端 API、带超时 Servlet 连接 |
| 4.0 | Java EE 8 | **HTTP/2**（PushBuilder 服务端推送）、ServletMapping 运行时查询、HttpFilter 抽象类 |
| **5.0** | Jakarta EE 9 | **javax.servlet → jakarta.servlet 命名空间大迁移**（Eclipse 基金会商标案，语义不变、import 全变、坐标全变——Boot 3/Spring 6 的 breaking change 根源，见 10_springboot.md 3.2.3） |
| 6.0 | Jakarta EE 10 | 移除废弃的 SecurityManager/单例模型 API，拥抱 Java 17 基线 |

**答题策略**：报"3.0（注解+异步+上传）、3.1（非阻塞 IO+WebSocket 地基）、5.0（jakarta 迁移）"三件就够覆盖 90% 追问；被问"javax→jakarta 对我们意味着什么"——所有 Servlet 容器/框架必须成代换版，混用即 `ClassNotFoundException`。

## 4.4 容器与框架关系

### 4.4.1 什么是 Servlet 容器？它和 Tomcat 是什么关系？

**Servlet 容器（Web 容器）** = Servlet 规范的运行时实现，职责四件：网络通信与协议解析（HTTP 报文 → Request 对象）、**生命周期管理**（实例化/init/service 线程调度/destroy）、多线程分发（线程池模型）、JSP 编译（Jasper）+ 安全声明（realm/约束）等配套服务。

**Tomcat = 容器 + HTTP 服务器**：它实现了容器职责，但**不是完整 Jakarta EE 应用服务器**（无 EJB/CDI 全家桶，与 GlassFish/Payara 的分界）。其他实现：Jetty（长连接轻量）、Undertow（非阻塞内核）、WebLogic/WildFly（商业/完整应用服务器内嵌它们各家的容器变体）。

**关系定位**：规范与实现——你的 Servlet 代码只依赖 `jakarta.servlet-api`，换容器不改业务代码（可移植性契约）。Boot 内嵌三选一切换见 10_springboot.md 4.1.1；容器线程池/连接器内部机制见 5_tomcat.md。

### 4.4.2 Servlet 和 Spring MVC 是什么关系？为什么一个应用只有一个 DispatcherServlet？

**关系**：**Spring MVC = Servlet 规范之上的框架实现**，不替代容器——整个 MVC 应用在容器眼里只是一个 Servlet：`DispatcherServlet` 以 `/` 注册（Boot 默认映射全部路径），**前端控制器模式**把所有请求收进来，再由它自己按 HandlerMapping 派发给成百上千个 `@RequestMapping` 方法（Bean，非 Servlet）。

**为什么"单 Servlet"**：① 每请求在容器里只有"一个线程 + 一个入口对象"，容器不需要知道你的路由表——路由与参数解析、消息转换、异常处理全部收编进 MVC 策略组件（九大组件见 8_spring_mvc.md 1.1.3）；② 避免"每个 Controller 方法注册一个 Servlet"的配置爆炸与容器级资源开销；③ 拦截链、视图解析、安全上下文等跨资源协作只能在统一入口做。

**追问闭环**：注册通道——Boot 经 ServletRegistrationBean 把 DispatcherServlet 交容器（8_spring_mvc.md 1.1.2）；一次请求穿过的完整链路背熟即同框答对两题（8_spring_mvc.md 1.1.1 + 5_tomcat.md）。