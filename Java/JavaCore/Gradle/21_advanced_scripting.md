
# 1. Provider API 深度解析

## 1.1 什么是 Provider API

Gradle 的 Provider API 是惰性配置 (Lazy Configuration) 的基石。Provider 代表一个在未来某个时间点才能计算/解析的值。与立即求值不同，Provider 将值的计算推迟到真正需要时才执行。这避免了在配置阶段执行不必要的计算，提升了配置缓存 (Configuration Cache) 的兼容性。

核心接口位于 `org.gradle.api.provider` 包中：

| 接口 | 说明 |
|------|------|
| `Provider<T>` | 只读的惰性值容器 |
| `Property<T>` | 可写的惰性值容器, 继承自 Provider |
| `ConfigurableFileCollection` | 惰性文件集合 |
| `ConfigurableFileTree` | 惰性文件树 |

```groovy
// 普通立即赋值 - 不推荐
def outputDir = file("$buildDir/output")
task generateFiles {
    doLast { println "Output: $outputDir" }
}

// Provider 惰性赋值 - 推荐
abstract class MyTask extends DefaultTask {
    @OutputDirectory
    abstract DirectoryProperty getOutputDir()
}

task myTask(type: MyTask) {
    outputDir = layout.buildDirectory.dir("output")
}
```

## 1.2 Provider.map - 值转换

`Provider.map` 用于对 Provider 内部的值进行变换，返回新的 Provider。该变换在值实际被消费时才执行。

```groovy
import org.gradle.api.provider.Provider

Provider<String> version = providers.provider { "1.0.0" }

// 惰性变换：只有需要时才会执行 toUpperCase()
Provider<String> upperVersion = version.map { it.toUpperCase() }

// 串联 map 操作
Provider<String> fullName = version.map { v -> "v${v}" }
                                   .map { v -> "${v}-RELEASE" }

println upperVersion.get() // 输出: 1.0.0 (大写)

// map 支持 null 安全
Provider<String> nullable = providers.provider { null }
Provider<String> safe = nullable.map { it?.toUpperCase() ?: "UNKNOWN" }
```

## 1.3 Provider.flatMap - 链式依赖

`Provider.flatMap` 用于将一个 Provider 的值转换为另一个 Provider。当转换逻辑本身也需要惰性计算时使用。

```groovy
import org.gradle.api.provider.ProviderFactory

abstract class VersionHelper {
    abstract ProviderFactory getProviders()
    
    Provider<String> resolveVersion(String projectPath) {
        // 根据项目路径从某处获取版本号的 Provider
        return providers.provider { "2.0.0" }
    }
}

// flatMap: 当转换函数返回 Provider 时使用
Provider<String> projectPath = providers.provider { ":lib-core" }
Provider<String> resolved = projectPath.flatMap { path ->
    versionHelper.resolveVersion(path)  // 返回 Provider<String>
}

// 对比 map 和 flatMap
// map:    Provider<T> -> (T -> R) -> Provider<R>
// flatMap: Provider<T> -> (T -> Provider<R>) -> Provider<R>
```

## 1.4 Provider.orElse - 默认值处理

```groovy
Provider<String> optionalValue = providers.provider { null }
Provider<String> fallback = optionalValue.orElse("默认值")

// orElse 接受另一个 Provider
Provider<String> backup = providers.provider { "备用版本" }
Provider<String> result = optionalValue.orElse(backup)

// 常量默认值
Property<String> name = objectFactory.property(String)
name.convention("Unnamed")  // 设置约定值 (更高优先级)
// 执行效果: name.isPresent() ? name.get() : "Unnamed"
```

## 1.5 Provider.zip - 多值组合

```groovy
Provider<String> group = providers.provider { "com.example" }
Provider<String> name = providers.provider { "my-lib" }
Provider<String> version = providers.provider { "1.0" }

// 组合两个 Provider
Provider<String> coordinates = group.zip(name) { g, n -> "$g:$n" }

// 使用 zip 组合三个值
Provider<String> gav = group.zip(name) { g, n -> "$g:$n" }
                             .zip(version) { gn, v -> "$gn:$v" }

// 实用案例：动态生成任务名
Provider<String> taskName = group.zip(name) { g, n ->
    "publish${g.replace('.','_')}${n}"
}
```

## 1.6 ProviderFactory 深入

`ProviderFactory` 通过 `providers` 访问，提供创建各种 Provider 的工厂方法。

```groovy
// 类型: ObjectFactory -> ProviderFactory
// 通过 project.providers 或 objectFactory 获取

// 1. provider(Callable) - 从闭包创建
Provider<String> computed = providers.provider {
    def env = System.getenv("BUILD_NUMBER")
    env ?: "local"
}

// 2. of(Class, Provider) - 类型转换
Provider<CharSequence> raw = providers.provider { "text" }
Provider<String> typed = providers.of(String, raw)

// 3. systemProperty / gradleProperty / environmentVariable
Provider<String> homeDir = providers.systemProperty("user.home")
Provider<Boolean> isCi = providers.gradleProperty("ci").map { it.toBoolean() }
Provider<String> path = providers.environmentVariable("PATH")

// 4. fileContents - 读取文件内容
Provider<String> version = providers.fileContents(
    layout.projectDirectory.file("version.txt")
).asText  // 需要 asText 或 asBytes

// 5. contents - 字符内容 Provider
Provider<String> content = providers.contents("text") {
    from(layout.projectDirectory.file("README.md"))
    filteringCharset("UTF-8")
}

// 6. 组合使用
Provider<String> effectiveVersion = providers.gradleProperty("appVersion")
    .orElse(providers.fileContents(layout.projectDirectory.file("VERSION")).asText)
    .orElse("0.0.0")
```

# 2. Property 类型详解

## 2.1 Property<T> 基础

`Property<T>` 是 Gradle 中最常用的属性类型,表示一个可配置的惰性值。它支持约定值 (convention) 和显式值 (set value)。

```groovy
public interface Property<T> extends Provider<T>, HasMultipleValues<T> {
    void set(T value)
    void set(Provider<? extends T> provider)
    void convention(T value)
    void convention(Provider<? extends T> provider)
    Property<T> value(T value)
    Property<T> value(Provider<? extends T> provider)
    void unset()
    void finalizeValue()
    void finalizeValueOnRead()
    void disallowChanges()
    T getOrElse(T defaultValue)
    T getOrNull()
}
```

```groovy
abstract class MyExtension {
    abstract Property<String> getAppName()
    abstract Property<Integer> getMaxConnections()
}

myExtension {
    appName = "MyApp"                      // 直接赋值
    appName.set("MyApp")                   // 显式 set
    appName.set(providers.provider {...})  // 从 Provider set
}

// 约定值模式
myExtension {
    appName.convention("DefaultApp")  // 设置默认值
    // 如果后续显式 set("Overridden"), 约定值失效
}
```

## 2.2 RegularFileProperty 和 DirectoryProperty

```groovy
abstract class GenerateTask extends DefaultTask {
    @InputFile
    abstract RegularFileProperty getConfigFile()
    
    @OutputDirectory
    abstract DirectoryProperty getOutputDir()
    
    @OutputFile
    abstract RegularFileProperty getReportFile()
}

task generate(type: GenerateTask) {
    configFile = layout.projectDirectory.file("config.yml")
    outputDir = layout.buildDirectory.dir("generated")
    // 注意：DirectoryProperty.file() 返回 RegularFileProperty
    reportFile = outputDir.file("report.html")
}

// 文件操作
RegularFileProperty inputFile = objectFactory.fileProperty()
inputFile.set(layout.projectDirectory.file("input.txt"))

// file(Provider) 方式
inputFile.fileProvider(provider)  // 从 Provider 设置文件和路径
inputFile.fileValue(someFile)     // 从 File 对象设置
inputFile.locationOnly(provider)  // 仅设置位置，不解析内容
```

## 2.3 ListProperty<T>

```groovy
abstract class CompileExtensions {
    abstract ListProperty<String> getExcludes()
    abstract ListProperty<File> getClasspath()
}

compileExtensions {
    excludes.add("**/test/**")
    excludes.addAll("**/generated/**", "**/internal/**")
    excludes.addAll(providers.provider { ["**/deprecated/**"] })
    
    // 设置整个列表（替换）
    excludes.set(["**/test/**", "**/demo/**"])
}

// ListProperty 常用方法
ListProperty<String> jvmArgs = objectFactory.listProperty(String)
jvmArgs.convention(["-Xmx256m"])    // 约定值
jvmArgs.add("-XX:+UseG1GC")         // 追加
jvmArgs.addAll(["-ea", "-Ddebug"])  // 批量追加
jvmArgs.empty()                     // 清空
jvmArgs.finalizeValue()            // 锁定，不允许后续修改
jvmArgs.disallowChanges()          // 禁止修改

// 在 Task 中声明
abstract class JavaExecTask extends DefaultTask {
    @Input
    abstract ListProperty<String> getJvmArgs()
    
    @Input
    abstract ListProperty<String> getProgramArgs()
}
```

## 2.4 SetProperty<T>

与 ListProperty 类似，但保证元素唯一性，且不保证顺序。

```groovy
abstract class SourceSetConfig {
    abstract SetProperty<String> getSourceDirs()
}

sourceSetConfig {
    sourceDirs.add("src/main/java")
    sourceDirs.add("src/main/java")  // 重复添加，不会抛错但只会保留一个
}
```

## 2.5 MapProperty<K,V>

```groovy
abstract class EnvironmentConfig {
    abstract MapProperty<String, String> getEnvVars()
    abstract MapProperty<String, Object> getMetadata()
}

environmentConfig {
    envVars.put("JAVA_HOME", "/usr/lib/jvm/java-17")
    envVars.putAll([
        "GRADLE_OPTS": "-Xmx2g",
        "CI": "true"
    ])
    
    // 从 Provider 设置
    envVars.put(providers.provider { "DYNAMIC_KEY" },
                providers.provider { "dynamic_value" })
}
```

## 2.6 HasConfigurableValue 接口体系

```
HasConfigurableValue
├── Configurable
│   ├── Property<T>
│   ├── RegularFileProperty
│   ├── DirectoryProperty
│   └── ConfigurableFileCollection
├── HasMultipleValues<T>
│   ├── ListProperty<T>
│   ├── SetProperty<T>
│   └── MapProperty<K,V>
└── Managed (内部)
```

关键方法签名：
```groovy
interface HasConfigurableValue {
    void finalizeValue()
    void finalizeValueOnRead()
    void disallowChanges()
}

interface Configurable extends HasConfigurableValue {
    void convention(T value)
    void convention(Provider<T> value)
}

interface HasMultipleValues<T> extends HasConfigurableValue {
    void convention(Iterable<? extends T> value)
    void convention(Provider<? extends Iterable<? extends T>> value)
}
```

# 3. 惰性配置模式

## 3.1 惰性 Task 配置

```groovy
// ❌ 非惰性配置 - 所有文件夹都在配置阶段解析
tasks.named("compileJava") {
    source = fileTree("src/main/java") {
        include "**/*.java"
    }
}

// ✅ 惰性配置 - 使用 Property
tasks.named("compileJava", JavaCompile) {
    // 使用 Task 的 abstract 属性
}

abstract class MyPlugin implements Plugin<Project> {
    void apply(Project project) {
        project.tasks.register("lazyTask", LazyTask) {
            // 只有需要执行时才配置
        }
    }
}
```

## 3.2 Task Provider 组合

```groovy
// register 返回 TaskProvider<T> 而非 Task
def compileProvider = tasks.register("compile", MyCompile) {
    sources = fileTree("src")
}

def jarProvider = tasks.register("package", Jar) {
    // 惰性关联：只有当 compile 执行时才读取其输出
    from(compileProvider.map { it.outputs.files })
    // 或
    from(compileProvider.flatMap { it.outputDir })
}

// 依赖链
tasks.register("publish") {
    dependsOn(jarProvider)  // 使用 Provider，而非 Task 本身
}
```

## 3.3 惰性集合与文件操作

```groovy
// ConfigurableFileCollection - 惰性文件集合
ConfigurableFileCollection classpath = objectFactory.fileCollection()
classpath.from(
    configurations.compileClasspath,
    layout.projectDirectory.dir("libs")
)
classpath.builtBy(compileTask)

// ConfigurableFileTree - 惰性文件树
ConfigurableFileTree sources = objectFactory.fileTree()
sources.from("src/main/java")
sources.include("**/*.java")
sources.exclude("**/generated/**")

// 文件集合操作
classpath.filter { it.name.endsWith(".jar") }
classpath.asFileTree.matching { include "**/*.class" }
```

# 4. DomainObjectCollection

## 4.1 all vs configureEach

```groovy
tasks.all { task ->
    // ❌ 对所有 task 立即执行（包括之后创建的）
    // 破坏了惰性，强制所有 task 被创建
}

tasks.configureEach { task ->
    // ✅ 对所有 task 惰性执行（包括之后创建的）
    // 不强制立即创建 task，延迟到需要时才执行
}

// 典型用法：为所有 JavaCompile 设置通用配置
tasks.withType(JavaCompile).configureEach {
    options.encoding = "UTF-8"
    options.release = 17
}

// 等价于：
tasks.withType(JavaCompile) {  // 这也是惰性的，底层使用 configureEach
    options.encoding = "UTF-8"
}
```

## 4.2 NamedDomainObjectContainer

```groovy
// 定义类型
interface Book {
    Property<String> getTitle()
    Property<String> getAuthor()
}

// 创建容器
NamedDomainObjectContainer<Book> books = container(Book) { name ->
    // name 是每个 item 的唯一标识名
}

// DSL 配置
books {
    register("greatexpectations") {
        title = "Great Expectations"
        author = "Charles Dickens"
    }
    register("warandpeace") {
        title = "War and Peace"
        author = "Leo Tolstoy"
    }
}

// 在 Extension 中使用
abstract class LibraryExtension {
    abstract NamedDomainObjectContainer<Book> getBooks()
}

// 命名对象容器的常用方法
books.named("greatexpectations")             // NamedDomainObjectProvider<Book>
books.getByName("greatexpectations")         // Book (会立即实例化)
books.matching { it.author.get() == "..." }  // 筛选
books.configureEach { book -> ... }          // 惰性配置所有
```

## 4.3 polymorphicDomainObjectContainer

支持不同类型元素的命名容器：

```groovy
interface Document {
    Property<String> getTitle()
}

interface TextDocument extends Document {
    Property<String> getContent()
}

interface ImageDocument extends Document {
    Property<Integer> getWidth()
}

abstract class DocsExtension {
    abstract PolymorphicDomainObjectContainer<Document> getDocuments()
}

docs {
    documents {
        register("readme", TextDocument) {
            title = "README"
            content = "# Project Documentation"
        }
        register("logo", ImageDocument) {
            title = "Logo"
            width = 128
        }
    }
}
```

# 5. ObjectFactory

## 5.1 newInstance

```groovy
// ObjectFactory 提供的工厂方法
// 通过 project.objects 或 AbstractPlugin.objectFactory 访问

// 1. 创建简单对象
class CompileOptions {
    String encoding = "UTF-8"
    boolean debug = false
}

CompileOptions options = objectFactory.newInstance(CompileOptions)

// 2. 创建托管对象（属性自动为 Property 类型）
abstract class ServerConfig {
    abstract Property<String> getHost()
    abstract Property<Integer> getPort()
}

ServerConfig config = objectFactory.newInstance(ServerConfig)
config.host = "localhost"
config.port = 8080
```

## 5.2 fileCollection 和 fileTree

```groovy
// 文件集合
ConfigurableFileCollection libs = objectFactory.fileCollection()
libs.from("libs/*.jar")
libs.from(project(":lib").configurations.getByName("runtimeClasspath"))

// 文件树
ConfigurableFileTree sources = objectFactory.fileTree()
    .from("src/main/java")
    .matching { include("com/example/**") }
```

## 5.3 domainObjectContainer 和 polymorphicDomainObjectContainer

```groovy
// 在插件中创建容器
NamedDomainObjectContainer<NamedThing> things = 
    objectFactory.domainObjectContainer(NamedThing)
    
PolymorphicDomainObjectContainer<BaseThing> polyThings =
    objectFactory.polymorphicDomainObjectContainer(BaseThing)
```

# 6. Extensions 机制

## 6.1 ExtensionContainer

```groovy
// project.extensions 提供了 ExtensionContainer
ExtensionContainer extensions = project.extensions

// 注册扩展
extensions.add("myConfig", MyExtension)

// DSL 访问
myConfig {
    appName = "GradleApp"
}

// 通过 ExtensionContainer 获取
MyExtension config = extensions.getByType(MyExtension)
MyExtension config = extensions.findByType(MyExtension)
MyExtension config = extensions.getByName("myConfig")
```

## 6.2 ExtensionAware

任何实现 `ExtensionAware` 接口的对象都可以拥有扩展，如 Task、SourceSet、Configuration。

```groovy
// Task 上的扩展
task myTask {
    extensions.add("taskConfig", TaskConfig)
    taskConfig {
        timeout = 30
    }
}

// SourceSet 上的扩展
sourceSets {
    main {
        extensions.add("mainConfig", SourceConfig)
    }
}

// 在 ExtensionAware 上创建扩展的常用方法
extensions.create(String name, Class type, Object... args)
extensions.add(Class type, String name, instance)
extensions.configure(Class type, Action)
extensions.configure(String name, Action)
```

## 6.3 Convention vs Extension

Convention 是老版 Gradle 中的配置机制（Gradle 1.x-3.x），Extension 是后来引入的替代方案：

```groovy
// Convention 方式 (已弃用)
project.convention.plugins.java  // 返回 JavaPluginConvention
convention.plugins.java.sourceCompatibility = JavaVersion.VERSION_11

// Extension 方式 (现代方式)
project.extensions.getByType(JavaPluginExtension)
java {
    sourceCompatibility = JavaVersion.VERSION_11
}

// 区别：
// Convention: 类型不安全，通过名称访问，每个插件只有一个 Convention 对象
// Extension:   类型安全，可以有多个同类型扩展，支持 Property 惰性求值
```

# 7. Groovy MetaClass 技巧

## 7.1 动态方法注入

```groovy
class MyExtension {
    // 标准属性
    String name
    Integer version
    
    // 通过 MetaClass 添加动态方法
    static void enhanceMetaClass(MetaClass mc) {
        mc.configure = { Closure config ->
            config.delegate = this
            config.resolveStrategy = Closure.DELEGATE_FIRST
            config()
        }
    }
}

// 使用
myExtension {
    configure {
        name = "dynamic"
    }
}
```

## 7.2 methodMissing 和 propertyMissing

```groovy
class DynamicConfig {
    Map<String, Object> settings = [:]
    
    def propertyMissing(String name) {
        settings[name]
    }
    
    def propertyMissing(String name, Object value) {
        settings[name] = value
    }
    
    def methodMissing(String name, Object args) {
        if (args.length == 1 && args[0] instanceof Closure) {
            // 嵌套配置块
            def nested = new DynamicConfig()
            args[0].delegate = nested
            args[0].resolveStrategy = Closure.DELEGATE_FIRST
            args[0]()
            settings[name] = nested.settings
        }
    }
}
```

# 8. Kotlin DSL 扩展和类型安全访问器

## 8.1 类型安全访问器生成

```kotlin
// build.gradle.kts - Kotlin DSL 自动生成类型安全访问器

// 对于已注册的扩展，Kotlin DSL 生成扩展函数
// plugins { `java-library` }
// 自动生成: fun Project.`java-library`(...)

// 访问 precompiled script plugins 时确保类型安全
// src/main/kotlin/my-convention.gradle.kts:
//   abstract class MyExtension { ... }
//   extensions.create<MyExtension>("myExt")

// 使用时自动完成：
//   the<MyExtension>() 或 extensions.configure<MyExtension> {}
```

## 8.2 ext vs extra 属性

```groovy
// Groovy DSL - ext
ext {
    groupId = "com.example"
    artifactId = "my-lib"
}

// Kotlin DSL - extra
extra["groupId"] = "com.example"
extra["artifactId"] = "my-lib"

// 带类型的 extra 属性
val groupId by extra { "com.example" }
val artifactId by extra { "my-lib" }

// 注意：Kotlin DSL 中 ext 的访问方式不同
// ext { ... } 仅存在于 Groovy DSL
```

## 8.3 扩展函数

```kotlin
// Kotlin DSL 常用的扩展函数
// the<T>() —— 获取已注册的扩展
val javaExt = the<JavaPluginExtension>()
javaExt.sourceCompatibility = JavaVersion.VERSION_17

// configure<T> {} —— 配置扩展
configure<JavaPluginExtension> {
    sourceCompatibility = JavaVersion.VERSION_17
}

// withType<T> {}.configureEach
tasks.withType<JavaCompile>().configureEach {
    options.encoding = "UTF-8"
}

// named()
tasks.named<Jar>("jar") {
    archiveFileName = "my-app.jar"
}

// register()
tasks.register<MyTask>("myTask") {
    parameter = "value"
}
```

# 9. Copy Specs 深入

## 9.1 CopySpec 接口

```groovy
interface CopySpec extends ContentFilterable, CopyProcessingSpec {
    CopySpec from(Object... sourcePaths, Closure cl)
    CopySpec into(Object destPath)
    CopySpec include(String... includes)
    CopySpec exclude(String... excludes)
    CopySpec rename(Closure closure)
    CopySpec rename(String sourceRegEx, String replaceWith)
    CopySpec eachFile(Action action)
    CopySpec filesMatching(String pattern, Action action)
    CopySpec filesNotMatching(String pattern, Action action)
    CopySpec filter(Class filterType)
    CopySpec filter(Map<String,?> properties, Class filterType)
    CopySpec expand(Map<String,?> properties)
    CopySpec with(CopySpec... specs)
}
```

## 9.2 CopySpec 构建器模式

```groovy
task copyResources(type: Copy) {
    from("src/main/resources")
    from("src/main/static") {
        include("**/*.html")
        exclude("**/*.scss")
    }
    into("$buildDir/output")
    
    eachFile { details ->
        if (details.name.endsWith(".template")) {
            details.name = details.name - ".template"
        }
        details.relativePath = details.relativePath.prepend("prefix")
    }
    
    rename("(.+)\\.template", '$1')
    
    // 资源过滤：替换占位符
    filesMatching("**/*.properties") {
        filter(org.apache.tools.ant.filters.ReplaceTokens,
               tokens: [version: project.version, buildDate: new Date()])
    }
    
    // 或使用 expand 简化
    expand([
        appName: "MyApp",
        version: project.version
    ])
}

// 组合 CopySpec
def baseSpec = project.copySpec {
    from("src/main/resources")
    exclude("**/*.tmp")
}

task copyProd(type: Copy) {
    with(baseSpec)
    from("src/main/prod-resources")
    into("$buildDir/prod")
}
```

## 9.3 Sync Task

```groovy
// Sync 类似于 Copy，但会删除目标目录中不存在于源中的文件
task syncConfigs(type: Sync) {
    from("config/dev")
    from("config/prod")
    into("$buildDir/runtime-config")
    preserve {
        include("local.properties")   // 保留此文件不删除
        include("*.local")
    }
}
```

## 9.4 Jar 和 Zip 的 CopySpec

```groovy
task customJar(type: Jar) {
    // Jar/Zip 本身实现了 CopySpec
    from(sourceSets.main.output)
    from("license") {
        into("META-INF")
    }
    
    include("**/*.class")
    exclude("**/internal/**")
    
    manifest {
        attributes(
            "Implementation-Title": project.name,
            "Implementation-Version": project.version,
            "Main-Class": "com.example.Main"
        )
    }
    
    // 过滤签名文件
    exclude("META-INF/*.SF", "META-INF/*.DSA", "META-INF/*.RSA")
    
    // 对文件内容进行预处理
    filesMatching("META-INF/services/**") {
        filter { line ->
            line.startsWith("#") ? null : line
        }
    }
}
```

# 10. 项目属性与环境

## 10.1 gradle.properties 系统

```properties
# gradle.properties (项目级别)
org.gradle.jvmargs=-Xmx2048m -XX:MaxMetaspaceSize=512m
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true

# 自定义项目属性
version=1.0.0
description=Sample Gradle Project
releaseType=stable

# Kotlin 编译选项
kotlin.code.style=official
```

```properties
# ~/.gradle/gradle.properties (用户级别)
org.gradle.daemon=true
systemProp.http.proxyHost=proxy.example.com
systemProp.http.proxyPort=8080
```

## 10.2 属性来源优先级

属性读取的优先级（从高到低）：

1. 命令行 `-P` 参数：`gradle build -Pversion=2.0.0`
2. 系统属性 `-D` 参数：`gradle build -Dorg.gradle.debug=true`
3. 项目 `gradle.properties`
4. 用户 `~/.gradle/gradle.properties`
5. 环境变量：`ORG_GRADLE_PROJECT_propName=value`

```groovy
// 读取属性
String version = findProperty("version") ?: "0.0.0"
String envVar = System.getenv("CUSTOM_ENV")

// 使用 Provider 安全读取
Provider<String> versionProvider = providers.gradleProperty("version")
    .orElse("0.0.0")

Provider<String> envProvider = providers.environmentVariable("CUSTOM_ENV")
    .orElse("default")
```

## 10.3 buildEnvironment 任务

```groovy
// gradle buildEnvironment 命令查看构建环境
// 输出内容包括：
//   - Gradle 版本
//   - JVM 信息
//   - 操作系统信息
//   - 环境变量
//   - gradle.properties 中的属性
//   - 命令行动态参数
```

## 10.4 构建完成钩子

```groovy
// 在 settings.gradle 或 build.gradle 中注册
gradle.buildFinished { result ->
    if (result.failure) {
        println "构建失败: ${result.failure.message}"
    } else {
        println "构建成功"
    }
}

// settingsEvaluated, projectsLoaded, projectsEvaluated 等生命周期钩子
gradle.settingsEvaluated { settings ->
    println "Settings 解析完成，项目列表: ${settings.rootProject.name}"
}

gradle.projectsLoaded { gradleObj ->
    gradleObj.rootProject.subprojects {
        println "子项目: ${it.name}"
    }
}

gradle.projectsEvaluated { gradleObj ->
    println "所有项目配置完成，共 ${gradleObj.rootProject.allprojects.size()} 个项目"
}

gradle.taskGraph.whenReady { graph ->
    println "将有 ${graph.allTasks.size()} 个任务执行"
    
    // 检查是否执行了特定任务
    if (graph.hasTask(":publish")) {
        println "注意：将要执行发布任务"
    }
}
```

## 10.5 Action 配置模式

```groovy
// Action 接口：org.gradle.api.Action
interface Action<T> {
    void execute(T t)
}

// Groovy 闭包自动转换为 Action
tasks.named("compileJava") { task ->  // Closure 自动转为 Action<JavaCompile>
    task.options.encoding = "UTF-8"
}

// 手动创建 Action
Action<JavaCompile> configure = new Action<JavaCompile>() {
    void execute(JavaCompile compile) {
        compile.options.encoding = "UTF-8"
        compile.options.release = 17
    }
}
tasks.named("compileJava", JavaCompile).configure(configure)

// Kotlin DSL 中 Action 是 lambda
tasks.named<JavaCompile>("compileJava") {
    // this 是 JavaCompile 类型
    options.encoding = "UTF-8"
}
```

## 10.6 Closure 到 Action 的转换

Gradle 自动将 Groovy Closure 适配为 Action：

```groovy
// Gradle 内部实现
class ClosureBackedAction<T> implements Action<T> {
    private final Closure closure
    
    ClosureBackedAction(Closure closure) {
        this.closure = closure
    }
    
    void execute(T t) {
        // 设置 delegate 和 resolveStrategy
        def cloned = closure.clone()
        cloned.delegate = t
        cloned.resolveStrategy = Closure.DELEGATE_FIRST
        cloned.call(t)
    }
}

// 所以以下等价：
task.configure(new Action<Task>() { void execute(Task t) { t.enabled = false }})
task.configure({ it.enabled = false } as Action<Task>)
task.configure { enabled = false }
```

# 11. 高级扩展技术

## 11.1 嵌套扩展

```groovy
abstract class ServerConfig {
    abstract Property<String> getHost()
    abstract Property<Integer> getPort()
    
    @Nested
    abstract SecurityConfig getSecurity()
    
    @Nested
    abstract DatabaseConfig getDatabase()
}

abstract class SecurityConfig {
    abstract Property<Boolean> getSsl()
    abstract Property<String> getCertPath()
}

abstract class DatabaseConfig {
    abstract Property<String> getUrl()
    abstract Property<String> getUsername()
    abstract Property<String> getPassword()
}

// DSL 使用
server {
    host = "localhost"
    port = 8080
    
    security {
        ssl = true
        certPath = "/etc/certs/server.pem"
    }
    
    database {
        url = "jdbc:postgresql://localhost:5432/mydb"
        username = "admin"
    }
}
```

## 11.2 多个相同类型扩展

```groovy
// 使用 NamedDomainObjectContainer 管理多个同类型扩展
abstract class AppExtension {
    abstract NamedDomainObjectContainer<ServerConfig> getServers()
}

app {
    servers {
        register("web") {
            host = "example.com"
            port = 80
        }
        register("api") {
            host = "api.example.com"
            port = 443
        }
    }
}
```

## 11.3 扩展配置的 lazy finalization

```groovy
// 在 project.afterEvaluate 中完成扩展的最终处理（但不推荐）
// 现代方式：使用 finalizeValueOnRead

abstract class MySettings {
    abstract Property<String> getMode()
    abstract Property<String> getEffectiveMode()
}

settings {
    mode.set("development")
    
    // 惰性最终化：只在读取时计算
    effectiveMode.set(
        mode.map { m ->
            if (m == "development") "debug"
            else m
        }
    )
    
    // 或使用 convention + finalizeValueOnRead
    effectiveMode.convention(mode.map { "$it-mode" })
    effectiveMode.finalizeValueOnRead()
}
```
