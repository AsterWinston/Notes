
# 1. 任务的定义与创建

## 1.1 使用 `register()` 与 `create()` 创建任务

Gradle 提供两种主要方式在构建脚本中声明任务：`tasks.register()`（延迟配置）和 `tasks.create()`（立即配置）。Gradle 5.0+ 强烈推荐使用 `register()`，因为它支持**任务配置避免（Task Configuration Avoidance）**机制。

```groovy
// 推荐：延迟创建 — 仅在任务需要执行时才配置
tasks.register('hello') {
    doLast {
        println 'Hello, Gradle!'
    }
}

// 不推荐：立即创建 — 始终配置，即使任务不会被运行
tasks.create('world') {
    doLast {
        println 'World!'
    }
}
```

```kotlin
// Kotlin DSL 等价写法
tasks.register("hello") {
    doLast {
        println("Hello, Gradle!")
    }
}

tasks.create("world") {
    doLast {
        println("World!")
    }
}
```

**内部机制**：`register()` 返回 `TaskProvider<T>`，这是一个惰性包装器。实际的 Task 对象仅在以下情况被实例化：（1）任务被明确加入执行图；（2）通过 `provider.get()` 或 `named()` 显式获取。`create()` 则立即实例化 `DefaultTask` 对象并加入 `TaskContainer`。

```groovy
// TaskProvider 的惰性特性
def myTask = tasks.register('deferred') {
    doLast { println 'Executed!' }
}
// 此时 DefaultTask 尚未创建，内存占用更低

// 显式获取会触发实例化
Task t = myTask.get()
```

## 1.2 任务动作（Actions）：`doLast` 与 `doFirst`

每个任务维护一个 `actions` 列表（`List<Action<? super Task>>`）。`doFirst` 在列表头部插入动作，`doLast` 在尾部追加。

```groovy
tasks.register('ordered') {
    doLast { println 'A: 主逻辑' }
    doFirst { println 'B: 前置操作' }
    doLast { println 'C: 清理操作' }
}
// 输出顺序：B → A → C
```

**内部实现**：`DefaultTask` 内部使用 `actions`（`CopyOnWriteArrayList`）存储动作列表。执行时按列表顺序依次调用。

```groovy
// 多个 doLast 按添加顺序执行
tasks.register('multi-action') {
    doLast { println 'Step 1' }
    doLast { println 'Step 2' }
    doLast { println 'Step 3' }
}
```

## 1.3 使用 `TaskContainer` 动态创建任务

```groovy
// 根据文件列表动态批量创建任务
fileTree('src').each { file ->
    def taskName = "process${file.name.capitalize()}"
    tasks.register(taskName) {
        doLast { println "Processing ${file.name}" }
    }
}
```

```kotlin
// Kotlin DSL 动态创建
fileTree("src").forEach { file ->
    tasks.register("process${file.name.capitalize()}") {
        doLast { println("Processing ${file.name}") }
    }
}
```

# 2. 任务依赖关系

## 2.1 `dependsOn` — 强依赖

`dependsOn` 是最基础的依赖声明。被依赖的任务总是在当前任务之前执行，且如果被依赖任务失败，当前任务不会执行。

```groovy
tasks.register('compile') {
    doLast { println 'Compiling...' }
}

tasks.register('package') {
    dependsOn 'compile'
    doLast { println 'Packaging...' }
}

tasks.register('upload') {
    dependsOn 'package'
    doLast { println 'Uploading...' }
}
// 执行 upload 将触发：compile → package → upload
```

`dependsOn` 支持多种参数形式：

```groovy
tasks.register('taskA') {
    // 字符串名称
    dependsOn 'compile'
    // 多个任务
    dependsOn ['compile', 'test']
    // Task 对象引用
    dependsOn tasks.named('compile')
    // Provider（惰性引用）
    dependsOn tasks.named('compile')
    // Callable / Closure（动态解析）
    dependsOn {
        tasks.findAll { it.name.startsWith('lib') }
    }
    // 绝对路径（跨项目）
    dependsOn ':lib:compileJava'
}
```

**注意**：`dependsOn` 中添加对 `clean` 的依赖是**反模式**，应使用 `finalizedBy` 或 `mustRunAfter`。

## 2.2 `mustRunAfter` — 严格顺序（非依赖）

`mustRunAfter` 仅规定执行顺序，**不引入依赖关系**。如果任务 B `mustRunAfter` 任务 A，同时执行 A 和 B 时，B 一定在 A 之后。但如果只执行 B 而不执行 A，B 仍可独立运行。

```groovy
tasks.register('integrationTest') {
    mustRunAfter 'test'
    doLast { println 'Integration tests' }
}
// 执行 integrationTest 不会自动触发 test
// 执行 test integrationTest 时，test 一定在前
```

## 2.3 `shouldRunAfter` — 软顺序（非依赖）

与 `mustRunAfter` 类似，但如果引入了循环依赖，`shouldRunAfter` 会被忽略而不报错。

```groovy
tasks.register('taskX') {
    shouldRunAfter 'taskY'
}
tasks.register('taskY') {
    shouldRunAfter 'taskX'
}
// 循环依赖下不报错，因为 shouldRunAfter 是软性的
```

## 2.4 `finalizedBy` — 终结任务

无论任务成功还是失败，`finalizedBy` 指定的任务都会执行。常用于清理操作。

```groovy
tasks.register('deploy') {
    finalizedBy 'notify'
    doLast { println 'Deploying...' }
}

tasks.register('notify') {
    doLast { println 'Sending notification...' }
}
// deploy 执行后 notify 一定会执行（即使 deploy 抛出异常）
```

**典型场景**：

```groovy
// 测试后始终生成报告
tasks.named('test') {
    finalizedBy 'jacocoTestReport'
}

// 清理临时文件
tasks.register('build') {
    finalizedBy 'cleanTemp'
}
```

## 2.5 依赖传递性

Gradle 任务依赖是**传递的**。如果 A → B → C（A dependsOn B，B dependsOn C），执行 A 时会按 C → B → A 的顺序执行。

```groovy
tasks.register('a') { dependsOn 'b' }
tasks.register('b') { dependsOn 'c' }
tasks.register('c') { doLast { println 'C' } }
// gradle a → 输出 C（c → b → a 依次执行）
```

**注意**：`dependsOn` 是传递的，但 `mustRunAfter`/`shouldRunAfter` **不是**传递的。

# 3. 任务配置 vs 执行

## 3.1 配置阶段与执行阶段

```groovy
tasks.register('demo') {
    // ===== 配置阶段（Configuration Phase）=====
    // 这里的代码在每次构建的配置阶段都会执行
    println 'Configuring demo task'

    doFirst {
        // ===== 执行阶段（Execution Phase）=====
        // 只有任务被调度时才执行
        println 'Execution: doFirst'
    }
    doLast {
        println 'Execution: doLast'
    }
}
// 执行 `gradle help` — 不运行 demo，但"Configuring demo task"仍会打印
// 执行 `gradle demo` — 所有消息都会打印
```

## 3.2 惰性属性配置

```groovy
// 反模式：在配置阶段读取可能不存在的文件
tasks.register('bad') {
    def content = file('data.txt').text  // 配置阶段即读取
    doLast { println content }
}

// 推荐：在执行阶段读取
tasks.register('good') {
    doLast {
        def content = file('data.txt').text  // 执行阶段读取
        println content
    }
}
```

## 3.3 使用 `Provider` 延迟属性解析

```groovy
tasks.register('lazy') {
    // Provider 在配置阶段不解析值，在执行阶段才解析
    onlyIf { Provider { file('input.txt').exists() } }
    inputs.file(project.layout.projectDirectory.file('input.txt'))
    outputs.file(project.layout.buildDirectory.file('output.txt'))
}
```

# 4. 任务输入与输出

## 4.1 增量构建基础

Gradle 通过比较任务的**输入**和**输出**的时间戳/内容哈希，决定是否可以**跳过**（UP-TO-DATE）或需要**重新执行**。

```groovy
abstract class MyTask extends DefaultTask {
    @Input
    abstract Property<String> getInputText()

    @InputFile
    abstract RegularFileProperty getSourceFile()

    @OutputFile
    abstract RegularFileProperty getOutputFile()

    @TaskAction
    void execute() {
        def content = inputText.get() + sourceFile.get().asFile.text
        outputFile.get().asFile.text = content.toUpperCase()
    }
}
```

## 4.2 输入输出注解详解

| 注解 | 用途 | 影响增量判断 |
|------|------|-------------|
| `@Input` | 简单值（String, int, boolean） | 值变更触发重执行 |
| `@InputFile` | 单个输入文件 | 路径+内容变更触发 |
| `@InputFiles` | 多个输入文件（`ConfigurableFileCollection`） | 任意文件变更触发 |
| `@InputDirectory` | 输入目录 | 目录内容变更触发 |
| `@OutputFile` | 单个输出文件 | 不存在或内容不符触发 |
| `@OutputFiles` | 多个输出文件 | - |
| `@OutputDirectory` | 输出目录 | - |
| `@Internal` | 不参与增量判断 | - |
| `@Nested` | 嵌套对象 | 递归检查其属性 |
| `@SkipWhenEmpty` | 配合 @InputFiles，集合为空时跳过 | - |
| `@Destroys` | 声明任务可能破坏的文件 | 破坏增量链 |
| `@LocalState` | 任务本地状态 | 不参与增量判断 |
| `@Console` | 控制台关联属性 | 不参与增量判断 |
| `@Classpath` | 类路径输入 | 类似 @InputFiles |
| `@CompileClasspath` | 编译类路径 | 顺序敏感 |

```groovy
abstract class ProcessData extends DefaultTask {
    @InputFile
    @SkipWhenEmpty
    abstract ConfigurableFileCollection getSourceFiles()

    @Input
    abstract Property<String> getMode()

    @Nested
    abstract Options getOptions()

    @OutputDirectory
    abstract DirectoryProperty getOutputDir()

    @Internal
    abstract Property<String> getDebugLog()  // 不影响增量

    @TaskAction
    void process() {
        // 处理逻辑
    }
}
```

## 4.3 增量构建 API v2（`@InputChanges`）

Gradle 7.0+ 支持增量任务 API v2，允许任务只处理变更的文件，而非全量处理。

```groovy
abstract class IncrementalProcessor extends DefaultTask {
    @InputFiles
    @Incremental
    abstract ConfigurableFileCollection getInputFiles()

    @OutputDirectory
    abstract DirectoryProperty getOutputDir()

    @TaskAction
    void execute(InputChanges inputChanges) {
        // 检查是否是增量构建
        if (!inputChanges.isIncremental()) {
            project.delete(outputDir.get().asFile)
        }

        // 遍历变更文件
        inputChanges.getFileChanges(inputFiles).each { change ->
            if (change.fileType == FileType.DIRECTORY) return

            def targetFile = outputDir.file(change.normalizedPath).get().asFile
            when (change.changeType) {
                ChangeType.ADDED, ChangeType.MODIFIED -> {
                    targetFile.text = processFile(change.file)
                }
                ChangeType.REMOVED -> {
                    targetFile.delete()
                }
            }
        }
    }

    private String processFile(File input) {
        return input.text.toUpperCase()
    }
}
```

**`InputChanges` 的限制**：
- 仅适用于任务**运行后再次运行**的场景（有历史记录）
- 首次运行或 `clean` 后，`isIncremental()` 返回 `false`
- `@InputFiles` 必须是 `ConfigurableFileCollection`

## 4.4 声明输入输出关系（`@Incremental`）

```kotlin
// Kotlin DSL 增量任务
abstract class IncrementalTask : DefaultTask() {
    @get:Incremental
    @get:InputDirectory
    abstract val inputDir: DirectoryProperty

    @get:OutputDirectory
    abstract val outputDir: DirectoryProperty

    @TaskAction
    fun execute(inputChanges: InputChanges) {
        val outputDir = outputDir.get().asFile
        if (!inputChanges.isIncremental) {
            outputDir.deleteRecursively()
        }
        inputChanges.getFileChanges(inputDir).forEach { change ->
            val target = outputDir.resolve(change.normalizedPath)
            when (change.changeType) {
                ChangeType.ADDED, ChangeType.MODIFIED -> {
                    target.writeText(change.file.readText().uppercase())
                }
                ChangeType.REMOVED -> target.delete()
            }
        }
    }
}
```

## 4.5 增量构建失效场景

```groovy
// 这些操作会使增量构建失效：
tasks.register('breaks-incremental') {
    doLast {
        // 1. 直接操作输出目录的父目录
        delete 'build/outputs'
        // 2. 在配置阶段修改输入
        // 3. 使用 @Input 但值不稳定的属性（如 new Date()）
        // 4. 使用 @Internal 标注实际应该参与增量的属性
        // 5. 任务抛出异常后输出不完整（clean output）
    }
}
```

# 5. 任务规则（Task Rules）

## 5.1 `addRule()` 动态任务创建

任务规则允许在不预先定义任务的情况下，根据命名模式动态创建任务。

```groovy
tasks.addRule('Pattern: ping<Hostname>') { String taskName ->
    if (taskName.startsWith('ping')) {
        tasks.register(taskName) {
            doLast {
                def host = taskName - 'ping'
                println "Pinging ${host}..."
            }
        }
    }
}

// 运行 gradle pingGoogle — 动态创建 pingGoogle 任务
// 运行 gradle pingBaidu — 动态创建 pingBaidu 任务
```

**高级用法 — 文件扩展名规则**：

```groovy
tasks.addRule('Pattern: compile<ID>@<Extension>') { String taskName ->
    if (taskName.startsWith('compile')) {
        tasks.register(taskName) {
            doLast {
                def idAndExt = (taskName - 'compile').split('@')
                println "Compiling ${idAndExt[0]}.${idAndExt[1]}"
            }
        }
    }
}
// gradle compileFoo@java  → 编译 Foo.java
// gradle compileBar@kt    → 编译 Bar.kt
```

## 5.2 Kotlin DSL 的任务规则

```kotlin
tasks.addRule("Pattern: run<Config>") { taskName ->
    if (taskName.startsWith("run")) {
        tasks.register(taskName) {
            doLast {
                val config = taskName.removePrefix("run")
                println("Running with config: $config")
            }
        }
    }
}
```

# 6. 默认任务（Default Tasks）

```groovy
// build.gradle
defaultTasks 'clean', 'build'

// 或指定多个
defaultTasks = ['clean', 'test', 'build']
```

```kotlin
// Kotlin DSL (settings.gradle.kts 或 build.gradle.kts)
defaultTasks("clean", "build")
```

当执行 `gradle`（不带参数）时，默认运行 `clean` 和 `build`。

# 7. `gradle.taskGraph` — 任务执行图

## 7.1 图准备回调：`whenReady`

```groovy
gradle.taskGraph.whenReady { TaskExecutionGraph graph ->
    println "将执行 ${graph.allTasks.size()} 个任务："
    graph.allTasks.each { println "  - ${it.path}" }

    // 按需禁用任务
    graph.allTasks.each { task ->
        if (task.name.startsWith('integration') && !project.hasProperty('runIntegration')) {
            task.enabled = false
        }
    }
}
```

## 7.2 任务执行前后回调：`beforeTask` / `afterTask`

```groovy
gradle.taskGraph.beforeTask { Task task ->
    println "[开始] ${task.path} at ${System.currentTimeMillis()}"
}

gradle.taskGraph.afterTask { Task task, TaskState state ->
    if (state.failure) {
        println "[失败] ${task.path}: ${state.failure.message}"
    } else if (state.skipped) {
        println "[跳过] ${task.path} - ${state.skipMessage}"
    } else {
        println "[完成] ${task.path}"
    }
}
```

## 7.3 检查特定任务是否在图中

```groovy
gradle.taskGraph.whenReady { graph ->
    if (graph.hasTask(':app:assembleRelease')) {
        println 'Release 构建模式'
    }
    if (graph.hasTask(':lib:test')) {
        println '将运行库测试'
    }
}
```

## 7.4 使用 `allTasks` 探索依赖链

```groovy
gradle.taskGraph.whenReady { graph ->
    def target = graph.allTasks.find { it.name == 'build' }
    if (target) {
        println "build 任务将执行以下依赖："
        printDependencies(target, 0)
    }
}

void printDependencies(Task task, int depth) {
    println "${'  ' * depth}${task.path}"
    task.taskDependencies.getDependencies(task).each {
        printDependencies(it, depth + 1)
    }
}
```

# 8. 跳过任务

## 8.1 `onlyIf` — 条件谓词

```groovy
tasks.register('deploy') {
    onlyIf { project.hasProperty('release') }
    doLast { println 'Deploying...' }
}
// gradle deploy           → 跳过
// gradle deploy -Prelease → 执行
```

```groovy
// 多个 onlyIf 必须全部满足（AND 逻辑）
tasks.register('publish') {
    onlyIf { file('release.flag').exists() }
    onlyIf { !System.getenv('CI').toBoolean() }
    onlyIf { project.hasProperty('signingKey') }
    doLast { println 'Publishing...' }
}
```

## 8.2 `StopExecutionException` — 任务内部终止

```groovy
tasks.register('conditional') {
    doFirst {
        if (!file('config.yml').exists()) {
            throw new StopExecutionException('配置文件缺失，跳过执行')
        }
    }
    doLast {
        println '正常处理...'
    }
}
// 抛出 StopExecutionException 会使任务被标记为 SKIPPED 而非 FAILED
```

**注意**：`StopExecutionException` 跳出当前 action 但仍会执行 `finalizedBy` 任务。

## 8.3 `enabled` 属性

```groovy
tasks.named('slowIntegrationTest') { enabled = false }

// 动态控制
gradle.taskGraph.whenReady { graph ->
    if (graph.hasTask(':test')) {
        tasks.named('slowTest').configure { enabled = false }
    }
}
```

## 8.4 跳过策略对比

| 方法 | 任务状态 | 是否执行 finalizers | 持久化状态 |
|------|---------|-------------------|-----------|
| `enabled = false` | SKIPPED | 否 | 是（改变后始终保持） |
| `onlyIf { false }` | SKIPPED | 否 | 每次评估 |
| `StopExecutionException` | SKIPPED | 是 | 运行时决定 |
| UP-TO-DATE（增量） | UP-TO-DATE | 否 | 自动判断 |

# 9. 任务超时控制

## 9.1 `timeout` 属性

```groovy
import java.time.Duration

tasks.register('longRunning') {
    timeout = Duration.ofMinutes(5)
    doLast {
        // 长时间操作
    }
}
```

```kotlin
// Kotlin DSL
tasks.register("longRunning") {
    timeout.set(Duration.ofMinutes(5))
    doLast { /* ... */ }
}
```

超时后 Gradle 抛出 `TimeoutException`，任务标记为 FAILED。

## 9.2 带超时的重试逻辑

```groovy
tasks.register('flaky') {
    doLast {
        def maxRetries = 3
        for (int i = 0; i < maxRetries; i++) {
            try {
                performFlakyOperation()
                break
            } catch (Exception e) {
                if (i == maxRetries - 1) throw e
                Thread.sleep(2000)
            }
        }
    }
}
```

# 10. 并行任务执行

## 10.1 启用并行执行

```properties
# gradle.properties
org.gradle.parallel=true
org.gradle.workers.max=4
```

或在命令行：

```bash
gradle build --parallel --max-workers=4
```

## 10.2 并行执行的前提条件

并行执行要求任务之间没有隐式依赖。Gradle 通过任务图拓扑排序确定哪些任务可以并行执行。

```groovy
// Project A 的任务
tasks.register('taskA1') { dependsOn 'taskB1' }
tasks.register('taskA2')

// Project B 的任务
tasks.register('taskB1')
tasks.register('taskB2')

// taskA1 → taskB1（跨项目依赖）
// taskA2 和 taskB2 无依赖 → 可并行执行
```

## 10.3 并行与 `mustRunAfter` 的交互

```groovy
// 即使声明了 mustRunAfter，Gradle 也不会在无依赖的任务间建立强顺序，
// 但 MUST-RUN-AFTER 关系在并行调度中仍被遵守。
tasks.register('step1') { mustRunAfter 'setup' }
tasks.register('step2') { mustRunAfter 'setup' }
// step1 和 step2 可在 setup 之后并行执行
```

## 10.4 Worker API 内部并行

```groovy
import org.gradle.workers.WorkerExecutor
import javax.inject.Inject

abstract class ParallelTask extends DefaultTask {
    @Inject
    abstract WorkerExecutor getWorkerExecutor()

    @TaskAction
    void execute() {
        // 提交多个工作项到隔离的 Worker 守护进程
        (1..10).each { id ->
            workerExecutor.submit(MyWorkAction) { config ->
                config.params(id)
                config.isolationMode = IsolationMode.PROCESS
            }
        }
    }
}

interface MyWorkParameters extends WorkParameters {
    Property<Integer> getId()
}

abstract class MyWorkAction implements WorkAction<MyWorkParameters> {
    @Override
    void execute() {
        println "Processing ${parameters.id.get()}"
    }
}
```

# 11. 任务排序最佳实践

## 11.1 依赖声明决策树

```
是否需要 B 在 A 之前执行，且 A 不能脱离 B 运行？
├── 是 → dependsOn
└── 否 → 是否需要严格顺序？
    ├── 是 → mustRunAfter
    └── 否 → shouldRunAfter
是否需要 A 执行完始终执行 B？
└── 是 → finalizedBy
```

## 11.2 常见反模式

```groovy
// 反模式 1：在配置阶段执行 I/O
tasks.register('bad') {
    def data = file('input.txt').text  // 配置阶段读文件
}

// 反模式 2：dependsOn clean
tasks.register('bad2') {
    dependsOn 'clean'  // 破坏了增量构建
}

// 反模式 3：将任务依赖与执行逻辑混合
tasks.register('bad3') {
    doLast {
        tasks.named('compile').get().execute()  // 永远不要手动调用 execute()
    }
}
```

## 11.3 任务分组与描述

```groovy
tasks.register('generateDocs') {
    group = 'documentation'
    description = '生成项目 API 文档'
    doLast { /* ... */ }
}
```

这样在 `gradle tasks` 输出中会按组分类显示，提升可发现性。

---

**总结**：Gradle 任务系统是整个构建引擎的核心。理解任务的生命周期、依赖模型和增量机制，是高效使用 Gradle 的基础。核心要点：（1）优先使用 `register()` 而非 `create()`；（2）正确声明 inputs/outputs 以获得增量构建收益；（3）使用 `dependsOn`/`mustRunAfter`/`finalizedBy` 构建清晰的任务依赖图；（4）利用 Worker API 实现任务内部并行化。
