# 1. logo设置

## 1.1 关闭logo图标

### 1.1.1 配置方式
spring.main.banner-mode=off
### 1.1.2 代码方式
第一种代码：
```
@SpringBootApplication
public class Springboot322WebServerApplication {
    public static void main(String[] args) {
        SpringApplication springApplication = new SpringApplication(Springboot322WebServerApplication.class);
        springApplication.setBannerMode(Banner.Mode.OFF);
        springApplication.run(args);
    }
}
```
第二种代码：流式编程/链式编程
```
new SpringApplicationBuilder()
                .sources(Springboot322WebServerApplication.class)
                .bannerMode(Banner.Mode.OFF)
                .run(args);
```
## 1.2 修改logo图标
在src/main/resources目录下存放一个banner.txt文件。文件名固定。
利用一些网站生成图标：
https://www.bootschool.net/ascii （支持中文、英文）
http://patorjk.com/software/taag/ （只支持英文）
https://www.degraeve.com/img2txt.php （只支持图片）
获取图标粘贴到banner.txt文件中运行程序即可。

# 2. PageHelper整合
官网地址：https://pagehelper.github.io/

## 2.1 引入依赖
```
<dependency>
    <groupId>com.github.pagehelper</groupId>

    <artifactId>pagehelper-spring-boot-starter</artifactId>

    <version>2.1.0</version>

</dependency>

```
## 2.2 编写代码
```
@RestController
public class VipController {
    @Autowired
    private VipService vipService;
    
    @GetMapping("/list/{pageNo}")
    public PageInfo<Vip> list(@PathVariable("pageNo") Integer pageNo) {
        // 1.设置当前页码和每页显示的记录条数
        PageHelper.startPage(pageNo, Constant.PAGE_SIZE);
        // 2.获取数据（PageHelper会自动给SQL语句添加limit）
        List<Vip> vips = vipService.findAll();
        // 3.将分页数据封装到PageInfo
        PageInfo<Vip> vipPageInfo = new PageInfo<>(vips);
        return vipPageInfo;
    }
}
```

# 3. web层响应结果封装

对于前后端分离的系统来说，为了降低沟通成本，我们有必要给前端系统开发人员返回统一格式的JSON数据。多数开发团队一般都会封装一个`R`对象来解决统一响应格式的问题。

## 3.1 封装R对象
```
@NoArgsConstructor
@AllArgsConstructor
@Data
@Builder
public class R<T> {

    private int code; // 响应的状态码
    private String msg; // 响应的消息
    private T data; // 响应的数据体

    // 用于构建成功的响应，不携带数据
    public static <T> R<T> OK() {
        return R.<T>builder()
                .code(200)
                .msg("成功")
                .build();
    }

    // 用于构建成功的响应，携带数据
    public static <T> R<T> OK(T data) {
        return R.<T>builder()
                .code(200)
                .msg("成功")
                .data(data)
                .build();
    }

    // 用于构建成功的响应，自定义消息，不携带数据
    public static <T> R<T> OK(String msg) {
        return R.<T>builder()
                .code(200)
                .msg(msg)
                .build();
    }

    // 用于构建成功的响应，自定义消息，携带数据
    public static <T> R<T> OK(String msg, T data) {
        return R.<T>builder()
                .code(200)
                .msg(msg)
                .data(data)
                .build();
    }

    // 用于构建失败的响应，不带任何参数，默认状态码为400，消息为"失败"
    public static <T> R<T> FAIL() {
        return R.<T>builder()
                .code(400)
                .msg("失败")
                .build();
    }

    // 用于构建失败的响应，自定义状态码和消息
    public static <T> R<T> FAIL(int code, String msg) {
        return R.<T>builder()
                .code(code)
                .msg(msg)
                .build();
    }
}

```

## 3.2 改进R对象
以上R对象存在的问题是，难以维护，项目中可能会出现很多这样的代码：R.FAIL(400, "修改失败")。

引入枚举类型进行改进：
```
@NoArgsConstructor
@AllArgsConstructor
public enum CodeEnum {

    OK(200, "成功"),
    FAIL(400, "失败"),
    BAD_REQUEST(400, "请求错误"),
    NOT_FOUND(404, "未找到资源"),
    INTERNAL_ERROR(500, "内部服务器错误"),
    MODIFICATION_FAILED(400, "修改失败"),
    DELETION_FAILED(400, "删除失败"),
    CREATION_FAILED(400, "创建失败");

    @Getter
    @Setter
    private int code;
    @Getter
    @Setter
    private String msg;

}
```
改进R：
```
@NoArgsConstructor
@AllArgsConstructor
@Data
@Builder
public class R<T> {

    private int code; // 响应的状态码
    private String msg; // 响应的消息
    private T data; // 响应的数据体

    // 用于构建成功的响应，不携带数据
    public static <T> R<T> OK() {
        return R.<T>builder()
                .code(CodeEnum.OK.getCode())
                .msg(CodeEnum.OK.getMsg())
                .build();
    }

    // 用于构建成功的响应，携带数据
    public static <T> R<T> OK(T data) {
        return R.<T>builder()
                .code(CodeEnum.OK.getCode())
                .msg(CodeEnum.OK.getMsg())
                .data(data)
                .build();
    }

    // 用于构建失败的响应，不带任何参数，默认状态码为400，消息为"失败"
    public static <T> R<T> FAIL() {
        return R.<T>builder()
                .code(CodeEnum.FAIL.getCode())
                .msg(CodeEnum.FAIL.getMsg())
                .build();
    }

    // 用于构建失败的响应，自定义状态码和消息
    public static <T> R<T> FAIL(CodeEnum codeEnum) {
        return R.<T>builder()
                .code(codeEnum.getCode())
                .msg(codeEnum.getMsg())
                .build();
    }
}

```

# 4. 事务管理
SpringBoot中的事务管理仍然使用的Spring框架中的事务管理机制，在代码实现上更为简单了。不需要手动配置事务管理器，SpringBoot自动配置完成了。我们只需要使用@Transactional注解标注需要控制事务的方法即可。另外事务的特性等仍然延用Spring框架。大家可以在老杜发布的Spring视频教程中详细学习事务管理机制。以下代码是在SpringBoot框架中进行的事务控制：
```
@Transactional(rollbackFor = Exception.class, propagation = Propagation.REQUIRED)
@Service
public class AccountServiceImpl implements AccountService {

    @Autowired
    private AccountMapper accountMapper;

    @Override
    public void transfer(String fromActNo, String toActNo, double money) {
        Account fromAct = accountMapper.selectByActNo(fromActNo);
        if(fromAct.getBalance() < money){
            throw new TransferException("余额不足");
        }
        Account toAct = accountMapper.selectByActNo(toActNo);
        fromAct.setBalance(fromAct.getBalance() - money);
        toAct.setBalance(toAct.getBalance() + money);
        int count = accountMapper.update(fromAct);
        if(1 == 1){
            throw new TransferException("转账失败");
        }
        count += accountMapper.update(toAct);
        if(count != 2){
            throw new TransferException("转账失败！");
        }
    }
}
```
我们只需要在需要控制事务的方法上，或者类上，使用@Transactional注解进行标注即可。然后事务的特性和之前Spring中是完全相同的。最重要的是其他的配置我们一律是不需要的。


# 5. SpringBoot打war包
第一步：将打包方式设置为war
```
<packaging>war</packaging>

```
第二步：排除内嵌tomcat
```
<dependency>
    <groupId>org.springframework.boot</groupId>

    <artifactId>spring-boot-starter-web</artifactId>

    <!--内嵌的tomcat服务器排除掉-->
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>

            <artifactId>spring-boot-starter-tomcat</artifactId>

        </exclusion>

    </exclusions>

</dependency>

```
第三步：添加servlet api依赖（引入tomcat，但scope设置为provided，这样这个tomcat服务器就不会打入war包了）
```
<!--额外添加一个tomcat服务器，实际上是为了添加servlet api。scope设置为provided表示这个不会被打入war包当中。-->
<dependency>
    <groupId>org.springframework.boot</groupId>

    <artifactId>spring-boot-starter-tomcat</artifactId>

    <scope>provided</scope>

</dependency>

```
第四步：修改主类
```
@MapperScan(basePackages = "com.powernode.transaction.repository")
@SpringBootApplication
public class Springboot324TransactionApplication extends SpringBootServletInitializer{

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(Springboot324TransactionApplication.class);
    }

    public static void main(String[] args) {
        SpringApplication.run(Springboot324TransactionApplication.class, args);
    }

}
```
第五步：执行package命令打war包
第六步：配置tomcat环境，将war包放入到webapps目录下，启动tomcat服务器，并访问。

# 6. SpringBoot的日志处理

## 6.1 日志概述
日志记录是一种关键的技术实践，用于捕捉软件运行时的信息、警告、错误以及其他重要数据。通过分析日志文件，开发人员、测试工程师和运维团队可以深入了解应用程序的行为，快速定位并解决出现的问题，同时对系统性能进行监控与优化。
虽然在开发初期或简单的应用场景下，使用 System.out.println("记录日志...") 这样的方式来输出信息非常直接且易于实现，但在生产环境中的大型或复杂应用中，这种方式存在诸多不足：
1. 性能问题：频繁地调用 System.out.println() 会占用额外的 CPU 资源，尤其是在高并发场景下，可能会影响应用程序的性能。
2. 日志管理不便：控制台输出的日志不易于长期保存和检索，也不便于与其他工具集成以进行更高级的日志分析。
3. 缺乏灵活性：通过 System.out.println() 输出的信息通常难以调整其格式、级别（如 DEBUG, INFO, WARN, ERROR）或输出目标（例如文件、数据库、远程服务器等）。
4. 安全性和隐私：在某些情况下，直接打印敏感信息可能会泄露用户的个人数据或公司的商业秘密。

因此，在实际的线上项目中，推荐采用专业的日志框架，它们提供了更为强大和灵活的日志处理能力：
- 支持多种日志级别，允许开发者根据需要过滤日志信息。
- 可配置的日志输出格式，使日志更加规范易读。
- 支持将日志写入不同的目的地，如文件、数据库、网络服务等。
- 提供异步日志功能，减少日志记录对应用性能的影响。
- 能够与各种监控系统集成，支持实时告警和问题追踪。

## 6.2 Java中的日志框架

java语言中存在多种日志框架，每种框架都有其特点和适用场景

### 6.2.1 抽象的日志框架
什么是抽象的日志框架？编译阶段可以使用这些抽象的日志框架，能够正常编译通过，但运行阶段必须提供具体的日志框架，这样做的目的是：具体的日志框架可灵活切换。
1. SLF4J (Simple Logging Facade for Java)
- 简介：SLF4J 不是一个具体的日志实现，而是一个简单的抽象层，允许开发者在应用程序中使用统一的日志 API，而实际的日志实现可以在部署时决定。
- 特点：提供了统一的日志记录接口，使得日志框架的选择更加灵活，易于切换不同的日志实现，减少了应用程序对特定日志框架的依赖。
- 运行时可绑定的具体日志框架：Log4j、Log4j2、Logback、JUL。
2. Commons Logging (Jakarta Commons Logging)
- 简介：Commons Logging 是 Apache Jakarta 项目的一部分，也是一种日志层面的抽象。
- 特点：允许在运行时动态选择日志实现，但因类加载机制的问题，有时会导致难以预料的行为，因此不如 SLF4J 受欢迎。
- 运行时可绑定的具体日志框架：Log4j、Log4j2、Logback、JUL。

### 6.2.2 具体的日志框架
1. Log4j（太老，没人用了）
- 简介：Log4j 是 Apache 软件基金会的一个开源项目，它提供了强大的日志记录功能，支持多种日志级别（如 DEBUG, INFO, WARN, ERROR, FATAL）、灵活的日志输出格式和目的地配置。
- 特点：易于配置和使用，支持日志滚动策略，适用于中小规模的应用。
2. Log4j 2
- 简介：Log4j 2 是 Log4j 的下一代版本，旨在解决原版的一些性能瓶颈和设计缺陷。
- 特点：性能更优，支持插件化配置，增强了安全性，支持异步日志记录，减少了线程间的竞争，提高了日志记录的速度。
3. Logback
- 简介：Logback 是 Log4j 的创始人 Ceki Gülcü 开发的另一个日志框架，设计之初就考虑到了性能和可靠性。
- 特点：与 SLF4J 无缝集成，提供了比 Log4j 更好的性能表现，支持自动重载配置文件，具有更简洁的 API 和更少的 bug。
4. JUL (Java Util Logging)
- 简介：JUL 是 Java SE 平台自带的日志框架，从 Java 1.4 版本开始提供。
- 特点：不需要额外的库文件，配置相对简单，适合小型应用。然而，相对于其他流行的日志框架，JUL 的功能较为有限，配置不够灵活。
这些日志框架各有优势，选择哪一种取决于项目的具体需求、团队的偏好以及现有的技术栈。例如，对于需要高性能和灵活配置的应用，Logback 或 Log4j 2 可能是更好的选择；而对于希望减少对外部库依赖的小型项目，JUL 可能更为合适。
**SpringBoot 框架默认使用的日志框架是：Logback。**

## 6.3 日志级别概述
在 Java 的日志框架中，日志级别（Log Level）是用来指定日志记录的严重程度的一个标准。不同的日志级别代表了不同的重要性和紧急程度，这有助于开发者根据实际需要来过滤和关注特定类型的日志信息。
常见的日志级别从低到高通常包括以下几种：
1. TRACE：这是最低级别的日志信息，用于记录最详细的信息，通常是在调试应用时使用。
2. DEBUG：用于记录程序运行时的详细信息，比如变量的值、进入或退出某个方法等，主要用于开发阶段的调试。
3. INFO：记录应用程序的一般信息，如系统启动、服务初始化完成等，表示程序运行正常。
4. WARN：警告信息，表示可能存在问题或者异常的情况，但是还不至于导致应用停止工作。
5. ERROR：错误信息，表示发生了一个错误事件，该错误可能会导致某些功能无法正常工作。

通过设置不同的日志级别，可以在生产环境中控制输出的日志量，从而减少对性能的影响。例如：
- 在生产环境中，我们可能会将日志级别设置为 INFO 或更高级别，以避免输出过多的调试信息；
- 而在开发或测试环境中，则可以将日志级别降低至 DEBUG 甚至 TRACE，以便于进行详细的错误追踪和调试。

日志级别从低到高的排序是：TRACE < DEBUG < INFO < WARN < ERROR
级别越低，打印的日志信息越多。SpringBoot 默认的日志级别是INFO。

- 开发环境下可以打印多一些，一般是用INFO级别。
- 生产环境下要严格控制，因为日志级别越低，磁盘空间受不了。
SpringBoot 默认将日志打印到控制台。（通过配置可以改变打印到日志文件）

## 6.4 更改日志级别
配置文件中：logging.level.root=...配置级别（全局生效）

## 6.5 丰富启动日志
```
trace=true
debug=true
```
这两个配置项主要影响 Spring Boot 框架本身的启动日志和跟踪信息，不影响你在应用程序中编写的日志记录。
- debug=true：启用 Spring Boot 的调试模式，增加启动日志的详细程度并显示自动配置报告。
- trace=true：启用 Spring Boot 的启动跟踪，生成包含启动信息的详细跟踪文件，内容是包含debug=true输出的内容的。
并且以上两个配置在 SpringBoot 框架启动完成之后，不再有任何作用。

## 6.6 日志的粗细粒度

```
# 设置根日志级别为 INFO
logging.level.root=INFO

# 为特定的包设置日志级别为 DEBUG
logging.level.com.powernode=DEBUG

# 为特定的类设置日志级别为 TRACE
logging.level.com.powernode.MyService=TRACE
```

## 6.7 日志输出到文件
如果日志要输出文件，可以进行如下配置：
```
logging.file.path=./log/
```
以上配置表示将日志文件输出到当前工程根目录下的 log 目录当中，文件名默认叫做 spring.log，并且文件名不可设置。
```
logging.file.name=my.log
```
以上的配置表示在工程的根目录下直接生成 my.log 文件，并且路径不可设置。
注意：以上两个配置同时存在时，只有 logging.file.name 生效，本质上两个配置不可共存，这是 Spring Boot 框架的一个小 bug。

## 6.8 滚动日志
滚动日志是一种日志管理机制，用于防止日志文件无限增长，通过将日志文件分割成多个文件，每个文件只包含一定时间段或大小的日志记录。滚动日志可以帮助你更好地管理和维护日志文件，避免单个日志文件过大导致难以处理。
```
# 日志文件达到多大时进行归档
logging.logback.rollingpolicy.max-file-size=100MB

# 所有的归档日志文件总共达到多大时进行删除，默认是0B表示不删除
logging.logback.rollingpolicy.total-size-cap=50GB

# 归档日志文件最多保留几天
logging.logback.rollingpolicy.max-history=60

# 启动项目时是否要清理归档日志文件
logging.logback.rollingpolicy.clean-history-on-start=false

# 归档日志文件名的格式
logging.logback.rollingpolicy.file-name-pattern=${LOG_FILE}.%d{yyyy-MM-dd}.%i.gz
```

## 6.9 日志框架切换
在实际开发中也可能会切换日志框架，按照如下配置进行切换：先排除掉，再添加
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <!-- 排除掉默认的日志依赖 -->
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- 使用Log4j2日志依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

注意：一旦切换到 Log4j2 的日志框架，对于之前的配置来说，只有 logback 的滚动日志不再生效，其他配置仍然生效。如果要在 log4j2 中进行滚动日志，需要编写 log4j2 相关的 xml 配置文件，比较麻烦。

# 7. CommandLineRunner 与 ApplicationRunner

Spring Boot 提供了两种方式在应用启动后执行初始化代码。

## 7.1 CommandLineRunner
```java
@Component
@Order(1)  // 数字越小越优先执行
public class MyCommandLineRunner implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {
        System.out.println("=== CommandLineRunner 执行 ===");
        // args 是原始的命令行参数
        for (String arg : args) {
            System.out.println("参数: " + arg);
        }
        // 初始化缓存、加载数据等操作
    }
}
```

## 7.2 ApplicationRunner
```java
@Component
@Order(2)
public class MyApplicationRunner implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("=== ApplicationRunner 执行 ===");
        // args 是经过解析的参数（支持 --key=value 格式）
        Set<String> optionNames = args.getOptionNames();
        for (String name : optionNames) {
            System.out.println(name + " = " + args.getOptionValues(name));
        }
        List<String> nonOptionArgs = args.getNonOptionArgs();
        System.out.println("非选项参数: " + nonOptionArgs);
    }
}
```

## 7.3 两者的区别

| 对比项 | CommandLineRunner | ApplicationRunner |
|--------|------------------|-------------------|
| 参数类型 | `String... args` 原始字符串数组 | `ApplicationArguments` 解析后的参数对象 |
| 参数解析 | 手动解析 | 自动解析 `--key=value` 格式 |
| 使用场景 | 简单场景 | 需要解析命名参数时 |

示例--不同参数格式：
```
# 启动参数
java -jar app.jar hello world --server.port=9090 --env=dev

# CommandLineRunner 收到的 args：
["hello", "world", "--server.port=9090", "--env=dev"]

# ApplicationRunner 收到的：
optionNames: ["server.port", "env"]
optionValues: {"server.port": ["9090"], "env": ["dev"]}
nonOptionArgs: ["hello", "world"]
```

# 8. 应用事件与监听器

Spring Boot在启动过程中会发布一系列事件，可以监听这些事件来执行自定义逻辑。

## 8.1 常用应用事件
| 事件 | 触发时机 |
|------|---------|
| ApplicationStartingEvent | 应用刚开始启动时（最早，此时Spring容器尚未创建） |
| ApplicationEnvironmentPreparedEvent | Environment准备完成后 |
| ApplicationContextInitializedEvent | ApplicationContext初始化完成后 |
| ApplicationPreparedEvent | ApplicationContext准备好、但尚未刷新时 |
| ApplicationStartedEvent | ApplicationContext刷新后、**CommandLineRunner/ApplicationRunner执行前** |
| ApplicationReadyEvent | **应用完全启动后**（CommandLineRunner执行完毕） |
| ApplicationFailedEvent | 启动过程中发生异常时 |

## 8.2 监听事件
```java
@Component
public class MyApplicationListener {

    @EventListener
    public void onReady(ApplicationReadyEvent event) {
        System.out.println("==================================");
        System.out.println("应用启动完成！耗时：" +
                event.getTimeTaken().toMillis() + "ms");
        System.out.println("==================================");
    }

    @EventListener
    public void onFailed(ApplicationFailedEvent event) {
        System.out.println("应用启动失败：" + event.getException().getMessage());
    }
}
```

## 8.3 事件监听的应用场景
- **ApplicationReadyEvent**：应用启动后预加载数据、检查第三方服务状态
- **ApplicationFailedEvent**：启动失败时发送告警通知
- **ApplicationEnvironmentPreparedEvent**：根据环境变量动态调整配置

# 9. 优雅关机（Graceful Shutdown）

Spring Boot 3.x 默认支持优雅关机，当收到关闭信号时，会先拒绝新请求，等待正在处理的请求完成后再关闭。

```properties
# 启用优雅关机（默认已启用在3.x中）
server.shutdown=graceful
# 等待正在处理的请求完成的超时时间
spring.lifecycle.timeout-per-shutdown-phase=30s
```

# 10. 邮件发送

Spring Boot 对邮件发送提供了自动配置。

## 10.1 引入依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

## 10.2 配置邮箱
```properties
# QQ邮箱示例
spring.mail.host=smtp.qq.com
spring.mail.port=587
spring.mail.username=your-email@qq.com
spring.mail.password=your-auth-code  # 不是邮箱密码，是授权码
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```

## 10.3 发送邮件
```java
@Service
public class EmailService {

    @Autowired
    private JavaMailSender mailSender;

    // 发送简单邮件
    public void sendSimpleMail(String to, String subject, String content) {
        SimpleMailMessage message = new SimpleMailMessage();
        message.setFrom("your-email@qq.com");
        message.setTo(to);
        message.setSubject(subject);
        message.setText(content);
        mailSender.send(message);
    }

    // 发送HTML邮件
    public void sendHtmlMail(String to, String subject, String htmlContent) throws MessagingException {
        MimeMessage message = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
        helper.setFrom("your-email@qq.com");
        helper.setTo(to);
        helper.setSubject(subject);
        helper.setText(htmlContent, true);  // true表示HTML
        mailSender.send(message);
    }

    // 发送带附件的邮件
    public void sendMailWithAttachment(String to, String subject, String content,
                                        File attachment) throws MessagingException {
        MimeMessage message = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
        helper.setFrom("your-email@qq.com");
        helper.setTo(to);
        helper.setSubject(subject);
        helper.setText(content);
        helper.addAttachment(attachment.getName(), attachment);
        mailSender.send(message);
    }
}
```
