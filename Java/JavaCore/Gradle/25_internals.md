
# 1. Gradle 架构概览

## 1.1 整体架构层次

```
┌─────────────────────────────────────────────────────────────┐
│                      用户构建脚本                            │
│          build.gradle / build.gradle.kts / settings           │
├─────────────────────────────────────────────────────────────┤
│                      Tooling API (IDE 连接层)                 │
├─────────────────────────────────────────────────────────────┤
│                      插件系统 (Plugin System)                 │
│   ┌──────────┬──────────┬──────────┬──────────────────┐    │
│   │  Java    │  Android │  Kotlin  │  自定义插件       │    │
│   └──────────┴──────────┴──────────┴──────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│                      Core API (核心 API)                     │
│   Project / Task / Configuration / Extension / Provider      │
├─────────────────────────────────────────────────────────────┤
│                    构建引擎 (Build Engine)                    │
│   ┌──────────────────────────────────────────────────────┐  │
│   │ 任务执行引擎    │  依赖解析引擎   │  文件系统引擎    │  │
│   │ TaskExecution  │  Dependency     │  FileSystem      │  │
│   │ Engine         │  Resolution     │  Operations      │  │
│   └──────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                    基础设施层 (Infrastructure)                 │
│   Daemon / ClassLoader / Logging / ServiceRegistry / Caching  │
└─────────────────────────────────────────────────────────────┘
```

## 1.2 核心组件交互

```
用户执行 gradle <task> 命令
         │
         ▼
  ┌──────────────┐     ┌──────────────────┐
  │  GradleMain   │────▶│ DefaultGradle    │
  │  (入口点)      │     │ Launcher         │
  └──────────────┘     └──────────────────┘
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
          ┌─────────────┐          ┌────────────────┐
          │ Settings     │          │ Project        │
          │ Script       │          │ Script         │
          │ Compilation  │          │ Compilation    │
          └─────────────┘          └────────────────┘
                    │                       │
                    ▼                       ▼
          ┌─────────────┐          ┌────────────────┐
          │ Settings     │          │ Project        │
          │ Evaluation   │          │ Evaluation     │
          └─────────────┘          └────────────────┘
                                           │
                                           ▼
                                 ┌────────────────┐
                                 │ Task Graph     │
                                 │ Calculation    │
                                 └────────────────┘
                                           │
                                           ▼
                                 ┌────────────────┐
                                 │ Task           │
                                 │ Execution      │
                                 └────────────────┘
```

# 2. BuildEngine 与 BuildOperation

## 2.1 BuildOperation 机制

BuildOperation 是 Gradle 内部的核心可观测性机制。每个重要的内部操作都会记录为一个 BuildOperation，形成树形结构。

```java
// 内部接口 (org.gradle.internal.operations)
public interface BuildOperation {
    BuildOperationDescriptor.Builder description();
}

public interface BuildOperationExecutor {
    <T> T run(BuildOperation operation);
    // 现代版本使用 BuildOperationRunner
}

public interface BuildOperationListener {
    void started(BuildOperationDescriptor buildOperation, OperationStartEvent startEvent);
    void progress(OperationIdentifier operationIdentifier, OperationProgressEvent progressEvent);
    void finished(BuildOperationDescriptor buildOperation, OperationFinishEvent finishEvent);
}
```

```
BuildOperation 树形结构示例：
Run main
├── Load settings                                (86ms)
│   ├── Compile settings file                    (12ms)
│   └── Evaluate settings                        (74ms)
├── Configure project :app                       (320ms)
│   ├── Apply plugin: java                       (45ms)
│   ├── Apply plugin: org.springframework.boot   (150ms)
│   ├── Resolve dependencies                     (80ms)
│   └── Configure tasks                          (45ms)
├── Calculate task graph                         (15ms)
└── Execute tasks                                (1200ms)
    ├── Execute :compileJava                     (500ms)
    │   ├── Resolve compile classpath            (100ms)
    │   ├── Compile Java sources                 (380ms)
    │   └── Store in build cache                 (20ms)
    └── Execute :test                            (700ms)
        ├── Resolve test classpath               (50ms)
        └── Execute tests                        (650ms)
```

## 2.2 BuildOperation 在构建扫描中的作用

```groovy
// 构建扫描中的 BuildOperation 时序图数据来源
// BuildScanPlugin 注册了全局 BuildOperationListener
// 所有 BuildOperation 都被记录并上传到构建扫描服务器

// 自定义 BuildOperation (高级用法)
// 通过 Gradle Internal API (不推荐在生产插件中使用)
import org.gradle.internal.operations.BuildOperationRunner

// 内置操作类型
// org.gradle.internal.operations.trace.BuildOperationTrace
// 可通过 -Dorg.gradle.internal.operations.trace=... 导出
```

```bash
# 导出 BuildOperation 跟踪文件
gradle build -Dorg.gradle.internal.operations.trace=build-ops.log
```

# 3. 任务执行引擎

## 3.1 任务图构建过程

```
配置阶段结束
      │
      ▼
TaskGraph 计算：
1. 从入口任务开始（如 build）
2. 递归查找所有依赖 (dependsOn)
3. 解析 finalizedBy / shouldRunAfter / mustRunAfter
4. 构建有向无环图 (DAG)
5. 拓扑排序得出执行顺序
      │
      ▼
并行执行调度：
┌────────────────────────────────────────────┐
│              Task Executor                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Worker 1 │  │ Worker 2 │  │ Worker N │  │
│  │ :cJ      │  │ :pT      │  │ :jar     │  │
│  │ :tC      │  │ :t       │  │          │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│       Task Scheduling & Worker Allocation   │
└────────────────────────────────────────────┘
```

```java
// 任务调度器内部 (简化)
public class DefaultTaskExecutionGraph implements TaskExecutionGraph {
    
    public void execute(Collection<? extends Task> entryTasks) {
        // 1. 计算任务图
        TaskDependencyGraph graph = calculateDependencyGraph(entryTasks);
        
        // 2. 按并行度分配 worker
        int maxWorkers = getMaxWorkers();
        ExecutorService executor = Executors.newFixedThreadPool(maxWorkers);
        
        // 3. 拓扑排序
        List<Task> sorted = topologicalSort(graph);
        
        // 4. 执行（保持依赖顺序）
        for (Task task : sorted) {
            if (task.getAllDependencies().allSatisfied()) {
                executor.submit(() -> executeTask(task));
            }
        }
    }
}
```

## 3.2 任务执行管道

```
Task Execution Pipeline (每条任务经过的步骤)：

Execute :compileJava
    │
    ├─ 1. SkipOnlyIf        检查 onlyIf { ... } 条件
    │
    ├─ 2. SkipEmpty         检查源是否为空
    │
    ├─ 3. Check Up-to-Date  比较输入/输出快照
    │   ├─ 输入未变 → UP-TO-DATE (跳过)
    │   └─ 输入改变 → 继续
    │
    ├─ 4. From Cache        从构建缓存恢复输出
    │   ├─ 缓存命中 → FROM-CACHE (跳过)
    │   └─ 缓存未命中 → 继续
    │
    ├─ 5. Execute           执行实际任务动作
    │   └─ 调用 @TaskAction 方法或 doLast 闭包
    │
    └─ 6. Store Cache       将输出存入构建缓存
```

## 3.3 Worker API 内部

```groovy
// Worker API 允许在独立 JVM (或同一进程隔离 ClassLoader) 中执行工作
// 内部实现

// 1. 提交工作
WorkerExecutor worker = project.workerExecutor
worker.submit(MyWorkAction.class) { config ->
    config.isolationMode = IsolationMode.PROCESS  // 或 CLASSLOADER
    config.forkOptions { jvmArgs("-Xmx512m") }
    config.params(someParameter)
}

// 内部执行流程：
// a. WorkQueue 接收提交
// b. WorkerProcessManager 管理进程池
// c. Serializer 序列化参数
// d. 子进程反序列化参数并执行
// e. 结果序列化返回

// 子进程通信协议：
// ┌──────────┐         socket/named-pipe         ┌──────────┐
// │ Daemon   │◀────────── 双向通信 ────────────▶│ Worker   │
// │ Process  │           (Kryo 序列化)             │ Process  │
// └──────────┘                                    └──────────┘
```

## 3.4 增量任务实现

```java
// Gradle 如何判断一个任务是 UP-TO-DATE
public class TaskExecution {
    
    public TaskState execute(Task task) {
        // 1. 收集所有输入快照
        Map<String, Object> inputs = collectInputSnapshots(task);
        
        // 2. 收集所有输出快照（如果存在）
        Map<String, Object> outputs = collectOutputSnapshots(task);
        
        // 3. 计算输入哈希
        HashCode inputHash = hashCode(inputs);
        
        // 4. 比较上次执行的哈希
        HashCode previousHash = loadPreviousHash(task.getPath());
        
        if (inputHash.equals(previousHash) && outputsExistAndValid(outputs)) {
            task.getState().setOutcome(TaskOutcome.UP_TO_DATE);
            return TaskState.UP_TO_DATE;
        }
        
        // 5. 检查构建缓存
        if (buildCacheEnabled) {
            BuildCacheEntry cached = buildCache.find(task.getPath(), inputHash);
            if (cached != null) {
                restoreOutputs(cached);
                task.getState().setOutcome(TaskOutcome.FROM_CACHE);
                return TaskState.FROM_CACHE;
            }
        }
        
        // 6. 执行任务
        executeTaskAction(task);
        
        // 7. 存储新快照和缓存
        saveInputHash(task.getPath(), inputHash);
        buildCache.store(task.getPath(), inputHash, task.getOutputs().getFiles());
        
        return TaskState.EXECUTED;
    }
}
```

# 4. 依赖解析引擎

## 4.1 依赖解析流程

```
依赖声明 (dependencies { implementation 'g:a:v' })
        │
        ▼
  ┌──────────────┐
  │ 配置信息收集   │  读取 Gradle module metadata / POM
  │ Configuration │
  └──────────────┘
        │
        ▼
  ┌──────────────┐
  │ 依赖图构建     │  递归解析传递依赖
  │ Dependency    │  处理 excludes, constraints
  │ Graph         │
  └──────────────┘
        │
        ▼
  ┌──────────────┐
  │ 版本冲突解决   │  选择每个模块的"最佳"版本
  │ Version       │  应用 force/strictly/prefer
  │ Conflict      │
  └──────────────┘
        │
        ▼
  ┌──────────────┐
  │ 变体选择       │  选择合适的 variant (runtime/compile/API)
  │ Variant       │  处理 platform, enforcedPlatform
  │ Selection     │
  └──────────────┘
        │
        ▼
  ┌──────────────┐
  │ 工件解析       │  下载 JAR/POM/AAR
  │ Artifact      │  应用工件转换 (downgrade, filter)
  │ Resolution    │
  └──────────────┘
        │
        ▼
  ┌──────────────┐
  │ 缓存结果       │  存储到依赖缓存
  │ Cache         │  ~/.gradle/caches/modules-2/
  └──────────────┘
```

## 4.2 版本冲突解决算法

```java
// Gradle 的版本冲突解决策略 (简化版)
public class VersionConflictResolver {
    
    public String resolve(Map<String, Set<ModuleVersion>> requests) {
        // 1. 收集所有请求的版本
        List<Version> candidates = requests.values().stream()
            .flatMap(Set::stream)
            .map(ModuleVersion::getVersion)
            .collect(toList());
        
        // 2. 检查是否有 force constraint
        Optional<Version> forced = candidates.stream()
            .filter(v -> v.getConstraint() == "strictly" || v.isForced())
            .findFirst();
        if (forced.isPresent()) return forced.get();
        
        // 3. 检查是否有 prefer constraint
        Optional<Version> preferred = candidates.stream()
            .filter(v -> v.getConstraint() == "prefer")
            .findFirst();
        
        // 4. 选择最高版本（默认策略）
        Version highest = candidates.stream()
            .max(Comparator.naturalOrder())
            .orElseThrow();
        
        // 5. 如果有 preferred 且在兼容范围内，使用 preferred
        if (preferred.isPresent() && preferred.get().compareTo(highest) >= 0) {
            return preferred.get();
        }
        
        return highest;
    }
}
```

## 4.3 变体选择 (Variant Selection)

```groovy
// Gradle 使用 "属性匹配" 来选择正确的 variant
// 属性包括: org.gradle.usage, org.gradle.libraryelements, 
//           org.gradle.jvm.version, org.gradle.dependency.bundling

// 当消费者请求 compileClasspath 时：
// 消费者属性: usage=java-api, libraryelements=classes
// 生产者属性: usage=java-api, libraryelements=jar
// → 属性兼容 → 选择该变体

// 当消费者请求 runtimeClasspath 时：
// 消费者属性: usage=java-runtime, libraryelements=jar
// 生产者属性: usage=java-runtime, libraryelements=jar
// → 完全匹配 → 选择该变体
```

```groovy
// 自定义变体匹配
configurations.all {
    attributes {
        attribute(Category.CATEGORY_ATTRIBUTE, objects.named(Category, Category.LIBRARY))
        attribute(Usage.USAGE_ATTRIBUTE, objects.named(Usage, Usage.JAVA_RUNTIME))
        attribute(Bundling.BUNDLING_ATTRIBUTE, objects.named(Bundling, Bundling.EXTERNAL))
        attribute(TargetJvmVersion.TARGET_JVM_VERSION_ATTRIBUTE, 17)
    }
}
```

# 5. ClassLoader 层次与隔离

## 5.1 Gradle ClassLoader 架构

```
Gradle ClassLoader 层次结构（从下到上）：

┌──────────────────────────────────────┐
│  Bootstrap ClassLoader (JVM 核心类)   │  ← Java 标准库
├──────────────────────────────────────┤
│  Gradle API ClassLoader              │  ← Gradle 公开 API
│  (org.gradle.api.*)                  │     所有插件和构建脚本依赖它
├──────────────────────────────────────┤
│  Gradle Core ClassLoader             │  ← Gradle 内部实现
│  (org.gradle.internal.*)             │     构建脚本不能直接访问
├──────────────────────────────────────┤
│  Plugin ClassLoader                  │  ← 每个插件独立的 ClassLoader
│  (org.gradle.api.plugins.*)          │     实现插件隔离
├──────────────────────────────────────┤
│  Build Script ClassLoader            │  ← 构建脚本的类
│  (build.gradle 编译后的类)            │     可以访问 Gradle API + Plugin API
└──────────────────────────────────────┘
```

## 5.2 ClassLoader 隔离的目的

```
为什么要 ClassLoader 隔离？

1. 防止版本冲突：
   - 插件 A 使用 Guava 30.0
   - 插件 B 使用 Guava 32.0
   - 各自有独立的 ClassLoader → 不会冲突

2. 安全性：
   - 构建脚本不能反射访问 Gradle 内部 API
   - 插件不能访问其他插件的私有类

3. 动态加载/卸载：
   - 新的构建可以加载新版插件
   - 旧版插件可以被 GC 回收
```

```java
// 内部实现 (简化)
public class DefaultClassLoaderFactory {
    
    public ClassLoader createPluginClassLoader(PluginDescriptor plugin) {
        // 插件的父 ClassLoader 是 Gradle API
        ClassLoader parent = getGradleApiClassLoader();
        
        // 插件自己的依赖
        List<File> pluginJars = resolvePluginDependencies(plugin);
        
        return new URLClassLoader(
            pluginJars.stream().map(File::toURI).map(URI::toURL).toArray(URL[]::new),
            parent
        );
    }
    
    public ClassLoader createBuildScriptClassLoader(File scriptClassDir) {
        // 构建脚本的父 ClassLoader 包含了 Gradle API 和插件 API
        ClassLoader parent = new CompositeClassLoader(
            getGradleApiClassLoader(),
            getAllPluginClassLoaders()
        );
        
        return new URLClassLoader(
            new URL[] { scriptClassDir.toURI().toURL() },
            new FilteringClassLoader(parent) {
                @Override
                protected boolean canLoad(String className) {
                    // 阻止访问 internal 包
                    return !className.startsWith("org.gradle.internal");
                }
            }
        );
    }
}
```

# 6. Gradle Daemon 架构

## 6.1 守护进程生命周期

```
Gradle 命令执行
      │
      ▼
┌─────────┐    查找空闲 Daemon    ┌──────────────┐
│ gradle  │ ──────────────────▶ │  Daemon      │
│ client  │ ◀────────────────── │  Registry    │
│ (JVM)   │     返回地址/端口     │  (文件锁)     │
└─────────┘                     └──────────────┘
      │                                  │
      │  连接并发送 build 请求              │
      ▼                                  ▼
┌─────────┐     协议: socket/named-pipe ┌──────────────┐
│ gradle  │ ◀────────────────────────▶ │  Daemon      │
│ client  │     (Kryo 序列化)            │  (JVM)       │
└─────────┘                             └──────────────┘
                                               │
                                         执行构建逻辑
                                               │
                                         返回结果给 client
                                               │
                                       ┌───────┴───────┐
                                       │  空闲等待     │
                                       │  (默认3小时)   │
                                       └───────────────┘
                                               │
                                         超时未使用 → 终止
```

## 6.2 守护进程通信协议

```java
// Daemon 通信细节
public class DaemonConnector {
    
    // 1. Daemon 注册
    public void register(DaemonInfo daemon) {
        // 写入 ~/.gradle/daemon/<version>/registry.bin
        // 包含: PID, 端口, 地址, 状态, 最后活跃时间
    }
    
    // 2. Client 连接
    public DaemonConnection connect() {
        List<DaemonInfo> daemons = loadRegistry();
        
        for (DaemonInfo daemon : daemons) {
            if (daemon.isIdle() && daemon.isCompatible()) {
                try {
                    return connectTo(daemon.getAddress(), daemon.getPort());
                } catch (ConnectException e) {
                    // 守护进程可能已死亡，移除注册
                    removeFromRegistry(daemon);
                }
            }
        }
        
        // 没有可用 Daemon → 启动新的
        return startAndConnect();
    }
    
    // 3. 通信序列化
    // Gradle 使用 Kryo (高性能序列化库) 进行消息交换
    // 消息类型: Build, Cancel, Stop, Result 等
}
```

## 6.3 锁文件机制

```
~/.gradle/daemon/<gradle-version>/daemon-<pid>.out.log   # 日志文件
~/.gradle/daemon/<gradle-version>/daemon-<pid>.port.lock  # 端口锁文件
~/.gradle/daemon/<gradle-version>/registry.bin            # 注册表

锁文件内容:
  - Daemon 进程 PID
  - 监听端口号
  - 绑定地址 (localhost)
  - UUID (唯一标识)
  - 启动时间戳
  
锁文件清理:
  - Daemon 正常停止 → 删除锁文件
  - Daemon 异常终止 → 下一次 client 尝试连接时清理
```

# 7. 脚本编译机制

## 7.1 Groovy 脚本编译

```java
// Gradle 使用 Groovy CompilationUnit 编译构建脚本
public class DefaultGroovyScriptCompiler {
    
    public Class<?> compile(ScriptSource source) {
        // 1. 创建 CompilationUnit
        CompilationUnit unit = new CompilationUnit(
            classLoader,
            CompilerConfiguration.DEFAULT
        );
        
        // 2. 添加 Gradle DSL 的 AST 转换
        unit.addPhaseOperation(new GradleDslTransformer(), Phases.CANONICALIZATION);
        
        // 3. 编译脚本
        GroovyClassLoader gcl = new GroovyClassLoader(parentClassLoader);
        return gcl.parseClass(source.getText(), source.getFileName());
    }
    
    // 4. 编译缓存
    // 脚本编译结果缓存到 ~/.gradle/caches/<version>/scripts/
    // 只有当脚本内容改变时才重新编译
}
```

## 7.2 Kotlin 脚本编译

```kotlin
// Kotlin DSL 的编译流程更复杂
object KotlinScriptCompiler {
    
    fun compile(script: File, gradleVersion: String): CompiledScript {
        // 1. 创建 Kotlin 编译环境
        val configuration = CompilerConfiguration().apply {
            put(CLASSPATH, getGradleClasspath())
            put(SCRIPT_TEMPLATES, listOf(
                "org.gradle.kotlin.dsl.KotlinBuildScript",
                "org.gradle.kotlin.dsl.KotlinSettingsScript",
                "org.gradle.kotlin.dsl.KotlinInitScript"
            ))
        }
        
        // 2. 生成类型安全访问器
        val accessors = TypeSafeAccessorsGenerator.generate(projectStructure)
        
        // 3. 编译脚本
        val compiler = KotlinJsr223ScriptEngineFactory().scriptEngine
        return compiler.compile(script, accessors, configuration)
    }
}

// Kotlin DSL 编译缓存
// ~/.gradle/caches/<version>/kotlin-dsl/
//   ├── accessors/       ← 生成的类型安全访问器
//   ├── scripts/          ← 编译后的脚本类
//   └── plugins/          ← 预编译脚本插件
```

## 7.3 编译回避与缓存

```java
// 脚本编译缓存 key 的计算
public class ScriptCacheKey {
    
    public static HashCode calculate(ScriptSource source) {
        Hasher hasher = Hashing.sha256().newHasher();
        
        // 1. 脚本文件内容哈希
        hasher.putBytes(source.getContent().getBytes(UTF_8));
        
        // 2. 脚本的 classpath (包括所有插件)
        for (File jar : scriptClasspath) {
            hasher.putString(jar.getAbsolutePath(), UTF_8);
            hasher.putLong(jar.lastModified());
        }
        
        // 3. Gradle 版本
        hasher.putString(GradleVersion.current().getVersion(), UTF_8);
        
        // 4. 脚本类型 (build / settings / init)
        hasher.putString(source.getScriptType().name(), UTF_8);
        
        return hasher.hash();
    }
}
```

# 8. 配置缓存内部

## 8.1 序列化模型

```java
// 配置缓存的原理：
// 1. 在第一次构建的配置阶段末尾，将配置结果序列化
// 2. 后续构建直接加载序列化状态，跳过配置阶段

public class ConfigurationCache {
    
    public void store(BuildState buildState) {
        // 序列化对象：
        // - 所有 task 的配置
        // - 所有 extension 的值
        // - 所有 Property 的 final value
        // - 任务依赖图
        // - 输入/输出声明
        
        try (ObjectOutputStream oos = new ObjectOutputStream(
                new FileOutputStream(cacheFile))) {
            
            // 写入任务图
            oos.writeObject(buildState.getTaskGraph());
            
            // 写入项目配置
            for (Project project : buildState.getAllProjects()) {
                oos.writeObject(captureProjectState(project));
            }
            
            // 写入插件状态
            for (Plugin plugin : buildState.getPlugins()) {
                if (plugin instanceof StatefulPlugin) {
                    oos.writeObject(((StatefulPlugin) plugin).captureState());
                }
            }
        }
    }
    
    public BuildState load() {
        try (ObjectInputStream ois = new ObjectInputStream(
                new FileInputStream(cacheFile))) {
            
            BuildState state = new BuildState();
            state.setTaskGraph((TaskGraph) ois.readObject());
            state.setProjectStates((Map<String, ProjectState>) ois.readObject());
            // ...
            return state;
        }
    }
}
```

## 8.2 配置缓存的状态捕获

```groovy
// 配置缓存如何捕获状态
// 示例：捕获一个任务配置

// 声明式状态捕获
abstract class MyTask extends DefaultTask {
    @Input
    abstract Property<String> getGreeting()
    
    @OutputFile
    abstract RegularFileProperty getOutputFile()
    
    @TaskAction
    void greet() {
        outputFile.get().asFile.text = greeting.get()
    }
}

// 当配置缓存启用时：
// 配置阶段：
task myTask(type: MyTask) {
    greeting = "Hello"     // ← 这个值被序列化
    outputFile = file("$buildDir/greeting.txt")  // ← 这个路径被序列化
}

// 后续构建：
// 1. 从缓存加载 myTask 的配置
// 2. greeting = "Hello" (已缓存)
// 3. outputFile = .../build/greeting.txt (已缓存)
// 4. 检查输入是否有变化 (只有 greeting 是输入)
// 5. 如果 greeting 改变 → 重新执行
// 6. 如果不变 → 从任务缓存恢复
```

## 8.3 可缓存性限制

```groovy
// 以下情况会导致配置缓存不可用
// (需要满足 ConfigurationCacheFriendly 标记)

// ❌ 不可缓存：使用外部可变状态
task badTask {
    doLast {
        def env = System.getenv("BUILD_NUM")  // 外部状态
        println env
    }
}

// ✅ 可缓存：使用 Provider 封装
task goodTask {
    def buildNum = providers.environmentVariable("BUILD_NUM")
        .orElse("local")
    doLast {
        println buildNum.get()
    }
}

// ❌ 不可缓存：在配置阶段读取文件
def version = file("version.txt").text.trim()

// ✅ 可缓存：使用 Provider.fileContents
def version = providers.fileContents(
    layout.projectDirectory.file("version.txt")
).asText.map { it.trim() }

// ❌ 不可缓存：调用外部进程在配置阶段
def gitHash = "git rev-parse HEAD".execute().text.trim()

// ✅ 可缓存：使用 Provider
def gitHash = providers.exec {
    commandLine("git", "rev-parse", "HEAD")
}.standardOutput.asText.map { it.trim() }
```

# 9. 文件操作引擎

## 9.1 文件系统快照

```java
// Gradle 使用 Virtual File System (VFS) 高效跟踪文件变更
public class VirtualFileSystem {
    
    // VFS 存储结构 (LevelDB)
    // 每个文件记录:
    //   - 路径 (绝对路径)
    //   - 内容哈希 (SHA-256)
    //   - 长度
    //   - 最后修改时间
    //   - 文件类型 (FILE / DIRECTORY / MISSING)
    
    public Snapshot snapshot(File file) {
        // 1. 从 VFS 查找缓存
        VfsEntry cached = levelDB.get(file.getAbsolutePath());
        
        if (cached != null && cached.isCurrent(file)) {
            return new Snapshot(cached.getHash(), file);
        }
        
        // 2. 计算新的哈希并更新
        HashCode hash = hashFile(file);
        levelDB.put(file.getAbsolutePath(), new VfsEntry(
            hash, file.length(), file.lastModified()
        ));
        
        return new Snapshot(hash, file);
    }
}

// 文件系统监听 (Watch Service)
public class FileSystemWatching {
    
    public void watch(File projectDir) {
        WatchService watcher = FileSystems.getDefault().newWatchService();
        projectDir.toPath().register(watcher,
            StandardWatchEventKinds.ENTRY_CREATE,
            StandardWatchEventKinds.ENTRY_MODIFY,
            StandardWatchEventKinds.ENTRY_DELETE
        );
        
        // 循环处理事件
        while (true) {
            WatchKey key = watcher.poll(100, TimeUnit.MILLISECONDS);
            if (key != null) {
                for (WatchEvent<?> event : key.pollEvents()) {
                    VFS.invalidate(key, event.context());
                }
                key.reset();
            }
        }
    }
}
```

## 9.2 增量文件快照

```java
// 对于大型文件集合的增量快照
public class IncrementalFileSnapshotter {
    
    // 仅重新哈希发生变化的文件
    public Map<File, Snapshot> snapshot(FileCollection files) {
        Map<File, Snapshot> result = new HashMap<>();
        
        for (File file : files) {
            // 检查 VFS 记录
            VfsEntry entry = vfs.get(file);
            
            if (entry != null && !hasChanged(file, entry)) {
                // 文件未变，使用缓存的哈希
                result.put(file, new Snapshot(entry.getHash(), file));
            } else {
                // 文件变更，重新计算哈希
                result.put(file, new Snapshot(hashFile(file), file));
            }
        }
        
        return result;
    }
    
    private boolean hasChanged(File file, VfsEntry entry) {
        return file.length() != entry.getLength() ||
               file.lastModified() != entry.getLastModified();
    }
}
```

# 10. ServiceRegistry 模式

## 10.1 服务定位器

```java
// Gradle 内部使用自己的依赖注入容器: ServiceRegistry
// (类似 Spring IoC 但更轻量级，专为构建工具设计)

@ServiceScope(Scope.Build.class)
public interface BuildServiceRegistry extends ServiceRegistry {
    
    ProjectFactory getProjectFactory();
    TaskFactory getTaskFactory();
    PluginManager getPluginManager();
    DependencyResolutionServices getDependencyResolutionServices();
    // ...
}

// 服务注册和作用域
public enum Scope {
    Global,     // 整个 Gradle 进程生命周期
    Build,      // 单次构建生命周期
    Project,    // 单个项目生命周期
    GradleUserHome  // Gradle 用户目录范围
}

// 典型的注册方式
public class GradleInternalServices {
    public void configure(ServiceRegistryBuilder builder) {
        builder.add(ProjectFactory.class, DefaultProjectFactory.class);
        builder.add(TaskFactory.class, DefaultTaskFactory.class);
        
        // 带装饰器的服务
        builder.addDecorator(LoggingTaskFactory.class, TaskFactory.class);
    }
}
```

## 10.2 插件如何获取服务

```java
// 插件获取内部服务的两种方式

// 方式1：通过 project.services 或 project.objects
public class MyPlugin implements Plugin<Project> {
    @Override
    public void apply(Project project) {
        // 获取公开的服务
        ObjectFactory objects = project.getObjects();
        ProviderFactory providers = project.getProviders();
        ExecOperations execOps = project.getExecOperations();
        FileSystemOperations fsOps = project.getFileSystemOperations();
        ArchiveOperations archiveOps = project.getArchiveOperations();
    }
}

// 方式2：通过 @Inject 依赖注入（在抽象类中）
abstract class MyTask extends DefaultTask {
    
    @Inject
    protected abstract ObjectFactory getObjectFactory();
    
    @Inject
    protected abstract ProviderFactory getProviderFactory();
    
    @Inject
    protected abstract ExecOperations getExecOperations();
    
    @Inject
    protected abstract FileSystemOperations getFileSystemOperations();
    
    @Inject
    protected abstract ArchiveOperations getArchiveOperations();
    
    @TaskAction
    void execute() {
        // 使用注入的服务
        getExecOperations().exec(spec -> {
            spec.commandLine("git", "status");
        });
    }
}
```

# 11. 日志架构

## 11.1 SLF4J 集成

```java
// Gradle 使用 SLF4J 作为日志门面，自定义日志路由

// 日志级别路由
// ┌───────────────┐
// │  SLF4J Logger  │  ← 插件和自定义代码使用
// └───────┬───────┘
//         ▼
// ┌───────────────┐
// │ LoggingManager │ ← Gradle 核心日志管理
// └───────┬───────┘
//         ▼
// ┌───────────────┐
// │ LogLevel       │ ← 路由到不同输出
// │ Router         │
// ├── ERROR → stderr
// ├── WARN  → stderr
// ├── LIFECYCLE → stdout
// ├── INFO  → stdout (使用 --info)
// └── DEBUG → stdout (使用 --debug)
//         ▼
// ┌───────────────┐
// │ LoggingBacked │ ← 最终输出实现
// │ OutputEvent   │
// └───────────────┘
```

## 11.2 自定义日志使用

```groovy
// 在任务中使用 SLF4J
import org.slf4j.LoggerFactory

task logExample {
    doLast {
        def logger = LoggerFactory.getLogger("MyTask")
        logger.info("使用 SLF4J 记录信息")
        logger.debug("调试信息")
    }
}

// 使用 Project 的 logger
task projectLog {
    doLast {
        logger.lifecycle("生命周期信息 - 始终显示")
        logger.info("信息 - 需要 --info")
        logger.debug("调试 - 需要 --debug")
        logger.warn("警告 - 始终显示")
        logger.error("错误 - 始终显示")
        logger.quiet("安静 - 需要 --quiet")
    }
}
```

## 11.3 LoggingManager

```groovy
// 控制日志级别
task customLogLevel {
    logging.level = LogLevel.INFO  // 覆盖全局日志级别
    
    // 或
    logging.captureStandardOutput(LogLevel.INFO)
    logging.captureStandardError(LogLevel.ERROR)
    
    doLast {
        println "输出到标准输出的文本"
        // captureStandardOutput 控制如何处理 println 的输出
    }
}

// 在 build.gradle 中配置全局日志
allprojects {
    logging.level = LogLevel.INFO
}
```

# 12. 插件解析机制

## 12.1 PluginResolutionService

```java
// 插件标记工件 (Plugin Marker Artifacts)
// 当声明 plugins { id "my-plugin" version "1.0" }
// Gradle 查找工件: my-plugin:my-plugin.gradle.plugin:1.0
// 该工件是一个只有 POM 的空 JAR，POM 中声明实际插件实现

// 插件标记工件 POM 示例:
/*
<project>
    <groupId>my-plugin</groupId>
    <artifactId>my-plugin.gradle.plugin</artifactId>
    <version>1.0</version>
    <dependencies>
        <dependency>
            <groupId>my-plugin</groupId>
            <artifactId>my-plugin-gradle</artifactId>
            <version>1.0</version>
        </dependency>
    </dependencies>
</project>
*/

public class PluginResolutionService {
    
    public PluginImplementation resolve(String pluginId, String version) {
        // 1. 构造标记工件坐标
        String markerGroup = pluginId;  // 或插件的 maven group
        String markerArtifact = pluginId + ".gradle.plugin";
        String markerVersion = version;
        
        // 2. 从仓库解析标记工件
        File markerPom = resolvePom(markerGroup, markerArtifact, markerVersion);
        
        // 3. 读取 POM 中的依赖
        List<Dependency> pluginDeps = parseDependencies(markerPom);
        
        // 4. 下载实际插件及其依赖
        List<File> pluginClasspath = resolveDependencies(pluginDeps);
        
        // 5. 创建插件 ClassLoader
        ClassLoader pluginLoader = createPluginClassLoader(pluginClasspath);
        
        // 6. 加载并实例化插件
        Class<?> pluginClass = pluginLoader.loadClass(
            findPluginImplementationClass(markerPom)
        );
        
        return new PluginImplementation(pluginClass, pluginLoader);
    }
}
```

## 12.2 插件的 settings pluginManagement

```groovy
// settings.gradle - 控制插件解析
pluginManagement {
    repositories {
        gradlePluginPortal()       // 默认: https://plugins.gradle.org/m2
        mavenCentral()
        maven { url 'https://repo.example.com/plugins' }
    }
    
    // 版本匹配规则
    resolutionStrategy {
        eachPlugin {
            if (requested.id.namespace == 'com.example') {
                useModule("com.example:gradle-plugins:1.0.0")
                // 从自定义 Maven 坐标加载
            }
        }
    }
    
    plugins {
        // 所有子项目的默认插件版本
        id 'com.example' version '1.0.0'
    }
}
```

# 13. Native 集成内部

## 13.1 NativePlatform 模型

```java
// Gradle 的 native 支持架构
// ┌────────────────────────────────┐
// │  Native Binary Plugin          │
// │  (C/C++/Swift/Objective-C)     │
// ├────────────────────────────────┤
// │  NativePlatform                │ ← 操作系统 + 架构
// │  NativeToolChain               │ ← GCC/Clang/MSVC
// │  NativeBinary                  │ ← 可执行文件/共享库
// ├────────────────────────────────┤
// │  SWIG (Simplified Wrapper)     │ ← JNI 生成
// │  (用于 C/C++ ↔ Java 互操作)     │
// └────────────────────────────────┘

// NativePlatform 描述目标平台
public class NativePlatform {
    Architecture architecture;  // x86, x86_64, arm64
    OperatingSystem operatingSystem;  // linux, windows, macOS
}

// NativeToolChain 描述编译器工具链
public interface NativeToolChain {
    String getName();       // "gcc", "clang", "visualCpp"
    List<File> getPath();   // 工具链的安装路径
    PlatformToolProvider getProvider();
}
```
