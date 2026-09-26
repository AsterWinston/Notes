
自定义 Gradle 插件是将可复用的构建逻辑打包、分发和共享的核心机制。无论是封装公司内部的最佳实践，还是为开源社区贡献构建工具，理解插件开发的完整生命周期至关重要。

# 1. 为什么需要自定义插件

## 1.1 插件的核心价值

在 Gradle 构建脚本中，随着项目规模增长和模块数量增多，构建脚本中会出现大量重复代码。自定义插件解决以下问题：

| 问题 | 不使用插件的表现 | 使用插件后的效果 |
|------|---------------|---------------|
| 代码重复 | 每个模块复制相同的任务配置、依赖声明 | 一行 `apply plugin` 即可 |
| 维护困难 | 修改构建逻辑需要更新所有模块 | 修改插件一处，所有模块自动生效 |
| 一致性缺失 | 不同模块的编译选项、代码风格不同 | 插件强制执行统一的标准 |
| 知识分散 | 构建最佳实践分散在文档和脚本中 | 插件编码化和类型化最佳实践 |
| 测试缺失 | 构建脚本变化缺少自动化验证 | 插件可通过 TestKit 编写测试 |

## 1.2 插件实现方式的演进

```
Groovy build.gradle 内联逻辑
    │ (简单，但不可测试，不可复用)
    ▼
buildSrc 中 Groovy 脚本
    │ (可复用，可测试，但类型不安全)
    ▼
buildSrc 中 Precompiled Script Plugin
    │ (类型安全，IDE 支持好，但灵活性有限)
    ▼
独立插件项目 (java-gradle-plugin)
    │ (完全隔离，可发布，可测试，最大的灵活性)
    ▼
复合构建 (Composite Build) 引入的插件
    (开发时即插即用，无需发布到仓库)
```

# 2. 插件项目结构

## 2.1 buildSrc 方式

```
my-project/
├── buildSrc/                          # 特殊的自动包含模块
│   ├── build.gradle.kts               # buildSrc 自身的构建脚本
│   └── src/
│       └── main/
│           ├── kotlin/
│           │   └── com/example/
│           │       ├── MyPlugin.kt                    # Plugin 实现类
│           │       └── MyExtension.kt                 # 扩展对象
│           └── resources/
│               └── META-INF/
│                   └── gradle-plugins/
│                       └── com.example.my-plugin.properties  # 插件注册
```

buildSrc 自身配置 (buildSrc/build.gradle.kts):

```kotlin
plugins {
    `kotlin-dsl`      // Kotlin DSL 内置插件, 提供 Gradle API 类型提示
    `java-gradle-plugin`
}

repositories {
    mavenCentral()
    gradlePluginPortal()
}

dependencies {
    // 插件开发需要的依赖
    implementation("com.squareup:javapoet:1.13.0")
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.2")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

插件注册文件 (`META-INF/gradle-plugins/com.example.my-plugin.properties`):

```properties
implementation-class=com.example.MyPlugin
```

## 2.2 独立插件项目

```
my-gradle-plugin/
├── build.gradle.kts
├── settings.gradle.kts
├── src/
│   ├── main/
│   │   ├── kotlin/
│   │   │   └── com/example/gradle/
│   │   │       ├── MyPlugin.kt
│   │   │       ├── MyExtension.kt
│   │   │       └── MyTask.kt
│   │   └── resources/
│   │       └── META-INF/
│   │           └── gradle-plugins/
│   │               └── com.example.my-plugin.properties
│   ├── test/
│   │   └── kotlin/
│   │       └── com/example/gradle/
│   │           └── MyPluginTest.kt
│   └── functionalTest/            # 功能测试 (可选)
│       └── kotlin/
│           └── com/example/gradle/
│               └── MyPluginFunctionalTest.kt
```

独立插件项目的 build.gradle.kts:

```kotlin
plugins {
    `kotlin-dsl`
    `java-gradle-plugin`
    `maven-publish`
    id("com.gradle.plugin-publish") version "1.2.1"
}

group = "com.example"
version = "1.0.0"

gradlePlugin {
    plugins {
        create("myPlugin") {
            id = "com.example.my-plugin"
            implementationClass = "com.example.gradle.MyPlugin"
            displayName = "My Custom Gradle Plugin"
            description = "A Gradle plugin that provides custom build logic"
            tags.set(listOf("example", "custom", "build"))
        }
    }
}

// Plugin Portal 发布配置
pluginBundle {
    website = "https://github.com/example/my-gradle-plugin"
    vcsUrl = "https://github.com/example/my-gradle-plugin.git"
}

publishing {
    repositories {
        maven {
            url = uri("https://nexus.company.com/repository/maven-releases/")
            credentials {
                username = System.getenv("NEXUS_USER")
                password = System.getenv("NEXUS_PASS")
            }
        }
    }
}
```

## 2.3 复合构建方式

复合构建 (Composite Build) 允许在开发阶段直接引用本地插件项目，无需发布：

```
my-application/                       # 主项目
├── settings.gradle.kts
├── build.gradle.kts
└── ...

my-gradle-plugin/                     # 插件项目 (独立仓库)
├── settings.gradle.kts
├── build.gradle.kts
└── src/...
```

```kotlin
// my-application/settings.gradle.kts
pluginManagement {
    includeBuild("../my-gradle-plugin")       // 复合构建引入
}
```

```kotlin
// my-application/build.gradle.kts
plugins {
    id("com.example.my-plugin") version "1.0.0"  // 使用本地插件
}
```

复合构建的优势：
- 修改插件后，主项目立即生效 (无需发布)
- 适合插件开发和调试阶段
- 可以在主项目中打断点调试插件代码

# 3. 预编译脚本插件 (Precompiled Script Plugins)

## 3.1 Kotlin DSL 预编译脚本插件

预编译脚本插件是 Groovy DSL 到 Kotlin DSL 再到完整插件之间的中间形态。它让普通的 `*.gradle.kts` 文件可被其他模块引用。

```kotlin
// buildSrc/src/main/kotlin/java-convention.gradle.kts
plugins {
    java
    checkstyle
    jacoco
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(17))
    }
    withSourcesJar()
    withJavadocJar()
}

checkstyle {
    maxWarnings = 0
    toolVersion = "10.15.0"
    configFile = rootProject.file("config/checkstyle/checkstyle.xml")
}

jacoco {
    toolVersion = "0.8.12"
}

tasks.withType<JavaCompile> {
    options.encoding = "UTF-8"
    options.compilerArgs.addAll(listOf("-Xlint:unchecked", "-Xlint:deprecation"))
}

tasks.withType<Test> {
    useJUnitPlatform()
    maxHeapSize = "1g"
    testLogging {
        events("passed", "skipped", "failed")
    }
}

tasks.named<Jar>("jar") {
    manifest {
        attributes(
            "Implementation-Title" to project.name,
            "Implementation-Version" to project.version,
            "Built-By" to System.getProperty("user.name"),
            "Built-JDK" to System.getProperty("java.version")
        )
    }
}
```

引用预编译脚本插件：

```kotlin
// 子模块 build.gradle.kts
plugins {
    id("java-convention")     // 自动发现 buildSrc 中的插件
}

dependencies {
    implementation("com.google.guava:guava:33.1.0-jre")
}
```

## 3.2 多约定插件实践

```
buildSrc/src/main/kotlin/
├── java-convention.gradle.kts        # Java 基础约定
├── spring-boot-convention.gradle.kts # Spring Boot 约定
├── library-convention.gradle.kts     # 库发布约定
├── docker-convention.gradle.kts      # Docker 镜像约定
└── testing-convention.gradle.kts     # 测试约定
```

```kotlin
// spring-boot-convention.gradle.kts
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
    id("java-convention")            // 组合基础约定
}

dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:2023.0.2")
    }
}

springBoot {
    buildInfo()
}

tasks.named<BootJar>("bootJar") {
    layered {
        enabled.set(true)
    }
}
```

```kotlin
// library-convention.gradle.kts
plugins {
    `maven-publish`
    signing
    id("java-convention")
}

publishing {
    publications {
        create<MavenPublication>("mavenJava") {
            from(components["java"])
            pom {
                name.set("My Library")
                description.set("A reusable Java library")
                url.set("https://github.com/example/my-library")
                licenses {
                    license {
                        name.set("Apache-2.0")
                        url.set("https://www.apache.org/licenses/LICENSE-2.0")
                    }
                }
            }
        }
    }
}
```

```kotlin
// 子模块中使用组合约定
plugins {
    id("spring-boot-convention")
    id("docker-convention")
}
```

预编译脚本插件 ID 生成规则：

```
文件路径                          -> 插件 ID
├── java-convention.gradle.kts    -> java-convention
├── spring-boot-convention.gradle.kts -> spring-boot-convention
├── my/custom.gradle.kts          -> my.custom
└── my-plugin/plugin.gradle.kts   -> my-plugin.plugin
```

## 3.3 Groovy DSL 预编译脚本插件

```groovy
// buildSrc/src/main/groovy/my-java-convention.gradle
plugins {
    id 'java'
    id 'checkstyle'
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

tasks.withType(JavaCompile).configureEach {
    options.encoding = 'UTF-8'
    options.compilerArgs += ['-Xlint:unchecked', '-Xlint:deprecation']
}
```

# 4. 实现 Plugin 接口

## 4.1 Plugin 接口基础

```java
// Java 实现
package com.example.gradle;

import org.gradle.api.Plugin;
import org.gradle.api.Project;

public class MyPlugin implements Plugin<Project> {
    @Override
    public void apply(Project project) {
        System.out.println("MyPlugin applied to project: " + project.getName());
        
        // 注册扩展
        project.getExtensions().create("myConfig", MyExtension.class);
        
        // 注册任务
        project.getTasks().register("myTask", MyTask.class, task -> {
            task.getOutputFile().set(
                project.getLayout().getBuildDirectory().file("my-output.txt")
            );
        });
    }
}
```

```kotlin
// Kotlin 实现 (推荐)
package com.example.gradle

import org.gradle.api.Plugin
import org.gradle.api.Project

class MyPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        println("MyPlugin applied to project: ${project.name}")

        // 注册扩展
        val extension = project.extensions.create("myConfig", MyExtension::class.java)

        // 注册任务
        project.tasks.register("myTask", MyTask::class.java) {
            it.outputFile.set(
                project.layout.buildDirectory.file("my-output.txt")
            )
        }

        // 在评估后配置任务 (连接扩展值到任务)
        project.afterEvaluate {
            // 注意: afterEvaluate 与配置缓存不兼容
            // 推荐使用 Provider 延迟配置
        }
    }
}
```

## 4.2 Plugin 对 Settings 的支持

Gradle 6.0+ 支持 Settings 级别的插件：

```kotlin
// Settings 插件 (应用于 settings.gradle.kts)
class MySettingsPlugin : Plugin<Settings> {
    override fun apply(settings: Settings) {
        println("Applied to settings: ${settings.rootProject.name}")

        // 配置仓库
        settings.pluginManagement {
            repositories {
                mavenCentral()
                gradlePluginPortal()
            }
        }

        // 动态包含子项目
        settings.gradle.settingsEvaluated {
            // 自定义子项目发现逻辑
        }
    }
}

// 注册 (settings.gradle.kts)
// plugins { id("my-settings-plugin") }
```

## 4.3 插件注册方式对比

| 方式 | 位置 | 用途 | IDE 支持 |
|------|------|------|:------:|
| `META-INF/gradle-plugins/*.properties` | 独立或 buildSrc 插件 | 传统注册方式 | ✓ |
| `gradlePlugin {}` DSL | build.gradle.kts | 类型安全注册 | ✓ |
| buildSrc auto-detection | buildSrc/ | 项目内约定插件 | ✓ |
| `pluginManagement {}` | settings.gradle.kts | 管理插件仓库和版本 | N/A |

# 5. 扩展 (Extension) 创建

## 5.1 使用 ExtensionContainer

```kotlin
// 定义扩展对象
abstract class MyExtension {
    // 字符串属性
    abstract val message: Property<String>

    // 布尔属性
    abstract val enabled: Property<Boolean>

    // 整数属性
    abstract val retryCount: Property<Int>

    // 文件属性
    abstract val configFile: RegularFileProperty

    // 目录属性
    abstract val outputDir: DirectoryProperty

    // 嵌套对象
    abstract val server: ServerConfig
}

abstract class ServerConfig {
    abstract val host: Property<String>
    abstract val port: Property<Int>
    abstract val useSSL: Property<Boolean>
}
```

```kotlin
// 在插件中注册扩展
class MyPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        val extension = project.extensions.create(
            "myConfig",           // 扩展名称, 在 build.gradle.kts 中通过 myConfig {} 使用
            MyExtension::class.java
        )

        // 注册任务, 延迟读取扩展值 (配置避免)
        project.tasks.register("configuredTask", ConfiguredTask::class.java) {
            it.message.set(extension.message)
            it.configFile.set(extension.configFile)
        }
    }
}
```

使用者配置扩展：

```kotlin
// build.gradle.kts
myConfig {
    message.set("Hello from config")
    enabled.set(true)
    retryCount.set(3)
    configFile.set(file("config/application.yml"))
    outputDir.set(layout.buildDirectory.dir("generated-config"))
    server {
        host.set("localhost")
        port.set(8080)
        useSSL.set(false)
    }
}
```

## 5.2 NamedDomainObjectContainer (命名域对象容器)

当扩展需要多个命名配置时，使用 `NamedDomainObjectContainer`：

```kotlin
// 可命名对象
abstract class DataSourceConfig @javax.inject.Inject constructor(
    private val name: String
) : Named {
    override fun getName(): String = name

    abstract val url: Property<String>
    abstract val username: Property<String>
    abstract val password: Property<String>
    abstract val poolSize: Property<Int>
}

// 在插件中创建容器
class MultiDatasourcePlugin : Plugin<Project> {
    override fun apply(project: Project) {
        val container = project.container(DataSourceConfig::class.java) { name ->
            project.objects.newInstance(DataSourceConfig::class.java, name)
        }

        project.extensions.add("dataSources", container)

        // 为每个 DataSource 创建任务
        container.all { dataSource ->
            project.tasks.register("validate-${dataSource.name}-datasource") {
                doLast {
                    println("Validating datasource: ${dataSource.url.get()}")
                }
            }
        }
    }
}
```

使用者配置：

```kotlin
// build.gradle.kts
dataSources {
    register("primary") {
        url.set("jdbc:postgresql://primary-db:5432/main")
        username.set("admin")
        password.set("secret")
        poolSize.set(10)
    }
    register("analytics") {
        url.set("jdbc:postgresql://read-replica:5432/analytics")
        username.set("reader")
        password.set("readonly")
        poolSize.set(5)
    }
}
```

## 5.3 扩展属性延迟绑定与映射

```kotlin
class MyPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        val extension = project.extensions.create("myConfig", MyExtension::class.java)

        // 设置默认值 (使用 convention, 可以被用户覆盖)
        extension.message.convention("Default message")
        extension.enabled.convention(true)
        extension.retryCount.convention(5)

        extension.server.host.convention("localhost")
        extension.server.port.convention(8080)

        // 值映射: 根据 message 值动态生成其他属性
        val derivedValue = extension.message.map { msg ->
            "DERIVED-${msg.uppercase()}"
        }

        // 使用映射后的值
        project.tasks.register("printConfig") {
            doLast {
                println("Message: ${extension.message.get()}")
                println("Derived: ${derivedValue.get()}")
            }
        }
    }
}
```

# 6. 任务配置与实现

## 6.1 自定义任务类型

```kotlin
// 带缓存支持的任务
abstract class GenerateReportTask : DefaultTask() {

    @get:Input
    abstract val reportType: Property<String>

    @get:InputFile
    abstract val dataFile: RegularFileProperty

    @get:InputFiles
    abstract val templates: ConfigurableFileCollection

    @get:Optional
    @get:Input
    abstract val author: Property<String>

    @get:OutputFile
    abstract val outputFile: RegularFileProperty

    @get:Internal   // 不参与 up-to-date 检查
    abstract val logger: Property<String>

    init {
        group = "reporting"
        description = "Generates a custom report from data and templates"
    }

    @TaskAction
    fun generate() {
        val type = reportType.get()
        val data = dataFile.get().asFile.readText()
        val output = outputFile.get().asFile

        output.parentFile.mkdirs()
        output.writeText("""
            |Report Type: $type
            |Generated by: ${author.getOrElse("Unknown")}
            |Data: $data
            |Timestamp: ${System.currentTimeMillis()}
        """.trimMargin())

        logger.quiet("Generated report: ${output.absolutePath}")
    }
}
```

## 6.2 任务注册与配置

```kotlin
class ReportPlugin : Plugin<Project> {
    override fun apply(project: Project) {

        // 方式一: register + configure
        val generateTask = project.tasks.register(
            "generateReport",
            GenerateReportTask::class.java
        ) {
            it.reportType.set("daily")
            it.dataFile.set(project.layout.projectDirectory.file("data/input.csv"))
            it.templates.from(project.layout.projectDirectory.dir("templates"))
            it.outputFile.set(
                project.layout.buildDirectory.file("reports/daily-report.txt")
            )
        }

        // 方式二: 创建后配置 (延迟配置)
        project.tasks.register("generateMonthlyReport") {
            it.dependsOn(generateTask)

            it.doLast {
                // 使用自定义 action
                println("Monthly report generation triggered")
            }
        }

        // 方式三: 配置已有任务
        project.tasks.named("jar") {
            it.doFirst {
                println("Building JAR...")
            }
        }

        // 方式四: 配置所有匹配类型的任务
        project.tasks.withType<GenerateReportTask> {
            // 为所有报告任务设置共同的输出目录
            it.outputFile.convention(
                project.layout.buildDirectory.file("reports/generated.txt")
            )
        }
    }
}
```

## 6.3 Worker API (并行工作)

对于耗时操作，使用 Worker API 在独立 JVM 进程中执行：

```kotlin
// Worker 实现
abstract class DataProcessorWorker : WorkAction<DataProcessorWorker.Parameters> {
    interface Parameters : WorkParameters {
        val inputFile: RegularFileProperty
        val outputFile: RegularFileProperty
        val mode: Property<String>
    }

    override fun execute() {
        val input = parameters.inputFile.get().asFile
        val output = parameters.outputFile.get().asFile
        val mode = parameters.mode.get()

        // 执行耗时操作
        val result = when (mode) {
            "compress" -> compress(input)
            "encrypt" -> encrypt(input)
            else -> input.readText()
        }

        output.parentFile.mkdirs()
        output.writeText(result)
    }

    private fun compress(file: File): String {
        // 压缩逻辑
        return file.readText().reversed()
    }

    private fun encrypt(file: File): String {
        // 加密逻辑
        return file.readText().toCharArray().map { it + 1 }.joinToString("")
    }
}

// 任务中使用 Worker API
abstract class ProcessDataTask : DefaultTask() {

    @get:InputFile
    abstract val dataFile: RegularFileProperty

    @get:OutputFile
    abstract val resultFile: RegularFileProperty

    @get:Input
    abstract val mode: Property<String>

    @get:Inject
    abstract val workerExecutor: WorkerExecutor

    @TaskAction
    fun process() {
        val workQueue = workerExecutor.processIsolation { workerSpec ->
            workerSpec.classpath.from(
                javaClass.classLoader.getResource("worker-deps.jar").toURI()
            )
            workerSpec.forkOptions {
                maxHeapSize = "512m"
                jvmArgs("-XX:+UseG1GC")
            }
        }

        workQueue.submit(DataProcessorWorker::class.java) { config ->
            config.inputFile.set(dataFile)
            config.outputFile.set(resultFile)
            config.mode.set(mode)
        }

        workQueue.await()  // 等待所有 worker 完成
    }
}
```

# 7. 插件配置避免 (Configuration Avoidance)

## 7.1 Provider 与 Property

Gradle 的惰性配置 API 是插件性能优化的核心：

```kotlin
// Provider: 只读值提供者, 延迟计算
// Property: 可写属性, 可设置值或约定值

// 属性创建方式
abstract class MyPluginExtension {
    // 使用 abstract val, 由 Gradle 自动创建实例
    abstract val name: Property<String>                     // 可写
    abstract val version: Property<String>                  // 可写
    abstract val configFile: RegularFileProperty            // 文件属性
    abstract val outputDir: DirectoryProperty               // 目录属性
    abstract val enabled: Property<Boolean>                 // 布尔属性
    abstract val maxRetries: Property<Int>                  // 整数属性
}

// Provider 使用示例
class MyPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        val extension = project.extensions.create("myConfig", MyPluginExtension::class.java)

        // Provider 映射 (不触发求值)
        val fullName = extension.name.map { "app-$it" }
        val configExists = extension.configFile.map { it.asFile.exists() }

        // Provider flatMap (链式依赖)
        val absoluteVersion = extension.version.flatMap { v ->
            project.providers.provider { "$v-${project.version}" }
        }

        // 在任务中使用 Provider (延迟绑定)
        project.tasks.register("configureTask") {
            doLast {
                // 仅在执行阶段求值 (符合配置避免原则)
                println("Full name: ${fullName.get()}")
                println("Config exists: ${configExists.get()}")
            }
        }

        // DirectoryProperty 的 Provider 操作
        val jarFile = extension.outputDir.file("app.jar")
        val libDir = extension.outputDir.dir("lib")
    }
}
```

## 7.2 Convention (约定值) 与 Finalize (定稿)

```kotlin
// Convention: 默认值, 可被用户覆盖
extension.name.convention("default-app")
extension.version.convention(project.version.toString())
extension.maxRetries.convention(3)

// Finalize: 锁定值, 后续修改会报错
// extension.name.finalizeValue()
// extension.maxRetries.finalizeValue()

// Finalize on Read: 读取后锁定
extension.name.finalizeValueOnRead()
```

## 7.3 针对延迟配置的模式总结

```kotlin
// 推荐模式 (配置避免友好)
project.tasks.register("myTask", MyTask::class.java) { task ->
    task.someProperty.set(extension.someValue)        // Provider 到 Property
    task.otherProperty.set(provider.map { ... })      // 映射 Provider
}

// 反模式 (破坏配置避免)
project.tasks.create("myTask") { task ->
    task.someProperty = extension.someValue.get()     // 立即求值
}

// 反模式 (配置缓存不兼容)
project.afterEvaluate {
    project.tasks.named("myTask") { task ->
        task.someProperty = "computed value"           // 时序依赖
    }
}
```

# 8. 插件测试

## 8.1 Gradle TestKit 基础

```kotlin
// build.gradle.kts — 添加 TestKit 依赖
dependencies {
    testImplementation(gradleTestKit())
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.2")
    testImplementation("org.assertj:assertj-core:3.25.3")
}
```

```kotlin
// MyPluginTest.kt — 基本 TestKit 测试
import org.gradle.testkit.runner.GradleRunner
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.io.TempDir
import java.io.File
import kotlin.test.assertTrue

class MyPluginTest {

    @TempDir
    lateinit var testProjectDir: File

    private fun setupProject(): File {
        // 创建 settings.gradle.kts
        File(testProjectDir, "settings.gradle.kts").writeText("""
            rootProject.name = "test-project"
        """.trimIndent())

        // 创建 build.gradle.kts
        File(testProjectDir, "build.gradle.kts").writeText("""
            plugins {
                id("com.example.my-plugin")
            }
            
            myConfig {
                message.set("Test Message")
                enabled.set(true)
                retryCount.set(3)
            }
        """.trimIndent())

        // 创建必要的源码目录
        File(testProjectDir, "src/main/java").mkdirs()
        File(testProjectDir, "src/main/java/App.java").writeText("""
            public class App {
                public static void main(String[] args) {
                    System.out.println("Hello, World!");
                }
            }
        """.trimIndent())

        return testProjectDir
    }

    @Test
    fun `plugin applies successfully`() {
        setupProject()

        val result = GradleRunner.create()
            .withProjectDir(testProjectDir)
            .withPluginClasspath()              // 自动包含当前项目的插件类路径
            .withArguments("tasks")
            .build()

        assertTrue(result.output.contains("myTask"))
        assertTrue(result.output.contains("BUILD SUCCESSFUL"))
    }

    @Test
    fun `plugin task produces expected output`() {
        setupProject()

        val result = GradleRunner.create()
            .withProjectDir(testProjectDir)
            .withPluginClasspath()
            .withArguments("myTask", "--info")
            .build()

        val outputFile = File(testProjectDir, "build/my-output.txt")
        assertTrue(outputFile.exists())
        assertTrue(outputFile.readText().contains("Test Message"))
    }

    @Test
    fun `plugin fails with invalid config`() {
        setupProject()

        // 修改配置为无效值
        File(testProjectDir, "build.gradle.kts").writeText("""
            plugins {
                id("com.example.my-plugin")
            }
            myConfig {
                enabled.set(false)  // 导致任务失败
            }
        """.trimIndent())

        val result = GradleRunner.create()
            .withProjectDir(testProjectDir)
            .withPluginClasspath()
            .withArguments("myTask")
            .buildAndFail()                     // 预期构建失败

        assertTrue(result.output.contains("FAILED"))
    }
}
```

## 8.2 TestKit 进阶功能

```kotlin
class PluginAdvancedTest {

    @TempDir
    lateinit var testProjectDir: File

    @Test
    fun `test with custom Gradle version`() {
        setupSimpleProject()

        val result = GradleRunner.create()
            .withProjectDir(testProjectDir)
            .withPluginClasspath()
            .withGradleVersion("8.7")          // 指定 Gradle 版本
            .withArguments("build")
            .build()

        assertEquals(TaskOutcome.SUCCESS, result.task(":build")?.outcome)
    }

    @Test
    fun `test with debug logging`() {
        setupSimpleProject()

        val result = GradleRunner.create()
            .withProjectDir(testProjectDir)
            .withPluginClasspath()
            .withArguments("tasks", "--debug")
            .forwardOutput()                    // 转发输出到测试进程
            .build()
    }

    @Test
    fun `test multi-module project`() {
        // 设置多模块项目
        File(testProjectDir, "settings.gradle.kts").writeText("""
            rootProject.name = "multi-test"
            include("core", "app")
        """.trimIndent())

        // core 模块
        File(testProjectDir, "core/build.gradle.kts").writeText("""
            plugins {
                id("com.example.my-plugin")
            }
        """.trimIndent())
        File(testProjectDir, "core/src/main/java").mkdirs()

        // app 模块
        File(testProjectDir, "app/build.gradle.kts").writeText("""
            plugins {
                id("com.example.my-plugin")
            }
            dependencies {
                implementation(project(":core"))
            }
        """.trimIndent())
        File(testProjectDir, "app/src/main/java").mkdirs()

        val result = GradleRunner.create()
            .withProjectDir(testProjectDir)
            .withPluginClasspath()
            .withArguments("build")
            .build()

        assertEquals(TaskOutcome.SUCCESS, result.task(":core:build")?.outcome)
        assertEquals(TaskOutcome.SUCCESS, result.task(":app:build")?.outcome)
    }

    @Test
    fun `test configuration cache compatibility`() {
        setupSimpleProject()

        // 首次运行 (填充配置缓存)
        GradleRunner.create()
            .withProjectDir(testProjectDir)
            .withPluginClasspath()
            .withArguments("help", "--configuration-cache")
            .build()

        // 第二次运行 (使用配置缓存)
        val result = GradleRunner.create()
            .withProjectDir(testProjectDir)
            .withPluginClasspath()
            .withArguments("help", "--configuration-cache")
            .build()

        assertTrue(
            result.output.contains("Reusing configuration cache"),
            "Expected configuration cache to be reused"
        )
    }
}
```

## 8.3 多版本 Gradle 兼容性矩阵测试

```kotlin
class PluginCompatibilityTest {

    @TempDir
    lateinit var testProjectDir: File

    // 参数化测试: 在不同 Gradle 版本下验证插件
    @ParameterizedTest
    @ValueSource(strings = ["7.6.4", "8.0.2", "8.5", "8.7", "8.8"])
    fun `plugin works with Gradle version`(gradleVersion: String) {
        File(testProjectDir, "settings.gradle.kts").writeText("""
            rootProject.name = "compat-test"
        """.trimIndent())

        File(testProjectDir, "build.gradle.kts").writeText("""
            plugins {
                id("com.example.my-plugin")
            }
        """.trimIndent())

        try {
            val result = GradleRunner.create()
                .withProjectDir(testProjectDir)
                .withPluginClasspath()
                .withGradleVersion(gradleVersion)
                .withArguments("tasks")
                .build()

            assertTrue(result.output.contains("BUILD SUCCESSFUL"))
        } catch (e: Exception) {
            // 记录但不中断, 用于生成兼容性报告
            System.err.println("FAILED with Gradle $gradleVersion: ${e.message}")
        }
    }
}
```

# 9. 插件发布

## 9.1 发布到 Maven 本地仓库

```bash
# 发布到本地 Maven 仓库 (~/.m2/repository/)
./gradlew publishToMavenLocal
```

```groovy
// 使用者 build.gradle
buildscript {
    repositories {
        mavenLocal()
        mavenCentral()
    }
    dependencies {
        classpath 'com.example:my-gradle-plugin:1.0.0'
    }
}

apply plugin: 'com.example.my-plugin'
```

## 9.2 发布到企业 Maven 仓库

```kotlin
// build.gradle.kts
plugins {
    `maven-publish`
}

publishing {
    repositories {
        maven {
            name = "CompanyNexus"
            url = uri(
                if (version.toString().endsWith("SNAPSHOT"))
                    "https://nexus.company.com/repository/maven-snapshots/"
                else
                    "https://nexus.company.com/repository/maven-releases/"
            )
            credentials {
                username = System.getenv("NEXUS_USER") ?: project.findProperty("nexusUser") as String?
                password = System.getenv("NEXUS_PASSWORD") ?: project.findProperty("nexusPassword") as String?
            }
        }
    }

    publications {
        create<MavenPublication>("pluginMaven") {
            groupId = "com.example"
            artifactId = "my-gradle-plugin"
            version = project.version.toString()

            pom {
                name.set("My Gradle Plugin")
                description.set("A custom Gradle plugin for enterprise use")
                url.set("https://github.com/example/my-gradle-plugin")
                developers {
                    developer {
                        id.set("team")
                        name.set("Platform Team")
                        email.set("platform@company.com")
                    }
                }
            }
        }
    }
}
```

## 9.3 发布到 Gradle Plugin Portal

```properties
# gradle.properties
gradle.publish.key=your-publish-key-here
gradle.publish.secret=your-publish-secret-here
```

```kotlin
// build.gradle.kts
plugins {
    id("com.gradle.plugin-publish") version "1.2.1"
}

group = "com.example"
version = "1.0.0"

gradlePlugin {
    website.set("https://github.com/example/my-gradle-plugin")
    vcsUrl.set("https://github.com/example/my-gradle-plugin.git")

    plugins {
        create("myPlugin") {
            id = "com.example.my-plugin"
            implementationClass = "com.example.gradle.MyPlugin"
            displayName = "My Custom Gradle Plugin"
            description = "A feature-rich Gradle plugin for enterprise Java projects"
            tags.set(listOf("java", "enterprise", "code-generation", "reporting"))
        }
    }
}
```

```bash
# 发布到 Gradle Plugin Portal
./gradlew publishPlugins

# 发布后, 使用者可直接引用 (无需配置仓库)
# plugins { id("com.example.my-plugin") version "1.0.0" }
```

## 9.4 发布到 GitHub Packages

```kotlin
publishing {
    repositories {
        maven {
            name = "GitHubPackages"
            url = uri("https://maven.pkg.github.com/example/my-gradle-plugin")
            credentials {
                username = System.getenv("GITHUB_ACTOR")
                password = System.getenv("GITHUB_TOKEN")
            }
        }
    }
}
```

使用者配置：

```kotlin
// settings.gradle.kts
pluginManagement {
    repositories {
        maven {
            url = uri("https://maven.pkg.github.com/example/my-gradle-plugin")
            credentials {
                username = System.getenv("GITHUB_ACTOR")
                password = System.getenv("GITHUB_TOKEN")
            }
        }
        gradlePluginPortal()
    }
}
```

# 10. java-gradle-plugin 插件

## 10.1 自动生成的配置

`java-gradle-plugin` 插件提供以下自动化功能：

```kotlin
plugins {
    `java-gradle-plugin`
}

// 自动生成：
// 1. gradlePlugin {} DSL
// 2. gradleApi() 依赖
// 3. gradleTestKit() 依赖
// 4. pluginUnderTestMetadata 任务
// 5. META-INF/gradle-plugins 自动生成 (通过 DSL 声明时)

// gradlePlugin {} DSL 自动注册插件
gradlePlugin {
    plugins {
        create("myPlugin") {
            id = "com.example.my-plugin"
            implementationClass = "com.example.gradle.MyPlugin"
        }
    }
}

// 等价于手动创建:
// src/main/resources/META-INF/gradle-plugins/com.example.my-plugin.properties
// implementation-class=com.example.gradle.MyPlugin
```

## 10.2 自动依赖管理

```kotlin
// java-gradle-plugin 自动添加以下依赖:
// api(gradleApi())  — Gradle API (编译时依赖)
// testImplementation(gradleTestKit()) — TestKit (测试依赖)

// 无需手动声明
// dependencies {
//     implementation(gradleApi())
//     testImplementation(gradleTestKit())
// }
```

## 10.3 插件元数据生成

```kotlin
gradlePlugin {
    plugins {
        create("codeGenerator") {
            id = "com.example.code-generator"
            implementationClass = "com.example.gradle.CodeGeneratorPlugin"
            displayName = "Code Generator"
            description = "Generates boilerplate code from templates"
            tags.set(listOf("code-generation", "template", "java"))
        }

        create("deploymentHelper") {
            id = "com.example.deployment-helper"
            implementationClass = "com.example.gradle.DeploymentHelperPlugin"
            displayName = "Deployment Helper"
            description = "Simplifies application deployment to various environments"
            tags.set(listOf("deployment", "docker", "kubernetes"))
        }
    }
}
```

# 11. 插件开发最佳实践

## 11.1 命名约定

| 元素 | 约定 | 示例 |
|------|------|------|
| 插件 ID | `group.target-feature` | `com.example.code-generator` |
| 扩展名 | 小驼峰, 简洁描述 | `myConfig`, `dataSources` |
| 任务名 | 小写+连字符, 动词开头 | `generateReport`, `validateConfig` |
| 扩展类 | `*Extension` 后缀 | `MyPluginExtension` |
| 任务类 | `*Task` 后缀 | `GenerateReportTask` |
| Worker 类 | `*Worker` 后缀 | `DataProcessorWorker` |

## 11.2 版本兼容性矩阵管理

```kotlin
// build.gradle.kts
// 在插件中声明支持的 Gradle 版本范围
tasks.withType<PluginUnderTestMetadata>().configureEach {
    pluginClasspath.from(
        configurations.runtimeClasspath.get().filter {
            it.name.startsWith("gradle-")
        }
    )
}

// 使用 Gradle API 版本注解 (在插件的 build.gradle.kts 中)
// 确保只使用声明的 Gradle API 版本范围内的 API
```

```groovy
// build.gradle — 使用 apiVersion 限制 API 使用
// 插件元数据中可以声明 requires 和 compatible Gradle 版本
gradlePlugin {
    plugins {
        create("myPlugin") {
            id = "com.example.my-plugin"
            implementationClass = "com.example.gradle.MyPlugin"
            // requires = "7.0"       // 最低 Gradle 版本
            // compatible = "7.0+"   // 兼容版本
        }
    }
}
```

## 11.3 插件开发检查清单

开发自定义插件时，按以下清单逐项检查：

```
[ ] 使用 kotlin-dsl 或 java-gradle-plugin 插件
[ ] 实现 Plugin<Project> 或 Plugin<Settings> 接口
[ ] 使用 abstract class 定义扩展和任务 (配置避免)
[ ] 使用 Property/Provider 而非直接赋值
[ ] 注册任务使用 register() 而非 create()
[ ] 避免在配置阶段使用 afterEvaluate
[ ] 至少编写单元测试 (TestKit) 和功能测试
[ ] 验证配置缓存兼容性
[ ] 声明最低 Gradle 版本要求
[ ] 提供合理的默认值 (convention)
[ ] 编写使用文档和示例
[ ] 考虑插件升级路径和向后兼容性
[ ] 在多个 Gradle 版本上测试兼容性
```

## 11.4 常见反模式

```kotlin
// 反模式 1: 立即配置而非延迟配置
class BadPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        // 错误: 在配置阶段执行耗时操作
        val result = "echo hello".execute().text
        println(result)

        project.tasks.register("bad") {
            doLast { println("Using result: $result") }
        }
    }
}

// 正确: 使用 Provider 延迟执行
class GoodPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        val resultProvider = project.providers.provider {
            "echo hello".execute().text.trim()
        }

        project.tasks.register("good") {
            doLast { println("Using result: ${resultProvider.get()}") }
        }
    }
}

// 反模式 2: 使用内部 API (可能在新版本中变更)
// import org.gradle.api.internal.*  // 避免使用!

// 反模式 3: 不处理插件重复应用
class BadMultipleApplyPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        project.tasks.register("uniqueTask") { }  // 第二次应用会报错
    }
}

// 正确: 处理重复应用
class GoodMultipleApplyPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        project.tasks.register("uniqueTask") {
            onlyIf("already applied?") {
                // 检查是否已经应用
                true
            }
        }
    }
}

// 反模式 4: 硬编码路径
// File("src/main/java")  // 错误

// 正确: 使用 Project 的布局 API
// project.layout.projectDirectory.dir("src/main/java")
```

## 11.5 完整生产级插件示例

```kotlin
// com.example.gradle.LicenseHeaderPlugin.kt
package com.example.gradle

import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.api.tasks.SourceSetContainer
import org.gradle.kotlin.dsl.*

abstract class LicenseHeaderPlugin : Plugin<Project> {

    override fun apply(project: Project) {
        val extension = project.extensions.create(
            "licenseHeader",
            LicenseHeaderExtension::class.java
        )

        extension.licenseFile.convention(
            project.rootProject.layout.projectDirectory.file("LICENSE.txt")
        )

        extension.fileExtensions.convention(listOf("java", "kt", "groovy", "xml", "properties"))

        project.afterEvaluate {
            val sourceSets = project.extensions.getByType(SourceSetContainer::class.java)

            val task = project.tasks.register("applyLicenseHeaders", ApplyLicenseTask::class.java) {
                it.licenseText.set(
                    extension.licenseFile.map { f -> f.asFile.readText() }
                )
                it.fileExtensions.set(extension.fileExtensions)

                sourceSets.forEach { sourceSet ->
                    sourceSet.allSource.srcDirs.forEach { dir ->
                        if (dir.exists()) {
                            it.sourceDirs.from(dir)
                        }
                    }
                }
            }

            project.tasks.named("compileJava") { compileTask ->
                compileTask.dependsOn(task)
            }
        }
    }
}

abstract class LicenseHeaderExtension {
    abstract val licenseFile: RegularFileProperty
    abstract val fileExtensions: ListProperty<String>
}

abstract class ApplyLicenseTask : DefaultTask() {

    @get:Input
    abstract val licenseText: Property<String>

    @get:Input
    abstract val fileExtensions: ListProperty<String>

    @get:InputFiles
    @get:SkipWhenEmpty
    abstract val sourceDirs: ConfigurableFileCollection

    @TaskAction
    fun applyLicense() {
        val header = licenseText.get().trimIndent().prependIndent("// ")
        val extensions = fileExtensions.get()

        sourceDirs.asFileTree
            .filter { it.extension in extensions }
            .forEach { file ->
                val content = file.readText()
                if (!content.startsWith("//")) {
                    file.writeText("$header\n\n$content")
                    logger.quiet("Added license header to: ${file.name}")
                }
            }
    }
}
```

## 11.6 插件开发工具总结

| 工具/库 | 用途 | 依赖 |
|---------|------|------|
| `kotlin-dsl` | Kotlin DSL 开发支持, 类型安全的 Gradle API 访问 | 内置 |
| `java-gradle-plugin` | 插件元数据自动生成, 测试基础设施 | 内置 |
| `gradleTestKit()` | 功能测试 runner | 内置 |
| `com.gradle.plugin-publish` | Plugin Portal 发布 | 第三方 |
| AssertJ | 流畅的测试断言 | testImplementation |
| JUnit 5 | 测试框架 | testImplementation |
| Spek / Kotest | Kotlin 优先的测试框架 | testImplementation |

