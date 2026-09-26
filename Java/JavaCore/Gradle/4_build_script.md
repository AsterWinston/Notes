
# 1. settings.gradle(.kts) 完全指南

## 1.1 settings 脚本的作用和时机

settings 脚本在 Gradle 构建生命周期的**初始化阶段**执行，它定义了哪些项目参与本次构建以及如何处理它们。

settings 文件的位置决定了根项目的边界。Gradle 从当前目录向上查找 settings 文件，找到的第一个就是根项目位置。

```kotlin
// settings.gradle.kts - 完整的配置示例
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
        google()
    }
    plugins {
        kotlin("jvm") version "1.9.20"
        id("org.springframework.boot") version "3.2.0"
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)
    repositories {
        mavenCentral()
        google()
        maven("https://jitpack.io")
    }
}

rootProject.name = "my-multimodule-app"

include("core")
include("core:model")       // 嵌套子项目
include("core:service")     // 嵌套子项目
include("web")
include("cli")

// 为非标准目录的子项目指定路径
include(":legacy-module")
project(":legacy-module").projectDir = file("old/legacy-module")

// 从外部目录引入项目（不推荐，用 composite build 更好）
// includeFlat("external-lib")
```

## 1.2 pluginManagement 详解

`pluginManagement {}` 是 settings 脚本中最优先执行的块，负责解析插件的版本和仓库。它出现在所有其他代码之前。

```kotlin
// settings.gradle.kts
pluginManagement {

    // repositories: 定义从哪里下载插件 JAR
    repositories {
        gradlePluginPortal()  // https://plugins.gradle.org/m2
        mavenCentral()
        google()
        // 私有插件仓库
        maven {
            url = uri("https://nexus.mycompany.com/repository/gradle-plugins/")
            credentials {
                username = System.getenv("NEXUS_USER") ?: "defaultUser"
                password = System.getenv("NEXUS_PASS") ?: "defaultPass"
            }
        }
    }

    // plugins: 集中声明插件版本（子项目引用时不需要写版本）
    plugins {
        // Kotlin 的语法糖（自动使用 kotlin.jvm、kotlin.android 等 ID）
        kotlin("jvm") version "1.9.20"
        kotlin("plugin.spring") version "1.9.20"

        // 标准的 id + version 声明
        id("org.springframework.boot") version "3.2.0"
        id("io.spring.dependency-management") version "1.1.4"
        id("com.diffplug.spotless") version "6.23.3"
        id("io.gitlab.arturbosch.detekt") version "1.23.4"
    }

    // resolutionStrategy: 自定义插件解析逻辑
    resolutionStrategy {
        // 为特定插件的每个请求应用自定义规则
        eachPlugin {
            when (requested.id.namespace) {
                "com.mycompany" -> {
                    // 将 plugin ID 映射到 Maven 坐标
                    useModule("com.mycompany:${requested.id.name}-plugin:${requested.version}")
                }
            }

            // 检查是否使用了快照版本
            if (requested.version?.endsWith("-SNAPSHOT") == true) {
                logger.warn("插件 ${requested.id} 使用了快照版本")
            }
        }
    }
}
```

pluginManagement 在 settings 中的位置和优先级：
```
settings.gradle.kts
    │
    ├── pluginManagement {}  ← 最先处理（甚至在 settings 脚本执行前）
    │     ├── 解析插件版本
    │     └── 建立 plugin → artifact 映射
    │
    ├── dependencyResolutionManagement {}  ← 全局依赖仓库配置
    │
    └── 其余 settings 代码  ← 项目结构定义
```

## 1.3 dependencyResolutionManagement 详解

`dependencyResolutionManagement {}` 用于集中管理所有项目的依赖仓库。它通过限制每个子项目独立声明仓库来确保构建的一致性。

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    // repositoriesMode 策略：
    // PREFER_PROJECT: 优先使用项目声明的仓库，fallback 到这里
    // PREFER_SETTINGS: 优先使用这里的仓库，忽略项目声明的
    // FAIL_ON_PROJECT_REPOS: 项目声明仓库时报错（最严格）
    repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)

    repositories {
        mavenCentral()
        google()

        maven {
            url = uri("https://repo.spring.io/milestone")
            content {
                includeGroup("org.springframework")
                includeModule("io.micrometer", "micrometer-core")
            }
        }

        maven {
            url = uri("https://nexus.mycompany.com/repository/internal/")
            credentials {
                username = providers.environmentVariable("NEXUS_USER")
                    .orElse("guest")
                password = providers.environmentVariable("NEXUS_PASS")
                    .orElse("guest")
            }
        }
    }

    // 版本目录规则
    versionCatalogs {
        create("libs") {
            from(files("gradle/libs.versions.toml"))
        }
        create("testLibs") {
            from(files("gradle/test-libs.versions.toml"))
        }
    }

    // 控制依赖解析的全局规则
    resolutionStrategy {
        // 强制使用特定版本
        force("com.google.guava:guava:32.1.3-jre")

        // 缓存动态版本的时间
        cacheDynamicVersionsFor(10, "minutes")
        cacheChangingModulesFor(4, "hours")

        // 当版本冲突时失败
        // failOnVersionConflict()
    }
}
```

## 1.4 include / includeFlat / includeBuild 的区别

```kotlin
// settings.gradle.kts

// === include：标准子项目 ===
// 路径相对于根项目目录
include("core")
include("core:model", "core:service")  // 嵌套项目
// 子项目在根项目的子目录中

// === includeFlat：同级目录的子项目 ===
// 子项目在根项目的同级目录中
includeFlat("shared-library", "shared-utils")
// 目录结构：
// workspace/
//   ├── my-app/              ← 根项目
//   │   ├── settings.gradle.kts
//   │   └── core/
//   ├── shared-library/       ← 通过 includeFlat 引入
//   └── shared-utils/         ← 通过 includeFlat 引入

// === includeBuild：组合构建 ===
// 将另一个完全独立的 Gradle 项目纳入构建
includeBuild("build-logic")     // 用于 buildSrc 替代方案
includeBuild("../other-project") // 外部项目
// 特点：
//   - 被包含的项目保留自己完整的 settings.gradle 和构建脚本
//   - 可以替换外部依赖（dependency substitution）
```

图示：include、includeFlat、includeBuild 的区别
```
my-app/                          ← 标准多项目构建
├── settings.gradle.kts
│     include("core")            ← 子目录中
│     include("web")
│     includeFlat("shared-lib")  ← 同级目录中
│     includeBuild("build-logic") ← 组合构建
├── core/
├── web/
├── build-logic/                 ← 独立的 Gradle 项目
│   ├── settings.gradle.kts
│   └── build.gradle.kts
└── gradlew

shared-lib/                      ← 独立的 Gradle 项目（同级目录）
├── build.gradle.kts
└── src/
```

# 2. build.gradle(.kts) 完全指南

## 2.1 Groovy DSL 语法深入

Groovy DSL 利用 Groovy 语言的动态特性（元编程、闭包委托、方法缺失处理）实现了极其简洁的构建脚本语法。

**闭包委托机制（Closure Delegation）：**

```groovy
// build.gradle - Groovy DSL 的底层委托机制

// 当你在闭包中写顶层代码时：
repositories {
    mavenCentral()
}

// Gradle 内部等价于：
project.repositories(new Closure<Object>(this, this) {
    void doCall() {
        delegate.mavenCentral()  // delegate = RepositoryHandler
    }
})

// Groovy DSL 允许你用代码来做声明
// 例如动态地决定要添加哪些依赖
def features = ['logging', 'monitoring', 'security']

dependencies {
    // 动态依赖声明
    features.each { feature ->
        if (feature == 'logging') {
            implementation 'org.slf4j:slf4j-api:2.0.9'
        } else if (feature == 'monitoring') {
            implementation 'io.micrometer:micrometer-core:1.12.0'
        } else if (feature == 'security') {
            implementation 'org.springframework.security:spring-security-core:6.2.0'
        }
    }
}
```

**Groovy DSL 的动态方法：**

```groovy
// build.gradle - Groovy DSL 的动态特性示例

// groovy 允许访问不存在的属性（通过 getProperty/setProperty）
version = '1.0.0'  // 等价于 project.setVersion('1.0.0')

// groovy 允许调用不存在的方法（通过 methodMissing）
task hello {
    doLast {
        println 'Hello Groovy DSL'
    }
}

// 类型不严格的依赖声明
dependencies {
    // 以下几种写法在 Groovy DSL 中都等价
    implementation 'com.google.guava:guava:32.1.3-jre'
    implementation('com.google.guava:guava:32.1.3-jre')
    implementation group: 'com.google.guava', name: 'guava', version: '32.1.3-jre'

    // 多模块依赖（动态版本）
    [':core', ':api', ':service'].each { module ->
        implementation project(module)
    }
}

// Gradle 属性可以通过 project.ext 扩展
ext {
    springVersion = '6.1.1'
    jacksonVersion = '2.16.0'
}

dependencies {
    // 使用扩展属性关联版本
    implementation "org.springframework:spring-core:${springVersion}"
    implementation "com.fasterxml.jackson.core:jackson-databind:${jacksonVersion}"
}
```

## 2.2 Kotlin DSL 语法深入

Kotlin DSL 利用 Kotlin 的类型安全、扩展函数和 Lambda with Receiver 来实现构建脚本。它提供了比 Groovy DSL 更好的 IDE 支持和编译时检查。

**Kotlin DSL 的 Lambda with Receiver：**

```kotlin
// build.gradle.kts - Kotlin DSL 语法解析

// repositories {} 块的类型分析
repositories {
    // this: RepositoryHandler
    mavenCentral()   // fun RepositoryHandler.mavenCentral(): MavenArtifactRepository
    google()         // fun RepositoryHandler.google(): MavenArtifactRepository
    maven {          // fun RepositoryHandler.maven(action: Action<MavenArtifactRepository>)
        // this: MavenArtifactRepository
        url = uri("https://jitpack.io")
    }
}

// dependencies {} 块的类型分析
dependencies {
    // this: DependencyHandler
    // fun DependencyHandler.implementation(notation: Any): Dependency?
    implementation("com.google.guava:guava:32.1.3-jre")

    // 使用版本目录（类型安全）
    implementation(libs.guava)
    testImplementation(libs.junit.jupiter)
}

// tasks {} 块的类型分析
tasks {
    // this: TaskContainer
    // fun TaskContainer.register(name: String, action: Action<Task>): TaskProvider<Task>
    register("hello") {
        // this: Task
        doLast {
            println("Hello Kotlin DSL")
        }
    }

    // 类型安全的命名任务
    named<Test>("test") {
        // this: Test（类型安全）
        useJUnitPlatform()
    }
}
```

**Kotlin DSL 的类型安全访问器（Type-Safe Accessors）：**

Gradle 为每个插件生成的类型安全访问器允许 IDE 进行精确的代码补全：

```kotlin
// 当应用 java 插件后，Gradle 生成以下类型安全访问器
// sourceSets {} → sourceSets(action: Action<SourceSetContainer>)
// configurations {} → configurations(action: Action<ConfigurationContainer>)

sourceSets {
    main {  // 类型安全的 SourceSet
        java {
            srcDir("src/custom")
        }
    }
}

// 当应用 application 插件后
application {
    mainClass.set("com.example.App")  // 类型安全的属性
}

// 当应用 spring boot 插件后
springBoot {
    mainClass.set("com.example.BootApp")
}
```

**Kotlin DSL 中的委托属性：**

```kotlin
// build.gradle.kts

// 从 gradle.properties 读取属性（惰性）
val appName: String by project
val maxMemory: String by project

// 从系统属性读取
val javaVersion: String by rootProject.extra

// 自定义委托（从 extensions 中获取）
val springBootVersion: String by lazy {
    project.extensions.extraProperties["springBootVersion"] as? String ?: "3.2.0"
}
```

## 2.3 Groovy DSL 与 Kotlin DSL 关键差异

| 特性 | Groovy DSL | Kotlin DSL |
|------|-----------|-----------|
| 文件扩展名 | `.gradle` | `.gradle.kts` |
| 类型安全 | 动态类型 | 静态类型（编译时检查） |
| IDE 支持 | 有限的自动补全 | 完整的代码补全和导航 |
| 依赖声明 | `implementation 'g:a:v'` | `implementation("g:a:v")` |
| Task 配置 | `taskName { ... }` | `tasks.named<T>("taskName") { ... }` |
| 字符串 | `'string'` 和 `"$interpolation"` | `"string"` 和 `"$interpolation"`，还有 `"""raw"""` |
| 属性访问 | 自动 fallback 到 project 属性 | 必须显式声明或使用委托 |
| 脚本缓存 | 较慢（Groovy 编译慢） | 更快（Kotlin 增量编译） |
| `ext` 扩展属性 | `ext { key = value }` | `extra["key"] = value` 或 `extra { set("key", value) }` |
| 脚本插件 | `apply from: 'file.gradle'` | `apply(from = "file.gradle.kts")` |

```kotlin
// Groovy → Kotlin DSL 对照表

// === Groovy ===
// version = '1.0'
// group = 'com.example'

// === Kotlin ===
// version = "1.0"
// group = "com.example"

// === Groovy（动态版本范围）===
// implementation 'com.google.guava:guava:31+'

// === Kotlin ===
// implementation("com.google.guava:guava:31+")

// === Groovy（从 ext 读取属性）===
// ext { springVersion = '3.2.0' }
// implementation "org.springframework:spring-core:$springVersion"

// === Kotlin（使用 extra）===
// extra["springVersion"] = "3.2.0"
// val springVersion: String by extra
// implementation("org.springframework:spring-core:$springVersion")

// === Groovy（Task 配置）===
// compileJava {
//     options.compilerArgs << '-Xlint:unchecked'
// }

// === Kotlin ===
// tasks.compileJava {
//     options.compilerArgs.add("-Xlint:unchecked")
// }
// 或使用 named:
// tasks.named<JavaCompile>("compileJava") {
//     options.compilerArgs.add("-Xlint:unchecked")
// }
```

## 2.4 构建脚本的编译与缓存

Gradle 构建脚本不是解释执行的，而是经过编译后运行的：

```
build.gradle.kts
    │
    ├── Kotlin Compiler → .class 字节码
    │
    ├── ClassLoader 加载
    │
    └── 执行编译后的类
```

Gradle 对构建脚本进行缓存以提高性能：

```properties
# gradle.properties - 构建脚本缓存的配置

# Kotlin DSL 脚本编译输出目录
# 默认：GRADLE_USER_HOME/caches/<version>/kotlin-dsl/

# 增量编译：只重新编译修改过的脚本
# 这是默认行为

# 编译后的脚本 JAR 缓存位置
# ~/.gradle/caches/<version>/scripts/
```

Kotlin DSL 编译缓存的优势：
- 增量编译：只有修改过的 `.gradle.kts` 会重新编译
- 类隔离：每个脚本编译为独立的 class
- 预编译插件：`buildSrc` 中的约定插件在有变动时增量编译

## 2.5 脚本中的操作符和语法糖

```kotlin
// Kotlin DSL 中的常见操作符

// << 操作符（已弃用，Groovy 中常见）
// task clean(type: Delete) << { ... }  ← Groovy 旧语法
// 现代写法：tasks.register("clean", Delete::class) { ... }

// ! 操作符（否定）
tasks.register("task1")
tasks.register("task2") {
    dependsOn(tasks.named("task1"), tasks.named("!task1"))  // 排除依赖
}

// 文件操作符
val sourceDir = file("src/main/java")
val allJavaFiles = fileTree("src") { include("**/*.java") }  // Groovy
val allJavaFiles2 = fileTree("src").matching { include("**/*.java") }  // Kotlin

// 路径简化
// Groovy: file('src/main/java')
// Kotlin: file("src/main/java")

// 等于和不等于
// Groovy: version == '1.0'  ← Groovy 的 == 等价于 Java 的 equals()
// Kotlin: version == "1.0"  ← Kotlin 的 == 等价于 equals()

// 集合操作
// Groovy: ['a', 'b', 'c'].each { }
// Kotlin: listOf("a", "b", "c").forEach { }
```

# 3. buildSrc 与复合构建

## 3.1 buildSrc 目录详解

`buildSrc` 是一个特殊的 Gradle 项目，用于存放构建逻辑的共享代码。它位于根项目目录下，在构建主项目之前自动编译并添加到构建脚本的 classpath 中。

```
my-project/
├── buildSrc/
│   ├── settings.gradle.kts               ← （可选）buildSrc 的 settings
│   ├── build.gradle.kts                  ← buildSrc 自身的构建脚本
│   └── src/
│       └── main/
│           ├── kotlin/                   ← 预编译脚本插件和工具类
│           │   ├── JavaConventionPlugin.kt
│           │   ├── DependencyVersions.kt
│           │   └── Utils.kt
│           └── resources/
│               └── META-INF/
│                   └── gradle-plugins/   ← 插件描述符（自动生成）
├── core/
│   └── build.gradle.kts
└── settings.gradle.kts
```

**buildSrc/build.gradle.kts：**
```kotlin
// buildSrc 使用 kotlin-dsl 插件获得所有 Gradle API
plugins {
    `kotlin-dsl`
}

repositories {
    mavenCentral()
}

dependencies {
    // 可以在此处添加第三方库用于构建逻辑
    implementation("org.yaml:snakeyaml:2.2")
    implementation("com.fasterxml.jackson.core:jackson-databind:2.16.0")
}

// Kotlin 编译设置
kotlin {
    jvmToolchain(17)
}
```

**buildSrc/src/main/kotlin/JavaConventionPlugin.kt：**
```kotlin
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.api.tasks.testing.Test
import org.gradle.api.plugins.JavaPluginExtension
import org.gradle.jvm.toolchain.JavaLanguageVersion

class JavaConventionPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        project.run {
            pluginManager.apply("java")

            extensions.configure<JavaPluginExtension> {
                sourceCompatibility = JavaVersion.VERSION_17
                targetCompatibility = JavaVersion.VERSION_17
            }

            repositories {
                mavenCentral()
            }

            dependencies {
                add("testImplementation", platform("org.junit:junit-bom:5.10.1"))
                add("testImplementation", "org.junit.jupiter:junit-jupiter")
            }

            tasks.withType<Test> {
                useJUnitPlatform()
            }
        }
    }
}
```

**预编译脚本插件（自动生成）：**

```
buildSrc/src/main/kotlin/
├── java-convention.gradle.kts  ← 文件名即插件 ID
└── spring-convention.gradle.kts
```

```kotlin
// buildSrc/src/main/kotlin/java-convention.gradle.kts
// 不需要类名和 Plugin 接口，Gradle 自动将其编译为插件
// 插件 ID 自动为 "java-convention"

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

tasks.withType<JavaCompile> {
    options.encoding = "UTF-8"
}

// 子项目中使用：
// plugins {
//     `java-convention`
// }
```

## 3.2 buildSrc 的变更是如何触发的

buildSrc 的变更检测规则：
- **修改了 buildSrc 的任何源文件** → 整个 buildSrc 重新编译 → 所有子项目失效 → 重新配置
- **修改了任何构建脚本** → 该脚本重新编译 → 所属项目重新配置
- 这意味着 buildSrc 的变更代价较高，因此应将稳定的构建逻辑放在这里

```bash
# buildSrc 变更时的典型输出
> Task :buildSrc:compileKotlin UP-TO-DATE
> Task :buildSrc:compileJava NO-SOURCE
# ... 如果 buildSrc 没变，跳过配置，节省时间
```

## 3.3 复合构建（Composite Builds）

复合构建是 buildSrc 的强大替代方案。它将构建逻辑放在一个独立的 Gradle 项目中，通过 `includeBuild` 引入：

```
workspace/
├── my-app/                      ← 主项目
│   ├── settings.gradle.kts
│   ├── build.gradle.kts
│   └── core/
├── build-logic/                 ← 构建逻辑项目（独立的 Gradle 项目）
│   ├── settings.gradle.kts
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│       ├── java-convention.gradle.kts
│       └── kotlin-convention.gradle.kts
└── shared-library/              ← （可选）共享库
    └── build.gradle.kts
```

**主项目的 settings.gradle.kts：**
```kotlin
// settings.gradle.kts

pluginManagement {
    includeBuild("build-logic")  // 将构建逻辑项目纳入复合构建
    // includeBuild("../other-logic")  // 可以包含多个
}

// 现在可以直接使用 build-logic 中的插件
```

**build-logic 项目的 settings.gradle.kts：**
```kotlin
// build-logic/settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        mavenCentral()
        gradlePluginPortal()
    }
}

rootProject.name = "build-logic"
```

**build-logic 项目的 build.gradle.kts：**
```kotlin
// build-logic/build.gradle.kts
plugins {
    `kotlin-dsl`
}

repositories {
    mavenCentral()
    gradlePluginPortal()
}

dependencies {
    // 构建逻辑的依赖（这些依赖不会泄露给主项目）
    implementation("org.jetbrains.kotlin:kotlin-gradle-plugin:1.9.20")
    implementation("org.springframework.boot:spring-boot-gradle-plugin:3.2.0")
}

kotlin {
    jvmToolchain(17)
}
```

**复合构建的优势：**
1. **隔离变更**：修改构建逻辑不需要重新编译主项目
2. **独立性**：构建逻辑项目可以独立发版和测试
3. **测试友好**：可以为构建逻辑编写单元测试
4. **可复用**：多个主项目可以共享同一个构建逻辑项目

# 4. 初始化脚本（Init Scripts）

## 4.1 Init Script 的位置和用途

初始化脚本在 Gradle 构建生命周期的**最早阶段**执行，甚至在 settings 脚本之前。它们用于全局配置。

| 位置 | 作用范围 | 说明 |
|------|----------|------|
| `GRADLE_USER_HOME/init.gradle(.kts)` | 所有 Gradle 调用 | 用户级全局初始化 |
| `GRADLE_USER_HOME/init.d/*.gradle(.kts)` | 所有 Gradle 调用 | 用户级全局（多个文件） |
| `GRADLE_HOME/init.d/*.gradle(.kts)` | 所有 Gradle 调用 | 安装级全局 |
| `--init-script` 命令行参数 | 单次构建 | 按需指定 |

```bash
# 指定初始化脚本
gradle build --init-script /path/to/custom-init.gradle.kts

# 多个初始化脚本的加载顺序：
# 1. GRADLE_HOME/init.d/*  (按字母顺序)
# 2. GRADLE_USER_HOME/init.d/*  (按字母顺序)
# 3. GRADLE_USER_HOME/init.gradle
# 4. --init-script 指定的脚本
# 后加载的脚本可以覆盖前面脚本的配置
```

## 4.2 Init Script 示例

```kotlin
// ~/.gradle/init.gradle.kts - 公司统一的全局初始化脚本

// 注入全局仓库（确保所有项目都能访问私有仓库）
settingsEvaluated {
    dependencyResolutionManagement {
        repositories {
            // 优先使用公司私有仓库
            maven {
                name = "CompanyRepo"
                url = uri("https://nexus.mycompany.com/repository/maven-public/")
            }
            // 公共仓库作为 fallback
            mavenCentral()
        }
    }
}

// 全局插件管理
beforeSettings {
    pluginManagement {
        repositories {
            maven("https://nexus.mycompany.com/repository/gradle-plugins/")
        }
    }
}

// 为所有项目添加通用配置
allprojects {
    // 全局编译器设置
    tasks.withType<JavaCompile> {
        options.encoding = "UTF-8"
        options.isFork = true
        options.forkOptions.jvmArgs = listOf("-Xmx512m")
    }

    // 全局测试配置
    tasks.withType<Test> {
        maxParallelForks = (Runtime.getRuntime().availableProcessors() / 2).coerceAtLeast(1)
        testLogging {
            events("passed", "skipped", "failed")
        }
    }

    // 添加构建信息 JAR Manifest
    tasks.withType<Jar> {
        manifest {
            attributes(
                "Built-By" to System.getProperty("user.name"),
                "Build-Jdk" to System.getProperty("java.version"),
                "Build-Timestamp" to java.time.Instant.now().toString(),
                "Implementation-Vendor" to "MyCompany"
            )
        }
    }
}

// 全局 Gradle 配置
gradle.allprojects {
    // 为每个项目添加一个 "hello" 任务（仅用于演示）
    tasks.register("initHello") {
        doLast {
            println("Hello from init script to project: $name")
        }
    }
}

// 监听构建完成
gradle.buildFinished { result ->
    if (result.failure != null) {
        println("构建失败: ${result.failure?.message}")
    } else {
        println("所有项目构建成功")
    }
}
```

## 4.3 Init Script 强制执行公司策略

```kotlin
// ~/.gradle/init.d/security-check.gradle.kts

// 全局 Gradle 实例
gradle.settingsEvaluated {
    // 检查 settings 中是否有不安全的仓库
    it.dependencyResolutionManagement.repositories.all { repo ->
        if (repo is MavenArtifactRepository) {
            val url = repo.url.toString()
            if (url.startsWith("http://") && !url.contains("localhost")) {
                throw GradleException("安全违规：settings 中包含不安全的 HTTP 仓库: $url")
            }
        }
    }
}

allprojects {
    afterEvaluate {
        repositories.forEach { repo ->
            if (repo is MavenArtifactRepository) {
                val url = repo.url.toString()
                if (url.startsWith("http://") && !url.contains("localhost")) {
                    logger.warn("警告：项目 ${name} 包含不安全的 HTTP 仓库: $url")
                }
            }
        }
    }

    // 强制要求每个项目必须应用 java 插件（或其他基本插件）
    // pluginManager.withPlugin("java") { ... }
}
```

# 5. 版本目录（Version Catalog）

## 5.1 libs.versions.toml

Gradle 7.0 引入了版本目录作为集中管理依赖版本的标准方式。使用 TOML 文件格式。

```toml
# gradle/libs.versions.toml

[versions]
kotlin = "1.9.20"
spring-boot = "3.2.0"
spring-dependency-management = "1.1.4"
coroutines = "1.7.3"
retrofit = "2.9.0"
okhttp = "4.12.0"
jackson = "2.16.0"
junit-jupiter = "5.10.1"
mockito = "5.8.0"
testcontainers = "1.19.3"

[libraries]
# Kotlin 标准库
kotlin-stdlib = { module = "org.jetbrains.kotlin:kotlin-stdlib", version.ref = "kotlin" }
kotlin-reflect = { module = "org.jetbrains.kotlin:kotlin-reflect", version.ref = "kotlin" }

# 协程
coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
coroutines-reactor = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-reactor", version.ref = "coroutines" }

# Spring Boot Starters
spring-boot-starter-web = { module = "org.springframework.boot:spring-boot-starter-web", version.ref = "spring-boot" }
spring-boot-starter-data-jpa = { module = "org.springframework.boot:spring-boot-starter-data-jpa", version.ref = "spring-boot" }
spring-boot-starter-test = { module = "org.springframework.boot:spring-boot-starter-test", version.ref = "spring-boot" }

# HTTP 客户端
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
retrofit-jackson = { module = "com.squareup.retrofit2:converter-jackson", version.ref = "retrofit" }
okhttp = { module = "com.squareup.okhttp3:okhttp", version.ref = "okhttp" }
okhttp-logging = { module = "com.squareup.okhttp3:logging-interceptor", version.ref = "okhttp" }

# Jackson（JSON 处理）
jackson-databind = { module = "com.fasterxml.jackson.core:jackson-databind", version.ref = "jackson" }
jackson-kotlin = { module = "com.fasterxml.jackson.module:jackson-module-kotlin", version.ref = "jackson" }

# 测试库
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit-jupiter" }
junit-jupiter-api = { module = "org.junit.jupiter:junit-jupiter-api", version.ref = "junit-jupiter" }
junit-jupiter-engine = { module = "org.junit.jupiter:junit-jupiter-engine", version.ref = "junit-jupiter" }
mockito-core = { module = "org.mockito:mockito-core", version.ref = "mockito" }
mockito-junit-jupiter = { module = "org.mockito:mockito-junit-jupiter", version.ref = "mockito" }
assertj-core = { module = "org.assertj:assertj-core", version = "3.24.2" }

# Testcontainers
testcontainers-bom = { module = "org.testcontainers:testcontainers-bom", version.ref = "testcontainers" }
testcontainers-postgresql = { module = "org.testcontainers:postgresql", version.ref = "testcontainers" }

[bundles]
# 依赖组合，可以一次引用多个依赖
kotlin-essentials = ["kotlin-stdlib", "kotlin-reflect", "coroutines-core"]

spring-web = [
    "spring-boot-starter-web",
    "spring-boot-starter-data-jpa",
    "jackson-databind",
    "jackson-kotlin"
]

http-client = ["retrofit", "retrofit-jackson", "okhttp", "okhttp-logging"]

testing = [
    "junit-jupiter",
    "mockito-core",
    "mockito-junit-jupiter",
    "assertj-core",
    "spring-boot-starter-test"
]

[plugins]
# 插件声明
kotlin-jvm = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
kotlin-spring = { id = "org.jetbrains.kotlin.plugin.spring", version.ref = "kotlin" }
spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
spring-dependency-management = { id = "io.spring.dependency-management", version.ref = "spring-dependency-management" }
```

## 5.2 在构建脚本中使用版本目录

```kotlin
// build.gradle.kts - 使用版本目录

plugins {
    alias(libs.plugins.kotlin.jvm)
    alias(libs.plugins.kotlin.spring)
    alias(libs.plugins.spring.boot)
    alias(libs.plugins.spring.dependency.management)
}

dependencies {
    // 使用 bundle（组合引用）
    implementation(libs.bundles.kotlin.essentials)
    implementation(libs.bundles.spring.web)
    implementation(libs.bundles.http.client)

    // 使用单个库
    implementation(libs.coroutines.reactor)

    // 测试依赖
    testImplementation(libs.bundles.testing)
    testImplementation(libs.testcontainers.postgresql)
}
```

## 5.3 多版本目录

```kotlin
// settings.gradle.kts - 声明多个版本目录
dependencyResolutionManagement {
    versionCatalogs {
        // 主版本目录（默认名称为 libs，对应 libs.versions.toml）
        create("libs") {
            from(files("gradle/libs.versions.toml"))
        }
    }
}

// 也可以直接通过 settings 声明
@Suppress("UnstableApiUsage")
dependencyResolutionManagement {
    versionCatalogs {
        create("springLibs") {
            library("spring-boot-starter-web", "org.springframework.boot:spring-boot-starter-web:3.2.0")
            library("spring-boot-starter-data-jpa", "org.springframework.boot:spring-boot-starter-data-jpa:3.2.0")
            library("spring-boot-starter-test", "org.springframework.boot:spring-boot-starter-test:3.2.0")
            bundle("web", listOf("spring-boot-starter-web", "spring-boot-starter-data-jpa"))
        }
    }
}
// 使用时：implementation(springLibs.bundles.web)
```

# 6. 多项目构建最佳实践

## 6.1 项目职责分离

```
my-multimodule-app/
├── build.gradle.kts              ← 根项目：公共配置 + 全局版本
├── settings.gradle.kts           ← 项目结构定义
├── gradle/
│   ├── libs.versions.toml        ← 集中版本管理
│   └── wrapper/
├── build-logic/                  ← 构建逻辑项目（复合构建）
│   ├── settings.gradle.kts
│   └── src/main/kotlin/
│       ├── java-convention.gradle.kts
│       ├── kotlin-convention.gradle.kts
│       └── spring-convention.gradle.kts
├── modules/
│   ├── common/                   ← 公共工具类模块
│   │   ├── build.gradle.kts
│   │   └── src/main/java/
│   ├── domain/                   ← 领域模型模块
│   │   ├── build.gradle.kts
│   │   └── src/main/java/
│   ├── application/              ← 应用服务层模块
│   │   ├── build.gradle.kts
│   │   └── src/main/java/
│   └── infrastructure/           ← 基础设施模块
│       ├── build.gradle.kts
│       └── src/main/java/
├── apps/
│   ├── web/                      ← Web 应用入口
│   │   ├── build.gradle.kts
│   │   └── src/main/java/
│   └── cli/                      ← 命令行工具入口
│       ├── build.gradle.kts
│       └── src/main/java/
└── testing/
    └── integration-test/         ← 集成测试模块
        ├── build.gradle.kts
        └── src/main/java/
```

## 6.2 根项目公共配置

```kotlin
// 根项目 build.gradle.kts - 仅包含公共配置

// 不对根项目本身应用任何构建插件（它可能不产生输出）
// 如果根项目有自己的代码，可以应用 java 等插件

allprojects {
    // 对所有项目（包括根项目）应用公共逻辑
    group = "com.example"
}

subprojects {
    // 对子项目应用公共逻辑
    apply(plugin = "java")

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

    tasks.withType<Test> {
        useJUnitPlatform()
    }

    tasks.withType<JavaCompile> {
        options.encoding = "UTF-8"
    }
}
```

## 6.3 子项目间依赖

```kotlin
// modules/domain/build.gradle.kts
plugins {
    id("java-convention")
}
// domain 模块不依赖任何其他模块（最内层）

// modules/application/build.gradle.kts
plugins {
    id("java-convention")
}

dependencies {
    implementation(project(":modules:domain"))
    implementation(project(":modules:common"))
}

// modules/infrastructure/build.gradle.kts
plugins {
    id("java-convention")
}

dependencies {
    implementation(project(":modules:domain"))
    implementation(project(":modules:application"))
}

// apps/web/build.gradle.kts
plugins {
    id("spring-convention")
}

dependencies {
    implementation(project(":modules:application"))
    implementation(project(":modules:infrastructure"))
    runtimeOnly(project(":modules:common"))  // 运行时才需要
}

// 依赖方向（单向，避免循环）：
// apps → modules/infrastructure → modules/application → modules/domain
//                                                        ↘ modules/common
```

## 6.4 跨项目 task 依赖

```kotlin
// 根项目 build.gradle.kts - 跨项目任务协调

// 确保所有模块的测试在根项目的 check 之前执行
tasks.named("check") {
    dependsOn(
        gradle.includedBuilds.map { it.task(":check") }
    )
}

// 全局注册的便利任务
tasks.register("runAllTests") {
    group = "verification"
    description = "运行所有模块的单元测试"
    dependsOn(subprojects.map { it.tasks.named("test") })
}

tasks.register("buildAll") {
    group = "build"
    description = "构建所有模块"
    dependsOn(subprojects.map { it.tasks.named("build") })
}
```
