
# 1. 流水线集成

- 常见流程：拉取代码 -> 依赖缓存 -> 编译测试 -> 打包发布。
- 推荐使用批处理模式：

```bash
mvn -B -V clean verify
```

- `-B`：非交互模式
- `-V`：输出版本信息便于排查

# 2. CI 平台集成示例

## 2.1 GitHub Actions

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Build with Maven
        run: mvn -B -V clean verify

      - name: Publish Test Report
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Maven Tests
          path: '**/target/surefire-reports/*.xml'
          reporter: java-junit

      - name: Deploy to Nexus
        if: github.ref == 'refs/heads/main'
        run: mvn -B -DskipTests deploy
```

## 2.2 Jenkins Pipeline (Jenkinsfile)

```groovy
pipeline {
    agent any
    tools { maven 'maven-3.9' }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Build & Test') {
            steps {
                sh 'mvn -B -V clean verify'
            }
        }
        stage('SonarQube') {
            steps {
                sh 'mvn sonar:sonar -Dsonar.host.url=$SONAR_URL'
            }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh 'mvn -B -DskipTests deploy'
            }
        }
    }
    post {
        always {
            junit '**/target/surefire-reports/*.xml'
            archiveArtifacts artifacts: '**/target/*.jar'
        }
    }
}
```

## 2.3 GitLab CI

```yaml
stages:
  - build
  - test
  - deploy

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"

cache:
  paths:
    - .m2/repository/

build:
  stage: build
  image: maven:3.9-eclipse-temurin-21
  script: mvn -B -V clean compile
  artifacts:
    paths:
      - target/

test:
  stage: test
  image: maven:3.9-eclipse-temurin-21
  script: mvn -B verify
  artifacts:
    reports:
      junit: "**/target/surefire-reports/TEST-*.xml"

deploy:
  stage: deploy
  image: maven:3.9-eclipse-temurin-21
  script: mvn -B -DskipTests deploy
  only:
    - main
```

## 2.4 Maven Daemon (mvnd) 加速

- `mvnd` 是 Apache 官方推出的 Maven 守护进程版本——JVM 常驻，消除 JVM 启动开销。
- 安装后替换 `mvn` 命令即可：

```bash
# 传统
mvn clean install

# mvnd（自动复用 Daemon JVM，构建快 30-50%）
mvnd clean install
```

- Gradle 风格指定线程数：
```bash
mvnd -T 1C clean install
```

- 特点：
	- 首次启动与 mvn 相同，后续构建复用 JVM。
	- 兼容所有 Maven 插件与命令。
	- GitHub Actions 中可通过 `mvnd` action 加速。

# 3. 缓存与加速

- 缓存本地仓库 `~/.m2/repository`。
- 使用私服镜像统一入口。
- 可指定 CI 临时仓库路径：

```bash
mvn -Dmaven.repo.local=$WORKSPACE/.m2/repository clean verify
```

# 4. 质量门禁与报告

- 常见插件：
	- `maven-surefire-plugin`（单测）
	- `jacoco-maven-plugin`（覆盖率）
	- `maven-checkstyle-plugin` / `spotbugs-maven-plugin`
	- `maven-enforcer-plugin`（规则约束）

建议将报告产物上传到 CI 工件库。

# 5. CI 中的版本管理

- 使用统一版本号策略：
	- 通过 `-Drevision=1.0.0` 控制版本
	- 结合 Git 标签生成版本

示例：
```bash
mvn -Drevision=1.2.0 clean package
```

# 6. 部署与凭据管理

- `settings.xml` 在 CI 中注入（不要写入源码库）。
- 凭据建议使用密钥管理或环境变量。
- 使用 `mvn deploy` 发布到私服。

# 7. 自动化依赖更新

## 7.1 Dependabot（GitHub 原生）

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "maven"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    target-branch: "develop"
    labels:
      - "dependencies"
```

## 7.2 Renovate

```json
// renovate.json
{
  "extends": ["config:base"],
  "packageRules": [
    {
      "matchPackagePatterns": ["org.springframework.*"],
      "groupName": "Spring Framework"
    }
  ]
}
```

## 7.3 versions-maven-plugin（手动检查）

```bash
# 查看所有可更新的依赖
mvn versions:display-dependency-updates

# 查看可更新的插件
mvn versions:display-plugin-updates

# 查看可更新的父 POM
mvn versions:display-parent-updates
```

- 建议在 CI 中定期执行并生成报告，但不要自动提交变更。

# 8. 常见失败与排查

- 依赖下载失败：检查网络、镜像、私服权限。
- 测试失败：优先查看 `target/surefire-reports`。
- 编码问题：设置 `project.build.sourceEncoding`。
- 版本冲突：使用 `mvn dependency:tree -Dverbose`。
- 本地仓库损坏：清理对应目录后重试。

# 9. 常见问题排查清单

- CI 机器构建慢：开启依赖缓存并使用私服。
- 本地能过 CI 失败：检查 JDK 版本与 Maven 版本一致性。
- settings.xml 未生效：确认 CI 是否正确挂载或注入。
- 版本号异常：确认 `-Drevision` 是否正确传递。
- 并行构建失败：尝试关闭并行或减少线程数。

# 10. 实战案例：CI 构建模板

目标：在 CI 中完成构建、测试、发布。

构建命令：
```bash
mvn -B -V -DskipTests clean package
mvn -B -DskipTests deploy
```

建议：
- 预先缓存 `~/.m2/repository`。
- 注入统一的 `settings.xml`。
- 失败后保留 `target/surefire-reports` 便于排查。
