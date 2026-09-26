
# 1. 三阶段概述

Gradle 构建生命周期严格分为三个阶段，每个阶段有明确的职责边界：

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Phase 1: 初始化 (Initialization)                  │
│                                                                      │
│  ┌──────────────────┐    ┌──────────────────┐    ┌───────────────┐  │
│  │ 读取 settings.   │───▶│ 解析 include /   │───▶│ 创建 Project  │  │
│  │ gradle(.kts)     │    │ includeBuild     │    │ 对象层次结构   │  │
│  └──────────────────┘    └──────────────────┘    └───────────────┘  │
│                                                                      │
│  输入: Settings 对象                                                  │
│  产物: Project 树 + Gradle 实例                                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   Phase 2: 配置 (Configuration)                       │
│                                                                      │
│  ┌──────────────────┐    ┌──────────────────┐    ┌───────────────┐  │
│  │ 按顺序执行各      │───▶│ 解析 build.gradle │───▶│ 构建任务注册  │  │
│  │ 项目构建脚本      │    │ 配置插件/任务/扩展 │    │ 依赖关系图    │  │
│  └──────────────────┘    └──────────────────┘    └───────────────┘  │
│                                                                      │
│  输入: 所有 Project 对象                                              │
│  产物: 完整配置的 Project + 任务注册表                                 │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Phase 3: 执行 (Execution)                          │
│                                                                      │
│  ┌──────────────────┐    ┌──────────────────┐    ┌───────────────┐  │
│  │ 构建任务执行 DAG  │───▶│ 拓扑排序并调度   │───▶│ 依次执行 Action│  │
│  │ (有向无环图)      │    │ 并行分配 Worker   │    │ 收集产出 + 报告│  │
│  └──────────────────┘    └──────────────────┘    └───────────────┘  │
│                                                                      │
│  输入: 要执行的任务名称                                               │
│  产物: 任务执行结果 (SUCCESS / FAILED / SKIPPED / UP-TO-DATE)         │
└─────────────────────────────────────────────────────────────────────┘
```

# 2. 初始化阶段详解

## 2.1 Settings 评估流程

初始化阶段的核心是 `Settings` 对象的评估：

```groovy
// settings.gradle
rootProject.name = 'my-app'

// 1. 注册子项目
include 'core', 'app', 'data'

// 2. 配置插件管理
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}

// 3. 配置依赖解析管理
dependencyResolutionManagement {
    repositories {
        mavenCentral()
    }
    versionCatalogs {
        create('libs') {
            from(files('gradle/libs.versions.toml'))
        }
    }
}

// 4. 注册构建内包含
includeBuild 'gradle/plugins'

// 5. 监听 Settings 评估完成
gradle.settingsEvaluated { settings ->
    println "项目树: ${settings.rootProject.allprojects.collect { it.path }}"
}
```

## 2.2 项目确定过程

```groovy
// Settings 内部运行时调用链
// Settings.executeSettingsScript()
//   ├── Settings.include(String[])
//   │     └── DefaultProjectDescriptorRegistry.addProject()
//   ├── Settings.includeBuild()
//   │     └── CompositeBuildContext.include()
//   └── Settings.includeFlat()
//         └── 同级目录中的项目
```

**关键点**：`Project` 对象在初始化阶段创建但**尚未配置**（即 `build.gradle` 尚未执行）。每个 `Project` 在此时仅有一个路径、名称和目录。

## 2.3 `Settings.findProject()` 方法

```groovy
// settings.gradle
gradle.settingsEvaluated { settings ->
    // 按路径查找（最可靠）
    def core = settings.findProject(':core')
    println "Core project dir: ${core?.projectDir}"

    // 按名称查找（可能在多个项目间歧义）
    def app = settings.findProject('app')

    // 检查项目是否存在
    if (settings.findProject(':optional') == null) {
        println 'optional 模块不存在'
    }
}
```

**注意**：`settings.findProject()` 在 Settings 评估期间可用，但在配置阶段应使用 `project.findProject()`。

## 2.4 `Gradle.settingsEvaluated` 事件

```groovy
// settings.gradle
gradle.settingsEvaluated { gradle ->
    // 此时所有 Project 对象已创建，但尚未配置
    def allProjects = gradle.rootProject.allprojects

    // 为每个项目附加元数据
    allProjects.each { project ->
        project.ext.buildTime = System.currentTimeMillis()
    }

    // 注册全局构建服务
    gradle.sharedServices.registerIfAbsent('globalCache', MyCacheService) {
        // 配置
    }
}

abstract class MyCacheService implements BuildService<MyCacheService.Params>, AutoCloseable {
    interface Params extends BuildServiceParameters {}

    private final Map<String, Object> cache = [:]

    Object get(String key) { cache[key] }
    void put(String key, Object value) { cache[key] = value }

    @Override
    void close() { cache.clear() }
}
```

## 2.5 初始化阶段的能力与限制

```groovy
// settings.gradle

// ✓ 可以做
rootProject.name = 'custom-name'                    // 设置根项目名
include ':lib'                                      // 注册子项目
project(':lib').name = 'library'                    // 重命名子项目
project(':lib').projectDir = file('libs/library')   // 自定义目录
project(':lib').buildFileName = 'lib.gradle'        // 自定义构建文件名
pluginManagement { repositories { ... } }           // 插件仓库配置
dependencyResolutionManagement { ... }              // 依赖仓库配置

// ✗ 不可以做
// apply plugin: 'java'       — plugins {} 块不可用（但 settings 本身可用）
// tasks.register('x')        — tasks 在此阶段不可用
// dependencies { ... }       — dependencies 在此阶段不可用
```

# 3. 配置阶段详解

## 3.1 项目评估机制

每个 `Project` 通过 `AbstractProject.evaluate()` 进入配置阶段。Gradle 内部维护一个**项目评估顺序**：

```
根项目 (:) → 直接子项目（字母顺序）
  ↓ (如果 :app evaluationDependsOn ':core')
  :core → :app → :data → :integration-test
```

```groovy
// Project 内部 evaluate 调用链
// Project.evaluate()
//   ├── 执行 build.gradle 脚本
//   ├── 执行初始化脚本（init scripts）
//   ├── 应用 settings 中声明的插件
//   ├── 执行 beforeEvaluate 回调
//   ├── 执行 build.gradle 闭包
//   └── 执行 afterEvaluate 回调
```

## 3.2 `beforeEvaluate` / `afterEvaluate` 钩子

```groovy
// 根项目 build.gradle — 监听特定子项目的生命周期
project(':core') {
    // 在 core 被评估前执行
    beforeEvaluate {
        println "[${it.path}] 评估开始..."
    }

    // 在 core 被评估完成后执行
    afterEvaluate { project ->
        println "[${project.path}] 评估完成，共 ${project.tasks.size()} 个任务"

        // 此时可以安全地操作任务
        if (project.plugins.hasPlugin('java')) {
            project.tasks.named('compileJava').configure {
                options.encoding = 'UTF-8'
            }
        }
    }
}

// 根项目的 afterEvaluate
afterEvaluate {
    println "根项目配置完成"
    subprojects.each { sub ->
        println "  - ${sub.path}: ${sub.tasks.size()} tasks"
    }
}
```

**关键陷阱**：

```groovy
// 反模式：在 afterEvaluate 内配置 — 破坏配置避免
project.afterEvaluate {
    tasks.withType(JavaCompile) {
        options.encoding = 'UTF-8'  // 立即配置所有 JavaCompile 任务
    }
}

// 推荐：使用 configureEach 延迟配置
tasks.withType(JavaCompile).configureEach {
    options.encoding = 'UTF-8'
}
```

## 3.3 `ProjectEvaluationListener`

```groovy
// 编程方式注册评估监听器
gradle.addProjectEvaluationListener(new ProjectEvaluationListener() {
    @Override
    void beforeEvaluate(Project project) {
        println ">>> 评估前: ${project.path}"
    }

    @Override
    void afterEvaluate(Project project, ProjectState state) {
        if (state.failure) {
            println "<<< 评估失败: ${project.path}: ${state.failure.message}"
        } else {
            println "<<< 评估成功: ${project.path}"
        }
    }
})
```

## 3.4 配置阶段状态图

```
                    ┌──────────┐
                    │  CREATED │  (刚创建，路径/名称已确定)
                    └────┬─────┘
                         │ beforeEvaluate 回调
                         ▼
                    ┌──────────┐
                    │EVALUATING│  (正在执行 build.gradle)
                    └────┬─────┘
                         │ afterEvaluate 回调
                         ▼
                    ┌──────────┐
               ┌───│CONFIGURED│  (配置完成)
               │   └──────────┘
               │         │ task graph building
               │         ▼
               │   ┌──────────┐
               │   │ EXECUTING│  (任务正在运行)
               │   └────┬─────┘
               │        │ buildFinished
               │        ▼
               │   ┌──────────┐
               └───│ FINISHED │
                   └──────────┘
```

## 3.5 惰性配置 vs 立即配置

```groovy
// 立即配置（Eager）— 不管任务是否需要都会配置
tasks.create('eager') {
    // 配置代码在配置阶段执行
    println "配置立即任务"  // 任何 gradle 命令都会打印
    doLast { println "执行" }
}

// 惰性配置（Lazy）— 只在任务需要时才配置
tasks.register('lazy') {
    // 配置代码仍然在配置阶段执行，但 DefaultTask 实例延迟创建
    println "配置惰性任务"  // 任然会在配置阶段打印！
    doLast { println "执行" }
}
```

**真正的区别在于 Task 对象的创建时机**：

```groovy
// register 返回 TaskProvider，Task 对象延迟创建
def provider = tasks.register('delay') { ... }
// 此时 DefaultTask 尚未实例化

// 仅当以下情况才实例化 Task：
// 1. provider.get() 被调用
// 2. tasks.named('delay').get() 被调用
// 3. 任务被加入执行图
```

## 3.6 配置避免（Configuration Avoidance）API

```groovy
// 反模式 — 立即配置
tasks.withType(JavaCompile) {
    sourceCompatibility = '17'
}

// 推荐 — 配置避免
tasks.withType(JavaCompile).configureEach {
    sourceCompatibility = '17'
}

// 反模式
tasks.named('compileJava').get()  // 立即获取 Task 对象

// 推荐
tasks.named('compileJava').configure {  // 惰性配置
    options.compilerArgs << '-verbose'
}
```

**`configureEach` 的内部行为**：
- 为将来创建的所有匹配任务注册一个回调
- 当前已存在的任务立即执行该回调
- 后续添加的匹配任务也会在创建时执行该回调
- 不影响 Task 对象的创建时机

# 4. 执行阶段详解

## 4.1 任务执行图（Task Execution Graph）

执行阶段从**任务图构建**开始：

```groovy
// Gradle 内部执行流程
// 1. 用户指定任务名称（从命令行）
// 2. Gradle 按名称在 TaskContainer 中查找
// 3. 解析每个请求任务的传递依赖
// 4. 构建 DAG（有向无环图）
// 5. 拓扑排序
// 6. 按排序结果执行
```

**任务图构建代码（内部伪代码）**：

```groovy
class TaskExecutionGraph {
    void populate(List<String> taskNames) {
        // 从命令行任务名称开始
        def requestedTasks = taskNames.collect { findTask(it) }

        // 遍历依赖图（BFS/DFS）
        def visited = [] as Set
        def queue = requestedTasks as Queue
        while (!queue.isEmpty()) {
            def task = queue.poll()
            if (visited.add(task)) {
                task.taskDependencies.getDependencies(task).each { dep ->
                    queue.add(dep)
                }
            }
        }
        // 拓扑排序
        this.executionPlan = topologicalSort(visited)
    }
}
```

## 4.2 `TaskExecutionGraph` API

```groovy
// 图准备就绪时
gradle.taskGraph.whenReady { TaskExecutionGraph graph ->
    // 此时所有任务依赖已解析，图已构建完成
    println "=== 任务执行计划 ==="
    println "总任务数: ${graph.allTasks.size()}"

    // 获取特定任务
    def compileTasks = graph.allTasks.findAll { it.name.startsWith('compile') }
    println "编译任务: ${compileTasks.size()} 个"

    // 检查任务是否存在
    if (graph.hasTask(':app:test')) {
        println "将运行 :app:test"
    }

    // 按需设置
    graph.allTasks.each { task ->
        if (task.name.contains('integration') && !project.hasProperty('runIT')) {
            task.enabled = false
        }
    }

    // 依赖分析
    graph.allTasks.each { task ->
        def deps = graph.getDependencies(task)
        if (deps) {
            println "  ${task.path} ← ${deps*.path}"
        }
    }
}

// 任务执行前后
gradle.taskGraph.beforeTask { Task task ->
    println "[EXEC] ${task.path} 开始执行"
}

gradle.taskGraph.afterTask { Task task, TaskState state ->
    def status = state.skipped ? 'SKIPPED' : state.failure ? 'FAILED' : 'OK'
    println "[EXEC] ${task.path} → ${status} (${state.didWork ? '有工作' : '无工作'})"
}
```

## 4.3 `TaskExecutionGraphListener`

```groovy
// 通过 Gradle 实例注册
gradle.addListener(new TaskExecutionGraphListener() {
    @Override
    void graphPopulated(TaskExecutionGraph graph) {
        println "任务图已填充: ${graph.allTasks.size()} 个任务"
    }
})

// 通过 Project 实例注册
project.gradle.taskGraph.addTaskExecutionGraphListener(new TaskExecutionGraphListener() {
    @Override
    void graphPopulated(TaskExecutionGraph graph) {
        // 等价于 whenReady
    }
})
```

## 4.4 `TaskState` 详情

```groovy
gradle.taskGraph.afterTask { task, state ->
    println "任务: ${task.path}"
    println "  执行了: ${state.executed}"    // 是否已执行
    println "  有工作: ${state.didWork}"     // TaskAction 产生了副作用
    println "  跳过原因: ${state.skipMessage}" // SKIPPED / UP-TO-DATE
    println "  失败原因: ${state.failure?.message}" // 异常信息
    println "  未捕获异常: ${state.failure != null}"
    println "  重执行: ${state.noSource}"    // NO-SOURCE 跳过
}
```

## 4.5 构建操作监听（BuildOperationListener）

Gradle 提供了一个底层的 `BuildOperationListener`，可以监听所有构建操作（包括内部操作）：

```groovy
// 这个 API 是内部的（org.gradle.internal.operations），但可以通过插件访问
// 通常通过 Gradle Enterprise 或 build scan 插件使用

// 工作中可通过 build scan 观察操作树：
// gradle build --scan
// 扫描报告显示：任务执行时间、依赖解析、工件转换等详细操作
```

# 5. 全局构建监听器

## 5.1 `BuildListener` 接口

```groovy
gradle.addBuildListener(new BuildAdapter() {
    @Override
    void projectsLoaded(Gradle gradle) {
        println "初始化完成: ${gradle.rootProject.allprojects.size()} 个项目"
    }

    @Override
    void projectsEvaluated(Gradle gradle) {
        println "配置完成: 所有项目已评估"
        int totalTasks = gradle.rootProject.allprojects.sum { it.tasks.size() }
        println "注册任务总数: ${totalTasks}"
    }

    @Override
    void buildFinished(BuildResult result) {
        if (result.failure) {
            println "构建失败: ${result.failure.message}"
            result.failure.printStackTrace()
        } else {
            println "构建成功"
        }
    }
})
```

## 5.2 事件监听注册时机

```groovy
// settings.gradle — 最早的注册时机
gradle.settingsEvaluated { settings ->
    // 在此注册监听器
    settings.gradle.addBuildListener(new BuildAdapter() {
        @Override
        void projectsLoaded(Gradle gradle) {
            println "[LISTENER] 项目加载完成"
        }
    })
}

// init.gradle — 全局初始化脚本
gradle.settingsEvaluated { settings ->
    settings.gradle.projectsLoaded {
        println "[INIT] 项目加载完成"
    }
}
```

# 6. 配置缓存（Configuration Cache）

## 6.1 配置缓存机制

Gradle 7.0+ 引入配置缓存功能，将配置阶段的结果序列化到磁盘，后续构建直接反序列化以跳过配置阶段。

```properties
# gradle.properties
org.gradle.configuration-cache=true
```

```bash
# 命令行
gradle build --configuration-cache
```

**配置缓存的生命周期**：

```
首次构建:
  初始化 → 配置(完整) → 序列化配置 → 执行
                                      │
                                      └──→ .gradle/configuration-cache/

后续构建:
  初始化 → 读取缓存 → 执行 (跳过配置阶段)
```

## 6.2 配置缓存的限制

```groovy
// ✗ 破坏配置缓存的操作
tasks.register('bad') {
    doLast {
        // 1. 使用外部进程的状态
        def gitCommit = 'git rev-parse HEAD'.execute().text

        // 2. 读取系统时间（应使用 Provider）
        def now = new Date()
    }
}

// ✓ 配置缓存兼容的写法
abstract class SafeTask extends DefaultTask {
    @Input
    abstract Property<String> getGitCommit()

    @TaskAction
    void execute() {
        println "Commit: ${gitCommit.get()}"
    }
}

// 使用 Gradle 的 ValueSource 获取外部状态
abstract class GitCommitValueSource implements ValueSource<String, ValueSourceParameters.None> {
    @Override
    String obtain() {
        return 'git rev-parse HEAD'.execute().text.trim()
    }
}

tasks.register('safe', SafeTask) {
    def gitProvider = providers.of(GitCommitValueSource) {}
    gitCommit.set(gitProvider)
}
```

## 6.3 配置缓存报告

```bash
# 检查配置缓存问题
gradle help --configuration-cache --configuration-cache-problems=warn

# 查看缓存状态
ls -la .gradle/configuration-cache/
```

Gradle 在缓存失败时会输出详细报告，指出哪些代码打破了缓存。

# 7. `gradle.buildFinished` 事件

## 7.1 构建完成处理

```groovy
gradle.buildFinished { BuildResult result ->
    def duration = System.currentTimeMillis() - gradle.rootProject.ext.buildStart
    println "构建耗时: ${duration}ms"

    if (result.failure) {
        // 记录失败
        def logFile = file("${gradle.rootProject.buildDir}/build-failure.log")
        logFile.parentFile.mkdirs()
        logFile.text = """
            |Time: ${new Date()}
            |Failure: ${result.failure.message}
            |${result.failure.stackTrace}
        """.stripMargin().trim()

        // 在 CI 中设置输出参数
        println "##[error]构建失败: ${result.failure.message}"
    } else {
        println "构建成功完成"
    }
}
```

## 7.2 在 CI 管道中使用

```groovy
// 持续集成中的构建后步骤
def ci = System.getenv('CI') == 'true'

if (ci) {
    gradle.buildFinished { result ->
        // 收集测试结果
        def testResults = fileTree("${buildDir}/test-results").include('**/*.xml')
        // 收集覆盖率报告
        def coverage = file("${buildDir}/reports/jacoco/test/jacocoTestReport.xml")

        // 上传制品
        if (result.failure == null) {
            println "##vso[task.setvariable variable=build.status]success"
        }
    }
}
```

# 8. 性能分析器（Profiler Integration）

## 8.1 Gradle Profiler

```bash
# 安装 Gradle Profiler
sdk install gradleprofiler

# 基准测试
gradle-profiler --benchmark --project-dir . \
    --scenario assemble \
    --warmups 3 --iterations 5 \
    --output-dir build/profile-results

# 分析
gradle-profiler --profile analyze --project-dir .
```

## 8.2 `--profile` 标志

```bash
gradle build --profile
```

生成 `build/reports/profile/profile-*.html`，包含：
- 配置阶段每项目的耗时
- 依赖解析耗时
- 任务执行耗时（含状态变化）

## 8.3 编程方式的性能分析

```groovy
// 在构建中记录自定义计时
gradle.taskGraph.beforeTask { task ->
    task.ext.startTime = System.nanoTime()
}

gradle.taskGraph.afterTask { task, state ->
    def duration = System.nanoTime() - (task.ext.startTime as long)
    task.ext.durationNs = duration
    if (duration > 1_000_000_000) {  // 超过 1 秒
        println "⚠ 慢任务: ${task.path} (${duration / 1_000_000}ms)"
    }
}

gradle.buildFinished { result ->
    // 找出最耗时的 5 个任务
    def slowTasks = gradle.rootProject.allprojects
        .collect { it.tasks }
        .flatten()
        .findAll { it.hasProperty('durationNs') }
        .sort { -it.durationNs }
        .take(5)

    println "\n=== 最耗时任务 TOP 5 ==="
    slowTasks.eachWithIndex { task, i ->
        println "${i + 1}. ${task.path}: ${task.durationNs / 1_000_000}ms"
    }
}
```

# 9. 初始化脚本（Init Scripts）

## 9.1 全局初始化脚本

```groovy
// ~/.gradle/init.gradle
gradle.settingsEvaluated { settings ->
    println "[INIT] Gradle ${settings.gradle.gradleVersion}"
}

allprojects {
    // 为所有项目添加公共仓库
    repositories {
        maven { url 'https://nexus.mycompany.com/repository/public/' }
        mavenCentral()
    }
}

// 全局缓存配置
gradle.settingsEvaluated { settings ->
    settings.caches {
        // 配置全局缓存
    }
}
```

## 9.2 项目级初始化脚本

```bash
# 通过命令行指定
gradle build --init-script common.gradle

# 或放在特定位置
# $GRADLE_HOME/init.d/*.gradle  (自动加载)
# $USER_HOME/.gradle/init.d/*.gradle (自动加载)
```

# 10. 生命周期完整示例

## 10.1 综合监听器注册

```groovy
// settings.gradle — 监听构建全过程
gradle.settingsEvaluated { settings ->
    settings.gradle.allprojects { project ->
        project.beforeEvaluate {
            println "[0] 项目评估前: ${project.path}"
        }
        project.afterEvaluate {
            println "[1] 项目评估完: ${project.path}"
        }
    }

    settings.gradle.projectsLoaded { gradle ->
        println "[2] 所有项目已加载"
    }

    settings.gradle.beforeProject { project ->
        println "[3] 开始评估: ${project.path}"
    }

    settings.gradle.afterProject { project ->
        println "[4] 完成评估: ${project.path}"
    }

    settings.gradle.projectsEvaluated { gradle ->
        println "[5] 项目评估阶段结束"
    }

    settings.gradle.taskGraph.whenReady { graph ->
        println "[6] 任务图就绪: ${graph.allTasks.size()} 个任务"
    }

    settings.gradle.taskGraph.beforeTask { task ->
        println "[7] 任务执行前: ${task.path}"
    }

    settings.gradle.taskGraph.afterTask { task, state ->
        println "[8] 任务执行后: ${task.path} (${state.skipped ? 'SKIP' : 'OK'})"
    }

    settings.gradle.buildFinished { result ->
        println "[9] 构建完成: ${result.failure ? 'FAIL' : 'SUCCESS'}"
    }
}
```

**执行 `gradle :core:build` 的完整输出顺序**：

```
[0] 项目评估前: :          ← beforeEvaluate
[3] 开始评估: :           ← beforeProject
[4] 完成评估: :           ← afterProject
[1] 项目评估完: :          ← afterEvaluate
[0] 项目评估前: :core     ← 按顺序评估子项目
[3] 开始评估: :core
[4] 完成评估: :core
[1] 项目评估完: :core
[2] 所有项目已加载          ← projectsLoaded (实际上在初始化完成时触发)
[5] 项目评估阶段结束         ← projectsEvaluated
[6] 任务图就绪: N 个任务     ← whenReady
[7] 任务执行前: :core:compileJava
[8] 任务执行后: :core:compileJava
[7] 任务执行前: :core:classes
...
[9] 构建完成: SUCCESS
```

## 10.2 构建阶段的不可逆性

```groovy
// 反模式：在配置阶段尝试引执行阶段的方法
project.tasks.register('build').configure {
    doLast {
        // 不能在任务执行时再去修改其他任务的配置
        // tasks.named('compile').configure { ... }  // 配置阶段已结束，太晚了
    }
}

// 正确做法：在配置阶段完成所有配置
tasks.named('compile').configure {
    dependsOn 'preProcess'
}
```

# 11. 配置阶段的可视化

## 11.1 使用 `--dry-run` 观察任务图

```bash
# 查看将要执行哪些任务，但不实际执行
gradle build --dry-run

# 输出示例:
# :core:compileJava SKIPPED
# :core:processResources SKIPPED
# :core:classes SKIPPED
# :core:jar SKIPPED
# :core:assemble SKIPPED
# :core:compileTestJava SKIPPED
# :core:processTestResources SKIPPED
# :core:testClasses SKIPPED
# :core:test SKIPPED
# :core:check SKIPPED
# :core:build SKIPPED
```

## 11.2 使用 `--console=verbose` 显示详细信息

```bash
gradle build --console=verbose
# 显示每个任务的详细状态：EXECUTED / UP-TO-DATE / FROM-CACHE / SKIPPED / NO-SOURCE
```

---

**总结**：深入理解 Gradle 的三阶段生命周期是高效开发和调试的关键。核心要点：（1）初始化阶段仅创建项目结构，不执行任务配置；（2）配置阶段执行构建脚本但**不运行任务**，善用 `configureEach` 实现配置避免；（3）执行阶段构建 DAG 并调度任务，利用 `whenReady` 对任务图做最后调整；（4）使用 `BuildListener` 和 `TaskExecutionGraphListener` 监控构建全过程；（5）配置缓存（Configuration Cache）可大幅缩短配置阶段时间，但需确保代码兼容；（6）性能分析器（`--profile`、`--scan`）是优化构建速度的利器。
