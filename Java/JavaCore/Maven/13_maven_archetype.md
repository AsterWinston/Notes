
# 1. 什么是 Archetype

- Archetype 是 Maven 的项目模板工具——通过一个命令生成标准化的项目骨架。
- 内置常用 Archetype：
	- `maven-archetype-quickstart`：简单 Java 项目
	- `maven-archetype-webapp`：Web 项目
	- `maven-archetype-site`：站点项目

# 2. 使用内置 Archetype 生成项目

```bash
mvn archetype:generate \
  -DgroupId=com.example \
  -DartifactId=my-app \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false
```

- 参数说明：
	- `-DgroupId`：项目 groupId
	- `-DartifactId`：项目名称
	- `-DarchetypeGroupId` / `-DarchetypeArtifactId` / `-DarchetypeVersion`：模板坐标
	- `-DinteractiveMode=false`：非交互模式（CI 中必须）

# 3. 创建自定义 Archetype

## 3.1 从现有项目生成 Archetype

```bash
# 在已存在的标准项目中
mvn archetype:create-from-project
```

执行后 archetype 生成在 `target/generated-sources/archetype/`。

## 3.2 安装到本地仓库

```bash
cd target/generated-sources/archetype
mvn clean install
```

## 3.3 使用自定义 Archetype

```bash
mvn archetype:generate \
  -DarchetypeGroupId=com.company \
  -DarchetypeArtifactId=company-project-archetype \
  -DarchetypeVersion=1.0.0 \
  -DgroupId=com.company.demo \
  -DartifactId=new-project
```

# 4. Archetype 目录结构

```
archetype-project
├── pom.xml
├── src
│   └── main
│       └── resources
│           ├── archetype-resources
│           │   ├── pom.xml              ← 模板 POM（可含占位符）
│           │   ├── src
│           │   │   ├── main
│           │   │   │   └── java
│           │   │   │       └── __groupId__.java  ← 占位符文件名
│           │   │   └── test
│           │   │       └── java
│           │   │           └── __groupId__Test.java
│           │   └── README.md
│           └── META-INF
│               └── maven
│                   └── archetype-metadata.xml  ← 核心元数据
```

# 5. archetype-metadata.xml 详解

```xml
<archetype-descriptor xmlns="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-descriptor/1.1.0"
                      name="company-project">

	<requiredProperties>
		<requiredProperty key="appName">
			<defaultValue>my-app</defaultValue>
		</requiredProperty>
		<requiredProperty key="javaVersion">
			<defaultValue>21</defaultValue>
		</requiredProperty>
		<requiredProperty key="port">
			<defaultValue>8080</defaultValue>
		</requiredProperty>
	</requiredProperties>

	<fileSets>
		<fileSet filtered="true" packaged="true">
			<directory>src/main/java</directory>
			<includes>
				<include>**/*.java</include>
			</includes>
		</fileSet>
		<fileSet filtered="true" packaged="false">
			<directory>src/main/resources</directory>
			<includes>
				<include>**/*.yml</include>
				<include>**/*.properties</include>
			</includes>
		</fileSet>
		<fileSet filtered="false" packaged="false">
			<directory>src/main/resources/static</directory>
		</fileSet>
	</fileSets>
</archetype-descriptor>
```

- `filtered="true"`：文件中的 `${xxx}` 占位符会被替换。
- `packaged="true"`：文件会被放入 `groupId` 对应的包路径下。
- `requiredProperties`：生成项目时提示用户输入的额外变量。

# 6. 模板变量替换

`archetype-resources/pom.xml` 模板示例：

```xml
<project>
	<groupId>${groupId}</groupId>
	<artifactId>${artifactId}</artifactId>
	<version>${version}</version>

	<properties>
		<java.version>${javaVersion}</java.version>
	</properties>
</project>
```

常用内置变量：
- `${groupId}` / `${artifactId}` / `${version}` / `${package}`
- 自定义变量通过 `requiredProperties` 声明后也可使用。

# 7. 多模块 Archetype

```xml
<!-- archetype-metadata.xml -->
<modules>
	<module id="common" dir="common" name="common">
		<fileSets>
			<fileSet filtered="true" packaged="true">
				<directory>src/main/java</directory>
			</fileSet>
		</fileSets>
	</module>
	<module id="web" dir="web" name="web">
		<fileSets>
			<fileSet filtered="true" packaged="true">
				<directory>src/main/java</directory>
			</fileSet>
		</fileSets>
	</module>
</modules>
```

# 8. 发布 Archetype 到私服

```bash
# 在 archetype 项目根目录
mvn clean deploy
```

之后团队成员可直接通过坐标引用：

```bash
mvn archetype:generate \
  -DarchetypeGroupId=com.company \
  -DarchetypeArtifactId=company-project-archetype \
  -DarchetypeVersion=1.0.0
```

# 9. 常见问题排查清单

- 生成项目后包路径不正确：检查 `archetype-metadata.xml` 中 `packaged` 与目录结构。
- 占位符未被替换：确认文件在 `fileSet` 中 `filtered="true"`。
- 自定义变量输入不出现：确认已在 `requiredProperties` 中声明。
- 多模块生成异常：检查 `modules` 配置中各模块 ID 与目录对应关系。
