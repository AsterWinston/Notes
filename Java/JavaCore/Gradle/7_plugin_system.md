
# 1. 插件的核心价值

## 1.1 约定优于配置（Convention over Configuration）

Gradle 插件体现"约定优于配置"理念。以 Java 插件为例，无需显式声明源码目录，插件默认约定：

```groovy
plugins {
    id 'java'
}
// 无需额外配置，以下约定自动生效：
// src/main/java        → 主源码
// src/test/java        → 测试源码
// src/main/resources   → 主资源
// build/classes        → 编译输出
// build/libs           → 打包输出
```

**插件提供的三大能力**：

| 能力 | 说明 | 示例 |
|------|------|------|
| **任务添加** | 向项目注册一组新任务 | java 插件添加 compileJava、test、jar 等 |
| **扩展对象** | 向项目添加 DSL 配置块 | `java { sourceCompatibility = 17 }` |
| **约定与默认值** | 设定合理的默认配置 | 源码目录约定、目标版本约定 |

## 1.2 无插件 vs 有插件的对比

```groovy
// 不使用 Java 插件的原始方式
tasks.register('compileMyJava', JavaCompile) {
    source = fileTree('src')
    classpath = files('lib/some.jar')
    destinationDirectory = file('build/classes')
}

// 使用 Java 插件后 — 一切自动化
plugins { id 'java' }
// compileJava、processResources、test、jar 等自动可用
```

# 2. 插件解析机制

## 2.1 PluginResolutionService 内部流程

Gradle 通过 `PluginResolutionService` 解析插件，流程如下：

```
用户声明 plugins { id 'x' version '1.0' }
        │
        ▼
┌─────────────────────────────────────┐
│  1. 解析 PluginDependenciesSpec     │
│     提取 id、version、apply         │
└─────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────┐
│  2. PluginRepositories 中查找       │
│     Gradle Plugin Portal (默认)     │
│     自定义仓库 (pluginManagement)   │
└─────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────┐
│  3. 下载 Plugin Marker Artifact     │
│     group: artifact.gradle.plugin   │
│     → 获取实际实现坐标               │
└─────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────┐
│  4. 解析实现 JAR 及其依赖           │
│     加入插件类路径                    │
└─────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────┐
│  5. 实例化 Plugin 实现类            │
│     调用 Plugin.apply(project)       │
└─────────────────────────────────────┘
```

## 2.2 PluginRepositories 配置

```groovy
// settings.gradle
pluginManagement {
    repositories {
        gradlePluginPortal()       // Gradle 官方仓库（默认）
        mavenCentral()             // Maven Central
        maven { url = 'https://repo.spring.io/plugins-release' }
        maven {
            url = 'https://nexus.mycompany.com/repository/gradle-plugins/'
            credentials {
                username = 'deployer'
                password = System.getenv('NEXUS_PASSWORD')
            }
        }
    }

    // 插件版本解析策略
    resolutionStrategy {
        eachPlugin {
            if (requested.id.namespace == 'com.mycompany') {
                useModule("com.mycompany:${requested.id.name}:${requested.version}")
            }
            if (requested.id.name == 'custom') {
                useVersion('2.0')  // 覆盖版本
            }
        }
    }
}
```

# 3. 核心插件（Core Plugins）

## 3.1 Java 系列插件

```groovy
plugins {
    id 'java'              // 基础 Java 编译/测试/打包
    id 'java-library'      // 扩展 java，添加 api/implementation 依赖区分
    id 'application'       // 可运行的 JVM 应用
    id 'war'               // Web 应用归档
    id 'ear'               // 企业应用归档
}
```

## 3.2 代码质量插件

```groovy
plugins {
    id 'checkstyle'   // Checkstyle 代码风格检查
    id 'pmd'          // PMD 静态分析
    id 'jacoco'       // 代码覆盖率
    id 'codenarc'     // Groovy 代码分析
}
```

**Checkstyle 插件配置示例**：

```groovy
checkstyle {
    toolVersion = '10.12.0'
    configFile = rootProject.file('config/checkstyle/checkstyle.xml')
    maxErrors = 0
    maxWarnings = 20
    reportsDir = file("$buildDir/reports/checkstyle")
}

tasks.named('checkstyleMain').configure {
    exclude '**/generated/**'
}
```

**JaCoCo 插件配置示例**：

```groovy
jacoco {
    toolVersion = '0.8.11'
}

jacocoTestReport {
    dependsOn test
    reports {
        xml.required = true
        csv.required = false
        html.outputLocation = layout.buildDirectory.dir('reports/jacoco')
    }
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = 0.80  // 80% 最低覆盖率
            }
        }
    }
}
```

## 3.3 其他常用核心插件

```groovy
plugins {
    id 'eclipse'       // Eclipse IDE 集成
    id 'idea'          // IntelliJ IDEA 集成
    id 'signing'       // 签名制品
    id 'maven-publish' // Maven 仓库发布
    id 'ivy-publish'   // Ivy 仓库发布
    id 'project-report'// 生成项目报告
    id 'distribution'  // 创建分发归档
}
```

# 4. 社区插件（Community Plugins）

```groovy
plugins {
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'com.google.protobuf' version '0.9.4'
    id 'org.jetbrains.kotlin.jvm' version '1.9.20'
    id 'com.github.ben-manes.versions' version '0.50.0'
    id 'io.gitlab.arturbosch.detekt' version '1.23.1'
}
```

**社区插件在 Plugin Portal 中的注册信息**：

```groovy
// settings.gradle — 自定义社区插件版本管理
pluginManagement {
    plugins {
        id 'org.springframework.boot' version '3.2.0'
        id 'io.spring.dependency-management' version '1.1.4'
        id 'org.jetbrains.kotlin.jvm' version '1.9.20'
    }
}
```

# 5. 应用插件的方式

## 5.1 `plugins {}` 块（推荐）

```groovy
plugins {
    id 'java'                        // 核心插件（无需版本）
    id 'org.springframework.boot' version '3.2.0'  // 社区插件（需版本）
    id 'io.spring.dependency-management' version '1.1.4' apply false  // 仅声明
}
```

**`plugins {}` 块的限制**：
- 只能出现在 `build.gradle(.kts)` 和 `settings.gradle(.kts)` 中
- 只能使用 `id` + `version` 语法（Gradle 6.0+ 支持 `id 'x' version 'y' apply false`）
- 不能在方法/闭包内部调用

## 5.2 `apply plugin:`（旧式语法）

```groovy
// 核心插件
apply plugin: 'java'
apply plugin: 'application'

// 社区插件（需先将插件加入 classpath）
buildscript {
    repositories { mavenCentral() }
    dependencies {
        classpath 'org.springframework.boot:spring-boot-gradle-plugin:3.2.0'
    }
}
apply plugin: 'org.springframework.boot'
```

`plugins {}` vs `apply plugin:` 对比：

| 特性 | `plugins {}` | `apply plugin:` |
|------|-------------|-----------------|
| 版本管理 | 内置 | 需 buildscript |
| 作用域 | 全局 PluginManager | 任意 Project |
| 解析时 | 初始化阶段 | 配置阶段 |
| 类型安全 | ID 校验 | 无校验 |
| 推荐度 | **推荐** | 旧项目兼容 |

## 5.3 `apply from:` — 脚本插件

```groovy
// 引入外部脚本
apply from: 'other.gradle'
apply from: 'https://company.com/scripts/common.gradle'
apply from: rootProject.file('gradle/jacoco.gradle')
```

**脚本插件示例**：

```groovy
// gradle/jacoco.gradle
apply plugin: 'jacoco'

jacoco {
    toolVersion = '0.8.11'
}

jacocoTestReport {
    reports {
        xml.required = true
    }
}
```

# 6. `pluginManagement` 块

## 6.1 集中管理插件仓库与版本

```groovy
// settings.gradle
pluginManagement {
    // 插件搜索仓库（仅对 plugins {} 块有效）
    repositories {
        gradlePluginPortal()
        mavenCentral()
        maven { url = 'https://repo.spring.io/milestone' }
    }

    // 集中声明插件版本
    plugins {
        id 'org.springframework.boot' version '3.2.0'
        id 'io.spring.dependency-management' version '1.1.4'
        id 'org.jetbrains.kotlin.jvm' version '1.9.20'
        id 'com.google.protobuf' version '0.9.4'
    }
}
```

在子项目中使用时无需指定版本：

```groovy
// subproject/build.gradle
plugins {
    id 'org.springframework.boot'
    id 'io.spring.dependency-management'
}
```

## 6.2 自定义插件解析映射

```groovy
// settings.gradle
pluginManagement {
    resolutionStrategy {
        eachPlugin {
            if (requested.id.namespace == 'com.mycompany') {
                useModule("com.mycompany.plugins:${requested.id.name}:${requested.version}")
            }
            // 将 Short ID 映射到完整坐标
            if (requested.id.name == 'shortname') {
                useModule("com.long.path:not-short-name:${requested.version}")
            }
        }
    }
}
```

# 7. 自定义脚本插件

## 7.1 本地脚本插件

```groovy
// buildSrc/src/main/groovy/common-conventions.gradle
plugins {
    id 'java'
    id 'checkstyle'
}

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

checkstyle {
    toolVersion = '10.12.0'
    configFile = rootProject.file('config/checkstyle.xml')
}

tasks.withType(JavaCompile).configureEach {
    options.encoding = 'UTF-8'
    options.compilerArgs << '-Xlint:unchecked'
}

tasks.withType(Test).configureEach {
    useJUnitPlatform()
    maxParallelForks = Runtime.runtime.availableProcessors()
}
```

在子项目中使用：

```groovy
// subproject/build.gradle
plugins {
    id 'common-conventions'
}
```

## 7.2 参数化脚本插件

```groovy
// buildSrc/src/main/groovy/java-module.gradle
plugins {
    id 'java-library'
}

// 使用 ext 属性接收参数
ext {
    moduleName = project.name
    moduleVersion = '1.0.0'
}

java {
    modularity.inferModulePath = true
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.0'
}
```

# 8. 预编译脚本插件（Precompiled Script Plugins）

## 8.1 目录结构与配置

```
buildSrc/
├── build.gradle.kts
│   └── plugins {
│         `kotlin-dsl`
│       }
│       repositories {
│         gradlePluginPortal()
│       }
├── settings.gradle.kts (可选)
└── src/main/kotlin/
    ├── my-java-library.gradle.kts        → id "my-java-library"
    ├── my-spring-boot.gradle.kts         → id "my-spring-boot"
    └── my-static-analysis.gradle.kts     → id "my-static-analysis"
```

## 8.2 Kotlin DSL 预编译插件示例

```kotlin
// buildSrc/src/main/kotlin/my-java-library.gradle.kts
plugins {
    `java-library`
    `maven-publish`
    checkstyle
    jacoco
}

group = "com.mycompany"
version = "1.0.0"

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
    withSourcesJar()
    withJavadocJar()
}

tasks.withType<JavaCompile>().configureEach {
    options.encoding = "UTF-8"
    options.compilerArgs.add("-Xlint:unchecked")
}

tasks.withType<Test>().configureEach {
    useJUnitPlatform()
    maxParallelForks = Runtime.getRuntime().availableProcessors()
    testLogging {
        events("passed", "skipped", "failed")
        showStandardStreams = true
    }
}

jacoco {
    toolVersion = "0.8.11"
}

publishing {
    publications {
        create<MavenPublication>("maven") {
            from(components["java"])
        }
    }
    repositories {
        maven {
            url = uri("https://nexus.mycompany.com/repository/releases/")
        }
    }
}
```

使用：

```kotlin
// subproject/build.gradle.kts
plugins {
    id("my-java-library")
}
```

## 8.3 Groovy DSL 预编译插件

```
buildSrc/src/main/groovy/my-java-library.gradle
```

```groovy
// buildSrc/src/main/groovy/my-java-library.gradle
plugins {
    id 'java-library'
    id 'maven-publish'
    id 'checkstyle'
    id 'jacoco'
}

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
    withSourcesJar()
    withJavadocJar()
}
```

在 `buildSrc/build.gradle` 中声明：

```groovy
plugins {
    id 'groovy-gradle-plugin'
}
```

# 9. 二进制插件结构

## 9.1 `Plugin<T>` 接口详解

```java
public interface Plugin<T> {
    void apply(T target);
}
```

- 对于项目插件：`T = Project` → `void apply(Project project)`
- 对于 Settings 插件：`T = Settings` → `void apply(Settings settings)`
- 对于初始化插件：`T = Gradle` → `void apply(Gradle gradle)`

## 9.2 实现一个项目插件

```groovy
// MyPlugin.groovy
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.api.tasks.testing.Test

class MyPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        // 1. 添加扩展
        def extension = project.extensions.create('myConfig', MyExtension)

        // 2. 注册任务
        project.tasks.register('hello', HelloTask) {
            group = 'custom'
            description = '自定义问候任务'
        }

        // 3. 添加约定
        project.afterEvaluate {
            def config = project.extensions.getByType(MyExtension)
            project.tasks.withType(Test).configureEach {
                maxHeapSize = config.maxHeapSize
            }
        }
    }
}

class MyExtension {
    String greeting = 'Hello'
    String name = 'World'
    String maxHeapSize = '1g'
}

class HelloTask extends DefaultTask {
    @TaskAction
    void sayHello() {
        def config = project.extensions.getByType(MyExtension)
        println "${config.greeting}, ${config.name}!"
    }
}
```

## 9.3 Kotlin 实现二进制插件

```kotlin
// MyPlugin.kt
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.api.tasks.testing.Test

open class MyExtension {
    var greeting: String = "Hello"
    var name: String = "World"
    var maxHeapSize: String = "1g"
}

class MyPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        val extension = project.extensions.create("myConfig", MyExtension::class.java)

        project.tasks.register("hello") {
            group = "custom"
            description = "自定义问候任务"
            doLast {
                val config = project.extensions.getByType(MyExtension::class.java)
                println("${config.greeting}, ${config.name}!")
            }
        }

        project.afterEvaluate {
            val config = project.extensions.getByType(MyExtension::class.java)
            project.tasks.withType(Test::class.java).configureEach {
                maxHeapSize = config.maxHeapSize
            }
        }
    }
}
```

## 9.4 Plugin ID 命名约定

| 类型 | 命名格式 | 示例 |
|------|---------|------|
| 核心插件 | 简短名称（无域名） | `java`、`application`、`maven-publish` |
| 社区插件 | 反向域名 | `org.springframework.boot`、`com.google.protobuf` |
| 自定义公司插件 | 公司反向域名 | `com.mycompany.myplugin` |

```groovy
// 资源文件: src/main/resources/META-INF/gradle-plugins/com.mycompany.myplugin.properties
// 内容:
// implementation-class=com.mycompany.MyPlugin
```

# 10. 扩展（Extensions）详解

## 10.1 创建扩展的多种方式

```groovy
// 方式 1：简单普通对象（Plain Object）
project.extensions.add('simpleConfig', new SimpleConfig(greeting: 'Hi'))

// 方式 2：ExtensionAware 扩展容器
project.extensions.create('nested', NestedExtension)

// 方式 3：延迟配置的 NamedDomainObjectContainer
project.extensions.add('flavors', project.container(ProductFlavor))

class ProductFlavor {
    String name
    String description
    int minSdk
}
```

## 10.2 带嵌套 DSL 的扩展

```groovy
class BuildConfig {
    String outputDir

    // 使用 Closure 实现嵌套 DSL
    void packaging(Closure closure) {
        closure.delegate = new PackagingConfig()
        closure.resolveStrategy = Closure.DELEGATE_FIRST
        closure.call()
    }

    static class PackagingConfig {
        String format = 'zip'
        boolean includeSources = false
    }
}
```

DSL 使用效果：

```groovy
buildConfig {
    outputDir = 'dist'
    packaging {
        format = 'tar.gz'
        includeSources = true
    }
}
```

# 11. 插件版本管理

## 11.1 版本目录（Version Catalog）

```toml
# gradle/libs.versions.toml
[versions]
spring-boot = "3.2.0"
kotlin = "1.9.20"

[plugins]
spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
kotlin-jvm = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
```

```kotlin
// settings.gradle.kts
@Suppress("UnstableApiUsage")
dependencyResolutionManagement {
    versionCatalogs {
        create("libs") {
            from(files("gradle/libs.versions.toml"))
        }
    }
}
```

使用：

```kotlin
// build.gradle.kts
plugins {
    alias(libs.plugins.spring.boot)
    alias(libs.plugins.kotlin.jvm)
}
```

## 11.2 通过 BOM 管理插件版本

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.2.0"))
}
```

# 12. 插件发布

## 12.1 发布到 Gradle Plugin Portal

```groovy
// build.gradle
plugins {
    id 'java-gradle-plugin'
    id 'maven-publish'
    id 'com.gradle.plugin-publish' version '1.2.1'
}

gradlePlugin {
    website = 'https://github.com/myorg/myplugin'
    vcsUrl = 'https://github.com/myorg/myplugin.git'

    plugins {
        myPlugin {
            id = 'com.mycompany.myplugin'
            implementationClass = 'com.mycompany.MyPlugin'
            displayName = 'My Custom Plugin'
            description = 'A plugin that does amazing things'
            tags.set(['java', 'code-generation'])
        }
    }
}
```

发布命令：

```bash
gradle publishPlugins -Pgradle.publish.key=<key> -Pgradle.publish.secret=<secret>
```

## 12.2 Gradle Plugin Marker Artifacts

当发布插件时，除了主 JAR，Gradle 会自动生成 Marker Artifact：

```
com.mycompany:com.mycompany.myplugin.gradle.plugin:1.0.0
  └── pom 依赖指向: com.mycompany:myplugin:1.0.0
```

Marker Artifact 的 groupId 规则：`<plugin-id>` 中最后一个 `.` 之前的部分 + 后缀 `.gradle.plugin`。artifactId 保留原样。这让 Gradle 能够在 `plugins {}` 块中通过 `id` 找到正确的实现坐标。

## 12.3 使用 Maven Central 发布

```groovy
publishing {
    publications {
        maven(MavenPublication) {
            from components.java
        }
    }
    repositories {
        maven {
            url = 'https://oss.sonatype.org/service/local/staging/deploy/maven2/'
            credentials {
                username = System.getenv('OSSRH_USERNAME')
                password = System.getenv('OSSRH_PASSWORD')
            }
        }
    }
}
```

**注意**：发布到 Maven Central 时，用户不能直接通过 `plugins {}` 块引用（因为 Plugin Portal 是默认仓库）。需要在 `settings.gradle` 中配置：

```groovy
// settings.gradle
pluginManagement {
    repositories {
        mavenCentral()
        gradlePluginPortal()
    }
}
```

---

**总结**：Gradle 插件系统是理解 Gradle 架构的关键。核心要点：（1）插件的本质是 `Plugin<Project>.apply()` 方法，负责注册任务、扩展和约定；（2）优先使用 `plugins {}` 块而非 `apply plugin:`；（3）利用 `pluginManagement` 集中管理插件版本和仓库；（4）预编译脚本插件是跨项目共享构建逻辑的最佳方式；（5）正确声明 inputs/outputs 的插件才能享受增量构建；（6）使用 `java-gradle-plugin` 和 Plugin Portal 发布自己的插件。
