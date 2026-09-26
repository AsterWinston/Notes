
# 1. 依赖仓库

## 1.1 仓库类型概述

Gradle 支持多种类型的仓库，用于声明从哪里下载依赖和插件。

| 仓库类型 | 格式 | 是否支持元数据 | 典型用途 |
|----------|------|---------------|----------|
| Maven 仓库 | `maven {}` | 支持（pom.xml） | 最常用的 JVM 仓库 |
| Ivy 仓库 | `ivy {}` | 支持（ivy.xml） | 遗留系统，Ant 项目 |
| 本地目录 | `flatDir {}` | 不支持 | 本地 JAR 文件（不推荐） |
| Google Maven | `google()` | 支持（pom.xml） | Android 库 |
| Gradle Plugin Portal | `gradlePluginPortal()` | 支持 | Gradle 插件 |

## 1.2 Maven 仓库配置

**标准 Maven 仓库：**
```kotlin
// build.gradle.kts
repositories {
    // 内置快捷方式
    mavenCentral()   // https://repo1.maven.org/maven2/

    google()         // https://dl.google.com/dl/android/maven2/

    // 自定义 Maven 仓库
    maven {
        name = "SonatypeSnapshot"
        url = uri("https://oss.sonatype.org/content/repositories/snapshots/")
        // 仅下载快照版本
        mavenContent {
            snapshotsOnly()
        }
    }

    maven {
        name = "SpringMilestone"
        url = uri("https://repo.spring.io/milestone/")
        // 仅下载里程碑版本
        mavenContent {
            releasesOnly()
        }
    }

    maven {
        name = "JitPack"
        url = uri("https://jitpack.io")
    }
}
```

**带认证的 Maven 仓库：**
```kotlin
repositories {
    maven {
        name = "PrivateReleases"
        url = uri("https://nexus.mycompany.com/repository/maven-releases/")

        // 方式1：直接硬编码（不推荐，不安全）
        // credentials {
        //     username = "admin"
        //     password = "secret123"
        // }

        // 方式2：从属性文件读取
        credentials {
            username = providers.gradleProperty("nexusUser").orElse("").get()
            password = providers.gradleProperty("nexusPassword").orElse("").get()
        }

        // 方式3：从环境变量读取
        credentials {
            username = System.getenv("NEXUS_USER") ?: "default_user"
            password = System.getenv("NEXUS_PASS") ?: "default_pass"
        }

        // 仅允许特定 group 的依赖从此仓库下载
        mavenContent {
            includeGroup("com.mycompany")
            includeGroupByRegex("com\\.mycompany\\..*")
            includeModule("org.example", "special-lib")
        }
    }

    maven {
        name = "PrivateSnapshots"
        url = uri("https://nexus.mycompany.com/repository/maven-snapshots/")
        credentials {
            username = System.getenv("NEXUS_USER")
            password = System.getenv("NEXUS_PASS")
        }
        // 仅允许下载快照版本
        mavenContent {
            snapshotsOnly()
        }
    }
}
```

**仓库的搜索顺序：**
```kotlin
repositories {
    mavenLocal()         // 1. 本地 Maven 缓存 (~/.m2/repository)
    maven {              // 2. 公司私有仓库（优先）
        url = uri("https://nexus.mycompany.com/repository/maven-public/")
    }
    mavenCentral()       // 3. Maven Central（fallback）
    google()             // 4. Google Maven（Android 依赖）
    maven {              // 5. JitPack（最后的 fallback）
        url = uri("https://jitpack.io")
    }
}
// Gradle 按声明顺序搜索仓库，当在某个仓库找到依赖后停止搜索
```

## 1.3 Ivy 仓库

Ivy 是 Apache Ivy 的仓库格式，使用 `ivy.xml` 描述模块元数据：

```kotlin
repositories {
    ivy {
        url = uri("https://repo.mycompany.com/ivy-repo/")
        // 自定义 Ivy 模式布局
        patternLayout {
            artifact("[organisation]/[module]/[revision]/[type]s/[artifact](.[ext])")
            ivy("[organisation]/[module]/[revision]/ivy-[revision].xml")
        }

        // 或使用预定义的布局
        // layout("maven")  // Maven 布局：groupId/artifactId/version/
        // layout("gradle") // Gradle 布局：group/artifact/version/

        metadataSources {
            ivyDescriptor()  // 首先查找 ivy.xml
            artifact()       // 然后通过 artifact 推断
        }
    }
}
```

## 1.4 flatDir 仓库（本地 JAR）

```kotlin
// 不推荐用于生产环境，仅适用于以下场景：
// 1. 临时使用尚未发布的本地 JAR
// 2. 遗留系统无法迁移到 Maven 仓库
// 3. 系统提供的 JAR（如特定 JDK 扩展）
repositories {
    flatDir {
        dirs("libs", "/usr/local/shared-jars")
        // 不需要元数据，直接匹配文件名
    }
}
// 注意：通过 flatDir 解析的依赖没有传递性依赖信息
```

## 1.5 仓库过滤与内容策略

```kotlin
repositories {
    mavenCentral {
        // 全局内容过滤：此仓库仅用于特定组织
        content {
            includeGroup("com.google")
            includeGroupByRegex("com\\.google\\..*")
            includeModule("org.slf4j", "slf4j-api")
            // excludeGroup("com.example")  // 排除特定组
        }
    }

    maven {
        url = uri("https://repo.mycompany.com/")
        content {
            // 此仓库仅提供公司内部的依赖
            includeGroup("com.mycompany")
        }
    }

    // 排除仓库（不从此仓库获取任何依赖）
    // exclusiveContent {
    //     forRepository { mavenCentral() }
    //     filter { includeGroup("com.unwanted") }
    // }
}
```

# 2. 依赖配置（Configurations）

## 2.1 配置概念

Gradle 中的 Configuration 是一个依赖的集合，代表项目的某个依赖范围。每个 Configuration 都有特定的用途和可见性规则。

```
Configuration = 一组依赖声明 + 可见性 + 传递性规则 + 可消耗性规则
```

## 2.2 Java 插件提供的标准配置

Java Library Plugin 提供了一组精细的配置来控制依赖的可见性：

```kotlin
plugins {
    `java-library`  // 提供 api 配置（java 插件不提供）
}

dependencies {
    // === 编译时+运行时 ===
    // api：暴露给消费者（下游项目可见）
    api("com.google.guava:guava:32.1.3-jre")
    // implementation：不暴露给消费者（仅本项目内部可见）
    implementation("org.slf4j:slf4j-api:2.0.9")

    // === 仅编译时 ===
    // compileOnly：仅编译时可见，不包含在运行时
    compileOnly("org.projectlombok:lombok:1.18.30")
    // compileOnlyApi：编译时可见，且暴露给下游项目编译时
    compileOnlyApi("javax.servlet:javax.servlet-api:4.0.1")

    // === 仅运行时 ===
    // runtimeOnly：编译时不可见，运行时可用
    runtimeOnly("ch.qos.logback:logback-classic:1.4.11")

    // === 注解处理器 ===
    annotationProcessor("org.projectlombok:lombok:1.18.30")

    // === 测试相关 ===
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.1")
    testCompileOnly("org.projectlombok:lombok:1.18.30")
    testAnnotationProcessor("org.projectlombok:lombok:1.18.30")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")

    // === 测试 fixture（共享测试工具）===
    testFixturesApi("org.mockito:mockito-core:5.8.0")
    testFixturesImplementation("org.assertj:assertj-core:3.24.2")
}
```

## 2.3 api vs implementation 的可见性差异

```
项目 A（库生产者）:
  src/main/java/
    └── MyLib.java  ← 使用了 Guava 的 ImmutableList
        public class MyLib {
            public ImmutableList<String> getList() { ... }
            //                                        ↑ public API 暴露了 Guava 类型
        }

项目 B（库消费者）:
  src/main/java/
    └── App.java
        import com.example.MyLib;
        // B 能否看到 Guava？取决于 A 如何声明 Guava 依赖
```

```kotlin
// 场景1：A 使用 api() 声明 Guava
// 项目 A 的 build.gradle.kts
dependencies {
    api("com.google.guava:guava:32.1.3-jre")  // API 暴露
}
// 结果：项目 B 的编译 classpath 自动包含 Guava ← ImmutableList 可用
```

```kotlin
// 场景2：A 使用 implementation() 声明 Guava
// 项目 A 的 build.gradle.kts
dependencies {
    implementation("com.google.guava:guava:32.1.3-jre")  // 内部使用
}
// 结果：项目 B 的编译 classpath 不包含 Guava ← ImmutableList 编译时报错
// 如果 B 确实需要 Guava，B 必须自己声明 Guava 依赖
```

**选择 api vs implementation 的决策矩阵：**

| 场景 | 使用 | 原因 |
|------|------|------|
| 库的 public API 暴露了依赖的类型 | `api` | 消费者需要编译时访问该依赖 |
| 依赖类型出现在 public 方法的签名中 | `api` | 同上 |
| 依赖类型出现在父类/接口的继承层次中 | `api` | 同上 |
| 依赖仅在内部实现中使用 | `implementation` | 消费者不需要知道 |
| 依赖是你在源代码中使用的注解 | `implementation`（通常） | 除非注解需要被消费者看到 |
| 不确定时 | `implementation` | 更安全，以后可以改为 api |

## 2.4 自定义 Configuration

```kotlin
// 定义自定义配置
val myCustomConfig by configurations.creating {
    description = "自定义配置，用于特殊的依赖范围"
    isVisible = true            // 对其他项目可见
    isCanBeConsumed = false     // 此项目不产出此配置的产物
    isCanBeResolved = true      // 此项目可以解析此配置的依赖
}

// 继承已有配置
val integrationTestImplementation by configurations.creating {
    extendsFrom(configurations.testImplementation.get())
    // 继承 testImplementation 的所有依赖
}

// 依赖其他配置的结果
val reportClasspath by configurations.creating {
    // 组合 runtimeClasspath 和特殊工具
}

dependencies {
    myCustomConfig("com.example:special-lib:1.0")
    integrationTestImplementation("org.testcontainers:postgresql:1.19.3")
}
```

**Configuration 的关键属性：**

```kotlin
// 深入理解 Configuration 的属性
val conf by configurations.creating {
    // isCanBeResolved: 此配置是否可以解析（计算完整的依赖图）
    //   可解析 = Gradle 可以解析此配置并获取所有 JAR 文件
    //   如 classpath 配置是可解析的
    isCanBeResolved = true

    // isCanBeConsumed: 此配置是否可以被其他项目消费
    //   可消费 = 此配置作为项目产物的对外接口
    //   如 apiElements 和 runtimeElements 是可消费的
    isCanBeConsumed = false

    // isVisible: 此配置是否对其他项目可见
    isVisible = true

    // isTransitive: 是否自动包含传递性依赖
    isTransitive = true

    // description: 人类可读的描述
    description = "Custom configuration for demonstration"
}
```

**Java Library Plugin 的配置关系图：**
```
                         ┌──────────────┐
                         │  api (声明)   │
                         └──────┬───────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
              ┌─────▼─────┐          ┌──────▼──────┐
              │compileOnly│          │implementation│
              │    Api    │          └──────┬───────┘
              └───────────┘                 │
                                    ┌───────┴───────┐
                                    │               │
                              ┌─────▼──────┐  ┌─────▼──────┐
                              │compileOnly │  │runtimeOnly │
                              └────────────┘  └────────────┘
                                    │               │
                              ┌─────▼───────────────▼─────┐
                              │    compileClasspath       │
                              │    (可解析，供编译使用)     │
                              └───────────────────────────┘
                              ┌─────┬───────────────┬─────┐
                              │ api │implementation │rt...│
                              └─────┴───────────────┴─────┘
                              │     runtimeClasspath        │
                              │     (可解析，供运行时使用)    │
                              └─────────────────────────────┘

    对外暴露 (Consumable):
    apiElements        ← 编译时暴露给消费者
    runtimeElements    ← 运行时暴露给消费者
```

# 3. 依赖声明语法

## 3.1 依赖 notations 的多种形式

```kotlin
dependencies {
    // === 基本 Maven 坐标 ===
    implementation("com.google.guava:guava:32.1.3-jre")
    // 等价于: implementation(group = "com.google.guava", name = "guava", version = "32.1.3-jre")

    // === 带分类器和扩展名 ===
    implementation("com.example:my-lib:1.0:javadoc@jar")
    // group:name:version:classifier@extension

    // === 动态版本 ===
    implementation("com.google.guava:guava:32.1.+")         // 最新补丁版本
    implementation("com.google.guava:guava:latest.release")   // 最新发布版
    implementation("com.google.guava:guava:[31.0,32.0]")      // 版本范围（含）
    implementation("com.google.guava:guava:(31.0,32.0]")      // 版本范围（开区间）

    // === 项目依赖 ===
    implementation(project(":core"))
    implementation(project(":core", configuration = "myCustomConfig"))

    // === 文件依赖 ===
    implementation(files("libs/my-custom.jar", "libs/another.jar"))
    implementation(fileTree("libs") { include("*.jar") })

    // === Gradle 发行版依赖 ===
    implementation(gradleApi())                  // Gradle API
    implementation(gradleTestKit())              // Gradle TestKit

    // === 使用依赖提供者创建 ===
    implementation(project.dependencies.create("com.example:lib:1.0") {
        isTransitive = false
    })

    // === 本地 flatDir 依赖 ===
    implementation(":my-local-lib:1.0")
    // 以上需要在 repositories 中配置 flatDir
}
```

## 3.2 依赖的外部属性

```kotlin
dependencies {
    // 依赖对象的额外配置
    implementation("com.google.guava:guava:32.1.3-jre") {
        // 排除传递性依赖
        exclude(group = "com.google.errorprone", module = "error_prone_annotations")
        exclude(group = "org.checkerframework")

        // 设置为非传递（不包含其传递性依赖）
        // isTransitive = false

        // 指定 artifact
        // artifact {
        //     name = "guava"
        //     type = "jar"
        //     extension = "jar"
        //     classifier = "sources"
        // }

        // 因为特定原因声明此依赖
        because("我们需要 Guava 的缓存和集合 API")

        // 更改版本（与其他版本冲突时使用）
        version {
            strictly("32.1.3-jre")     // 严格版本（不允许升级/降级）
            require("32.1.0")          // 最低要求版本
            prefer("32.1.3-jre")       // 首选版本（允许协商）
            reject("30.0-jre")         // 拒绝特定版本
            rejectAll()                // 拒绝所有版本
        }

        // 目标配置（当依赖方有多个配置时）
        targetConfiguration = "runtime"
    }
}
```

# 4. 传递性依赖管理

## 4.1 依赖解析流程

当 Gradle 解析一个依赖时，它会：

```
1. implementation("com.example:A:1.0")
     │
     ├─► 2. 在配置的仓库中搜索 com.example:A:1.0
     │
     ├─► 3. 找到后，下载 POM/GMM（Gradle Module Metadata）
     │      └─► 提取 A 的依赖：B:1.0, C:2.0
     │
     ├─► 4. 递归解析 B 和 C 的依赖
     │      B:1.0 → D:3.0, E:1.0
     │      C:2.0 → F:1.0, B:1.5  ← 冲突！
     │
     ├─► 5. 冲突解决：B:1.0 vs B:1.5 → 选择 B:1.5（最高版本）
     │
     ├─► 6. D:3.0 → 无依赖
     │    E:1.0 → 无依赖
     │    F:1.0 → G:2.0
     │
     ├─► 7. G:2.0 → 无依赖
     │
     └─► 8. 最终依赖图：A, B:1.5, C:2.0, D:3.0, E:1.0, F:1.0, G:2.0
```

## 4.2 冲突解决策略

Gradle 默认使用**最新版本策略**来解决传递性依赖冲突。即当依赖图中存在同一模块的多个版本时，选择最新版本。

```kotlin
// build.gradle.kts
configurations.all {
    resolutionStrategy {
        // === 版本冲突策略 ===

        // failOnVersionConflict：当版本冲突时构建失败
        failOnVersionConflict()

        // 强制使用特定版本（最高优先级）
        force("com.google.guava:guava:32.1.3-jre")
        force("com.fasterxml.jackson.core:jackson-databind:2.16.0")

        // === 动态版本缓存 ===
        // 缓存动态版本（如 1.+）的解析结果，避免每次都查询
        cacheDynamicVersionsFor(10, TimeUnit.MINUTES)

        // 缓存 changing 模块（如 SNAPSHOT）的结果
        cacheChangingModulesFor(4, TimeUnit.HOURS)

        // === 组件选择规则 ===
        eachDependency {
            when (requested.group) {
                "commons-logging" -> {
                    // 用 jcl-over-slf4j 替换 commons-logging
                    useTarget("org.slf4j:jcl-over-slf4j:2.0.9")
                    because("统一使用 SLF4J")
                }

                // 锁定特定模块版本
                requested.module.toString() == "com.google.guava:guava" -> {
                    useVersion("32.1.3-jre")
                    because("项目标准版本")
                }
            }
        }
    }
}
```

## 4.3 全局排除依赖

```kotlin
// 方式1：在 Configuration 级别排除
configurations.all {
    exclude(group = "commons-logging", module = "commons-logging")
    exclude(group = "log4j", module = "log4j")
    exclude(group = "org.apache.tomcat")  // 仅指定 group 即可
}

// 方式2：在依赖声明时排除
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web:3.2.0") {
        exclude(group = "org.springframework.boot", module = "spring-boot-starter-tomcat")
        // 排除 Tomcat，改用 Jetty
    }
    implementation("org.springframework.boot:spring-boot-starter-jetty:3.2.0")
}

// 方式3：使用模块替换（更优雅）
configurations.all {
    resolutionStrategy {
        eachDependency {
            if (requested.group == "commons-logging") {
                useTarget("org.slf4j:jcl-over-slf4j:2.0.9")
                because("桥接到 SLF4J")
            }
        }
    }
}
```

## 4.4 依赖锁定

Gradle 支持依赖锁定来确保团队和 CI 使用完全相同的依赖版本：

```kotlin
// build.gradle.kts
dependencyLocking {
    lockAllConfigurations()  // 锁定所有配置
    // lockMode = LockMode.STRICT  // 严格模式：锁定文件必须存在
    // lockMode = LockMode.LENIENT  // 宽松模式（默认）
}
```

```bash
# 生成或更新锁定文件
./gradlew resolveAndLockAll --write-locks

# 生成的锁定文件：gradle/dependency-locks/*.lockfile
# 格式示例：
# com.google.guava:guava:32.1.3-jre=compileClasspath,runtimeClasspath
```

生成的锁定文件 (`gradle/dependency-locks/compileClasspath.lockfile`)：
```
# This is a Gradle generated file for dependency locking.
com.google.guava:guava:32.1.3-jre
com.google.guava:failureaccess:1.0.1
com.google.errorprone:error_prone_annotations:2.21.1
...
```

## 4.5 依赖约束（Dependency Constraints）

依赖约束允许库或项目对传递性依赖的版本提出建议或要求，而不直接引入该依赖：

```kotlin
dependencies {
    // 直接依赖
    implementation("com.example:library-a:1.0")

    // 依赖约束：不直接依赖 jackson，但如果传递性依赖图中出现了，
    // 则约束其版本
    constraints {
        implementation("com.fasterxml.jackson.core:jackson-databind:2.16.0") {
            because("所有 Jackson 模块必须使用统一版本 2.16")
        }
        implementation("com.fasterxml.jackson.module:jackson-module-kotlin:2.16.0")
    }

    // 版本约束还可以用于 BOM
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.2.0"))
}
```

# 5. BOM 与平台依赖

## 5.1 BOM（Bill of Materials）

BOM 是一个特殊的 POM，它不提供 JAR，只提供依赖版本的建议。类似于依赖管理的 "目录"。

```kotlin
dependencies {
    // 方式1：导入 BOM（推荐）
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.2.0"))
    implementation(platform("com.google.cloud:libraries-bom:26.28.0"))

    // 现在可以不写版本号，版本由 BOM 管理
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("com.google.cloud:google-cloud-storage")

    // 方式2：enforcedPlatform（强制执行，不允许其他版本覆盖）
    implementation(enforcedPlatform("org.springframework.boot:spring-boot-dependencies:3.2.0"))

    // 方式3：仅影响特定配置
    testImplementation(platform("org.junit:junit-bom:5.10.1"))
}
```

## 5.2 Spring Dependency Management Plugin 的替代

Gradle 原生支持 BOM，但有些项目仍在使用 Spring Dependency Management Plugin。以下是两种方式的对比：

```kotlin
// === 使用 Spring Dependency Management Plugin（旧方式）===
plugins {
    id("io.spring.dependency-management") version "1.1.4"
}

dependencyManagement {
    imports {
        mavenBom("org.springframework.boot:spring-boot-dependencies:3.2.0")
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:2023.0.0")
    }
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
}

// === 使用 Gradle 原生 BOM 支持（推荐新方式） ===
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.2.0"))
    implementation(platform("org.springframework.cloud:spring-cloud-dependencies:2023.0.0"))

    implementation("org.springframework.boot:spring-boot-starter-web")
}
```

## 5.3 发布 BOM

```kotlin
// 发布一个自定义 BOM
plugins {
    `java-platform`  // 特殊的平台插件
    `maven-publish`
}

dependencies {
    // BOM 中的约束
    constraints {
        api("com.google.guava:guava:32.1.3-jre")
        api("com.fasterxml.jackson.core:jackson-databind:2.16.0")
        api("org.slf4j:slf4j-api:2.0.9")
        api("ch.qos.logback:logback-classic:1.4.11")
        api("org.junit.jupiter:junit-jupiter:5.10.1")
    }
}

publishing {
    publications {
        create<MavenPublication>("maven") {
            from(components["javaPlatform"])
        }
    }
}
```

# 6. 依赖替换与模块元数据

## 6.1 Dependency Substitution

依赖替换允许在构建时将某个依赖替换为另一个（常见于复合构建）：

```kotlin
// settings.gradle.kts - 使用 includeBuild 进行依赖替换
includeBuild("../my-forked-library") {
    dependencySubstitution {
        // 当项目依赖 com.example:original-lib 时，替换为本地构建的项目
        substitute(module("com.example:original-lib")).using(project(":"))
    }
}
// 这使得你可以在开发时不发布本地修改的库而直接使用
```

在 build.gradle.kts 中进行替换：
```kotlin
// build.gradle.kts
configurations.all {
    resolutionStrategy {
        // 依赖替换
        dependencySubstitution {
            // 用 SLF4J 桥接替换 commons-logging
            substitute(module("commons-logging:commons-logging"))
                .using(module("org.slf4j:jcl-over-slf4j:2.0.9"))
                .because("统一使用 SLF4J 日志门面")

            // 用 Jakarta 替换 javax
            substitute(module("javax.annotation:javax.annotation-api"))
                .using(module("jakarta.annotation:jakarta.annotation-api:2.1.1"))

            // 替换为本地项目
            substitute(module("com.example:core"))
                .using(project(":local-core"))
        }
    }
}
```

## 6.2 Gradle Module Metadata

Gradle Module Metadata（GMM）是 Gradle 发布的额外元数据文件（`.module` 文件），补充 Maven 的 POM 信息，提供更丰富的依赖表达：

```json
// 一个 Gradle Module Metadata 文件 (.module)
{
  "formatVersion": "1.1",
  "component": {
    "group": "com.example",
    "module": "my-lib",
    "version": "1.0.0",
    "attributes": {
      "org.gradle.status": "release"
    }
  },
  "createdBy": {
    "gradle": {
      "version": "8.5"
    }
  },
  "variants": [
    {
      "name": "apiElements",
      "attributes": {
        "org.gradle.category": "library",
        "org.gradle.usage": "java-api",
        "org.gradle.jvm.version": 17
      },
      "dependencies": [
        {
          "group": "com.google.guava",
          "module": "guava",
          "version": { "requires": "32.1.3-jre" }
        }
      ],
      "files": [
        { "name": "my-lib-1.0.0.jar", "url": "my-lib-1.0.0.jar" }
      ]
    },
    {
      "name": "runtimeElements",
      "attributes": {
        "org.gradle.category": "library",
        "org.gradle.usage": "java-runtime"
      },
      "dependencies": [
        {
          "group": "com.google.guava",
          "module": "guava",
          "version": { "requires": "32.1.3-jre" }
        },
        {
          "group": "ch.qos.logback",
          "module": "logback-classic",
          "version": { "requires": "1.4.11" }
        }
      ],
      "files": [
        { "name": "my-lib-1.0.0.jar", "url": "my-lib-1.0.0.jar" }
      ]
    }
  ]
}
```

GMM 的核心优势：
1. **变体感知（Variant-aware）**：精确定义了编译时和运行时需要哪些依赖
2. **丰富的属性**：支持平台、JVM 版本、使用方式等属性
3. **更精确的版本约束**：`strictly`、`require`、`prefer`、`reject` 语义

## 6.3 模块替换（Module Replacement）

当两个 JAR 提供相同的功能但使用不同的 Maven 坐标时（例如迁移后的 Jakarte EE）：

```kotlin
// build.gradle.kts
dependencies {
    modules {
        // 声明两个模块是彼此替换
        module("commons-logging:commons-logging") {
            replacedBy("org.slf4j:jcl-over-slf4j", "SLF4J JCL Binding 是更好的替代")
        }

        module("log4j:log4j") {
            replacedBy("org.apache.logging.log4j:log4j-core", "升级到 Log4j2")
        }

        // Jakarta EE 迁移
        module("javax.annotation:javax.annotation-api") {
            replacedBy("jakarta.annotation:jakarta.annotation-api", "迁移到 Jakarta EE")
        }

        module("javax.servlet:javax.servlet-api") {
            replacedBy("jakarta.servlet:jakarta.servlet-api", "迁移到 Jakarta EE")
        }
    }
}
```

# 7. 依赖解析的高级特性

## 7.1 Capability（能力）

Capability 是 Gradle 5.0+ 引入的概念，用于解决不同模块提供同一功能时的冲突：

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.apache.logging.log4j:log4j-core:2.21.1")
    implementation("org.apache.logging.log4j:log4j-slf4j-impl:2.21.1")
}

configurations.all {
    resolutionStrategy {
        // 当两个模块提供了冲突的 Capability 时
        capabilitiesResolution {
            withCapability("org.apache.logging.log4j", "log4j-impl") {
                // 选择哪个模块来提供此能力
                select("org.apache.logging.log4j:log4j-slf4j-impl:2.21.1")
            }
        }
    }
}
```

## 7.2 Component Metadata Rules

组件元数据规则允许在依赖解析时修改模块的元数据（POM 或 .module 文件）：

```kotlin
// build.gradle.kts - 修改远程模块的元数据

// 注册自定义元数据规则
dependencies {
    components {
        // 对特定模块应用元数据规则
        withModule("com.google.guava:guava") {
            // 所有版本
            allVariants {
                // 添加一个 Capability
                withCapabilities {
                    addCapability("com.google.guava", "guava-core", id.version)
                }
            }
        }

        // 对符合规则的所有模块应用
        withModule("com.google.guava:guava") {
            // 修改其依赖
            withDependencies {
                // 移除对 error_prone_annotations 的依赖
                removeIf { it.group == "com.google.errorprone" }
            }
        }
    }
}

// 也可以使用 componentSelection 筛选组件
configurations.all {
    resolutionStrategy {
        componentSelection {
            all {
                // 拒绝非稳定版
                if (candidate.version.contains("alpha") ||
                    candidate.version.contains("beta") ||
                    candidate.version.contains("rc")) {
                    reject("拒绝不稳定版本: ${candidate.version}")
                }
            }

            // 对特定模块的筛选
            withModule("com.example:internal-lib") {
                // 仅允许公司仓库的版本
                if (candidate.version.startsWith("1.")) {
                    reject("版本 1.x 已废弃")
                }
            }
        }
    }
}
```

## 7.3 依赖验证（Dependency Verification）

Gradle 支持使用校验和和签名来验证依赖的完整性，防止供应链攻击：

```bash
# 生成依赖验证文件
./gradlew --write-verification-metadata sha256,pgp help
# 这会生成 gradle/verification-metadata.xml
```

生成的 `gradle/verification-metadata.xml`：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<verification-metadata xmlns="https://schema.gradle.org/dependency-verification"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <configuration>
        <verify-metadata>true</verify-metadata>
        <verify-signatures>true</verify-signatures>
        <trusted-artifacts>
            <trust file=".*-javadoc[.]jar" regex="true"/>
            <trust file=".*-sources[.]jar" regex="true"/>
        </trusted-artifacts>
    </configuration>
    <components>
        <component group="com.google.guava" name="guava" version="32.1.3-jre">
            <artifact name="guava-32.1.3-jre.jar">
                <sha256 value="abc123def456..." origin="Verified"/>
                <pgp value="key123..." origin="Verified"/>
            </artifact>
        </component>
    </components>
</verification-metadata>
```

# 8. 依赖管理最佳实践

## 8.1 版本管理策略

```kotlin
// build.gradle.kts - 推荐的依赖管理结构

// 1. 使用版本目录（gradle/libs.versions.toml）统一管理版本
// 2. 根项目中定义 BOM
dependencies {
    // 导入公司或生态系统的 BOM
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.2.0"))
    implementation(platform("software.amazon.awssdk:bom:2.21.29"))
}

// 3. 子项目中不写版本号
// module/build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("software.amazon.awssdk:s3")
}

// 4. 使用 dependencyLocking 确保可重现构建
dependencyLocking {
    lockAllConfigurations()
}
```

## 8.2 避免的常见陷阱

```kotlin
// ❌ 错误：在依赖中使用 + 动态版本（不可重现）
dependencies {
    implementation("com.google.guava:guava:32+")
}

// ✅ 正确：使用精确版本
dependencies {
    implementation("com.google.guava:guava:32.1.3-jre")
}

// ❌ 错误：硬编码仓库凭据
repositories {
    maven {
        url = uri("https://nexus.mycompany.com/")
        credentials {
            username = "admin"
            password = "secret123"  // ← 永远不要这样做
        }
    }
}

// ✅ 正确：从环境变量或安全的属性源读取
repositories {
    maven {
        url = uri("https://nexus.mycompany.com/")
        credentials {
            username = System.getenv("NEXUS_USER")
            password = System.getenv("NEXUS_PASS")
        }
    }
}

// ❌ 错误：使用 flatDir 作为主要仓库（丢失传递性依赖信息）
repositories {
    flatDir { dirs("libs") }
}

// ✅ 正确：使用 Maven 本地仓库作为补充
repositories {
    mavenLocal()  // 仅在必要时使用
    mavenCentral()
}
```

## 8.3 大型项目的依赖优化

```kotlin
// settings.gradle.kts - settings 级别的依赖仓库统一管理
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)
    repositories {
        mavenCentral()
        maven { url = uri("https://nexus.mycompany.com/repository/maven-public/") }
    }
}

// build.gradle.kts（根项目）
subprojects {
    configurations.all {
        resolutionStrategy {
            // 每个子项目都强制使用统一版本
            force(
                "com.google.guava:guava:32.1.3-jre",
                "com.fasterxml.jackson.core:jackson-databind:2.16.0",
                "org.slf4j:slf4j-api:2.0.9"
            )

            // 缓存动态版本（减少网络请求）
            cacheDynamicVersionsFor(10, "minutes")
            cacheChangingModulesFor(4, "hours")
        }

        // 全局排除不需要的传递性依赖
        exclude(group = "commons-logging")
        exclude(group = "log4j")
    }

    // 打印依赖树（调试用）
    // gradle :module:dependencies --configuration compileClasspath
}

// 查看依赖树的命令
// ./gradlew dependencies                     ← 所有配置
// ./gradlew :core:dependencies               ← 特定项目
// ./gradlew :core:dependencies --configuration compileClasspath  ← 特定配置
// ./gradlew buildEnvironment                 ← 构建脚本的 classpath
// ./gradlew dependencyInsight --dependency guava  ← 深度分析特定依赖
```

## 8.4 dependencyInsight 使用

```bash
# 分析某个依赖为什么出现在 classpath 中
./gradlew dependencyInsight --dependency guava --configuration compileClasspath

# 输出示例：
# com.google.guava:guava:32.1.3-jre
#   Variant runtime:
#     | Attribute Name              | Provided | Requested    |
#     |-----------------------------+----------+--------------|
#     | org.gradle.status           | release  |              |
#     | org.gradle.category         | library  | library      |
#     | org.gradle.usage            | java-runtime | java-api  |
#     ...
#    Selection reasons:
#       - By constraint from project :
#       - Forced
#
# com.google.guava:guava:32.1.3-jre
# \--- compileClasspath
#
# A web-based, searchable dependency report is available by adding the --scan option.

# 查看所有配置中的特定依赖
./gradlew dependencyInsight --dependency spring-boot-starter-web
```

## 8.5 依赖对齐（Dependency Alignment）

当使用一个生态系统的多个模块时，确保它们使用同一个版本：

```kotlin
// 手动对齐 Spring 模块版本
abstract class SpringAlignmentRule : ComponentMetadataRule {
    override fun execute(ctx: ComponentMetadataContext) {
        ctx.details.run {
            if (id.group.startsWith("org.springframework")) {
                belongsTo("org.springframework:spring-platform:${id.version}")
            }
        }
    }
}

dependencies {
    components {
        withModule<SpringAlignmentRule>("org.springframework:*")
    }
}
// 但更好的方式还是使用 BOM/property
```
