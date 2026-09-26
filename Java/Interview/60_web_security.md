# 1. 知识点
- **XSS(Cross-Site Scripting)**: 跨站脚本攻击，攻击者将恶意脚本注入网页，在用户浏览器中执行。分为反射型、存储型、DOM型
- **反射型XSS**: 恶意脚本通过URL参数传递，服务端将参数值直接反射到响应页面。如`?q=<script>alert('xss')</script>`直接回显
- **存储型XSS**: 恶意脚本存储在数据库(如评论内容)，当用户访问页面时从DB读出并执行。危害最大，影响所有访问用户
- **DOM型XSS**: 前端JS直接处理用户输入并动态修改DOM，恶意代码不经过服务端。如`innerHTML = userInput`
- **XSS防护**: 输出编码(HTML实体转义<转为&lt; >转为&gt;)，富文本使用白名单过滤器(如JSoup)，设置CSP内容安全策略头
- **XSS防护响应头**: X-XSS-Protection(浏览器端XSS过滤器，1; mode=block启用)，CSP Content-Security-Policy(限制资源加载来源和脚本执行)
- **CSP内容安全策略**: `Content-Security-Policy: default-src 'self'; script-src 'self' cdn.example.com; style-src 'self' 'unsafe-inline'` 限制脚本/样式/图片等资源的来源
- **CSRF(Cross-Site Request Forgery)**: 跨站请求伪造。攻击者利用用户已登录态(Cookie自动携带)，在第三方网站诱导用户发起恶意请求(如转账)
- **CSRF攻击条件**: 1)用户登录了目标网站A 2)在未退出A的情况下访问了恶意网站B 3)B发送请求到A时浏览器自动携带A的Cookie
- **CSRF防护-CSRF Token**: 服务端生成随机Token返回页面，前端每次请求携带Token(请求头/参数)，服务端校验。攻击者无法获取Token
- **CSRF防护-SameSite Cookie**: `Set-Cookie: session=xxx; SameSite=Strict`。Strict(完全禁止跨站携带)、Lax(允许GET导航请求携带，默认)、None(允许跨站，需配合Secure)
- **CSRF防护-Referer/Origin校验**: 检查请求的Referer/Origin头是否来自合法域名。可被伪造(部分环境不发送)
- **CSRF防护-验证码**: 关键操作(支付/修改密码)要求输入验证码，攻击者不知道验证码无法完成攻击
- **SQL注入**: 用户输入拼接到SQL语句中，导致SQL语义改变。如`' OR '1'='1' --`绕过密码验证。极其危险的漏洞
- **SQL注入防护**: 使用PreparedStatement参数化查询(MyBatis的#{}预编译)，不要拼接SQL字符串(${}直接替换)。输入校验(类型/长度/格式)。数据库最小权限原则
- **MyBatis #和$区别**: `#{}`是参数占位符(预编译，安全)，`${}`是字符串直接替换(不预编译，有注入风险)。动态表名/列名必须用$时需白名单校验
- **SSRF(Server-Side Request Forgery)**: 服务端请求伪造。攻击者操控服务器向内网发起请求(如访问内网API、读取元数据、端口扫描)
- **SSRF场景**: 服务端支持用户传入URL访问并返回结果(如网页快照、文件下载、图片代理)。攻击者传入如`http://192.168.1.1/admin`探测内网
- **SSRF防护**: 白名单限制域名/IP、禁用内网IP(127.0.0.1/10.0.0.0/8/172.16.0.0/12/192.168.0.0/16)、禁用非HTTP协议(file/gopher/dict)、禁止302跳转到内网
- **文件上传漏洞**: 用户上传恶意脚本文件(如JSP/PHP)，服务器保存后访问该文件执行恶意代码
- **文件上传防护**: 校验文件类型(后缀白名单+MIME魔术数字Magic Number)、限制上传大小、存储到外部不可直接访问的位置(MinIO/OSS)、重命名文件避免用户控制路径、病毒扫描
- **文件类型校验**: 后缀校验不可靠(可修改)，MIME校验可伪造(来自客户端Content-Type)，魔术数字校验(读文件头字节，如JPEG=FF D8 FF)较可靠
- **DDoS(Distributed Denial of Service)**: 分布式拒绝服务攻击。攻击者控制大量肉鸡向目标发送请求耗尽带宽/资源
- **CC攻击(Challenge Collapsar)**: 应用层DDoS，模拟正常用户请求(如频繁搜索/刷新)，消耗服务器资源(CPU/DB连接)
- **DDoS/CC防护**: 频率限制(Rate Limit)、IP黑名单/灰名单、WAF(Web Application Firewall如ModSecurity)、云防护(阿里云Anti-DDoS/CDN)、验证码挑战(CC)
- **JSON注入**: 恶意JSON数据可能导致JSON解析器逻辑问题、JSON劫持(JSON Hijacking)、原型污染(JavaScript Prototype Pollution)
- **XML外部实体注入(XXE)**: XML解析时引用外部实体，读取服务器文件、SSRF、DoS。`<!ENTITY xxe SYSTEM "file:///etc/passwd">`
- **XXE防护**: 禁用DTD和外部实体。配置`XMLInputFactory.setProperty(XMLInputFactory.SUPPORT_DTD, false)` 或 `setProperty(XMLConstants.ACCESS_EXTERNAL_DTD, "")`
- **OAuth2常见漏洞**: redirect_uri未严格校验(可跳转到钓鱼网站劫持授权码)、state参数缺失/可预测(CSRF)、implicit模式截获token(浏览器历史)、scope权限过大
- **HTTPS安全**: TLS 1.2/1.3加密传输、强制HTTPS(HSTS头)、证书有效(不被吊销)、密钥长度和算法配置(禁用弱算法如RC4/3DES)
- **密码存储**: 严禁明文存储。使用BCrypt/SCrypt/PBKDF2等慢哈希算法(加盐)。BCrypt内置随机盐，每个hash不同盐值，cost factor减慢暴力破解
- **BCrypt**: BCryptPasswordEncoder每次产生不同salt，相同的密码每次hash结果不同。Strength参数越大越慢(默认10，2^10次迭代)
- **Session和Cookie的关系**: Session的实现依赖Cookie。服务器创建Session后，将JSESSIONID通过Set-Cookie响应头发给浏览器；浏览器后续请求自动携带该Cookie；禁用Cookie时可通过URL重写(url-rewriting)传递SessionID
- **Cookie与Session的区别**: 存储位置(Cookie浏览器端 vs Session服务端); 大小限制(Cookie 4KB vs Session无限制); 数据类型(Cookie仅字符串 vs Session任意Object); 安全性(Session更安全); 生命周期(Cookie可长期保存 vs Session默认30分钟)
- **如何实现Session共享**: (1) Session持久化到Redis/DB，所有服务共享查询;(2) 无状态JWT替代Session，服务端不存状态;(3) Spring Session框架简化Redis集中存储
- **CSRF攻击**: 跨站请求伪造，攻击者利用用户已登录态Cookie，在另一个标签页发起恶意请求。JWT可避免(令牌不自动发送)。防护: CSRF Token/SameSite Cookie/Referer检查/验证码
- **为什么要用双Token无感刷新**: Access_Token短有效期(用于API认证) + Refresh_Token长有效期(用于刷新AccessToken)。避免频繁重新登录，用户无感知刷新
- **JWT和Token的区别**: 普通Token: 任意字符串/无固定结构/安全靠实现; JWT: Header.Payload.Signature三段式/自带签名/无状态/适合分布式SSO和移动端
- **常见安全响应头**: CSP(内容来源限制)、HSTS(强制HTTPS, max-age秒, includeSubDomains包含子域)、X-Frame-Options(防止点击劫持, DENY禁止/SAMEORIGIN同源)、X-Content-Type-Options: nosniff(禁止MIME类型嗅探)
- **点击劫持(ClickJacking)**: 攻击者将目标网站透明iframe覆盖在诱饵按钮上，用户实际点击了iframe内的按钮。防护：`X-Frame-Options: DENY`
- **MIME类型嗅探**: 浏览器根据内容猜测MIME类型(忽略Content-Type)。如文本文件可能被嗅探为HTML执行。防护：`X-Content-Type-Options: nosniff`

# 2. 面试题

## 2.1 XSS攻击有哪几种类型？如何防护？
三类XSS：
1) **反射型**：恶意脚本通过URL参数传入，服务端反射到页面。防护：服务端对输出进行HTML实体编码(如Spring的HtmlUtils.htmlEscape)
2) **存储型**：恶意脚本存入数据库，从DB读取后展示执行。防护：入库时过滤(白名单标签)、输出时编码，富文本用JSoup等做白名单过滤
3) **DOM型**：前端JS直接把用户输入插入DOM。防护：避免使用innerHTML/dangerouslySetInnerHTML，使用textContent；必须用时做转义

通用防护措施：
- 输出编码(HTML实体转义)：`<`→`&lt;` `>`→`&gt;` `&`→`&amp;`
- 设置CSP头：`Content-Security-Policy: script-src 'self'`
- 设置X-XSS-Protection：`1; mode=block`
- HttpOnly Cookie(防止JS读取Cookie，降低XSS窃取Cookie风险)
- 输入校验(类型、长度、格式白名单)

## 2.2 CSRF攻击的原理和防护措施有哪些？
原理：用户登录网站A(Cookie记录登录态)，在未退出A时访问恶意网站B。B中嵌入自动提交的表单/脚本`<form action="http://A.com/transfer" method="POST"><input name="to" value="attacker"/></form>`，提交时浏览器自动携带A的Cookie，A认为是用户本人操作。

关键：浏览器默认行为是跨域请求也携带Cookie(未设置SameSite时)。

防护：
1) **CSRF Token**：页面生成Token存Session/Redis，前端请求携带Token(header X-CSRF-TOKEN)。攻击者无法获取Token
2) **SameSite Cookie**：`Set-Cookie: session=xxx; SameSite=Strict`，跨站请求不携带Cookie。最彻底的方案
3) **Referer/Origin校验**：服务端检查请求来源是否在白名单域名内(有局限性，部分环境不发送Referer)
4) **验证码**：关键操作(转账/支付)加验证码，增加攻击成本
5) **自定义请求头**：AJAX请求加自定义Header，简单请求(表单)无法添加自定义Header，浏览器在跨域情况下需要预检

## 2.3 SQL注入的常见攻击手法和防御方式？
攻击手法：
- `' OR '1'='1' --` ：绕过登录判断(WHERE条件永远为真)
- `' UNION SELECT ...` ：联合查询获取其他表数据
- `'; DROP TABLE users; --` ：执行多条语句(部分数据库支持)
- 时间盲注：`' AND IF(1=1, SLEEP(5), 0)` 通过响应时间判断

防御：
1) **参数化查询(PreparedStatement)**：`SELECT * FROM users WHERE name = ?`。MyBatis用`#{}`(预编译)，不用`${}`(字符串替换)
2) **输入校验**：对参数类型、长度、格式做服务器端校验
3) **最小权限原则**：应用数据库账户只有增删改查权限，没有DDL(建表删表)权限
4) **ORM框架**：使用JPA/MyBatis-Plus等，它们内部使用预编译
5) **WAF**：Web防火墙自动检测和拦截SQL注入请求

MyBatis中`${}`不安全示例：`SELECT * FROM users ORDER BY ${column}`。column参数传入`id; DROP TABLE users`即被注入。应使用白名单映射column值。

## 2.4 SSRF服务端请求伪造如何防护？
攻击场景：服务端提供URL代理(如`/fetch?url=http://xxx`)，攻击者传入内网地址探测内部服务。

防护措施：
1) **白名单**：只允许访问预先配置的域名列表
2) **禁用内网IP**：解析URL的IP，拒绝RFC 1918私有地址(10.x、172.16.x、192.168.x)、回环地址(127.x)、链路本地(169.254.x)
3) **禁用危险协议**：只允许HTTP/HTTPS，拒绝file:// gopher:// dict:// ftp://等
4) **禁止302跳转**：请求库设置不跟踪重定向，或重定向后重新做安全校验
5) **内网服务认证**：内网API加上Token认证，即使被SSRF也无法直接访问
6) **响应限制**：限制返回内容大小，防止大文件导致OOM

Java实现内网IP检测：用InetAddress解析域名得到IP，检查isAnyLocalAddress/isLoopbackAddress/isLinkLocalAddress/isSiteLocalAddress。

## 2.5 文件上传功能有哪些安全风险？如何防范？
风险：上传WebShell(如JSP/PHP/ASPX恶意脚本) → 访问上传文件URL → 服务器执行恶意代码获得控制权。

防范：
1) **文件类型校验(多层)**：后缀白名单(只允许jpg/png/pdf等非脚本类型)、MIME校验(检查Content-Type)、魔术数字校验(检查文件头字节，如PNG头=89504E47)
2) **文件内容扫描**：对文本/图片文件用杀毒引擎扫描
3) **存储位置隔离**：保存在MinIO/OSS外部存储而非Web容器目录下，或Nginx配置上传目录禁止脚本执行
4) **文件名重命名**：使用UUID/NanoID生成随机文件名，避免用户输入控制路径(../上级目录遍历)
5) **文件大小限制**：防止大文件耗尽磁盘
6) **访问控制**：上传目录设置只读权限，不执行脚本；下载通过接口+权限校验而非直接URL

## 2.6 JWT签名HS256和RS256有什么区别？安全性如何？
| 维度 | HS256(HMAC-SHA256) | RS256(RSA-SHA256) |
|------|---------------------|--------------------|
| 密钥类型 | 对称密钥(同一个secret) | 非对称(公钥+私钥) |
| 签发和验证 | 同一方持有secret | 私钥签发，公钥验证 |
| 密钥分发 | secret需安全共享(难) | 公钥可公开分发 |
| 性能 | 快 | 慢 |
| 适用场景 | 单体应用、内部服务 | 微服务/SSO(多个验证方) |

安全性考量：
- HS256的secret必须足够复杂(256位随机值)，且不能被泄露。多服务共享secret有泄露风险。
- RS256更安全，私钥只有签发方持有，即使某服务公钥被泄露也无法伪造JWT。
- 推荐：内部微服务间用HS256(简单)，对外开放/SSO场景用RS256(安全)。

## 2.7 如何安全地存储用户密码？
1) **禁止明文存储**：永远不能存储明文密码
2) **使用慢哈希算法**：BCrypt(Java中BCryptPasswordEncoder)、SCrypt、Argon2。这些算法计算慢(故意消耗计算资源)，增加暴力破解成本
3) **加盐(Salt)**：每个密码生成随机盐值，相同密码不同盐hash结果不同。BCrypt内置随机盐
4) **BCrypt Strength(Work Factor)**：strength=10表示2^10=1024次迭代。每次迭代加倍，破解难度也加倍。推荐10-12(平衡安全与性能)
5) **密码长度限制**：不限制最大长度(需要防范DoS，限制如128字符)
6) **不要自定义加密方案**：使用经过验证的标准算法，不要自己设计加密

```java
BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(10);
String hash = encoder.encode("password123"); // 自动生成salt，每次不同
boolean match = encoder.matches("password123", hash); // 验证
```

## 2.8 常见的安全响应头及其作用？
| Header | 作用 | 示例值 |
|--------|------|--------|
| Content-Security-Policy | 限制资源加载来源，防XSS | default-src 'self'; script-src 'self' cdn.example.com |
| Strict-Transport-Security (HSTS) | 强制HTTPS访问 | max-age=31536000; includeSubDomains |
| X-Frame-Options | 防点击劫持 | DENY(禁止iframe) / SAMEORIGIN(同源允许) |
| X-Content-Type-Options | 禁止MIME类型嗅探 | nosniff |
| X-XSS-Protection | 浏览器XSS过滤器 | 1; mode=block |
| Referrer-Policy | 控制Referer信息传递 | strict-origin-when-cross-origin |
| Permissions-Policy | 控制浏览器特性使用 | camera=(), microphone=() |

Spring Security配置示例：
```java
http.headers(headers -> headers
    .contentSecurityPolicy("default-src 'self'")
    .frameOptions().deny()
    .xssProtection().and()
    .hsts().includeSubdomains(true).maxAgeInSeconds(31536000)
    .contentTypeOptions()
);
```

## 2.9 Session和Cookie的关系是什么？
Session和Cookie是配合使用的。**Session的运行依赖Cookie**：

1) 用户首次访问，服务器创建Session对象(生成唯一JSESSIONID)
2) 服务器通过HTTP响应的`Set-Cookie`头将`JSESSIONID=xxxxx`发送给浏览器
3) 浏览器存储该Cookie，后续每次请求自动在`Cookie`请求头中携带JSESSIONID
4) 服务器收到JSESSIONID后从Session存储(内存/Redis)中查找对应Session对象，识别用户身份

**如果禁用Cookie怎么办**：
通过**URL重写(URL Rewriting)**将JSESSIONID附加在URL后面，如`http://example.com/index;jsessionid=ABC123`。Servlet中通过`response.encodeURL()`自动添加。但URL重写不美观、容易泄露(被复制分享)、不利于SEO，一般不建议。

## 2.10 Cookie与Session有什么区别？
| 维度 | Cookie | Session |
|------|--------|---------|
| 存储位置 | 浏览器端(客户端) | 服务器端 |
| 大小限制 | 单个Cookie不超过4KB，每个域名最多20-50个 | 无限制(取决于服务器内存/存储) |
| 数据类型 | 仅字符串 | 任意Java Object |
| 安全性 | 较低(存储在客户端可被查看/篡改) | 较高(数据在服务端，客户端仅有SessionID) |
| 生命周期 | 可通过Expires/Max-Age设置长期保存(如"记住我"7天) | 默认30分钟(inactive超时)，浏览器关闭后JSESSIONID Cookie被清除(会话Cookie) |
| 网络开销 | 每次请求都携带Cookie，增加请求大小 | 仅传递SessionID，开销极小 |
| 跨域 | 不支持跨域名(可通过Domain属性控制) | 不支持(需处理跨域Session共享) |

## 2.11 如何实现Session共享？
分布式架构中多台服务器需要共享Session，否则用户请求被Nginx轮询到不同服务器时登录态丢失。

**方案1——Session持久化到Redis/DB(推荐)**：
所有服务将Session统一存储在Redis(内存)或MySQL中。请求到达任意服务器时，服务器使用JSESSIONID到Redis查找Session对象。
- 优点：集中管理、性能高(Redis内存操作)、可持久化
- 缺点：Redis故障影响全局，需Redis集群保证高可用

**方案2——无状态JWT替代Session**：
不使用服务端Session，改用JWT令牌。用户登录后服务端签发JWT返回给客户端，客户端每次请求携带JWT(Header Authorization)。服务端不存储任何会话状态，只验证签名。
- 优点：天然支持分布式/跨域、服务端无状态、不依赖存储
- 缺点：无法主动踢出用户(需额外黑名单)、令牌一经签发在有效期内一直有效

**方案3——Spring Session框架**：
Spring生态的首选方案，只需简单配置即可将Session存储到Redis/JDBC/MongoDB等后端，对业务代码透明。

```java
// application.yml
spring:
  session:
    store-type: redis  // 存储类型
    timeout: 30m       // Session超时
```

- 注解`@EnableRedisHttpSession`即可启用
- 对开发者完全透明，仍使用`HttpSession session = request.getSession()`读写Session
- Spring Session自动处理序列化/反序列化

## 2.12 什么是CSRF攻击？如何防护？
**CSRF(Cross-Site Request Forgery)**跨站请求伪造是一种利用用户已登录身份在第三方站点发起恶意请求的攻击方式。

**攻击原理**：
1) 用户登录了银行网站A(获取登录态Cookie)
2) 在未退出A的情况下去浏览一个黑客网站B
3) B网站中嵌入隐藏的表单/图片：`<img src="http://bank.com/transfer?to=hacker&amount=10000" />`
4) 浏览器加载图片时自动携带A网站的Cookie发起`/transfer`请求
5) 银行服务器认为这是已登录用户本人的合法操作

**核心**：浏览器默认行为——发起跨域请求时也会自动携带目标域名的Cookie。

**为什么JWT可以避免CSRF**：
普通Session Cookie在跨域请求时自动携带。而JWT通常存储在localStorage或由前端手动添加到Authorization请求头中，**跨域请求不会自动携带JWT**，攻击者无法伪装。

**防护措施**：
1) **CSRF Token**：服务器生成随机Token放在页面(隐藏字段/Header)，请求时提交Token，服务端验证。攻击者的伪造表单无法获取Token
2) **SameSite Cookie**：`Set-Cookie: session=xxx; SameSite=Strict/Lax`。SameSite=Strict跨站请求完全不发送Cookie；SameSite=Lax仅GET导航请求发送。从根源上阻止Cookie携带
3) **Referer/Origin校验**：服务端检查请求头Referer或Origin是否是可信域名。但部分浏览器/网络环境不发送Referer头，有局限性
4) **验证码**：关键操作(转账/支付/修改密码)要求输入验证码或二次认证
5) **自定义请求头**：前端发AJAX请求时加自定义Header头，浏览器会先发OPTIONS预检请求，若服务端不返回对应CORS头则请求被拦截

## 2.13 为什么要用双Token无感刷新？解决了什么问题？
双Token策略即 **Access Token(访问令牌) + Refresh Token(刷新令牌)**，解决的是JWT过期后用户无感知续期的体验问题。

**痛点**：如果只有单一Token：
- Token有效期设短(如15分钟)：用户频繁需要重新登录，体验极差
- Token有效期设长(如7天)：Token一旦泄露，攻击者可长期冒用，安全风险大

**双Token方案**：
| 维度 | Access Token | Refresh Token |
|------|-------------|---------------|
| 用途 | API请求认证 | 刷新Access Token |
| 有效期 | 短(15-30分钟) | 长(7-30天) |
| 存储 | 内存/变量 | HttpOnly Cookie/安全存储 |
| 暴露风险 | 高(每次请求携带) | 低(仅在刷新接口使用) |

**工作流程**：
1) 用户登录 → 服务端返回Access Token(15分钟) + Refresh Token(7天)
2) 前端每次API请求携带Access Token在Authorization头
3) 服务端返回401(过期)时，前端自动用Refresh Token调`/refresh`接口换取新的Access Token
4) 用户完全无感知，无需重新登录
5) Refresh Token也过期时，才跳转到登录页

**解决的问题**：
- **用户体验**：Access Token频繁过期，前端自动刷新，用户无感知
- **安全性**：Access Token短有效期即使泄露影响也有限；Refresh Token更长但使用频率低、暴露面小
- **权限控制**：可将Refresh Token也加入黑名单实现强制踢人

## 2.14 JWT和普通Token有什么区别？
| 维度 | 普通Token | JWT(Json Web Token) |
|------|-----------|---------------------|
| 定义 | 无固定格式，只是一个随机字符串 | 三段式结构(Header.Payload.Signature)的JSON格式令牌 |
| 结构 | 无固定结构，如`abc123xyz` | `eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjF9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c` |
| 签名 | 不内置签名(安全性靠实现) | 内置Signature，使用密钥对Header+Payload签名，防篡改 |
| 状态性 | 通常需要将Token和用户信息在服务端关联(查DB/Redis) | **无状态(Stateless)**，Payload中可携带用户信息，验证签名后直接使用，不需要查数据库 |
| 安全性 | 依赖实现 | 签名防篡改，支持多种算法(HS256/RS256/ES256) |
| 适用场景 | 简单Token认证(如API密钥) | 分布式SSO、微服务间认证、移动端 |
| 实现复杂度 | 简单 | 较复杂(需管理密钥、过期、刷新) |

JWT组成：`Header.Payload.Signature`
- **Header**：指定签名算法(`{"alg":"HS256","typ":"JWT"}`)
- **Payload(Claims)**：存储业务数据(过期时间/用户ID/角色等)，是Base64编码而非加密，**不要存敏感信息**
- **Signature**：`HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)`，防止数据篡改
