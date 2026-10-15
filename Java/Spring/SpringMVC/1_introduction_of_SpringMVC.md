# 1. 学习本章课程的知识储备
```
JavaSE
HTML+CSS+JS
Vue
AJAX+axios
Thymeleaf
Servlet
Maven
Spring
```
# 2. 什么是MVC
```
MVC架构模式相关课程，在老杜的JavaWeb课程中已经详细的讲解了，如果没有学过的，可以看这个视频：https://www.bilibili.com/video/BV1Z3411C7NZ
MVC是一种软件架构模式（是一种软件架构设计思想，不止Java开发中用到，其它语言也需要用到），它将应用分为三块：
	M：Model（模型）
	V：View（视图）
	C：Controller（控制器）
应用为什么要被分为三块，优点是什么？
	低耦合，扩展能力增强
	代码复用性增强
	代码可维护性增强
	高内聚，让程序员更加专注业务的开发
MVC将应用分为三块，每一块各司其职，都有自己专注的事情要做，他们属于分工协作，互相配合：
	Model：负责业务处理及数据的收集。
	View：负责数据的展示
	Controller：负责调度。它是一个调度中心，它来决定什么时候调用Model来处理业务，什么时候调用View视图来展示数据。
MVC架构模式如下所示：	
```
<img src="images/01.png" width="100%">

```
MVC架构模式的描述：前端浏览器发送请求给web服务器，web服务器中的Controller接收到用户的请求，Controller负责将前端提交的数据进行封装，然后Controller调用Model来处理业务，当Model处理完业务后会返回处理之后的数据给Controller，Controller再调用View来完成数据的展示，最终将结果响应给浏览器，浏览器进行渲染展示页面。

面试题：什么是三层模型，并说一说MVC架构模式与三层模型的区别？
```
<img src="images/02.png" width="100%">

```
MVC 和三层模型都采用了分层结构来设计应用程序，都是降低耦合度，提高扩展力，提高组件复用性。区别在于：他们的关注点不同，三层模型更加关注业务逻辑组件的划分。
MVC架构模式关注的是整个应用程序的层次关系和分离思想。现代的开发方式大部分都是MVC架构模式结合三层模型一起用。
```
# 3. 什么是SpringMVC
```
```
## 3.1 SpringMVC概述
```
SpringMVC是一个实现了MVC架构模式的Web框架，底层基于Servlet实现。
SpringMVC已经将MVC架构模式实现了，因此只要我们是基于SpringMVC框架写代码，编写的程序就是符合MVC架构模式的。（MVC的架子搭好了，我们只需要添添补补）
Spring框架中有一个子项目叫做Spring Web，Spring Web子项目当中包含很多模块，例如：
	Spring MVC
	Spring WebFlux
	Spring Web Services
	Spring Web Flow
    Spring WebSocket
    Spring Web Services Client
可见 SpringMVC是Spring Web子项目当中的一个模块。因此也可以说SpringMVC是Spring框架的一部分。
所以学习SpringMVC框架之前要先学习Spring框架中的IoC和AOP等内容。
另外，使用SpringMVC框架的时候同样也可以使用IoC和AOP。
以下就是Spring官方给出的Spring架构图，其中Web中的servlet指的就是Spring MVC：
```
<img src="images/03.png" width="100%">

## 3.2 SpringMVC帮我们做了什么
```
SpringMVC框架帮我们做了什么，与纯粹的Servlet开发有什么区别？
1.  入口控制：SpringMVC框架通过DispatcherServlet作为入口控制器，负责接收请求和分发请求。而在Servlet开发中，需要自己编写Servlet程序，并在web.xml中进行配置，才能接受和处理请求。 
2. 在SpringMVC中，表单提交时可以自动将表单数据绑定到相应的JavaBean对象中，只需要在控制器方法的参数列表中声明该JavaBean对象即可，无需手动获取和赋值表单数据。而在纯粹的Servlet开发中，这些都是需要自己手动完成的。
3.  IoC容器：SpringMVC框架通过IoC容器管理对象，只需要在配置文件中进行相应的配置即可获取实例对象，而在Servlet开发中需要手动创建对象实例。 
4.  统一处理请求：SpringMVC框架提供了拦截器、异常处理器等统一处理请求的机制，并且可以灵活地配置这些处理器。而在Servlet开发中，需要自行编写过滤器、异常处理器等，增加了代码的复杂度和开发难度。 
5.  视图解析：SpringMVC框架提供了多种视图模板，如JSP、Freemarker、Velocity等，并且支持国际化、主题等特性。而在Servlet开发中需要手动处理视图层，增加了代码的复杂度。
总之，与Servlet开发相比，SpringMVC框架可以帮我们节省很多时间和精力，减少代码的复杂度，更加专注于业务开发。同时，也提供了更多的功能和扩展性，可以更好地满足企业级应用的开发需求。
```
## 3.3 SpringMVC框架的特点
```
1. 	轻量级：相对于其他Web框架，Spring MVC框架比较小巧轻便。（只有几个几百KB左右的Jar包文件） 
2.  模块化：请求处理过程被分成多个模块，以模块化的方式进行处理。 
  a. 控制器模块：Controller
  b. 业务逻辑模块：Model
  c. 视图模块：View
3.  依赖注入：Spring MVC框架利用Spring框架的依赖注入功能实现对象的管理，实现松散耦合。 
4.  易于扩展：提供了很多口子，允许开发者根据需要插入自己的代码，以扩展实现应用程序的特殊需求。 
  a. Spring MVC框架允许开发人员通过自定义模块和组件来扩展和增强框架的功能。
  b. Spring MVC框架与其他Spring框架及第三方框架集成得非常紧密，这使得开发人员可以非常方便地集成其他框架，以获得更好的功能。
5.  易于测试：支持单元测试框架，提高代码质量和可维护性。 （对SpringMVC中的Controller测试时，不需要依靠Web服务器。）
6.  自动化配置：提供自动化配置，减少配置细节。 
  a. Spring MVC框架基于约定大于配置的原则，对常用的配置约定进行自动化配置。
7.  灵活性：Spring MVC框架支持多种视图技术，如JSP、FreeMarker、Thymeleaf、FreeMarker等，针对不同的视图配置不同的视图解析器即可。 
```
# 4. 本套教材相关版本
```
JDK版本：Java21
Maven版本：3.9.6
Tomcat版本：10
Spring版本：6.1.4
SpringMVC版本：6.1.4
IDEA版本：2023.3
Thymeleaf版本：3.1.2
```
# 5. 第一个SpringMVC程序
```
```

## 5.1 创建Maven模块
```
第一步：创建Empty Project，起名：springmvc。
第二步：设置springmvc工程的JDK版本：Java21。
第三步：设置maven。
第四步：创建Maven模块
第五步：将pom.xml文件中的打包方式修改为war
<groupId>com.powernode.springmvc</groupId>
<artifactId>springmvc-001</artifactId>
<version>1.0-SNAPSHOT</version>
<!-- 打包方式设置为war方式 -->
<packaging>war</packaging>
第六步：添加以下依赖
<dependencies>
    <!-- Spring MVC依赖 -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>6.1.4</version>
    </dependency>
    <!--日志框架Logback依赖-->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.5.3</version>
    </dependency>
    <!--Servlet依赖-->
    <dependency>
        <groupId>jakarta.servlet</groupId>
        <artifactId>jakarta.servlet-api</artifactId>
        <version>6.0.0</version>
        <scope>provided</scope>
    </dependency>
    <!--Spring6和Thymeleaf整合依赖-->
    <dependency>
        <groupId>org.thymeleaf</groupId>
        <artifactId>thymeleaf-spring6</artifactId>
        <version>3.1.2.RELEASE</version>
    </dependency>
</dependencies>
```
## 5.2 添加web支持（免了）
```
第一步：在main目录下创建一个webapp目录
第二步：添加web.xml配置文件
注意 web.xml 文件的位置：E:\Spring MVC\code\springmvc\springmvc-001\src\main\webapp\WEB-INF\web.xml
```
## 5.3 配置web.xml文件
```
Spring MVC是一个web框架，在javaweb中谁来负责接收请求，处理请求，以及响应呢？当然是Servlet。在SpringMVC框架中已经为我们写好了一个Servlet，它的名字叫做：DispatcherServlet，我们称其为前端控制器。既然是Servlet，那么它就需要在web.xml文件中进行配置：

<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/web-app_6_0.xsd"
         version="6.0">

    <!--SpringMVC提供的前端控制器-->
    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <!-- /* 表示任何一个请求都交给DispatcherServlet来处理 -->
        <!-- / 表示当请求不是xx.jsp的时候，DispatcherServlet来负责处理本次请求-->
        <!-- jsp本质就是Servlet，因此如果请求是jsp的话，应该走它自己的Servlet，而不应该走DispatcherServlet -->
        <!-- 因此我们的 url-pattern 使用 / -->
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>

DispatcherServlet是SpringMVC框架为我们提供的最核心的类，它是整个SpringMVC框架的前端控制器，负责接收HTTP请求、将请求路由到处理程序、处理响应信息，最终将响应返回给客户端。DispatcherServlet是Web应用程序的主要入口点之一，它的职责包括：
1. 	接收客户端的HTTP请求：DispatcherServlet监听来自Web浏览器的HTTP请求，然后根据请求的URL将请求数据解析为Request对象。 
2.  处理请求的URL：DispatcherServlet将请求的URL（Uniform Resource Locator）与处理程序进行匹配，确定要调用哪个控制器（Controller）来处理此请求。 
3.  调用相应的控制器：DispatcherServlet将请求发送给找到的控制器处理，控制器将执行业务逻辑，然后返回一个模型对象（Model）。 
4.  渲染视图：DispatcherServlet将调用视图引擎，将模型对象呈现为用户可以查看的HTML页面。 
5.  返回响应给客户端：DispatcherServlet将为用户生成的响应发送回浏览器，响应可以包括表单、JSON、XML、HTML以及其它类型的数据
```
## 5.4 编写控制器FirstController
```
DispatcherServlet接收到请求之后，会根据请求路径分发到对应的Controller，Controller来负责处理请求的核心业务。在SpringMVC框架中Controller是一个普通的Java类（一个普通的POJO类，不需要继承任何类或实现任何接口），需要注意的是：POJO类要纳入IoC容器来管理，POJO类的生命周期由Spring来管理，因此要使用注解标注：
package com.powernode.springmvc.controller;

import org.springframework.stereotype.Controller;

/**
* ClassName: FirstController
* Description:
* Datetime: 2024/3/13 11:56
* Author: 老杜@动力节点
* Version: 1.0
*/
@Controller
public class FirstController {
}

```
## 5.5 配置springmvc-servlet.xml文件
```
SpringMVC框架有它自己的配置文件，该配置文件的名字默认为：<servlet-name>-servlet.xml，默认存放的位置是WEB-INF 目录下：
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans 
      http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context 
      https://www.springframework.org/schema/context/spring-context.xsd">
    <!--组件扫描-->
    <context:component-scan base-package="com.powernode.springmvc.controller"/>
    <!--视图解析器-->
    <bean id="thymeleafViewResolver" class="org.thymeleaf.spring6.view.ThymeleafViewResolver">
        <!--作用于视图渲染的过程中，可以设置视图渲染后输出时采用的编码字符集-->
        <property name="characterEncoding" value="UTF-8"/>
        <!--如果配置多个视图解析器，它来决定优先使用哪个视图解析器，它的值越小优先级越高-->
        <property name="order" value="1"/>
        <!--当 ThymeleafViewResolver 渲染模板时，会使用该模板引擎来解析、编译和渲染模板-->
        <property name="templateEngine">
            <bean class="org.thymeleaf.spring6.SpringTemplateEngine">
                <!--用于指定 Thymeleaf 模板引擎使用的模板解析器。模板解析器负责根据模板位置、
                                模板资源名称、文件编码等信息，加载模板并对其进行解析-->
                <property name="templateResolver">
                    <bean class="org.thymeleaf.spring6.templateresolver.SpringResourceTemplateResolver">
                        <!--设置模板文件的位置（前缀）-->
                        <property name="prefix" value="/WEB-INF/templates/"/>
                        <!--设置模板文件后缀（后缀），Thymeleaf文件扩展名不一定是html，也可以是其他，例如txt，大部分都是html-->
                        <property name="suffix" value=".html"/>
                        <!--设置模板类型，例如：HTML,TEXT,JAVASCRIPT,CSS等-->
                        <property name="templateMode" value="HTML"/>
                        <!--用于模板文件在读取和解析过程中采用的编码字符集-->
                        <property name="characterEncoding" value="UTF-8"/>
                    </bean>
                </property>
            </bean>
        </property>
    </bean>
</beans>

在WEB-INF目录下新建springmvc-servlet.xml文件，并且提供以上配置信息。

以上配置主要两项：
    第一项：组件扫描。spring扫描这个包中的类，将这个包中的类实例化并纳入IoC容器的管理。
    第二项：视图解析器。视图解析器（View Resolver）的作用主要是将Controller方法返回的逻辑视图名称解析成实际的视图对象。视图解析器将解析出的视图对象返回给DispatcherServlet，并最终由DispatcherServlet将该视图对象转化为响应结果，呈现给用户。

注意：如果采用了其它视图，请配置对应的视图解析器，例如：
    JSP的视图解析器：InternalResourceViewResolver
    FreeMarker视图解析器：FreeMarkerViewResolver
    Velocity视图解析器：VelocityViewResolver
    
```

## 5.6 提供视图
```
在WEB-INF目录下新建templates目录，在templates目录中新建html文件，例如：first.html，并提供以下代码：
<!DOCTYPE html>
<!--指定 th 命名空间，让 Thymeleaf 标准表达式可以被解析和执行-->
<!--th不是固定的，可以指定其它的命名空间，只不过大部分情况下用th-->
<!--表示程序中出现的 th 开头的后面代码都是 Thymeleaf语法，需要被 Thymeleaf识别-->
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>first springmvc</title>
</head>
<body>
<h1>我的第一个Spring MVC程序</h1>
</body>
</html>

对于每一个Thymeleaf文件来说 xmlns:th="http://www.thymeleaf.org" 是必须要写的，为了方便后续开发，可以将其添加到html模板文件中
```

## 5.7 控制器FirstController处理请求返回逻辑视图名称
```
package com.powernode.springmvc.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

/**
 * ClassName: FirstController
 * Description:
 * Datetime: 2024/3/13 11:56
 * Author: 老杜@动力节点
 * Version: 1.0
 */
@Controller
public class FirstController {
    @RequestMapping(value="/haha")
    public String 名字随意(){
        System.out.println("正在处理请求....");
        // 返回逻辑视图名称（决定跳转到哪个页面）
        return "first";
    }
}
```
## 5.8 测试
```
第一步：配置Tomcat服务器
第二步：部署web模块到Tomcat服务器
第三步：启动Tomcat服务器。如果在控制台输出的信息有中文乱码，请修改tomcat服务器配置文件：apache-tomcat-10.1.19\conf\logging.properties（实则加启动参数-Dfile.encoding=UTF-8即可）
第四步：打开浏览器，在浏览器地址栏上输入地址：http://localhost:8080/springmvc/haha
```
## 5.9 执行流程总结
```
1. 浏览器发送请求：http://localhost:8080/springmvc/haha
2. SpringMVC的前端控制器DispatcherServlet接收到请求
3. DispatcherServlet根据请求路径 /haha 映射到 FirstController#名字随意()，调用该方法
4. FirstController#名字随意() 处理请求
5. FirstController#名字随意() 返回逻辑视图名称 first 给视图解析器
6. 视图解析器找到 /WEB-INF/templates/first.html 文件，并进行解析，生成视图解析对象返回给前端控制器DispatcherServlet
7. 前端控制器DispatcherServlet响应结果到浏览器。
```
## 5.10 一个Controller可以编写多个方法
```
一个Controller可以提供多个方法，每个方法通常是处理对应的请求，例如：
@Controller
public class FirstController {
    @RequestMapping(value="/haha")
    public String 名字随意(){
        System.out.println("正在处理请求....");
        // 返回逻辑视图名称（决定跳转到哪个页面）
        return "first";
    }

    @RequestMapping("/other")
    public String other(){
        System.out.println("正在处理其它请求...");
        return "other";
    }
}

提供 other.html 文件
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>other</title>
</head>
<body>
<h1>other ...</h1>
</body>
</html>

在 first.html 文件中，添加超链接，用超链接发送 /other 请求：
<!DOCTYPE html>
<!--指定 th 命名空间，让 Thymeleaf 标准表达式可以被解析和执行-->
<!--th不是固定的，可以指定其它的命名空间，只不过大部分情况下用th-->
<!--表示程序中出现的 th 开头的后面代码都是 Thymeleaf语法，需要被 Thymeleaf识别-->
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>first springmvc</title>
</head>
<body>
<h1>我的第一个Spring MVC程序</h1>
<!-- th: 表示后面的代码可以编写Thymeleaf语法，可以被Thymeleaf语法解析 -->
<!-- Thymeleaf检测到以 / 开始，表示绝对路径，自动会将webapp的上下文路径加上去 -->
<!-- 最终的效果是：href="/springmvc/other" -->
<a th:href="@{/other}">other请求</a>
</body>
</html>

启动Tomcat，打开浏览器，输入请求路径：http://localhost:8080/springmvc/haha
点击超链接：other请求
```
# 6. 第二个SpringMVC程序
```
```
## 6.1 创建Maven模块
```
1. pom.xml文件中添加依赖
  a. springmvc依赖
  b. logback依赖
  c. servlet依赖（scope为provided）
  d. thymeleaf与spring6整合依赖
2. 打包方式war
```
## 6.2 添加web支持（社区版做梦去）

```
webapp目录没有小蓝点怎么办？添加web支持
```
## 6.3 配置web.xml文件
```
重点：SpringMVC配置文件的名字和路径是可以手动设置的，如下
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <!--配置前端控制器-->
    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!--手动设置springmvc配置文件的路径及名字-->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:springmvc.xml</param-value>
        </init-param>
        <!--为了提高用户的第一次访问效率，建议在web服务器启动时初始化前端控制器-->
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>

通过<init-param>来设置SpringMVC配置文件的路径和名字。在DispatcherServlet的init方法执行时设置的。
<load-on-startup>1</load-on-startup>建议加上，这样可以提高用户第一次访问的效率。表示在web服务器启动时初始化DispatcherServlet。
```
## 6.4 编写IndexController
```
package com.powernode.springmvc.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

/**
 * ClassName: IndexController
 * Description:
 * Datetime: 2024/3/13 15:47
 * Author: 老杜@动力节点
 * Version: 1.0
 */
@Controller
public class IndexController {
    @RequestMapping("/")
    public String toIndex(){
        return "index";
    }
}

表示请求路径如果是：http://localhost:8080/springmvc/ ，则进入 /WEB-INF/templates/index.html 页面。
```

## 6.5 在resources目录下配置springmvc.xml文件
```
配置内容和之前一样，一个是视图解析器，一个是组件扫描。
```
## 6.6 提供视图
```
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>index page</title>
</head>
<body>
<h1>index page</h1>
</body>
</html>
```
## 6.7 测试
```
部署到web服务器，启动web服务器，打开浏览器，在地址栏上输入：http://localhost:8080/springmvc/
```

# 7. SpringMVC核心组件架构
```
SpringMVC框架围绕DispatcherServlet设计了多个核心组件，各组件职责分明、协同工作：

┌─────────────────────────────────────────────────────┐
│                   DispatcherServlet                  │
│                    （前端控制器）                      │
└─────────────────────────────────────────────────────┘
        │                │                │
        ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│HandlerMapping│ │HandlerAdapter│ │ ViewResolver │
│ （处理器映射） │ │ （处理器适配） │ │ （视图解析器） │
└──────────────┘ └──────────────┘ └──────────────┘
        │                │                │
        ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Handler    │ │  Controller  │ │    View      │
│ （处理器链）   │ │  （控制器）   │ │   （视图）   │
└──────────────┘ └──────────────┘ └──────────────┘

各组件职责：
● DispatcherServlet：前端控制器，统一接收所有请求，分发给各组件处理
● HandlerMapping：根据请求URL查找对应的处理器（Controller方法）和拦截器
  常用实现：RequestMappingHandlerMapping（处理@RequestMapping注解）
● HandlerAdapter：调用处理器方法，处理参数绑定、类型转换、返回值处理
  常用实现：RequestMappingHandlerAdapter
● HandlerInterceptor：拦截器，在处理器执行前后进行拦截处理
● ViewResolver：将逻辑视图名解析为具体的View对象
  常用实现：ThymeleafViewResolver、InternalResourceViewResolver
● View：负责渲染页面，将Model数据填充到视图中
  常用实现：ThymeleafView、InternalResourceView（JSP）

扩展组件：
● HttpMessageConverter：HTTP消息转换器，处理@RequestBody和@ResponseBody
● HandlerExceptionResolver：异常处理器，统一处理Controller抛出的异常
● MultipartResolver：文件上传解析器
● LocaleResolver：国际化区域解析器
```

# 8. SpringMVC与Spring Boot的关系
```
Spring Boot是对Spring框架的进一步封装，它自动完成了SpringMVC的大部分配置工作。

传统SpringMVC开发（本课程的方式）：
● 手动配置web.xml注册DispatcherServlet
● 手动配置springmvc.xml（组件扫描、视图解析器、注解驱动等）
● 手动配置CharacterEncodingFilter、HiddenHttpMethodFilter等过滤器
● 手动部署到外部Tomcat

Spring Boot方式：
● 无需web.xml：Spring Boot自动注册DispatcherServlet
● 无需springmvc.xml：通过application.properties/yml配置
● 自动配置：@SpringBootApplication包含@EnableAutoConfiguration
  Spring Boot会根据classpath中的依赖自动配置：
  - 有spring-webmvc → 自动配置DispatcherServlet
  - 有thymeleaf → 自动配置ThymeleafViewResolver
  - 有jackson → 自动配置MappingJackson2HttpMessageConverter
● 内嵌Tomcat：无需外部Servlet容器，直接java -jar运行

示例对比：
传统方式（5个文件）：web.xml + springmvc.xml + pom.xml + Controller + HTML
Spring Boot方式（2个文件）：pom.xml（spring-boot-starter-web） + Controller

理解传统SpringMVC的配置方式，有助于深入理解Spring Boot自动配置的底层原理。
在实际工作中，大部分新项目使用Spring Boot，但遗留系统维护和对底层原理的理解需要掌握传统SpringMVC配置。
```

# 9. Jakarta EE与javax命名空间迁移
```
从Spring 6 / Spring Boot 3开始，Spring全面迁移到了Jakarta EE 9+规范。

历史背景：
● Java EE由Oracle主导，使用javax.*命名空间（如javax.servlet）
● 2017年Oracle将Java EE捐献给Eclipse基金会
● Eclipse基金会将其更名为Jakarta EE，但Oracle不允许继续使用javax命名空间
● 因此Jakarta EE 9+所有API从javax.*迁移到了jakarta.*

关键变化对比：
| 旧（Java EE 8及以前） | 新（Jakarta EE 9+） |
|-----------------------|---------------------|
| javax.servlet.Servlet | jakarta.servlet.Servlet |
| javax.servlet.http.HttpServletRequest | jakarta.servlet.http.HttpServletRequest |
| javax.servlet.annotation.WebServlet | jakarta.servlet.annotation.WebServlet |
| javax.validation.Valid | jakarta.validation.Valid |

本课程使用的版本：
● Spring 6.1.4 → 基于Jakarta EE 9+
● Tomcat 10 → 实现了Jakarta EE 9+规范
● Servlet API：jakarta.servlet-api 6.0.0

注意事项：
● 如果项目使用Spring 5 / Spring Boot 2，则仍然使用javax.*
● 不能将javax和jakarta的依赖混用，会导致ClassNotFoundException
● Tomcat 9及以下只支持javax.*，Tomcat 10只支持jakarta.*
● 升级项目时需要全局替换import语句：javax.servlet.* → jakarta.servlet.*
```

# 10. Spring根容器与SpringMVC子容器
```
在SpringMVC项目中，存在两层IoC容器架构：
● 根容器（Root ApplicationContext）：由ContextLoaderListener创建，管理Service、Dao、数据源等
● 子容器（Servlet ApplicationContext）：由DispatcherServlet创建，管理Controller、ViewResolver、HandlerMapping等

继承关系：子容器可以访问父容器中的Bean，但父容器不能访问子容器中的Bean。

设计目的：
● 职责分离：业务层归根容器管，Web层归子容器管
● 隔离性：多个DispatcherServlet可以共享同一个根容器
● Controller中可以注入Service（子访问父），Service中不能注入Controller（父不能访问子）

配置方式：
web.xml中：
<!-- 根容器 -->
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:applicationContext.xml</param-value>
</context-param>

<!-- 子容器（DispatcherServlet） -->
<servlet>
    <servlet-name>dispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:springmvc.xml</param-value>
    </init-param>
</servlet>

在实际开发中，如果项目规模不大，也可以将所有Bean都配置在DispatcherServlet的容器中。
前面示例中的springmvc.xml没有拆分根容器，采用的是简单模式。
```

# 11. 初学者常见问题与调试技巧
```
11.1 访问404：找不到页面
原因排查：
● 检查@Controller是否被组件扫描到（base-package是否正确）
● 检查@RequestMapping的路径是否正确（是否加了项目前缀）
● 检查是否配置了<mvc:annotation-driven/>（注解驱动）
● 检查Thymeleaf模板文件是否存在（路径：/WEB-INF/templates/xxx.html）
● 检查DispatcherServlet的url-pattern是否正确（一般是"/"或"/*"）

11.2 访问500：服务器内部错误
原因排查：
● 检查Thymeleaf模板语法是否正确
● 检查Model中的属性名和模板中是否一致
● 查看控制台异常堆栈，定位具体错误

11.3 视图返回字符串没有渲染，直接显示字符串
原因排查：
● 检查是否添加了@ResponseBody注解（去掉或改用@Controller）
● 检查是否误用@RestController（它会自动添加@ResponseBody）

11.4 中文乱码
● GET请求：检查Tomcat server.xml中URIEncoding="UTF-8"
● POST请求：检查是否配置了CharacterEncodingFilter
● 模板文件：确保文件本身是UTF-8编码

11.5 IDEA热部署配置
● 配置Tomcat时选择"Update classes and resources"
● On 'Update' action: Update classes and resources
● On frame deactivation: Update classes and resources
● Thymeleaf开发时关闭缓存：spring.thymeleaf.cache=false
```

# 12. SpringMVC执行流程总结
```
一次完整的SpringMVC请求处理流程：

1. 用户发送请求 → 被DispatcherServlet拦截
2. DispatcherServlet调用HandlerMapping，根据URL查找Handler（Controller方法）
3. HandlerMapping返回HandlerExecutionChain（包含Handler + Interceptors）
4. DispatcherServlet调用HandlerAdapter，执行处理器方法
5. HandlerAdapter调用Controller方法：
   ● 参数解析：HandlerMethodArgumentResolver将请求参数绑定到方法参数
   ● 类型转换：Spring的类型转换器将字符串转为目标类型
   ● 数据校验：@Valid触发Bean Validation校验
6. Controller方法执行完毕，返回ModelAndView（或String/JSON等）
7. HandlerAdapter将返回值包装为ModelAndView
8. DispatcherServlet调用ViewResolver解析逻辑视图名为物理视图路径
9. View对象渲染页面（将Model数据填充到模板）
10. 响应返回给用户

如果是RESTful API（@ResponseBody/@RestController）：
● 第6步返回对象后，由HttpMessageConverter将对象转为JSON
● 第8-9步跳过（不需要视图渲染）
● 直接将JSON写入响应体返回

（详细的执行过程源码分析见第11章）
```