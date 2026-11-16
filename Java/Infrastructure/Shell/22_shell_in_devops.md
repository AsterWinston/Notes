
# 1. CI/CD 流水线脚本

## 1.1 Jenkins Pipeline 中的 Shell

Jenkins Pipeline 使用 `sh` 步骤执行 Shell 脚本。由于 Jenkins 默认使用 `sh -xe` 模式，任何命令的非零退出码都会导致流水线失败。

### 1.1.1 基本语法

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh '''
                    echo "Building project..."
                    ./gradlew clean build
                '''
            }
        }
    }
}
```

### 1.1.2 退出码控制

```groovy
// 方式一：使用 returnStatus 捕获退出码而不中断流水线
def exitCode = sh(script: './risky_script.sh', returnStatus: true)
if (exitCode != 0) {
    echo "Script failed with exit code ${exitCode}, continuing..."
}

// 方式二：返回标准输出
def output = sh(script: 'git describe --tags', returnStdout: true).trim()

// 方式三：显式设置 set +e 容忍失败
sh '''
    set +e
    ./optional_task.sh
    EXIT_CODE=$?
    set -e
    echo "Exit code: ${EXIT_CODE}"
'''
```

### 1.1.3 环境变量传递

```groovy
pipeline {
    environment {
        APP_VERSION = sh(script: 'git describe --tags', returnStdout: true).trim()
        BUILD_TIME  = sh(script: "date '+%Y-%m-%d %H:%M:%S'", returnStdout: true).trim()
    }
    stages {
        stage('Deploy') {
            steps {
                sh '''
                    echo "Deploying version ${APP_VERSION} at ${BUILD_TIME}"
                    docker build -t myapp:${APP_VERSION} .
                '''
            }
        }
    }
}
```

### 1.1.4 高级模式：条件执行

```groovy
stage('Test') {
    steps {
        script {
            if (env.BRANCH_NAME == 'main') {
                sh './run_full_tests.sh'
            } else {
                sh './run_quick_tests.sh'
            }
        }
    }
}
```

## 1.2 GitHub Actions 中的 Shell

GitHub Actions 通过 `run` 字段执行 Shell。Ubuntu/macOS 默认 Shell 为 `bash`，Windows 为 `powershell`。可在 workflow 或 job 级别通过 `defaults.run.shell` 统一配置。

### 1.2.1 基本用法

```yaml
name: CI Pipeline
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    defaults:
      run:
        shell: bash
    steps:
      - uses: actions/checkout@v4
      - name: Build and Test
        run: |
          set -euo pipefail
          echo "Building project..."
          mvn clean verify -B
```

### 1.2.2 多 Shell 与自定义 Shell

```yaml
jobs:
  multi-shell:
    runs-on: ubuntu-latest
    steps:
      - name: Bash step
        shell: bash {0}        # {0} 表示脚本作为位置参数传递
        run: |
          echo "Running in Bash $BASH_VERSION"

      - name: Python inline
        shell: python {0}
        run: |
          import json
          data = {"status": "ok", "env": "production"}
          print(json.dumps(data))

      - name: Custom shell (heredoc)
        shell: bash
        run: |
          cat << 'EOF' > config.json
          {
            "db": {
              "host": "${{ secrets.DB_HOST }}",
              "port": 3306
            }
          }
          EOF
```

### 1.2.3 退出码与步骤控制

```yaml
- name: Optional step
  continue-on-error: true      # 即使失败也继续
  run: ./may_fail.sh

- name: Conditional on failure
  if: failure()
  run: echo "Previous step failed, running cleanup..."

- name: Capture exit code
  run: |
    ./check_health.sh
    echo "EXIT_CODE=$?" >> $GITHUB_ENV

- name: Use captured code
  run: |
    if [[ "$EXIT_CODE" -ne 0 ]]; then
      echo "Health check failed"
    fi
```

## 1.3 GitLab CI 中的 Shell

GitLab CI 使用 `script` 关键字定义 Shell 命令。可以在 `.gitlab-ci.yml` 中通过 `before_script` 和 `after_script` 做全局处理。

### 1.3.1 基本结构

```yaml
stages:
  - build
  - test
  - deploy

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"

cache:
  paths:
    - .m2/repository

build:
  stage: build
  script:
    - echo "Building ${CI_PROJECT_NAME}..."
    - mvn compile
  artifacts:
    paths:
      - target/*.jar

test:
  stage: test
  script:
    - mvn test
    - |
      if [ -f target/surefire-reports/TEST-*.xml ]; then
        echo "Test reports generated"
      else
        echo "No test reports found!" >&2
        exit 1
      fi
```

### 1.3.2 before_script / after_script

```yaml
default:
  before_script:
    - |
      if ! command -v kubectl &> /dev/null; then
        curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
        chmod +x kubectl
        mv kubectl /usr/local/bin/
      fi
    - export DEPLOY_VERSION="${CI_COMMIT_TAG:-${CI_COMMIT_SHORT_SHA}}"
  after_script:
    - echo "Job finished with exit code ${CI_JOB_STATUS}"
    - |
      if [ "${CI_JOB_STATUS}" != "success" ]; then
        curl -X POST "$ALERT_WEBHOOK" -d "{\"text\":\"Job ${CI_JOB_NAME} failed\"}"
      fi
```

### 1.3.3 退出码与 failure 判定

```yaml
# GitLab 默认 script 中任何非零退出码都标记为失败
# 特殊场景：手动控制失败
check:
  script:
    - set +e
    - ./retry_script.sh
    - RETRY_EXIT=$?
    - set -e
    - |
      if [ $RETRY_EXIT -ne 0 ]; then
        echo "Retry script failed, attempting fallback..."
        ./fallback_script.sh
      fi
```

## 1.4 CI/CD Shell 最佳实践

| 原则 | 说明 | 示例 |
|------|------|------|
| 尽早失败 | 使用 `set -euo pipefail` | `set -euo pipefail` |
| 可观测性 | 打印关键变量和执行上下文 | `echo "Building version=${VERSION}"` |
| 幂等设计 | 同一脚本多次执行结果一致 | 用 `mkdir -p` 而非 `mkdir` |
| 敏感信息 | 使用 Secrets 而非硬编码 | `${{ secrets.DB_PASSWORD }}` |
| 超时保护 | 避免挂起操作 | `timeout 300 ./long_task.sh` |
| 日志分离 | stdout 与 stderr 分离重定向 | `./task.sh 2>error.log` |

---

# 2. Dockerfile 中的 Shell 技巧

## 2.1 SHELL 指令

Dockerfile 默认在 Linux 上使用 `/bin/sh -c`，在 Windows 上使用 `cmd /S /C`。可通过 `SHELL` 指令切换，用于启用 `pipefail` 等高级特性。

```dockerfile
# 默认 Shell：/bin/sh -c
# 切换为 bash 并启用 pipefail
SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

RUN apt-get update && apt-get install -y curl
# pipefail 使得管道中任何命令失败都会导致整个 RUN 失败
RUN curl -sSf https://example.com/script.sh | bash

# 切换回 sh
SHELL ["/bin/sh", "-c"]
```

## 2.2 exec form vs shell form

```dockerfile
# Shell Form：通过 /bin/sh -c 执行，支持变量替换
CMD echo $HOME
CMD ./start.sh

# Exec Form：直接调用可执行文件，不经过 Shell
CMD ["echo", "$HOME"]          # 字面量 $HOME，不展开
CMD ["/app/start.sh"]          # 推荐：信号转发正确
ENTRYPOINT ["/entrypoint.sh"]

# 混合模式：ENTRYPOINT exec + CMD 参数
ENTRYPOINT ["/entrypoint.sh"]
CMD ["default_arg"]
```

| 特性 | Shell Form | Exec Form |
|------|-----------|-----------|
| 变量替换 | Shell 处理 | 不会展开，需手动处理 |
| 信号转发 | Shell 进程接收，子进程可能收不到 | 直接转发给 PID 1 进程 |
| 进程数 | 多一个 `/bin/sh` 进程 | 仅目标进程 |
| 适用场景 | 简单命令 | 生产容器入口 |

## 2.3 信号转发问题

```dockerfile
# 问题：Shell form 中 sh 作为 PID 1 不转发信号
CMD /app/server.sh

# 解决一：exec 替换（推荐）
CMD exec /app/server.sh

# 解决二：exec form（最推荐）
CMD ["/app/server"]

# 解决三：entrypoint 脚本中使用 exec
# entrypoint.sh
#!/bin/bash
set -euo pipefail

# 预处理逻辑
initialize_config() {
    envsubst < /etc/app/config.tmpl > /etc/app/config.yml
}

initialize_config

# exec 替换当前进程，确保应用成为 PID 1
exec "$@"
```

```dockerfile
# 配套 Dockerfile
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["/app/server"]
```

### 2.3.1 信号处理完整示例

```dockerfile
FROM alpine:3.19

RUN apk add --no-cache bash tini

COPY entrypoint.sh /
COPY app /app

# tini 作为 init 进程，负责信号转发和僵尸进程回收
# 如果不想用 tini，也可以用 docker run --init
ENTRYPOINT ["/sbin/tini", "--", "/entrypoint.sh"]
```

## 2.4 多阶段构建中的 Shell

```dockerfile
# 阶段一：构建 - 使用完整的构建工具和缓存
FROM maven:3.9-eclipse-temurin-21 AS builder
SHELL ["/bin/bash", "-c"]
WORKDIR /src
COPY pom.xml .
# 利用 Docker 层缓存：先下载依赖
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests -B

# 阶段二：运行 - 仅保留运行必要的文件
FROM eclipse-temurin:21-jre-alpine AS runtime
SHELL ["/bin/bash", "-euo", "pipefail", "-c"]
RUN apk add --no-cache curl tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone && \
    addgroup -S app && adduser -S app -G app

# 从构建阶段复制产物
COPY --from=builder --chown=app:app /src/target/*.jar /app/app.jar

# 健康检查最优化
COPY healthcheck.sh /healthcheck.sh
RUN chmod +x /healthcheck.sh

USER app
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --retries=3 CMD /healthcheck.sh
CMD ["java", "-jar", "/app/app.jar"]
```

## 2.5 健康检查脚本

```dockerfile
# healthcheck.sh —— 放在 Dockerfile 同目录
#!/bin/bash
# Docker HEALTHCHECK 通过退出码判断：
#   0 = healthy, 1 = unhealthy, 2 = reserved

# 方式一：简单 HTTP 检查
check_http() {
    curl -sf http://localhost:8080/actuator/health/liveness || exit 1
}

# 方式二：进程检查 + 端口检查
check_process() {
    if ! pgrep -x "java" > /dev/null; then
        echo "Java process not running" >&2
        exit 1
    fi
    if ! ss -tlnp | grep -q ':8080'; then
        echo "Port 8080 not listening" >&2
        exit 1
    fi
}

# 方式三：综合检查（推荐）
check_comprehensive() {
    # HTTP 端点
    local http_code
    http_code=$(curl -s -o /dev/null -w "%{http_code}" \
        --max-time 5 http://localhost:8080/actuator/health)

    if [[ "$http_code" -eq 200 ]]; then
        exit 0
    fi

    # 回退检查：日志中是否有启动完成标记
    if tail -50 /app/logs/app.log | grep -q "Started Application"; then
        exit 0
    fi

    exit 1
}

check_comprehensive
```

---

# 3. Kubernetes 中的 Shell

## 3.1 initContainer 脚本

Init Container 在主容器启动前执行，完成后必须退出。常用于数据库初始化、配置准备等工作。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      initContainers:
        # Init 1: 等待数据库就绪
        - name: wait-for-db
          image: busybox:1.36
          command:
            - /bin/sh
            - -c
            - |
              echo "Waiting for MySQL to be ready..."
              until nslookup mysql-service.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do
                echo "MySQL not ready, sleeping..."
                sleep 2
              done
              echo "MySQL is reachable"

        # Init 2: 数据库迁移
        - name: db-migrate
          image: myapp-migrate:v1.2.3
          env:
            - name: DB_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          command:
            - /bin/sh
            - -c
            - |
              set -euo pipefail
              echo "Running database migrations..."
              /app/flyway migrate \
                -url="${DB_URL}" \
                -user="${DB_USER}" \
                -password="${DB_PASSWORD}" \
                -locations="filesystem:/migrations"
              echo "Migrations complete"

        # Init 3: 预检 — 修改内核参数等（需 privileged）
        - name: sysctl
          image: busybox:1.36
          securityContext:
            privileged: true
          command:
            - /bin/sh
            - -c
            - |
              sysctl -w net.core.somaxconn=65535
              sysctl -w net.ipv4.tcp_max_syn_backlog=8192

      containers:
        - name: app
          image: myapp:v1.2.3
```

## 3.2 lifecycle hooks

Kubernetes 提供 `postStart` 和 `preStop` 生命周期钩子，支持 `exec` 和 `httpGet` 两种方式。

```yaml
containers:
  - name: app
    image: myapp:v1.2.3
    lifecycle:
      postStart:
        exec:
          command:
            - /bin/sh
            - -c
            - |
              # postStart 与 ENTRYPOINT 并行执行，无顺序保证
              # 适合：注册服务、文件初始化等
              echo "Container started, registering to service registry..."
              until curl -sf -X POST http://registry/api/v1/register \
                -H "Content-Type: application/json" \
                -d "{\"host\":\"${POD_IP}\",\"port\":8080}"; do
                sleep 1
              done
              echo "Registered successfully"

      preStop:
        exec:
          command:
            - /bin/sh
            - -c
            - |
              # preStop 在 SIGTERM 发送前执行
              # 必须在一定时间内完成（terminationGracePeriodSeconds，默认 30s）
              echo "Shutting down gracefully..."

              # 步骤 1: 从注册中心摘除
              curl -sf -X DELETE "http://registry/api/v1/deregister/${POD_IP}"

              # 步骤 2: 等待正在处理的请求完成（简化为 sleep）
              sleep 10

              # 步骤 3: 清理临时文件
              rm -f /tmp/*.pid

              echo "PreStop hook completed"
```

## 3.3 readiness / liveness probe 脚本

```yaml
containers:
  - name: app
    image: myapp:v1.2.3
    env:
      - name: APP_PORT
        value: "8080"
    ports:
      - containerPort: 8080
    readinessProbe:
      exec:
        command:
          - /bin/sh
          - -c
          - |
            # Readiness：容器是否准备好接受流量
            # 检查应用内部状态：数据库连接池、消息队列连接等
            STATUS=$(curl -sf http://localhost:${APP_PORT}/actuator/health/readiness | \
                     jq -r '.status')
            test "$STATUS" = "UP"
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3

    livenessProbe:
      exec:
        command:
          - /bin/sh
          - -c
          - |
            # Liveness：容器是否需要重启
            # 简单检查：进程是否还在运行并响应
            STATUS=$(curl -sf http://localhost:${APP_PORT}/actuator/health/liveness | \
                     jq -r '.status')

            # 额外检查：死锁检测
            if [[ -f /tmp/deadlock.detected ]]; then
              echo "Deadlock detected!" >&2
              exit 1
            fi

            test "$STATUS" = "UP"
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
```

## 3.4 高级探针脚本

```yaml
# 探针脚本放到 ConfigMap 中统一管理
apiVersion: v1
kind: ConfigMap
metadata:
  name: probe-scripts
data:
  readiness.sh: |
    #!/bin/bash
    set -euo pipefail

    # 检查管理端口（独立于业务端口）
    HEALTH_URL="http://localhost:${MANAGEMENT_PORT:-8081}/health/ready"
    RESPONSE=$(curl -sf --max-time 3 "$HEALTH_URL" 2>/dev/null) || exit 1

    # 解析 JSON 健康详情
    DB_STATUS=$(echo "$RESPONSE" | jq -r '.components.db.status // "UNKNOWN"')
    MQ_STATUS=$(echo "$RESPONSE" | jq -r '.components.rabbit.status // "UNKNOWN"')

    echo "DB: ${DB_STATUS}, MQ: ${MQ_STATUS}"

    if [[ "$DB_STATUS" != "UP" ]] || [[ "$MQ_STATUS" != "UP" ]]; then
        echo "Dependency not ready: DB=${DB_STATUS}, MQ=${MQ_STATUS}" >&2
        exit 1
    fi

    exit 0

  liveness.sh: |
    #!/bin/bash
    set -euo pipefail

    PID_FILE="/tmp/app.pid"
    if [[ ! -f "$PID_FILE" ]]; then
        echo "PID file not found" >&2
        exit 1
    fi

    PID=$(cat "$PID_FILE")
    if ! kill -0 "$PID" 2>/dev/null; then
        echo "Process ${PID} is dead" >&2
        exit 1
    fi

    # 检查进程是否占用过多内存
    RSS_MB=$(ps -o rss= -p "$PID" 2>/dev/null | awk '{print int($1/1024)}')
    MAX_MB=2048
    if [[ "$RSS_MB" -gt "$MAX_MB" ]]; then
        echo "Memory usage ${RSS_MB}MB exceeds ${MAX_MB}MB limit" >&2
        exit 1
    fi

    exit 0
```

使用 ConfigMap 挂载探针脚本：

```yaml
containers:
  - name: app
    image: myapp:v1.2.3
    volumeMounts:
      - name: probe-scripts
        mountPath: /probes
        readOnly: true
    livenessProbe:
      exec:
        command: ["/bin/bash", "/probes/liveness.sh"]
    readinessProbe:
      exec:
        command: ["/bin/bash", "/probes/readiness.sh"]
  volumes:
    - name: probe-scripts
      configMap:
        name: probe-scripts
        defaultMode: 0555
```

---

# 4. Makefile 与 Shell

## 4.1 基础 Shell 使用

Makefile 中每行配方（recipe）都在独立的子 Shell 中执行，因此变量赋值在行间不共享。

```makefile
SHELL := /bin/bash

.PHONY: build test deploy

# 问题示例：多行之间变量不共享
broken:
	cd /tmp
	pwd          # 输出的是当前目录，而非 /tmp（因为每行是独立 Shell）

# 解决方案一：单行连接
fixed-one-line:
	cd /tmp && pwd

# 解决方案二：.ONESHELL 指令
.ONESHELL:
fixed-oneshell:
	cd /tmp
	pwd              # 正确输出 /tmp
	echo "Current: $$(pwd)"
```

## 4.2 变量传递

```makefile
# Make 变量与 Shell 变量是两个不同的命名空间

# Make 变量
APP_NAME  := myapp
VERSION   := $(shell git describe --tags --always)
BUILD_TIME := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)

# 传递 Make 变量给 Shell：单美元符作为前缀
show-vars:
	@echo "Application: $(APP_NAME)"
	@echo "Version:     $(VERSION)"
	@echo "Build time:  $(BUILD_TIME)"

# Shell 变量在 Make 中需要用 $$ 转义
# Make 将 $$ 替换为 $，Shell 才能看到 $ 符号
list-files:
	@for file in *.go; do \
		echo "File: $$file"; \
	done

# 完整示例：带错误处理的构建
build:
	@set -euo pipefail; \
	LDFLAGS="-X main.version=$(VERSION) -X main.buildTime=$(BUILD_TIME)"; \
	echo "Building $(APP_NAME) version $(VERSION)..."; \
	go build -ldflags "$$LDFLAGS" -o bin/$(APP_NAME) .; \
	echo "Build successful"
```

## 4.3 .ONESHELL 与 .SHELLFLAGS

```makefile
SHELL := /bin/bash

# .ONESHELL 使整个配方在同一个 Shell 实例中执行
.ONESHELL:

# .SHELLFLAGS 传递给 Shell 的参数
# -e: 命令失败则退出
# -o pipefail: 管道中任何命令失败则退出
.SHELLFLAGS := -euo pipefail -c

deploy:
	@echo "=== Deploying $(APP_NAME) ==="

	# 预检查
	if [[ ! -f "bin/$(APP_NAME)" ]]; then
		echo "ERROR: Binary not found, run make build first" >&2
		exit 1
	fi

	# 构建 Docker 镜像
	docker build -t $(APP_NAME):$(VERSION) .
	docker tag $(APP_NAME):$(VERSION) $(APP_NAME):latest

	# 推送镜像
	docker push $(APP_NAME):$(VERSION)
	docker push $(APP_NAME):latest

	# K8s 滚动更新
	kubectl set image deployment/$(APP_NAME) $(APP_NAME)=$(APP_NAME):$(VERSION)
	kubectl rollout status deployment/$(APP_NAME) --timeout=5m

	@echo "=== Deploy complete ==="
```

## 4.4 错误处理模式

```makefile
SHELL := /bin/bash

# 模式一：忽略特定命令错误
check-docker:
	-@docker info > /dev/null 2>&1 || \
		{ echo "Docker not running, please start Docker daemon"; exit 1; }

# 模式二：带清理的构建
build: clean
	@echo "Building..."
	@mkdir -p build
	go build -o build/app .
	@trap 'rm -f build/*.tmp' EXIT; \
	cp build/app build/app.tmp; \
	# 若构建失败，清理中间产物已在 trap 中定义

# 模式三：条件目标
.PHONY: test-unit test-all
test: test-unit test-all

test-unit:
	@echo "Running unit tests..."
	@go test -short -count=1 ./... || { echo "Unit tests FAILED" >&2; exit 1; }

test-all:
	@echo "Running all tests..."
	@go test -v -count=1 -race ./...

# 模式四：重试机制
migrate-db:
	@for i in 1 2 3 4 5; do \
		echo "Attempt $$i: running migration..."; \
		./scripts/migrate.sh && break || \
			{ echo "Attempt $$i failed"; if [ $$i -eq 5 ]; then exit 1; fi; } \
		sleep 2; \
	done
```

## 4.5 Makefile 调用外部 Shell 脚本

```makefile
.PHONY: setup dev prod

# Makefile 负责编排调用，复杂逻辑放在独立脚本中
setup:
	@bash scripts/setup.sh

dev:
	@ENV=development bash scripts/run.sh

prod:
	@ENV=production bash scripts/run.sh

# scripts/run.sh 示例
# #!/bin/bash
# set -euo pipefail
# ENV="${ENV:-development}"
# CONFIG="config/${ENV}.yaml"
# if [[ ! -f "$CONFIG" ]]; then
#     echo "Config file $CONFIG not found" >&2
#     exit 1
# fi
# ./app --config="$CONFIG"
```

---

# 5. 一次性运维脚本设计原则

## 5.1 幂等性

同一脚本多次执行结果相同，不会产生副作用。

```bash
#!/bin/bash
set -euo pipefail

# 幂等原则：
# 1. 检查状态再行动
# 2. 使用 mkdir -p 而非 mkdir
# 3. 使用 CREATE IF NOT EXISTS
# 4. 检查文件是否已存在再创建

# 错误示例：非幂等
mkdir /etc/myapp              # 第二次执行报错
echo "config" > /etc/myapp/config.yml  # 每次覆盖

# 正确示例：幂等
ensure_directory() {
    local dir="$1"
    if [[ ! -d "$dir" ]]; then
        mkdir -p "$dir"
        echo "Created directory: $dir"
    else
        echo "Directory already exists: $dir"
    fi
}

ensure_user() {
    local user="$1"
    if ! id "$user" &>/dev/null; then
        useradd -r -s /bin/false "$user"
        echo "Created user: $user"
    else
        echo "User already exists: $user"
    fi
}

ensure_package() {
    local pkg="$1"
    if ! dpkg -l "$pkg" &>/dev/null; then
        apt-get install -y "$pkg"
        echo "Installed package: $pkg"
    else
        echo "Package already installed: $pkg"
    fi
}

# 数据库幂等操作
ensure_database() {
    local db_name="$1"
    if ! mysql -u root -e "SELECT 1" "$db_name" &>/dev/null 2>&1; then
        mysql -u root -e "CREATE DATABASE IF NOT EXISTS ${db_name}"
        echo "Created database: $db_name"
    else
        echo "Database already exists: $db_name"
    fi
}

ensure_directory "/etc/myapp"
ensure_user "myapp"
ensure_package "nginx"
ensure_database "myapp_prod"
```

## 5.2 可重入性（Reentrant）

脚本从中断点继续执行，而非从头开始。

```bash
#!/bin/bash
set -euo pipefail

# 可重入设计：基于步骤标记文件
STATE_DIR="/var/lib/myapp/setup_state"
mkdir -p "$STATE_DIR"

step_done() {
    local step="$1"
    touch "$STATE_DIR/.step_${step}"
    echo "[OK] Step $step completed"
}

is_step_done() {
    local step="$1"
    test -f "$STATE_DIR/.step_${step}"
}

run_step() {
    local step="$1"
    local description="$2"
    shift 2

    if is_step_done "$step"; then
        echo "[SKIP] Step $step ($description) already done"
        return 0
    fi

    echo "[RUN]  Step $step: $description"
    if "$@"; then
        step_done "$step"
        return 0
    else
        echo "[FAIL] Step $step: $description" >&2
        return 1
    fi
}

# 使用示例
run_step 01 "System update"          apt-get update
run_step 02 "Install dependencies"   apt-get install -y curl jq
run_step 03 "Create directories"     ensure_directory /opt/myapp
run_step 04 "Database migration"     /opt/myapp/bin/migrate.sh
run_step 05 "Start service"          systemctl start myapp

echo "All steps completed. To re-run, delete: rm -rf $STATE_DIR"
```

## 5.3 回滚机制

```bash
#!/bin/bash
set -euo pipefail

ROLLBACK_FILE="/tmp/myapp_rollback_$(date +%Y%m%d_%H%M%S).sh"

# 注册回滚操作
register_rollback() {
    echo "$1" >> "$ROLLBACK_FILE"
}

# 执行回滚
do_rollback() {
    echo "!!! ROLLBACK !!!"
    if [[ -f "$ROLLBACK_FILE" ]]; then
        # 逆序执行回滚操作
        tac "$ROLLBACK_FILE" | bash -ex
        rm -f "$ROLLBACK_FILE"
    fi
    exit 1
}

# 主逻辑
main() {
    # 优雅回滚：信号触发
    trap do_rollback ERR INT TERM

    # 步骤 1: 备份旧配置
    if [[ -f /etc/myapp/config.yml ]]; then
        cp /etc/myapp/config.yml "/etc/myapp/config.yml.bak.$(date +%s)"
        register_rollback "cp /etc/myapp/config.yml.bak.$(date +%s) /etc/myapp/config.yml"
    fi

    # 步骤 2: 部署新版本
    if [[ -f /opt/myapp/app.jar ]]; then
        cp /opt/myapp/app.jar "/opt/myapp/app.jar.bak.$(date +%s)"
        register_rollback "mv /opt/myapp/app.jar.bak.$(date +%s) /opt/myapp/app.jar"
    fi
    cp /tmp/deploy/app.jar /opt/myapp/app.jar

    # 步骤 3: 重启服务
    systemctl restart myapp
    register_rollback "systemctl restart myapp"

    # 步骤 4: 健康检查
    sleep 5
    if ! curl -sf http://localhost:8080/health; then
        echo "Health check failed, rolling back..."
        do_rollback
    fi

    # 成功——清除 rollback 文件
    rm -f "$ROLLBACK_FILE"
    echo "Deployment successful"
}

main "$@"
```

## 5.4 预检查（Pre-check）

```bash
#!/bin/bash
set -euo pipefail

# 预检查：在实际操作前验证所有条件
precheck() {
    local errors=0

    echo "=== Pre-check ==="

    # 检查当前用户
    if [[ "$(id -u)" -ne 0 ]]; then
        echo "[ERROR] Must run as root" >&2
        ((errors++))
    fi

    # 检查操作系统
    if [[ ! -f /etc/os-release ]] || ! grep -q 'Ubuntu' /etc/os-release; then
        echo "[ERROR] Only Ubuntu is supported" >&2
        ((errors++))
    fi

    # 检查磁盘空间（至少 10GB）
    local available_gb
    available_gb=$(df -BG /opt | awk 'NR==2 {print $4}' | tr -d 'G')
    if [[ "$available_gb" -lt 10 ]]; then
        echo "[ERROR] Insufficient disk space: ${available_gb}GB available, need 10GB" >&2
        ((errors++))
    fi

    # 检查内存（至少 2GB）
    local mem_mb
    mem_mb=$(free -m | awk 'NR==2 {print $7}')
    if [[ "$mem_mb" -lt 2048 ]]; then
        echo "[ERROR] Insufficient memory: ${mem_mb}MB available, need 2048MB" >&2
        ((errors++))
    fi

    # 检查端口占用
    for port in 8080 8443 9090; do
        if ss -tlnp | grep -q ":${port} "; then
            echo "[ERROR] Port ${port} is already in use" >&2
            ((errors++))
        fi
    done

    # 检查必需命令
    local required_cmds=("curl" "jq" "docker" "kubectl")
    for cmd in "${required_cmds[@]}"; do
        if ! command -v "$cmd" &>/dev/null; then
            echo "[ERROR] Required command not found: $cmd" >&2
            ((errors++))
        fi
    done

    if [[ "$errors" -gt 0 ]]; then
        echo "[ERROR] Pre-check failed with ${errors} error(s)" >&2
        exit 1
    fi

    echo "[OK] Pre-check passed"
}

precheck
```

## 5.5 dry-run 模式

```bash
#!/bin/bash
set -euo pipefail

# dry-run 模式设计：
# 1. 通过环境变量或命令行参数控制
# 2. 打印将要执行的操作而非真正执行
# 3. 对危险操作（删除、重启）尤为有用

DRY_RUN="${DRY_RUN:-false}"

run_or_dry() {
    local description="$1"
    shift

    if [[ "$DRY_RUN" == "true" ]]; then
        echo "[DRY-RUN] Would execute: ${description} -> $*"
        return 0
    else
        echo "[EXEC] $description"
        "$@"
    fi
}

# 更精细的 dry-run：分离危险操作
dangerous() {
    local description="$1"
    shift
    if [[ "$DRY_RUN" == "true" ]]; then
        echo "[DRY-RUN] DANGEROUS: ${description} -> $*"
        return 0
    fi
    echo "[DANGER] $description"
    read -p "Confirm? (yes/no): " CONFIRM
    if [[ "$CONFIRM" != "yes" ]]; then
        echo "Aborted"
        exit 1
    fi
    "$@"
}

# 使用
main() {
    run_or_dry "Create directory"     mkdir -p /opt/myapp
    run_or_dry "Install packages"     apt-get install -y curl
    run_or_dry "Copy config"          cp config.yml /etc/myapp/
    dangerous  "Restart service"      systemctl restart myapp

    # 带条件的 dry-run
    local new_config="/etc/myapp/config.yml.new"
    if [[ -f "$new_config" ]]; then
        run_or_dry "Apply new config" mv "$new_config" /etc/myapp/config.yml
    else
        echo "[WARN] No new config to apply"
    fi
}

# 命令行参数解析
while [[ $# -gt 0 ]]; do
    case "$1" in
        --dry-run)
            DRY_RUN=true
            shift
            ;;
        --force)
            DRY_RUN=false
            shift
            ;;
        *)
            echo "Usage: $0 [--dry-run] [--force]"
            exit 1
            ;;
    esac
done

main
```

## 5.6 一次性脚本综合模板

```bash
#!/bin/bash
# ============================================================
# 脚本：deploy_hotfix.sh
# 描述：生产环境热修复部署
# 作者：SRE Team
# 注意：此脚本设计为一次性执行，支持 --dry-run 和 --rollback
# ============================================================
set -euo pipefail

# ---- 配置 ----
readonly SCRIPT_NAME="$(basename "$0")"
readonly LOG_FILE="/var/log/${SCRIPT_NAME%.sh}_$(date +%Y%m%d_%H%M%S).log"
readonly STATE_FILE="/tmp/${SCRIPT_NAME%.sh}_state"
readonly LOCK_FILE="/var/lock/${SCRIPT_NAME%.sh}.lock"

# ---- 日志 ----
log()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }
info() { log "INFO  $*"; }
warn() { log "WARN  $*" >&2; }
error(){ log "ERROR $*" >&2; exit 1; }

# ---- 锁 ----
acquire_lock() {
    exec 9>"$LOCK_FILE"
    if ! flock -n 9; then
        error "Another instance is running. Exiting."
    fi
}

# ---- 预检查（幂等） ----
precheck() {
    info "Running precheck..."
    [[ "$(id -u)" -eq 0 ]] || error "Must run as root"
    command -v kubectl &>/dev/null || error "kubectl not found"
    info "Precheck passed"
}

# ---- 主流程（可重入） ----
deploy() {
    local dry_run="${1:-false}"

    # Step 1: 备份（幂等：仅在首次执行时备份）
    if [[ ! -f "$STATE_FILE" ]] || ! grep -q "backup" "$STATE_FILE"; then
        info "Backing up current deployment..."
        if [[ "$dry_run" == "true" ]]; then
            echo "[DRY-RUN] Would backup deployment"
        else
            kubectl get deployment myapp -o yaml > "/tmp/myapp_backup_$(date +%s).yaml"
        fi
        echo "backup" >> "$STATE_FILE"
    else
        info "Backup already done, skipping"
    fi

    # Step 2: 滚动更新（幂等：检查当前版本）
    local current_version
    current_version=$(kubectl get deployment myapp -o jsonpath='{.spec.template.spec.containers[0].image}')
    local target_version="myapp:${HOTFIX_VERSION:-latest}"

    if [[ "$current_version" == "$target_version" ]]; then
        info "Already running target version $target_version, skipping deploy"
    else
        info "Deploying $target_version (current: $current_version)..."

        if [[ "$dry_run" == "true" ]]; then
            echo "[DRY-RUN] Would update to $target_version"
        else
            kubectl set image deployment/myapp "myapp=$target_version" --record
            kubectl rollout status deployment/myapp --timeout=5m
        fi
        echo "deploy" >> "$STATE_FILE"
    fi

    # Step 3: 验证（每次执行）
    info "Running post-deploy verification..."
    if [[ "$dry_run" == "true" ]]; then
        echo "[DRY-RUN] Would run health checks"
    else
        for i in {1..6}; do
            if curl -sf http://localhost:8080/actuator/health > /dev/null; then
                info "Health check passed"
                break
            fi
            if [[ "$i" -eq 6 ]]; then
                error "Health check failed after 6 attempts"
            fi
            sleep 10
        done
    fi

    rm -f "$STATE_FILE"
    info "Deployment completed successfully"
}

# ---- 参数解析 ----
main() {
    local DRY_RUN=false

    while [[ $# -gt 0 ]]; do
        case "$1" in
            --dry-run) DRY_RUN=true; shift ;;
            -v|--version) HOTFIX_VERSION="$2"; shift 2 ;;
            -h|--help)
                echo "Usage: $SCRIPT_NAME [--dry-run] [-v VERSION]"
                exit 0
                ;;
            *) error "Unknown option: $1" ;;
        esac
    done

    acquire_lock
    precheck
    deploy "$DRY_RUN"
}

main "$@"
```

---

# 6. 基础设施即代码中的 Shell

## 6.1 Terraform local-exec

`local-exec` 在运行 Terraform 的本地机器上执行 Shell 命令，用于资源创建后的初始化。

```hcl
# 基础用法
resource "null_resource" "cluster_init" {
  provisioner "local-exec" {
    command = <<-EOT
      echo "Waiting for EKS cluster to be ready..."
      aws eks wait cluster-active --name ${var.cluster_name}
      aws eks update-kubeconfig --region ${var.region} --name ${var.cluster_name}
      echo "EKS cluster is ready and kubeconfig updated"
    EOT
  }
}

# 带环境变量
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"

  provisioner "local-exec" {
    command = "ansible-playbook -i '${self.public_ip},' playbooks/web.yml"

    environment = {
      ANSIBLE_HOST_KEY_CHECKING = "False"
      APP_VERSION               = var.app_version
    }
  }
}

# 条件执行：仅在资源变化时执行
resource "aws_ecr_repository" "app" {
  name = "myapp"

  provisioner "local-exec" {
    command = <<-EOT
      aws ecr get-login-password --region ${var.region} \
        | docker login --username AWS --password-stdin ${self.repository_url}
      docker build -t ${self.repository_url}:latest .
      docker push ${self.repository_url}:latest
    EOT

    # 仅当仓库被创建时执行
    when = create
  }
}

# 销毁时执行
resource "null_resource" "cleanup" {
  provisioner "local-exec" {
    when    = destroy
    command = "rm -f ${path.module}/generated/*.yaml"
  }
}

# 多步初始化的实际案例
resource "null_resource" "bootstrap" {
  depends_on = [aws_eks_cluster.main, aws_eks_node_group.main]

  provisioner "local-exec" {
    command = <<-SHELL
      set -euo pipefail

      # 1. 更新 kubeconfig
      aws eks update-kubeconfig \
        --region ${var.region} \
        --name ${var.cluster_name}

      # 2. 安装 Ingress Controller
      kubectl apply -f ${path.module}/manifests/ingress-nginx.yaml
      kubectl wait --namespace ingress-nginx \
        --for=condition=ready pod \
        --selector=app.kubernetes.io/component=controller \
        --timeout=300s

      # 3. 安装 cert-manager
      kubectl apply -f ${path.module}/manifests/cert-manager.yaml

      # 4. 应用应用清单
      envsubst < ${path.module}/manifests/app.yaml.tmpl | kubectl apply -f -

      # 5. 等待部署就绪
      kubectl wait --for=condition=available deployment/myapp --timeout=300s

      echo "Cluster bootstrap complete"
    SHELL
  }
}

# 复杂示例：数据库初始化
resource "aws_db_instance" "main" {
  engine         = "postgres"
  engine_version = "16"
  identifier     = "myapp-db"
  username       = var.db_username
  password       = random_password.db.result
  skip_final_snapshot = true
}

resource "null_resource" "db_init" {
  depends_on = [aws_db_instance.main, random_password.db]

  triggers = {
    # 密码变更时重新执行
    db_password = random_password.db.result
  }

  provisioner "local-exec" {
    command = <<-SHELL
      set -euo pipefail

      DB_HOST="${aws_db_instance.main.address}"
      DB_PORT="${aws_db_instance.main.port}"
      DB_USER="${var.db_username}"
      DB_PASS="${random_password.db.result}"
      PGPASSWORD="$DB_PASS"

      export PGPASSWORD

      # 等待数据库就绪
      echo "Waiting for database to be ready..."
      for i in $(seq 1 30); do
        if psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d postgres -c "SELECT 1" &>/dev/null; then
          echo "Database is ready"
          break
        fi
        [ "$i" -eq 30 ] && { echo "Timeout waiting for database"; exit 1; }
        sleep 5
      done

      # 创建数据库和扩展
      psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d postgres <<'SQL'
        CREATE DATABASE myapp;
        \c myapp
        CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
        CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";
      SQL

      # 运行迁移
      flyway -url="jdbc:postgresql://$DB_HOST:$DB_PORT/myapp" \
             -user="$DB_USER" \
             -password="$DB_PASS" \
             migrate

      unset PGPASSWORD
    SHELL
  }
}
```

## 6.2 Ansible shell 模块

```yaml
# 简单命令
- name: Check disk usage
  shell: df -h /
  register: disk_usage
  changed_when: false

- name: Alert if disk usage exceeds 80%
  debug:
    msg: "Disk usage is critical!"
  when: >
    disk_usage.stdout_lines | select('search', '/$') | list
    | first | regex_replace('.* (\\d+)%.*', '\\1') | int > 80

# 带管道和重定向
- name: Find large files
  shell: |
    find /var/log -type f -size +100M -exec ls -lh {} \; > /tmp/large_files.txt
  changed_when: false

# 多步骤 Shell
- name: Initialize application
  block:
    - name: Create directories
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - /opt/myapp
        - /etc/myapp
        - /var/log/myapp

    - name: Generate config from template
      template:
        src: app.conf.j2
        dest: /etc/myapp/app.conf

    - name: Run setup script
      shell: |
        set -euo pipefail
        # 下载依赖
        curl -sfL https://example.com/bootstrap.sh -o /tmp/bootstrap.sh
        chmod +x /tmp/bootstrap.sh
        /tmp/bootstrap.sh --role={{ app_role }}
      args:
        creates: /opt/myapp/.installed  # 幂等：已安装则跳过
        warn: false
      register: setup_result

    - name: Show setup output
      debug:
        var: setup_result.stdout_lines

# shell vs command 模块区别
- name: Shell 支持管道、变量、重定向
  shell: cat /etc/passwd | grep {{ username }}
  # 注意：{{ }} 中的变量会被 Jinja2 替换

- name: Command 不经过 Shell，更安全
  command: ls -la /opt/myapp
  # 推荐优先使用 command，仅在需要 Shell 特性时使用 shell

# 错误处理
- name: Task that may fail
  shell: /opt/myapp/risky_script.sh
  register: result
  failed_when: result.rc not in [0, 1]  # 退出码 0 或 1 都是正常的
  changed_when: "'CHANGED' in result.stdout"
  ignore_errors: true  # 结果记录但不阻塞后续任务

# 从外部文件执行脚本
- name: Execute complex script
  script: scripts/prepare_node.sh --role=worker
  args:
    creates: /opt/.node_prepared
  environment:
    K8S_VERSION: "1.29"
    CONTAINER_RUNTIME: "containerd"

# Ansible 调用结果处理
- name: Get application status
  shell: |
    curl -sf http://localhost:8080/actuator/info
  register: app_info
  changed_when: false
  retries: 5
  delay: 10
  until: app_info.rc == 0
```

## 6.3 Packer provisioner

```json
{
  "builders": [{
    "type": "amazon-ebs",
    "region": "us-east-1",
    "source_ami_filter": {
      "filters": {
        "name": "ubuntu/images/*ubuntu-jammy-22.04-amd64-server-*"
      },
      "owners": ["099720109477"],
      "most_recent": true
    },
    "instance_type": "t3.medium",
    "ssh_username": "ubuntu",
    "ami_name": "myapp-{{timestamp}}"
  }],
  "provisioners": [
    {
      "type": "shell",
      "inline": [
        "sudo apt-get update",
        "sudo apt-get install -y curl unzip",
        "curl -fsSL https://get.docker.com | sudo bash",
        "sudo usermod -aG docker ubuntu"
      ]
    },
    {
      "type": "shell",
      "script": "scripts/install_dependencies.sh",
      "environment_vars": [
        "APP_VERSION=1.2.3",
        "INSTALL_DIR=/opt/myapp"
      ]
    },
    {
      "type": "shell",
      "inline": [
        "echo 'Cleaning up...'",
        "sudo apt-get clean",
        "sudo rm -rf /var/lib/apt/lists/*",
        "sudo rm -rf /tmp/*"
      ],
      "pause_before": "5s"
    }
  ]
}
```

Packer Shell 脚本示例：

```bash
#!/bin/bash
# scripts/install_dependencies.sh —— Packer provisioner 调用的脚本
set -euo pipefail

APP_VERSION="${APP_VERSION:-latest}"
INSTALL_DIR="${INSTALL_DIR:-/opt/myapp}"

echo "Installing app version ${APP_VERSION} to ${INSTALL_DIR}"

# 系统基础依赖
apt-get update -qq
apt-get install -y -qq \
    apt-transport-https \
    ca-certificates \
    gnupg \
    lsb-release \
    wget

# 安装 CloudWatch Agent
wget -q https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
dpkg -i amazon-cloudwatch-agent.deb
rm -f amazon-cloudwatch-agent.deb

# 应用目录
mkdir -p "${INSTALL_DIR}/bin" "${INSTALL_DIR}/logs"
useradd -r -s /bin/false myapp || true

# 下载应用
curl -sfL "https://releases.example.com/myapp/${APP_VERSION}/myapp-linux-amd64.tar.gz" \
    | tar -xz -C "${INSTALL_DIR}/bin/"
chown -R myapp:myapp "${INSTALL_DIR}"

# 创建 systemd unit
cat << EOF > /etc/systemd/system/myapp.service
[Unit]
Description=MyApp Service
After=network.target

[Service]
Type=simple
User=myapp
ExecStart=${INSTALL_DIR}/bin/myapp
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable myapp

echo "Installation complete"
```

---

# 7. 环境初始化脚本

## 7.1 开发环境搭建

```bash
#!/bin/bash
# dev_setup.sh —— 新成员开发环境一键搭建
set -euo pipefail

OS="$(uname -s)"
ARCH="$(uname -m)"

log()  { echo "[$(date +%H:%M:%S)] $*"; }
error(){ echo "[ERROR] $*" >&2; exit 1; }

log "Setting up development environment on ${OS} (${ARCH})..."

# === 包管理器适配 ===
if [[ "$OS" == "Darwin" ]]; then
    PACKAGE_MANAGER="brew"
elif [[ "$OS" == "Linux" ]]; then
    if command -v apt-get &>/dev/null; then
        PACKAGE_MANAGER="apt"
    elif command -v dnf &>/dev/null; then
        PACKAGE_MANAGER="dnf"
    else
        error "Unsupported Linux distribution"
    fi
else
    error "Unsupported OS: $OS"
fi

install_pkg() {
    case "$PACKAGE_MANAGER" in
        brew) brew install "$@" ;;
        apt)  sudo apt-get install -y "$@" ;;
        dnf)  sudo dnf install -y "$@" ;;
    esac
}

# === 基础工具 ===
log "Installing base tools..."
BASE_TOOLS=("git" "curl" "jq" "wget" "vim" "tree" "htop")
for tool in "${BASE_TOOLS[@]}"; do
    if command -v "$tool" &>/dev/null; then
        log "  [SKIP] $tool already installed"
    else
        install_pkg "$tool"
        log "  [OK]   $tool installed"
    fi
done

# === 编程语言 ===
log "Setting up programming languages..."

# Java
JAVA_VERSION="${JAVA_VERSION:-21}"
if command -v java &>/dev/null; then
    log "  Java $(java -version 2>&1 | head -1)"
else
    log "  Installing Java ${JAVA_VERSION}..."
    if [[ "$PACKAGE_MANAGER" == "brew" ]]; then
        brew install "openjdk@${JAVA_VERSION}"
    else
        sudo apt-get install -y "openjdk-${JAVA_VERSION}-jdk"
    fi
fi

# Maven
if command -v mvn &>/dev/null; then
    log "  Maven $(mvn --version | head -1 | awk '{print $3}')"
else
    install_pkg maven
    log "  Maven installed"
fi

# Node.js（通过 nvm 管理）
if [[ ! -d "$HOME/.nvm" ]]; then
    log "  Installing nvm..."
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
    export NVM_DIR="$HOME/.nvm"
    [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
    nvm install --lts
    log "  Node.js $(node --version) installed via nvm"
fi

# === Docker ===
if command -v docker &>/dev/null; then
    log "Docker already installed"
else
    log "Installing Docker..."
    if [[ "$PACKAGE_MANAGER" == "brew" ]]; then
        brew install --cask docker
    else
        curl -fsSL https://get.docker.com | sudo bash
        sudo usermod -aG docker "$USER"
        log "  NOTE: Logout and login again for docker group to take effect"
    fi
fi

# === 项目配置 ===
log "Setting up project..."

# 克隆仓库（幂等）
REPO_DIR="${HOME}/workspace/myapp"
if [[ -d "$REPO_DIR" ]]; then
    log "Repository already cloned at $REPO_DIR"
else
    git clone git@github.com:myorg/myapp.git "$REPO_DIR"
    log "Repository cloned to $REPO_DIR"
fi

cd "$REPO_DIR"

# 配置 Git hooks
cp scripts/git-hooks/* .git/hooks/
chmod +x .git/hooks/*
log "Git hooks configured"

# IDE 配置（生成 .idea 或 .vscode）
mkdir -p .vscode
cat << 'EOF' > .vscode/settings.json
{
  "java.home": "/usr/lib/jvm/java-21-openjdk-amd64",
  "java.configuration.runtimes": [
    { "name": "JavaSE-21", "path": "/usr/lib/jvm/java-21-openjdk-amd64" }
  ],
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.organizeImports": true
  }
}
EOF

# 安装项目依赖
mvn dependency:resolve -q
log "Maven dependencies resolved"

# 环境变量
if ! grep -q "MYAPP_HOME" ~/.bashrc; then
    cat << 'EOF' >> ~/.bashrc
# MyApp 环境变量
export MYAPP_HOME="$HOME/workspace/myapp"
export JAVA_HOME="$(dirname $(dirname $(readlink -f $(which java))))"
alias myapp-dev="cd $MYAPP_HOME && mvn spring-boot:run -Dspring-boot.run.profiles=dev"
alias myapp-test="cd $MYAPP_HOME && mvn test"
alias myapp-logs="tail -f $MYAPP_HOME/logs/app.log"
EOF
    log "Environment variables added to ~/.bashrc"
fi

# === 验证 ===
log "Verifying installation..."
declare -A checks=(
    ["git"]="git --version"
    ["java"]="java --version"
    ["mvn"]="mvn --version"
    ["node"]="node --version"
    ["docker"]="docker --version"
    ["curl"]="curl --version | head -1"
)
all_ok=true
for tool in "${!checks[@]}"; do
    if eval "${checks[$tool]}" &>/dev/null; then
        log "  [OK]  $tool"
    else
        log "  [FAIL] $tool"
        all_ok=false
    fi
done

if $all_ok; then
    log "=== Setup complete! Run 'source ~/.bashrc' to apply environment ==="
else
    error "Some tools failed to install. Check logs above."
fi
```

## 7.2 生产环境基线检查

```bash
#!/bin/bash
# baseline_check.sh —— 生产环境基线检查（CIS 风格）
set -euo pipefail

REPORT_FILE="baseline_report_$(hostname)_$(date +%Y%m%d).txt"
PASS=0
FAIL=0
WARN=0

check() {
    local id="$1" desc="$2" result="$3" detail="${4:-}"
    case "$result" in
        PASS) ((PASS++)); echo "[PASS] $id: $desc" >> "$REPORT_FILE" ;;
        FAIL) ((FAIL++)); echo "[FAIL] $id: $desc — $detail" >> "$REPORT_FILE" ;;
        WARN) ((WARN++)); echo "[WARN] $id: $desc — $detail" >> "$REPORT_FILE" ;;
    esac
}

echo "=== Baseline Check Report ===" > "$REPORT_FILE"
echo "Host: $(hostname)" >> "$REPORT_FILE"
echo "Date: $(date)" >> "$REPORT_FILE"
echo "==================================" >> "$REPORT_FILE"
echo "" >> "$REPORT_FILE"

# 1. 操作系统安全
# 1.1 检查 SSH 是否禁用 root 登录
if grep -q "^PermitRootLogin no" /etc/ssh/sshd_config 2>/dev/null; then
    check "OS-01" "SSH root login disabled" "PASS"
else
    check "OS-01" "SSH root login disabled" "FAIL" "PermitRootLogin not set to 'no'"
fi

# 1.2 检查 SSH 是否禁用密码认证
if grep -q "^PasswordAuthentication no" /etc/ssh/sshd_config 2>/dev/null; then
    check "OS-02" "SSH password auth disabled" "PASS"
else
    check "OS-02" "SSH password auth disabled" "FAIL" "Password auth still enabled"
fi

# 1.3 检查防火墙状态
if command -v ufw &>/dev/null; then
    if ufw status | grep -q "Status: active"; then
        check "OS-03" "Firewall (ufw) active" "PASS"
    else
        check "OS-03" "Firewall (ufw) active" "FAIL" "ufw is inactive"
    fi
elif command -v firewalld &>/dev/null; then
    if systemctl is-active --quiet firewalld; then
        check "OS-03" "Firewall (firewalld) active" "PASS"
    else
        check "OS-03" "Firewall (firewalld) active" "FAIL" "firewalld is inactive"
    fi
fi

# 1.4 检查 SELinux/AppArmor
if command -v getenforce &>/dev/null; then
    if [[ "$(getenforce)" == "Enforcing" ]]; then
        check "OS-04" "SELinux enforcing" "PASS"
    else
        check "OS-04" "SELinux enforcing" "FAIL" "SELinux is $(getenforce)"
    fi
elif command -v aa-status &>/dev/null; then
    if aa-status --enabled &>/dev/null; then
        check "OS-04" "AppArmor enabled" "PASS"
    else
        check "OS-04" "AppArmor enabled" "FAIL" "AppArmor not enabled"
    fi
fi

# 2. 系统资源
# 2.1 磁盘使用率
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')
if [[ "$DISK_USAGE" -lt 80 ]]; then
    check "SYS-01" "Disk usage below 80%" "PASS"
elif [[ "$DISK_USAGE" -lt 90 ]]; then
    check "SYS-01" "Disk usage below 80%" "WARN" "Disk usage at ${DISK_USAGE}%"
else
    check "SYS-01" "Disk usage below 80%" "FAIL" "Disk usage at ${DISK_USAGE}%"
fi

# 2.2 内存
MEM_AVAILABLE_MB=$(free -m | awk 'NR==2 {print $7}')
if [[ "$MEM_AVAILABLE_MB" -gt 1024 ]]; then
    check "SYS-02" "Available memory > 1GB" "PASS"
else
    check "SYS-02" "Available memory > 1GB" "WARN" "Only ${MEM_AVAILABLE_MB}MB available"
fi

# 2.3 Swap 使用
SWAP_USED=$(free -m | awk 'NR==3 {print $3}')
SWAP_TOTAL=$(free -m | awk 'NR==3 {print $2}')
if [[ "$SWAP_TOTAL" -gt 0 ]]; then
    SWAP_PCT=$(( SWAP_USED * 100 / SWAP_TOTAL ))
    if [[ "$SWAP_PCT" -lt 20 ]]; then
        check "SYS-03" "Swap usage low" "PASS"
    else
        check "SYS-03" "Swap usage low" "WARN" "Swap at ${SWAP_PCT}%"
    fi
fi

# 3. 服务状态
# 3.1 NTP 时间同步
if timedatectl show | grep -q "NTPSynchronized=yes"; then
    check "SRV-01" "NTP synchronized" "PASS"
else
    check "SRV-01" "NTP synchronized" "FAIL" "Time not synchronized"
fi

# 3.2 关键服务运行状态
for svc in docker kubelet nginx; do
    if systemctl is-active --quiet "$svc" 2>/dev/null; then
        check "SRV-02" "$svc running" "PASS"
    else
        check "SRV-02" "$svc running" "FAIL" "$svc not active"
    fi
done

# 4. 容器/K8s 检查
# 4.1 异常 Pod
if command -v kubectl &>/dev/null; then
    FAILED_PODS=$(kubectl get pods --all-namespaces \
        --field-selector=status.phase!=Running,status.phase!=Succeeded \
        -o json 2>/dev/null | jq -r '.items | length' 2>/dev/null || echo 0)
    if [[ "$FAILED_PODS" -eq 0 ]]; then
        check "K8S-01" "No failed pods" "PASS"
    else
        check "K8S-01" "No failed pods" "FAIL" "${FAILED_PODS} pods not healthy"
    fi
fi

# 汇总
{
    echo ""
    echo "=================================="
    echo "Summary:"
    echo "  PASS: $PASS"
    echo "  FAIL: $FAIL"
    echo "  WARN: $WARN"
    echo "=================================="
} >> "$REPORT_FILE"

cat "$REPORT_FILE"
echo "Report saved to $REPORT_FILE"

if [[ "$FAIL" -gt 0 ]]; then
    exit 1
fi
```

---

# 8. 数据库迁移脚本

## 8.1 Flyway 包装脚本

```bash
#!/bin/bash
# flyway_migrate.sh —— Flyway 数据库迁移包装
set -euo pipefail

# === 配置 ===
FLYWAY_VERSION="${FLYWAY_VERSION:-10.10.0}"
FLYWAY_HOME="/opt/flyway-${FLYWAY_VERSION}"
MIGRATIONS_DIR="${MIGRATIONS_DIR:-./db/migration}"

# 数据库配置（从环境变量或配置文件读取）
DB_URL="${DB_URL:-jdbc:postgresql://localhost:5432/myapp}"
DB_USER="${DB_USER:-myapp}"
DB_PASSWORD="${DB_PASSWORD:-}"

# === 函数 ===
log()   { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; exit 1; }

# 预检查
precheck() {
    log "Running pre-migration checks..."

    # Flyway 是否安装
    if [[ ! -f "${FLYWAY_HOME}/flyway" ]]; then
        error "Flyway not found at ${FLYWAY_HOME}. Run ./install_flyway.sh first."
    fi

    # 迁移目录是否存在
    if [[ ! -d "$MIGRATIONS_DIR" ]]; then
        error "Migrations directory not found: $MIGRATIONS_DIR"
    fi

    # 统计迁移文件数量
    local count
    count=$(find "$MIGRATIONS_DIR" -name "V*__*.sql" | wc -l)
    log "  Found ${count} migration files"

    # 数据库连通性
    local host port
    host=$(echo "$DB_URL" | sed -n 's|.*://.*@\([^:/]*\).*|\1|p')
    port=$(echo "$DB_URL" | sed -n 's|.*:\([0-9]*\)/.*|\1|p')
    port="${port:-5432}"

    if command -v nc &>/dev/null; then
        if nc -z -w5 "$host" "$port" 2>/dev/null; then
            log "  Database reachable at ${host}:${port}"
        else
            error "Cannot connect to database at ${host}:${port}"
        fi
    fi

    log "Precheck passed."
}

# 备份（迁移前）
backup() {
    local backup_file="db_backup_$(date +%Y%m%d_%H%M%S).sql"

    log "Creating backup: ${backup_file}"

    # 从 JDBC URL 解析连接信息
    local host db_name
    host=$(echo "$DB_URL" | sed -n 's|.*://.*@\([^:/]*\).*|\1|p')
    db_name=$(echo "$DB_URL" | sed -n 's|.*/\([^?]*\).*|\1|p')

    PGPASSWORD="$DB_PASSWORD" pg_dump \
        -h "$host" \
        -U "$DB_USER" \
        -d "$db_name" \
        -F c \
        -f "$backup_file"

    log "  Backup saved to ${backup_file}"
    echo "$backup_file"
}

# 执行迁移
run_migration() {
    log "Running Flyway migration..."

    # info：检查迁移状态
    log "Current migration status:"
    "${FLYWAY_HOME}/flyway" \
        -url="$DB_URL" \
        -user="$DB_USER" \
        -password="$DB_PASSWORD" \
        -locations="filesystem:${MIGRATIONS_DIR}" \
        -table="flyway_schema_history" \
        info 2>&1 | tail -20

    echo ""
    log "Applying pending migrations..."

    # migrate：执行迁移
    "${FLYWAY_HOME}/flyway" \
        -url="$DB_URL" \
        -user="$DB_USER" \
        -password="$DB_PASSWORD" \
        -locations="filesystem:${MIGRATIONS_DIR}" \
        -table="flyway_schema_history" \
        -baselineOnMigrate="false" \
        -validateMigrationNaming="true" \
        -outOfOrder="false" \
        migrate

    log "Migration completed."
}

# 校验
validate() {
    log "Validating migrations..."
    "${FLYWAY_HOME}/flyway" \
        -url="$DB_URL" \
        -user="$DB_USER" \
        -password="$DB_PASSWORD" \
        -locations="filesystem:${MIGRATIONS_DIR}" \
        validate
    log "Validation passed."
}

# 修复（谨慎使用）
repair() {
    log "WARNING: Repairing Flyway schema history table..."
    read -p "This will realign checksums. Continue? (yes/no): " confirm
    if [[ "$confirm" != "yes" ]]; then
        log "Repair cancelled"
        exit 0
    fi
    "${FLYWAY_HOME}/flyway" \
        -url="$DB_URL" \
        -user="$DB_USER" \
        -password="$DB_PASSWORD" \
        -locations="filesystem:${MIGRATIONS_DIR}" \
        repair
}

# 回滚（需要创建 Undo 迁移脚本：U1.1__xxx.sql）
rollback() {
    local target_version="${1:-}"
    if [[ -z "$target_version" ]]; then
        error "Usage: $0 rollback <version>"
    fi

    log "Rolling back to version: ${target_version}"

    "${FLYWAY_HOME}/flyway" \
        -url="$DB_URL" \
        -user="$DB_USER" \
        -password="$DB_PASSWORD" \
        -locations="filesystem:${MIGRATIONS_DIR}" \
        -target="$target_version" \
        migrate

    log "Rollback to ${target_version} completed."
}

# === 主逻辑 ===
main() {
    local cmd="${1:-migrate}"

    case "$cmd" in
        migrate)
            precheck
            backup
            run_migration
            validate
            ;;
        clean)
            precheck
            backup
            read -p "CLEAN will drop all objects! Continue? (yes/no): " confirm
            [[ "$confirm" == "yes" ]] || { echo "Aborted"; exit 0; }
            "${FLYWAY_HOME}/flyway" \
                -url="$DB_URL" -user="$DB_USER" -password="$DB_PASSWORD" clean
            ;;
        info)
            "${FLYWAY_HOME}/flyway" \
                -url="$DB_URL" -user="$DB_USER" -password="$DB_PASSWORD" info
            ;;
        validate)
            validate
            ;;
        repair)
            repair
            ;;
        rollback)
            rollback "$2"
            ;;
        *)
            echo "Usage: $0 {migrate|clean|info|validate|repair|rollback <version>}"
            exit 1
            ;;
    esac
}

main "$@"
```

## 8.2 Liquibase 包装脚本

```bash
#!/bin/bash
# liquibase_update.sh —— Liquibase 更新包装
set -euo pipefail

LIQUIBASE_VERSION="4.28.0"
LIQUIBASE_HOME="/opt/liquibase-${LIQUIBASE_VERSION}"

DB_URL="${DB_URL:-jdbc:postgresql://localhost:5432/myapp}"
DB_USER="${DB_USER:-myapp}"
DB_PASSWORD="${DB_PASSWORD:-}"
CHANGELOG_FILE="${CHANGELOG_FILE:-db/changelog/db.changelog-master.yaml}"

log()   { echo "[$(date +%H:%M:%S)] $*"; }
error() { echo "[ERROR] $*" >&2; exit 1; }

# 运行 Liquibase 命令的通用包装
run_liquibase() {
    local command="$1"
    shift

    "${LIQUIBASE_HOME}/liquibase" \
        --url="${DB_URL}" \
        --username="${DB_USER}" \
        --password="${DB_PASSWORD}" \
        --changeLogFile="${CHANGELOG_FILE}" \
        --hub-mode=off \
        "$command" "$@"
}

# 检查待执行的变更
check_pending() {
    log "Checking for pending changesets..."
    local pending
    pending=$(run_liquibase status --verbose 2>/dev/null | grep -c "has not been applied" || true)

    if [[ "$pending" -gt 0 ]]; then
        echo "${pending} changesets pending"
        run_liquibase update-sql
    else
        echo "Database is up to date"
    fi
}

# 主更新逻辑
update() {
    log "Starting database update..."

    # 步骤 1：检查状态
    run_liquibase status

    # 步骤 2：生成 SQL 预览
    local preview_file="liquibase_preview_$(date +%Y%m%d_%H%M%S).sql"
    run_liquibase update-sql > "$preview_file"
    log "SQL preview saved to: ${preview_file}"

    # 步骤 3：确认执行（交互式）
    if [[ "${AUTO_APPROVE:-false}" != "true" ]]; then
        echo ""
        read -p "Apply these changes? (yes/no): " confirm
        if [[ "$confirm" != "yes" ]]; then
            log "Update cancelled"
            exit 0
        fi
    fi

    # 步骤 4：执行更新
    run_liquibase update

    # 步骤 5：验证
    run_liquibase history --last 3

    log "Update completed."
}

# 回滚
rollback() {
    local tag="${1:-}"

    if [[ -z "$tag" ]]; then
        # 获取最近的 tag
        log "Recent tags:"
        run_liquibase tag-exists --tag="v1.0"
        error "Usage: $0 rollback <tag>"
    fi

    log "Rolling back to tag: ${tag}"

    if [[ "${AUTO_APPROVE:-false}" != "true" ]]; then
        run_liquibase rollback-sql "$tag"
        read -p "Apply rollback? (yes/no): " confirm
        [[ "$confirm" == "yes" ]] || { log "Rollback cancelled"; exit 0; }
    fi

    run_liquibase rollback "$tag"
    log "Rollback completed."
}

# 生成差异（对比当前数据库和目标状态）
diff() {
    local reference_url="${1:-}"
    [[ -z "$reference_url" ]] && error "Usage: $0 diff <reference_url>"

    local diff_file="liquibase_diff_$(date +%Y%m%d_%H%M%S).yaml"

    run_liquibase diffChangeLog \
        --referenceUrl="$reference_url" \
        --referenceUsername="$DB_USER" \
        --referencePassword="$DB_PASSWORD" \
        --changeLogFile="$diff_file"

    log "Diff changelog saved to: ${diff_file}"
}

main() {
    case "${1:-}" in
        update)       update ;;
        status)       run_liquibase status --verbose ;;
        check)        check_pending ;;
        rollback)     rollback "$2" ;;
        diff)         diff "$2" ;;
        history)      run_liquibase history ;;
        *)
            echo "Usage: $0 {update|status|check|rollback <tag>|diff <url>|history}"
            exit 1
            ;;
    esac
}

main "$@"
```

---

# 9. 日志收集与管道

## 9.1 Filebeat 预处理脚本

Filebeat 支持通过 `script` processor 执行 Shell 脚本处理日志行：

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/myapp/*.log
    exclude_lines: ['^\s*$']   # 跳过空行
    multiline:                  # 多行合并：Java 堆栈
      pattern: '^\d{4}-\d{2}-\d{2}'
      negate: true
      match: after

    processors:
      # 1. Shell 脚本处理器：自定义字段提取
      - script:
          lang: javascript
          source: >
            function process(event) {
                var msg = event.Get("message");
                if (!msg) return;

                // 提取耗时
                var durationMatch = msg.match(/duration=(\d+)ms/);
                if (durationMatch) {
                    event.Put("fields.duration_ms", parseInt(durationMatch[1]));
                }

                // 提取状态码
                var codeMatch = msg.match(/status=(\d{3})/);
                if (codeMatch) {
                    event.Put("fields.status_code", parseInt(codeMatch[1]));
                }

                // 提取 traceId
                var traceMatch = msg.match(/traceId=([a-f0-9-]+)/);
                if (traceMatch) {
                    event.Put("fields.trace_id", traceMatch[1]);
                }
            }

      # 2. Shell exec processor：调用外部脚本
      - script:
          lang: javascript
          tag_on_exception:
            - script_error
          timeout: "5s"
          source: >
            function process(event) {
                // 构造要执行的命令
                var line = event.Get("message");
                var cmd = "./scripts/enrich_log.sh " + JSON.stringify(line);
                var result = new Console();

                try {
                    var output = fs.run(cmd);
                    var enriched = JSON.parse(output);
                    event.Put("app", enriched);
                } catch(e) {
                    event.Tag("enrich_failed");
                }
            }
```

外部 Shell 脚本（enrich_log.sh）：

```bash
#!/bin/bash
# enrich_log.sh —— 被 Filebeat script processor 调用
# 输入：单行日志
# 输出：JSON 格式的富化数据

set -euo pipefail

LOG_LINE="$1"

# 从日志内容提取信息
ip=$(echo "$LOG_LINE" | grep -oP '(\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})' | head -1)
endpoint=$(echo "$LOG_LINE" | grep -oP '"(GET|POST|PUT|DELETE) [^"]+"' | head -1 | cut -d' ' -f2)
method=$(echo "$LOG_LINE" | grep -oP '"(GET|POST|PUT|DELETE)' | tr -d '"')

# 地理信息查询（使用本地 GeoIP 数据库）
geo_json="{}"
if [[ -n "$ip" ]]; then
    geo_json=$(curl -sf "http://localhost:8088/geoip/${ip}" 2>/dev/null || echo "{}")
fi

# 输出 JSON
jq -n \
    --arg ip "$ip" \
    --arg method "$method" \
    --arg endpoint "$endpoint" \
    --argjson geo "$geo_json" \
    '{
        ip: $ip,
        method: $method,
        endpoint: $endpoint,
        geo: $geo
    }'

exit 0
```

## 9.2 Fluentd exec 输入

```xml
<source>
  @type exec
  tag system.metrics
  command /opt/scripts/collect_metrics.sh
  <parse>
    @type json
  </parse>
  run_interval 60s
</source>

<source>
  @type exec
  tag app.status
  command /opt/scripts/check_app_status.sh
  <parse>
    @type json
  </parse>
  run_interval 30s
</source>
```

```bash
#!/bin/bash
# collect_metrics.sh —— Fluentd exec input 采集脚本
# 输出 JSON 到 stdout，Fluentd 解析并转发

set -euo pipefail

# CPU 使用率
cpu_pct=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)

# 内存
mem_total=$(free -m | awk 'NR==2{print $2}')
mem_used=$(free -m | awk 'NR==2{print $3}')
mem_pct=$(awk "BEGIN {printf \"%.1f\", ($mem_used/$mem_total)*100}")

# 磁盘 I/O
disk_util=$(iostat -x 1 2 | awk 'END {print $NF}' | head -1 || echo "0")

# TCP 连接数
tcp_conns=$(ss -tan | wc -l)

# 应用 JVM 指标
jvm_heap_used=$(curl -sf http://localhost:8080/actuator/metrics/jvm.memory.used \
    | jq -r '.measurements[0].value // 0')

jvm_threads=$(curl -sf http://localhost:8080/actuator/metrics/jvm.threads.live \
    | jq -r '.measurements[0].value // 0')

# 输出 JSON（一行，Fluentd 要求）
jq -n -c \
    --arg host "$(hostname)" \
    --argjson cpu "$cpu_pct" \
    --argjson mem_pct "$mem_pct" \
    --argjson mem_used "$mem_used" \
    --argjson disk "$disk_util" \
    --argjson tcp "$tcp_conns" \
    --argjson heap "$jvm_heap_used" \
    --argjson threads "$jvm_threads" \
    '{host: $host, cpu_percent: $cpu, mem_percent: $mem_pct,
      mem_used_mb: $mem_used, disk_util_percent: $disk,
      tcp_connections: $tcp, jvm_heap_bytes: $heap, jvm_threads: $threads}'
```

## 9.3 完整日志管道示例

```bash
#!/bin/bash
# log_pipeline.sh —— 日志预处理与转发管道
# 用法: tail -f app.log | ./log_pipeline.sh
set -euo pipefail

# 配置
ELASTICSEARCH_URL="${ES_URL:-http://localhost:9200}"
INDEX_PREFIX="myapp-logs"
BUFFER_FILE="/tmp/log_buffer.ndjson"
BATCH_SIZE=100
FLUSH_INTERVAL=5   # 秒

# 处理函数：日志富化
enrich() {
    while IFS= read -r line; do
        local timestamp
        timestamp=$(date -u +"%Y-%m-%dT%H:%M:%S.%3NZ")

        # 提取日志级别
        local level="INFO"
        case "$line" in
            *ERROR*)   level="ERROR" ;;
            *WARN*)    level="WARN"  ;;
            *DEBUG*)   level="DEBUG" ;;
        esac

        # 提取 traceId
        local trace_id=""
        if [[ "$line" =~ traceId=([a-f0-9-]+) ]]; then
            trace_id="${BASH_REMATCH[1]}"
        fi

        # 提取耗时
        local duration=0
        if [[ "$line" =~ duration=([0-9]+)ms ]]; then
            duration="${BASH_REMATCH[1]}"
        fi

        # 构建 JSON
        jq -n -c \
            --arg ts "$timestamp" \
            --arg level "$level" \
            --arg msg "$line" \
            --arg trace "$trace_id" \
            --argjson dur "$duration" \
            '{"@timestamp":$ts, "level":$level, "message":$msg,
              "trace_id":$trace, "duration_ms":$dur}' \
            || echo "{}"
    done
}

# 批量发送到 Elasticsearch
flush_to_es() {
    local buf="$1"

    if [[ ! -s "$buf" ]]; then
        return
    fi

    # 构建 bulk API 请求体
    local index_name="${INDEX_PREFIX}-$(date +%Y.%m.%d)"
    local bulk_body=""

    while IFS= read -r line; do
        bulk_body+="{\"index\":{\"_index\":\"$index_name\"}}"$'\n'
        bulk_body+="$line"$'\n'
    done < "$buf"

    if [[ -n "$bulk_body" ]]; then
        curl -sf -X POST "${ELASTICSEARCH_URL}/_bulk" \
            -H 'Content-Type: application/x-ndjson' \
            --data-binary "$bulk_body" > /dev/null || {
            echo "[ERROR] Failed to flush to Elasticsearch" >&2
            return 1
        }
        echo "$(wc -l < "$buf") documents flushed to $index_name"
    fi

    : > "$buf"
}

# 主循环
main() {
    echo "Starting log pipeline..."

    local line_count=0
    local last_flush
    last_flush=$(date +%s)

    # 管道：stdin -> enrich -> buffer -> ES batch
    enrich | while IFS= read -r json_line; do
        if [[ "$json_line" != "{}" ]]; then
            echo "$json_line" >> "$BUFFER_FILE"
            ((line_count++))

            local now
            now=$(date +%s)

            # 批量触发条件：行数足够 或 时间间隔到了
            if [[ "$line_count" -ge "$BATCH_SIZE" ]] || \
               [[ $(( now - last_flush )) -ge "$FLUSH_INTERVAL" ]]; then
                flush_to_es "$BUFFER_FILE"
                line_count=0
                last_flush="$now"
            fi
        fi
    done

    # 最终刷新
    flush_to_es "$BUFFER_FILE"
    echo "Log pipeline stopped"
}

main
```

---

# 10. Git Hooks

## 10.1 pre-commit 检查

```bash
#!/bin/bash
# .git/hooks/pre-commit —— 提交前自动检查
set -euo pipefail

echo "=== Running pre-commit checks ==="

STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM)
ERRORS=0

# 1. 禁止提交 TODO / FIXME（可配置排除）
check_todo() {
    local files="$1"
    if [[ -z "$files" ]]; then return; fi

    # 检查新增或修改的行中的 TODO
    local todos
    todos=$(git diff --cached -U0 | grep -E '^\+.*TODO' || true)
    if [[ -n "$todos" ]]; then
        echo ""
        echo "[WARN] Found TODO comments:"
        echo "$todos" | sed 's/^/  /'
        echo ""
        # WARN 不阻断提交，若要阻断将 return 改为 exit 1
    fi
}

# 2. 检查尾随空白
check_trailing_whitespace() {
    local files="$1"
    if [[ -z "$files" ]]; then return; fi

    local trailing
    trailing=$(git diff --cached --check | grep -v "^$" || true)
    if [[ -n "$trailing" ]]; then
        echo "[ERROR] Trailing whitespace found:"
        echo "$trailing"
        ((ERRORS++))
    fi
}

# 3. 文件大小检查
check_file_size() {
    local files="$1"
    if [[ -z "$files" ]]; then return; fi
    local MAX_SIZE_KB=500  # 最大 500KB

    while IFS= read -r file; do
        if [[ -f "$file" ]]; then
            local size_kb
            size_kb=$(du -k "$file" | cut -f1)
            if [[ "$size_kb" -gt "$MAX_SIZE_KB" ]]; then
                echo "[ERROR] File too large: $file (${size_kb}KB > ${MAX_SIZE_KB}KB)"
                ((ERRORS++))
            fi
        fi
    done <<< "$files"
}

# 4. 禁止提交私钥
check_private_keys() {
    local files="$1"
    if [[ -z "$files" ]]; then return; fi

    while IFS= read -r file; do
        if grep -lqE -- "-----BEGIN (RSA|DSA|EC|OPENSSH) PRIVATE KEY-----" "$file" 2>/dev/null; then
            echo "[ERROR] Private key detected in: $file"
            ((ERRORS++))
        fi
        if grep -lqE -- "(password|secret|api_key)\s*=\s*['\"]?\w{20,}" "$file" 2>/dev/null; then
            echo "[ERROR] Possible hardcoded secret in: $file"
            ((ERRORS++))
        fi
    done <<< "$files"
}

# 5. 代码风格检查（Java）
check_java_style() {
    local java_files
    java_files=$(echo "$STAGED_FILES" | grep '\.java$' || true)
    if [[ -z "$java_files" ]]; then return; fi

    # 格式检查（使用项目统一的 formatter）
    if [[ -f "build.gradle" ]] || [[ -f "build.gradle.kts" ]]; then
        if command -v gradle &>/dev/null; then
            gradle spotlessCheck -q 2>/dev/null || {
                echo "[ERROR] Code formatting issues detected. Run: gradle spotlessApply"
                ((ERRORS++))
            }
        fi
    fi
}

# 6. 单元测试（仅检查测试类本身不被注释）
check_test_annotations() {
    local test_files
    test_files=$(echo "$STAGED_FILES" | grep 'Test\.java$' || true)
    if [[ -z "$test_files" ]]; then return; fi

    while IFS= read -r file; do
        if grep -El '@Disabled|@Ignore' "$file" 2>/dev/null | grep -q .; then
            echo "[WARN] Disabled tests found in: $file"
        fi
    done <<< "$test_files"
}

# 7. 大型二进制文件检查
check_binary_files() {
    local files="$1"
    if [[ -z "$files" ]]; then return; fi

    while IFS= read -r file; do
        if [[ -f "$file" ]] && file "$file" | grep -qE "(ELF|Mach-O|PE32)"; then
            echo "[ERROR] Binary/executable file detected: $file"
            ((ERRORS++))
        fi
    done <<< "$files"
}

# === 执行所有检查 ===
check_trailing_whitespace "$STAGED_FILES"
check_private_keys "$STAGED_FILES"
check_binary_files "$STAGED_FILES"
check_file_size "$STAGED_FILES"
check_todo "$STAGED_FILES"
check_java_style
check_test_annotations

# === 结果 ===
echo ""
if [[ "$ERRORS" -gt 0 ]]; then
    echo "[FAIL] Pre-commit checks found ${ERRORS} error(s). Commit aborted."
    exit 1
fi

echo "[OK] Pre-commit checks passed"
exit 0
```

## 10.2 commit-msg 规范校验

```bash
#!/bin/bash
# .git/hooks/commit-msg —— 校验提交信息格式
# 规范：Conventional Commits
# <type>(<scope>): <subject>
# 类型：feat, fix, docs, style, refactor, perf, test, chore, ci, build, revert
set -euo pipefail

COMMIT_MSG_FILE="$1"
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# 允许 merge 和 revert 自动生成的提交
if echo "$COMMIT_MSG" | grep -qE "^(Merge|Revert)"; then
    exit 0
fi

# 规范正则
PATTERN="^(feat|fix|docs|style|refactor|perf|test|chore|ci|build|revert)(\([a-zA-Z0-9._-]+\))?: .{1,72}$"

if ! echo "$COMMIT_MSG" | head -1 | grep -qE "$PATTERN"; then
    echo "============================================================"
    echo "ERROR: Commit message does not follow Conventional Commits"
    echo "============================================================"
    echo ""
    echo "Format: <type>(<scope>): <subject>"
    echo ""
    echo "Allowed types:"
    echo "  feat     - New feature"
    echo "  fix      - Bug fix"
    echo "  docs     - Documentation"
    echo "  style    - Code style (formatting, missing semi-colons, etc)"
    echo "  refactor - Code refactoring"
    echo "  perf     - Performance improvement"
    echo "  test     - Adding or updating tests"
    echo "  chore    - Build process or auxiliary tool changes"
    echo "  ci       - CI configuration changes"
    echo "  build    - Build system changes"
    echo "  revert   - Revert a previous commit"
    echo ""
    echo "Examples:"
    echo "  feat(user): add login endpoint"
    echo "  fix(order): correct total amount calculation"
    echo "  docs(api): update swagger documentation"
    echo "  refactor(payment): extract strategy pattern"
    echo ""
    echo "Your commit message:"
    echo "  $(head -1 "$COMMIT_MSG_FILE")"
    echo ""
    exit 1
fi

# 检查第一行长度（不超过 72 字符）
FIRST_LINE=$(head -1 "$COMMIT_MSG_FILE")
if [[ ${#FIRST_LINE} -gt 72 ]]; then
    echo "ERROR: Commit subject line too long (${#FIRST_LINE} > 72 characters)"
    exit 1
fi

# 检查第一行不以句号结尾
if echo "$FIRST_LINE" | grep -q '\.$'; then
    echo "WARN: Commit subject should not end with a period"
fi

# 检查第二行是否为空行（主体与标题分离）
SECOND_LINE=$(sed -n '2p' "$COMMIT_MSG_FILE")
if [[ -n "$SECOND_LINE" ]] && [[ "$SECOND_LINE" != "" ]]; then
    echo "WARN: Second line should be blank (separate subject from body)"
fi

exit 0
```

## 10.3 pre-push 测试

```bash
#!/bin/bash
# .git/hooks/pre-push —— 推送前自动运行测试
set -euo pipefail

# 获取推送信息
# pre-push 从标准输入读取: <local ref> <local sha1> <remote ref> <remote sha1>
REMOTE="$1"
REMOTE_URL="$2"

echo "=== Pre-push checks ==="
echo "Remote: $REMOTE_URL"

ERRORS=0

# 读取推送的引用范围
while read -r local_ref local_sha remote_ref remote_sha; do
    echo "Pushing $local_ref ($local_sha -> $remote_sha)"
done

# 1. 获取当前分支
BRANCH=$(git rev-parse --abbrev-ref HEAD)

# 2. 针对主分支的保护
if [[ "$BRANCH" == "main" ]] || [[ "$BRANCH" == "master" ]]; then
    echo "============================================"
    echo "WARNING: You are pushing directly to $BRANCH"
    echo "Consider using a feature branch and PR"
    echo "============================================"

    # 阻止直接推送到 main（可选）
    if [[ "${ALLOW_MAIN_PUSH:-false}" != "true" ]]; then
        read -p "Are you sure? Type 'yes' to continue: " confirm
        if [[ "$confirm" != "yes" ]]; then
            echo "Push aborted"
            exit 1
        fi
    fi
fi

# 3. 运行单元测试
echo ""
echo "=== Running unit tests ==="

if [[ -f "build.gradle" ]] || [[ -f "build.gradle.kts" ]]; then
    if ! gradle test -q --no-daemon; then
        echo "[ERROR] Unit tests failed"
        ((ERRORS++))
    fi
elif [[ -f "pom.xml" ]]; then
    if ! mvn test -q -B; then
        echo "[ERROR] Unit tests failed"
        ((ERRORS++))
    fi
fi

# 4. 代码风格检查
echo ""
echo "=== Running lint ==="

if [[ -f "build.gradle.kts" ]]; then
    if ! gradle ktlintCheck 2>/dev/null; then
        echo "[ERROR] Lint check failed"
        ((ERRORS++))
    fi
fi

# 5. 禁止推送包括 WIP 的提交
echo ""
echo "=== Checking for WIP commits ==="

# 获取当前分支上未推送的提交
UNPUSHED_COMMITS=$(git log origin/"$BRANCH"..HEAD --oneline 2>/dev/null || git log --oneline -10)

if echo "$UNPUSHED_COMMITS" | grep -qiE "^\w+ (WIP|wip)"; then
    echo "[ERROR] Found WIP (Work In Progress) commits:"
    echo "$UNPUSHED_COMMITS" | grep -iE "^\w+ (WIP|wip)" | sed 's/^/  /'
    ((ERRORS++))
fi

# 6. 禁止推送大文件（> 10MB）
echo ""
echo "=== Checking for large files ==="

MAX_SIZE_KB=10240
LARGE_FILES=""
while read -r local_ref local_sha remote_ref remote_sha; do
    # 找到本地和远程之间的差异文件
    if [[ "$remote_sha" != "0000000000000000000000000000000000000000" ]]; then
        DIFF_FILES=$(git diff --name-only "$remote_sha" "$local_sha" 2>/dev/null || true)
    else
        # 新分支
        DIFF_FILES=$(git ls-files)
    fi

    while IFS= read -r file; do
        if [[ -f "$file" ]]; then
            size_kb=$(du -k "$file" | cut -f1)
            if [[ "$size_kb" -gt "$MAX_SIZE_KB" ]]; then
                echo "  [WARN] Large file: $file (${size_kb}KB > ${MAX_SIZE_KB}KB)"
                LARGE_FILES+="$file"$'\n'
            fi
        fi
    done <<< "$DIFF_FILES"
done

if [[ -n "$LARGE_FILES" ]]; then
    echo "  Consider using Git LFS for large files"
fi

# 7. 禁止泄露密钥
echo ""
echo "=== Checking for secrets ==="

while read -r local_ref local_sha remote_ref remote_sha; do
    if [[ "$remote_sha" != "0000000000000000000000000000000000000000" ]]; then
        DIFF_CONTENT=$(git diff "$remote_sha".."$local_sha" -U0 || true)
    else
        DIFF_CONTENT=$(git diff --cached -U0 || true)
    fi

    SECRETS_FOUND=false

    # AWS Access Key
    if echo "$DIFF_CONTENT" | grep -qE '^\+.*AKIA[0-9A-Z]{16}'; then
        echo "  [ERROR] AWS Access Key detected in diff"
        SECRETS_FOUND=true
        ((ERRORS++))
    fi

    # Private Key
    if echo "$DIFF_CONTENT" | grep -qE '^\+.*-----BEGIN (RSA|DSA|EC|OPENSSH) PRIVATE KEY-----'; then
        echo "  [ERROR] Private key detected in diff"
        SECRETS_FOUND=true
        ((ERRORS++))
    fi

    if [[ "$SECRETS_FOUND" == "false" ]]; then
        echo "  No secrets detected"
    fi
done

# === 结果 ===
echo ""
echo "============================================"
if [[ "$ERRORS" -gt 0 ]]; then
    echo "[FAIL] Pre-push checks found ${ERRORS} error(s). Push aborted."
    echo "  Use 'git push --no-verify' to skip checks (not recommended)"
    exit 1
fi

echo "[OK] Pre-push checks passed. Pushing..."
exit 0
```

## 10.4 全局 Git Hooks 模板

```bash
#!/bin/bash
# setup_git_hooks.sh —— 配置全局 Git Hooks 模板
set -euo pipefail

HOOKS_SOURCE_DIR="$(cd "$(dirname "$0")/git-hooks" && pwd)"
GIT_TEMPLATE_DIR="$HOME/.git-templates/hooks"

echo "Setting up global Git hooks template..."

mkdir -p "$GIT_TEMPLATE_DIR"

# 复制所有 hooks 到模板目录
for hook in pre-commit commit-msg pre-push prepare-commit-msg; do
    if [[ -f "${HOOKS_SOURCE_DIR}/${hook}" ]]; then
        cp "${HOOKS_SOURCE_DIR}/${hook}" "$GIT_TEMPLATE_DIR/"
        chmod +x "$GIT_TEMPLATE_DIR/$hook"
        echo "  Installed: $hook"
    fi
done

# 配置 Git 使用此模板
git config --global init.templateDir "$HOME/.git-templates"

echo ""
echo "Global hooks configured. New repos (git init/clone) will auto-use these hooks."
echo "To apply to existing repos, run:"
echo "  cd your-repo && git init     # safe, won't overwrite anything"
```

## 10.5 prepare-commit-msg 示例

```bash
#!/bin/bash
# .git/hooks/prepare-commit-msg —— 自动填充提交信息模板
set -euo pipefail

COMMIT_MSG_FILE="$1"
COMMIT_SOURCE="$2"   # message, template, merge, squash, commit

# 仅在普通提交时自动填充分支名
if [[ "$COMMIT_SOURCE" == "message" ]] || [[ -z "$COMMIT_SOURCE" ]]; then
    BRANCH=$(git rev-parse --abbrev-ref HEAD)

    # 从分支名提取 issue 编号（如 feature/JIRA-1234）
    if [[ "$BRANCH" =~ ([A-Z]+-[0-9]+) ]]; then
        ISSUE_ID="${BASH_REMATCH[1]}"
        sed -i "1s/^/[${ISSUE_ID}] /" "$COMMIT_MSG_FILE"
    fi

    # 如果是 feature 分支，添加类型前缀
    if [[ "$BRANCH" =~ ^feature/ ]]; then
        sed -i '1s/^/feat: /' "$COMMIT_MSG_FILE"
    elif [[ "$BRANCH" =~ ^bugfix/ ]] || [[ "$BRANCH" =~ ^hotfix/ ]]; then
        sed -i '1s/^/fix: /' "$COMMIT_MSG_FILE"
    fi
fi
```

---

# 11. 附录：DevOps Shell 常用命令速查

## 11.1 CI/CD 常用

| 命令 | 用途 |
|------|------|
| `git describe --tags --always` | 生成基于标签的版本号 |
| `date -u +%Y-%m-%dT%H:%M:%SZ` | UTC 时间戳 |
| `${GITHUB_SHA::7}` | GitHub Actions 短提交哈希 |
| `${CI_COMMIT_SHORT_SHA}` | GitLab CI 短提交哈希 |
| `set -euo pipefail` | 严格模式，管道中任何命令失败都退出 |
| `trap 'cleanup' EXIT` | 脚本退出时执行清理 |
| `curl -sf` | 静默失败模式（不输出进度，失败不输出错误内容） |

## 11.2 Docker 常用

| 命令 | 用途 |
|------|------|
| `docker system prune -f` | 清理未使用的镜像/容器/网络 |
| `docker build --build-arg VERSION=1.0 .` | 构建参数传递 |
| `docker inspect -f '{{.State.Health.Status}}' <id>` | 健康状态查询 |
| `docker run --init` | 使用 tini init 进程 |

## 11.3 Kubernetes 常用

| 命令 | 用途 |
|------|------|
| `kubectl rollout status deploy/xxx` | 等待滚动更新完成 |
| `kubectl wait --for=condition=ready pod -l app=xxx` | 等待 Pod 就绪 |
| `kubectl get events --sort-by='.lastTimestamp'` | 按时间查看事件 |
| `kubectl exec deploy/xxx -- ls /app` | 在容器中执行命令 |

## 11.4 脚本安全

```bash
# 必须放到每个脚本顶部
set -euo pipefail
# -e: 命令失败则退出
# -u: 未定义变量视为错误
# -o pipefail: 管道中任一命令失败则整个管道失败

# 信号陷阱
trap 'echo "Interrupted"; cleanup; exit 130' INT TERM
trap 'echo "Exit with \$$?"' EXIT

# 锁定防止并发
exec 200>/var/lock/myapp.lock
flock -n 200 || { echo "Another instance is running"; exit 1; }
```
