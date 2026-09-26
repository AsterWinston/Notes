
# 1. 构建失败诊断工作流

## 1.1 系统化排查流程

```
                    构建失败
                       │
                       ▼
              ┌─ 查看错误消息 ─────────┐
              │   gradle <task> --info │
              └───────────────────────┘
                       │
                       ▼
              ┌─ 查看完整堆栈 ───────────┐
              │   gradle <task>         │
              │   --stacktrace          │
              │   --debug               │
              └────────────────────────┘
                       │
                       ▼
              ┌─ 隔离问题 ──────────────┐
              │   gradle clean <task>  │
              │   --refresh-dependencies│
              │   --no-daemon          │
              │   --no-build-cache     │
              └────────────────────────┘
                       │
                       ▼
              ┌─ 分析构建扫描 ──────────┐
              │   gradle <task> --scan │
              └────────────────────────┘
                       │
                       ▼
              ┌─ 检查环境 ──────────────┐
              │   gradle --version     │
              │   java -version        │
              │   gradle properties    │
              └────────────────────────┘
```

## 1.2 日志级别使用场景

| 日志级别 | 命令标志 | 适用场景 |
|---------|---------|---------|
| ERROR (默认) | `-q` / `--quiet` | 仅错误信息 |
| WARN | 无 | 警告以上 |
| LIFECYCLE (默认) | 无 | 任务执行进度 |
| INFO | `-i` / `--info` | 依赖解析、插件加载 |
| DEBUG | `-d` / `--debug` | 完整内部日志，ClassLoader 操作 |

```bash
# 常用调试组合
gradle build -i                                     # 查依赖解析问题
gradle build -d 2>&1 | tee debug.log                # 完整调试日志
gradle build --stacktrace                           # 异常堆栈（最常用）
gradle build --full-stacktrace                      # 完整堆栈（含 Gradle 内部）
gradle build -d --no-daemon --no-build-cache        # 排除缓存和守护进程干扰
gradle build -d -Dorg.gradle.debug=true             # 开启远程调试
```

# 2. 读取 Gradle 堆栈跟踪

## 2.1 堆栈跟踪结构

```text
FAILURE: Build failed with an exception.

* Where:                           ← 定位到文件和行号
Build file 'D:\project\build.gradle' line: 42

* What went wrong:                 ← 人类可读的错误描述
Execution failed for task ':compileJava'.
> Could not resolve all files for configuration ':compileClasspath'.
   > Could not find com.example:missing-lib:1.0.0.

* Try:                             ← 建议的修复方向
> Run with --stacktrace option to get the stack trace.
> Run with --info or --debug option to get more log output.
> Run with --scan to get full insights.
> Get more help at https://help.gradle.org.

* Exception is:                    ← 异常类型和消息
org.gradle.api.internal.tasks.TaskExecutionException: Execution failed for task ':compileJava'.
    at org.gradle.api.internal.tasks.execution...
    ...
Caused by: org.gradle.api.internal.artifacts.ivyservice.DefaultLenientConfiguration$ArtifactResolveException:
    Could not resolve all files for configuration ':compileClasspath'.
    ...
```

## 2.2 理解 Caused by 链

```text
TaskExecutionException                          ← 顶层：任务执行失败
  Caused by: ArtifactResolveException           ← 第二层：依赖解析失败
    Caused by: ModuleVersionNotFoundException   ← 根本原因：找不到模块版本
    
# 排查思路：从最内层 Caused by 开始分析
```

# 3. 常见错误分类与解决方案

## 3.1 依赖解析错误

### 3.1.1 Unresolved dependencies

```text
错误信息:
Could not resolve all files for configuration ':compileClasspath'.
> Could not find com.example:artifact:1.0.0.
```

```bash
# 排查步骤
# 1. 检查仓库配置
gradle dependencies --configuration compileClasspath

# 2. 刷新缓存
gradle build --refresh-dependencies

# 3. 检查命名
# 手动访问仓库 URL: https://repo.example.com/com/example/artifact/1.0.0/

# 4. 检查代理/网络
gradle build -Dhttp.proxyHost=proxy.example.com -Dhttp.proxyPort=8080

# 5. 验证 POM/IVY 文件
curl -I https://repo.maven.apache.org/maven2/com/example/artifact/1.0.0/artifact-1.0.0.pom
```

```groovy
// 解决：添加缺失的仓库
repositories {
    mavenCentral()
    maven { url 'https://repo.example.com/releases' }
    mavenLocal()
    
    // 如果仓库顺序很重要
    maven {
        url 'https://internal-repo.example.com'
        content {
            includeGroup 'com.example.internal'
        }
    }
}
```

### 3.1.2 版本冲突

```text
错误信息:
Conflict found for module 'com.google.guava:guava'.
  Selected: 33.0.0-jre
  Required by ':core' : 32.1.3-jre
  Required by ':api' : 31.1-jre
```

```groovy
// 解决方案1：显式声明依赖约束
dependencies {
    constraints {
        implementation('com.google.guava:guava:33.0.0-jre') {
            because '统一版本，修复兼容性问题'
        }
    }
}

// 解决方案2：resolutionStrategy
configurations.all {
    resolutionStrategy {
        force 'com.google.guava:guava:33.0.0-jre'
        failOnVersionConflict()  // 将冲突视为错误
    }
}

// 解决方案3：Gradle 模块元数据
// 在 settings.gradle
dependencyResolutionManagement {
    versionCatalogs {
        // version catalog 自动处理冲突
    }
}
```

### 3.1.3 传递依赖排除问题

```groovy
// 错误：传递依赖导致类重复
// Caused by: java.lang.ClassNotFoundException 或 NoClassDefFoundError

// 解决方案
dependencies {
    implementation('org.springframework.boot:spring-boot-starter-web') {
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-tomcat'
    }
    implementation 'org.springframework.boot:spring-boot-starter-undertow'
}

// 全局排除
configurations.all {
    exclude group: 'commons-logging', module: 'commons-logging'
}
```

## 3.2 插件错误

### 3.2.1 Failed to apply plugin

```text
错误信息:
An exception occurred applying plugin request [id: 'com.example.myplugin', version: '1.0.0']
> Could not create task of type 'MyTask'.
```

```bash
# 排查
# 1. 检查插件是否存在于 Plugin Portal
gradle plugin --search com.example.myplugin

# 2. 检查插件版本兼容性
# 查看插件的文档，确认支持的 Gradle 版本

# 3. 使用 --debug 查看插件解析过程
gradle build -d | grep -i "plugin"
```

```groovy
// 解决方案
plugins {
    id 'com.example.myplugin' version '1.0.0' apply false
}
// 如果插件无法解析，添加自定义 repository
pluginManagement {
    repositories {
        gradlePluginPortal()
        maven { url 'https://repo.example.com/plugins' }
    }
}
```

### 3.2.2 插件 ID 不匹配

```text
错误信息:
Plugin [id: 'org.springframework.boot'] was not found in any of the following sources:
- Gradle Core Plugins
- Plugin Repositories
```

```groovy
// Groovy DSL 中 legacy plugin application
// buildscript 方式需要明确 repository
buildscript {
    repositories {
        maven { url 'https://repo.spring.io/milestone' }
    }
    dependencies {
        classpath 'org.springframework.boot:spring-boot-gradle-plugin:3.2.0'
    }
}
apply plugin: 'org.springframework.boot'

// 现代方式 (plugins {} 块):
plugins {
    id 'org.springframework.boot' version '3.2.0'
}
```

## 3.3 任务执行错误

### 3.3.1 Task failed

```text
错误信息:
Execution failed for task ':compileJava'.
> Compilation failed; see the compiler error output for details.

排查：
1. 查看完整的编译器输出 (通常显示在 task 输出上方)
2. 使用 --info 查看详细编译器参数
3. 检查 sourceCompatibility 和目标 JVM 版本
```

### 3.3.2 增量编译 / UP-TO-DATE 问题

```bash
# 任务总是 UP-TO-DATE 但不执行预期工作
gradle compileJava --rerun-tasks          # 强制重新执行
gradle clean compileJava                  # 先清理再编译

# 任务从不 UP-TO-DATE，始终全量执行
gradle compileJava --info                 # 查看增量构建为何失效
gradle compileJava --debug | grep -i "up-to-date\|incremental"
```

```groovy
// 常见 UP-TO-DATE 失效原因
// 1. 输入/输出声明不正确
abstract class GenerateTask extends DefaultTask {
    @Input                      // 普通输入
    abstract Property<String> getConfig()
    
    @InputFile                  // 文件输入
    abstract RegularFileProperty getInputFile()
    
    @OutputFile                 // 文件输出
    abstract RegularFileProperty getOutputFile()
    
    // 缺少 @Input/@Output 注解 → Gradle 无法判断
    File someFile  // ❌ 未声明，不在增量检测范围内
}

// 2. @Input 对象未实现 equals/hashCode
class CustomInput {
    String name  // ❌ 自定义类型必须正确实现 equals/hashCode
}

// 3. 时间戳问题
// 如果任务总是重建，检查文件系统时间戳
```

### 3.3.3 循环依赖

```text
错误信息:
Circular dependency between tasks:
  :compileJava → :generateSources → :compileJava
```

```groovy
// 解决方案
// 1. 重构任务依赖关系
tasks.named('compileJava') {
    dependsOn generateSources
    // ❌ 不能在 generateSources 中又依赖 compileJava
}

// 2. 使用 Provider 间接引用
abstract class GenerateSources extends DefaultTask {
    @InputFiles
    abstract ConfigurableFileCollection getSourceFiles()
}

def generator = tasks.register('generateSources', GenerateSources)
def compiler = tasks.named('compileJava', JavaCompile)

// 使用 flatMap 构建合法的依赖图
generator.configure {
    sourceFiles.from(
        compiler.flatMap { it.source }  // 惰性引用，不形成循环
    )
}
```

## 3.4 OOM 和 JVM 问题

### 3.4.1 OutOfMemoryError

```text
错误信息:
java.lang.OutOfMemoryError: Java heap space
java.lang.OutOfMemoryError: GC overhead limit exceeded
java.lang.OutOfMemoryError: Metaspace
```

```properties
# gradle.properties
# 堆内存
org.gradle.jvmargs=-Xmx4096m -Xms1024m

# 元空间 (存储类元数据)
org.gradle.jvmargs=-Xmx4096m -XX:MaxMetaspaceSize=1024m

# 使用 G1GC 改进大内存性能
org.gradle.jvmargs=-Xmx4096m -XX:+UseG1GC -XX:MaxGCPauseMillis=200

# 设置内存分配比例
org.gradle.jvmargs=-Xmx4096m -XX:MaxMetaspaceSize=1024m -XX:ReservedCodeCacheSize=512m

# 调试 OOM 时生成 heap dump
org.gradle.jvmargs=-Xmx4096m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/gradle-oom.hprof
```

```bash
# 排查 OOM 的工具
# 1. 使用 --scan 分析内存使用时间线
# 2. 使用 jstat 监控 JVM
jstat -gcutil $(jps | grep GradleMain | awk '{print $1}') 1000

# 3. 连接 JVisualVM 进行实时监控
```

### 3.4.2 tools.jar 问题

```text
错误信息:
Could not find tools.jar. Please check that JAVA_HOME points to a JDK installation.
```

```bash
# Gradle 需要 JDK 而不是 JRE
# 检查 JAVA_HOME
echo $env:JAVA_HOME       # Windows
echo $JAVA_HOME            # Linux/Mac

# 设置 JDK 路径
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk
export PATH=$JAVA_HOME/bin:$PATH

# Gradle wrapper 可以指定 JDK
# gradle/wrapper/gradle-wrapper.properties
# distributionUrl=https\://services.gradle.org/distributions/gradle-8.5-bin.zip
```

# 4. 依赖解析进阶

## 4.1 依赖版本选择机制

```
依赖版本选择的优先级：
1. 直接依赖的硬约束 (force / strictly)
2. 依赖图中所有版本的最高版本
3. 平台约束 (BOM / enforcedPlatform)
4. 传递依赖的最低版本 (当有 strictly lower bound 时)
5. 仓库中可用的任意版本

当有多个版本时：
  精确版本 > 版本范围 > latest.release
  同一层级有多个版本 → 选择最高的
```

```bash
# 查看依赖解析信息
gradle dependencyInsight --dependency guava --configuration runtimeClasspath

# 输出:
# com.google.guava:guava:33.0.0-jre (by constraint)
#   Variant runtime:
#     ...
# com.google.guava:guava:33.0.0-jre
#   Selection reasons:
#     - By constraint
#     - Forced
#
# com.google.guava:guava:32.1.3-jre -> 33.0.0-jre
#   Requested by:
#     project :core
```

## 4.2 依赖 Insight 深入

```bash
# 看特定依赖为什么被选中
gradle dependencyInsight --dependency spring-core

# 指定配置
gradle dependencyInsight --dependency spring-core --configuration compileClasspath

# 查看单个依赖的所有请求路径
gradle dependencyInsight --dependency guava --configuration runtimeClasspath

# 结合 --scan 更直观
gradle dependencyInsight --dependency guava --configuration runtimeClasspath --scan
```

## 4.3 依赖锁定

```groovy
// 使用依赖锁定确保可重复构建
dependencyLocking {
    lockAllConfigurations()
}

// 生成锁定文件
// gradle dependencies --write-locks

// 更新锁定
// gradle dependencies --update-locks guava,spring-core

// build.gradle.kts
dependencyLocking {
    lockAllConfigurations()
}
configurations.all {
    resolutionStrategy {
        activateDependencyLocking()
    }
}
```

## 4.4 组件选择规则

```groovy
// 自定义组件选择规则
configurations.all {
    resolutionStrategy {
        componentSelection {
            all { selection ->
                // 拒绝所有 alpha/beta 版本
                if (selection.candidate.version.contains('alpha') ||
                    selection.candidate.version.contains('beta') ||
                    selection.candidate.version.contains('rc')) {
                    selection.reject("不选择预发布版本")
                }
            }
            // 针对特定模块
            withModule("com.example:unstable-lib") { selection ->
                selection.reject("已弃用的库，使用替代方案")
            }
            // 针对特定组
            withModule("com.deprecated:*") { selection ->
                selection.reject("废弃的组织")
            }
        }
    }
}

// 或者使用 component metadata rule
dependencies {
    components {
        withModule("com.google.guava:guava") {
            allVariants {
                withDependencies {
                    // 移除传递依赖
                    removeAll { it.group == "com.google.code.findbugs" }
                }
            }
        }
    }
}
```

# 5. Gradle Daemon 问题

## 5.1 常见守护进程错误

```text
# 连接被拒绝
Could not connect to the Gradle daemon.
> Connection refused: connect

# 过期的锁文件
Unexpected lock file found in the Gradle user home directory.

# 守护进程僵死
Timeout waiting to connect to the Gradle daemon.
```

```bash
# 守护进程管理命令
gradle --stop                          # 停止所有守护进程
gradle --status                        # 查看守护进程状态

# 输出示例：
# PID  STATUS   INFO
# 12345 IDLE    8.5
# 67890 BUSY    8.5

# 强制清理
rm -rf ~/.gradle/daemon
# Windows: Remove-Item -Recurse -Force $env:USERPROFILE\.gradle\daemon

# 不启动守护进程 (调试用)
gradle build --no-daemon

# 重置守护进程 TCP 端口
# 删除锁文件
rm ~/.gradle/daemon/*.lock
```

## 5.2 守护进程配置

```properties
# gradle.properties - 守护进程控制
org.gradle.daemon=true                # 启用守护进程 (默认 true)
org.gradle.daemon.idletimeout=10800000 # 空闲超时 (毫秒, 默认 3 小时)
org.gradle.jvmargs=-Xmx2048m         # 守护进程 JVM 参数
org.gradle.workers.max=8             # 最大 worker 线程数
```

# 6. 缓存问题

## 6.1 缓存未命中调查

```bash
# 查看构建缓存统计
gradle build --build-cache

# BUILD SUCCESSFUL in 10s
# 12 actionable tasks: 4 executed, 8 from cache

# 调试缓存未命中
gradle build --info | grep -i "cache"
# 输出：
# Task ':compileJava' is not up-to-date because:
#   Input property 'source' has changed...
# Stored cache entry for ':compileJava' with cache key xxx
```

```bash
# 查看特定 task 的缓存 keys
gradle help --scan  # 在构建扫描中查看

# 查看本地缓存目录
ls ~/.gradle/caches/build-cache-1/
```

## 6.2 损坏的缓存

```bash
# 清理特定缓存
rm -rf ~/.gradle/caches/6.5/                        # 特定版本
rm -rf ~/.gradle/caches/modules-2/files-2.1/        # 依赖缓存
rm -rf ~/.gradle/caches/build-cache-1/              # 构建缓存
rm -rf ~/.gradle/caches/journal-1/                  # 文件变更日志
rm -rf ~/.gradle/caches/transforms-3/               # 工件转换缓存
rm -rf ~/.gradle/wrapper/dists/                     # Gradle 发行版缓存

# 或使用 Gradle 内置方式
gradle clean
gradle --refresh-dependencies                       # 重新下载依赖

# 项目级别缓存
rm -rf .gradle/                                     # 项目本地缓存
```

## 6.3 配置缓存问题

```bash
# 配置缓存错误
# Configuration cache problems found.

# 解决问题
gradle --configuration-cache-problems=warn build    # 降级为警告
gradle build --no-configuration-cache               # 完全禁用

# 清理配置缓存
rm -rf .gradle/configuration-cache

# 查看配置缓存问题详情
gradle build --info 2>&1 | grep "Configuration cache"
```

# 7. Kotlin DSL 特定问题

## 7.1 类型不匹配

```kotlin
// 常见类型错误
// ❌ 错误: Kotlin DSL 中类型必须精确匹配
tasks.named<JavaCompile>("compileJava") {
    sourceCompatibility = "17"  // ❌ 需要 JavaVersion 类型
}
// ✅ 正确:
tasks.named<JavaCompile>("compileJava") {
    sourceCompatibility = JavaVersion.VERSION_17.toString()
}

// 或通过 java extension
java {
    sourceCompatibility = JavaVersion.VERSION_17
}
```

```kotlin
// Provider 类型问题
val version: Provider<String> = provider { "1.0.0" }
// ❌ version 不能直接赋值给 Property<String>，需要 .set()
myTask.myProperty.set(version)

// ❌ 错误: 类型推断失败
dependencies {
    implementation("com.example:lib:1.0")
    // Kotlin DSL 中字符串直接传递给 implementation 可能会被推断为 ExternalModuleDependency
}

// ✅ 正确: 使用版本目录或显式 GroupArtifactVersion
dependencies {
    implementation(libs.someLibrary)
    // 或
    implementation("com.example:lib:1.0")
}
```

## 7.2 Unresolved reference 问题

```kotlin
// Kotlin DSL 类型安全访问器
// 问题: 访问器不生成或过期
// 原因: 1. 依赖未正确声明 2. IDE 缓存未刷新

// 解决步骤:
// 1. 刷新 Gradle 项目: ./gradlew clean
// 2. IDE: File → Invalidate Caches and Restart
// 3. 删除 .idea 和 .gradle: rm -rf .idea .gradle
// 4. 重新导入项目

// 常见场景
plugins {
    `java`  // ✅ 访问器自动生成
}

tasks.named<Test>("test") {  // ✅ 类型安全
    useJUnitPlatform()
}

// 使用 the<>() 获取扩展
the<JavaPluginExtension>().apply {
    sourceCompatibility = JavaVersion.VERSION_17
}

// 或简写
configure<JavaPluginExtension> {
    sourceCompatibility = JavaVersion.VERSION_17
}
```

## 7.3 脚本编译问题

```bash
# Kotlin 脚本编译失败
gradle build --debug | grep -A5 "script compilation"

# 查看编译错误
gradle help 2>&1 | Select-String "error:"

# 清理脚本缓存
rm -rf ~/.gradle/caches/kotlin-dsl/
rm -rf .gradle/kotlin/

# 调整 Kotlin DSL 编译内存
org.gradle.jvmargs=-Xmx2048m -XX:MaxMetaspaceSize=512m
```

# 8. 性能调试

## 8.1 构建扫描性能分析

```bash
# 生成构建扫描（最推荐的性能分析方式）
gradle build --scan

# 查看任务执行时间
gradle build --profile
# 生成: build/reports/profile/profile-YYYY-MM-DD-HH-mm-ss.html

# 查看配置时间 vs 执行时间
gradle build --info 2>&1 | grep -E "Total time|CONFIGURE|BUILD"
# 输出:
# Configure project : 1.234s
# BUILD SUCCESSFUL in 5.678s

# 度量配置时间
gradle help --profile
# 对比 gradle help 和 gradle build 的配置时间
```

## 8.2 常见性能问题

```groovy
// 问题1：配置阶段执行了过多工作
// ❌ 在配置阶段解析文件
def config = new XmlSlurper().parse(file('config.xml'))
tasks.register('myTask') {
    doLast { println config.someProperty }
}

// ✅ 延迟到执行阶段
tasks.register('myTask') {
    doLast {
        def config = new XmlSlurper().parse(file('config.xml'))
        println config.someProperty
    }
}

// 问题2：不必要的分辨率
// ❌ 在配置阶段解析依赖
configurations.compileClasspath.resolve()
def classpathFiles = configurations.compileClasspath.files
// ✅ 使用 Provider 延迟解析

// 问题3：过度使用 afterEvaluate
// ❌ 
afterEvaluate {
    // 强制所有项目先完成配置
}
// ✅ 使用惰性配置
tasks.configureEach { ... }
```

## 8.3 性能优化命令

```bash
# 并行构建
gradle build --parallel
# gradle.properties: org.gradle.parallel=true

# 构建缓存
gradle build --build-cache
# gradle.properties: org.gradle.caching=true

# 配置缓存
gradle build --configuration-cache
# gradle.properties: org.gradle.configuration-cache=true

# 按需配置
gradle build --configure-on-demand
# gradle.properties: org.gradle.configure-on-demand=true

# 跳过测试
gradle build -x test
gradle assemble

# 离线构建 (缓存中有依赖时)
gradle build --offline

# 并行 + 缓存 + 按需配置 + 守护进程（组合最佳性能）
gradle build --parallel --build-cache --configure-on-demand
```

# 9. 构建环境问题

## 9.1 JDK 版本不匹配

```bash
# 检查当前 Gradle 使用的 JDK
gradle --version
# Gradle 8.5
# Build time:   2023-12-19
# JVM:          17.0.9
# OS:           ...

# 检查项目的 sourceCompatibility
gradle properties | grep -i "source\|target\|java"
# java.sourceCompatibility: 17
# java.targetCompatibility: 17

# JDK 工具链配置 (替代 sourceCompatibility)
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
        vendor = JvmVendorSpec.ADOPTIUM
    }
}
```

## 9.2 文件权限和网络

```bash
# 权限问题 (Linux/Mac)
sudo chown -R $(whoami) ~/.gradle
sudo chown -R $(whoami) .gradle

# 代理配置
# gradle.properties
systemProp.http.proxyHost=proxy.example.com
systemProp.http.proxyPort=8080
systemProp.http.proxyUser=username
systemProp.http.proxyPassword=password
systemProp.https.proxyHost=proxy.example.com
systemProp.https.proxyPort=8080
systemProp.http.nonProxyHosts=*.local|localhost|127.0.0.1

# 或命令行
gradle build -Dhttp.proxyHost=proxy.example.com -Dhttp.proxyPort=8080
```

## 9.3 磁盘空间不足

```bash
# 检查 Gradle 缓存占用
du -sh ~/.gradle/caches/

# 清理旧的 Gradle 版本
ls ~/.gradle/wrapper/dists/
rm -rf ~/.gradle/wrapper/dists/gradle-8.4-bin/

# 清理依赖缓存中未使用的版本
rm -rf ~/.gradle/caches/modules-2/

# 清理构建缓存
rm -rf ~/.gradle/caches/build-cache-1/

# 项目级别清理
gradle clean
rm -rf .gradle/
```

# 10. 实用排查命令速查

```bash
# === 基础信息 ===
gradle --version                          # Gradle + JVM 版本
gradle --status                           # 守护进程状态
gradle projects                           # 项目结构
gradle tasks --all                        # 所有任务列表
gradle properties                         # 所有属性
gradle buildEnvironment                   # 构建环境信息
gradle dependencies                       # 依赖树
gradle dependencyInsight --dependency <name>  # 依赖溯源

# === 调试标志 ===
gradle build --stacktrace                 # 异常堆栈
gradle build --full-stacktrace            # 完整堆栈（含内部）
gradle build --info                       # INFO 日志
gradle build --debug                      # DEBUG 日志
gradle build --scan                       # 构建扫描
gradle build --profile                    # 性能报告

# === 构建控制 ===
gradle build --no-daemon                  # 不使用 daemon
gradle build --no-build-cache             # 不使用构建缓存
gradle build --no-configuration-cache     # 不检查配置缓存
gradle build --refresh-dependencies       # 刷新依赖
gradle build --rerun-tasks                # 强制重新执行所有任务
gradle build --offline                    # 离线模式
gradle build -x test                      # 跳过测试任务
gradle build --parallel                   # 并行模式
gradle build --configure-on-demand        # 按需配置

# === 守护进程管理 ===
gradle --stop                             # 停止守护进程
rm -rf ~/.gradle/daemon                   # 强制清除守护进程

# === 缓存管理 ===
gradle clean                              # 清理构建产物
rm -rf .gradle                            # 清理项目级别的 Gradle 缓存
rm -rf ~/.gradle/caches                   # 清理用户级别的 Gradle 缓存
rm -rf ~/.gradle/caches/transforms-3      # 清理工件转换缓存
```
