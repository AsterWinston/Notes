
# 1. 概述

自定义任务是 Gradle 构建系统的核心扩展点。通过创建自定义 Task 类型，可以将构建逻辑封装为可复用的构建单元。Gradle 提供了丰富的 API 来支持任务的输入/输出声明、增量构建、Worker API 并发执行等高级特性。

## 1.1 自定义任务的本质

在 Gradle 中，每个 Task 本质上是一个实现了 `org.gradle.api.Task` 接口的 Java 类。Gradle 的内部实现中，`DefaultTask` 是所有自定义任务的推荐基类，它由 `TaskContainer` 通过 `ObjectFactory` 或反射机制进行实例化。

```
Gradle 任务生命周期
┌──────────────────────────────────────────────────┐
│  1. 配置阶段 (Configuration Phase)                │
│     ├─ 任务实例化 (Task Instantiation)            │
│     ├─ 任务属性配置 (Property Wiring)              │
│     └─ 任务依赖解析 (Dependency Resolution)        │
│                                                    │
│  2. 执行阶段 (Execution Phase)                     │
│     ├─ 增量检查 (Incremental Check)                │
│     ├─ 输入快照 (Input Snapshot)                   │
│     └─ 动作执行 (Action Execution)                 │
└──────────────────────────────────────────────────┘
```

# 2. DefaultTask 基类

## 2.1 基本结构

`DefaultTask` 是 `org.gradle.api.DefaultTask` 类，提供了任务的基础设施。创建自定义任务的最简单方式是在 Groovy/Kotlin DSL 中扩展此类。

**Groovy 示例：**

```groovy
import org.gradle.api.DefaultTask
import org.gradle.api.tasks.TaskAction

abstract class GreetingTask extends DefaultTask {

    @Input
    abstract Property<String> getMessage()

    GreetingTask() {
        message.convention('Hello, Gradle!')
    }

    @TaskAction
    void greet() {
        println message.get()
    }
}

tasks.register('greet', GreetingTask) {
    message = 'Hello from custom task!'
}
```

**Kotlin 示例：**

```kotlin
import org.gradle.api.DefaultTask
import org.gradle.api.tasks.TaskAction
import org.gradle.api.tasks.Input

abstract class GreetingTask : DefaultTask() {

    @get:Input
    abstract val message: Property<String>

    init {
        message.convention("Hello, Gradle!")
    }

    @TaskAction
    fun greet() {
        println(message.get())
    }
}

tasks.register<GreetingTask>("greet") {
    message.set("Hello from custom task!")
}
```

## 2.2 任务内部机制

`DefaultTask` 内部维护了一个 `TaskStateInternal` 对象来跟踪任务执行状态，以及一个 `TaskActions` 列表来存储所有注册的动作。当任务被执行时，Gradle 会依次调用所有已注册的 `@TaskAction` 方法。

```
DefaultTask 内部结构
┌─────────────────────────────────────┐
│ DefaultTask                         │
│  ├── name: String                   │
│  ├── project: Project               │
│  ├── state: TaskStateInternal       │
│  ├── actions: List<Action>          │
│  ├── inputs: TaskInputs             │
│  ├── outputs: TaskOutputs           │
│  ├── dependsOn: Set<Object>         │
│  ├── finalizedBy: Set<Object>       │
│  └── mustRunAfter: Set<Object>      │
└─────────────────────────────────────┘
```

# 3. 任务动作注解 (@TaskAction)

## 3.1 动作定义

`@TaskAction` 注解用于标记任务中实际执行工作的方法。Gradle 允许在一个任务类中定义多个 `@TaskAction` 方法，它们将按照声明顺序依次执行。

```groovy
import org.gradle.api.DefaultTask
import org.gradle.api.tasks.TaskAction

abstract class MultiStepTask extends DefaultTask {

    @TaskAction
    void stepOne() {
        println "Step 1: 准备数据..."
    }

    @TaskAction
    void stepTwo() {
        println "Step 2: 处理数据..."
    }

    @TaskAction
    void stepThree() {
        println "Step 3: 清理工作..."
    }
}
```

## 3.2 doFirst 与 doLast 的优先级

通过 DSL 注册的 `doFirst` 和 `doLast` 闭包与 `@TaskAction` 注解的执行顺序：

```
执行顺序 (由上到下):
┌──────────────────────────────┐
│ doFirst { }                  │  ← DSL 注册，最先执行
│ @TaskAction (1)              │
│ @TaskAction (2)              │
│ @TaskAction (n)              │
│ doLast { }                   │  ← DSL 注册，最后执行
└──────────────────────────────┘
```

**示例：**

```groovy
tasks.register('ordered', MultiStepTask) {
    doFirst {
        println "=== 任务开始 ==="
    }
    doLast {
        println "=== 任务结束 ==="
    }
}
```

# 4. 任务输入/输出注解

Gradle 的增量构建机制依赖准确的任务输入/输出声明。Gradle 在任务执行前后对输入/输出进行快照比较，如果所有输入快照未发生变化，则跳过执行（UP-TO-DATE）。

## 4.1 输入注解详解

| 注解 | 适用类型 | 说明 |
|------|----------|------|
| `@Input` | 基本类型、String | 通用输入值，参与缓存键计算 |
| `@InputFile` | File, RegularFileProperty | 单个文件输入，路径和内容都参与缓存键 |
| `@InputFiles` | FileCollection, Iterable\<File\> | 多个文件输入 |
| `@InputDirectory` | File, DirectoryProperty | 目录输入，内容敏感 |
| `@Nested` | 自定义类型 | 嵌套对象，递归展开其属性 |
| `@Classpath` | FileCollection | 类路径，仅路径参与缓存键 |
| `@CompileClasspath` | FileCollection | 编译类路径，忽略内部变化 |

## 4.2 输出注解详解

| 注解 | 适用类型 | 说明 |
|------|----------|------|
| `@OutputFile` | File, RegularFileProperty | 单个输出文件 |
| `@OutputFiles` | FileCollection, Iterable\<File\> | 多个输出文件 |
| `@OutputDirectory` | File, DirectoryProperty | 单个输出目录 |
| `@OutputDirectories` | Map\<String, File\> | 多个输出目录 |

## 4.3 完整注解示例

```groovy
import org.gradle.api.DefaultTask
import org.gradle.api.file.*
import org.gradle.api.provider.Property
import org.gradle.api.tasks.*

abstract class ProcessDataTask extends DefaultTask {

    @InputFile
    @PathSensitive(PathSensitivity.RELATIVE)
    abstract RegularFileProperty getInputFile()

    @InputDirectory
    @PathSensitive(PathSensitivity.RELATIVE)
    abstract DirectoryProperty getConfigDir()

    @Input
    abstract Property<String> getMode()

    @Input
    @Optional
    abstract Property<String> getEncoding()

    @OutputFile
    abstract RegularFileProperty getOutputFile()

    @OutputDirectory
    abstract DirectoryProperty getLogDir()

    @Internal
    @Console
    abstract Property<ConsoleOutput> getConsole()

    ProcessDataTask() {
        encoding.convention('UTF-8')
    }

    @TaskAction
    void process() {
        File input = inputFile.get().asFile
        File output = outputFile.get().asFile
        String modeValue = mode.get()
        String encodingValue = encoding.get()

        output.text = input.text.toUpperCase()

        File logDirFile = logDir.get().asFile
        logDirFile.mkdirs()
        new File(logDirFile, 'process.log').text = "Processed with mode: $modeValue"
    }
}
```

## 4.4 内部机制：输入快照与缓存键

Gradle 内部维护了一个 `InputChanges` 系统。当任务执行前，Gradle 会：
1. 遍历所有带注解的属性
2. 对每个属性计算哈希值（SHA-256）
3. 将所有输入哈希组合为任务的缓存键
4. 将缓存键与上一次构建的状态比较

```
缓存键计算流程
┌─────────────────────────────────────┐
│ 1. 收集所有 @Input/@InputFile 属性   │
│ 2. 对每个属性值计算 SHA-256 哈希     │
│ 3. 包含任务类名和方法名到哈希        │
│ 4. 包含 Gradle 版本信息             │
│ 5. 组合所有哈希生成最终缓存键        │
│ 6. 与本地/远程缓存匹配比较          │
└─────────────────────────────────────┘
```

# 5. 内部注解 (@Internal)

`@Internal` 注解用于标记不参与缓存键计算的属性。这对于临时状态、控制台输出对象或其他不影响输出的属性非常有用。

```groovy
abstract class ReportingTask extends DefaultTask {

    @Input
    abstract Property<String> getReportName()

    @Internal
    List<String> warnings = []

    @Internal
    Logger logger = Logging.getLogger(ReportingTask)

    @TaskAction
    void generate() {
        logger.lifecycle("生成报告: ${reportName.get()}")
        warnings.each { logger.warn(it) }
    }
}
```

## 5.1 @Console 注解

`@Console` 注解是 `@Internal` 的扩展，标记一个属性为控制台敏感但不出现在 up-to-date 检查中。通常用于可配置的控制台输出格式。

```groovy
abstract class PrintableTask extends DefaultTask {

    @Input
    abstract Property<String> getContent()

    @Console
    abstract Property<Boolean> getAnsiColor()

    @TaskAction
    void print() {
        if (ansiColor.getOrElse(false)) {
            println "\u001B[31m${content.get()}\u001B[0m"
        } else {
            println content.get()
        }
    }
}
```

# 6. @Optional 与 @SkipWhenEmpty

## 6.1 @Optional

`@Optional` 注解允许输入属性为 null，此时该属性不参与缓存键计算。

```groovy
abstract class OptionalInputTask extends DefaultTask {

    @InputFile
    @Optional
    abstract RegularFileProperty getHeaderFile()

    @InputFile
    abstract RegularFileProperty getContentFile()

    @OutputFile
    abstract RegularFileProperty getOutputFile()
}
```

## 6.2 @SkipWhenEmpty

`@SkipWhenEmpty` 与 `@InputFiles` 或 `@InputDirectory` 配合使用，当输入文件集合为空时跳过任务执行。

```groovy
abstract class BatchCopyTask extends DefaultTask {

    @InputFiles
    @SkipWhenEmpty
    abstract ConfigurableFileCollection getSourceFiles()

    @OutputDirectory
    abstract DirectoryProperty getOutputDir()
}
```

# 7. @PathSensitive

`@PathSensitive` 注解控制输入文件路径对缓存键计算的影响程度。

| 路径敏感度 | 行为 |
|------------|------|
| `ABSOLUTE` | 绝对路径参与缓存键（默认） |
| `RELATIVE` | 仅相对路径参与缓存键 |
| `NAME_ONLY` | 仅文件名参与缓存键 |
| `NONE` | 文件路径完全忽略 |

```groovy
abstract class PathSensitiveTask extends DefaultTask {

    @InputFile
    @PathSensitive(PathSensitivity.RELATIVE)
    abstract RegularFileProperty getConfigFile()

    @InputFiles
    @PathSensitive(PathSensitivity.NAME_ONLY)
    abstract ConfigurableFileCollection getResources()
}
```

**内部机制：** 使用 `RELATIVE` 时，Gradle 将绝对路径中的根部分（如项目目录）移除，这样在不同机器或不同工作目录下也能命中缓存。

# 8. @Nested 注解

`@Nested` 注解用于递归展开自定义类型的属性，使嵌套对象的所有字段都参与缓存键计算。

```groovy
import org.gradle.api.tasks.Nested

abstract class CompilerOptions {
    @Input
    abstract Property<Boolean> getDebug()

    @Input
    @Optional
    abstract Property<String> getTargetVersion()

    @Input
    abstract Property<List<String>> getFlags()
}

abstract class CompileTask extends DefaultTask {

    @Nested
    abstract CompilerOptions getOptions()

    @InputFile
    abstract RegularFileProperty getSourceFile()

    @OutputFile
    abstract RegularFileProperty getClassFile()
}
```

`@Nested` 的内部实现使用 Java Bean 内省来递归发现所有带输入/输出注解的属性。

# 9. @Option 注解

`@Option` 注解将任务属性暴露为命令行选项，方便用户通过 `-P` 或直接参数方式传值。

```groovy
import org.gradle.api.tasks.options.Option

abstract class DeployTask extends DefaultTask {

    @Input
    @Option(option = "env", description = "部署环境 (dev/staging/prod)")
    abstract Property<String> getEnvironment()

    @Input
    @Option(option = "version", description = "部署版本")
    abstract Property<String> getVersion()

    @TaskAction
    void deploy() {
        println "部署版本 ${version.get()} 到 ${environment.get()} 环境"
    }
}
```

使用方式：

```bash
./gradlew deploy --env=production --version=2.0.1
```

## 9.1 @OptionValues

为 `@Option` 提供可选值列表：

```groovy
import org.gradle.api.tasks.options.OptionValues

abstract class DeployTask extends DefaultTask {

    @Option(option = "env", description = "部署环境")
    abstract Property<String> getEnvironment()

    @OptionValues("env")
    List<String> getAvailableEnvironments() {
        return ['dev', 'staging', 'production']
    }
}
```

# 10. 增量任务支持

## 10.1 @Incremental 注解

`@Incremental` 注解允许任务只处理自上次执行以来发生变化的输入文件，大幅提升大型项目的构建性能。

```groovy
import org.gradle.api.tasks.incremental.IncrementalTaskInputs
import org.gradle.work.InputChanges

abstract class IncrementalProcessTask extends DefaultTask {

    @InputFiles
    @Incremental
    @PathSensitive(PathSensitivity.RELATIVE)
    abstract ConfigurableFileCollection getSourceFiles()

    @OutputDirectory
    abstract DirectoryProperty getOutputDir()

    @TaskAction
    void execute(InputChanges inputChanges) {
        if (!inputChanges.isIncremental()) {
            // 非增量模式：全量处理
            project.delete(outputDir.get().asFile)
        }

        // 处理变更的文件
        inputChanges.getFileChanges(sourceFiles).each { change ->
            File targetFile = new File(outputDir.get().asFile, change.normalizedPath)
            switch (change.changeType) {
                case ADDED:
                case MODIFIED:
                    println "处理文件: ${change.file}"
                    targetFile.text = change.file.text.reverse()
                    break
                case REMOVED:
                    println "删除输出: ${targetFile}"
                    targetFile.delete()
                    break
            }
        }
    }
}
```

## 10.2 InputChanges 接口

`InputChanges` 是 Gradle 4.0 引入的新增 API，替代了已废弃的 `IncrementalTaskInputs`。

```groovy
import org.gradle.work.InputChanges
import org.gradle.work.FileChange

abstract class ModernIncrementalTask extends DefaultTask {

    @InputFiles
    @Incremental
    @PathSensitive(PathSensitivity.RELATIVE)
    abstract ConfigurableFileCollection getSourceFiles()

    @OutputDirectory
    abstract DirectoryProperty getDestinationDir()

    @TaskAction
    void execute(InputChanges inputChanges) {
        Iterable<FileChange> changes = inputChanges.getFileChanges(sourceFiles)

        for (FileChange change in changes) {
            File target = new File(destinationDir.get().asFile, change.normalizedPath)

            if (change.fileType == FileType.DIRECTORY) {
                target.mkdirs()
                return
            }

            switch (change.changeType) {
                case ChangeType.ADDED:
                case ChangeType.MODIFIED:
                    target.text = processContent(change.file.text)
                    break
                case ChangeType.REMOVED:
                    target.delete()
                    break
            }
        }
    }

    private String processContent(String content) {
        return content.replaceAll(/old/, 'new')
    }
}
```

## 10.3 增量任务的内部机制

```
增量任务执行流程
┌────────────────────────────────────────┐
│ 1. Gradle 检测 @Incremental 注解        │
│ 2. 遍历输入文件集合                      │
│ 3. 与上次快照比较，生成变更列表          │
│    ├── ADDED: 新增文件                   │
│    ├── MODIFIED: 修改文件                │
│    └── REMOVED: 删除文件                 │
│ 4. 调用 @TaskAction 方法，传入变更信息    │
│ 5. 任务只处理变更的文件                   │
│ 6. 更新输出快照                          │
└────────────────────────────────────────┘
```

# 11. 任务属性类型

Gradle 提供了类型安全的属性类型，支持延迟配置（Lazy Configuration）。

## 11.1 Property 体系

```groovy
import org.gradle.api.provider.Property
import org.gradle.api.provider.ListProperty
import org.gradle.api.provider.SetProperty
import org.gradle.api.provider.MapProperty
import org.gradle.api.file.RegularFileProperty
import org.gradle.api.file.DirectoryProperty
import org.gradle.api.file.ConfigurableFileCollection

abstract class PropertyTypesTask extends DefaultTask {

    // 基本属性
    @Input
    abstract Property<String> getSimpleProperty()

    @Input
    abstract ListProperty<String> getListProperty()

    @Input
    abstract SetProperty<String> getSetProperty()

    @Input
    abstract MapProperty<String, Integer> getMapProperty()

    // 文件属性
    @InputFile
    abstract RegularFileProperty getSingleFile()

    @InputDirectory
    abstract DirectoryProperty getSingleDir()

    // 文件集合
    @InputFiles
    abstract ConfigurableFileCollection getMultiFiles()

    // 配置默认值
    PropertyTypesTask() {
        simpleProperty.convention('default')
        listProperty.convention(['a', 'b', 'c'])
        setProperty.convention(['x', 'y'] as Set)
        mapProperty.convention([key1: 1, key2: 2])
    }
}
```

## 11.2 Property 与 Provider 的链式变换

```groovy
abstract class TransformationTask extends DefaultTask {

    @Input
    abstract Property<String> getInput()

    @Input
    abstract Property<String> getTransformed()

    TransformationTask() {
        // Provider 链式变换：延迟执行
        transformed.set(
            input.map { it.toUpperCase() }
                 .orElse('UNKNOWN')
                 .map { ">>> $it <<<" }
        )
    }
}
```

## 11.3 Property 内部机制

Property 接口定义在 `org.gradle.api.provider` 包中，其实现类（如 `DefaultProperty`）内部使用 `Provider` 来支持延迟求值。当 `get()` 被调用时，Property 会触发其内部的惰性计算链。

```
Property 延迟求值机制
┌──────────────────────────────────┐
│ Property<String>                 │
│  ├── convention: String          │  ← 默认值
│  ├── explicitValue: String       │  ← 显式设置值
│  └── finalValue: Supplier        │  ← 最终求值提供者
│                                   │
│ get() 优先级:                     │
│  explicitValue > convention      │
└──────────────────────────────────┘
```

# 12. 任务配置避免 (Configuration Avoidance)

## 12.1 为什么需要配置避免

Gradle 的配置阶段会执行所有构建脚本内容。如果使用 `task()` 或 `create()` 来创建任务，即使任务永远不会执行，其配置逻辑也会被触发，导致不必要的性能开销。

```groovy
// ❌ 不好的方式：立即配置
task myTask(type: Copy) {  // 即使不执行也会运行配置代码
    from 'src'
    into 'dest'
}

// ✅ 正确的方式：延迟配置
tasks.register('myTask', Copy) {  // 仅在需要时配置
    from 'src'
    into 'dest'
}
```

## 12.2 ObjectFactory 的使用

`ObjectFactory` 是 Gradle 提供的对象工厂，用于创建任务属性实例，避免直接使用 `new`。

```groovy
import org.gradle.api.model.ObjectFactory
import javax.inject.Inject

abstract class ObjectFactoryTask extends DefaultTask {

    private final ObjectFactory objects

    @Inject
    ObjectFactoryTask(ObjectFactory objects) {
        this.objects = objects
    }

    @InputFiles
    @PathSensitive(PathSensitivity.RELATIVE)
    final ConfigurableFileCollection sources = objects.fileCollection()

    @Input
    final ListProperty<String> options = objects.listProperty(String)
}
```

## 12.3 Provider 延迟连接

```groovy
abstract class LazyTask extends DefaultTask {

    @Input
    abstract Property<String> getMessage()

    @Input
    abstract Property<Integer> getRetryCount()

    @Input
    abstract Property<String> getCombinedOutput()

    LazyTask() {
        // 延迟组合两个 Provider
        combinedOutput.set(
            message.zip(retryCount) { msg, count ->
                "$msg (重试次数: $count)"
            }
        )
    }
}

tasks.register('lazy', LazyTask) {
    message = providers.gradleProperty('my.message')
    retryCount = providers.gradleProperty('my.retry').map { it.toInteger() }
}
```

# 13. Worker API

Worker API 允许任务将工作提交到独立的工作线程或进程，实现并发构建和类加载器隔离。

## 13.1 WorkerExecutor

```groovy
import org.gradle.workers.WorkerExecutor
import javax.inject.Inject

abstract class WorkerExampleTask extends DefaultTask {

    @Inject
    abstract WorkerExecutor getWorkerExecutor()

    @InputFiles
    abstract ConfigurableFileCollection getSourceFiles()

    @OutputDirectory
    abstract DirectoryProperty getOutputDir()

    @TaskAction
    void process() {
        WorkerExecutor executor = getWorkerExecutor()

        sourceFiles.each { sourceFile ->
            executor.classLoaderIsolation { workerSpec ->
                workerSpec.classpath.from(configurations.compileClasspath)
            }.submit(ProcessWorkAction) { config ->
                config.sourceFile = sourceFile
                config.outputDir = outputDir.get().asFile
            }
        }
    }
}
```

## 13.2 工作单元定义

```groovy
import org.gradle.workers.WorkAction
import org.gradle.workers.WorkParameters

interface ProcessParams extends WorkParameters {
    RegularFileProperty getSourceFile()
    DirectoryProperty getOutputDir()
}

abstract class ProcessWorkAction implements WorkAction<ProcessParams> {

    @Override
    void execute() {
        File source = parameters.sourceFile.get().asFile
        File output = new File(parameters.outputDir.get().asFile, source.name)

        String processed = source.text.split('\n')
            .collect { line -> line.toUpperCase() }
            .join('\n')

        output.text = processed
        println "工作线程[${Thread.currentThread().name}]: 处理完成 ${source.name}"
    }
}
```

## 13.3 提交模式对比

```groovy
abstract class WorkerModesTask extends DefaultTask {

    @Inject
    abstract WorkerExecutor getWorkerExecutor()

    @TaskAction
    void execute() {

        // 模式 1: 类加载器隔离（默认，推荐）
        workerExecutor.classLoaderIsolation {
            forkOptions { forkOption ->
                forkOption.maxHeapSize = '512m'
                forkOption.jvmArgs = ['-XX:+UseG1GC']
            }
        }.submit(MyWorkAction) { }

        // 模式 2: 进程隔离（完全独立 JVM）
        workerExecutor.processIsolation {
            forkOptions {
                jvmArgs('-Dcustom.property=value')
                maxHeapSize = '1g'
            }
        }.submit(MyWorkAction) { }

        // 模式 3: 无隔离（共享类加载器）
        workerExecutor.noIsolation().submit(MyWorkAction) { }
    }
}
```

## 13.4 Worker API 内部机制

```
Worker 执行流程
┌─────────────────────────────────────────────┐
│ 主构建进程                                    │
│  ├── WorkerExecutor (入口)                   │
│  ├── WorkerDaemonManager                    │
│  │   ├── classLoaderIsolation Daemon 1      │
│  │   │   └── 独立的 ClassLoader             │
│  │   ├── classLoaderIsolation Daemon 2      │
│  │   ├── processIsolation Daemon 1          │
│  │   │   └── 独立的 JVM 进程                │
│  │   └── noIsolation (直接执行)             │
│  └── WorkQueue (等待执行的工件)              │
└─────────────────────────────────────────────┘
```

## 13.5 并行执行与 fork 选项

```groovy
abstract class ParallelWorkerTask extends DefaultTask {

    @Inject
    abstract WorkerExecutor getWorkerExecutor()

    @InputFiles
    abstract ConfigurableFileCollection getSources()

    @OutputDirectory
    abstract DirectoryProperty getOutputDir()

    @TaskAction
    void process() {
        workerExecutor.classLoaderIsolation { spec ->
            spec.forkOptions { fork ->
                fork.maxHeapSize = '1g'
                fork.jvmArgs = [
                    '-XX:+UseParallelGC',
                    '-Djava.security.egd=file:/dev/urandom'
                ]
            }
        }.submit(ParallelWorkAction) { params ->
            params.sourceFiles.setFrom(sources)
            params.outputDir.set(outputDir)
        }
    }
}

interface ParallelWorkParams extends WorkParameters {
    ConfigurableFileCollection getSourceFiles()
    DirectoryProperty getOutputDir()
}

abstract class ParallelWorkAction implements WorkAction<ParallelWorkParams> {

    @Override
    void execute() {
        parameters.sourceFiles.each { file ->
            println "[${Thread.currentThread().name}] 并行处理: $file.name"
            File output = new File(parameters.outputDir.get().asFile,
                                   file.name + '.processed')
            output.text = file.text.toUpperCase()
        }
    }
}
```

| 隔离模式 | 类加载器 | JVM | 适用场景 |
|----------|----------|-----|----------|
| `classLoaderIsolation` | 独立 | 共享 | 需要隔离依赖版本冲突的场景 |
| `processIsolation` | 独立 | 独立 | 需要避免内存泄漏或 JVM 崩溃的场景 |
| `noIsolation` | 共享 | 共享 | 高性能要求的轻量级任务 |

# 14. 任务验证

## 14.1 输入验证

Gradle 提供了 `@Input` 属性的自动验证机制。可以在 `@TaskAction` 执行前验证输入。

```groovy
abstract class ValidatedTask extends DefaultTask {

    @Input
    abstract Property<String> getVersion()

    @Input
    @Optional
    abstract Property<Integer> getPort()

    ValidatedTask() {
        outputs.upToDateWhen { false }
    }

    @TaskAction
    void execute() {
        String versionValue = version.get()
        if (!versionValue.matches(/^\d+\.\d+\.\d+$/)) {
            throw new IllegalArgumentException(
                "版本格式无效: $versionValue, 期望格式: X.Y.Z"
            )
        }

        Integer portValue = port.getOrNull()
        if (portValue != null && (portValue < 1024 || portValue > 65535)) {
            throw new IllegalArgumentException("端口号超出范围: $portValue")
        }

        println "任务执行: version=$versionValue, port=$portValue"
    }
}
```

## 14.2 任务启用/禁用

```groovy
abstract class ConditionalTask extends DefaultTask {

    @Input
    abstract Property<Boolean> getEnabled()

    ConditionalTask() {
        onlyIf("启用检查") {
            enabled.getOrElse(true)
        }
    }

    @TaskAction
    void execute() {
        println "任务已启用，正在执行..."
    }
}
```

# 15. 任务依赖管理

## 15.1 dependsOn 声明

```groovy
abstract class DependentTask extends DefaultTask {

    @Input
    abstract Property<String> getData()

    @TaskAction
    void execute() {
        println "处理数据: ${data.get()}"
    }
}

tasks.register('taskA', DependentTask) {
    data = 'A'
}

tasks.register('taskB', DependentTask) {
    data = 'B'
}

tasks.register('taskC', DependentTask) {
    data = 'C'
    dependsOn 'taskA', 'taskB'  // 依赖多个任务
}
```

## 15.2 动态依赖

```groovy
tasks.register('dynamicDeps', DependentTask) {
    data = 'dynamic'

    // 延迟解析依赖
    dependsOn tasks.withType(DependentTask).matching { task ->
        task.name.startsWith('task')
    }
}
```

## 15.3 TaskDependency 接口

`TaskDependency` 是 Gradle 内部用于表示任务依赖关系的接口。所有 `dependsOn` 参数最终都被包装为 `TaskDependency` 实现。

```groovy
import org.gradle.api.Task
import org.gradle.api.tasks.TaskDependency
import java.util.concurrent.Callable

abstract class DynamicDependencyTask extends DefaultTask {

    @Input
    abstract Property<String> getTargetTaskName()

    DynamicDependencyTask() {
        dependsOn new TaskDependency() {
            @Override
            Set<? extends Task> getDependencies(Task task) {
                String name = targetTaskName.get()
                Task target = task.project.tasks.findByName(name)
                return target != null ? [target] as Set : [] as Set
            }
        }
    }
}
```

# 16. 任务排序

## 16.1 mustRunAfter 与 shouldRunAfter

```groovy
tasks.register('first') {
    doLast { println 'first' }
}

tasks.register('second') {
    doLast { println 'second' }
    mustRunAfter 'first'  // 强制排序
}

tasks.register('third') {
    doLast { println 'third' }
    shouldRunAfter 'second'  // 软排序
}

// 执行: gradle second third   -->   first (自动触发), second, third
// 注意: mustRunAfter 不创建依赖关系，只在两者都执行时生效
```

## 16.2 finalizedBy

```groovy
tasks.register('cleanup') {
    doLast { println '清理临时文件' }
}

tasks.register('mainTask') {
    doLast { println '主要任务执行' }
    finalizedBy 'cleanup'  // 无论成功或失败都会执行
}
```

# 17. 任务描述与分组

```groovy
abstract class DocumentedTask extends DefaultTask {

    DocumentedTask() {
        group = 'custom'
        description = '这是一个描述清晰的自定义任务'
    }

    @TaskAction
    void execute() {
        println "执行: $description"
    }
}
```

## 17.1 任务报告

```bash
# 查看所有任务
gradle tasks

# 查看特定分组
gradle tasks --group=custom

# 查看任务详情
gradle help --task=customTask
```

# 18. 测试自定义任务 (TestKit)

## 18.1 TestKit 基本设置

TestKit 允许在功能测试中执行 Gradle 构建，验证自定义任务行为。

```groovy
// build.gradle 中的测试配置
plugins {
    id 'groovy'  // 或 id 'java-gradle-plugin'
}

dependencies {
    testImplementation gradleTestKit()
    testImplementation 'org.spockframework:spock-core:2.3-groovy-3.0'
}

tasks.withType(Test).configureEach {
    useJUnitPlatform()
    systemProperty 'isTest', 'true'
}
```

## 18.2 功能测试示例

```groovy
import org.gradle.testkit.runner.GradleRunner
import spock.lang.Specification
import spock.lang.TempDir

class GreetingTaskFunctionalTest extends Specification {

    @TempDir
    File testProjectDir

    File buildFile
    File settingsFile

    def setup() {
        settingsFile = new File(testProjectDir, 'settings.gradle')
        buildFile = new File(testProjectDir, 'build.gradle')

        buildFile << """
            abstract class GreetingTask extends org.gradle.api.DefaultTask {
                @org.gradle.api.tasks.Input
                abstract org.gradle.api.provider.Property<String> getMessage()

                GreetingTask() {
                    message.convention('Hello!')
                }

                @org.gradle.api.tasks.TaskAction
                void greet() {
                    println "[TASK] " + message.get()
                }
            }

            tasks.register('greet', GreetingTask)
        """.stripIndent()
    }

    def "当消息配置时应该打印正确内容"() {
        given:
        buildFile << """
            tasks.named('greet', GreetingTask) {
                message = '自定义问候'
            }
        """

        when:
        def result = GradleRunner.create()
            .withProjectDir(testProjectDir)
            .withArguments('greet', '--stacktrace')
            .build()

        then:
        result.output.contains('[TASK] 自定义问候')
        result.task(':greet').outcome == org.gradle.testkit.runner.TaskOutcome.SUCCESS
    }

    def "增量构建应该跳过未变更的任务"() {
        when:
        def firstRun = GradleRunner.create()
            .withProjectDir(testProjectDir)
            .withArguments('greet')
            .build()

        def secondRun = GradleRunner.create()
            .withProjectDir(testProjectDir)
            .withArguments('greet')
            .build()

        then:
        firstRun.task(':greet').outcome == org.gradle.testkit.runner.TaskOutcome.SUCCESS
        secondRun.task(':greet').outcome == org.gradle.testkit.runner.TaskOutcome.UP_TO_DATE
    }

    def "当输入文件变更时应重新执行"() {
        given:
        buildFile.text = """
            abstract class FileTask extends org.gradle.api.DefaultTask {
                @org.gradle.api.tasks.InputFile
                abstract org.gradle.api.file.RegularFileProperty getInputFile()

                @org.gradle.api.tasks.OutputFile
                abstract org.gradle.api.file.RegularFileProperty getOutputFile()

                @org.gradle.api.tasks.TaskAction
                void process() {
                    outputFile.get().asFile.text = inputFile.get().asFile.text.toUpperCase()
                }
            }

            tasks.register('fileTask', FileTask) {
                inputFile = layout.projectDirectory.file('input.txt')
                outputFile = layout.buildDirectory.file('output.txt')
            }
        """.stripIndent()

        def inputFile = new File(testProjectDir, 'input.txt')
        inputFile.text = 'hello world'

        when:
        def firstRun = GradleRunner.create()
            .withProjectDir(testProjectDir)
            .withArguments('fileTask')
            .build()

        inputFile.text = 'updated world'

        def secondRun = GradleRunner.create()
            .withProjectDir(testProjectDir)
            .withArguments('fileTask')
            .build()

        then:
        firstRun.task(':fileTask').outcome == org.gradle.testkit.runner.TaskOutcome.SUCCESS
        secondRun.task(':fileTask').outcome == org.gradle.testkit.runner.TaskOutcome.SUCCESS

        def outputFile = new File(testProjectDir, 'build/output.txt')
        outputFile.text == 'UPDATED WORLD'
    }
}
```

## 18.3 测试调试

```groovy
// 查看 Gradle 构建的调试输出
def result = GradleRunner.create()
    .withProjectDir(testProjectDir)
    .withArguments('greet', '--info', '--stacktrace')
    .withDebug(true)  // 启用调试模式（端口 5005）
    .build()
```

# 19. @Classpath 与 @CompileClasspath

```groovy
abstract class ClasspathTask extends DefaultTask {

    @Classpath
    abstract ConfigurableFileCollection getRuntimeClasspath()

    @CompileClasspath
    abstract ConfigurableFileCollection getCompileClasspath()

    @OutputFile
    abstract RegularFileProperty getOutputFile()

    @TaskAction
    void execute() {
        // @Classpath: 仅文件路径集合参与缓存键，文件内容变化不影响增量
        println "运行时类路径: ${runtimeClasspath.files*.name}"

        // @CompileClasspath: 行为类似 @Classpath，但标准化规则更激进
        // 内部文件变更不会导致缓存失效（因为编译器行为相同）
        println "编译类路径: ${compileClasspath.files*.name}"
    }
}
```

# 20. 总结

| 特性 | 用途 | 关键注解/接口 |
|------|------|---------------|
| 输入输出声明 | 增量构建支持 | @Input, @InputFile, @OutputFile, @OutputDirectory |
| 路径敏感度 | 跨环境缓存复用 | @PathSensitive |
| 嵌套属性 | 复杂输入类型 | @Nested |
| 增量输入 | 高效处理文件变更 | @Incremental, InputChanges |
| 延迟配置 | 构建性能优化 | Property, Provider, ConfigurableFileCollection |
| Worker API | 并发和隔离执行 | WorkerExecutor, WorkAction, ClassLoaderIsolation |
| 命令行选项 | 用户可配置 | @Option, @OptionValues |
| 测试验证 | 质量保证 | TestKit, GradleRunner |

自定义任务是 Gradle 扩展性的核心。合理使用输入/输出注解、增量支持、Worker API 和 Property 体系，能够构建出高效、可维护、可在多环境复用的构建逻辑。
