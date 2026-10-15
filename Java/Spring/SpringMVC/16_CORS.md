# 1. 什么是跨域（CORS）
```
跨域，全称Cross-Origin Resource Sharing（跨域资源共享），是浏览器的一种安全机制。

1.1 同源策略（Same-Origin Policy）：
浏览器规定，当前页面只能访问同源服务器上的资源。
"同源"指：协议、域名、端口三者完全相同。

例如：
当前页面URL：http://localhost:8080/springmvc/index.html

可以访问：     http://localhost:8080/springmvc/api/user       （同源）
不能访问：     http://localhost:3000/api/user                 （端口不同）
不能访问：     http://www.baidu.com/api/user                  （域名不同）
不能访问：     https://localhost:8080/springmvc/api/user      （协议不同）

1.2 为什么会有同源策略：
主要是为了防止恶意网站窃取用户数据。例如：
用户登录了银行网站（bank.com），同时访问了一个恶意网站（hack.com）。
如果没有同源策略，hack.com就可以通过AJAX请求访问bank.com的接口，
获取用户的银行账户信息。

1.3 前后端分离架构的跨域问题：
在现代开发中，前后端分离已成为主流架构：
● 前端（Vue/React）运行在 localhost:3000（Node.js开发服务器）
● 后端（SpringMVC）运行在 localhost:8080（Tomcat服务器）

前端通过AJAX请求访问后端API时，由于端口不同，浏览器会阻止跨域请求。
因此需要后端配置CORS策略，允许来自前端地址的请求。
```

# 2. CORS的工作原理
```
2.1 简单请求和预检请求：
浏览器将跨域请求分为两类：

简单请求（同时满足以下条件）：
● 请求方式为 GET、POST 或 HEAD
● Content-Type 仅限于：text/plain、multipart/form-data、application/x-www-form-urlencoded
● 没有自定义请求头

对于简单请求，浏览器直接发送请求，并在请求头中自动添加 Origin 字段。
服务器通过响应头 Access-Control-Allow-Origin 告知浏览器是否允许该请求。

预检请求（Preflight Request）：
对于非简单请求（如 Content-Type: application/json 或 PUT、DELETE请求），
浏览器会先发送一个 OPTIONS 请求（称为"预检请求"），询问服务器是否允许该跨域请求。

预检请求的请求头包含：
● Origin：请求来源
● Access-Control-Request-Method：实际请求的HTTP方法
● Access-Control-Request-Headers：实际请求的自定义头

服务器通过以下响应头回复：
● Access-Control-Allow-Origin：允许的来源
● Access-Control-Allow-Methods：允许的HTTP方法
● Access-Control-Allow-Headers：允许的请求头
● Access-Control-Max-Age：预检请求的缓存时间（秒）

如果预检请求通过，浏览器才会发送实际的跨域请求。

2.2 常见的CORS响应头：
| 响应头 | 说明 |
|--------|------|
| Access-Control-Allow-Origin | 允许的源，* 表示允许所有源 |
| Access-Control-Allow-Methods | 允许的HTTP方法 |
| Access-Control-Allow-Headers | 允许的请求头 |
| Access-Control-Allow-Credentials | 是否允许携带Cookie |
| Access-Control-Expose-Headers | 允许前端读取的响应头 |
| Access-Control-Max-Age | 预检请求缓存时间 |
```

# 3. @CrossOrigin注解（局部跨域配置）
```
SpringMVC提供了@CrossOrigin注解，可以在Controller类或方法上配置跨域策略。

3.1 注解在方法上（只针对该接口）：
@RestController
@RequestMapping("/api/user")
public class UserController {

    @CrossOrigin(origins = "http://localhost:3000")  // 只允许这个源访问该接口
    @GetMapping("/{id}")
    public User getById(@PathVariable("id") Long id) {
        return userService.getById(id);
    }

    @CrossOrigin(origins = {"http://localhost:3000", "http://localhost:8081"})
    @PostMapping
    public String save(@RequestBody User user) {
        userService.save(user);
        return "ok";
    }
}

3.2 注解在类上（对该Controller所有方法生效）：
@RestController
@RequestMapping("/api/user")
@CrossOrigin(origins = "http://localhost:3000")
public class UserController {
    // 该类中所有接口都允许 localhost:3000 跨域访问
    // ...
}

3.3 @CrossOrigin的常用属性：
@CrossOrigin(
    origins = {"http://localhost:3000"},   // 允许的来源
    methods = {RequestMethod.GET, RequestMethod.POST},  // 允许的HTTP方法
    allowedHeaders = {"*"},               // 允许的请求头
    exposedHeaders = {"X-Custom-Header"}, // 允许前端读取的响应头
    allowCredentials = "true",            // 是否允许携带Cookie
    maxAge = 3600L                        // 预检请求缓存时间（秒）
)

注意：
● origins默认值为*（允许所有来源），生产环境必须指定具体域名
● allowCredentials = "true" 时，origins不能为*，必须指定具体域名
● @CrossOrigin只在SpringMVC管理的请求中生效（被DispatcherServlet拦截的请求）
```

# 4. 全局CORS配置（推荐）
```
当项目中有很多Controller需要配置跨域时，逐个添加@CrossOrigin会很繁琐。
推荐使用全局CORS配置，一次性为所有接口设置跨域策略。

4.1 XML配置方式（springmvc.xml）：
<mvc:cors>
    <mvc:mapping path="/api/**"
                 allowed-origins="http://localhost:3000"
                 allowed-methods="GET,POST,PUT,DELETE,PATCH,OPTIONS"
                 allowed-headers="*"
                 allow-credentials="true"
                 max-age="3600"/>
</mvc:cors>

注意：需要引入mvc命名空间，确保xml中已有 xmlns:mvc 声明。

4.2 注解配置方式（实现WebMvcConfigurer接口）：
package com.powernode.springmvc.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")                     // 匹配哪些路径
                .allowedOrigins("http://localhost:3000")    // 允许的来源
                .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
                .allowedHeaders("*")                        // 允许所有请求头
                .allowCredentials(true)                     // 允许携带Cookie
                .maxAge(3600);                              // 预检请求缓存1小时

        // 可以配置多条规则
        registry.addMapping("/public/**")
                .allowedOrigins("*")                        // 公开接口允许所有源
                .allowedMethods("GET")
                .maxAge(1800);
    }
}

注意：
● 全局CORS配置和@CrossOrigin可以共存，@CrossOrigin的优先级更高
● addCorsMappings中路径匹配的是Ant风格（/api/**匹配/api/下的所有路径）
● allowedOrigins不能和allowCredentials(true)同时使用*，否则浏览器会拒绝
```

# 5. 跨域问题常见场景和解决方案
```
5.1 场景一：前端Axios请求携带Cookie
默认情况下，Axios的跨域请求不会携带Cookie。
如果需要携带Cookie（如Session认证），需要前后端同时配置：

前端配置（Axios）：
axios.defaults.withCredentials = true;  // 全局开启
// 或单次请求：
axios.get('/api/user/1', { withCredentials: true });

后端配置（SpringMVC）：
@CrossOrigin(origins = "http://localhost:3000", allowCredentials = "true")

注意：开后端开启allowCredentials后，origins不能为*。

5.2 场景二：自定义请求头
如果前端请求携带了自定义请求头（如Token认证），
需要在CORS配置中显式允许：

前端：
axios.get('/api/user/1', {
    headers: { 'Authorization': 'Bearer xxxx' }
});

后端：
@CrossOrigin(allowedHeaders = {"Authorization", "Content-Type"})
// 或全局配置：
registry.addMapping("/api/**")
        .allowedHeaders("Authorization", "Content-Type", "X-Requested-With");

5.3 场景三：预检请求返回403/401
如果项目使用了Spring Security或自定义拦截器，
默认情况下会拦截OPTIONS请求（预检请求），导致跨域失败。

解决方案：在拦截器中放行OPTIONS请求

public class AuthInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // 放行OPTIONS预检请求
        if ("OPTIONS".equalsIgnoreCase(request.getMethod())) {
            return true;
        }
        // 其他请求的拦截逻辑...
        return true;
    }
}

或者在拦截器配置中排除OPTIONS：
registry.addInterceptor(authInterceptor)
        .addPathPatterns("/**")
        .excludePathPatterns("/login", "/register");  // 预检是自动的，不影响
```

# 6. 生产环境的CORS建议
```
1. 不要使用 allowedOrigins("*")：
   生产环境必须明确指定允许的域名，如："https://www.yourdomain.com"

2. 不要使用 allowedHeaders("*")：
   只开放实际需要的请求头，减少安全风险

3. 合理设置maxAge：
   预检请求有网络开销，适当缓存（如3600秒=1小时）可以提升性能

4. 区分开发环境和生产环境配置：
   开发环境：允许localhost:3000
   生产环境：仅允许上线的前端域名

   示例（使用Spring的@Profile）：
   @Profile("dev")
   @Configuration
   public class DevCorsConfig implements WebMvcConfigurer {
       @Override
       public void addCorsMappings(CorsRegistry registry) {
           registry.addMapping("/api/**")
                   .allowedOrigins("http://localhost:3000")
                   .allowedMethods("*")
                   .allowCredentials(true);
       }
   }

   @Profile("prod")
   @Configuration
   public class ProdCorsConfig implements WebMvcConfigurer {
       @Override
       public void addCorsMappings(CorsRegistry registry) {
           registry.addMapping("/api/**")
                   .allowedOrigins("https://www.yourdomain.com")
                   .allowedMethods("GET", "POST")
                   .allowCredentials(false);
       }
   }

5. 考虑使用反向代理（Nginx）解决跨域：
   如果前后端部署在同一域名下（通过Nginx反向代理），
   可以直接避免跨域问题，这是生产环境的最佳实践。

   Nginx配置示例：
   location /api/ {
       proxy_pass http://localhost:8080/springmvc/api/;
   }
   location / {
       proxy_pass http://localhost:3000/;
   }
   这样前端和API都在同一个域名(端口)下，不存在跨域问题。
```
