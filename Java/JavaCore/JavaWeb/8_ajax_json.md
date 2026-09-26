# 1.什么是 AJAX？

```
AJAX = Asynchronous JavaScript And XML（异步的 JavaScript 和 XML）

AJAX 不是一门新的编程语言，而是一种使用现有标准的新方法。
它可以在不重新加载整个页面的情况下，与服务器交换数据并更新部分网页。

传统 Web 交互（同步）：
  用户点击 → 浏览器发送请求 → 服务器处理 → 返回整个页面 → 浏览器刷新整个页面
  缺点：用户体验差，有白屏闪烁，传输数据量大

AJAX 交互（异步）：
  用户操作 → JavaScript发送异步请求 → 服务器处理 → 返回数据(非页面)
  → JavaScript接收数据 → 局部更新页面（页面不刷新）
  优点：用户体验好，无刷新，传输数据量小，响应快
```

# 2.XMLHttpRequest 对象

```
XMLHttpRequest 是 AJAX 的核心对象，所有现代浏览器都内置了这个对象。

// 创建 XMLHttpRequest 对象
var xhr = new XMLHttpRequest();

基本使用步骤：
  1. 创建 XMLHttpRequest 对象
  2. 注册回调函数（onreadystatechange）
  3. 打开连接（open方法：指定请求方式和URL）
  4. 发送请求（send方法）

readyState 状态码：
  0 —— 请求未初始化（open 未调用）
  1 —— 服务器连接已建立（open 已调用）
  2 —— 请求已接收（send 已调用，响应头已收到）
  3 —— 请求处理中（响应体正在接收）
  4 —— 请求已完成，响应就绪

status HTTP状态码：
  200 —— 成功
  404 —— 资源未找到
  500 —— 服务器错误
```

# 3.AJAX 发送 GET 请求

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>AJAX GET 示例</title>
</head>
<body>
    <button onclick="sendGetRequest()">发送GET请求</button>
    <div id="result"></div>

    <script type="text/javascript">
        function sendGetRequest() {
            // 1. 创建 XMLHttpRequest 对象
            var xhr = new XMLHttpRequest();
            
            // 2. 注册回调函数
            xhr.onreadystatechange = function() {
                if (xhr.readyState == 4) {
                    if (xhr.status == 200) {
                        // 获取服务器返回的文本
                        document.getElementById("result").innerHTML = xhr.responseText;
                    } else {
                        alert("请求失败：" + xhr.status);
                    }
                }
            };
            
            // 3. 打开连接（GET请求，参数拼在URL后面）
            xhr.open("GET", "/oa/ajax?username=zhangsan&age=20", true);
            
            // 4. 发送请求
            xhr.send();
        }
    </script>
</body>
</html>
```

# 4.AJAX 发送 POST 请求

```html
<script type="text/javascript">
    function sendPostRequest() {
        var xhr = new XMLHttpRequest();
        
        xhr.onreadystatechange = function() {
            if (xhr.readyState == 4 && xhr.status == 200) {
                document.getElementById("result").innerHTML = xhr.responseText;
            }
        };
        
        xhr.open("POST", "/oa/ajax", true);
        
        // POST请求必须设置请求头
        xhr.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
        
        // 发送数据（格式：name=value&name=value）
        xhr.send("username=zhangsan&age=20");
    }
</script>
```

# 5.Servlet 端处理 AJAX 请求

```java
@WebServlet("/ajax")
public class AjaxServlet extends HttpServlet {
    
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) 
            throws ServletException, IOException {
        request.setCharacterEncoding("UTF-8");
        response.setContentType("text/plain;charset=UTF-8");
        
        String username = request.getParameter("username");
        String age = request.getParameter("age");
        
        // 业务处理...
        System.out.println("接收到GET请求：username=" + username + ", age=" + age);
        
        // 返回普通文本
        PrintWriter out = response.getWriter();
        out.print("接收到数据：username=" + username + ", age=" + age);
    }
    
    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) 
            throws ServletException, IOException {
        request.setCharacterEncoding("UTF-8");
        response.setContentType("text/plain;charset=UTF-8");
        
        String username = request.getParameter("username");
        String age = request.getParameter("age");
        
        // 业务处理...
        System.out.println("接收到POST请求：username=" + username + ", age=" + age);
        
        PrintWriter out = response.getWriter();
        out.print("POST请求已处理！");
    }
}
```

# 6.JSON 格式

```
JSON（JavaScript Object Notation）是一种轻量级的数据交换格式。

它是一种纯文本格式，独立于编程语言，易于人阅读和编写，也易于机器解析和生成。

JSON 的两种结构：
  1. 对象：{ "key1": "value1", "key2": "value2" }
  2. 数组：[ "element1", "element2", "element3" ]

JSON 的值类型：
  数字、字符串、布尔值（true/false）、null、对象、数组

示例：
{
    "id": 1001,
    "name": "张三",
    "age": 25,
    "hobbies": ["篮球", "足球"],
    "address": {
        "city": "北京",
        "street": "长安街"
    }
}
```

# 7.Servlet 返回 JSON 数据

```
后端将 Java 对象转换成 JSON 字符串，响应给前端。

常用的 JSON 解析库：
  1. Jackson（Spring Boot 默认）
  2. Fastjson（阿里巴巴）
  3. Gson（Google）

方式一：手动拼接 JSON 字符串（不推荐，仅限简单场景）

String json = "{\"id\":1001,\"name\":\"" + user.getName() + "\",\"age\":" + user.getAge() + "}";
response.getWriter().print(json);

方式二：使用 Jackson（推荐）

// 引入 Jackson 的 jar 包到 WEB-INF/lib：
// jackson-databind-2.x.x.jar
// jackson-core-2.x.x.jar
// jackson-annotations-2.x.x.jar

import com.fasterxml.jackson.databind.ObjectMapper;

@WebServlet("/userList")
public class UserListServlet extends HttpServlet {
    
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) 
            throws ServletException, IOException {
        
        // 查询数据库，获取用户列表
        List<User> userList = userService.findAll();
        
        // 使用 Jackson 将 List<User> 转成 JSON 字符串
        ObjectMapper mapper = new ObjectMapper();
        String jsonStr = mapper.writeValueAsString(userList);
        
        // 设置响应类型为 JSON
        response.setContentType("application/json;charset=UTF-8");
        
        // 输出 JSON
        response.getWriter().print(jsonStr);
    }
}

方式三：使用 Fastjson

import com.alibaba.fastjson.JSON;

String jsonStr = JSON.toJSONString(userList);
response.getWriter().print(jsonStr);

方式四：使用 Gson

import com.google.gson.Gson;

Gson gson = new Gson();
String jsonStr = gson.toJson(userList);
response.getWriter().print(jsonStr);
```

# 8.JSON 格式的 AJAX 交互（完整示例）

```html
<!-- 前端页面：动态加载用户列表，无刷新 -->
<table border="1">
    <thead>
        <tr>
            <th>编号</th>
            <th>姓名</th>
            <th>年龄</th>
        </tr>
    </thead>
    <tbody id="userTableBody">
        <!-- 数据通过AJAX动态填充 -->
    </tbody>
</table>

<script type="text/javascript">
    window.onload = function() {
        var xhr = new XMLHttpRequest();
        xhr.onreadystatechange = function() {
            if (xhr.readyState == 4 && xhr.status == 200) {
                // 将JSON字符串解析为JavaScript对象
                var userList = JSON.parse(xhr.responseText);
                
                // 遍历数据，动态生成表格行
                var html = "";
                for (var i = 0; i < userList.length; i++) {
                    var user = userList[i];
                    html += "<tr>";
                    html += "<td>" + user.id + "</td>";
                    html += "<td>" + user.name + "</td>";
                    html += "<td>" + user.age + "</td>";
                    html += "</tr>";
                }
                document.getElementById("userTableBody").innerHTML = html;
            }
        };
        xhr.open("GET", "/oa/userList", true);
        xhr.send();
    };
</script>
```

# 9.经典案例：用户名异步校验

```html
<!-- 注册页面 -->
用户名：<input type="text" id="username" onblur="checkUsername()" />
<span id="usernameTip"></span>

<script type="text/javascript">
    function checkUsername() {
        var username = document.getElementById("username").value;
        
        if (username.trim() == "") {
            document.getElementById("usernameTip").innerHTML = "用户名不能为空";
            return;
        }
        
        var xhr = new XMLHttpRequest();
        xhr.onreadystatechange = function() {
            if (xhr.readyState == 4 && xhr.status == 200) {
                var result = JSON.parse(xhr.responseText);
                if (result.available) {
                    document.getElementById("usernameTip").innerHTML = "✓ 用户名可用";
                    document.getElementById("usernameTip").style.color = "green";
                } else {
                    document.getElementById("usernameTip").innerHTML = "✗ 用户名已被占用";
                    document.getElementById("usernameTip").style.color = "red";
                }
            }
        };
        xhr.open("GET", "/oa/checkUsername?username=" + encodeURIComponent(username), true);
        xhr.send();
    }
</script>
```

```java
// 后端 Servlet
@WebServlet("/checkUsername")
public class CheckUsernameServlet extends HttpServlet {
    
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) 
            throws ServletException, IOException {
        response.setContentType("application/json;charset=UTF-8");
        
        String username = request.getParameter("username");
        
        // 查询数据库，判断用户名是否存在
        User user = userDao.findByUsername(username);
        
        // 构造JSON响应
        String json;
        if (user == null) {
            json = "{\"available\": true, \"message\": \"用户名可用\"}";
        } else {
            json = "{\"available\": false, \"message\": \"用户名已被占用\"}";
        }
        
        response.getWriter().print(json);
    }
}
```

# 10.Servlet 接收前端发送的 JSON 数据

```
前端发送 JSON 数据，Servlet 端接收并解析。

前端代码：

var user = {
    "username": "zhangsan",
    "password": "123456",
    "age": 25
};

var xhr = new XMLHttpRequest();
xhr.open("POST", "/oa/ajax", true);
xhr.setRequestHeader("Content-Type", "application/json;charset=UTF-8");
xhr.send(JSON.stringify(user));  // 将JS对象转为JSON字符串发送

后端 Servlet 接收：

@WebServlet("/ajax")
public class AjaxServlet extends HttpServlet {
    
    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) 
            throws ServletException, IOException {
        
        // JSON 数据在请求体中，不能通过 request.getParameter() 获取
        // 需要从输入流中读取
        
        // 方式一：手动读取并解析
        BufferedReader reader = request.getReader();
        StringBuilder sb = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) {
            sb.append(line);
        }
        String jsonStr = sb.toString();
        
        // 使用 Jackson 反序列化为 Java 对象
        ObjectMapper mapper = new ObjectMapper();
        User user = mapper.readValue(jsonStr, User.class);
        
        System.out.println("接收到JSON数据：" + user.getUsername() + ", " + user.getPassword());
        
        // 返回响应
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().print("{\"success\": true}");
    }
}
```

# 11.AJAX 跨域请求

```
浏览器的同源策略（Same-Origin Policy）：
  AJAX 请求默认只能访问同源（相同协议+域名+端口）的资源。
  不同源之间的请求称为跨域请求，默认会被浏览器拦截。

  例如：http://localhost:8080 向 http://localhost:9090 发请求，就是跨域。

解决跨域的方式：

方式一：JSONP（只支持 GET 请求，利用 script 标签不受同源策略限制的特性）

方式二：CORS（跨域资源共享，Cross-Origin Resource Sharing）
  在服务器端设置响应头，允许跨域访问：

@Override
protected void doGet(HttpServletRequest request, HttpServletResponse response) {
    // 允许所有域名跨域
    response.setHeader("Access-Control-Allow-Origin", "*");
    // 允许的请求方式
    response.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE");
    // 允许的请求头
    response.setHeader("Access-Control-Allow-Headers", "Content-Type, Authorization");
    
    // 正常处理请求...
}

也可以使用 Filter 为所有 Servlet 统一设置 CORS 响应头。
```

# 12.Axios（现代 AJAX 方案）

```
Axios 是一个基于 Promise 的 HTTP 客户端库，可以替代原生的 XMLHttpRequest。
在真实项目开发中，Axios 已成为主流选择。

使用 Axios 发送 GET 请求：

axios.get('/oa/userList')
    .then(function(response) {
        // 响应数据
        console.log(response.data);
    })
    .catch(function(error) {
        console.error(error);
    });

使用 Axios 发送 POST 请求：

axios.post('/oa/saveUser', {
    username: 'zhangsan',
    age: 25
})
.then(function(response) {
    console.log('保存成功');
})
.catch(function(error) {
    console.error('保存失败', error);
});

相比原生 XMLHttpRequest，Axios 的优点：
  1. 语法简洁，链式调用
  2. 自动转换 JSON 数据
  3. 支持请求和响应拦截器
  4. 支持取消请求
  5. 客户端支持防止 CSRF/XSRF
```
