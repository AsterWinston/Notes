
# 1. 生命周期与阶段

- 生命周期是 Maven 预定义的构建流程：
	- `clean`：清理构建产物
	- `default`：编译、测试、打包、发布
	- `site`：生成站点报告

常见阶段：
- `validate` -> `compile` -> `test` -> `package` -> `verify` -> `install` -> `deploy`

执行某个阶段会自动执行其之前的阶段。

## 1.1 Default 生命周期完整阶段与默认插件绑定

| 阶段（Phase） | 绑定的默认插件目标（Goal） | 说明 |
|---------------|--------------------------|------|
| `validate` | — | 验证项目配置是否完整 |
| `initialize` | — | 初始化构建状态 |
| `generate-sources` | — | 生成源码（如代码生成器） |
| `process-sources` | — | 处理源码 |
| `generate-resources` | — | 生成资源文件 |
| `process-resources` | `resources:resources` | 复制资源到 output |
| `compile` | `compiler:compile` | 编译主源码 |
| `process-classes` | — | 后处理字节码 |
| `generate-test-sources` | — | 生成测试源码 |
| `process-test-sources` | — | 处理测试源码 |
| `generate-test-resources` | — | 生成测试资源 |
| `process-test-resources` | `resources:testResources` | 复制测试资源 |
| `test-compile` | `compiler:testCompile` | 编译测试代码 |
| `process-test-classes` | — | 后处理测试字节码 |
| `test` | `surefire:test` | 执行单元测试 |
| `prepare-package` | — | 打包前准备 |
| `package` | `jar:jar` / `war:war` | 打包 |
| `pre-integration-test` | — | 集成测试前准备 |
| `integration-test` | — | 执行集成测试 |
| `post-integration-test` | — | 集成测试后清理 |
| `verify` | — | 验证包完整性 |
| `install` | `install:install` | 安装到本地仓库 |
| `deploy` | `deploy:deploy` | 发布到远程仓库 |

- 打包类型决定了 `package` 阶段的默认插件（`jar` -> `maven-jar-plugin`，`war` -> `maven-war-plugin`）。
- 空 `—` 表示没有默认绑定，需要自行配置。

# 2. 常用目标（goal）

- 目标是插件中的具体任务。
- 例：`mvn compiler:compile` 执行 `maven-compiler-plugin` 的 `compile` 目标。

常用命令：
```bash
mvn clean
mvn test
mvn package
mvn install
```

# 3. 插件与执行机制

- 插件是 Maven 构建能力的来源。
- 通过 `executions` 将插件目标绑定到生命周期阶段。

示例：
```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-jar-plugin</artifactId>
	<version>3.3.0</version>
	<executions>
		<execution>
			<phase>package</phase>
			<goals>
				<goal>jar</goal>
			</goals>
		</execution>
	</executions>
</plugin>
```

# 4. 常用插件清单

- 编译：`maven-compiler-plugin`
- 资源：`maven-resources-plugin`
- 测试：`maven-surefire-plugin`
- 集成测试：`maven-failsafe-plugin`
- 打包：`maven-jar-plugin` / `maven-war-plugin`
- 依赖分析：`maven-dependency-plugin`
- 代码质量：`maven-checkstyle-plugin` / `spotbugs-maven-plugin`
- 覆盖率：`jacoco-maven-plugin`
- 企业常见：`maven-enforcer-plugin`

## 4.1 Spring Boot Maven Plugin

- Spring Boot 项目必配插件，生成可执行 JAR：

```xml
<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
	<version>3.3.4</version>
	<executions>
		<execution>
			<goals>
				<goal>repackage</goal>
			</goals>
		</execution>
	</executions>
</plugin>
```

- `repackage` 目标：将原始 JAR 重新打包为可执行的 fat JAR，包含 `BOOT-INF` 目录结构。

## 4.2 maven-shade-plugin（Uber JAR）

- 将所有依赖合并到一个 JAR 中（类冲突时建议使用）：

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-shade-plugin</artifactId>
	<version>3.5.1</version>
	<executions>
		<execution>
			<phase>package</phase>
			<goals>
				<goal>shade</goal>
			</goals>
			<configuration>
				<transformers>
					<transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
						<mainClass>com.example.Main</mainClass>
					</transformer>
				</transformers>
			</configuration>
		</execution>
	</executions>
</plugin>
```

- 与 `maven-assembly-plugin` 的区别：shade 会重命名冲突的类（relocation），避免 JAR 冲突。

## 4.3 maven-assembly-plugin（自定义分发包）

- 适用于生成 tar.gz / zip 等分发归档包：

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-assembly-plugin</artifactId>
	<version>3.6.0</version>
	<configuration>
		<descriptorRefs>
			<descriptorRef>jar-with-dependencies</descriptorRef>
		</descriptorRefs>
		<archive>
			<manifest>
				<mainClass>com.example.Main</mainClass>
			</manifest>
		</archive>
	</configuration>
	<executions>
		<execution>
			<phase>package</phase>
			<goals>
				<goal>single</goal>
			</goals>
		</execution>
	</executions>
</plugin>
```

- 自定义 assembly descriptor（XML）可精确控制文件选择、目录结构和权限。

# 5. 单元测试与集成测试

- Surefire 默认匹配：`*Test`、`*Tests`、`*TestCase`。
- Failsafe 默认匹配：`*IT`、`*ITCase`。

跳过测试：
```bash
mvn -DskipTests package
```

完全跳过测试编译：
```bash
mvn -Dmaven.test.skip=true package
```

# 6. 编译与资源插件配置

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-compiler-plugin</artifactId>
	<version>3.11.0</version>
	<configuration>
		<source>21</source>
		<target>21</target>
	</configuration>
</plugin>
```

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-resources-plugin</artifactId>
	<version>3.3.1</version>
	<configuration>
		<encoding>UTF-8</encoding>
	</configuration>
</plugin>
```

# 7. 构建优化与企业实践

- 多核并行构建：
```bash
mvn -T 1C clean install
```

- 关闭下载进度日志：
```bash
mvn -ntp clean verify
```

- 使用父 POM 统一插件版本，避免不同模块版本漂移。
- 在 CI 中启用 `-B` 非交互模式。

# 8. 常见问题

- 插件版本缺失：企业环境建议统一在父 POM 中锁定版本。
- 构建慢：检查是否重复下载依赖或未使用私服。

# 9. 常见问题排查清单

- 测试未执行：检查测试类命名是否符合 Surefire 规则。
- 集成测试未执行：确认 Failsafe 是否绑定到 `integration-test`/`verify`。
- 插件配置不生效：确认插件配置放在 `build/plugins` 而非 `pluginManagement`。
- 编译版本不一致：检查 `maven-compiler-plugin` 的 `source/target`。
- 构建慢或超时：使用私服镜像、启用并行构建。

# 10. 实战案例：单测与集成测试分离

目标：单测在 `test` 阶段执行，集成测试在 `verify` 阶段执行。

配置：
```xml
<build>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-surefire-plugin</artifactId>
			<version>3.2.5</version>
		</plugin>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-failsafe-plugin</artifactId>
			<version>3.2.5</version>
			<executions>
				<execution>
					<goals>
						<goal>integration-test</goal>
						<goal>verify</goal>
					</goals>
				</execution>
			</executions>
		</plugin>
	</plugins>
</build>
```
