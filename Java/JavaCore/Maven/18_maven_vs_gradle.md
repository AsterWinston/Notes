
# 1. 核心理念对比

| 维度 | Maven | Gradle |
|------|-------|--------|
| 范式 | 声明式（XML） | 脚本式（Groovy / Kotlin DSL） |
| 构建模型 | 固定生命周期 + 阶段 | 有向无环任务图（DAG） |
| 配置文件 | `pom.xml` | `build.gradle` / `build.gradle.kts` |
| 扩展方式 | 插件 + 目标绑定到阶段 | 任务 + dependsOn / finalizedBy |
| 学习曲线 | 平缓——标准化约定 | 较陡——DSL 灵活性大 |
| 构建速度 | 中等（可并行） | 快（增量 + 缓存 + Daemon） |

# 2. 概念映射

| Maven | Gradle | 说明 |
|-------|--------|------|
| `pom.xml` | `build.gradle(.kts)` | 构建配置文件 |
| `groupId` + `artifactId` + `version` | `group` + `name` + `version` | 项目坐标 |
| `<parent>` | 继承 `from` 或 platform | 父配置 |
| `<modules>` | `settings.gradle.kts` 中 `include` | 多模块 |
| `dependencyManagement` | `dependencyManagement`（platform 插件） | 版本统一管理 |
| `mvn clean` | `gradle clean` | 清理 |
| `mvn compile` | `gradle compileJava` | 编译 |
| `mvn test` | `gradle test` | 测试 |
| `mvn package` | `gradle jar` / `gradle bootJar` | 打包 |
| `mvn install` | `gradle publishToMavenLocal` | 安装到本地仓库 |
| `mvn deploy` | `gradle publish` | 发布到远程仓库 |
| Profile | Build Variant / `-P` 参数 | 环境构建 |
| `META-INF/maven` | `META-INF/gradle` | 元信息目录 |

# 3. POM 到 Gradle 对照

## 3.1 Maven (pom.xml)

```xml
<project>
	<groupId>com.example</groupId>
	<artifactId>demo</artifactId>
	<version>1.0.0</version>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
			<version>3.3.4</version>
		</dependency>
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>4.13.2</version>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```

## 3.2 Gradle (build.gradle.kts)

```kotlin
plugins {
	java
	id("org.springframework.boot") version "3.3.4"
	id("io.spring.dependency-management") version "1.1.6"
}

group = "com.example"
version = "1.0.0"

repositories {
	mavenCentral()
}

dependencies {
	implementation("org.springframework.boot:spring-boot-starter-web")
	testImplementation("junit:junit:4.13.2")
}
```

# 4. 多模块对照

## 4.1 Maven

```xml
<!-- 父 POM -->
<modules>
	<module>common</module>
	<module>web</module>
</modules>
```

```xml
<!-- 子模块 POM -->
<parent>
	<groupId>com.example</groupId>
	<artifactId>parent</artifactId>
	<version>1.0.0</version>
</parent>
<artifactId>web</artifactId>
```

## 4.2 Gradle

```kotlin
// settings.gradle.kts
rootProject.name = "parent"
include("common", "web")
```

```kotlin
// web/build.gradle.kts
dependencies {
	implementation(project(":common"))
}
```

# 5. 从 Maven 迁移到 Gradle

## 5.1 自动迁移

```bash
# 在 Maven 项目目录中执行
gradle init
```

- Gradle 会读取 `pom.xml`，自动生成 `build.gradle` 和 `settings.gradle`。
- 支持单模块和多模块项目。

## 5.2 迁移检查清单

| 检查项 | Maven | Gradle |
|--------|-------|--------|
| 基本坐标 | GAV | group/name/version |
| 依赖 | `<dependencies>` | `dependencies {}` |
| BOM | `<dependencyManagement>` import | `platform()` |
| 编译配置 | `maven-compiler-plugin` | `java.toolchain` |
| 资源过滤 | `<resources>` + `<filtering>` | `processResources {}` |
| Profile | `<profiles>` | 条件逻辑或 `-P` |
| 测试 | `surefire` | `test {}` + `useJUnitPlatform()` |
| 打包 | `maven-jar-plugin` | `jar {}` / `bootJar {}` |
| 发布 | `distributionManagement` | `publishing {}` |

## 5.3 渐进式迁移策略

1. **先跑通 CI**：在 CI 中同时运行 `mvn` 和 `gradle`，对比产物一致。
2. **模块优先级**：先迁移 leaf 模块（无下游依赖），逐步向 core 推进。
3. **保留 Maven 发布格式**：Gradle 可以发布到 Maven 仓库，与其余 Maven 模块兼容。
4. **混合构建**：短期内可使用 `maven-publish` 插件让 Gradle 模块发布产物给 Maven 模块消费。

# 6. 何时不要迁移

- 团队 Maven 经验丰富，无性能瓶颈。
- 多模块依赖复杂，迁移成本高。
- 已有成熟的 Maven 父 POM 和私服体系。
- 构建时间在可接受范围内（< 5 分钟）。

# 7. Gradle 独有优势

- **增量编译**：只编译变更的 Java 文件。
- **构建缓存**：跨项目共享Task输出。
- **Configuration Cache**：跳过配置阶段，直接执行。
- **复合构建**：多个独立 Gradle 项目联合构建。
- **Kotlin DSL**：类型安全的构建脚本，IDE 智能提示。
- **更好的 Monorepo 支持**：无需固定父子关系。

# 8. 常见问题排查清单

- 迁移后依赖下载失败：检查 Gradle 仓库配置与 Maven 私服地址是否一致。
- 插件行为不一致：确认 Gradle 插件版本与 Maven 插件版本对齐。
- 资源过滤失效：使用 `processResources` task 的 `expand()` 方法替代。
- 版本管理混乱：使用 Gradle Version Catalog（`libs.versions.toml`）统一管理。
