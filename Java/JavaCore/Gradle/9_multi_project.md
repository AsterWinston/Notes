
# 1. 多项目构建基础

## 1.1 典型多项目目录结构

```
my-application/
├── settings.gradle(.kts)           ← 声明项目树
├── build.gradle(.kts)              ← 根项目构建脚本
├── gradle/
│   ├── wrapper/
│   ├── libs.versions.toml          ← 版本目录
│   └── plugins/
│       └── conventions.gradle      ← 约定插件
├── buildSrc/                       ← 构建逻辑
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│       └── my-conventions.gradle.kts
├── app/                            ← 应用程序模块
│   ├── build.gradle(.kts)
│   └── src/main/java/...
├── core/                           ← 核心库模块
│   ├── build.gradle(.kts)
│   └── src/main/java/...
├── data/                           ← 数据层模块
│   ├── build.gradle(.kts)
│   └── src/main/java/...
├── api/                            ← API 模块
│   ├── build.gradle(.kts)
│   └── src/main/java/...
└── integration-test/               ← 集成测试模块
    ├── build.gradle(.kts)
    └── src/test/java/...
```

## 1.2 `settings.gradle` — 项目注册

```groovy
// settings.gradle
rootProject.name = 'my-application'

// 注册子项目
include 'app'
include 'core'
include 'data'
include 'api'
include 'integration-test'

// 自定义构建文件名
include 'legacy-module'
project(':legacy-module').buildFileName = 'legacy.gradle'

// 深层嵌套项目
include 'services:user-service'
include 'services:order-service'
include 'services:payment-service'
// 生成 :services:user-service 等路径

// 扁平布局（非标准目录结构）
include 'lib-a'
project(':lib-a').projectDir = file('libraries/lib-a')

// 条件包含
if (file('experimental').exists()) {
    include 'experimental'
}

// 程序化包含
file('modules').eachDir { dir ->
    if (file("${dir}/build.gradle").exists()) {
        include "modules:${dir.name}"
    }
}
```

```kotlin
// settings.gradle.kts
rootProject.name = "my-application"

include("app", "core", "data", "api", "integration-test")

include(":services:user-service")
include(":services:order-service")
include(":services:payment-service")

// 自定义构建文件
project(":legacy").buildFileName = "legacy.gradle.kts"
```

## 1.3 项目路径、名称与目录的关系

| 概念 | 示例 | 说明 |
|------|------|------|
| **Project Path** | `:services:user-service` | 以 `:` 分隔的路径标识 |
| **Project Name** | `user-service` | 最后一个路径段，不包含 `:` |
| **Project Directory** | `services/user-service/` | 相对于根项目的文件系统目录 |
| **Root Project Path** | `:` | 根项目的路径始终为 `:` |

```groovy
// 获取项目引用
def appProject = project(':app')
def serviceProject = project(':services:user-service')
def rootProject = project(':')

// 常用查找方式
def projectByName = project.findProject('core')           // 按名称（不可靠，可能有重名）
def projectByPath = project.findProject(':data')          // 按路径（可靠）
def allSubprojects = project.subprojects                 // 所有直接子项目
def allProjects = project.allprojects                    // 所有项目（含根项目）
```

# 2. `allprojects` / `subprojects` 配置

## 2.1 统一配置模式

```groovy
// 根项目 build.gradle

// 所有项目（含根项目）的公共配置
allprojects {
    group = 'com.mycompany'
    version = '1.0.0-SNAPSHOT'

    repositories {
        mavenCentral()
    }

    // 插件管理（注意：plugins {} 不能放在 allprojects 中，需用 apply plugin）
}

// 仅子项目的公共配置
subprojects {
    apply plugin: 'java'

    java {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    tasks.withType(JavaCompile).configureEach {
        options.encoding = 'UTF-8'
    }

    dependencies {
        testImplementation 'org.junit.jupiter:junit-jupiter:5.10.0'
    }
}
```

**注意事项**：
- `plugins {}` 块**不能**放在 `allprojects`/`subprojects` 闭包内
- 推荐使用 `apply plugin:` 代替（在 `allprojects`/`subprojects` 中）
- 或者使用 `pluginManagement` 集中管理，各模块各自 `plugins {}`

## 2.2 配置注入 vs 配置传播

```groovy
// 反模式：在 allprojects 中操作所有项目的任务（立即配置）
allprojects {
    tasks.withType(JavaCompile) {
        // 这个块在配置阶段立即执行，即使某些子项目没有 Java 插件
        options.encoding = 'UTF-8'
    }
}

// 推荐：使用 configureEach 延迟配置
subprojects {
    plugins.withType(JavaPlugin) {
        // 仅在 Java 插件被应用后才配置
        tasks.withType(JavaCompile).configureEach {
            options.encoding = 'UTF-8'
        }
    }
}
```

## 2.3 Kotlin DSL 中的跨项目配置

```kotlin
// build.gradle.kts (根项目)
subprojects {
    apply(plugin = "java")

    configure<JavaPluginExtension> {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    tasks.withType<JavaCompile>().configureEach {
        options.encoding = "UTF-8"
        options.compilerArgs.add("-Xlint:unchecked")
    }
}
```

# 3. 跨项目任务依赖

## 3.1 建立项目间任务依赖

```groovy
// 根项目 build.gradle
tasks.register('buildAll') {
    dependsOn subprojects.collect { it.tasks.named('build') }
}

// app 模块依赖于 core 模块的 jar
project(':app') {
    tasks.named('jar').configure {
        dependsOn project(':core').tasks.named('jar')
    }
}

// 确保集成测试在所有单元测试之后运行
tasks.register('fullTest') {
    dependsOn ':core:test', ':data:test', ':app:test'
    dependsOn ':integration-test:test'  // 集成测试最后
}

// mustRunAfter 跨项目排序
project(':integration-test').tasks.named('test').configure {
    mustRunAfter subprojects.collect { it.tasks.named('test') }
}
```

## 3.2 使用 Task Provider 建立依赖

```groovy
// 推荐使用 Provider 而非直接引用 Task 对象
def coreJar = project(':core').tasks.named('jar')

tasks.register('assembleWithCore') {
    dependsOn coreJar
    doLast {
        println "Core JAR: ${coreJar.get().archiveFile.get().asFile}"
    }
}
```

## 3.3 获取所有子项目的特定任务

```groovy
// 获取所有具有 'test' 任务的子项目任务集合
def allTestTasks = subprojects.collect { sub ->
    sub.tasks.matching { it.name == 'test' }
}.flatten()

tasks.register('runAllTests') {
    dependsOn allTestTasks
}
```

# 4. 项目依赖（`implementation project(':lib')`）

## 4.1 基本项目依赖声明

```groovy
// app/build.gradle
dependencies {
    implementation project(':core')
    implementation project(':data')
    implementation project(':api')

    // 外部依赖
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

## 4.2 项目依赖的配置传递

```
app/build.gradle:
  dependencies { implementation project(':core') }

core/build.gradle (java-library):
  dependencies { api 'com.google.guava:guava' }

结果: app 的 compileClasspath 包含 guava
       （因为 core 使用 api，所以 guava 对消费者可见）
```

```groovy
// 更精确的项目依赖
dependencies {
    // 仅编译时可见（compile-only project dependency）
    compileOnly project(':annotations')

    // 运行时可见（分离编译和运行时依赖）
    runtimeOnly project(':native-libs')

    // 测试时依赖
    testImplementation project(':test-fixtures')

    // 包含 capabilities
    implementation(project(':core')) {
        capabilities {
            requireCapability('com.mycompany:core-database-support')
        }
    }
}
```

## 4.3 循环依赖检测

Gradle 自动检测项目间的循环依赖，并在配置阶段抛出 `CircularReferenceException`：

```groovy
// 这会导致构建失败
// :core 依赖 :data，:data 依赖 :core
```

解决方法：提取公共接口到独立模块或使用依赖反转。

```groovy
// 解决方案：提取 :common 或 :api 模块
include ':api'       // 接口定义
include ':core'      // 依赖 :api
include ':data'      // 依赖 :api
```

# 5. 项目评估顺序

## 5.1 默认评估顺序

Gradle 按**项目名称字母顺序**评估项目（在满足依赖约束的前提下）：

```
根项目 (:) → 直接子项目按字母顺序
每层级的项目按字母顺序评估

:app, :core, :data, :integration-test
```

## 5.2 控制评估顺序

```groovy
// 方式 1：显式依赖
project(':app') {
    evaluationDependsOn(':core')  // 确保 core 在 app 之前评估
}

// 方式 2：使用 afterEvaluate
project(':core').afterEvaluate {
    println "core 评估完成"
}

// 方式 3：通过子项目声明
project(':data') {
    evaluationDependsOnChildren()  // 等待所有子项目评估完成
}

// 方式 4：使用 evaluationDependsOn 确保交叉项目评估顺序
// settings.gradle
include ':app'
include ':core'
// core/build.gradle: evaluationDependsOn(':data')
// data/build.gradle: evaluationDependsOn(':core') ← 循环依赖错误！
```

## 5.3 `configuration-on-demand`（按需配置）

```properties
# gradle.properties
org.gradle.configureondemand=true
```

启用后，Gradle 仅配置参与构建的项目。例如执行 `gradle :app:build` 时，如果 `:core` 不是 `:app` 的依赖项，`:core` 不会被配置。

**限制**：
- 不兼容某些访问 `allprojects` 的模式
- 某些跨项目配置可能不生效
- Gradle 8.0+ 计划成为默认行为

```groovy
// 在 settings.gradle 中声明
// settings.gradle
gradle.settingsEvaluated { settings ->
    if (settings.gradle.startParameter.configureOnDemand) {
        println '按需配置已启用'
    }
}
```

# 6. 构建逻辑共享

## 6.1 `buildSrc` 目录

```
buildSrc/
├── build.gradle.kts
│   plugins { `kotlin-dsl` }
│   repositories { mavenCentral() }
├── settings.gradle.kts (可选)
│   rootProject.name = "build-conventions"
│   dependencyResolutionManagement {
│       versionCatalogs { create("libs") { from(files("../gradle/libs.versions.toml")) } }
│   }
└── src/main/kotlin/
    ├── my-java-conventions.gradle.kts    → id("my-java-conventions")
    └── my-spring-conventions.gradle.kts  → id("my-spring-conventions")
```

**`buildSrc` 的优缺点**：

| 优点 | 缺点 |
|------|------|
| 自动加入 classpath，IDE 原生支持 | 任何变更都会使整个构建失效 |
| 代码可在插件中直接引用 | 不支持复合构建 |
| 简单直接 | 大型项目编译 `buildSrc` 可能耗时 |

## 6.2 约定插件（Convention Plugins）

```kotlin
// buildSrc/src/main/kotlin/my-java-conventions.gradle.kts
plugins {
    java
    checkstyle
    jacoco
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

tasks.withType<JavaCompile>().configureEach {
    options.encoding = "UTF-8"
    options.compilerArgs.add("-parameters")
}

tasks.withType<Test>().configureEach {
    useJUnitPlatform()
    maxParallelForks = Runtime.getRuntime().availableProcessors()
}

dependencies {
    "testImplementation"(platform("org.junit:junit-bom:5.10.0"))
    "testImplementation"("org.junit.jupiter:junit-jupiter")
}
```

使用约定插件：

```kotlin
// core/build.gradle.kts
plugins {
    id("my-java-conventions")  // 应用约定
}

dependencies {
    implementation("com.google.guava:guava:32.1.3-jre")
}
```

## 6.3 独立约定插件项目（Included Builds）

```
gradle/
├── plugins/
│   ├── settings.gradle.kts
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│       └── my-conventions.gradle.kts
settings.gradle.kts
build.gradle.kts
```

```kotlin
// settings.gradle.kts (根项目)
pluginManagement {
    includeBuild("gradle/plugins")  // 包含独立构建
}

// gradle/plugins/settings.gradle.kts
rootProject.name = "convention-plugins"

// gradle/plugins/build.gradle.kts
plugins {
    `kotlin-dsl`
}
```

# 7. 复合构建（Composite Builds）

## 7.1 `includeBuild` 配置

```groovy
// settings.gradle
includeBuild '../shared-library' {    // 相对路径
    dependencySubstitution {
        substitute module('com.mycompany:shared-library') using project(':')
    }
}

includeBuild 'gradle/plugins'         // 包含构建逻辑
```

## 7.2 依赖替换（Dependency Substitution）

```groovy
// 当 shared-library 是复合构建的一部分时，
// dependencySubstitution 在配置阶段自动发生
dependencies {
    // 这个外部依赖声明会被替换为复合构建中的本地项目
    implementation 'com.mycompany:shared-library:1.0.0'

    // 明确使用 includeBuild 中项目的产物
    // Gradle 自动将本地产出替换外部依赖
}

// 在 included build 中参与构建
// gradle :app:build --include-build ../shared-library
```

## 7.3 复合构建与子项目的区别

| 特性 | 子项目（Subproject） | 复合构建（includeBuild） |
|------|---------------------|------------------------|
| 共享配置 | 可通过 allprojects/subprojects | 独立配置 |
| 依赖可见性 | 可通过 api/implementation | 消费发布的产物 |
| 生命周期 | 同一次 Gradle 构建 | 独立的 Gradle 调用 |
| 适用场景 | 紧密耦合的模块 | 独立开发、独立版本的库 |
| 构建速度 | 一起编译 | 可分别缓存 |

## 7.4 多仓库复合构建

```groovy
// settings.gradle
includeBuild('../database-lib')
includeBuild('../logging-lib')
includeBuild('../common-utils')

// 每个复合构建有自己的 settings.gradle 和 build.gradle
// 它们在独立进程中运行，产物可被主构建消费
```

# 8. 多项目构建阶段

## 8.1 多项目下的三阶段流程

```
┌─────────────────────────────────────────────────┐
│ 初始化阶段 (Initialization)                       │
│ ├── 读取 settings.gradle                         │
│ ├── 解析 include / includeBuild                  │
│ ├── 构建项目树                                    │
│ └── 为每个项目创建 Project 对象                    │
└─────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│ 配置阶段 (Configuration)                          │
│ ├── 按评估顺序执行每个项目的 build.gradle           │
│ ├── 根项目 → 子项目（字母或显式顺序）               │
│ ├── afterEvaluate 回调触发                        │
│ └── 任务图尚未构建                                 │
└─────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│ 执行阶段 (Execution)                              │
│ ├── 构建任务执行图（DAG）                          │
│ ├── 确定任务执行顺序                               │
│ ├── 并行调度任务                                   │
│ └── beforeTask/afterTask 触发                     │
└─────────────────────────────────────────────────┘
```

## 8.2 多项目中的生命周期监听

```groovy
// settings.gradle
gradle.settingsEvaluated { settings ->
    println "Settings 评估完成，共 ${settings.rootProject.allprojects.size()} 个项目"
}

// build.gradle (根项目)
gradle.projectsLoaded { gradle ->
    println "所有项目已加载: ${gradle.rootProject.subprojects*.name}"
}

gradle.beforeProject { project ->
    println "开始配置: ${project.path}"
}

gradle.afterProject { project ->
    println "完成配置: ${project.path}"
}

gradle.projectsEvaluated { gradle ->
    println "所有项目配置完成"
}

gradle.taskGraph.whenReady { graph ->
    println "任务图就绪，共 ${graph.allTasks.size()} 个任务"
}

gradle.buildFinished { result ->
    println "构建完成，耗时: ${result.failure ? '失败' : '成功'}"
}
```

# 9. 多项目的并行执行

## 9.1 配置并行执行

```properties
# gradle.properties
org.gradle.parallel=true
org.gradle.configureondemand=true
org.gradle.caching=true
org.gradle.workers.max=8
```

```bash
# 命令行启用
gradle build --parallel --max-workers=8
```

## 9.2 并行执行的条件

并行执行要求：
1. 任务在不同项目中，且无跨项目依赖
2. 任务没有共享的可变状态
3. Worker 数量足够

```groovy
// 可以并行执行：
// :core:compileJava, :data:compileJava, :api:compileJava
//（这三个任务之间没有依赖关系，属于不同项目）

// 不能并行执行：
// :data:compileJava 和 :data:jar
//（jar 依赖 compileJava，属于同一项目内）
```

## 9.3 跨项目并行任务配置

```groovy
// 在 gradle.properties 中
org.gradle.parallel=true
org.gradle.workers.max=4

// 在代码中控制并行
tasks.whenTaskAdded { task ->
    // 某些任务不适合并行
    if (task.name == 'integrationTest') {
        task.usesService(project.gradle.sharedServices
            .registerIfAbsent('integration', BuildService) {}.get())
    }
}
```

# 10. 多项目版本管理

## 10.1 集中式版本声明

```groovy
// gradle.properties
springBootVersion=3.2.0
kotlinVersion=1.9.20
guavaVersion=32.1.3-jre

// build.gradle (根项目)
subprojects {
    dependencies {
        implementation "org.springframework.boot:spring-boot-starter:${springBootVersion}"
    }
}
```

## 10.2 使用 Version Catalog 集中管理

```toml
# gradle/libs.versions.toml
[versions]
spring-boot = "3.2.0"
spring-dependency-management = "1.1.4"
kotlin = "1.9.20"
junit-jupiter = "5.10.0"
guava = "32.1.3-jre"
jackson = "2.16.0"
logback = "1.4.11"

[libraries]
spring-boot-starter = { module = "org.springframework.boot:spring-boot-starter", version.ref = "spring-boot" }
spring-boot-starter-web = { module = "org.springframework.boot:spring-boot-starter-web", version.ref = "spring-boot" }
spring-boot-starter-test = { module = "org.springframework.boot:spring-boot-starter-test", version.ref = "spring-boot" }
guava = { module = "com.google.guava:guava", version.ref = "guava" }
jackson-databind = { module = "com.fasterxml.jackson.core:jackson-databind", version.ref = "jackson" }
logback-classic = { module = "ch.qos.logback:logback-classic", version.ref = "logback" }
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit-jupiter" }

[bundles]
spring-web = ["spring-boot-starter-web", "spring-boot-starter-test"]
jackson = ["jackson-databind", "jackson-dataformat-xml"]

[plugins]
spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
spring-dependency-management = { id = "io.spring.dependency-management", version.ref = "spring-dependency-management" }
kotlin-jvm = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
```

使用：

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.spring.boot)
}

dependencies {
    implementation(libs.spring.boot.starter.web)
    implementation(libs.guava)
    runtimeOnly(libs.logback.classic)
    testImplementation(libs.junit.jupiter)
    testImplementation(libs.bundles.spring.web)
}
```

# 11. 多项目发布

## 11.1 统一发布策略

```groovy
// build.gradle (根项目)
subprojects {
    apply plugin: 'maven-publish'

    // 跳过没有发布物的项目
    plugins.withId('java-library') {
        publishing {
            publications {
                maven(MavenPublication) {
                    from components.java
                    pom {
                        name = project.name
                        description = "Module: ${project.name}"
                        url = 'https://github.com/myorg/myproject'
                        licenses {
                            license {
                                name = 'Apache-2.0'
                                url = 'https://www.apache.org/licenses/LICENSE-2.0'
                            }
                        }
                    }
                }
            }
        }
    }

    // 统一仓库配置
    publishing {
        repositories {
            maven {
                name = 'nexus'
                url = version.endsWith('-SNAPSHOT')
                    ? 'https://nexus.mycompany.com/repository/snapshots/'
                    : 'https://nexus.mycompany.com/repository/releases/'
                credentials {
                    username = System.getenv('NEXUS_USER')
                    password = System.getenv('NEXUS_PASS')
                }
            }
        }
    }
}
```

## 11.2 选择性发布

```groovy
// 仅发布指定模块
gradle.taskGraph.whenReady { graph ->
    subprojects.each { sub ->
        def publishTask = sub.tasks.findByName('publishMavenPublicationToNexusRepository')
        if (publishTask) {
            publishTask.onlyIf {
                // 只发布有变更的模块
                def changedFiles = 'git diff --name-only HEAD~1'.execute().text
                changedFiles.contains(sub.name)
            }
        }
    }
}
```

# 12. Build Scan 在多项目中的应用

## 12.1 启用 Build Scan

```groovy
// settings.gradle
plugins {
    id 'com.gradle.enterprise' version '3.15'
}

gradleEnterprise {
    buildScan {
        termsOfServiceUrl = 'https://gradle.com/terms-of-service'
        termsOfServiceAgree = 'yes'

        // 标签
        tag 'CI'  // 仅在 CI 环境打标签

        // 自定义值
        value 'Git Branch', 'git rev-parse --abbrev-ref HEAD'.execute().text.trim()
        value 'Git Commit', 'git rev-parse --short HEAD'.execute().text.trim()

        // 仅失败时发布
        publishOnFailure()
    }
}
```

## 12.2 分析多项目构建性能

```bash
gradle build --scan
```

Build Scan 提供：
- 各项目配置时间分布
- 任务执行时间线（甘特图）
- 并行执行效率
- 缓存命中率
- 依赖解析耗时
- 测试执行分布

# 13. 大型多项目最佳实践

## 13.1 项目结构原则

```
✓ 推荐
├── :app (入口)
├── :feature-a:api, :feature-a:impl (API/实现分离)
├── :common (共享基础)
└── :platform (BOM)

✗ 避免
├── :app
├── :module1 (无意义命名)
├── :module2
└── 深度 > 3 级
```

## 13.2 配置避免（Configuration Avoidance）

```groovy
// 反模式
subprojects {
    tasks.withType(Test) {
        // 立即配置所有测试任务
        maxHeapSize = '1g'
    }
}

// 推荐
subprojects {
    tasks.withType(Test).configureEach {
        maxHeapSize = '1g'
    }
}
```

## 13.3 构建缓存策略

```groovy
// 跨平台缓存
buildCache {
    local {
        enabled = true
        directory = file("${rootDir}/.gradle/build-cache")
        removeUnusedEntriesAfterDays = 30
    }
    remote(HttpBuildCache) {
        url = 'https://buildcache.mycompany.com/cache/'
        push = System.getenv('CI') != null
        credentials {
            username = 'ci'
            password = System.getenv('BUILD_CACHE_PASSWORD')
        }
    }
}
```

---

**总结**：多项目构建是 Gradle 的核心优势之一。关键要点：（1）使用 `settings.gradle` 声明项目树；（2）优先使用约定插件（convention plugins）而非 `subprojects` 共享构建逻辑；（3）利用 `java-library` 的 `api`/`implementation` 控制依赖可见性；（4）结合 Version Catalog 统一版本管理；（5）启用并行执行和配置缓存以提升大规模项目性能；（6）对于独立开发的库，使用复合构建（`includeBuild`）而非子项目。
