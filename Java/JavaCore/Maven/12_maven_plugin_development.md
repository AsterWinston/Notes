
# 1. 什么是 MOJO

- MOJO = Maven plain Old Java Object，即 Maven 插件的实现类。
- 每个插件目标（goal）对应一个 MOJO 类。
- 继承 `AbstractMojo`，使用注解声明元信息。

# 2. 开发第一个插件

## 2.1 POM 配置

```xml
<project>
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.example</groupId>
	<artifactId>hello-maven-plugin</artifactId>
	<version>1.0.0</version>
	<packaging>maven-plugin</packaging>

	<dependencies>
		<dependency>
			<groupId>org.apache.maven</groupId>
			<artifactId>maven-plugin-api</artifactId>
			<version>3.9.6</version>
			<scope>provided</scope>
		</dependency>
		<dependency>
			<groupId>org.apache.maven.plugin-tools</groupId>
			<artifactId>maven-plugin-annotations</artifactId>
			<version>3.11.0</version>
			<scope>provided</scope>
		</dependency>
	</dependencies>
</project>
```

## 2.2 MOJO 实现

```java
package com.example;

import org.apache.maven.plugin.AbstractMojo;
import org.apache.maven.plugin.MojoExecutionException;
import org.apache.maven.plugins.annotations.Mojo;
import org.apache.maven.plugins.annotations.Parameter;

@Mojo(name = "hello")
public class HelloMojo extends AbstractMojo {

	@Parameter(property = "message", defaultValue = "Hello, Maven!")
	private String message;

	@Parameter(property = "skip", defaultValue = "false")
	private boolean skip;

	public void execute() throws MojoExecutionException {
		if (skip) {
			getLog().info("Skipped.");
			return;
		}
		getLog().info("==========================================");
		getLog().info(message);
		getLog().info("==========================================");
	}
}
```

## 2.3 安装与使用

```bash
# 安装到本地仓库
mvn clean install

# 在其他项目中使用
mvn com.example:hello-maven-plugin:1.0.0:hello
mvn com.example:hello-maven-plugin:1.0.0:hello -Dmessage="Hello from CLI"
```

# 3. 常用注解

| 注解 | 作用 |
|------|------|
| `@Mojo` | 声明 MOJO 类，`name` 定义目标名 |
| `@Parameter` | 声明可配置参数，支持 POM `<configuration>` 与命令行 `-D` |
| `@Component` | 注入 Maven 组件（如 `MavenProject`、`MavenSession`） |
| `@Execute` | 在执行当前 MOJO 前先执行另一个生命周期阶段 |
| `@ResolutionRequired` | 声明需要解析的依赖范围 |

# 4. 注入 Maven 上下文

```java
@Mojo(name = "info")
public class InfoMojo extends AbstractMojo {

	@Parameter(defaultValue = "${project}", readonly = true)
	private MavenProject project;

	@Parameter(defaultValue = "${session}", readonly = true)
	private MavenSession session;

	public void execute() throws MojoExecutionException {
		getLog().info("Project: " + project.getName());
		getLog().info("GroupId: " + project.getGroupId());
		getLog().info("ArtifactId: " + project.getArtifactId());
		getLog().info("Version: " + project.getVersion());
		getLog().info("Basedir: " + project.getBasedir());
	}
}
```

# 5. 绑定到生命周期阶段

```java
@Mojo(name = "validate-config", defaultPhase = LifecyclePhase.VALIDATE)
public class ValidateConfigMojo extends AbstractMojo {

	@Parameter(defaultValue = "${project.basedir}/src/main/resources/application.yml")
	private File configFile;

	public void execute() throws MojoExecutionException {
		if (!configFile.exists()) {
			throw new MojoExecutionException("Config file not found: " + configFile);
		}
		getLog().info("Config file check passed.");
	}
}
```

- `defaultPhase`：指定默认绑定的生命周期阶段（用户 POM 中无需再配置 `<executions>`）。

# 6. 多目标插件

一个插件可以包含多个 MOJO（多个目标）：

```
hello-maven-plugin
├── HelloMojo.java          (goal: hello)
├── InfoMojo.java            (goal: info)
└── ValidateConfigMojo.java  (goal: validate-config)
```

# 7. 参数类型支持

```java
// 简单类型
@Parameter(property = "count", defaultValue = "10")
private int count;

// 数组
@Parameter(property = "includes")
private String[] includes;

// 集合
@Parameter(property = "excludes")
private List<String> excludes;

// 复杂对象（POM 嵌套配置）
@Parameter
private ServerConfig server;

// properties 映射
@Parameter
private Properties settings;
```

# 8. 发布到私服

```xml
<distributionManagement>
	<repository>
		<id>company-releases</id>
		<url>https://repo.company.com/releases</url>
	</repository>
</distributionManagement>
```

```bash
mvn -B -DskipTests deploy
```

其他项目引用时声明依赖并配置 execution 即可。

# 9. 插件测试

- 使用 `maven-plugin-testing-harness` 进行单元测试。
- 使用 `maven-invoker-plugin` 进行集成测试（在真实 Maven 构建中验证插件）。

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-invoker-plugin</artifactId>
	<version>3.6.0</version>
	<configuration>
		<projectsDirectory>src/it</projectsDirectory>
		<cloneProjectsTo>${project.build.directory}/it</cloneProjectsTo>
	</configuration>
</plugin>
```

# 10. 常见问题排查清单

- 插件目标不可见：检查 MOJO 类是否有 `@Mojo` 注解，且 `packaging` 是否为 `maven-plugin`。
- 参数不生效：确认 `@Parameter` 的 `property` 属性与 `-D` 参数一致。
- 注入失败：确认 `@Parameter(defaultValue = "${project}")` 表达式拼写正确。
- 依赖缺失：插件依赖必须自身声明（不会从项目传递）。
