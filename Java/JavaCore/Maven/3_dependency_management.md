
# 1. 依赖范围（scope）

- `compile`：默认范围，编译/测试/运行均有效。
- `test`：仅测试编译和运行有效。
- `provided`：编译/测试有效，运行时由容器提供（如 Servlet API）。
- `runtime`：编译不需要，运行需要（如 JDBC 驱动）。
- `system`：本地路径依赖（不推荐）。
- `import`：仅用于导入 BOM（POM 类型）。

示例：
```xml
<dependency>
	<groupId>jakarta.servlet</groupId>
	<artifactId>jakarta.servlet-api</artifactId>
	<version>6.0.0</version>
	<scope>provided</scope>
</dependency>
```

## 1.1 system scope 陷阱

- `system` scope 依赖不会从仓库下载，而是使用本地绝对路径。
- **不推荐**的原因：
	- 路径硬编码，不跨平台。
	- 不会传递给下游模块。
	- `mvn package` 不会将该 JAR 打入选程包。
- 若必须使用，可通过 `maven-install-plugin` 将其安装到本地仓库后改为 `compile` scope。

```xml
<!-- 不推荐的写法 -->
<dependency>
	<groupId>com.oracle</groupId>
	<artifactId>ojdbc8</artifactId>
	<version>19.3</version>
	<scope>system</scope>
	<systemPath>${project.basedir}/lib/ojdbc8.jar</systemPath>
</dependency>
```

- 替代方案：将第三方 JAR 上传到私服，或在 POM 中声明私服仓库指向。

## 1.2 版本范围语法

- Maven 支持版本范围声明，但**企业项目不建议使用**（破坏构建可重复性）。
- 版本范围表达式：

| 表达式 | 含义 |
|--------|------|
| `[1.0,2.0)` | 1.0 <= x < 2.0 |
| `[1.0,2.0]` | 1.0 <= x <= 2.0 |
| `(1.0,2.0)` | 1.0 < x < 2.0 |
| `[1.0,)` | x >= 1.0 |
| `(,1.0]` | x <= 1.0 |
| `LATEST` | 最新版本（禁止在生产使用） |
| `RELEASE` | 最新发布版（禁止在生产使用） |

- 版本范围在传递依赖解析时可能导致不可预期的版本选择。
- 企业建议：固定版本，用 `maven-enforcer-plugin` 禁止范围版本。

# 2. 传递依赖与范围传递规则

- A 依赖 B，B 依赖 C，则 A 会传递引入 C。
- 传递规则要点：
	- `compile` 与 `runtime` 会传递。
	- `test` 与 `provided` 不会传递。

查看依赖树：
```bash
mvn dependency:tree
```

## 2.1 依赖仲裁（Dependency Mediation）算法详解

Maven 传递依赖冲突时，按以下规则决定最终使用的版本：

**规则一：最短路径优先（Nearest Wins）**
- A -> B -> C -> D(v1.0)   ← 路径长度 3
- A -> E -> D(v2.0)        ← 路径长度 2
- 结果：D(v2.0) 胜出（路径更短）

**规则二：路径相同时，先声明优先**
- A -> B -> D(v1.0)
- A -> C -> D(v2.0)
- 两条路径长度相同（都是 2），结果取决于 A 的 POM 中 B 和 C 谁先声明。

**依赖仲裁示意：**

```
A
├── B
│   └── D:v1  (路径: A->B->D, 长度=2)
└── C
    └── D:v2  (路径: A->C->D, 长度=2)

路径相同 → 看 POM 中 B 和 C 的声明顺序
若 B 先声明 → 使用 D:v1
若 C 先声明 → 使用 D:v2
```

**最佳实践：**
- 不要依赖仲裁规则，在父 POM 的 `dependencyManagement` 中显式指定版本。
- 使用 `maven-enforcer-plugin` 的 `dependencyConvergence` 规则强制版本统一。

# 3. 排除、optional、classifier 与 type

- `exclusions` 用于排除不需要的传递依赖。
- `optional` 表示可选依赖，不会被下游传递。
- `classifier` 用于区分构件变体（sources、javadoc）。
- `type` 可指定依赖类型（如 `pom`）。

排除示例：
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
	<exclusions>
		<exclusion>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-logging</artifactId>
		</exclusion>
	</exclusions>
</dependency>
```

# 4. BOM 与版本对齐

- BOM（Bill of Materials）用于统一一组依赖版本。
- 通过 `dependencyManagement` + `import` 引入。

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

# 5. 依赖冲突解决策略

- Maven 默认“最近优先”（依赖路径更短优先）。
- 若路径相同，按声明顺序优先。
- 常见解决方式：
	- 在 `dependencyManagement` 中强制版本
	- 使用 `exclusions` 移除冲突依赖
	- 统一依赖版本并锁定（企业父 POM）

查看冲突：
```bash
mvn dependency:tree -Dverbose
```

# 6. 依赖分析与治理

- 查看依赖列表：
```bash
mvn dependency:list
```

- 检测未使用或缺失依赖：
```bash
mvn dependency:analyze
```

- 强制依赖收敛与版本规则：
```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-enforcer-plugin</artifactId>
	<version>3.4.1</version>
	<executions>
		<execution>
			<goals>
				<goal>enforce</goal>
			</goals>
			<configuration>
				<rules>
					<dependencyConvergence />
				</rules>
			</configuration>
		</execution>
	</executions>
</plugin>
```

# 7. 企业常见问题与建议

- 版本漂移：统一用父 POM 锁版本。
- “Jar 冲突”导致运行异常：使用依赖树排查。
- 不要使用动态版本（`LATEST`、`RELEASE`）。
- 关键依赖建议在私服中做白名单控制。

# 8. 常见问题排查清单

- 运行时报 `ClassNotFound`：确认依赖 scope，`provided` 不会打包。
- 依赖版本异常：检查是否被传递依赖覆盖，查看依赖树。
- 排除不生效：确认 `exclusion` 的 `groupId`/`artifactId` 正确。
- 依赖不下载：检查仓库是否允许该版本，或私服是否缓存成功。
- 出现多个同名类：用 `mvn dependency:tree -Dverbose` 定位冲突。

# 9. 实战案例：依赖冲突治理

场景：A 模块依赖 `logback`，B 模块依赖 `log4j`，运行时冲突。

方案：
1. 在父 POM 中统一日志实现。
2. 对不需要的日志依赖进行排除。

示例：
```xml
<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>ch.qos.logback</groupId>
			<artifactId>logback-classic</artifactId>
			<version>1.5.6</version>
		</dependency>
	</dependencies>
</dependencyManagement>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
	<exclusions>
		<exclusion>
			<groupId>org.apache.logging.log4j</groupId>
			<artifactId>log4j-to-slf4j</artifactId>
		</exclusion>
	</exclusions>
</dependency>
```
