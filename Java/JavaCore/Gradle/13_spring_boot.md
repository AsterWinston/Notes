
Spring Boot Gradle Plugin 是 Spring Boot 生态中官方提供的 Gradle 构建插件，它封装了可执行 JAR 打包、依赖管理、Docker 镜像构建、AOT 编译等一系列能力。

# 1. 插件概述与版本对齐

## 1.1 插件声明与版本管理

Spring Boot Gradle Plugin 的版本应与使用的 Spring Boot 版本保持一致：

```groovy
// build.gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.0'
    id 'io.spring.dependency-management' version '1.1.5'
}

// Spring Boot 版本与 Gradle 版本兼容性
// Spring Boot 3.x → Gradle 7.5+ (推荐 8.x), JDK 17+
// Spring Boot 2.7.x → Gradle 7.x, JDK 8-17
// Spring Boot 2.5.x → Gradle 6.8+ , JDK 8-16
```

Kotlin DSL 版本：

```kotlin
// build.gradle.kts
plugins {
    id("org.springframework.boot") version "3.3.0"
    id("io.spring.dependency-management") version "1.1.5"
    kotlin("jvm") version "1.9.24"
    kotlin("plugin.spring") version "1.9.24"
}
```

版本目录（Version Catalog）管理：

```toml
# gradle/libs.versions.toml
[versions]
spring-boot = "3.3.0"
spring-dependency-management = "1.1.5"

[plugins]
spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
spring-dependency-management = { id = "io.spring.dependency-management", version.ref = "spring-dependency-management" }
```

```kotlin
// build.gradle.kts — 使用版本目录
plugins {
    alias(libs.plugins.spring.boot)
    alias(libs.plugins.spring.dependency.management)
}
```

## 1.2 插件提供的任务

Spring Boot Gradle Plugin 注册的主要任务：

| 任务名称 | 类型 | 说明 |
|---------|------|------|
| `bootJar` | `BootJar` | 打包可执行 JAR |
| `bootWar` | `BootWar` | 打包可执行 WAR |
| `bootRun` | `BootRun` | 直接运行 Spring Boot 应用 |
| `bootBuildImage` | `BootBuildImage` | 构建 OCI/Docker 镜像 |
| `bootRun` | `BootRun` | 运行应用（支持 DevTools） |
| `bootStartScripts` | `CreateStartScripts` | 生成启动脚本 |

Gradle 7.6+ / Spring Boot 3.0+ 新增 AOT 任务：

| 任务名称 | 说明 |
|---------|------|
| `processAot` | 处理应用上下文 AOT（ApplicationContext AOT） |
| `compileAot` | AOT 编译（仅 GraalVM Native） |
| `processTestAot` | 测试 AOT 处理 |
| `compileTestAot` | 测试 AOT 编译 |

## 1.3 插件扩展 DSL 全貌

在 `build.gradle` 中通过 `springBoot {}` 块配置：

```groovy
springBoot {
    // 指定主类
    mainClass = 'com.example.MyApplication'
    
    // 构建信息生成
    buildInfo {
        // 生成 META-INF/build-info.properties
        properties {
            // 添加自定义属性
            additional = [
                'java.version': System.getProperty('java.version'),
                'os.name': System.getProperty('os.name'),
                'build.time': new Date().format("yyyy-MM-dd HH:mm:ss"),
                'build.number': System.getenv('BUILD_NUMBER') ?: 'local'
            ]
        }
    }
    
    // 可执行 JAR 的分类器
    // classifier = 'boot'
}
```

# 2. bootJar — 可执行 JAR 深度解析

## 2.1 bootJar 任务配置

```groovy
bootJar {
    // 主类（覆盖 springBoot.mainClass）
    mainClass = 'com.example.Application'
    
    // 归档文件命名
    archiveFileName = "${project.name}-${project.version}.jar"
    archiveBaseName = project.name
    archiveVersion = project.version
    archiveClassifier = ''
    
    // 排除特定文件
    excludes = ['META-INF/*.SF', 'META-INF/*.DSA', 'META-INF/*.RSA']
    
    // 自定义 manifest
    manifest {
        attributes(
            'Implementation-Title': project.name,
            'Implementation-Version': project.version,
            'Built-By': System.getProperty('user.name'),
            'Built-JDK': System.getProperty('java.version'),
            'Start-Class': mainClass
        )
    }
    
    // 是否需要可执行（默认 true）
    // launchScript { properties { } }
    
    // 自定义 Classifier
    // archiveClassifier.set('boot')
    
    // 层级配置（分层 JAR）
    layered {
        enabled = true  // 默认启用（Spring Boot 3.x）
        
        // 自定义层级内容
        application {
            // 应用自己的类和资源
            intoLayer("application") {
                // 自定义内容来源
            }
        }
    }
    
    // 需要解压的依赖（当使用 requiresUnpack 时）
    requiresUnpack {
        // 可以在这里指定必须解压到文件系统的库（如某些需要文件系统访问的库）
    }
}
```

## 2.2 bootJar vs bootWar vs jar

```groovy
// 根据应用类型自动选择打包方式
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.0'
    id 'war'  // 添加 war 插件后，bootWar 替代 bootJar 成为默认
}

// 手动禁用 bootJar 启用 bootWar
tasks.named('bootJar') { enabled = false }
tasks.named('bootWar') { enabled = true }

// 同时生成可执行 JAR 和普通 JAR
tasks.named('bootJar') {
    archiveClassifier = 'boot'
}
tasks.named('jar') {
    enabled = true
    archiveClassifier = ''
}
```

WAR 部署配置：

```groovy
bootWar {
    // 对嵌入式 Servlet 容器标记为 provided
    // 以便部署到外部 Servlet 容器时排除这些依赖
}

configurations {
    // WAR 部署时排除 Tomcat
    compileOnly 'org.springframework.boot:spring-boot-starter-tomcat'
    // 但保留 Servlet API
    compileOnly 'jakarta.servlet:jakarta.servlet-api'
}
```

## 2.3 可执行 JAR 内部结构

Spring Boot 可执行 JAR 的目录布局：

```
myapp-1.0.0.jar
│
├── META-INF/
│   ├── MANIFEST.MF                # JAR 清单文件
│   └── maven/                     # Maven 元数据（如使用 Maven 坐标）
│
├── BOOT-INF/
│   ├── classes/                   # 应用自己的 .class 文件和资源
│   │   ├── com/example/
│   │   │   └── Application.class
│   │   ├── application.yml
│   │   └── static/
│   │
│   ├── lib/                       # 第三方依赖 JAR
│   │   ├── spring-boot-3.3.0.jar
│   │   ├── spring-core-6.1.8.jar
│   │   ├── jackson-databind-2.17.0.jar
│   │   └── ... (所有运行时依赖)
│   │
│   └── classpath.idx             # 类路径索引（分层 JAR 使用）
│
└── org/
    └── springframework/
        └── boot/
            └── loader/            # Spring Boot 类加载器
                ├── JarLauncher.class
                ├── WarLauncher.class
                ├── Launcher.class
                ├── PropertiesLauncher.class
                ├── ExecutableArchiveLauncher.class
                ├── LaunchedURLClassLoader.class
                └── jar/
                    ├── JarFile.class
                    ├── JarEntry.class
                    └── ... (嵌套 JAR 读取支持)
```

MANIFEST.MF 内容示例：

```
Manifest-Version: 1.0
Created-By: Gradle 8.7
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: com.example.Application
Spring-Boot-Version: 3.3.0
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Spring-Boot-Classpath-Index: BOOT-INF/classpath.idx
Spring-Boot-Layers-Index: BOOT-INF/layers.idx
```

## 2.4 Spring Boot Loader 启动机制

```
用户执行: java -jar myapp.jar
    │
    ▼
┌──────────────────────────────────────┐
│ 1. JVM 读取 META-INF/MANIFEST.MF    │
│    Main-Class: org.springframework   │
│    .boot.loader.JarLauncher          │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│ 2. JarLauncher.main()               │
│    - 创建 LaunchedURLClassLoader     │
│    - 解析 BOOT-INF/lib/ 中的 JAR     │
│    - 注册 java.protocol.handler.pkgs │
│    - 支持嵌套 JAR 访问 (jar:file:...) │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│ 3. LaunchedURLClassLoader 加载       │
│    BOOT-INF/classes/ 中的应用类     │
│    BOOT-INF/lib/ 中的依赖            │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│ 4. 反射调用 Start-Class 的 main()   │
│    (com.example.Application)         │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│ 5. SpringApplication.run()          │
│    正常 Spring Boot 启动流程        │
└──────────────────────────────────────┘
```

# 3. 分层 JAR 与 Docker 优化

## 3.1 分层 JAR 概念

分层 JAR 将应用内容按变更频率分成不同层级，最大化 Docker 层缓存利用率：

```
┌─────────────────────────────────────┐
│  Layer: dependencies                │ ← 变更频率: 极低
│  (SNAPSHOT 之外的第三方依赖)         │    缓存命中率: 很高
├─────────────────────────────────────┤
│  Layer: spring-boot-loader          │ ← 变更频率: 极低
│  (Spring Boot Loader 类)            │    缓存命中率: 很高
├─────────────────────────────────────┤
│  Layer: snapshot-dependencies       │ ← 变更频率: 中等
│  (SNAPSHOT 依赖)                    │    缓存命中率: 中等
├─────────────────────────────────────┤
│  Layer: application                 │ ← 变更频率: 很高
│  (应用自己的类和资源)               │    缓存命中率: 很低（每次构建都变）
└─────────────────────────────────────┘
```

## 3.2 分层 DSL 配置

```groovy
bootJar {
    layered {
        enabled = true
        
        // 自定义分层（Spring Boot 3.3+）
        application {
            intoLayer("application") {
                // 自定义内容映射
            }
        }
    }
}
```

自定义分层策略 — 按业务模块拆分：

```groovy
bootJar {
    layered {
        enabled = true
        
        // 使用 Groovy 闭包自定义分层逻辑
        application {
            intoLayer("infrastructure") {
                into("BOOT-INF/lib/") {
                    from configurations.runtimeClasspath
                    include {
                        it.name.contains("spring-") ||
                        it.name.contains("jackson-") ||
                        it.name.contains("slf4j-") ||
                        it.name.contains("logback-")
                    }
                }
            }
            
            intoLayer("business-libs") {
                into("BOOT-INF/lib/") {
                    from configurations.runtimeClasspath
                    include {
                        it.name.contains("mybatis-") ||
                        it.name.contains("hibernate-") ||
                        it.name.contains("redis-")
                    }
                }
            }
            
            intoLayer("internal-deps") {
                into("BOOT-INF/lib/") {
                    from configurations.runtimeClasspath
                    include {
                        it.name.startsWith("company-")
                    }
                }
            }
            
            intoLayer("application") {
                into("BOOT-INF/classes/") {
                    from sourceSets.main.output
                }
            }
        }
    }
}
```

## 3.3 分层 JAR 在 Dockerfile 中的使用

```dockerfile
# 多阶段构建 + 分层 JAR 利用
FROM eclipse-temurin:17-jre AS builder
WORKDIR /app
COPY build/libs/*.jar app.jar
# 提取分层 JAR 的各层到不同目录
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:17-jre
WORKDIR /app

# 按变更频率从低到高 COPY
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

# 这是利用 Docker 层缓存的关键：
# 每次构建时，dependencies 和 spring-boot-loader 几乎不变
# 只有 application 层会变化 → Docker 层缓存最大化

ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

分层 JAR 提取后的目录：

```
/app/
├── dependencies/
│   └── BOOT-INF/lib/  (不含 SNAPSHOT 的第三方 JAR)
├── spring-boot-loader/
│   └── org/springframework/boot/loader/
├── snapshot-dependencies/
│   └── BOOT-INF/lib/  (仅 SNAPSHOT 依赖)
└── application/
    ├── BOOT-INF/classes/
    ├── BOOT-INF/classpath.idx
    └── BOOT-INF/layers.idx
```

## 3.4 layers.idx 文件

`BOOT-INF/layers.idx` 是分层信息的索引文件：

```
- "dependencies":
  - "BOOT-INF/lib/spring-beans-6.1.8.jar"
  - "BOOT-INF/lib/spring-core-6.1.8.jar"
  - "BOOT-INF/lib/jackson-databind-2.17.0.jar"
  - ...
- "spring-boot-loader":
  - "org/"
- "snapshot-dependencies":
  - "BOOT-INF/lib/company-lib-1.0.0-SNAPSHOT.jar"
- "application":
  - "BOOT-INF/classes/"
  - "BOOT-INF/classpath.idx"
  - "BOOT-INF/layers.idx"
  - "META-INF/"
```

# 4. bootBuildImage — Cloud Native Buildpacks

## 4.1 基本构建配置

```groovy
bootBuildImage {
    // 镜像名称（默认: docker.io/library/${project.name}:${project.version}）
    imageName = "registry.company.com/${project.name}:${project.version}"
    
    // 构建器（Builder）— Paketo Buildpacks
    builder = "paketobuildpacks/builder-jammy-base:latest"
    
    // 自定义运行镜像（Run Image）
    runImage = "paketobuildpacks/run-jammy-base:latest"
    
    // 环境变量
    environment = [
        "BP_JVM_VERSION": "17",
        "BP_JVM_TYPE": "JRE",
        "BPE_APPEND_JAVA_TOOL_OPTIONS": "-XX:+UseZGC -XX:+ZGenerational",
        "BP_NATIVE_IMAGE": "false"
    ]
    
    // JVM 选项
    // bindings = []
    
    // 网络配置
    network = "host"
    
    // 是否清理缓存
    cleanCache = false
    
    // 是否发布到镜像仓库
    publish = false
    
    // 构建时携带的卷
    volumes = [
        "/tmp/volume1:/mnt/volume1"
    ]
}
```

## 4.2 Paketo Buildpacks 高级配置

```groovy
bootBuildImage {
    imageName = "registry.company.com/${project.name}:${project.version}"
    builder = "paketobuildpacks/builder-jammy-base:latest"
    
    // 详细的 Paketo 环境变量
    environment = [
        // JVM 配置
        "BP_JVM_VERSION": "17",
        "BP_JVM_TYPE": "JRE",                       // JRE 而非 JDK（减小镜像）
        
        // 内存计算
        "BPL_JVM_HEAD_ROOM": "2",
        "BPL_JVM_LOADED_CLASS_COUNT": "35",
        "BPL_JVM_THREAD_COUNT": "50",
        
        // Java 工具选项
        "BPE_DELIM_JAVA_TOOL_OPTIONS": " ",
        "BPE_APPEND_JAVA_TOOL_OPTIONS": "-Dspring.profiles.active=prod",
        "JAVA_TOOL_OPTIONS": "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0",
        
        // Spring Boot 特定
        "BP_SPRING_CLOUD_BINDINGS_DISABLED": "false",
        
        // 构建包配置
        "BP_IMAGE_LABELS": "maintainer=team@company.com",
        "BP_OCI_AUTHORS": "Platform Team",
        "BP_OCI_TITLE": "${project.name}",
        "BP_OCI_DESCRIPTION": "${project.description}",
        
        // CA 证书（企业环境）
        // "BP_EMBED_CERTS": "true"
    ]
    
    // 绑定配置（连接数据库等服务）
    bindings = [
        "/path/to/bindings/ca-certificates:/platform/bindings/ca-certificates"
    ]
    
    // 发布配置
    publish = true
    docker {
        publishRegistry {
            // 镜像仓库认证
            username = System.getenv('REGISTRY_USERNAME') ?: project.findProperty('registryUsername')
            password = System.getenv('REGISTRY_PASSWORD') ?: project.findProperty('registryPassword')
            url = "https://registry.company.com"
            email = "team@company.com"
        }
        
        // 构建器使用的 Docker 守护进程
        // host = "tcp://docker-host:2375"
        // tlsVerify = true
        // certPath = "/path/to/certs"
        
        // Dockerfile 方式（替代 Buildpacks）
        // builderRegistry { ... }
    }
}
```

## 4.3 构建包原理

```
┌──────────────────────────────────────────────────┐
│              bootBuildImage 任务                  │
├──────────────────────────────────────────────────┤
│  1. 确定构建器 (paketobuildpacks/builder-*)      │
│                                                     │
│  2. 检测 (Detect)                                   │
│     ├── 分析源码和依赖                              │
│     ├── 确定需要哪些 Buildpack                     │
│     └── Paketo 检测流程:                           │
│         ├── 检测 build.gradle / pom.xml → Java    │
│         ├── 检测 application.yml → Spring Boot    │
│         ├── 检测 *.jar → 可执行 JAR               │
│         └── 生成构建计划 (Build Plan)              │
│                                                     │
│  3. 构建 (Build)                                    │
│     ├── 安装 JDK (BellSoft Liberica)               │
│     ├── 执行 ./gradlew bootJar (或类似)           │
│     ├── 创建分层 JAR                               │
│     ├── 生成 CA 证书存储                           │
│     └── 生成 OCI 层 blobs                          │
│                                                     │
│  4. 导出 (Export)                                   │
│     ├── 创建最终镜像                                │
│     └── 推送到 registry（如果 publish=true）       │
└──────────────────────────────────────────────────┘
```

## 4.4 自定义构建器

```groovy
bootBuildImage {
    // 使用自定义构建器
    builder = "registry.company.com/builder/custom-builder:latest"
    
    // 覆盖运行镜像
    runImage = "registry.company.com/run/java-base:latest"
    
    // 添加额外的构建包
    // buildpacks = [
    //     "gcr.io/paketo-buildpacks/java-native-image",
    //     "registry.company.com/buildpacks/custom-metrics"
    // ]
    
    // 绑定 SSL 证书
    bindings = [
        "${System.getProperty('user.home')}/.docker/certs:/platform/bindings/ca-certificates"
    ]
}
```

## 4.5 Native Image 构建

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.0'
    id 'org.graalvm.buildtools.native' version '0.10.2'
}

graalvmNative {
    binaries {
        main {
            // 主类
            mainClass = 'com.example.Application'
            
            // 构建参数
            buildArgs.add('--verbose')
            buildArgs.add('-H:+ReportExceptionStackTraces')
            
            // 资源包含
            // resources.autodetect()
            
            // 调试信息
            debug = false
            
            // 快速构建模式
            quickBuild = false
            
            // 允许不完整类路径（某些反射场景）
            // fallback = false
        }
    }
}

bootBuildImage {
    // 构建 GraalVM Native Image
    builder = "paketobuildpacks/builder-jammy-tiny:latest"
    environment = [
        "BP_NATIVE_IMAGE": "true"
    ]
}
```

# 5. Spring AOT 编译

## 5.1 AOT 引擎与 processAot

AOT（Ahead-of-Time）处理用于生成 GraalVM Native Image 所需的反射配置、代理配置和资源提示：

```groovy
// AOT 任务自动注册（Spring Boot 3.0+）

// processAot — 处理 ApplicationContext AOT 优化
tasks.named('processAot') {
    // 主类
    mainClass = 'com.example.Application'
    
    // 应用参数
    args = ['--spring.profiles.active=production']
    
    // JVM 参数
    jvmArgs = ['-Xmx2g']
    
    // 系统属性
    systemProperty 'spring.aot.enabled', 'true'
}

// compileAot — 将 AOT 源码编译为 class
tasks.named('compileAot') {
    // 编译器参数
    // options.compilerArgs << '-parameters'
}
```

AOT 生成的源码位于：

```
build/generated/aotSources/
├── com/example/
│   └── Application__ApplicationContextInitializer.java
├── META-INF/native-image/
│   └── com.example/
│       ├── reflect-config.json      # 反射配置
│       ├── proxy-config.json        # 代理配置
│       ├── resource-config.json     # 资源配置
│       ├── serialization-config.json
│       └── jni-config.json
└── ...
```

## 5.2 AOT 工作流程

```
执行 ./gradlew processAot
    │
    ▼
┌──────────────────────────────────────────┐
│ 1. Spring AOT Engine 启动                │
│    - 构建 ApplicationContext（不启动）    │
│    - 执行 Bean 定义分析                  │
│    - 评估条件注解 (@ConditionalOnXxx)    │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│ 2. 生成 AOT 源码                         │
│    - ApplicationContextInitializer       │
│    - Bean 定义替换为静态代码             │
│    - 代理类配置                          │
│    - 资源提示                             │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│ 3. compileAot — 编译生成的源码           │
│    - 输出到 build/generated/aotClasses   │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│ 4. 打包到 bootJar                        │
│    - AOT 生成的 class 替代运行时反射     │
│    - native-image 配置随 JAR 携带        │
└──────────────────────────────────────────┘
```

# 6. bootRun 任务

## 6.1 开发时运行配置

```groovy
bootRun {
    // 主类
    mainClass = 'com.example.Application'
    
    // 程序参数
    args = ['--spring.profiles.active=dev', '--server.port=9090']
    
    // JVM 参数
    jvmArgs = [
        '-Xmx512m',
        '-Xms256m',
        '-XX:+UseG1GC',
        '-Dspring.devtools.restart.enabled=true',
        '-Duser.timezone=UTC'
    ]
    
    // 系统属性
    systemProperty 'spring.output.ansi.enabled', 'ALWAYS'
    systemProperty 'debug', 'true'
    
    // 环境变量
    environment 'DATABASE_URL', 'jdbc:postgresql://localhost:5432/devdb'
    environment 'API_KEY', 'dev-key'
    
    // 工作目录
    workingDir = file("$rootDir/run")
    
    // 源码资源（支持热重载）
    sourceResources sourceSets.main
    
    // 优化 JVM 启动参数
    optimizedLaunch = true // 跳过 JarLauncher，直接运行 main 方法
}
```

## 6.2 Spring Boot DevTools 集成

```groovy
dependencies {
    // DevTools
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    
    // 生产环境排除 DevTools
    // 注意：devtools 打包时会自动禁用
}

// 确保 DevTools 仅开发时可用
configurations {
    developmentOnly
    runtimeClasspath {
        extendsFrom developmentOnly
    }
}

bootRun {
    // DevTools 重启配置
    systemProperty 'spring.devtools.restart.enabled', 'true'
    systemProperty 'spring.devtools.livereload.enabled', 'true'
    systemProperty 'spring.devtools.restart.additional-paths', 'src/main/resources'
    
    // 排除不需要触发重启的路径
    // spring.devtools.restart.exclude=static/**,templates/**,public/**
}
```

## 6.3 bootRun 与 sourceResources

```groovy
bootRun {
    // 配置源码资源（使模板、静态资源等可被 DevTools 热重载）
    sourceResources sourceSets.main {
        // 可以在此过滤
        exclude '**/*.java'
    }
    
    // 等价于将源码目录加入运行时 classpath
    // 这样修改静态资源不需要重启
}
```

# 7. 依赖管理插件

## 7.1 io.spring.dependency-management 插件

此插件独立于 Spring Boot Gradle Plugin，提供类似 Maven BOM 的依赖版本管理：

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.0'
    id 'io.spring.dependency-management' version '1.1.5'
}

// 无需手动指定 Spring 相关依赖的版本
dependencies {
    // 以下依赖版本由 dependency-management 插件从 BOM 导入管理
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    
    // 第三方库被 BOM 管理的版本也自动继承
    implementation 'com.fasterxml.jackson.core:jackson-databind'
    implementation 'io.micrometer:micrometer-core'
}
```

## 7.2 BOM 导入与版本覆盖

```groovy
dependencyManagement {
    imports {
        // 导入 Spring Boot BOM（自动执行）
        // mavenBom org.springframework.boot.gradle.plugin.SpringBootPlugin.BOM_COORDINATES
        
        // 导入额外的 BOM
        mavenBom 'org.springframework.cloud:spring-cloud-dependencies:2023.0.2'
        mavenBom 'com.amazonaws:aws-java-sdk-bom:1.12.710'
        mavenBom 'org.testcontainers:testcontainers-bom:1.19.7'
    }
    
    // 覆盖管理的版本
    dependencies {
        // 强制指定某个依赖的版本
        dependency 'com.fasterxml.jackson.core:jackson-databind:2.17.1'
        dependency 'io.netty:netty-codec-http:4.1.109.Final'
        dependencySet(group: 'io.micrometer', version: '1.13.0') {
            entry 'micrometer-core'
            entry 'micrometer-registry-prometheus'
        }
    }
}

// 优先级: 直接依赖版本 > dependencyManagement 覆盖 > BOM 导入版本
```

## 7.3 生成 POM 依赖管理（Maven 互操作）

```groovy
plugins {
    id 'java-platform'
    id 'maven-publish'
}

// 将 BOM 发布为 Maven POM
dependencies {
    constraints {
        api 'org.springframework.boot:spring-boot-starter-web:3.3.0'
        api 'com.fasterxml.jackson.core:jackson-databind:2.17.0'
        api 'com.google.guava:guava:33.1.0-jre'
    }
}

publishing {
    publications {
        mavenPlatform(MavenPublication) {
            from components.javaPlatform
        }
    }
}
```

## 7.4 查看依赖版本来源

```bash
# 查看所有依赖及其版本来源
./gradlew dependencies --configuration runtimeClasspath

# 查看特定配置的依赖
./gradlew dependencies --configuration compileClasspath

# 查看依赖冲突解决结果
./gradlew dependencyInsight --dependency jackson-databind --configuration runtimeClasspath

# 生成依赖锁文件
./gradlew dependencies --write-locks
```

# 8. 多模块 Spring Boot 项目

## 8.1 多模块项目结构

```
spring-boot-multi-module/
├── settings.gradle.kts
├── build.gradle.kts
├── gradle/
│   └── libs.versions.toml
├── application/                    # 启动模块
│   └── build.gradle.kts
├── domain/                         # 领域模块
│   └── build.gradle.kts
├── infrastructure/                 # 基础设施模块
│   └── build.gradle.kts
└── api/                            # API 接口模块
    └── build.gradle.kts
```

## 8.2 根项目配置

```kotlin
// settings.gradle.kts
rootProject.name = "myapp"
include("application", "domain", "infrastructure", "api")
```

```kotlin
// build.gradle.kts（根项目）
plugins {
    id("org.springframework.boot") version "3.3.0" apply false
    id("io.spring.dependency-management") version "1.1.5" apply false
    kotlin("jvm") version "1.9.24" apply false
    kotlin("plugin.spring") version "1.9.24" apply false
}

allprojects {
    group = "com.example"
    version = "1.0.0-SNAPSHOT"
    
    repositories {
        mavenCentral()
    }
}

subprojects {
    apply(plugin = "java")
    
    java {
        toolchain {
            languageVersion.set(JavaLanguageVersion.of(17))
        }
    }
    
    tasks.withType<Test> {
        useJUnitPlatform()
    }
}
```

## 8.3 各模块配置

```kotlin
// domain/build.gradle.kts — 纯领域模块（不需要 Spring Boot 插件）
plugins {
    java
}

dependencies {
    implementation("jakarta.validation:jakarta.validation-api:3.0.2")
}
```

```kotlin
// infrastructure/build.gradle.kts
plugins {
    java
    id("org.springframework.boot")  // 需要插件支持可执行 JAR
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":domain"))
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql")
}

// 禁用自动创建 bootJar（基础设施模块不独立运行）
tasks.bootJar {
    enabled = false  // 生成为普通 JAR
}
tasks.jar {
    enabled = true
    archiveClassifier.set("")
}
```

```kotlin
// application/build.gradle.kts — 启动模块
plugins {
    java
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":domain"))
    implementation(project(":infrastructure"))
    implementation(project(":api"))
    implementation("org.springframework.boot:spring-boot-starter-web")
}

springBoot {
    mainClass.set("com.example.Application")
}

tasks.bootJar {
    archiveFileName.set("app.jar")
}
```

## 8.4 共享配置与版本管理

使用 Convention Plugin 统一管理：

```kotlin
// buildSrc/src/main/kotlin/spring-module.gradle.kts
plugins {
    java
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(17))
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}

dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

```kotlin
// 子模块中使用
// infrastructure/build.gradle.kts
plugins {
    id("spring-module")
}
```

# 9. Thin JAR 策略

## 9.1 Thin Launcher 概念

对于频繁部署的场景，可使用 Thin JAR 将依赖与业务代码分离：

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.0'
    id 'io.spring.dependency-management' version '1.1.5'
}

bootJar {
    // 默认已是 Fat JAR
    // Thin JAR 需要额外的插件或配置
    
    // 方式一：使用 Spring Boot Thin Launcher
    // requiresUnpack { ... }
}
```

使用 `spring-boot-thin-launcher` 库实现 Thin JAR：

```groovy
dependencies {
    implementation 'org.springframework.boot.experimental:spring-boot-thin-launcher:1.0.31.RELEASE'
}

// 将依赖外部化到单独目录
tasks.register('copyDependencies', Copy) {
    from configurations.runtimeClasspath
    into layout.buildDirectory.dir('dependencies')
}

bootJar {
    // 排除依赖 JAR（由 Thin Launcher 在运行时下载）
    archiveClassifier = 'thin'
    
    // 注意：生产部署需要依赖仓库可用
}
```

## 9.2 依赖外部化的 Docker 策略

```dockerfile
# 将依赖与业务代码分离
FROM eclipse-temurin:17-jre AS dependencies
WORKDIR /app
COPY build/dependencies/ dependencies/
# 依赖层（很少变化）

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=dependencies /app/dependencies/ /app/lib/
COPY build/libs/*-thin.jar /app/app.jar
ENTRYPOINT ["java", "-Dloader.path=lib/", "-jar", "/app/app.jar"]
```

# 10. 测试配置

## 10.1 测试切片配置

```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test' {
        // 排除 JUnit 4 Vintage 引擎（如果只用 JUnit 5）
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
    
    // 测试切片专用依赖
    testImplementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    testImplementation 'com.h2database:h2'  // 测试数据库
    testImplementation 'org.testcontainers:postgresql'
    
    // Web 测试切片
    testImplementation 'org.springframework.boot:spring-boot-starter-webflux'
}
```

## 10.2 测试属性配置

```groovy
test {
    useJUnitPlatform()
    
    systemProperty 'spring.profiles.active', 'test'
    systemProperty 'spring.jpa.hibernate.ddl-auto', 'create-drop'
    
    testLogging {
        events "passed", "skipped", "failed"
        showStandardStreams = true
    }
}
```

## 10.3 Spring Boot 测试注解与 Gradle 集成

```java
// 使用 @SpringBootTest（启动完整上下文）
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserControllerTest {
    @LocalServerPort
    private int port;
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void testGetUser() {
        // ...
    }
}

// 使用 @WebMvcTest（仅 Web 层）
@WebMvcTest(UserController.class)
class UserControllerWebMvcTest {
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void testGetUser() throws Exception {
        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}

// 使用 @DataJpaTest（仅持久层）
@DataJpaTest
class UserRepositoryTest {
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testFindByName() {
        // ...
    }
}
```

# 11. 常见配置总结

## 11.1 完整的生产级 build.gradle

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.0'
    id 'io.spring.dependency-management' version '1.1.5'
    id 'jacoco'
}

group = 'com.example'
version = '1.0.0-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    
    compileOnly 'org.projectlombok:lombok'
    runtimeOnly 'com.h2database:h2'
    runtimeOnly 'org.postgresql:postgresql'
    annotationProcessor 'org.projectlombok:lombok'
    
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.testcontainers:postgresql'
}

springBoot {
    mainClass = 'com.example.Application'
    buildInfo {
        properties {
            additional = [
                'time': new Date().format("yyyy-MM-dd'T'HH:mm:ss'Z'", TimeZone.getTimeZone("UTC"))
            ]
        }
    }
}

bootJar {
    archiveFileName = "${project.name}.jar"
    layered {
        enabled = true
    }
}

bootBuildImage {
    imageName = "registry.company.com/${project.name}:${project.version}"
    environment = [
        "BP_JVM_VERSION": "17"
    ]
    publish = false
}

tasks.named('test') {
    useJUnitPlatform()
    maxHeapSize = '1g'
}

tasks.jacocoTestReport {
    dependsOn test
    reports {
        xml.required = true
        html.required = true
    }
}
```

## 11.2 Spring Boot 与 Gradle 版本兼容矩阵

| Spring Boot | Gradle | JDK | Kotlin | 说明 |
|-------------|--------|-----|--------|------|
| 3.3.x | 8.5 - 8.8 | 17 - 22 | 1.9.x | 当前最新 |
| 3.2.x | 8.4 - 8.7 | 17 - 21 | 1.9.x | |
| 3.1.x | 8.0 - 8.4 | 17 - 20 | 1.8.x | |
| 3.0.x | 7.5 - 8.2 | 17 - 19 | 1.7.x - 1.8.x | Jakarta EE 9+ |
| 2.7.x | 7.x | 8 - 18 | 1.6.x - 1.7.x | 最后一个 2.x |
| 2.5.x | 6.8 - 7.x | 8 - 16 | 1.4.x - 1.5.x | |
