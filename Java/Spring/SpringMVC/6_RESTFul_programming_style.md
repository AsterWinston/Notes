# 1. RESTFul编程风格
## 1.1 RESTFul是什么
```
RESTFul是WEB服务接口的一种设计风格。
RESTFul定义了一组约束条件和规范，可以让WEB服务接口更加简洁、易于理解、易于扩展、安全可靠。
RESTFul对一个WEB服务接口都规定了哪些东西？
● 对请求的URL格式有约束和规范
● 对HTTP的请求方式有约束和规范
● 对请求和响应的数据格式有约束和规范
● 对HTTP状态码有约束和规范
REST对请求方式的约束是这样的：
● 查询必须发送GET请求
● 新增必须发送POST请求
● 修改必须发送PUT请求
● 删除必须发送DELETE请求
REST对URL的约束是这样的：
● 传统的URL：get请求，/springmvc/getUserById?id=1
● REST风格的URL：get请求，/springmvc/user/1
● 传统的URL：get请求，/springmvc/deleteUserById?id=1
● REST风格的URL：delete请求, /springmvc/user/1
RESTFul对URL的约束和规范的核心是：通过采用不同的请求方式+ URL来确定WEB服务中的资源。
RESTful 的英文全称是 Representational State Transfer（表述性状态转移）。简称REST。
表述性（Representational）是：URI + 请求方式。
状态（State）是：服务器端的数据。
转移（Transfer）是：变化。
表述性状态转移是指：通过 URI + 请求方式 来控制服务器端数据的变化。
```
## 1.2 RESTFul风格与传统方式对比
```
传统的 URL 与 RESTful URL 的区别是传统的 URL 是基于方法名进行资源访问和操作，而 RESTful URL 是基于资源的结构和状态进行操作的。下面是一张表格，展示两者之间的具体区别：
传统的 URL	RESTful URL
GET /getUserById?id=1	GET /user/1
GET /getAllUser	GET /user
POST /addUser	POST /user
POST /modifyUser	PUT /user
GET /deleteUserById?id=1	DELETE /user/1
从上表中我们可以看出，传统的URL是基于动作的，而 RESTful URL 是基于资源和状态的，因此 RESTful URL 更加清晰和易于理解，这也是 REST 架构风格被广泛使用的主要原因之一。
```
## 1.3 RESTFul方式演示查询
```
RESTFul规范中规定，如果要查询数据，需要发送GET请求。
```
### 1.3.1 根据id查询(GET /api/user/1)
```
首先是springmvc.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans 
        http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context 
        https://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/mvc 
        https://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <!--组件扫描-->
    <context:component-scan base-package="com.powernode.springmvc.controller"/>

    <!--视图解析器-->
    <bean id="thymeleafViewResolver" class="org.thymeleaf.spring6.view.ThymeleafViewResolver">
        <property name="characterEncoding" value="UTF-8"/>
        <property name="order" value="1"/>
        <property name="templateEngine">
            <bean class="org.thymeleaf.spring6.SpringTemplateEngine">
                <property name="templateResolver">
                    <bean class="org.thymeleaf.spring6.templateresolver.SpringResourceTemplateResolver">
                        <property name="prefix" value="/WEB-INF/thymeleaf/"/>
                        <property name="suffix" value=".html"/>
                        <property name="templateMode" value="HTML"/>
                        <property name="characterEncoding" value="UTF-8"/>
                    </bean>
                </property>
            </bean>
        </property>
    </bean>

    <!--启用注解-->
    <mvc:annotation-driven/>

    <!--视图控制器映射-->
    <mvc:view-controller path="/" view-name="index"/>
</beans>

其次是index.html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>index</title>
</head>
<body>
<h1>index page</h1>
<hr>
<!--根据id查询：GET /api/user/1 -->
<a th:href="@{/api/user/1}">根据id查询用户信息</a><br>

</body>
</html>

然后是controller
package com.powernode.springmvc.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

@Controller
public class UserController {

    @RequestMapping(value = "/api/user/{id}", method = RequestMethod.GET)
    public String getById(@PathVariable("id") Integer id){
        System.out.println("根据用户id查询用户信息，用户id是" + id);
        return "ok";
    }

}

视图页面
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>ok</title>
</head>
<body>
<h1>ok</h1>
</body>
</html>

最后启动测试
```
### 1.3.2 查询所有(GET /api/user)
```
index.html
<!--查询所有-->
<a th:href="@{/api/user}">查询所有</a><br>

UserController
@RequestMapping(value = "/api/user", method = RequestMethod.GET)
public String getAll(){
    System.out.println("查询所有用户信息");
    return "ok";
}

测试

```
## 1.4 RESTFul方式演示增加（POST/api/user）
```
RESTFul规范中规定，如果要进行保存操作，需要发送POST请求
index.html
<!--保存用户-->
<form th:action="@{/api/user}" method="post">
    <input type="submit" th:value="保存">
</form>

UserController
@RequestMapping(value = "/api/user", method = RequestMethod.POST)
public String save(){
    System.out.println("保存用户信息");
    return "ok";
}
```
## 1.5 RESTFul方式演示修改
```
RESTFul规范中规定，如果要进行保存操作，需要发送PUT请求。（但是实际上RESTful 规范里从来没有“PUT 要先 POST，再靠 _method=PUT”这种规定。这一套，不是 REST 规范，而是 Spring MVC 为 HTML 表单提供的“兼容方案”）
如何发送PUT请求？
第一步：首先你必须是一个POST请求。
第二步：在发送POST请求的时候，提交这样的数据：_method=PUT
第三步：在web.xml文件配置SpringMVC提供的过滤器：HiddenHttpMethodFilter
实践一下：
index.html
<!--修改用户-->
<hr>
<form th:action="@{/api/user}" method="post">
    <!--隐藏域的方式提交 _method=put -->
    <input type="hidden" name="_method" value="put">
    用户名：<input type="text" name="username"><br>
    <input type="submit" th:value="修改">
</form>

web.xml
<!--隐藏的HTTP请求方式过滤器-->
<filter>
    <filter-name>hiddenHttpMethodFilter</filter-name>
    <filter-class>org.springframework.web.filter.HiddenHttpMethodFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>hiddenHttpMethodFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>

UserController
@RequestMapping(value = "/api/user", method = RequestMethod.PUT)
public String update(String username){
    System.out.println("修改用户信息，用户名：" + username);
    return "ok";
}

pom.xml添加如下配置
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.12.1</version>
            <configuration>
                <source>21</source>
                <target>21</target>
                <compilerArgs>
                    <arg>-parameters</arg>
                </compilerArgs>
            </configuration>
        </plugin>
    </plugins>
</build>

重新build一下

测试

```
## 1.6 HiddenHttpMethodFilter
```
HiddenHttpMethodFilter是Spring MVC框架提供的，专门用于RESTFul编程风格。
实现原理可以通过源码查看：
...
通过源码可以看到，if语句中，首先判断是否为POST请求，如果是POST请求，调用request.getParameter(this.methodParam)。可以看到this.methodParam是_method，这样就要求我们在提交请求方式的时候必须采用这个格式：_method=put。获取到请求方式之后，调用了toUpperCase转换成大写了。因此前端页面中小写的put或者大写的PUT都是可以的。if语句中嵌套的if语句说的是，只有请求方式是 PUT,DELETE,PATCH的时候会创建HttpMethodRequestWrapper对象。而HttpMethodRequestWrapper对象的构造方法是这样的：
...
这样method就从POST变成了：PUT/DELETE/PATCH。
重点注意事项：CharacterEncodingFilter和HiddenHttpMethodFilter的顺序
细心的同学应该注意到了，在HiddenHttpMethodFilter源码中有这样一行代码：
...
大家是否还记得，字符编码过滤器执行之前不能调用 request.getParameter方法，如果提前调用了，乱码问题就无法解决了。因为request.setCharacterEncoding()方法的执行必须在所有request.getParameter()方法之前执行。因此这两个过滤器就有先后顺序的要求，在web.xml文件中，应该先配置CharacterEncodingFilter，然后再配置HiddenHttpMethodFilter。
```
## 1.7 RESTFul方式演示删除（DELETE /api/user/1）
```
RESTFul规范中规定，如果要删除数据，需要发送DELETE请求。
在浏览器中，HTML表单只原生支持GET和POST请求方式。和PUT请求一样，发送DELETE请求也需要使用POST表单 + HiddenHttpMethodFilter的配合。

index.html中添加删除表单：
<!--删除用户-->
<hr>
<form th:action="@{/api/user/1}" method="post">
    <!--隐藏域的方式提交 _method=delete -->
    <input type="hidden" name="_method" value="delete">
    <input type="submit" th:value="删除id=1的用户">
</form>

UserController中添加删除方法：
@RequestMapping(value = "/api/user/{id}", method = RequestMethod.DELETE)
public String delete(@PathVariable("id") Integer id){
    System.out.println("删除用户信息，用户id是" + id);
    return "ok";
}

也可以使用更加简洁的 @DeleteMapping 注解：
@DeleteMapping("/api/user/{id}")
public String deleteById(@PathVariable("id") Integer id){
    System.out.println("删除用户信息，用户id是" + id);
    return "ok";
}

测试：
点击删除按钮，控制台输出：删除用户信息，用户id是1

注意：
● DELETE请求也有路径变量 {id}，因此和 GET 请求的URL格式相同（都是 /api/user/{id}），但请求方式不同，不会产生冲突。
● DELETE和GET的区别在于HTTP请求方式不同，SpringMVC会根据method属性进行区分。
● 同样需要在web.xml中配置HiddenHttpMethodFilter（如果之前已配置则无需重复配置）。
```