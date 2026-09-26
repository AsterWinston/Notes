
# 1. 概述

依赖解析是 Gradle 构建系统的核心功能之一。Gradle 不仅能够从远程仓库下载依赖，还提供了丰富的版本冲突解决策略、变体感知解析、依赖锁定、能力管理等功能。本章深入剖析 Gradle 依赖解析的内部机制，涵盖从配置阶段到缓存管理的完整流程。

## 1.1 依赖解析全貌

```
依赖解析架构
┌──────────────────────────────────────────────────────────────┐
│ 配置文件 (Configuration)                                      │
│  ├── 依赖声明 (Dependency Declarations)                      │
│  ├── 依赖约束 (Dependency Constraints)                       │
│  ├── 排除规则 (Excludes)                                     │
│  └── 版本目录 (Version Catalogs)                             │
│                                                               │
│ 解析引擎 (Resolution Engine)                                  │
│  ├── 仓库查询 (Repository Query)                             │
│  ├── 版本选择 (Version Selection)                            │
│  ├── 变体选择 (Variant Selection)                            │
│  ├── 冲突解决 (Conflict Resolution)                          │
│  ├── 能力冲突 (Capability Conflict)                          │
│  └── 替换规则 (Substitution Rules)                           │
│                                                               │
│ 结果 (Resolution Result)                                      │
│  ├── 传递依赖图 (Transitive Dependency Graph)                │
│  ├── 已解析工件 (Resolved Artifacts)                         │
│  └── 依赖锁定文件 (Dependency Lock File)                     │
└──────────────────────────────────────────────────────────────┘
```

# 2. 配置 (Configuration)

## 2.1 配置的基本概念

Configuration 是 Gradle 中依赖分组的核心抽象。每个配置代表一组依赖的集合，用于特定的构建目的（编译、测试、运行时等）。

```groovy
// 声明自定义配置
configurations {
    myCustom

    // 创建配置并设置属性
    myCustomWithAttributes {
        canBeResolved = true
        canBeConsumed = false
        description = '自定义依赖解析配置'

        // 设置属性用于变体匹配
        attributes {
            attribute(Bundling.BUNDLING_ATTRIBUTE,
                      project.objects.named(Bundling, Bundling.EXTERNAL))
        }
    }
}

dependencies {
    myCustom 'com.google.guava:guava:33.0.0-jre'
    myCustom 'org.apache.commons:commons-lang3:3.14.0'
}
```

## 2.2 配置的类型

| 配置角色 | 说明 |
|----------|------|
| `canBeResolved` | 可被解析：读取依赖并下载工件 |
| `canBeConsumed` | 可被消费：被其他项目作为依赖使用 |
| 旧式 (Legacy) | 同时支持解析和消费（不推荐） |

```groovy
configurations {
    // 仅用于解析的配置
    myResolution {
        canBeResolved = true
        canBeConsumed = false
        description = '专门用于下载和处理依赖的配置'
    }

    // 仅用于消费的配置（暴露给其他项目）
    myApi {
        canBeResolved = false
        canBeConsumed = true
    }
}
```

## 2.3 内置标准配置

```
Java 插件标准配置
┌────────────────────────────────────────┐
│ compileClasspath (解析，仅编译)          │
│   └── implementation                  │
│   └── compileOnly                     │
│                                        │
│ annotationProcessor (解析，仅注解处理)    │
│   └── annotationProcessor             │
│                                        │
│ runtimeClasspath (解析，编译+运行时)     │
│   └── implementation                  │
│   └── runtimeOnly                     │
│                                        │
│ testCompileClasspath                   │
│ testRuntimeClasspath                   │
└────────────────────────────────────────┘
```

# 3. 解析策略 (ResolutionStrategy)

## 3.1 基本配置

`ResolutionStrategy` 允许精细控制依赖解析的每个环节。

```groovy
configurations.all {
    resolutionStrategy {
        // 失败时缓存动态版本 10 分钟
        cacheDynamicVersionsFor 10, 'minutes'

        // 失败时缓存变化模块 24 小时
        cacheChangingModulesFor 24, 'hours'

        // 当有版本冲突时立即失败
        failOnVersionConflict()

        // 当有动态版本时显示警告
        failOnDynamicVersions()

        // 当版本冲突时优先选择 newer
        preferProjectModules()
    }
}
```

## 3.2 eachDependency

`eachDependency` 钩子允许在解析过程中拦截每一个依赖，进行版本调整或替换。

```groovy
configurations.all {
    resolutionStrategy.eachDependency { DependencyResolveDetails details ->
        // 统一 Jackson 版本
        if (details.requested.group == 'com.fasterxml.jackson.core') {
            details.useVersion '2.17.0'
        }

        // 统一日志框架
        if (details.requested.group == 'org.slf4j') {
            details.useVersion '2.0.12'
        }

        // 替换特定模块
        if (details.requested.name == 'old-library') {
            details.useTarget 'com.example:new-library:3.0.0'
        }

        println "解析依赖: ${details.requested.group}:${details.requested.name}:${details.requested.version}"
    }
}
```

**内部机制：** `eachDependency` 在版本选择阶段之后、冲突解决之前被调用。钩子允许修改 `DependencyResolveDetails.requested` 的版本或替换目标。

## 3.3 dependencySubstitution

`dependencySubstitution` 提供更强大的依赖替换能力，包括将模块替换为项目依赖。

```groovy
configurations.all {
    resolutionStrategy.dependencySubstitution {
        // 用本地项目替换远程模块
        substitute module('com.example:core') using project(':core')

        // 用模块替换项目
        substitute project(':legacy') using module('com.example:legacy:2.0.0')

        // 用变体替换（支持不同 GAV 坐标）
        substitute module('com.example:api') using variant(module('com.example:api-v2:2.0')) {
            attributes {
                attribute(TargetJvmVersion.TARGET_JVM_VERSION_ATTRIBUTE, 17)
            }
        }

        // 用分类器替换
        substitute module('com.example:lib') using module('com.example:lib:1.0:all')
    }
}
```

## 3.4 componentSelection

`componentSelection` 允许在组件选择阶段过滤或接受特定版本。

```groovy
configurations.all {
    resolutionStrategy.componentSelection {
        all { ComponentSelection selection ->
            // 排除 alpha 和 beta 版本
            if (selection.candidate.version.contains('alpha') ||
                selection.candidate.version.contains('beta')) {
                selection.reject("不允许使用预发布版本")
            }
        }

        withModule('com.example:sensitive-lib') { ComponentSelection selection ->
            // 只接受特定版本范围
            if (!selection.candidate.version.startsWith('3.')) {
                selection.reject("sensitive-lib 只允许 3.x 版本")
            }
        }

        withModule('org.apache.logging.log4j:log4j-core') { ComponentSelection selection ->
            if (selection.candidate.version.startsWith('2.0') ||
                selection.candidate.version.startsWith('2.1') ||
                selection.candidate.version.startsWith('2.2') ||
                selection.candidate.version.startsWith('2.3') ||
                selection.candidate.version.startsWith('2.4') ||
                selection.candidate.version.startsWith('2.5') ||
                selection.candidate.version.startsWith('2.6') ||
                selection.candidate.version.startsWith('2.7') ||
                selection.candidate.version.startsWith('2.8') ||
                selection.candidate.version.startsWith('2.9') ||
                selection.candidate.version.startsWith('2.10') ||
                selection.candidate.version.startsWith('2.11') ||
                selection.candidate.version.startsWith('2.12') ||
                selection.candidate.version.startsWith('2.13') ||
                selection.candidate.version.startsWith('2.14') ||
                selection.candidate.version.startsWith('2.15') ||
                selection.candidate.version.startsWith('2.16')) {
                selection.reject("log4j-core 存在已知安全漏洞")
            }
        }
    }
}
```

## 3.5 force

`force` 方法可以强制指定依赖版本，覆盖所有传递依赖的版本请求。

```groovy
configurations.all {
    resolutionStrategy {
        // 强制特定版本
        force 'com.google.guava:guava:33.0.0-jre'
        force 'org.slf4j:slf4j-api:2.0.12', 'ch.qos.logback:logback-classic:1.5.6'
    }
}

// 推荐方式：使用 constraints 替代 force
dependencies {
    constraints {
        implementation('com.google.guava:guava') {
            version {
                strictly('33.0.0-jre')
            }
        }
    }
}
```

# 4. 版本冲突解决

## 4.1 冲突解决策略

Gradle 默认选择图中最高的版本来解决版本冲突。可以通过 `ResolutionStrategy` 修改此行为。

```groovy
configurations.all {
    resolutionStrategy {
        // 策略 1: 选择最新版本 (默认)
        // 无需额外配置

        // 策略 2: 版本冲突时构建失败
        failOnVersionConflict()

        // 策略 3: 当版本可能不是预期版本时失败
        // failOnNonReproducibleResolution()  // 需要 Gradle 7.x+

        // 策略 4: 偏向项目模块而非外部库
        preferProjectModules()
    }
}
```

## 4.2 版本冲突解决算法

```
版本冲突解决流程
┌──────────────────────────────────────────────┐
│ 1. 收集所有版本请求                           │
│    ├── 直接依赖版本                           │
│    ├── 传递依赖版本                           │
│    └── 约束 (Constraints) 版本               │
│                                                │
│ 2. 应用严格版本 (strictly)                    │
│    └── 如果存在 strictly，直接使用该版本       │
│                                                │
│ 3. 应用 prefer 版本                           │
│    └── 如果不存在 strictly，优先使用 prefer    │
│                                                │
│ 4. 取最高版本（默认策略）                     │
│    └── 在所有非 rejected 版本中选择最高       │
│                                                │
│ 5. 应用 componentSelection 规则               │
│    └── 使用规则过滤候选版本                   │
│                                                │
│ 6. 如果 failOnVersionConflict() 启用，        │
│    且存在多个非严格版本，构建失败              │
└──────────────────────────────────────────────┘
```

# 5. 富版本声明 (Rich Versioning)

## 5.1 版本约束语法

Gradle 支持声明版本约束来精细控制依赖的版本选择：

```groovy
dependencies {
    implementation('org.apache.commons:commons-lang3') {
        version {
            // require: 最低版本要求
            require '3.10'

            // strictly: 严格版本（必须使用该版本）
            strictly '3.14.0'

            // prefer: 首选版本（在冲突时优先选择）
            prefer '3.14.0'

            // reject: 拒绝特定版本
            reject '3.10.1', '3.11.0'

            // rejectAll: 拒绝所有版本（需要配合其他规则）
            // rejectAll()
        }
    }

    // 简写语法
    implementation('com.google.guava:guava') {
        version {
            strictly('[33.0.0, 34.0.0[')  // 版本范围
            prefer '33.5.0-jre'
        }
    }
}
```

## 5.2 版本范围语法

| 符号 | 示例 | 含义 |
|------|------|------|
| `[` `]` | `[1.0, 2.0]` | 闭区间：包含 1.0 和 2.0 |
| `]` `[` | `]1.0, 2.0[` | 开区间：不包含 1.0 和 2.0 |
| `+` | `1.+` | 通配符：匹配 1.x 所有版本 |
| `latest.release` | 特殊版本 | 最新的正式版 |
| `latest.integration` | 特殊版本 | 最新的集成版本（含快照） |

```groovy
dependencies {
    // 版本范围
    implementation('org.springframework:spring-core:[5.3.0, 6.0.0[')
    implementation('com.fasterxml.jackson.core:jackson-databind:2.+')

    // 严禁在生产中使用 latest.release（非确定性构建）
    // implementation('org.example:lib:latest.release')
}
```

## 5.3 严格版本与依赖约束

```groovy
dependencies {
    // 全局约束：影响所有配置中的依赖
    constraints {
        implementation('org.apache.logging.log4j:log4j-core') {
            version {
                strictly('[2.17.0,)')  // 至少 2.17.0
            }
            because '早期版本存在 Log4Shell 漏洞 CVE-2021-44228'
        }

        implementation('com.google.guava:guava') {
            version {
                strictly('33.0.0-jre')
                reject('33.1.0-jre', '33.2.0-jre')
            }
            because '版本存在已知问题'
        }
    }
}
```

# 6. 动态版本与变化模块

## 6.1 动态版本

动态版本（如 `1.+`、`latest.release`）在每次解析时可能返回不同的结果。

```groovy
configurations.all {
    resolutionStrategy {
        // 缓存动态版本结果的时间
        cacheDynamicVersionsFor 10, 'minutes'

        // 缓存变化模块的时间
        cacheChangingModulesFor 0, 'seconds'  // 总是检查

        // 禁止动态版本
        // failOnDynamicVersions()
    }
}

dependencies {
    // 动态版本（不推荐在生产中使用）
    implementation 'com.example:lib:1.+'  // 匹配 1.x 最新

    // 变化模块（SNAPSHOT）
    implementation 'com.example:lib:1.0-SNAPSHOT'
}
```

## 6.2 缓存控制

```bash
# 强制刷新所有依赖
gradle build --refresh-dependencies

# 仅刷新变化模块
gradle build --refresh-keys

# 清除缓存
rm -rf %USERPROFILE%\.gradle\caches
```

## 6.3 内部机制：缓存键

```
缓存键组成 (ModuleResolutionCacheKey)
┌──────────────────────────────────────┐
│ 1. 仓库 URL                          │
│ 2. 模块 GAV 坐标                     │
│ 3. 版本字符串                        │
│ 4. 动态版本: 过期时间戳 + 版本范围   │
│ 5. 变化模块: 上次检查时间            │
│                                       │
│ 缓存目录结构:                        │
│ .gradle/caches/                      │
│   modules-2/                         │
│     metadata-2.x/   ← 元数据         │
│     files-2.x/      ← JAR 文件       │
│   transforms-3/     ← 工件转换缓存   │
└──────────────────────────────────────┘
```

# 7. 模块替换

## 7.1 modules Replacement

当某个库被重命名或分叉时，需要将旧坐标映射到新坐标。

```groovy
dependencies {
    modules {
        // 将 javax 迁移到 jakarta
        module('javax.annotation:javax.annotation-api') {
            replacedBy('jakarta.annotation:jakarta.annotation-api', '迁移到 Jakarta')
        }

        module('com.google.collections:google-collections') {
            replacedBy('com.google.guava:guava', 'Google Collections 已被合并到 Guava')
        }

        module('commons-logging:commons-logging') {
            replacedBy('org.slf4j:jcl-over-slf4j', '使用 SLF4J 替代 Commons Logging')
        }
    }
}
```

## 7.2 组件元数据规则 (Component Metadata Rule)

通过组件元数据规则可以在解析时动态修改模块的元数据（POM/ivy.xml）。

```groovy
class JacksonFixRule implements ComponentMetadataRule {
    void execute(ComponentMetadataContext context) {
        context.details.allVariants {
            withDependencies { dependencies ->
                dependencies.findAll { dep ->
                    dep.group == 'org.codehaus.jackson'
                }.each { dep ->
                    dep.group = 'com.fasterxml.jackson.core'
                    dep.version {
                        prefer '2.17.0'
                    }
                }
            }
        }
    }
}

dependencies {
    components {
        withModule('com.example:some-lib', JacksonFixRule)

        all { ComponentMetadataDetails details ->
            if (details.id.group.startsWith('com.example')) {
                // 将状态从 "integration" 改为 "release"
                if (details.status == 'integration') {
                    details.status = 'release'
                }
            }
        }
    }
}
```

# 8. 变体感知依赖解析

## 8.1 变体与属性

Gradle 支持变体感知依赖解析，允许根据属性（如 JVM 版本、OS 架构）选择合适的依赖变体。

```groovy
// 定义属性模式
configurations.all {
    attributes {
        attribute(TargetJvmVersion.TARGET_JVM_VERSION_ATTRIBUTE, 17)
        attribute(Usage.USAGE_ATTRIBUTE, project.objects.named(Usage, Usage.JAVA_RUNTIME))
        attribute(Bundling.BUNDLING_ATTRIBUTE,
                  project.objects.named(Bundling, Bundling.EXTERNAL))
    }
}

// 在消费端声明属性约束
dependencies {
    attributesSchema {
        // 定义兼容性规则
        attribute(TargetJvmVersion.TARGET_JVM_VERSION_ATTRIBUTE) {
            compatibilityRules {
                // 高版本兼容低版本
                TargetJvmVersionCompatibilityRule
            }
            disambiguationRules {
                // 在多个变体中优先选择高版本
                TargetJvmVersionDisambiguationRule
            }
        }
    }
}

// 兼容性规则实现
abstract class TargetJvmVersionCompatibilityRule
        implements AttributeCompatibilityRule<Integer> {

    @Override
    void execute(CompatibilityCheckDetails<Integer> details) {
        Integer consumer = details.consumerValue
        Integer producer = details.producerValue
        if (producer != null && consumer != null && producer <= consumer) {
            details.compatible()
        }
    }
}

// 消歧规则实现
abstract class TargetJvmVersionDisambiguationRule
        implements AttributeDisambiguationRule<Integer> {

    @Override
    void execute(MultipleCandidatesDetails<Integer> details) {
        List<Integer> values = details.candidateValues as List
        if (!values.isEmpty()) {
            details.closestMatch(values.max())
        }
    }
}
```

## 8.2 变体选择流程

```
变体选择流程
┌──────────────────────────────────────────────┐
│ 1. 消费者声明所需属性                        │
│    例如: {jvm:17, usage:java-runtime}        │
│                                                │
│ 2. 查询生产者变体列表                         │
│    变体A: {jvm:11, usage:java-api}            │
│    变体B: {jvm:17, usage:java-runtime}        │
│    变体C: {jvm:17, usage:java-api}            │
│                                                │
│ 3. 兼容性匹配                                 │
│    变体A: jvm 11 <= 17 → compatible          │
│    变体B: jvm 17 <= 17 → compatible✓         │
│    变体C: jvm 17 <= 17 → compatible (但 usage 不匹配) │
│                                                │
│ 4. 消歧规则                                   │
│    如果多个变体兼容，应用 disambiguation 规则  │
│    优先选择变体 B                              │
└──────────────────────────────────────────────┘
```

# 9. 能力 (Capabilities)

## 9.1 能力声明

能力 (Capability) 用于声明模块提供的功能。当一个功能被多个模块提供时，Gradle 会检测冲突。

```groovy
// 在发布端声明能力
components.withType(AdhocComponentWithVariants) { component ->
    component.addVariantsFromConfiguration(configurations.apiElements) {
        capability('com.example:logging:1.0.0')
    }
}
```

## 9.2 能力冲突解决

```groovy
configurations.all {
    resolutionStrategy.capabilitiesResolution {
        // 当 log4j 和 logback 都提供 "slf4j-binding" 能力时
        withCapability('com.example:slf4j-binding') {
            // 手动选择 logback
            select('ch.qos.logback:logback-classic')
        }

        withCapability('org.slf4j:slf4j-binding') {
            // 选择具有最高版本的候选者
            selectHighestVersion()
        }
    }
}

// 另一种方式：在 dependencies 中声明
dependencies {
    components {
        // 声明模块提供的能力
        withModule('ch.qos.logback:logback-classic') {
            allVariants {
                withCapabilities {
                    addCapability('com.example', 'logging', '1.0.0')
                }
            }
        }
    }
}
```

## 9.3 能力冲突检测

```groovy
configurations.all {
    resolutionStrategy {
        // 能力冲突时立即失败
        capabilitiesResolution {
            all {
                selectHighestVersion()  // 或选择特定模块
            }
        }
    }
}
```

# 10. 依赖锁定 (Dependency Locking)

## 10.1 启用依赖锁定

依赖锁定将传递依赖的确切版本记录到锁文件中，确保跨环境构建的可重复性。

```groovy
dependencyLocking {
    lockAllConfigurations()  // 锁定所有配置
    // 或锁定特定配置
    // lockMode = LockMode.STRICT  // 严格模式：锁定所有依赖
}

// 生成锁定文件
// gradle resolveAndLockAll --write-locks  // Gradle 7.x+
// gradle dependencies --write-locks       // Gradle 6.x+
```

生成的文件位于 `gradle/dependency-locks/` 目录下：

```
gradle/dependency-locks/
  compileClasspath.lockfile
  runtimeClasspath.lockfile
  testCompileClasspath.lockfile
  testRuntimeClasspath.lockfile
```

## 10.2 锁文件格式

```
# This is a Gradle generated file for dependency locking.
# Manual edits can break the build and are not advised.
com.fasterxml.jackson.core:jackson-annotations:2.17.0=compileClasspath,runtimeClasspath
com.fasterxml.jackson.core:jackson-core:2.17.0=compileClasspath,runtimeClasspath
com.fasterxml.jackson.core:jackson-databind:2.17.0=compileClasspath,runtimeClasspath
com.google.guava:guava:33.0.0-jre=compileClasspath,runtimeClasspath
com.google.guava:failureaccess:1.0.2=compileClasspath,runtimeClasspath
org.slf4j:slf4j-api:2.0.12=compileClasspath,runtimeClasspath
empty=annotationProcessor
```

## 10.3 更新锁文件

```bash
# 更新所有配置的锁文件
gradle dependencies --write-locks

# 更新特定配置
gradle dependencies --configuration runtimeClasspath --write-locks

# 更新单个依赖的锁
gradle dependencies --update-locks com.google.guava:guava

# 更新多个依赖
gradle dependencies --update-locks 'com.google.guava:*' --write-locks
```

## 10.4 内部机制

```
依赖锁定流程
┌──────────────────────────────────────────────┐
│ 1. 解析依赖图（应用所有 resolutionStrategy） │
│ 2. 获取每个模块的已解析版本                  │
│ 3. 与 lockfile 中的版本比较                  │
│    ├── 匹配: 使用锁定的版本                  │
│    ├── 锁文件中存在新模块: 添加              │
│    └── 不匹配: 根据 lockMode 处理            │
│        ├── STRICT: 构建失败                  │
│        └── LENIENT: 使用新版本（默认）       │
│ 4. 写入新的 lockfile（如果指定 --write-locks）│
└──────────────────────────────────────────────┘
```

# 11. 平台对齐 (Alignment)

## 11.1 平台声明

平台 (Platform) 用于对齐一组相关依赖的版本，类似 Maven 的 BOM。

```groovy
// 声明一个平台
dependencies {
    // BOM 导入
    implementation platform('org.springframework.boot:spring-boot-dependencies:3.2.5')
    implementation platform('com.fasterxml.jackson:jackson-bom:2.17.0')

    // 强制执行平台
    implementation enforcedPlatform('org.springframework.boot:spring-boot-dependencies:3.2.5')

    // 普通依赖：版本由平台管理
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'com.fasterxml.jackson.core:jackson-databind'
}
```

## 11.2 虚拟平台

虚拟平台 (Virtual Platform) 不发布实际的 BOM 文件，仅在构建脚本中定义：

```groovy
// 构建一个自定义 BOM 项目
plugins {
    id 'java-platform'
}

javaPlatform {
    allowDependencies()  // 允许声明依赖
}

dependencies {
    // 对 BOM 项目可直接声明依赖
    constraints {
        api 'com.google.guava:guava:33.0.0-jre'
        api 'org.slf4j:slf4j-api:2.0.12'
        api 'com.fasterxml.jackson.core:jackson-databind:2.17.0'
    }
}
```

## 11.3 平台内部机制

```
平台对齐流程
┌──────────────────────────────────────────────┐
│ 1. 识别 platform/enforcedPlatform 声明       │
│ 2. 解析平台 BOM，提取所有版本约束            │
│ 3. 将约束注入依赖图                          │
│    ├── platform: 作为 prefer 约束            │
│    └── enforcedPlatform: 作为 strict 约束    │
│ 4. 应用版本冲突解决规则                      │
│ 5. 确保所有来自平台的依赖版本一致            │
└──────────────────────────────────────────────┘
```

# 12. 排除规则 (Excludes)

## 12.1 局部排除

```groovy
dependencies {
    implementation('org.springframework.boot:spring-boot-starter-web') {
        // 排除传递依赖
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-tomcat'
        exclude group: 'com.fasterxml.jackson.core'
    }

    implementation('com.example:some-lib') {
        // 使用属性进行排除
        exclude group: 'commons-logging', module: 'commons-logging'
        // 因为要使用 jcl-over-slf4j 替代
    }

    runtimeOnly 'org.springframework.boot:spring-boot-starter-undertow'
}
```

## 12.2 全局排除

```groovy
configurations.all {
    exclude group: 'commons-logging', module: 'commons-logging'
    exclude group: 'log4j', module: 'log4j'
}

configurations {
    all {
        exclude group: 'org.codehaus.groovy', module: 'groovy-all'
    }

    // 在特定配置中排除
    compileClasspath {
        exclude group: 'org.springframework.boot',
                 module: 'spring-boot-devtools'
    }
}

// 使用组件元数据规则进行更精细的排除
dependencies {
    components {
        withModule('org.springframework.boot:spring-boot-starter-web') {
            allVariants {
                withDependencies {
                    removeAll { it.group == 'org.springframework.boot' &&
                                it.name == 'spring-boot-starter-tomcat' }
                }
            }
        }
    }
}
```

## 12.3 排除的内部机制

```
排除规则处理流程
┌──────────────────────────────────────────────┐
│ 1. 构建依赖图（包含所有传递依赖）            │
│ 2. 对每个依赖应用排除规则                    │
│    ├── 局部排除: 仅影响声明的依赖            │
│    └── 全局排除: 影响所有依赖                │
│ 3. 从依赖图中移除被排除的模块                │
│ 4. 被排除模块的传递依赖也会被排除            │
│    除非被其他模块直接引用                    │
│                                               │
│ 注意: 如果模块 A 排除模块 B，但模块 C 也     │
│ 依赖模块 B，则模块 B 仍会被包含              │
│ (除非 C 也排除了 B 或使用全局排除)           │
└──────────────────────────────────────────────┘
```

# 13. 传递依赖分析

## 13.1 gradle dependencies

```bash
# 查看所有依赖
gradle dependencies

# 查看特定配置
gradle dependencies --configuration runtimeClasspath

# 查看依赖树（扫描到特定深度）
gradle dependencies --configuration compileClasspath --scan

# 导出为文件
gradle dependencies --configuration runtimeClasspath > deps.txt
```

输出示例：

```
runtimeClasspath - 运行时依赖
+--- com.google.guava:guava:33.0.0-jre
|    +--- com.google.guava:failureaccess:1.0.2
|    +--- com.google.guava:listenablefuture:9999.0-empty-to-avoid-conflict-with-guava
|    +--- com.google.code.findbugs:jsr305:3.0.2
|    +--- org.checkerframework:checker-qual:3.41.0
|    \--- com.google.errorprone:error_prone_annotations:2.23.0
+--- org.slf4j:slf4j-api:2.0.12
\--- org.projectlombok:lombok:1.18.32 (c)
```

## 13.2 dependencyInsight

`dependencyInsight` 提供了比 `dependencies` 更深入的视图，展示特定依赖为什么被包含。

```bash
# 查看特定依赖的来源
gradle dependencyInsight --dependency guava

# 指定配置
gradle dependencyInsight --configuration runtimeClasspath --dependency slf4j-api

# 显示所有解析冲突
gradle dependencyInsight --dependency spring-core --configuration compileClasspath
```

输出示例：

```
com.google.guava:guava:33.0.0-jre
  变体 "jreRuntimeElements" [
    org.gradle.category            = library
    org.gradle.libraryelements     = jar
    org.gradle.status              = release
    org.gradle.usage               = java-runtime
  ]
  选择原因: 版本冲突 (选择了最高版本)

com.google.guava:guava:33.0.0-jre
+--- compileClasspath
\--- com.example:my-lib:1.0.0
     \--- compileClasspath

com.google.guava:guava -> 33.0.0-jre
+--- commons-configuration:commons-configuration:1.10 (要求 30.0-jre)
\--- com.example:another-lib:2.0.0 (要求 32.0-jre)
```

## 13.3 程序化依赖分析

```groovy
// 在构建脚本中分析依赖
task analyzeDependencies {
    doLast {
        configurations.runtimeClasspath.resolvedConfiguration.resolvedArtifacts.each { artifact ->
            println "---"
            println "模块:  ${artifact.moduleVersion.id}"
            println "文件:  ${artifact.file}"
            println "类型:  ${artifact.type}"
            println "分类器: ${artifact.classifier}"

            // 获取依赖来源
            configurations.runtimeClasspath.incoming.resolutionResult
                .getDependencies()
                .findAll { it.selected?.moduleVersion?.name == artifact.name }
                .each { dep ->
                    println "来源:  ${dep.from}"
                    println "请求:  ${dep.requested}"
                }
        }
    }
}
```

# 14. 依赖缓存机制

## 14.1 缓存存储结构

```
.gradle/caches/
├── modules-2/
│   ├── metadata-2.110/             ← 元数据缓存
│   │   └── descriptors/            ← POM/ivy 描述文件
│   └── files-2.1/                  ← 二进制工件缓存
│       └── <module-id>/            ← 模块目录
│           └── <version>/          ← 版本目录
│               └── <hash>/         ← 哈希目录
│                   └── <artifact>  ← 实际文件
├── transforms-3/                   ← 工件转换缓存
├── journal-1/                      ← 缓存日志
└── <build-cache>/                  ← 构建输出缓存
```

## 14.2 缓存清理

```bash
# 删除所有 Gradle 缓存
rm -rf %USERPROFILE%\.gradle\caches

# 仅删除工件缓存
rm -rf %USERPROFILE%\.gradle\caches\modules-2\files-2.1

# 使用 Gradle 内置清理
gradle cleanBuildCache

# 删除未使用的缓存（超过 30 天）
# buildCache {
#     local {
#         removeUnusedEntriesAfterDays = 30
#     }
# }
```

# 15. 仓库配置

## 15.1 多仓库策略

```groovy
repositories {
    // Maven Central
    mavenCentral()

    // Google
    google()

    // 自定义 Maven 仓库
    maven {
        url = uri('https://repo.example.com/releases')
        credentials {
            username = project.findProperty('repo.user') ?: System.getenv('REPO_USER')
            password = project.findProperty('repo.pass') ?: System.getenv('REPO_PASS')
        }
        // 仓库内容过滤
        content {
            includeGroup 'com.example'
            includeGroupByRegex 'com\\.example\\..*'
        }
        // 仅用于发布
        mavenContent {
            releasesOnly()
        }
    }

    // Ivy 仓库
    ivy {
        url = uri('https://repo.example.com/ivy')
        patternLayout {
            artifact '[organisation]/[module]/[revision]/[artifact]-[revision](-[classifier]).[ext]'
            ivy '[organisation]/[module]/[revision]/ivy-[revision].xml'
        }
    }

    // 本地仓库
    mavenLocal()

    // 扁平目录仓库
    flatDir {
        dirs 'libs'
    }
}
```

## 15.2 仓库认证

```groovy
repositories {
    maven {
        url = uri('https://private.repo.com/releases')

        // 方式 1: 基本认证
        credentials {
            username 'deployer'
            password 'secret123'
        }

        // 方式 2: 环境变量
        // credentials {
        //     username = System.getenv('NEXUS_USER')
        //     password = System.getenv('NEXUS_PASS')
        // }

        // 方式 3: 加密凭据（需要 gradle-credentials-plugin）
        // credentials(Credentials)
    }
}
```

# 16. 依赖解析调试

```bash
# 查看解析过程的详细日志
gradle build --info | findstr "Resolve"

# 调试依赖解析问题
gradle build --debug 2>&1 | findstr "Dependency" > resolution-debug.log

# 强制刷新（忽略缓存）
gradle build --refresh-dependencies

# 离线模式（仅使用缓存）
gradle build --offline
```

```groovy
// 在构建脚本中调试
configurations.all {
    incoming.beforeResolve {
        println "=== 开始解析: ${name} ==="
    }
    incoming.afterResolve {
        println "=== 解析完成: ${name} ==="
    }

    resolutionStrategy.eachDependency { details ->
        logger.debug("解析: ${details.requested.group}:${details.requested.name}:${details.requested.version}")
    }
}

// 记录解析时间
long startTime = System.currentTimeMillis()
gradle.taskGraph.whenReady { graph ->
    println "依赖解析耗时: ${System.currentTimeMillis() - startTime}ms"
}
```

# 17. 总结

| 功能 | 核心 API | 用途 |
|------|----------|------|
| 版本控制 | `ResolutionStrategy`, `eachDependency` | 统一版本、替换依赖 |
| 冲突解决 | `failOnVersionConflict()`, `strictly`, `prefer` | 处理版本冲突 |
| 变体选择 | `attributesSchema`, `AttributeCompatibilityRule` | 根据环境选变体 |
| 模块替换 | `modules.replacedBy()`, `ComponentMetadataRule` | 迁移和替代 |
| 依赖锁定 | `dependencyLocking`, `--write-locks` | 可重复构建 |
| 平台对齐 | `platform()`, `enforcedPlatform()` | BOM 对齐 |
| 能力管理 | `capabilities`, `withCapability()` | 冲突检测和解决 |
| 排除规则 | `exclude`, `components.allVariants` | 移除不必要依赖 |

依赖解析是 Gradle 最复杂的子系统之一。深入理解其内部机制，有助于构建稳定、可重复、高效的依赖管理策略。
