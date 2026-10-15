
# 1. Nacos 单机模式部署

## 1.1 环境要求

| 依赖 | 版本要求 |
|------|---------|
| JDK | 1.8+（推荐 JDK 11） |
| Maven | 3.6.x+（源码编译需要） |
| MySQL | 5.7+ / 8.0+（集群模式必选） |
| OS | Linux/Windows/Mac |

## 1.2 下载与启动

**方式一：源码编译**

```bash
git clone https://github.com/alibaba/nacos.git
cd nacos/
mvn -Prelease-nacos -Dmaven.test.skip=true clean install -U
cd distribution/target/nacos-server-$version/nacos/bin
```

**方式二：二进制包下载**

```bash
# 下载最新稳定版
wget https://github.com/alibaba/nacos/releases/download/2.2.3/nacos-server-2.2.3.zip
unzip nacos-server-2.2.3.zip
cd nacos/bin
```

**启动命令**

```bash
# Linux/Unix/Mac
sh startup.sh -m standalone

# Windows
startup.cmd -m standalone
```

## 1.3 验证启动

```bash
# 查看启动日志
tail -f logs/nacos.log

# 访问控制台
# http://localhost:8848/nacos
# 默认账号：nacos/nacos
```

## 1.4 单机持久化配置

默认使用 Derby 内嵌数据库，生产环境建议切换到 MySQL：

```sql
-- 1. 创建数据库
CREATE DATABASE IF NOT EXISTS `nacos` DEFAULT CHARACTER SET utf8mb4;

-- 2. 执行初始化脚本（位于 conf/mysql-schema.sql）
```

```properties
# 3. 修改 conf/application.properties
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false&allowPublicKeyRetrieval=true
db.user.0=root
db.password.0=your_password
```

# 2. Nacos 集群模式部署

## 2.1 集群架构图

```
                    ┌──────────────┐
                    │   Nginx/VIP  │
                    │   (SLB)      │
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌────────────┐  ┌────────────┐  ┌────────────┐
    │ Nacos-1    │  │ Nacos-2    │  │ Nacos-3    │
    │ 10.0.0.1   │  │ 10.0.0.2   │  │ 10.0.0.3   │
    │ :8848      │  │ :8848      │  │ :8848      │
    │ :7848(gRPC)│  │ :7848(gRPC)│  │ :7848(gRPC)│
    │ :9849(Raft)│  │ :9849(Raft)│  │ :9849(Raft)│
    └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
          └───────────────┼───────────────┘
                          ▼
                 ┌────────────────┐
                 │  MySQL Cluster │
                 │  (共享存储)    │
                 └────────────────┘
```

## 2.2 端口说明

| 端口 | 用途 | 说明 |
|------|------|------|
| 8848 | HTTP REST API | 主通信端口，控制台访问 |
| 7848 | gRPC 客户端通信 | Nacos 2.x 新增 |
| 9848 | gRPC 服务端请求 | 客户端 gRPC 请求服务端端口（偏移 1000）|
| 9849 | gRPC 集群通信 | 集群间 Raft/gRPC 通信（偏移 1001）|

## 2.3 集群配置步骤

**步骤 1：配置 MySQL 共享存储**

在所有节点上使用相同的数据库配置（见 1.4）。

**步骤 2：配置集群节点列表**

```properties
# conf/cluster.conf (每行一个节点 IP:Port)
10.0.0.1:8848
10.0.0.2:8848
10.0.0.3:8848
```

**步骤 3：调整 JVM 参数（可选）**

```bash
# bin/startup.sh 中可调整
JAVA_OPT="${JAVA_OPT} -server -Xms2g -Xmx2g -Xmn1g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```

**步骤 4：逐个启动节点**

```bash
# 每个节点执行
sh startup.sh
# 查看同步状态
tail -f logs/nacos-cluster.log
```

**步骤 5：配置 Nginx 反向代理**

```nginx
upstream nacos-cluster {
    server 10.0.0.1:8848;
    server 10.0.0.2:8848;
    server 10.0.0.3:8848;
}

server {
    listen 8848;
    server_name nacos.example.com;

    location /nacos/ {
        proxy_pass http://nacos-cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## 2.4 验证集群状态

```bash
# 查看集群节点列表
curl -X GET 'http://10.0.0.1:8848/nacos/v1/core/cluster/nodes'

# 预期返回
{
  "code": 200,
  "message": null,
  "data": [
    {"address": "10.0.0.1:8848", "state": "UP", ...},
    {"address": "10.0.0.2:8848", "state": "UP", ...},
    {"address": "10.0.0.3:8848", "state": "UP", ...}
  ]
}
```

# 3. Docker 部署

## 3.1 单机 Docker 部署

```bash
# 拉取镜像
docker pull nacos/nacos-server:latest

# 运行容器（单机模式，内置 Derby）
docker run -d \
  --name nacos-standalone \
  -p 8848:8848 \
  -p 9848:9848 \
  -e MODE=standalone \
  nacos/nacos-server:latest
```

## 3.2 Docker Compose 集群部署

```yaml
# docker-compose.yml
version: "3.8"
services:
  nacos1:
    image: nacos/nacos-server:latest
    container_name: nacos1
    hostname: nacos1
    environment:
      - MODE=cluster
      - NACOS_SERVERS=nacos1:8848 nacos2:8848 nacos3:8848
      - SPRING_DATASOURCE_PLATFORM=mysql
      - MYSQL_SERVICE_HOST=mysql_host
      - MYSQL_SERVICE_DB_NAME=nacos
      - MYSQL_SERVICE_USER=root
      - MYSQL_SERVICE_PASSWORD=root
      - NACOS_AUTH_ENABLE=true
      - NACOS_AUTH_IDENTITY_KEY=serverIdentity
      - NACOS_AUTH_IDENTITY_VALUE=security
      - NACOS_AUTH_TOKEN=SecretKey012345678901234567890123456789012345678901234567890123456789
    ports:
      - "8848:8848"
      - "9848:9848"
      - "9849:9849"
    networks:
      - nacos_net

  nacos2:
    image: nacos/nacos-server:latest
    container_name: nacos2
    hostname: nacos2
    environment:
      - MODE=cluster
      - NACOS_SERVERS=nacos1:8848 nacos2:8848 nacos3:8848
      - SPRING_DATASOURCE_PLATFORM=mysql
      - MYSQL_SERVICE_HOST=mysql_host
      - MYSQL_SERVICE_DB_NAME=nacos
      - MYSQL_SERVICE_USER=root
      - MYSQL_SERVICE_PASSWORD=root
    networks:
      - nacos_net

  nacos3:
    image: nacos/nacos-server:latest
    container_name: nacos3
    hostname: nacos3
    environment:
      - MODE=cluster
      - NACOS_SERVERS=nacos1:8848 nacos2:8848 nacos3:8848
      - SPRING_DATASOURCE_PLATFORM=mysql
      - MYSQL_SERVICE_HOST=mysql_host
      - MYSQL_SERVICE_DB_NAME=nacos
      - MYSQL_SERVICE_USER=root
      - MYSQL_SERVICE_PASSWORD=root
    networks:
      - nacos_net

networks:
  nacos_net:
    driver: bridge
```

# 4. Kubernetes 部署

## 4.1 使用 Helm 部署

```bash
# 添加仓库
helm repo add nacos https://nacos-group.github.io/nacos-helm-chart

# 安装
helm install nacos nacos/nacos \
  --set global.mode=cluster \
  --set nacos.replicaCount=3 \
  --set nacos.storage.db.type=mysql \
  --set nacos.storage.db.host=mysql.default.svc.cluster.local \
  --set nacos.storage.db.name=nacos \
  --set nacos.storage.db.username=root \
  --set nacos.storage.db.password=root
```

## 4.2 StatefulSet 部署示例

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nacos
spec:
  serviceName: nacos-headless
  replicas: 3
  selector:
    matchLabels:
      app: nacos
  template:
    metadata:
      labels:
        app: nacos
    spec:
      containers:
      - name: nacos
        image: nacos/nacos-server:latest
        ports:
        - containerPort: 8848
          name: http
        - containerPort: 9848
          name: client-rpc
        - containerPort: 9849
          name: raft-rpc
        env:
        - name: MODE
          value: "cluster"
        - name: NACOS_SERVERS
          value: "nacos-0.nacos-headless:8848 nacos-1.nacos-headless:8848 nacos-2.nacos-headless:8848"
        - name: SPRING_DATASOURCE_PLATFORM
          value: "mysql"
        - name: MYSQL_SERVICE_HOST
          value: "mysql-service"
        - name: MYSQL_SERVICE_DB_NAME
          value: "nacos"
        - name: MYSQL_SERVICE_USER
          valueFrom:
            secretKeyRef:
              name: nacos-mysql-secret
              key: username
        - name: MYSQL_SERVICE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: nacos-mysql-secret
              key: password
```

# 5. 关键配置参数

## 5.1 application.properties 核心配置

```properties
# ============ 服务端口 ============
server.port=8848

# ============ 数据库配置 ============
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false
db.user.0=root
db.password.0=root

# ============ 鉴权配置 ============
nacos.core.auth.enabled=true
nacos.core.auth.system.type=nacos
nacos.core.auth.plugin.nacos.token.secret.key=VGhpc0lzTXlDdXN0b21TZWNyZXRLZXkwMTIzNDU2Nzg=

# ============ 集群配置 ============
nacos.inetutils.ip-address=10.0.0.1
nacos.inetutils.prefer-hostname-over-ip=false

# ============ 日志配置 ============
nacos.naming.log.level=INFO
nacos.config.log.level=INFO
```

## 5.2 JVM 参数推荐

```bash
# 生产环境推荐配置（4C8G 机器）
-Xms4g
-Xmx4g
-Xmn2g
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
-XX:+UseG1GC
-XX:G1HeapRegionSize=16m
-XX:MaxGCPauseMillis=200
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/nacos/logs/heapdump.hprof
```

## 5.3 系统参数优化

```bash
# 文件描述符限制
ulimit -n 65535

# 内核参数优化
echo 'net.core.somaxconn=4096' >> /etc/sysctl.conf
echo 'vm.swappiness=10' >> /etc/sysctl.conf
sysctl -p
```

# 6. 部署模式对比

| 模式 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **单机 Derby** | 本地开发、测试 | 零依赖，开箱即用 | 无高可用，数据易丢失 |
| **单机 MySQL** | 测试环境、小规模生产 | 数据持久化，可后续扩展 | 仍存在单点 |
| **集群 MySQL** | 生产环境（中等规模） | 高可用，读写分离 | 需要独立 MySQL 集群 |
| **Docker 集群** | 容器化环境 | 快速部署、弹性伸缩 | 需要容器编排平台 |
| **K8s 集群** | 云原生环境 | 自动恢复、滚动升级 | 运维复杂度高 |

# 7. 升级策略

## 7.1 灰度升级步骤

1. 将新版本部署到其中一个节点
2. 验证该节点功能正常（注册、发现、配置推送）
3. 逐步替换其他节点
4. 所有节点升级完毕后，验证集群状态

## 7.2 1.x 升级到 2.x 注意事项

- gRPC 端口（偏移 1000/1001）必须在防火墙开放
- 客户端 SDK 需同步升级
- 长连接数增加，注意调整系统参数（ulimit）
- 建议在非业务高峰期执行升级
