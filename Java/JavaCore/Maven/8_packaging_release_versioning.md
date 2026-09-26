
# 1. 打包类型

- `jar`：Java 程序常用。
- `war`：部署到外部容器。
- `pom`：聚合或父工程。
- `ear`：企业级应用打包（较少使用）。

```xml
<packaging>jar</packaging>
```

## 1.1 maven-shade-plugin（Uber JAR / Fat JAR）

- 将所有依赖解包后与项目代码合并到一个 JAR。
- 适合需要独立运行、没有外部 classpath 的场景：

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-shade-plugin</artifactId>
	<version>3.5.1</version>
	<executions>
		<execution>
			<phase>package</phase>
			<goals><goal>shade</goal></goals>
			<configuration>
				<transformers>
					<transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
						<mainClass>com.example.App</mainClass>
					</transformer>
				</transformers>
				<relocations>
					<relocation>
						<pattern>com.google.common</pattern>
						<shadedPattern>com.example.shaded.google.common</shadedPattern>
					</relocation>
				</relocations>
			</configuration>
		</execution>
	</executions>
</plugin>
```

- `relocations` 用于解决类名冲突——将第三方包重定位到阴影命名空间。
- 与 `assembly-plugin` 的区别：shade 会将所有类合并到单个 JAR（适合直接 `java -jar`），assembly 生成带 `lib/` 目录的归档。

## 1.2 Spring Boot 可执行 JAR

- `spring-boot-maven-plugin` 的 `repackage` 目标生成可执行 JAR，结构如下：
```
boot-app.jar
├── META-INF/
├── BOOT-INF/
│   ├── classes/         (项目代码)
│   └── lib/             (所有依赖)
└── org/springframework/boot/loader/  (Spring Boot 启动器)
```

- 无需额外配置，Spring Initializr 生成项目自带。

# 2. 版本策略

- 语义化版本：`主版本.次版本.修订号`。
- `SNAPSHOT` 表示快照版本，允许频繁发布。
- 正式发布使用无 `SNAPSHOT` 的版本。

例：
- `1.0.0-SNAPSHOT`
- `1.0.0`

CI 友好版本：
```xml
<version>${revision}${changelist}</version>

<properties>
	<revision>1.2.0</revision>
	<changelist>-SNAPSHOT</changelist>
</properties>
```

# 3. Release 插件基础

- Release 插件用于版本升级、打标签、发布。
- 要求：
	- 工作区干净
	- 已配置 SCM
	- 具备发布仓库权限

```bash
mvn release:prepare
mvn release:perform
```

# 4. 标签与 SCM

```xml
<scm>
	<connection>scm:git:https://github.com/example/repo.git</connection>
	<developerConnection>scm:git:ssh://git@github.com:example/repo.git</developerConnection>
	<tag>v1.0.0</tag>
</scm>
```

# 5. 发布产物与附加包

- 常见附加产物：`sources`、`javadoc`。

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-source-plugin</artifactId>
	<version>3.3.0</version>
	<executions>
		<execution>
			<goals>
				<goal>jar</goal>
			</goals>
		</execution>
	</executions>
</plugin>
```

# 6. 可重复构建

- 固定依赖版本，避免动态版本。
- 设置构建时间戳，保证产物一致。

```xml
<properties>
	<project.build.outputTimestamp>2024-01-01T00:00:00Z</project.build.outputTimestamp>
</properties>
```

# 7. 企业实践建议

- Release 操作建议在 CI 中执行，避免本地环境差异。
- 正式版本必须发布到私服 `releases` 仓库。

## 7.1 maven-flatten-plugin（CI 友好版本利器）

- 解决 `${revision}` 变量在子模块 POM 中的一致性问题。
- 安装后自动生成扁平化的 `.flattened-pom.xml`，所有变量被替换为实际值。

```xml
<plugin>
	<groupId>org.codehaus.mojo</groupId>
	<artifactId>flatten-maven-plugin</artifactId>
	<version>1.6.0</version>
	<configuration>
		<updatePomFile>true</updatePomFile>
		<flattenMode>resolveCiFriendliesOnly</flattenMode>
	</configuration>
	<executions>
		<execution>
			<id>flatten</id>
			<phase>process-resources</phase>
			<goals><goal>flatten</goal></goals>
		</execution>
		<execution>
			<id>flatten.clean</id>
			<phase>clean</phase>
			<goals><goal>clean</goal></goals>
		</execution>
	</executions>
</plugin>
```

- 搭配 CI 友好版本使用效果最佳：
```bash
mvn -Drevision=1.2.0 -Dchangelist= clean deploy
```

## 7.2 GPG 签名（开源发布必备）

- Maven Central 要求构件附带 GPG 签名。
- 配置 `maven-gpg-plugin`：

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-gpg-plugin</artifactId>
	<version>3.1.0</version>
	<executions>
		<execution>
			<id>sign-artifacts</id>
			<phase>verify</phase>
			<goals><goal>sign</goal></goals>
		</execution>
	</executions>
</plugin>
```

- 前提：
	1. 已安装 GPG 并生成密钥对：`gpg --gen-key`
	2. 已发布公钥到密钥服务器：`gpg --keyserver keyserver.ubuntu.com --send-keys <KEYID>`
	3. CI 中通过环境变量传递 passphrase：`MAVEN_GPG_PASSPHRASE`

## 7.3 发布到 Maven Central（OSSRH）

- 开源发布流程（通过 Sonatype OSSRH）：

1. **注册 Sonatype JIRA 账号**并创建工单申请 `groupId`。
2. **配置父 POM**（OSS Parent）：
```xml
<parent>
	<groupId>org.sonatype.oss</groupId>
	<artifactId>oss-parent</artifactId>
	<version>9</version>
</parent>
```
3. **配置 `distributionManagement`** 指向 OSSRH：
```xml
<distributionManagement>
	<snapshotRepository>
		<id>ossrh</id>
		<url>https://s01.oss.sonatype.org/content/repositories/snapshots</url>
	</snapshotRepository>
	<repository>
		<id>ossrh</id>
		<url>https://s01.oss.sonatype.org/content/repositories/releases</url>
	</repository>
</distributionManagement>
```
4. **同时配置 GPG 签名**（见 1.9）。
5. **同时配置 sources + javadoc 插件**（Maven Central 强制要求）。
6. **执行发布**：
```bash
mvn clean deploy -DskipTests
```
7. 登录 Sonatype Nexus UI，在 Staging Repositories 中 Close -> Release。

# 8. 常见问题排查清单

- 发布失败：确认 `distributionManagement` 与 `settings.xml` 配置一致。
- SNAPSHOT 发布到 releases：检查版本号是否包含 `SNAPSHOT`。
- Release 插件报错：确认工作区干净且 SCM 配置正确。
- 附加包未生成：检查 source/javadoc 插件是否绑定到生命周期。
- 产物不可复现：固定依赖版本并设置构建时间戳。

# 9. 实战案例：正式版发布流程

目标：从快照版本发布正式版本到私服。

步骤：
1. 确保代码已合并并通过测试。
2. 更新版本为无 `SNAPSHOT`。
3. 执行 `mvn -B -DskipTests deploy`。

命令示例：
```bash
mvn -B -Drevision=1.2.0 -Dchangelist= deploy
```
