
# 1. POM 文件整体结构

- POM 是 Maven 项目的核心描述文件，定义坐标、依赖、构建与发布规则。
- 常见顶层元素：
	- `modelVersion`：模型版本（固定 4.0.0）
	- `groupId` / `artifactId` / `version` / `packaging`
	- `name` / `description` / `url`
	- `organization` / `licenses` / `developers` / `scm`
	- `dependencies` / `dependencyManagement`
	- `build` / `reporting`
	- `profiles` / `distributionManagement`

最小可用 POM：
```xml
<project>
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.example</groupId>
	<artifactId>order-service</artifactId>
	<version>1.0.0</version>
</project>
```

## 1.1 Super POM（超级 POM）

- 所有 POM 默认继承自 Maven 内置的 Super POM。
- Super POM 预定义了：
	- 标准目录结构（`src/main/java`、`src/test/java` 等）。
	- 中央仓库地址（`https://repo.maven.apache.org/maven2`）。
	- 默认插件绑定（如 `maven-compiler-plugin` 绑定到 `compile` 阶段）。
	- 默认的 `resources` / `testResources` 配置。
- 查看 Super POM（Java 8 之前有效）：
```bash
mvn help:effective-pom
```
- 核心意义：用户 POM 只声明与默认约定不同的部分，实现"约定优于配置"。

# 2. 坐标与构件命名规范

- `groupId`：公司域名反写 + 业务线（如 `com.company.pay`）。
- `artifactId`：模块名，用短横线分割（如 `order-service`）。
- `version`：建议语义化版本，发布版去掉 `SNAPSHOT`。
- `packaging`：`jar`、`war`、`pom` 等。
- `classifier`：区分构件变体（如 `sources`、`javadoc`）。

示例（带 classifier）：
```xml
<dependency>
	<groupId>org.projectlombok</groupId>
	<artifactId>lombok</artifactId>
	<version>1.18.30</version>
	<classifier>sources</classifier>
</dependency>
```

# 3. parent 与继承

- Maven 默认继承 `super POM`（内置默认配置）。
- 通过 `parent` 继承公司统一父 POM，集中管理版本与插件。
- `relativePath` 控制父 POM 的本地查找路径。

父 POM：
```xml
<project>
	<groupId>com.example</groupId>
	<artifactId>parent</artifactId>
	<version>1.0.0</version>
	<packaging>pom</packaging>

	<properties>
		<java.version>21</java.version>
	</properties>

	<modules>
		<module>service-api</module>
		<module>service-impl</module>
	</modules>
</project>
```

子模块：
```xml
<project>
	<parent>
		<groupId>com.example</groupId>
		<artifactId>parent</artifactId>
		<version>1.0.0</version>
		<relativePath>../pom.xml</relativePath>
	</parent>
	<artifactId>service-api</artifactId>
</project>
```

# 4. dependencyManagement 与 dependencies

- `dependencyManagement`：只“管理版本”，不引入依赖。
- `dependencies`：真正引入依赖。
- 企业常用做法：
	- 父 POM 定义 `dependencyManagement`
	- 子模块只声明 `groupId`/`artifactId`

```xml
<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-dependencies</artifactId>
			<version>3.3.4</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>
```

# 5. build 与 reporting

- `build` 主要内容：
	- `finalName`：产物文件名
	- `resources` / `testResources`
	- `plugins` / `pluginManagement`
- `reporting` 用于 `site` 生命周期（Javadoc、测试报告、依赖报告）。

```xml
<build>
	<finalName>order-service</finalName>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-compiler-plugin</artifactId>
			<version>3.11.0</version>
		</plugin>
	</plugins>
</build>
```

## 5.1 repositories 与 pluginRepositories

- `repositories`：声明依赖下载的远程仓库地址。
- `pluginRepositories`：声明插件下载的远程仓库（与依赖仓库分开管理）。

```xml
<repositories>
	<repository>
		<id>central</id>
		<name>Maven Central</name>
		<url>https://repo.maven.apache.org/maven2</url>
		<releases>
			<enabled>true</enabled>
		</releases>
		<snapshots>
			<enabled>false</enabled>
		</snapshots>
	</repository>
	<repository>
		<id>company</id>
		<url>https://repo.company.com/releases</url>
	</repository>
</repositories>

<pluginRepositories>
	<pluginRepository>
		<id>central</id>
		<url>https://repo.maven.apache.org/maven2</url>
	</pluginRepository>
</pluginRepositories>
```

- 常见策略：`releases` 开启，`snapshots` 关闭——避免正式构建混入快照依赖。
- `updatePolicy`：控制多久检查一次更新（`daily`、`always`、`never`）。

## 5.2 distributionManagement（发布管理）

- 配置构件发布的目标仓库，必须与 `settings.xml` 的 `server.id` 对应。

```xml
<distributionManagement>
	<repository>
		<id>company-releases</id>
		<name>Company Releases</name>
		<url>https://repo.company.com/releases</url>
	</repository>
	<snapshotRepository>
		<id>company-snapshots</id>
		<name>Company Snapshots</name>
		<url>https://repo.company.com/snapshots</url>
	</snapshotRepository>
	<site>
		<id>company-site</id>
		<url>scp://server.company.com/site/project</url>
	</site>
</distributionManagement>
```

- `mvn deploy` 会将构件上传到 `distributionManagement` 指定的仓库。
- Snapshot 版本自动路由到 `snapshotRepository`，正式版本路由到 `repository`。

# 6. properties 与插值

- 常用属性来源：
	- 命令行 `-Dkey=value`
	- POM `<properties>`
	- `settings.xml` 的 profile 属性
	- 系统环境变量 `${env.JAVA_HOME}`

示例：
```xml
<properties>
	<java.version>21</java.version>
	<spring.version>6.1.12</spring.version>
</properties>

<dependencies>
	<dependency>
		<groupId>org.springframework</groupId>
		<artifactId>spring-context</artifactId>
		<version>${spring.version}</version>
	</dependency>
</dependencies>
```

# 7. Effective POM 与常用诊断

- 查看最终生效配置（合并父 POM 与 profile 后）：
```bash
mvn help:effective-pom
```

- 查看插件的默认配置与目标：
```bash
mvn help:describe -Dplugin=maven-compiler-plugin -Ddetail
```

- 解析某个表达式：
```bash
mvn help:evaluate -Dexpression=project.version
```

# 8. 企业实践建议

- 使用公司级 `parent` 统一依赖版本与插件版本。
- 禁用 `LATEST` / `RELEASE` 等动态版本。
- 在父 POM 中统一编码、Java 版本、测试框架。
- 引入 `maven-enforcer-plugin` 强制构建规则。

# 9. 常见问题排查清单

- POM 配置不生效：用 `mvn help:effective-pom` 查看最终生效配置。
- 父 POM 找不到：检查 `relativePath`，并确认父 POM 已安装或可从私服下载。
- 属性未替换：检查属性是否定义在正确的 POM 或 profile 中。
- 插件目标未执行：确认插件是否绑定到正确阶段，或在命令中显式执行。
- 发布信息无效：确认 `distributionManagement` 与 `settings.xml` 的 `server.id` 一致。

# 10. 实战案例：企业父 POM 统一依赖与插件

目标：建立公司级父 POM，统一版本、插件与编码。

父 POM 示例：
```xml
<project>
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.company</groupId>
	<artifactId>company-parent</artifactId>
	<version>1.0.0</version>
	<packaging>pom</packaging>

	<properties>
		<java.version>21</java.version>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	</properties>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-dependencies</artifactId>
				<version>3.3.4</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<build>
		<pluginManagement>
			<plugins>
				<plugin>
					<groupId>org.apache.maven.plugins</groupId>
					<artifactId>maven-compiler-plugin</artifactId>
					<version>3.11.0</version>
					<configuration>
						<source>${java.version}</source>
						<target>${java.version}</target>
					</configuration>
				</plugin>
			</plugins>
		</pluginManagement>
	</build>
</project>
```
