# 1. EL表达式

```
EL表达式是干什么用的？

Expression Language（表达式语言）
EL表达式可以代替JSP中的java代码，让JSP文件中的程序看起来更加整洁，美观。
JSP中夹杂着各种java代码，例如<% java代码 %>、<%=%>等，导致JSP文件很混乱，不好看，不好维护。所以才有了后期的EL表达式。
EL表达式可以算是JSP语法的一部分。EL表达式归属于JSP。
EL表达式出现在JSP中主要是：

从某个作用域中取数据，然后将其转换成字符串，然后将其输出到浏览器。这就是EL表达式的功效。三大功效：
第一功效：从某个域中取数据。
四个域：
pageContext
request
session
application
第二功效：将取出的数据转成字符串。
如果是一个java对象，也会自动调用java对象的toString方法将其转换成字符串。
第三功效：将字符串输出到浏览器。
和这个一样：<%= %>，将其输出到浏览器。
```

# 2. EL表达式很好用，基本的语法格式：

```
${表达式}
```

# 3. EL表达式的使用：

```
<%
	// 创建User对象
	User user = new User();
	user.setUsername("jackson");
	user.setPassword("1234");
	user.setAge(50);

	// 将User对象存储到某个域当中。一定要存，因为EL表达式只能从某个范围中取数据。
	// 数据是必须存储到四大范围之一的。
	request.setAttribute("userObj", user);
%>

<%--使用EL表达式取--%>
${这个位置写什么？？？？这里写的一定是存储到域对象当中时的name}
要这样写：
${userObj}
等同于java代码：<%=request.getAttribute("userObj")%>
你不要这样写：${"userObj"}

面试题：
	${abc} 和 ${"abc"}的区别是什么？
		${abc}表示从某个域中取出数据，并且被取的这个数据的name是"abc"，之前一定有这样的代码: 域.setAttribute("abc", 对象);
		${"abc"} 表示直接将"abc"当做普通字符串输出到浏览器。不会从某个域中取数据了。

${userObj} 底层是怎么做的？从域中取数据，取出user对象，然后调用user对象的toString方法，转换成字符串，输出到浏览器。

<%--如果想输出对象的属性值，怎么办？--%>
${userObj.username} 使用这个语法的前提是：User对象有getUsername()方法。
${userObj.password} 使用这个语法的前提是：User对象有getPassword()方法。
${userObj.age} 使用这个语法的前提是：User对象有getAge()方法。
${userObj.email} 使用这个语法的前提是：User对象有getEmail()方法。
EL表达式中的. 这个语法，实际上调用了底层的getXxx()方法。
注意：如果没有对应的get方法，则出现异常。报500错误。

${userObj.addr222.zipcode}
以上EL表达式对应的java代码：
user.getAddr222().getZipcode()

EL表达式优先从小范围中读取数据。

pageContext < request < session < application
EL表达式中有四个隐含的隐式的范围：

pageScope 对应的是 pageContext范围。
requestScope 对应的是 request范围。
sessionScope 对应的是 session范围。
applicationScope 对应的是 application范围。
EL表达式对null进行了预处理。如果是null，则向浏览器输出一个空字符串。

EL表达式取数据的时候有两种形式：

第一种：. （大部分使用这种方式）
第二种：[ ] （如果存储到域的时候，这个name中含有特殊字符，可以使用 [ ]）
request.setAttribute(“abc.def”, “zhangsan”);
${requestScope.abc.def} 这样是无法取值的。
应该这样：${requestScope[“abc.def”]}
掌握使用EL表达式，怎么从Map集合中取数据：

${map.key}
掌握使用EL表达式，怎么从数组和List集合中取数据：

${数组[0]}
${数组[1]}
${list[0]}
page指令当中，有一个属性，可以忽略EL表达式

<%@page contentType="text/html;charset=UTF-8" isELIgnored="true" %>
isELIgnored="true" 表示忽略EL表达式
isELIgnored="false" 表示不忽略EL表达式。（这是默认值）

isELIgnored="true" 这个是全局的控制。

可以使用反斜杠进行局部控制：\${username} 这样也可以忽略EL表达式。

通过EL表达式获取应用的根：

${pageContext.request.contextPath}
EL表达式中其他的隐式对象：

pageContext
param
paramValues
initParam
EL表达式的运算符

算术运算符
+、-、*、/、%
关系运算符
== eq != > >= < <=
逻辑运算符
! && || not and or
条件运算符
? :
取值运算符
[ ]和.
empty运算符
empty运算符的结果是boolean类型
${empty param.username}
${not empty param.username}
${!empty param.password}
```

# 4. EL隐式对象详解

```
EL表达式提供了11个隐式对象，分为以下几类：

一、作用域对象（4个）
pageScope       --> javax.servlet.jsp.PageContext（页面作用域）
requestScope    --> javax.servlet.http.HttpServletRequest（请求作用域）
sessionScope    --> javax.servlet.http.HttpSession（会话作用域）
applicationScope --> javax.servlet.ServletContext（应用作用域）

用法：${requestScope.userObj.username}
注意：不写作用域时，EL会按照 pageScope → requestScope → sessionScope → applicationScope 的顺序查找。

二、请求参数对象（2个）
param           --> 获取单个请求参数
paramValues     --> 获取请求参数数组（适用于checkbox等多选场景）

${param.username}          -- 等价于 request.getParameter("username")
${paramValues.aihao[0]}    -- 等价于 request.getParameterValues("aihao")[0]
${empty param.username}    -- 判断参数是否为空

三、请求头对象（2个）
header          --> 获取单个请求头
headerValues    --> 获取请求头数组

${header["User-Agent"]}            -- 获取浏览器信息
${header.referer}                  -- 获取来源页面地址
${header["Accept-Language"]}       -- 获取客户端语言偏好
${headerValues["Accept-Language"][0]}  -- 获取语言偏好数组第一个值

四、Cookie对象（1个）
cookie          --> 获取Cookie信息

${cookie.JSESSIONID.value}         -- 获取JSESSIONID的值
${cookie.username.value}           -- 获取名为username的cookie的值
${empty cookie.username}           -- 判断cookie是否存在

五、初始化参数对象（1个）
initParam       --> 获取web.xml中<context-param>配置的全局初始化参数

<!-- web.xml配置 -->
<context-param>
    <param-name>companyName</param-name>
    <param-value>xxx科技有限公司</param-value>
</context-param>

${initParam.companyName}           -- 获取companyName的值

六、JSP页面上下文对象（1个）
pageContext     --> javax.servlet.jsp.PageContext，提供了对JSP所有其他隐式对象的访问

${pageContext.request.contextPath}         -- 获取应用根路径（最常用，动态获取不写死）
${pageContext.request.method}              -- 获取请求方式（GET/POST）
${pageContext.request.remoteAddr}          -- 获取客户端IP
${pageContext.session.id}                  -- 获取session ID
${pageContext.servletContext.serverInfo}   -- 获取服务器信息
${pageContext.request.requestURI}          -- 获取请求URI
```

# 5. EL函数

```
EL函数允许在EL表达式中调用Java类的静态方法。

使用步骤：
第一步：编写一个Java类，包含public static方法。

package com.example.util;
public class StringUtil {
    public static String toUpperCase(String str) {
        return str == null ? "" : str.toUpperCase();
    }
    
    public static boolean isEmpty(String str) {
        return str == null || str.trim().length() == 0;
    }
}

第二步：在WEB-INF目录下创建一个tld文件，例如：mytag.tld

<?xml version="1.0" encoding="UTF-8"?>
<taglib xmlns="http://java.sun.com/xml/ns/javaee"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
        http://java.sun.com/xml/ns/javaee/web-jsptaglibrary_2_1.xsd"
        version="2.1">
    <tlib-version>1.0</tlib-version>
    <short-name>myfn</short-name>
    <uri>http://example.com/functions</uri>
    
    <function>
        <name>toUpper</name>
        <function-class>com.example.util.StringUtil</function-class>
        <function-signature>java.lang.String toUpperCase(java.lang.String)</function-signature>
    </function>
    
    <function>
        <name>isEmpty</name>
        <function-class>com.example.util.StringUtil</function-class>
        <function-signature>boolean isEmpty(java.lang.String)</function-signature>
    </function>
</taglib>

第三步：在JSP页面中引入并使用

<%@taglib prefix="myfn" uri="http://example.com/functions" %>
${myfn:toUpper("hello world")}  -- 输出 HELLO WORLD
${myfn:isEmpty(param.name)}     -- 判断参数是否为空
```