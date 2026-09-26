
# 1. 概述

## 1.1 什么是 JSF

JSF（JavaServer Faces）是 JavaEE 的**服务端组件式 Web 框架**，核心思想是"**组件 + 事件驱动**"——开发者面向可复用的 UI 组件编程，框架负责组件渲染、请求处理、状态管理。

```
┌────────────────────────────────────────────────────┐
│                    JSF 框架                        │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐  │
│  │ Facelets   │   │  组件树    │   │  托管Bean   │  │
│  │ (视图模板) │──►│(UIComponent│──►│ (Backing    │  │
│  │            │   │   树)     │   │  Bean)      │  │
│  └────────────┘   └────────────┘   └────────────┘  │
│         ▲              ▲                  ▲        │
│         └──────────────┼──────────────────┘        │
│                        │                           │
│         FacesServlet（前端控制器）                  │
└────────────────────────────────────────────────────┘
```

## 1.2 发展历程

- JSF 1.0（2004）：JavaEE 5 引入，参考实现是 Mojarra。
- JSF 2.0（2009）：JavaEE 6，引入 Facelets、注解、Ajax 支持。
- JSF 2.2（2013）：JavaEE 7，Pass-through 属性、CDI 集成。
- JSF 2.3（2017）：JavaEE 8，与 CDI 深度融合。
- Jakarta Faces 3.0（2020）：Jakarta EE 9，包名迁移。
- Jakarta Faces 4.0（2022）：Jakarta EE 10，支持 Faces 4、jakarta.faces 命名空间。

## 1.3 参考实现

| 实现 | 说明 |
|------|------|
| Mojarra | Oracle 官方参考实现 |
| MyFaces | Apache 实现 |

# 2. 核心概念

## 2.1 FacesServlet 与 faces-config.xml

JSF 应用唯一的 Servlet 是 `FacesServlet`，负责所有 `.xhtml` 页面的请求处理：

```xml
<servlet>
    <servlet-name>FacesServlet</servlet-name>
    <servlet-class>jakarta.faces.webapp.FacesServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>FacesServlet</servlet-name>
    <url-pattern>*.xhtml</url-pattern>
</servlet-mapping>
```

## 2.2 托管 Bean（Managed Bean）

```java
@Named          // CDI 托管 Bean 注解
@ViewScoped     // 视图作用域：页面存活期间保持状态
public class UserBean {
    private String name;
    private int age;

    @Inject
    private UserService userService;

    public String save() {
        userService.save(this);
        return "success";   // 导航结果
    }
    // getter/setter...
}
```

JSF 3.0 后推荐使用 CDI 注解（`@Named` + 作用域注解），`@ManagedBean` 已废弃。

## 2.3 Facelets 视图模板

JSF 2.0 起使用 **Facelets** 取代 JSP 作为视图技术，文件后缀 `.xhtml`：

```xhtml
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:h="http://xmlns.jcp.org/jsf/html"
      xmlns:f="http://xmlns.jcp.org/jsf/core"
      xmlns:ui="http://xmlns.jcp.org/jsf/facelets">
<h:head>
    <title>用户注册</title>
</h:head>
<h:body>
    <h:form>
        <h:outputLabel value="姓名" for="name"/>
        <h:inputText id="name" value="#{userBean.name}" required="true"/>
        <h:message for="name" style="color:red"/>

        <h:commandButton value="提交" action="#{userBean.save()}"/>
    </h:form>
</h:body>
</html>
```

## 2.4 标签库前缀

| 前缀 | 命名空间 | 用途 |
|------|----------|------|
| `h` | `http://xmlns.jcp.org/jsf/html` | HTML 组件（inputText、commandButton 等） |
| `f` | `http://xmlns.jcp.org/jsf/core` | 核心组件（f:ajax、f:validator 等） |
| `ui` | `http://xmlns.jcp.org/jsf/facelets` | 模板组件（ui:composition、ui:repeat 等） |
| `c` | JSTL core | 迭代/条件（与 f:view 配合） |

# 3. JSF 生命周期

JSF 请求处理包含 6 个阶段，由框架自动执行：

```
①恢复视图(Restore View)
        │
        ▼
②应用请求值(Apply Request Values)
        │
        ▼
③处理校验(Process Validations)
        │ 失败 → 跳转到 ⑥ 渲染响应
        ▼
④更新模型值(Update Model Values)
        │
        ▼
⑤调用应用(Invoke Application)   ← 执行 action 方法
        │
        ▼
⑥渲染响应(Render Response)
```

| 阶段 | 说明 |
|------|------|
| 恢复视图 | 重建组件树（回发时从 ViewState 恢复） |
| 应用请求值 | 把请求参数写入组件（本地值） |
| 处理校验 | 执行 `required`、`f:validator`、Bean Validation |
| 更新模型值 | 把本地值写入托管 Bean 属性 |
| 调用应用 | 调用 `action` / `actionListener` 方法，返回导航结果 |
| 渲染响应 | 生成 HTML 响应 |

## 3.1 状态管理

- JSF 在客户端保存 **ViewState**（默认方案，加密的隐藏字段）或在服务端保存视图状态。
- 状态管理带来的问题：传输体积大、资源消耗高，这也是 JSF 被诟病的原因之一。

# 4. 托管 Bean 作用域

| 作用域 | 注解 | 存活时间 |
|--------|------|----------|
| 请求 | `@RequestScoped` | 一次请求 |
| 视图 | `@ViewScoped` | 一次视图（回发间保持，JSF 特有） |
| 会话 | `@SessionScoped` | 一次会话 |
| 应用 | `@ApplicationScoped` | 整个应用 |
| 自定义 | `@CustomScoped` | 自定义 Map 管理 |

# 5. 校验与转换

## 5.1 内置校验器

```xhtml
<h:inputText value="#{userBean.age}" required="true">
    <f:validateLongRange minimum="18" maximum="65"/>
</h:inputText>

<h:inputText value="#{userBean.email}">
    <f:validator validatorId="jakarta.faces.Email"/>
</h:inputText>
```

## 5.2 自定义校验器

```java
@FacesValidator("passwordValidator")
public class PasswordValidator implements Validator {
    @Override
    public void validate(FacesContext context, UIComponent component, Object value)
            throws ValidatorException {
        String pwd = (String) value;
        if (pwd == null || pwd.length() < 6) {
            throw new ValidatorException(new FacesMessage("密码长度至少6位"));
        }
    }
}
```

# 6. 转换器

```java
@FacesConverter("dateConverter")
public class DateConverter implements Converter {
    private static final DateTimeFormatter FORMATTER =
            DateTimeFormatter.ofPattern("yyyy-MM-dd");

    @Override
    public Object getAsObject(FacesContext ctx, UIComponent comp, String value) {
        return value == null ? null : LocalDate.parse(value, FORMATTER);
    }

    @Override
    public String getAsString(FacesContext ctx, UIComponent comp, Object value) {
        return value == null ? "" : FORMATTER.format((LocalDate) value);
    }
}
```

# 7. Ajax 支持

JSF 通过 `f:ajax` 实现局部刷新，不需要写 JavaScript：

```xhtml
<h:inputText value="#{userBean.city}">
    <f:ajax event="blur" render="msg"/>
</h:inputText>
<h:outputText id="msg" value="城市：#{userBean.city}"/>
```

# 8. JSF 的优缺点

## 8.1 优点

- 组件复用：企业级 UI 组件库丰富（PrimeFaces、ICEFaces、RichFaces）。
- 类型安全：校验、转换、事件模型完善。
- 标准化：与 JavaEE 容器深度集成。

## 8.2 缺点

- **学习曲线陡峭**：生命周期、组件树、状态管理复杂。
- **性能开销大**：ViewState 传输与恢复开销高。
- **前后端耦合**：组件在服务端渲染，不利于前后端分离。
- 生态萎缩：主流新项目基本弃用 JSF。

# 9. JSF vs 主流框架对比

| 对比项 | JSF | Spring MVC | Thymeleaf | 前后端分离 |
|--------|-----|------------|-----------|-----------|
| 模型 | 组件 + 事件 | MVC + 控制器 | 模板引擎 | REST + SPA |
| 渲染位置 | 服务端组件树 | 服务端模板 | 服务端模板 | 客户端 |
| 状态管理 | ViewState 内置 | 手动/无状态 | 无 | 客户端 |
| 前后端解耦 | 差 | 中 | 中 | 优 |
| 现代适用性 | 存量系统 | 主流 | 主流 | 主流 |

> **结论**：JSF 是 JavaEE 历史上重要的 Web 框架，但现代开发中已被 Spring MVC/前后端分离取代，仅需理解其组件模型与生命周期作为知识储备。
