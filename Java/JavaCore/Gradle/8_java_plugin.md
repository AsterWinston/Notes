
# 1. Java 插件概述

## 1.1 插件提供的核心组件

```groovy
plugins {
    id 'java'
}
```

应用 `java` 插件后，Gradle 自动向项目添加以下组件：

| 组件类型 | 默认约定 | 自定义方式 |
|---------|---------|-----------|
| 两个 SourceSet | `main` + `test` | `sourceSets { create('integration') }` |
| 六个编译/处理配置 | compileClasspath, runtimeClasspath 等 | `configurations {}` |
| 生命周期任务 | `build`, `assemble`, `check` | - |
| 功能任务 | `compileJava`, `processResources`, `classes`, `jar`, `test`, `compileTestJava`, `processTestResources`, `testClasses`, `javadoc`, `clean` | `tasks.named {}` |
| 产物 | `default` 配置 + `archives` 配置 | `configurations.archives` |

## 1.2 Java 插件添加的完整任务图

```
build
├── assemble
│   └── jar
│       ├── classes
│       │   ├── compileJava
│       │   └── processResources
│       └── (META-INF 包含资源)
│
└── check
    └── test
        ├── testClasses
        │   ├── compileTestJava
        │   │   └── classes (main output)
        │   └── processTestResources
        └── (运行 tests)
```

# 2. SourceSet 概念

## 2.1 SourceSet 的结构

`SourceSet` 是 Java 插件的核心抽象，它将一组源文件、资源和输出目录组织成一个逻辑单元。

```
SourceSet (如 "main")
├── java (SourceDirectorySet)
│   └── src/main/java/**/*.java
├── resources (SourceDirectorySet)
│   └── src/main/resources/**/*
├── output (SourceSetOutput)
│   ├── classesDirs (main 编译输出)
│   └── resourcesDir (resources 复制目标)
├── compileClasspathConfigurationName → compileClasspath
├── runtimeClasspathConfigurationName → runtimeClasspath
└── 关联的 Compile/ProcessResources 任务
```

## 2.2 自定义 SourceSet

```groovy
sourceSets {
    // 添加集成测试 source set
    integrationTest {
        java.srcDir file('src/integration-test/java')
        resources.srcDir file('src/integration-test/resources')
        // 继承 main 的输出
        compileClasspath += sourceSets.main.output
        runtimeClasspath += sourceSets.main.output
    }
}

// Gradle 自动创建：
// - compileIntegrationTestJava
// - processIntegrationTestResources
// - integrationTestClasses
```

**引入自定义配置**：

```groovy
configurations {
    integrationTestImplementation.extendsFrom testImplementation
    integrationTestRuntimeOnly.extendsFrom testRuntimeOnly
}

dependencies {
    integrationTestImplementation 'org.junit.jupiter:junit-jupiter:5.10.0'
}

tasks.register('integrationTest', Test) {
    description = '运行集成测试'
    group = 'verification'

    testClassesDirs = sourceSets.integrationTest.output.classesDirs
    classpath = sourceSets.integrationTest.runtimeClasspath
    mustRunAfter tasks.named('test')
}
```

## 2.3 SourceSetOutput

```groovy
// 跨 SourceSet 共享输出
dependencies {
    // 让另一个 source set 依赖 main 的输出
    integrationTestImplementation sourceSets.main.output

    // 在 Groovy 插件中依赖 Java 输出
    // groovy 编译依赖 java 编译输出
    implementation sourceSets.main.output
}
```

**内部机制**：`sourceSets.main.output` 返回 `SourceSetOutput`，其 `classesDirs` 属性指向编译后的 `.class` 目录，`resourcesDir` 指向处理后的资源目录。在实际配置中，Gradle 将它们分别映射到对应的 `Configuration`。

## 2.4 `SourceSet` 的资源过滤

```groovy
sourceSets {
    main {
        resources {
            // 排除某些文件
            exclude '**/*.psd'
            exclude '**/*.ai'

            // 只包含特定文件
            include '**/*.properties'
            include '**/*.xml'
        }
    }
}
```

## 2.5 多目录源码布局

```groovy
sourceSets {
    main {
        java {
            srcDirs = [
                'src/main/java',
                'src/gen/java',           // 生成的源码
                "${buildDir}/generated/sources/annotationProcessor" // APT 输出
            ]
        }
        resources {
            srcDirs = [
                'src/main/resources',
                'src/main/conf'           // 额外的资源目录
            ]
        }
    }
}
```

# 3. JavaCompile 任务详解

## 3.1 编译选项配置

```groovy
tasks.withType(JavaCompile).configureEach {
    // 源码和目标版本
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17

    // 编译选项
    options.encoding = 'UTF-8'
    options.compilerArgs = [
        '-Xlint:unchecked',
        '-Xlint:deprecation',
        '-parameters',             // 保留参数名（Spring 等框架需要）
        '-Xmaxerrs', '1000',
        '-Xmaxwarns', '1000'
    ]

    // 分叉编译（优化内存）
    options.fork = true
    options.forkOptions.memoryMaximumSize = '2g'
    options.forkOptions.jvmArgs = ['-XX:+UseG1GC']

    // 增量编译（默认开启）
    options.incremental = true

    // 调试信息级别
    options.debugOptions.debugLevel = 'source,lines,vars'

    // 注解处理器路径
    options.annotationProcessorPath = configurations.annotationProcessor
}
```

```kotlin
// Kotlin DSL 等价写法
tasks.withType<JavaCompile>().configureEach {
    sourceCompatibility = JavaVersion.VERSION_17.toString()
    targetCompatibility = JavaVersion.VERSION_17.toString()
    options.encoding = "UTF-8"
    options.compilerArgs.addAll(listOf("-Xlint:unchecked", "-parameters"))
    options.isFork = true
    options.forkOptions.memoryMaximumSize = "2g"
    options.isIncremental = true
}
```

## 3.2 编译优化 — 模块化编译

```groovy
// 启用多模块并行编译
tasks.withType(JavaCompile).configureEach {
    options.compilerArgs << '--module-version' << project.version
    options.compilerArgs << '--module-source-path'
    options.compilerArgs << files(
        sourceSets.main.java.srcDirs,
        sourceSets.test.java.srcDirs
    ).asPath
}
```

## 3.3 跨 SourceSet 共享编译器选项

```groovy
// 为所有 JavaCompile 任务设置公共选项
tasks.withType(JavaCompile).configureEach {
    options.encoding = 'UTF-8'
    doFirst {
        // 在执行阶段注入编译参数（基于当前状态）
        if (project.hasProperty('debugCompile')) {
            options.compilerArgs << '-verbose'
        }
    }
}

// 针对特定编译任务的特殊配置
tasks.named('compileJava').configure {
    options.compilerArgs << '-Aproject=${project.group}/${project.name}'
}

tasks.named('compileTestJava').configure {
    options.compilerArgs << '-proc:none'  // 测试编译跳过注解处理
}
```

# 4. `processResources` 任务

## 4.1 资源处理配置

```groovy
tasks.named('processResources', ProcessResources).configure {
    // 复制时进行变量替换（占位符替换）
    filesMatching('**/*.properties') {
        expand(project.properties)
    }
    filesMatching('**/*.yml') {
        filter { line ->
            line.replace('${project.version}', project.version.toString())
        }
    }

    // 指定输入输出编码
    filteringCharset = 'UTF-8'

    // 复制时排除文件
    exclude '**/*.psd'
}
```

## 4.2 资源过滤与转换

```groovy
import org.apache.tools.ant.filters.ReplaceTokens

tasks.named('processResources').configure {
    filesMatching('**/application.properties') {
        filter(ReplaceTokens, tokens: [
            'APP_VERSION': project.version,
            'BUILD_TIME': new Date().format('yyyy-MM-dd HH:mm:ss'),
            'COMMIT_HASH': 'git rev-parse --short HEAD'.execute().text.trim()
        ])
    }
}
```

# 5. `jar` 任务详解

## 5.1 JAR 清单配置

```groovy
tasks.named('jar', Jar).configure {
    // 基本属性
    archiveBaseName = 'my-app'
    archiveVersion = '2.0.0'
    archiveClassifier = ''

    // Manifest 配置
    manifest {
        attributes(
            'Implementation-Title': project.name,
            'Implementation-Version': project.version,
            'Implementation-Vendor': 'My Company',
            'Built-By': System.getProperty('user.name'),
            'Built-JDK': System.getProperty('java.version'),
            'Build-Time': new Date().format("yyyy-MM-dd'T'HH:mm:ssZ"),
            'Created-By': "Gradle ${gradle.gradleVersion}",
            'Main-Class': 'com.mycompany.Main'
        )
    }

    // 包含/排除规则
    exclude '**/*.psd'
    exclude '**/internal/**'

    // 保留特定目录结构
    eachFile { details ->
        if (details.path.startsWith('META-INF/services/')) {
            details.path = details.path  // 保留
        }
    }

    // 压缩级别
    entryCompression = ZipEntryCompression.STORED
}
```

## 5.2 Uber-JAR / Fat-JAR 构建

```groovy
tasks.register('fatJar', Jar) {
    archiveClassifier = 'fat'

    manifest {
        attributes 'Main-Class': 'com.mycompany.Main'
    }

    from {
        configurations.runtimeClasspath.collect {
            it.isDirectory() ? it : zipTree(it)
        }
    }

    // 处理重复文件
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE

    with jar  // 继承 jar 任务的配置
}
```

## 5.3 自定义 JAR 复制逻辑

```groovy
tasks.named('jar').configure {
    into('lib') {
        from configurations.runtimeClasspath
    }

    into('docs') {
        from 'README.md', 'LICENSE'
    }

    // 重命名文件
    rename '(.+)-(.+)\\.jar', '$1.jar'
}
```

# 6. `test` 任务详解

## 6.1 测试配置

```groovy
tasks.withType(Test).configureEach {
    // 测试框架
    useJUnitPlatform()  // JUnit 5
    // useTestNG()      // TestNG

    // JVM 参数
    jvmArgs = [
        '-Xmx1g',
        '-XX:+UseG1GC',
        '-Dfile.encoding=UTF-8'
    ]

    // 系统属性
    systemProperty 'test.env', System.getProperty('test.env', 'dev')
    systemProperty 'spring.profiles.active', 'test'

    // 并行执行
    maxParallelForks = Runtime.runtime.availableProcessors().intdiv(2) ?: 1
    forkEvery = 100  // 每 100 个测试重启一次 JVM

    // 测试日志
    testLogging {
        events 'passed', 'skipped', 'failed'
        showStandardStreams = true
        exceptionFormat = 'full'
        showCauses = true
        showStackTraces = true
        showExceptions = true
        info.events = ['passed', 'skipped', 'failed']
    }

    // 失败处理
    ignoreFailures = false
    maxFailures = 10

    // 超时
    timeout = Duration.ofMinutes(10)

    // 过滤
    filter {
        includeTestsMatching '*UnitTest'
        excludeTestsMatching '*IntegrationTest'
    }

    // 特定包
    if (project.hasProperty('testPackage')) {
        filter {
            includeTestsMatching "${project.property('testPackage')}.*"
        }
    }
}
```

## 6.2 JUnit Platform 配置

```groovy
dependencies {
    testImplementation platform('org.junit:junit-bom:5.10.0')
    testImplementation 'org.junit.jupiter:junit-jupiter'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

test {
    useJUnitPlatform {
        includeTags 'fast'
        excludeTags 'slow', 'integration'
        includeEngines 'junit-jupiter'
        // excludeEngines 'junit-vintage'
    }

    // JUnit Platform 配置参数
    systemProperty 'junit.jupiter.execution.parallel.enabled', 'true'
    systemProperty 'junit.jupiter.execution.parallel.mode.default', 'concurrent'
    systemProperty 'junit.jupiter.execution.parallel.config.strategy', 'dynamic'
}
```

## 6.3 测试报告

```groovy
tasks.named('test').configure {
    reports {
        html.required = true
        html.outputLocation = layout.buildDirectory.dir('reports/tests')
        junitXml.required = true
        junitXml.outputLocation = layout.buildDirectory.dir('test-results')
    }
}

// 汇总测试报告（多模块项目）
tasks.register('testReport', TestReport) {
    destinationDirectory = file("$buildDir/reports/allTests")
    testResults.from(subprojects*.test*.binaryResultsDirectory)
}
```

# 7. Java Plugin Extension

## 7.1 `java {}` 块配置

```groovy
java {
    // 源码/目标兼容性
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17

    // 模块化支持
    modularity.inferModulePath = true
    modularity.getModuleName().set('com.mycompany.mymodule')

    // 自动生成 Sources Jar 和 Javadoc Jar
    withSourcesJar()
    withJavadocJar()

    // 注册特性变体（Feature Variants）
    registerFeature('mysqlSupport') {
        usingSourceSet(sourceSets.main)
        capability('com.mycompany', 'my-library-mysql-support', project.version.toString())
    }
}
```

## 7.2 `toolchain` — 工具链管理

```groovy
java {
    toolchain {
        // 指定 JDK 版本（不同于 sourceCompatibility）
        languageVersion = JavaLanguageVersion.of(17)
        vendor = JvmVendorSpec.ADOPTIUM  // 或 AMAZON, MICROSOFT 等

        // 实现（可选，自动检测）
        implementation = JvmImplementation.VENDOR_SPECIFIC
    }
}

// 查看当前使用的 JDK
tasks.register('showJdk') {
    doLast {
        println "Java: ${java.toolchain.languageVersion.get()}"
        println "JDK: ${System.getProperty('java.home')}"
    }
}
```

**重要性**：`toolchain` 与 `sourceCompatibility` 不同。`sourceCompatibility` 只影响 `javac -source` 标志（源码语法级别），而 `toolchain` 确保使用**特定版本的 JDK** 来执行编译、测试和 Javadoc 生成。

# 8. Java Library Plugin

## 8.1 `api` vs `implementation`

```groovy
plugins {
    id 'java-library'
}

dependencies {
    // api：暴露给消费者（传递到编译类路径）
    api 'com.google.guava:guava:32.1.3-jre'

    // implementation：内部实现，不暴露给消费者
    implementation 'org.apache.commons:commons-lang3:3.14.0'

    // api 的另一个用途：注解库应为 api
    api 'org.immutables:value:2.9.0'

    // implementation 适用于内部依赖
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.16.0'
}
```

**内部机制 — 类路径泄露（Classpath Leakage）**：

```
Project A (Library)
├── api → guava          (暴露，消费者可见)
└── implementation → commons-lang3  (隐藏，消费者不可见)

Project B (Consumer: implementation project(':A'))
├── compileClasspath 包含: guava ✓
└── compileClasspath 不含: commons-lang3  (避免类路径泄露)
```

```kotlin
// Kotlin DSL 等价示例
plugins {
    `java-library`
}

dependencies {
    api("com.google.guava:guava:32.1.3-jre")
    implementation("org.apache.commons:commons-lang3:3.14.0")
    compileOnly("org.projectlombok:lombok:1.18.30")
    annotationProcessor("org.projectlombok:lombok:1.18.30")
    runtimeOnly("ch.qos.logback:logback-classic:1.4.11")
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.0")
}
```

## 8.2 `java-library` 创建的配置

应用 `java-library` 后，除了 `java` 插件的配置，还增加了：

```
configurations:
├── api (可传递)
├── implementation (不可传递)
├── compileOnlyApi (对消费者可用的编译期依赖)
├── apiElements (变体)
├── runtimeElements (变体)
│...
```

## 8.3 配置选择决策表

| 场景 | 使用 |
|------|------|
| 依赖类型在公共 API 中暴露 | `api` |
| 依赖仅在模块内部使用 | `implementation` |
| 编译时需要但运行时由容器提供 | `compileOnly` |
| 运行时需要但不参与编译 | `runtimeOnly` |
| 注解处理器 | `annotationProcessor` |
| 测试依赖 | `testImplementation` |
| API 编译时需要但不传递 | `compileOnlyApi` |

# 9. Application Plugin

## 9.1 基本配置

```groovy
plugins {
    id 'application'
}

application {
    // 主类入口
    mainClass = 'com.mycompany.Main'
    // 或通过模块
    mainModule = 'com.mycompany.app'

    // JVM 参数
    applicationDefaultJvmArgs = [
        '-Xmx512m',
        '-Dfile.encoding=UTF-8',
        '-Dapp.env=production'
    ]

    // 可执行脚本
    executableDir = 'bin'
}
```

## 9.2 Application Plugin 添加的任务

```
application
├── run — 直接运行应用（gradle run）
├── startScripts — 生成启动脚本（.bat/.sh）
├── installDist — 安装分发到 build/install
└── distZip / distTar — 创建分发包
```

## 9.3 自定义分发

```groovy
distributions {
    main {
        distributionBaseName = 'myapp'

        contents {
            from('README.md')
            from('LICENSE')
            into('config') {
                from 'src/main/conf'
            }
            into('lib') {
                from configurations.runtimeClasspath
            }
        }
    }
}

// 额外的分发配置
distributions {
    source {
        distributionBaseName = 'myapp-sources'
        contents {
            from sourceSets.main.allSource
            from 'build.gradle'
        }
    }
}

// 自定义启动脚本
tasks.named('startScripts').configure {
    defaultJvmOpts = ['-Xmx2g', '-Dspring.profiles.active=prod']
    classpath = files(jar.archiveFile) + configurations.runtimeClasspath
    applicationName = 'start-myapp'
}
```

# 10. Java Platform Plugin

## 10.1 约束声明

```groovy
plugins {
    id 'java-platform'
}

javaPlatform {
    allowDependencies()
}

dependencies {
    // 约束了版本，但不实际引入
    constraints {
        api 'com.google.guava:guava:32.1.3-jre'
        api 'org.apache.commons:commons-lang3:3.14.0'
        api 'com.fasterxml.jackson.core:jackson-databind:2.16.0'
        api 'org.slf4j:slf4j-api:2.0.9'
    }
}
```

消费者使用：

```groovy
// 消费者
dependencies {
    implementation platform(project(':platform'))
    implementation 'com.google.guava:guava'  // 版本由 platform 约束
    implementation 'org.slf4j:slf4j-api'     // 版本由 platform 约束
}
```

## 10.2 对齐依赖版本

```groovy
// 消费者端强制版本对齐
configurations.all {
    resolutionStrategy {
        // 强制特定版本
        force 'com.google.guava:guava:32.1.3-jre'

        // 或缓存动态版本时间
        cacheDynamicVersionsFor 10, 'minutes'
        cacheChangingModulesFor 0, 'seconds'
    }
}
```

# 11. Java Module System（JPMS）

## 11.1 模块化配置

```groovy
java {
    modularity.inferModulePath = true
}

// module-info.java 位于 src/main/java/
// module com.mycompany.app {
//     requires com.google.guava;
//     requires org.slf4j;
//     exports com.mycompany.app.api;
// }
```

## 11.2 模块路径的编译配置

```groovy
tasks.withType(JavaCompile).configureEach {
    // 将依赖放在模块路径上
    doFirst {
        options.compilerArgs.addAll([
            '--module-path', classpath.asPath
        ])
        classpath = files()
    }
}

// 添加模块化的系统属性
tasks.withType(Test).configureEach {
    jvmArgs = [
        '--module-path', classpath.asPath,
        '--add-modules', 'ALL-MODULE-PATH'
    ]
}
```

## 11.3 传统库的自动模块化

```groovy
// 为非模块化 JAR 提供自动模块名
tasks.named('jar').configure {
    manifest {
        attributes('Automatic-Module-Name': 'com.mycompany.library')
    }
}
```

# 12. JVM 语言集成

## 12.1 Groovy 插件

```groovy
plugins {
    id 'groovy'
}

dependencies {
    implementation 'org.codehaus.groovy:groovy:4.0.18'
}
// 添加 src/main/groovy 源码目录
// 提供 compileGroovy / compileTestGroovy 任务
// 支持 Java 和 Groovy 联合编译
```

## 12.2 Scala 插件

```groovy
plugins {
    id 'scala'
}

dependencies {
    implementation 'org.scala-lang:scala3-library_3:3.3.1'
}

// Scala 编译选项
tasks.withType(ScalaCompile).configureEach {
    scalaCompileOptions.additionalParameters = [
        '-feature',
        '-language:implicitConversions'
    ]
}
```

## 12.3 Kotlin 插件

```kotlin
plugins {
    kotlin("jvm") version "1.9.20"
}

// 配置
kotlin {
    jvmToolchain(17)
}

dependencies {
    implementation(kotlin("stdlib"))
}

// Kotlin 编译选项
tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompile>().configureEach {
    kotlinOptions {
        jvmTarget = "17"
        freeCompilerArgs = listOf("-Xjsr305=strict")
    }
}
```

## 12.4 JVM 语言混合编译的图像

```
src/main/
├── java/     → compileJava ─────┐
├── groovy/   → compileGroovy ───┤
├── scala/    → compileScala ────┤
└── kotlin/   → compileKotlin ───┘
                                  ▼
                           classes/
                               │
                       jar ◄────┘
```

# 13. 高级编译策略

## 13.1 编译器守护进程

```groovy
// gradle.properties
org.gradle.jvmargs=-Xmx2g -XX:+UseG1GC -Dfile.encoding=UTF-8
org.gradle.daemon=true
```

## 13.2 编译避免（Compilation Avoidance）

Gradle 的 Java 编译内置了编译避免机制：如果 JAR 的 public API 未变更，下游模块不会重新编译。

```groovy
// 在 java-library 模块中声明 API 依赖
dependencies {
    api 'com.google.guava:guava:32.1.3-jre'  // API 变更会触发下游重编译
    implementation 'commons-io:commons-io:2.15.0'  // 内部变更不触发下游
}
```

**工作原理**：Gradle 分析每个输出的 public API（类、方法、字段签名），仅当 API 发生变更时才重新编译下游。这比时间戳检测更精确。

---

**总结**：Java 插件是 Gradle 生态中使用最广泛的插件。掌握 SourceSet 自定义、`api`/`implementation` 区别、`java-library` 的类路径隔离、以及 `toolchain` 机制，是构建健壮 JVM 项目的关键。核心建议：（1）使用 `java-library` 而非裸 `java` 以获得 API 隔离；（2）利用 `toolchain` 管理 JDK 版本；（3）自定义 SourceSet 处理非标准源码布局；（4）始终显式声明 `annotationProcessor` 配置以支持注解处理器。
