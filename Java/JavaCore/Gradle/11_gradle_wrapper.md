
Gradle Wrapper（包装器）是 Gradle 官方强烈推荐的项目构建入口方式。它确保项目在任何环境中都能使用统一、正确版本的 Gradle 进行构建，无需预先安装 Gradle。

# 1. Wrapper 概述与核心价值

## 1.1 什么是 Gradle Wrapper

Gradle Wrapper 是一组脚本和 JAR 文件，作为项目的一部分提交到版本控制系统。当开发者克隆项目后，只需运行 `gradlew`（Linux/macOS）或 `gradlew.bat`（Windows），Wrapper 会自动下载并使用项目所声明的 Gradle 版本进行构建。

核心价值：
- **零预装依赖**：开发者无需手动安装 Gradle
- **版本一致性**：所有 CI 环境和团队成员使用完全相同的 Gradle 版本
- **可复现构建**：消除"在我机器上能跑"的版本差异问题
- **平滑升级**：升级 Gradle 版本只需修改一个配置文件

## 1.2 Wrapper 文件结构

一个标准 Wrapper 由以下文件组成：

```
project-root/
├── gradlew                    # Unix shell 启动脚本
├── gradlew.bat                # Windows 批处理启动脚本
└── gradle/
    └── wrapper/
        ├── gradle-wrapper.jar          # Wrapper 核心 JAR
        └── gradle-wrapper.properties   # Wrapper 配置文件
```

每个文件的作用：

| 文件 | 作用 | 是否应提交到 VCS |
|------|------|:---:|
| `gradlew` | Unix 启动脚本，执行引导逻辑并启动 Gradle | ✓ |
| `gradlew.bat` | Windows 启动脚本，功能等价于 gradlew | ✓ |
| `gradle-wrapper.jar` | 包含下载、验证、启动 Gradle 发行版的引导代码 | ✓ |
| `gradle-wrapper.properties` | 声明 Gradle 版本、分发类型、下载地址等配置 | ✓ |
| `.gradle/` | 下载的 Gradle 发行版缓存目录（用户 ~/.gradle/ 下） | ✗ |

# 2. Wrapper 工作机制详解

## 2.1 gradlew 脚本执行流程

`gradlew` 是一个 Shell 脚本，其核心执行流程如下：

```
用户执行 ./gradlew build
        │
        ▼
┌──────────────────────────────┐
│ 1. 确定 JAVA_HOME / JRE     │
│    查找可用的 Java 运行时    │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ 2. 设置 JVM 参数            │
│    DEFAULT_JVM_OPTS          │
│    JAVA_OPTS / GRADLE_OPTS   │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ 3. 定位 gradle-wrapper.jar  │
│    从脚本所在目录的相对路径  │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ 4. 执行 wrapper jar         │
│    java -jar wrapper.jar     │
│    传入用户的任务和参数      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ 5. Wrapper 检查本地缓存      │
│    读取 wrapper.properties   │
│    计算 distributionUrl      │
│    检查 ~/.gradle/wrapper/dists/ │
└──────────────┬───────────────┘
               │
       ┌───────┴────────┐
       │ 本地有缓存吗？  │
       └──┬─────────┬───┘
     有   │         │  没有
          │         │
          ▼         ▼
   ┌──────────┐  ┌──────────────────────┐
   │直接使用   │  │ 从 distributionUrl   │
   │缓存版本   │  │ 下载 Gradle 发行版    │
   └────┬─────┘  │ 验证 SHA-256 校验和   │
        │        │ 解压到缓存目录        │
        │        └──────────┬───────────┘
        │                   │
        └────────┬──────────┘
                 │
                 ▼
      ┌──────────────────────┐
      │ 6. 启动 Gradle 守护进程 │
      │    执行用户指定的任务   │
      └──────────────────────┘
```

## 2.2 gradle-wrapper.jar 内部机制

`gradle-wrapper.jar` 是一个可执行 JAR，其 MANIFEST.MF 中指定了主类为 `org.gradle.wrapper.GradleWrapperMain`。核心类结构：

```
org.gradle.wrapper
├── GradleWrapperMain          # 入口点，解析参数、协调流程
├── Install                    # 负责下载和解压 Gradle 发行版
├── Download                   # 网络下载实现（支持 HTTP/HTTPS）
├── PathAssembler              # 构建本地缓存路径
├── WrapperConfiguration       # 解析 wrapper.properties
├── Logger                     # 日志输出
└── BootstrapMainStarter       # 启动真正的 Gradle 进程
```

关键源码逻辑（简化说明）：

```java
// GradleWrapperMain.main() 核心流程
public static void main(String[] args) throws Exception {
    // 1. 解析 wrapper.properties
    WrapperConfiguration config = WrapperConfiguration.forWrapperProperties();
    
    // 2. 创建 Install 实例
    Install install = new Install(new Download(), new PathAssembler());
    
    // 3. 下载并解压（如果未缓存）
    File gradleHome = install.createDist(config);
    
    // 4. 构建启动命令
    // gradleHome/bin/gradle + 用户传入的任务和参数
    // 注意：这里会切换到真正的 Gradle 进程，不是一直跑在 wrapper 里
    new BootstrapMainStarter().start(args, gradleHome);
}
```

`wrapper.properties` 可以在 classpath 中找到，也可以在 `gradle/wrapper/` 目录中找到。查找优先级：
1. 系统属性 `gradle.wrapper.properties` 指定的路径
2. `gradle/wrapper/gradle-wrapper.properties`
3. classpath 根目录下的 `gradle-wrapper.properties`

## 2.3 本地缓存目录结构

Gradle 发行版缓存位于 `~/.gradle/wrapper/dists/` 下：

```
~/.gradle/wrapper/dists/
└── gradle-8.7-bin/
    └── bhs2wmbdwecv87pi65oeuq5iu/     # 随机目录名（基于 URL 哈希）
        ├── gradle-8.7-bin.zip         # 下载的原始 ZIP
        ├── gradle-8.7-bin.zip.lck     # 下载锁文件
        ├── gradle-8.7-bin.zip.ok      # 下载完成标记
        └── gradle-8.7/                # 解压后的 Gradle 目录
            ├── bin/
            ├── lib/
            └── ...
```

缓存的随机目录名由 distribution URL 的 MD5 哈希生成，避免不同版本的发行版发生冲突。

# 3. gradle-wrapper.properties 配置详解

## 3.1 核心配置项

```properties
# distributionUrl — 声明 Gradle 发行版的下载地址
distributionUrl=https\://services.gradle.org/distributions/gradle-8.7-bin.zip

# distributionBase — 发行版解压后的父目录
# 默认 GRADLE_USER_HOME (通常为 ~/.gradle)
distributionBase=GRADLE_USER_HOME

# distributionPath — 相对于 distributionBase 的路径
# 默认 wrapper/dists
distributionPath=wrapper/dists

# zipStoreBase — ZIP 文件下载存放的父目录
zipStoreBase=GRADLE_USER_HOME

# zipStorePath — ZIP 文件相对于 zipStoreBase 的路径
zipStorePath=wrapper/dists

# networkTimeout — 下载超时时间（毫秒）
# Gradle 7.6+ 支持
networkTimeout=10000

# validateDistributionUrl — 是否校验 distributionUrl 的格式（Gradle 8.11+）
validateDistributionUrl=true
```

## 3.2 distributionUrl 详细说明

`distributionUrl` 的格式：

```
https://services.gradle.org/distributions/gradle-{version}-{type}.zip
```

版本号格式：
- 完整版本：`gradle-8.7`
- 里程碑版本：`gradle-8.8-milestone-1`
- RC 版本：`gradle-8.8-rc-1`
- 快照版本：`gradle-8.9-20240501000000+0000`

## 3.3 分发类型：bin vs all

| 类型 | 包含内容 | 大小（约） | 适用场景 |
|------|---------|-----------|---------|
| `bin` | Gradle 运行时、核心 API | ~130 MB | CI/CD 环境、生产构建 |
| `all` | bin + 源代码 + Javadoc | ~230 MB | 开发环境（IDE 代码补全） |

```properties
# bin 类型（默认，最小化）
distributionUrl=https\://services.gradle.org/distributions/gradle-8.7-bin.zip

# all 类型（含源码，IDE 友好）
distributionUrl=https\://services.gradle.org/distributions/gradle-8.7-all.zip
```

选择建议：
- 本地开发使用 `all`，以便 IDE 能显示 Gradle API 的源码和 Javadoc
- CI/CD 使用 `bin`，减少下载时间和存储占用
- 通过环境变量或 CI 配置动态切换是高级用法

## 3.4 自定义发行版路径

企业环境常需要托管 Gradle 发行版：

```properties
# 企业 Nexus / Artifactory 代理
distributionUrl=https\://nexus.company.com/repository/gradle-distributions/gradle-8.7-bin.zip

# 本地文件系统（网络隔离环境）
# 注意：Windows 路径中的反斜杠和冒号需要转义
distributionUrl=file\:/mnt/shared/gradle/gradle-8.7-bin.zip
```

# 4. 生成和配置 Wrapper

## 4.1 使用 gradle wrapper 任务

在已有 Gradle 项目的 `build.gradle` 中配置后运行：

```groovy
// build.gradle
wrapper {
    // 指定 Gradle 版本
    gradleVersion = '8.7'
    
    // 指定分发类型
    distributionType = Wrapper.DistributionType.BIN
    
    // 自定义 distributionUrl（优先级高于 gradleVersion）
    // distributionUrl = 'https://services.gradle.org/distributions/gradle-8.7-bin.zip'
    
    // 自定义下载目标路径
    distributionPath = 'wrapper/dists'
    
    // 自定义 archive 路径
    archivePath = 'wrapper/dists'
    
    // 网络超时（Gradle 7.6+）
    networkTimeout = 10000
}
```

运行命令：

```bash
# 使用默认配置生成 Wrapper
gradle wrapper

# 在命令行指定版本（覆盖 build.gradle 中的配置）
gradle wrapper --gradle-version 8.7

# 指定分发类型
gradle wrapper --gradle-version 8.7 --distribution-type all

# 指定下载 URL
gradle wrapper --gradle-distribution-url https://services.gradle.org/distributions/gradle-8.7-bin.zip
```

Kotlin DSL 等效配置：

```kotlin
// build.gradle.kts
tasks.wrapper {
    gradleVersion = "8.7"
    distributionType = Wrapper.DistributionType.BIN
    networkTimeout = 10000
}
```

## 4.2 从零初始化 Wrapper

在没有 Wrapper 的项目中，有两种方式初始化：

方式一：使用已安装的 Gradle 生成

```bash
# 在项目根目录执行（需要本地安装了 Gradle）
gradle init
# 选择项目类型后会自动生成 Wrapper 文件
```

方式二：使用 Gradle 的 init 脚本

```bash
gradle wrapper --gradle-version 8.7 --project-dir /path/to/project
```

方式三：手动从官方 Wrapper JAR 引导

```bash
# 1. 手动创建 gradle/wrapper/ 目录
mkdir -p gradle/wrapper

# 2. 下载 gradle-wrapper.jar（从任意 Gradle 项目的 gradle/wrapper/ 复制）
# 3. 创建 gradle-wrapper.properties
echo "distributionUrl=https\://services.gradle.org/distributions/gradle-8.7-bin.zip" > gradle/wrapper/gradle-wrapper.properties

# 4. 复制 gradlew 和 gradlew.bat 脚本
```

# 5. 升级 Wrapper 版本

## 5.1 使用 wrapper 任务升级

```bash
# 升级到指定版本
./gradlew wrapper --gradle-version 8.8

# 升级到最新版
./gradlew wrapper --gradle-version latest
```

执行后 `gradle-wrapper.properties` 会自动更新。

## 5.2 直接修改 properties 文件

```properties
# 将 version 改为目标版本即可
distributionUrl=https\://services.gradle.org/distributions/gradle-8.8-bin.zip
```

修改后再次执行 `./gradlew` 命令会自动下载新版本。

## 5.3 跨大版本升级注意事项

| 升级路径 | 关键 Breaking Changes |
|---------|---------------------|
| 7.x → 8.0 | Kotlin DSL 版本升级、配置缓存重写、`buildDir` 废弃 |
| 6.x → 7.0 | `compile`/`runtime` 配置移除、Gradle Module Metadata 默认启用 |
| 5.x → 6.0 | `maven` 插件移除、API/Implementation 分离强制化 |

升级前建议使用 Gradle 官方升级指南，并使用 `gradle help --warning-mode all` 检查废弃 API 使用情况。

# 6. Wrapper 校验与安全

## 6.1 SHA-256 校验和验证

Gradle 官方在 `services.gradle.org` 上为每个发行版提供 SHA-256 校验和。Wrapper 下载完成后会自动计算 ZIP 文件的 SHA-256 并与官方校验和对比。

校验流程：
1. Wrapper 从 `{distributionUrl}.sha256` 获取官方校验和
2. 下载 ZIP 文件后计算本地 SHA-256
3. 比对校验和不一致则抛出错误并删除损坏的下载

```bash
# 手动校验 Gradle 发行版
curl -O https://services.gradle.org/distributions/gradle-8.7-bin.zip
curl -O https://services.gradle.org/distributions/gradle-8.7-bin.zip.sha256

# Linux/macOS
shasum -a 256 -c gradle-8.7-bin.zip.sha256

# Windows PowerShell
(Get-FileHash gradle-8.7-bin.zip -Algorithm SHA256).Hash.ToLower() -eq (Get-Content gradle-8.7-bin.zip.sha256).Split()[0]
```

## 6.2 Wrapper JAR 校验

Gradle Wrapper JAR 本身也有官方校验和。建议使用 `gradle/wrapper/gradle-wrapper.jar.sha256` 文件存储校验和：

```bash
# 获取官方 gradle-wrapper.jar 的 SHA-256
curl -L https://github.com/gradle/gradle/raw/v8.7.0/gradle/wrapper/gradle-wrapper.jar.sha256

# 验证本地文件
shasum -a 256 -c gradle/wrapper/gradle-wrapper.jar.sha256
```

## 6.3 Wrapper 验证任务（Gradle 8.4+）

Gradle 8.4 引入了 `wrapper` 任务的验证功能：

```bash
# 验证 wrapper jar 的完整性
./gradlew wrapper --validate

# 更新 wrapper jar 到官方最新版本
./gradlew wrapper --update
```

# 7. 网络配置

## 7.1 代理配置

方式一：gradle.properties 配置

```properties
# gradle.properties（项目级或全局 ~/.gradle/gradle.properties）
systemProp.http.proxyHost=proxy.company.com
systemProp.http.proxyPort=8080
systemProp.http.proxyUser=username
systemProp.http.proxyPassword=password

systemProp.https.proxyHost=proxy.company.com
systemProp.https.proxyPort=8443
systemProp.https.proxyUser=username
systemProp.https.proxyPassword=password

# 不使用代理的地址
systemProp.http.nonProxyHosts=*.company.com|localhost
```

方式二：gradlew 环境变量

```bash
# Unix
export GRADLE_OPTS="-Dhttp.proxyHost=proxy.company.com -Dhttp.proxyPort=8080"
./gradlew build

# Windows PowerShell
$env:GRADLE_OPTS = "-Dhttp.proxyHost=proxy.company.com -Dhttp.proxyPort=8080"
./gradlew build
```

方式三：gradlew 内设置代理

打开 `gradlew` 脚本，在 `DEFAULT_JVM_OPTS` 中添加代理参数：

```bash
DEFAULT_JVM_OPTS='"-Dhttp.proxyHost=proxy.company.com" "-Dhttp.proxyPort=8080"'
```

## 7.2 timeout 配置

```properties
# gradle/wrapper/gradle-wrapper.properties
# Gradle 7.6+ 支持
networkTimeout=30000  # 30 秒超时
```

## 7.3 认证配置

对于需要认证的企业仓库：

```properties
# gradle.properties
systemProp.gradle.wrapperUser=myuser
systemProp.gradle.wrapperPassword=mypassword
```

这些属性会被 `Download` 类读取，用于 HTTP Basic Authentication。

# 8. CI/CD 最佳实践

## 8.1 Wrapper 验证步骤

在 CI 流水线中增加 Wrapper 完整性检查：

```bash
# GitHub Actions 示例
- name: Verify Gradle Wrapper
  run: |
    # 校验 gradle-wrapper.jar 的 SHA-256
    cd gradle/wrapper
    curl -L -o gradle-wrapper.jar.sha256 \
      https://raw.githubusercontent.com/gradle/gradle/v8.7.0/gradle/wrapper/gradle-wrapper.jar.sha256
    echo "  gradle-wrapper.jar" >> gradle-wrapper.jar.sha256
    shasum -a 256 -c gradle-wrapper.jar.sha256
```

```yaml
# 完整的 GitHub Actions CI 配置
name: Build
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Validate Gradle Wrapper
        uses: gradle/wrapper-validation-action@v2
      
      - name: Build with Gradle
        run: ./gradlew build
```

## 8.2 缓存策略

```yaml
# Jenkins Pipeline 示例
pipeline {
    agent any
    environment {
        GRADLE_USER_HOME = "${WORKSPACE}/.gradle"
    }
    stages {
        stage('Build') {
            steps {
                // 缓存 .gradle 目录可加速后续构建
                sh './gradlew build --build-cache'
            }
        }
    }
    post {
        always {
            // 保留 wrapper 下载的 Gradle 发行版缓存
            stash name: 'gradle-cache', 
                  includes: '.gradle/wrapper/dists/**, .gradle/caches/**'
        }
    }
}
```

GitLab CI 缓存示例：

```yaml
# .gitlab-ci.yml
build:
  stage: build
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - .gradle/wrapper/
      - .gradle/caches/
  script:
    - ./gradlew build
```

## 8.3 企业仓库场景

```properties
# gradle/wrapper/gradle-wrapper.properties — 指向企业 Nexus
distributionUrl=https\://nexus.internal.com/repository/gradle-dist/org/gradle/gradle-8.7-bin.zip
```

配合 init 脚本实现统一的仓库配置：

```groovy
// $GRADLE_USER_HOME/init.d/enterprise-repos.gradle
allprojects {
    repositories {
        maven { url 'https://nexus.internal.com/repository/maven-public/' }
    }
    buildscript {
        repositories {
            maven { url 'https://nexus.internal.com/repository/maven-public/' }
        }
    }
}
```

## 8.4 Docker 镜像中的最佳实践

```dockerfile
# 多阶段构建，利用 Docker 缓存
FROM eclipse-temurin:17-jdk AS builder
WORKDIR /app
COPY gradlew gradlew.bat settings.gradle.kts build.gradle.kts ./
COPY gradle/ gradle/
# 先下载依赖（利用 Docker 层缓存）
RUN ./gradlew dependencies --no-daemon
COPY src/ src/
RUN ./gradlew build --no-daemon

FROM eclipse-temurin:17-jre
COPY --from=builder /app/build/libs/*.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

# 9. 高级主题

## 9.1 自定义 Wrapper 脚本

可以在 `gradlew` 脚本中添加自定义逻辑：

```bash
#!/bin/sh

# 自定义：确保 correct Java version
REQUIRED_JAVA_VERSION="17"
JAVA_VERSION=$("$JAVACMD" -version 2>&1 | head -1 | cut -d'"' -f2 | cut -d'.' -f1)
if [ "$JAVA_VERSION" -lt "$REQUIRED_JAVA_VERSION" ]; then
    echo "ERROR: Java $REQUIRED_JAVA_VERSION+ required, found Java $JAVA_VERSION"
    exit 1
fi

# 原始 gradlew 逻辑...
```

## 9.2 多个 Gradle 版本的共存管理

在一个多项目仓库中，子项目可以使用不同版本的 Gradle：

```
monorepo/
├── settings.gradle.kts
├── gradle/wrapper/              # 根项目 Wrapper (Gradle 8.7)
│   └── gradle-wrapper.properties
├── project-a/
│   ├── gradle/wrapper/          # 子项目 A (Gradle 8.5)
│   │   └── gradle-wrapper.properties
│   └── build.gradle.kts
└── project-b/
    ├── gradle/wrapper/          # 子项目 B (Gradle 8.7)
    │   └── gradle-wrapper.properties
    └── build.gradle.kts
```

每个子项目使用自己的 `gradlew`，互不干扰。

## 9.3 Wrapper Jar 的版本兼容性

`gradle-wrapper.jar` 通常向前兼容。一个相对较新的 wrapper JAR 可以引导下载任意版本的 Gradle。最佳实践是安装项目使用的最新 Gradle 版本附带的 wrapper JAR。

## 9.4 离线模式

```bash
# 当网络不可用时（Gradle 发行版已在本地缓存）
./gradlew build --offline

# 或者设置 gradle.properties
# offline=true
```

注意：`--offline` 要求 Wrapper 下载的 Gradle 发行版已经存在于本地缓存中。

## 9.5 Wrapper 工作流总结

```
开发者克隆项目
    │
    ├── 执行 ./gradlew build
    │       │
    │       ├─ Wrapper 检查 ~/.gradle/wrapper/dists/ 是否有缓存的 Gradle
    │       │    ├─ 有 → 直接使用
    │       │    └─ 没有 → 从 distributionUrl 下载 → 验证 SHA-256 → 解压到缓存
    │       │
    │       └─ 使用正确的 Gradle 版本执行 build 任务
    │
    ├── CI 环境执行 ./gradlew build
    │       └─ 同样的流程（可配合 CI 缓存加速）
    │
    └── 升级 Gradle
            └── ./gradlew wrapper --gradle-version 8.8
                 → 更新 gradle-wrapper.properties
                 → git commit & push
                 → 所有环境自动使用新版本
```

## 9.6 Wrapper 常见问题排查

**问题 1：`Could not determine Java version`**

```
解决方案：确保 JAVA_HOME 指向有效的 JDK，或项目使用的 Gradle 版本支持当前 JDK。
Gradle 版本与 JDK 版本兼容性参考：
  - Gradle 8.x 支持 JDK 17-21
  - Gradle 7.x 支持 JDK 8-19
  - Gradle 6.x 支持 JDK 8-13
```

**问题 2：`Exception in thread "main" java.io.IOException: Downloading from ... failed`**

```
排查步骤：
1. 检查网络连接和代理配置
2. 验证 distributionUrl 中的 URL 是否正确可访问
3. 尝试手动下载并放入缓存目录
4. 检查公司防火墙是否阻止了 .zip 文件下载
```

**问题 3：Wrapper JAR 损坏**

```bash
# 重新下载 gradle-wrapper.jar
curl -L -o gradle/wrapper/gradle-wrapper.jar \
  https://raw.githubusercontent.com/gradle/gradle/v8.7.0/gradle/wrapper/gradle-wrapper.jar
```

**问题 4：不同操作系统路径问题**

```bash
# 提交到 Git 前确保 gradlew 有可执行权限
git update-index --chmod=+x gradlew
```

