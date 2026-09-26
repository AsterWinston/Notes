
Gradle 性能调优是一个系统化工程，涉及 Daemon 配置、并行执行、缓存策略、增量编译、依赖管理和 JVM 参数等多个维度。本章从基础到高级内部机制进行全面梳理。

# 1. Gradle Daemon 深度解析

## 1.1 Daemon 架构与生命周期

Gradle Daemon 是一个长时间运行的后台 JVM 进程，它缓存构建脚本信息、类加载信息和任务图，避免每次构建都从头启动 JVM。

```
┌───────────────────────────────────────────────────────┐
│                    Gradle 客户端 (gradlew)             │
│                      (短生命周期进程)                   │
├───────────────────────────────────────────────────────┤
│  1. 查找或启动 Daemon                                  │
│  2. 通过 socket 发送构建请求                           │
│  3. 接收构建结果                                       │
└───────────────┬───────────────────────────────────────┘
                │  IPC (本地 socket)
                ▼
┌───────────────────────────────────────────────────────┐
│                  Gradle Daemon (后台进程)              │
│                 (长时间运行、可复用)                    │
├───────────────────────────────────────────────────────┤
│  生命周期:                                             │
│    - 启动: 首次构建或 gradle --daemon                 │
│    - 空闲: 等待新构建请求 (默认 3 小时后停止)          │
│    - 活跃: 正在处理构建请求                            │
│    - 停止: gradle --stop / 空闲超时 / 内存不足        │
│                                                       │
│  缓存:                                                 │
│    - 构建脚本 class (编译后的 build.gradle)            │
│    - 类加载器缓存                                      │
│    - 任务执行状态                                      │
│    - buildSrc 编译结果                                 │
└───────────────────────────────────────────────────────┘
```

## 1.2 Daemon 配置

```properties
# gradle.properties — 全局 Daemon 调优

# 始终使用 Daemon (Gradle 3.0+ 已是默认)
org.gradle.daemon=true

# Daemon JVM 参数 (最重要)
org.gradle.jvmargs=-Xmx2048m -XX:MaxMetaspaceSize=512m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8

# 空闲超时 (毫秒)
org.gradle.daemon.idletimeout=3600000

# 中小型项目 GC 推荐
org.gradle.jvmargs=-Xmx2048m -Xms512m -XX:+UseG1GC -XX:MaxMetaspaceSize=512m -Dfile.encoding=UTF-8

# 大型项目 GC 推荐 (ZGC)
org.gradle.jvmargs=-Xmx4096m -Xms1024m -XX:+UseZGC -XX:+ZGenerational -XX:MaxMetaspaceSize=512m -XX:ReservedCodeCacheSize=256m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8
```

不同规模项目的推荐 JVM 参数：

| 项目规模 | Xmx | Xms | MaxMetaspaceSize | CodeCache | 适用场景 |
|---------|-----|-----|-----------------|-----------|---------|
| 小型 (< 50 模块) | 1g | 256m | 256m | 128m | 简单应用 |
| 中型 (50-200 模块) | 2g - 4g | 512m - 1g | 512m | 256m | 多模块企业应用 |
| 大型 (> 200 模块) | 4g - 8g | 2g - 4g | 1g | 512m | 微服务 monorepo |

## 1.3 Daemon 管理命令

```bash
# 查看当前运行的 Daemon
gradle --status

# 停止所有 Daemon
gradle --stop

# 启动 Daemon (但不构建)
gradle --daemon

# 不使用 Daemon (一次性构建, 仅调试用)
gradle build --no-daemon

# 查看 Daemon 健康信息
gradle --info help
```

## 1.4 Daemon 内存溢出排查

```bash
# 查看 Daemon 日志 (位置: ~/.gradle/daemon/<version>/)
ls ~/.gradle/daemon/8.7/

# 分析日志
tail -100 ~/.gradle/daemon/8.7/daemon-12345.out.log
```

常见 Daemon 问题与解决方案：

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| `java.lang.OutOfMemoryError: Metaspace` | 类加载过多 | 增大 `-XX:MaxMetaspaceSize=512m` |
| `java.lang.OutOfMemoryError: Java heap space` | 堆内存不足 | 增大 `-Xmx` |
| Daemon 频繁重启 | 内存泄漏或崩溃 | 检查日志 + 增大内存 |
| Daemon 消失 | 空闲超时 | 增加 `org.gradle.daemon.idletimeout` |

## 1.5 Daemon 内部原理

Gradle Daemon 通过以下机制提升性能：

1. **JVM 预热 (Warm-up)**：首次构建后，JIT 编译器已优化热点代码，后续构建直接受益
2. **类加载器缓存**：构建脚本编译后的 class 保留在内存中，避免反复编译
3. **任务执行状态缓存**：维护任务输入/输出的哈希状态，加速 up-to-date 检查
4. **Gradle 运行时缓存**：保留 Gradle API 内部状态，避免重复初始化

Daemon 与客户端的通信协议：

```
客户端进程                    Daemon 进程
    │                             │
    │  1. 通过 DaemonConnector    │
    │     扫描 ~/.gradle/daemon/   │
    │     获取可用 Daemon 列表     │
    │                             │
    │  2. 选择兼容的 Daemon       │
    │     (版本匹配、空闲状态)     │
    │                             │
    │  3. 通过 Unix Domain       │
    │     Socket 或 TCP 连接      │
    │─────────────────────────────▶
    │                             │
    │  4. 序列化请求 (BuildAction)│
    │─────────────────────────────▶
    │                             │  5. 反序列化请求
    │                             │  6. 执行构建
    │                             │  7. 序列化结果
    │  8. 接收结果                │
    │◀─────────────────────────────
    │                             │
    │  9. 客户端退出              │  Daemon 继续等待
```

# 2. 并行执行

## 2.1 org.gradle.parallel 原理

```properties
# gradle.properties
org.gradle.parallel=true
org.gradle.workers.max=8
```

并行执行原理图解：

```
跨模块依赖图自动并行分析:

    :core:compileJava ────────────── 无依赖, 立即执行
         │
    ┌────┴────────────┬──────────────────┐
    │                 │                  │
:service:         :web:            :admin:
compileJava       compileJava       compileJava
    │ 依赖 :core     │ 依赖 :core      │ 依赖 :core
    │                 │                  │
    │  这三个任务可以并行执行 (因为它们只依赖 :core:compileJava)
    │
    ▼
:service:test ───── 依赖 :service:compileJava, 并行于 :web:compileJava
```

关键规则：
- 有依赖关系的任务严格串行
- 无直接依赖关系的任务可并行
- 同一模块内的任务按生命周期顺序

## 2.2 Worker 数量计算

```bash
# 命令行指定
./gradlew build --parallel --max-workers=4

# gradle.properties
org.gradle.workers.max=6
```

选择 Worker 数量建议：

| CPU 核心数 | 推荐 Worker 数 | 原因 |
|-----------|--------------|------|
| 2 核 | 2 | 充分利用 |
| 4 核 | 3-4 | 留一点给系统 |
| 8 核 | 5-6 | 避免过度竞争 |
| 16 核 | 8-10 | 受 JVM GC 开销制约 |
| 32 核 | 12-16 | 大量模块并行 |

## 2.3 并行执行与测试

```groovy
test {
    maxParallelForks = Runtime.runtime.availableProcessors().intdiv(2) ?: 1
    minHeapSize = "256m"
    maxHeapSize = "2g"
}
// 总并发 = parallel workers x test maxParallelForks
// 注意内存消耗可能暴增
```

## 2.4 configure-on-demand

```properties
# gradle.properties
org.gradle.configureondemand=true
```

`configure-on-demand` 仅配置请求的项目及其下游依赖项目，跳过与当前构建无关的项目：

```
仅构建 :service 时, ./gradlew :service:build

configure-on-demand=false (默认):
    配置 :core -> 配置 :service -> 配置 :web -> 配置 :admin -> 执行 :service 任务
    (所有模块都被配置, 即使不需要)

configure-on-demand=true:
    配置 :core -> 配置 :service -> 执行 :service 任务
    (只配置相关模块, 节省配置时间)
```

注意: 此特性在 Gradle 8.x 中仍处于孵化阶段, 某些插件可能不兼容。

# 3. 构建缓存 (Build Cache)

## 3.1 本地构建缓存

```properties
# gradle.properties
org.gradle.caching=true
```

构建缓存工作原理：

```
第一次构建:
    :compileJava
        inputs: src/**/*.java  --hash--> "a3f2b1c4"
                jdk version
                compile classpath
        计算 input hash  -> 本地无缓存
        执行任务 -> 产生 output
        存储: hash -> output

第二次构建 (未修改源码):
    :compileJava
        inputs 不变 -> hash 仍为 "a3f2b1c4"
        查找缓存: "a3f2b1c4" 命中
        直接恢复 output (跳过执行)
```

缓存键 (Cache Key) 的构成：

```
Cache Key = Hash(
    task.class.name          # 任务类名
    + task.path              # 任务路径 (如 :core:compileJava)
    + task.inputs.properties # 所有 @Input 属性值
    + task.inputs.files      # 所有 @InputFiles 的文件内容哈希
)
```

## 3.2 远程构建缓存

```groovy
// settings.gradle.kts
buildCache {
    local {
        directory = file("$rootDir/.gradle/build-cache")
        enabled = true
        push = true
        pull = true
        removeUnusedEntriesAfterDays = 30
    }

    remote<HttpBuildCache> {
        url = uri("https://build-cache.company.com/cache/")
        isAllowUntrustedServer = false
        credentials {
            username = System.getenv("BUILD_CACHE_USER")
            password = System.getenv("BUILD_CACHE_PASS")
        }
        isEnabled = true
        pull = true
        push = System.getenv("CI") == "true"  // 仅 CI 推送
    }
}
```

远程缓存层级 (查找优先级):

```
任务读取缓存:
    1. 本地缓存查找 (快)
    2. 远程缓存查找 (慢, 需网络) -> 命中 -> 存储到本地缓存

任务写入缓存:
    1. 任务执行完毕后
    2. 写入本地缓存 (快)
    3. 异步推送到远程缓存 (如果 push=true)
```

## 3.3 构建缓存节点部署

```bash
# 使用 Gradle 官方缓存节点 Docker 镜像
docker run -p 5071:5071 gradle/build-cache-node:latest
```

## 3.4 任务缓存配置

```groovy
tasks.register('myTask') {
    inputs.property('version', project.version)
    inputs.file('config.xml')
    inputs.dir('src/data')
    outputs.file('build/output.txt')
    outputs.dir('build/generated')
    outputs.cacheIf { true }
    doLast { /* 任务逻辑 */ }
}

// 禁用特定任务的缓存
tasks.named('compileJava') {
    outputs.cacheIf { false }
}
```

# 4. 配置缓存 (Configuration Cache)

## 4.1 配置缓存原理

配置缓存是 Gradle 7.0+ 引入的重大性能特性，它将"配置阶段"的结果序列化到磁盘，后续构建直接从缓存加载，跳过配置阶段。

```
首次构建 (冷缓存):
    [初始化 settings] -> [配置 build.gradle] -> [执行 tasks]
                                |
                          序列化配置结果
                                v
                        .gradle/configuration-cache/

后续构建 (热缓存):
    [初始化 settings] --加载缓存--> [执行 tasks]
                       (跳过配置, 这是巨大的时间节省!)
```

## 4.2 启用配置缓存

```properties
# gradle.properties
org.gradle.configuration-cache=true
```

```bash
# 命令行启用
./gradlew build --configuration-cache

# 查看配置缓存状态
./gradlew build --configuration-cache-problems=warn

# 配置缓存问题报告输出到: build/reports/configuration-cache/
```

## 4.3 配置缓存兼容性要求

配置缓存要求所有构建逻辑对环境不敏感。兼容性检查清单：

```groovy
// 不兼容: 在配置阶段读取系统属性
version = System.getProperty('app.version') ?: '1.0.0'

// 兼容: 使用 Provider API
version = providers.systemProperty('app.version').orElse('1.0.0').get()

// 不兼容: 在配置阶段读取环境变量
def env = System.getenv('DEPLOY_ENV')

// 兼容: 使用 Provider API
def envProvider = providers.environmentVariable('DEPLOY_ENV')

// 不兼容: 在配置阶段使用文件系统操作
def gitHash = 'git rev-parse HEAD'.execute().text

// 兼容: 将逻辑移到任务或使用 ValueSource
```

常见不兼容模式对照表：

| 不兼容行为 | 为何不兼容 | 解决方案 |
|-----------|----------|---------|
| 配置阶段读取 System.getProperty() | 属性在 JVM 间可能不同 | 使用 providers.systemProperty() |
| 配置阶段读取 System.getenv() | 环境变量可能不同 | 使用 providers.environmentVariable() |
| 配置阶段执行 file.delete() | 副作用不可复现 | 移到任务执行阶段 |
| 配置阶段使用 new Date() | 时间不断变化 | 使用 Provider 延迟求值 |
| afterEvaluate / beforeEvaluate | 时序依赖不确定 | 使用 Provider.map/flatMap |
| 使用 Gradle 对象之外的全局状态 | 不可序列化 | 消除全局状态 |

## 4.4 配置缓存兼容性修复示例

```groovy
// 修复前 (不兼容)
ext {
    buildTime = new Date().format('yyyy-MM-dd HH:mm:ss')
}
task printBuildTime {
    doLast {
        println "Build time: $buildTime"
    }
}

// 修复后 (兼容) — 使用 ValueSource
abstract class BuildTimeProvider implements ValueSource<String, ValueSourceParameters.None> {
    @Override
    String obtain() {
        return new Date().format('yyyy-MM-dd HH:mm:ss')
    }
}
task printBuildTime {
    def buildTime = providers.of(BuildTimeProvider) { }
    doLast {
        println "Build time: ${buildTime.get()}"
    }
}
```

## 4.5 排查配置缓存问题

```bash
# 详细诊断模式
./gradlew build --configuration-cache --info

# 仅检查兼容性
./gradlew help --configuration-cache

# 清除配置缓存
rm -rf .gradle/configuration-cache

# 查看 HTML 报告
# 打开 build/reports/configuration-cache/configuration-cache-report.html
```

# 5. 增量编译

## 5.1 Java 增量编译

Gradle 内置了增量 Java 编译，仅重新编译发生变化的源文件及其传递依赖：

```
增量编译判定流程:

源文件 A.java 修改了
        |
        v
分析 A.java 的依赖:
    - 导入 B.java
    - 导入 C.java
    - 使用 String (JDK)
        |
        v
需重新编译:
    - A.java (必须)
    - 依赖了 A 的类?
      -> 如果 A 的变更仅涉及方法体 (无 public API 变更) -> 不需要
      -> 如果 A 的 public API 变更了 -> 需要重新编译
```

```groovy
tasks.withType(JavaCompile) {
    options.incremental = true
}
```

## 5.2 Kotlin 增量编译

```kotlin
tasks.withType<KotlinCompile> {
    kotlinOptions {
        incremental = true
    }
}

tasks.withType<KotlinCompile>().configureEach {
    kotlinDaemonJvmArguments.set(listOf(
        "-Xmx2g",
        "-XX:MaxMetaspaceSize=512m"
    ))
}
```

## 5.3 Groovy 增量编译

```groovy
tasks.withType(GroovyCompile) {
    groovyOptions.incremental = true
    groovyOptions.encoding = 'UTF-8'
    groovyOptions.forkOptions.jvmArgs = ['-Xmx1g']
}
```

# 6. 文件系统监控 (VFS)

## 6.1 VFS Watch 原理

Gradle 6.5+ 引入了文件系统监控，追踪文件变化以优化 up-to-date 检查和增量编译：

```properties
# gradle.properties
org.gradle.vfs.watch=true        # 启用文件系统监控
org.gradle.vfs.verbose=false     # 输出详细 VFS 日志
```

VFS 工作原理：

```
Gradle Virtual File System (VFS)
    - 通过 OS 文件监控 API 接收文件事件:
      Linux: inotify
      macOS: FSEvents
      Windows: ReadDirectoryChangesW
    - 维护每个文件的元数据 (大小, 修改时间, hash)
    - 维护文件层级关系
    - 用途:
      * 加速 up-to-date 检查
      * 为增量编译器提供变更列表
      * 避免昂贵的全量文件扫描
```

# 7. Up-to-Date 检查优化

## 7.1 任务输入/输出优化

```groovy
tasks.register('generateConfig') {
    inputs.property('env', 'production')
    inputs.file('config-template.yml')
    inputs.dir('src/config')
    inputs.properties(['version': project.version, 'debug': false])

    outputs.file('build/config.yml')
    outputs.dir('build/generated-config')
    outputs.cacheIf { System.getenv('CI') == 'true' }

    doLast { /* 实际工作 */ }
}
```

Up-to-Date 判定流程图：

```
任务执行前:
    1. 检查 outputs 是否存在?
       └─ 否 -> 必须执行
    2. 检查 inputs 是否变化?
       - 比较 input.properties
       - 比较 input.files hash
       └─ 变化 -> 必须执行
    3. 检查 outputs 是否比 inputs 更新?
       └─ 否 -> 必须执行
       └─ 是 -> UP-TO-DATE (跳过执行)
```

## 7.2 自定义 up-to-date 检查

```groovy
abstract class MyTask extends DefaultTask {
    @InputFile
    abstract RegularFileProperty getInputFile()

    @Input
    abstract Property<String> getMode()

    @OutputFile
    abstract RegularFileProperty getOutputFile()

    @TaskAction
    void execute() { /* ... */ }
}
```

## 7.3 常见陷阱

```groovy
// 错误: 每次构建的 input 都不一样
task badTask {
    inputs.property('timestamp', System.currentTimeMillis())  // 每次都变
}

// 正确: input 稳定
task goodTask {
    inputs.property('version', project.version)
    inputs.file('template.txt')
}

// 错误: 输出目录未声明
task badGenerate {
    doLast {
        file('build/foo.txt').text = 'content'  // 没有声明 outputs
    }
}

// 正确: 声明所有 outputs
task goodGenerate {
    outputs.file('build/foo.txt')
    doLast {
        file('build/foo.txt').text = 'content'
    }
}
```

# 8. 依赖解析优化

## 8.1 依赖锁定

```groovy
dependencyLocking {
    lockAllConfigurations()
}
```

生成的锁文件 (gradle/dependency-locks/*.lockfile):

```
com.fasterxml.jackson.core:jackson-annotations:2.17.0=compileClasspath,runtimeClasspath
com.fasterxml.jackson.core:jackson-core:2.17.0=compileClasspath,runtimeClasspath
com.fasterxml.jackson.core:jackson-databind:2.17.0=compileClasspath,runtimeClasspath
```

```groovy
// 灵活配置
dependencyLocking {
    configurations.computeIfAbsent("runtimeClasspath") {
        lockMode = LockMode.STRICT
    }
    configurations.computeIfAbsent("compileClasspath") {
        lockMode = LockMode.LENIENT
    }
}
```

生成和更新锁文件:

```bash
# 生成锁文件
./gradlew dependencies --write-locks

# 更新指定依赖的锁
./gradlew dependencies --update-locks 'com.fasterxml.jackson.*'
```

## 8.2 避免动态版本

```groovy
dependencies {
    // 避免动态版本
    // implementation 'com.google.guava:guava:33.+'             // 范围版本
    // implementation 'com.google.guava:guava:latest.release'    // 最新版本

    // 使用固定版本
    implementation 'com.google.guava:guava:33.1.0-jre'
}
```

## 8.3 依赖解析缓存策略

```groovy
configurations.all {
    // 动态版本缓存 10 分钟
    resolutionStrategy.cacheDynamicVersionsFor 10, 'minutes'

    // 变化模块 (SNAPSHOT) 每次检查
    resolutionStrategy.cacheChangingModulesFor 0, 'seconds'

    // 强制刷新
    // resolutionStrategy.refresh()
}
```

## 8.4 依赖解析规则与冲突解决

```groovy
configurations.all {
    resolutionStrategy {
        // 强制特定版本
        force 'com.google.guava:guava:33.1.0-jre'
        force 'com.fasterxml.jackson.core:jackson-databind:2.17.0'

        // 排除不需要的传递依赖
        eachDependency { DependencyResolveDetails details ->
            if (details.requested.group == 'commons-logging') {
                details.useTarget 'org.slf4j:jcl-over-slf4j:2.0.12'
            }
        }

        // 全局排除
        exclude group: 'commons-logging', module: 'commons-logging'
    }
}
```

## 8.5 依赖分析命令

```bash
# 查看依赖树
./gradlew dependencies

# 查看特定配置的依赖树
./gradlew :core:dependencies --configuration runtimeClasspath

# 依赖洞察
./gradlew dependencyInsight --dependency jackson-databind --configuration runtimeClasspath

# 检查依赖更新
./gradlew dependencyUpdates  # 需要 ben-manes.versions 插件
```

# 9. 编译避免 (Compile Avoidance)

## 9.1 原理

编译避免是 Gradle 6.x+ 的核心优化：如果 JAR 的 public API 未改变，依赖它的模块无需重新编译。

```
Module A 依赖 Module B

情况 1: B 的方法体修改 (非 public API)
    B 需重新编译
    A 无需重新编译  -> 编译避免生效!

情况 2: B 的 public 方法签名修改
    B 需重新编译
    A 也需重新编译  -> 编译避免不生效

情况 3: B 的 public 常量修改
    B 需重新编译
    A 也需重新编译  -> 常量内联, 编译避免不生效
```

## 9.2 启用编译避免

```groovy
// 编译避免在 Gradle 6.0+ 默认启用
// 确保使用 implementation 而非 compile
dependencies {
    // implementation: B 的内部实现变更不影响 A
    implementation 'com.example:module-b:1.0.0'

    // api: B 的 API 变更会影响 A (慎重使用)
    api 'com.example:module-c:1.0.0'
}
```

```groovy
// 验证编译避免是否生效
tasks.withType(JavaCompile) {
    // 输出详细的编译避免信息
    // options.compilerArgs << '-XprintRounds'
}
```

## 9.3 API vs Implementation 的性能影响

```
compile 配置 (Gradle 6.x 废弃):
    A 依赖 B 时, B 的所有传递依赖对 A 的编译可见
    任何传递依赖变更都会触发 A 重新编译 -> 编译避免失效

implementation 配置:
    只有 B 自身和 api 依赖对 A 的编译可见
    implementation 依赖的变更不影响 A -> 编译避免最大化

最佳实践: 始终使用 implementation, 仅在必要时使用 api
```

# 10. 注解处理器优化

## 10.1 注解处理器增量编译

```groovy
// 将注解处理器声明为 annotationProcessor 配置
dependencies {
    annotationProcessor 'org.projectlombok:lombok:1.18.32'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'
    implementation 'org.mapstruct:mapstruct:1.5.5.Final'
}

// 注解处理器的增量编译支持 (需要处理器本身支持)
tasks.withType(JavaCompile) {
    options.compilerArgs += [
        '-Adebug',                      // 处理器调试信息
        '-Amapstruct.verbose=true',     // MapStruct 详细输出
        '-Alombok.addLombokGeneratedAnnotation=true'   // Lombok 标记生成代码
    ]
}
```

注解处理器目录结构：

```
build/generated/sources/annotationProcessor/
├── java/main/
│   └── com/example/mapper/
│       └── UserMapperImpl.java       # MapStruct 生成
└── java/test/
    └── ...
```

## 10.2 隔离注解处理器

```groovy
// Gradle 5.2+ 注解处理器隔离 (避免处理器依赖污染运行时)
// 默认已隔离, 但可手动控制

// 如果处理器 A 依赖库 X 版本 1.0, 处理器 B 依赖库 X 版本 2.0
// Gradle 自动为每个处理器创建隔离的类路径
```

# 11. Kotlin DSL 脚本编译优化

## 11.1 Kotlin DSL 脚本编译缓存

```properties
# gradle.properties
# Kotlin DSL 脚本编译缓存
org.gradle.kotlin.dsl.caching.build-cache=true

# Kotlin 编译守护进程 JVM 参数
org.gradle.kotlin.dsl.daemon.jvmargs=-Xmx1g -XX:MaxMetaspaceSize=256m
```

Kotlin DSL 编译流程：

```
*.gradle.kts 文件
        |
        v
Kotlin 脚本编译器 (嵌入在 Gradle Daemon 中)
        |
        v
编译缓存到: ~/.gradle/caches/<version>/kotlin-dsl/
        |
        v
如果脚本未修改, 直接加载缓存的 .class 文件 (快!)
```

## 11.2 预编译脚本插件 (Precompiled Script Plugins)

```kotlin
// buildSrc/src/main/kotlin/my-convention.gradle.kts
plugins {
    java
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(17))
    }
}

tasks.withType<JavaCompile> {
    options.encoding = "UTF-8"
}
```

预编译脚本插件会被编译为 JAR，后续引用时不需要再次编译：

```kotlin
// 子项目 build.gradle.kts
plugins {
    id("my-convention")  // 直接使用已编译的插件
}
```

# 12. Gradle Profiler 使用

## 12.1 安装与基本使用

```bash
# 安装 Gradle Profiler
sdk install gradleprofiler

# 或直接下载
# https://github.com/gradle/gradle-profiler/releases

# 基准测试
gradle-profiler --benchmark --project-dir . --output-dir profile-out assemble

# 生成火焰图 (需要 async-profiler)
gradle-profiler --benchmark --profile async-profiler --project-dir . assemble
```

## 12.2 Profiler 场景文件

```json
// performance.scenarios
{
  "scenarios": [
    {
      "name": "clean-build",
      "title": "Clean Build",
      "tasks": ["clean", "build"],
      "warm-ups": 2,
      "iterations": 5,
      "gradle-args": ["--no-build-cache", "--no-configuration-cache"]
    },
    {
      "name": "incremental-build-no-change",
      "title": "Incremental Build (No Change)",
      "tasks": ["build"],
      "warm-ups": 2,
      "iterations": 5,
      "apply-non-abi-change-to": "src/main/java/com/example/Application.java"
    },
    {
      "name": "incremental-build-abi-change",
      "title": "Incremental Build (ABI Change)",
      "tasks": ["build"],
      "warm-ups": 2,
      "iterations": 5,
      "apply-abi-change-to": "src/main/java/com/example/service/UserService.java"
    }
  ]
}
```

```bash
# 执行场景
gradle-profiler --benchmark --scenario-file performance.scenarios --project-dir .
```

## 12.3 构建扫描 (Build Scan) 分析

```bash
# 生成构建扫描
./gradlew build --scan

# 自动发布构建扫描
./gradlew build --scan -Dscan.tag=ci -Dscan.linkType=public

# 或通过 gradle.properties 配置
# systemProp.org.gradle.build-scan.termsOfServiceUrl=https://gradle.com/terms-of-service
# systemProp.org.gradle.build-scan.termsOfServiceAgree=yes
```

构建扫描报告包含：
- 时间线视图 (Timeline) — 每个任务的开始/结束时间
- 并行执行可视化 — 哪些任务并行运行了
- 依赖解析时间 — 详细展示每个依赖的解析耗时
- 配置时间 — 各模块的配置阶段耗时
- 缓存命中率 — 构建缓存和配置缓存的命中情况
- 内存使用 — Daemon 堆内存变化曲线

# 13. JVM GC 调优

## 13.1 GC 对比与选择

| GC 类型 | 选项 | 特点 | 适用场景 |
|--------|------|------|---------|
| G1GC | `-XX:+UseG1GC` | 低延迟, 分代回收 | 通用 (4GB 以上堆) |
| ZGC | `-XX:+UseZGC` | 亚毫秒暂停, 大堆友好 | 大型 monorepo (JDK 17+) |
| Parallel GC | `-XX:+UseParallelGC` | 高吞吐, 但暂停较长 | 对延迟不敏感的 CI 环境 |
| Shenandoah | `-XX:+UseShenandoahGC` | 低暂停, 大堆 | 大型 monorepo |

## 13.2 GC 日志

```properties
# gradle.properties — GC 日志配置
org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=512m \
  -Xlog:gc*:file=$HOME/.gradle/gc.log:time,level,tags:filecount=5,filesize=10m
```

## 13.3 完整的大项目 JVM 调优配置

```properties
# gradle.properties — 大型项目 (200+ 模块)
org.gradle.jvmargs=-Xmx8g -Xms4g \
  -XX:+UseZGC \
  -XX:+ZGenerational \
  -XX:MaxMetaspaceSize=1g \
  -XX:ReservedCodeCacheSize=512m \
  -XX:SoftRefLRUPolicyMSPerMB=50 \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=$HOME/.gradle/heapdump.hprof \
  -Dfile.encoding=UTF-8 \
  -Dkotlin.daemon.jvm.options=-Xmx2g

org.gradle.parallel=true
org.gradle.workers.max=12
org.gradle.caching=true
org.gradle.configuration-cache=true
org.gradle.vfs.watch=true
org.gradle.configureondemand=true
```

# 14. 性能基准与监控

## 14.1 构建性能度量

```bash
# 显示构建概要 (时间分解)
./gradlew build --profile
# 报告输出到: build/reports/profile/

# 查看 profile 报告
# build/reports/profile/profile-YYYY-MM-DD-HH-mm-ss.html
# 包含: 配置时间, 任务执行时间, 依赖解析时间
```

## 14.2 构建性能回归检测

```groovy
// build.gradle — 利用 Gradle 内置的性能断言 (Gradle Enterprise)
// 在 CI 中对比历史构建性能, 当性能退化时告警
```

## 14.3 各优化手段效果对比

| 优化手段 | 首次构建 | 增量构建 | 复杂度 | 稳定性 |
|---------|---------|---------|--------|--------|
| Daemon | 无影响 | 大幅提升 (30-50%) | 低 | 稳定 |
| parallel | 有提升 (20-40%) | 有提升 | 低 | 稳定 |
| 构建缓存 | 无影响 | 大幅提升 (50-90%) | 中 | 稳定 |
| 配置缓存 | 无影响 | 大幅提升 (40-60%) | 高 | 8.x 稳定 |
| 增量编译 | 无影响 | 大幅提升 (60-80%) | 低 | 稳定 |
| VFS Watch | 有提升 | 有提升 | 低 | 稳定 |
| 依赖锁定 | 有提升 | 有提升 | 低 | 稳定 |
| compile avoidance | 无影响 | 有提升 | 中 | 稳定 |
| GC 调优 | 有提升 | 有提升 | 高 | 视 GC 而定 |

## 14.4 推荐的分阶段实施路径

```
第一阶段 (立即执行):
    - 启用 Daemon (默认)
    - 启用 parallel
    - 调优 JVM 参数 (Xmx, MetaspaceSize)

第二阶段 (1 周内):
    - 启用本地构建缓存
    - 启用 VFS Watch
    - 锁定依赖版本
    - 使用 implementation 替代 compile

第三阶段 (1 月内):
    - 启用配置缓存
    - 设置远程构建缓存 (CI)
    - Kotlin DSL 脚本编译优化

第四阶段 (持续优化):
    - Gradle Profiler 基准测试
    - 构建扫描分析
    - GC 深度调优
```

