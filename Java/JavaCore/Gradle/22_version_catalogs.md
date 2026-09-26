
# 1. 版本目录概述

## 1.1 什么是版本目录

版本目录 (Version Catalog) 是 Gradle 7.0 引入的依赖管理功能，通过声明式的方式集中管理项目的依赖版本。它将依赖坐标（group、artifact、version）从构建脚本中抽离，实现在多个项目间共享。

版本目录的核心文件是 `libs.versions.toml`，使用 TOML 格式。

## 1.2 为什么使用版本目录

| 传统方式 | 版本目录方式 |
|----------|-------------|
| 依赖分散在多个 build.gradle | 集中在 libs.versions.toml |
| 版本号硬编码 | 版本声明统一管理 |
| 跨项目共享困难 | 天然支持多项目共享 |
| IDE 支持有限 | 类型安全访问器，完整自动补全 |
| 升级依赖需全局搜索 | 一处修改，全局生效 |

# 2. libs.versions.toml 格式

## 2.1 基本结构

```toml
# libs.versions.toml - 标准位置: gradle/libs.versions.toml

[versions]
# 版本声明区域
spring-boot = "3.2.0"
kotlin = "1.9.20"
junit = "5.10.1"
mockito = "5.7.0"

[libraries]
# 库声明区域
spring-boot-starter = { module = "org.springframework.boot:spring-boot-starter", version.ref = "spring-boot" }
spring-boot-starter-web = { module = "org.springframework.boot:spring-boot-starter-web", version.ref = "spring-boot" }
spring-boot-starter-test = { module = "org.springframework.boot:spring-boot-starter-test", version.ref = "spring-boot" }
kotlin-stdlib = { module = "org.jetbrains.kotlin:kotlin-stdlib-jdk8", version.ref = "kotlin" }
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit" }
mockito-core = { module = "org.mockito:mockito-core", version.ref = "mockito" }

# 带 group/name 分离的声明
commons-lang3 = { group = "org.apache.commons", name = "commons-lang3", version = "3.13.0" }

# 不带版本的库（使用 BOM 管理版本）
jackson-databind = { module = "com.fasterxml.jackson.core:jackson-databind" }

[bundles]
# 依赖包（一组相关依赖）
test-deps = ["junit-jupiter", "mockito-core", "spring-boot-starter-test"]
spring-web = ["spring-boot-starter", "spring-boot-starter-web"]

[plugins]
# 插件声明
spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
kotlin-jvm = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
```

## 2.2 TOML 语法要点

```toml
# TOML 的特性
# 1. 注释使用 #
# 2. 基本类型: 字符串, 整数, 浮点数, 布尔, 日期时间
# 3. 数组: [a, b, c]
# 4. 表: [section]
# 5. 内联表: { key = "value" }
# 6. 键名使用字母数字和下划线，不支持点号

# 版本可以使用复杂版本约束
[versions]
spring = "3.2.0"
# 不支持 rich version 字符串在 value 中，需要通过 version.ref 间接引用来添加约束
strict-guava = { strictly = "[28.0, 30.0[" }
prefer-log4j = { prefer = "2.20.0" }
```

## 2.3 库声明的多种形式

```toml
[libraries]
# 形式1: module 字符串 (推荐)
guava = { module = "com.google.guava:guava", version = "33.0.0-jre" }

# 形式2: module + version.ref
guava-versioned = { module = "com.google.guava:guava", version.ref = "guava-version" }

# 形式3: group + name + version
guava-detailed = { group = "com.google.guava", name = "guava", version = "33.0.0-jre" }

# 形式4: group + name + version.ref
guava-detailed-ref = { group = "com.google.guava", name = "guava", version.ref = "guava-version" }

# 形式5: 不带版本（由 BOM 或约束管理）
jackson-annotations = { module = "com.fasterxml.jackson.core:jackson-annotations" }

# 形式6: 简化字符串（仅当使用默认版本引用时）
# 不推荐，但允许: some-lib = "com.example:some-lib:1.0"
```

# 3. 在构建脚本中使用版本目录

## 3.1 settings.gradle 配置

```groovy
// settings.gradle
dependencyResolutionManagement {
    repositories {
        mavenCentral()
        google()
    }
    
    versionCatalogs {
        // 方式1: 使用默认名称 libs，自动从 gradle/libs.versions.toml 加载
        // 无需显式声明
        
        // 方式2: 创建命名的 catalog
        create("testLibs") {
            from(files("gradle/test-libs.versions.toml"))
        }
        
        // 方式3: 从 Maven BOM 创建 catalog
        create("springLibs") {
            from("org.springframework.boot:spring-boot-dependencies:3.2.0")
        }
        
        // 方式4: 从外部项目导入
        create("sharedLibs") {
            from(files("../shared/gradle/libs.versions.toml"))
        }
        
        // 方式5: 组合多个来源
        create("combinedLibs") {
            from(files("gradle/libs.versions.toml"))
            from(files("gradle/extra-libs.toml"))
            from("com.example:platform:1.0")
        }
    }
}
```

## 3.2 在 Groovy DSL 中使用

```groovy
// build.gradle - Groovy DSL
dependencies {
    // library 访问器：libs.<alias>
    implementation libs.guava
    implementation libs.commons.lang3
    implementation libs.spring.boot.starter
    
    // bundle 访问器：libs.bundles.<alias>
    testImplementation libs.bundles.test.deps
    implementation libs.bundles.spring.web
    
    // 带分类器的访问
    // 在 toml 中: mylib = { module = "com.example:mylib", version = "1.0", classifier = "sources" }
    // 使用: libs.mylib 自动处理分类器
    
    // 带有 excludes 的依赖
    implementation(libs.spring.boot.starter.web) {
        exclude group: "org.springframework.boot", module: "spring-boot-starter-tomcat"
    }
}
```

## 3.3 在 Kotlin DSL 中使用

```kotlin
// build.gradle.kts - Kotlin DSL
dependencies {
    // 类型安全访问器：自动补全
    implementation(libs.guava)
    implementation(libs.commons.lang3)
    implementation(libs.spring.boot.starter)
    
    // Bundle 使用
    testImplementation(libs.bundles.test.deps)
    implementation(libs.bundles.spring.web)
    
    // 带 excludes
    implementation(libs.spring.boot.starter.web) {
        exclude(group = "org.springframework.boot", module = "spring-boot-starter-tomcat")
    }
    
    // 带约束
    implementation(libs.spring.boot.starter) {
        version {
            strictly("3.2.0")
        }
    }
}

// 插件使用
plugins {
    alias(libs.plugins.spring.boot)
    alias(libs.plugins.kotlin.jvm)
}
```

## 3.4 访问器命名规则

| TOML 别名 | Groovy 访问器 | Kotlin 访问器 | 说明 |
|-----------|--------------|--------------|------|
| `guava` | `libs.guava` | `libs.guava` | 单段别名 |
| `commons-lang3` | `libs.commons.lang3` | `libs.commons.lang3` | 连字符转为点号 |
| `spring-boot-starter` | `libs.spring.boot.starter` | `libs.spring.boot.starter` | 多段连字符 |
| `bundles.test-deps` | `libs.bundles.test.deps` | `libs.bundles.test.deps` | Bundle 访问 |
| `plugins.kotlin-jvm` | `libs.plugins.kotlin.jvm` | `libs.plugins.kotlin.jvm` | 插件访问 |

```toml
# 命名建议：使用连字符分隔而非下划线
# ✅ 推荐
[libraries]
spring-boot-starter-web = { module = "...", version.ref = "spring-boot" }
# 访问: libs.spring.boot.starter.web

# ❌ 不推荐
[libraries]
spring_boot_starter_web = { module = "...", version.ref = "spring-boot" }
# 访问: libs.spring_boot_starter_web（Kotlin 可能有问题）
```

# 4. 版本目录高级特性

## 4.1 Rich Version 支持

```toml
[versions]
# 严格版本范围
guava-strict = { strictly = "[28.0, 30.0[" }

# 首选版本
log4j-prefer = { prefer = "2.20.0" }

# 严格 + 首选
spring-strict-prefer = { strictly = "3.2.0", prefer = "3.2.0" }

# 拒绝版本
junit-reject = { strictly = "5.10.1", reject = ["5.9.0", "5.8.0"] }

# 需求版本范围
kafka-range = { require = "[3.0.0, 4.0.0[" }

# 在库声明中使用
[libraries]
guava-rich = { module = "com.google.guava:guava", version.ref = "guava-strict" }
```

## 4.2 依赖约束声明

```toml
# 在 [versions] 中无法直接声明约束条件
# 但可以通过在 build.gradle 中使用版本目录时附加约束

# build.gradle.kts
dependencies {
    implementation(libs.guava) {
        version {
            prefer("33.0.0-jre")
            reject("32.0.0-jre")
        }
    }
    
    constraints {
        implementation("org.apache.commons:commons-lang3") {
            version {
                strictly("[3.0, 4.0[")
                prefer("3.13.0")
            }
        }
    }
}
```

## 4.3 BOM 与版本目录集成

```toml
# 方式1：直接在 catalog 中引用 BOM
[versions]
spring-boot = "3.2.0"

[libraries]
spring-boot-bom = { module = "org.springframework.boot:spring-boot-dependencies", version.ref = "spring-boot" }
spring-boot-starter-web = { module = "org.springframework.boot:spring-boot-starter-web" }
# 不带版本号，由 BOM 管理

# build.gradle 中引入 BOM
dependencies {
    implementation platform(libs.spring.boot.bom)
    implementation libs.spring.boot.starter.web
}
```

```groovy
// 方式2：从 Maven BOM 直接导入到 catalog
// settings.gradle
dependencyResolutionManagement {
    versionCatalogs {
        create("springLibs") {
            from("org.springframework.boot:spring-boot-dependencies:3.2.0")
        }
    }
}

// 使用时：springLibs.spring.boot.starter.web
// 自动继承了 BOM 中声明的版本
```

## 4.4 多文件 Catalog 组织

```
gradle/
├── libs.versions.toml        # 主 catalog
├── test-libs.versions.toml   # 测试依赖 catalog
├── build-libs.versions.toml  # 构建工具依赖 catalog
└── plugins.toml              # 插件 catalog
```

```toml
# gradle/test-libs.versions.toml
[versions]
junit = "5.10.1"
mockito = "5.7.0"

[libraries]
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit" }
mockito-core = { module = "org.mockito:mockito-core", version.ref = "mockito" }
junit-vintage = { module = "org.junit.vintage:junit-vintage-engine", version.ref = "junit" }
assertj-core = { module = "org.assertj:assertj-core", version = "3.24.2" }

[bundles]
testing = ["junit-jupiter", "mockito-core", "assertj-core"]
integration-test = ["junit-jupiter", "junit-vintage"]
```

```groovy
// settings.gradle
dependencyResolutionManagement {
    versionCatalogs {
        create("testLibs") {
            from(files("gradle/test-libs.versions.toml"))
        }
        create("buildLibs") {
            from(files("gradle/build-libs.versions.toml"))
        }
    }
}

// build.gradle 中使用
dependencies {
    testImplementation testLibs.bundles.testing
    integrationTestImplementation testLibs.bundles.integration.test
}
```

# 5. 多项目共享版本目录

## 5.1 跨项目共享策略

```
# 项目结构
root-project/
├── settings.gradle
├── build.gradle
├── gradle/
│   └── libs.versions.toml        # 根项目 catalog
├── subproject-a/
│   └── build.gradle
├── subproject-b/
│   └── build.gradle
└── buildSrc/                      # 可选: 用于自定义插件
    └── src/main/kotlin/
        └── Dependencies.kt        # 代码方式声明依赖（传统方式）
```

```groovy
// settings.gradle - 根项目
rootProject.name = "my-app"

include "subproject-a"
include "subproject-b"

dependencyResolutionManagement {
    repositories {
        mavenCentral()
    }
    
    // 默认 catalog 位于 gradle/libs.versions.toml
    // 所有子项目自动可用
}
```

## 5.2 从独立仓库共享 Catalog

```groovy
// settings.gradle
dependencyResolutionManagement {
    versionCatalogs {
        // 从本地目录加载共享 catalog
        create("companyLibs") {
            from(files("../shared-build-config/gradle/libs.versions.toml"))
        }
        
        // 从发布到 Maven 仓库的 catalog 加载
        create("publishedLibs") {
            from("com.mycompany:catalog:1.0")
        }
        
        // 从 URL 加载 (需要自定义 PluginResolver)
        // 不直接支持 URL，可以先将文件拷贝到本地
    }
}
```

## 5.3 发布版本目录

Gradle 支持将版本目录发布为 Maven 工件，供其他项目引用：

```groovy
// build.gradle - catalog 发布项目
plugins {
    id("version-catalog")
    id("maven-publish")
}

catalog {
    versionCatalog {
        from(files("gradle/libs.versions.toml"))
    }
}

publishing {
    publications {
        create<MavenPublication>("maven") {
            from(components["versionCatalog"])
        }
    }
    repositories {
        maven {
            url = uri("https://repo.example.com/releases")
        }
    }
}
```

# 6. 版本目录迁移与集成

## 6.1 从 buildSrc 迁移

```kotlin
// 旧方式: buildSrc/src/main/kotlin/Versions.kt
object Versions {
    const val spring = "3.2.0"
    const val kotlin = "1.9.20"
}

object Libs {
    const val springBootStarter = "org.springframework.boot:spring-boot-starter"
    const val kotlinStdlib = "org.jetbrains.kotlin:kotlin-stdlib-jdk8"
}

// 新方式: gradle/libs.versions.toml
// [versions]
// spring = "3.2.0"
// kotlin = "1.9.20"
//
// [libraries]
// spring-boot-starter = { module = "org.springframework.boot:spring-boot-starter", version.ref = "spring" }
// kotlin-stdlib = { module = "org.jetbrains.kotlin:kotlin-stdlib-jdk8", version.ref = "kotlin" }
```

迁移步骤：
1. 在项目的 `gradle/` 目录下创建 `libs.versions.toml`
2. 将 `Versions.kt` 中的版本常量迁移到 `[versions]` 节
3. 将依赖声明迁移到 `[libraries]` 节
4. 更新 `build.gradle.kts`，将 `implementation(Libs.xxx)` 替换为 `implementation(libs.xxx)`
5. 测试构建，确保所有依赖正确解析
6. 备份并移除 `buildSrc` 中的旧常量文件

## 6.2 从 buildscript ext 迁移

```groovy
// 旧方式：ext + dependencyManagement plugin
buildscript {
    ext {
        springBootVersion = "3.2.0"
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    }
}

// 新方式：
// gradle/libs.versions.toml
// [versions]
// spring-boot = "3.2.0"
//
// [plugins]
// spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
//
// build.gradle
// plugins {
//     alias(libs.plugins.spring.boot)
// }
```

## 6.3 结合 Platform 依赖

```groovy
// 当需要引入外部的 BOM Platform 时
// gradle/libs.versions.toml
// [versions]
// spring-cloud = "2023.0.0"
//
// [libraries]
// spring-cloud-dependencies = { module = "org.springframework.cloud:spring-cloud-dependencies", version.ref = "spring-cloud" }

// build.gradle
dependencies {
    implementation platform(libs.spring.cloud.dependencies)
    // 其他 Spring Cloud 依赖不需要版本号，由 platform 管理
}
```

# 7. 版本目录工作流与验证

## 7.1 依赖版本更新工具

```bash
# 使用 Gradle 命令查看所有依赖及版本
gradle dependencies --configuration runtimeClasspath

# 查看特定配置的依赖树
gradle dependencies --configuration compileClasspath --scan

# 使用 dependencyUpdates 插件检查可用更新
# build.gradle
plugins {
    id("com.github.ben-manes.versions") version "0.50.0"
}

# gradle dependencyUpdates - 查看哪些依赖有更新
# 输出格式类似：
# ------------------------------------------------------------
# :project
# ------------------------------------------------------------
# The following dependencies have later milestone versions:
#  - com.google.guava:guava [33.0.0-jre -> 33.1.0-jre]
#  - org.springframework.boot:spring-boot-starter [3.2.0 -> 3.2.1]
```

## 7.2 Catalog 验证

```groovy
// settings.gradle - 启用 catalog 严格验证
dependencyResolutionManagement {
    versionCatalogs {
        create("libs") {
            from(files("gradle/libs.versions.toml"))
        }
    }
}

// Gradle 在构建时自动验证 catalog 格式
// 常见错误：
// - TOML 语法错误
// - 重复的别名
// - 引用未声明的版本
// - 缺少必需字段 (module 或 group+name)

// 手动验证 catalog
// gradle tasks --group "help"
// 不直接提供 catalog 验证任务，但会在构建时抛出明确错误
```

## 7.3 调试 Catalog 问题

```bash
# 打印所有可用的 catalog 库
gradle help  # 检查构建配置是否正常

# 查看所有依赖解析
gradle dependencies --configuration runtimeClasspath

# 使用 --info 查看详细日志
gradle build --info | grep -i "catalog"

# 检查 TOML 语法
# 使用在线工具或 IDE 内置 TOML 编辑器
```

## 7.4 常见问题与解决方案

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| `Unresolved reference: libs` | settings.gradle 中未配置 catalog | 确保 `gradle/libs.versions.toml` 存在 |
| `Could not resolve` | 版本引用错误 | 检查 `version.ref` 是否正确 |
| 类型安全访问器不工作 | IDE 缓存问题 | 刷新 Gradle 项目 |
| Bundle 中的依赖版本冲突 | 隐式依赖被覆盖 | 使用 `constraints` 声明约束 |
| 插件版本未生效 | 插件语法错误 | 使用 `alias(libs.plugins.xxx)` |

# 8. 版本目录最佳实践

## 8.1 命名规范

```toml
[versions]
# ✅ 推荐：简洁有意义的名称
spring-boot = "3.2.0"
kotlin = "1.9.20"

# ❌ 避免：过于冗长或含糊不清
spring-boot-framework-version = "3.2.0"
v1 = "1.9.20"

[libraries]
# ✅ 推荐：反映 artifact 名称，用连字符
spring-boot-starter-web = { module = "...", version.ref = "spring-boot" }
guava = { module = "com.google.guava:guava", version = "33.0.0-jre" }

# ❌ 避免：与 artifact 名差异过大
web-starter = { module = "org.springframework.boot:spring-boot-starter-web" }

[bundles]
# ✅ 推荐：描述用途
spring-web = ["spring-boot-starter", "spring-boot-starter-web"]
test-deps = ["junit-jupiter", "mockito-core"]

# ❌ 避免：用途不明
bundle1 = ["junit-jupiter", "mockito-core"]
```

## 8.2 结构组织

```toml
# 推荐的 catalog 内部组织方式

[versions]
# === 框架版本 ===
spring-boot = "3.2.0"
spring-cloud = "2023.0.0"
kotlin = "1.9.20"

# === 数据库 ===
postgresql = "42.7.1"
flyway = "10.4.0"

# === 测试 ===
junit = "5.10.1"
mockito = "5.7.0"
testcontainers = "1.19.3"

# === 工具库 ===
guava = "33.0.0-jre"
commons-lang3 = "3.13.0"

[libraries]
# === Spring Boot Starters ===
spring-boot-starter = { module = "org.springframework.boot:spring-boot-starter", version.ref = "spring-boot" }
spring-boot-starter-web = { module = "org.springframework.boot:spring-boot-starter-web", version.ref = "spring-boot" }
spring-boot-starter-data-jpa = { module = "org.springframework.boot:spring-boot-starter-data-jpa", version.ref = "spring-boot" }
spring-boot-starter-test = { module = "org.springframework.boot:spring-boot-starter-test", version.ref = "spring-boot" }

# === 数据库 ===
postgresql = { module = "org.postgresql:postgresql", version.ref = "postgresql" }
flyway-core = { module = "org.flywaydb:flyway-core", version.ref = "flyway" }

# === 测试框架 ===
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit" }
mockito-core = { module = "org.mockito:mockito-core", version.ref = "mockito" }
testcontainers-postgresql = { module = "org.testcontainers:postgresql", version.ref = "testcontainers" }

# === 工具库 ===
guava = { module = "com.google.guava:guava", version.ref = "guava" }
commons-lang3 = { module = "org.apache.commons:commons-lang3", version.ref = "commons-lang3" }

[bundles]
# === 功能包 ===
spring-web = ["spring-boot-starter", "spring-boot-starter-web"]
spring-data = ["spring-boot-starter", "spring-boot-starter-data-jpa"]
testing = ["junit-jupiter", "mockito-core", "spring-boot-starter-test"]
utils = ["guava", "commons-lang3"]
```

## 8.3 版本映射表

```toml
# 对于大型团队，建议维护一个版本映射表
# 在 catalog 注释中记录为什么选择某个版本

[versions]
# LTS 版本，保留至 2026 年
spring-boot = "3.2.0"
# 与 Spring Boot 3.2.x 兼容的最高 Kotlin 版本
kotlin = "1.9.20"
# Jackson 版本由 Spring Boot 管理，此处仅声明约束
jackson = "2.16.0"

[libraries]
# 使用 Jackson BOM 统一管理所有 Jackson 模块版本
jackson-bom = { module = "com.fasterxml.jackson:jackson-bom", version.ref = "jackson" }
jackson-databind = { module = "com.fasterxml.jackson.core:jackson-databind" }
jackson-module-kotlin = { module = "com.fasterxml.jackson.module:jackson-module-kotlin" }
```

## 8.4 避免反模式

```toml
# ❌ 反模式 1：在 catalog 中使用动态版本
[libraries]
guava-bad = { module = "com.google.guava:guava", version = "latest.release" }
# 问题：构建不可重复

# ❌ 反模式 2：过多的 bundle
[bundles]
# 每个库一个 bundle，没有使用价值
guava-bundle = ["guava"]
commons-bundle = ["commons-lang3"]

# ❌ 反模式 3：混合不同关注点的版本管理
[versions]
# Spring Boot 版本和 npm 包版本混在一起
spring-boot = "3.2.0"
react = "18.2.0"
eslint = "8.54.0"

# ✅ 正确：使用工具插件管理各平台依赖
plugins {
    id("com.github.node-gradle.node") version "7.0.2"
}

node {
    version = "20.10.0"
}
```
