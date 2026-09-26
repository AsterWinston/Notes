
Gradle 提供了强大且灵活的测试框架集成能力。通过 `Test` 任务类型，可以统一管理 JUnit、TestNG、Spock 等各类测试框架的执行、报告和过滤。

# 1. Test 任务深度解析

## 1.1 Test 任务与测试框架检测

Gradle 的 `Test` 任务（`org.gradle.api.tasks.testing.Test`）继承自 `AbstractTestTask`，它通过类路径扫描自动检测测试框架：

```
测试框架检测优先级（Gradle 内置的 TestFrameworkDetector）：
    1. 检测 testRuntimeClasspath 中的框架 JAR
    2. 按优先级尝试匹配：
       ├── JUnit Platform (JUnit 5) — 存在 junit-platform-engine
       ├── TestNG — 存在 testng
       ├── JUnit 4 — 存在 junit (且无 JUnit 5)
       └── Spock — 存在 spock-core
    3. 使用匹配到的 TestFramework 实例运行全部测试
```

```groovy
// build.gradle — 基本测试配置
dependencies {
    // JUnit 5 (JUnit Platform)
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
    
    // 或者 JUnit 4
    // testImplementation 'junit:junit:4.13.2'
    
    // 或者 TestNG
    // testImplementation 'org.testng:testng:7.9.0'
}

test {
    // 强制使用特定框架（一般不需要）
    // useJUnit()
    // useJUnitPlatform()
    // useTestNG()
}
```

## 1.2 框架选择与 useJUnitPlatform 详解

`Test.useJUnitPlatform()` 是 JUnit 5 的标准配置方式：

```groovy
test {
    useJUnitPlatform {
        // 包含/排除引擎
        includeEngines 'junit-jupiter', 'junit-vintage'
        excludeEngines 'custom-engine'
        
        // 包含/排除标签
        includeTags 'fast', 'integration'
        excludeTags 'slow', 'manual'
        
        // 配置属性（传递给 JUnit Platform）
        configurationParameter 'junit.jupiter.execution.parallel.enabled', 'true'
        configurationParameter 'junit.jupiter.execution.parallel.mode.default', 'concurrent'
        configurationParameter 'junit.jupiter.execution.parallel.mode.classes.default', 'concurrent'
    }
}
```

JUnit 5 引擎架构：

```
┌────────────────────────────────────────────┐
│              JUnit Platform Launcher        │
│         (org.junit.platform.launcher)      │
├────────────────────────────────────────────┤
│  TestEngine SPI                            │
├──────────────┬──────────────┬──────────────┤
│ Jupiter      │ Vintage       │ 自定义引擎    │
│ (JUnit 5)    │ (JUnit 4/3)   │ (如 Spek,   │
│              │               │  Kotest)    │
├──────────────┴──────────────┴──────────────┤
│         Gradle Test 任务                    │
└────────────────────────────────────────────┘
```

## 1.3 TestNG 专用配置

```groovy
test {
    useTestNG {
        // 指定 suites 文件
        suites 'src/test/resources/testng.xml'
        
        // 并行模式
        parallel 'methods'
        threadCount 4
        
        // 包含/排除分组
        includeGroups 'unit', 'integration'
        excludeGroups 'performance'
        
        // 预设（用于选择测试）
        // preserveOrder true
        
        // 输出目录
        // outputDirectory = file("$buildDir/testng-output")
    }
}
```

## 1.4 Spock 框架配置

```groovy
dependencies {
    testImplementation platform('org.spockframework:spock-bom:2.4-M1-groovy-4.0')
    testImplementation 'org.spockframework:spock-core'
    testImplementation 'org.apache.groovy:groovy-all:4.0.18'
}

test {
    useJUnitPlatform()  // Spock 2.x 在 JUnit Platform 上运行
    
    testLogging {
        events "passed", "skipped", "failed"
        showStandardStreams = true
    }
}
```

# 2. 测试过滤

## 2.1 DSL 级别的包含/排除

```groovy
test {
    // 按类名模式过滤
    filter {
        // 包含匹配的测试类
        includeTestsMatching "*ServiceTest"
        includeTestsMatching "com.example.service.*Test"
        
        // 排除匹配的测试类
        excludeTestsMatching "*IntegrationTest"
        excludeTestsMatching "com.example.slow.*"
    }
}
```

按类名和方法名的精细过滤：

```groovy
test {
    filter {
        // 指定具体的测试类
        includeTestsMatching "com.example.UserServiceTest"
        
        // 方法级别过滤
        includeTestsMatching "com.example.UserServiceTest.testCreate*"
        
        // 使用正则
        includeTestsMatching "com.example.*Test.test*"
    }
}
```

Kotlin DSL：

```kotlin
tasks.test {
    filter {
        includeTestsMatching("com.example.service.*Test")
        excludeTestsMatching("*IntegrationTest")
    }
}
```

## 2.2 --tests 命令行过滤

```bash
# 运行单个测试类
./gradlew test --tests "com.example.UserServiceTest"

# 运行类中的单个方法
./gradlew test --tests "com.example.UserServiceTest.testCreateUser"

# 使用通配符
./gradlew test --tests "com.example.*Test"

# 通配符匹配方法名
./gradlew test --tests "com.example.UserServiceTest.testCreate*"

# 运行所有匹配"ServiceTest"的测试
./gradlew test --tests "*ServiceTest"

# 多级包匹配
./gradlew test --tests "com.example.**.service.*Test"

# 排除特定测试
./gradlew test --tests "*" --tests "!*IntegrationTest"

# 方法名精确匹配（区分大小写）
./gradlew test --tests "com.example.UserServiceTest.testCreateUser" --exact
```

## 2.3 TestFilter 高级用法

通过 `--tests` 的完整语法：

```bash
# 多维度组合过滤
./gradlew test \
  --tests "com.example.unit.*Test" \
  --tests "com.example.integration.*Test" \
  --tests "!com.example.slow.*"

# 通过系统属性控制
./gradlew test -Dtest.single=UserServiceTest              # JUnit 4
./gradlew test -Dtest.include=UserServiceTest             # TestNG
./gradlew test -Dtest.include=*ServiceTest                # 通配符
```

在 `build.gradle` 中使用 `TestFilter` 接口：

```groovy
test {
    filter {
        // 包含包含特定注解的测试
        setIncludePatterns('*Test', '*Tests', '*Spec')
        
        // 命令行动态覆盖
        if (project.hasProperty('testSingle')) {
            includeTestsMatching "*${project.testSingle}*"
        }
        
        // 失败时仅重新运行失败的测试
        failFast = true  // 首次失败即停止
    }
}
```

# 3. 测试分组与套件

## 3.1 JUnit 5 标签与分类

JUnit 5 中使用 `@Tag` 注解：

```java
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

@Tag("unit")
@Tag("fast")
class UserServiceTest {
    
    @Test
    @Tag("database")
    void testSaveUser() {
        // ...
    }
    
    @Test
    @Tag("validation")
    void testValidateEmail() {
        // ...
    }
    
    @Test
    @Tag("slow")
    void testComplexWorkflow() {
        // ...
    }
}
```

Gradle 中按标签执行：

```groovy
// 定义不同的测试任务，按标签分组
tasks.register('unitTest', Test) {
    description = "Runs unit tests (tagged 'unit')"
    group = "verification"
    
    useJUnitPlatform {
        includeTags 'unit'
        excludeTags 'slow', 'integration'
    }
    testClassesDirs = sourceSets.test.output.classesDirs
    classpath = sourceSets.test.runtimeClasspath
}

tasks.register('integrationTest', Test) {
    description = "Runs integration tests (tagged 'integration')"
    group = "verification"
    
    useJUnitPlatform {
        includeTags 'integration'
    }
    testClassesDirs = sourceSets.test.output.classesDirs
    classpath = sourceSets.test.runtimeClasspath
    
    shouldRunAfter test
}

tasks.check { dependsOn integrationTest }
```

## 3.2 Gradle 测试套件（Test Suites — Gradle 7.3+）

使用 `jvm-test-suite` 插件：

```groovy
plugins {
    id 'java'
    id 'jvm-test-suite'  // 内置插件
}

testing {
    suites {
        // 默认 test 套件（自动创建）
        test {
            useJUnitJupiter()
            targets {
                all {
                    testTask.configure {
                        // 这里可以使用 test 任务的完整 DSL
                    }
                }
            }
        }
        
        // 自定义集成测试套件
        integrationTest(JvmTestSuite) {
            dependencies {
                implementation project()
                // 自动添加主源码集的输出到 classpath
            }
            
            targets {
                all {
                    testTask.configure {
                        shouldRunAfter(test)
                    }
                }
            }
        }
        
        // 性能测试套件
        performanceTest(JvmTestSuite) {
            testType = TestSuiteType.PERFORMANCE_TEST
            
            dependencies {
                implementation project()
                implementation 'org.openjdk.jmh:jmh-core:1.37'
                implementation 'org.openjdk.jmh:jmh-generator-annprocess:1.37'
            }
            
            targets {
                all {
                    testTask.configure {
                        useJUnitPlatform()
                        maxHeapSize = '4g'
                    }
                }
            }
        }
    }
}
```

测试套件自动生成的源码目录：

```
src/
├── main/java/...
├── test/java/...              # test 套件
├── integrationTest/java/...   # integrationTest 套件
└── performanceTest/java/...   # performanceTest 套件
```

## 3.3 JUnit 4 套件（传统方式）

```java
import org.junit.runner.RunWith;
import org.junit.runners.Suite;

@RunWith(Suite.class)
@Suite.SuiteClasses({
    UserServiceTest.class,
    OrderServiceTest.class,
    PaymentServiceTest.class
})
public class ServiceTestSuite {
    // 空类，仅用于套件定义
}
```

# 4. 并行测试执行

## 4.1 maxParallelForks 与 forkEvery

```groovy
test {
    // 并行分叉（最大 JVM 进程数）
    maxParallelForks = 4  // 同时运行最多 4 个测试 JVM
    
    // 每个分叉执行多少个测试类后重启 JVM
    forkEvery = 50  // 每 50 个测试类后新建 JVM（避免内存泄漏）
    
    // JVM 分叉配置
    forkOptions {
        // 额外的 JVM 参数
        jvmArgs '-Xmx2g', '-XX:MaxMetaspaceSize=512m'
        
        // 环境变量
        environment 'TEST_ENV', 'ci'
        
        // 工作目录
        // workingDir = file("$buildDir/test-work")
    }
    
    // 最大堆内存（Gradle 7.0+）
    maxHeapSize = '2g'
    
    // 最小堆内存
    minHeapSize = '512m'
}
```

并行执行原理图解：

```
                     ┌─────────────────┐
                     │   Gradle Daemon  │
                     │   (主进程)        │
                     └────────┬────────┘
                              │ 调度测试类
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
   ┌───────────┐       ┌───────────┐       ┌───────────┐
   │ Fork 1    │       │ Fork 2    │       │ Fork N    │
   │ JVM 进程   │       │ JVM 进程   │       │ JVM 进程   │
   ├───────────┤       ├───────────┤       ├───────────┤
   │ TestA.class│      │ TestC.class│      │ TestE.class│
   │ TestB.class│      │ TestD.class│      │ TestF.class│
   └───────────┘       └───────────┘       └───────────┘
```

## 4.2 JUnit 5 自身并行配置

```groovy
test {
    useJUnitPlatform {
        configurationParameter 'junit.jupiter.execution.parallel.enabled', 'true'
        configurationParameter 'junit.jupiter.execution.parallel.mode.default', 'same_thread'
        configurationParameter 'junit.jupiter.execution.parallel.mode.classes.default', 'concurrent'
        configurationParameter 'junit.jupiter.execution.parallel.config.strategy', 'fixed'
        configurationParameter 'junit.jupiter.execution.parallel.config.fixed.parallelism', '4'
    }
}
```

配置组合策略：

```properties
# junit-platform.properties（src/test/resources/）
junit.jupiter.execution.parallel.enabled = true
junit.jupiter.execution.parallel.mode.default = concurrent
junit.jupiter.execution.parallel.config.strategy = dynamic
junit.jupiter.execution.parallel.config.dynamic.factor = 2.0
# factor=2.0 表示线程数 = CPU 核心数 × 2
```

## 4.3 parallel 示例完整配置

```groovy
// gradle.properties
org.gradle.parallel=true
org.gradle.workers.max=4

// build.gradle
test {
    maxParallelForks = Runtime.runtime.availableProcessors().intdiv(2) ?: 1
    forkEvery = 0  // 0 表示不重启（适合无内存泄漏的测试）
    
    systemProperty 'junit.jupiter.execution.parallel.enabled', 'true'
    systemProperty 'junit.jupiter.execution.parallel.mode.default', 'concurrent'
}
```

# 5. 测试排序

## 5.1 Gradle 级测试执行顺序

```groovy
test {
    // 指定测试执行顺序器
    testExecutionOrder = TestExecutionOrder.RANDOM  // 随机（默认）

    // 按类名字母序排序
    // testExecutionOrder = TestExecutionOrder.SORTED_BY_CLASS_NAME
    
    // 按方法名字母序排序
    // testExecutionOrder = TestExecutionOrder.SORTED_BY_METHOD_NAME
}
```

## 5.2 JUnit 5 测试排序

```java
import org.junit.jupiter.api.MethodOrderer;
import org.junit.jupiter.api.Order;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestMethodOrder;

// 方式一：按 @Order 注解
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderedTest {
    @Test
    @Order(1)
    void firstTest() { }
    
    @Test
    @Order(2)
    void secondTest() { }
    
    @Test
    @Order(3)
    void thirdTest() { }
}

// 方式二：随机排序（检测隐式依赖）
@TestMethodOrder(MethodOrderer.Random.class)
class RandomTest {
    @Test
    void testA() { }
    @Test
    void testB() { }
}

// 方式三：按方法名字母序
@TestMethodOrder(MethodOrderer.MethodName.class)
class AlphabeticalTest { }
```

# 6. 测试日志

## 6.1 testLogging DSL

```groovy
test {
    testLogging {
        // 显示哪些事件
        events "passed", "skipped", "failed", "standardOut", "standardError"
        // 简写：events "passed", "skipped", "failed"
        
        // 异常格式（short, full, none）
        exceptionFormat = org.gradle.api.tasks.testing.logging.TestExceptionFormat.FULL
        
        // 是否显示标准输出/错误
        showStandardStreams = true
        
        // 是否显示异常堆栈跟踪
        showExceptions = true
        
        // 是否显示测试失败的原因
        showCauses = true
        
        // 是否显示堆栈跟踪
        showStackTraces = true
        
        // 特定日志级别的配置
        debug {
            events "started", "skipped", "failed", "passed", "standardOut", "standardError"
            exceptionFormat = org.gradle.api.tasks.testing.logging.TestExceptionFormat.FULL
        }
        
        info {
            events = ["failed", "skipped"]
        }
        
        // 输出摘要
        afterSuite { descriptor, result ->
            if (!descriptor.parent) { // 根套件
                println "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
                println "测试结果: ${result.resultType}"
                println "  总计: ${result.testCount}"
                println "  通过: ${result.successfulTestCount}"
                println "  失败: ${result.failedTestCount}"
                println "  跳过: ${result.skippedTestCount}"
                println "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
            }
        }
        
        beforeSuite { descriptor ->
            if (descriptor.className != null) {
                logger.lifecycle("▶ 开始测试类: ${descriptor.className}")
            }
        }
        
        afterTest { descriptor, result ->
            if (result.resultType == TestResult.ResultType.FAILURE) {
                logger.lifecycle("✗ FAILED: ${descriptor.className}.${descriptor.name}")
            }
        }
    }
}
```

## 6.2 带颜色的测试输出

```groovy
test {
    testLogging {
        afterTest { descriptor, result ->
            def color = result.resultType == TestResult.ResultType.SUCCESS ? '\u001B[32m' : '\u001B[31m'
            def reset = '\u001B[0m'
            logger.lifecycle("${color}${descriptor.name}: ${result.resultType}${reset}")
        }
    }
}
```

## 6.3 命令行日志控制

```bash
# 显示完整异常
./gradlew test --info

# 调试模式（最详细）
./gradlew test --debug

# 控制台输出模式
./gradlew test --console=verbose

# 显示标准输出
./gradlew test -i  # --info 的简写
```

# 7. 测试报告

## 7.1 HTML 报告

Gradle 默认在 `$buildDir/reports/tests/test/index.html` 生成 HTML 报告：

```groovy
test {
    reports {
        // JUnit XML 报告（默认启用，供 CI 工具使用）
        junitXml {
            required = true
            outputLocation = layout.buildDirectory.dir("test-results/test")
            // 合并多个执行周期的结果
            mergeReruns = true  // Gradle 7.6+
        }
        
        // HTML 报告
        html {
            required = true
            outputLocation = layout.buildDirectory.dir("reports/tests/test")
        }
    }
}
```

报告目录结构：

```
build/
├── reports/tests/test/
│   ├── index.html          # 总览页面
│   ├── classes/
│   │   └── com.example.UserServiceTest.html
│   ├── packages/
│   │   └── com.example.html
│   └── css/ js/
└── test-results/test/
    ├── TEST-com.example.UserServiceTest.xml   # JUnit XML
    └── TEST-com.example.OrderServiceTest.xml
```

## 7.2 JUnit XML 报告格式

JUnit XML 是 CI 工具（Jenkins、GitLab、GitHub Actions）解析测试结果的标准格式：

```xml
<!-- TEST-com.example.UserServiceTest.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<testsuite name="com.example.UserServiceTest"
           tests="5" failures="1" errors="0" skipped="1"
           time="3.451" timestamp="2024-05-15T10:30:00">
    <testcase name="testCreateUser"
              classname="com.example.UserServiceTest"
              time="0.823"/>
    <testcase name="testDeleteUser"
              classname="com.example.UserServiceTest"
              time="0.456">
        <failure message="Expected user to be deleted"
                 type="org.opentest4j.AssertionFailedError">
            org.opentest4j.AssertionFailedError: Expected user to be deleted
                at com.example.UserServiceTest.testDeleteUser(UserServiceTest.java:45)
        </failure>
    </testcase>
    <testcase name="testUpdateUser"
              classname="com.example.UserServiceTest"
              time="0.0">
        <skipped message="Not yet implemented"/>
    </testcase>
</testsuite>
```

## 7.3 CI 平台集成示例

GitHub Actions 集成：

```yaml
- name: Test
  run: ./gradlew test

- name: Publish Test Results
  uses: dorny/test-reporter@v1
  if: success() || failure()
  with:
    name: Gradle Test Results
    path: '**/build/test-results/test/TEST-*.xml'
    reporter: java-junit
```

Jenkins Pipeline 集成：

```groovy
pipeline {
    stages {
        stage('Test') {
            steps {
                sh './gradlew test'
            }
            post {
                always {
                    junit '**/build/test-results/test/TEST-*.xml'
                    publishHTML(target: [
                        reportName: 'Gradle Test Report',
                        reportDir: 'build/reports/tests/test',
                        reportFiles: 'index.html'
                    ])
                }
            }
        }
    }
}
```

# 8. 代码覆盖率 — JaCoCo 深度解析

## 8.1 JaCoCo 插件配置

```groovy
plugins {
    id 'java'
    id 'jacoco'
}

jacoco {
    // 指定 JaCoCo 版本
    toolVersion = "0.8.12"
    
    // 报告输出目录
    reportsDirectory = layout.buildDirectory.dir("customJacocoReportDir")
}

// 版本目录（使用 TOML 版本管理）
// jacoco { toolVersion = libs.versions.jacoco.get() }
```

## 8.2 Test 任务与 JaCoCo 集成

```groovy
test {
    // 启用 JaCoCo 检测（默认自动启用）
    finalizedBy jacocoTestReport
}

jacocoTestReport {
    // 依赖 test 任务
    dependsOn test
    
    reports {
        // XML 报告（供 SonarQube、Codecov 等使用）
        xml {
            required = true
            outputLocation = layout.buildDirectory.file("reports/jacoco/test/jacocoTestReport.xml")
        }
        
        // CSV 报告（供 Excel 等使用）
        csv {
            required = true
            outputLocation = layout.buildDirectory.file("reports/jacoco/test/jacocoTestReport.csv")
        }
        
        // HTML 报告（人工阅读）
        html {
            required = true
            outputLocation = layout.buildDirectory.dir("reports/jacoco/test/html")
        }
    }
    
    // 自定义报告标题和页脚
    // html.destination = file("...")
    
    // 排除某些类的覆盖率统计
    classDirectories.setFrom(
        files(classDirectories.files.collect {
            fileTree(dir: it, exclude: [
                '**/*Application*',
                '**/dto/**',
                '**/config/**',
                '**/*Exception*',
                '**/generated/**',
                '**/Q*.class'   // Querydsl 生成类
            ])
        })
    )
    
    // 排除某些类的执行数据
    // executionData.setFrom(files(executionData.files.collect { ... }))
}

jacocoTestCoverageVerification {
    // 覆盖率规则
    violationRules {
        rule {
            // 限定到特定元素
            element = 'CLASS'
            
            // 包含/排除
            includes = ['com.example.service.*']
            excludes = ['com.example.service.ConfigurationService']
            
            // 覆盖率阈值
            limit {
                counter = 'LINE'
                value = 'COVEREDRATIO'
                minimum = 0.80  // 80% 行覆盖率
            }
            
            limit {
                counter = 'BRANCH'
                value = 'COVEREDRATIO'
                minimum = 0.70  // 70% 分支覆盖率
            }
            
            limit {
                counter = 'METHOD'
                value = 'COVEREDRATIO'
                minimum = 0.90  // 90% 方法覆盖率
            }
        }
        
        // 对特定包的规则
        rule {
            element = 'PACKAGE'
            includes = ['com.example.controller.*']
            limit {
                counter = 'LINE'
                value = 'COVEREDRATIO'
                minimum = 0.90
            }
        }
    }
}

// 将覆盖率验证加入 check 生命周期
check.dependsOn jacocoTestCoverageVerification
```

JaCoCo 计数器类型说明：

| Counter | 说明 | 示例 |
|---------|------|------|
| `LINE` | 代码行 | 第 10-15 行被覆盖 |
| `BRANCH` | 分支（if/else, switch） | `if (x > 0)` 的 true/false 分支 |
| `INSTRUCTION` | 字节码指令（最底层） | 每个字节码是否被执行 |
| `METHOD` | 方法 | `public void doWork()` 是否被调用 |
| `CLASS` | 类 | `UserService` 是否被测试 |

## 8.3 多任务覆盖率聚合

```groovy
// 场景：分别有单元测试和集成测试，需要合并覆盖率

tasks.register('integrationTest', Test) {
    description = 'Runs integration tests.'
    group = 'verification'
    testClassesDirs = sourceSets.integrationTest.output.classesDirs
    classpath = sourceSets.integrationTest.runtimeClasspath
}

tasks.register('jacocoIntegrationTestReport', JacocoReport) {
    description = 'Generates code coverage report for integration tests.'
    executionData integrationTest
    sourceSets sourceSets.main
    reports {
        xml.required = true
        html.required = true
    }
}

// 合并单元测试和集成测试覆盖率
tasks.register('jacocoMergedReport', JacocoReport) {
    dependsOn test, integrationTest
    executionData test, integrationTest
    sourceSets sourceSets.main
    reports {
        xml.required = true
        html.required = true
    }
    
    // 必须设置 class directories
    classDirectories.setFrom(files(
        sourceSets.main.output.classesDirs.collected { 
            fileTree(it).exclude('**/dto/**', '**/config/**') 
        }
    ))
}
```

## 8.4 JaCoCo 与 Kotlin

```kotlin
// build.gradle.kts — Kotlin DSL
tasks.jacocoTestReport {
    dependsOn(tasks.test)
    
    reports {
        xml.required.set(true)
        html.required.set(true)
        csv.required.set(false)
    }
    
    classDirectories.setFrom(
        sourceSets.main.get().output.asFileTree.matching {
            exclude(
                "**/*Application*",
                "**/dto/**",
                "**/config/**",
                "**/generated/**"
            )
        }
    )
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = BigDecimal.valueOf(0.80)
            }
        }
    }
}
```

# 9. 测试重试

## 9.1 Test Retry Gradle 插件

从 Gradle 7.3 开始，内置了测试重试功能：

```groovy
plugins {
    id 'java'
    id 'org.gradle.test-retry' version '1.5.8'  // 第三方插件方式
}

// Gradle 7.3+ 内置方式（不需要额外插件）
test {
    retry {
        maxRetries = 3       // 最多重试 3 次
        maxFailures = 20     // 最多允许 20 个测试失败时重试
        failOnPassedAfterRetry = true  // 重试后通过也视为失败（检测不稳定测试）
    }
}
```

详细配置：

```groovy
test {
    retry {
        // 重试次数
        maxRetries = 2
        
        // 最大失败数限制（超过此值不再重试）
        maxFailures = 10
        
        // 重试条件过滤
        filter {
            // 仅重试特定测试类
            includeClasses.add("*ServiceTest")
            // 排除不需要重试的测试
            excludeClasses.add("*IntegrationTest")
            
            // 仅重试特定注解的测试
            includeAnnotationClasses.add("com.example.FlakyTest")
        }
        
        // 检测不稳定的测试
        failOnPassedAfterRetry = true
        
        // 重试时使用的新 JVM 参数
        // jvmArgs = ['-Xmx512m']
    }
    
    testLogging {
        // 显示重试信息
        events "failed", "retried"
    }
}
```

## 9.2 高级重试策略

```groovy
import org.gradle.api.tasks.testing.logging.TestExceptionFormat

test {
    retry {
        maxRetries = 3
        
        // 重试时的类级别粒度
        // 默认情况下，只重试失败的测试方法
        // 可以设置 classRetry 重试整个类
        classRetry {
            maxRetries = 1
        }
    }
    
    testLogging {
        afterTest { descriptor, result ->
            if (result.resultType == TestResult.ResultType.FAILURE) {
                def count = test.retry.maxRetries
                logger.lifecycle("测试失败，将重试 (最多 ${test.retry.maxRetries} 次): ${descriptor.name}")
            }
        }
    }
}
```

# 10. 测试 Fixtures

## 10.1 java-test-fixtures 插件

测试夹具允许你将测试辅助类分享给下游项目：

```groovy
plugins {
    id 'java'
    id 'java-test-fixtures'
}

dependencies {
    // 测试夹具需要的依赖
    testFixturesImplementation 'junit:junit:4.13.2'
    testFixturesImplementation 'org.assertj:assertj-core:3.25.3'
    
    // 测试夹具可以使用主源码的 API（Implementation 级别）
    // 注意：主源码的 implementation 依赖对 testFixtures 不可见
    testFixturesImplementation 'com.fasterxml.jackson.core:jackson-databind:2.17.0'
}
```

源码目录结构：

```
src/
├── main/java/                     # 主源码
├── test/java/                      # 测试源码
└── testFixtures/java/             # 测试夹具（可被下游引用）
    └── com/example/testfixtures/
        ├── TestDataFactory.java
        ├── UserFixtures.java
        └── TestDatabaseSetup.java
```

下游项目引用测试夹具：

```groovy
// 下游项目的 build.gradle
dependencies {
    testImplementation(testFixtures(project(":core")))
    
    // 或者指定具体模块
    testImplementation(testFixtures("com.example:core:1.0.0"))
}
```

测试夹具使用示例：

```java
// src/testFixtures/java/com/example/testfixtures/UserFixtures.java
public final class UserFixtures {
    
    public static User aDefaultUser() {
        return User.builder()
            .id(UUID.randomUUID())
            .name("Test User")
            .email("test@example.com")
            .createdAt(LocalDateTime.now())
            .build();
    }
    
    public static User anInactiveUser() {
        return User.builder()
            .id(UUID.randomUUID())
            .name("Inactive User")
            .email("inactive@example.com")
            .active(false)
            .build();
    }
    
    public static List<User> aListOfUsers(int count) {
        return IntStream.range(0, count)
            .mapToObj(i -> User.builder()
                .id(UUID.randomUUID())
                .name("User " + i)
                .email("user" + i + "@example.com")
                .build())
            .collect(Collectors.toList());
    }
}
```

# 11. 集成测试分离

## 11.1 自定义 SourceSet 方式

```groovy
sourceSets {
    integrationTest {
        compileClasspath += sourceSets.main.output
        runtimeClasspath += sourceSets.main.output
        
        // 继承 test 的依赖
        compileClasspath += sourceSets.test.compileClasspath
        runtimeClasspath += sourceSets.test.runtimeClasspath
    }
}

configurations {
    integrationTestImplementation.extendsFrom testImplementation
    integrationTestRuntimeOnly.extendsFrom testRuntimeOnly
    integrationTestAnnotationProcessor.extendsFrom testAnnotationProcessor
}

dependencies {
    integrationTestImplementation 'org.testcontainers:testcontainers:1.19.7'
    integrationTestImplementation 'org.testcontainers:postgresql:1.19.7'
    integrationTestImplementation 'io.rest-assured:rest-assured:5.4.0'
}

tasks.register('integrationTest', Test) {
    description = 'Runs integration tests.'
    group = 'verification'
    
    testClassesDirs = sourceSets.integrationTest.output.classesDirs
    classpath = sourceSets.integrationTest.runtimeClasspath
    
    shouldRunAfter test
    
    // 集成测试通常需要更多内存
    maxHeapSize = '1g'
    
    // 集成测试一般不适合并行
    maxParallelForks = 1
    
    useJUnitPlatform {
        includeTags 'integration'
    }
    
    testLogging {
        events "passed", "skipped", "failed"
        showStandardStreams = true
    }
}

// 将集成测试加入 check 生命周期
tasks.check { dependsOn integrationTest }
```

## 11.2 JVM Test Suite 方式

```groovy
plugins {
    id 'java'
    id 'jvm-test-suite'
}

testing {
    suites {
        integrationTest(JvmTestSuite) {
            dependencies {
                implementation project()
                implementation 'org.testcontainers:testcontainers:1.19.7'
                implementation 'org.testcontainers:postgresql:1.19.7'
            }
            
            targets {
                all {
                    testTask.configure {
                        shouldRunAfter(tasks.test)
                        maxHeapSize = '1g'
                        
                        // 启动 Docker 容器
                        systemProperty 'spring.profiles.active', 'integration'
                    }
                }
            }
        }
    }
}
```

## 11.3 集成测试与 TestContainers

```groovy
dependencies {
    testImplementation 'org.testcontainers:testcontainers:1.19.7'
    testImplementation 'org.testcontainers:junit-jupiter:1.19.7'
    testImplementation 'org.testcontainers:postgresql:1.19.7'
    testImplementation 'org.testcontainers:kafka:1.19.7'
}

test {
    // 确保 Docker 可用
    // 可使用 Testcontainers 的 ryuk 容器自动清理
    
    systemProperty 'testcontainers.ryuk.disabled', 'false'
    
    // 重用容器加速测试
    systemProperty 'testcontainers.reuse.enable', 'true'
}
```

# 12. 测试调试与进阶

## 12.1 远程调试测试

```bash
# 启动 Gradle Daemon 时开启调试端口
./gradlew test -Dorg.gradle.debug=true

# 或者在 gradle.properties 配置
# org.gradle.jvmargs=-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005
```

```groovy
test {
    // 为每个测试 JVM fork 开启调试
    jvmArgs '-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=*:5005'
    
    // 或者使用 forkOptions
    forkOptions {
        jvmArgs = ['-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=*:5005']
    }
}
```

## 12.2 传递系统属性与环墶变量

```groovy
test {
    // 系统属性
    systemProperty 'env', System.getProperty('env', 'local')
    systemProperty 'app.version', project.version
    systemProperty 'test.data.dir', layout.projectDirectory.dir('src/test/data').asFile.absolutePath
    
    // 从命令行传入
    // ./gradlew test -Dcustom.property=value
    systemProperty 'custom.property', System.getProperty('custom.property')
    
    // 环境变量
    environment 'DATABASE_URL', 'jdbc:postgresql://localhost:5432/testdb'
    environment 'API_KEY', System.getenv('API_KEY') ?: 'test-key'
    
    // Java 系统属性透传
    systemProperties System.properties
    
    // 选择性透传
    systemProperties([
        'user.timezone': System.getProperty('user.timezone', 'UTC'),
        'file.encoding': 'UTF-8'
    ])
}
```

## 12.3 测试配置文件

```groovy
test {
    // 使用 -P 参数
    // ./gradlew test -Penv=staging
    if (project.hasProperty('env')) {
        systemProperty 'spring.profiles.active', project.property('env')
    }
    
    // 选择性执行测试子集
    if (project.hasProperty('testGroup')) {
        useJUnitPlatform {
            includeTags project.property('testGroup')
        }
    }
}
```

## 12.4 测试超时控制

```groovy
test {
    // 单个测试方法的超时时间
    timeout = Duration.ofMinutes(5)
    
    // 支持的时间单位
    // timeout = Duration.ofSeconds(30)
    // timeout = Duration.ofHours(1)
    
    // 高级超时处理
    timeout.testTimeout = Duration.ofMinutes(2)       // 单个测试
    timeout.forkTimeout = Duration.ofMinutes(10)       // 整个分叉
    timeout.taskTimeout = Duration.ofMinutes(30)       // 整个任务
}
```

## 12.5 测试总结的最佳实践

```groovy
test {
    useJUnitPlatform()
    
    maxParallelForks = Runtime.runtime.availableProcessors()
    forkEvery = 0
    maxHeapSize = '2g'
    
    testLogging {
        events "failed", "skipped"
        exceptionFormat = TestExceptionFormat.FULL
        showExceptions = true
        showCauses = true
        showStackTraces = true
        
        afterSuite { descriptor, result ->
            if (descriptor.parent == null) {
                def duration = (result.endTime - result.startTime) / 1000.0
                println "\n" + "=".multiply(50)
                println "测试执行完毕 | ${result.resultType}"
                println "  总: ${result.testCount} | 通过: ${result.successfulTestCount}"
                println "  失败: ${result.failedTestCount} | 跳过: ${result.skippedTestCount}"
                println "  耗时: ${String.format('%.2f', duration)}s"
                println "=".multiply(50)
            }
        }
    }
    
    filter {
        failFast = false
    }
    
    retry {
        maxRetries = 2
        failOnPassedAfterRetry = true
    }
    
    reports {
        junitXml.required = true
        html.required = true
    }
}
```

