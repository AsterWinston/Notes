
> 本文以 CentOS 7/8 和 Ubuntu 20.04/22.04 为主要示例环境。

---

# 1. 包管理器安装

## 1.1 CentOS / RHEL（yum / dnf）

```bash
# 安装 EPEL 源（CentOS 7）
yum install -y epel-release
yum install -y nginx

# CentOS 8 / RHEL 8
dnf install -y nginx

# 启动与开机自启
systemctl start nginx
systemctl enable nginx
```

## 1.2 Ubuntu / Debian（apt）

```bash
apt update
apt install -y nginx

# 启动与开机自启
systemctl start nginx
systemctl enable nginx
```

> 包管理器安装的版本通常较旧，可通过配置官方仓库获取最新版本。

---

# 2. 官方仓库安装（推荐）

```bash
# 以 CentOS 7 为例 —— 配置 Nginx 官方 yum 源
cat > /etc/yum.repos.d/nginx.repo << 'EOF'
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF

yum install -y nginx
```

---

# 3. 源码编译安装

```bash
# 安装编译依赖（CentOS）
yum install -y gcc pcre-devel zlib-devel openssl-devel make

# 下载源码
wget http://nginx.org/download/nginx-1.24.0.tar.gz
tar -zxvf nginx-1.24.0.tar.gz
cd nginx-1.24.0

# 配置（常用模块说明见下方）
./configure \
    --prefix=/usr/local/nginx \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-http_realip_module \
    --with-http_gzip_static_module \
    --with-http_stub_status_module \
    --with-stream \
    --with-stream_ssl_module

# 编译与安装
make && make install
```

| 编译选项 | 作用 |
|----------|------|
| `--with-http_ssl_module` | HTTPS/SSL 支持 |
| `--with-http_v2_module` | HTTP/2 支持 |
| `--with-http_realip_module` | 获取客户端真实 IP |
| `--with-http_gzip_static_module` | 预压缩文件支持 |
| `--with-http_stub_status_module` | 状态监控 |
| `--with-stream` | TCP/UDP 四层代理 |

---

# 4. 基础操作命令

```bash
nginx                      # 启动
nginx -s stop              # 快速停止
nginx -s quit              # 优雅停止（处理完当前请求）
nginx -s reload            # 平滑重载配置
nginx -t                   # 检查配置文件语法
nginx -T                   # 检查配置并打印所有配置内容
nginx -V                   # 查看编译参数与版本
ps -ef | grep nginx        # 查看进程
```

---

# 5. 目录结构（yum 安装）

| 路径 | 说明 |
|------|------|
| `/etc/nginx/nginx.conf` | **主配置文件** |
| `/etc/nginx/conf.d/` | 子配置文件目录（`*.conf` 会自动包含） |
| `/usr/sbin/nginx` | 可执行文件 |
| `/usr/share/nginx/html/` | 默认站点目录 |
| `/var/log/nginx/` | 日志目录（access.log / error.log） |
| `/var/cache/nginx/` | 缓存目录 |
| `/etc/nginx/mime.types` | MIME 类型定义文件 |

## 5.1 源码编译安装目录

| 路径 | 说明 |
|------|------|
| `/usr/local/nginx/sbin/nginx` | 可执行文件 |
| `/usr/local/nginx/conf/nginx.conf` | 主配置文件 |
| `/usr/local/nginx/html/` | 默认站点目录 |
| `/usr/local/nginx/logs/` | 日志目录 |

---

# 6. Docker 安装（快速体验）

```bash
docker run -d --name nginx \
    -p 80:80 \
    -v /host/html:/usr/share/nginx/html:ro \
    -v /host/nginx.conf:/etc/nginx/nginx.conf:ro \
    nginx:1.24
```
