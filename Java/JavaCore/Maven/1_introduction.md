
# 1. 什么是 Maven

- Maven 是 Java 的构建与依赖管理工具。
- 主要解决三件事：
	- 统一项目结构与构建流程（编译、测试、打包、发布）。
	- 统一依赖管理（解决 jar 包版本与传递依赖问题）。
	- 统一团队协作（相同的构建规则和版本管理方式）。
- Maven 的核心产物是 `pom.xml`，它定义了项目坐标、依赖、构建与插件。

## 1.1 约定优于配置（Convention over Configuration）

- Maven 的核心理念——为项目提供合理的默认约定，减少冗余配置。
- 约定内容：
	- 标准目录结构：`src/main/java`、`src/main/resources`、`src/test/java` 等。
	- 标准生命周期：`compile` -> `test` -> `package` -> `install` -> `deploy`。
	- 默认插件绑定：每个生命周期阶段都有默认绑定的插件目标。
- 好处：
	- 团队统一——所有项目结构一致，新人上手快。
	- 减少配置——只需声明差异化部分。
	- 可扩展——通过插件覆盖或增强默认行为。
- 对比：Ant 需要逐一定义每个构建步骤，Maven 只需声明"我要打包"，其余自动完成。

# 2. Maven 的典型工作流

- 开发阶段：`compile` + `test` + `package`。
- 集成阶段：`verify`（含集成测试）。
- 发布阶段：`install`（本地）/ `deploy`（私服）。
- 企业建议：
	- 开发用 `mvn test` 或 `mvn -DskipTests package`
	- CI 用 `mvn -B -V clean verify`
	- 发布用 `mvn -B -DskipTests deploy`

# 3. Maven 核心概念

## 3.1 坐标（GAV）

- 由 `groupId`、`artifactId`、`version` 组成，唯一标识一个组件。
- 例子：
```xml
<groupId>com.example</groupId>
<artifactId>demo-app</artifactId>
<version>1.0.0</version>
```

### 3.1.1 命名规范建议

- `groupId` 使用公司域名反写 + 业务线。
- `artifactId` 用短横线分隔，保持统一风格。
- 版本建议遵循语义化：`主.次.修订`。

## 3.2 仓库

- 本地仓库：Maven 下载依赖后会缓存到本地。
- 远程仓库：中央仓库（Maven Central）或私服（Nexus/Artifactory）。
- 依赖解析顺序：本地 -> 私服/远程 -> 中央仓库。

### 3.2.1 仓库类型

- `releases`：正式版本
- `snapshots`：快照版本
- `pluginRepository`：插件仓库

## 3.3 生命周期与阶段

- Maven 有固定的生命周期：`clean`、`default`、`site`。
- 常用阶段：`validate`、`compile`、`test`、`package`、`verify`、`install`、`deploy`。
- 执行某个阶段会自动执行它之前的所有阶段。

### 3.3.1 常见阶段作用

- `compile`：编译主代码
- `test`：运行单元测试
- `package`：生成 jar/war
- `install`：安装到本地仓库
- `deploy`：发布到私服

## 3.4 依赖管理

- 依赖范围（scope）：`compile`、`test`、`provided`、`runtime`、`system`、`import`。
- 传递依赖：依赖的依赖也会被引入。

### 3.4.1 企业常见策略

- 使用父 POM 统一版本。
- 引入 BOM 统一框架版本。
- 禁止动态版本（`LATEST`、`RELEASE`）。

# 4. 安装与环境配置

## 4.1 安装步骤

- 安装 JDK 并配置 `JAVA_HOME`。
- 下载 Maven，解压并配置 `MAVEN_HOME`。
- 将 `%MAVEN_HOME%\bin` 加入 `PATH`。

## 4.2 验证安装

```bash
mvn -v
```

输出包含 Maven 版本、Java 版本与本机系统信息即安装成功。

## 4.3 常用配置文件

- 用户级：`~/.m2/settings.xml`
- 全局级：`MAVEN_HOME/conf/settings.xml`
- 优先级：用户级覆盖全局级。

## 4.4 常用 settings.xml 配置

- 自定义本地仓库位置：
```xml
<settings>
	<localRepository>D:/maven-repo</localRepository>
</settings>
```

- 配置镜像加速：
```xml
<mirrors>
	<mirror>
		<id>company</id>
		<mirrorOf>*</mirrorOf>
		<url>https://repo.company.com/group</url>
	</mirror>
</mirrors>
```

# 5. Maven Wrapper

## 5.1 作用

- Wrapper 让项目在不安装 Maven 的环境中也能构建。
- 团队保证使用同一 Maven 版本，降低构建差异。

## 5.2 常见文件

- `mvnw`、`mvnw.cmd`
- `.mvn/wrapper/maven-wrapper.properties`

## 5.3 使用方式

```bash
./mvnw -v
./mvnw clean package
```

## 5.4 适用场景

- 团队成员 JDK 或 Maven 版本不一致。
- CI 机器不安装 Maven。
- 保证构建可重复性。

# 6. 常用命令

- `mvn clean`：清理 `target` 目录。
- `mvn compile`：编译主代码。
- `mvn test`：执行单元测试。
- `mvn package`：打包（jar/war）。
- `mvn install`：安装到本地仓库。
- `mvn deploy`：发布到远程仓库。
- `mvn -DskipTests package`：跳过测试打包。

## 6.1 常用参数

- `-DskipTests`：跳过测试执行
- `-Dmaven.test.skip=true`：跳过测试编译与执行
- `-T 1C`：多线程构建
- `-B`：CI 非交互模式
- `-U`：强制更新快照

# 7. 本地仓库结构

- 默认路径：`~/.m2/repository`
- 目录结构示例：
```
~/.m2/repository
└── com
		└── example
				└── demo-app
						└── 1.0.0
								├── demo-app-1.0.0.jar
								├── demo-app-1.0.0.pom
								└── _remote.repositories
```
- 通常包含：jar、pom、checksum（.sha1/.md5）文件。

# 8. Maven vs Gradle 对比

| 维度 | Maven | Gradle |
|------|-------|--------|
| 构建模型 | 生命周期 + 阶段（声明式 XML） | 任务图（脚本式 Groovy/Kotlin） |
| 配置文件 | `pom.xml`（XML） | `build.gradle` / `build.gradle.kts` |
| 依赖管理 | GAV 坐标 + 传递依赖（最近优先） | 类似 Maven，但支持动态版本、版本目录 |
| 构建速度 | 一般（可启用并行 `-T`） | 快（增量构建 + 构建缓存 + Daemon） |
| 灵活性 | 低——严格生命周期，扩展靠插件 | 高——可任意定义任务与依赖逻辑 |
| 学习曲线 | 简单——标准化约定 | 较陡——DSL 语法 + 概念多 |
| 多模块 | 聚合 + 继承（Reactor） | 复合构建（composite builds） |
| 生态 | 插件丰富，企业私服成熟 | 兼容 Maven 仓库，插件增长快 |
| 适用场景 | 企业标准化项目、多模块、严控版本 | 大型 Monorepo、Android、需要自定义构建逻辑 |

## 8.1 如何选择
- 团队以标准化、稳定性为首要目标 → **Maven**。
- 需要极速构建、灵活定制、多语言混合 → **Gradle**。
- 已有遗留 Maven 项目，可渐进式迁移到 Gradle，也可维持 Maven。

# 9. IDE 集成

## 9.1 IntelliJ IDEA
- IDEA 内置 Maven 支持，无需额外插件。
- 导入项目：`File -> Open -> 选择 pom.xml`，IDEA 自动识别并建立 Maven 项目模型。
- Maven 工具窗口：右侧 `Maven` 面板，可双击执行生命周期阶段、查看依赖树。
- 常用操作：
	- `Ctrl+Shift+O` / `Reimport`：强制刷新 Maven 依赖。
	- `Generate Sources and Update Folders`：重新生成源码目录。
- 设置：`Settings -> Build, Execution, Deployment -> Build Tools -> Maven`，可指定 Maven 版本、settings.xml 路径、JVM 参数。

## 9.2 Eclipse
- 通过 `m2e`（Maven to Eclipse）插件原生支持。
- 导入项目：`File -> Import -> Existing Maven Projects`。
- 右键项目 -> `Maven -> Update Project` 同步依赖。
- 常见问题：m2e 插件版本与项目插件冲突时，可通过 lifecycle-mapping 忽略或手动绑定。

## 9.3 VS Code
- 安装 `Extension Pack for Java`，内含 Maven 支持。
- 打开包含 `pom.xml` 的文件夹即可自动识别。
- 左侧 `MAVEN` 面板可执行目标和管理依赖。

# 10. 企业实践建议

- 使用公司统一父 POM 控制依赖与插件版本。
- 所有构件发布到私服，禁止直接访问公网仓库。
- 在 CI 中固定 `settings.xml`，避免本地差异。
- 常用诊断命令：
```bash
mvn help:effective-pom
mvn dependency:tree -Dverbose
```

# 11. 常见问题排查清单

- `mvn` 命令找不到：检查 `PATH` 是否包含 Maven 的 `bin` 目录。
- Java 版本异常：检查 `JAVA_HOME`，并确认 `mvn -v` 输出的 JDK 版本。
- 下载依赖慢或失败：检查 `settings.xml` 中镜像与代理配置。
- 快照版本不更新：使用 `-U` 强制更新或删除本地快照目录。
- Wrapper 失败：检查 `mvnw` 权限与 `maven-wrapper.properties` 的 `distributionUrl`。
- 本地仓库损坏：删除对应依赖目录后重新构建。

# 12. 实战案例：从零初始化企业项目

目标：快速初始化一个企业项目骨架，并确保团队构建一致。

步骤：
1. 生成项目骨架（以 Maven 为构建工具）。
2. 引入 Maven Wrapper，固定 Maven 版本。
3. 配置私服镜像与本地仓库路径。

示例命令：
```bash
mvn -B archetype:generate -DgroupId=com.company.demo -DartifactId=demo-app -DarchetypeArtifactId=maven-archetype-quickstart
```

生成 Wrapper：
```bash
mvn -N io.takari:maven:wrapper
```

settings.xml（公司统一下发）：
```xml
<settings>
	<localRepository>D:/maven-repo</localRepository>
	<mirrors>
		<mirror>
			<id>company</id>
			<mirrorOf>*</mirrorOf>
			<url>https://repo.company.com/group</url>
		</mirror>
	</mirrors>
</settings>
```
