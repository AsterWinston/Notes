# 1. 国际化概述
```
国际化（i18n，Internationalization的缩写，首尾字母i和n中间有18个字母）是指让应用程序支持多种语言和地区。

SpringMVC提供了完善的国际化支持，主要包括以下组件：
● MessageSource：管理国际化消息资源（如文本、错误信息等）
● LocaleResolver：解析用户的区域信息（语言、国家）
● LocaleChangeInterceptor：允许用户动态切换语言

国际化的核心流程是：
1. 解析用户的语言偏好（通过浏览器Accept-Language、URL参数、Cookie等）
2. 根据语言偏好从MessageSource中获取对应语言的消息
3. 在视图中展示对应语言的内容
```

# 2. 配置MessageSource（消息资源）
```
2.1 创建消息属性文件：
在类根路径（resources目录）下创建以下文件：

messages.properties（默认语言，通常是中文）：
greeting=你好
welcome=欢迎使用SpringMVC
user.title=用户管理
user.add=添加用户
user.delete=删除用户
user.edit=修改用户
btn.submit=提交
btn.cancel=取消
error.notfound=资源不存在
error.server=服务器内部错误

messages_en_US.properties（英文）：
greeting=Hello
welcome=Welcome to SpringMVC
user.title=User Management
user.add=Add User
user.delete=Delete User
user.edit=Edit User
btn.submit=Submit
btn.cancel=Cancel
error.notfound=Resource Not Found
error.server=Internal Server Error

messages_ja_JP.properties（日文）：
greeting=こんにちは
welcome=SpringMVCへようこそ
user.title=ユーザー管理
user.add=ユーザー追加
user.delete=ユーザー削除
user.edit=ユーザー編集
btn.submit=送信
btn.cancel=キャンセル
error.notfound=リソースが見つかりません
error.server=サーバー内部エラー

注意：
● 默认文件(messages.properties)是当没有匹配的语言包时的兜底文件
● 文件名格式：basename_language_country.properties
  例如：messages_zh_CN.properties（中文-中国）、messages_en_US.properties（英语-美国）
● 如果只有语言没有国家，如：messages_en.properties
```

2.2 XML配置MessageSource（springmvc.xml）：
<bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
    <!-- 设置消息文件的基础名称（不含后缀和语言代码） -->
    <property name="basename" value="messages"/>
    <!-- 设置默认编码 -->
    <property name="defaultEncoding" value="UTF-8"/>
    <!-- 如果没有找到对应语言的消息，是否使用默认消息 -->
    <property name="fallbackToSystemLocale" value="false"/>
    <!-- 设置缓存时间（秒），开发时可以设为0关闭缓存 -->
    <property name="cacheSeconds" value="60"/>
</bean>

注意：Bean的id必须命名为 "messageSource"，SpringMVC会按名称查找。

2.3 注解配置MessageSource（SpringMvcConfig中）：
@Bean
public MessageSource messageSource() {
    ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
    messageSource.setBasename("messages");            // 消息文件的基础名称
    messageSource.setDefaultEncoding("UTF-8");        // 编码
    messageSource.setCacheSeconds(60);                // 缓存时间
    messageSource.setFallbackToSystemLocale(false);   // 是否回退到系统区域
    return messageSource;
}

# 3. 配置LocaleResolver（区域解析器）
LocaleResolver负责解析用户的区域信息（语言、国家）。
SpringMVC提供了多种LocaleResolver实现：

3.1 AcceptHeaderLocaleResolver（默认）：
根据请求头 Accept-Language 来确定用户的语言偏好。
这是SpringMVC默认使用的LocaleResolver，无需额外配置。
当浏览器语言设置为英文时，会自动使用英文消息。

3.2 SessionLocaleResolver（推荐）：
将用户选择的语言保存在Session中。用户切换语言后，整个会话期间保持一致。

配置SessionLocaleResolver：
<bean id="localeResolver" class="org.springframework.web.servlet.i18n.SessionLocaleResolver">
    <!-- 设置默认语言 -->
    <property name="defaultLocale" value="zh_CN"/>
</bean>

注解配置：
@Bean
public LocaleResolver localeResolver() {
    SessionLocaleResolver resolver = new SessionLocaleResolver();
    resolver.setDefaultLocale(Locale.SIMPLIFIED_CHINESE);  // 默认中文
    return resolver;
}

注意：Bean的id必须命名为 "localeResolver"，SpringMVC会按名称查找。

3.3 CookieLocaleResolver：
将用户选择的语言保存在Cookie中，即使关闭浏览器后重新打开也能保持。

配置CookieLocaleResolver：
<bean id="localeResolver" class="org.springframework.web.servlet.i18n.CookieLocaleResolver">
    <property name="defaultLocale" value="zh_CN"/>
    <property name="cookieName" value="language"/>
    <property name="cookieMaxAge" value="604800"/> <!-- 7天 -->
</bean>

注解配置：
@Bean
public LocaleResolver localeResolver() {
    CookieLocaleResolver resolver = new CookieLocaleResolver();
    resolver.setDefaultLocale(Locale.SIMPLIFIED_CHINESE);
    resolver.setCookieName("language");
    resolver.setCookieMaxAge(7 * 24 * 60 * 60);  // 7天
    return resolver;
}

# 4. 配置LocaleChangeInterceptor（语言切换拦截器）
LocaleChangeInterceptor允许用户通过URL参数动态切换语言。

4.1 配置拦截器（XML方式）：
<!-- 配置LocaleChangeInterceptor -->
<bean id="localeChangeInterceptor" class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor">
    <property name="paramName" value="lang"/>
</bean>

<!-- 注册拦截器 -->
<mvc:interceptors>
    <ref bean="localeChangeInterceptor"/>
</mvc:interceptors>

4.2 配置拦截器（注解方式）：
在SpringMvcConfig中重写addInterceptors方法：
@Override
public void addInterceptors(InterceptorRegistry registry) {
    LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
    interceptor.setParamName("lang");  // 通过lang参数切换语言
    registry.addInterceptor(interceptor);
}

4.3 使用方式：
用户通过URL参数 lang 切换语言：
http://localhost:8080/springmvc/index?lang=zh_CN  （切换到中文）
http://localhost:8080/springmvc/index?lang=en_US  （切换到英文）
http://localhost:8080/springmvc/index?lang=ja_JP  （切换到日文）

切换后，后续的所有请求都会使用新语言（SessionLocaleResolver或CookieLocaleResolver会记录）。

# 5. 在Controller中使用国际化消息
5.1 注入MessageSource手动获取消息：

@Controller
public class IndexController {

    @Autowired
    private MessageSource messageSource;

    @RequestMapping("/index")
    public String index(HttpServletRequest request, Model model) {
        // 获取当前用户的Locale
        Locale locale = RequestContextUtils.getLocale(request);

        // 根据key获取国际化消息
        String greeting = messageSource.getMessage("greeting", null, locale);
        String welcome = messageSource.getMessage("welcome", null, locale);

        model.addAttribute("greeting", greeting);
        model.addAttribute("welcome", welcome);
        return "index";
    }
}

5.2 带占位符的消息：
messages.properties：
user.welcome=欢迎您，{0}！今天是{1}。

Controller中获取：
String msg = messageSource.getMessage("user.welcome", new Object[]{"张三", "星期一"}, locale);
// 输出：欢迎您，张三！今天是星期一。

# 6. 在Thymeleaf视图中使用国际化消息
SpringMVC集成的Thymeleaf支持国际化消息表达式 #{key}：

<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title th:text="#{user.title}">用户管理</title>
</head>
<body>
    <!-- 使用 #{key} 获取国际化文本 -->
    <h1 th:text="#{greeting}">你好</h1>
    <p th:text="#{welcome}">欢迎使用SpringMVC</p>

    <div>
        <button th:text="#{user.add}">添加用户</button>
        <button th:text="#{user.delete}">删除用户</button>
        <button th:text="#{user.edit}">修改用户</button>
    </div>

    <div>
        <a th:href="@{/index(lang=zh_CN)}">中文</a> |
        <a th:href="@{/index(lang=en_US)}">English</a> |
        <a th:href="@{/index(lang=ja_JP)}">日本語</a>
    </div>
</body>
</html>

注意：
● #{key} 会自动根据当前Locale获取对应的语言消息
● @{/index(lang=zh_CN)} 是Thymeleaf生成带参数的URL

# 7. 国际化综合示例
完整配置流程总结（注解方式）：

第一步：创建消息文件
resources/
├── messages.properties        （默认中文）
├── messages_en_US.properties  （英文）
└── messages_ja_JP.properties  （日文）

第二步：SpringMvcConfig中配置
@Configuration
@ComponentScan("com.powernode.springmvc.controller")
@EnableWebMvc
public class SpringMvcConfig implements WebMvcConfigurer {

    // MessageSource Bean（id必须是messageSource）
    @Bean
    public MessageSource messageSource() {
        ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
        messageSource.setBasename("messages");
        messageSource.setDefaultEncoding("UTF-8");
        messageSource.setCacheSeconds(0);  // 开发时关闭缓存
        return messageSource;
    }

    // LocaleResolver Bean（id必须是localeResolver）
    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver resolver = new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.SIMPLIFIED_CHINESE);
        return resolver;
    }

    // 注册国际化切换拦截器
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
        interceptor.setParamName("lang");
        registry.addInterceptor(interceptor);
    }

    // ... 其他配置（视图解析器、静态资源等）
}

第三步：在Controller或Thymeleaf中使用
Controller层：
@Autowired
private MessageSource messageSource;

Thymeleaf视图层：
<h1 th:text="#{greeting}">默认文字</h1>

第四步：测试
访问 http://localhost:8080/springmvc/index?lang=zh_CN → 中文界面
访问 http://localhost:8080/springmvc/index?lang=en_US → 英文界面
访问 http://localhost:8080/springmvc/index?lang=ja_JP → 日文界面

常见问题：
1. 中文乱码：确保消息文件是UTF-8编码，并配置 defaultEncoding="UTF-8"
2. 修改消息文件不生效：开发阶段将cacheSeconds设为0
3. LocaleResolver不生效：确保Bean的id是 "localeResolver"
4. Thymeleaf无法显示 #{key}：检查是否配置了MessageSource Bean
