
# 1. 概述

Gradle 是一个强大且灵活的构建工具，但灵活性也意味着容易引入反模式。本章汇集了从项目结构组织、构建脚本编写、依赖管理到性能优化等各方面的最佳实践，帮助开发者构建高效、可维护的 Gradle 项目。

## 1.1 最佳实践核心原则

```
Gradle 最佳实践金字塔
          ┌──────────────┐
          │   安全性      │  ← 凭据管理、签名
         ┌┴──────────────┴┐
         │   CI/CD 集成   │  ← 自动化、缓存
        ┌┴────────────────┴┐
        │   性能优化       │  ← 缓存、并行、避免
       ┌┴──────────────────┴┐
       │   依赖管理         │  ← 版本目录、BOM
      ┌┴────────────────────┴┐
      │   任务设计           │  ← 增量、输入输出
     ┌┴──────────────────────┴┐
     │   项目组织             │  ← 模块化、约定
    ┌┴────────────────────────┴┐
    │   构建脚本可读性         │  ← 命名、结构、注释
   └───────────────────────────┘
```

# 2. 项目组织最佳实践

## 2.1 单一职责构建文件

每个构建脚本应只负责一个方面。不要在根项目中混合所有配置。

```groovy
// ✅ 推荐: 单一职责 - 每个文件只做一件事
// root build.gradle
plugins {
    id 'base'
    id 'java-library' apply false
}

allprojects {
    group = 'com.example'
    version = '1.0.0'
}

// root/settings.gradle
rootProject.name = 'my-application'

include 'core'
include 'web'
include 'cli'

// core/build.gradle - 只处理核心模块
plugins {
    id 'java-library'
}

dependencies {
    api 'com.google.guava:guava:33.0.0-jre'
}

// web/build.gradle - 只处理 Web 模块
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.5'
}

dependencies {
    implementation project(':core')
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

**反模式识别：**

```groovy
// ❌ 反模式: 在 root 的 allprojects 中混合所有模块的配置
allprojects {
    apply plugin: 'java'
    dependencies {
        implementation 'com.google.guava:guava:33.0.0-jre'  // 所有模块都依赖?
        implementation 'org.springframework.boot:spring-boot-starter-web'  // 不应该
    }
    group = 'com.example'
}
```

## 2.2 提取逻辑到插件

重复的配置逻辑应提取为约定插件（Convention Plugin）。

```groovy
// buildSrc/src/main/groovy/my-project.java-conventions.gradle
plugins {
    id 'java'
    id 'checkstyle'
    id 'jacoco'
}

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17

    withSourcesJar()
    withJavadocJar()
}

checkstyle {
    toolVersion = '10.14.2'
    configFile = rootProject.file('config/checkstyle/checkstyle.xml')
}

jacoco {
    toolVersion = '0.8.11'
}

tasks.withType(Test).configureEach {
    useJUnitPlatform()
    finalizedBy tasks.named('jacocoTestReport')
}

// 使用约定插件
// core/build.gradle
plugins {
    id 'my-project.java-conventions'
    id 'java-library'
}

// web/build.gradle
plugins {
    id 'my-project.java-conventions'
    id 'org.springframework.boot'
}
```

## 2.3 buildSrc vs 独立插件

| 特性 | buildSrc | 独立插件 (composite build) |
|------|----------|----------------------------|
| 可见性 | 仅本项目 | 可跨项目共享 |
| 修改成本 | 低（直接修改） | 需发布新版本 |
| 编译依赖 | 自动管理 | 需显式声明 |
| 适用场景 | 项目专属逻辑 | 组织级约定 |
| 变更检测 | 自动（修改即生效） | 需发布后使用 |

```groovy
// settings.gradle - 使用复合构建引入插件
pluginManagement {
    includeBuild '../my-convention-plugins'
}

plugins {
    id 'com.example.java-conventions' version '1.0.0'
}
```

# 3. 配置避免 (Configuration Avoidance)

## 3.1 使用 register 而非 create

```groovy
// ❌ 反模式: 立即配置
task myTask(type: Copy) {
    from 'src'
    into 'dest'
}

// ✅ 推荐: 延迟配置
tasks.register('myTask', Copy) {
    from 'src'
    into 'dest'
}
```

**性能对比：**

```
假设项目有 100 个任务，但只执行 compileJava:

- 使用 task() + create(): 所有 100 个任务被创建和配置
  配置时间: ~800ms

- 使用 tasks.register(): 只有用到的任务被创建和配置
  配置时间: ~50ms

节省: 750ms × 每天 50 次构建 = 37.5 秒/天
```

## 3.2 Provider 延迟连接

```groovy
// ❌ 反模式: 急切的属性解析
abstract class EagerTask extends DefaultTask {
    @Input
    String message = System.getProperty('user.dir')  // 配置阶段就解析
}

// ✅ 推荐: 延迟属性解析
abstract class LazyTask extends DefaultTask {

    @Input
    abstract Property<String> getMessage()

    LazyTask() {
        message.convention(
            providers.systemProperty('user.dir')
        )
    }
}

// 延迟组合多个 Provider
tasks.register('complex', MyTask) {
    message = providers.gradleProperty('app.message')
        .map { it.toUpperCase() }
        .orElse('Default Message')
}
```

## 3.3 文件集合延迟解析

```groovy
// ❌ 反模式: 配置阶段解析文件
def inputFiles = fileTree('src/data') { include '**/*.csv' }  // 立即解析

tasks.register('processData') {
    inputs.files(inputFiles)  // 已解析的结果
}

// ✅ 推荐: 使用 Provider 延迟
abstract class DataProcessTask extends DefaultTask {

    @InputFiles
    @PathSensitive(PathSensitivity.RELATIVE)
    abstract ConfigurableFileCollection getDataFiles()

    @TaskAction
    void process() {
        // 仅在执行时获取文件列表
        dataFiles.each { file ->
            println "处理: ${file.name}"
        }
    }
}

tasks.register('processData', DataProcessTask) {
    // 延迟配置仅在任务执行时解析
    dataFiles.from(
        layout.projectDirectory.dir('src/data').asFileTree.matching {
            include '**/*.csv'
        }
    )
}
```

# 4. 构建脚本可读性

## 4.1 命名约定

```groovy
// ✅ 推荐的命名规范
ext {
    // 版本号: 使用有意义的名称
    springBootVersion = '3.2.5'
    jacksonVersion = '2.17.0'
    junitVersion = '5.10.2'

    // 构建参数: 使用前缀描述用途
    buildReleaseCandidate = false
    deployTargetEnvironment = 'staging'

    // 常量: 使用 UPPER_CASE
    DEFAULT_ENCODING = 'UTF-8'
    JVM_TARGET = 17
}

// ❌ 模糊的命名
ext {
    ver1 = '3.2.5'   // ver1 是什么?
    flag = true       // 什么标志?
    str = 'staging'   // 什么字符串?
}
```

## 4.2 提取常量与实用方法

```groovy
// buildSrc/src/main/groovy/Dependencies.groovy
class Versions {
    static final String SPRING_BOOT = '3.2.5'
    static final String SPRING_CLOUD = '2023.0.1'
    static final String JACKSON = '2.17.0'
    static final String GUAVA = '33.0.0-jre'
    static final String LOMBOK = '1.18.32'
}

class Libs {
    static final String SPRING_BOOT_STARTER_WEB =
        "org.springframework.boot:spring-boot-starter-web:${Versions.SPRING_BOOT}"
    static final String GUAVA =
        "com.google.guava:guava:${Versions.GUAVA}"
    static final String LOMBOK =
        "org.projectlombok:lombok:${Versions.LOMBOK}"
}

// build.gradle 中使用
dependencies {
    implementation Libs.SPRING_BOOT_STARTER_WEB
    implementation Libs.GUAVA
    compileOnly Libs.LOMBOK
    annotationProcessor Libs.LOMBOK
}
```

## 4.3 代码组织

```groovy
// build.gradle 推荐的代码组织顺序
// ==========================================
// 1. 插件声明
plugins {
    id 'java-library'
    id 'maven-publish'
    id 'checkstyle'
}

// ==========================================
// 2. 基础配置
group = 'com.example'
version = '1.0.0'

java {
    sourceCompatibility = JavaVersion.VERSION_17
}

// ==========================================
// 3. 仓库配置
repositories {
    mavenCentral()
}

// ==========================================
// 4. 依赖声明
dependencies {
    api 'com.google.guava:guava:33.0.0-jre'
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2'
}

// ==========================================
// 5. 自定义任务
tasks.register('generateMetadata') {
    // ...
}

// ==========================================
// 6. 插件特定配置
checkstyle {
    toolVersion = '10.14.2'
}

// ==========================================
// 7. 发布配置
publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java
        }
    }
}
```

# 5. 依赖管理最佳实践

## 5.1 版本目录 (Version Catalogs)

```toml
# gradle/libs.versions.toml
[versions]
spring-boot = "3.2.5"
spring-cloud = "2023.0.1"
jackson = "2.17.0"
guava = "33.0.0-jre"
lombok = "1.18.32"
junit-jupiter = "5.10.2"
mockito = "5.11.0"

[libraries]
spring-boot-starter-web = { module = "org.springframework.boot:spring-boot-starter-web" }
spring-boot-starter-test = { module = "org.springframework.boot:spring-boot-starter-test" }
guava = { module = "com.google.guava:guava", version.ref = "guava" }
jackson-databind = { module = "com.fasterxml.jackson.core:jackson-databind", version.ref = "jackson" }
lombok = { module = "org.projectlombok:lombok", version.ref = "lombok" }
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit-jupiter" }

[bundles]
spring-web = ["spring-boot-starter-web", "spring-boot-starter-test"]
testing = ["junit-jupiter"]

[plugins]
spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
spring-dependency-management = { id = "io.spring.dependency-management", version = "1.1.5" }
```

```groovy
// build.gradle 中使用版本目录
dependencies {
    implementation libs.guava
    implementation libs.bundles.spring.web
    compileOnly libs.lombok
    annotationProcessor libs.lombok
    testImplementation libs.bundles.testing
}
```

## 5.2 BOM 的使用

```groovy
dependencies {
    // 导入 BOM 但不过度约束
    implementation platform('org.springframework.boot:spring-boot-dependencies:3.2.5')
    implementation platform('com.fasterxml.jackson:jackson-bom:2.17.0')

    // 强制对齐（仅在确有必要时）
    implementation enforcedPlatform('org.apache.logging.log4j:log4j-bom:2.23.1')

    // 依赖版本由 BOM 管理
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'com.fasterxml.jackson.core:jackson-databind'
}
```

## 5.3 避免动态版本

```groovy
// ❌ 反模式: 使用动态版本
dependencies {
    implementation 'com.example:lib:1.+'                    // 不确定的版本
    implementation 'com.example:lib:latest.release'          // 最不推荐
    implementation 'com.example:lib:1.0-SNAPSHOT'           // 仅开发环境
}

// ✅ 推荐: 使用确定版本
dependencies {
    implementation 'com.example:lib:1.2.3'
    implementation 'com.example:lib:1.2.3'  // 精确版本
}

// 仅在本地开发时使用 SNAPSHOT
if (project.hasProperty('use-snapshots')) {
    dependencies {
        implementation 'com.example:lib:1.2.4-SNAPSHOT'
    }
}
```

## 5.4 依赖分组与注释

```groovy
dependencies {
    // ====================
    // 核心库
    // ====================
    implementation 'com.google.guava:guava:33.0.0-jre'
    implementation 'org.apache.commons:commons-lang3:3.14.0'
    implementation 'org.slf4j:slf4j-api:2.0.12'

    // ====================
    // Web 框架
    // ====================
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-validation'

    // ====================
    // 数据库
    // ====================
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
    runtimeOnly 'com.h2database:h2:2.2.224'

    // ====================
    // 开发工具 (不打包到最终产物)
    // ====================
    compileOnly 'org.projectlombok:lombok:1.18.32'
    annotationProcessor 'org.projectlombok:lombok:1.18.32'

    // ====================
    // 测试
    // ====================
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2'
    testImplementation 'org.mockito:mockito-core:5.11.0'
}
```

# 6. 任务设计最佳实践

## 6.1 单一职责任务

```groovy
// ❌ 反模式: 一个任务做太多事
tasks.register('buildAndDeploy') {
    doLast {
        // 编译
        // 测试
        // 打包
        // 部署
        // 通知
    }
}

// ✅ 推荐: 拆分为独立任务
tasks.register('compile') { /* 编译 */ }
tasks.register('test') { /* 测试 */ }
tasks.register('package') { /* 打包 */ }
tasks.register('deploy') { /* 部署 */ }
tasks.register('notify') { /* 通知 */ }

tasks.register('buildAndDeploy') {
    dependsOn 'compile', 'test', 'package', 'deploy'
    finalizedBy 'notify'
}
```

## 6.2 声明输入输出

```groovy
// ❌ 反模式: 不声明输入输出
tasks.register('processData') {
    doLast {
        def input = file('data.csv').text
        def output = input.toUpperCase()
        file('build/output.csv').text = output
    }
    // 每次都会执行，即使没有任何变化
}

// ✅ 推荐: 明确声明输入输出
abstract class ProcessDataTask extends DefaultTask {

    @InputFile
    @PathSensitive(PathSensitivity.RELATIVE)
    abstract RegularFileProperty getInputFile()

    @OutputFile
    abstract RegularFileProperty getOutputFile()

    @TaskAction
    void process() {
        String input = inputFile.get().asFile.text
        String output = input.toUpperCase()
        outputFile.get().asFile.text = output
    }
}
```

## 6.3 设计可增量任务

```groovy
abstract class IncrementalSyntaxCheckTask extends DefaultTask {

    @InputFiles
    @Incremental
    @PathSensitive(PathSensitivity.RELATIVE)
    abstract ConfigurableFileCollection getSourceFiles()

    @OutputDirectory
    abstract DirectoryProperty getReportDir()

    @TaskAction
    void check(InputChanges changes) {
        if (!changes.isIncremental()) {
            project.delete(reportDir.get().asFile)
        }

        changes.getFileChanges(sourceFiles).each { fileChange ->
            File target = new File(reportDir.get().asFile,
                                   fileChange.normalizedPath + '.report')

            switch (fileChange.changeType) {
                case ADDED:
                case MODIFIED:
                    target.text = analyzeFile(fileChange.file)
                    break
                case REMOVED:
                    target.delete()
                    break
            }
        }
    }

    private String analyzeFile(File file) {
        def errors = []
        file.eachLine { line, index ->
            if (line.contains('TODO')) {
                errors << "行 ${index + 1}: 包含 TODO"
            }
        }
        return errors.join('\n')
    }
}
```

# 7. 性能优化

## 7.1 并行执行

```bash
# 启用并行执行
gradle build --parallel

# 在 gradle.properties 中永久启用
# org.gradle.parallel=true

# 控制最大工作线程数
# org.gradle.workers.max=4
```

```properties
# gradle.properties - 性能优化配置
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true

# 守护进程配置
org.gradle.daemon=true
org.gradle.jvmargs=-Xmx2g -XX:MaxMetaspaceSize=512m -XX:+HeapDumpOnOutOfMemoryError

# 按需配置
org.gradle.configureondemand=true
```

## 7.2 配置缓存

```groovy
// settings.gradle
// 配置缓存将配置阶段的结果缓存起来，大幅减少构建时间
buildCache {
    local {
        enabled = true
    }
}

// 启用配置缓存（Gradle 8.x+）
// 在 gradle.properties 中设置:
// org.gradle.configuration-cache=true
```

**配置缓存注意事项：**

```groovy
// ❌ 配置缓存不兼容的代码
task myTask {
    doLast {
        // 在配置阶段使用 System.currentTimeMillis()
        def timestamp = System.currentTimeMillis()
    }
}

// ✅ 配置缓存兼容的代码
abstract class MyTask extends DefaultTask {

    @Input
    @Optional
    abstract Property<Long> getTimestamp()

    @TaskAction
    void execute() {
        // 在执行阶段获取时间
        if (!timestamp.isPresent()) {
            timestamp.set(System.currentTimeMillis())
        }
    }
}
```

## 7.3 守护进程优化

```bash
# 查看守护进程状态
gradle --status

# 停止守护进程
gradle --stop

# 强制使用守护进程
gradle build --daemon

# 不使用守护进程（CI 环境）
gradle build --no-daemon
```

```properties
# gradle.properties - 守护进程优化
org.gradle.daemon.idletimeout=3600000   # 1 小时空闲超时 (毫秒)
org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=1g
org.gradle.jvmargs=-XX:+UseParallelGC
org.gradle.jvmargs=-XX:TieredStopAtLevel=1  # 加速 JVM 启动
```

# 8. 多项目构建

## 8.1 复合构建

```groovy
// settings.gradle
rootProject.name = 'my-app'

// 标准包含
include 'core', 'web', 'cli'

// 复合构建：包含外部项目
includeBuild '../shared-conventions'
includeBuild '../common-utils'

// 使用复合构建的项目
dependencies {
    implementation 'com.example:common-utils'
    // common-utils 来自复合构建，直接使用本地源码编译
}
```

## 8.2 约定插件 vs buildSrc

```
选择指南
┌─────────────────────────────────────────────────────┐
│                                                     │
│  buildSrc                                           │
│  ├── 适合: 单个项目或 monorepo                     │
│  ├── 优势: 自动编译，无需发布                       │
│  └── 劣势: 修改后整个项目重新配置                   │
│                                                     │
│  published convention plugins (独立仓库)            │
│  ├── 适合: 多个项目共用                             │
│  ├── 优势: 版本化管理，跨项目共享                   │
│  └── 劣势: 需要发布流程                             │
│                                                     │
│  included build (复合构建)                          │
│  ├── 适合: 开发和测试期间的插件                     │
│  ├── 优势: 无需发布即可共享                         │
│  └── 劣势: 额外配置                                 │
└─────────────────────────────────────────────────────┘
```

## 8.3 跨项目配置

```groovy
// settings.gradle
gradle.beforeProject { project ->
    if (project != rootProject) {
        project.apply plugin: 'java'
        // 在所有子项目中应用基础配置
    }
}

// 所有子项目的通用配置
subprojects {
    apply plugin: 'java'

    group = 'com.example'
    version = '1.0.0'

    java {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    repositories {
        mavenCentral()
    }
}
```

# 9. 测试最佳实践

## 9.1 集成测试分离

```groovy
// 将集成测试与单元测试分离
sourceSets {
    integrationTest {
        compileClasspath += main.output + test.output
        runtimeClasspath += main.output + test.output
    }
}

configurations {
    integrationTestImplementation.extendsFrom testImplementation
    integrationTestRuntimeOnly.extendsFrom testRuntimeOnly
    integrationTestCompileOnly.extendsFrom testCompileOnly
}

tasks.register('integrationTest', Test) {
    description = '执行集成测试'
    group = 'verification'

    testClassesDirs = sourceSets.integrationTest.output.classesDirs
    classpath = sourceSets.integrationTest.runtimeClasspath

    shouldRunAfter test
    useJUnitPlatform()

    maxParallelForks = 2
}

tasks.named('check') {
    dependsOn integrationTest
}
```

## 9.2 测试并行化

```groovy
tasks.withType(Test).configureEach {
    // 在每个测试任务内并行
    maxParallelForks = Runtime.runtime.availableProcessors().intdiv(2) ?: 1

    // JUnit 平台配置
    useJUnitPlatform {
        // 启用并行测试执行
        systemProperty 'junit.jupiter.execution.parallel.enabled', 'true'
        systemProperty 'junit.jupiter.execution.parallel.mode.default', 'concurrent'
        systemProperty 'junit.jupiter.execution.parallel.config.strategy', 'dynamic'
    }

    // 配置 JVM 参数
    jvmArgs '-Xmx1g', '-XX:+UseG1GC'

    // 在 CI 环境增加日志
    if (System.getenv('CI')) {
        testLogging {
            events 'passed', 'skipped', 'failed'
            exceptionFormat 'full'
        }
    }
}
```

## 9.3 测试报告

```groovy
// 聚合所有子项目的测试报告
tasks.register('testAggregateReport', TestReport) {
    destinationDirectory = layout.buildDirectory.dir('reports/tests/aggregate')
    testResults.from subprojects*.tasks*.withType(Test)*.binaryResultsDirectory
}
```

# 10. CI/CD 集成

## 10.1 Wrapper 最佳实践

```bash
# 生成 Wrapper（应提交到版本控制）
gradle wrapper --gradle-version 8.7

# 更新 Wrapper
gradle wrapper --gradle-version 8.7 --distribution-type all

# 验证 Wrapper JAR（防止供应链攻击）
gradle wrapper --gradle-distribution-sha256-sum <checksum>
```

```properties
# gradle/wrapper/gradle-wrapper.properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.7-bin.zip
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

## 10.2 CI 环境配置

```groovy
// 检测 CI 环境并调整行为
def isCi = System.getenv('CI')?.toBoolean() ?: false

tasks.withType(Test).configureEach {
    // CI 环境: 更多 fork, 更详细的日志
    maxParallelForks = isCi ? 1 : Runtime.runtime.availableProcessors()

    testLogging {
        events 'passed', 'skipped', 'failed'
        showStandardStreams = isCi
    }

    // CI 上不缓存测试结果
    outputs.upToDateWhen { !isCi }
}
```

# 11. 安全性最佳实践

## 11.1 凭据处理

```groovy
// ❌ 反模式: 硬编码凭据
repositories {
    maven {
        url = uri('https://private.repo.com/releases')
        credentials {
            username = 'admin'           // 不安全！
            password = 'secret123456'    // 严重安全隐患！
        }
    }
}

// ✅ 推荐: 使用环境变量或 Gradle 属性
repositories {
    maven {
        url = uri('https://private.repo.com/releases')
        credentials {
            username = System.getenv('REPO_USER')
                ?: project.findProperty('repo.username')
            password = System.getenv('REPO_PASS')
                ?: project.findProperty('repo.password')
        }
    }
}

// ✅ 最佳: 使用 gradle.properties 并加入 .gitignore
// ~/.gradle/gradle.properties (全局)
// repo.username=myuser
// repo.password=mypassword

// 或使用系统环境变量
// export REPO_USER=myuser
// export REPO_PASS=mypassword
```

## 11.2 .gitignore 配置

```gitignore
# Gradle
.gradle/
build/
!gradle/wrapper/gradle-wrapper.jar

# IDE
.idea/
*.iml
*.iws
*.ipr
.vscode/

# 凭据
gradle.properties        # 如果包含凭据
local.properties
*.env

# 敏感数据
**/secrets/
**/credentials.json
```

## 11.3 签名验证

```groovy
// 启用依赖签名验证
dependencies {
    components {
        // 验证特定模块的签名
        withModule('com.example:critical-lib') {
            // Gradle 会自动验证 PGP 签名
        }
    }
}

// 配置签名验证策略
dependencyVerification {
    // 提供验证元数据文件
    verify = [
        'com.google.guava:guava:33.0.0-jre': '...',
        'org.apache.commons:commons-lang3:3.14.0': '...',
    ]
}
```

# 12. 脚本维护

## 12.1 废弃处理

```groovy
// 标记废弃的任务
abstract class OldTask extends DefaultTask {

    OldTask() {
        description = '此任务已废弃，请使用 newTask'

        doNotTrackState('任务已废弃')

        doLast {
            logger.warn('警告: oldTask 已废弃，将在 2.0 版本移除')
        }
    }
}

// 废弃插件
plugins {
    id 'java'
    // id 'old-plugin'  // 已被弃用，使用新插件替代
}

// 使用 Gradle 的废弃警告
tasks.register('migrate') {
    doLast {
        // 使用 @Deprecated 注解标记 API
        project.logger.warn('此 API 已废弃')
    }
}
```

## 12.2 渐进式升级

```groovy
// 使用 Gradle 版本兼容性检查
if (GradleVersion.current() >= GradleVersion.version('8.0')) {
    // 使用新 API
    tasks.named('test', Test) {
        useJUnitPlatform()
    }
} else {
    // 向后兼容
    tasks.withType(Test) {
        useJUnitPlatform()
    }
}

// Feature Preview API
// enableFeaturePreview('STABLE_CONFIGURATION_CACHE')
// enableFeaturePreview('TYPESAFE_PROJECT_ACCESSORS')
```

## 12.3 构建脚本健康检查

```bash
# 检查构建脚本问题
gradle build --warning-mode all

# 检查废弃 API 使用
gradle build --warning-mode fail

# 查看完整堆栈
gradle build --stacktrace

# 构建扫描（推荐）
gradle build --scan
```

# 13. 代码质量插件

## 13.1 Checkstyle

```groovy
plugins {
    id 'checkstyle'
}

checkstyle {
    toolVersion = '10.14.2'
    configDirectory = rootProject.layout.projectDirectory.dir('config/checkstyle')
    maxErrors = 0
    maxWarnings = 0

    // 排除特定文件
    // configProperties = [
    //     'suppressionFile': project(':').file('config/checkstyle/suppressions.xml')
    // ]
}

// 配置 Checkstyle 任务
tasks.withType(Checkstyle).configureEach {
    reports {
        xml.required = true  // CI 需要
        html.required = !System.getenv('CI')
    }
}
```

## 13.2 SpotBugs (FindBugs 继任者)

```groovy
plugins {
    id 'com.github.spotbugs' version '6.0.14'
}

spotbugs {
    toolVersion = '4.8.5'
    ignoreFailures = false
    showStackTraces = true
    showProgress = false

    // 排除过滤器
    excludeFilter = rootProject.file('config/spotbugs/exclude.xml')
}

tasks.withType(com.github.spotbugs.snom.SpotBugsTask).configureEach {
    reports {
        xml.required = true
        html.required = !System.getenv('CI')
    }
}
```

## 13.3 PMD

```groovy
plugins {
    id 'pmd'
}

pmd {
    toolVersion = '7.0.0'
    ignoreFailures = false
    ruleSetFiles = rootProject.files('config/pmd/rules.xml')
    ruleSets = []  // 禁用默认规则
    consoleOutput = true
    incrementalAnalysis = true
}

tasks.withType(Pmd).configureEach {
    reports {
        xml.required = true
        html.required = !System.getenv('CI')
    }
}
```

# 14. 可重复构建

## 14.1 消除非确定性

```groovy
// 确保输出文件的时间戳一致
tasks.withType(AbstractArchiveTask).configureEach {
    preserveFileTimestamps = false
    reproducibleFileOrder = true
}

// 标准化 ZIP 条目的顺序和时间戳
tasks.withType(Zip).configureEach {
    reproducibleFileOrder = true
    preserveFileTimestamps = false
}

// JAR 文件标准化
tasks.withType(Jar).configureEach {
    reproducibleFileOrder = true
    preserveFileTimestamps = false

    manifest {
        attributes(
            'Build-Jdk': "${System.getProperty('java.version')}",
            'Created-By': "Gradle ${gradle.gradleVersion}",
            // 不包含 Build-Timestamp 避免不可重复
        )
    }
}
```

## 14.2 标准化配置

```groovy
normalization {
    runtimeClasspath {
        ignore 'META-INF/MANIFEST.MF'
        ignore 'META-INF/*.SF'
        ignore 'META-INF/*.DSA'
        ignore 'META-INF/*.RSA'
    }
}
```

## 14.3 依赖锁定

```groovy
dependencyLocking {
    lockAllConfigurations()
}
```

# 15. 总结

| 类别 | 最佳实践 | 反模式 |
|------|----------|--------|
| 项目组织 | 单一职责文件、提取约定插件 | `allprojects` 混合配置 |
| 配置避免 | `register()` + Provider | `create()` + 硬编码值 |
| 依赖管理 | 版本目录、BOM、锁定 | 动态版本、无版本声明 |
| 任务设计 | 输入/输出注解、增量支持 | 无输入/输出声明 |
| 性能 | 并行、缓存、守护进程 | 全量构建、忽略缓存 |
| 测试 | 单元/集成分离、并行化 | 混在一起、串行执行 |
| 安全性 | 环境变量凭据、Wrapper | 硬编码密码、无 Wrapper |
| 可重复性 | 时间戳标准化、依赖锁定 | 动态版本、非确定性输出 |

遵循这些最佳实践，能够构建出高效、可维护、安全且可跨环境复制的 Gradle 项目。
