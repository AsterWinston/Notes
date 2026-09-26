
# 1. 什么是 Build Scan

## 1.1 概述

Build Scan（构建扫描）是 Gradle 提供的构建诊断工具，每次构建都可生成一份详细的执行报告。它将构建过程中的所有数据（任务执行、依赖解析、测试结果、性能指标等）以可视化 Web 页面的形式展示。

```
构建扫描 = 构建的 X 光片
- 时间线：每个任务的精确执行时间和重叠
- 依赖图：所有依赖的解析路径和原因
- 性能：配置时间 vs 执行时间，瓶颈识别
- 环境：JDK 版本、系统信息、Gradle 配置
- 测试：每个测试用例的执行时间和结果
```

## 1.2 为什么使用 Build Scan

| 场景 | Build Scan 提供的价值 |
|------|---------------------|
| 构建变慢 | 定位最慢的任务、配置时间过长 |
| 依赖冲突 | 分析为什么某个版本被选中 |
| 测试不稳定 | 查看测试执行时间线和失败原因 |
| CI 构建差异 | 对比本地和 CI 的构建性能 |
| 团队协作 | 分享构建扫描链接，共同分析问题 |
| 插件开发调试 | 查看任务的输入/输出、UP-TO-DATE 判断依据 |

# 2. 生成 Build Scan

## 2.1 命令行方式

```bash
# 发布到 scans.gradle.com (需接受服务条款)
gradle build --scan

# 首次使用会提示：
# Publishing a build scan to scans.gradle.com requires accepting the
# Gradle Terms of Service defined at https://scans.gradle.com/terms-of-service.
# Do you accept these terms? [yes, no]
# yes

# 仅生成但不自动打开浏览器
gradle build --scan --no-scan-auto-open

# 禁用构建扫描
gradle build --no-scan

# 始终在构建失败时生成扫描 (build.gradle 中配置)
```

## 2.2 在构建脚本中配置

```groovy
// build.gradle - Groovy DSL
buildScan {
    // 始终生成构建扫描 (即使没有 --scan 参数)
    publishAlways()
    
    // 仅在构建失败时生成
    publishOnFailure()
    
    // 仅在 CI 环境中生成
    if (System.getenv("CI") != null) {
        publishAlways()
    }
}
```

```kotlin
// build.gradle.kts - Kotlin DSL
buildScan {
    publishAlways()
    
    // 条件生成
    if (System.getenv()["CI"] == "true") {
        publishAlways()
    }
}
```

## 2.3 发布选项对比

| 选项 | 行为 | 使用场景 |
|------|------|---------|
| `publishAlways()` | 每次构建都发布扫描 | CI 环境、调试复杂问题 |
| `publishOnFailure()` | 仅失败时发布 | 减少不必要的扫描 |
| `--scan` 命令行 | 一次性发布当前构建 | 开发时临时调试 |
| `--no-scan` | 禁止发布 | 关闭配置中 forced scan |

```groovy
// 高级条件发布策略
buildScan {
    // 基于构建结果
    if (gradle.startParameter.taskNames.contains("publish")) {
        publishAlways()
    }
    
    // 基于属性
    if (project.hasProperty("scan")) {
        publishAlways()
    }
    
    // 限定发布地址
    termsOfServiceUrl = "https://gradle.com/terms-of-service"
    termsOfServiceAgree = "yes"
}
```

# 3. Build Scan 内容详解

## 3.1 Timeline（时间线）

```
构建扫描的时间线视图展示了任务的并行执行情况：

时间轴 (ms)  0     1000    2000    3000    4000    5000
             │       │       │       │       │       │
Worker 1    [=====:compileJava=====]      [=:compileTestJava=]
Worker 2      [===:processResources===]     [=:processTestResources=]
Worker 3         [====:checkstyleMain====]
Worker 4            [=====:pmdMain======]
             │       │       │       │       │       │

关键指标：
- 配置时间 (Configuration time): 评估项目和构建任务图的时间
- 执行时间 (Execution time): 实际运行任务的时间  
- 关键路径 (Critical path): 不能并行的最长任务链
```

```groovy
// 解释时间线指标
// 配置时间 (Configuration):
//   - 包括: settings 评估 + project 评估 + task graph 计算
//   - 优化: 使用 configuration-cache, 减少 afterEvaluate
//
// 执行时间 (Execution):
//   - 包括: 所有任务的实际运行时间
//   - 优化: 并行构建, 增量构建, 构建缓存
//
// 空闲时间 (Idle):
//   - 配置和执行之间的等待时间
//   - 原因: Worker 数量不足, 网络/IO 等待
```

## 3.2 Performance（性能面板）

```
性能面板显示的关键指标：

┌─────────────────────────────────────────┐
│  构建总耗时: 12.5s                       │
│  ├── 初始化 (init): 0.2s (1.6%)         │
│  ├── 配置 (configure): 3.1s (24.8%)     │
│  │   ├── 项目 :app: 2.1s               │
│  │   ├── 项目 :lib: 0.8s               │
│  │   └── 任务图计算: 0.2s              │
│  └── 执行 (execute): 9.2s (73.6%)      │
│      ├── :compileJava: 4.5s            │
│      ├── :test: 3.2s                   │
│      ├── :jar: 0.8s                    │
│      └── 其他: 0.7s                    │
└─────────────────────────────────────────┘

优化建议：
- 配置 > 30%: 考虑 Configuration Cache
- 任务序列化: 增加并行度 (maxParallelForks)
- 重复任务: 检查是否有不必要的串行依赖
```

## 3.3 Task Execution（任务执行）

每个任务在构建扫描中展示：

```
Task: compileJava
├── 结果: FROM-CACHE (或 SUCCESS, UP-TO-DATE, FAILED)
├── 耗时: 4.5s
├── 原因: "因为输入属性 'sourceFiles' 已更改"
├── 输入:
│   ├── sourceFiles: 142 files (1 changed)
│   ├── classpath: 87 files (unchanged)
│   ├── options.encoding: "UTF-8"
│   └── ...
├── 输出:
│   ├── classesDir: 218 .class files
│   └── ...
└── 依赖:
    ├── :generateSources (UP-TO-DATE)
    ├── :compileAnnotationProcessors (SUCCESS)
    └── ...

特别关注：
- 输入哈希变更原因 (某个文件被修改)
- 哪些依赖是 FROM-CACHE vs EXECUTED
- 为什么 UP-TO-DATE 判定失败
```

## 3.4 Dependency Resolution（依赖解析）

```
Configuration: compileClasspath
├── 解析耗时: 235ms
├── 仓库查询:
│   ├── Maven Central: 12 queries
│   ├── Google: 3 queries
│   └── 总计: 18 queries (15 from cache)
├── 下载:
│   ├── 总大小: 45MB
│   └── 下载时间: 1.2s
├── 版本冲突:
│   ├── com.google.guava:guava → 33.0.0-jre
│   └── 选择原因: "请求了 33.0.0-jre"
└── 依赖树:
    ├── project :app
    │   ├── com.example:lib:1.0
    │   │   └── com.google.guava:guava:33.0.0-jre
    │   └── org.apache.commons:commons-lang3:3.13.0
    └── ...

解析面板信息：
- 每个模块在哪些仓库被搜索过
- 哪些搜索被缓存
- 动态版本的解析结果
- 每个依赖的 "为什么被包含" 说明
```

## 3.5 Test Execution（测试执行）

```
Tests: 248 passed, 0 failed, 3 skipped

测试时间线:
test                                                        [=====]
  ├── com.example.FastTest (15 tests, 1.2s)
  ├── com.example.SlowTest (3 tests, 8.5s)         [==========]
  │   ├── testLongRunningOperation (4.2s)          [=====]
  │   ├── testDatabaseIntegration (3.1s)           [=== ]
  │   └── testComplexCalculation (1.2s)            [=   ]
  └── com.example.Parameterized (30 tests, 0.8s)

测试详情:
- 每个测试的耗时、状态 (passed/failed/skipped)
- 失败测试的堆栈跟踪
- 测试输出 (stdout/stderr)
- 参数化测试的每个参数结果
- 测试的进程分配 (fork)
```

## 3.6 Switches（构建参数）

```
构建命令分析：
原始命令: gradle build --parallel --build-cache --scan

Gradle 属性:
  org.gradle.parallel = true
  org.gradle.caching = true  
  org.gradle.jvmargs = -Xmx2048m
  org.gradle.daemon = true

有效参数组合:
  Switches:
    √ 构建缓存      (--build-cache)
    √ 并行构建      (--parallel)
    √ 配置缓存      (来自 gradle.properties)
    × 增量编译      (未启用)
    √ 守护进程      (默认)
    × 离线模式      (未启用)
    
任务排除:  (-x test)
请求任务: build
```

## 3.7 Environment（环境信息）

```
环境面板:
  Gradle: 8.5
  Java: 17.0.9 (Adoptium)
  OS: Windows 11 (amd64)
  CPU cores: 16
  Max heap: 2048MB
  
  系统属性:
    user.dir = D:\projects\my-app
    user.country = CN
    file.encoding = UTF-8
    
  环境变量:
    JAVA_HOME = C:\Program Files\Eclipse Adoptium\jdk-17.0.9.9-hotspot\
    GRADLE_HOME = (未设置)
    CI = true
    
  Gradle 缓存:
    User home: C:\Users\user\.gradle
    Build cache: C:\Users\user\.gradle\caches\build-cache-1
```

# 4. Build Scan 高级配置

## 4.1 自定义标签和链接

```groovy
buildScan {
    // 标签：方便在 Gradle Enterprise 中搜索和分类
    tag "JDK17"
    tag System.getProperty('os.name').toLowerCase().contains('linux') ? 'linux' : 'other'
    tag project.version.toString().contains("SNAPSHOT") ? "snapshot" : "release"
    
    // 自定义链接：关联到外部系统
    link "CI Build", System.getenv("BUILD_URL") ?: "local"
    link "Git Commit", "https://github.com/mycompany/myapp/commit/${gitHash()}"
    link "Issue Tracker", "https://jira.example.com/browse/PROJ-123"
    link "Deployment", "https://deploy.example.com/app/${project.version}"
    
    // 自定义值：添加任意键值对
    value "Git Branch", gitBranch()
    value "Build Number", System.getenv("BUILD_NUMBER") ?: "N/A"
    value "Docker Image", "${project.name}:${project.version}"
    value "Test Profile", project.hasProperty('testProfile') ? project.testProfile : 'default'
}

// 辅助方法
def gitHash() {
    def stdout = new ByteArrayOutputStream()
    exec {
        commandLine 'git', 'rev-parse', '--short', 'HEAD'
        standardOutput = stdout
        ignoreExitValue = true
    }
    return stdout.toString().trim()
}

def gitBranch() {
    def stdout = new ByteArrayOutputStream()
    exec {
        commandLine 'git', 'rev-parse', '--abbrev-ref', 'HEAD'
        standardOutput = stdout
        ignoreExitValue = true
    }
    return stdout.toString().trim()
}
```

## 4.2 背景信息设置

```groovy
buildScan {
    // 背景信息用作断言的构建扫描描述
    background {
        // 自定义背景信息 (注解形式)
        annotation("CI 流水线", System.getenv("CI_PIPELINE_NAME") ?: "本地")
        annotation("触发者", System.getenv("CI_TRIGGER_USER") ?: System.getProperty('user.name'))
        annotation("目标环境", project.hasProperty('env') ? project.env : 'development')
    }
    
    // 或者在构建结束时设置
    buildFinished {
        background {
            annotation("构建结果", it.failure ? "失败" : "成功")
            annotation("构建耗时", "${it.duration}ms")
        }
    }
}
```

## 4.3 条件链接

```groovy
// 仅在特定条件下添加链接
buildScan {
    def ciUrl = System.getenv("CI_JOB_URL")
    if (ciUrl != null) {
        link "CI Job", ciUrl
    }
    
    def prUrl = System.getenv("CI_MERGE_REQUEST_URL")
    if (prUrl != null) {
        link "Merge Request", prUrl
    }
    
    // 根据 Gradle Task 添加不同链接
    gradle.taskGraph.whenReady { graph ->
        if (graph.hasTask(":publish")) {
            link "Release Notes", 
                "https://github.com/mycompany/myapp/releases/tag/v${project.version}"
        }
        
        if (graph.hasTask(":jacocoTestReport")) {
            link "Code Coverage", 
                "${project.buildDir}/reports/jacoco/test/html/index.html"
        }
    }
}
```

# 5. Build Scan 在 CI 中的最佳实践

## 5.1 CI 环境配置

```groovy
// build.gradle - CI 友好的 Build Scan 配置
buildScan {
    if (System.getenv("CI") == "true") {
        // CI 环境：始终发布
        publishAlways()
        
        // 同意服务条款（非交互式）
        termsOfServiceUrl = "https://gradle.com/terms-of-service"
        termsOfServiceAgree = "yes"
        
        // 添加 CI 相关信息
        tag "CI"
        tag System.getenv("CI_PLATFORM") ?: "unknown-ci"
        
        // 链接到 CI 构建页面
        if (System.getenv("BUILD_URL")) {
            link "CI Build", System.getenv("BUILD_URL")
        }
        
        // Git 信息
        if (System.getenv("GIT_COMMIT")) {
            value "Git Commit", System.getenv("GIT_COMMIT")
            link "Git Commit", 
                "https://github.com/mycompany/myapp/commit/${System.getenv("GIT_COMMIT")}"
        }
        
        if (System.getenv("GIT_BRANCH")) {
            tag "branch:${System.getenv("GIT_BRANCH")}"
        }
        
        // 构建编号
        if (System.getenv("BUILD_NUMBER")) {
            value "Build Number", System.getenv("BUILD_NUMBER")
        }
        
        // PR 信息
        if (System.getenv("PR_NUMBER")) {
            tag "PR"
            value "PR Number", System.getenv("PR_NUMBER")
            link "Pull Request", 
                "https://github.com/mycompany/myapp/pull/${System.getenv("PR_NUMBER")}"
        }
    } else {
        // 本地：仅在指定 --scan 时发布
        // 或仅失败时发布
        publishOnFailure()
    }
}
```

## 5.2 收集构建扫描链接

```bash
# 从构建输出中提取扫描链接
gradle build --scan 2>&1 | grep "https://scans.gradle.com/s/"

# 在 CI 脚本中保存链接
gradle build --scan 2>&1 | tee build.log
BUILD_SCAN_URL=$(grep -oP 'https://scans\.gradle\.com/s/\w+' build.log)
echo "BUILD_SCAN_URL=$BUILD_SCAN_URL" >> build.env

# 在 GitHub Actions 中使用
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Build with Gradle
        run: ./gradlew build --scan
      - name: Extract Build Scan URL
        run: |
          BUILD_SCAN=$(grep -oP 'https://scans\.gradle\.com/s/\w+' build.log || true)
          echo "BUILD_SCAN=$BUILD_SCAN" >> $GITHUB_ENV
      - name: Comment PR with Build Scan
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `📊 [Build Scan](${process.env.BUILD_SCAN})`
            })
```

## 5.3 Gradle Enterprise 集成

```groovy
// settings.gradle - 配置 Gradle Enterprise 服务器
plugins {
    id("com.gradle.enterprise") version "3.15.1"
}

gradleEnterprise {
    server = "https://gradle-enterprise.mycompany.com"
    
    buildScan {
        // 发布到企业服务器而非 scans.gradle.com
        publishAlways()
        
        // 数据捕获配置
        capture {
            taskInputFiles = true    // 捕获任务输入文件列表
            fileFingerprints = true  // 捕获文件指纹
        }
        
        // 上传配置
        uploadInBackground = !System.getenv("CI") // 本地后台上传, CI 同步上传
        
        // 观察模式 (不执行实际构建)
        // obfuscation { ... }
    }
}
```

# 6. Build Scan API

## 6.1 buildScan 扩展完整 API

```groovy
// buildScan 扩展的完整能力
buildScan {
    // === 发布控制 ===
    publishAlways()                       // 始终发布
    publishOnFailure()                    // 失败时发布
    publishOnDemand()                     // 仅 --scan 时发布 (默认)
    
    // === 服务条款 ===
    termsOfServiceUrl = "https://..."
    termsOfServiceAgree = "yes"
    
    // === 元数据 ===
    tag("标签")                           // 添加标签 (可多次调用)
    link("名称", "URL")                   // 添加链接 (可多次调用)
    value("键", "值")                     // 添加自定义值 (可多次调用)
    
    // === 后台任务 ===
    background {
        annotation("键", "值")            // 背景注解
    }
    
    // === 生命周期钩子 ===
    buildFinished { result ->
        // result: BuildResult 包含 failure 和 duration
    }
    
    // === 高级 (Gradle Enterprise) ===
    capture {
        taskInputFiles = true
        fileFingerprints = true
        buildLogging = true
        testLogging = true
    }
    
    uploadInBackground = true
}
```

## 6.2 程序化访问扫描数据

```groovy
// 在插件中访问构建扫描的 API
// 需要 com.gradle.scan.plugin 或 enterprise plugin

import com.gradle.scan.plugin.BuildScanExtension

class MyPerformancePlugin implements Plugin<Project> {
    void apply(Project project) {
        def ext = project.extensions.findByType(BuildScanExtension)
        
        if (ext) {
            ext.tag("performance-monitored")
            
            ext.buildFinished { result ->
                if (!result.failure) {
                    ext.value("Build Status", "SUCCESS")
                }
            }
        }
    }
}
```

## 6.3 与 Configuration Cache 的交互

```groovy
// Build Scan 和 Configuration Cache 的注意点
// Configuration Cache 存储时：
//   - buildScan 扩展的配置会被序列化
//   - 发布决策 (publishAlways/publishOnFailure) 会保留
//   - 标签、链接、值不会在存储时执行 (它们是执行阶段概念)

// 构建扫描的配置方式：
buildScan {
    // ✅ 这些是配置阶段决定的：
    publishAlways()
    termsOfServiceAgree = "yes"
    capture {
        taskInputFiles = true
    }
    
    // ⚠️ 这些在 Configuration Cache 恢复后需要重新执行：
    tag("dynamic-${project.name}")
    
    // ✅ 使用 buildFinished 确保正确行为：
    buildFinished {
        tag("outcome-${it.failure ? 'failed' : 'success'}")
    }
}
```

# 7. Build Scan vs Build Profile

## 7.1 功能对比

| 特性 | Build Scan (`--scan`) | Build Profile (`--profile`) |
|------|----------------------|---------------------------|
| 输出格式 | Web 页面 (云端) | HTML (本地) |
| 任务时间线 | 可视化 Gantt 图 | 表格数据 |
| 依赖解析详情 | 完整的解析路径 | 有限的依赖信息 |
| 测试详情 | 每个测试的时间和结果 | 无 |
| 配置时间分析 | 每个项目分别显示 | 汇总配置时间 |
| 环境信息 | 完整的系统和构建配置 | 有限的系统信息 |
| 历史对比 | 支持 (Gradle Enterprise) | 不支持 |
| 离线使用 | 需要网络连接 | 完全离线 |
| 隐私 | 上传到云端 | 保留在本地 |

## 7.2 使用 --profile 的详细分析

```bash
# 生成性能报告
gradle build --profile

# 报告路径: build/reports/profile/profile-2024-01-01-12-00-00.html

# 报告包含：
# ┌─────────────────────────────────────────┐
# │ Summary                                  │
# │  Total Build Time: 12.5s                 │
# │  Startup: 0.8s                          │
# │  Settings and BuildSrc: 0.2s            │
# │  Loading Projects: 0.1s                 │
# │  Configuring Projects: 3.1s             │
# │  Artifact Transforms: 0.0s              │
# │  Task Execution: 9.2s                   │
# └─────────────────────────────────────────┘
#
# ┌─────────────────────────────────────────┐
# │ Task Execution (按耗时排序)               │
# │  :compileJava         4.5s              │
# │  :test                3.2s              │
# │  :compileTestJava     0.8s              │
# │  :jar                 0.8s              │
# │  ...                                    │
# └─────────────────────────────────────────┘
```

# 8. 解释 Build Scan 结果

## 8.1 识别性能瓶颈

```
构建扫描性能诊断分类：

问题类型 1: 配置时间过长 (> 30% 总时间)
  ────────────────────────────────
  特征：
  - Configure project 阶段耗时长
  - 多个项目顺序配置
  
  诊断步骤：
  1. 查看 Timeline → 每个项目的配置时间
  2. 查看 Performance → Configuration 时间占比
  3. 检查是否有 afterEvaluate / allprojects 块
  
  解决方案：
  - 启用 Configuration Cache: --configuration-cache
  - 减少配置阶段的 I/O 操作
  - 使用 configureEach 替代 all
  - 使用 lazy task configuration

问题类型 2: 任务执行序列化 (CPU 利用率低)
  ────────────────────────────────
  特征：
  - Timeline 显示大量空白，Worker 空闲
  - 任务之间存在不必要的串行依赖
  
  解决方案：
  - 启用并行执行: --parallel
  - 增加 maxParallelForks
  - 检查不必要的 dependsOn 声明
  - 使用 mustRunAfter 替代 dependsOn (无严格依赖时)

问题类型 3: 依赖解析缓慢
  ────────────────────────────────
  特征：
  - Dependency Resolution 时间占比较高
  - 大量远程仓库查询
  
  解决方案：
  - 启用依赖缓存
  - 使用 Gradle Enterprise 远程缓存
  - 配置 repository 的内容过滤
  - 使用 Maven Local 或代理仓库

问题类型 4: 测试执行缓慢
  ────────────────────────────────
  特征：
  - Test 任务成为瓶颈
  - 个别测试耗时长
  
  解决方案：
  - 使用 maxParallelForks 并行测试
  - 使用 forkEvery 控制 JVM 复用
  - 分拆慢速测试到独立 source set
  - 使用 JUnit 并行执行
```

## 8.2 依赖分析诊断

```
构建扫描依赖分析：

"为什么这个版本被选中？"
1. 在 Dependency Resolution 面板找到对应模块
2. 查看 Selection Reasons (选择原因)：
   - "Requested": 直接声明了此版本
   - "By constraint": 依赖约束指定
   - "Conflict resolution": 冲突解决算法选择
   - "Forced": force/forcedPlatform 强制
   - "By ancestor": 传递依赖

"这个依赖是从哪里来的？"
1. 点击依赖 → 查看 Requested by (请求者)
2. 显示完整的传递链：
   我的项目
     └── spring-boot-starter-web
          └── spring-boot-starter-tomcat
               └── tomcat-embed-core

"为什么有重复的类？"
1. 检查 Switches → Duplicate Classes 警告
2. 通常原因是两个不同的模块包含了同一个类
3. 解决方案：
   - 排除其中一个依赖
   - 使用 Gradle Module Metadata 声明 capabilites
   - 添加 dependency substitution 规则
```

## 8.3 UP-TO-DATE 判定诊断

```
构建扫描中查看 Task 的 UP-TO-DATE 原因：

Task: compileJava
Status: EXECUTED (而非 UP-TO-DATE)
Reason: "Input property 'source' file com/example/NewClass.java has been added."

"检查输入属性变化"：
  ┌──────────────────────────────────────┐
  │ Input Property         │  Status     │
  ├──────────────────────────────────────┤
  │ source                 │ CHANGED     │ ← 原因！
  │ classpath              │ UNCHANGED   │
  │ options.encoding       │ UNCHANGED   │
  │ options.release        │ UNCHANGED   │
  │ destinationDirectory   │ UNCHANGED   │
  └──────────────────────────────────────┘

"理解文件集合的增量"：
  - 文件添加: *.java 新文件 → CHANGED
  - 文件修改: *.java 修改 → CHANGED  
  - 文件删除: *.java 删除 → CHANGED
  - .class 文件: 不在 source 输入中，不触发
```

# 9. 隐私与安全

## 9.1 发送的数据

```text
Build Scan 发送到 scans.gradle.com 的数据包括：

1. 构建环境信息：
   - Gradle 版本, Java 版本, OS 类型和版本
   - CPU 核心数, 内存大小
   - 系统属性和环境变量名称 (但值被脱敏)

2. 构建配置：
   - 项目名称, 任务名称
   - 插件名称和版本
   - 仓库 URL (不含认证信息)
   - 依赖组名和工件名 (含版本号)

3. 构建执行：
   - 任务执行时间和结果
   - 文件路径 (相对路径, 不包含文件内容)
   - 测试类名和方法名
   - 异常堆栈跟踪 (可能包含文件路径)

4. 不会发送的数据：
   - 源代码内容
   - 资源文件内容
   - 认证信息 (密码/密钥)
   - 完整的环境变量值
   - 私有仓库的 URL (如果配置为敏感)
```

## 9.2 禁用特定数据

```groovy
// Gradle Enterprise 中控制数据捕获
gradleEnterprise {
    buildScan {
        capture {
            // 禁用文件指纹捕获 (减少数据传输量)
            fileFingerprints = false
            
            // 禁用任务输入文件列表
            taskInputFiles = false
            
            // 禁用构建日志捕获
            buildLogging = false
            
            // 禁用测试日志捕获
            testLogging = false
        }
        
        // 混淆 IP 地址
        obfuscation {
            ipAddresses { obfuscate -> obfuscate.mask("0.0.0.0") }
            hostname { obfuscate -> obfuscate.mask("unknown") }
            username { obfuscate -> obfuscate.mask("anonymous") }
        }
    }
}
```

## 9.3 自托管 Build Scan

```groovy
// 使用 Gradle Enterprise 自托管
// settings.gradle
plugins {
    id("com.gradle.enterprise") version "3.15.1"
}

gradleEnterprise {
    // 指向企业服务器，数据不会发送到 scans.gradle.com
    server = "https://gradle-enterprise.mycompany.com"
    
    buildScan {
        publishAlways()
        
        // 数据安全策略
        termsOfServiceUrl = "https://mycompany.com/gradle-enterprise-tos"
        termsOfServiceAgree = "yes"
        
        // 控制哪些项目可以上传
        // ...
    }
}
```

# 10. 构建扫描实用技巧

## 10.1 对比两次构建

```bash
# 在 Gradle Enterprise 中对比两次构建
# 使用标签或自定义值来查找和对比

# 构建 A (基线)
gradle clean build --scan -PbuildId=baseline

# 构建 B (变更后)
gradle clean build --scan -PbuildId=after-change

# 在 Gradle Enterprise 仪表板中：
# 1. 同时选中这两个构建
# 2. 使用 Compare 功能
# 3. 查看差异：
#    - 配置时间变化
#    - 任务执行时间变化
#    - 依赖版本变化
#    - 缓存命中率变化
```

## 10.2 构建扫描书签

```groovy
// 使用唯一的 tag 或 link 标记重要的构建
buildScan {
    // 标记为性能基线
    if (project.hasProperty('performanceBaseline')) {
        tag "performance-baseline"
        value "Baseline Date", new Date().format("yyyy-MM-dd")
        value "Java Version", System.getProperty("java.version")
        value "Gradle Version", gradle.gradleVersion
    }
    
    // 标记为依赖升级
    if (project.hasProperty('dependencyUpgrade')) {
        tag "dependency-upgrade"
        link "Upgrade PR", "https://github.com/mycompany/myapp/pull/${project.upgradePR}"
    }
    
    // 标记为发布构建
    if (gradle.startParameter.taskNames.contains("publish")) {
        tag "release"
        tag "version:${project.version}"
        value "Release Date", new Date().format("yyyy-MM-dd HH:mm:ss")
        link "Release", "https://github.com/mycompany/myapp/releases/tag/v${project.version}"
    }
}
```

## 10.3 自动化性能回归检测

```groovy
// 基于构建扫描数据做性能回归检测
// 使用 Gradle Enterprise API (需企业版)

// 通过 API 获取历史构建数据
// GET https://gradle-enterprise.mycompany.com/api/builds
// 参数: since=NOW-7D, tags=performance-baseline

// 在 CI 中集成
task checkPerformanceRegression {
    doLast {
        def baselineTime = 4500  // ms, 从历史扫描获取或配置文件读取
        def threshold = 1.2      // 20% 性能退化阈值
        
        // 从当前构建获取任务时间 (需要扫描已发布)
        def currentTaskTime = getCurrentTaskTime("compileJava")
        
        if (currentTaskTime > baselineTime * threshold) {
            logger.warn "性能退化告警: compileJava ($currentTaskTime ms > ${baselineTime * threshold} ms)"
            // 生成报告或发送通知
        }
    }
}

def getCurrentTaskTime(String taskName) {
    // 实际的实现需要访问构建扫描的结果或使用 Gradle Enterprise API
    // 这是一个简化版本
    return 5000
}
```
