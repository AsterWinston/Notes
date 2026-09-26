
# 1. Gradle 的对象模型

## 1.1 Project 对象

在 Gradle 中，Project 是构建的核心概念。每个 `build.gradle(.kts)` 文件都对应一个 `org.gradle.api.Project` 实例。Project 对象是构建脚本的执行上下文，脚本中的顶层方法和属性实际上都派发给了 Project 对象。

**单项目构建**：一个 Project 对象
**多项目构建**：一个根 Project + 每个子模块一个 Project 对象

Gradle 中 Project 对象的层次结构：
```
Root Project 'my-app'
├── Project ':core'           ← domain 逻辑模块
├── Project ':web'            ← Web 层模块
│   └── 依赖于 :core
├── Project ':cli'            ← 命令行工具模块
│   └── 依赖于 :core
└── Project ':integration'    ← 集成测试模块
    └── 依赖于 :core, :web
```

Project 对象提供的核心能力：

| 能力 | 方法/属性 | 说明 |
|------|-----------|------|
| 属性管理 | `property()`, `setProperty()`, `findProperty()` | 读写项目属性 |
| 插件应用 | `apply()`, `plugins {}` | 应用插件 |
| 仓库配置 | `repositories {}` | 配置依赖仓库 |
| 依赖管理 | `dependencies {}` | 声明项目依赖 |
| 任务创建 | `tasks.register()`, `tasks.create()` | 创建和配置 Task |
| 文件操作 | `file()`, `files()`, `fileTree()`, `copy()`, `delete()` | 文件和目录操作 |
| 日志 | `logger` | 构建日志输出 |
| 项目关系 | `parent`, `subprojects`, `allprojects`, `rootProject` | 多项目导航 |
| 构建目录 | `buildDir`, `layout.buildDirectory` | 构建输出目录 |
| 子进程 | `exec()`, `javaexec()` | 执行外部命令和 Java 程序 |

## 1.2 构建脚本中隐式的 Project 委派

Gradle 构建脚本通过委托（Delegation）机制将所有顶层方法调用转发给 Project 对象：

```kotlin
// build.gradle.kts
// 以下代码中所有顶层函数调用实际上都是 Project 的方法

// "group" 实际上是 Project.setGroup()
group = "com.example"

// "repositories" 实际上是 Project.repositories(action: Action<RepositoryHandler>)
repositories {
    mavenCentral()  // RepositoryHandler.mavenCentral()
}

// "dependencies" 实际上是 Project.dependencies(action: Action<DependencyHandler>)
dependencies {
    implementation("com.google.guava:guava:32.1.3-jre")
}

// "tasks" 实际上是 Project.getTasks() -> TaskContainer
tasks.register("hello") {
    doLast {
        println("Hello, Gradle!")
    }
}
```

在 Groovy DSL 中，这种委托通过 Groovy 的闭包委托机制实现。在 Kotlin DSL 中，通过 Kotlin 的类型安全接收器（typed receiver）和扩展函数实现。

## 1.3 Gradle 对象

`org.gradle.api.invocation.Gradle` 对象代表一次 Gradle 调用（一次构建）。它位于 Project 之上，全局唯一。

```kotlin
// settings.gradle.kts
// settings 脚本中可以直接访问 Gradle 对象
gradle.allprojects {
    // 配置所有项目
}

// 获取 Gradle 版本
println("Gradle 版本: ${gradle.gradleVersion}")

// Gradle 对象的生命周期监听
gradle.beforeProject {
    println("正在配置项目: ${it.name}")
}

gradle.afterProject {
    println("项目 ${it.name} 配置完成")
}

// 构建完成回调
gradle.buildFinished {
    println("构建完成，是否失败: ${it.failure != null}")
}

// 任务图就绪回调
gradle.taskGraph.whenReady {
    println("将执行 ${it.allTasks.size} 个任务")
}
```

Gradle 对象的层次位置：
```
Gradle  ← 一次构建调用（全局）
  └── Settings  ← settings.gradle 的执行上下文
       └── Project  ← build.gradle 的执行上下文（根项目）
            ├── Project (子项目 1)
            ├── Project (子项目 2)
            └── Project (子项目 3)
```

## 1.4 Settings 对象

`org.gradle.api.initialization.Settings` 对象是 settings 脚本的执行上下文，负责定义哪些项目参与构建以及仓库等全局配置。

```kotlin
// settings.gradle.kts - Settings 对象的完整用法
rootProject.name = "my-application"

// 包含子项目
include("core", "web", "cli")

// 包含非标准目录的子项目
include(":legacy-api")
project(":legacy-api").projectDir = file("old-code/api")

// 扁平目录结构
includeFlat("shared-lib", "shared-utils")

// 插件管理（集中管理插件版本）
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
    plugins {
        kotlin("jvm") version "1.9.20"
        id("org.springframework.boot") version "3.2.0"
    }
}

// 依赖解析管理（集中管理仓库）
dependencyResolutionManagement {
    repositories {
        mavenCentral()
        google()
    }
    // 不推荐：限制子项目的仓库
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
}

// 构建缓存配置
buildCache {
    local {
        isEnabled = true
        directory = file("${rootDir}/.gradle/build-cache")
    }
    remote<HttpBuildCache> {
        url = uri("https://cache.internal/")
        isPush = true
    }
}
```

## 1.5 ExtensionContainer —— 扩展机制

ExtensionContainer 是 Gradle 插件系统的基础。每个 Project 对象都有一个 `extensions` 属性（类型为 `ExtensionContainer`），插件可以通过它向 Project 添加领域特定扩展。

```kotlin
// ExtensionContainer 的内部工作机制（简化）
interface ExtensionContainer {
    fun <T> add(name: String, type: Class<T>, extension: T)
    fun <T> create(name: String, type: Class<T>, vararg args: Any): T
    fun getByName(name: String): Any
    fun <T> configure(type: Class<T>, action: Action<in T>)
    fun getExtensionsSchema(): Collection<ExtensionSchema>
}
```

当你在构建脚本中编写 `android {}` 或 `java {}` 时，实际上是在操作 ExtensionContainer：

```kotlin
// 插件向 ExtensionContainer 注册扩展
// Android 插件的简化等价代码
project.extensions.create("android", AndroidExtension::class.java, project)

// 构建脚本中使用
android {  // 等价于 project.extensions.configure(AndroidExtension::class.java) { ... }
    compileSdk = 34
    defaultConfig {
        minSdk = 26
    }
}
```

# 2. Task 概念深入

## 2.1 Task 的定义与类型

Task 是 Gradle 中最小的工作单元。每个 Task 代表一个独立的构建步骤——编译 Java 代码、复制文件、运行测试等。

Gradle 中有多种定义 Task 的方式：

**方式一：直接使用 `register` / `create`**
```kotlin
// 注册 Task（延迟创建，推荐）
tasks.register("hello") {
    doLast {
        println("Hello, Gradle!")
    }
}

// 立即创建 Task（不推荐，除非有特殊原因）
tasks.create("hello2") {
    doLast {
        println("Hello again!")
    }
}
```

**方式二：自定义 Task 类**
```kotlin
// 定义自定义 Task 类
abstract class GenerateReportTask : DefaultTask() {

    @get:Input
    abstract val reportName: Property<String>

    @get:InputFile
    abstract val dataFile: RegularFileProperty

    @get:OutputFile
    abstract val reportFile: RegularFileProperty

    @TaskAction
    fun generate() {
        val data = dataFile.asFile.get().readText()
        val report = "Report: $reportName\nData: $data"
        reportFile.asFile.get().writeText(report)
    }
}

// 注册自定义 Task
tasks.register<GenerateReportTask>("generateReport") {
    reportName.set("Monthly Sales")
    dataFile.set(file("data/sales.csv"))
    reportFile.set(layout.buildDirectory.file("reports/sales-report.txt"))
}
```

**方式三：使用现成的 Task 类型**
```kotlin
// Gradle 内置了大量 Task 类型
tasks.register<Copy>("copyResources") {
    from("src/main/resources")
    into(layout.buildDirectory.dir("resources"))
}

tasks.register<Delete>("cleanTemp") {
    delete(layout.buildDirectory.dir("tmp"))
}

tasks.register<Zip>("packageApp") {
    from(layout.buildDirectory.dir("classes"))
    archiveFileName.set("app.zip")
}

tasks.register<Exec>("runScript") {
    commandLine("node", "scripts/build.js")
}

tasks.register<JavaExec>("runApp") {
    classpath = sourceSets.main.get().runtimeClasspath
    mainClass.set("com.example.App")
}
```

## 2.2 Task 的输入与输出

Gradle 增量构建的核心是 Task 的输入/输出声明。只有正确定义了输入和输出的 Task 才能受益于增量构建。

```kotlin
abstract class ProcessDataTask : DefaultTask() {

    // 输入：简单值类型
    @get:Input
    abstract val threshold: Property<Int>

    // 输入：文件
    @get:InputFile
    abstract val inputFile: RegularFileProperty

    // 输入：目录
    @get:InputDirectory
    abstract val inputDir: DirectoryProperty

    // 输入：文件集合（忽略文件顺序）
    @get:InputFiles
    @get:PathSensitive(PathSensitivity.RELATIVE)
    abstract val sourceFiles: ConfigurableFileCollection

    // 嵌套的输入对象
    @get:Nested
    abstract val options: Property<NestedOptions>

    // 输出：文件
    @get:OutputFile
    abstract val outputFile: RegularFileProperty

    // 输出：目录
    @get:OutputDirectory
    abstract val outputDir: DirectoryProperty

    // 不参与增量判断（如日志目录）
    @get:Internal
    abstract val logDir: DirectoryProperty

    @TaskAction
    fun process() {
        // 实际处理逻辑
    }
}
```

输入/输出注解类型说明：

| 注解 | 应用于 | 参与 UP-TO-DATE 检查 | 说明 |
|------|--------|----------------------|------|
| `@Input` | 原始类型、String、集合 | 是 | 值比较 |
| `@InputFile` | 单个文件 | 是 | 文件内容哈希 |
| `@InputFiles` | 文件集合 | 是 | 文件集合内容哈希 |
| `@InputDirectory` | 目录 | 是 | 目录结构和内容 |
| `@Nested` | 自定义对象 | 是 | 递归检查内嵌属性 |
| `@OutputFile` | 单个文件 | 是 | 必须被 Task 创建 |
| `@OutputFiles` | 文件集合 | 是 | 必须被 Task 创建 |
| `@OutputDirectory` | 目录 | 是 | 必须被 Task 创建 |
| `@OutputDirectories` | 目录集合 | 是 | 必须被 Task 创建 |
| `@Internal` | 任意 | 否 | 明确标记不参与检查 |
| `@Classpath` | 文件集合 | 是 | 类路径（顺序敏感） |

## 2.3 Task 的依赖关系

Gradle 中 Task 可以声明对其他 Task 的依赖，形成执行顺序的有向无环图：

```kotlin
// 方式1：显式声明依赖
tasks.register("compile") {
    doLast { println("编译...") }
}

tasks.register("package") {
    dependsOn("compile")  // package 依赖 compile
    doLast { println("打包...") }
}

tasks.register("deploy") {
    dependsOn("package")  // deploy 依赖 package
    doLast { println("部署...") }
}

// 方式2：多个依赖
tasks.register("integrationTest") {
    dependsOn("compile", "package")
    doLast { println("集成测试...") }
}

// 方式3：通过 Task 提供者的输出建立隐式依赖
val classesDir = tasks.compileJava.flatMap { it.destinationDirectory }

tasks.register<Copy>("copyClasses") {
    from(classesDir)  // 隐式依赖 compileJava
    into(layout.buildDirectory.dir("backup"))
}

// 方式4：使用 finalizedBy（总是执行，无论成功或失败）
tasks.register("startServer") {
    finalizedBy("stopServer")  // 始终在 startServer 之后执行 stopServer
}

// 方式5：使用 mustRunAfter / shouldRunAfter（软约束）
tasks.register("taskA") {
    mustRunAfter("taskB")  // 如果 A 和 B 都执行，A 必须在 B 之后
}
```

执行顺序的可视化（Task DAG）：
```
         [clean]
           │
      ┌────┴────┐
      │         │
 [compileJava]  │
      │         │
 [processResources]
      │         │
      └────┬────┘
           │
       [classes]
           │
      [compileTestJava]
           │
         [test]
           │
        [build]
```

## 2.4 Task 配置与执行的分离

Gradle 的一个重要概念是 Task 的**配置**和**执行**必须分离。配置代码在配置阶段运行，执行代码在执行阶段运行。

```kotlin
// 错误示例：在配置阶段执行了应该在执行阶段做的事
tasks.register("badTask") {
    // ❌ 这段代码在配置阶段执行（每次构建都会运行）
    val data = file("data.json").readText()
    println("配置阶段读取了文件: $data")

    doLast {
        // 这段代码在执行阶段执行
        println("实际任务执行")
    }
}

// 正确示例：延迟操作到执行阶段
tasks.register("goodTask") {
    // 配置阶段：仅声明输入
    val dataFile = file("data.json")
    val outputFile = layout.buildDirectory.file("output.txt")

    inputs.file(dataFile)
    outputs.file(outputFile)

    doLast {
        // 执行阶段：实际读取和处理
        val data = dataFile.readText()
        println("处理数据: $data")
        outputFile.get().asFile.writeText("Processed: $data")
    }
}
```

**配置阶段 vs 执行阶段的行为差异：**

```kotlin
// 演示配置阶段和执行阶段的区别
tasks.register("demo") {
    // === 配置阶段（始终执行）===
    println("[配置阶段] Task 'demo' 正在被配置")
    val configTime = System.currentTimeMillis()

    // === 执行阶段（仅在 Task 需要运行时执行）===
    doFirst {
        println("[执行阶段:doFirst] Task 'demo' 开始执行")
    }

    doLast {
        println("[执行阶段:doLast] Task 'demo' 执行完成")
        println("  配置时间: $configTime")
        println("  执行时间: ${System.currentTimeMillis()}")
    }
}
```

输出（第一次执行）：
```
[配置阶段] Task 'demo' 正在被配置
> Task :demo
[执行阶段:doFirst] Task 'demo' 开始执行
[执行阶段:doLast] Task 'demo' 执行完成
  配置时间: 1700000000000
  执行时间: 1700000001000
```

输出（第二次执行，demo 标记为 UP-TO-DATE）：
```
[配置阶段] Task 'demo' 正在被配置  ← 配置阶段仍然执行
> Task :demo UP-TO-DATE             ← 执行阶段被跳过
```

## 2.5 Task 的生命周期回调

Gradle 提供了多个 Task 生命周期回调：

```kotlin
tasks.register("lifecycleDemo") {
    doFirst {
        // 在 doLast 之前执行（可以有多个 doFirst）
        println("doFirst #1: 准备执行")
    }

    doFirst {
        println("doFirst #2: 再次准备")
    }

    doLast {
        // 主要逻辑（可以有多个 doLast）
        println("doLast #1: 主要逻辑")
    }

    doLast {
        println("doLast #2: 清理工作")
    }

    onlyIf {
        // 条件判断：返回 false 则跳过此 Task
        System.getenv("SKIP_DEMO") != "true"
    }

    enabled = true  // 显式启用/禁用
}
```

执行顺序：
```
1. onlyIf 检查
2. enabled 检查
3. doFirst #1
4. doFirst #2
5. doLast #1（@TaskAction 方法如果使用自定义 Task 类）
6. doLast #2
```

# 3. 构建生命周期三阶段深究

## 3.1 初始化阶段

初始化阶段的主要工作是：
1. 查找并解析 `settings.gradle(.kts)` 文件
2. 创建 `Settings` 对象并执行 settings 脚本
3. 根据 `include` 语句确定参与构建的项目
4. 创建 `Project` 对象层次结构

```kotlin
// settings.gradle.kts
// 在初始化阶段执行
println("[初始化] settings 脚本开始执行")

rootProject.name = "my-advanced-build"

include(":core")
include(":web")
include(":cli")

// 在初始化阶段可以使用 beforeProject / afterProject
gradle.beforeProject {
    println("[初始化→配置过渡] 即将配置项目: ${it.name}")
}

// 监听 settings 评估完成
gradle.settingsEvaluated {
    println("[初始化] settings 评估完成，项目数: ${gradle.rootProject.subprojects.size}")
}
```

## 3.2 配置阶段

配置阶段是 Gradle 构建中最重要的阶段。在此阶段：
1. 插件被解析和应用
2. Task 被创建和配置
3. 依赖被解析（依赖声明，不是下载 JAR）
4. Extension 被注册和配置
5. Task 执行图被构建

**配置阶段的执行顺序：**
```kotlin
// settings.gradle.kts
gradle.beforeProject {
    println("beforeProject: ${it.name}")
}

gradle.afterProject {
    println("afterProject: ${it.name}")
}

gradle.projectsLoaded {
    println("projectsLoaded: 所有项目已加载")
}

// build.gradle.kts (root)
println("=== 配置根项目 ${name} ===")

gradle.projectsEvaluated {
    println("projectsEvaluated: 所有项目配置完成")
}

gradle.taskGraph.whenReady {
    println("taskGraph.whenReady: DAG 构建完成")
}
```

典型输出：
```
projectsLoaded: 所有项目已加载
beforeProject: root
=== 配置根项目 root ===
afterProject: root
beforeProject: core
=== 配置子项目 core ===
afterProject: core
beforeProject: web
=== 配置子项目 web ===
afterProject: web
projectsEvaluated: 所有项目配置完成
taskGraph.whenReady: DAG 构建完成
```

**跨项目配置的传播：**
```kotlin
// 根项目的 build.gradle.kts - 向所有子项目传播配置
subprojects {
    // this = 每个子项目的 Project 对象
    apply(plugin = "java")

    group = rootProject.group
    version = rootProject.version

    repositories {
        mavenCentral()
    }

    java {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    dependencies {
        testImplementation("org.junit.jupiter:junit-jupiter:5.10.1")
    }

    tasks.test {
        useJUnitPlatform()
    }
}

// 仅对名字匹配的子项目进行特殊配置
project(":web") {
    apply(plugin = "war")

    dependencies {
        implementation("org.springframework.boot:spring-boot-starter-web:3.2.0")
    }
}

// allprojects 包括根项目本身
allprojects {
    // 对根项目和所有子项目生效
    println("配置项目: ${project.name}")
}
```

## 3.3 执行阶段

执行阶段是 Gradle 实际执行工作的阶段。在此阶段：
1. Gradle 根据 Task 之间的依赖关系计算执行顺序
2. 对每个 Task 进行增量检查（比较输入/输出）
3. 按顺序（或并行）执行需要执行的 Task
4. 收集和报告执行结果

```kotlin
// 监听执行阶段的事件
gradle.taskGraph.beforeTask {
    println("[执行前] 即将执行 Task: ${it.name}")
}

gradle.taskGraph.afterTask {
    println("[执行后] Task: ${it.name} - 状态: ${it.state.failure?.message ?: "成功"}")
}

gradle.buildFinished {
    println("[构建完成] 结果: ${it.failure?.message ?: "成功"}")
}
```

## 3.4 配置缓存对生命周期的影响

配置缓存将配置阶段的结果序列化到磁盘，使得后续构建可以跳过配置阶段：

```
普通构建:
  初始化 → 配置（评估所有 .gradle 文件） → 执行

配置缓存命中时:
  初始化 → 加载缓存（反序列化） → 执行
          ↑ 跳过了配置阶段
```

要启用配置缓存，Task 的配置代码必须满足可缓存要求：

```kotlin
// ❌ 不兼容配置缓存的代码（配置阶段读取外部状态）
val buildTime = System.currentTimeMillis()  // 不可缓存的系统时间访问
val env = System.getenv("HOME")             // 不可缓存的环境变量访问

tasks.register("timestamp") {
    doLast {
        println("构建时间: $buildTime")
    }
}

// ✅ 兼容配置缓存的代码（外部状态访问延迟到执行阶段）
tasks.register("timestamp") {
    doLast {
        val buildTime = System.currentTimeMillis()
        println("构建时间: $buildTime")
    }
}

// ✅ 使用 Provider 延迟外部状态访问
val buildTimeProvider = providers.systemProperty("build.start.time")
tasks.register("timestamp") {
    doLast {
        println("构建时间: ${buildTimeProvider.get()}")
    }
}
```

# 4. 插件机制深入

## 4.1 插件的三种类型

Gradle 支持三种类型的插件：

| 类型 | 路径 | 范围 | 说明 |
|------|------|------|------|
| **脚本插件** | 本地文件系统 | 项目内 | 把构建逻辑抽到独立 .gradle 文件 |
| **预编译脚本插件** | `buildSrc/src/main/*` | 项目内 | 编译后成为二进制插件 |
| **二进制插件** | JAR 文件 | 发布 | 发布的声明式插件 |

**脚本插件示例：**
```groovy
// gradle/java-convention.gradle（Groovy 脚本插件）
plugins {
    id 'java'
}

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.1'
}

tasks.named('test') {
    useJUnitPlatform()
}

// build.gradle - 应用脚本插件
// apply from: 'gradle/java-convention.gradle'  // Groovy
apply(from = "gradle/java-convention.gradle")  // Kotlin
```

## 4.2 预编译脚本插件（Precompiled Script Plugin）

从 Gradle 6.0 开始，推荐使用预编译脚本插件代替传统的脚本插件。它们位于 `buildSrc` 目录下，自动编译为 JAR。

```
buildSrc/
├── build.gradle.kts          ← buildSrc 自身的构建文件
├── settings.gradle.kts        ← （可选）
└── src/
    └── main/
        └── kotlin/
            ├── java-convention.gradle.kts     ← 预编译脚本插件
            ├── kotlin-convention.gradle.kts
            └── spring-convention.gradle.kts
```

```kotlin
// buildSrc/build.gradle.kts
plugins {
    `kotlin-dsl`  // 启用 Kotlin DSL 和预编译脚本插件
}

repositories {
    mavenCentral()
}

// java-convention.gradle.kts 的内容（文件名即插件 ID）
plugins {
    java
}

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

dependencies {
    testImplementation(platform("org.junit:junit-bom:5.10.1"))
    testImplementation("org.junit.jupiter:junit-jupiter")
}

tasks.test {
    useJUnitPlatform()
}

// 模块的 build.gradle.kts 中应用
plugins {
    id("java-convention")  // 文件名即为插件 ID
}
```

## 4.3 二进制插件开发

自定义 Gradle 插件是一个实现了 `org.gradle.api.Plugin<Project>` 接口的类：

```kotlin
// 简单的自定义插件
class MyPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        // 注册扩展
        val extension = target.extensions.create("myConfig", MyExtension::class.java)

        // 注册任务
        target.tasks.register("myTask", MyTask::class.java) {
            group = "My Plugin"
            description = "Custom task from MyPlugin"
        }

        // 添加约定
        target.afterEvaluate {
            val config = it.extensions.getByType(MyExtension::class.java)
            println("MyPlugin 已应用于 ${it.name}, 配置: ${config.message}")
        }
    }
}

// 插件关联的扩展类
open class MyExtension {
    var message: String = "Hello from MyPlugin"
    var enabled: Boolean = true
}

// 插件关联的 Task 类
abstract class MyTask : DefaultTask() {
    @get:Input
    abstract val message: Property<String>

    init {
        group = "My Plugin"
        description = "Executes MyPlugin custom task"
    }

    @TaskAction
    fun execute() {
        println("MyTask 执行: ${message.get()}")
    }
}
```

插件注册（`build.gradle.kts` 中）：
```kotlin
// gradlePlugin 块用于 Gradle Plugin Portal 发布
gradlePlugin {
    plugins {
        create("myPlugin") {
            id = "com.example.my-plugin"
            implementationClass = "com.example.MyPlugin"
            displayName = "My Custom Plugin"
            description = "A plugin that demonstrates Gradle plugin development"
            tags.set(listOf("demo", "example"))
        }
    }
}
```

## 4.4 插件解析和应用机制

当 Gradle 遇到 `plugins {}` 块时：

```
plugins { id("java") version "1.0" }
    │
    ├─► 1. 从 Plugin Specification 中提取 id 和 version
    │
    ├─► 2. 查询本地缓存：GRADLE_USER_HOME/caches/modules-2/
    │
    ├─► 3. 如果未缓存，从仓库解析：
    │       - Plugin Portal (plugins.gradle.org)
    │       - 自定义仓库（通过 pluginManagement）
    │
    ├─► 4. 下载插件 JAR 和传递性依赖
    │
    ├─► 5. 加载插件类
    │
    ├─► 6. 实例化插件并调用 apply(Project)
    │       - 注册扩展
    │       - 注册任务
    │       - 添加约定
    │
    └─► 7. 插件在 Project 的生命周期内保持活动
```

**plugins {} vs apply plugin: 的区别：**

```kotlin
// plugins {} 块（推荐）
// - 在脚本的最顶层，必须在任何其他代码之前
// - 使用 Plugin Marker Artifacts（自动生成）
// - 支持版本管理
plugins {
    id("java")
    id("org.springframework.boot") version "3.2.0"
}

// apply() 方法（遗留方式）
// - 可以在脚本的任何位置调用
// - 可以应用脚本插件
apply(plugin = "java")
apply(from = "other.gradle")
apply<MyPlugin>()  // 类型安全的二进制插件应用
```

## 4.5 pluginManagement 集中管理

```kotlin
// settings.gradle.kts - 全局插件管理
pluginManagement {
    // 定义插件解析仓库
    repositories {
        gradlePluginPortal()    // 官方插件门户
        mavenCentral()
        maven("https://plugins.example.com/repository")
    }

    // 解析策略
    resolutionStrategy {
        // 在插件版本冲突时中断构建
        failOnVersionConflict()

        // 为每个插件指定策略
        eachPlugin {
            if (requested.id.namespace == "com.example") {
                useModule("com.example:${requested.id.name}:${requested.version}")
            }
        }
    }

    // 集中声明插件版本
    plugins {
        kotlin("jvm") version "1.9.20"
        id("io.spring.dependency-management") version "1.1.4"
    }
}

// 子项目中直接引用（无需版本号）
// subproject/build.gradle.kts
plugins {
    kotlin("jvm")  // 版本已在 pluginManagement 中定义
    id("io.spring.dependency-management")
}
```

# 5. 域对象容器内部设计

## 5.1 DomainObjectContainer

`DomainObjectContainer` 是 Gradle 中管理命名域对象的核心数据结构。它扩展了 `NamedDomainObjectSet`。

```kotlin
// DomainObjectContainer 的使用（如 sourceSets）
// 这是 java 插件提供的 sourceSets 容器
sourceSets {
    // create() 创建并配置
    create("integrationTest") {
        java.srcDir("src/integration-test/java")
        resources.srcDir("src/integration-test/resources")
        compileClasspath += sourceSets.main.get().output + configurations.testRuntimeClasspath
        runtimeClasspath += output + compileClasspath
    }

    // 配置已有的
    main {
        java.srcDir("src/java")
    }
}
```

## 5.2 NamedDomainObjectCollection

`NamedDomainObjectCollection` 是所有命名域对象容器的基类。它提供：

- **按名称访问**：`container.getByName("name")`、`container["name"]`
- **过滤**：`container.matching { ... }`、`container.withType<T>()`
- **懒配置**：`container.configureEach { ... }`
- **规则驱动**：`container.addRule("pattern") { ... }`

```kotlin
// NamedDomainObjectCollection 的高级用法
tasks.configureEach {
    // 配置所有 Task（包括未来注册的）
    if (this is JavaCompile) {
        options.encoding = "UTF-8"
        options.compilerArgs.add("-Xlint:unchecked")
    }
}

// 按名称筛选
tasks.matching { it.name.startsWith("compile") }.configureEach {
    doFirst {
        println("编译任务: ${this.name}")
    }
}

// 按类型筛选
tasks.withType<Test>().configureEach {
    useJUnitPlatform()
    maxParallelForks = 4
}

// 命名规则
tasks.addRule("Pattern: generate<Name>Report") {
    val taskName = this
    tasks.register(taskName) {
        doLast {
            val reportName = taskName.removePrefix("generate").removeSuffix("Report")
            println("生成报告: $reportName")
        }
    }
}
// 现在可以执行 generateSalesReport（即使未显式注册）
```

## 5.3 内部延迟配置机制

Gradle 的 `configureEach`、`whenObjectAdded` 和 `all` 方法实现了延迟配置，确保即使在容器中添加新元素后配置仍然生效：

```kotlin
// configureEach 的实现原理（简化）
class MyContainer<T : Named> {
    private val elements = mutableMapOf<String, T>()
    private val configureActions = mutableListOf<Action<in T>>()
    private val addActions = mutableListOf<Action<in T>>()

    // 注册配置动作，对已有和未来的元素都生效
    fun configureEach(action: Action<in T>) {
        configureActions.add(action)
        elements.values.forEach { action.execute(it) }
    }

    // 注册添加回调，仅对未来元素生效
    fun whenObjectAdded(action: Action<in T>) {
        addActions.add(action)
    }

    // 添加元素
    fun add(element: T) {
        elements[element.name] = element
        configureActions.forEach { it.execute(element) }
        addActions.forEach { it.execute(element) }
    }

    // 立即配置所有现有元素
    fun all(action: Action<in T>) {
        elements.values.forEach { action.execute(it) }
        whenObjectAdded(action)  // 也对未来元素生效
    }
}
```

## 5.4 Provider API

Gradle 的 Provider API 是延迟计算和属性绑定的核心：

```kotlin
// Provider 类型层次
// Provider<T>  ← 基本延迟值
//   ├── Property<T>  ← 可读写
//   │   ├── RegularFileProperty
//   │   ├── DirectoryProperty
//   │   └── ListProperty<T> / SetProperty<T>
//   └── ConfigurableFileCollection / ConfigurableFileTree

// Provider 的变换和组合
val version: Provider<String> = providers.gradleProperty("appVersion")
    .orElse("1.0.0")

val upperVersion = version.map { it.uppercase() }

val combined = version.zip(upperVersion) { v, uv ->
    "Version: $v ($uv)"
}

// 在 Task 中使用 Provider 建立隐式依赖
abstract class ReportTask : DefaultTask() {
    @get:Input
    abstract val reportName: Property<String>

    @get:OutputFile
    abstract val outputFile: RegularFileProperty

    @TaskAction
    fun generate() {
        outputFile.asFile.get().writeText("Report: ${reportName.get()}")
    }
}

tasks.register<ReportTask>("report") {
    reportName.set("Sales")  // 直接设置
    outputFile.set(layout.buildDirectory.file("reports/sales.txt"))
}

// Provider 链：一个 Task 的输出作为另一个 Task 的输入
val reportFile = tasks.named<ReportTask>("report").flatMap { it.outputFile }

tasks.register<Copy>("publishReport") {
    from(reportFile)  // 隐式依赖 report Task
    into(layout.buildDirectory.dir("published"))
}
```
