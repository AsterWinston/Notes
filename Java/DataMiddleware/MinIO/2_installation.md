# 1.部署模式概述

MinIO 支持三种部署模式：

| 模式 | 节点数 | 每节点磁盘数 | 总磁盘数 | 数据保护 | 适用场景 |
|------|--------|-------------|----------|----------|----------|
| 单节点单磁盘 (SNSD) | 1 | 1 | 1 | 无冗余保护 | 本地开发测试 |
| 单节点多磁盘 (SNMD) | 1 | >= 2 | >= 2 | EC 保护（仅磁盘级） | 实验环境、小规模生产 |
| 多节点多磁盘 (MNMD) | >= 4 | >= 4 | >= 16 | EC 保护 + 节点级容错 | 生产环境（推荐） |

> **生产环境最低要求**：4 个节点 × 4 块磁盘 = 16 块磁盘（EC 默认 8 数据 + 8 校验，可容忍 8 块磁盘或 4 个节点故障）

---

# 2.Docker 方式部署

Docker 是最快捷的部署方式，适合开发、测试和学习场景。

## 2.1Docker 环境准备

### Windows — Docker Desktop

```powershell
# 前置条件：启用 WSL2 或 Hyper-V
# 下载安装 Docker Desktop for Windows
# https://www.docker.com/products/docker-desktop/

# 验证安装
docker --version
docker run hello-world
```

### Linux — Docker Engine

```bash
# Ubuntu/Debian 安装
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker

# 将当前用户加入 docker 组（免 sudo）
sudo usermod -aG docker $USER
# 需要重新登录生效

# 验证安装
docker --version
```

## 2.2单节点单磁盘 — 快速启动

```powershell
# 拉取最新镜像
docker pull minio/minio:latest

# 启动单节点（单磁盘）
docker run -d `
  --name minio `
  -p 9000:9000 `
  -p 9001:9001 `
  -e "MINIO_ROOT_USER=minioadmin" `
  -e "MINIO_ROOT_PASSWORD=minioadmin" `
  -v D:\minio\data:/data `
  minio/minio server /data --console-address ":9001"
```

```bash
# Linux 等效命令
docker run -d \
  --name minio \
  -p 9000:9000 \
  -p 9001:9001 \
  -e "MINIO_ROOT_USER=minioadmin" \
  -e "MINIO_ROOT_PASSWORD=minioadmin" \
  -v /data/minio:/data \
  minio/minio server /data --console-address ":9001"
```

启动后访问：
- **API 端点**：`http://localhost:9000`（S3 API）
- **Console 控制台**：`http://localhost:9001`（Web 管理界面）

> 登录 Console 使用上面设置的 `MINIO_ROOT_USER` 和 `MINIO_ROOT_PASSWORD`

## 2.3单节点多磁盘 — 带纠删码

```bash
# 先创建多个数据目录（模拟多磁盘）
sudo mkdir -p /data/minio/disk{1..4}

docker run -d \
  --name minio-multi-disk \
  -p 9000:9000 \
  -p 9001:9001 \
  -e "MINIO_ROOT_USER=minioadmin" \
  -e "MINIO_ROOT_PASSWORD=minioadmin" \
  -v /data/minio/disk1:/disk1 \
  -v /data/minio/disk2:/disk2 \
  -v /data/minio/disk3:/disk3 \
  -v /data/minio/disk4:/disk4 \
  minio/minio server /disk{1...4} --console-address ":9001"
```

**EC 参数说明**（4 块磁盘的场景）：
- 数据块 (Data)：2
- 校验块 (Parity)：2
- 容错能力：可容忍 2 块磁盘故障
- 存储效率：50%（4 块盘中 2 块用于校验）

## 2.4Docker Compose 生产级部署

完整 `docker-compose.yml`（包含 MinIO + Nginx 负载均衡 + MinIO Console）：

```yaml
version: "3.8"

x-minio-common: &minio-common
  image: quay.io/minio/minio:latest
  command: server --console-address ":9001" http://minio{1...4}/disk{1...4}
  environment:
    MINIO_ROOT_USER: minioadmin
    MINIO_ROOT_PASSWORD: minioadmin
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
    interval: 30s
    timeout: 20s
    retries: 3

services:
  # 4 节点 × 4 磁盘 = 16 磁盘集群（模拟）
  minio1:
    <<: *minio-common
    hostname: minio1
    volumes:
      - minio1-data1:/disk1
      - minio1-data2:/disk2
      - minio1-data3:/disk3
      - minio1-data4:/disk4

  minio2:
    <<: *minio-common
    hostname: minio2
    volumes:
      - minio2-data1:/disk1
      - minio2-data2:/disk2
      - minio2-data3:/disk3
      - minio2-data4:/disk4

  minio3:
    <<: *minio-common
    hostname: minio3
    volumes:
      - minio3-data1:/disk1
      - minio3-data2:/disk2
      - minio3-data3:/disk3
      - minio3-data4:/disk4

  minio4:
    <<: *minio-common
    hostname: minio4
    volumes:
      - minio4-data1:/disk1
      - minio4-data2:/disk2
      - minio4-data3:/disk3
      - minio4-data4:/disk4

  # Nginx 负载均衡
  nginx:
    image: nginx:alpine
    hostname: nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "9000:9000"
      - "9001:9001"
    depends_on:
      - minio1
      - minio2
      - minio3
      - minio4

volumes:
  minio1-data1: minio1-data2: minio1-data3: minio1-data4:
  minio2-data1: minio2-data2: minio2-data3: minio2-data4:
  minio3-data1: minio3-data2: minio3-data3: minio3-data4:
  minio4-data1: minio4-data2: minio4-data3: minio4-data4:
```

对应的 `nginx.conf`：

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    # MinIO S3 API 负载均衡
    upstream minio_s3 {
        least_conn;
        server minio1:9000;
        server minio2:9000;
        server minio3:9000;
        server minio4:9000;
    }

    # MinIO Console 负载均衡
    upstream minio_console {
        least_conn;
        server minio1:9001;
        server minio2:9001;
        server minio3:9001;
        server minio4:9001;
    }

    server {
        listen 9000;
        # 处理大文件上传
        client_max_body_size 0;
        proxy_request_buffering off;

        location / {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_connect_timeout 300;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            chunked_transfer_encoding off;
            proxy_pass http://minio_s3;
        }
    }

    server {
        listen 9001;
        client_max_body_size 0;
        proxy_request_buffering off;

        location / {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-NginX-Proxy true;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_pass http://minio_console;
        }
    }
}
```

启动：
```powershell
docker compose up -d
# 查看启动日志
docker compose logs -f
# 查看集群状态
docker compose ps
```

## 2.5Docker 常用管理命令

```bash
# 查看容器日志
docker logs -f minio

# 查看最近 200 条日志
docker logs --tail 200 minio

# 进入容器
docker exec -it minio bash

# 查看容器资源占用
docker stats minio

# 临时停止
docker stop minio

# 重新启动
docker start minio

# 完全删除（含数据卷）
docker stop minio; docker rm minio; docker volume prune
```

---

# 3.二进制文件直接部署（Linux）

## 3.1下载与安装

```bash
# 下载最新稳定版 MinIO Server（x86_64）
wget https://dl.min.io/server/minio/release/linux-amd64/minio -O /usr/local/bin/minio

# 添加执行权限
chmod +x /usr/local/bin/minio

# 验证版本
minio --version
```

## 3.2创建系统用户（安全最佳实践）

```bash
# 创建 minio-user 系统用户（不能登录）
sudo useradd -r minio-user -s /sbin/nologin

# 创建数据目录
sudo mkdir -p /data/minio

# 设置目录所有权
sudo chown -R minio-user:minio-user /data/minio
```

## 3.3单节点单磁盘启动

```bash
# 最低配置启动
MINIO_ROOT_USER=minioadmin \
MINIO_ROOT_PASSWORD=minioadmin \
minio server /data/minio --console-address ":9001"
```

## 3.4单节点多磁盘启动

```bash
# 4 块磁盘，EC 模式
MINIO_ROOT_USER=minioadmin \
MINIO_ROOT_PASSWORD=minioadmin \
minio server \
  /mnt/disk1/data \
  /mnt/disk2/data \
  /mnt/disk3/data \
  /mnt/disk4/data \
  --console-address ":9001"
```

> **注意**：`/mnt/diskX/` 应该是空的、格式化好的挂载点。如果在同一块物理盘的不同目录下启动多盘模式，MinIO 会检测到并拒绝启动（会提示 "detected overlapping roots"）。

## 3.5配置为 systemd 服务

创建服务文件 `/etc/systemd/system/minio.service`：

```ini
[Unit]
Description=MinIO Object Storage Server
Documentation=https://min.io/docs/minio/linux/index.html
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
Type=notify
WorkingDirectory=/usr/local

# 环境变量配置
Environment="MINIO_ROOT_USER=minioadmin"
Environment="MINIO_ROOT_PASSWORD=minioadmin"
Environment="MINIO_VOLUMES=/data/minio"
Environment="MINIO_OPTS=--console-address :9001"

# 运行用户
User=minio-user
Group=minio-user

# 安全加固
ProtectProc=invisible
ProcSubset=pid
ProtectSystem=strict
ProtectHome=on
PrivateDevices=on
NoNewPrivileges=yes

# 文件描述符限制（生产环境建议调大）
LimitNOFILE=65536

# 任务数限制
TasksMax=infinity

# 优雅关闭超时
TimeoutStopSec=30
SendSIGKILL=no

ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES

# 自动重启
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 启动 MinIO
sudo systemctl start minio

# 设置开机自启
sudo systemctl enable minio

# 查看运行状态
sudo systemctl status minio

# 查看日志
sudo journalctl -u minio -f

# 查看最近 50 条日志
sudo journalctl -u minio -n 50
```

---

# 4.Linux 包管理器安装

## 4.1通过 DEB 包安装 (Ubuntu/Debian)

```bash
# 下载 DEB 包
wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio_latest_amd64.deb

# 安装
sudo dpkg -i minio_latest_amd64.deb

# 验证安装
minio --version
```

## 4.2通过 RPM 包安装 (CentOS/RHEL)

```bash
# 下载 RPM 包
wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio_latest_amd64.rpm

# 安装
sudo rpm -ivh minio_latest_amd64.rpm

# 验证安装
minio --version
```

---

# 5.Kubernetes 部署

## 5.1使用 MinIO Operator（推荐）

MinIO Operator 是官方推荐的 K8s 部署方式，通过 CRD (Custom Resource Definition) 管理 Tenant。

```bash
# 安装 Operator
kubectl apply -k "github.com/minio/operator?ref=latest"

# 或者使用 Helm
helm repo add minio-operator https://operator.min.io
helm install minio-operator minio-operator/operator

# 查看 Operator 状态
kubectl get pods -n minio-operator
```

创建 Tenant YAML (`minio-tenant.yaml`)：

```yaml
apiVersion: minio.min.io/v2
kind: Tenant
metadata:
  name: minio-tenant
  namespace: minio
spec:
  # Tenant 镜像
  image: quay.io/minio/minio:latest

  # Server Pool 配置
  pools:
    - servers: 4          # 节点数
      name: pool-0
      volumesPerServer: 4 # 每节点磁盘数
      volumeClaimTemplate:
        metadata:
          name: data
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 100Gi  # 每块磁盘的容量
          storageClassName: local-path

  # Root 用户凭证（生产环境使用 Secret 引用）
  credsSecret:
    name: minio-creds

  # Console 配置
  console:
    image: quay.io/minio/console:latest
    replicas: 1
    consoleSecret:
      name: console-secret

  # 请求资源配置
  pools:
    - servers: 4
      name: pool-0
      volumesPerServer: 4
      runtimeClassName: ""

  # 对外暴露
  requestAutoCert: false
  exposeServices:
    minio: true
    console: true
  serviceMetadata:
    minioServiceLabels:
      app: minio
    consoleServiceLabels:
      app: minio-console
```

```bash
# 创建命名空间
kubectl create namespace minio

# 创建凭证 Secret
kubectl create secret generic minio-creds \
  --from-literal=rootUser=minioadmin \
  --from-literal=rootPassword=minioadmin \
  -n minio

# 创建 Tenant
kubectl apply -f minio-tenant.yaml -n minio

# 查看状态
kubectl get tenants -n minio
kubectl get pods -n minio
```

## 5.2使用 Helm Chart 直接部署

```bash
# 添加 MinIO Helm 仓库
helm repo add minio https://charts.min.io

# 安装（单节点模式，测试环境）
helm install minio minio/minio \
  --set rootUser=minioadmin \
  --set rootPassword=minioadmin \
  --set mode=standalone \
  --set persistence.enabled=true \
  --set persistence.size=10Gi \
  --set service.type=NodePort

# 安装（分布式模式，4 节点）
helm install minio minio/minio \
  --set rootUser=minioadmin \
  --set rootPassword=minioadmin \
  --set mode=distributed \
  --set replicas=4 \
  --set persistence.enabled=true \
  --set persistence.size=100Gi \
  --set drivesPerNode=4 \
  --set service.type=ClusterIP
```

---

# 6.关键环境变量详解

MinIO 通过环境变量进行核心配置，以下是完整的环境变量列表：

## 6.1认证相关

| 环境变量 | 说明 | 示例 |
|----------|------|------|
| `MINIO_ROOT_USER` | Root 用户 Access Key（长度 ≥ 3，最大 128） | `minioadmin` |
| `MINIO_ROOT_PASSWORD` | Root 用户 Secret Key（长度 ≥ 8，最大 128） | `minioadmin` |
| `MINIO_ROOT_USER_FILE` | 从文件中读取 Access Key（Docker Secrets） | `/run/secrets/minio_user` |
| `MINIO_ROOT_PASSWORD_FILE` | 从文件中读取 Secret Key（Docker Secrets） | `/run/secrets/minio_password` |

> **安全建议**：生产环境不要使用 `minioadmin:minioadmin`，且在 K8s 环境中通过 Secret 挂载凭证。

## 6.2存储相关

| 环境变量 | 说明 |
|----------|------|
| `MINIO_VOLUMES` | 数据卷路径，多路径用空格分隔 |
| `MINIO_OPTS` | 额外的命令行选项 |

## 6.3网络与安全

| 环境变量 | 说明 | 默认值 |
|----------|------|--------|
| `MINIO_SERVER_URL` | 对外暴露的服务 URL | - |
| `MINIO_BROWSER` | 是否启用 Console | `on` |
| `MINIO_BROWSER_REDIRECT_URL` | Console 重定向 URL | - |
| `MINIO_DOMAIN` | 虚拟托管风格域名（如 s3.example.com） | - |
| `MINIO_PUBLIC_CERTS` | TLS 证书目录 | - |
| `MINIO_API_CORS_ALLOW_ORIGIN` | CORS 允许的域名 | `*` |

## 6.4身份与访问管理

| 环境变量 | 说明 |
|----------|------|
| `MINIO_IDENTITY_OPENID_CONFIG_URL` | OIDC 发现端点 URL |
| `MINIO_IDENTITY_OPENID_CLIENT_ID` | OIDC Client ID |
| `MINIO_IDENTITY_OPENID_CLIENT_SECRET` | OIDC Client Secret |
| `MINIO_IDENTITY_OPENID_CLAIM_NAME` | OIDC JWT 中的声明名（默认 `sub`） |
| `MINIO_IDENTITY_LDAP_SERVER_ADDR` | LDAP 服务器地址 |
| `MINIO_IDENTITY_LDAP_LOOKUP_BIND_DN` | LDAP 绑定 DN |
| `MINIO_IDENTITY_LDAP_LOOKUP_BIND_PASSWORD` | LDAP 绑定密码 |

## 6.5站点复制

| 环境变量 | 说明 |
|----------|------|
| `MINIO_SITE_REGION` | 站点区域标识 |
| `MINIO_SITE_NAME` | 站点名称 |

---

# 7.配置文件详解（minio配置目录）

从较新版本开始，MinIO 支持将部分配置通过环境变量注入，但高级配置仍通过 mc admin 命令或 K8s Operator 处理。MinIO 的运行时配置存储在 `.minio.sys` 内部 Bucket 中，不需要外部配置文件。

如果需要通过文件配置（某些部署场景），可使用 MinIO 的 `--config-dir` 参数或使用 `mc admin config`。

### 使用 mc 查看和修改运行时配置

```bash
# 连接 MinIO
mc alias set myminio http://localhost:9000 minioadmin minioadmin

# 查看所有可配置项
mc admin config get myminio

# 查看特定子系统配置
mc admin config get myminio api
mc admin config get myminio heal
mc admin config get myminio scanner

# 设置配置（示例：设置磁盘使用率告警阈值）
mc admin config set myminio heal max_sleep=5s max_io=10
```

---

# 8.MinIO 目录结构

MinIO 实例的数据目录结构如下：

```text
/data/minio/                           # 数据根目录
├── .minio.sys/                        # MinIO 内部元数据 Bucket
│   ├── config/                        # 集群配置
│   │   └── config.json/
│   ├── buckets/                       # Bucket 元数据
│   └── tmp/                           # 临时数据
├── bucket-name/                       # 用户 Bucket 数据目录
│   ├── object-key-a/
│   │   ├── part.1                     # EC 编码后的数据分片
│   │   └── xl.meta                    # 对象元数据（JSON格式）
│   └── object-key-b/
│       ├── part.1
│       └── xl.meta
└── .usage-cache.bin                   # 容量使用统计缓存
```

### xl.meta 文件示例（简化）

```json
{
  "version": "1.0",
  "format": "xl",
  "stat": {
    "size": 1048576,
    "modTime": "2025-01-15T10:30:00Z"
  },
  "erasure": {
    "data": 8,
    "parity": 8,
    "blockSize": 10485760,
    "index": 3,
    "distribution": [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15],
    "checksum": [...]
  },
  "metaUser": {
    "X-Amz-Meta-Author": "john"
  }
}
```

---

# 9.MinIO Client (mc) 安装

`mc` 是 MinIO 的命令行客户端，功能对标 AWS CLI，是管理 MinIO 的必备工具。

## 9.1安装 mc

```bash
# Linux (amd64)
wget https://dl.min.io/client/mc/release/linux-amd64/mc -O /usr/local/bin/mc
chmod +x /usr/local/bin/mc

# macOS
brew install minio/stable/mc

# Windows (PowerShell)
Invoke-WebRequest -Uri "https://dl.min.io/client/mc/release/windows-amd64/mc.exe" -OutFile "C:\mc.exe"
# 将 C:\ 或自定义路径添加到 PATH 环境变量
```

## 9.2mc 基础配置与使用

```bash
# 添加 MinIO 服务别名
mc alias set myminio http://localhost:9000 minioadmin minioadmin

# 查看所有别名
mc alias list

# 列举 Bucket
mc ls myminio

# 创建 Bucket
mc mb myminio/mybucket

# 上传文件
mc cp /path/to/file.txt myminio/mybucket/

# 下载文件
mc cp myminio/mybucket/file.txt /path/to/download/

# 上传目录（递归）
mc cp --recursive /path/to/dir/ myminio/mybucket/dir/

# 同步目录（镜像，只传输有差异的文件）
mc mirror /path/to/local/ myminio/mybucket/backup/

# 删除对象
mc rm myminio/mybucket/file.txt

# 删除 Bucket（Bucket 必须为空）
mc rb myminio/mybucket

# 删除 Bucket（强制，即使不为空）
mc rb --force myminio/mybucket

# 查看对象信息
mc stat myminio/mybucket/file.txt

# 查找对象
mc find myminio/mybucket --name "*.jpg"

# 设置 Bucket 策略（公开读）
mc anonymous set public myminio/mybucket

# 设置 Bucket 策略（私有）
mc anonymous set private myminio/mybucket

# 查看磁盘使用
mc du myminio/mybucket
```

---

# 10.启动参数详解

`minio server` 命令的完整参数列表：

```bash
minio server [FLAGS] DIR1 [DIR2 ...]

# 主要参数：
--address ":9000"           # S3 API 监听地址
--console-address ":9001"   # Web Console 监听地址
--certs-dir "/path/certs"   # TLS 证书目录
--config-dir "/path/config" # 配置目录
--quiet                     # 安静模式，减少日志输出
--anonymous                 # 禁用Console中的用户名/密码登录
--ftp "address"             # FTP 服务地址（实验性）
--sftp "address"            # SFTP 服务地址（实验性）
--interface "eth0"          # 网络接口选择
```

```bash
# 完整启动示例（4节点分布式）
# 在每个节点上执行相同的命令：
minio server \
  --address ":9000" \
  --console-address ":9001" \
  http://node{1...4}.example.com/data/minio
```

---

# 11.多节点分布式部署详解

## 11.1环境要求

| 要求项 | 说明 |
|--------|------|
| 节点数 | 最少 4 个（生产环境建议 4、6、8、10...） |
| 每节点磁盘数 | 最少 4 块（生产建议 8~16 块） |
| 磁盘类型 | XFS（推荐）或 ext4，不要使用 ZFS/btrfs |
| 网络 | 节点间千兆起，推荐万兆（10Gbps） |
| 时间同步 | 所有节点必须配置 NTP 同步 |
| 主机名 | 所有节点间必须 DNS 可解析（或配置 /etc/hosts） |

> **为什么推荐 XFS**：MinIO 利用 XFS 的 `O_DIRECT` 模式和 fallocate 特性实现高性能直接 I/O。ext4 也支持，但性能略差。ZFS/btrfs 因为其自身的校验和计算会与 MinIO 的 Bit Rot Protection 冲突，导致性能下降。

## 11.2节点规划示例

```text
集群规模: 4 节点 × 8 磁盘 = 32 磁盘
EC 配置:    16 数据 + 16 校验（N/2 校验）
容错能力:   可容忍 16 块磁盘或 4 个节点故障
存储效率:   50%
总可用容量: 50% × 总物理容量

节点信息:
  minio-1.example.com  (192.168.1.101)  /mnt/disk{1..8}/data
  minio-2.example.com  (192.168.1.102)  /mnt/disk{1..8}/data
  minio-3.example.com  (192.168.1.103)  /mnt/disk{1..8}/data
  minio-4.example.com  (192.168.1.104)  /mnt/disk{1..8}/data
```

## 11.3磁盘准备（每节点执行）

```bash
# 1. 查看可用磁盘
lsblk
fdisk -l

# 2. 格式化磁盘为 XFS
sudo mkfs.xfs /dev/sdb -f -L DISK1
sudo mkfs.xfs /dev/sdc -f -L DISK2
# ... 依次格式化所有磁盘

# 3. 创建挂载点并挂载
sudo mkdir -p /mnt/disk{1..8}
sudo mount /dev/sdb /mnt/disk1
sudo mount /dev/sdc /mnt/disk2
# ... 依次挂载所有磁盘

# 4. 配置 /etc/fstab 实现开机自动挂载
echo "LABEL=DISK1 /mnt/disk1 xfs defaults,noatime 0 2" | sudo tee -a /etc/fstab
echo "LABEL=DISK2 /mnt/disk2 xfs defaults,noatime 0 2" | sudo tee -a /etc/fstab
# ... 依次配置所有磁盘

# 5. 创建 MinIO 数据目录
sudo mkdir -p /mnt/disk{1..8}/data
sudo chown -R minio-user:minio-user /mnt/disk{1..8}
```

## 11.4启动集群

在每个节点上执行相同的 `minio server` 命令：

```bash
MINIO_ROOT_USER=minioadmin \
MINIO_ROOT_PASSWORD=minioadmin \
minio server \
  http://minio-{1...4}.example.com/mnt/disk{1...8}/data \
  --console-address ":9001"
```

`http://minio-{1...4}.example.com/` 的展开规则：
- `minio-{1...4}` 展开为 `minio-1 minio-2 minio-3 minio-4`
- `disk{1...8}` 展开为 `disk1 disk2 ... disk8`
- 最终形成 4×8=32 个磁盘端点

## 11.5扩容 — 添加新的 Server Pool

```bash
# 查看当前集群状态
mc admin info myminio

# 添加新 Pool（4节点 × 8磁盘，与旧 Pool 规格相同）
MINIO_ROOT_USER=minioadmin \
MINIO_ROOT_PASSWORD=minioadmin \
minio server \
  http://minio-{1...4}.example.com/mnt/disk{1...8}/data \      # Pool 0
  http://minio-{5...8}.example.com/mnt/disk{1...8}/data \      # Pool 1 (新)
  --console-address ":9001"
```

> **扩容注意事项**：
> - 不能在已有 Pool 中增加节点/磁盘（因为 EC 条带大小在 Pool 创建时已固定，修改会破坏现有数据）
> - 新 Pool 与旧 Pool 的 Erasure Set 大小应相同（确保相同的容错能力）
> - 扩容后需要在**所有节点**上重启 MinIO（使用新的启动命令）
> - 扩容不会自动重平衡已有数据（新数据会按策略分布到所有 Pool，旧数据停留在原 Pool）
> - 如果旧 Pool 容量已满，新数据将自动写入新 Pool

---

# 12.环境变量文件方式管理（推荐生产环境）

为安全起见，不要直接在命令行传递密码。使用环境变量文件：

```bash
# /etc/default/minio
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
MINIO_VOLUMES=http://minio-{1...4}.example.com/mnt/disk{1...8}/data
MINIO_OPTS=--console-address ":9001"

# MINIO_ROOT_USER_FILE=/run/secrets/minio/RootUser
# MINIO_ROOT_PASSWORD_FILE=/run/secrets/minio/RootPassword
```

在 systemd 的 `minio.service` 中通过 `EnvironmentFile` 引入：

```ini
[Service]
EnvironmentFile=/etc/default/minio
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
```

---

# 13.测试连接的几种方式

安装完成后，验证 MinIO 是否正常运行：

```bash
# 方式1：使用 mc 测试
mc alias set myminio http://localhost:9000 minioadmin minioadmin
mc admin info myminio

# 方式2：使用 curl 测试 S3 API
curl http://localhost:9000/minio/health/live
# 返回: 200 OK

curl http://localhost:9000/minio/health/ready
# 返回: 200 OK (所有节点就绪)

# 方式3：浏览器访问 Console
# 打开 http://localhost:9001

# 方式4：使用 AWS CLI 测试
aws configure set aws_access_key_id minioadmin
aws configure set aws_secret_access_key minioadmin
aws s3 ls --endpoint-url http://localhost:9000
```

### Spring Boot 简易测试端点

```java
@RestController
public class MinioTestController {

    private final MinioClient minioClient;

    public MinioTestController(MinioClient minioClient) {
        this.minioClient = minioClient;
    }

    @GetMapping("/minio/health")
    public ResponseEntity<String> health() {
        try {
            boolean exists = minioClient.bucketExists(
                BucketExistsArgs.builder().bucket("test").build()
            );
            return ResponseEntity.ok("MinIO连接正常，Bucket test 存在: " + exists);
        } catch (Exception e) {
            return ResponseEntity.status(500).body("MinIO连接失败: " + e.getMessage());
        }
    }
}
```
