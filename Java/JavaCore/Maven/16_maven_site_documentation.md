
# 1. Site 生命周期

- Maven 提供 `site` 生命周期用于生成项目文档和报告网站。
- 主要阶段：
	- `pre-site`：生成站点前准备
	- `site`：生成站点文档
	- `post-site`：后续处理
	- `site-deploy`：部署站点到服务器

```bash
mvn site
```

生成后站点位于 `target/site/`，打开 `index.html` 浏览。

# 2. 站点描述符（site.xml）

```xml
<!-- src/site/site.xml -->
<project name="Order Service">
	<bannerLeft>
		<name>Order Service</name>
		<src>images/logo.png</src>
	</bannerLeft>

	<body>
		<menu name="Overview">
			<item name="Introduction" href="index.html"/>
			<item name="Getting Started" href="getting-started.html"/>
		</menu>
		<menu name="Documentation">
			<item name="API" href="apidocs/index.html"/>
		</menu>
		<menu ref="reports"/>
	</body>

	<skin>
		<groupId>org.apache.maven.skins</groupId>
		<artifactId>maven-fluido-skin</artifactId>
		<version>1.12.0</version>
	</skin>
</project>
```

- `src/site/resources/`：存放静态资源（CSS、图片、HTML）。
- `<menu ref="reports"/>`：自动插入所有生成的报告。

# 3. 常用报告插件

## 3.1 Javadoc

```xml
<reporting>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-javadoc-plugin</artifactId>
			<version>3.6.3</version>
			<configuration>
				<source>21</source>
			</configuration>
		</plugin>
	</plugins>
</reporting>
```

## 3.2 测试报告

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-surefire-report-plugin</artifactId>
	<version>3.2.5</version>
</plugin>
```

## 3.3 JaCoCo 覆盖率报告

```xml
<plugin>
	<groupId>org.jacoco</groupId>
	<artifactId>jacoco-maven-plugin</artifactId>
	<version>0.8.12</version>
	<reportSets>
		<reportSet>
			<reports>
				<report>report</report>
			</reports>
		</reportSet>
	</reportSets>
</plugin>
```

## 3.4 Checkstyle 报告

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-checkstyle-plugin</artifactId>
	<version>3.3.1</version>
	<configuration>
		<configLocation>google_checks.xml</configLocation>
	</configuration>
</plugin>
```

## 3.5 依赖分析报告

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-project-info-reports-plugin</artifactId>
	<version>3.5.0</version>
	<reportSets>
		<reportSet>
			<reports>
				<report>dependencies</report>
				<report>dependency-info</report>
				<report>index</report>
				<report>summary</report>
			</reports>
		</reportSet>
	</reportSets>
</plugin>
```

# 4. 生成多模块聚合站点

在父 POM 中配置：

```xml
<distributionManagement>
	<site>
		<id>company-site</id>
		<url>scp://server.company.com/site/${project.artifactId}</url>
	</site>
</distributionManagement>
```

```bash
# 生成聚合站点（包含所有子模块）
mvn site site:stage
```

站点输出在 `target/staging/`，包含所有子模块的报告链接。

# 5. 部署站点

```bash
mvn site-deploy
```

- 依赖 `distributionManagement` 中的 `<site>` 配置。
- 支持协议：`scp`、`ftp`、`file`、`dav`。

# 6. 编写项目文档

在 `src/site/markdown/` 中放置 Markdown 文件：

````markdown
<!-- src/site/markdown/getting-started.md -->
# Getting Started

## Prerequisites

- JDK 21+
- Maven 3.9+

## Build

```bash
mvn clean package
```
````

需配置 `maven-site-plugin` 支持 Markdown 渲染（通过 Doxia 模块）。

# 7. 常见问题排查清单

- 站点为空：确认已在 `<reporting>` 中配置报告插件。
- 模块聚合站缺少子模块：检查父 POM 是否包含 `<modules>`。
- 报告生成失败：确认插件版本兼容当前 Maven 版本。
- 部署失败：确认 `settings.xml` 的 `server.id` 与 `<site>` 的 `id` 一致。
