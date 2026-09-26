
# 1. 什么是Freestyle Job

Freestyle Job 是 Jenkins 最基础的 Job 类型，通过 Web UI 以填空和点选的方式配置构建流程。虽然 Jenkins 2.x 推荐使用 Pipeline，但 Freestyle Job 仍有其适用场景：

- **快速原型**：简单任务几分钟内搭建完成
- **运维人员**：不熟悉代码的运维人员可通过 UI 配置
- **上下游简单任务**：如定时执行脚本、文件同步等

# 2. General配置

## 2.1 基本设置

```yaml
# Job 基本配置
General:
  - 名称: my-freestyle-job
  - 描述: 这是一个示例 Freestyle Job
  - 丢弃旧的构建:
    - Strategy: Log Rotation
    - Days to keep builds: 30
    - Max # of builds to keep: 100
    - Days to keep artifacts: 7
    - Artifacts max size (MB): 500
  - 禁用项目: false
  - 参数化构建: true
    - 参数列表: ...
  - 关闭并发构建: false  # 阻止同一 Job 同时执行多次
  - 在必要时执行并发构建: false
  - 构建标签: ""
  - Throttle builds: disabled
```

## 2.2 构建丢弃策略

合理的构建保留策略可以控制磁盘空间消耗：

| 策略 | 配置 | 效果 |
|------|------|------|
| 天数保留 | `Days to keep builds: 30` | 只保留 30 天内的构建记录 |
| 数量保留 | `Max # of builds to keep: 100` | 只保留最近 100 次构建 |
| Artifact 天数 | `Days to keep artifacts: 7` | 构建产物只保留 7 天 |
| Artifact 大小 | `Artifacts max size (MB): 500` | 构建产物总大小不超过 500MB |

**注意**：如果不设置丢弃策略，构建记录会无限积累，最终填满磁盘。

## 2.3 参数化构建

Freestyle Job 支持多种参数类型：

```
参数化构建过程：
┌──────────────────────────────────────────┐
│  用户触发构建 → 弹出参数填写界面          │
│  ├── String: BRANCH = main               │
│  ├── Choice: ENV = [dev, test, prod]     │
│  ├── Boolean: SKIP_TESTS = false         │
│  └── Password: DB_PASS = ******         │
│  点击 Build → 参数传入构建过程             │
└──────────────────────────────────────────┘
```

# 3. 源码管理（Source Code Management）

## 3.1 Git 配置

Freestyle Job 最常用的 SCM 是 Git。

```yaml
# Git 配置
Source Code Management: Git
  - Repository URL: https://github.com/myorg/myrepo.git
  - Credentials: github-token (凭据 ID)
  - Branches to build:
    - Branch Specifier: */main  # 或 */feature-*, */release/*
  - Repository browser: githubweb
    - URL: https://github.com/myorg/myrepo
  - Additional Behaviours:
    - Check out to specific local branch: main
    - Clean before checkout: true
    - Prune stale remote-tracking branches: true
    - Shallow clone: depth=1  # 深度克隆，减少拉取时间
    - Fetch tags: false
    - Advanced clone behaviours:
      - Clone timeout: 10 (minutes)
      - Reference repository: /opt/git/reference.git
    - Sparse Checkout paths: src/  # 仅检出特定目录
    - Submodule: Recursively update submodules
```

**Git 配置优化**：

```yaml
# 大型仓库优化策略
# 1. 浅克隆（Shallow Clone）：只拉取最近 N 次提交
Shallow clone: depth=1
# 注意：depth=1 会丢失历史，某些分析工具可能异常

# 2. 稀疏检出（Sparse Checkout）：只检出需要的目录
Sparse Checkout paths:
  - src/main/java/
  - pom.xml

# 3. 引用仓库（Reference Repository）：共享对象库
# 先在本地创建引用仓库
git clone --bare https://github.com/myorg/myrepo.git /opt/git/reference.git
# Jenkins 配置 Reference repository: /opt/git/reference.git
# 定期更新引用仓库
cd /opt/git/reference.git && git fetch --all

# 4. 禁用标签获取
Fetch tags: false
```

## 3.2 多 SCM 配置（需要 Multiscm 插件）

```yaml
# 同时检出多个仓库
Source Code Management: Multiple SCMs
  - Repository 1: Git
    URL: https://github.com/myorg/app.git
    Branches: */main
    Local subdirectory for repo (optional): app
  - Repository 2: Git
    URL: https://github.com/myorg/common-lib.git
    Branches: */main
    Local subdirectory for repo (optional): common
```

# 4. 构建触发器（Build Triggers）

## 4.1 触发器配置

```yaml
Build Triggers:
  # 方法一：其他项目构建后触发
  - Build after other projects are built:
    - Projects to watch: upstream-job
    - Trigger when: Build is stable or unstable
    - Trigger even if the build is: Not triggered

  # 方法二：定时构建
  - Build periodically:
    - Schedule: H 2 * * *  # 每天凌晨 2 点执行

  # 方法三：SCM 轮询
  - Poll SCM:
    - Schedule: H/5 * * * *  # 每 5 分钟检查 SCM 变更

  # 方法四：GitHub/GitLab Webhook
  - GitHub hook trigger for GITScm polling
  - GitLab CI Service Webhook
```

## 4.2 触发器的选择原则

| 场景 | 推荐触发方式 | 原因 |
|------|-------------|------|
| 持续集成 | Webhook | 实时触发，资源消耗小 |
| 夜间构建 | 定时触发 | 在低负载时段执行耗时构建 |
| 上游依赖 | 项目依赖触发 | 保持构建顺序 |
| 历史迁移 | SCM 轮询 | Webhook 不可用时的备选 |
| 发布流程 | 手动触发+参数 | 需要人工确认版本号等信息 |

# 5. 构建环境（Build Environment）

```yaml
Build Environment:
  # 删除工作空间（构建前）
  - Delete workspace before build starts

  # 如果失败则中止构建
  - Abort the build if it's stuck
    - Timeout: 30 minutes

  # 添加时间戳
  - Add timestamps to the Console Output

  # 设置构建名称
  - Set Build Name
    - Set Build Name: ${BUILD_NUMBER}-${BRANCH}

  # 注入密码（隐藏日志中的密码）
  - Mask passwords and regions

  # SSH Agent
  - SSH Agent
    - Credentials: gitlab-ssh-key

  # 绑定凭据
  - Use secret text(s) or file(s)
    - Binding: SONAR_TOKEN
    - Credential: sonar-token
```

# 6. 构建步骤（Build Steps）

## 6.1 构建步骤类型

```yaml
Build Steps:
  # 执行 Shell 命令（Linux）
  - Execute shell:
    command: |
      echo "Building branch ${BRANCH}..."
      ./mvnw clean package -DskipTests=${SKIP_TESTS}
      if [ -f target/*.jar ]; then
        echo "Build success"
      else
        echo "Build failed"
        exit 1
      fi

  # 执行 Windows 批处理
  - Execute Windows batch command:
    command: |
      echo Building branch %BRANCH%...
      mvnw.cmd clean package -DskipTests=%SKIP_TESTS%

  # 使用 Maven 构建
  - Invoke top-level Maven targets:
    - Maven Version: Maven3
    - Goals: clean package -DskipTests=${SKIP_TESTS}
    - POM: pom.xml
    - JVM Options: -Xmx1024m
    - Properties: -Dmaven.test.failure.ignore=false

  # 使用 Gradle 构建
  - Invoke Gradle script:
    - Gradle Version: Gradle8
    - Tasks: clean build
    - Build File: build.gradle
    - Switches: --no-daemon -x test
    - Root Build Script: false

  # 执行 Ant 构建
  - Invoke Ant:
    - Targets: clean dist
    - Build File: build.xml

  # 执行 Shell 脚本（来自 Jenkins 节点文件系统）
  - Execute shell script on remote host (using SSH)
    - SSH site: dev-server-1
    - Command: /opt/deploy/deploy.sh
    - Exec in pty: true
```

## 6.2 常用构建步骤示例

**Java Maven 项目构建**：

```bash
# Execute shell
set -e  # 任何错误立即退出

# 编译
./mvnw clean compile

# 单元测试
./mvnw test

# 打包
./mvnw package -DskipTests=false

# 生成 JaCoCo 覆盖率报告（可选）
./mvnw jacoco:report

# 生成代码质量报告（如果需要）
./mvnw checkstyle:checkstyle pmd:pmd
```

**Node.js 前端构建**：

```bash
# Execute shell
set -e

# 安装依赖
npm ci  # 使用 package-lock.json 保证一致性

# 代码检查
npm run lint

# 单元测试
npm run test -- --coverage

# 构建
npm run build

# 压缩产物
tar -czf dist.tar.gz dist/
```

**Python 项目构建**：

```bash
# Execute shell
set -e

# 创建虚拟环境
python3 -m venv venv
source venv/bin/activate

# 安装依赖
pip install -r requirements.txt -r requirements-dev.txt

# 代码检查
flake8 src/
pylint src/

# 测试
pytest tests/ --junitxml=reports/junit.xml --cov=src/

# 打包
python setup.py sdist bdist_wheel
```

# 7. 后构建操作（Post-build Actions）

## 7.1 后构建操作类型

```yaml
Post-build Actions:
  # 归档构建产物
  - Archive the artifacts:
    - Files to archive: target/*.jar, target/*.war
    - Exclude: target/*-sources.jar
    - Only archive if build is successful: true

  # 发布 JUnit 测试报告
  - Publish JUnit test result report:
    - Test report XMLs: target/surefire-reports/*.xml
    - Retain long standard output/error: true
    - Health report amplification: 1.0

  # 收集 JaCoCo 覆盖率报告
  - Record JaCoCo coverage report:
    - Path to exec files: target/jacoco.exec
    - Path to class directories: target/classes
    - Path to source directories: src/main/java

  # 发布 HTML 报告
  - Publish HTML reports:
    - HTML directory to archive: target/site
    - Index page: index.html
    - Report title: Site Report

  # 邮件通知
  - Editable Email Notification:
    - Project Recipient List: dev-team@company.com
    - Project Reply-To List: build-master@company.com
    - Content Type: HTML (text/html)
    - Default Subject: $PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!
    - Default Content: ${JELLY_SCRIPT,template="html"}
    - Attach Build Log: true (只对失败构建)
    - Send to: Developers (触发构建的用户)
    - Triggers:
      - Failure
      - Unstable
      - Success
      - Fixed (第一次从失败变成功)

  # 触发其他项目
  - Build other projects:
    - Projects to build: downstream-job
    - Trigger only if build is: stable
    - Block until the triggered projects finish their builds: false
    - Also trigger if build is: unstable

  # Slack 通知
  - Slack Notifications:
    - Team Domain: company
    - Integration Token: ****
    - Project Channel: #ci-builds
    - Custom Message: Build ${BUILD_NUMBER} ${BUILD_STATUS}
    - Notify:
      - Build Start: false
      - Build Success: true
      - Build Failure: true
      - Build Unstable: true
      - Back to Normal: true

  # 发布到 SSH 服务器
  - Publish Over SSH:
    - SSH Server: dev-server
    - Transfers:
      - Source files: target/*.jar
      - Remove prefix: target/
      - Remote directory: /opt/app/deploy/
      - Exec command: /opt/app/deploy.sh

  # 更新下游产品的状态
  - Git Publisher:
    - Tags:
      - Tag to push: v${BUILD_NUMBER}
      - Target remote: origin
    - Push Only If Build Succeeds: true
```

## 7.2 邮件通知模板

HTML 邮件模板（Jenkins Jelly 模板）：

```html
<html>
<body>
<h2>${PROJECT_NAME} - Build #${BUILD_NUMBER} - ${BUILD_STATUS}</h2>
<hr/>
<ul>
    <li><b>项目名称：</b>${PROJECT_NAME}</li>
    <li><b>构建编号：</b>#${BUILD_NUMBER}</li>
    <li><b>构建状态：</b>${BUILD_STATUS}</li>
    <li><b>持续时间：</b>${BUILD_DURATION}</li>
    <li><b>构建 URL：</b><a href="${BUILD_URL}">${BUILD_URL}</a></li>
</ul>

<h3>变更日志</h3>
${CHANGES, showPaths=true, format="%a: %m%n"}

<h3>测试结果</h3>
${TEST_COUNTS, var="total"} 个测试，
${TEST_COUNTS, var="fail"} 个失败，
${TEST_COUNTS, var="skip"} 个跳过

${FAILED_TESTS}

<hr/>
<em>自动发送自 Jenkins CI 系统</em>
</body>
</html>
```

# 8. Freestyle Job 完整示例

## 8.1 Java 微服务构建

```yaml
# Java Maven 微服务 Freestyle Job 配置
General:
  - 丢弃旧的构建:
    - Days to keep: 30
    - Max # of builds: 200
  - 参数化构建:
    - String: BRANCH=main
    - Choice: ENV=[dev, test, prod]
    - Boolean: RUN_TESTS=true
    - String: VERSION=1.0.0-${BUILD_NUMBER}

Source Code Management: Git
  - Repository: https://github.com/myorg/user-service.git
  - Credentials: github-credentials
  - Branches: */${BRANCH}
  - Clean before checkout: true
  - Shallow clone: depth=5

Build Triggers:
  - GitHub hook trigger for GITScm polling
  - Poll SCM: H/5 * * * *

Build Environment:
  - Delete workspace before build starts
  - Add timestamps to Console Output

Build Steps:
  - Execute shell:
    command: |
      echo "Building version: ${VERSION}"
      echo "Environment: ${ENV}"

      # Maven 构建
      ./mvnw clean package \
        -DskipTests=${SKIP_TESTS} \
        -Drevision=${VERSION} \
        -P${ENV}

      # 生成 Docker 镜像
      docker build \
        --build-arg JAR_FILE=target/*.jar \
        -t registry.company.com/user-service:${VERSION} \
        .

      # 推送镜像
      docker push registry.company.com/user-service:${VERSION}

Post-build Actions:
  - Archive the artifacts: target/*.jar
  - Publish JUnit test result:
    - Test report XMLs: target/surefire-reports/*.xml
  - Editable Email Notification:
    - Recipients: dev-team@company.com
    - Triggers: Failure, Unstable, Fixed
  - Slack Notifications:
    - Channel: #ci-builds
    - Triggers: Failure, Back to Normal
```

## 8.2 Freestyle Job vs Pipeline

| 对比项 | Freestyle Job | Pipeline |
|--------|-------------|----------|
| 配置方式 | Web UI 表单 | Jenkinsfile（代码） |
| 版本控制 | 配置存储在 Jenkins 中 | Jenkinsfile 可纳入 Git |
| 复杂度 | 适合简单流程 | 支持复杂逻辑 |
| 可重用性 | 低（跨 Job 复制配置） | 高（共享库、模板） |
| 代码审查 | 不支持 | 支持 PR/MR 审查 Pipeline 变更 |
| 条件逻辑 | 受限（需插件） | 原生支持（when, script） |
| 并行执行 | 需插件 | 原生支持（parallel） |
| 可视化 | 列表视图 | Blue Ocean + Stage View |
| 调试 | 日志查看 | 重放、断点（通过插件） |

**迁移建议**：
- 如果 Freestyle Job 中的 Shell 脚本超过 50 行，建议迁移到 Pipeline
- 如果涉及条件分支、并行、循环等逻辑，使用 Pipeline
- 如果仅执行简单的定时任务，Freestyle 仍然适用
