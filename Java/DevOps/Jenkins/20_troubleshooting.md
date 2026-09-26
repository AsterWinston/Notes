
# 1. 启动问题

## 1.1 Jenkins 启动失败

```bash
# 检查启动日志
tail -100 /var/log/jenkins/jenkins.log
journalctl -u jenkins -n 100

# 常见原因
1. Java 版本不兼容
   ❌ 错误: "Unsupported class file major version"
   ✅ 解决: 使用正确 Java 版本运行
   java -version  # 需要 Java 11 或 17

2. JENKINS_HOME 权限问题
   ❌ 错误: "Access is denied" 或 "Permission denied"
   ✅ 解决: 修复目录权限
   chown -R jenkins:jenkins /var/lib/jenkins
   chmod -R 755 /var/lib/jenkins

3. 端口被占用
   ❌ 错误: "Failed to start Jenkins" "Address already in use"
   ✅ 解决: 修改端口或释放端口
   netstat -tlnp | grep 8080
   # 修改端口：在 /etc/default/jenkins 中设置 HTTP_PORT=8081

4. 磁盘空间不足
   ❌ 错误: "No space left on device"
   ✅ 解决: 清理磁盘
   df -h
   du -sh /var/lib/jenkins/*
```

## 1.2 Jenkins 启动慢

```yaml
# 排查步骤
1. 检查启动日志
   tail -f /var/log/jenkins/jenkins.log | grep -E "(WARNING|ERROR|SEVERE)"

2. 检查插件加载
   # 禁用不必要的插件
   mv /var/lib/jenkins/plugins/unused-plugin.hpi /tmp/

3. 检查网络连接
   # Jenkins 启动时需要连接 update center
   # 如果网络慢，配置代理或离线模式
```

# 2. 构建失败问题

## 2.1 Pipeline 语法错误

```groovy
// ❌ 常见错误
// 1. 缺少必要的指令
pipeline {
    agent any
    // stages 指令缺失 ← 错误
    stage('Build') {
        steps { echo 'Building' }
    }
}
// ✅ 正确
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { echo 'Building' }
        }
    }
}

// 2. steps 中缺少 step
stage('Build') {
    steps {
        'mvn clean package'  // ← 错误：缺少 sh
    }
}
// ✅ 正确
stage('Build') {
    steps {
        sh 'mvn clean package'
    }
}

// 3. 不必要的 Groovy 语法
stage('Test') {
    steps {
        if (env.BRANCH_NAME == 'main') {  // ← 错误：需要 script 包裹
            echo 'Main branch'
        }
    }
}
// ✅ 正确
stage('Test') {
    steps {
        script {
            if (env.BRANCH_NAME == 'main') {
                echo 'Main branch'
            }
        }
    }
}

// 4. 并行中的嵌套
stage('Tests') {
    parallel {
        stage('Unit') { steps { sh 'mvn test' } }
        stage('Integration') { steps { sh 'mvn verify' } }
    }
    // parallel 必须在 stages 内，不能在 steps 内
}
```

## 2.2 Git 检出错误

```bash
# ❌ 错误: "Host key verification failed"
# ✅ 解决: 添加到 known_hosts
ssh-keyscan -H github.com >> /home/jenkins/.ssh/known_hosts

# ❌ 错误: "Authentication failed"
# ✅ 解决: 检查凭据
# 1. 确认凭据 ID 是否正确
# 2. 确认 Token 未过期
# 3. 测试凭据
git clone https://user:token@github.com/org/repo.git

# ❌ 错误: "Could not read from remote repository"
# ✅ 解决: 检查仓库权限和 URL

# ❌ 错误: "SSL certificate problem"
# ✅ 解决: 关闭 SSL 验证（不推荐）或配置正确证书
git config --global http.sslVerify false
```

## 2.3 Agent 相关问题

```bash
# ❌ 错误: "Agent disconnected" / "Agent went offline"
# ✅ 排查步骤:
1. 检查 Agent 日志
   tail -50 /var/log/jenkins/agent.log

2. 检查网络连通性
   ping jenkins-master
   telnet jenkins-master 50000

3. 检查 Agent 资源
   free -m
   df -h
   top

4. 重启 Agent
   systemctl restart jenkins-agent
   # 或 JNLP Agent
   java -jar agent.jar -url http://jenkins:8080 -secret xxx -name agent1

# ❌ 错误: "Node is not known to Jenkins"
# ✅ 解决: 重新注册 Agent
# 删除旧节点配置，重新连接

# ❌ 错误: "No executor available" / "All nodes are offline"
# ✅ 解决: 增加 Agent 或检查 Agent 配置
```

# 3. 性能问题

## 3.1 Jenkins 响应慢

```yaml
# 排查步骤
1. 检查 JVM 堆内存
   # 通过 Metrics 插件或 JMX 查看
   curl -u admin:token http://jenkins:8080/metrics/

2. 检查 GC 情况
   # 查看 GC 日志
   grep "GC pause" /var/log/jenkins/gc.log | tail -20

3. 检查构建队列
   curl -u admin:token http://jenkins:8080/queue/api/json

4. 检查插件
   # 禁用不必要的插件
   # 更新过时的插件

5. 检查 JENKINS_HOME 大小
   du -sh /var/lib/jenkins
   du -sh /var/lib/jenkins/jobs/*/builds  # 构建历史大小

# 优化方案
- 增加 JVM 堆内存
- 清理构建历史
- 增加 Agent 数量
- 使用 G1GC（默认已在 Java 9+ 启用）
```

## 3.2 构建慢

```groovy
// 1. 分析构建耗时
stage('Build') {
    steps {
        // 记录各步骤耗时
        script {
            def start = System.currentTimeMillis()
            sh 'mvn clean package'
            def elapsed = System.currentTimeMillis() - start
            echo "Maven build took: ${elapsed}ms"
        }
    }
}

// 2. 检查是否是 Git 拉取慢
// ✅ 使用浅克隆
checkout([$class: 'GitSCM',
    extensions: [[$class: 'CloneOption', depth: 1, shallow: true]]
])

// 3. 检查是否是测试慢
// ✅ 并行测试
stage('Tests') {
    parallel {
        stage('Module A') { steps { sh 'mvn test -pl module-a' } }
        stage('Module B') { steps { sh 'mvn test -pl module-b' } }
    }
}
```

## 3.3 磁盘空间不足

```bash
# 检查磁盘
df -h
du -sh /var/lib/jenkins/* | sort -rh | head -10

# 主要占用空间
# 1. 构建历史
du -sh /var/lib/jenkins/jobs/*/builds/

# 2. 构建产物
du -sh /var/lib/jenkins/jobs/*/builds/*/archive/

# 3. 日志
du -sh /var/lib/jenkins/logs/

# 4. 工作空间
du -sh /var/lib/jenkins/workspace/

# 清理方法
# 1. 调整构建丢弃策略（在 Job 配置中）
# 2. 批量删除旧构建
java -jar jenkins-cli.jar -s http://localhost:8080 \
  delete-builds my-job 1-500

# 3. 清理工作空间
curl -X POST -u admin:token \
  http://localhost:8080/job/my-job/doWipeOutWorkspace

# 4. 清理日志
find /var/lib/jenkins/logs -name "*.log" -mtime +30 -delete
```

# 4. 插件问题

## 4.1 插件升级后兼容性问题

```bash
# 症状：插件升级后 Jenkins 无法启动或功能异常

# 排查步骤
1. 查看启动日志
   tail -100 /var/log/jenkins/jenkins.log

2. 进入安全模式
   # 创建文件，重启后进入安全模式（仅加载核心插件）
   touch /var/lib/jenkins/jenkins.install.InstallUtil.lastExecVersion

3. 在安全模式下禁用/降级问题插件
   mv /var/lib/jenkins/plugins/problem-plugin.hpi /tmp/

4. 重新启动 Jenkins
   systemctl restart jenkins

# 预防措施
- 先在测试环境验证插件升级
- 升级前备份 JENKINS_HOME
- 记录插件的版本号
```

## 4.2 插件加载失败

```bash
# 症状：插件未加载或加载失败
# 日志中看到：Failed to load plugin XXX

# 解决方法
1. 检查插件依赖
   # 某些插件依赖其他插件，需要一起安装

2. 下载正确的版本
   # 确保插件版本与 Jenkins 版本兼容

3. 手动安装插件
   # 下载 .hpi 文件
   wget https://updates.jenkins.io/download/plugins/git/5.2.0/git.hpi
   mv git.hpi /var/lib/jenkins/plugins/
   chown jenkins:jenkins /var/lib/jenkins/plugins/git.hpi
   systemctl restart jenkins
```

# 5. 网络问题

## 5.1 Agent 连接问题

```bash
# ❌ 症状：Agent 无法连接到 Master

# 排查
1. 检查 Master 端口
   netstat -tlnp | grep 50000
   # JNLP Agent 需要 50000 端口开放

2. 检查防火墙
   iptables -L -n | grep 50000
   # 确保 50000 端口已开放

3. 检查 NAT/路由
   # 如果使用 JNLP，确保 Agent 可以访问 Master:50000

4. 检查 DNS
   nslookup jenkins-master
   # Agent 需要能解析 Master 主机名

# 使用 WebSocket 替代（推荐）
# WebSocket 使用 8080 端口，无需额外开放端口
java -jar agent.jar -url http://jenkins:8080 \
  -secret xxx -name agent1 -webSocket
```

## 5.2 Update Center 连接问题

```bash
# ❌ 症状：插件安装/更新失败

# 排查
1. 测试网络连接
   curl -I https://updates.jenkins.io

2. 配置代理
   # Jenkins > Manage Jenkins > Plugin Manager > Advanced
   # HTTP Proxy Configuration

3. 使用国内镜像
   # 修改 Update Center URL
   # Jenkins > Manage Jenkins > Plugin Manager > Advanced
   # Update Site: https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json

4. 离线安装
   # 下载 .hpi 文件手动上传安装
```

# 6. 安全相关

## 6.1 忘记管理员密码

```bash
# 方法一：通过脚本重置
cat << 'EOF' > /tmp/reset_password.groovy
import jenkins.model.*
import hudson.security.*

def instance = Jenkins.instance
def realm = instance.getSecurityRealm()
def user = realm.getUser('admin')
user.setPassword('newpassword')
user.save()
println "Password reset successfully"
EOF

# 上传并执行
java -jar jenkins-cli.jar -s http://localhost:8080 \
  groovy = @/tmp/reset_password.groovy

# 方法二：直接修改用户配置
# 编辑 users/admin/config.xml
# 使用在线 bcrypt 工具生成新密码哈希
# 替换 <passwordHash> 中的值
```

## 6.2 凭据无法解密

```bash
# ❌ 症状：迁移 Jenkins 后凭据无法使用

# 原因
master.key 或 hudson.util.Secret 丢失或更改

# 解决方法
1. 从备份中恢复 secrets/ 目录
   # /var/lib/jenkins/secrets/master.key
   # /var/lib/jenkins/secrets/hudson.util.Secret

2. 如果没有备份：
   # 无法恢复旧凭据，需要重新创建
   # 删除旧的 secrets/ 目录，重启 Jenkins 后重建

# 预防
- 始终备份 secrets/ 目录
- 迁移时必须同时迁移 secrets/
```

# 7. Jenkins 状态检查清单

## 7.1 日常检查

```bash
#!/bin/bash
# jenkins-diagnostics.sh

echo "=== Jenkins Diagnostics ==="
echo ""

# 1. 检查进程
echo "1. Process:"
ps aux | grep jenkins | grep -v grep
echo ""

# 2. 检查端口
echo "2. Port:"
netstat -tlnp | grep -E "(8080|50000)"
echo ""

# 3. 检查磁盘
echo "3. Disk:"
df -h /var/lib/jenkins
echo ""

# 4. 检查内存
echo "4. Memory:"
free -m
echo ""

# 5. 检查文件描述符
echo "5. File descriptors:"
JENKINS_PID=$(pgrep -f jenkins)
if [ -n "$JENKINS_PID" ]; then
    ls /proc/$JENKINS_PID/fd 2>/dev/null | wc -l
fi
echo ""

# 6. 检查 Agent 连接
echo "6. Agents:"
curl -s -u admin:token \
  "http://localhost:8080/computer/api/json?tree=computer[displayName,offline]" | \
  jq -r '.computer[] | "  \(.displayName): offline=\(.offline)"'
echo ""

# 7. 检查队列
echo "7. Queue:"
QUEUE_SIZE=$(curl -s -u admin:token \
  http://localhost:8080/queue/api/json | jq '.items | length')
echo "  Queue size: $QUEUE_SIZE"
echo ""

# 8. 检查最近构建状态
echo "8. Recent builds (last 10):"
curl -s -u admin:token \
  "http://localhost:8080/api/json?tree=jobs[name,color,lastBuild[number,result,timestamp]]" | \
  jq -r '.jobs[] | select(.lastBuild != null) | "  \(.name): #\(.lastBuild.number) \(.lastBuild.result)"' | \
  head -10
```

## 7.2 故障快速定位

```yaml
# 常见问题快速定位表
问题                  → 检查位置                      → 常见原因
Jenkins 无法启动       → /var/log/jenkins/jenkins.log  → Java 版本/端口占用/权限
构建无法开始           → Build Queue                  → 无可用 Agent
构建失败               → Console Output               → 语法错误/凭据问题/网络
Agent 离线             → Agent 状态页面               → 网络/资源/JNLP 问题
插件无法安装           → Plugin Manager               → 网络/依赖/版本兼容
Jenkins 响应慢         → Metrics/JMX                  → 内存不足/GC/磁盘
凭据失效               → Credentials                  → master.key 变更
权限异常               → Security Realm               → 用户/角色配置错误
磁盘空间不足           → df -h                        → 构建历史/日志/产物
API 调用失败           → crumbIssuer                  → CSRF Token 缺失
定时构建不触发         → Trigger 配置                  → Cron 语法错误/时区
```

## 7.3 报告问题

```yaml
# 向社区报告问题时的信息模板
问题描述:
  - Jenkins 版本: 2.440.3
  - Java 版本: OpenJDK 17.0.9
  - 操作系统: Ubuntu 22.04 / CentOS 7
  - 插件列表: (相关的插件和版本)
  - 错误日志: (关键错误信息)

复现步骤:
  1. ...
  2. ...
  3. ...

预期行为:
  ...

实际行为:
  ...

临时解决方案:
  ...

# 参考资源
- Jenkins 社区: https://community.jenkins.io/
- 问题跟踪: https://issues.jenkins.io/
- 插件文档: https://plugins.jenkins.io/
```
