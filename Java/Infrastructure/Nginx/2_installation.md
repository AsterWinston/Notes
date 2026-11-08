
> 本文以 CentOS 7/8 和 Ubuntu 20.04/22.04 为主要示例环境。

安装 Nginx 看起来是抄一条命令的事，但选错来源会让后面的路很别扭：发行版自带的 Nginx 版本常年停留在老分支，缺新指令、带补丁差异；源码编译装完没有 systemd 单元、日志轮转要自己管。本篇按"发行版包 → 官方仓库 → 源码编译 → Docker"四条路线梳理取舍，再统一讲清目录布局、常用运维命令、systemd 开机自启与生产升级回滚流程。

---

# 1. 三种安装方式对比

动手之前先选型。三种路线的差别不在"能不能跑"，而在**版本新旧、模块定制能力、以及后续运维是否省心**：

| 维度 | 发行版包（yum/apt） | 官方仓库 | 源码编译 |
|------|--------------------|----------|----------|
| 版本 | 旧（CentOS 7 停在 1.20，Ubuntu 20.04 停在 1.18） | 跟随官方 Stable/Mainline | 自选任意版本 |
| 安装/升级 | 一条命令，随系统更新 | 一条命令，`yum upgrade nginx` | 手动 configure/make，升级需替换二进制 |
| 模块 | 发行版预先编好，缺什么补什么包 | 官方预编译，模块固定 | 自由裁剪，可加第三方模块 |
| 用户/路径约定 | 遵循 FHS（/etc/nginx 等） | 同左 | 默认 /usr/local/nginx，一切自己定 |
| systemd/日志轮转 | 自带 | 自带 | 需自己写 unit 与 logrotate |
| 适用场景 | 快速起步、内网不敏感版本 | **生产推荐（默认选择）** | 需要第三方模块或极致裁剪 |

发行版包最大的问题是**版本落后**：CentOS 7 的 base/EPEL 源里 nginx 长期是 1.20 之前，很多新指令（如 `http2 on;`、`proxy_ssl_name` 相关修复）要么没有要么行为不同，而教程又大多基于新版，排查时容易踩"配置合法但不生效"的坑。

> **行业实践**：生产环境默认走**官方仓库**——版本新、维护省心、路径规范。只有当需要编译进第三方 C 模块（如 Lua 之外的某些限流/去重模块、Tengine 的 check 模块）、或对镜像/二进制做极致裁剪时才上源码编译，而这类需求如今更多用 OpenResty 预编译包或容器镜像解决。

---

# 2. 包管理器安装

## 2.1 CentOS / RHEL（yum / dnf）

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

CentOS 7 的 nginx 不在 base 仓库里，必须先装 EPEL，否则 `yum install nginx` 直接报 "No package nginx available"。RHEL 8+ 移到了 AppStream 模块流，可以用 `dnf module list nginx` 查看可选版本流。

## 2.2 Ubuntu / Debian（apt）

```bash
apt update
apt install -y nginx

# 启动与开机自启
systemctl start nginx
systemctl enable nginx
```

Debian 系包结构有一个与 CentOS 不同的约定：配置入口是 `/etc/nginx/nginx.conf`（含 `include /etc/nginx/sites-enabled/*`），**站点文件放 `sites-available/`，用 `ln -s` 启用**；而 CentOS 官方包用 `conf.d/*.conf`。跨发行版抄配置时注意这个目录差异。安装完成后 Ubuntu 会立刻启动一个 default 站点监听 80，本机要跑别的 Web 服务时先 `rm /etc/nginx/sites-enabled/default`。

> 包管理器安装的版本通常较旧，可通过配置官方仓库获取最新版本。

---

# 3. 官方仓库安装（推荐）

官方仓库由 nginx.org 维护，同时提供 **stable** 与 **mainline** 两条线，GPG key 校验，装完即带 systemd unit 与 logrotate 配置。

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

Ubuntu/Debian 对应做法：

```bash
# 导入官方 GPG key 与源
curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
    -o /usr/share/keyrings/nginx-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/ubuntu jammy nginx" > /etc/apt/sources.list.d/nginx.list
apt update && apt install -y nginx
```

装完可用 `nginx -V` 确认版本与编译参数。国内访问 nginx.org 慢时，把 baseurl 换成镜像站（如 `mirrors.aliyun.com` 上的 nginx 包镜像），或用 `yum install yum-utils && yumdownloader` 下 rpm 离线分发。

> **注意**：如果机器上先用发行版包装过 nginx，再切官方源，`yum install nginx` 可能因 Vendor 不同被拒绝或降级，先 `rpm -qa | grep nginx` 检查并统一来源；两者配置文件路径一致，冲突点主要在 `nginx.conf` 默认内容不同，切换前备份 /etc/nginx。

---

# 4. 源码编译安装

源码安装适合需要**定制模块集合**的场景，流程是标准的 GNU 三段式。先装编译链与依赖库——PCRE 提供正则（rewrite 依赖）、zlib 提供 gzip、openssl-devel 提供 TLS：

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

**理解 configure 的默认行为很重要**：不带任何 `--with` 时，Nginx 只编译 http 核心、events、auto-index 等**最小模块集**（`nginx -V` 可见 `--without-http_*` 系列并未被排除，但 ssl/v2/stream 等默认**不在**编译里）。也就是说源码装完发现 `listen 443 ssl` 报 "unknown parameter"，多半是编译时忘了 `--with-http_ssl_module`，只能重新 configure + make，没有运行时动态加载一说（除非用 Tengine 的动态模块或 Lua）。

| 编译选项 | 作用 |
|----------|------|
| `--with-http_ssl_module` | HTTPS/SSL 支持 |
| `--with-http_v2_module` | HTTP/2 支持 |
| `--with-http_realip_module` | 获取客户端真实 IP |
| `--with-http_gzip_static_module` | 预压缩文件支持 |
| `--with-http_stub_status_module` | 状态监控 |
| `--with-stream` | TCP/UDP 四层代理 |

其他常见选项补充：

```bash
./configure \
    --user=nginx --group=nginx \            # Worker 运行账号
    --with-pcre-jit \                       # PCRE JIT，复杂正则提速明显
    --with-http_sub_module \                # 响应内容替换
    --with-http_auth_request_module \       # 子请求鉴权
    --add-module=/opt/nginx-module-xxx      # 第三方 C 模块
```

> **注意**：`--prefix` 只是"安装根目录"，并不等于配置里写的路径；源码安装默认以 prefix 相对路径找配置和日志（`/usr/local/nginx/conf/nginx.conf`），与系统包 `/etc/nginx` 的 FHS 约定完全不同。第三方模块与目标版本的源码兼容性要逐个确认，编译期报符号错误通常是模块落后于 Nginx 核心 API。

## 4.1 源码安装的 systemd 单元

源码安装不带服务文件，需自行创建 `/etc/systemd/system/nginx.service`：

```ini
[Unit]
Description=nginx - high performance web server
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
LimitNOFILE=65535
```

```bash
systemctl daemon-reload && systemctl enable --now nginx
```

`Type=forking` 是因为 Nginx Master 启动后会 fork 并让父进程退出，systemd 需要靠 `PIDFile` 跟踪主进程；`ExecStartPre` 里挂 `nginx -t` 能保证配置有语法错误时服务直接拒绝启动，避免"重启后起不来"。

---

# 5. 基础操作命令

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

这组命令里最值得展开的是 `-s` 信号语义和 `-t/-T` 的差异：

| 命令/信号 | 行为 | 使用场景 |
|-----------|------|----------|
| `nginx -s stop`（TERM） | 立即退出，丢弃在途请求 | 紧急止血、测试环境 |
| `nginx -s quit`（QUIT） | 处理完当前请求再退出 | 计划内停机 |
| `nginx -s reload`（HUP） | Master 重读配置，旧 Worker 干完活退出，新 Worker 用新配置接手 | 改配置后生效 |
| `nginx -s reopen`（USR1） | 重新打开日志文件 | 配合 logrotate 切割日志 |
| `nginx -t` | 仅做语法 + 语义检查（含文件是否存在、upstream 名字合法性） | 任何变更前的第一步 |
| `nginx -T` | 在 `-t` 基础上打印展开 include 后的完整配置 | 排查"实际生效的是哪份配置" |
| `nginx -V` | 版本 + 编译参数（可判断模块是否编入） | 排错必备 |

> **注意**：`nginx -s reload` 的前提是**旧配置正确**——它给正在运行的 Master 发信号；若 Master 没在跑，`-s` 全部无效。改坏配置导致 reload 失败时，运行中的实例仍按旧配置服务，此时修复文件后再次 reload 即可，无需重启，这正是热部署的安全网。`-T` 输出含私钥路径等敏感信息，注意脱敏。

---

# 6. 目录结构（yum 安装）

| 路径 | 说明 |
|------|------|
| `/etc/nginx/nginx.conf` | **主配置文件** |
| `/etc/nginx/conf.d/` | 子配置文件目录（`*.conf` 会自动包含） |
| `/usr/sbin/nginx` | 可执行文件 |
| `/usr/share/nginx/html/` | 默认站点目录 |
| `/var/log/nginx/` | 日志目录（access.log / error.log） |
| `/var/cache/nginx/` | 缓存目录 |
| `/etc/nginx/mime.types` | MIME 类型定义文件 |

## 6.1 源码编译安装目录

| 路径 | 说明 |
|------|------|
| `/usr/local/nginx/sbin/nginx` | 可执行文件 |
| `/usr/local/nginx/conf/nginx.conf` | 主配置文件 |
| `/usr/local/nginx/html/` | 默认站点目录 |
| `/usr/local/nginx/logs/` | 日志目录 |

两套布局的对照关系是固定心智模型：**系统包按 FHS 拆开**（配置进 /etc、二进制进 /usr/sbin、日志进 /var、缓存进 /var/cache/nginx，proxy_cache 默认就在这），**源码安装全部收敛在 prefix 之下**。排错第一步是确认"我改的是不是实际加载的那份配置"：`ps -ef | grep "nginx: master"` 看启动命令行里的 `-c` 参数，或直接 `nginx -T | head` 看加载路径。

---

# 7. Docker 安装（快速体验）

```bash
docker run -d --name nginx \
    -p 80:80 \
    -v /host/html:/usr/share/nginx/html:ro \
    -v /host/nginx.conf:/etc/nginx/nginx.conf:ro \
    nginx:1.24
```

官方镜像 `nginx:1.24`（Stable）/ `nginx:1.25`（Mainline）/ `nginx:alpine`（体积小但基于 musl libc）三条线可选，配置遵循 Debian 包约定，站点文件挂载到 `/etc/nginx/conf.d/` 即可自动生效。

> **注意**：挂载单个 `nginx.conf` 文件时，宿主上如果用编辑器"另存新文件替换"的方式更新，inode 会变，容器里看到的内容不会更新（bind mount 跟着旧 inode）；生产更稳的做法是挂载**目录**（如 `-v /host/conf.d:/etc/nginx/conf.d:ro`），改完目录内文件后 `docker exec nginx nginx -s reload`。容器内 Nginx 默认 `worker_processes auto` 会看到宿主机核数而非 cgroup 限额，CPU 限制严格的小容器需要手工指定为 1。

---

# 8. 平滑升级与回滚

源码安装下升级新版本，核心手法是"**只换二进制，不动配置，发 USR2**"，全程不中断服务：

```bash
# 1) 新介质编译：make 之后【不要 make install】，否则会覆盖旧文件
make

# 2) 备份旧二进制，替换新二进制
mv /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.old
cp objs/nginx /usr/local/nginx/sbin/nginx

# 3) 校验新二进制与现有配置兼容
/usr/local/nginx/sbin/nginx -t

# 4) 向旧 Master 发 USR2：拉起新 Master/Worker，两套实例并存
kill -USR2 $(cat /usr/local/nginx/logs/nginx.pid)

# 5) 旧 Master 优雅退休：WINCH 让旧 Worker 处理完存量连接后退出
kill -QUIT $(cat /usr/local/nginx/logs/nginx.pid.oldbin)
```

```text
升级时序（两套实例并存，端口由旧 Master 的监听 fd 继承共享）：

 旧Master ──USR2──▶ 新Master 启动并绑定成功
    │                  │
    ├─ 旧Worker×N 服务中┤新Worker×N 开始接受新连接
    │                  │
    └─WINCH─▶ 旧Worker 处理完在途请求后退出
               旧Master 收到 QUIT 后退出
```

**回滚**发生在"新实例有问题"时：对新 Master 发 `QUIT`（或 `TERM`），再对旧 Master 发 `HUP` 唤醒旧 Worker 继续接手，二进制换回 `nginx.old` 即可——这也是第 4 步要求保留 oldbin 和旧进程一段时间的原因。用包管理器升级则简单得多：`yum upgrade nginx` 覆盖二进制后，systemd/nginx 自身会触发平滑重载流程，无需手工 USR2。

> **行业实践**：不管哪种升级路径，动手前三件事——`nginx -T` 导出当前生效配置存档、备份二进制、在低峰期执行并盯着 error.log。跨大版本升级（如 1.22 → 1.26）先读官方 changelog 里的**不兼容变更**条目（历史上的例子：`listen http2` 参数被 `http2 on;` 取代），必要时先在预发环境用生产全量配置跑一遍 `nginx -t` 和核心流量回归。

---

# 9. 小结

| 问题 | 结论 |
|------|------|
| 用哪个来源装 | 默认官方仓库（版本新 + 带 systemd/logrotate）；发行版包仅图省事；要第三方 C 模块才源码编译 |
| 源码装的关键坑 | 模块在编译期决定（ssl/stream 默认不带）；路径体系与 FHS 不同；unit 与轮转自建 |
| 变更配置的标准动作 | `nginx -t` → `nginx -s reload`；reload 失败不影响在跑的实例 |
| 版本/模块排查 | `nginx -V` 看编译参数，`nginx -T` 看实际生效配置 |
| 升级 | 换二进制 + USR2 双 Master 并存 → WINCH/QUIT 旧实例退休；回滚 = QUIT 新 + HUP 旧 |
| Docker | 挂目录不挂单文件；CPU 限额下 worker_processes 手工指定 |

下一篇（3_core_concepts）深入 Master/Worker 进程模型、事件循环与信号机制，与本篇的升级回滚流程正好互相印证。
