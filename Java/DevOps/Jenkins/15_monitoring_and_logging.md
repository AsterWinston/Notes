
# 1. Jenkins 日志系统

## 1.1 日志级别

Jenkins 使用 java.util.logging，支持以下日志级别：

| 级别 | 说明 | 使用场景 |
|------|------|---------|
| SEVERE | 严重错误 | Jenkins 崩溃、插件加载失败等 |
| WARNING | 警告 | 配置异常、非致命错误 |
| INFO | 信息 | 正常操作信息 |
| FINE | 调试信息 | 方法级别调试 |
| FINER | 更详细调试 | 跟踪详细的执行路径 |
| FINEST | 最详细调试 | 最底层跟踪 |
| OFF | 关闭 | 关闭特定 logger |
| ALL | 全部 | 启用所有日志 |

## 1.2 配置日志级别

**方式一：通过 UI**

```
Manage Jenkins > System Log > Log Levels
├── 全局日志级别
└── 特定 Package 日志级别
```

**方式二：通过系统属性**

```bash
# 启动时设置日志级别
export JAVA_OPTS="$JAVA_OPTS \
  -Dhudson.model.Run=ALL \
  -Dorg.jenkinsci.plugins.workflow=ALL \
  -Djenkins.security=FINE"
```

**方式三：通过日志文件**

```properties
# log.properties
# 在 JENKINS_HOME 下创建

# 全局级别
.level = INFO

# 特定 logger 级别
hudson.model.Run.level = ALL
org.jenkinsci.plugins.workflow.level = FINE
jenkins.security.level = FINE
hudson.security.level = FINE

# 输出到控制台
handlers = java.util.logging.ConsoleHandler
java.util.logging.ConsoleHandler.level = ALL
```

## 1.3 系统日志查看

```
通过 UI 查看日志：
Manage Jenkins > System Log
├── All Jenkins Logs（所有日志）
├── Jenkins Log（jenkins.log）
└── 自定义日志查看器

查看关键日志文件：
tail -f /var/log/jenkins/jenkins.log
tail -f /var/log/jenkins/access.log

日志存储位置：
Linux: /var/lib/jenkins/logs/
Docker: /var/jenkins_home/logs/
Windows: %JENKINS_HOME%/logs/
```

## 1.4 日志轮转

```bash
# logrotate 配置
cat <<EOF > /etc/logrotate.d/jenkins
/var/log/jenkins/jenkins.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    copytruncate
    dateext
    dateformat -%Y%m%d
    maxsize 500M
}
EOF
```

# 2. Metrics 插件

## 2.1 安装和配置

```
插件名称：Metrics Plugin
功能：
├── 暴露 Jenkins 运行指标
├── 支持 Prometheus 格式
├── 支持 Health Check
└── 支持自定义指标
```

```yaml
# Metrics 插件配置
Manage Jenkins > Configure System > Metrics:
  Server:
    Enabled: true
    Port: 0  # 0 = 使用 Jenkins HTTP 端口
    Path: /metrics

  Prometheus:
    Enabled: true
    Path: /metrics/prometheus

  Health Check:
    Enabled: true
    Path: /metrics/healthcheck

  Ping:
    Enabled: true
    Path: /metrics/ping
```

## 2.2 关键指标

**JVM 指标**：

```
# 堆内存使用
jvm_memory_used_bytes{area="heap"} 1.5e9
jvm_memory_max_bytes{area="heap"} 4.0e9
jvm_memory_committed_bytes{area="heap"} 2.0e9

# 非堆内存
jvm_memory_used_bytes{area="nonheap"} 3.0e8

# GC
jvm_gc_collection_seconds_count{gc="G1 Young Generation"} 1234
jvm_gc_collection_seconds_sum{gc="G1 Full GC"} 45.6

# 线程
jvm_threads_current 120
jvm_threads_daemon 80
```

**Jenkins 指标**：

```
# 节点/Executor
jenkins_node_count{node_name="master"} 1
jenkins_node_count{node_name="agent-1"} 1
jenkins_executor_count{node_name="master"} 2
jenkins_executor_in_use{node_name="master"} 0
jenkins_executor_available{node_name="agent-1"} 4

# 队列
jenkins_queue_size 0
jenkins_queue_waiting_time_ms 0

# Job 构建
jenkins_job_build_duration_ms{job_name="my-pipeline",result="SUCCESS"} 120000
jenkins_job_build_count{job_name="my-pipeline",result="SUCCESS"} 150
jenkins_job_build_count{job_name="my-pipeline",result="FAILURE"} 5
jenkins_job_build_result{job_name="my-pipeline"} 1.0

# 插件
jenkins_plugin_count 120
jenkins_plugin_failed_count 0

# 系统
jenkins_uptime_seconds 864000
```

## 2.3 访问指标

```bash
# Prometheus 格式
curl -u admin:token http://jenkins:8080/metrics/prometheus

# JSON 格式
curl -u admin:token http://jenkins:8080/metrics/

# Health Check（返回 200 表示健康）
curl -u admin:token http://jenkins:8080/metrics/healthcheck

# Ping
curl -u admin:token http://jenkins:8080/metrics/ping

# 源码级别的计数器
curl -u admin:token 'http://jenkins:8080/metrics/?pretty=true'
```

# 3. Prometheus + Grafana 集成

## 3.1 Prometheus 配置

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'jenkins'
    metrics_path: '/metrics/prometheus'
    static_configs:
      - targets: ['jenkins:8080']
    basic_auth:
      username: 'admin'
      password: 'api_token'

  - job_name: 'jenkins-node'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['jenkins-node:9100']  # node_exporter
```

## 3.2 Grafana 仪表盘

推荐使用社区提供的 [Jenkins Dashboard](https://grafana.com/grafana/dashboards/)：

```
推荐仪表盘：
├── Jenkins Performance and Health Overview
│   ├── JVM 内存使用趋势
│   ├── GC 频率和耗时
│   ├── Executor 利用率
│   └── 队列大小趋势
│
├── Jenkins Jobs Dashboard
│   ├── 构建成功率 / 失败率
│   ├── 构建耗时趋势（P50/P95/P99）
│   ├── 最慢的 Job 列表
│   └── 构建结果分布
│
└── Jenkins CI/CD Pipeline Dashboard
    ├── Pipeline 各阶段耗时
    ├── 部署频率
    ├── 变更失败率
    └── 恢复时间（MTTR）
```

## 3.3 关键告警规则

```yaml
# Prometheus 告警规则
groups:
  - name: jenkins-alerts
    rules:
      # Jenkins 服务不可用
      - alert: JenkinsDown
        expr: up{job="jenkins"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Jenkins is down"
          description: "Jenkins instance has been unreachable for 1 minute"

      # 队列积压
      - alert: JenkinsQueueBacklog
        expr: jenkins_queue_size > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Jenkins queue backlog"
          description: "Build queue has {{ $value }} pending jobs for 5 minutes"

      # 堆内存使用率过高
      - alert: JenkinsHighHeapUsage
        expr: (jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}) > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Jenkins heap memory high"
          description: "Heap usage is {{ $value | humanizePercentage }}"

      # 构建失败率过高
      - alert: JenkinsHighFailureRate
        expr: rate(jenkins_job_build_count{result="FAILURE"}[1h]) / rate(jenkins_job_build_count[1h]) > 0.2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High build failure rate"
          description: "Build failure rate is {{ $value | humanizePercentage }} in the last hour"

      # 磁盘空间不足
      - alert: JenkinsLowDiskSpace
        expr: (node_filesystem_avail_bytes{mountpoint="/var/lib/jenkins"} / node_filesystem_size_bytes{mountpoint="/var/lib/jenkins"}) < 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Jenkins disk space low"
          description: "Available disk space is {{ $value | humanizePercentage }}"
```

# 4. 构建监控

## 4.1 构建耗时分析

```groovy
// Pipeline 中添加耗时跟踪
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                script {
                    def startTime = System.currentTimeMillis()
                    try {
                        sh 'mvn clean package'
                    } finally {
                        def duration = System.currentTimeMillis() - startTime
                        echo "[METRICS] Build stage took ${duration}ms"
                        // 发送到 Prometheus（通过 Pushgateway）
                        sh """
                            echo "jenkins_stage_duration_ms{job='${JOB_NAME}',stage='Build'} ${duration}" | \
                            curl --data-binary @- http://pushgateway:9091/metrics/job/jenkins/instance/${JOB_NAME}
                        """
                    }
                }
            }
        }
    }
}
```

## 4.2 构建状态跟踪

```groovy
// 跟踪构建状态和指标
def trackBuildMetrics() {
    script {
        def metrics = [
            job: env.JOB_NAME,
            buildNumber: env.BUILD_NUMBER,
            branch: env.BRANCH_NAME,
            result: currentBuild.result ?: 'SUCCESS',
            duration: currentBuild.duration,
            timestamp: currentBuild.startTimeInMillis,
            node: env.NODE_NAME
        ]

        // 发送到日志系统（Elasticsearch）
        sh """
            curl -X POST http://elasticsearch:9200/jenkins-builds/_doc \
                -H 'Content-Type: application/json' \
                -d '${groovy.json.JsonOutput.toJson(metrics)}'
        """
    }
}

pipeline {
    agent any
    stages { /* ... */ }
    post {
        always {
            trackBuildMetrics()
        }
    }
}
```

# 5. 外部监控集成

## 5.1 ELK 集成（Elasticsearch + Logstash + Kibana）

```yaml
# Logstash 配置收集 Jenkins 日志
input {
  file {
    path => "/var/log/jenkins/jenkins.log"
    type => "jenkins"
    start_position => "beginning"
    codec => multiline {
      pattern => "^\d{4}-\d{2}-\d{2}"
      negate => true
      what => "previous"
    }
  }
}

filter {
  if [type] == "jenkins" {
    grok {
      match => {
        "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} \[%{DATA:logger}\] %{GREEDYDATA:message}"
      }
    }
    date {
      match => ["timestamp", "ISO8601"]
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "jenkins-logs-%{+YYYY.MM.dd}"
  }
}
```

```groovy
// Pipeline 中直接发送日志到 ELK
stage('Send Logs') {
    steps {
        script {
            def logEntry = [
                '@timestamp': new Date().toInstant().toString(),
                job: env.JOB_NAME,
                buildNumber: env.BUILD_NUMBER,
                level: 'INFO',
                message: "Build completed: ${currentBuild.result}",
                duration: currentBuild.duration,
                node: env.NODE_NAME
            ]
            sh """
                curl -X POST http://elasticsearch:9200/jenkins-builds/_doc \
                    -H 'Content-Type: application/json' \
                    -d '${groovy.json.JsonOutput.toJson(logEntry)}'
            """
        }
    }
}
```

## 5.2 Datadog 集成

通过 Datadog Plugin 将 Jenkins 指标发送到 Datadog：

```yaml
# Jenkins > Manage Jenkins > Configure System > Datadog Plugin
Datadog:
  Datadog Agent Host: 172.17.0.1
  Datadog Agent Port: 8125
  Datadog API URL: https://api.datadoghq.com
  Datadog API Key: ${DATADOG_API_KEY}

  # 收集的指标
  Collect Build Logs: true
  Collect Pipeline Stage Metrics: true
  Collect System Metrics: true
  Collect Agent Metrics: true
```

```
Datadog Dashboards 中的 Jenkins Metric 示例：
├── jenkins.job.duration（构建耗时）
├── jenkins.job.success（成功数）
├── jenkins.job.failure（失败数）
├── jenkins.queue.size（队列大小）
├── jenkins.executor.count（Executor 数）
├── jenkins.executor.free（空闲 Executor 数）
└── jenkins.node.health（节点健康状态）
```

# 6. Health Check

## 6.1 Health Check 端点

Jenkins 提供多种健康检查方式：

```bash
# 基本可用性
curl -I http://jenkins:8080/login

# Metrics 插件的 Health Check
curl -u admin:token http://jenkins:8080/metrics/healthcheck
# 返回:
# {
#   "deadlocks": {"healthy": true},
#   "plugins": {"healthy": true, "plugins": {...}},
#   "disk-space": {"healthy": true, "message": "134.5GB"}
# }

# 自定义健康检查
curl -u admin:token http://jenkins:8080/instance/health
```

## 6.2 健康检查脚本

```bash
#!/bin/bash
# jenkins-health-check.sh

JENKINS_URL="http://jenkins:8080"
AUTH="admin:api_token"

echo "=== Jenkins Health Check ==="

# 1. 检查服务是否可用
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://jenkins:8080/login)
if [ "$HTTP_CODE" == "200" ]; then
    echo "✓ Jenkins is accessible"
else
    echo "✗ Jenkins is not accessible (HTTP $HTTP_CODE)"
    exit 1
fi

# 2. 检查 Metrics 健康
HEALTH=$(curl -s -u $AUTH $JENKINS_URL/metrics/healthcheck)
if echo "$HEALTH" | grep -q '"healthy": true'; then
    echo "✓ Health check passed"
else
    echo "✗ Health check failed: $HEALTH"
fi

# 3. 检查磁盘空间
DISK=$(curl -s -u $AUTH $JENKINS_URL/metrics/healthcheck | \
    python -c "import sys,json; print(json.load(sys.stdin).get('disk-space',{}).get('message','unknown'))")
echo "✓ Disk space: $DISK"

# 4. 检查队列长度
QUEUE=$(curl -s -u $AUTH $JENKINS_URL/metrics/prometheus | \
    grep "^jenkins_queue_size" | awk '{print $2}')
echo "✓ Build queue size: $QUEUE"

# 5. 检查插件状态
PLUGINS_HEALTHY=$(curl -s -u $AUTH $JENKINS_URL/metrics/healthcheck | \
    python -c "import sys,json; print(json.load(sys.stdin)['plugins']['healthy'])")
echo "✓ Plugins healthy: $PLUGINS_HEALTHY"
```

# 7. 性能监控

## 7.1 性能指标采集

```groovy
// Pipeline 步骤级别性能采集
def measureStage(String stageName, Closure body) {
    def start = System.currentTimeMillis()
    try {
        body.call()
    } finally {
        def elapsed = System.currentTimeMillis() - start
        echo "[PERF] Stage '${stageName}' took ${elapsed}ms"
        // 保存到 Prometheus Pushgateway
        sh """
            cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/jenkins_stage
            # HELP jenkins_stage_duration_ms Stage duration in milliseconds
            # TYPE jenkins_stage_duration_ms gauge
            jenkins_stage_duration_ms{job="${env.JOB_NAME}",stage="${stageName}",build="${env.BUILD_NUMBER}"} ${elapsed}
            EOF
        """
    }
}

// 使用
stage('Build') {
    steps {
        script {
            measureStage('Build') {
                sh 'mvn clean package'
            }
        }
    }
}
```

## 7.2 性能优化指标

| 指标 | 正常范围 | 警告阈值 | 临界阈值 | 优化措施 |
|------|---------|---------|---------|---------|
| 队列大小 | < 5 | 5-20 | > 20 | 增加 Agent |
| 队列等待时间 | < 30s | 30s-5m | > 5m | 增加 Agent/优化流水线 |
| 构建耗时 | < 10m | 10-30m | > 30m | 优化构建步骤/增加资源 |
| JVM 堆使用率 | < 70% | 70-85% | > 85% | 增加 -Xmx/检查内存泄漏 |
| GC 暂停时间 | < 200ms | 200-1000ms | > 1s | 优化 GC 参数/增加堆大小 |
| 磁盘使用率 | < 70% | 70-85% | > 85% | 清理构建历史/增加磁盘 |
| 插件响应时间 | < 100ms | 100-500ms | > 500ms | 检查插件兼容性/禁用问题插件 |
| Agent 离线率 | < 5% | 5-20% | > 20% | 检查网络和 Agent 配置 |

# 8. 构建日志管理

## 8.1 日志保留策略

```groovy
// Pipeline 级别的日志保留设置
pipeline {
    agent any
    options {
        buildDiscarder(
            logRotator(
                numToKeepStr: '30',           // 保留最近 30 次构建
                artifactNumToKeepStr: '5',    // artifact 保留 5 次
                daysToKeepStr: '90',          // 保留 90 天
                artifactDaysToKeepStr: '30'   // artifact 保留 30 天
            )
        )
    }
    stages { /* ... */ }
}
```

## 8.2 日志清理

```bash
# 手动清理构建日志
# 删除指定天数前的构建记录
java -jar jenkins-cli.jar -s http://localhost:8080 \
  delete-builds my-job 1-1000

# 通过 Groovy 脚本批量清理
java -jar jenkins-cli.jar -s http://localhost:8080 \
  groovy = << 'EOF'
import jenkins.model.Jenkins
import hudson.model.Job

Jenkins.instance.allItems(Job.class).each { job ->
    println "Processing ${job.fullName}"
    job.builds.each { build ->
        if (build.time.time < System.currentTimeMillis() - 90 * 24 * 60 * 60 * 1000L) {
            println "Deleting build #${build.number} from ${job.fullName}"
            build.delete()
        }
    }
}
EOF
```

## 8.3 日志存储优化

```yaml
# 减少日志存储的配置
1. 限制日志输出长度
   Jenkins > Configure System > Console Output:
   - Maximum lines to keep: 50000

2. 外部日志存储
   - 将日志发送到 ELK/Splunk
   - Jenkins 不长期本地存储日志

3. Pipeline 日志优化
   - 使用 sh(label:'description', script:'...') 隐藏命令细节
   - 避免大量 echo 输出
   - 使用 hide-sensitive-logs 插件保护敏感信息
```

# 9. 告警通知

## 9.1 构建失败告警

```groovy
// 构建失败时的多层次告警
post {
    failure {
        script {
            // 第一次失败：邮件通知
            emailext(
                to: 'dev-team@company.com',
                subject: "[FAIL] ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build failed: ${env.BUILD_URL}"
            )

            // 连续失败：升级告警
            def previousBuild = currentBuild.previousBuild
            if (previousBuild?.result == 'FAILURE') {
                // 连续两次失败 → Slack 告警
                slackSend(
                    color: 'danger',
                    message: "🚨 *连续失败*: ${env.JOB_NAME} #${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"
                )

                // 连续三次失败 → PagerDuty
                if (previousBuild.previousBuild?.result == 'FAILURE') {
                    sh """
                        curl -X POST https://events.pagerduty.com/v2/enqueue \
                            -H 'Content-Type: application/json' \
                            -d '{
                                "routing_key": "${PD_ROUTING_KEY}",
                                "event_action": "trigger",
                                "payload": {
                                    "summary": "Jenkins job ${env.JOB_NAME} failed 3 times",
                                    "severity": "critical",
                                    "source": "jenkins"
                                }
                            }'
                    """
                }
            }
        }
    }
}
```

## 9.2 Slow Build 告警

```groovy
// 检查构建是否比历史平均慢
def checkBuildPerformance() {
    def currentDuration = currentBuild.duration
    def previousBuilds = currentBuild.previousBuild
    def totalDuration = 0
    def count = 0

    // 计算最近 10 次构建的平均耗时
    for (def i = 0; i < 10 && previousBuilds != null; i++) {
        totalDuration += previousBuilds.duration
        count++
        previousBuilds = previousBuilds.previousBuild
    }

    if (count > 0) {
        def avgDuration = totalDuration / count
        def threshold = avgDuration * 2  // 2 倍阈值

        if (currentDuration > threshold) {
            echo "[WARN] Build is significantly slower than average"
            echo "  Current: ${currentDuration}ms, Average: ${avgDuration}ms"

            slackSend(
                color: 'warning',
                message: "⚠️ *Slow Build*: ${env.JOB_NAME} took ${currentDuration}ms (avg: ${avgDuration}ms)"
            )
        }
    }
}

pipeline {
    agent any
    stages { /* ... */ }
    post {
        success {
            script { checkBuildPerformance() }
        }
    }
}
```
