
# 1. 标准目录结构

```
project
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   └── resources
│   └── test
│       ├── java
│       └── resources
└── target
```

- `src/main/java`：主代码
- `src/main/resources`：主资源
- `src/test/java`：测试代码
- `src/test/resources`：测试资源
- `target`：构建输出目录

## 1.1 Web 项目布局（WAR）

```
project
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   ├── resources
│   │   └── webapp
│   │       ├── WEB-INF
│   │       │   ├── web.xml
│   │       │   └── classes (构建后)
│   │       ├── static (CSS/JS/图片)
│   │       └── index.jsp
│   └── test
│       ├── java
│       └── resources
└── target
    └── project.war
```

- `src/main/webapp`：Web 应用根目录。
- `WEB-INF/web.xml`：Servlet 部署描述符（Spring Boot 可用注解替代）。
- `WEB-INF/classes`：编译后的类和资源自动放入此目录。

## 1.2 META-INF 目录

- 位于 `src/main/resources/META-INF/` 或由插件自动生成。
- 常见文件：
	- `MANIFEST.MF`：JAR 清单文件（Main-Class、Class-Path 等）。
	- `spring.factories` / `spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`：Spring Boot 自动配置声明。
	- `services/`：Java SPI 服务声明。
	- `beans.xml`：Spring XML 配置（遗留）。

```xml
<!-- 自定义 MANIFEST.MF 内容 -->
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-jar-plugin</artifactId>
	<version>3.3.0</version>
	<configuration>
		<archive>
			<manifest>
				<addClasspath>true</addClasspath>
				<mainClass>com.example.Main</mainClass>
			</manifest>
		</archive>
	</configuration>
</plugin>
```

## 1.3 多资源目录

- 可以声明多个 `<resource>` 来自不同目录：

```xml
<build>
	<resources>
		<resource>
			<directory>src/main/resources</directory>
		</resource>
		<resource>
			<directory>src/main/conf</directory>
			<filtering>true</filtering>
			<includes>
				<include>**/*.properties</include>
				<include>**/*.xml</include>
			</includes>
		</resource>
	</resources>
</build>
```

- `includes` / `excludes` 用于精确控制哪些文件被复制。
- 常用 exclude：`**/*.java`（排除源码）、`**/.gitkeep`。

# 2. 资源过滤与占位符

- 资源过滤允许在资源文件中使用占位符。
- 默认占位符为 `${}`，易与 Spring 配置冲突。

启用过滤：
```xml
<build>
	<resources>
		<resource>
			<directory>src/main/resources</directory>
			<filtering>true</filtering>
		</resource>
	</resources>
</build>
```

资源文件示例：
```
app.name=demo
app.version=${project.version}
```

避免与 Spring `${}` 冲突的写法：
```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-resources-plugin</artifactId>
	<version>3.3.1</version>
	<configuration>
		<useDefaultDelimiters>false</useDefaultDelimiters>
		<delimiters>
			<delimiter>@</delimiter>
		</delimiters>
	</configuration>
</plugin>
```

# 3. 测试资源

- `src/test/resources` 中的文件只在测试阶段可见。
- 适合存放测试数据、模拟配置、SQL 脚本。

# 4. 编码与行尾规范

- 企业项目建议统一 `UTF-8` 编码。
- 统一行尾（LF/CRLF）可减少跨平台问题。

常用设置：
```xml
<properties>
	<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
</properties>
```

# 5. 多环境配置管理

- 以 Profile 区分环境配置。
- 常见文件：
	- `application-dev.properties`
	- `application-test.properties`
	- `application-prod.properties`

示例：
```xml
<profiles>
	<profile>
		<id>dev</id>
		<properties>
			<env.name>dev</env.name>
		</properties>
	</profile>
</profiles>
```

# 6. 静态资源与打包

- Web 项目常用静态资源目录：`src/main/resources/static`。
- 打包时资源会进入 `target/classes`。

# 7. 构建输出与清理

- 所有构建产物默认生成到 `target`。
- 使用 `mvn clean` 清理，避免旧文件干扰构建。

# 8. 常见问题排查清单

- 资源未打包：检查资源目录是否在 `build/resources` 中声明。
- 资源过滤无效：确认 `filtering` 是否开启。
- 资源被错误替换：更换占位符分隔符，避免与 Spring `${}` 冲突。
- 编码乱码：检查 `project.build.sourceEncoding` 与编辑器编码。
- 测试资源找不到：确认文件放在 `src/test/resources`。

# 9. 实战案例：多环境配置打包

目标：根据不同 profile 打包不同配置文件。

配置：
```xml
<profiles>
	<profile>
		<id>dev</id>
		<properties>
			<env>dev</env>
		</properties>
	</profile>
	<profile>
		<id>prod</id>
		<properties>
			<env>prod</env>
		</properties>
	</profile>
</profiles>

<build>
	<resources>
		<resource>
			<directory>src/main/resources</directory>
			<filtering>true</filtering>
		</resource>
	</resources>
</build>
```

执行：
```bash
mvn -Pdev clean package
```
