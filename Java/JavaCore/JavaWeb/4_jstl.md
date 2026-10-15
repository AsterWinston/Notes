# 1. JSTL标签库

```
什么是JSTL标签库？

Java Standard Tag Lib（Java标准的标签库）
JSTL标签库通常结合EL表达式一起使用。目的是让JSP中的java代码消失。
标签是写在JSP当中的，但实际上最终还是要执行对应的java程序。（java程序在jar包当中。）
使用JSTL标签库的步骤：

第一步：引入JSTL标签库对应的jar包。

tomcat10之后引入的jar包是：
jakarta.servlet.jsp.jstl-2.0.0.jar
jakarta.servlet.jsp.jstl-api-2.0.0.jar
在IDEA当中怎么引入？
在WEB-INF下新建lib目录，然后将jar包拷贝到lib当中。然后将其“Add Lib…”
一定是要和mysql的数据库驱动一样，都是放在WEB-INF/lib目录下的。
什么时候需要将jar包放到WEB-INF/lib目录下？如果这个jar是tomcat服务器没有的。
第二步：在JSP中引入要使用标签库。（使用taglib指令引入标签库。）

JSTL提供了很多种标签，你要引入哪个标签？？？？重点掌握核心标签库。
<%@taglib prefix="c" uri="[http://java.sun.com/jsp/jstl/core](http://java.sun.com/jsp/jstl/core)" %>
这个就是核心标签库。
prefix="这里随便起一个名字就行了，核心标签库，大家默认的叫做c，你随意。"
第三步：在需要使用标签的位置使用即可。表面使用的是标签，底层实际上还是java程序。
```

# 2. JSTL标签的原理

```
<%@taglib prefix="c" uri="[http://java.sun.com/jsp/jstl/core](http://java.sun.com/jsp/jstl/core)" %>
以上uri后面的路径实际上指向了一个xxx.tld文件。
tld文件实际上是一个xml配置文件。
在tld文件中描述了“标签”和“java类”之间的关系。
以上核心标签库对应的tld文件是：c.tld文件。它在哪里。
在jakarta.servlet.jsp.jstl-2.0.0.jar里面META-INF目录下，有一个c.tld文件。

源码解析：配置文件tld解析
<tag>
    <description>对该标签的描述</description>
    <name>catch</name> 标签的名字
    <tag-class>org.apache.taglibs.standard.tag.common.core.CatchTag</tag-class> 标签对应的java类。
    <body-content>JSP</body-content> 标签体当中可以出现的内容，如果是JSP，就表示标签体中可以出现符合JSP所有语法的代码。例如EL表达式。
    <attribute>
        <description>
        	对这个属性的描述
        </description>
        <name>var</name> 属性名
        <required>false</required> false表示该属性不是必须的。true表示该属性是必须的。
        <rtexprvalue>false</rtexprvalue> 这个描述说明了该属性是否支持EL表达式。false表示不支持。true表示支持EL表达式。
    </attribute>
  </tag>

<c:catch var="">
	JSP....
</c:catch>
jstl中的核心标签库core当中有哪些常用的标签呢？

c:if

<c:if test=“boolean类型，支持EL表达式”></c: if>
c:forEach

<c:forEach items=“集合，支持EL表达式” var=“集合中的元素” varStatus=“元素状态对象”> ${元素状态对象.count} </c: forEach>
<c:forEach var=“i” begin=“1” end=“10” step=“2”> ${i} </c: forEach>
c:choose c:when c:otherwise
<c:choose>
    <c:when test="${param.age < 18}">
        青少年
    </c:when>
    <c:when test="${param.age < 35}">
        青年
    </c:when>
    <c:when test="${param.age < 55}">
        中年
    </c:when>
    <c:otherwise>
        老年
    </c:otherwise>
</c:choose>
```

# 3. 核心标签库core的更多标签

```
c:set —— 向域中存储数据

<!-- 向pageContext域存（默认） -->
<c:set var="username" value="zhangsan"/>

<!-- 指定域范围 -->
<c:set var="username" value="zhangsan" scope="request"/>
<c:set var="username" value="zhangsan" scope="session"/>
<c:set var="username" value="zhangsan" scope="application"/>

<!-- 向JavaBean中存数据 -->
<c:set target="${user}" property="username" value="lisi"/>
等价于：user.setUsername("lisi")

c:out —— 输出数据（自动转义HTML特殊字符，防止XSS攻击）

<c:out value="${username}"/>
<c:out value="${username}" default="游客"/>  <!-- 为null时输出默认值 -->
<c:out value="${htmlContent}" escapeXml="true"/>  <!-- 默认true，转义HTML标签 -->

c:remove —— 从域中删除数据

<c:remove var="username"/>                <!-- 删除所有域中的username -->
<c:remove var="username" scope="session"/> <!-- 只删除session域中的username -->

c:url —— 动态生成URL（自动添加sessionid、自动编码中文参数）

<c:url value="/dept/list.jsp" var="url">
    <c:param name="deptno" value="${dept.deptno}"/>
    <c:param name="name" value="张三"/>
</c:url>
<a href="${url}">查看详情</a>
<!-- 生成的URL会自动添加contextPath，并对中文参数进行URL编码 -->
<!-- 如果浏览器禁用了Cookie，还会自动追加jsessionid -->

c:import —— 导入外部资源（比<jsp:include>更强大，可以引入站外资源）

<c:import url="http://www.example.com/news.html" var="news" charEncoding="UTF-8"/>
${news}

c:redirect —— 重定向

<c:redirect url="/login.jsp"/>
<c:redirect url="/error.jsp">
    <c:param name="msg" value="操作失败"/>
</c:redirect>

c:forTokens —— 按分隔符遍历字符串

<%
    request.setAttribute("hobbies", "篮球,足球,游泳,跑步");
%>
<c:forTokens items="${hobbies}" delims="," var="hobby">
    ${hobby} &nbsp;&nbsp;
</c:forTokens>
<!-- 输出：篮球 足球 游泳 跑步 -->

c:forEach 的 varStatus 属性详解

<c:forEach items="${list}" var="item" varStatus="status">
    ${status.count}    <!-- 循环计数(从1开始)  类似 rowNum -->
    ${status.index}    <!-- 循环索引(从0开始) -->
    ${status.first}    <!-- 是否为第一个元素(boolean) -->
    ${status.last}     <!-- 是否为最后一个元素(boolean) -->
    ${status.current}  <!-- 当前元素对象，等价于 ${item} -->
    ${status.begin}    <!-- begin属性值 -->
    ${status.end}      <!-- end属性值 -->
    ${status.step}     <!-- step属性值 -->
</c:forEach>

c:catch —— 捕获异常（类似于try-catch）

<c:catch var="ex">
    <%
        int i = 10 / 0;
    %>
</c:catch>
<c:if test="${not empty ex}">
    发生了异常：${ex.message}
</c:if>
```

# 4. 格式化标签库（fmt）

```
fmt标签库主要用来格式化数据（日期、数字）和实现国际化（i18n）。

使用前引入：
<%@taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt" %>

fmt:formatDate —— 格式化日期

<%
    request.setAttribute("now", new java.util.Date());
%>
<fmt:formatDate value="${now}" pattern="yyyy-MM-dd"/>
<fmt:formatDate value="${now}" pattern="yyyy-MM-dd HH:mm:ss"/>
<fmt:formatDate value="${now}" pattern="yyyy年MM月dd日 EEEE"/>

<!-- 日期风格格式化 -->
<fmt:formatDate value="${now}" type="date" dateStyle="long"/>     <!-- 长日期 -->
<fmt:formatDate value="${now}" type="time" timeStyle="short"/>    <!-- 短时间 -->
<fmt:formatDate value="${now}" type="both" dateStyle="full" timeStyle="medium"/>

fmt:parseDate —— 将字符串解析为日期

<fmt:parseDate value="2024-01-15" pattern="yyyy-MM-dd" var="parsedDate"/>

fmt:formatNumber —— 格式化数字

${balance} = 1234567.89

<fmt:formatNumber value="1234567.89" pattern="#,###.##"/>     <!-- 1,234,567.89 -->
<fmt:formatNumber value="1234567.89" pattern="000,000.00"/>   <!-- 空格补齐 -->
<fmt:formatNumber value="0.75" type="percent"/>               <!-- 75% 百分比 -->
<fmt:formatNumber value="12345.67" type="currency" currencySymbol="¥"/> <!-- ¥12,345.67 -->

fmt:parseNumber —— 将字符串解析为数字

<fmt:parseNumber value="1,234,567.89" pattern="#,###.##" var="num"/>

fmt:setLocale —— 设置国际化区域

<fmt:setLocale value="zh_CN"/>  <!-- 中国 -->
<fmt:setLocale value="en_US"/>  <!-- 美国 -->
<fmt:setLocale value="ja_JP"/>  <!-- 日本 -->

fmt:setBundle —— 设置资源包

<fmt:setBundle basename="messages" var="bundle"/>

fmt:message —— 读取国际化资源文件中的内容

<!-- messages.properties 或 messages_zh_CN.properties -->
<fmt:message key="welcome.title"/>
<fmt:message key="login.btn">
    <fmt:param value="张三"/>  <!-- 传递参数 -->
</fmt:message>

fmt:setTimeZone / fmt:timeZone —— 设置时区

<fmt:setTimeZone value="Asia/Shanghai" var="tz"/>
<fmt:timeZone value="${tz}">
    <fmt:formatDate value="${now}" type="both"/>
</fmt:timeZone>
```

# 5. JSTL函数库（fn）

```
fn函数库提供了一系列字符串操作函数，可以配合EL表达式使用。

使用前引入：
<%@taglib prefix="fn" uri="http://java.sun.com/jsp/jstl/functions" %>

常用fn函数一览：

${fn:length(list)}                    <!-- 获取集合/数组/字符串的长度 -->
${fn:contains("abcdef", "cd")}        <!-- 判断是否包含子串，返回boolean -->
${fn:containsIgnoreCase("ABC", "ab")} <!-- 忽略大小写判断是否包含 -->
${fn:startsWith("hello.jsp", "hello")}<!-- 判断是否以某字符串开头 -->
${fn:endsWith("hello.jsp", ".jsp")}   <!-- 判断是否以某字符串结尾 -->
${fn:indexOf("abcdef", "cd")}         <!-- 返回子串第一次出现的索引，找不到返回-1 -->
${fn:substring("abcdef", 1, 3)}       <!-- 截取子串：[1, 3)  结果：bc -->
${fn:substringAfter("a/b/c", "/")}    <!-- 截取某子串之后的部分  结果：b/c -->
${fn:substringBefore("a/b/c", "/")}   <!-- 截取某子串之前的部分  结果：a -->
${fn:toUpperCase("hello")}            <!-- 转大写：HELLO -->
${fn:toLowerCase("HELLO")}            <!-- 转小写：hello -->
${fn:trim("  hello  ")}               <!-- 去除首尾空格 -->
${fn:replace("hello world", "world", "jsp")} <!-- 替换：hello jsp -->
${fn:split("a,b,c", ",")}             <!-- 按分隔符拆分成数组 -->
${fn:join(array, "-")}                <!-- 用连接符将数组连成字符串 -->
${fn:escapeXml("<b>hello</b>")}       <!-- 转义XML/HTML特殊字符 -->

实战示例：判断用户名长度和内容

<c:if test="${fn:length(param.username) < 3}">
    用户名不能少于3个字符！
</c:if>

<c:if test="${fn:contains(param.username, '@')}">
    用户名不能包含@符号！
</c:if>
```

