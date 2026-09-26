
# 1. 包管理概述

## 1.1 为什么需要包管理

包管理器解决了软件安装的核心问题：
- **依赖管理**：自动解决库依赖
- **版本管理**：跟踪已安装软件的版本
- **完整性验证**：通过校验和确保包未被破坏
- **卸载干净**：可完全移除软件及相关配置
- **签名验证**：验证包的来源和完整性

## 1.2 主要包管理工具

| 发行版 | 包格式 | 底层工具 | 高层工具 |
|--------|--------|----------|----------|
| Debian/Ubuntu | .deb | dpkg | apt / apt-get |
| RHEL/CentOS/Fedora | .rpm | rpm | yum / dnf |
| Arch Linux | .pkg.tar.zst | pacman | — |
| openSUSE | .rpm | rpm | zypper |
| Alpine | .apk | apk | — |

# 2. dpkg — Debian 底层包工具

## 2.1 基本命令

```bash
# 安装
sudo dpkg -i package.deb              # 安装本地 .deb 包
sudo dpkg -i --force-depends package.deb  # 强制安装（忽略依赖）

# 卸载
sudo dpkg -r package                   # 卸载（保留配置文件）
sudo dpkg -P package                   # 彻底卸载（含配置文件）

# 查询
dpkg -l                                # 列出所有已安装包
dpkg -l | grep nginx                  # 搜索已安装包
dpkg -s package                        # 查看包状态
dpkg -L package                        # 列出包安装的文件
dpkg -S /path/to/file                  # 查找文件属于哪个包
dpkg -c package.deb                    # 查看 .deb 包内容

# 重新配置
sudo dpkg-reconfigure package          # 重新配置安装后的选项
```

## 2.2 dpkg 数据库

dpkg 维护一个本地数据库记录所有已安装包的状态：

```bash
/var/lib/dpkg/
├── available          # 可用包信息
├── status            # 已安装包状态
└── info/             # 每个包的详细信息
    ├── package.list  # 包包含的文件列表
    └── package.md5sums  # 文件的 MD5 校验和
```

## 2.3 dpkg 的局限性

- **不自动解决依赖**：必须手动安装所有依赖
- apt 等高层工具解决了此问题

# 3. apt — 高级包管理工具

## 3.1 基本命令

```bash
# 更新索引
sudo apt update                       # 从仓库更新包列表

# 安装
sudo apt install nginx                # 安装包（自动解决依赖）
sudo apt install ./package.deb        # 安装本地 .deb（自动安装依赖）
sudo apt install nginx=1.24.0-1       # 安装指定版本
sudo apt install --no-install-recommends nginx  # 不安装推荐的包

# 卸载
sudo apt remove nginx                 # 卸载（保留配置文件）
sudo apt purge nginx                  # 彻底卸载（含配置文件）
sudo apt autoremove                   # 卸载不需要的依赖

# 升级
sudo apt upgrade                      # 升级所有可升级的包
sudo apt full-upgrade                 # 升级并处理依赖变更（可能卸载包）
sudo apt dist-upgrade                 # 同上（全系统升级）
```

## 3.2 查询搜索

```bash
apt search keyword                    # 搜索包
apt show nginx                        # 查看包详细信息
apt list                              # 列出所有可用包
apt list --installed                  # 列出已安装
apt list --upgradable                 # 列出可升级包
apt-cache depends nginx               # 查看依赖关系
apt-cache rdepends nginx              # 查看哪些包依赖此包
```

## 3.3 apt 版本历史查看

```bash
apt list -a nginx                     # 查看所有可用版本
apt-cache policy nginx                # 查看安装的和可用的版本
```

## 3.4 apt vs apt-get

| 命令 | apt | apt-get |
|------|-----|---------|
| 安装 | `apt install` | `apt-get install` |
| 卸载 | `apt remove` | `apt-get remove` |
| 升级 | `apt upgrade` | `apt-get upgrade` |
| 更新 | `apt update` | `apt-get update` |
| 搜索 | `apt search` | `apt-cache search` |
| 信息 | `apt show` | `apt-cache show` |
| 编辑源 | `apt edit-sources` | 无 |

**apt** 是 apt-get 和 apt-cache 的子集，增加了进度条和彩色输出，适合交互操作。

# 4. 软件源配置

## 4.1 sources.list

配置文件位置：`/etc/apt/sources.list` 和 `/etc/apt/sources.list.d/`

```bash
# Ubuntu 22.04 (Jammy) 示例
# /etc/apt/sources.list
deb http://archive.ubuntu.com/ubuntu jammy main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu jammy-updates main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu jammy-backports main restricted universe multiverse
deb http://security.ubuntu.com/ubuntu jammy-security main restricted universe multiverse

# 格式说明
# deb <仓库URL> <发行版代号> <组件>
# 组件: main(官方支持), universe(社区), restricted(专有驱动), multiverse(非自由)
```

## 4.2 PPA（Personal Package Archive）

由 Launchpad 托管的第三方仓库：

```bash
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.12
```

## 4.3 国内镜像源

替换为国内镜像以加速下载：

```bash
# 设置阿里云镜像
sudo sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list

# 或使用清华镜像
sudo sed -i 's/archive.ubuntu.com/mirrors.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list
```

# 5. 依赖解决机制

## 5.1 依赖类型

```bash
# 查看依赖
apt depends curl
# curl
#   Depends: libc6
#   Depends: libcurl4
#   Recommends: ca-certificates
#   Suggests: fish
```

| 类型 | 含义 | 行为 |
|------|------|------|
| Depends | 硬依赖 | 必须安装 |
| Recommends | 推荐 | apt 默认安装（可用 `--no-install-recommends` 禁止）|
| Suggests | 建议 | 不自动安装 |
| Conflicts | 冲突 | 不能同时安装 |
| Breaks | 破坏 | 当前版本不兼容 |
| Replaces | 替换 | 替换其他包的文件 |

## 5.2 依赖解决算法

apt 使用 SAT 求解器（基于递推算法）解决复杂的依赖约束：

```bash
# 模拟安装（不实际执行）
apt install --simulate nginx

# 修复依赖问题
sudo apt --fix-broken install         # 修复损坏的依赖
```

## 5.3 常见问题处理

```bash
# 依赖关系损坏
sudo apt --fix-broken install

# 锁定问题
sudo rm /var/lib/apt/lists/lock
sudo rm /var/cache/apt/archives/lock
sudo rm /var/lib/dpkg/lock

# 升级发行版
sudo do-release-upgrade                # Ubuntu
```

# 6. 缓存与清理

```bash
# apt 下载的 .deb 包缓存
ls /var/cache/apt/archives/

# 清理
sudo apt clean                         # 清除所有缓存的 .deb
sudo apt autoclean                     # 清除不再可下载的过时 .deb

# 查看磁盘使用
du -sh /var/cache/apt/
```

# 7. 源码编译安装

## 7.1 编译安装流程

```bash
# 1. 安装构建工具
sudo apt install build-essential
sudo apt build-dep nginx               # 安装编译 nginx 所需的所有依赖

# 2. 下载源码
apt source nginx                       # 下载包源码
# 或直接从官网下载 tar.gz

# 3. 配置
cd nginx-1.24.0
./configure --prefix=/usr/local/nginx --with-http_ssl_module

# 4. 编译
make -j$(nproc)                        # -j 并行编译

# 5. 安装
sudo make install

# 6. 清理
make clean
```

## 7.2 使用 checkinstall 替代 make install

编译安装后用 dpkg 管理，便于后续卸载：

```bash
sudo apt install checkinstall
./configure && make && sudo checkinstall
# 会创建 .deb 包并安装，dpkg 可跟踪
```

# 8. 快照与回滚

## 8.1 apt 日志

```bash
# 查看 apt 操作历史
less /var/log/apt/history.log
less /var/log/apt/term.log

# 查看 dpkg 日志
less /var/log/dpkg.log
```

## 8.2 回滚操作

apt 本身不提供回滚功能，但可通过以下方式实现：

```bash
# 方法1：从日志获取之前版本
apt install package=version

# 方法2：使用 apt-listbugs 阻止已知故障包
sudo apt install apt-listbugs

# 方法3：使用 etckeeper 跟踪 /etc 变更
sudo apt install etckeeper
```

# 9. apt 安全

## 9.1 GPG 签名验证

apt 使用 GPG 验证每个包的签名，确保未被篡改：

```bash
# 添加第三方 GPG 密钥
sudo apt-key add keyfile.asc           # 已弃用的方式

# 现代方式
sudo cp keyfile.asc /etc/apt/trusted.gpg.d/

# 列出已添加的密钥
apt-key list
```

## 9.2 安全安装确认

```bash
# 包信息中的签名状态
apt show nginx
# ...
# SHA256: ...
# Signatures: ...
```

# 10. 批量管理

```bash
# 列出需要升级的包
apt list --upgradable

# 批量升级
sudo apt upgrade

# 查看安全更新
apt list --upgradable 2>/dev/null | grep security

# 自动删除孤儿包
sudo apt autoremove --purge

# 筛选已安装包
dpkg --get-selections | grep -v deinstall
```
