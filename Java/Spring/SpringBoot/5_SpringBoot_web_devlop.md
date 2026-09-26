# 1.SpringBoot的web自动配置

创建模块，添加web启动器，添加Lombok依赖。

## 1.1web自动配置的依赖是如何传递的
1. 首先引入了web启动器，如下：

```
<dependency>
  <groupId>org.springframework.boot</groupId>

  <artifactId>spring-boot-starter-web</artifactId>

</dependency>

```
2. web启动器传递引入了spring-boot-starter，如下：

```
<dependency>
  <groupId>org.springframework.boot</groupId>

  <artifactId>spring-boot-starter</artifactId>

  <version>3.3.5</version>

  <scope>compile</scope>

</dependency>

```
3. spring-boot-starter会传递引入一个spring-boot-autoconfigure包，如下：

```
<dependency>
  <groupId>org.springframework.boot</groupId>

  <artifactId>spring-boot-autoconfigure</artifactId>

  <version>3.3.5</version>

  <scope>compile</scope>

</dependency>

```
<img src="images/63.png" width="100%">

在spring-boot-autoconfigure包中的.imports文件中罗列的需要导入的自动配置类，如下图：
<img src="images/64.png" width="100%">

## 1.2web自动配置的实现原理
1. 从入口程序开始：

```
package com.powernode.sb309web;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Sb309WebApplication {

    public static void main(String[] args) {
        SpringApplication.run(Sb309WebApplication.class, args);
    }
}
```
入口程序被@SpringBootApplication注解标注。
2. @SpringBootApplication注解被@EnableAutoConfiguration注解标注。表示启用自动配置。
3. @EnableAutoConfiguration注解被@Import({AutoConfigurationImportSelector.class})注解标注。
4. 因此AutoConfigurationImportSelector决定哪些自动配置类是需要导入的。
5. 底层实现步骤具体如下：
<img src="images/65.png" width="100%">
<img src="images/66.png" width="100%">
最终找的文件是：META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
注意：任何jar包，包括第三方的依赖，自动配置类所在的路径以及文件名都是完全相同的，都是**META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports**
例如mybatis的自动配置类的列表文件也是这样，如下图：
<img src="images/67.png" width="100%">

## 1.3通过web自动配置类逆推web配置的prefix
在自动配置列表中找到web自动配置相关的类：
<img src="images/68.png" width="100%">

以下就是web自动配置类列表：
```
org.springframework.boot.autoconfigure.web.client.RestTemplateAutoConfiguration
org.springframework.boot.autoconfigure.web.embedded.EmbeddedWebServerFactoryCustomizerAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```
通过web自动配置类的源码可以逆推web配置的prefix：

1. WebMvcAutoConfiguration
   <img src="images/69.png" width="100%">
   <img src="images/70.png" width="100%">
   <img src="images/71.png" width="100%">
2. MultipartAutoConfiguration
<img src="images/72.png" width="100%">
<img src="images/73.png" width="100%">
3. HttpEncodingAutoConfiguration
<img src="images/74.png" width="100%">
<img src="images/75.png" width="100%">
4. ErrorMvcAutoConfiguration
5. ServletWebServerFactoryAutoConfiguration
6. DispatcherServletAutoConfiguration
7. EmbeddedWebServerFactoryCustomizerAutoConfiguration
8. RestTemplateAutoConfiguration
通过查看源码，得知，web开发时，在application.properties配置文件中可以配置的前缀是：
```
# SpringMVC相关配置
spring.mvc.

# web开发通用配置
spring.web.

# 文件上传配置
spring.servlet.multipart.

# 服务器配置
server.
```
## 1.4web自动配置都默认配置了什么
查看官方文档：
<img src="images/76.png" width="100%">
翻译如下：
Spring Boot 为 Spring MVC 提供了自动配置，这在大多数应用程序中都能很好地工作。除了已经实现了 Spring MVC 的默认功能外，自动配置还提供了以下特性：
● 包括 ContentNegotiatingViewResolver 和 BeanNameViewResolver 的 Bean。
  ○ ContentNegotiatingViewResolver 自动根据HTTP请求头中Accept字段来选择合适的视图技术渲染响应。
  ○ <font style="color:rgb(44, 44, 54);">BeanNameViewResolver</font> 的作用是根据视图名称找到视图View对象。
● 支持提供静态资源，包括对 WebJars的支持。
  ○ 静态资源路径默认已经配置好了。默认会去static目录下找。
● 自动注册 Converter、GenericConverter 和 Formatter 的 Bean。
  ○ Converter：转换器，做类型转换的，例如表单提交了用户数据，将表单数据转换成User对象。
  ○ Formatter：格式化器，做数据格式化的，例如将Java中的日期类型对象格式化为特定格式的日期字符串。或者将用户提交的日期字符串，转换为Java中的日期对象。
● 支持 HttpMessageConverters。
  ○ 内置了很多的HTTP消息转换器。例如：MappingJackson2HttpMessageConverter可以将json转换成java对象，也可以将java对象转换为json字符串。
● 自动注册 MessageCodesResolver。
  ○ SpringBoot会自动注册一个默认的消息代码解析器
  ○ 帮助你在表单验证出错时生成一些特殊的代码。这些代码让你能够更精确地定位问题，并提供更友好的错误提示。
● 静态 index.html 文件支持。
  ○ Spring Boot 会自动处理位于项目静态资源目录下的 index.html 文件，使其成为应用程序的默认主页
● 自动使用 ConfigurableWebBindingInitializer Bean。
  ○ 用它来指定默认使用哪个转换器，默认使用哪个格式化器。在这个类当中都已经配好了。
如果您不想使用自动配置并希望完全控制 Spring MVC，可以添加您自己的带有 **<font style="color:#DF2A3F;">@EnableWebMvc</font>** 注解的 @Configuration。
如果您希望保留这些 Spring Boot MVC 定制化设置并进行更多的 MVC 定制化（如拦截器、格式化程序、视图控制器等其他功能），可以添加您自己的类型为 WebMvcConfigurer 的 @Configuration 类。但不能使用**<font style="color:#DF2A3F;">@EnableWebMvc</font>**注解。

## 1.5WebMvcAutoConfiguration原理
通过源码分析的方式，学习WebMvc的自动配置原理。
### 1.5.1WebMvc自动配置是否生效的条件
```
@AutoConfiguration(after = { DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
        ValidationAutoConfiguration.class })
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@ImportRuntimeHints(WebResourcesRuntimeHints.class)
public class WebMvcAutoConfiguration {}
```
● @AutoConfiguration(after = { DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,ValidationAutoConfiguration.class })
  ○ WebMvcAutoConfiguration自动配置类加载顺序在以上自动配置类加载后加载。
● @ConditionalOnWebApplication(type = Type.SERVLET)
  ○ WebMvcAutoConfiguration自动配置类只在servlet环境中生效。
● @ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
  ○ 类路径中必须存在Servlet.class``DispatcherServlet.class``WebMvcConfigurer.class，WebMvcAutoConfiguration自动配置类才会生效。
● @ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
  ○ 类路径中不存在WebMvcConfigurationSupport.class时WebMvcAutoConfiguration自动配置类才会生效。
  ○ 注意：当使用@EnableWebMvc注解后，类路径中就会注册一个WebMvcConfigurationSupport这样的bean。
● @AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10) 不重要
  ○ 指定WebMvcAutoConfiguration自动配置类的加载顺序
● @ImportRuntimeHints(WebResourcesRuntimeHints.class) 不重要
  ○ 运行时引入WebResourcesRuntimeHints这个类，这个类的作用是给JVM或者其他组件提示信息的，提示一下系统应该如何处理类和资源。

总结来说，WebMvcAutoConfiguration类将在以下条件下生效：
1. 应用程序是一个Servlet类型的Web应用；
2. 环境中有Servlet、DispatcherServlet和WebMvcConfigurer类；
3. 容器中没有WebMvcConfigurationSupport的bean。
如果这些条件都满足的话，那么这个自动配置类就会被激活，并进行相应的自动配置工作。

### 1.5.2WebMvc自动配置生效后引入了两个Filter Bean
1. 引入了HiddenHttpMethodFilter Bean
```
@Bean
@ConditionalOnMissingBean(HiddenHttpMethodFilter.class)
@ConditionalOnProperty(prefix = "spring.mvc.hiddenmethod.filter", name = "enabled")
public OrderedHiddenHttpMethodFilter hiddenHttpMethodFilter() {
    return new OrderedHiddenHttpMethodFilter();
}
```
这个过滤器是专门处理Rest请求的。GET POST PUT DELETE请求。
2. 引入了FormContentFilter Bean
```
@Bean
@ConditionalOnMissingBean(FormContentFilter.class)
@ConditionalOnProperty(prefix = "spring.mvc.formcontent.filter", name = "enabled", matchIfMissing = true)
public OrderedFormContentFilter formContentFilter() {
    return new OrderedFormContentFilter();
}
```
OrderedFormContentFilter 是 Spring Boot 中用于处理 HTTP 请求的一个过滤器，特别是针对 PUT 和 DELETE 请求。这个过滤器的主要作用是在处理 PUT 和 DELETE 请求时，确保如果请求体中有表单格式的数据，这些数据会被正确解析并可用。

### 1.5.3WebMvc自动配置生效后引入了WebMvcConfigurer接口的实现类
在SpringBoot框架的WebMvcAutoConfiguration类中提供了一个内部类：WebMvcAutoConfigurationAdapter
<img src="images/77.png" width="100%">
SpringBoot在这个类WebMvcAutoConfigurationAdapter中进行了一系列的Spring MVC相关配置。
我们开发中要对Spring MVC的相关配置进行修改，可以编写一个类继承**<font style="color:#DF2A3F;">WebMvcAutoConfigurationAdatper</font>**，然后重写对应的方法即可。
因此，通过对**<font style="color:#DF2A3F;">WebMvcAutoConfigurationAdapter</font>**类中的方法进行重写来修改Web MVC的默认配置。


关于WebMvcConfigurer接口
这个接口不是SpringBoot框架提供的，是Spring MVC提供的，在Spring框架4.3版本中引入的。这个接口的作用主要是允许开发者通过实现这个接口来定制Spring MVC的行为。
在这个接口中提供了很多方法，需要改变Spring MVC的哪个行为，则重写对应的方法即可，下面是这个接口中所有的方法，以及每个方法对应的Spring MVC行为的解释：
```
public interface WebMvcConfigurer {
    // 用于定制 Spring MVC 如何匹配请求路径到控制器
    default void configurePathMatch(PathMatchConfigurer configurer) {}
    // 用于定制 Spring MVC 的内容协商策略，以确定如何根据请求的内容类型来选择合适的处理方法或返回数据格式
    default void configureContentNegotiation(ContentNegotiationConfigurer configurer) {}
    // 用于定制 Spring MVC 处理异步请求的方式
    default void configureAsyncSupport(AsyncSupportConfigurer configurer) {}
    // 用于定制是否将某些静态资源请求转发WEB容器默认的Servlet处理
    default void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {}
    // 用于定制 Spring MVC 解析视图的方式，以确定如何将控制器返回的视图名称转换为实际的视图资源。
    default void configureViewResolvers(ViewResolverRegistry registry) {}
    // 用于定制 Spring MVC 如何处理 HTTP 请求和响应的数据格式，包括 JSON、XML 等内容类型的转换
    default void configureMessageConverters(List<HttpMessageConverter<?>> converters) {}
    // 用于定制 Spring MVC 如何处理控制器方法中发生的异常，并提供相应的错误处理逻辑。
    default void configureHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {}

    // 用于定制 Spring MVC 如何处理数据的格式化和解析，例如日期、数值等类型的对象的输入和输出格式。
    default void addFormatters(FormatterRegistry registry) {}
    // 用于定制 Spring MVC 如何使用拦截器来处理请求和响应，包括在请求进入控制器之前和之后执行特定的操作。
    default void addInterceptors(InterceptorRegistry registry) {}
    // 用于定制 Spring MVC 如何处理静态资源（如 CSS、JavaScript、图片等文件）的请求。
    default void addResourceHandlers(ResourceHandlerRegistry registry) {}
    // 用于定制 Spring MVC 如何处理跨域请求，确保应用程序可以正确地响应来自不同域名的 AJAX 请求或其他跨域请求。
    default void addCorsMappings(CorsRegistry registry) {}
    // 用于快速定义简单的 URL 到视图的映射，而无需编写完整的控制器类和方法。
    default void addViewControllers(ViewControllerRegistry registry) {}
    // 用于定制 Spring MVC 如何解析控制器方法中的参数，包括如何从请求中获取并转换参数值。
    default void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {}
    // 用于定制 Spring MVC 如何处理控制器方法的返回值，包括如何将返回值转换为实际的 HTTP 响应。
    default void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> handlers) {}

    // 用于定制 Spring MVC 如何处理 HTTP 请求和响应的数据格式，允许你添加或调整默认的消息转换器，以支持特定的数据格式。
    default void extendMessageConverters(List<HttpMessageConverter<?>> converters) {}
    // 用于定制 Spring MVC 如何处理控制器方法中抛出的异常，允许你添加额外的异常处理逻辑。
    default void extendHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {}
}
```
WebMvcConfigurer接口的实现类WebMvcAutoConfigurationAdapter
WebMvcAutoConfigurationAdapter是Spring Boot框架提供的，实现了Spring MVC中的WebMvcConfigurer接口，对Spring MVC的所有行为进行了默认的配置。
如果想要改变这些默认配置，应该怎么办呢？看源码：
<img src="images/78.png" width="100%">
可以看到，该类上有一个注解@EnableConfigurationProperties({ WebMvcProperties.class, WebProperties.class })，该注解负责启用配置属性。会将配置文件application.properties或application.yml中的配置传递到该类中。因此可以通过application.properties或application.yml配置文件来改变Spring Boot对SpringMVC的默认配置。WebMvcProperties和WebProperties源码如下： 
<img src="images/79.png" width="100%">
<img src="images/80.png" width="100%">
通过以上源码得知要改变SpringBoot对SpringMVC的默认配置，需要在配置文件中使用以下前缀的配置：
● spring.mvc：主要用于配置 Spring MVC 的相关行为，例如路径匹配、视图解析、静态资源处理等
● spring.web：通常用于配置一些通用的 Web 层设置，如资源处理、安全性配置等。

## 1.6自动配置中的静态资源处理
web站点中的静态资源指的是：js、css、图片等。

### 1.6.1静态资源处理源码分析
关于SpringBoot对静态资源处理的默认配置，查看WebMvcAutoConfigurationAdapter源码，核心源码如下：
<img src="images/81.png" width="100%">
对以上源码进行解释：
```
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {

    // 检查 resourceProperties 中的 addMappings 属性是否为 false。如果为 false，则表示不启用默认的静态资源映射处理。
    // 在application.properties配置文件中进行`spring.web.resources.add-mappings=false`配置，可以将其设置为false。
    // 当然，如果没有配置的话，默认值是true。
    if (!this.resourceProperties.isAddMappings()) {
        logger.debug("Default resource handling disabled");
        return;
    }

    // 配置 WebJars 的静态资源处理。
    // this.mvcProperties.getWebjarsPathPattern()的执行结果是：/webjars/**
    // 也就是说，如果请求路径是 http://localhost:8080/webjars/** ，则自动去类路径下的 /META-INF/resources/webjars/ 目录中找静态资源。
    // 如果要改变这个默认的配置，需要在application.properties文件中进行这样的配置：`spring.mvc.webjars-path-pattern=...`
    addResourceHandler(registry, this.mvcProperties.getWebjarsPathPattern(),
            "classpath:/META-INF/resources/webjars/");

    // 配置普通静态资源处理
    // this.mvcProperties.getStaticPathPattern()的执行结果是：/**
    // this.resourceProperties.getStaticLocations()的执行结果是：{ "classpath:/META-INF/resources/","classpath:/resources/", "classpath:/static/", "classpath:/public/" }
    // 也就是说，如果请求路径是：http://localhost:8080/**，根据控制器方法优先原则，会先去找合适的控制器方法，如果没有合适的控制器方法，静态资源处理才会生效，则自动去类路径下的/META-INF/resources/、/resources/、/static/、/public/ 4个位置找。
    // 如果要改变这个默认的配置，需要在application.properties中进行如下的两个配置：
    // 配置URL：spring.mvc.static-path-pattern=...
    // 配置物理路径：spring.web.resources.static-locations=...,...,...,...
    addResourceHandler(registry, this.mvcProperties.getStaticPathPattern(), (registration) -> {
        registration.addResourceLocations(this.resourceProperties.getStaticLocations());
        if (this.servletContext != null) {
            ServletContextResource resource = new ServletContextResource(this.servletContext, SERVLET_LOCATION);
            registration.addResourceLocations(resource);
        }
    });
}
```
### 1.6.2关于WebJars静态资源处理
默认规则是：当请求路径是/webjars/**，则会去classpath:/META-INF/resources/webjars/找。

WebJars介绍
WebJars 是一种将常用的前端库（如 jQuery、Bootstrap、Font Awesome 等）打包成 JAR 文件的形式，方便在 Java 应用程序中使用。WebJars 提供了一种标准化的方式来管理前端库，使其更容易集成到 Java 项目中，并且可以利用 Maven 的依赖管理功能。

WebJars在SpringBoot中的使用
WebJars官网：https://www.webjars.org/
<img src="images/82.png" width="100%">

在官网上可以找到某个webjars的maven依赖，将依赖加入到SpringBoot项目中，例如我们添加vue的依赖：
```
<dependency>
    <groupId>org.webjars.npm</groupId>

    <artifactId>vue</artifactId>

    <version>3.5.12</version>

</dependency>

```
如下图表示加入成功：
<img src="images/83.png" width="100%">
在jar包列表中也可以看到：
<img src="images/84.png" width="100%">

在SpringBoot中，对WebJars的默认访问规则是：当请求路径是/webjars/**，则会去classpath:/META-INF/resources/webjars/找。
因此我们要想访问上图的index.js，则应该发送这样的请求路径：http://localhost:8080/webjars/vue/3.5.12/index.js

启动服务器，打开浏览器，访问
### 1.6.3关于普通静态资源处理
SpringBoot对普通静态资源处理的规则是：
当请求路径是http://localhost:8080/**，根据控制器方法优先原则，会先去找合适的控制器方法，如果没有合适的控制器方法，静态资源处理才会生效，则自动去类路径下的以下4个位置查找：
● classpath:/META-INF/resources/
● classpath:/resources/
● classpath:/static/
● classpath:/public/

### 1.6.4关于静态资源缓存处理
不管是webjars的静态资源还是普通静态资源，统一都会执行以下这个方法，这个方法最后几行代码就是关于静态资源的缓存处理方式。
<img src="images/85.png" width="100%">
什么是静态资源缓存，谁缓存，有什么用？
静态资源缓存指的是浏览器的缓存行为，浏览器可以将静态资源（js、css、图片、声音、视频）缓存到浏览器中，只要下一次用户访问同样的静态资源直接从缓存中取，不再从服务器中获取，可以降低服务器的压力，提高用户的体验。而这个缓存策略可以在服务器端程序中进行设置，SpringBoot对静态资源缓存的默认策略就是以下这三行代码：
<img src="images/86.png" width="100%">
以上三行代码的解释如下：
● registration.setCachePeriod(getSeconds(this.resourceProperties.getCache().getPeriod()));
  ○ 设置缓存的过期时间（如果没有指定单位，默认单位是秒）
  ○ 浏览器会根据响应头中的缓存控制信息决定是否从本地缓存中加载资源，而不是每次都从服务器重新请求。这有助于减少网络流量和提高页面加载速度。
  ○ 假设你配置了静态资源缓存过期时间为 1 小时（3600 秒），那么浏览器在首次请求某个静态资源后，会在接下来的一小时内从本地缓存加载该资源，而不是重新请求服务器。
  ○ 可以通过application.properties的来修改默认的过期时间，例如：<font style="color:rgb(51, 51, 51);">spring.web.resources.cache.period=3600</font>或者<font style="color:rgb(51, 51, 51);">spring.web.resources.cache.period=1h</font>
● registration.setCacheControl(this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl());
  ○ 设置静态资源的 Cache-Control HTTP 响应头，告诉浏览器如何去缓存这些资源。
  ○ <font style="color:rgb(44, 44, 54);">Cache-Control</font> HTTP 响应头   是HTTP响应协议的一部分内容。如下图响应协议的响应头信息中即可看到<font style="color:rgb(44, 44, 54);">Cache-Control</font>的字样：
<img src="images/87.png" width="100%">

```
- <font style="color:rgb(44, 44, 54);">常见的 Cache-Control 指令包括：</font>

    * <font style="color:rgb(44, 44, 54);">max-age=<seconds>：表示响应在多少秒内有效。</font>

    * <font style="color:rgb(44, 44, 54);">public：表示响应可以被任何缓存机制（如代理服务器）缓存。</font>

    * <font style="color:rgb(44, 44, 54);">private：表示响应只能被用户的浏览器缓存。</font>

    * <font style="color:rgb(44, 44, 54);">no-cache：表示在使用缓存的资源之前必须重新发送一次请求进行验证。</font>

    * <font style="color:rgb(44, 44, 54);">no-store：表示不缓存任何响应的资源。</font>

- 例如：max-age=3600, public：表示响应在 3600 秒内有效，并且可以被任何缓存机制缓存。
- 可以通过`spring.web.resources.cache.cachecontrol.max-age=3600`以及`spring.web.resources.cache.cachecontrol.cache-public=true`进行重新配置。
```
● registration.setUseLastModified(this.resourceProperties.getCache().isUseLastModified());
  ○ 设置静态资源在响应时，是否在响应头中添加资源的最后一次修改时间。SpringBoot默认配置的是：在响应头中添加响应资源的最后一次修改时间。
  ○ 浏览器发送请求时，会将缓存中的资源的最后修改时间和服务器端资源的最后一次修改时间进行比对，如果没有变化，仍然从缓存中获取。
  ○ 可以通过spring.web.resources.cache.use-last-modified=false来进行重新配置。

### 1.6.5静态资源缓存测试
根据之前源码分析，得知静态资源缓存相关的配置应该使用spring.web.resources.cache：
<img src="images/88.png" width="100%">

<img src="images/89.png" width="100%">

<img src="images/90.png" width="100%">

<img src="images/91.png" width="100%">

在application.properties文件中对缓存进行如下的配置：
```
# 静态资源缓存设置
# 1. 缓存有效期
spring.web.resources.cache.period=100
# 2. 缓存控制（cachecontrol配置的话，period会失效）
spring.web.resources.cache.cachecontrol.max-age=20
# 3. 是否使用缓存的最后修改时间（默认是：使用）
spring.web.resources.cache.use-last-modified=true
# 4. 是否开启静态资源默认处理方式（默认是：开启）
spring.web.resources.add-mappings=true
```
注意：cachecontrol.max-age配置的话，period会被覆盖。

<img src="images/92.png" width="100%">

启动服务器测试：看看是否在20秒内走缓存，20秒之后是不是就不走缓存了！！！
第一次访问：请求服务器
<img src="images/93.png" width="100%">
第二次访问：20秒内开启一个新的浏览器窗口，再次访问，发现走了缓存
<img src="images/94.png" width="100%">
第三次访问：20秒后开启一个新的浏览器窗口，再次访问，发现重新请求服务器
<img src="images/95.png" width="100%">
提示，为什么显示304，这是因为这个配置：spring.web.resources.cache.use-last-modified=true

### 1.6.6web应用的欢迎页面
欢迎页测试
先说结论：只要在静态资源路径下提供index.html，则被当做欢迎页面。静态资源路径指的是之前的4个路径：
```
{ "classpath:/META-INF/resources/", "classpath:/resources/", "classpath:/static/", "classpath:/public/" }
```
测试一下，在classpath:/static/目录下新建index.html页面
...正常访问
如果同时在4个静态资源路径下都提供index.html，哪个页面会被当做欢迎页呢？

优先走classpath:/META-INF/resources/
原因是什么呢？这是因为classpath:/META-INF/resources/是数组的首元素，因此先从这个路径下找欢迎页。
<img src="images/96.png" width="100%">

欢迎页源码分析
在WebMvcAutoConfiguration类中有一个内部类EnableWebMvcConfiguration，这个类中有这样一段代码：
<img src="images/97.png" width="100%">

<img src="images/98.png" width="100%">

<img src="images/99.png" width="100%">

<img src="images/100.png" width="100%">
通过以上源码追踪，得出结论：只要请求路径是/**的，会依次去{ "classpath:/META-INF/resources/", "classpath:/resources/", "classpath:/static/", "classpath:/public/" }这四个位置找index.html页面作为欢迎页。

一个小小的疑惑
我们来看一下WebMvcAutoConfiguration的生效条件：
<img src="images/101.png" width="100%">
上图红框内表示，要求Spring容器中缺失WebMvcConfigurationSupport这个Bean，WebMvcAutoConfiguration才会生效。

但是我们来看一下EnableWebMvcConfiguration的继承结构：
<img src="images/102.png" width="100%">
很明显，EnableWebMvcConfiguration就是一个WebMvcConfigurationSupport这样的Bean。
那疑问就有了：既然容器中存在WebMvcConfigurationSupport这样的Bean，WebMvcAutoConfiguration为什么还会生效呢？
原因是因为：EnableWebMvcConfiguration是WebMvcAutoConfiguration类的内部类。在WebMvcAutoConfiguration进行加载的时候，EnableWebMvcConfiguration这个内部类还没有加载。因此这个时候在容器中还不存在WebMvcConfigurationSupport的Bean，所以WebMvcAutoConfiguration仍然会生效。

以上所说的**<font style="color:#DF2A3F;">WebMvcAutoConfiguration</font>**类中的内部类**<font style="color:#DF2A3F;">EnableWebMvcConfiguration</font>**，是用来启用Web MVC默认配置的。

注意区分：WebMvcAutoConfiguration的两个内部类：
● WebMvcAutoConfigurationAdapter作用是用来：修改配置的
● EnableWebMvcConfiguration作用是用来：启用配置的

favorite icon
favicon（也称为“收藏夹图标”或“网站图标”）是大多数现代网页浏览器的默认行为之一。当用户访问一个网站时，浏览器通常会尝试从该网站的根目录下载名为 favicon.ico 的文件，并将其用作标签页的图标。
如果网站没有提供 favicon.ico 文件，浏览器可能会显示一个默认图标，或者根本不显示任何图标。为了确保良好的用户体验，网站开发者通常会在网站的根目录下放置一个 favicon.ico 文件。

Spring Boot项目中favicon.ico文件应该放在哪里呢？Spring Boot官方是这样说明的：
<img src="images/103.png" width="100%">
这段话翻译为：
与其他静态资源一样，Spring Boot 会在配置的静态内容位置检查是否存在 favicon.ico文件。如果存在这样的文件，它将自动作为应用程序的 favicon 使用。

以上官方说明的：将favicon.ico文件放到静态资源路径下即可。

我们在https://www.iconfont.cn/ （阿里巴巴提供的图标库）上随便找一个图标，然后将图片名字命名为favicon.ico，然后将其放到SpringBoot项目的静态资源路径下
...
测试成功显示图标
# 2.SpringBoot的web手动配置（静态资源处理）
如果你对SpringBoot默认的静态资源处理方式不满意。可以通过两种方式来改变这些默认的配置：
● 第一种：配置文件方式
  ○ 通过修改application.properties或application.yml
  ○ 添加spring.mvc和spring.web相关的配置。
<img src="images/104.png" width="100%">
<img src="images/105.png" width="100%">
<img src="images/106.png" width="100%">
● 第二种：编写代码方式
  ○ SpringMVC框架为我们提供了WebMvcConfigurer接口，需要改变默认的行为，可以编写一个类实现WebMvcConfigurer接口，并对应重写接口中的方法即可改变默认的配置行为。

## 2.1配置文件方式
要修改访问静态资源URL的前缀，这样配置：
```
# Spring MVC的相关配置
# 1. 设置webjars静态资源的请求路径的前缀
spring.mvc.webjars-path-pattern=/wjs/**
# 2. 设置普通静态资源的请求路径的前缀
spring.mvc.static-path-pattern=/static/**
```
要修改静态资源的存放位置，这样配置：
```
spring.web.resources.static-locations=classpath:/static1/,classpath:/static2/
```
进行以上配置之后：
1. 访问webjars的请求路径应该是这样的：http://localhost:8080/wjs/....
2. 访问普通静态资源的请求路径应该是这样的：http://localhost:8080/static/....
3. 普通静态资源的存放位置也应该放到classpath:/static1/,classpath:/static2/下面，其他位置无效。

但是，存储在classpath:/META-INF/resources/目录下的dog1.jpg仍然是可以访问的：
...
因此，存储在classpath:/META-INF/resources/位置的静态资源会被默认加载，不受手动配置的影响。

## 2.2编写代码方式
我们在前面提到过，想要定制Spring MVC的行为，也可以编写类实现Spring MVC框架提供的一个接口WebMvcConfigurer，想定制哪个行为就重写哪个方法即可。
编写的类只要纳入IoC容器的管理。因此有以下两种实现方式：
● 第一种：编写类实现WebMvcConfigurer接口，重写对应的方法。
● 第二种：以组件的形式存在：编写一个方法，用@Bean注解标注。
### 2.2.1第一种方式
编写配置类，对于web开发来说，配置类一般起名为：WebConfig。配置类一般存放到config包下，因此在SpringBoot主入口程序同级目录下新建config包，在config包下新建WebConfig类：
```
package com.powernode.springboot.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

// 使用该注解标注，表示该类为配置类。
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static1/", "classpath:/static2/");
    }
}
```
注意：将application.properties文件中之前的所有配置全部注释掉。让其恢复到最原始的默认配置。

Spring Boot对Spring MVC的默认自动配置是生效的。
因此，以上的方式只是在Spring MVC默认行为之外扩展行为。

如果你不想再继续使用SpringBoot提供的默认行为，可以使用@EnableWebMvc进行标注。例如：
```
package com.powernode.springboot.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@EnableWebMvc
// 使用该注解标注，表示该类为配置类。
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static1/", "classpath:/static2/");
    }
}

```
### 2.2.2第二种方式
采用@Bean注解提供一个WebMvcConfigurer组件，代码如下：
```
package com.powernode.springboot.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig2 {

    @Bean
    public WebMvcConfigurer addResourceHandlers(){
        return new WebMvcConfigurer() {
            @Override
            public void addResourceHandlers(ResourceHandlerRegistry registry) {
                registry.addResourceHandler("/static/**")
                        .addResourceLocations("classpath:/static1/", "classpath:/static2/");
            }
        };
    }
}

```
通过了测试，并且以上代码也是在原有配置基础上进行扩展。

如果要弃用默认配置，仍然使用	@EnableWebMvc注解进行标注。自行测试！

### 2.2.3其他配置也这样做即可
以上对静态资源处理进行了手动配置，也可以做其他配置，例如拦截器：
```
package com.powernode.springboot.config;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig2 {

    @Bean
    public WebMvcConfigurer addResourceHandlers(){
        return new WebMvcConfigurer() {
            @Override
            public void addResourceHandlers(ResourceHandlerRegistry registry) {
                registry.addResourceHandler("/static/**")
                        .addResourceLocations("classpath:/static1/", "classpath:/static2/");
            }
        };
    }

    // 拦截器配置。
    @Bean
    public WebMvcConfigurer addInterceptor(){
        return new WebMvcConfigurer() {
            @Override
            public void addInterceptors(InterceptorRegistry registry) {
                registry.addInterceptor(new HandlerInterceptor() {
                    @Override
                    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
                        System.out.println("Interceptor's preHandle......");
                        return true;
                    }

                    @Override
                    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
                        System.out.println("Interceptor's postHandle......");
                    }

                    @Override
                    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
                        System.out.println("Interceptor's afterCompletion......");
                    }
                });
            }
        };
    }
}

```

### 2.2.4为什么只要容器中有WebMvcConfigurer组件即可
源码分析：

WebMvcAutoConfiguration部分源码：
<img src="images/107.png" width="100%">
WebMvcAutoConfiguration类的内部类EnableWebMvcConfiguration，这个类继承了DelegatingWebMvcConfiguration（Delegating是委派的意思。）

DelegatingWebMvcConfiguration部分源码：
<img src="images/108.png" width="100%">
DelegatingWebMvcConfiguration中的setConfigurers()方法用来设置配置。而配置参数是一个List集合，这个List集合中存放的是WebMvcConfigurer接口的实例，并且可以看到这个方法上面使用了@Autowired进行了自动注入，这也就是说为什么只要是IoC容器中的组件就能生效的原因。
我们再次进入到this.configurers.addWebMvcConfigurers(configurers);方法中进一步查看源码：
<img src="images/109.png" width="100%">
对于WebMvcConfigurerComposite类的代码来说，它是一个非常典型的**<font style="color:#DF2A3F;">组合模式</font>**。
组合模式的关键点：
1. 组合多个 WebMvcConfigurer 实例：WebMvcConfigurerComposite 通过 delegates 列表组合了多个 WebMvcConfigurer 实例。
2. 统一接口：WebMvcConfigurerComposite 实现了 WebMvcConfigurer 接口，因此可以像一个单一的 WebMvcConfigurer 一样被使用。
3. 代理调用：在实现 WebMvcConfigurer 接口的方法时，WebMvcConfigurerComposite 会遍历 delegates 列表，调用每个 WebMvcConfigurer 实例的相应方法。
总结：WebMvcConfigurerComposite 主要采用了组合模式的思想，将多个 WebMvcConfigurer 实例组合在一起，形成一个整体。
注意：组合模式是GoF 23种设计模式中的结构型设计模式。

# 3.web请求的路径匹配
我们在学习SpringMVC的时候，路径匹配规则中学习了Ant风格的路径匹配规则。大家可以翻看一下之前的Spring MVC视频。
在Spring Boot3中，对web请求的路径匹配提供了两种规则：
● 第一种：AntPathMatcher（Ant风格）【较旧】
● 第二种：PathPatternParser（从Spring5.3中引入的。在SpringBoot2.4中引入的。）【较新：效率高】
  ○ 效率比Ant高，一般新项目中使用**<font style="color:#DF2A3F;">PathPatternParser</font>**

SpringBoot3中默认使用的是PathPatternParser，不需要任何配置。如果要使用AntPathMatcher，就需要进行如下的配置：
```
spring.mvc.pathmatch.matching-strategy=ant_path_matcher
```
## 3.1AntPathMatcher
Ant风格的路径匹配规则回顾：
*	匹配任意长度的任意字符序列（不包括路径分隔符）。示例：/foo/*.html 匹配 /foo/bar.html 和 /foo/baz.html。
**	匹配任意数量的目录层级。示例：/foo/** 匹配 /foo/bar、/foo/bar/baz 和 /foo/bar/baz/qux。
?**	匹配任意单个字符**。示例：/foo?bar 匹配 /foobar 和 /fooxbar。
[]**	**匹配指定范围内的单个字符。示例：/foo[a-z]bar 匹配 /fooabar、/foobbar 等。
{}**	**路径变量，用于提取路径的一部分作为参数。示例：/users/{userId} 匹配 /users/123，提取 userId=123。

如果在SpringBoot3中启用Ant风格，记得配置：
```
spring.mvc.pathmatch.matching-strategy=ant_path_matcher
```
如下代码：请分析以下路径匹配的是什么样的路径。
```
package com.powernode.springboot.controller;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class PathController {

    @GetMapping("/{path:[a-z]+}/a?/**/*.do")
    public String path(HttpServletRequest request, @PathVariable String path){
        return request.getRequestURI() + "," + path;
    }
}
```
## 3.2PathPatternParser
项目中不做配置，或者按照以下方式配置，都是PathPatternParser：
```
spring.mvc.pathmatch.matching-strategy=path_pattern_parser
```
PathPatternParser风格是兼容Ant风格的。只有一个地方PathPatternParser不支持，Ant支持。在Ant风格中，**可以出现在任意位置。在PathPatternParser中只允许**出现在路径的末尾。

可以测试一下，将配置文件中的Ant风格注释掉，采用PathPatternParser风格。然后控制器代码如下：
```
package com.powernode.springboot.controller;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class PathController {

    @GetMapping("/{path:[a-z]+}/a?/**/*.do")
    public String path(HttpServletRequest request, @PathVariable String path){
        return request.getRequestURI() + "," + path;
    }
}
```
提示你，如果在路径当中出现了**，需要将路径匹配规则替换为Ant风格。因此路径当中如果出现**，那么必须使用Ant风格。除此之外，PathPatternParser均可用。

我们再来测试一下，**放到末尾，对于PathPatternParser是否可用？
```
@GetMapping("/{path:[a-z]+}/a?/*.do/**")
public String path(HttpServletRequest request, @PathVariable String path){
    return request.getRequestURI() + "," + path;
}
```
启动访问，可用
## 3.3路径匹配相关源码
底层选择路径匹配规则的源码是：
<img src="images/110.png" width="100%">

# 4.内容协商
内容协商机制是Spring MVC框架提供的，接下来主要是学习在SpringBoot中是如何支持SpringMVC内容协商机制的。
## 4.1对内容协商的理解
内容协商机制是指服务器根据客户端的请求来决定返回资源的最佳表示形式。
白话文描述：客户端要什么格式的数据，咱后端就应该返回什么格式的数据。
● 客户端要JSON，咱就响应JSON。
● 客户端要XML，咱就响应XML。
● 客户端要YAML，咱就响应YAML。
你可能会有疑问：客户端接收数据时统一采用一种格式，例如JSON，不就行了吗。哪那么多事儿呀！！！
但在实际的开发中，不是这样的，例如：
● 遗留的老客户端系统，仍然处理的是XML格式的数据。
● 要求处理速度快的这种客户端系统，一般要求返回JSON格式的数据。
● 要求安全性高的客户端系统，一般要求返回XML格式的数据。
因此，在现代的开发中，不同的客户端可能需要后端系统返回不同格式的数据。总之后端应该满足这种多样化的需求。
## 4.2实现内容协商的两种方式
通常通过HTTP请求头（如 Accept）或请求参数（如 format）来指定客户端偏好接收的内容类型（如JSON、XML等）。服务器会根据这些信息选择最合适的格式进行响应。
### 4.2.1通过HTTP请求头（如Accept）
SpringBoot框架中，在程序员不做任何配置的情况下，优先考虑的是这种方式。
服务器会根据客户端发送请求时提交的请求头中的"Accept: application/json" 或 "Accept: application/xml" 或 "Accept: text/html"来决定响应什么格式的数据。

客户端发送请求给服务器的时候，如何设置请求头的Accept？有以下几种常见实现方式：
● 写代码
  ○ fetch API
  ○ ajax的XMLHttpRequest
  ○ axios库
  ○ jQuery库......
● 用工具
  ○ 接口测试工具，例如：Postman、Apifox等。
  ○ 命令行工具：curl

对于我们编写的以下Controller来说：
```
package com.powernode.springboot.controller;

import com.powernode.springboot.bean.User;
import com.powernode.springboot.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping(value = "/detail")
    public User detail(){
        return userService.getUser();
    }
}
```
我们使用了@RestController，也就是使用了@ResponseBody。因此默认支持的是返回JSON数据。怎么才能支持返回XML格式的数据呢？需要做以下两步：
第一步：引入一个依赖
```
<dependency>
  <groupId>com.fasterxml.jackson.dataformat</groupId>

  <artifactId>jackson-dataformat-xml</artifactId>

</dependency>

```
第二步：在实体类上添加一个注解
```
package com.powernode.springboot.bean;

import com.fasterxml.jackson.dataformat.xml.annotation.JacksonXmlRootElement;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@JacksonXmlRootElement
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private String name;
    private String password;
}
```
接下来，我们使用curl命令行工具，来模拟发送请求，并在请求头中设置Accept：
<img src="images/111.png" width="100%">
可以很清楚的看到，服务器根据不同的请求头返回了不同格式的数据。
● Accept: application/xml则返回XML格式的数据
● Accept: application/json则返回JSON格式的数据
### 4.2.2通过请求参数（如format）
接下来我们使用请求参数的方式，来测试一下服务器的响应，注意：默认的请求参数名为format。
我们仍然使用curl命令行工具进行测试：
<img src="images/112.png" width="100%">
我们可以看到，并没有达到我们想要的效果，这是因为SpringBoot优先考虑的不是通过请求参数format方式。如何优先考虑使用format方式呢？做如下配置：
```
# 内容协商时，优先考虑请求参数format方式。
spring.mvc.contentnegotiation.favor-parameter=true
```
再次测试：
<img src="images/113.png" width="100%">
可以看到，现在SpringBoot已经优先考虑使用请求参数format方式了。

当然，请求参数的名字可以不使用format吗？支持定制化吗？答案是支持的，例如你希望请求参数的名字为type，可以做如下配置：
```
# 内容协商时，设置请求参数的名字，默认为format
spring.mvc.contentnegotiation.parameter-name=type
```
再次使用curl工具进行测试：
<img src="images/114.png" width="100%">

## 4.3HttpMessageConverter接口

### 4.3.1HttpMessageConverter的理解
HttpMessageConverter接口，对于这个接口来说，大家应该不陌生，在Spring MVC的课程当中咱们已经详细的学习过了。在这里简单回顾一下。
HttpMessageConverter接口被翻译为：Http消息转换器。它起到转换Http消息的作用。
什么是Http消息？所谓的Http消息本质上就是浏览器向服务器发送请求时提交的数据，或者服务器向浏览器响应的数据。
而HttpMessageConverter接口就是负责完成请求/响应时的数据格式转换的。
在Spring MVC中提供了很多HttpMessageConverter接口的实现类，不同的Http消息转换器具有不同的转换效果，有的负责将Java对象转换为JSON格式的字符串，有的负责将Java对象转换成XML格式的字符串。
### 4.3.2常见的HttpMessageConverter
内置的常见的HttpMessageConverter的实现类包括：
● 【请求】提交的表单（form）数据转换成Java对象的主要任务是由 FormHttpMessageConverter 消息转换器完成的
● 【请求】提交的JSON数据转换成Java对象的主要任务是由 MappingJackson2HttpMessageConverter 消息转换器完成的。（我们通常使用的<font style="color:#DF2A3F;">@RequestBody</font>注解）
● 【响应】将Java对象转换成JSON格式的数据，并将其写入HTTP响应体的任务是由 MappingJackson2HttpMessageConverter 消息转换器完成。（我们通常使用的@ResponseBody注解)
● 【响应】将Java对象转换成XML格式的数据，并将其写入HTTP响应体的任务通常由 Jaxb2RootElementHttpMessageConverter 消息转换器完成。
● 【响应】将 String 直接写入到响应体的任务是由 StringHttpMessageConverter 消息转换器完成。
● ......
### 4.3.3请求时通过哪些条件确定使用哪个转换器
请求时通常根据以下条件来确定使用哪个消息转换器：
1. 请求的 Content-Type 字段：
Spring MVC 会检查请求的 Content-Type 字段，以确定请求体的数据格式（例如 application/json、application/x-www-form-urlencoded、application/xml 等）。
2. 方法参数类型：
控制器方法中接收请求体的参数类型（例如 @RequestBody）。
### 4.3.4响应时通过哪些条件确定使用哪个转换器
响应时通常根据以下条件来确定使用哪个消息转换器：
1. 请求提交时，请求头上的Accept字段 ：
Spring MVC 会检查客户端请求的 Accept 字段，以确定客户端期望的响应格式（例如 application/json、application/xml 等）。
2. 方法返回值类型：
控制器方法的返回值类型（例如 @ResponseBody）。
例如1：@ResponseBody + 控制器方法的返回值是String，则使用StringHttpMessageConverter转换器。（将字符串直接写入响应体）
例如2：@ResponseBody + 控制器方法的返回值是User，则使用MappingJackson2HttpMessageConverter转换器。（将java对象转换成json格式的字符串写入到响应体）
### 4.3.5系统默认提供了哪些HttpMessageConverter
查看源码：
WebMvcAutoConfiguration.EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport
WebMvcAutoConfiguration内部类EnableWebMvcConfiguration
EnableWebMvcConfiguration继承了DelegatingWebMvcConfiguration
DelegatingWebMvcConfiguration继承了WebMvcConfigurationSupport

在WebMvcConfigurationSupport类中有这样一个方法：addDefaultHttpMessageConverters() 用来添加默认的HttpMessageConverter对象。
通过断点调试，可以发现默认支持6个HttpMessageConverter，如下：
<img src="images/115.png" width="100%">
这6个HttpMessageConverter作用如下：
1. ByteArrayHttpMessageConverter:
用于将字节数组(byte[])与HTTP消息体之间进行转换。这通常用于处理二进制数据，如图片或文件。
2. StringHttpMessageConverter:
用于将字符串(String)与HTTP消息体之间进行转换。它支持多种字符集编码，能够处理纯文本内容。
3. ResourceHttpMessageConverter:
用于将Spring的Resource对象与HTTP消息体之间进行转换。Resource是Spring中表示资源的接口，可以读取文件等资源。这个转换器对于下载文件或发送静态资源有用。
4. ResourceRegionHttpMessageConverter:
用于处理资源的部分内容（即“Range”请求），特别是当客户端请求大文件的一部分时。这对于实现视频流媒体等功能很有用。
5. AllEncompassingFormHttpMessageConverter:
用于处理表单，是一个比较全面的form消息转换器。处理标准的application/x-www-form-urlencoded格式的数据，以及包含文件上传的multipart/form-data格式的数据。
6. MappingJackson2HttpMessageConverter:
使用Jackson库来序列化和反序列化JSON数据。可以将Java对象转换为JSON格式的字符串，反之亦然。

另外，通过以下源码，也可以看到SpringBoot是根据类路径中是否存在某个类，而决定是否添加对应的消息转换器的：
<img src="images/116.png" width="100%">

<img src="images/117.png" width="100%">
因此，我们只要引入相关的依赖，让类路径存在某个类，则对应的消息转换器就会被加载。
## 4.4定义自己的HttpMessageConverter
可以看到以上6个消息转换器中没有yaml相关的消息转换器，可见，如果要实现yaml格式的内容协商，yaml格式的消息转换器就需要我们自定义了。
### 4.4.1第一步：引入能够处理yaml格式的依赖
任何一个能够处理yaml格式数据的库都可以，这里选择使用jackson的库，因为它既可以处理json，xml，又可以处理yaml。
```
<dependency>
  <groupId>com.fasterxml.jackson.dataformat</groupId>

  <artifactId>jackson-dataformat-yaml</artifactId>

</dependency>

```
编写测试程序，简单测试一下这个库的用法：
```
package com.powernode.springboot;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import com.fasterxml.jackson.dataformat.yaml.YAMLGenerator;
import com.powernode.springboot.bean.User;

public class Jackson2YamlTest {
    public static void main(String[] args) throws JsonProcessingException {
        // 创建YAML工厂类
        YAMLFactory yamlFactory = new YAMLFactory().disable(YAMLGenerator.Feature.WRITE_DOC_START_MARKER); // 禁止使用文档头标记
        // 创建对象映射器
        ObjectMapper objectMapper = new ObjectMapper(yamlFactory);
        // 准备数据
        User user = new User("jackson", "jack123");
        // 将数据转换成YAML格式
        String s = objectMapper.writeValueAsString(user);
        System.out.println(s);
    }
}
```
### 4.4.2第二步：新增一种媒体类型yaml
默认支持xml和json两种媒体类型，要支持yaml格式的，需要新增一个yaml媒体类型，在springboot的配置文件中进行如下配置：
```
spring.mvc.contentnegotiation.media-types.yaml=text/yaml
```
注意，以上types后面的yaml是媒体类型的名字，名字随意，如果媒体类型起名为xyz，那么发送请求时的路径应该是这样的：http://localhost:8080/detail?format=xyz

### 4.4.3第三步：自定义HttpMessageConverter
编写类YamlHttpMessageConverter继承AbstractHttpMessageConverter，代码如下：
```
package com.powernode.springboot.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import com.fasterxml.jackson.dataformat.yaml.YAMLGenerator;
import com.powernode.springboot.bean.User;
import org.springframework.http.HttpInputMessage;
import org.springframework.http.HttpOutputMessage;
import org.springframework.http.MediaType;
import org.springframework.http.converter.AbstractHttpMessageConverter;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.http.converter.HttpMessageNotWritableException;

import java.io.IOException;
import java.nio.charset.Charset;

public class YamlHttpMessageConverter extends AbstractHttpMessageConverter<Object> {

    private ObjectMapper objectMapper = new ObjectMapper(new YAMLFactory().disable(YAMLGenerator.Feature.WRITE_DOC_START_MARKER));

    public YamlHttpMessageConverter() {
        // 让 消息转换器 和 媒体类型 application/yaml 绑定在一起。
        super(new MediaType("text", "yaml", Charset.forName("UTF-8")));
    }

    @Override
    protected boolean supports(Class<?> clazz) {
        // 表示User类型的数据支持yaml，其他类型不支持
        return User.class.isAssignableFrom(clazz);
    }

    // 处理 @RequestBody（将提交的yaml格式数据转换为java对象）
    @Override
    protected Object readInternal(Class<?> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        return null;
    }

    // 处理 @ResponseBody（将java对象转换为yaml格式的数据）
    @Override
    protected void writeInternal(Object o, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        this.objectMapper.writeValue(outputMessage.getBody(), o);
        // 注意：spring框架会自动关闭输出流，无需程序员手动释放。
    }
}
```

### 4.4.4配置消息转换器
重写WebMvcConfigurer接口的configureMessageConverters方法：
```
package com.powernode.springboot.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import java.util.List;

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new YamlHttpMessageConverter());
    }
}
```
启动服务器并测试：http://localhost:8080/detail?type=yaml

# 5.Spring boot整合Thymeleaf

## 5.1传统web应用和前后端分离

### 5.1.1传统web应用（服务端渲染）
传统Web应用中，页面由服务端渲染生成，浏览器接收到的是完整的HTML页面。典型的技术栈是：Spring Boot + Thymeleaf/JSP + jQuery。

特点：
- 服务端负责组装数据和渲染页面
- 页面跳转由服务端控制
- SEO友好，首屏加载快
- 前后端耦合度高，维护成本大

### 5.1.2前后端分离
前后端分离模式下，后端只提供RESTful API接口（返回JSON数据），前端通过Ajax/Fetch调用接口获取数据并渲染页面。典型技术栈：Spring Boot + Vue/React/Angular。

特点：
- 前后端独立开发、独立部署
- 后端专注于业务逻辑和接口
- 前端负责页面渲染和用户交互
- 可同时支持Web、移动端、小程序等多种客户端

## 5.2Spring Boot整合Thymeleaf

### 5.2.1引入依赖
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```
引入该启动器后，会自动传递引入 thymeleaf-spring6 和 thymeleaf 核心库。

### 5.2.2编写Controller
Thymeleaf是服务端渲染模板引擎，Controller中需要返回视图名称而不是@ResponseBody的JSON数据。因此Controller使用@Controller注解，而不是@RestController：

```java
@Controller
public class HelloController {

    @GetMapping("/hello")
    public String hello(Model model){
        // 向模板传递数据
        model.addAttribute("msg", "Hello Thymeleaf!");
        // 返回视图名称（对应 templates/hello.html）
        return "hello";
    }
}
```

### 5.2.3创建模板文件
在 src/main/resources/templates/ 目录下创建 hello.html：

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Thymeleaf Demo</title>
</head>
<body>
    <h1 th:text="${msg}">默认文本</h1>
</body>
</html>
```

注意：
- 必须引入 xmlns:th="http://www.thymeleaf.org" 命名空间，IDEA才会有语法提示
- 模板文件默认放在 classpath:/templates/ 目录下
- Thymeleaf模板本质上是合法的HTML文件，即使不经过模板引擎处理也能在浏览器中预览

## 5.3Thymeleaf的自动配置

### 5.3.1自动配置类 ThymeleafAutoConfiguration
Spring Boot为Thymeleaf提供了自动配置类 `ThymeleafAutoConfiguration`，其中配置了模板引擎、模板解析器、视图解析器等组件。

### 5.3.2属性类 ThymeleafProperties
Thymeleaf的默认配置由 `ThymeleafProperties` 管理，配置前缀为 `spring.thymeleaf`：

```java
@ConfigurationProperties(prefix = "spring.thymeleaf")
public class ThymeleafProperties {
    private static final Charset DEFAULT_ENCODING = StandardCharsets.UTF_8;
    public static final String DEFAULT_PREFIX = "classpath:/templates/";
    public static final String DEFAULT_SUFFIX = ".html";
    // ...
}
```

关键默认值：
- 模板前缀（prefix）：`classpath:/templates/`
- 模板后缀（suffix）：`.html`
- 编码（encoding）：`UTF-8`
- 模板模式（mode）：`HTML`
- 缓存（cache）：`true`（生产环境默认开启）

### 5.3.3自定义配置示例
```
# 修改模板前缀
spring.thymeleaf.prefix=classpath:/views/
# 修改模板后缀
spring.thymeleaf.suffix=.htm
# 关闭缓存（开发时建议关闭）
spring.thymeleaf.cache=false
# 设置编码
spring.thymeleaf.encoding=UTF-8
```

## 5.4Thymeleaf核心语法

### 5.4.1th:text 替换标签体内容
将标签体内的文本内容替换为表达式的值，会对HTML特殊字符进行转义：

```html
<span th:text="${user.name}">用户名</span>
<!-- 如果 user.name = "Jack<Boss>"，页面显示：Jack<Boss> -->
```

### 5.4.2th:utext 替换标签体内容（不转义）
与 th:text 类似，但不会对HTML标签进行转义，会原样输出：

```html
<div th:utext="${content}">内容</div>
<!-- 如果 content = "<b>重要</b>"，页面显示为加粗的"重要" -->
```

注意：使用 th:utext 需要确保内容安全，避免XSS攻击。

### 5.4.3th:任意属性名 动态替换该属性的值
可以动态设置任意HTML属性的值：

```html
<input type="text" th:value="${user.name}" th:placeholder="${placeholder}" />
<a th:href="@{/user/detail(id=${user.id})}">详情</a>
<img th:src="@{/images/logo.png}" th:alt="${altText}" />
```

### 5.4.4th:attr 属性合并设置
一次性设置多个属性：

```html
<img th:attr="src=@{/images/logo.png},alt=${altText},title=${title}" />
<!-- 等价于分别使用 th:src, th:alt, th:title -->
```

### 5.4.5th:指令概览
常用指令列表：

| 指令 | 作用 |
|------|------|
| th:text | 替换文本内容（转义） |
| th:utext | 替换文本内容（不转义） |
| th:value | 设置表单元素value |
| th:href | 设置链接地址 |
| th:src | 设置资源路径 |
| th:each | 循环遍历 |
| th:if / th:unless | 条件判断 |
| th:switch / th:case | 多条件判断 |
| th:object | 指定表单对象 |
| th:field | 表单字段绑定 |
| th:fragment | 定义代码片段 |
| th:replace | 替换为代码片段 |
| th:insert | 插入代码片段 |
| th:with | 定义局部变量 |

### 5.4.6@{} 表达式
用于构建URL，自动添加上下文路径：

```html
<!-- 绝对路径（相对于服务器根） -->
<a th:href="@{/user/list}">用户列表</a>
<!-- 带参数 -->
<a th:href="@{/user/detail(id=${user.id}, type=${type})}">详情</a>
<!-- 带路径变量（RESTful风格） -->
<a th:href="@{/user/{id}/delete(id=${user.id})}">删除</a>
```

渲染结果示例（假设上下文路径为 /app）：
```
/app/user/detail?id=10&type=admin
/app/user/10/delete
```

### 5.4.7Thymeleaf的内置工具
Thymeleaf提供了一些内置的工具对象，使用 `#` 前缀访问：

| 工具对象 | 说明 | 示例 |
|---------|------|------|
| #dates | 日期格式化 | `#dates.format(date, 'yyyy-MM-dd')` |
| #numbers | 数字格式化 | `#numbers.formatDecimal(price, 1, 2)` |
| #strings | 字符串操作 | `#strings.isEmpty(name)` |
| #lists | 列表操作 | `#lists.size(list)` |
| #maps | Map操作 | `#maps.size(map)` |
| #arrays | 数组操作 | `#arrays.length(array)` |
| #bools | 布尔操作 | `#bools.isTrue(flag)` |

使用示例：
```html
<span th:text="${#dates.format(createTime, 'yyyy-MM-dd HH:mm:ss')}">时间</span>
<span th:text="${#numbers.formatDecimal(price, 1, 2)}">价格</span>
<span th:text="${#strings.toUpperCase(name)}">名称</span>
```

### 5.4.8Thymeleaf也支持运算符
支持常见的运算符：

```html
<!-- 算术运算 -->
<div th:text="${price * quantity}">金额</div>

<!-- 比较运算：gt, lt, ge, le, eq, ne -->
<div th:if="${age gt 18}">成年</div>

<!-- 逻辑运算：and, or, not -->
<div th:if="${vip and age gt 18}">成年VIP</div>

<!-- 三元运算 -->
<div th:text="${gender eq 1} ? '男' : '女'">性别</div>

<!-- Elvis运算符（默认值） -->
<div th:text="${name} ?: '匿名用户'">名称</div>
```

### 5.4.9Thymeleaf字符串拼接
两种方式拼接字符串：

```html
<!-- 方式一：使用 + 号 -->
<span th:text="${firstName} + ' ' + ${lastName}">姓名</span>

<!-- 方式二：使用 | 管道符（字面量替换） -->
<span th:text="|${firstName} ${lastName}|">姓名</span>
<!-- 等价于：firstName + ' ' + lastName -->

<!-- 复杂管道符示例 -->
<span th:text="|欢迎您，${user.name}！（编号：${user.id}）|">欢迎语</span>
```

### 5.4.10循环遍历
使用 th:each 进行循环，支持迭代状态变量：

```html
<!-- 基本遍历 -->
<tr th:each="user : ${userList}">
    <td th:text="${user.id}">ID</td>
    <td th:text="${user.name}">名称</td>
</tr>

<!-- 带迭代状态变量 -->
<tr th:each="user, iterStat : ${userList}">
    <td th:text="${iterStat.index}">序号(0开始)</td>
    <td th:text="${iterStat.count}">序号(1开始)</td>
    <td th:text="${user.name}">名称</td>
</tr>
```

迭代状态对象属性：
- index：当前索引（从0开始）
- count：当前计数（从1开始）
- size：集合总大小
- odd：是否奇数行
- even：是否偶数行
- first：是否是第一个
- last：是否是最后一个

### 5.4.11条件判断th:if
```html
<!-- 单条件 -->
<div th:if="${user.vip}">VIP用户可看内容</div>

<!-- 取反判断：th:unless -->
<div th:unless="${user.vip}">普通用户可看内容</div>
<!-- unless 等价于 if 条件取反 -->

<!-- 复合条件 -->
<div th:if="${user.age gt 18 and user.vip}">成年VIP专属</div>
```

### 5.4.12条件判断th:switch
```html
<div th:switch="${user.role}">
    <p th:case="'admin'">管理员</p>
    <p th:case="'editor'">编辑</p>
    <p th:case="*">普通用户</p>  <!-- * 表示默认 -->
</div>
```

### 5.4.13Thymeleaf属性优先级
当同一个标签上有多个 th:* 属性时，处理顺序由优先级决定：

| 优先级 | 属性 | 说明 |
|--------|------|------|
| 1 | th:insert, th:replace | 片段包含（最高） |
| 2 | th:each | 循环遍历 |
| 3 | th:if, th:unless, th:switch, th:case | 条件判断 |
| 4 | th:object, th:with | 定义变量 |
| 5 | th:attr, th:attrprepend, th:attrappend | 属性设置 |
| 6 | th:value, th:href, th:src... | 特定属性 |
| 7 | th:text, th:utext, th:fragment | 文本/片段 |

### 5.4.14*{...}表达式
`*{}` 是选择变量表达式，必须在 th:object 指定的对象范围内使用：

```html
<!-- 方式一：直接使用 ${} -->
姓名：<span th:text="${user.name}">name</span>
年龄：<span th:text="${user.age}">age</span>

<!-- 方式二：使用 th:object + *{} -->
<form th:object="${user}">
    姓名：<input type="text" th:field="*{name}" />
    年龄：<input type="text" th:field="*{age}" />
    邮箱：<input type="text" th:field="*{email}" />
</form>
```

使用 `*{}` 的好处是代码更简洁，尤其是在表单绑定场景下。

### 5.4.15代码片段
Thymeleaf支持模板片段复用，类似于"组件"的概念。

定义片段（common.html）：
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<!-- th:fragment 定义片段 -->
<header th:fragment="header(title)">
    <h1 th:text="${title}">默认标题</h1>
    <nav>
        <a th:href="@{/}">首页</a>
        <a th:href="@{/about}">关于</a>
    </nav>
</header>

<div th:fragment="footer">
    <p>&copy; 2024 My Company</p>
</div>
</html>
```

引用片段：
```html
<!-- th:replace：用片段替换整个标签 -->
<div th:replace="~{common :: header('我的网站')}"></div>

<!-- th:insert：将片段插入到标签内部 -->
<div th:insert="~{common :: footer}"></div>

<!-- 简化写法（省略～{}） -->
<div th:replace="common :: header('我的网站')"></div>
```

th:replace 和 th:insert 的区别：
- th:replace：用模板片段**替换**当前标签
- th:insert：将模板片段**插入到**当前标签内部

参数传递：
```html
<!-- 定义带参数的片段 -->
<div th:fragment="alert(type, message)">
    <div th:class="'alert alert-' + ${type}" th:text="${message}"></div>
</div>

<!-- 调用时传参 -->
<div th:replace="common :: alert('success', '操作成功！')"></div>
```

## 5.5Thymeleaf页面修改如何立即生效

### 5.5.1关闭模板缓存
在开发阶段，希望修改模板后刷新浏览器立即看到效果，需要关闭Thymeleaf缓存：

```properties
# 关闭Thymeleaf模板缓存
spring.thymeleaf.cache=false
```

### 5.5.2配合 DevTools 实现自动刷新
更高效的方案是配合 spring-boot-devtools，修改代码或模板后应用自动重启，配合浏览器LiveReload插件实现页面自动刷新：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

DevTools + 关闭缓存 = 修改模板后浏览器自动刷新，极大提升开发效率。

### 5.5.3IDEA设置（重要）
使用DevTools时需要在IDEA中做以下设置：
1. File → Settings → Build, Execution, Deployment → Compiler → 勾选 "Build project automatically"
2. Ctrl+Shift+Alt+/ → Registry → 勾选 "compiler.automake.allow.when.app.running"

# 6.异常处理
在controller层如果程序出现了异常，并且这个异常未被捕获，springboot提供的异常处理机制将生效。
Spring Boot 提供异常处理机制主要是为了提高应用的健壮性和用户体验。它的好处包括：
1. 统一错误响应：可以定义全局异常处理器来统一处理各种异常，确保返回给客户端的错误信息格式一致，便于前端解析。
2. 提升用户体验：能够优雅地处理异常情况，避免直接将技术性错误信息暴露给用户，而是显示更加友好的提示信息。
3. 简化代码：开发者不需要在每个可能抛出异常的方法中重复编写异常处理逻辑，减少冗余代码，使业务代码更加清晰简洁。
4. 增强安全性：通过控制异常信息的输出，防止敏感信息泄露，增加系统的安全性。
## 6.1自适应的错误处理机制
springboot会根据请求头的Accept字段来决定错误的响应格式。
这种机制的好处就是：客户端设备自适应，提高用户的体验。


## 6.2SpringMVC的错误处理方案
重点：如果程序员使用了SpringMVC的错误处理方案，SpringBoot的错误处理方案不生效。
### 6.2.1局部控制 @ExceptionHandler
局部控制 @ExceptionHandler
在控制器当中编写一个方法，方法使用@ExceptionHandler注解进行标注，凡是这个控制器当中出现了对应的异常，则走这个方法来进行异常的处理。局部生效。
```
package com.powernode.test.controller;

import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {

    @GetMapping("/resource/{id}")
    public String getResource(@PathVariable Long id){
        if(id == 1){
            throw new IllegalArgumentException("无效ID：" + id);
        }
        return "ID = " + id;
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public String handler(IllegalArgumentException e){
        return "错误信息：" + e.getMessage();
    }
}

```
可以再编写一个OtherController，让它也发生IllegalArgumentException异常，看看它会不会走局部的错误处理机制。
```
package com.powernode.test.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class OtherController {
    @GetMapping("/resource2/{id}")
    public String getResource(@PathVariable Long id){
        if(id == 1){
            throw new IllegalArgumentException("无效ID：" + id);
        }
        return "ID = " + id;
    }
}

```
通过测试，确实局部生效。

### 6.2.2全局控制 @ControllerAdvice + @ExceptionHadler
也可以把以上局部生效的方法单独放到一个类当中，这个类使用@ControllerAdvice注解标注，凡是任何控制器当中出现了对应的异常，则走这个方法来进行异常的处理。全局生效。
将之前的局部处理方案的代码注释掉。使用全局处理方式，编写以下类：
```
package com.powernode.test.controller;

import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;

@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    @ResponseBody
    public String handler(IllegalArgumentException e){
        return "错误信息：" + e.getMessage();
    }
}

```
通过测试，确实全局生效。

## 6.3SpringBoot的错误处理方案
重点：如果SpringMVC没有对应的处理方案，会开启SpringBoot默认的错误处理方案。
SpringBoot默认的错误处理方案如下：
1. 如果客户端要的是json，则直接响应json格式的错误信息。
2. 如果客户端要的是html页面，则按照下面方案：
● 第一步（精确错误码文件）：去classpath:/templates/error/目录下找404.html``500.html等精确错误码.html文件。如果找不到，则去静态资源目录下的/error目录下找。如果还是找不到，才会进入下一步。
● 第二步（模糊错误码文件）：去classpath:/templates/error/目录下找4xx.html``5xx.html等模糊错误码.html文件。如果找不到，则去静态资源目录下的/error目录下找。如果还是找不到，才会进入下一步。
● 第三步（通用错误页面）：去找classpath:/templates/error.html如果找不到则进入下一步。
● 第四步（默认错误处理）：如果上述所有步骤都未能找到合适的错误页面，Spring Boot 会使用内置的默认错误处理机制，即 /error 端点。
## 6.4如何在错误页获取错误信息
Spring Boot 默认会在模型Model中放置以下信息：
● timestamp: 错误发生的时间戳
● status: HTTP 状态码
● error: 错误类型（如 "Not Found"）
● exception: 异常类名
● message: 错误消息
● trace: 堆栈跟踪
在thymeleaf中使用 ${message}即可取出信息。

注意：springboot3.3.5版本默认只向Model对象中绑定了timestamp``status``error。如果要保存exception``message``trace，需要开启以下三个配置：
```
server.error.include-stacktrace=always
server.error.include-exception=true
server.error.include-message=always
```
## 6.5前后端分离项目的错误处理方案
统一使用SpringMVC的错误处理方案，定义全局的异常处理机制：@ControllerAdvice + @ExceptionHandler
返回json格式的错误信息，其它的就不需要管了，因为前端接收到错误信息怎么处理是他自己的事儿。

## 6.6服务器端负责页面渲染的错误处理方案
建议使用SpringBoot的错误处理方案：
1. 如果发生的异常是HTTP错误状态码：
    a. 建议常见的错误码给定精确错误码.html
    b. 建议不常见的错误码给定模糊错误码.html
2. 如果发生的异常不是HTTP错误状态码，而是业务相关异常：
    a. 在程序中处理具体的业务异常，自己通过程序来决定跳转到哪个错误页面。
3. 建议提供classpath:/templates/error.html来处理通用错误。

# 7.国际化
在Spring Boot中实现国际化（i18n）是一个常见的需求，它允许应用程序根据用户的语言和地区偏好显示不同的文本。

## 7.1实现国际化

### 7.1.1第一步：创建资源文件
创建包含不同语言版本的消息文件。这些文件通常放在src/main/resources目录下，并且以.properties为扩展名。例如：
● messages.properties (默认语言，如英语)
● messages_zh_CN.properties (简体中文)
● messages_fr.properties (法语)


每个文件都应包含相同的消息键，但值应对应于相应的语言。例如：
messages.properties:
```
welcome.message=Welcome to our application!
```
messages_zh_CN.properties:
```
welcome.message=欢迎来到我们的应用！
```
messages_fr.properties:
```
welcome.message=Bienvenue dans notre application !
```
### 7.1.2第二步：在模板文件中取出消息
语法格式为：#{welcome.message}
```
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>

</head>

<body>
<h1 th:text="#{welcome.message}"></h1>

</body>

</html>

```

## 7.2国际化实现原理
做国际化的自动配置类是：MessageSourceAutoConfiguration
<img src="images/118.png" width="100%">
通过以上源码得知，国际化对应的配置前缀是：spring.message
例如在application.properties中进行如下配置：
```
# 配置国际化文件命名的基础名称
spring.messages.basename=messages

# 指定国际化信息的字符编码方式
spring.messages.encoding=UTF-8
```
注意：标准标识符：en_US 和 zh_CN 这样的标识符是固定的，不能更改。可以设置的是basename。
## 7.3在程序当中如何获取国际化信息
在国际化自动配置类中可以看到这样一个Bean：MessageSource，它是专门用来处理国际化的。我们可以将它注入到我们的程序中，然后调用相关方法在程序中获取国际化信息。
```
@Controller
public class MyController {

    @Autowired
    private MessageSource messageSource;

    @GetMapping("/test")
    @ResponseBody
    public String test(HttpServletRequest request){
        Locale locale = request.getLocale();
        String message = messageSource.getMessage("welcome.message", null, locale);
        return message;
    }
}
```

# 8.定制web容器

## 8.1web服务器切换为jetty
springboot默认嵌入的web服务器是Tomcat，如何切换到jetty服务器？
实现方式：排除Tomcat，添加Jetty依赖
修改 pom.xml 文件：在 pom.xml 中，确保你使用 spring-boot-starter-web 并排除 Tomcat，然后添加 Jetty 依赖。
```
<!-- 排除 Tomcat -->
<dependency>
    <groupId>org.springframework.boot</groupId>

    <artifactId>spring-boot-starter-web</artifactId>

    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>

            <artifactId>spring-boot-starter-tomcat</artifactId>

        </exclusion>

    </exclusions>

</dependency>

<!-- 添加 Jetty 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>

    <artifactId>spring-boot-starter-jetty</artifactId>

</dependency>

```

## 8.2web服务器切换原理
从哪里可以看出springboot是直接将tomcat服务器嵌入到应用中的呢？看这个类：ServletWebServerFactoryAutoConfiguration
<img src="images/119.png" width="100%">
以上代码显示嵌入的是3个服务器。但并不是都生效，我们来看一下生效条件：
<img src="images/120.png" width="100%">

生效条件是，看类路径当中是否有对应服务器相关的类，如果有则生效。spring-boot-web-starter这个web启动器引入的时候，大家都知道，它间接引入的是tomcat服务器的jar包。因此默认Tomcat服务器被嵌入。如果想要切换web服务器，将tomcat相关jar包排除掉，引入jetty的jar包之后，jetty服务器就会生效，这就是切换web服务器的原理。
## 8.3web服务器优化
通过以下源码得知，web服务器的相关配置和ServerProperties有关系：
<img src="images/121.png" width="100%">
查看ServerProperties源码：
<img src="images/122.png" width="100%">
得知web服务器的配置都是以server开头的。

那么如果要配置tomcat服务器怎么办？要配置jetty服务器怎么办？请看一下源码
<img src="images/123.png" width="100%">

<img src="images/124.png" width="100%">
通过以上源码得知，如果要对tomcat服务器进行配置，前缀为：server.tomcat
如果要对jetty服务器进行配置，前缀为：server.jetty。

在以后的开发中关于tomcat服务器的常见优化配置有：
```
# 这个参数决定了 Tomcat 在接收请求时，如果在指定的时间内没有收到完整的请求数据，将会关闭连接。这个超时时间是从客户端发送请求开始计算的。
# 防止长时间占用资源：如果客户端发送请求后，长时间没有发送完所有数据，Tomcat 会在这个超时时间内关闭连接，从而释放资源，避免资源被长时间占用。
server.tomcat.connection-timeout=20000

# 设置 Tomcat 服务器处理请求的最大线程数为 200。
# 如果超过这个数量的请求同时到达，Tomcat 会将多余的请求放入一个等待队列中。
# 如果等待队列也满了（由 server.tomcat.accept-count 配置），新的请求将被拒绝，通常会返回一个“503 Service Unavailable”错误。
server.tomcat.max-threads=200

# 用来设置等待队列的最大容量
server.tomcat.accept-count=100

# 设置 Tomcat 服务器在空闲时至少保持 10 个线程处于活动状态，以便快速响应新的请求。
server.tomcat.min-spare-threads=10

# 允许 Tomcat 服务器在关闭后立即重新绑定相同的地址和端口，即使该端口还在 TIME_WAIT 状态
# 当一个网络连接关闭时，操作系统会将该连接的端口保持在 TIME_WAIT 状态一段时间（通常是 2-4 分钟），以确保所有未完成的数据包都能被正确处理。在这段时间内，该端口不能被其他进程绑定。
server.tomcat.address-reuse-enabled=true

# 设置 Tomcat 服务器绑定到所有可用的网络接口，使其可以从任何网络地址访问。
server.tomcat.bind-address=0.0.0.0

# 设置 Tomcat 服务器使用 HTTP/1.1 协议处理请求。
server.tomcat.protocol=HTTP/1.1

# 设置 Tomcat 服务器的会话(session)超时时间为 30 分钟。具体来说，如果用户在 30 分钟内没有与应用进行任何交互，其会话将被自动注销。
server.tomcat.session-timeout=30

# 设置 Tomcat 服务器的静态资源缓存时间为 3600 秒（即 1 小时），这意味着浏览器会在 1 小时内缓存这些静态资源，减少重复请求。
server.tomcat.resource-cache-period=3600

# 解决get请求乱码。对请求行url进行编码。
server.tomcat.uri-encoding=UTF-8

# 设置 Tomcat 服务器的基础目录为当前工作目录（. 表示当前目录）。这个配置指定了 Tomcat 服务器的工作目录，包括日志文件、临时文件和其他运行时生成的文件的存放位置。 生产环境中可能需要重新配置。
server.tomcat.basedir=.
```