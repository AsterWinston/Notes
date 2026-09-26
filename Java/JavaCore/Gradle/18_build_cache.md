
# 1. 概述

构建缓存是 Gradle 的核心性能优化功能。它通过缓存任务输出，使得在输入不变的情况下可以跳过任务执行，直接复用之前的输出。Gradle 支持本地构建缓存和远程构建缓存，两者可以同时启用以实现最佳的性能。

## 1.1 构建缓存的工作原理

```
构建缓存核心流程
┌─────────────────────────────────────────────────────────┐
│                      任务执行请求                         │
│                          │                               │
│                          ▼                               │
│            ┌────────────────────────┐                    │
│            │ 计算缓存键 (Cache Key)  │                    │
│            │  (基于任务输入哈希)      │                    │
│            └───────────┬────────────┘                    │
│                        │                                 │
│                        ▼                                 │
│            ┌────────────────────────┐                    │
│            │ 查询本地缓存             │                    │
│            │ (DirectoryBuildCache)    │                    │
│            └───────────┬────────────┘                    │
│                        │                                 │
│                  ┌─────┴──────┐                          │
│                  │ 命中 (HIT)  │  未命中 (MISS)           │
│                  └─────┬──────┘                          │
│                        │         │                       │
│                        ▼         ▼                       │
│            ┌────────────────────────────────┐            │
│            │ 查询远程缓存                      │            │
│            │ (HttpBuildCache)                  │            │
│            └───────────┬────────────────────┘            │
│                        │                                 │
│                  ┌─────┴──────┐                          │
│                  │ 命中 (HIT)  │  未命中 (MISS)           │
│                  └─────┬──────┘       │                   │
│                        │              ▼                   │
│                        │   ┌──────────────────┐          │
│                        │   │ 执行任务          │          │
│                        │   │ 存储输出到缓存    │          │
│                        │   └──────────────────┘          │
│                        │              │                   │
│                        ▼              ▼                   │
│               ┌────────────────────────────┐             │
│               │ 从缓存恢复输出到输出目录    │             │
│               │ (Unpack cached result)      │             │
│               └────────────────────────────┘             │
└─────────────────────────────────────────────────────────┘
```

# 2. 缓存键计算机制

## 2.1 缓存键的组成

```groovy
// 缓存键计算因素
// 以下所有因素组合为 SHA-256 哈希：
//
// 1. 任务类名和类加载器标识
// 2. 任务输入属性值（@Input, @InputFile 等）
// 3. 输入的哈希值
//    - 文件内容: SHA-256 哈希
//    - 文件路径: 经过标准化后的相对路径
// 4. Gradle 版本
// 5. BuildScript 内容（classpath 配置影响）
// 6. 系统属性（如果声明为 @Input）
// 7. 环境变量（如果声明为输入）
```

## 2.2 缓存键的调试

```bash
# 启用缓存调试日志
gradle build --build-cache -Dorg.gradle.caching.debug=true

# 查看详细的缓存键信息
gradle build --build-cache --info 2>&1 | findstr "Build cache key"

# 使用 build scan 查看缓存键
gradle build --build-cache --scan
```

输出示例：

```
Build cache key for task ':compileJava' is 8b9e3a2c1d4f5e6a7b8c9d0e1f2a3b4c
  Input property 'classpath' hash: a1b2c3d4e5f6...
  Input property 'source' hash: f6e5d4c3b2a1...
  Input property 'sourceCompatibility' hash: 1a2b3c4d...
  Input property 'targetCompatibility' hash: 4d3c2b1a...
```

## 2.3 缓存键冲突排查

```groovy
// 查看导致缓存未命中的原因
tasks.withType(JavaCompile).configureEach {
    outputs.doNotCacheIf("开发模式不缓存") {
        project.hasProperty('dev')
    }
}

// 自定义缓存键计算
tasks.register('myTask', MyCustomTask) {
    inputs.property('buildNumber', {
        // 使用构建号作为输入，但不参与缓存
        System.getenv('BUILD_NUMBER') ?: 'local'
    })
}
```

# 3. 本地构建缓存

## 3.1 DirectoryBuildCache

```groovy
// settings.gradle
buildCache {
    local {
        // 启用本地缓存
        directory = file("$rootDir/.gradle/build-cache")

        // 自动删除 30 天未使用的缓存条目
        removeUnusedEntriesAfterDays = 30

        // 启用本地缓存（默认已启用）
        enabled = true

        // 仅在远程缓存未命中时从本地推送
        // push = true  // 默认 true
    }
}
```

## 3.2 本地缓存目录结构

```
.gradle/build-cache/
├── cache-entries/          ← 缓存条目 (GZip 压缩的 tar 文件)
│   ├── 01/
│   │   └── 0123456789abcdef...
│   ├── ab/
│   │   └── abcdef0123456789...
│   └── ...
├── gc.properties           ← 目标连接配置
└── journal/                ← 缓存操作日志
```

## 3.3 本地缓存管理命令

```bash
# 清理本地构建缓存
gradle cleanBuildCache

# 清理所有 Gradle 缓存（包括构建缓存）
# Windows:
# Remove-Item -Recurse -Force %USERPROFILE%\.gradle\caches\build-cache-1
```

# 4. 远程构建缓存

## 4.1 HttpBuildCache

```groovy
// settings.gradle
buildCache {
    remote(HttpBuildCache) {
        // 远程缓存服务 URL（Gradle Enterprise 或自定义实现）
        url = uri('https://build-cache.company.com/cache/')

        // 认证凭据
        credentials {
            username = System.getenv('BUILD_CACHE_USER')
            password = System.getenv('BUILD_CACHE_PASSWORD')
        }

        // 启用/禁用推送和拉取
        push = !System.getenv('CI').toBoolean()  // 本地开发推送，CI 不推送
        pull = true                              // 总是拉取

        // 连接超时设置
        // allowUntrustedServer = false
        // allowInsecureProtocol = false
    }
}
```

## 4.2 推送/拉取策略

```groovy
// 针对不同环境的缓存策略
def isCi = System.getenv('CI')?.toBoolean() ?: false
def isMainBranch = System.getenv('GIT_BRANCH') == 'main'

buildCache {
    remote(HttpBuildCache) {
        url = uri('https://build-cache.company.com/cache/')

        // CI 环境：主分支推送，其他分支仅拉取
        push = isCi && isMainBranch
        pull = true

        credentials {
            username = System.getenv('GRADLE_CACHE_USER')
            password = System.getenv('GRADLE_CACHE_PASS')
        }
    }

    local {
        // 本地缓存总是启用
        enabled = true
        // 仅在没有远程缓存可用时使用本地推送
        push = true
    }
}

// 使用 UseDevelopmentOnly 标记
tasks.withType(JavaCompile).configureEach {
    outputs.cacheIf { !project.hasProperty('no-cache') }
}
```

## 4.3 远程缓存协议

```
HTTP 缓存协议交互
┌────────────────────────────────────────┐
│ 客户端 (Gradle)    →    服务器         │
│                                         │
│ 查询 (GET):                             │
│ GET /cache/{cache-key}                 │
│  → 200 OK + 缓存内容                   │
│  → 404 Not Found                       │
│                                         │
│ 存储 (PUT):                             │
│ PUT /cache/{cache-key}                 │
│  Content-Type: application/octet-stream │
│  Body: GZip 压缩的 tar 包              │
│  → 200 OK / 201 Created                │
│                                         │
│ 认证:                                   │
│  Header: Authorization: Basic base64    │
└────────────────────────────────────────┘
```

# 5. 可缓存任务配置

## 5.1 声明任务可缓存

```groovy
tasks.register('generateDocs', GenerateDocsTask) {
    // 明确声明任务可通过构建缓存恢复
    outputs.cacheIf {
        // 仅在特定条件下缓存（例如：非开发模式）
        !project.hasProperty('noDocsCache')
    }

    // 禁止缓存
    // outputs.doNotCacheIf("开发模式不缓存") { project.hasProperty('dev') }

    // 无条件缓存（默认行为，如果任务有明确输出）
    // outputs.cacheIf { true }
}
```

## 5.2 输出标准化配置

```groovy
abstract class GenerateDocsTask extends DefaultTask {

    @InputFile
    abstract RegularFileProperty getSource()

    @OutputDirectory
    abstract DirectoryProperty getOutputDir()

    @TaskAction
    void generate() {
        // 生成文档
        File output = outputDir.get().asFile
        output.mkdirs()

        // 注意：以下代码会导致缓存失效
        // new File(output, 'timestamp.txt').text = new Date().toString()

        // 正确做法：使用确定性内容
        String content = source.get().asFile.text
        new File(output, 'docs.html').text = "<html>$content</html>"
    }
}
```

## 5.3 缓存键输入控制

```groovy
tasks.register('cacheAwareTask') {

    // 使用 property 添加缓存键输入
    inputs.property('buildType', 'release')
    inputs.property('debug', false)

    // 使用 file 添加缓存键输入
    inputs.file(layout.projectDirectory.file('config.json'))

    // 跳过对某些属性的缓存键计算
    inputs.property('timestamp', System.currentTimeMillis())
        .withPropertyName('timestamp')
        .optional(true)  // 标记为可选

    // 使用文件属性的标准化
    inputs.file(layout.projectDirectory.file('version.txt'))
        .withPropertyName('versionFile')
        .withPathSensitivity(PathSensitivity.RELATIVE)

    outputs.dir(layout.buildDirectory.dir('output'))
        .withPropertyName('outputDir')

    doLast {
        println '执行任务'
    }
}
```

# 6. 输出标准化

## 6.1 运行时类路径标准化

Gradle 对类路径进行标准化处理，使其在不同构建环境间保持一致。

```groovy
// Gradle 默认的类路径标准化
// 1. JAR 文件顺序标准化（字母序）
// 2. 忽略 JAR 内部时间戳
// 3. 目录路径标准化（使用相对路径）

// 自定义标准化规则
normalization {
    runtimeClasspath {
        // 忽略类路径中的版本号差异
        ignore 'version-info.properties'
        ignore 'META-INF/MANIFEST.MF'
        ignore 'META-INF/maven/**'

        // 标准化属性文件
        metaInf {
            ignoreAttribute('Implementation-Version')
            ignoreAttribute('Build-Jdk')
        }

        // 忽略特定文件模式
        properties {
            ignoreProperty('generated.timestamp')
        }
    }
}
```

## 6.2 属性文件标准化

```groovy
normalization {
    // 忽略特定属性文件中的属性
    properties('**/build-info.properties') {
        ignoreProperty('build.timestamp')
        ignoreProperty('build.number')
    }

    // 忽略版本号固定模式的文件
    properties('**/gradle.properties') {
        ignoreProperty('version')
    }
}

// 针对文件集合的忽略
normalization {
    runtimeClasspath {
        // 忽略整个文件
        ignore '**/README.md'
        ignore '**/*.txt'
    }
}
```

## 6.3 标准化内部机制

```
输出标准化流程
┌──────────────────────────────────────────────┐
│ 1. 任务执行后，输出文件被收集                │
│ 2. 输出文件打包为 tar (GZip)                 │
│                                               │
│ 打包前标准化:                                │
│   ├── ZIP 条目排序标准化                     │
│   ├── 文件时间戳设置为固定值 (1980-01-01)    │
│   ├── 忽略 META-INF/MANIFEST.MF 可变属性     │
│   └── 应用 normalization 配置                │
│                                               │
│ 3. 计算标准化后的 tar 哈希                   │
│ 4. 以哈希为键存储到缓存                      │
└──────────────────────────────────────────────┘
```

# 7. 缓存未命中调试

## 7.1 调试命令与日志

```bash
# 启用详细的缓存调试信息
gradle build --build-cache --info -Dorg.gradle.caching.debug=true

# 输出示例:
# Build cache key for task ':compileJava' is 123abc...
# Build cache key is 123abc...
# Build cache store result for task ':compileJava' (size = 45.2 KB)

# 比较两次构建的缓存键
gradle compileJava --build-cache -Dorg.gradle.caching.debug=true > build1.log
gradle compileJava --build-cache -Dorg.gradle.caching.debug=true > build2.log
# 比较两个日志的缓存键是否相同
```

## 7.2 常见缓存未命中原因

| 原因 | 说明 | 解决方案 |
|------|------|----------|
| 文件内容变化 | 输入源文件被修改 | 正常行为，无需处理 |
| 绝对路径 | 缓存键包含绝对路径 | 使用 `@PathSensitive(PathSensitivity.RELATIVE)` |
| 时间戳输入 | 任务使用 `new Date()` 作为输入 | 使用确定性输入 |
| 系统属性变化 | `System.getProperty()` 作为输入 | 通过 `normalization` 忽略 |
| 类加载器变化 | 任务类来源不同 | 确保使用相同版本的任务类 |
| Gradle 版本变化 | 不同版本产生不同缓存键 | 统一开发环境 Gradle 版本 |

## 7.3 编程方式调试

```groovy
// 自定义缓存键调试
tasks.register('debugCacheKey') {
    doLast {
        def buildCache = gradle.services.get(
            org.gradle.caching.BuildCacheService
        )

        println "构建缓存服务类型: ${buildCache?.class?.name}"
        println "本地缓存目录: ${gradle.gradleUserHomeDir}/caches/build-cache-1"
    }
}

// 打印每个任务的缓存状态
gradle.taskGraph.afterTask { task, state ->
    if (state.hasProperty('buildCacheResult')) {
        println "[缓存] ${task.path}: ${state.buildCacheResult}"
    }
}
```

# 8. 构建缓存后端

## 8.1 HTTP 后端配置

```groovy
// settings.gradle - Gradle Enterprise 配置
buildCache {
    remote(HttpBuildCache) {
        url = uri('https://gradle-enterprise.company.com/cache/')

        // 使用 Gradle Enterprise 访问密钥
        credentials {
            username = System.getenv('GRADLE_CACHE_USER') ?: 'ci'
            password = System.getenv('GRADLE_CACHE_PASS')
        }

        // 高级配置
        push = true
        pull = true

        // 使用 Expect: 100-Continue
        // allowUntrustedServer = false
    }
}
```

## 8.2 自定义缓存后端

```groovy
// 实现自定义 BuildCacheService
import org.gradle.caching.BuildCacheService
import org.gradle.caching.BuildCacheEntryReader
import org.gradle.caching.BuildCacheEntryWriter
import org.gradle.caching.BuildCacheKey
import org.gradle.caching.BuildCacheServiceFactory
import org.gradle.caching.BuildCacheServiceFactory.Describer

class S3BuildCacheService implements BuildCacheService {
    private final AmazonS3 s3Client
    private final String bucketName
    private final Logger logger

    S3BuildCacheService(AmazonS3 client, String bucket) {
        this.s3Client = client
        this.bucketName = bucket
        this.logger = Logging.getLogger(S3BuildCacheService)
    }

    @Override
    boolean load(BuildCacheKey key, BuildCacheEntryReader reader) {
        try {
            S3Object object = s3Client.getObject(bucketName, key.hashCode)
            if (object != null) {
                InputStream stream = object.getObjectContent()
                try {
                    reader.readFrom(stream)
                    logger.info("缓存命中: ${key.hashCode}")
                    return true
                } finally {
                    stream.close()
                }
            }
            return false
        } catch (Exception e) {
            logger.warn("从 S3 加载缓存失败: ${e.message}")
            return false
        }
    }

    @Override
    void store(BuildCacheKey key, BuildCacheEntryWriter writer) {
        try {
            ByteArrayOutputStream baos = new ByteArrayOutputStream()
            writer.writeTo(baos)

            ObjectMetadata metadata = new ObjectMetadata()
            metadata.setContentLength(baos.size())

            s3Client.putObject(bucketName, key.hashCode,
                               new ByteArrayInputStream(baos.toByteArray()),
                               metadata)

            logger.info("缓存存储: ${key.hashCode} (${baos.size()} bytes)")
        } catch (Exception e) {
            logger.warn("存储缓存到 S3 失败: ${e.message}")
        }
    }

    @Override
    void close() {
        s3Client.shutdown()
    }
}

// 注册自定义缓存服务
class S3BuildCacheServiceFactory implements BuildCacheServiceFactory<S3BuildCache> {
    @Override
    BuildCacheService createBuildCacheService(S3BuildCache config, Describer describer) {
        describer.type("S3").config("bucket", config.bucket)
        return new S3BuildCacheService(createS3Client(), config.bucket)
    }

    private AmazonS3 createS3Client() {
        return AmazonS3ClientBuilder.standard()
            .withRegion(Regions.US_EAST_1)
            .build()
    }
}

// 自定义缓存配置类
interface S3BuildCache extends BuildCache {
    @Input
    Property<String> getBucket()
}
```

## 8.3 本地 + 远程缓存协同

```groovy
// 协同工作策略
buildCache {
    local {
        // 本地缓存始终作为第一级缓存
        enabled = true
    }

    remote(HttpBuildCache) {
        url = uri('https://cache.company.com/cache/')
        // CI 环境：允许推送以预热缓存
        push = true
        pull = true

        credentials {
            username = System.getenv('BUILD_CACHE_USER')
            password = System.getenv('BUILD_CACHE_PASS')
        }
    }
}

// 两阶段查找：
// 第 1 步: 检查本地缓存 → 如果命中，立即使用
// 第 2 步: 检查远程缓存 → 如果命中，下载到本地，然后使用
// 第 3 步: 都未命中 → 执行任务，结果存储到本地 + 远程
```

# 9. CI 集成模式

## 9.1 CI 缓存策略

```groovy
// CI 环境变量检测
def isCi = System.getenv('CI')?.toBoolean() ?: false
def isPullRequest = System.getenv('GITHUB_EVENT_NAME') == 'pull_request'

ext {
    ci = isCi
    pr = isPullRequest
    branch = System.getenv('GIT_BRANCH') ?: 'local'
}

// settings.gradle - CI 环境配置
buildCache {
    local {
        enabled = true
    }

    remote(HttpBuildCache) {
        url = uri('https://build-cache.company.com/cache/')

        // 主分支和发布分支的 CI 构建可以推送到缓存
        push = isCi && (branch == 'main' || branch.startsWith('release/'))
        pull = true  // 总是拉取

        credentials {
            username = System.getenv('CACHE_USER')
            password = System.getenv('CACHE_PASSWORD')
        }
    }
}
```

## 9.2 缓存预热 (Cache Warming)

```groovy
// 夜间构建任务 - 缓存预热
tasks.register('cacheWarming') {
    group = 'build-cache'
    description = '执行全部任务以预热构建缓存'

    dependsOn tasks.matching {
        it.name in ['compileJava', 'compileTestJava', 'test',
                    'javadoc', 'checkstyleMain', 'checkstyleTest']
    }
}

// CI 定时任务触发
// 0 3 * * * ./gradlew cacheWarming --build-cache
```

## 9.3 缓存命中率监控

```groovy
// 监控缓存命中率
gradle.taskGraph.beforeTask { task ->
    task.logging.captureStandardOutput LogLevel.INFO
}

gradle.taskGraph.afterTask { task, state ->
    if (state.hasProperty('buildCacheResult')) {
        String result = state.buildCacheResult.toString()
        if (result == 'HIT') {
            println "[CACHE-HIT] ${task.path}"
        } else if (result == 'MISS') {
            println "[CACHE-MISS] ${task.path}"
        }
    }
}
```

# 10. 构建缓存安全

## 10.1 认证机制

```groovy
buildCache {
    remote(HttpBuildCache) {
        url = uri('https://build-cache.company.com/cache/')

        // Basic 认证
        credentials {
            username = System.getenv('CACHE_USER')
            password = System.getenv('CACHE_PASS')
        }

        // 客户端证书认证（需要额外的 SSL 配置）
        // 注意：credentials 密码应使用环境变量或加密存储
    }
}
```

## 10.2 访问控制

```
缓存访问控制矩阵
┌─────────────────┬──────────┬──────────┬──────────┐
│ 角色             │ 读 (PULL)│ 写 (PUSH)│ 管理     │
├─────────────────┼──────────┼──────────┼──────────┤
│ 开发者           │    ✓     │    ✓     │    ✗     │
│ CI (Pull Request)│    ✓     │    ✗     │    ✗     │
│ CI (Main Branch) │    ✓     │    ✓     │    ✗     │
│ 运维             │    ✓     │    ✓     │    ✓     │
└─────────────────┴──────────┴──────────┴──────────┘
```

## 10.3 敏感数据处理

```groovy
// 防止敏感数据进入缓存
tasks.configureEach {
    // 不要在缓存中存储凭据信息
    outputs.doNotCacheIf("包含凭据信息") {
        task.hasProperty('credential') || task.name.contains('deploy')
    }
}

// 使用临时密钥
tasks.register('buildWithCredentials') {
    inputs.property('apiKey', System.getenv('API_KEY') ?: 'test-key')
    outputs.cacheIf { System.getenv('API_KEY') == null }

    doLast {
        // 使用凭据执行任务
    }
}
```

# 11. 性能影响分析

## 11.1 缓存命中率计算

```groovy
// 自定义缓存统计收集
class CacheStatistics {
    int totalTasks = 0
    int cacheHits = 0
    int cacheMiss = 0
    long totalTimeSaved = 0

    double getHitRate() {
        return totalTasks > 0 ? (double) cacheHits / totalTasks * 100 : 0
    }

    @Override
    String toString() {
        """
        === 构建缓存统计 ===
        总任务数:     ${totalTasks}
        缓存命中:     ${cacheHits}
        缓存未命中:   ${cacheMiss}
        缓存命中率:   ${String.format('%.1f%%', hitRate)}
        节省时间估算: ${totalTimeSaved}ms
        """.stripIndent()
    }
}

def stats = new CacheStatistics()
gradle.taskGraph.afterTask { task, state ->
    stats.totalTasks++
    if (state.hasProperty('buildCacheResult')) {
        if (state.buildCacheResult.toString() == 'HIT') {
            stats.cacheHits++
        } else {
            stats.cacheMiss++
        }
    }
}

gradle.buildFinished {
    println stats
}
```

## 11.2 成本收益分析

| 缓存类型 | 收益 | 成本 | 适用场景 |
|----------|------|------|----------|
| 本地缓存 | 最快速，零网络开销 | 磁盘空间 | 所有场景 |
| 远程缓存 | 跨机器共享，CI 提速 | 网络带宽 + 服务器 | 团队开发 + CI |
| 无缓存 | 不受缓存问题影响 | 每次完整构建 | 不推荐 |

**估算公式：**

```
缓存收益 = (FullBuildTime - CachedBuildTime) × 缓存命中次数

例如:
  完整构建时间: 5 分钟
  缓存构建时间: 30 秒
  每天构建 10 次
  命中率: 70%

  每天节省: (300 - 30) × 10 × 0.7 = 1890 秒 = 31.5 分钟
  每年节省: 31.5 × 250 ≈ 131 小时
```

## 11.3 缓存配置优化

```groovy
// 根据项目规模调整缓存策略
buildCache {
    local {
        // 大型项目: 保留更长时间
        removeUnusedEntriesAfterDays = 60
    }

    remote(HttpBuildCache) {
        url = uri('https://cache.company.com/cache/')

        // 大项目: 使用更大的连接超时
        // timeout = 30_000  // 30 秒
    }
}
```

# 12. 缓存日志与统计

## 12.1 启用缓存日志

```bash
# 基本缓存日志
gradle build --build-cache --info 2>&1 | findstr "cache"

# 详细的缓存调试日志
gradle build --build-cache -Dorg.gradle.caching.debug=true

# Build Scan 方式（推荐）
gradle build --build-cache --scan
```

## 12.2 编程方式访问缓存信息

```groovy
// 获取缓存统计
tasks.register('cacheReport') {
    doLast {
        def cacheStats = project.gradle.sharedServices.registrations
            .find { it.name.contains('buildCache') }

        println "构建缓存类型: ${cacheStats?.class?.name}"
        println "当前缓存服务: ${gradle.sharedServices}"
    }
}

// 使用 BuildService 记录缓存统计
abstract class CacheMonitorService implements BuildService<CacheMonitorParams>,
                                              BuildListener {

    interface CacheMonitorParams extends BuildServiceParameters {
        Property<String> getReportPath()
    }

    private int hitCount = 0
    private int missCount = 0

    @Override
    void afterTask(Project project, Task task, TaskState state) {
        if (state.hasProperty('buildCacheResult')) {
            String result = state.buildCacheResult.toString()
            if (result == 'HIT') hitCount++
            if (result == 'MISS') missCount++
        }
    }

    @Override
    void buildFinished(BuildResult result) {
        int total = hitCount + missCount
        double rate = total > 0 ? (double) hitCount / total * 100 : 0
        println """
            |=== 构建缓存报告 ===
            |缓存命中:   $hitCount
            |缓存未命中: $missCount
            |命中率:     ${String.format('%.1f%%', rate)}
        """.stripMargin()
    }
}

// 注册服务
gradle.sharedServices.registerIfAbsent('cacheMonitor', CacheMonitorService) {
    parameters.reportPath = layout.buildDirectory.file('cache-report.txt')
}
```

# 13. 任务输出标准化详解

## 13.1 运行时类路径标准化

```groovy
normalization {
    runtimeClasspath {
        // 忽略特定文件模式
        ignore 'META-INF/MANIFEST.MF'

        // 忽略 JAR 中的元数据
        metaInf {
            // 忽略 JAR 清单文件中的动态属性
            ignoreAttribute('Built-By')
            ignoreAttribute('Build-Jdk')
            ignoreAttribute('Created-By')
            ignoreAttribute('Implementation-Version')
            ignoreAttribute('Implementation-Vendor')
            ignoreAttribute('Implementation-Build-Date')
        }
    }
}
```

## 13.2 属性标准化

```groovy
normalization {
    // 属性文件标准化
    properties('**/gradle.properties') {
        ignoreProperty('gradleVersion')
    }

    properties('**/version.properties') {
        ignoreProperty('buildTimestamp')
        ignoreProperty('gitHash')
    }

    // 资源文件标准化
    properties('**/application.properties') {
        ignoreProperty('server.port')
    }
}
```

## 13.3 完全标准化示例

```groovy
apply plugin: 'java'

normalization {
    runtimeClasspath {
        ignore 'version.properties'
        ignore 'META-INF/MANIFEST.MF'
        ignore 'META-INF/*.SF'
        ignore 'META-INF/*.DSA'
        ignore 'META-INF/*.RSA'

        metaInf {
            ignoreAttribute('Implementation-Title')
            ignoreAttribute('Implementation-Version')
            ignoreAttribute('Implementation-Vendor')
            ignoreAttribute('Implementation-Vendor-Id')
            ignoreAttribute('Specification-Title')
            ignoreAttribute('Specification-Version')
            ignoreAttribute('Specification-Vendor')
            ignoreAttribute('Built-By')
            ignoreAttribute('Build-Jdk')
            ignoreAttribute('Created-By')
            ignoreAttribute('Build-Timestamp')
        }
    }
}
```

# 14. 总结

| 概念 | 核心配置 | 关键点 |
|------|----------|--------|
| 缓存键 | 任务输入属性 | 所有 @Input 属性组合的 SHA-256 哈希 |
| 本地缓存 | `DirectoryBuildCache` | 共享工作目录，最快读取速度 |
| 远程缓存 | `HttpBuildCache` | 跨机器共享，CI 环境必备 |
| 输出标准化 | `normalization {}` | 忽略非确定性内容 |
| 缓存标记 | `outputs.cacheIf()` | 控制任务是否可缓存 |
| 调试 | `-Dorg.gradle.caching.debug=true` | 查看缓存键详细信息 |
| 安全性 | `credentials {}` | 使用环境变量保护凭据 |

构建缓存是 Gradle 最强大的性能优化功能之一。合理配置本地 + 远程缓存、正确声明任务输入输出、应用标准化规则，可以显著提升构建速度。
