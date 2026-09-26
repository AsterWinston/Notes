
# 1. 高可用架构概述

Jenkins 的高可用架构有多种实现方式，从简单到复杂：

```
┌────────────────────────────────────────────────────────────┐
│           Jenkins 高可用方案对比                            │
├────────────┬──────────────┬──────────────┬────────────────-┤
│   方案      │  复杂度       │  可用性       │   成本         │
├────────────┼──────────────┼──────────────┼────────────────-┤
│ 定时备份    │ 低           │ 一般          │ 低             │
│ Active/Passive │ 中       │ 高            │ 中             │
│ Active/Active  │ 高       │ 非常高        │ 高             │
│ K8s 部署    │ 中           │ 高            │ 中             │
└────────────┴──────────────┴──────────────┴────────────────-┘
```

## 1.1 单点故障分析

```
┌─────────────────────────────────────────┐
│          单节点 Jenkins 的风险           │
│                                         │
│ Master 宕机 → 所有构建停止               │
│    ├── 正在执行的 Pipeline 丢失状态      │
│    ├── 无法触发新构建                    │
│    └── Web UI 和 API 不可用              │
│                                         │
│ 磁盘故障 → JENKINS_HOME 丢失            │
│    ├── 所有配置丢失                      │
│    ├── 插件数据丢失                      │
│    └── 构建历史丢失                      │
└─────────────────────────────────────────┘
```

# 2. Active/Passive 高可用

## 2.1 NFS 方式

使用共享存储 + 负载均衡实现 Active/Passive 切换：

```
                    ┌──────────┐
                    │  Load    │
                    │  Balancer│
                    └────┬─────┘
                         │
              ┌──────────┼──────────┐
              │          │          │
        ┌─────▼────┐ ┌──▼─────┐    │
        │ Jenkins  │ │Jenkins │    │
        │  Active  │ │Passive │    │
        │  Master  │ │Master  │    │
        └─────┬────┘ └───┬────┘    │
              │          │          │
              └────┬─────┘          │
                   │                │
             ┌─────▼──────┐   ┌────▼────┐
             │  NFS/EFS   │   │  DB     │
             │ (Config)   │   │(可选)   │
             └────────────┘   └─────────┘
```

**实现要点**：

```yaml
# 1. NFS 共享存储
# 两台 Jenkins Master 共享同一 JENKINS_HOME
# 使用 NFS、EFS、GlusterFS 等

# 2. 配置 NFS 导出
/etc/exports:
/data/jenkins_home 10.0.1.0/24(rw,sync,no_subtree_check,no_root_squash)

# 3. 两台 Master 挂载相同路径
mount -t nfs nfs-server:/data/jenkins_home /var/lib/jenkins

# 4. 负载均衡器健康检查
# 检查 /metrics/healthcheck 或 /login
haproxy:
  backend jenkins
    option httpchk GET /login
    server jenkins-1 10.0.1.10:8080 check
    server jenkins-2 10.0.1.11:8080 check backup
```

## 2.2 共享存储注意事项

```yaml
# NFS 共享存储的风险与应对
风险:
  - NFS 锁冲突（并发写 config.xml）
  - 网络分区导致脑裂
  - NFS 性能瓶颈

应对措施:
  1. Active/Passive 模式（避免并发写）
  2. 使用集群文件系统（GlusterFS, Ceph）
  3. NFS 版本 ≥ 4.0（更好的锁机制）
  4. 监控 NFS 延迟和可用性

# 推荐的 NFS 挂载参数
mount -t nfs -o rw,hard,intr,noatime,timeo=600,retrans=5 \
  nfs-server:/data/jenkins_home /var/lib/jenkins
```

# 3. Kubernetes 高可用

## 3.1 K8s 上的 Jenkins 高可用

```yaml
# Jenkins Master 部署（带 PDB）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1  # Active/Passive 模式
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      terminationGracePeriodSeconds: 120
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts-jdk17
        ports:
        - containerPort: 8080
        - containerPort: 50000
        env:
        - name: JAVA_OPTS
          value: "-Xms2048m -Xmx4096m"
        volumeMounts:
        - mountPath: /var/jenkins_home
          name: jenkins-home
        livenessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 120
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
      volumes:
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins-home-pvc
---
# PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: jenkins-pdb
  namespace: jenkins
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: jenkins
---
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-home-pvc
  namespace: jenkins
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: ebs-gp3  # 使用 AWS EBS 或等效
```

## 3.2 Pod 恢复机制

```yaml
# K8s 自动恢复流程
1. Liveness Probe 失败 → K8s 重启容器
2. Node 宕机 → Pod 被重新调度到健康节点
3. Deployment 滚动更新 → 零停机更新
4. PDB 保证最小可用副本

# 注意
# - 由于使用 ReadWriteOnce 的 PVC，K8s 上的 Jenkins 仍为 Active/Passive
# - 需要 ReadWriteMany 的存储才能实现 Active/Active
```

# 4. Agent 扩展策略

## 4.1 动态 Agent 扩展

Jenkins 支持多种方式动态扩展 Agent：

```yaml
# Agent 扩展方式对比
扩展方式:
  ├── 静态 Agent（传统方式）
  │   ├── 手动添加永久节点
  │   ├── 维护成本高
  │   └── 资源利用率低
  │
  ├── Docker Cloud
  │   ├── 按需创建容器
  │   ├── 环境隔离
  │   └── 构建完自动销毁
  │
  ├── Kubernetes Cloud
  │   ├── 动态创建 Pod
  │   ├── 资源限制灵活
  │   ├── 自动伸缩
  │   └── 推荐用于云原生环境
  │
  └── EC2 Fleet / Azure VM
      ├── 动态创建虚拟机
      ├── 适合需要完整 OS 的构建
      └── 成本较高
```

## 4.2 Kubernetes Agent 弹性伸缩

```yaml
# Kubernetes Cloud 配置建议
jenkins:
  clouds:
    - kubernetes:
        name: "k8s-cloud"
        # 最大同时运行的 Agent Pod
        containerCapStr: "100"
        # Pod 模板
        templates:
          - name: "default-agent"
            label: "k8s-agent"
            # 资源请求
            containers:
              - name: "jnlp"
                image: "jenkins/inbound-agent:latest-jdk17"
                resourceRequestCpu: "500m"
                resourceRequestMemory: "512Mi"
                resourceLimitCpu: "2"
                resourceLimitMemory: "2Gi"
            # 空闲回收
            idleMinutes: 10
            # 主动终止时间
            activeDeadlineSeconds: 3600
            # 保留策略
            podRetention: "never"

# 配合 Cluster Autoscaler
# 当 Agent Pod 因资源不足 Pending 时，自动扩展 K8s 节点
Cluster Autoscaler:
  min_nodes: 3
  max_nodes: 20
  scale_down_unneeded_time: 10m
```

## 4.3 Agent 负载均衡

```groovy
// Pipeline 中使用标签策略分配负载
pipeline {
    agent none
    stages {
        // 使用最空闲的 Agent
        stage('Build') {
            agent { label 'linux' }  // 任意 linux Agent
            steps {
                sh 'mvn clean package'
            }
        }
        // 使用特定资源 Agent
        stage('Heavy Build') {
            agent { label 'high-memory' }
            steps {
                sh './heavy-build.sh'
            }
        }
        // 使用 K8s Agent
        stage('Cloud Build') {
            agent { label 'k8s-agent' }
            steps {
                sh './cloud-build.sh'
            }
        }
    }
}
```

# 5. 数据备份与恢复

## 5.1 备份策略

```yaml
# 分层备份策略
备份分层:
  ├── 每日备份（关键数据）
  │   ├── config.xml
  │   ├── jobs/*/config.xml
  │   ├── secrets/
  │   ├── plugins/
  │   └── users/
  │
  ├── 每周备份（中等数据）
  │   ├── 每日备份的内容
  │   └── builds/（最近的构建记录）
  │
  └── 每月备份（完整备份）
      ├── 所有数据
      └── 排除 workspace/ 和 caches/

保留策略:
  ├── 每日备份 → 保留 7 天
  ├── 每周备份 → 保留 4 周
  └── 每月备份 → 保留 12 个月
```

## 5.2 自动备份脚本

```bash
#!/bin/bash
# jenkins-backup.sh

JENKINS_HOME=/var/lib/jenkins
BACKUP_BASE=/data/backup/jenkins
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_TYPE=${1:-daily}  # daily, weekly, monthly

# 根据备份类型设置排除和保留策略
case $BACKUP_TYPE in
    daily)
        EXCLUDE="--exclude=workspace --exclude=builds --exclude=logs --exclude=caches"
        RETENTION=7
        ;;
    weekly)
        EXCLUDE="--exclude=workspace --exclude=logs --exclude=caches"
        RETENTION=4
        ;;
    monthly)
        EXCLUDE="--exclude=workspace --exclude=logs --exclude=caches"
        RETENTION=12
        ;;
esac

BACKUP_DIR=$BACKUP_BASE/$BACKUP_TYPE
mkdir -p $BACKUP_DIR

# 执行备份
tar $EXCLUDE \
    -czf $BACKUP_DIR/jenkins_${BACKUP_DATE}.tar.gz \
    -C $(dirname $JENKINS_HOME) \
    $(basename $JENKINS_HOME)

# 记录备份信息
echo "Backup completed: $BACKUP_DIR/jenkins_${BACKUP_DATE}.tar.gz"
echo "Size: $(du -h $BACKUP_DIR/jenkins_${BACKUP_DATE}.tar.gz | cut -f1)"

# 清理旧备份
find $BACKUP_DIR -name "jenkins_*.tar.gz" -mtime +${RETENTION} -delete

# 验证备份完整性
tar tzf $BACKUP_DIR/jenkins_${BACKUP_DATE}.tar.gz > /dev/null
if [ $? -eq 0 ]; then
    echo "Backup integrity check: PASSED"
else
    echo "Backup integrity check: FAILED"
    exit 1
fi
```

## 5.3 thinBackup 插件配置

```yaml
# thinBackup 插件配置
Manage Jenkins > ThinBackup > Settings:
  Backup directory: /data/backup/jenkins
  Full backup every: 7 days
  Differential backup every: 1 day
  Max number of backup sets: 12
  Backup strategy:
    Include:
      - **/config.xml
      - secrets/
      - plugins/
    Exclude:
      - **/workspace/**
      - **/builds/**
      - **/logs/**
      - **/caches/**
```

## 5.4 恢复流程

```bash
#!/bin/bash
# jenkins-restore.sh

RESTORE_FILE=$1
RESTORE_DIR=/tmp/jenkins_restore

if [ -z "$RESTORE_FILE" ]; then
    echo "Usage: $0 <backup-file.tar.gz>"
    exit 1
fi

if [ ! -f "$RESTORE_FILE" ]; then
    echo "Backup file not found: $RESTORE_FILE"
    exit 1
fi

echo "=== Jenkins Restore Procedure ==="

# 1. 停止 Jenkins
echo "1. Stopping Jenkins..."
systemctl stop jenkins

# 2. 备份当前数据（回滚用）
echo "2. Backing up current data..."
mv /var/lib/jenkins /var/lib/jenkins.rollback

# 3. 解压备份
echo "3. Extracting backup..."
mkdir -p $RESTORE_DIR
tar xzf $RESTORE_FILE -C $RESTORE_DIR

# 4. 恢复数据
echo "4. Restoring data..."
mv $RESTORE_DIR/jenkins /var/lib/jenkins

# 5. 设置权限
echo "5. Setting permissions..."
chown -R jenkins:jenkins /var/lib/jenkins

# 6. 启动 Jenkins
echo "6. Starting Jenkins..."
systemctl start jenkins

# 7. 验证
echo "7. Verifying restoration..."
sleep 30  # 等待启动
if curl -s -I http://localhost:8080/login | grep -q "200"; then
    echo "✓ Jenkins restored successfully"
else
    echo "✗ Jenkins restoration failed, rolling back..."
    systemctl stop jenkins
    rm -rf /var/lib/jenkins
    mv /var/lib/jenkins.rollback /var/lib/jenkins
    systemctl start jenkins
fi

# 清理
rm -rf $RESTORE_DIR

echo "=== Restore Complete ==="
```

# 6. 灾难恢复

## 6.1 灾难恢复计划

```yaml
# Jenkins 灾难恢复计划
严重级别:
  P0 - 完全宕机（Master 不可用）
  P1 - 部分功能不可用（插件故障、性能退化）
  P2 - 非关键功能异常（UI 显示问题）

恢复流程:
  P0 恢复步骤:
    D-00: 确认故障并通知团队
    D-05: 尝试重启 Jenkins
    D-10: 如果重启失败，切换到备用 Master
    D-20: 验证备用 Master 功能
    D-30: 确认 Agent 连接正常
    D-45: 恢复构建队列
    D-60: 通知团队已恢复

  P1 恢复步骤:
    H-00: 分析问题原因
    H-01: 回滚最近的变更（插件升级、配置修改）
    H-02: 需要时升级/降级插件版本
    H-04: 验证功能正常

预防措施:
  1. 定期备份（每日）
  2. 备份恢复演练（每月）
  3. 配置版本管理（JCasC）
  4. 插件版本锁定
  5. 监控告警配置
```

## 6.2 快速重建

通过 JCasC + Job DSL 可以快速重建 Jenkins：

```yaml
# 快速重建清单
1. 部署新 Jenkins 实例
2. 安装 JCasC 插件
3. 导入配置文件（从 Git）
4. 配置凭据（从 Vault/环境变量注入）
5. 安装插件（从 Plugin Manager / JCasC）
6. 导入 Job 配置（Job DSL）
7. 验证功能
8. 恢复构建历史（可选）
9. 切换 DNS/Load Balancer
```

# 7. 性能扩展

## 7.1 Master 性能优化

```yaml
# Master 性能优化清单
JVM 配置:
  - Xms: 4096m（最小堆）
  - Xmx: 8192m（最大堆）
  - 使用 G1GC
  - GC 日志启用

Web 配置:
  - 禁用不必要的插件
  - 配置线程池大小
  - 启用资源缓存
  - 配置请求超时

数据库配置（如果使用外部数据库）:
  - 连接池大小
  - 查询超时
  - 索引优化

# Jenkins 系统配置优化
Manage Jenkins > Configure System:
  # Executor
  # of executors: 0（Master 不执行构建）

  # SCM
  SCM checkout retry count: 0

  # Workspace
  Workspace clean up: enabled
```

## 7.2 Agent 横向扩展

```groovy
// Agent 推荐配置
// 1. 使用标签区分 Agent 能力
agent label 'k8s-agent'  // 通用 Agent
agent label 'gpu'        // GPU Agent
agent label 'windows'    // Windows Agent
agent label 'high-memory' // 大内存 Agent

// 2. 不同任务使用不同 Agent
pipeline {
    agent none
    stages {
        stage('Quick Unit Tests') {
            agent { label 'k8s-agent' }
            steps { sh 'mvn test' }
        }
        stage('Heavy Integration') {
            agent { label 'high-memory' }
            steps { sh 'mvn verify' }
        }
    }
}
```

## 7.3 Pipeline 性能优化

```groovy
// 1. 避免不必要的 Agent 分配
pipeline {
    agent any  // 如果所有 stage 在同一 Agent 上
    // 不要在每个 stage 中重新指定 agent
    stages {
        stage('Build') { steps { sh 'mvn compile' } }
        stage('Test') { steps { sh 'mvn test' } }
        stage('Package') { steps { sh 'mvn package' } }
    }
}

// ❌ 低效：每个 stage 创建新 Agent
// stage('Build') { agent { label 'linux' } ... }
// stage('Test') { agent { label 'linux' } ... }

// ✅ 高效：复用同一个 Agent
// agent any（Pipeline 级别）

// 2. 使用浅克隆
checkout([
    $class: 'GitSCM',
    extensions: [[$class: 'CloneOption', depth: 1, shallow: true]]
])

// 3. 合理设置构建丢弃策略
options {
    buildDiscarder(logRotator(numToKeepStr: '20'))
}
```

# 8. 实际案例：大规模 Jenkins 部署

## 8.1 企业级部署架构

```
                     ┌──────────────────┐
                     │    DNS/Load      │
                     │    Balancer      │
                     │  (Active/Passive)│
                     └────────┬─────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
        ┌─────▼─────┐  ┌─────▼─────┐   ┌─────▼─────┐
        │  Jenkins  │  │  Jenkins  │   │  Jenkins  │
        │  Master 1 │  │  Master 2 │   │  Master 3 │
        │  (Active) │  │ (Passive) │   │ (Passive) │
        └─────┬─────┘  └─────┬─────┘   └─────┬─────┘
              │              │               │
              └──────┬───────┘               │
                     │                       │
              ┌──────▼──────┐         ┌──────▼──────┐
              │  Shared FS  │         │  Monitoring │
              │  (EFS/NFS)  │         │ Prometheus  │
              └─────────────┘         │ + Grafana   │
                                      └─────────────┘
                              ┌──────────────────┐
                              │   Agent 扩展      │
                              │                  │
                              │   ┌──────────┐   │
                              │   │ K8s Pods │   │
                              │   ├──────────┤   │
                              │   │  Docker  │   │
                              │   ├──────────┤   │
                              │   │   EC2    │   │
                              │   └──────────┘   │
                              └──────────────────┘
```

## 8.2 多数据中心部署

```yaml
# 多数据中心架构
数据中心 1（主）:
  - Jenkins Master Active
  - Agent 集群（K8s）
  - 共享存储（EFS）
  - 数据同步到数据中心 2

数据中心 2（备）:
  - Jenkins Master Passive
  - Agent 集群（K8s）
  - 数据从数据中心 1 同步

# 切换条件
Primary Down → DNS 切换到 Secondary
Secondary 激活 → 启动 Jenkins → 挂载数据 → 接受流量
```
