# 1. 我的第一个JSP程序

```
在WEB-INF目录之外创建一个index.jsp文件，然后这个文件中没有任何内容。
将上面的项目部署之后，启动服务器，打开浏览器，访问以下地址：

http://localhost:8080/jsp/index.jsp 展现在大家面前的是一个空白。
实际上访问以上的这个：index.jsp，底层执行的是：index_jsp.class 这个java程序。
这个index.jsp会被tomcat翻译生成index_jsp.java文件，然后tomcat服务器又会将index_jsp.java编译生成index_jsp.class文件
访问index.jsp，实际上执行的是index_jsp.class中的方法。
```

# 2. JSP实际上就是一个Servlet

```
index.jsp访问的时候，会自动翻译生成index_jsp.java，会自动编译生成index_jsp.class，那么index_jsp 这就是一个类。
index_jsp 类继承 HttpJspBase，而HttpJspBase类继承的是HttpServlet。所以index_jsp类就是一个Servlet类。
jsp的生命周期和Servlet的生命周期完全相同。完全就是一个东西。没有任何区别。
jsp和servlet一样，都是单例的。（假单例。）
jsp文件第一次访问的时候是比较慢的，为什么？

为什么大部分的运维人员在给客户演示项目的时候，为什么提前先把所有的jsp文件先访问一遍。
第一次比较麻烦：
要把jsp文件翻译生成java源文件
java源文件要编译生成class字节码文件
然后通过class去创建servlet对象
然后调用servlet对象的init方法
最后调用servlet对象的service方法。
第二次就比较快了，为什么？
因为第二次直接调用单例servlet对象的service方法即可。
```

# 3. JSP是什么？

```
JSP是java程序。（JSP本质还是一个Servlet）
JSP是：JavaServer Pages的缩写。（基于Java语言实现的服务器端的页面。）
Servlet是JavaEE的13个子规范之一，那么JSP也是JavaEE的13个子规范之一。
JSP是一套规范。所有的web容器/web服务器都是遵循这套规范的，都是按照这套规范进行的“翻译”
每一个web容器/web服务器都会内置一个JSP翻译引擎。
对JSP进行错误调试的时候，还是要直接打开JSP文件对应的java文件，检查java代码。

开发JSP的最高境界：

眼前是JSP代码，但是脑袋中呈现的是java代码。
JSP既然本质上是一个Servlet，那么JSP和Servlet到底有什么区别呢？

职责不同：
Servlet的职责是什么：收集数据。（Servlet的强项是逻辑处理，业务处理，然后链接数据库，获取/收集数据。）
JSP的职责是什么：展示数据。（JSP的强项是做数据的展示）
```

# 4. JSP的基础语法

```
在jsp文件中直接编写文字，都会自动被翻译到哪里？
翻译到servlet类的service方法的out.write(“翻译到这里”)，直接翻译到双引号里，被java程序当做普通字符串打印输出到浏览器。
在JSP中编写的HTML CSS JS代码，这些代码对于JSP来说只是一个普通的字符串。但是JSP把这个普通的字符串一旦输出到浏览器，浏览器就会对HTML CSS JS进行解释执行。展现一个效果。
JSP的page指令（这个指令后面再详细说，这里先解决一下中文乱码问题），解决响应时的中文乱码问题：
通过page指令来设置响应的内容类型，在内容类型的最后面添加：charset=UTF-8
<%@page contentType=“text/html;charset=UTF-8”%>，表示响应的内容类型是text/html，采用的字符集UTF-8
<%@page import=“java.util.List,java.util.ArrayList”%>
怎么在JSP中编写Java程序：
<% java语句; %>
在这个符号当中编写的被视为java程序，被翻译到Servlet类的service方法内部。
这里你要细心点，你要思考，在<% %>这个符号里面写java代码的时候，你要时时刻刻的记住你正在“方法体”当中写代码，方法体中可以写什么，不可以写什么，你心里是否明白呢？
在service方法当中编写的代码是有顺序的，方法体当中的代码要遵循自上而下的顺序依次逐行执行。
service方法当中不能写静态代码块，不能写方法，不能定义成员变量。。。。。。
在同一个JSP当中 <%%> 这个符号可以出现多个。
<%! %>
在这个符号当中编写的java程序会自动翻译到service方法之外。
这个语法很少用，为什么？不建议使用，因为在service方法外面写静态变量和实例变量，都会存在线程安全问题，因为JSP就是servlet，servlet是单例的，多线程并发的环境下，这个静态变量和实例变量一旦有修改操作，必然会存在线程安全问题。
JSP的输出语句
怎么向浏览器上输出一个java变量。
<% String name = “jack”; out.write("name = " + name); %>
注意：以上代码中的out是JSP的九大内置对象之一。可以直接拿来用。当然，必须只能在service方法内部使用。
如果向浏览器上输出的内容中没有“java代码”，例如输出的字符串是一个固定的字符串，可以直接在jsp中编写，不需要写到<%%> 这里。
如果输出的内容中含有“java代码”，这个时候可以使用以下语法格式：
<%= %> 注意：在=的后面编写要输出的内容。
<%= %> 这个符号会被翻译到哪里？最终翻译成什么？
翻译成了这个java代码： out.print();
翻译到service方法当中了。
什么时候使用<%=%> 输出呢？输出的内容中含有java的变量，输出的内容是一个动态的内容，不是一个死的字符串。如果输出的是一个固定的字符串，直接在JSP文件中编写即可。
在JSP中如何编写JSP的专业注释
<%–JSP的专业注释，不会被翻译到java源代码当中。–%>
JSP基础语法总结：
JSP中直接编写普通字符串
翻译到service方法的out.write(“这里”)
<%%>
翻译到service方法体内部，里面是一条一条的java语句。
<%! %>
翻译到service方法之外。
<%= %>
翻译到service方法体内部，翻译为：out.print();
<%@page contentType=“text/html;charset=UTF-8”%>
page指令，通过contentType属性用来设置响应的内容类型。
使用Servlet + JSP完成oa项目的改造。
使用Servlet处理业务，收集数据。 使用JSP展示数据。

将之前原型中的html文件，全部修改为jsp，然后在jsp文件头部添加page指令（指定contentType防止中文乱码），将所有的JSP直接拷贝到web目录下。

完成所有页面的正常流转。（页面仍然能够正常的跳转。修改超链接的请求路径。）

<%=request.getContextPath() %> 在JSP中动态的获取应用的根路径。
Servlet中连接数据库，查询所有的部门，遍历结果集。

遍历结果集的过程中，取出部门编号、部门名、位置等信息，封装成java对象。
将java对象存放到List集合中。
将List集合存储到request域当中。
转发forward到jsp。
在JSP中：

从request域当中取出List集合。
遍历List集合，取出每个部门对象。动态生成tr。
思考一个问题：如果我只用JSP这一个技术，能不能开发web应用？

当然可以使用JSP来完成所有的功能。因为JSP就是Servlet，在JSP的<%%>里面写的代码就是在service方法当中的，所以在<%%>当中完全可以编写JDBC代码，连接数据库，查询数据，也可以在这个方法当中编写业务逻辑代码，处理业务，都是可以的，所以使用单独的JSP开发web应用完全没问题。
虽然JSP一个技术就可以完成web应用，但是不建议，还是建议采用servlet + jsp的方式进行开发。这样都能将各自的优点发挥出来。JSP就是做数据展示。Servlet就是做数据的收集。（JSP中编写的Java代码越少越好。）一定要职责分明。
JSP文件的扩展名必须是xxx.jsp吗？

jsp文件的扩展名是可以配置的。不是固定的。

在CATALINA_HOME/conf/web.xml，在这个文件当中配置jsp文件的扩展名。

<servlet-mapping>
    <servlet-name>jsp</servlet-name>
    <url-pattern>*.jsp</url-pattern>
    <url-pattern>*.jspx</url-pattern>
</servlet-mapping>

xxx.jsp文件对于小猫咪来说，只是一个普通的文本文件，web容器会将xxx.jsp文件最终生成java程序，最终调用的是java对象相关的方法，真正执行的时候，和jsp文件就没有关系了。

小窍门：JSP如果看不懂，建议把jsp翻译成java代码，就能看懂了。

同学问：包名bean是什么意思？

javabean（java的logo是一杯冒着热气的咖啡。javabean被翻译为：咖啡豆）
java是一杯咖啡，咖啡又是由一粒一粒的咖啡豆研磨而成。
整个java程序中有很多bean的存在。由很多bean组成。
什么是javabean？实际上javabean你可以理解为符合某种规范的java类，比如：
有无参数构造方法
属性私有化
对外提供公开的set和get方法
实现java.io.Serializable接口
重写toString
重写hashCode+equals
…
javabean其实就是java中的实体类。负责数据的封装。
由于javabean符合javabean规范，具有更强的通用性。
完成剩下所有功能的改造。
```

# 5. 当前的oa应用存在的问题

```
任何一个用户都可以访问这个系统，都可以对这个系统当中的数据进行增删改这些危险的操作。我只想让合法的用户去使用这个系统，不合法的用户不能访问这个系统，怎么办？
加一个登录功能。登录成功的可以访问该系统，登录失败不能访问。
实现登录功能：
步骤1：数据库当中添加一个用户表：t_user
t_user表当中存储的是用户的登录信息，最基本的也包括：登录的用户名和登录的密码。
密码一般在数据库表当中存储的是密文。一般不以明文的形式存储。（这里先使用明文方式。）
向t_user表中插入数据。
步骤2：再实现一个登录页面。
登录页面上应该有一个登录的表单。有用户名和密码输入的框。
用户点击登录，提交表单，提交用户名和密码。form是post方式提交。
步骤3：后台要有一个对应的Servlet来处理登录的请求。
登录成功：跳转到部门列表页面。
登录失败：跳转到失败的页面。
步骤4：再提供一个登录失败的页面。
登录功能实现了，目前存在的最大的问题：

这个登录功能目前只是一个摆设，没有任何作用。只要用户知道后端的请求路径，照样可以在不登录的情况下访问。
这个登录没有真正起到拦截的作用。怎么解决？
```

# 6. JSP的指令

```
指令的作用：指导JSP的翻译引擎如何工作（指导当前的JSP翻译引擎如何翻译JSP文件。）

指令包括哪些呢？

include指令：包含指令，在JSP中完成静态包含，很少用了。（这里不讲）
taglib指令：引入标签库的指令。这个到JJSTL标签库的时候再学习。现在先不管。
page指令：目前重点学习一个page指令。
指令的使用语法是什么？

<%@指令名 属性名=属性值 属性名=属性值 属性名=属性值…%>
关于page指令当中都有哪些常用的属性呢？

<%@page session="true|false" %>
true表示启用JSP的内置对象session，表示一定启动session对象。没有session对象会创建。
如果没有设置，默认值就是session="true"
session="false" 表示不启动内置对象session。当前JSP页面中无法使用内置对象session。

<%@page contentType="text/json" %>
contentType属性用来设置响应的内容类型
但同时也可以设置字符集。
<%@page contentType="text/json;charset=UTF-8" %>

<%@page pageEncoding="UTF-8" %>
pageEncoding="UTF-8" 表示设置响应时采用的字符集。

<%@page import="java.util.List, java.util.Date, java.util.ArrayList" %>
<%@page import="java.util.*" %>
import语句，导包。

<%@page errorPage="/error.jsp" %>
当前页面出现异常之后，跳转到error.jsp页面。
errorPage属性用来指定出错之后的跳转位置。

<%@page isErrorPage="true" %>
表示启用JSP九大内置对象之一：exception
默认值是false。
```

# 7. JSP的九大内置对象

```
jakarta.servlet.jsp.PageContext pageContext 页面作用域

jakarta.servlet.http.HttpServletRequest request 请求作用域

jakarta.servlet.http.HttpSession session 会话作用域

jakarta.servlet.ServletContext application 应用作用域

pageContext < request < session < application
以上四个作用域都有：setAttribute、getAttribute、removeAttribute方法。
以上作用域的使用原则：尽可能使用小的域。
java.lang.Throwable exception

jakarta.servlet.ServletConfig config

java.lang.Object page （其实是this，当前的servlet对象）

jakarta.servlet.jsp.JspWriter out （负责输出）

jakarta.servlet.http.HttpServletResponse response （负责响应）
```

# 8. JSP的标准动作（Standard Actions）

```
JSP标准动作是使用XML语法格式的标签，用来控制JSP引擎的行为。

常用的JSP标准动作有哪些？

<jsp:forward>
转发动作。用于将请求转发给另一个资源（JSP、Servlet、HTML等）。

<jsp:forward page="/target.jsp">
    <jsp:param name="username" value="zhangsan"/>
</jsp:forward>

等价于java代码：request.getRequestDispatcher("/target.jsp").forward(request, response);

<jsp:include>
动态包含动作。在请求处理阶段包含另一个资源的输出。

<jsp:include page="/header.jsp">
    <jsp:param name="title" value="首页"/>
</jsp:include>

<%@include %> 与 <jsp:include> 的区别：
1. <%@include %>：静态包含（编译时包含）。先将两个文件合并，再翻译成一个Servlet。被包含的文件中不能有重复的变量名。
2. <jsp:include>：动态包含（运行时包含）。分别翻译成两个独立的Servlet，运行时将另一个的输出内容合并进来。两个文件可以有同名变量，互不影响。
3. 能用<jsp:include>尽量不用<%@include %>，因为动态包含更灵活，耦合度更低。
4. 静态包含因为合并成一个文件，效率略高，但可维护性差。

<jsp:useBean>
创建或查找JavaBean实例。

<jsp:useBean id="user" class="com.example.User" scope="request"/>

等价于java代码：
User user = (User)request.getAttribute("user");
if(user == null){
    user = new User();
    request.setAttribute("user", user);
}

<jsp:setProperty>
设置JavaBean的属性值。

<jsp:setProperty name="user" property="username" value="zhangsan"/>
等价于：user.setUsername("zhangsan")

<!-- 可以通过通配符，一次性将表单中的所有数据注入到JavaBean中 -->
<jsp:setProperty name="user" property="*"/>
注意：表单中input的name必须和bean的属性名一致才能自动注入。

<jsp:getProperty>
获取JavaBean的属性值，输出到页面。

<jsp:getProperty name="user" property="username"/>
等价于：<%=user.getUsername()%>

<jsp:param>
为其他动作标签（forward、include）传递参数。不能单独使用。

<jsp:element> / <jsp:attribute> / <jsp:body>
用于动态生成XML元素（较少使用）。

总结：JSP标准动作的本质
这些标签最终都会被翻译成对应的Java代码：
- <jsp:forward>     → request.getRequestDispatcher().forward()
- <jsp:include>     → request.getRequestDispatcher().include()
- <jsp:useBean>     → request.getAttribute() / class.newInstance()
- <jsp:setProperty> → bean.setXxx()
- <jsp:getProperty> → bean.getXxx() 并输出
```
