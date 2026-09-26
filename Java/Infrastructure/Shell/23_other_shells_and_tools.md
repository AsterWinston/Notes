
# 1. Zsh 深入

Zsh（Z Shell）自 macOS Catalina (2019) 起成为默认 Shell，以其强大的插件生态和高度可定制化著称。它基本兼容 Bash，但拥有大量 Bash 不具备的增强特性。

## 1.1 安装与基本配置

**各平台安装：**

```bash
# macOS（预装，但可通过 Homebrew 安装最新版）
brew install zsh
# 查看 macOS 自带版本
/bin/zsh --version

# Debian/Ubuntu
sudo apt update && sudo apt install -y zsh

# RHEL/CentOS/Fedora
sudo dnf install -y zsh

# Arch
sudo pacman -S zsh

# 验证安装
zsh --version
```

**切换默认 Shell：**

```bash
# 查看当前 Shell
echo $SHELL

# 查看可用 Shell 列表
cat /etc/shells

# 切换默认 Shell（需要重新登录生效）
chsh -s $(which zsh)

# 或直接
chsh -s /bin/zsh
```

**Zsh 启动文件加载顺序（关键！）：**

```
登录 Shell:
  /etc/zprofile            # 全局登录配置
  → ~/.zprofile            # 用户登录配置（一般放环境变量）
  → /etc/zshrc             # 全局交互配置
  → ~/.zshrc               # 用户交互配置（主题/插件/别名都在此）

非登录交互 Shell:
  /etc/zshrc
  → ~/.zshrc

脚本（非交互、非登录）:
  不读取任何 RC 文件（与 Bash 相同，通过脚本 shebang 控制）
```

**最小化 `.zshrc` 模板：**

```zsh
# ~/.zshrc 最小化配置
# ============================================

# -- 环境变量（放 .zprofile 也可） --
export EDITOR='nvim'
export LANG='zh_CN.UTF-8'

# -- 历史记录 --
HISTSIZE=50000
SAVEHIST=50000
HISTFILE=~/.zsh_history
setopt SHARE_HISTORY          # 多终端共享历史
setopt HIST_IGNORE_DUPS       # 忽略连续重复命令
setopt HIST_IGNORE_SPACE      # 以空格开头的命令不记录

# -- 补全系统 --
autoload -Uz compinit && compinit
zstyle ':completion:*' menu select   # 菜单式补全
zstyle ':completion:*' matcher-list 'm:{a-zA-Z}={A-Za-z}'  # 大小写不敏感

# -- 常用选项 --
setopt AUTO_CD              # 输入目录名自动 cd
setopt AUTO_PUSHD           # cd 自动入栈
setopt CORRECT              # 拼写纠正（慎用，有时误触发）
setopt EXTENDED_GLOB        # 启用增强通配符

# -- Vi 模式（可选） --
# bindkey -v

# -- 自动补全高亮 --
zmodload zsh/complist
```

## 1.2 Oh-My-Zsh

Oh-My-Zsh 是最流行的 Zsh 配置框架，提供了主题、插件和便捷配置的生态系统。

**安装：**

```bash
# 官方安装脚本
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# 或通过 wget
sh -c "$(wget https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh -O -)"
```

**目录结构：**

```
~/.oh-my-zsh/
├── lib/              # 核心库文件（自动加载）
│   ├── theme-and-appearance.zsh
│   ├── history.zsh
│   ├── completion.zsh
│   └── ...
├── plugins/          # 官方插件目录
│   ├── git/
│   ├── kubectl/
│   ├── docker/
│   └── ...
├── themes/           # 主题目录
└── custom/           # 用户自定义（不会被更新覆盖）
    ├── plugins/      # 自定义插件
    └── themes/       # 自定义主题
```

**`.zshrc` 中 Oh-My-Zsh 配置示例：**

```zsh
# Oh-My-Zsh 安装路径
export ZSH="$HOME/.oh-my-zsh"

# 主题选择
ZSH_THEME="robbyrussell"   # 默认主题

# 插件列表（按需加载，每个插件增加启动时间）
plugins=(
    git                     # 大量 git 别名和函数
    z                       # 智能目录跳转（基于频率）
    extract                 # 万能解压命令（自动识别格式）
    docker                  # docker 命令补全
    kubectl                 # kubectl 补全和别名
    fzf                     # 模糊搜索集成
    sudo                    # 按两次 Esc 在命令前加 sudo
    colored-man-pages       # man 手册语法高亮
    command-not-found       # 命令未找到时提示安装包名
    copybuffer              # Ctrl+O 复制命令行缓冲区
    web-search              # 从终端搜索
    history                 # 历史命令相关别名
)

source $ZSH/oh-my-zsh.sh

# 自定义别名
alias zshconfig="$EDITOR ~/.zshrc"
alias ohmyzsh="$EDITOR ~/.oh-my-zsh"
```

**常用内置插件详解：**

| 插件 | 功能 | 关键别名/函数 |
|------|------|--------------|
| `git` | Git 增强，约 150+ 别名 | `gst`=status, `ga`=add, `gco`=checkout, `gl`=pull, `gp`=push, `gcb`=checkout -b |
| `z` | 频率学习目录跳转 | `z foo` 跳转到最常访问的包含 foo 的目录 |
| `extract` | 万能解压 | `extract file.{tar.gz,zip,rar,7z,...}` 自动识别 |
| `docker` | Docker 补全 | `dc`=docker-compose 别名链 |
| `sudo` | Esc 两次加 sudo | 无需移动到行首 |
| `copypath` | 复制当前路径 | `copypath` 复制绝对路径到剪贴板 |

**Oh-My-Zsh 的性能问题：**

Oh-My-Zsh 最大的痛点是启动速度。默认加载所有 lib 文件加上大量插件，启动时间可能超过 0.5 秒。解决方案：

```zsh
# 使用 omz 的延迟加载（oh-my-zsh 自带）
zsh -i -c 'time exit'   # 先测启动时间

# 减少插件数量 → 只加载真正需要的
# 或换用 zinit 实现 Turbo 懒加载（见 1.1.4）
# 或换用 prezto（更轻量的 OMZ 替代）
```

## 1.3 Powerlevel10k

Powerlevel10k (p10k) 是目前 Zsh 最流行的主题，以美观、信息丰富、速度快著称。它是 Powerlevel9k 的纯 Zsh 重写，速度提升约 100 倍。

**安装（配合 Oh-My-Zsh）：**

```bash
# 1. 安装推荐字体（Meslo Nerd Font 或任意 Nerd Font）
#    https://github.com/romkatv/powerlevel10k#fonts
#    macOS: brew install --cask font-meslo-lg-nerd-font

# 2. 克隆主题到 OMZ 自定义主题目录
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k

# 3. 在 .zshrc 中设置
# ZSH_THEME="powerlevel10k/powerlevel10k"

# 4. 重启终端，首次运行自动进入配置向导
```

**安装（配合 zinit，推荐）：**

```zsh
# 在 .zshrc 中（zinit 方式，支持 Turbo 加载）
zinit ice depth=1; zinit light romkatv/powerlevel10k

# 可选：开启即时提示（Instant Prompt）
# 放在 .zshrc 最顶部
if [[ -r "${XDG_CACHE_HOME:-$HOME/.cache}/p10k-instant-prompt-${(%):-%n}.zsh" ]]; then
  source "${XDG_CACHE_HOME:-$HOME/.cache}/p10k-instant-prompt-${(%):-%n}.zsh"
fi
```

**p10k 配置项说明（`~/.p10k.zsh`）：**

```zsh
# p10k 完整配置可通过 p10k configure 交互生成
# 以下是部分关键配置项说明

# -- 左侧提示符（Prompt）结构 --
# 典型布局：os_icon dir vcs
POWERLEVEL9K_LEFT_PROMPT_ELEMENTS=(
    os_icon                 # 操作系统图标
    dir                     # 当前目录（支持缩短）
    vcs                     # VCS 信息（git 分支/状态）
    # prompt_char           # 提示符字符 ❯
)

# -- 右侧提示符（RPROMPT）结构 --
POWERLEVEL9K_RIGHT_PROMPT_ELEMENTS=(
    status                  # 上一条命令退出码
    command_execution_time  # 执行时间（超过阈值才显示）
    background_jobs         # 后台任务数
    virtualenv              # Python 虚拟环境
    node_version            # Node.js 版本
    time                    # 当前时间
)

# -- 目录截断风格 --
# 默认只显示当前目录名，可改为全路径或智能截断
POWERLEVEL9K_SHORTEN_STRATEGY=truncate_to_unique   # 唯一前缀截断
POWERLEVEL9K_SHORTEN_DELIMITER='...'
POWERLEVEL9K_SHORTEN_DIR_LENGTH=1

# -- 即时提示（Instant Prompt）--
# p10k 的核心创新：在 zsh 加载插件前先显示缓存的提示符
# 原理：第一次提示符渲染后缓存到文件，下次启动时直接读取缓存
# 用户体验：打开终端 → 即时显示提示符 → 后台加载插件（无感知延迟）
```

**p10k 分段详解：**

| 分段 | 含义 | 显示条件 |
|------|------|----------|
| `dir` | 当前工作目录 | 始终显示；可读/写权限异常时变色 |
| `vcs` | 版本控制状态 | 在 Git/Mercurial 仓库中时显示分支、★ 脏标记、↑↓ 未推送 |
| `status` | 退出码 | 非零时显示；支持信号名（如 KILL、SEGV） |
| `command_execution_time` | 命令耗时 | 超过阈值（默认 3s）显示，格式如 `4m32s` |
| `background_jobs` | 后台任务数 | 有后台作业时显示数量 |
| `virtualenv` | Python venv | 激活虚拟环境时显示名称 |
| `aws` | AWS profile | `AWS_PROFILE` 环境变量非 default 时显示 |
| `kubecontext` | K8s 上下文 | 设置 `POWERLEVEL9K_KUBECONTEXT_SHOW=true` |

## 1.4 Zinit（原 zplugin）

Zinit 是目前最强大的 Zsh 插件管理器，核心优势是 Turbo Mode（延迟/异步加载插件）和代码片段（snippet）支持。

**安装 Zinit：**

```bash
# 自动安装脚本
bash -c "$(curl -fsSL https://raw.githubusercontent.com/zdharma-continuum/zinit/main/scripts/install.sh)"

# 或手动安装
ZINIT_HOME="${XDG_DATA_HOME:-${HOME}/.local/share}/zinit/zinit.git"
git clone https://github.com/zdharma-continuum/zinit.git "$ZINIT_HOME"
```

**Zinit 基础语法：**

```zsh
# .zshrc 中配置 Zinit
ZINIT_HOME="${XDG_DATA_HOME:-${HOME}/.local/share}/zinit/zinit.git"
source "${ZINIT_HOME}/zinit.zsh"

# === 基本用法 ===

# 1. light — 普通加载（无跟踪，无 Turbo）
zinit light zsh-users/zsh-autosuggestions

# 2. load — 跟踪模式加载（可后续用 report / times 查看速度）
zinit load zsh-users/zsh-syntax-highlighting

# 3. snippet — 加载远程代码片段（如 OMZ 库文件）
zinit snippet OMZ::lib/completion.zsh
zinit snippet OMZ::lib/history.zsh

# 4. ice + load — 带修饰符的加载（核心用法）
# ice = "instruction for compilation and execution"
zinit ice wait'0' lucid
zinit load zsh-users/zsh-completions

# --- ice 修饰符速查 ---
# wait'0'      : Turbo Mode，延迟到 prompt 后加载（0=立即开始）
# wait'1'      : 延迟 1 秒后加载（给 prompt 更多渲染时间）
# lucid        : 跳过加载提示信息
# atload'cmd'  : 插件加载完成后执行的命令
# atinit'cmd'  : 插件加载前执行的命令
# blockf       : 阻止插件修改 fpath
# silent       : 静默模式
# as'program'  : 将插件作为命令安装
# src'file'    : 指定要 source 的文件
# pick'file'   : 指定要加载的主文件
# depth'1'     : git clone 深度（节省空间）
# from'gh-r'   : 从 GitHub Releases 下载（而非 clone 仓库）
# mv, cp       : 安装后移动/复制文件
# compile      : 编译插件脚本为 zwc 加速
# nocompile    : 不编译
```

**Turbo Mode 原理：**

```
传统加载：  source p1 → source p2 → source p3 → 显示 prompt
Turbo 加载： 显示 prompt → [后台 source p1] → [后台 source p2] → ...
              ↑ 用户立即可以输入命令

实现机制：
1. Zsh 支持 zle（Zsh Line Editor），允许在行编辑器激活时执行后台任务
2. zinit 利用 zle -F 监听文件描述符，在 I/O 就绪时加载插件
3. wait'0' 让 zinit 立即将插件加入后台加载队列，prompt 先渲染
```

**完整 `.zshrc` 示例（Zinit + p10k）：**

```zsh
# ===== Zinit 初始化 =====
ZINIT_HOME="${XDG_DATA_HOME:-${HOME}/.local/share}/zinit/zinit.git"
[ ! -d "$ZINIT_HOME" ] && mkdir -p "$(dirname "$ZINIT_HOME")"
[ ! -d "$ZINIT_HOME/.git" ] && \
    git clone https://github.com/zdharma-continuum/zinit.git "$ZINIT_HOME"
source "${ZINIT_HOME}/zinit.zsh"

# ===== OMZ 代码片段（Turbo） =====
zinit wait'0a' lucid for \
    OMZ::lib/git.zsh \
    OMZ::lib/completion.zsh \
    OMZ::lib/history.zsh \
    OMZ::lib/key-bindings.zsh \
    OMZ::lib/theme-and-appearance.zsh

# ===== 核心插件（同步加载，影响 prompt） =====
zinit load romkatv/powerlevel10k               # p10k 主题（同步，prompt 需要）

# ===== 功能插件（Turbo 懒加载） =====
zinit wait'0b' lucid for \
    zsh-users/zsh-autosuggestions              # 命令自动建议（灰色提示）
zinit wait'0b' lucid for \
    zsh-users/zsh-syntax-highlighting          # 命令语法高亮
zinit wait'0b' lucid for \
    zsh-users/zsh-completions                  # 额外补全定义
zinit wait'0b' lucid for \
    OMZ::plugins/git/git.plugin.zsh            # git 别名

# ===== 二进制工具（from GitHub Releases） =====
zinit ice from'gh-r' as'program' mv'fd* -> fd' pick'fd/fd'; zinit light sharkdp/fd
zinit ice from'gh-r' as'program' pick'*/bat'; zinit light sharkdp/bat

# ===== 手动补全加载 =====
autoload -Uz compinit && compinit

# ===== p10k 配置（由 p10k configure 生成） =====
[[ -f ~/.p10k.zsh ]] && source ~/.p10k.zsh
```

**Zinit 诊断命令：**

```bash
zinit times        # 查看各插件加载耗时（需要 load 模式）
zinit report       # 查看各插件详细信息
zinit status       # 所有插件状态
zinit update --all # 更新所有插件
zinit delete 插件名  # 删除插件
zinit cclean       # 清理未使用的插件
zinit cd 插件名     # 进入插件源码目录
```

## 1.5 Zplug（另一个选择）

Zplug 是另一款注重简洁的插件管理器，语法更直观（类似 Vim-Plug）。

```zsh
# zplug 安装
curl -sL --proto-redir -all,https https://raw.githubusercontent.com/zplug/installer/master/installer.zsh | zsh

# 配置示例
source ~/.zplug/init.zsh

# 插件声明（类似 vim-plug）
zplug "zsh-users/zsh-syntax-highlighting", defer:2   # defer 延迟加载
zplug "zsh-users/zsh-autosuggestions", defer:2
zplug "plugins/git", from:oh-my-zsh

# 管理命令
zplug "zplug/zplug", hook-build:'zplug --self-manage'

# 安装未安装的插件
if ! zplug check --verbose; then
    printf "Install? [y/N]: "
    if read -q; then
        echo; zplug install
    fi
fi

# 加载插件
zplug load
```

## 1.6 Bash 迁移到 Zsh 指南

从 Bash 迁移到 Zsh 时，大部分交互式使用体验无缝，但脚本编写有差异需要留意。

**兼容矩阵：**

| 特性 | Bash | Zsh | 迁移建议 |
|------|------|-----|----------|
| 索引数组 `arr=(a b c)` | ✅ | ✅ | 完全兼容 |
| 关联数组 `declare -A` | ✅ (4.0+) | ✅ | 兼容 |
| `[[ ]]` 条件测试 | ✅ | ✅ | 兼容，但部分细节不同 |
| `${var:offset:len}` | ✅ | ✅ | 兼容 |
| `${var/pat/repl}` | ✅ | ✅ | 兼容 |
| `$RANDOM` | ✅ | ✅ | 兼容 |
| `shopt` | ✅ | ❌ | Zsh 用 `setopt` / `unsetopt` |
| `source` 命令 | ✅ | ✅ | 兼容 |
| `>&` 重定向 | ✅ | 写法不同 | Zsh 默认不需要 `&>`，应使用 `> file 2>&1` |
| `<<<` Here-String | ✅ | ✅ | 兼容 |
| `<()` 进程替换 | ✅ | ✅ | 兼容 |
| `${!var}` 间接引用 | ✅ | `${(P)var}` | 语法不同 |
| `declare -n` nameref | ✅ | ❌ | Zsh 无此特性 |
| `read -p 提示` | ✅ | `read '?提示'` 或 `-p` (zsh 5.x+) | 推荐 `print -n "提示"; read var` |
| `BASH_SOURCE` | ✅ | `$0` (脚本) / `${funcstack[1]}` (函数) | 不同变量名 |

**Zsh 脚本差异清单：**

```zsh
#!/bin/zsh
# ===== 1. 数组索引从 1 开始（Bash 从 0 开始） =====
arr=(one two three)
echo $arr[1]        # zsh: one  | bash: one
echo $arr[2]        # zsh: two  | bash: two
# 兼容写法：始终用 1-based 即可，或显式开启 KSH_ARRAYS

# ===== 2. 未加双引号的变量默认不进行单词分割 =====
files="*.txt"
echo $files          # zsh: *.txt（不展开）  | bash: 展开为文件列表
echo $=files         # zsh 强制分割
# 想获得 Bash 行为：setopt SH_WORD_SPLIT（不推荐）

# ===== 3. 通配符不匹配时报告错误而非原样保留 =====
ls *.nonexistent     # zsh: no matches found  | bash: ls: *.nonexistent: No such file
# 禁用：setopt NULL_GLOB（不匹配时移除）或 unsetopt NOMATCH

# ===== 4. 函数的参数访问 =====
function myfunc {
    echo $1 $2       # 与 Bash 兼容
}
# Zsh 特有：$* 和 $@ 在双引号内行为与 Bash 相同
# Zsh 支持 $functrace 和 $funcstack 用于调试

# ===== 5. 算术运算差异 =====
# Zsh 原生支持浮点数（Bash 不支持）
echo $(( 3.14 * 2 ))    # zsh: 6.28 | bash: 错误

# ===== 6. 循环读文件 =====
# 推荐 Zsh 原生方式（不需要 cat 的子进程）
while IFS= read -r line; do
    echo "$line"
done < file.txt         # 兼容写法
# Zsh 独有：< file.txt 在当前 Shell 执行无需子 Shell

# ===== 7. 别名在脚本中不展开 =====
# Bash: 默认不展开（需 shopt -s expand_aliases）
# Zsh:  默认不展开，行为相同
```

**如果你写的是给 `#!/bin/sh` 或 `#!/bin/bash` 的脚本，迁移到 `#!/bin/zsh` 时建议全面测试，或保持 Bash 脚本不变，仅将 Zsh 用于交互式环境。**

## 1.7 Zsh Glob 通配符增强

Zsh 的 glob 系统是它最强的特性之一，远超 Bash。

**开启增强通配符：**

```zsh
setopt EXTENDED_GLOB    # 启用 # ~ ^ 等增强语法
```

**完整速查表：**

```zsh
# ===== 递归匹配 =====
echo **/*.py                # 递归查找所有 .py 文件（Bash 5.0 前需 shopt -s globstar）

# ===== 文件类型限定符（Glob Qualifiers） =====
# 语法：pattern(qualifiers)
echo *(.)                   # 只匹配普通文件
echo *(/)                   # 只匹配目录
echo *(@)                   # 只匹配符号链接
echo *(*)                   # 只匹配可执行文件
echo *(F)                   # 只匹配非空文件
echo *(L0)                  # 只匹配空文件
echo *(Lm+10)               # 只匹配 >10MB 的文件
echo *(Lk-100)              # 只匹配 <100KB 的文件

# ===== 时间限定符 =====
echo *(m-7)                 # 7 天内修改的文件 (modification time)
echo *(a-1)                 # 1 天内访问的文件 (access time)
echo *(c+30)                # 30 天前状态变更的文件 (change time)
echo *(mh-3)                # 3 小时内修改的文件
echo *(ms-60)               # 60 秒内修改的文件

# ===== 排序 =====
echo *(oL)                  # 按大小升序 (L=length, o=ascending, O=descending)
echo *(OL)                  # 按大小降序
echo *(om)                  # 按修改时间升序
echo *(OM)                  # 按修改时间降序（最新的在前）
echo *(on)                  # 按名称升序
echo *(oN)                  # 不排序（快）
echo *(om[1,10])            # 最近修改的 10 个文件

# ===== 所有者限定 =====
echo *(u:root:)             # 属于 root 的文件
echo *(g:wheel:)            # 属于 wheel 组的文件
echo *(u:$(id -u):)        # 属于当前用户的文件

# ===== 权限限定 =====
echo *(f:644:)              # 权限为 644 的文件
echo *(f:u+x:)              # 用户有执行权的文件

# ===== 取反与组合 =====
echo *(^/)                  # 匹配非目录（注意要用 EXTENDED_GLOB 开启 ^）
echo *(.-*)                 # 普通文件 且 可执行
echo *(Lm+1Lm-10)          # 大于 1MB 且小于 10MB 的文件

# ===== 扩展通配模式 =====
echo **/*.(jpg|png|gif)(.Lm+5)  # 递归查找 >5MB 的图片
setopt EXTENDED_GLOB
echo *.(#i)readme           # 大小写不敏感 (#i) = insensitive
echo *.(#i)readme(#q.)      # 大小写不敏感 + 普通文件限定
```

# 2. Fish Shell

Fish（Friendly Interactive Shell）的设计哲学是**零配置、开箱即用**。它完全放弃了 POSIX 兼容性，因此提供了前所未有的现代化交互体验。

## 2.1 安装与开箱即用特性

```bash
# 安装
brew install fish          # macOS
sudo apt install fish      # Debian/Ubuntu
sudo dnf install fish      # Fedora
sudo pacman -S fish        # Arch

# 设置为默认 Shell（可选，推荐保留默认 Shell 仅试用）
chsh -s $(which fish)
# 更安全的方式：在 .bashrc / .zshrc 末尾加 exec fish
```

**开箱即用特性（无需任何插件）：**

| 特性 | 说明 |
|------|------|
| 语法高亮 | 输入命令时实时高亮（有效命令蓝色、无效红色、路径下划线） |
| 自动建议 | 灰色提示历史命令或补全（按 → 或 Ctrl+F 接受） |
| Tab 补全 | 菜单式补全，来自 man page 的描述信息 |
| Web 配置界面 | `fish_config` 在浏览器中修改颜色、提示符、别名等 |
| 脚本错误提示 | 友好的错误信息，而非晦涩的 "command not found" |
| 24 位真彩色 | 终端颜色支持 |
| 自动 `$PATH` 补全 | 无需 compinit 等初始化 |

**交互体验示例：**

```fish
# 在 ~/.config/fish/config.fish 中
# Fish 不用 export，用 set -x
set -x EDITOR nvim
set -x LANG zh_CN.UTF-8

# 别名（fish 用 abbr 实现"展开式别名"更佳）
abbr -a gco git checkout
abbr -a gst git status
abbr -a gd  git diff

# 或传统 alias
alias ll 'ls -lh'
alias la 'ls -la'

# 路径（不需要 export PATH=...:$PATH 的模板）
fish_add_path ~/.local/bin
fish_add_path /usr/local/go/bin
```

## 2.2 Fish 语法差异速查

Fish 的语法与 POSIX Shell 完全不同，以下是核心差异：

| 操作 | Bash/Zsh | Fish |
|------|----------|------|
| 变量赋值 | `var=value` | `set var value` |
| 环境变量 | `export VAR=value` | `set -x VAR value` |
| 局部变量 | `local var=value` | `set -l var value` |
| 全局变量 | 默认全局 | `set -g var value` |
| 通用变量 | 无 | `set -U var value`（跨会话持久化） |
| 命令替换 | `$(cmd)` | `(cmd)` 无 `$` |
| 算术 | `$((1+2))` | `math 1+2` |
| if 语句 | `if [[ ]]; then ... fi` | `if test ...; ...; end` 或 `if cmd; ...; end` |
| for 循环 | `for i in ...; do ... done` | `for i in ...; ...; end` |
| 函数 | `func() { ... }` | `function func; ...; end` |
| 退出状态 | `$?` | `$status` |
| 管道状态 | `${PIPESTATUS[@]}` | `$pipestatus` |
| 后台任务 | `cmd &` | `cmd &`（兼容） |
| 条件与或 | `cmd1 && cmd2` | `cmd1; and cmd2` |
| 条件或 | `cmd1 \|\| cmd2` | `cmd1; or cmd2` |
| 非 | `! cmd` | `not cmd` |

**Fish 函数定义：**

```fish
# 定义函数（存储在 ~/.config/fish/functions/ 下自动加载）
function mkcd
    set -l dir $argv[1]
    mkdir -p $dir
    cd $dir
end

# 保存为独立文件（推荐）
funcsave mkcd                # 将交互式定义的函数保存到文件

# 参数访问
# $argv      : 参数列表（数组）
# $argv[1]   : 第一个参数（1-indexed 与 Zsh 相同）
# $argv[2..-1] : 第 2 到最后一个参数
```

## 2.3 Web 配置界面

```bash
fish_config    # 启动 Web 配置界面，自动打开浏览器
```

Web 界面可配置：
- **Colors**：终端颜色主题（选择后立即生效）
- **Prompt**：提示符风格（多种预设可选）
- **Functions**：图形化管理函数
- **Variables**：查看/搜索/导出变量
- **Abbreviations**：管理展开式缩写
- **History**：查看命令历史
- **Bindings**：键盘绑定

## 2.4 为什么 Fish 不适合作为脚本语言

Fish 的设计目标纯粹是**交互式 Shell**，官方文档明确指出：

> Fish is not a scripting language. It's designed to be a great interactive command-line environment.

**具体原因：**

1. **不兼容 POSIX**：所有 `#!/bin/sh` 脚本在 Fish 中无法执行。`(cmd)`  vs `$(cmd)`、`set var val` vs `var=val` 语法完全不同。
2. **无 `source`**：Fish 用 `.` 相当于 `source`（但与 POSIX source 不兼容）。
3. **命令替换行为不同**：`(cmd)` 默认不拆分为行，需 `(cmd | string split0)` 处理。
4. **管道中变量作用域**：每个管道命令在子 Shell 中执行（bash `lastpipe` 可避免，Fish 无法）。
5. **缺乏企业采纳**：CI/CD 系统、Docker 镜像、云 init 脚本普遍依赖 `#!/bin/sh` 或 `#!/bin/bash`。
6. **生态系统孤立**：绝大多数 Shell 教程、Stack Overflow 答案、运维脚本为 Bash/POSIX 语法。

**最佳实践：**

```
场景                      推荐
─────────────────────────────────────────
交互式使用（个人终端）    Fish / Zsh
系统脚本（可移植）        #!/bin/sh
项目构建脚本              #!/bin/bash
CI/CD 脚本                #!/bin/bash
Dockerfile                SHELL /bin/bash 或 /bin/sh
一次性数据处理            Fish 可行（很方便）
```

# 3. Dash / Ash

## 3.1 背景与定位

Dash（Debian Almquist Shell）和 Ash（Almquist Shell）是轻量级 POSIX Shell 实现。

| Shell | 来源 | 二进制大小 | 定位 |
|-------|------|-----------|------|
| Dash | Debian 从 NetBSD ash 分支 | ~120KB | Debian/Ubuntu `/bin/sh` 默认实现 |
| Ash | NetBSD 原始版本 | ~100KB | 嵌入式 Linux（BusyBox 内置 ash） |
| Bash | GNU 项目 | ~1.2MB | 功能丰富的交互式/脚本 Shell |
| BusyBox ash | BusyBox 内嵌 | ~1MB (整个 BusyBox) | 嵌入式、initramfs、Docker 精简镜像 |

**Ubuntu/Debian `/bin/sh` 变迁：**

```
Ubuntu 6.06 之前: /bin/sh → bash
Ubuntu 6.10 (2006) 起: /bin/sh → dash
原因：启动速度快（dash 远小于 bash）、POSIX 严格、资源占用少
```

**验证当前系统的 `/bin/sh`：**

```bash
ls -la /bin/sh             # 查看符号链接指向
readlink -f /bin/sh        # 解析真实路径

# 测试脚本执行速度差异
time dash -c 'for i in $(seq 1 1000); do :; done'
time bash -c 'for i in $(seq 1 1000); do :; done'
```

## 3.2 POSIX 严格模式

Dash 对 POSIX 的遵循比 Bash 更严格，在 Bash 下"碰巧能运行"的脚本可能在 Dash 下失败。

**Dash 与 Bash 差异清单：**

| 特性 | Bash | Dash | 说明 |
|------|------|------|------|
| `[[ ]]` 条件测试 | ✅ | ❌ | Dash 只支持 `[ ]` 和 `test` |
| `==` 字符串比较 | ✅ (在 `[[ ]]` 内) | ❌ | `[ ]` 内只能用 `=` |
| 数组 `arr=(a b c)` | ✅ | ❌ | Dash 无数组（用 `$@` 位置参数替代） |
| `declare -A` 关联数组 | ✅ | ❌ | 不支持 |
| `<<<` Here-String | ✅ | ❌ | 用 `echo "str" \| cmd` 替代 |
| `<()` 进程替换 | ✅ | ❌ | 用命名管道或临时文件替代 |
| `>&` 合并重定向 | ✅ (如 `&>file`) | ❌ | 用 `>file 2>&1` |
| `source` 命令 | ✅ | ❌ | 用 `.` 替代 |
| `shopt` | ✅ | ❌ | 不支持 |
| `function` 关键字 | ✅ | ❌ | 只用 `funcname() { }` 语法 |
| Substring `${var:0:3}` | ✅ | ❌ | 用 `$(echo "$var" \| cut -c1-3)` 或 awk |
| `${var/pat/repl}` | ✅ | ❌ | 用 sed/awk 替代 |
| `${!var}` 间接引用 | ✅ | ❌ | 用 `eval` 替代（注意安全） |
| `trap ERR` | ✅ | ❌ | 不支持 |
| `$RANDOM` | ✅ | ❌ | 用 `/dev/urandom` 或 awk `srand()` |
| `read -p` | ✅ | ❌ | 用 `printf` + `read`（无 -p 选项） |
| `read -n` | ✅ | ❌ | 用 `dd` 或 `stty` 替代 |
| `echo -n` | 可能 OK | 可能不 OK | 用 `printf` 替代 `echo -n`（POSIX 不保证 echo -n） |
| `local` | ✅ | ✅ (某些版本) | POSIX 无 local，用子 Shell 或 `_` 前缀 |
| Pipefail | `set -o pipefail` | ❌ | 手动检查 `${PIPESTATUS[@]}`（dash 无此变量） |

**常见 Dash 兼容性问题及修复：**

```bash
#!/bin/sh
# ===== 问题 1: [[ ]] 不被支持 =====
# 错误写法（bashism）
if [[ "$name" == "admin" ]]; then
    echo "Welcome admin"
fi

# 正确写法（POSIX）
if [ "$name" = "admin" ]; then
    echo "Welcome admin"
fi

# ===== 问题 2: == 在 [ ] 内 =====
# 错误写法
[ "$a" == "$b" ]          # Dash 下可能不工作

# 正确写法
[ "$a" = "$b" ]

# ===== 问题 3: function 关键字 =====
# 错误写法
function greet {
    echo "Hello $1"
}

# 正确写法
greet() {
    echo "Hello $1"
}

# ===== 问题 4: Here-String <<< =====
# 错误写法
grep "error" <<< "$log_line"

# 正确写法
echo "$log_line" | grep "error"

# ===== 问题 5: source =====
# 错误写法
source ./config.sh

# 正确写法
. ./config.sh     # 注意 . 后必须有空格

# ===== 问题 6: echo -n =====
# 错误写法
echo -n "Processing..."

# 正确写法
printf "Processing..."
```

**检测 Bashism 的工具：**

```bash
# checkbashisms（Debian 的 devscripts 包）
sudo apt install devscripts
checkbashisms script.sh

# shellcheck 也可检测 Bashism
shellcheck -s sh script.sh    # 指定 shell 为 POSIX sh

# 用 dash 直接运行检查
dash -n script.sh    # 只检查语法
dash script.sh       # 实际运行（需要完整的测试用例）
```

## 3.3 性能对比

```bash
# Dash 在系统启动脚本和循环密集的场景下有显著优势
# 以下是参考数据（相对值）：

操作                    bash    dash    倍率
───────────────────────────────────────────
启动时间（空脚本）      0.8ms   0.3ms   2.6x
100 次字符串比较        5ms     2ms     2.5x
1000 次简单循环         30ms    12ms    2.5x
source 加载脚本          包含大量初始化  极简初始化
内存占用                 ~3MB    ~0.5MB  6x
```

**结论：** 如果目标是最大可移植性（如 init 脚本、Docker entrypoint、CI 脚本），始终以 `#!/bin/sh` 并遵循 POSIX 标准编写。重要项目应 CI 中同时用 `bash` 和 `dash` 测试。

# 4. Nushell

Nushell 是一个用 Rust 编写的现代化 Shell，核心创新是**将管道传递的数据结构化为类型化数据**（表格、记录、列表），而非纯文本流。

## 4.1 安装与核心概念

```bash
# 安装
brew install nushell          # macOS
winget install nushell        # Windows
cargo install nu              # 通过 Rust
sudo apt install nushell      # Debian/Ubuntu (较新版本)

# 启动
nu
```

**核心哲学："Everything is Structured Data"**

```nu
# 传统 Shell：纯文本解析
$ ps aux | grep node | awk '{print $2, $11}'
# 依赖空格分割，字段偏移可能因系统而异，脚本脆弱

# Nushell：结构化数据
> ps | where name =~ "node" | select pid name
╭──────┬──────────────────╮
│ pid  │ name             │
├──────┼──────────────────┤
│ 1234 │ node server.js   │
│ 5678 │ node worker.js   │
╰──────┴──────────────────╯
```

## 4.2 核心数据类型

| 类型 | 示例 | 说明 |
|------|------|------|
| `int` | `42` | 整数 |
| `float` | `3.14` | 浮点数 |
| `string` | `"hello"` | 字符串 |
| `bool` | `true` / `false` | 布尔值 |
| `list` | `[1, 2, 3]` | 有序列表 |
| `record` | `{name: "Alice", age: 30}` | 键值对 |
| `table` | 内置命令输出 | 行列表，每行是一个 record |
| `filesize` | `1.5MB` | 带单位文件大小 |
| `duration` | `2min 30sec` | 时间段 |
| `date` | `2024-01-15` | 日期时间 |
| `nothing` | `null` | 空值（类似 null） |
| `closure` | `{\|x\| $x + 1}` | 闭包（匿名函数） |

## 4.3 核心命令与管道

Nushell 命令命名风格为：动词-名词（与 PowerShell 类似），支持缩写（`ls`、`cd` 等保留）。

```nu
# ===== 基本命令 =====
ls                                # 列出当前目录（输出表格）
ls | where size > 10mb            # 过滤 >10MB 的文件
ls | sort-by size --reverse       # 按大小逆序排序
ls | where type == dir            # 只列目录
ls **/*.rs                        # 递归 glob（与 Bash globstar 不同，nu 原生支持）
ls | get name                     # 提取 name 列（返回列表）
ls | select name size             # 选取多列
ls | reject modified              # 移除指定列
ls | first 5                      # 取前 5 行
ls | last 3                       # 取后 3 行

# ===== 数据处理管道 =====
# HTTP 请求 + JSON 解析 + 表格化
> http get https://api.github.com/repos/nushell/nushell
| get stargazers_count
29854

# 处理 CSV
> open data.csv
| where age > 18
| select name email
| sort-by name
| to csv                  # 输出回 CSV 格式

# 处理 JSON
> open package.json
| get dependencies
| transpose key value       # 将 record 转为 key-value 表格
| sort-by key

# 系统信息
> sys | get host
╭──────────────┬──────────────────────╮
│ name         │ my-machine           │
│ os_version   │ 22.04                │
│ kernel       │ 5.15.0               │
╰──────────────┴──────────────────────╯

> sys | get cpu | select name brand
```

## 4.4 与传统 Shell 的对比

| 维度 | Bash/Zsh | Nushell |
|------|----------|---------|
| 数据类型 | 纯文本 | 结构化类型（string、int、table、record） |
| 管道内容 | 文本流 | 类型化数据流 |
| 解析方式 | 正则/grep/awk 人工解析 | 字段化直接访问 |
| 错误处理 | `$?` 手动检查 | 原生 Error 类型，`try/catch` |
| 脚本编写 | 命令式为主 | 函数式管道（`each`、`where`、`reduce`） |
| JSON/CSV/YAML | 需要 jq/yq/mlr 等外部工具 | 内置 `open` / `to json` / `from csv` |
| 命令发现 | man / --help | 内置 `help commands`（结构化文档） |
| 并行处理 | `xargs -P` / GNU parallel | `par-each` 原生并行 |
| 跨平台 | Linux/macOS（WSL） | Windows/Linux/macOS 原生 |
| 包管理 | 无 | 内置插件系统，`register` 命令 |

**Nushell 适合的场景：**

- JSON/CSV/YAML 等结构化数据的大量处理
- 跨平台脚本（Windows + Linux）
- API 数据查询与转换
- 数据探索和交互式分析

**Nushell 不适合的场景：**

- 传统运维脚本（生态不兼容）
- 需要 POSIX 兼容的场景
- 极度依赖传统 Unix 工具链的任务
- 作为系统 Shell（还在早期阶段，变化较快）

## 4.5 脚本编写

```nu
#!/usr/bin/env nu
# Nushell 脚本

# 函数定义
def greet [name: string] {
    echo $"Hello, ($name)!"
}

# 带类型的函数
def "sum columns" [file: string, col: string] {
    open $file
    | from csv
    | get $col
    | math sum
}

# 错误处理
try {
    http get https://httpbin.org/status/404
} catch { |e|
    echo $"Request failed: ($e.msg)"
}

# 环境变量
$env.EDITOR = "nvim"
load-env { OPENAI_API_KEY: "sk-xxx" }

# 模块系统
module utils {
    export def my-command [] {
        echo "from module"
    }
}
use utils.nu
utils my-command
```

# 5. Oil Shell / Oils

Oil Shell（项目更名为 Oils）的目标是**创造一个兼容 Bash 但又修复了 Bash 历史缺陷的新型 Shell 语言**。项目包含两个主要组件：

- **OSH**（Oil Shell）：与 Bash 高度兼容的运行时，可直接运行大多数 Bash 脚本
- **YSH**（Oil Language）：全新的 Shell 语言，修复了 Bash 的痛苦设计

## 5.1 安装与运行

```bash
# 安装（下载预编译二进制）
# https://github.com/oils-for-unix/oils/releases
wget https://github.com/oils-for-unix/oils/releases/download/0.24.0/oils-for-unix-0.24.0-linux-x86_64.tar.gz
tar xzf oils-for-unix-0.24.0-linux-x86_64.tar.gz
sudo cp oils-for-unix-0.24.0/osh /usr/local/bin/
sudo cp oils-for-unix-0.24.0/ysh /usr/local/bin/

# 或通过包管理
sudo apt install oils-for-unix   # Debian/Ubuntu
brew install oils-for-unix       # macOS

# 验证
osh --version
ysh --version
```

**OSH 兼容模式：运行现有 Bash 脚本**

```bash
# osh 可以直接运行大多数 Bash 脚本
osh existing_script.sh

# 开启严格模式检测潜在问题
osh --strict existing_script.sh

# --strict 会启用：
#   - strict_errexit: 更严格的 -e 模式
#   - strict_nameref: 引用未定义变量报错
#   - strict_word_eval: 更安全的词法求值
```

## 5.2 YSH 语言特性

YSH 是从零设计的新型 Shell 语言，解决 Bash 的核心痛点：

```ysh
#!/usr/bin/env ysh

# ===== 1. 变量声明（无需 $ 前缀） =====
var name = "world"
var count = 42
var items = ["a", "b", "c"]       # 真正的列表（不是字符串分割的伪数组）
var user = {name: "Alice", age: 30}  # 真正的字典

# 引用变量用 $ 前缀
echo "Hello, $name"

# ===== 2. 表达式语法（无需 (( ))） =====
var total = 10 + 20                # 自然算术表达式
var is_admin = user.name == "admin"  # 自然布尔表达式

# ===== 3. 真正的 JSON 处理 =====
json write (user)                   # 将记录输出为 JSON

# ===== 4. 命令执行与捕获 =====
var output = $(ls -la | wc -l)     # $( ) 捕获命令输出
# 等价于传统 Bash: output=$(ls -la | wc -l)

# ===== 5. 真正的函数（带参数名和类型注解） =====
proc greet(name; greeting = "Hello") {
    echo "$greeting, $name"
}

greet "Alice"
greet "Bob" --greeting "Hi"

# ===== 6. 真正的循环（处理列表） =====
for item in (items) {              # 安全迭代列表元素
    echo $item
}

# ===== 7. 模式匹配 =====
# 不再需要 case 的 ;; 痛苦语法
match (name) {
    "admin" { echo "Administrator" }
    "guest" { echo "Guest user" }
    *       { echo "Other user" }
}

# ===== 8. 错误处理 =====
try {
    cp /tmp/source /tmp/dest
}
if (_status !== 0) {
    echo "Copy failed"
}
```

## 5.3 Oils 修复的 Bash 历史问题

| Bash 问题 | Oils 解决方案 |
|-----------|--------------|
| 未引用的变量默认做单词分割 + 通配符展开 | `strict_word_eval`：未引用变量不分割、不展开 glob |
| `$@` 和 `$*` 的行为难以理解 | 用 Python/JSON 风格的列表 `var items = ["a", "b"]` |
| 数组实际是字符串（空格分割） | 真正的列表类型，包含空格的项目无需引号处理 |
| `[ ]` vs `[[ ]]` 双语法混乱 | 统一用自然布尔表达式语法 |
| 错误处理依赖 `$?` 和 `set -e` | `try`/`catch` 和 `_status` 变量 |
| `echo $var` 可能解析转义序列 | `write` 命令安全输出 |
| 算术必须 `$(( ))` | `var x = a + b` 自然表达式语法的子集 |
| JSON 解析需要 jq 外部工具 | 内置 `json read` / `json write` |
| subshell 创建不必要的子进程 | 在可能的情况下内联执行 |
| 变量的引用方式不统一 | `$var` 引用 / `var = value` 定义，统一语法 |

## 5.4 使用场景

```
场景                        推荐
─────────────────────────────────────────
运行现有 Bash 脚本          osh script.sh
检测 Bash 脚本中的问题      osh --strict script.sh
新编写的 Shell 脚本         ysh
CI 检查脚本质量             osh --strict + shellcheck
渐进式迁移                  Bash → OSH → YSH（逐步升级）
```

# 6. ShellCheck 深度使用

ShellCheck 是 Shell 脚本的静态分析工具，类似于其他语言的 Linter。它能检测常见错误、不推荐写法、可移植性问题和潜在安全风险。Haskell 编写。

## 6.1 安装与命令行使用

```bash
# 安装
brew install shellcheck         # macOS
sudo apt install shellcheck     # Debian/Ubuntu
sudo dnf install ShellCheck     # Fedora
scoop install shellcheck        # Windows (Scoop)
winget install shellcheck       # Windows (winget)

# 也可通过 Docker
docker run --rm -v "$PWD:/mnt" koalaman/shellcheck:stable script.sh

# 验证安装
shellcheck --version
```

**基本使用：**

```bash
# 单文件检查
shellcheck script.sh

# 递归检查目录中所有 shell 脚本
shellcheck **/*.sh

# 指定 Shell 方言（影响检查规则）
shellcheck --shell=sh script.sh    # POSIX sh 模式
shellcheck --shell=bash script.sh  # Bash 模式（默认）
shellcheck --shell=dash script.sh  # Dash 模式
shellcheck --shell=ksh script.sh   # Korn Shell 模式

# 输出格式
shellcheck --format=json script.sh     # JSON 输出（编辑器/CI 解析用）
shellcheck --format=gcc script.sh      # GCC 兼容格式（IDE 跳转用）
shellcheck --format=tty script.sh      # 终端带颜色输出（默认）
shellcheck --format=diff script.sh     # Diff 格式
shellcheck --format=checkstyle script.sh  # Checkstyle XML 格式（Jenkins）

# 严格度控制
shellcheck --severity=warning script.sh  # 只显示 warning 及以上（默认 style）
shellcheck --severity=error script.sh    # 只显示 error
```

## 6.2 常见规则编号与含义详解

ShellCheck 规则以 `SC` 为前缀，四位数字。以下是高频规则分类详解：

**一、引用问题（最常见）：**

| 编号 | 严重度 | 检测内容 | 说明 |
|------|--------|----------|------|
| SC2086 | warning | `$var` 未加双引号 | 会导致单词分割和 glob 展开。**这是 ShellCheck 最常见的警告** |
| SC2046 | warning | `$(cmd)` 未加双引号 | 命令替换的输出会被分割，应写 `"$(cmd)"` |
| SC2068 | error | 数组未正确加双引号 | `"$@"` 正确，`"$*"` 和 `$@`（未引号）不正确 |
| SC2053 | warning | 引号嵌套错误 | 内层引号与外层冲突 |

```bash
# SC2086 示例
# 错误写法
file="my document.txt"
rm $file               # SC2086: 空格导致误删多个文件

# 正确写法
rm "$file"

# SC2046 示例
# 错误写法
for f in $(ls *.txt); do ... done    # SC2045/SC2046
# 正确写法
for f in *.txt; do ... done          # 直接用 glob
for f in ./*.txt; do ... done        # 处理以 - 开头的文件名
```

**二、条件测试问题：**

| 编号 | 严重度 | 检测内容 | 说明 |
|------|--------|----------|------|
| SC2166 | warning | `[ $a == $b ]` 而非 `[ $a = $b ]` | POSIX 单等号 |
| SC2181 | style | 直接用 `[ $? -eq 0 ]` | 应用 `if cmd; then ...` |
| SC2015 | info | `A && B \|\| C` 非三元运算符 | 不是 if-then-else 的可靠替代 |
| SC2154 | warning | 引用了未赋值的变量 | 可能是拼写错误 |
| SC2034 | warning | 变量声明了但未使用 | 可能是拼写错误 |
| SC2143 | style | `if echo "$str" \| grep -q ...` | 可用 `[[ $str =~ ... ]]` 替代 |
| SC2120 | warning | 函数引用了未传递的参数 | 函数期望参数但调用者未传 |

```bash
# SC2181 示例
# 错误写法
grep pattern file
if [ $? -eq 0 ]; then
    echo "found"
fi

# 正确写法
if grep pattern file; then
    echo "found"
fi

# SC2015 示例
# 错误写法（这不是三目运算符！）
[[ -f file ]] && echo "exists" || echo "missing"
# 如果 echo "exists" 失败，会继续执行 echo "missing"

# 正确写法
if [[ -f file ]]; then
    echo "exists"
else
    echo "missing"
fi
```

**三、常见错误模式：**

| 编号 | 严重度 | 检测内容 | 说明 |
|------|--------|----------|------|
| SC1009 | error | 未识别的语法错误 | 通用语法分析失败 |
| SC1073 | error | `if` 缺少 `then` | 或 `elif`/`else` 后的 `then` 缺失 |
| SC1091 | warning | `source`/`.` 的文件找不到 | 源文件路径问题，静态分析可能误报 |
| SC1113 | warning | `#!` shebang 行以空格开头 | shebang 必须在行首 |
| SC2002 | style | 无用的 `cat` (UUOC) | `cat file \| cmd` 可写为 `cmd < file` |
| SC2016 | info | 单引号内的 `$var` | 变量不会展开（可能是预期行为也可能不是） |
| SC2028 | warning | `echo` 中的 `\t` 等转义序列 | `echo` 默认不解析（用 `printf` 或 `echo -e`） |
| SC2164 | warning | `cd` 失败后继续执行 | 应 `cd dir \|\| exit` |
| SC2162 | warning | `read` 未带 `-r` | `read` 默认会解释反斜杠转义 |
| SC2206 | warning | 对带有空格的变量进行了单词分割 | 引号、数组 |
| SC2207 | warning | `arr=($var)` 单词分割 | 应使用 `mapfile` 或 `readarray` |
| SC210 | style | `which` 非 POSIX 标准 | 用 `command -v` 替代 |

```bash
# SC2164 示例
# 错误写法
cd /some/dir
rm -rf *          # cd 失败会在当前目录执行！灾难性后果

# 正确写法
cd /some/dir || exit
rm -rf *

# SC2002 示例（UUOC）
# 错误写法
cat file.txt | grep pattern

# 正确写法
grep pattern file.txt
# 或（如果必须要重定向）
grep pattern < file.txt

# SC2162 示例
# 错误写法
read line                     # 反斜杠会被转义

# 正确写法
IFS= read -r line             # -r 禁用反斜杠转义
```

**四、安全与健壮性：**

| 编号 | 严重度 | 检测内容 | 说明 |
|------|--------|----------|------|
| SC2094 | info | 同文件读写竞争 | `cat file \| grep ... > file` |
| SC2119 | warning | 函数用 `$@` 但未传参 | 可能意图使用函数的参数 |
| SC2250 | style | `${var/}` 无替换目标 | 可能是拼写错误 |
| SC2295 | warning | `${var%%pat}` 在 `[ ]` 中未加引号 | glob 展开风险 |

## 6.3 IDE 集成

**VS Code：**

```bash
# 安装 ShellCheck 扩展
code --install-extension timonwong.shellcheck

# 配置 settings.json
{
    "shellcheck.customArgs": ["--shell=bash", "--severity=warning"],
    "shellcheck.enableQuickFix": true,
    "shellcheck.run": "onType",       // "onType" | "onSave"
    "shellcheck.ignorePatterns": {
        "**/vendor/**": true,
        "**/generated/**": true
    }
}
```

**Vim/Neovim：**

```vim
" 使用 ALE（异步 Lint Engine）
Plug 'dense-analysis/ale'
let g:ale_linters = { 'sh': ['shellcheck'] }
let g:ale_sh_shellcheck_options = '-s bash'

" 或使用 null-ls (Neovim)
" null-ls 自动检测 shellcheck
```

**JetBrains (IDEA/GoLand/...)：**
- 内置 Shell Script 插件自动集成 ShellCheck（无需额外安装）
- Settings → Tools → ShellCheck → Enable

## 6.4 CI 集成

**GitHub Actions：**

```yaml
# .github/workflows/shellcheck.yml
name: ShellCheck
on: [push, pull_request]

jobs:
  shellcheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run ShellCheck
        uses: ludeeus/action-shellcheck@master
        with:
          severity: warning
          scandir: './scripts'
          format: tty
```

**GitLab CI：**

```yaml
# .gitlab-ci.yml
shellcheck:
  image: koalaman/shellcheck-alpine:stable
  stage: test
  script:
    - shellcheck --severity=warning scripts/*.sh
  allow_failure: false
```

**Pre-commit Hook：**

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/koalaman/shellcheck-precommit
    rev: v0.10.0
    hooks:
      - id: shellcheck
        args: ["--severity=warning"]
```

**Jenkins Pipeline：**

```groovy
stage('ShellCheck') {
    steps {
        sh '''
            docker run --rm -v "$PWD:/mnt" koalaman/shellcheck:stable \
                --format=checkstyle scripts/*.sh > shellcheck.xml
        '''
    }
}
```

## 6.5 排除规则

```bash
# 方法 1: 命令行排除
shellcheck --exclude=SC2034,SC1091 script.sh

# 方法 2: 注释形式排除（行级）
# shellcheck disable=SC2034     # 排除下一行
unused_variable="this is intentional"

# shellcheck disable=SC2086     # 排除同一行
echo $special_var  # 我知道这里不需要引号

# 方法 3: 注释形式排除（代码块级）
# shellcheck disable=SC1091,SC1090
source ./some_dynamic_path.sh
# shellcheck disable=SC1090
# shellcheck source=./lib/lib.sh
source ./lib/lib.sh

# 方法 4: .shellcheckrc 配置文件（项目根目录）
# .shellcheckrc
exclude=SC1091,SC2034
shell=bash
severity=warning
# 也可以在文件头声明
# shellcheck shell=bash
```

# 7. shfmt 代码格式化

shfmt 是用 Go 编写的 Shell 脚本格式化工具，提供统一代码风格。支持 Bash、POSIX sh 和 mksh。

## 7.1 安装

```bash
# 安装
brew install shfmt               # macOS
sudo apt install shfmt           # Debian/Ubuntu
sudo dnf install shfmt           # Fedora
go install mvdan.cc/sh/v3/cmd/shfmt@latest  # Go 方式

# 验证
shfmt --version
```

## 7.2 基本用法与选项

```bash
# ===== 基本使用 =====

# 检查不写入（类似 lint 的 dry-run）
shfmt -d script.sh          # 显示 diff（不修改文件）
shfmt -l script.sh          # 只列出需要格式化的文件名

# 格式化写入
shfmt -w script.sh          # 原地格式化（write）
shfmt -w *.sh               # 批量格式化

# ===== 缩进选项 =====
shfmt -i 2 script.sh        # 缩进 2 空格（默认 tab）
shfmt -i 4 script.sh        # 缩进 4 空格
shfmt -ci script.sh         # case 语句中缩进（case-indent）

# ===== 空格选项 =====
shfmt -bn script.sh         # 二元运算符两边留空格（binary-next-line）
shfmt -sr script.sh         # 重定向操作符前后留空格（space-redirect）

# ===== 扩展控制 =====
shfmt -s script.sh          # 简化表达式（如 $((x + 0)) → $((x)))
shfmt -mn script.sh         # 最小化（minify），只保留最少空白
shfmt -ln bash script.sh    # 指定方言为 bash（默认 auto 自动检测）

# ===== 函数定义风格 =====
shfmt -fn script.sh         # 函数开头花括号另起一行
shfmt -p script.sh          # 保留原来的缩进（不重新计算）

# ===== 常用组合 =====
# 典型 CI 检查设置
shfmt -d -i 2 -ci -bn -sr scripts/*.sh
```

**风格选项对照表：**

| 选项 | 效果 | 默认 |
|------|------|------|
| `-i 0` | 使用 Tab 缩进（默认） | `-i 0` |
| `-i 2` | 使用 2 空格缩进 | |
| `-ci` | `case` 分支缩进 | 默认不缩进 case 分支 |
| `-bn` | 二元运算符前后换行缩进（管道和逻辑运算符） | 默认在操作符前换行 |
| `-sr` | 重定向 `>` `<` 两边加空格 | 默认紧凑 |
| `-fn` | `function foo()`→`function foo()\n{` | 默认 `{` 跟在 `()` 同行 |
| `-kp` | 保留开括号后的空行 | 默认压缩空行 |
| `-ln` | 语言方言（bash/posix/mksh） | 自动检测 |

**格式化前后对比：**

```bash
# 格式化前
if [[ -f /tmp/test ]];then
    echo "found";fi
for i in 1 2 3;do
echo $i
done
case $x in
a)echo "A";;
b)echo "B";;
esac

# shfmt -w -i 2 -ci -bn -sr script.sh 后：
if [[ -f /tmp/test ]]; then
    echo "found"
fi
for i in 1 2 3; do
    echo $i
done
case $x in
    a) echo "A" ;;
    b) echo "B" ;;
esac
```

## 7.3 编辑器集成

**VS Code：**

```json
// settings.json
{
    "[shellscript]": {
        "editor.defaultFormatter": "foxundermoon.shell-format",
        "editor.formatOnSave": true,
        "shellformat.flag": "-i 2 -ci -bn -sr"
    }
}
```

**Vim/Neovim：**

```vim
" 安装 shfmt 后用 ALE 或 formatter.nvim 集成
" Neovim 使用 conform.nvim
require("conform").setup({
    formatters_by_ft = {
        sh = { "shfmt" },
        bash = { "shfmt" },
        zsh = { "shfmt" },
    },
    formatters = {
        shfmt = {
            prepend_args = { "-i", "2", "-ci", "-bn", "-sr" },
        },
    },
})
```

**Pre-commit 集成：**

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/scop/pre-commit-shfmt
    rev: v3.7.0
    hooks:
      - id: shfmt
        args: ["-w", "-i", "2", "-ci", "-bn", "-sr"]
```

# 8. Bats 测试框架

Bats（Bash Automated Testing System）是 Bash 原生测试框架，所有测试逻辑都是合法的 Bash 语法。

## 8.1 安装

```bash
# 安装 Bats-core（推荐，Bats 的活跃 fork）
brew install bats-core          # macOS
sudo apt install bats           # Debian/Ubuntu
sudo dnf install bats           # Fedora
npm install -g bats             # npm

# 安装辅助库
# bats-support: 提供错误输出工具
# bats-assert: 提供断言库
# bats-file: 提供文件和目录断言

# 作为 Git submodule 安装（推荐，CI 可重现）
git submodule add https://github.com/bats-core/bats-core.git test/bats
git submodule add https://github.com/bats-core/bats-support.git test/test_helper/bats-support
git submodule add https://github.com/bats-core/bats-assert.git test/test_helper/bats-assert
git submodule add https://github.com/bats-core/bats-file.git test/test_helper/bats-file

# 验证
bats --version
```

## 8.2 基本语法

**最小测试文件：**

```bash
#!/usr/bin/env bats

# ===== 测试函数 =====
# 以 @test "描述" 开头，{ 必须在同一行或下一行

@test "测试加法运算" {
    result=$((1 + 1))
    [ "$result" -eq 2 ]
}

@test "测试字符串比较" {
    name="hello"
    [ "$name" == "hello" ]
}

@test "测试命令执行成功" {
    run ls /tmp
    [ "$status" -eq 0 ]
}

@test "测试命令输出" {
    run echo "Hello World"
    [ "$output" == "Hello World" ]
    [ "${lines[0]}" == "Hello World" ]     # 第一行
}

# ===== 辅助函数 =====
setup() {
    # 每个测试前执行
    export TEST_DIR="$(mktemp -d)"
    cd "$TEST_DIR" || exit
}

teardown() {
    # 每个测试后执行
    rm -rf "$TEST_DIR"
}
```

**Bats 执行模型：**

```
对每个 @test 测试用例：
  1. 执行 setup_file()（如果定义，文件级，仅执行一次）
  2. 执行 setup()（测试级，每个 @test 前执行）
  3. 执行 @test 测试体
  4. 执行 teardown()（测试级，每个 @test 后执行）
  5. （所有测试完成后）执行 teardown_file()（文件级）

注意：每个 @test 都在独立的子 Shell 中运行
      这意味着一个测试中的变量修改不会影响其他测试
      setup_file / teardown_file 在同一个 Shell 中执行
```

**运行测试：**

```bash
# 运行单个测试文件
bats test_script.bats

# 运行目录下所有测试
bats tests/

# 递归运行
bats -r tests/

# Filter: 只运行匹配名称的测试
bats -f "加法" tests/           # --filter

# 详细输出
bats --verbose-run tests/       # 打印每条命令的执行

# 输出格式
bats --formatter junit tests/   # JUnit XML 格式（CI 集成）
bats --formatter tap tests/     # TAP 格式
bats --formatter pretty tests/  # 默认美观输出

# 并行运行（v1.7.0+）
bats --jobs 4 tests/            # 4 个并行任务
```

## 8.3 setup / teardown 详解

```bash
#!/usr/bin/env bats

# -- 文件级初始化（所有测试前只执行一次） --
setup_file() {
    echo ">>> 文件级 setup" >&2
    export SHARED_CONFIG="$(mktemp -d)/config.json"
    echo '{"env": "test"}' > "$SHARED_CONFIG"
}

teardown_file() {
    echo ">>> 文件级 teardown" >&2
    rm -rf "$(dirname "$SHARED_CONFIG")"
}

# -- 测试级初始化（每个 @test 前执行） --
setup() {
    load 'test_helper/bats-support/load'
    load 'test_helper/bats-assert/load'
    load 'test_helper/bats-file/load'

    # 创建临时工作目录
    TEST_TEMP_DIR="$(mktemp -d)"
    cp "$SHARED_CONFIG" "$TEST_TEMP_DIR/"
    cd "$TEST_TEMP_DIR"

    # 设置测试环境变量
    export APP_ENV=test
    export DEBUG=false
}

teardown() {
    # 清理临时目录
    rm -rf "$TEST_TEMP_DIR"
}
```

## 8.4 断言

**Bats 原生断言（基于 test / `[ ]`）：**

```bash
@test "原生断言" {
    # 数值比较
    [ 1 -eq 1 ]         # 等于
    [ 2 -ne 3 ]         # 不等于
    [ 2 -gt 1 ]         # 大于
    [ 1 -lt 2 ]         # 小于
    [ 2 -ge 2 ]         # 大于等于
    [ 1 -le 2 ]         # 小于等于

    # 字符串比较
    [ "a" = "a" ]       # 相等
    [ "a" != "b" ]      # 不等
    [ -z "" ]           # 长度为零
    [ -n "hello" ]      # 长度非零

    # 文件测试
    [ -e "/tmp" ]       # 存在
    [ -f "/etc/hosts" ] # 是普通文件
    [ -d "/tmp" ]       # 是目录
    [ -x "/bin/ls" ]    # 是可执行
    [ -s "/etc/hosts" ] # 非空文件
}
```

**bats-assert 扩展断言（推荐）：**

```bash
#!/usr/bin/env bats

setup() {
    load 'test_helper/bats-support/load'
    load 'test_helper/bats-assert/load'
}

@test "assert 断言示例" {
    # 相等断言（推荐，输出更友好的错误信息）
    run echo "hello world"
    assert_success
    assert_output "hello world"

    # 部分匹配
    run echo "Error: file not found at /tmp/test"
    assert_failure
    assert_output --partial "file not found"

    # 正则匹配
    run echo "123-456-7890"
    assert_output --regexp '^[0-9]{3}-[0-9]{3}-[0-9]{4}$'

    # 行匹配
    run printf "line1\nline2\nline3\n"
    assert_line --index 0 "line1"    # 第 0 行
    assert_line --index 1 --partial "in"  # 第 1 行部分匹配
    assert_line -n 2 "line3"         # 第 2 行（n 是 1-indexed）

    # 输出计数
    assert_equal "${#lines[@]}" 3
}
```

**bats-file 扩展断言：**

```bash
#!/usr/bin/env bats

setup() {
    load 'test_helper/bats-file/load'
}

@test "文件断言" {
    mkdir -p /tmp/bats-test
    echo "data" > /tmp/bats-test/file.txt
    ln -s /tmp/bats-test/file.txt /tmp/bats-test/link.ln
    chmod 755 /tmp/bats-test/file.txt

    # 文件存在性
    assert_exist /tmp/bats-test/file.txt
    assert_not_exist /tmp/bats-test/nonexistent

    # 文件类型
    assert_file_executable /tmp/bats-test/file.txt
    assert_file_not_executable /tmp/bats-test/file.txt   # 配合 chmod 644
    assert_symlink_to /tmp/bats-test/file.txt /tmp/bats-test/link.ln

    # 文件内容
    assert_files_equal /tmp/bats-test/file.txt expected.txt
}
```

## 8.5 Mock 与 Stub

Bats 本身不提供 mock 框架，但可以通过以下技术实现：

```bash
#!/usr/bin/env bats

# ===== 方法 1: 函数覆盖（函数 Mock） =====
@test "Mock: 覆盖内置命令" {
    # 覆盖 curl 函数
    curl() {
        echo '{"status": "ok"}'
    }
    # 被测试的脚本需要 source 进来或定义函数
    run my_function_that_calls_curl
    assert_output '{"status": "ok"}'
}

# ===== 方法 2: PATH 优先 Mock =====
@test "Mock: 路径覆盖" {
    # 创建临时 bin 目录，优先于系统 PATH
    MOCK_BIN="$(mktemp -d)"
    export PATH="$MOCK_BIN:$PATH"

    # 创建 mock curl 脚本
    cat > "$MOCK_BIN/curl" << 'EOF'
#!/bin/bash
echo '{"status": "mocked"}'
EOF
    chmod +x "$MOCK_BIN/curl"

    run curl http://example.com/api
    assert_output '{"status": "mocked"}'
}

# ===== 方法 3: 使用 shellmock 辅助库 =====
# https://github.com/capitalone/bash_shell_mock
setup() {
    load 'test_helper/shellmock/load'
    shellmock_clean
}

@test "Mock: 使用 shellmock" {
    shellmock_expect curl --status 0 --output '{"name": "test"}' \
        --match-args '--silent' --type partial

    run my_curl_wrapper --silent http://example.com
    assert_output '{"name": "test"}'

    shellmock_verify
    shellmock_verify_times 1    # 确认 curl 被调用了 1 次
}
```

## 8.6 覆盖率

```bash
# 使用 kcov 收集 Shell 脚本覆盖率
# 安装 kcov
sudo apt install kcov           # Debian/Ubuntu
brew install kcov               # macOS

# 运行 Bats 并收集覆盖率
kcov --include-pattern=.sh coverage_dir bats tests/

# 生成 HTML 报告
# coverage_dir/index.html 包含每行的执行次数

# 在 CI 中使用
kcov --include-path=. \
     --exclude-path=tests/ \
     coverage_dir \
     bats tests/
```

# 9. 现代化脚本替代方案

当 Shell 脚本过于复杂时，以下现代工具有助于用更高级的语言逻辑编写自动化任务。

## 9.1 Just（命令执行器）

Just 是 Make 的现代替代品，专注于**命令执行**而非构建系统，无 `.PHONY` 痛苦、无 Tab/空格问题。

**安装：**

```bash
brew install just        # macOS
scoop install just       # Windows
cargo install just       # 通过 Rust
# 或下载 GitHub Releases 单二进制文件
```

**Justfile 示例：**

```makefile
# justfile（或 Justfile）
# 语法简洁，支持变量、依赖、参数

# 默认配方
default:
    @just --list

# 变量
app_name := "myapp"
version  := "0.1.0"

# 简单配方
build:
    @echo "Building {{app_name}} v{{version}}..."
    go build -ldflags "-X main.version={{version}}" -o bin/{{app_name}} .

# 带依赖的配方
test: lint
    go test -v -race -cover ./...

lint:
    golangci-lint run ./...

# 带参数的配方
release tag='' message='':
    @just test
    git tag -a "v{{tag}}" -m "{{message}}"
    git push origin "v{{tag}}"

# 环境变量
[working-directory: 'frontend']
frontend-build:
    npm install
    npm run build

# 配方组
[private]
docker-build: docker-login
    docker build -t {{app_name}}:{{version}} .

docker-login:
    @echo "{{DOCKER_PASSWORD}}" | docker login -u "{{DOCKER_USER}}" --password-stdin

# 平台检测
install:
    @if [ "{{os()}}" = "macos" ]; then \
        brew install {{app_name}}; \
    else \
        cargo install {{app_name}}; \
    fi

# 列表显示选项
[no-cd]
deploy:
    kubectl apply -f k8s/
```

**Just vs Make 对比：**

| 特性 | Make | Just |
|------|------|------|
| 语法 | Tab 缩进严格、隐式规则复杂 | 无缩进要求、简洁直观 |
| 构建系统 | 是（文件依赖追踪） | 否（命令执行器） |
| .PHONY | 必须声明 | 所有配方默认为 phony |
| 第一个配方 | 默认执行 | 可指定或不指定 |
| 参数传递 | 需通过变量 | 原生参数支持 `just release v1.2` |
| 三斜杠注释 | 无 | `/// 注释` 在 `--list` 中显示 |
| Shebang 行 | 无 | `#!/usr/bin/env just` 可独立运行 |
| 跨平台 | Unix | Windows/Linux/macOS |
| 单二进制 | 否（需 make） | 是（Rust 编译） |

## 9.2 Task（Go Task）

Task 是 YAML 格式的任务执行器，使用 Go 编写，类似 Make 但配置更结构化。

**安装：**

```bash
npm install -g @go-task/cli   # npm
brew install go-task           # macOS
scoop install task             # Windows
go install github.com/go-task/task/v3/cmd/task@latest
```

**Taskfile.yml 示例：**

```yaml
# Taskfile.yml
version: '3'

vars:
  APP_NAME: myapp
  VERSION: 0.1.0

env:
  GO111MODULE: 'on'

tasks:
  default:
    desc: 显示所有可用任务
    cmds:
      - task --list

  build:
    desc: 编译应用程序
    sources:
      - ./**/*.go
    generates:
      - ./bin/{{.APP_NAME}}
    cmds:
      - go build -ldflags "-X main.version={{.VERSION}}" -o bin/{{.APP_NAME}} .

  test:
    desc: 运行测试
    deps: [lint]
    cmds:
      - go test -v -race -cover ./...

  lint:
    desc: 代码检查
    cmds:
      - golangci-lint run ./...

  docker-build:
    desc: 构建 Docker 镜像
    vars:
      TAG: '{{.VERSION | default "latest"}}'
    cmds:
      - docker build -t {{.APP_NAME}}:{{.TAG}} .

  deploy:
    desc: 部署到 K8s
    deps: [docker-build]
    cmds:
      - kubectl apply -f k8s/

  # 有条件执行
  release:
    desc: 创建 GitHub Release
    cmds:
      - echo "Creating release {{.CLI_ARGS}}"
      - gh release create {{.CLI_ARGS}}

  # 平台特定
  setup:
    desc: 安装开发依赖
    cmds:
      - cmd: brew bundle
        platforms: [darwin]
      - cmd: sudo apt install -y build-essential
        platforms: [linux]
```

**Task vs Just vs Make：**

| 维度 | Make | Just | Task |
|------|------|------|------|
| 配置格式 | Makefile（自定义） | justfile | Taskfile.yml (YAML) |
| 文件依赖追踪 | ✅ | ❌ | ✅ (sources/generates) |
| 语言 | C | Rust | Go |
| 模板引擎 | 基础 | 基础 (Cheetah-like) | Go template |
| 学习曲线 | 高（隐式规则） | 低 | 低 |
| 社区规模 | 非常大 | 增长中 | 增长中 |

## 9.3 Zx（Google 的 Node.js + Shell 混合）

Zx 由 Google 开发，允许在 JavaScript/TypeScript 中编写 Shell 脚本，用 Node.js 的生态辅助命令行操作。

**安装：**

```bash
# 全局安装
npm install -g zx

# 或使用 npx（无需全局安装）
npx zx script.mjs
```

**Zx 脚本示例：**

```js
#!/usr/bin/env zx
// script.mjs

// 在 $`...` 模板中直接写 Shell 命令
await $`echo "Hello, Zx!"`;

// 变量自动转义
const dir = '/path/with spaces';
await $`ls -la ${dir}`;           // 自动处理空格，无需担心注入

// 捕获输出
const branch = (await $`git branch --show-current`).stdout.trim();
console.log(`当前分支: ${branch}`);

// 高级用法
const files = await glob('**/*.js');               // 文件 glob
console.log(`找到 ${files.length} 个 JS 文件`);

// 并行执行
const promises = files.map(file =>
    $`eslint ${file}`
);
await Promise.all(promises);

// 环境变量
$.env.NODE_ENV = 'production';
await $`npm run build`;

// 条件执行
const hasDocker = await which('docker');
if (hasDocker) {
    await $`docker build -t app .`;
}

// 彩色输出
echo(chalk.green('构建成功!'));
echo(chalk.red.bold('错误:'), '文件未找到');

// 用户交互
const response = await question('是否继续部署? [y/N] ');
if (response.toLowerCase() !== 'y') {
    echo('已取消');
    process.exit(0);
}

// 管道
const { stdout } = await $`cat package.json | jq .version`;
echo(`版本: ${stdout}`);

// 错误处理
try {
    await $`kubectl apply -f deployment.yaml`;
} catch (p) {
    echo(`部署失败: ${p.stderr}`);
}

// Node.js 生态集成
import { readFile, writeFile } from 'fs/promises';
const pkg = JSON.parse(await readFile('package.json', 'utf-8'));
pkg.scripts.test = 'zx test.mjs';
await writeFile('package.json', JSON.stringify(pkg, null, 2));

// Markdown 脚本（.md 文件中的代码块可直接运行）
// zx script.md
```

**Zx 优势与限制：**

| 优势 | 限制 |
|------|------|
| JavaScript 完整能力（JSON 解析、异步、HTTP） | 需要 Node.js 运行时 |
| 自动参数转义（防注入） | 启动速度慢于纯 Shell |
| async/await 并行天然 | 不适合极简环境（Docker 微型镜像） |
| 可在 Markdown 中编写脚本 | 错误信息有时不如纯 Shell 直观 |
| npm 生态无限扩展 | 不能给所有运维人员开箱即用 |

## 9.4 Python Invoke

Python Invoke 是 Python 版本的 Rake/任务执行器，面向 Python 开发者。

```python
# tasks.py
from invoke import task

@task
def build(c, docs=False):
    """构建项目"""
    c.run("python setup.py build")
    if docs:
        c.run("sphinx-build docs docs/_build")

@task
def test(c, coverage=False):
    """运行测试"""
    cmd = "python -m pytest"
    if coverage:
        cmd += " --cov=myapp"
    c.run(cmd)

@task(build, test)
def ci(c):
    """CI 流水线（build → test）"""
    c.run("echo 'CI 完成'")

@task
def deploy(c, env='staging'):
    """部署到指定环境"""
    c.run(f"ansible-playbook -i inventory/{env} deploy.yml")
```

```bash
# 运行
inv build
inv test --coverage
inv ci
inv deploy --env=production
inv --list          # 列出所有任务
```

**工具选择决策树：**

```
是否需要基于文件的增量构建？
├─ 是 → 是否已有 Makefile？
│       ├─ 是 → Make（保持兼容）
│       └─ 否 → Task（YAML 配置，sources/generates）
└─ 否 → 项目语言是什么？
        ├─ Node.js/前端 → Zx
        ├─ Python → Invoke / Just
        ├─ Go/Rust → Just / Task
        └─ 多语言/运维 → Just（最简单）
```

# 10. 终端复用器

## 10.1 Tmux 脚本化

Tmux 不仅是交互式的终端复用器，也可以通过脚本精确控制窗口和窗格的布局。

**Tmux 配置文件（`~/.tmux.conf`）：**

```bash
# === 基础配置 ===
set -g default-terminal "tmux-256color"
set -g mouse on                           # 鼠标支持
set -g history-limit 50000                # 回滚行数
set -g base-index 1                       # 窗口编号从 1 开始
set -g pane-base-index 1                  # 窗格编号从 1 开始
set -g renumber-windows on                # 关闭窗口后重新编号

# === 快捷键修改 ===
unbind C-b
set -g prefix C-a                         # 前缀改为 Ctrl+A
bind C-a send-prefix

bind r source-file ~/.tmux.conf \; display "已重新加载配置"

# 分割窗格快捷键
bind \\ split-window -h -c "#{pane_current_path}"  # 水平分割
bind -  split-window -v -c "#{pane_current_path}"  # 垂直分割

# Vim 风格导航
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# === 状态栏美化 ===
set -g status-style fg=white,bg=black
set -g status-left-length 30
set -g status-right-length 60
set -g status-left '#[fg=green]#S #[fg=yellow]#I #P'
set -g status-right '#[fg=cyan]%Y-%m-%d %H:%M'

# === 插件（tpm 插件管理器） ===
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'     # 通用合理默认值
set -g @plugin 'tmux-plugins/tmux-resurrect'    # 保存/恢复会话
set -g @plugin 'tmux-plugins/tmux-continuum'    # 自动保存/恢复

# tpm 初始化（在配置末尾）
run '~/.tmux/plugins/tpm/tpm'
```

**Tmux 脚本化布局：**

```bash
#!/bin/bash
# tmux-dev.sh — 创建开发环境布局

SESSION="dev"

# 如果会话已存在，附加之；否则创建
tmux has-session -t "$SESSION" 2>/dev/null
if [ $? -eq 0 ]; then
    tmux attach-session -t "$SESSION"
    exit 0
fi

# 创建新会话（分离模式 detach）
tmux new-session -d -s "$SESSION" -n "editor" -c "$HOME/projects"

# 窗格 0: 主编辑区（占 70% 高度）
tmux send-keys -t "$SESSION:editor" "nvim ." C-m

# 窗格 1: 底部终端（30% 高度）
tmux split-window -v -p 30 -t "$SESSION:editor" -c "$HOME/projects"
tmux send-keys -t "$SESSION:editor.1" "git status" C-m

# 窗口 2: 服务运行
tmux new-window -t "$SESSION" -n "server" -c "$HOME/projects"
tmux send-keys -t "$SESSION:server" "npm run dev" C-m

# 窗口 3: 数据库
tmux new-window -t "$SESSION" -n "database" -c "$HOME/projects"
tmux send-keys -t "$SESSION:database" "psql -U postgres mydb" C-m

# 窗口 4: 日志监控
tmux new-window -t "$SESSION" -n "logs"
tmux split-window -h -t "$SESSION:logs"
tmux send-keys -t "$SESSION:logs.0" "tail -f /var/log/app/access.log" C-m
tmux send-keys -t "$SESSION:logs.1" "tail -f /var/log/app/error.log" C-m

# 回到第一个窗口
tmux select-window -t "$SESSION:editor"

# 附加到会话
tmux attach-session -t "$SESSION"
```

**多服务器部署窗口布局：**

```bash
#!/bin/bash
# tmux-servers.sh — 为多台服务器创建 SSH 窗口

SESSION="servers"
SERVERS=("web01" "web02" "db01" "cache01")

tmux new-session -d -s "$SESSION" -n "ops"

for i in "${!SERVERS[@]}"; do
    if [ "$i" -eq 0 ]; then
        tmux send-keys -t "$SESSION:ops" "ssh ${SERVERS[$i]}" C-m
    else
        tmux split-window -v -t "$SESSION:ops"
        tmux select-layout -t "$SESSION:ops" tiled
        tmux send-keys -t "$SESSION:ops" "ssh ${SERVERS[$i]}" C-m
    fi
done

# 开启同步输入（一次操作所有窗格）
tmux set-window-option -t "$SESSION:ops" synchronize-panes on

tmux attach-session -t "$SESSION"
```

## 10.2 Screen 基本使用

GNU Screen 是 Tmux 的前身，功能类似但更轻量。在现代场景下 Tmux 已基本取代 Screen，但 Screen 仍在一些老系统和嵌入式环境中使用。

```bash
# === Screen 基础命令 ===

# 创建命名会话
screen -S mysession

# 列出会话
screen -ls

# 附加到会话
screen -r mysession

# 分离（detach）
Ctrl+A, d

# 创建新窗口
Ctrl+A, c

# 切换窗口
Ctrl+A, n (next) / p (prev) / 0-9 (按编号)

# 分割窗口（Screen 4.x+）
Ctrl+A, S                    # 水平分割
Ctrl+A, |                    # 垂直分割
Ctrl+A, Tab                  # 切换到下一个区域

# 复制模式
Ctrl+A, [                    # 进入复制模式（VI 键导航）

# 重命名窗口
Ctrl+A, A

# 关闭窗口
Ctrl+A, k  或  exit
```

**Screen 配置文件（`~/.screenrc`）：**

```bash
# ~/.screenrc
startup_message off                # 关闭启动消息
defscrollback 10000                # 回滚行数
term screen-256color               # 终端类型

# 状态栏
hardstatus alwayslastline
hardstatus string '%{= kG}%-w%{= kW}%n %t%{-}%+w %=%{= kG}%H %l %c %D'

# 快捷键
escape ^Aa                         # 前缀设为 Ctrl+A（默认）
bindkey -k k1 screen 1             # F1 切换窗口 1
bindkey -k k2 screen 2             # F2 切换窗口 2
```

**Screen 脚本化：**

```bash
#!/bin/bash
# screen-daemon.sh — 在 Screen 中启动守护进程

screen -dmS myapp                   # 创建分离的会话
screen -S myapp -X stuff $'cd /opt/app\n'
screen -S myapp -X stuff $'./start.sh\n'
echo "应用已在 Screen 会话 'myapp' 中启动"
```

**Tmux vs Screen 对比：**

| 特性 | Tmux | Screen |
|------|------|-------|
| 分割窗口 | 原生，灵活 | 4.x 后支持，功能较弱 |
| 客户端/服务器模型 | ✅ 原生设计 | 较老旧 |
| 会话持久化 | 需插件 (tmux-resurrect) | ❌ |
| 脚本化 | 优秀（tmux CLI） | 基础 |
| 配置文件 | `~/.tmux.conf` | `~/.screenrc` |
| 社区活跃度 | 高（现代标准） | 低（遗留维护） |
| 认知负担 | 中 | 低 |
| 适用场景 | 个人开发、运维 | 嵌入式、旧系统兼容 |

# 11. 终端增强工具

现代终端工作流依赖一系列高性能工具替代传统 Unix 工具。这些工具多数由 Rust 编写，速度快，用户体验好。

## 11.1 Fzf（模糊搜索）

Fzf 是通用的命令行模糊搜索器，可以过滤任何列表输入。

**安装：**

```bash
brew install fzf           # macOS
sudo apt install fzf       # Debian/Ubuntu
sudo dnf install fzf       # Fedora
scoop install fzf          # Windows
# 安装后运行 install 脚本以绑定快捷键
$(brew --prefix)/opt/fzf/install
```

**核心用法：**

```bash
# ===== 交互式模糊搜索 =====

# 从 stdin 搜索
find . -type f | fzf

# Ctrl+T: 粘贴文件名到命令行
# Ctrl+R: 模糊搜索命令历史
# Alt+C:  模糊搜索目录并 cd

# ===== 预览模式 =====
fzf --preview 'bat --color=always {}'           # 预览文件内容
fzf --preview 'ls -la {}'                        # 预览文件信息
fzf --preview 'cat {} | head -100'               # 预览前 100 行

# ===== 自定义预览窗口 =====
fzf --preview 'bat --style=numbers --color=always {}' \
    --preview-window 'right:60%:wrap'            # 右侧 60%

# ===== 多选 =====
fzf -m                                           # Tab 多选
# 输出以换行分隔

# ===== 绑定快捷键 =====
# ~/.bashrc 或 ~/.zshrc
eval "$(fzf --bash)"  # 或 $(fzf --zsh)

# ===== Shell 集成脚本示例 =====
# 交互式 git checkout
gco() {
    git branch --all | fzf --preview 'git log --oneline {} | head -20' | \
        sed 's/.* //' | xargs git checkout
}

# 交互式 kill 进程
fkill() {
    ps aux | fzf --multi --preview 'echo {}' | \
        awk '{print $2}' | xargs kill -9
}

# 交互式 docker
fzfdocker() {
    docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}' | \
        fzf --preview 'docker inspect {} | jq .' | \
        awk '{print $1}'
}
```

## 11.2 Ripgrep（rg）

Ripgrep 是用 Rust 编写的超高速文本搜索工具，设计为 grep 的直接替代且更快。

**安装：**

```bash
brew install ripgrep         # macOS
sudo apt install ripgrep    # Debian/Ubuntu
sudo dnf install ripgrep    # Fedora
scoop install ripgrep       # Windows
```

**核心用法对比：**

```bash
# ===== 基本搜索 =====
rg pattern                    # 递归搜索当前目录
rg pattern dir/               # 指定目录
rg pattern file.txt           # 单文件搜索
rg -g '*.py' pattern          # glob 过滤

# ===== grep vs rg 对比 =====
grep -rn "TODO" .             # 传统方式
rg TODO                       # ripgrep（自动递归，自动忽略 .gitignore）

# ===== 文件类型过滤 =====
rg -t py pattern              # 只搜索 Python 文件
rg -T py pattern              # 排除 Python 文件
rg --type-list                # 列出所有支持的文件类型

# ===== 搜索选项 =====
rg -i pattern                 # 忽略大小写
rg -w pattern                 # 匹配整个单词
rg -v pattern                 # 反向匹配
rg -l pattern                 # 只列出文件名
rg -c pattern                 # 统计匹配次数
rg --no-heading pattern       # 不显示文件名头部
rg -n pattern                 # 显示行号（默认）
rg -H pattern                 # 始终显示文件名

# ===== 上下文 =====
rg -C 3 pattern               # 前后各 3 行上下文
rg -B 2 pattern               # 前 2 行
rg -A 5 pattern               # 后 5 行

# ===== 高级 =====
rg --hidden pattern           # 搜索隐藏文件和目录
rg --no-ignore pattern        # 不遵守 .gitignore
rg -g '!*.min.js' pattern     # 排除 .min.js 文件
rg -g 'src/**/*.rs' pattern   # 限制到特定路径

# ===== 替换（rg 可以替换，但通常用 sd） =====
rg -l 'old_name' | xargs sed -i 's/old_name/new_name/g'

# ===== 多行模式 =====
rg -U 'function\s+\w+\s*\{[^}]*\}'   # -U 多行模式

# ===== JSON 输出 =====
rg --json pattern > results.json      # 供程序解析
```

**性能对比（1GB 代码库搜索）：**

| 工具 | 时间 | 相对速度 |
|------|------|----------|
| `grep -r` | 12.3s | 1x |
| `ag` (The Silver Searcher) | 3.1s | 4x |
| `git grep` | 1.8s | 7x |
| `rg` (ripgrep) | 0.4s | 30x |

## 11.3 Fd

Fd 是 `find` 的现代替代，更快、更直观的语法。

**安装：**

```bash
brew install fd              # macOS
sudo apt install fd-find     # Debian/Ubuntu (命令名为 fdfind)
sudo dnf install fd-find     # Fedora
scoop install fd             # Windows
```

**核心用法对比：**

```bash
# find vs fd 对比
find . -name '*.py'                       # find
fd '\.py$'                                # fd（正则默认，更简洁）
# 或
fd -e py                                  # 按扩展名
fd -g '*.py'                              # 按 glob 模式

# ===== 文件类型过滤 =====
fd -t f                  # 只匹配文件
fd -t d                  # 只匹配目录
fd -t l                  # 只匹配符号链接
fd -t x                  # 只匹配可执行文件
fd -t e                  # 只匹配空文件/目录

# ===== 大小过滤 =====
fd -s +10m               # >10MB
fd -s -1k                # <1KB

# ===== 时间过滤 =====
fd --changed-within 24h     # 24 小时内修改
fd --changed-before 7d      # 7 天前修改
fd --changed-within 30min   # 30 分钟内修改

# ===== 执行命令 =====
fd -e py -x wc -l              # 对每个匹配执行 wc -l
fd -e py -X wc -l              # 所有文件作为参数传给 wc -l（批量）
fd -e py -x echo {.}           # {.} 不带扩展名, {} 完整路径, {/} 文件名

# ===== 高级查询 =====
fd '^test' --type f            # 以 test 开头的文件
fd -H pattern                  # 包含隐藏文件
fd -I pattern                  # 包含 .gitignore 排除的文件
fd -p 'src/(app|lib)'          # 匹配完整路径
fd -d 2                        # 最大递归深度 2 层

# ===== 输出格式 =====
fd -0                          # null 分隔输出（配合 xargs -0）
fd -l                          # 详细列表模式（类似 ls -l）
```

**Fd vs Find 对比：**

| 特性 | Find | Fd |
|------|-------|-----|
| 默认递归 | ✅ | ✅ |
| 遵守 .gitignore | ❌ | ✅（默认！） |
| 正则表达式 | `-regex` 选项 | 默认（`-g` 切换 glob） |
| 彩色输出 | ❌ | ✅ |
| 智能大小写 | ❌ | ✅（全小写自动忽略大小写） |
| 速度（热缓存） | 基准 | 3-5x |
| 速度（冷缓存） | 基准 | 5-9x |
| 语法复杂度 | 高 | 低 |

## 11.4 Bat（cat 带语法高亮）

Bat 是 `cat` 的高亮替代，支持语法高亮、行号、Git 修改标记、分页。

**安装：**

```bash
brew install bat             # macOS
sudo apt install bat         # Debian/Ubuntu (命令为 batcat)
scoop install bat            # Windows
```

**核心用法：**

```bash
# 基本使用
bat file.py                            # 语法高亮 + 行号
bat -n file.py                         # 只显示行号（无高亮）
bat -p file.py                         # 纯文本模式（禁用行号和边框）
bat --style=plain file.py              # 同上

# 多文件
bat file1.py file2.sh                  # 逐个显示

# 语言覆盖
bat -l json config.yml                 # 以 JSON 语法高亮处理 YAML

# 主题
bat --list-themes                      # 列出所有主题
bat --theme="Dracula" file.py          # 指定主题
export BAT_THEME="Nord"               # 环境变量永久设置

# Git 集成
bat --diff file.py                     # 显示 Git 修改（增量行标记 ~ + -）
bat --show-all file.py                 # 显示空白字符和行尾

# 作为 man 的语法高亮器
export MANPAGER="sh -c 'col -bx | bat -l man -p'"
# 或
export MANROFFOPT="-c"
export MANPAGER="bat -l man -p"

# 范围读取
bat -r 100:200 file.py                 # 只显示 100 到 200 行

# 十六进制查看
bat --hex file.bin                     # 十六进制 + ASCII 双栏显示
```

## 11.5 Delta（Git Diff 增强）

Delta 是 `git diff` 的语法高亮查看器，用 Rust 编写。

**安装：**

```bash
brew install git-delta        # macOS
scoop install delta           # Windows
cargo install git-delta       # Rust
```

**Git 配置（`~/.gitconfig`）：**

```ini
[core]
    pager = delta

[interactive]
    diffFilter = delta --color-only

[delta]
    features = side-by-side line-numbers decorations
    syntax-theme = Monokai Extended
    plus-style    = "syntax #003300"
    minus-style   = "syntax #330000"

[delta "decorations"]
    commit-decoration-style = bold yellow box ul
    file-style              = bold yellow ul
    file-decoration-style   = none
    hunk-header-decoration-style = yellow box
    hunk-header-style       = file line-number syntax

[delta "line-numbers"]
    line-numbers-left-style  = cyan
    line-numbers-right-style = cyan
    line-numbers-minus-style = 124
    line-numbers-plus-style  = 28

[delta "side-by-side"]
    side-by-side = true

[merge]
    conflictstyle = diff3

[diff]
    colorMoved = default
```

**使用效果：**

```bash
git diff                    # 自动使用 delta 渲染
git diff --color-words      # 单词级 diff（delta 高亮）
git show HEAD               # 查看 commit 时使用 delta
git log -p                  # 日志中的 patch 也用 delta
```

## 11.6 Zoxide（智能目录跳转）

Zoxide 是 `cd` 的智能替代，使用频率和最近访问来预测你要去的目录。兼容 `z`、`autojump`。

**安装：**

```bash
brew install zoxide          # macOS
sudo apt install zoxide      # Debian/Ubuntu
scoop install zoxide         # Windows
cargo install zoxide         # Rust
```

**Shell 集成：**

```bash
# ~/.bashrc 或 ~/.zshrc
eval "$(zoxide init bash)"   # 或 zsh / fish / nushell

# 之后可以使用：
z foo                        # 跳转到最匹配 "foo" 的目录
zi foo                       # 交互式选择（使用 fzf）
z bar/                       # 精确子路径匹配
z ..                         # 跳转到父目录

# 管理数据库
z -l                         # 列出所有已知目录
z -r /tmp                    # 从数据库中移除某目录
z query/                     # 只匹配查询字符串在路径末尾的目录
```

**Zoxide vs cd vs z vs autojump：**

| 特性 | cd | z | autojump | zoxide |
|------|----|---|----------|--------|
| 算法 | 无 | 频率 | 频率 | 频率 + Fresnel |
| 交互模式 | ❌ | ❌ | ❌ | ✅（`zi` 集成 fzf） |
| 数据库 | ❌ | `~/.z` | `~/.local/share/autojump/` | 跨 Shell 共享数据库 |
| 跨 Shell | 天然 | 否 | 否 | ✅（bash/zsh/fish/elvish/nushell） |
| 跨会话 | ❌ | ✅ | ✅ | ✅ |
| 查询语法 | 精确 | 部分匹配 | 部分匹配 | 部分匹配 + 子路径 |
| 速度 | 即时 | 即时 | 即时 | 即时 |

## 11.7 工具组合推荐

**个人终端工具集（最小化推荐）：**

```bash
# 一键安装所有现代工具
# macOS
brew install fzf ripgrep fd bat git-delta zoxide

# Debian/Ubuntu
sudo apt install fzf ripgrep fd-find bat git-delta zoxide

# Shell 配置整合
# ~/.zshrc 或 ~/.bashrc

# Fzf
eval "$(fzf --zsh)"                      # zsh
# eval "$(fzf --bash)"                   # bash
export FZF_DEFAULT_OPTS='--height 40% --layout reverse --border'

# Bat
export BAT_THEME="Dracula"
alias cat="bat --paging=never"

# Fd
alias find="fd"

# Ripgrep (alias)
alias grep="rg"

# Zoxide
eval "$(zoxide init zsh)"
alias cd="z"

# Delta (通过 gitconfig 集成)
```

**开发工作流速查：**

```bash
# 1. 模糊搜索文件并编辑
nvim "$(fd -t f | fzf --preview 'bat --color=always {}')"

# 2. 在搜索结果中模糊选文件
rg -l "TODO" | fzf --preview 'bat --color=always {}' | xargs nvim

# 3. 交互式 Git 分支切换
git branch | fzf --preview 'git log --oneline {} | head -10' | tr -d ' *' | xargs git checkout

# 4. 智能目录跳转
z proj<Tab>          # zoxide 自动补全

# 5. 彩色日志查看
tail -f app.log | bat --paging=never -l log

# 6. 进程搜索与 kill
ps aux | fzf -m | awk '{print $2}' | xargs kill

# 7. 搜索代码中的 pattern 并在编辑器打开
rg -l 'pattern' | fzf --preview 'bat --color=always {} --style=numbers' | xargs nvim

# 8. Docker 容器日志
docker ps | fzf --preview 'docker logs --tail=50 {-1}' | awk '{print $1}' | \
    xargs -I {} docker logs -f {}
```
