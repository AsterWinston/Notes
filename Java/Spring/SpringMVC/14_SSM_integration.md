# 1.引入相关依赖
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.powernode</groupId>
    <artifactId>ssmtest</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>war</packaging>

    <dependencies>
        <!--springmvc-->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>6.1.4</version>
        </dependency>
        <!--spring jdbc-->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
            <version>6.1.4</version>
        </dependency>
        <!--mybatis-->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>3.5.15</version>
        </dependency>
        <!--mybatis spring-->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis-spring</artifactId>
            <version>3.0.3</version>
        </dependency>
        <!--mysql驱动-->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>8.3.0</version>
        </dependency>
        <!--德鲁伊连接池-->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.2.22</version>
        </dependency>
        <!--jackson-->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.17.0</version>
        </dependency>
        <!--servlet api-->
        <dependency>
            <groupId>jakarta.servlet</groupId>
            <artifactId>jakarta.servlet-api</artifactId>
            <version>6.0.0</version>
            <scope>provided</scope>
        </dependency>
        <!--logback-->
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.5.3</version>
        </dependency>
        <!--thymeleaf和spring6的整合依赖-->
        <dependency>
            <groupId>org.thymeleaf</groupId>
            <artifactId>thymeleaf-spring6</artifactId>
            <version>3.1.2.RELEASE</version>
        </dependency>
    </dependencies>

    <properties>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

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

</project>
```
# 2.SSM整合
## 2.1创建包结构

## 2.2创建webapp目录

## 2.3Spring整合Mybatis
### 2.3.1编写jdbc.properties
```
在类根路径下创建属性配置文件，配置连接数据库的信息：jdbc.properties
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/powernode?useUnicode=true&serverTimezone=Asia/Shanghai&useSSL=true&characterEncoding=utf-8
jdbc.username=root
jdbc.password=1234
```
### 2.3.2编写DataSourceConfig
```
package com.powernode.ssm.config;

import com.alibaba.druid.pool.DruidDataSource;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;

import javax.sql.DataSource;

/**
 * ClassName: DataSourceConfig
 * Description:
 * Datetime: 2024/4/1 14:25
 * Author: 老杜@动力节点
 * Version: 1.0
 */
public class DataSourceConfig {

    @Value("${jdbc.driver}")
    private String driver;

    @Value("${jdbc.url}")
    private String url;

    @Value("${jdbc.username}")
    private String username;

    @Value("${jdbc.password}")
    private String password;

    @Bean
    public DataSource dataSource(){
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setDriverClassName(driver);
        dataSource.setUrl(url);
        dataSource.setUsername(username);
        dataSource.setPassword(password);
        return dataSource;
    }
}

```
### 2.3.3编写MybatisConfig
```
package com.powernode.ssm.config;

import org.mybatis.spring.SqlSessionFactoryBean;
import org.mybatis.spring.mapper.MapperScannerConfigurer;
import org.springframework.context.annotation.Bean;

import javax.sql.DataSource;

/**
 * ClassName: MyBatisConfig
 * Description:
 * Datetime: 2024/4/1 14:25
 * Author: 老杜@动力节点
 * Version: 1.0
 */
public class MyBatisConfig {

    @Bean
    public SqlSessionFactoryBean sqlSessionFactory(DataSource dataSource){
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        sqlSessionFactoryBean.setTypeAliasesPackage("com.powernode.ssm.bean");
        return sqlSessionFactoryBean;
    }

    @Bean
    public MapperScannerConfigurer mapperScannerConfigurer(){
        MapperScannerConfigurer msc = new MapperScannerConfigurer();
        msc.setBasePackage("com.powernode.ssm.dao");
        return msc;
    }

}
```
### 2.3.4编写SpringConfig
```
package com.powernode.ssm.config;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.context.annotation.PropertySource;

/**
 * ClassName: SpringConfig
 * Description:
 * Datetime: 2024/4/1 14:22
 * Author: 老杜@动力节点
 * Version: 1.0
 */
@Configuration
@ComponentScan({"com.powernode.ssm.service"})
@PropertySource("classpath:jdbc.properties")
@Import({DataSourceConfig.class, MyBatisConfig.class})
public class SpringConfig {
}
```
## 2.4Spring整合Spring MVC
### 2.4.1编写WebAppInitializer（web.xml）
```
package com.powernode.ssm.config;

import jakarta.servlet.Filter;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.filter.CharacterEncodingFilter;
import org.springframework.web.filter.HiddenHttpMethodFilter;
import org.springframework.web.servlet.support.AbstractAnnotationConfigDispatcherServletInitializer;

/**
 * ClassName: WebAppInitializer
 * Description:
 * Datetime: 2024/4/1 14:59
 * Author: 老杜@动力节点
 * Version: 1.0
 */
public class WebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    /**
     * Spring的配置
     * @return
     */
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{SpringConfig.class};
    }

    /**
     * SpringMVC的配置
     * @return
     */
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{SpringMvcConfig.class};
    }

    /**
     * 用来配置DispatcherServlet的 <url-pattern>
     * @return
     */
    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }

    /**
     * 配置过滤器
     * @return
     */
    @Override
    protected Filter[] getServletFilters() {
        // 配置字符编码过滤器
        CharacterEncodingFilter characterEncodingFilter = new CharacterEncodingFilter();
        characterEncodingFilter.setEncoding("UTF-8");
        characterEncodingFilter.setForceResponseEncoding(true);
        characterEncodingFilter.setForceRequestEncoding(true);
        // 配置HiddenHttpMethodFilter
        HiddenHttpMethodFilter hiddenHttpMethodFilter = new HiddenHttpMethodFilter();
        return new Filter[]{characterEncodingFilter, hiddenHttpMethodFilter};
    }
}

```
###  2.4.2编写SpringMVCConfig
```
package com.powernode.ssm.config;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.HandlerExceptionResolver;
import org.springframework.web.servlet.config.annotation.*;
import org.thymeleaf.spring6.SpringTemplateEngine;
import org.thymeleaf.spring6.templateresolver.SpringResourceTemplateResolver;
import org.thymeleaf.spring6.view.ThymeleafViewResolver;
import org.thymeleaf.templatemode.TemplateMode;
import org.thymeleaf.templateresolver.ITemplateResolver;

import java.util.List;

/**
 * ClassName: SpringMvcConfig
 * Description:
 * Datetime: 2024/4/1 15:02
 * Author: 老杜@动力节点
 * Version: 1.0
 */
@Configuration
@ComponentScan("com.powernode.ssm.handler")
@EnableWebMvc
public class SpringMvcConfig implements WebMvcConfigurer {

    // 以下三个方法合并起来就是开启视图解析器
    @Bean
    public ThymeleafViewResolver getViewResolver(SpringTemplateEngine springTemplateEngine) {
        ThymeleafViewResolver resolver = new ThymeleafViewResolver();
        resolver.setTemplateEngine(springTemplateEngine);
        resolver.setCharacterEncoding("UTF-8");
        resolver.setOrder(1);
        return resolver;
    }

    @Bean
    public SpringTemplateEngine templateEngine(ITemplateResolver iTemplateResolver) {
        SpringTemplateEngine templateEngine = new SpringTemplateEngine();
        templateEngine.setTemplateResolver(iTemplateResolver);
        return templateEngine;
    }

    @Bean
    public ITemplateResolver templateResolver(ApplicationContext applicationContext) {
        SpringResourceTemplateResolver resolver = new SpringResourceTemplateResolver();
        resolver.setApplicationContext(applicationContext);
        resolver.setPrefix("/WEB-INF/thymeleaf/");
        resolver.setSuffix(".html");
        resolver.setTemplateMode(TemplateMode.HTML);
        resolver.setCharacterEncoding("UTF-8");
        resolver.setCacheable(false);//开发时关闭缓存，改动即可生效
        return resolver;
    }

    // 开启静态资源处理，开启默认的Servlet处理
    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }

    // 视图控制器
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {}
    // 配置异常处理器
    @Override
    public void configureHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {}

    // 配置拦截器
    @Override
    public void addInterceptors(InterceptorRegistry registry) {}
}

```
## 2.5添加事务控制
```
第一步：在SpringConfig中开启事务管理器
@EnableTransactionManagement
public class SpringConfig {
}

第二步：在DataSourceConfig中添加事务管理器对象
@Bean
public PlatformTransactionManager platformTransactionManager(DataSource dataSource){
    DataSourceTransactionManager dataSourceTransactionManager = new DataSourceTransactionManager();
    dataSourceTransactionManager.setDataSource(dataSource);
    return dataSourceTransactionManager;
}

第三步：在service类上添加如下注解：
@Transactional
public class UserService {}
```

# 3.实现功能测试SSM整合
## 3.1数据库表
```
CREATE DATABASE IF NOT EXISTS powernode DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;

USE powernode;

DROP TABLE IF EXISTS tbl_user;
CREATE TABLE tbl_user (
    id BIGINT NOT NULL AUTO_INCREMENT COMMENT '用户ID',
    name VARCHAR(50) NOT NULL COMMENT '用户名',
    password VARCHAR(100) NOT NULL COMMENT '密码',
    email VARCHAR(100) DEFAULT NULL COMMENT '邮箱',
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';

INSERT INTO tbl_user (name, password, email) VALUES
('张三', '123456', 'zhangsan@powernode.com'),
('李四', '654321', 'lisi@powernode.com'),
('王五', '111111', 'wangwu@powernode.com');
```
## 3.2pojo类
```
package com.powernode.ssm.bean;

/**
 * ClassName: User
 * Description:
 * Datetime: 2024/4/1 15:42
 * Author: 老杜@动力节点
 * Version: 1.0
 */
public class User {
    private Long id;
    private String name;
    private String password;
    private String email;

    @Override
    public String toString() {
        return "User{" +
        "id=" + id +
        ", name='" + name + '\'' +
        ", password='" + password + '\'' +
        ", email='" + email + '\'' +
        '}';
    }

    public User() {
    }

    public User(Long id, String name, String password, String email) {
        this.id = id;
        this.name = name;
        this.password = password;
        this.email = email;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }
}

```
## 3.3dao编写
```
package com.powernode.ssm.dao;

import com.powernode.ssm.bean.User;
import org.apache.ibatis.annotations.Select;

/**
 * ClassName: UserDao
 * Description:
 * Datetime: 2024/4/1 15:43
 * Author: 老杜@动力节点
 * Version: 1.0
 */
public interface UserDao {

    @Select("select * from tbl_user where id = #{id}")
    User selectById(Long id);

}

```
## 3.4service编写
```
package com.powernode.ssm.service;

import com.powernode.ssm.bean.User;

/**
 * ClassName: UserService
 * Description:
 * Datetime: 2024/4/1 15:45
 * Author: 老杜@动力节点
 * Version: 1.0
 */
public interface UserService {

    /**
     * 根据id获取用户信息
     * @param id
     * @return
     */
    User getById(Long id);

}

package com.powernode.ssm.service.impl;

import com.powernode.ssm.bean.User;
import com.powernode.ssm.dao.UserDao;
import com.powernode.ssm.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * ClassName: UserServiceImpl
 * Description:
 * Datetime: 2024/4/1 15:45
 * Author: 老杜@动力节点
 * Version: 1.0
 */
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserDao userDao;

    @Override
    public User getById(Long id) {
        return userDao.selectById(id);
    }
}
```
## 3.5handler编写
```
package com.powernode.ssm.handler;

import com.powernode.ssm.bean.User;
import com.powernode.ssm.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * ClassName: UserHandler
 * Description:
 * Datetime: 2024/4/1 15:46
 * Author: 老杜@动力节点
 * Version: 1.0
 */
@RestController
@RequestMapping("/users")
public class UserHandler {

    @Autowired
    private UserService userService;

    @GetMapping("/{id}")
    public User detail(@PathVariable("id") Long id){
        return userService.getById(id);
    }
}

```
## 3.6前端发送ajax
### 3.6.1引入js文件
```
在webapp目录下创建static/js目录，将vue3和axios的js文件放入该目录：
webapp/
├── static/
│   └── js/
│       ├── vue3.4.21.js
│       └── axios.min.js
├── WEB-INF/
│   └── thymeleaf/
│       └── index.html
└── index.html

Vue3可以从 https://unpkg.com/vue@3.4.21/dist/vue.global.prod.js 下载。
Axios可以从 https://unpkg.com/axios@1.6.7/dist/axios.min.js 下载。
```
### 3.6.2开启静态资源处理
```
SpringMVC的DispatcherServlet默认会拦截所有请求（url-pattern为"/"），但不会处理静态资源（js、css、图片等）。
因此需要在SpringMvcConfig中配置静态资源处理，让SpringMVC将 /static/** 路径下的请求映射到 webapp/static/ 目录。

有两种方式：

方式一：使用默认Servlet处理（适用于所有静态资源）：
@Override
public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
    configurer.enable();
}
开启后，SpringMVC会将无法处理的静态资源请求交给Tomcat的DefaultServlet去处理。

方式二：精确指定静态资源映射（推荐，更灵活）：
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    // 将 /static/** 的URL映射到 /static/ 目录下的实际文件
    registry.addResourceHandler("/static/**")
            .addResourceLocations("/static/");
}

两种方式可以同时配置。方式二更推荐，因为它可以精确控制哪些路径下的文件可以公开访问。
```
### 3.6.3视图控制器
```
public void addViewControllers(ViewControllerRegistry registry) {
    registry.addViewController("/").setViewName("index");
}
```
### 3.6.4编写ajax
```
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>ssm整合</title>
    <!--引入vue-->
    <script th:src="@{/static/js/vue3.4.21.js}"></script>
    <!--引入axios-->
    <script th:src="@{/static/js/axios.min.js}"></script>
</head>
<body>
<div id="app">
    <button @click="getMessage">查看id=1的用户信息</button>
    <h1>{{message}}</h1>
</div>
<script th:inline="javascript">
    Vue.createApp({
        data(){
            return {
                message : ''
            }
        },
        methods : {
            async getMessage(){
                let response = await axios.get([[@{/}]] + 'users/1')
                this.message = response.data
            }
        }
    }).mount("#app")
</script>
</body>
</html>
```