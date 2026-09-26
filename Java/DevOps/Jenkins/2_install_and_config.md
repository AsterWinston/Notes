
# 1. 环境准备

## 1.1 硬件要求

| 规模 | CPU | 内存 | 磁盘 | 适用场景 |
|------|-----|------|------|---------|
| 小型 | 2核 | 4GB | 50GB | 个人开发/小团队（< 10个Pipeline） |
| 中型 | 4核 | 8GB | 200GB | 中型团队（10~50个Pipeline） |
| 大型 | 8核+ | 16GB+ | 500GB+ | 企业级（50+ Pipeline，大量构建） |

注意：如果使用大量 Agent，Master 本身不需要太多资源，但构建历史、日志和 artifact 会占用磁盘空间。

## 1.2 软件要求

- **Java**：Jenkins 2.440+ LTS 要求 Java 11 或 17（推荐 Java 17）
- **Web浏览器**：Chrome/Firefox/Edge 最新版本
- **操作系统**：Linux（推荐）、Windows、macOS

# 2. 安装方式

## 2.1 方式一：WAR包安装（通用方式）

WAR 包是 Jenkins 最标准的安装方式，可以部署在任何支持 Servlet 3.1+ 的容器中，也可以直接通过 `java -jar` 运行。

```bash
# 下载 Jenkins WAR 包
wget https://get.jenkins.io/war-stable/2.440.3/jenkins.war

# 直接运行（内嵌 Winstone Servlet 容器）
java -jar jenkins.war --httpPort=8080

# 指定参数运行
java -jar jenkins.war \
  --httpPort=8080 \
  --httpsPort=8443 \
  --httpsKeyStore=/path/to/keystore \
  --httpsKeyStorePassword=password \
  --prefix=/jenkins
```

常用启动参数：

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--httpPort` | HTTP 监听端口 | 8080 |
| `--httpListenAddress` | HTTP 监听地址 | 0.0.0.0 |
| `--httpsPort` | HTTPS 监听端口（需配置证书） | 禁用 |
| `--prefix` | URL 前缀 | 空 |
| `--webroot` | Web 资源目录 | `~/.jenkins/war` |
| `--sessionTimeout` | Session 超时时间（分钟） | 30 |

## 2.2 方式二：系统包管理安装（Linux）

```bash
# CentOS/RHEL
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum install -y fontconfig java-17-openjdk jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins

# Ubuntu/Debian
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install -y fontconfig openjdk-17-jre jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

包管理安装的默认配置：

| 配置项 | 默认值 |
|--------|--------|
| 监听端口 | 8080 |
| Jenkins Home | `/var/lib/jenkins` |
| 用户 | `jenkins` |
| 日志文件 | `/var/log/jenkins/jenkins.log` |
| 配置文件 | `/etc/default/jenkins` 或 `/etc/sysconfig/jenkins` |

## 2.3 方式三：Docker安装

```bash
# 拉取 Jenkins 镜像
docker pull jenkins/jenkins:lts-jdk17

# 运行 Jenkins 容器
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e TZ=Asia/Shanghai \
  jenkins/jenkins:lts-jdk17
```

参数说明：
- `-p 8080:8080`：Web 访问端口
- `-p 50000:50000`：JNLP Agent 通信端口
- `-v jenkins_home:/var/jenkins_home`：持久化 Jenkins 数据
- `-v /var/run/docker.sock:/var/run/docker.sock`：允许容器内使用 Docker

**Docker Compose 方式**：

```yaml
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    container_name: jenkins
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - TZ=Asia/Shanghai
      - JAVA_OPTS=-Djava.util.logging.config.file=/var/jenkins_home/log.properties
    restart: unless-stopped

volumes:
  jenkins_home:
```

## 2.4 方式四：Kubernetes安装

使用 Helm Chart 安装：

```bash
# 添加 Jenkins Helm 仓库
helm repo add jenkins https://charts.jenkins.io
helm repo update

# 创建 values.yaml
cat <<EOF > values.yaml
controller:
  imageTag: "lts-jdk17"
  serviceType: NodePort
  servicePort: 8080
  targetPort: 8080
  installPlugins:
    - kubernetes:latest
    - workflow-aggregator:latest
    - git:latest
    - configuration-as-code:latest
  resources:
    requests:
      cpu: "1000m"
      memory: "2Gi"
    limits:
      cpu: "2000m"
      memory: "4Gi"
  javaOpts: "-Xms2048m -Xmx4096m"
  jenkinsUriPrefix: ""
agent:
  enabled: true
  image: jenkins/inbound-agent:latest-jdk17
  resources:
    requests:
      cpu: "500m"
      memory: "512Mi"
persistence:
  enabled: true
  size: "50Gi"
EOF

# 安装 Jenkins
helm install jenkins jenkins/jenkins -f values.yaml

# 获取初始密码
kubectl exec -it svc/jenkins -- cat /var/jenkins_home/secrets/initialAdminPassword

# 获取访问地址
export NODE_PORT=$(kubectl get svc jenkins -o jsonpath='{.spec.ports[0].nodePort}')
export NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
echo http://$NODE_IP:$NODE_PORT
```

# 3. 初始化配置

## 3.1 初次启动

Jenkins 初次启动时自动完成以下流程：

1. 在 `JENKINS_HOME` 目录生成初始配置文件
2. 生成初始管理员密码（存储在 `JENKINS_HOME/secrets/initialAdminPassword`）
3. 打开浏览器访问 `http://localhost:8080`，输入初始密码
4. 进入安装向导

## 3.2 插件安装选择

安装向导提供两个选项：

**a）安装推荐插件（Install suggested plugins）**

推荐插件包含：
- `folders`：文件夹管理 Job
- `pipeline-milestone-step`、`pipeline-stage-step` 等：Pipeline 支持
- `git`：Git 集成
- `blueocean`：Blue Ocean 界面
- `credentials-binding`：凭据绑定
- `email-ext`：邮件扩展通知
- 等约 30+ 插件

**b）选择要安装的插件（Select plugins to install）**

适合有经验的用户，仅安装需要的插件以减小体积和启动时间。

## 3.3 创建管理员用户

安装完插件后，创建第一个管理员用户：
- **用户名**：admin（建议，但可自定义）
- **密码**：强密码（建议 12 位以上，含大小写字母、数字、特殊字符）
- **邮箱**：务必填写真实邮箱，用于密码找回和通知

## 3.4 实例配置

配置 Jenkins URL（实例地址）：
- 默认值：`http://localhost:8080/`
- 生产环境：应配置为反向代理后的地址（如 `https://jenkins.company.com/`）

# 4. Jenkins Home目录结构

`JENKINS_HOME` 是 Jenkins 的核心数据目录，默认位置：

| 安装方式 | 默认路径 |
|----------|---------|
| WAR/jar | `~/.jenkins` |
| 系统包（Linux） | `/var/lib/jenkins` |
| Docker | `/var/jenkins_home` |

目录结构：

```
JENKINS_HOME/
├── config.xml                    # 全局配置文件
├── secrets/                      # 密钥存储目录
│   ├── initialAdminPassword      # 初始管理员密码
│   └── master.key                # Master 加密密钥
├── jobs/                         # Job 配置目录
│   └── <job-name>/
│       ├── config.xml            # Job 配置
│       ├── builds/               # 构建历史
│       │   └── <build-number>/
│       │       ├── build.xml     # 构建元数据
│       │       ├── log           # 构建日志
│       │       ├── changelog.xml # 变更日志
│       │       └── archive/      # 构建产物
│       └── lastSuccessful/       # 最后一次成功构建的符号链接
├── plugins/                      # 插件目录
│   ├── <plugin>.jpi             # 插件文件
│   └── <plugin>/                 # 插件解压目录
├── nodes/                        # Agent 节点配置
├── users/                        # 用户配置
│   └── <username>/
│       └── config.xml
├── updates/                      # 插件更新数据
├── workspace/                    # 默认工作空间目录
├── logs/                         # 日志目录
└── fingerprint/                  # 文件指纹目录
```

# 5. 全局配置

## 5.1 系统配置

通过 **Manage Jenkins > System** 进入，常用配置项：

```yaml
# 关键配置（通过 Web UI 配置）
# #Jenkins 地址：External URL（用于通知和 Agent 通信）
Jenkins Location:
  Jenkins URL: http://jenkins.company.com:8080/
  System Admin e-mail address: admin@company.com

# #系统消息：显示在 Jenkins 首页的横幅
System Message: Welcome to Company CI/CD Platform

# #执行者数量（Master 内置 Agent）
# #生产环境建议设为 0，由外部 Agent 执行构建
# # of executors: 0

# #环境变量
Global properties:
  - name: JAVA_HOME
    value: /usr/lib/jvm/java-17-openjdk
  - name: MAVEN_HOME
    value: /opt/apache-maven-3.9.6
```

## 5.2 全局工具配置

通过 **Manage Jenkins > Tools** 配置构建工具。

**JDK 配置**：

```bash
# 方式一：自动安装（从 Oracle/Adoptium 下载）
JDK:
  Name: JDK17
  JAVA_HOME: /usr/lib/jvm/java-17-openjdk  # 本地路径，如已安装

# 方式二：手动安装后指定路径
Name: JDK17
INSTALL automatically: false
JAVA_HOME: /opt/jdk-17
```

**Maven 配置**：

```bash
Maven:
  Name: Maven3
  Install automatically:
    - Version: 3.9.6 (from Apache)
  # 或手动安装
  Name: Maven3-Local
  MAVEN_HOME: /opt/apache-maven-3.9.6
```

**Gradle 配置**：

```bash
Gradle:
  Name: Gradle8
  Install automatically:
    - Version: 8.5 (from Gradle.org)
```

**Docker 配置**（需要在 Agent 上安装 Docker CLI）：

```bash
Docker:
  Name: Docker
  Docker Host URI: unix:///var/run/docker.sock  # 本地 Docker
  # 或远程 Docker
  Docker Host URI: tcp://docker-host:2375
```

## 5.3 插件管理

通过 **Manage Jenkins > Plugins** 管理插件。

**插件安装命令**（CLI 方式）：

```bash
# 使用 Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080/ install-plugin \
  pipeline-stage-view:latest \
  blueocean:latest \
  git:latest

# 使用 REST API
curl -X POST \
  http://localhost:8080/pluginManager/installNecessaryPlugins \
  -H "Content-Type: application/xml" \
  -d '<install plugin="git@latest"/>'

# 使用 Jenkinsfile 安装
@NonCPS
def installPlugin(String pluginId) {
  // 需要在 Jenkins 脚本审批中允许
  def pluginManager = Jenkins.instance.pluginManager
  def plugin = pluginManager.getPlugin(pluginId)
  if (plugin == null) {
    pluginManager.install(pluginId, true)
  }
}
```

**常用插件清单**：

| 分类 | 插件名称 | 用途 |
|------|---------|------|
| Pipeline | pipeline-model-definition | Declarative Pipeline 支持 |
| Pipeline | workflow-aggregator | Pipeline 聚合插件 |
| Pipeline | pipeline-stage-view | Pipeline 阶段视图 |
| Pipeline | pipeline-utility-steps | Pipeline 工具步骤 |
| SCM | git | Git 集成 |
| SCM | github-integration | GitHub 增强集成 |
| SCM | gitlab-plugin | GitLab 集成 |
| 构建工具 | maven-plugin | Maven 集成 |
| 构建工具 | gradle-plugin | Gradle 集成 |
| 容器 | docker-workflow | Docker 流水线支持 |
| 容器 | kubernetes | Kubernetes 集成 |
| 质量 | sonar | SonarQube 集成 |
| 通知 | email-ext | 邮件扩展通知 |
| 通知 | slack | Slack 通知 |
| 安全 | credentials-binding | 凭据绑定 |
| 安全 | role-strategy | 基于角色的权限控制 |
| 配置 | configuration-as-code | 配置即代码（JCasC） |
| UI | blueocean | Blue Ocean 界面 |
| 监控 | metrics | Prometheus 指标暴露 |
| 备份 | thinBackup | 备份恢复 |

## 5.4 凭据配置

通过 **Manage Jenkins > Credentials** 配置凭据。

Jenkins 支持的凭据类型：

| 类型 | 存储 ID | 使用场景 |
|------|---------|---------|
| Username with password | `username-password` | Git 账号、SSH 密码 |
| SSH Username with private key | `ssh-key` | Git SSH 认证 |
| Secret text | `string` | API Token、密码 |
| Secret file | `file` | 证书文件、配置文件 |
| Certificate | `certificate` | PKCS#12 证书 |
| Docker Host Certificate Authentication | `docker-host` | Docker 认证 |

**凭据存储域**：

- **全局（Global）**：对所有 Job 可见
- **系统（System）**：仅 Jenkins 内部使用
- **文件夹（Folder）**：仅在特定文件夹及其子项中可见

## 5.5 安全配置

通过 **Manage Jenkins > Security** 配置安全策略。

**认证方式**：

```yaml
# Jenkins 默认用户数据库
Security Realm:
  - Jenkins' own user database

# LDAP 集成
Security Realm:
  - LDAP
    server: ldap://ldap.company.com:389
    rootDN: dc=company,dc=com
    userSearchBase: ou=people
    userSearchFilter: uid={0}
    groupSearchBase: ou=groups
    managerDN: cn=admin,dc=company,dc=com
    managerPassword: ****

# SSO（通过插件支持）
# 使用 SAML/CAS/OpenID Connect 等插件
```

**授权策略**：

```yaml
# 任何人可做任何事（仅适合测试环境）
Authorization:
  - Anyone can do anything

# 登录用户可以做任何事
Authorization:
  - Logged-in users can do anything

# 基于矩阵的安全策略
Authorization:
  - Matrix-based security
    User/Group: admin - 所有权限
    User/Group: anonymous - Read(Job/View)
    User/Group: developers - Read(Job/View/SCM), Build(Job)

# 基于角色的策略（需安装 Role Strategy Plugin）
Authorization:
  - Role-Based Strategy
```

# 6. 反向代理配置

生产环境通常在 Jenkins 前面配置反向代理（Nginx/Apache）以提供 SSL 终结、域名转发等。

## 6.1 Nginx反向代理

```nginx
server {
    listen 443 ssl;
    server_name jenkins.company.com;

    ssl_certificate /etc/nginx/certs/jenkins.crt;
    ssl_certificate_key /etc/nginx/certs/jenkins.key;

    # 关键配置：必须设置，否则 Jenkins 无法正确生成链接
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Port $server_port;

    # WebSocket 支持（Pipeline 日志、Blue Ocean 需要）
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    # 请求体大小限制（上传 artifact 时可能需要调整）
    client_max_body_size 100m;

    # 连接超时
    proxy_connect_timeout 90;
    proxy_send_timeout 300;
    proxy_read_timeout 300;

    location / {
        proxy_pass http://jenkins-backend:8080;
        proxy_redirect http:// https://;
    }
}
```

配置完成后，需要在 Jenkins 的 **System Configuration > Jenkins Location** 中设置：
- Jenkins URL：`https://jenkins.company.com/`

## 6.2 Apache反向代理

```apache
<VirtualHost *:443>
    ServerName jenkins.company.com

    SSLEngine on
    SSLCertificateFile /etc/apache2/certs/jenkins.crt
    SSLCertificateKeyFile /etc/apache2/certs/jenkins.key

    ProxyRequests Off
    ProxyPreserveHost On

    # WebSocket 支持
    ProxyPass /ws/ ws://jenkins-backend:8080/jenkins/ws/
    ProxyPassReverse /ws/ ws://jenkins-backend:8080/jenkins/ws/

    ProxyPass / http://jenkins-backend:8080/
    ProxyPassReverse / http://jenkins-backend:8080/

    <Proxy http://jenkins-backend:8080/*>
        Order allow,deny
        Allow from all
    </Proxy>

    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Port "443"
</VirtualHost>
```

# 7. JVM参数调优

合理的 JVM 参数可以显著提升 Jenkins 性能。

## 7.1 常用 JVM 参数

```bash
# JVM 参数通过 JAVA_OPTS 环境变量设置
export JAVA_OPTS="\
  # 堆内存配置（根据机器内存调整）
  -Xms2048m \
  -Xmx4096m \
  # 元空间（替代永久代）
  -XX:MetaspaceSize=256m \
  -XX:MaxMetaspaceSize=512m \
  # GC 选择（推荐 G1GC）
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  # GC 日志
  -Xlog:gc*:file=/var/log/jenkins/gc.log:time,uptime,pid,tags:filecount=5,filesize=10m \
  # JMX 远程监控
  -Dcom.sun.management.jmxremote \
  -Dcom.sun.management.jmxremote.port=9090 \
  -Dcom.sun.management.jmxremote.authenticate=false \
  -Dcom.sun.management.jmxremote.ssl=false \
  # Jenkins 特定参数
  -Dhudson.model.ParametersAction.keepUndefinedParameters=true \
  -Djenkins.install.runSetupWizard=false \
  # 时区
  -Duser.timezone=Asia/Shanghai \
  # HTTP 参数
  -Dorg.apache.tomcat.websocket.DISABLE_BUILTIN_EXTENSIONS=true \
"
```

## 7.2 内存建议

| Jenkins Job 数量 | 堆内存建议 |
|-----------------|-----------|
| < 50 | 2GB - 4GB |
| 50 - 200 | 4GB - 8GB |
| 200 - 500 | 8GB - 16GB |
| > 500 | 16GB+ |

## 7.3 如何处理 OOM（Out of Memory）

1. 增加堆内存 `-Xmx`
2. 减少 Pipeline 日志保留天数
3. 清理构建历史
4. 增加 Agent 数量减少 Master 负载
5. 使用 Pipeline `-Xms` 和 `-Xmx` 设置为相同值，避免动态调整
6. 开启 `-XX:+HeapDumpOnOutOfMemoryError` 用于事后分析

# 8. 备份与恢复

## 8.1 手动备份

```bash
#!/bin/bash
# Jenkins 备份脚本
JENKINS_HOME=/var/lib/jenkins
BACKUP_DIR=/data/backup/jenkins
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE=$BACKUP_DIR/jenkins_backup_$DATE.tar.gz

# 创建备份目录
mkdir -p $BACKUP_DIR

# 停止 Jenkins（可选，保证数据一致性）
# systemctl stop jenkins

# 备份 Jenkins Home（排除 workspace 和 build 历史，以节省空间）
tar --exclude='workspace' \
    --exclude='builds' \
    --exclude='logs' \
    --exclude='caches' \
    -czf $BACKUP_FILE \
    -C $(dirname $JENKINS_HOME) $(basename $JENKINS_HOME)

# 重启 Jenkins
# systemctl start jenkins

# 删除 30 天前的备份
find $BACKUP_DIR -name "jenkins_backup_*.tar.gz" -mtime +30 -delete

echo "Backup completed: $BACKUP_FILE"
```

## 8.2 关键数据说明

| 数据 | 目录/文件 | 重要程度 | 备份建议 |
|------|-----------|---------|---------|
| 全局配置 | `config.xml` | 关键 | 必须备份 |
| Job 配置 | `jobs/*/config.xml` | 关键 | 必须备份 |
| 插件 | `plugins/` | 重要 | 备份清单即可 |
| 凭据密钥 | `secrets/` | 关键 | 必须备份 |
| 用户配置 | `users/` | 重要 | 建议备份 |
| 构建历史 | `jobs/*/builds/` | 一般 | 可选（保留最近记录） |
| 构建产物 | `jobs/*/builds/*/archive/` | 一般 | 可选 |
| 工作空间 | `workspace/` | 不备份 | 运行时数据 |

## 8.3 thinBackup 插件

推荐使用 `thinBackup` 插件自动备份：

```
1. 安装 thinBackup 插件
2. Manage Jenkins > ThinBackup > Settings
   - Backup directory: /data/backup/jenkins
   - Full backup every: 7 days
   - Differential backup every: 1 day
   - Max number of backup sets: 12
   - Exclude patterns: **/workspace/**,**/builds/**
3. 手动触发备份验证
4. 设置 cron 定时任务检查备份状态
```

# 9. 升级与迁移

## 9.1 版本升级原则

1. **LTS 优先**：始终使用 LTS 版本，每 12 周发布一次
2. **小版本升级**（如 2.440.1 → 2.440.3）：可直接升级
3. **大版本升级**（如 2.346 → 2.440）：需阅读发版说明中的 Breaking Changes
4. **升级前备份**：务必备份 `JENKINS_HOME`
5. **插件兼容性**：检查插件与目标版本的兼容性

## 9.2 升级步骤

```bash
# 1. 备份
systemctl stop jenkins
tar czf /backup/jenkins_pre_upgrade_$(date +%Y%m%d).tar.gz /var/lib/jenkins

# 2. 下载新版本
wget -O /tmp/jenkins.war https://get.jenkins.io/war-stable/2.440.3/jenkins.war

# 3. 替换 WAR 文件
cp /tmp/jenkins.war /usr/share/java/jenkins.war

# 4. 重启
systemctl start jenkins

# 5. 验证
tail -f /var/log/jenkins/jenkins.log
# 检查是否有插件兼容性错误
```

## 9.3 迁移到新服务器

```bash
# 源服务器
systemctl stop jenkins
tar czf /tmp/jenkins_full_backup.tar.gz /var/lib/jenkins
scp /tmp/jenkins_full_backup.tar.gz user@new-server:/tmp/

# 目标服务器（安装相同版本 Jenkins）
systemctl stop jenkins
mv /var/lib/jenkins /var/lib/jenkins.orig
tar xzf /tmp/jenkins_full_backup.tar.gz -C /
# 确保目录结构正确
# 注意：JENKINS_HOME 路径必须一致

# 修改配置（如需要）
vi /etc/default/jenkins  # 更新 IP、端口等

# 启动
systemctl start jenkins
```
