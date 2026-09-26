
# 1. 概述

持续集成（Continuous Integration）和持续部署（Continuous Deployment）是现代软件开发的核心实践。Gradle 提供了丰富的功能和配置选项，能够与主流 CI/CD 平台（GitHub Actions、Jenkins、GitLab CI、TeamCity 等）深度集成，实现自动化构建、测试、发布流程。

## 1.1 CI/CD 中的 Gradle 架构

```
CI/CD 流水线中的 Gradle
┌────────────────────────────────────────────────────────────┐
│                                                             │
│  代码提交 → 触发 CI                                         │
│  ┌──────────────────────────────────────────────┐          │
│  │  1. Checkout & Setup                          │          │
│  │     ├── JDK 安装 (setup-java)                 │          │
│  │     └── 凭据配置                              │          │
│  │                                                │          │
│  │  2. Build & Test                               │          │
│  │     ├── gradle build --build-cache            │          │
│  │     ├── gradle test --parallel                │          │
│  │     └── gradle check (代码质量)               │          │
│  │                                                │          │
│  │  3. 报告 & 扫描                                │          │
│  │     ├── gradle --scan                         │          │
│  │     ├── 测试报告归档                           │          │
│  │     └── Build Scan 链接                       │          │
│  │                                                │          │
│  │  4. 发布 (可选)                                │          │
│  │     ├── gradle publish                        │          │
│  │     ├── 签名                                  │          │
│  │     └── 标签/Release                          │          │
│  └──────────────────────────────────────────────┘          │
│                                                             │
│  输出: 构建产物、报告、Build Scan 链接                       │
└────────────────────────────────────────────────────────────┘
```

# 2. CI/CD 通用原则

## 2.1 CI 环境下的关键 Gradle 参数

```bash
# CI 环境的标准 Gradle 执行命令
gradle build \
  --no-daemon \          # CI 环境不使用守护进程（容器生命周期短）
  --build-cache \        # 启用构建缓存
  --scan \               # 生成 Build Scan
  --parallel \           # 并行执行
  --stacktrace \         # 错误时显示完整堆栈
  --warning-mode all \   # 显示所有警告
  --info                 # 信息级别日志（出问题时有用）
```

## 2.2 CI 环境变量检测

```groovy
// 在 build.gradle 或约定插件中检测 CI 环境
def isCi = System.getenv('CI') || System.getenv('GITHUB_ACTIONS') ||
           System.getenv('JENKINS_HOME') || System.getenv('GITLAB_CI') ||
           System.getenv('TEAMCITY_VERSION')

ext {
    ci = isCi
    ciPlatform = detectCiPlatform()
}

def detectCiPlatform() {
    if (System.getenv('GITHUB_ACTIONS')) return 'github'
    if (System.getenv('JENKINS_HOME')) return 'jenkins'
    if (System.getenv('GITLAB_CI')) return 'gitlab'
    if (System.getenv('TEAMCITY_VERSION')) return 'teamcity'
    if (System.getenv('CI')) return 'generic-ci'
    return 'local'
}

// 根据 CI 环境调整行为
tasks.withType(Test).configureEach {
    maxParallelForks = isCi ? 1 : Runtime.runtime.availableProcessors()

    testLogging {
        events = isCi ? ['passed', 'skipped', 'failed'] : ['failed']
        showStandardStreams = isCi
        exceptionFormat = 'full'
    }

    // CI 环境：更严格的配置
    if (isCi) {
        outputs.upToDateWhen { false }  // 总是重新测试
    }
}
```

# 3. GitHub Actions 集成

## 3.1 基础工作流

```yaml
# .github/workflows/ci.yml
name: Java CI with Gradle

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        java: [ 17, 21 ]

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK ${{ matrix.java }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java }}
          distribution: 'temurin'

      - name: Set up Gradle
        uses: gradle/gradle-build-action@v3
        with:
          gradle-version: wrapper
          cache-disabled: false

      - name: Build with Gradle
        run: ./gradlew build --no-daemon --stacktrace

      - name: Upload Build Scan
        if: always()
        uses: gradle/gradle-build-action@v3
        with:
          arguments: build --scan
```

## 3.2 使用 gradle-build-action 的高级配置

```yaml
# .github/workflows/ci-advanced.yml
name: Advanced CI with Gradle

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - uses: gradle/gradle-build-action@v3
        with:
          # 启用 Gradle 构建缓存
          cache-read-only: ${{ github.event_name == 'pull_request' }}

          # 为构建扫描添加标签
          build-scan-publish: true
          build-scan-terms-of-use-agree: yes

          # 上传构建扫描链接作为工作流摘要
          build-scan-pr-comment: true

          # 指定要执行的参数（可选）
          arguments: |
            --no-daemon
            --build-cache
            --scan
            build

      - name: Publish Test Report
        uses: dorny/test-reporter@v1
        if: success() || failure()
        with:
          name: JUnit Tests
          path: '**/build/test-results/test/TEST-*.xml'
          reporter: java-junit
```

## 3.3 矩阵构建 + 缓存

```yaml
# .github/workflows/matrix-ci.yml
name: Matrix Build

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ ubuntu-latest, windows-latest, macos-latest ]
        java: [ 17, 21 ]
        exclude:
          # 排除不需要的组合
          - os: macos-latest
            java: 17

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java }}
          distribution: 'temurin'

      - uses: gradle/gradle-build-action@v3
        with:
          # PR 构建：只读缓存
          cache-read-only: ${{ github.event_name == 'pull_request' }}
          # 主分支推送：可写缓存
          cache-write-only: false

          # 为每个矩阵组合添加标签
          build-scan-publish: true
          build-scan-terms-of-use-agree: yes
          build-scan-tags: |
            os-${{ matrix.os }}
            java-${{ matrix.java }}

      - name: Build
        run: ./gradlew build --no-daemon --build-cache
```

## 3.4 构建扫描发布

```yaml
# .github/workflows/build-scan.yml
name: Publish Build Scan

on:
  workflow_dispatch:  # 手动触发
  schedule:
    - cron: '0 6 * * 1'  # 每周一早上 6 点

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - uses: gradle/gradle-build-action@v3
        with:
          # 自定义 Gradle Enterprise 服务器
          gradle-enterprise-server: 'https://ge.mycompany.com'
          build-scan-publish: true
          build-scan-terms-of-use-agree: yes

          # 添加自定义标签和链接
          build-scan-tags: 'scheduled', 'nightly'
          build-scan-links: |
            Repository=${{ github.server_url }}/${{ github.repository }}

      - name: Run Full Build with Scan
        run: ./gradlew build --scan --no-daemon
```

# 4. Jenkins 集成

## 4.1 Declarative Pipeline

```groovy
// Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any

    tools {
        jdk 'jdk21'
    }

    environment {
        GRADLE_OPTS = '-Dorg.gradle.daemon=false -Dorg.gradle.parallel=true'
        GRADLE_USER_HOME = "${WORKSPACE}/.gradle"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh './gradlew build --build-cache --no-daemon'
            }
        }

        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh './gradlew test --no-daemon'
                    }
                    post {
                        always {
                            junit '**/build/test-results/test/TEST-*.xml'
                        }
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh './gradlew integrationTest --no-daemon'
                    }
                }
            }
        }

        stage('Code Quality') {
            steps {
                sh './gradlew checkstyleMain checkstyleTest pmdMain pmdTest'
            }
        }

        stage('Publish') {
            when {
                branch 'main'
            }
            steps {
                sh './gradlew publish --no-daemon'
            }
            post {
                success {
                    emailext(
                        subject: "发布成功: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                        body: "构建成功，产物已发布到仓库。",
                        to: 'team@company.com'
                    )
                }
            }
        }
    }

    post {
        always {
            // 归档构建扫描链接
            script {
                if (fileExists('build/reports/build-scan-url.txt')) {
                    def scanUrl = readFile('build/reports/build-scan-url.txt').trim()
                    echo "Build Scan: ${scanUrl}"
                }
            }
            // 清理
            cleanWs()
        }
        failure {
            emailext(
                subject: "构建失败: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: "构建失败，请检查日志。",
                to: 'team@company.com'
            )
        }
    }
}
```

## 4.2 使用 Jenkins Gradle Plugin

```groovy
// Jenkinsfile - 使用 Gradle Plugin
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                // 使用 Gradle Wrapper（推荐）
                gradle(
                    switches: '--build-cache --no-daemon',
                    tasks: 'build'
                )
            }
        }

        stage('Test') {
            steps {
                gradle(tasks: 'test --parallel')
            }
            post {
                always {
                    // 自动收集测试结果
                    junit '**/build/test-results/**/*.xml'
                }
            }
        }
    }
}
```

## 4.3 构建缓存与凭据

```groovy
// Jenkinsfile - 使用远程构建缓存
pipeline {
    agent any

    environment {
        GRADLE_CACHE_USER = credentials('gradle-build-cache-user')
        GRADLE_CACHE_PASS = credentials('gradle-build-cache-pass')
    }

    stages {
        stage('Build with Cache') {
            steps {
                sh '''
                    BUILD_CACHE_USER="${GRADLE_CACHE_USER}" \
                    BUILD_CACHE_PASS="${GRADLE_CACHE_PASS}" \
                    ./gradlew build --build-cache --scan
                '''
            }
        }
    }

    post {
        always {
            // 保存 Gradle 缓存用于增量构建
            stash includes: '.gradle/**', name: 'gradle-cache'
        }
    }
}
```

# 5. GitLab CI 集成

## 5.1 基础配置

```yaml
# .gitlab-ci.yml
image: eclipse-temurin:21-jdk

variables:
  GRADLE_OPTS: "-Dorg.gradle.daemon=false -Dorg.gradle.parallel=true"
  GRADLE_USER_HOME: "$CI_PROJECT_DIR/.gradle"

stages:
  - build
  - test
  - quality
  - deploy

# 缓存配置
cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - .gradle/caches/
    - .gradle/wrapper/
    - .gradle/build-cache/
    - build/

before_script:
  - chmod +x gradlew

build:
  stage: build
  script:
    - ./gradlew assemble --build-cache --no-daemon
  artifacts:
    paths:
      - build/libs/
      - build/distributions/
    expire_in: 1 week

unit-test:
  stage: test
  script:
    - ./gradlew test --no-daemon
  artifacts:
    when: always
    reports:
      junit: '**/build/test-results/test/TEST-*.xml'
    paths:
      - '**/build/reports/tests/'

integration-test:
  stage: test
  script:
    - ./gradlew integrationTest --no-daemon
  artifacts:
    when: always
    reports:
      junit: '**/build/test-results/integrationTest/TEST-*.xml'

checkstyle:
  stage: quality
  script:
    - ./gradlew checkstyleMain checkstyleTest --no-daemon
  allow_failure: true

deploy-to-staging:
  stage: deploy
  script:
    - ./gradlew publish -Penv=staging --no-daemon
  environment:
    name: staging
  only:
    - develop

deploy-to-production:
  stage: deploy
  script:
    - ./gradlew publish -Penv=production --no-daemon
  environment:
    name: production
  only:
    - main
  when: manual
```

## 5.2 多阶段缓存优化

```yaml
# .gitlab-ci.yml - 多阶段缓存
stages:
  - setup
  - build
  - test
  - release

# 全局缓存（依赖 + 构建产物）
.global-cache: &global-cache
  cache:
    key: "$CI_COMMIT_REF_SLUG-$CI_JOB_STAGE"
    policy: pull-push
    paths:
      - .gradle/caches/
      - .gradle/build-cache/

# 使用 Gradle Wrapper 验证
validate-wrapper:
  stage: setup
  script:
    - ./gradlew --version
  <<: *global-cache

compile:
  stage: build
  script:
    - ./gradlew compileJava compileTestJava --build-cache --no-daemon
  <<: *global-cache

test:
  stage: test
  script:
    - ./gradlew test --build-cache --no-daemon
  <<: *global-cache
  artifacts:
    reports:
      junit: '**/build/test-results/**/TEST-*.xml'
```

## 5.3 远程缓存集成

```yaml
# .gitlab-ci.yml - 使用 Gradle Enterprise 远程缓存
variables:
  GRADLE_CACHE_REMOTE_URL: "https://gradle-enterprise.company.com/cache/"
  GRADLE_ENTERPRISE_ACCESS_KEY: "$GRADLE_ENTERPRISE_TOKEN"

build-with-remote-cache:
  stage: build
  script:
    - ./gradlew build
      --build-cache
      --no-daemon
      -Dgradle.cache.remote.url="${GRADLE_CACHE_REMOTE_URL}"
      -Dgradle.cache.remote.push=true
      -Dgradle.cache.remote.username=ci
      -Dgradle.cache.remote.password="${GRADLE_ENTERPRISE_ACCESS_KEY}"
  cache:
    paths:
      - .gradle/
```

# 6. TeamCity 集成

## 6.1 Gradle Runner 配置

在 TeamCity 中配置 Gradle Runner:

```xml
<!-- TeamCity 构建步骤: Gradle -->
<build-step>
  <name>Build with Gradle</name>
  <type>gradle</type>
  <properties>
    <property name="gradle.tasks" value="clean build --build-cache" />
    <property name="gradle.params" value="--no-daemon --parallel" />
    <property name="use.gradle.wrapper" value="true" />
    <property name="teamcity.build.workingDir" value="" />
  </properties>
</build-step>
```

## 6.2 Kotlin DSL Pipeline

```kotlin
// .teamcity/settings.kts (TeamCity 配置即代码)
import jetbrains.buildServer.configs.kotlin.*
import jetbrains.buildServer.configs.kotlin.buildFeatures.gradleBuildCache
import jetbrains.buildServer.configs.kotlin.buildSteps.gradle

version = "2024.03"

project {
    buildType(Build)
}

object Build : BuildType({
    name = "Gradle Build"

    vcs {
        root(DslContext.settingsRoot)
    }

    steps {
        gradle {
            name = "Compile and Test"
            tasks = "build"
            gradleParams = """
                --no-daemon
                --build-cache
                --parallel
                --stacktrace
                -Pteamcity=true
            """.trimIndent()
            useGradleWrapper = true
        }

        gradle {
            name = "Code Quality"
            tasks = "checkstyleMain checkstyleTest pmdMain"
            gradleParams = "--no-daemon"
            useGradleWrapper = true
        }
    }

    features {
        // 启用 TeamCity 构建缓存集成
        gradleBuildCache {
            push = true
            url = "https://teamcity.company.com/cache/"
        }
    }
})
```

## 6.3 Build Scan 集成

```groovy
// build.gradle - TeamCity Build Scan 集成
if (System.getenv('TEAMCITY_VERSION')) {
    gradleEnterprise {
        buildScan {
            publishAlways()
            tag 'teamcity'
            tag System.getenv('TEAMCITY_BUILDCONF_NAME') ?: 'unknown'

            link 'TeamCity Build',
                 System.getenv('BUILD_URL') ?: 'https://teamcity.company.com'

            // 将 TeamCity 构建号添加到自定义值
            value 'Build Number', System.getenv('BUILD_NUMBER') ?: 'local'

            // 将 Git 信息添加到 Build Scan
            if (System.getenv('BUILD_VCS_NUMBER')) {
                value 'Git Revision', System.getenv('BUILD_VCS_NUMBER')
                tag System.getenv('BUILD_VCS_BRANCH') ?: 'unknown-branch'
            }

            // 设置后台任务失败时也发布
            background {
                publishAlways()
            }
        }
    }
}
```

# 7. Docker 构建中的 Gradle

## 7.1 多阶段 Docker 构建

```dockerfile
# Dockerfile - 多阶段构建
# 阶段 1: 构建阶段
FROM eclipse-temurin:21-jdk AS builder

WORKDIR /app

# 复制 Gradle 配置（利用缓存分层）
COPY gradlew .
COPY gradle/ gradle/
COPY build.gradle .
COPY settings.gradle .

# 下载依赖（利用 Docker 缓存）
RUN ./gradlew dependencies --no-daemon || return 0

# 复制源代码并构建
COPY src/ src/
RUN ./gradlew build --no-daemon -x test

# 阶段 2: 运行阶段
FROM eclipse-temurin:21-jre

WORKDIR /app

# 创建非 root 用户
RUN groupadd -r appuser && useradd -r -g appuser appuser

# 复制构建产物
COPY --from=builder /app/build/libs/*.jar app.jar

USER appuser

EXPOSE 8080

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 7.2 Jib 插件

Jib 是 Google 开发的容器镜像构建工具，可直接从 Gradle 构建 Docker 镜像而无需 Docker daemon。

```groovy
plugins {
    id 'com.google.cloud.tools.jib' version '3.4.4'
}

jib {
    from {
        image = 'eclipse-temurin:21-jre'
        platforms {
            platform {
                architecture = 'amd64'
                os = 'linux'
            }
        }
    }

    to {
        image = "registry.company.com/${project.name}:${project.version}"
        tags = ['latest']
        auth {
            username = System.getenv('REGISTRY_USER')
            password = System.getenv('REGISTRY_PASS')
        }
    }

    container {
        jvmFlags = [
            '-Xms512m',
            '-Xmx1g',
            '-XX:+UseG1GC',
            '-Djava.security.egd=file:/dev/urandom'
        ]
        ports = ['8080', '8443']
        environment = [
            'SPRING_PROFILES_ACTIVE': 'production'
        ]
        user = 'nobody:nogroup'
        creationTime = 'USE_CURRENT_TIMESTAMP'
        format = 'OCI'
    }
}

// CI 环境构建并推送
tasks.register('dockerBuildAndPush') {
    dependsOn 'jib'
}

// 本地开发构建到 Docker daemon
tasks.register('dockerBuildLocal') {
    dependsOn 'jibDockerBuild'
}
```

## 7.3 Spring Boot BuildImage

```groovy
plugins {
    id 'org.springframework.boot' version '3.2.5'
}

// 使用 Cloud Native Buildpacks
tasks.named('bootBuildImage') {
    imageName = "registry.company.com/${project.name}:${project.version}"
    environment = [
        'BP_JVM_VERSION': '21',
        'BP_NATIVE_IMAGE': 'false'
    ]
    publish = System.getenv('CI')?.toBoolean() ?: false

    docker {
        publishRegistry {
            username = System.getenv('REGISTRY_USER')
            password = System.getenv('REGISTRY_PASS')
            url = 'https://registry.company.com'
        }
    }

    buildpacks = [
        'urn:cnb:builder:paketo-buildpacks/java',
        'gcr.io/paketo-buildpacks/liberty'
    ]
}
```

# 8. 环境特定配置

## 8.1 构建类型与风味

```groovy
// 自定义 SourceSet 或配置来区分环境
ext {
    buildEnv = project.findProperty('buildEnv') ?: 'development'
}

// 构建类型配置
configurations {
    developmentImplementation
    stagingImplementation
    productionImplementation
}

dependencies {
    if (buildEnv == 'development') {
        developmentImplementation 'com.h2database:h2:2.2.224'
    } else {
        runtimeOnly 'org.postgresql:postgresql:42.7.3'
    }
}

// 资源过滤
processResources {
    filesMatching('**/application.properties') {
        filter { line ->
            line.replace('@buildEnv@', buildEnv)
        }
    }
}

// 条件任务注册
tasks.register('deploy') {
    doLast {
        switch (buildEnv) {
            case 'staging':
                println "部署到预发布环境..."
                break
            case 'production':
                println "需要手动确认才能部署到生产环境"
                break
            default:
                println "当前环境: $buildEnv (仅本地部署)"
        }
    }
}
```

## 8.2 环境配置文件

```groovy
// 从外部文件加载环境配置
def loadEnvConfig(String env) {
    def configFile = file("config/${env}.gradle")
    if (configFile.exists()) {
        apply from: configFile
    } else {
        logger.warn("环境配置文件不存在: ${configFile}")
    }
}

def activeEnv = project.findProperty('env') ?:
    System.getenv('DEPLOY_ENV') ?: 'development'

loadEnvConfig(activeEnv)
```

```groovy
// config/staging.gradle
ext {
    deployUrl = 'https://staging-api.company.com'
    databaseUrl = 'jdbc:postgresql://staging-db:5432/app'
}

// config/production.gradle
ext {
    // 不要硬编码凭据！使用环境变量
    deployUrl = System.getenv('PROD_API_URL')
    databaseUrl = System.getenv('PROD_DB_URL')
}
```

## 8.3 Gradle 属性系统

```properties
# gradle.properties - 通用默认值
app.name=my-application
app.version=1.0.0

# 开发环境使用 H2 数据库
database.driver=org.h2.Driver
database.url=jdbc:h2:mem:testdb

# CI 环境通过 -P 或环境变量覆盖
# ./gradlew build -Pdatabase.url=jdbc:mysql://prod-db:3306/app
```

# 9. 标签与发布版本

## 9.1 SNAPSHOT vs Release

```groovy
// 自动判断是否为 SNAPSHOT 版本
group = 'com.example'
version = determineVersion()

def determineVersion() {
    def baseVersion = project.findProperty('releaseVersion') ?: '1.0.0'

    if (System.getenv('CI') && System.getenv('GITHUB_REF')?.startsWith('refs/tags/')) {
        // 从 Git 标签获取版本号
        def tag = System.getenv('GITHUB_REF').replace('refs/tags/v', '')
        return tag
    }

    if (project.hasProperty('release')) {
        return baseVersion  // 正式发布版本
    }

    // 开发构建：使用 SNAPSHOT
    return "${baseVersion}-SNAPSHOT"
}

// 打印版本
println "构建版本: ${project.version}"
```

## 9.2 Git 标签驱动发布

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Extract version from tag
        run: echo "RELEASE_VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_ENV

      - name: Build and Publish
        run: |
          ./gradlew publish \
            -PreleaseVersion=$RELEASE_VERSION \
            -Prelease=true \
            --no-daemon
        env:
          OSSRH_USERNAME: ${{ secrets.OSSRH_USERNAME }}
          OSSRH_PASSWORD: ${{ secrets.OSSRH_PASSWORD }}
          SIGNING_KEY: ${{ secrets.SIGNING_KEY }}
          SIGNING_PASSWORD: ${{ secrets.SIGNING_PASSWORD }}
```

## 9.3 发布到 Maven Central

```groovy
plugins {
    id 'java-library'
    id 'maven-publish'
    id 'signing'
}

java {
    withJavadocJar()
    withSourcesJar()
}

publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java

            pom {
                name = 'My Library'
                description = 'A high-performance Java library'
                url = 'https://github.com/mycompany/project'

                licenses {
                    license {
                        name = 'Apache License 2.0'
                        url = 'https://www.apache.org/licenses/LICENSE-2.0'
                    }
                }
                developers {
                    developer {
                        id = 'developer1'
                        name = 'Developer Name'
                        email = 'dev@company.com'
                    }
                }
                scm {
                    connection = 'scm:git:git://github.com/mycompany/project.git'
                    developerConnection = 'scm:git:ssh://github.com/mycompany/project.git'
                    url = 'https://github.com/mycompany/project'
                }
            }
        }
    }

    repositories {
        maven {
            def releasesRepoUrl = 'https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/'
            def snapshotsRepoUrl = 'https://s01.oss.sonatype.org/content/repositories/snapshots/'
            url = version.endsWith('SNAPSHOT') ? snapshotsRepoUrl : releasesRepoUrl

            credentials {
                username = System.getenv('OSSRH_USERNAME')
                password = System.getenv('OSSRH_PASSWORD')
            }
        }
    }
}

signing {
    def signingKey = System.getenv('SIGNING_KEY')
    def signingPassword = System.getenv('SIGNING_PASSWORD')

    if (signingKey && signingPassword) {
        useInMemoryPgpKeys(signingKey, signingPassword)
        sign publishing.publications.mavenJava
    }
}
```

# 10. CI 中的工件签名

## 10.1 GPG 签名

```yaml
# .github/workflows/release-sign.yml
name: Sign and Release

on:
  release:
    types: [ published ]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Import GPG Key
        run: |
          echo "${{ secrets.GPG_PRIVATE_KEY }}" | gpg --batch --import
          echo "${{ secrets.GPG_PASSPHRASE }}" | gpg --batch --passphrase-fd 0 --pinentry-mode loopback --sign

      - name: Publish with Signing
        run: ./gradlew publish --no-daemon
        env:
          OSSRH_USERNAME: ${{ secrets.OSSRH_USERNAME }}
          OSSRH_PASSWORD: ${{ secrets.OSSRH_PASSWORD }}
          ORG_GRADLE_PROJECT_signingKey: ${{ secrets.GPG_PRIVATE_KEY }}
          ORG_GRADLE_PROJECT_signingPassword: ${{ secrets.GPG_PASSPHRASE }}
```

## 10.2 校验和生成

```groovy
// 生成 SHA-256 和 MD5 校验和
tasks.register('generateChecksums') {
    dependsOn assemble

    doLast {
        def artifactsDir = layout.buildDirectory.dir('libs').get().asFile
        artifactsDir.eachFileMatch(~/.*\.(jar|war)$/) { file ->
            ['md5', 'sha1', 'sha256'].each { algo ->
                def checksum = file.text.digest(algo.toUpperCase())
                new File(file.parentFile, "${file.name}.${algo}").text = checksum
            }
        }
    }
}

// Maven Central 需要：JAR, Sources JAR, Javadoc JAR 及其校验和
tasks.named('publish') {
    dependsOn 'generateChecksums'
}
```

# 11. 多分支构建

## 11.1 分支策略

```yaml
# GitHub Actions: 根据分支调整行为
name: Branch-specific Build

on:
  push:
    branches: [ main, develop, 'feature/**', 'release/**' ]

jobs:
  determine-strategy:
    runs-on: ubuntu-latest
    outputs:
      build-type: ${{ steps.set-strategy.outputs.build-type }}
    steps:
      - id: set-strategy
        run: |
          if [[ "$GITHUB_REF" == "refs/heads/main" ]]; then
            echo "build-type=release" >> $GITHUB_OUTPUT
          elif [[ "$GITHUB_REF" == refs/heads/feature/* ]]; then
            echo "build-type=snapshot" >> $GITHUB_OUTPUT
          elif [[ "$GITHUB_REF" == refs/heads/release/* ]]; then
            echo "build-type=rc" >> $GITHUB_OUTPUT
          else
            echo "build-type=development" >> $GITHUB_OUTPUT
          fi

  build:
    needs: determine-strategy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build with strategy
        run: |
          ./gradlew build --no-daemon \
            -PbuildType=${{ needs.determine-strategy.outputs.build-type }}
```

```groovy
// build.gradle - 根据构建类型调整行为
def buildType = project.findProperty('buildType') ?: 'development'

version = switch (buildType) {
    case 'release' -> project.findProperty('releaseVersion') ?: '1.0.0'
    case 'rc' -> "${project.findProperty('releaseVersion') ?: '1.0.0'}-rc.${System.getenv('BUILD_NUMBER') ?: 1}"
    case 'snapshot' -> "${project.findProperty('releaseVersion') ?: '1.0.0'}-SNAPSHOT"
    default -> '0.0.0-dev'
}

tasks.register('deploy') {
    onlyIf { buildType in ['release', 'rc'] }
    doLast { println "Deploying version ${version}" }
}
```

# 12. 矩阵构建策略

## 12.1 OS + JDK 矩阵

```yaml
# .github/workflows/matrix-full.yml
name: Full Matrix Build

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    strategy:
      fail-fast: false  # 其他配置继续，不要因一个失败而取消
      matrix:
        os: [ ubuntu-22.04, ubuntu-24.04, windows-2022, macos-14 ]
        java: [ 17, 21, 22 ]
        include:
          # 添加特殊组合
          - os: ubuntu-24.04
            java: 23
            experimental: true
        exclude:
          # 排除已知有问题的组合
          - os: windows-2022
            java: 22

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java }}
          distribution: 'temurin'

      - uses: gradle/gradle-build-action@v3

      - name: Build and Test
        run: |
          ./gradlew build \
            --no-daemon \
            -Dorg.gradle.jvmargs="-Xmx2g" \
            --build-cache \
            --scan

      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ matrix.os }}-${{ matrix.java }}
          path: '**/build/test-results/'
```

## 12.2 多维矩阵

```groovy
// build.gradle - 从矩阵中获取参数
ext {
    // 通过 -P 参数从 CI 矩阵传入
    targetOs = project.findProperty('targetOs') ?: System.getProperty('os.name')
    targetJdk = project.findProperty('targetJdk') ?: JavaVersion.current()
}
```

# 13. CI 缓存策略

## 13.1 分层缓存

```yaml
# GitHub Actions: 分层缓存策略
- uses: gradle/gradle-build-action@v3
  with:
    # 缓存策略
    cache-encryption-key: ${{ secrets.CACHE_ENCRYPTION_KEY }}

    # Gradle 用户目录的附加缓存路径
    gradle-home-cache-includes: |
      caches/
      notifications/
      jdks/

    # 排除不需要缓存的目录
    gradle-home-cache-excludes: |
      caches/build-cache-1/journal/
      caches/transforms-3/journal/
      daemon/
```

## 13.2 条件缓存

```groovy
// settings.gradle - CI 条件缓存
def isCi = System.getenv('CI')?.toBoolean() ?: false
def isPr = System.getenv('GITHUB_EVENT_NAME') == 'pull_request'

buildCache {
    local {
        enabled = true
    }

    remote(HttpBuildCache) {
        url = uri(System.getenv('BUILD_CACHE_URL') ?: 'https://cache.company.com/cache/')

        // PR 构建：只拉取，不推送（防止污染）
        push = isCi && !isPr
        pull = true

        credentials {
            username = System.getenv('CACHE_USER')
            password = System.getenv('CACHE_PASS')
        }
    }
}
```

## 13.3 缓存大小管理

```groovy
// settings.gradle - 缓存大小限制
buildCache {
    local {
        // 自动清理 30 天前的缓存
        removeUnusedEntriesAfterDays = 30

        // 设置最大缓存大小（不直接支持，但可以监控）
        // 定期运行: gradle cleanBuildCache
    }
}
```

# 14. Build Scan 在 CI 中的应用

## 14.1 构建扫描链接作为 CI 工件

```groovy
// build.gradle - 收集构建扫描链接
abstract class BuildScanPublisher extends DefaultTask {

    @OutputFile
    abstract RegularFileProperty getScanUrlFile()

    BuildScanPublisher() {
        outputs.upToDateWhen { false }
    }

    @TaskAction
    void publish() {
        // Gradle Enterprise 会自动导出扫描 URL
        // 将 URL 写入文件供 CI 系统读取
        if (System.getenv('CI')) {
            def scanUrl = System.getProperty('gradle.scan.url') ?: 'not-available'
            scanUrlFile.get().asFile.text = scanUrl
            println "构建扫描: $scanUrl"
        }
    }
}

gradleEnterprise {
    buildScan {
        publishAlways()
        tag(System.getenv('CI') ? 'CI' : 'local')

        capture {
            taskInputFiles = true
        }

        uploadInBackground = !System.getenv('CI')  // CI 中同步上传
    }
}
```

## 14.2 自定义构建扫描标签

```groovy
gradleEnterprise {
    buildScan {
        // CI 环境标签
        tag System.getenv('CI') ? 'ci' : 'local'
        tag System.getenv('CI_PLATFORM') ?: 'unknown'

        // 分支标签
        def branch = System.getenv('GIT_BRANCH') ?:
            System.getenv('GITHUB_REF_NAME') ?: 'unknown'
        tag branch.replace('/', '-')

        // 构建类型标签
        tag project.findProperty('buildType') ?: 'development'

        // 链接回 CI 构建
        def buildUrl = System.getenv('BUILD_URL') ?:
            System.getenv('CI_JOB_URL') ?:
            "${System.getenv('GITHUB_SERVER_URL')}/${System.getenv('GITHUB_REPOSITORY')}/actions/runs/${System.getenv('GITHUB_RUN_ID')}"

        if (buildUrl) {
            link 'CI Build', buildUrl
        }

        // 自定义值
        value 'JDK Version', System.getProperty('java.version')
        value 'OS', System.getProperty('os.name')
        value 'Git SHA', System.getenv('GITHUB_SHA') ?:
            System.getenv('CI_COMMIT_SHA') ?: 'unknown'
    }
}
```

# 15. 故障排查与监控

## 15.1 构建时间监控

```groovy
// 记录每个任务的执行时间
long buildStartTime

gradle.buildStarted {
    buildStartTime = System.currentTimeMillis()
}

gradle.taskGraph.beforeTask { task ->
    task.ext.startTime = System.currentTimeMillis()
}

gradle.taskGraph.afterTask { task, state ->
    long duration = System.currentTimeMillis() - task.ext.startTime

    if (duration > 5000) {
        logger.warn("慢任务: ${task.path} 耗时 ${duration}ms (${state.failure ? '失败' : '成功'})")
    }
}

gradle.buildFinished { result ->
    long totalDuration = System.currentTimeMillis() - buildStartTime
    println "=================================="
    println "构建完成: ${result.action}"
    println "总耗时: ${totalDuration / 1000}s"
    println "=================================="

    if (System.getenv('CI')) {
        def report = file("${buildDir}/reports/build-timing.txt")
        report.parentFile.mkdirs()
        report.text = "total=${totalDuration}ms\nresult=${result.action}\n"
    }
}
```

## 15.2 CI 环境调试

```bash
# 打印环境信息用于调试
gradle build --info \
  -Dorg.gradle.caching.debug=true \
  -Dorg.gradle.dependency.verification=off

# 查看所有任务执行时间
gradle build --profile

# 生成 HTML 性能报告
# 查看 ${buildDir}/reports/profile/
```

## 15.3 失败自动重试

```groovy
// 配置任务失败重试（适用于 CI 环境的网络不稳定）
tasks.withType(Test).configureEach {
    if (System.getenv('CI')) {
        retries {
            maxRetries = 2
            failOnPassedAfterRetry = false
        }
    }
}

// Gradle 8.x+ 的任务重试
// gradle.properties:
// org.gradle.retry.max=2
```

# 16. 完整的 CI/CD 流水线参考

## 16.1 端到端流水线

```yaml
# .github/workflows/full-pipeline.yml
name: Full CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  GRADLE_OPTS: "-Dorg.gradle.daemon=false"
  JAVA_VERSION: "21"

jobs:
  # ========== 阶段 1: 验证 ==========
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: "${{ env.JAVA_VERSION }}", distribution: 'temurin' }
      - uses: gradle/gradle-build-action@v3
      - run: ./gradlew wrapper --gradle-version 8.7

  # ========== 阶段 2: 构建和测试 ==========
  build-and-test:
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: "${{ env.JAVA_VERSION }}", distribution: 'temurin' }
      - uses: gradle/gradle-build-action@v3
        with:
          cache-read-only: ${{ github.event_name == 'pull_request' }}

      - name: Compile
        run: ./gradlew compileJava compileTestJava --no-daemon

      - name: Test
        run: ./gradlew test integrationTest --no-daemon

      - name: Upload Test Reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: '**/build/reports/tests/'

  # ========== 阶段 3: 代码质量 ==========
  quality:
    needs: build-and-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: "${{ env.JAVA_VERSION }}", distribution: 'temurin' }
      - uses: gradle/gradle-build-action@v3
      - run: |
          ./gradlew \
            checkstyleMain checkstyleTest \
            pmdMain pmdTest \
            spotbugsMain spotbugsTest \
            jacocoTestReport \
            --no-daemon

  # ========== 阶段 4: 发布（仅 main 分支）==========
  publish:
    needs: quality
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: "${{ env.JAVA_VERSION }}", distribution: 'temurin' }
      - uses: gradle/gradle-build-action@v3
      - name: Publish to Repository
        run: ./gradlew publish --no-daemon
        env:
          REPO_USER: ${{ secrets.REPO_USER }}
          REPO_PASS: ${{ secrets.REPO_PASS }}

  # ========== 阶段 5: Docker 镜像（仅 main 分支）==========
  docker:
    needs: publish
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: "${{ env.JAVA_VERSION }}", distribution: 'temurin' }
      - uses: gradle/gradle-build-action@v3
      - name: Build and Push Docker Image
        run: ./gradlew jib --no-daemon
        env:
          REGISTRY_USER: ${{ secrets.REGISTRY_USER }}
          REGISTRY_PASS: ${{ secrets.REGISTRY_PASS }}

  # ========== 阶段 6: 通知 ==========
  notify:
    needs: [ pytest, docker ]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Slack Notification
        uses: slackapi/slack-github-action@v2
        with:
          payload: |
            {
              "text": "构建 ${{ job.status }}: ${{ github.repository }} @${{ github.ref }}"
            }
```

# 17. 总结

| 平台 | 关键工具/配置 | 核心特性 |
|------|--------------|----------|
| GitHub Actions | `gradle-build-action`, `setup-java` | 自动缓存、构建扫描 |
| Jenkins | Gradle Plugin, Pipeline DSL | 灵活脚本、凭据管理 |
| GitLab CI | `image`, `cache`, `artifacts` | 原生缓存、工件管理 |
| TeamCity | Gradle Runner, Kotlin DSL | Enterprise 集成、构建缓存 |
| Docker | Jib, Buildpacks, 多阶段 Dockerfile | 容器化构建 |
| Maven Central | `maven-publish`, `signing` | 发布和签名 |
| 通用 | `--build-cache`, `--scan`, `--no-daemon` | CI 环境优化 |

CI/CD 集成的核心在于：使用 Wrapper 确保 Gradle 版本一致、启用构建缓存提升速度、使用 Build Scan 诊断问题、通过环境变量管理凭据和配置。合理的流水线设计能够显著提高团队的开发效率和交付质量。
