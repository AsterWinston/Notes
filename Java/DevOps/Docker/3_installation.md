
# 1. 安装的是什么：Docker 组件与版本关系

很多人以为“安装 Docker = 安装一个命令”，但实际是一组组件协作：

1. **Docker CLI**：`docker` 命令行工具。
2. **dockerd（Docker Engine）**：守护进程，负责镜像、网络、卷、容器生命周期管理。
3. **containerd / runc**：更底层的容器运行时组件（Docker Engine 会调用它们）。
4. **Buildx / Compose Plugin**：构建与编排的插件能力（现代 Docker 推荐插件化安装）。

工程结论：

1. 出现“客户端有版本但运行不了”时，优先检查 dockerd 是否启动、socket 权限是否正确。
2. Buildx/Compose 常见问题来自“插件未安装/版本不匹配/使用了旧版 docker-compose 二进制”。

# 2. Linux 安装

## 2.1 CentOS/RHEL 7

```bash
# 卸载旧版本
yum remove docker docker-client docker-common

# 安装依赖
yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加 Docker 仓库
yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo

# 安装 Docker CE
yum install -y docker-ce docker-ce-cli containerd.io

# 启动
systemctl start docker
systemctl enable docker
```

## 2.2 Ubuntu 20.04/22.04

```bash
apt update
apt install -y ca-certificates curl gnupg

# 添加 Docker GPG 密钥
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 添加仓库
echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装
apt update
apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

## 2.3 安装后建议做的 3 件事

1. 确认存储驱动与 cgroup driver（关系到性能与资源限制是否生效）。
2. 配置日志轮转（避免 `json-file` 日志无限增长）。
3. 配置镜像加速/代理（企业网络尤其重要）。

# 3. daemon.json 常用配置与风险边界

Docker daemon 常见配置文件：`/etc/docker/daemon.json`。

```json
{
  "registry-mirrors": [
    "https://registry.cn-hangzhou.aliyuncs.com",
    "https://mirror.ccs.tencentyun.com"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

应用配置：

```bash
systemctl daemon-reload
systemctl restart docker
```

关键配置项解释：

| 配置项 | 作用 | 备注 |
|--------|------|------|
| `registry-mirrors` | 镜像加速 | 解决 pull 慢/超时 |
| `log-driver` + `log-opts` | 日志驱动与轮转 | `json-file` 默认不轮转，必须显式配置 |
| `storage-driver` | 存储驱动 | Linux 生产通常用 `overlay2` |
| `exec-opts` | cgroup driver | 常见为 `systemd`，和宿主机一致更稳定 |
| `data-root` | Docker 数据目录 | 磁盘策略/迁移时很重要 |
| `insecure-registries` | 非 TLS 仓库 | **有安全风险**，尽量只用于内网测试 |

# 4. 镜像加速与企业代理

## 4.1 镜像加速

镜像加速本质是把对 Docker Hub 的请求导向镜像代理/缓存。

常见现象：

1. `i/o timeout`
2. `net/http: TLS handshake timeout`

优先排查：网络/代理、DNS、镜像加速器配置。

## 4.2 HTTP(S) 代理（常见于企业网络）

如果宿主机需要代理才能访问外网，Docker daemon 也需要配置代理（否则 `docker pull` 仍然失败）。

思路（不展开具体 systemd drop-in 细节）：

1. 给 docker service 配置 `HTTP_PROXY/HTTPS_PROXY/NO_PROXY`。
2. `NO_PROXY` 至少包含：`127.0.0.1,localhost,*.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,<你的私有仓库域名>`。

# 5. 非 root 运行 Docker（docker 组）

```bash
# 创建 docker 组并添加当前用户
groupadd docker
usermod -aG docker $USER

# 重新登录（或 newgrp docker）
newgrp docker
docker ps  # 无需 sudo
```

注意：把用户加入 `docker` 组，本质上等价于授予宿主机 root 级控制能力（因为 docker.sock 可控制宿主机资源）。在生产机器上要谨慎授权。

# 6. Windows（WSL2）

Windows 上常用 Docker Desktop + WSL2 后端。

```bash
# 1) 启用 WSL2
wsl --install

# 2) 安装 Docker Desktop
# https://docs.docker.com/desktop/windows/install/

# 3) 确保使用 WSL2 后端
wsl -l -v
#   NAME            STATE           VERSION
# * Ubuntu-22.04    Running         2
```

常见坑：

1. WSL2 发行版没开启 Docker Desktop 的集成，导致 Linux 终端中 `docker` 连接不到 daemon。
2. Windows 文件系统与 WSL2 文件系统的 I/O 性能差异，可能导致构建/挂载变慢（尤其 bind mount 大量小文件）。

# 7. 安装验证与自检清单

## 7.1 安装验证

```bash
docker version
docker run --rm hello-world
docker info
```

## 7.2 自检清单（常用排错入口）

```bash
docker info                 # 存储驱动、cgroup driver、rootless 等关键信息
systemctl status docker     # dockerd 是否启动
ls -la /var/run/docker.sock # socket 权限
```

常见问题定位：

| 现象 | 常见原因 | 快速定位 |
|------|----------|----------|
| `Cannot connect to the Docker daemon` | dockerd 未启动/socket 权限不足 | `systemctl status docker`、检查 docker.sock |
| pull 超时 | 代理/DNS/网络策略 | 配置镜像加速与 daemon 代理 |
| 资源限制不生效 | cgroup/driver 不匹配 | `docker info` 看 cgroup driver |

# 8. Docker Compose 安装（插件 vs 二进制）

现代 Docker 推荐安装 `docker-compose-plugin`，使用方式为：

```bash
docker compose version
```

如果你使用旧版二进制 `docker-compose`，常见问题是版本过旧或与引擎能力不匹配。

# 9. 卸载

```bash
# CentOS
yum remove docker-ce docker-ce-cli containerd.io
rm -rf /var/lib/docker /var/lib/containerd

# Ubuntu
apt purge docker-ce docker-ce-cli containerd.io
rm -rf /var/lib/docker /var/lib/containerd
```

# 10. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 安装 Docker 包含哪些组件 | CLI、dockerd、containerd、runc、Buildx/Compose 插件 |
| docker 组安全吗 | 等价 root 控制能力，生产要谨慎授权 |
| 为什么要配日志轮转 | `json-file` 默认不轮转，长期运行会撑爆磁盘 |
| pull 慢怎么处理 | 镜像加速、代理、DNS、并发配置 |
