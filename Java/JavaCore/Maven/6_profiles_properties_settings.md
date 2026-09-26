
# 1. Profile 与激活方式

- Profile 用于区分不同构建环境（dev/test/prod）。
- 激活方式：
	- 命令行：`-Pdev`
	- JDK 版本
	- OS 条件
	- 系统属性
	- 文件存在与否
	- `activeByDefault`

示例：
```xml
<profiles>
	<profile>
		<id>dev</id>
		<activation>
			<property>
				<name>env</name>
				<value>dev</value>
			</property>
		</activation>
		<properties>
			<env.name>dev</env.name>
		</properties>
	</profile>
</profiles>
```

## 1.1 activeByDefault 与互斥行为

- 声明 `<activeByDefault>true</activeByDefault>` 的 profile 在没有其他 profile 被激活时自动启用。
- 一旦通过 `-P` 或条件激活了任意 profile，`activeByDefault` 会自动失效。

```xml
<profile>
	<id>local</id>
	<activation>
		<activeByDefault>true</activeByDefault>
	</activation>
	<properties>
		<env.name>local</env.name>
	</properties>
</profile>
```

- 典型应用：开发环境用 `activeByDefault`，CI/生产环境通过 `-P` 强制切换。

## 1.2 JDK 范围激活

- 根据 JDK 版本范围自动选择 profile：

```xml
<profile>
	<id>java21</id>
	<activation>
		<jdk>[21,)</jdk>
	</activation>
</profile>

<profile>
	<id>java17</id>
	<activation>
		<jdk>[17,21)</jdk>
	</activation>
</profile>
```

## 1.3 查看当前激活的 Profile

```bash
mvn help:active-profiles
```

输出当前所有激活的 profile ID、来源（POM / settings.xml）和激活方式。

## 1.4 加载外部属性文件

- 使用 `properties-maven-plugin` 从外部文件读取属性：

```xml
<plugin>
	<groupId>org.codehaus.mojo</groupId>
	<artifactId>properties-maven-plugin</artifactId>
	<version>1.2.1</version>
	<executions>
		<execution>
			<phase>initialize</phase>
			<goals>
				<goal>read-project-properties</goal>
			</goals>
			<configuration>
				<files>
					<file>${user.home}/build.config.properties</file>
				</files>
			</configuration>
		</execution>
	</executions>
</plugin>
```

- 用途：CI 环境注入动态配置（如密钥、数据库地址），不污染 POM 或 settings.xml。

# 2. 属性优先级与来源

优先级从高到低：
1. 命令行 `-Dkey=value`
2. POM 中的 `<properties>`
3. Profile 中的 `<properties>`
4. `settings.xml` 中的 `<properties>`
5. 依赖 POM 中的属性

# 3. settings.xml 结构

- 常见配置区域：
	- `localRepository`
	- `mirrors`
	- `proxies`
	- `servers`
	- `profiles` / `activeProfiles`

示例：
```xml
<settings>
	<localRepository>D:/maven-repo</localRepository>
	<interactiveMode>true</interactiveMode>
	<offline>false</offline>
</settings>
```

# 4. 镜像与 mirrorOf 规则

- 镜像用于加速依赖下载或强制走私服。
- `mirrorOf` 常见规则：
	- `central`：仅中央仓库
	- `*`：所有仓库
	- `external:*`：所有外部仓库
	- `!,` 排除某个仓库

示例：
```xml
<mirrors>
	<mirror>
		<id>company</id>
		<mirrorOf>*</mirrorOf>
		<url>https://repo.company.com/group</url>
	</mirror>
</mirrors>
```

# 5. 代理配置

- 企业内网访问外网仓库时使用代理。

```xml
<proxies>
	<proxy>
		<id>corp</id>
		<active>true</active>
		<protocol>http</protocol>
		<host>proxy.company.com</host>
		<port>8080</port>
	</proxy>
</proxies>
```

# 6. 服务器与凭据

- `servers` 用于配置私服账号密码。
- `id` 必须与 `distributionManagement` 一致。

```xml
<servers>
	<server>
		<id>company-releases</id>
		<username>deploy</username>
		<password>******</password>
	</server>
</servers>
```

密码加密：
```bash
mvn --encrypt-master-password
mvn --encrypt-password
```

## 6.1 密码加密完整流程

1. **创建主密码**（用于加密所有其他密码）：
```bash
mvn --encrypt-master-password
# 输入主密码后得到加密字符串: {abc123...}
```

2. **保存主密码到 `settings-security.xml`**（位于 `~/.m2/`）：
```xml
<settingsSecurity>
	<master>{abc123...}</master>
</settingsSecurity>
```

3. **加密私服密码**：
```bash
mvn --encrypt-password myServerPassword
# 得到: {xyz456...}
```

4. **在 `settings.xml` 中使用加密密码**：
```xml
<servers>
	<server>
		<id>company-releases</id>
		<username>deploy</username>
		<password>{xyz456...}</password>
	</server>
</servers>
```

- Maven 运行时自动解密，无需应用代码感知。
- `settings-security.xml` 必须严格限制权限（`chmod 600`），不能提交到代码仓库。

# 7. 企业实践建议

- CI 中统一下发 `settings.xml`，避免本地差异。
- 私服镜像配置在 `settings.xml`，POM 保持干净。
- 凭据不要写入 POM，使用环境变量或密钥管理。

# 8. 常见问题排查清单

- Profile 未生效：检查是否正确激活（`-P` 或激活条件）。
- 镜像不生效：检查 `mirrorOf` 规则是否覆盖目标仓库。
- 私服部署失败：核对 `server.id` 与 `distributionManagement` 是否一致。
- 代理无效：确认代理 `active` 为 true，且未被禁用。
- settings 未加载：确认用户级文件路径是否正确。

# 9. 实战案例：公司私服与代理统一配置

目标：统一开发环境与 CI 的仓库入口和代理策略。

settings.xml：
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
	<proxies>
		<proxy>
			<id>corp</id>
			<active>true</active>
			<protocol>http</protocol>
			<host>proxy.company.com</host>
			<port>8080</port>
		</proxy>
	</proxies>
</settings>
```
