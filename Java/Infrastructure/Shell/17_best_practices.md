
# 1. 脚本头部模板

## 1.1 最小化脚本模板

```bash
#!/bin/bash
#
# script_name.sh - 一句话描述脚本功能
#
# 作者:    Your Name <email@example.com>
# 日期:    2026-01-01
# 版本:   1.0.0
# 许可:   MIT
#
# 描述:
#   详细描述脚本的作用、适用场景以及关键逻辑。
#
# 用法:
#   ./script_name.sh [options] <arguments>
#
# 示例:
#   ./script_name.sh -v -o output.txt input.txt
#   ./script_name.sh --help
#
# 依赖:
#   - jq >= 1.6
#   - curl >= 7.0
#   - bash >= 4.0

set -euo pipefail
IFS=$'\n\t'

# ── 全局变量 ──────────────────────────────────────────
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
readonly SCRIPT_VERSION="1.0.0"

# ── 函数定义 ──────────────────────────────────────────
usage() {
    cat <<EOF
用法: $SCRIPT_NAME [options] <arguments>

选项:
    -h, --help        显示此帮助信息
    -v, --verbose     输出详细信息
    -o, --output FILE 指定输出文件（默认: stdout）

示例:
    $SCRIPT_NAME -v input.txt
    $SCRIPT_NAME -o result.json data.csv
EOF
    exit "${1:-0}"
}

error() {
    echo "[${SCRIPT_NAME}] 错误: $*" >&2
    exit 1
}

main() {
    # 主逻辑在此
    echo "Hello, World!"
}

# ── 入口 ──────────────────────────────────────────────
main "$@"
```

## 1.2 Shebang 最佳选择

```bash
#!/usr/bin/env bash   # 推荐：通过 PATH 查找 bash，可移植性最佳
#!/bin/bash           # 常用：直接指定绝对路径
#!/bin/sh             # POSIX 兼容脚本（不可用 bashisms）
#!/bin/zsh            # zsh 脚本
```

`#!/usr/bin/env bash` 的优点：

- 用户可能将 bash 安装在不同路径（如 `/usr/local/bin/bash`、`/opt/homebrew/bin/bash`）
- `env` 在 `/usr/bin/env` 的位置高度统一，几乎永远是 `/usr/bin/env`
- **注意**：`env` 方式无法传递额外的解释器参数（如 `#!/usr/bin/env bash -e` 无效）
- 如果必须传递参数且需要可移植性，使用 polyglot 技术：

```bash
#!/bin/sh
# 这一行是注释，但 /bin/sh 执行后下一行生效
# 下面使用 set 来代替 shebang 参数
set -e
# 然后用一个技巧重新以 bash 执行自身（如有必要）
```

## 1.3 set 选项深度解析

```bash
set -euo pipefail
```

| 选项 | 全称 | 效果 | 典型场景 |
|------|------|------|----------|
| `-e` | errexit | 任何命令返回非零退出码时立即退出脚本 | 防止错误传播 |
| `-u` | nounset | 引用未定义变量时退出 | 防止拼写错误 |
| `-o pipefail` | — | 管道中任一命令失败则整个管道失败 | 防止静默失败 |
| `-x` | xtrace | 打印每条执行的命令（调试用） | 调试复杂脚本 |
| `-C` | noclobber | 重定向 `>` 不会覆盖已存在文件 | 防止意外数据丢失 |
| `-o functrace` | — | trap 对函数也生效 | 错误追踪 |
| `-o errtrace` | — | ERR trap 在函数中继承 | 错误处理 |

**`set -e` 的陷阱与例外：**

```bash
#!/bin/bash
set -e

# ❌ 以下场景 set -e 不会触发退出：

# 1. if / while / until 条件中的命令
if grep -q "pattern" file.txt; then    # grep 失败不会退出
    echo "found"
fi

# 2. 管道最右侧命令（需配合 pipefail）
false | true                            # 不会退出（true 成功）

# 3. && 或 || 左侧的命令
false && echo "won't run"               # 不会退出
true || false                           # 不会退出

# 4. 子 Shell 中的命令
( false )                               # 子 Shell 退出但父 Shell 继续

# ✅ 解决方案：
set -o pipefail                         # 搭配 -e 使用
# 在子 Shell 中传播错误：父 Shell 检查子 Shell 的 $?
```

**完整的安全选项组合：**

```bash
#!/bin/bash

# 严格模式（Strict Mode）—— 推荐所有脚本使用
set -euo pipefail

# 可选增强
set -o noclobber                        # 禁止覆盖已存在文件
shopt -s failglob                       # 通配符无匹配时报错而非原样保留
shopt -s nullglob                       # 通配符无匹配时展开为空字符串
# failglob 与 nullglob 二选一，不要同时开启

# 清理 IFS，防止意外单词分割
IFS=$' \n\t'
# 或者更严格的：
IFS=$'\n\t'
```

## 1.4 元数据注释规范

```bash
#!/usr/bin/env bash
#══════════════════════════════════════════════════════════════════
# 文件名:    deploy.sh
# 描述:      应用部署脚本，支持多环境、回滚和健康检查
# 作者:      DevOps Team <devops@company.com>
# 创建日期:  2026-01-15
# 更新日期:  2026-05-10
# 版本:      2.3.1
# 许可:      MIT
# 兼容性:    bash >= 4.0, Linux/macOS
#
# 依赖:
#   - docker >= 20.10
#   - kubectl >= 1.25
#   - jq >= 1.6
#
# 配置文件:
#   - config/deploy.conf        # 主配置文件
#   - config/environments/      # 各环境配置
#
# 变更日志:
#   2.3.1 - 修复健康检查超时问题
#   2.3.0 - 新增蓝绿部署策略
#   2.2.0 - 支持 Kubernetes 部署
#
# 参考:
#   - https://internal-wiki/deploy-guide
#   - RFC-4231: 部署流程规范
#══════════════════════════════════════════════════════════════════
```

# 2. 命名规范

## 2.1 变量命名

```bash
# ── 全局变量：大写 + 下划线分隔 ──
readonly MAX_RETRIES=5
readonly DEFAULT_OUTPUT_DIR="/var/log/app"
readonly API_BASE_URL="https://api.example.com/v2"

# ── 环境变量：全大写 ──
# 由外部传入，脚本只读
readonly DATABASE_URL="${DATABASE_URL:-}"
readonly LOG_LEVEL="${LOG_LEVEL:-INFO}"

# ── 局部变量：小写 + 下划线分隔 ──
function process_file() {
    local input_file="$1"
    local output_dir="$2"
    local file_count=0
    local temp_buffer=""
    local -a file_list=()        # 显式声明数组类型
    local -A config_map=()       # 显式声明关联数组

    for input_file in *.txt; do
        local relative_path="${input_file##*/}"
        file_count=$((file_count + 1))
    done
}

# ── 临时变量：小写，使用删除线风格或 temp_ 前缀 ──
_tmp=$(mktemp)
tmp_output="$(mktemp -d)/result.json"
```

**命名约定速查表：**

| 类型 | 风格 | 示例 | 说明 |
|------|------|------|------|
| 全局常量 | `UPPER_SNAKE` | `MAX_CONNECTIONS` | 只读变量 |
| 环境变量 | `UPPER_SNAKE` | `DATABASE_URL` | 外部传入 |
| 局部变量 | `lower_snake` | `file_count` | 函数内使用 |
| 布尔变量 | `is_` / `has_` 前缀 | `is_ready` `has_error` | 语义化 |
| 临时文件 | `_tmp` / `temp_` | `_tmp_fifo` `temp_dir` | 须清理 |
| 循环变量 | 单字母或简短名 | `i` `j` `item` `line` | 作用域最小 |

## 2.2 函数命名

```bash
# ── 函数名：小写 + 下划线分隔 + 动词开头 ──

# ✅ 推荐
check_prerequisites()   { ... }
parse_arguments()       { ... }
validate_input()        { ... }
build_docker_image()    { ... }
deploy_to_kubernetes()  { ... }
send_notification()     { ... }
cleanup_temp_files()    { ... }
print_usage()           { ... }

# ❌ 避免
parseArgs()             # 驼峰命名，与 Shell 惯例不符
CheckDependencies()     # 大写开头，容易与环境变量混淆
do_it()                 # 太模糊，不表达意图
func1()                 # 无意义命名
_private_helper()       # 下划线前缀表示"私有"是可选的约定
```

**常用动词前缀：**

| 动词 | 含义 | 示例 |
|------|------|------|
| `get_` | 获取数据 | `get_container_id` |
| `set_` | 设置值 | `set_config_value` |
| `check_` | 检查条件 | `check_port_open` |
| `validate_` | 验证合法性 | `validate_email` |
| `parse_` | 解析数据 | `parse_json_response` |
| `build_` | 构建资源 | `build_docker_image` |
| `create_` | 创建资源 | `create_log_dir` |
| `remove_` | 删除资源 | `remove_stale_containers` |
| `print_` | 输出信息 | `print_help` `print_error` |
| `run_` | 执行流程 | `run_migrations` |
| `ensure_` | 确保状态 | `ensure_directory_exists` |

## 2.3 文件名规范

```bash
# ✅ 推荐：小写 + 下划线分隔 + .sh 后缀
deploy_application.sh
database_backup.sh
user_data_cleanup.sh
health_check.sh

# ✅ 可执行工具（不加 .sh 也可，放在 bin/ 目录下）
bin/appctl
bin/logrotate

# ❌ 避免
deployApplication.sh      # 驼峰
Deploy-App.sh             # 大小写混用 + 短横
deploy_app.SH             # 大小写不一致
setup.sh                  # 太泛化，不表达具体用途
script1.sh                # 无意义
```

**目录结构约定：**

```
project/
├── bin/                   # 可执行脚本（可无 .sh 后缀）
│   ├── appctl
│   └── deploy
├── lib/                   # 共享函数库（用 source 引入）
│   ├── logging.sh
│   ├── utils.sh
│   └── validation.sh
├── config/                # 配置文件
│   ├── default.conf
│   └── production.conf
├── tests/                 # 测试脚本
│   ├── test_utils.bats
│   └── test_main.bats
├── docs/                  # 文档
│   └── usage.md
├── .shellcheckrc          # ShellCheck 配置
└── README.md
```

# 3. 引号使用原则

## 3.1 黄金法则

> **永远用双引号包裹变量展开和命令替换，除非你明确知道需要单词分割或通配符展开。**

```bash
# ── 核心原则演示 ──

name="John Smith"
file_path="/path/with spaces/data.txt"

# ❌ 无引号：单词分割导致错误
cp $file_path /tmp/       # 变成 cp /path/with spaces/data.txt /tmp/（4个参数）

# ✅ 双引号：保持为单一参数
cp "$file_path" /tmp/     # 正确

# ❌ 命令替换无引号：输出被单词分割
for file in $(find . -name "*.txt"); do   # 文件名包含空格时会断裂
    echo "$file"
done

# ✅ 使用 while read 或 mapfile
find . -name "*.txt" -print0 | while IFS= read -r -d '' file; do
    echo "$file"
done
```

## 3.2 何时必须使用双引号

| 场景 | 示例 | 不引用的后果 |
|------|------|-------------|
| 变量包含空格 | `"$full_name"` | 单词分割 |
| 变量可能为空 | `"$optional_env"` | 参数消失 |
| 变量包含通配符 | `"$pattern"` | 意外的 glob 展开 |
| 命令替换 | `"$(command)"` | 输出被分割 |
| 包含特殊字符 | `"$url"` (含 `&`) | 被解析为其他操作 |

```bash
# ── 常见需要引用的场景 ──

# 1. 条件判断中
if [ "$name" = "admin" ]; then          # [ ] 中必须引用
    echo "welcome"
fi

if [[ $name == admin ]]; then           # [[ ]] 中可以不引用（但建议引用）
    echo "welcome"
fi

# 2. 赋值时
result="$(some_command)"                 # 赋值时不需要引号但不影响
# 等同于
result=$(some_command)                   # 赋值时单词分割被抑制

# 3. echo / printf 输出
echo "$variable"                         # ✅
printf '%s\n' "$variable"               # ✅

# 4. 命令参数
rsync -av "$source_dir/" "$dest_dir/"   # ✅ 路径必须引用

# 5. 函数参数传递
my_function "$arg1" "$arg2"             # ✅

# 6. heredoc 分隔符引用（禁止展开时）
cat <<'EOF'                              # 引用 EOF 禁止变量展开
Current user: $USER       # 原样输出 $USER
EOF
```

## 3.3 何时不需要双引号

```bash
# 1. 赋值语句中等号右边（单词分割自动抑制）
count=0
message=hello
path=/usr/local/bin

# 2. case 语句的匹配模式
case $status in
    ok|error) echo "valid" ;;            # 不需要引用
esac

# 3. [[ ]] 内部（bash/zsh 专有的条件表达式）
if [[ $name == admin && $age -gt 18 ]]; then   # 内部不进行单词分割
    echo "valid"
fi

# 4. 整数变量的算术展开
if (( count > 5 )); then                # (( )) 内部自动展开变量
    echo "large"
fi

# 5. 变量立即接字符串时的某些情况
echo "${prefix}_suffix"                  # 需要 {} 界定，但引号可省略（仍建议加）

# 6. 通配符展开的意图本身
files=( *.txt )                          # 故意让 glob 展开
```

## 3.4 常见引号错误

```bash
# ── 错误 1：单引号阻止变量展开 ──
name='Alice'
echo '$name'                  # 输出: $name（字面值）
echo "$name"                  # 输出: Alice

# ── 错误 2：引号嵌套未转义 ──
# ❌
echo "She said "Hello""
# ✅
echo "She said \"Hello\""
echo 'She said "Hello"'       # 单引号内部的双引号无需转义

# ── 错误 3：命令替换中引号错位 ──
# ❌ awk 内部的 $1 被 Shell 展开
awk "{print $1}" data.txt
# ✅
awk '{print $1}' data.txt     # 单引号保护
awk "{print \$1}" data.txt    # 或转义美元符

# ── 错误 4：heredoc 中意外的变量展开 ──
cat <<EOF
PATH=$PATH                    # PATH 被展开
EOF
# 如果不想展开：
cat <<'EOF'
PATH=$PATH                    # 原样输出
EOF

# ── 错误 5：$@ 和 $* 不加引号 ──
set -- "arg with spaces" "arg2"

# ❌ 不加引号：单词分割
for arg in $@; do echo "$arg"; done      # 输出 3 行
for arg in $*; do echo "$arg"; done      # 同上

# ✅ 加引号：保持参数完整性
for arg in "$@"; do echo "$arg"; done    # 输出 2 行（推荐）
for arg in "$*"; do echo "$arg"; done    # 输出 1 行（所有参数合并为一个字符串）
```

## 3.5 单引号 vs 双引号速查

| 引号类型 | 变量展开 | 命令替换 | 转义字符 | 使用场景 |
|----------|----------|----------|----------|----------|
| `" "` 双引号 | 是 | 是 | `\$` `\"` `\\` `` \` `` `\n` | 绝大部分场景 |
| `' '` 单引号 | 否 | 否 | 无（所有字符字面值） | 字面值字符串、awk/sed 脚本 |
| `$' '` ANSI-C | 是 | 是 | `\n` `\t` `\e` 等 | 需要转义序列时 |

```bash
# ANSI-C quoting 示例
tab_separated=$'col1\tcol2\tcol3'
red_text=$'\e[31mError\e[0m'
newline=$'line1\nline2'
```

# 4. 函数设计原则

## 4.1 单一职责原则

> 每个函数只做一件事，并把它做好。

```bash
# ❌ 违反单一职责：一个函数做太多事
deploy_application() {
    # 检查依赖
    command -v docker >/dev/null || exit 1
    command -v kubectl >/dev/null || exit 1
    # 构建镜像
    docker build -t app:latest .
    # 推送镜像
    docker push registry.example.com/app:latest
    # 部署
    kubectl apply -f deployment.yaml
    # 发送通知
    curl -X POST "$SLACK_WEBHOOK" -d '{"text":"部署完成"}'
}

# ✅ 拆分后：每个函数只做一件事
check_dependencies() {
    local -ra required=(docker kubectl jq)
    for cmd in "${required[@]}"; do
        command -v "$cmd" >/dev/null || {
            echo "错误: 缺少依赖 $cmd" >&2
            return 1
        }
    done
}

build_image() {
    local tag="$1"
    docker build -t "$tag" . || return $?
}

push_image() {
    local tag="$1"
    docker push "$tag" || return $?
}

deploy_to_cluster() {
    local manifest="$1"
    kubectl apply -f "$manifest" || return $?
}

send_notification() {
    local message="${1:-部署完成}"
    curl -s -X POST "$SLACK_WEBHOOK" \
        -H 'Content-Type: application/json' \
        -d "{\"text\":\"$message\"}" >/dev/null
}

# 编排函数：将各步骤组合起来
deploy_application() {
    local tag="$1"
    check_dependencies        || return $?
    build_image "$tag"        || return $?
    push_image "$tag"         || return $?
    deploy_to_cluster         || return $?
    send_notification "部署完成: $tag"
}
```

## 4.2 返回值规范

```bash
# ── 函数返回值约定 ──

# 1. 使用 return 返回退出码（0 = 成功，非 0 = 失败）
is_port_open() {
    local host="$1"
    local port="$2"
    timeout 2 bash -c "echo >/dev/tcp/$host/$port" 2>/dev/null
}

if is_port_open "localhost" 8080; then
    echo "端口已开放"
fi

# 2. 通过 stdout 返回数据（echo / printf）
get_container_id() {
    local name="$1"
    docker ps -q --filter "name=$name"
}

container_id="$(get_container_id "web_app")"

# 3. 通过全局变量返回多个值（谨慎使用）
parse_url() {
    local url="$1"
    # 使用 nameref (bash 4.3+) 或 eval
    local -n _scheme=$2
    local -n _host=$3
    _scheme="${url%%://*}"
    _host="${url#*://}"
    _host="${_host%%/*}"
}

parse_url "https://example.com/path" scheme host
echo "$scheme"   # https
echo "$host"     # example.com

# 4. 使用 nameref 返回复杂结果（bash 4.3+）
get_file_stats() {
    local file="$1"
    local -n _result=$2
    IFS=':' read -r size mtime <<< "$(stat -c '%s:%Y' "$file" 2>/dev/null)"
    _result=([size]="$size" [mtime]="$mtime")
}

declare -A stats
get_file_stats "/etc/hosts" stats
echo "${stats[size]}"
```

**返回值语义约定：**

| 退出码 | 含义 | 使用场景 |
|--------|------|----------|
| 0 | 成功 | 正常完成 |
| 1 | 一般错误 | 未分类的错误 |
| 2 | 误用 | 参数错误、语法错误 |
| 126 | 不可执行 | 命令找到但无执行权限 |
| 127 | 未找到 | 命令不存在 |
| 128+N | 被信号 N 终止 | SIGINT(130)、SIGTERM(143) |

```bash
# 自定义退出码范围（建议在 3-63 之间）
readonly ERR_INVALID_ARGS=3
readonly ERR_FILE_NOT_FOUND=4
readonly ERR_PERMISSION_DENIED=5
readonly ERR_NETWORK_FAILURE=6
readonly ERR_DEPENDENCY_MISSING=7
readonly ERR_CONFIG_ERROR=8
readonly ERR_TIMEOUT=9
```

## 4.3 函数参数处理

```bash
# ── 函数参数最佳实践 ──

# 1. 参数数量检查
copy_file() {
    if (( $# != 2 )); then
        echo "用法: copy_file <source> <dest>" >&2
        return 2
    fi
    local source="$1"
    local dest="$2"
    cp "$source" "$dest"
}

# 2. 参数类型检查
divide() {
    local dividend="$1"
    local divisor="$2"

    # 检查是否为数值
    [[ "$dividend" =~ ^-?[0-9]+\.?[0-9]*$ ]] || {
        echo "错误: 参数1必须是数值" >&2
        return 2
    }
    [[ "$divisor" =~ ^-?[0-9]+\.?[0-9]*$ ]] || {
        echo "错误: 参数2必须是数值" >&2
        return 2
    }
    # 除零检查
    (( divisor == 0 )) && { echo "错误: 除数不能为零" >&2; return 2; }

    bc <<< "scale=6; $dividend / $divisor"
}

# 3. 默认参数值
log_message() {
    local level="${1:-INFO}"           # 第1参数默认 INFO
    local message="${2:-}"            # 第2参数默认空

    if [[ -z "$message" ]]; then
        echo "错误: 日志消息不能为空" >&2
        return 2
    fi
    printf '[%s] [%-5s] %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$level" "$message"
}

# 4. 命名参数模式（适合参数较多时）
deploy_app() {
    local env="staging"
    local tag="latest"
    local dry_run="false"
    local force="false"

    while (( $# > 0 )); do
        case "$1" in
            --env)      env="$2";      shift 2 ;;
            --tag)      tag="$2";      shift 2 ;;
            --dry-run)  dry_run="true"; shift   ;;
            --force)    force="true";   shift   ;;
            --)         shift; break ;;
            -*)         echo "未知选项: $1" >&2; return 2 ;;
            *)          break ;;
        esac
    done

    echo "部署到 $env 环境，标签: $tag"
    [[ "$dry_run" == "true" ]] && echo "（演习模式）"
}
```

## 4.4 函数长度控制

```bash
# ── 函数长度建议 ──
# - 理想：< 20 行
# - 可接受：< 50 行
# - 警告：> 50 行时应考虑拆分
# - 红线：> 100 行必须拆分

# 如果函数超过 50 行，检查：
# 1. 是否包含可以提取为独立函数的逻辑块？
# 2. 是否有多个不同的抽象层级？
# 3. 是否有重复的代码模式？
```

# 5. 全局变量管理

## 5.1 readonly 保护关键变量

```bash
#!/bin/bash

# ── 使用 readonly 防止意外修改 ──
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
readonly CONFIG_FILE="${SCRIPT_DIR}/config.conf"

# 数组也可以设为只读
readonly -a SUPPORTED_PLATFORMS=(linux darwin windows)
readonly -A DEFAULT_PORTS=(
    [http]=80
    [https]=443
    [ssh]=22
    [mysql]=3306
)

# 尝试修改会报错
# CONFIG_FILE="/new/path"    # bash: CONFIG_FILE: readonly variable
```

## 5.2 local 隔离函数作用域

```bash
# ❌ 未使用 local 导致的变量泄漏
count_files() {
    count=0                     # 污染全局作用域！
    for f in "$1"/*; do
        ((count++))
    done
    echo "$count"
}

count=100                       # 全局变量
result="$(count_files "/tmp")"
echo "全局 count 现在是: $count" # 不再是 100！

# ✅ 使用 local 隔离
count_files() {
    local count=0               # 局部变量，不影响外部
    local f
    for f in "$1"/*; do
        ((count++))
    done
    echo "$count"
}

# ✅ 批量声明 local
process_data() {
    local input_file output_dir temp_file
    local -i line_count=0       # 类型化整数
    local -a file_list=()       # 类型化数组
    local -A config=()          # 类型化关联数组
}
```

**local 的使用约束：**

```bash
# 注意：local 声明与赋值在同一行时的退出码问题
some_function() {
    local result
    result="$(some_command)"       # ✅ $? 来自 some_command
    # vs
    local result="$(some_command)" # ❌ $? 总是 0（local 总是成功）
}

# 更好的模式（需要捕获退出码时）
some_function() {
    local result
    result="$(some_command)" && {
        echo "命令成功: $result"
    } || {
        echo "命令失败，退出码: $?" >&2
        return 1
    }
}
```

## 5.3 命名冲突避免策略

```bash
#!/bin/bash

# ── 策略 1：前缀命名空间 ──
# 为全局变量使用项目/脚本特定的前缀
readonly MYAPP_CONFIG_DIR="${HOME}/.config/myapp"
readonly MYAPP_LOG_DIR="${HOME}/.local/share/myapp/logs"
readonly MYAPP_MAX_RETRIES=5

myapp_initialize() { ... }
myapp_cleanup() { ... }
myapp_deploy() { ... }

# ── 策略 2：命名空间函数封装 ──
# 将相关变量和函数封装在命名空间函数中
MyApp::Config::load() {
    readonly CONFIG_FILE="${1:-}"
    readonly CONFIG_DIR="$(dirname "$CONFIG_FILE")"
}
MyApp::Config::get() {
    local key="$1"
    grep "^${key}=" "$CONFIG_FILE" 2>/dev/null | cut -d'=' -f2-
}
MyApp::Deploy::run() {
    local env="$1"
    MyApp::Config::load "configs/${env}.conf"
    # ...
}

# ── 策略 3：使用 subshell 隔离 ──
# 在子 Shell 中执行操作，不污染父 Shell
(
    # 此处的变量不影响外部
    local temp_dir="/tmp/$$"
    cd "$temp_dir" || exit 1
    # ...
)

# ── 策略 4：环境变量命名空间 ──
# 对外暴露的环境变量使用统一前缀
export MYAPP_ENV="production"
export MYAPP_LOG_LEVEL="info"
export MYAPP_DATABASE_URL="postgresql://localhost/app"
```

## 5.4 全局状态管理

```bash
#!/bin/bash

# ── 集中管理全局状态 ──

# 使用单一全局关联数组管理状态
declare -A APP_STATE=(
    [initialized]="false"
    [config_loaded]="false"
    [dry_run]="false"
    [verbose]="false"
    [exit_code]=0
)

set_state() {
    local key="$1"
    local value="$2"
    if [[ -v APP_STATE[$key] ]]; then
        APP_STATE[$key]="$value"
    else
        echo "警告: 未知状态键: $key" >&2
    fi
}

get_state() {
    local key="$1"
    echo "${APP_STATE[$key]:-}"
}

is_state() {
    local key="$1"
    local expected="$2"
    [[ "${APP_STATE[$key]:-}" == "$expected" ]]
}
```

# 6. 可移植性考量

## 6.1 POSIX 兼容性层级选择

```bash
# ── 三级可移植性策略 ──

# 层级 1：最大可移植性（POSIX sh）
#!/bin/sh
# 可用于任何 Unix-like 系统
# 禁用：[[ ]], arrays, <<<, source, (( )), ${!var}, function 关键字

# 层级 2：Bash 可移植性（bash 3.x+，跨平台）
#!/usr/bin/env bash
# 可用于 Linux/macOS/WSL/Git Bash
# 禁用：关联数组(bash 4+), nameref(bash 4.3+), fallthrough ;&

# 层级 3：现代 Bash（bash 4.3+，现代 Linux）
#!/usr/bin/env bash
# 需要 bash 4.3+
# 可用所有 bash 特性：关联数组, nameref, wait -n, ${var@Q} 等
```

## 6.2 跨平台常见陷阱

```bash
# ── 陷阱 1：echo -e 不可移植 ──
# ❌
echo -e "Line1\nLine2"       # 在 dash/sh 中输出 "-e Line1\nLine2"
# ✅
printf 'Line1\nLine2\n'      # 所有 Shell 一致

# ── 陷阱 2：readlink 不存在于 macOS ──
# ❌
canonical_path="$(readlink -f "$file")"
# ✅
canonical_path="$(cd "$(dirname "$file")" && pwd -P)/$(basename "$file")"

# ── 陷阱 3：sed -i 语法差异 ──
# ❌ macOS 必须提供备份后缀
sed -i 's/old/new/' file.txt        # macOS: sed: -i may not be used with stdin
# ✅
sed -i.bak 's/old/new/' file.txt && rm file.txt.bak   # macOS
sed -i 's/old/new/' file.txt                            # Linux

# ── 陷阱 4：date 命令差异 ──
# macOS date 不支持 -d
# ✅
if [[ "$(uname)" == "Darwin" ]]; then
    yesterday="$(date -v-1d '+%Y-%m-%d')"      # macOS
else
    yesterday="$(date -d 'yesterday' '+%Y-%m-%d')"  # Linux
fi

# ── 陷阱 5：mktemp 参数差异 ──
# ✅ 跨平台安全写法
temp_file="$(mktemp -t "${SCRIPT_NAME}.XXXXXX")"     # macOS/Linux 都支持
temp_dir="$(mktemp -d -t "${SCRIPT_NAME}.XXXXXX")"

# ── 陷阱 6：shopt 仅在 bash 可用 ──
# ❌ 在 #!/bin/sh 中
shopt -s globstar
# ✅ bash 检测
if [[ -n "${BASH_VERSION:-}" ]]; then
    shopt -s globstar 2>/dev/null || true
    shopt -s nullglob 2>/dev/null || true
fi
```

## 6.3 可移植性工具选择

```bash
# ── 选择跨平台一致的工具 ──

# 字符串处理：优先使用 Shell 内置展开，其次 awk
str="hello/world/test.txt"
basename="${str##*/}"          # 内置展开（最快、最可移植）
dirname="${str%/*}"
extension="${str##*.}"

# 避免：basename / dirname 外部命令（多一次 fork）
# 在路径包含换行符等极端情况时仍应使用外部命令

# ── 跨平台安全的外部命令 ──
# ✅ 高度可移植
awk '{print $1}'              # POSIX awk
tr '[:lower:]' '[:upper:]'    # POSIX tr
grep -c 'pattern' file        # POSIX grep
sort -u                        # POSIX sort

# ⚠️ 需谨慎
grep -P                        # PCRE（-P 不是 POSIX）
sed -E                         # macOS: -E, Linux: -r, POSIX: 无扩展正则
cut -b                         # 大多可用，但建议用 awk 替代

# ── 检测操作系统并适配 ──
detect_os() {
    local os
    case "$(uname -s)" in
        Linux*)     os="linux" ;;
        Darwin*)    os="macos" ;;
        CYGWIN*|MINGW*|MSYS*) os="windows" ;;
        *)          os="unknown" ;;
    esac
    echo "$os"
}

OS="$(detect_os)"

get_cpu_count() {
    case "$OS" in
        linux)   nproc ;;
        macos)   sysctl -n hw.ncpu ;;
        windows) echo "$NUMBER_OF_PROCESSORS" ;;
        *)       echo 1 ;;
    esac
}
```

## 6.4 使用 ShellCheck 检测可移植性问题

```bash
# 安装 ShellCheck
# macOS: brew install shellcheck
# Linux: apt install shellcheck / yum install ShellCheck

# 针对 sh 的检查（检测 bashisms）
shellcheck -s sh script.sh

# 针对 bash 的检查
shellcheck -s bash script.sh

# 针对 dash 的检查
shellcheck -s dash script.sh

# 在脚本中内联控制检查规则
# shellcheck disable=SC1090,SC1091  # 禁用 source 外部文件警告
# shellcheck disable=SC2034         # 禁用未使用变量警告
```

# 7. 代码组织

## 7.1 模块化与 source 拆分

```bash
# ── 主脚本结构 ──
#!/usr/bin/env bash
set -euo pipefail

# 解析库路径（相对于脚本自身）
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly LIB_DIR="${SCRIPT_DIR}/lib"

# 按依赖顺序加载库
source "${LIB_DIR}/logging.sh"    || exit 1
source "${LIB_DIR}/utils.sh"      || exit 1
source "${LIB_DIR}/config.sh"     || exit 1
source "${LIB_DIR}/validation.sh" || exit 1

# 加载后验证关键函数是否存在
for func in log_info log_error validate_config main; do
    if ! declare -F "$func" >/dev/null 2>&1; then
        echo "错误: 缺少必要函数 $func" >&2
        exit 1
    fi
done

main "$@"
```

**库文件范例（lib/logging.sh）：**

```bash
# lib/logging.sh - 日志功能模块
#
# 提供统一的日志输出函数，支持日志级别、颜色输出和日志文件写入。
#
# 依赖环境变量:
#   LOG_LEVEL  - 日志级别 (DEBUG|INFO|WARN|ERROR)，默认 INFO
#   LOG_FILE   - 日志文件路径（可选），不设置则只输出到 stderr

# 防止重复加载
if [[ -n "${_LOGGING_SH_LOADED:-}" ]]; then
    return 0
fi
readonly _LOGGING_SH_LOADED="true"

# 检查依赖
if [[ -z "${SCRIPT_NAME:-}" ]]; then
    echo "警告: SCRIPT_NAME 未设置，logging.sh 应在主脚本之后加载" >&2
fi

# ── 日志级别定义 ──
readonly LOG_LEVEL_DEBUG=0
readonly LOG_LEVEL_INFO=1
readonly LOG_LEVEL_WARN=2
readonly LOG_LEVEL_ERROR=3

# 解析日志级别
_parse_log_level() {
    case "${LOG_LEVEL:-INFO}" in
        DEBUG) echo "$LOG_LEVEL_DEBUG" ;;
        INFO)  echo "$LOG_LEVEL_INFO"  ;;
        WARN)  echo "$LOG_LEVEL_WARN"  ;;
        ERROR) echo "$LOG_LEVEL_ERROR" ;;
        *)     echo "$LOG_LEVEL_INFO"  ;;
    esac
}
readonly _CURRENT_LOG_LEVEL="$(_parse_log_level)"

# ── 内部日志函数 ──
_log() {
    local level="$1"
    local level_num="$2"
    local color_code="$3"
    shift 3

    if (( level_num < _CURRENT_LOG_LEVEL )); then
        return 0
    fi

    local timestamp
    timestamp="$(date -u '+%Y-%m-%dT%H:%M:%SZ')"

    # 彩色输出到 stderr
    printf "\e[%sm[%s] [%-5s]\e[0m %s\n" \
        "$color_code" "$timestamp" "$level" "$*" >&2

    # 同时写入日志文件（如果指定）
    if [[ -n "${LOG_FILE:-}" ]]; then
        printf '[%s] [%-5s] %s\n' "$timestamp" "$level" "$*" >> "$LOG_FILE"
    fi
}

# ── 公开日志函数 ──
log_debug() { _log "DEBUG" "$LOG_LEVEL_DEBUG" "36" "$@"; }
log_info()  { _log "INFO"  "$LOG_LEVEL_INFO"  "32" "$@"; }
log_warn()  { _log "WARN"  "$LOG_LEVEL_WARN"  "33" "$@"; }
log_error() { _log "ERROR" "$LOG_LEVEL_ERROR" "31" "$@"; }

# 致命错误（打印后退出）
log_fatal() {
    _log "FATAL" "$LOG_LEVEL_ERROR" "31" "$@"
    exit 1
}
```

## 7.2 目录结构最佳实践

```
my-shell-project/
│
├── README.md                    # 项目说明、快速开始
├── Makefile                     # 常用任务入口（install, test, lint）
├── .editorconfig                # 编辑器配置（缩进等）
├── .gitignore                   # Git 忽略规则
├── .shellcheckrc                # ShellCheck 全局配置
├── .shellcheckrc                # ShellCheck 配置
│
├── bin/                         # 可执行入口脚本
│   ├── appctl                   # 主控制脚本
│   └── app-service              # 服务管理脚本
│
├── lib/                         # 共享函数库
│   ├── logging.sh               # 日志模块
│   ├── utils.sh                 # 通用工具函数
│   ├── config.sh                # 配置管理
│   ├── validation.sh            # 输入验证
│   ├── network.sh               # 网络相关
│   └── docker.sh                # Docker 操作封装
│
├── src/                         # 非共享的业务逻辑脚本
│   ├── deploy.sh                # 部署逻辑
│   ├── backup.sh                # 备份逻辑
│   └── migrate.sh               # 迁移逻辑
│
├── config/                      # 配置文件
│   ├── defaults.conf            # 默认配置
│   ├── development.conf         # 开发环境配置
│   ├── staging.conf             # 预发布环境配置
│   └── production.conf          # 生产环境配置
│
├── tests/                       # 测试
│   ├── test_logging.bats        # BATS 测试
│   ├── test_utils.bats
│   ├── test_config.bats
│   └── fixtures/                # 测试用的固定数据
│       ├── sample_config.conf
│       └── sample_data.csv
│
├── docs/                        # 文档
│   ├── usage.md                 # 使用说明
│   ├── api.md                   # 函数 API 文档
│   └── architecture.md          # 架构说明
│
└── scripts/                     # 辅助/开发用脚本
    ├── setup_dev_env.sh         # 开发环境初始化
    └── run_all_tests.sh         # 运行所有测试
```

## 7.3 主脚本组织模板

```bash
#!/usr/bin/env bash
#══════════════════════════════════════════════════════════════════
# 脚本模板 - 展示推荐的文件组织结构
#══════════════════════════════════════════════════════════════════

# ═══ 0. 安全和初始化 ═══
set -euo pipefail
IFS=$'\n\t'

# ═══ 1. 全局常量 ═══
readonly SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly LIB_DIR="${SCRIPT_DIR}/../lib"
readonly CONFIG_DIR="${SCRIPT_DIR}/../config"

# ═══ 2. 加载依赖库 ═══
source "${LIB_DIR}/logging.sh"
source "${LIB_DIR}/utils.sh"
source "${LIB_DIR}/config.sh"

# ═══ 3. 全局变量（非只读） ═══
VERBOSE=false
DRY_RUN=false
CONFIG_FILE=""

# ═══ 4. 函数定义（按调用顺序排列） ═══

# ── 4.1 帮助与文档函数 ──
usage() {
    cat <<EOF
用法: $SCRIPT_NAME [选项] <参数>

选项:
    -h, --help        显示帮助
    -v, --verbose     详细输出
    -n, --dry-run     演习模式
    -c, --config FILE 配置文件路径

示例:
    $SCRIPT_NAME -v -c config/production.conf
EOF
    exit "${1:-0}"
}

# ── 4.2 参数解析 ──
parse_args() {
    while (( $# > 0 )); do
        case "$1" in
            -h|--help)    usage 0 ;;
            -v|--verbose) VERBOSE=true; shift ;;
            -n|--dry-run) DRY_RUN=true; shift ;;
            -c|--config)  CONFIG_FILE="$2"; shift 2 ;;
            --)           shift; break ;;
            -*)           log_error "未知选项: $1"; usage 2 ;;
            *)            break ;;
        esac
    done

    if [[ -z "$CONFIG_FILE" ]]; then
        log_error "必须指定配置文件 (-c)"
        usage 2
    fi
}

# ── 4.3 前置检查 ──
check_prerequisites() {
    local -a missing=()
    for cmd in jq curl; do
        command -v "$cmd" >/dev/null 2>&1 || missing+=("$cmd")
    done
    if ((${#missing[@]} > 0)); then
        log_fatal "缺少依赖: ${missing[*]}"
    fi
}

# ── 4.4 核心业务逻辑 ──
do_main_work() {
    log_info "开始处理..."

    if [[ "$DRY_RUN" == "true" ]]; then
        log_warn "演习模式 - 不会执行实际操作"
        return 0
    fi

    # 实际业务逻辑
    # ...

    log_info "处理完成"
}

# ── 4.5 清理函数 ──
cleanup() {
    local exit_code=$?
    # 清理临时文件等
    if [[ -n "${temp_dir:-}" && -d "$temp_dir" ]]; then
        rm -rf "$temp_dir"
    fi
    if (( exit_code != 0 )); then
        log_error "脚本异常退出，退出码: $exit_code"
    fi
    exit $exit_code
}

# ═══ 5. 注册信号处理器 ═══
trap cleanup EXIT INT TERM

# ═══ 6. 主函数 ═══
main() {
    parse_args "$@"
    check_prerequisites
    load_config "$CONFIG_FILE"
    do_main_work
}

# ═══ 7. 入口 ═══
main "$@"
```

# 8. 文档化

## 8.1 注释规范

```bash
#!/usr/bin/env bash

# ── 注释的四个层级 ──

# 层级 1：文件头注释（见 1.1 节）

# 层级 2：函数注释（每个公开函数必须）
#
# 函数名: validate_ip_address
# 描述:   验证字符串是否为合法 IPv4 地址
# 参数:
#   $1 - 待验证的 IP 地址字符串
# 返回值:
#   0 - 是合法 IPv4 地址
#   1 - 不是合法 IPv4 地址
# 示例:
#   if validate_ip_address "192.168.1.1"; then
#       echo "合法"
#   fi
validate_ip_address() {
    local ip="$1"
    local IFS=.
    local -a octets
    read -r -a octets <<< "$ip"

    (( ${#octets[@]} == 4 )) || return 1
    for octet in "${octets[@]}"; do
        [[ "$octet" =~ ^[0-9]+$ ]] || return 1
        (( octet >= 0 && octet <= 255 )) || return 1
    done
    return 0
}

# 层级 3：代码块注释（解释"为什么"，而不是"是什么"）
process_files() {
    # macOS 和 Linux 的 stat 语法不同，统一使用 find 获取文件信息
    # find 的 -printf 在 macOS 不支持，因此用 -exec stat 替代
    find "$1" -type f -exec stat -f '%z %N' {} \;
}

# 层级 4：行内注释（仅用于非显而易见的逻辑）
calculate_hash() {
    local data="$1"
    # 先做一次 MD5 再做 SHA256，因为上游 API 要求这种双重哈希格式
    local md5_hash
    md5_hash="$(echo -n "$data" | md5sum | cut -d' ' -f1)"
    echo -n "$md5_hash" | sha256sum | cut -d' ' -f1
}
```

**注释的反模式：**

```bash
# ❌ 注释"是什么"——代码已经说明了
i=$((i + 1))          # i 增加 1

# ❌ 过时的注释
# 使用 HTTP API（已过时，现在用 gRPC）
curl -X POST https://api.example.com/v1/data

# ❌ 注释掉的代码（应删除，Git 有历史记录）
# old_backup_method
# rsync -av /data/ /backup/

# ❌ TODO 没有责任人和日期
# TODO: fix this

# ✅ 有用的 TODO
# TODO(zhangsan): 2026-06-01 前将文件传输改为流式，解决大文件 OOM 问题 (ISSUE-42)
```

## 8.2 help 和 usage 函数

```bash
#!/usr/bin/env bash

# ── 基础 usage 函数 ──
usage() {
    cat >&2 <<EOF
用法: $(basename "$0") [选项] <源文件> [目标路径]

选项:
    -h                显示此帮助信息
    -v                详细输出
    -f                强制执行（覆盖已存在文件）
    -o <目录>         指定输出目录
    -t <线程数>       并发线程数（默认: 4）

参数:
    <源文件>           要处理的文件路径
    [目标路径]         输出路径（默认: 当前目录）

示例:
    $(basename "$0") -v -o /tmp/output data.csv
    $(basename "$0") -f -t 8 large_file.txt

退出码:
    0   成功
    1   一般错误
    2   参数错误
EOF
    exit "${1:-2}"
}

# ── 增强 help 函数（支持长选项和更多细节） ──
print_help() {
    cat <<EOF
════════════════════════════════════════════════════════════════
  $(basename "$0") v${VERSION:-1.0.0} - ${DESCRIPTION:-文件处理工具}
════════════════════════════════════════════════════════════════

用法: $(basename "$0") [全局选项] <命令> [命令选项] [参数]

全局选项:
    -h, --help              显示此帮助信息
        --version           显示版本信息
    -q, --quiet             静默模式（只输出错误）
    -v, --verbose           详细输出
        --log-file <文件>    日志输出到文件

命令:
    convert    文件格式转换
    validate   验证文件格式
    batch      批量处理文件

运行 '$(basename "$0") <命令> --help' 获取子命令的帮助。

环境变量:
    LOG_LEVEL              日志级别 (DEBUG|INFO|WARN|ERROR)
    HTTP_PROXY             HTTP 代理地址
    MAX_RETRIES            最大重试次数 (默认: 3)

配置文件:
    ~/.config/$(basename "$0")/config.conf

项目主页: https://github.com/example/tool
报告问题: https://github.com/example/tool/issues
EOF
    exit "${1:-0}"
}
```

## 8.3 错误消息规范

```bash
# ── 结构化错误消息 ──
#
# 格式: [脚本名] [ERROR|WARN|INFO] 简短描述
#        原因: xxx
#        建议: xxx

die() {
    local code="${2:-1}"
    echo "[$(basename "$0")] 致命错误: $1" >&2
    echo "  脚本中止，退出码: $code" >&2
    exit "$code"
}

error_with_hint() {
    local message="$1"
    local reason="$2"
    local suggestion="$3"

    cat >&2 <<EOF
[$(basename "$0")] 错误: $message
  原因: $reason
  建议: $suggestion
EOF
}

# 使用示例
if [[ ! -f "$config_file" ]]; then
    error_with_hint \
        "配置文件不存在" \
        "找不到文件: $config_file" \
        "运行 '$(basename "$0") --init' 初始化配置文件"
    exit 4
fi
```

# 9. 版本控制

## 9.1 Git 友好的脚本实践

```bash
#!/usr/bin/env bash
# ── Git 友好设计原则 ──

# 1. 一行一条语句（便于 diff 阅读）
# ❌ 多条语句在一行
if [[ -f "$file" ]]; then process "$file"; else echo "missing"; fi

# ✅ 分行书写
if [[ -f "$file" ]]; then
    process "$file"
else
    echo "missing" >&2
fi

# 2. 尾随逗号/反斜杠便于添加参数
docker run \
    --name myapp \
    --restart unless-stopped \
    -p 8080:8080 \
    -v /data:/app/data \
    myapp:latest

# 3. 使用 Here Document 替代长字符串拼接
# ❌
json="{\"name\":\"${name}\",\"age\":${age},\"city\":\"${city}\"}"
# ✅
json=$(cat <<EOF
{
    "name": "${name}",
    "age": ${age},
    "city": "${city}"
}
EOF
)

# 4. 避免硬编码敏感信息
# ❌
API_KEY="sk-1234567890abcdef"
# ✅
API_KEY="${MYAPP_API_KEY:-}"
if [[ -z "$API_KEY" ]]; then
    echo "错误: 请设置环境变量 MYAPP_API_KEY" >&2
    exit 1
fi

# 5. 版本信息可查询
readonly VERSION="1.2.3"
readonly GIT_HASH="$(git -C "$SCRIPT_DIR" rev-parse --short HEAD 2>/dev/null || echo 'unknown')"
```

## 9.2 .gitignore 模板

```gitignore
# ── Shell 项目 .gitignore ──

# 临时文件
*.swp
*.swo
*~
.*.swp
*.bak

# 日志文件
*.log
logs/

# 编辑器配置（团队共享的保留）
.vscode/
.idea/
*.sublime-workspace

# macOS
.DS_Store
.AppleDouble

# 敏感配置文件（包含密钥/密码）
*.env
.env.*
!env.example
config/production.conf
credentials/

# 构建产物
dist/
build/
*.tar.gz
*.zip

# 依赖
node_modules/
vendor/

# 测试覆盖率报告
coverage/
*.lcov

# 备份
*.backup
backup/

# Python 字节码（当项目同时包含 Python 辅助脚本时）
__pycache__/
*.pyc
```

## 9.3 版本号与 CHANGELOG

```bash
#!/usr/bin/env bash
# ── 语义化版本号管理 ──

# 在脚本中内置版本信息
readonly VERSION="2.3.1"

# 提供 --version 选项
case "${1:-}" in
    --version|-V)
        echo "${SCRIPT_NAME} version ${VERSION}"
        echo "Git: ${GIT_HASH:-unknown}"
        echo "Bash: ${BASH_VERSION}"
        exit 0
        ;;
esac

# ── 版本比较函数 ──
# 返回值: 0=相等 1=左>右 2=左<右
compare_versions() {
    if [[ "$1" == "$2" ]]; then
        return 0
    fi
    local IFS=.
    local -a ver1=($1) ver2=($2)
    for ((i=0; i<${#ver1[@]} || i<${#ver2[@]}; i++)); do
        local v1="${ver1[$i]:-0}"
        local v2="${ver2[$i]:-0}"
        if (( v1 > v2 )); then
            return 1
        elif (( v1 < v2 )); then
            return 2
        fi
    done
    return 0
}

check_bash_version() {
    local required="$1"
    compare_versions "${BASH_VERSION%%.*}.${BASH_VERSION#*.*.*}" "$required"
    if [[ $? -eq 2 ]]; then
        echo "错误: 需要 bash >= ${required}，当前: ${BASH_VERSION}" >&2
        exit 1
    fi
}
```

# 10. Google Shell Style Guide 精要

## 10.1 核心规则摘要

Google Shell Style Guide 是业界最广泛采纳的 Shell 编码规范之一。以下是精要提炼：

**文件规范：**

| 规则 | 说明 |
|------|------|
| 文件扩展名 | 可执行脚本无后缀，库文件用 `.sh` |
| SUID/SGID | 禁止使用（安全风险） |
| 文件长度 | 尽量控制在 200 行以内 |
| 编码 | 统一 UTF-8，LF 换行符 |

**命名：**

```bash
# ✅ 函数名：小写 + 下划线
my_function_name() { ... }

# ✅ 变量名：小写 + 下划线
local my_variable="value"

# ✅ 常量与环境变量：全大写
readonly MAX_SIZE=1024
export DATABASE_HOST="localhost"

# ❌ 不要使用驼峰
# myFunctionName  MyVariable
```

**何时使用 Shell（Google 的建议）：**

> Shell 只应用于小型脚本或简单的包装器。如果一个脚本的性能至关重要，或者包含任何复杂逻辑，请使用 Python、Go 或其他更合适的语言。

**Google 的特殊要求：**

```bash
# 1. 使用 [[ ]] 而不是 [ ]（Google 明确要求 bash，不追求 POSIX sh）
if [[ "$name" == "admin" ]]; then
    echo "welcome"
fi

# 2. 避免使用 eval
# ❌ eval "$command"
# ✅ 使用数组
args=("command" "--flag" "$value")
"${args[@]}"

# 3. 管道中使用 while read 替代 for 循环
# ❌
for line in $(cat file.txt); do
    process "$line"
done
# ✅
while IFS= read -r line; do
    process "$line"
done < file.txt

# 4. 使用 $() 代替反引号
# ❌ result=`command`
# ✅ result="$(command)"

# 5. 使用 [[ ... ]] 进行字符串/文件测试，(( ... )) 进行算术测试
# ❌ if [ $x -gt 5 ]
# ✅ if (( x > 5 ))
# ✅ if [[ $str == pattern ]]

# 6. 测试 [[ -z $var ]] vs [[ -n $var ]]
if [[ -z "$var" ]]; then echo "空"; fi
if [[ -n "$var" ]]; then echo "非空"; fi

# 7. 文件名通配符优先于 find（当目录层级较浅时）
# ✅
for file in /path/to/*.txt; do
    [[ -f "$file" ]] || continue
    process "$file"
done
```

## 10.2 Google 风格的主函数模式

```bash
#!/usr/bin/env bash
#
# 符合 Google Shell Style Guide 的脚本模板

set -euo pipefail

# 全局常量
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

#######################################
# 打印用法信息并退出
# 参数:
#   退出码（默认 0）
#######################################
usage() {
    cat <<EOF
用法: ${SCRIPT_NAME} [选项]

选项:
  -h, --help    显示帮助
  -v, --version 显示版本
EOF
    exit "${1:-0}"
}

#######################################
# 主函数
# 参数:
#   命令行参数列表
#######################################
main() {
    local args=("$@")

    # 参数解析
    while ((${#args[@]} > 0)); do
        case "${args[0]}" in
            -h|--help)    usage 0 ;;
            -v|--version) echo "1.0.0"; exit 0 ;;
            --)           args=("${args[@]:1}"); break ;;
            -*)           echo "未知选项: ${args[0]}" >&2; usage 2 ;;
            *)            break ;;
        esac
        args=("${args[@]:1}")
    done

    # 业务逻辑
    echo "Hello, World!"
}

main "$@"
```

## 10.3 Google 的陷阱避免清单

```bash
# ── Google 风格指南中强调的常见陷阱 ──

# 1. 赋值中等号两边不要有空格
# ❌ var = "value"
# ✅ var="value"

# 2. 使用 "$@" 而不是 $@（即使 Google 内部指南也有此强调）
# ❌ for arg in $@; do
# ✅ for arg in "$@"; do

# 3. 使用 mapfile / readarray 读取命令输出（bash 4+）
# ❌ lines=($(some_command))
# ✅
mapfile -t lines < <(some_command)

# 4. 检查命令是否存在
# ✅
if ! command -v openssl >/dev/null 2>&1; then
    echo "需要安装 openssl" >&2
    exit 1
fi

# 5. 不严格依赖 set -e
# 在关键命令后显式检查退出码
docker build -t myapp .
if (( $? != 0 )); then
    echo "Docker 构建失败" >&2
    exit 1
fi
# 或使用：
docker build -t myapp . || { echo "构建失败" >&2; exit 1; }
```

# 11. 代码审查清单

## 11.1 审查维度

代码审查时，从以下 7 个维度系统性检查 Shell 脚本：

```
┌─────────────────────────────────────────────────────────────────┐
│                     Shell 脚本代码审查清单                         │
├──────────────┬──────────────────────────────────────────────────┤
│  1. 正确性   │ 逻辑是否正确？边界条件是否处理？                    │
│  2. 安全性   │ 是否存在注入风险？敏感信息是否泄露？                │
│  3. 健壮性   │ 错误是否被正确处理？失败是否快速终止？              │
│  4. 可移植性 │ 是否能在目标平台运行？有无 bashisms？               │
│  5. 可读性   │ 命名是否清晰？注释是否充分？                        │
│  6. 性能     │ 是否不必要的 fork？循环内是否有重操作？             │
│  7. 可维护性 │ 模块化程度？测试覆盖？                              │
└──────────────┴──────────────────────────────────────────────────┘
```

## 11.2 详细检查项

```bash
# ═════════════════════════════════════════════════════════════════
# 检查项 1：正确性
# ═════════════════════════════════════════════════════════════════

# [ ] 退出码检查是否正确？
if command; then                      # ✅ 隐式检查退出码
    echo "成功"
fi

if [[ $? -eq 0 ]]; then              # ❌ 多余的 $? 检查（上一条命令会覆盖 $?）
    echo "成功"
fi

# [ ] 字符串比较使用了正确操作符？
[[ "$a" == "$b" ]]                    # ✅ 字符串比较用 ==
(( a == b ))                          # ✅ 数字比较用 (( ))
[[ "$a" -eq "$b" ]]                   # ❌ -eq 用于数字（不报错但语义错误）

# [ ] 变量是否在双引号内？
# [ ] 管道失败是否被处理？（pipefail？）
# [ ] 循环中是否误用了 $?（每次迭代都会改变）？

# ═════════════════════════════════════════════════════════════════
# 检查项 2：安全性
# ═════════════════════════════════════════════════════════════════

# [ ] 用户输入是否正确转义？
filename="${1}"                       # ❌ 未验证直接使用
# ✅
filename="${1}"
[[ "$filename" =~ ^[a-zA-Z0-9._-]+$ ]] || { echo "非法文件名" >&2; exit 1; }

# [ ] 是否使用了 eval？（几乎总是有替代方案）
# ❌ eval "$user_input"
# ✅ 使用数组、间接引用、case 等替代

# [ ] 是否硬编码了密钥/密码？
# [ ] 临时文件是否使用安全的方式创建？（mktemp）
# [ ] 是否检查了 setuid/setgid 脚本？

# ═════════════════════════════════════════════════════════════════
# 检查项 3：健壮性
# ═════════════════════════════════════════════════════════════════

# [ ] 是否使用 set -euo pipefail？
# [ ] 是否注册了 trap 处理异常退出？
# [ ] 函数参数数量是否检查？
# [ ] 文件操作前是否检查文件存在性？
# [ ] 网络请求是否有超时设置？
# [ ] 是否处理了大文件/空文件/二进制文件的情况？

# ═════════════════════════════════════════════════════════════════
# 检查项 4：可移植性
# ═════════════════════════════════════════════════════════════════

# [ ] 目标 Shell 是什么？(bash? sh? ash?)
# [ ] 是否使用了 bashisms？(检查 [[ ]],数组,<<<,source)
# [ ] echo 是否该替换为 printf？
# [ ] 是否假设了 GNU 工具的行为？
# [ ] 路径分隔符是否跨平台？(使用 /，避免反斜杠)

# ═════════════════════════════════════════════════════════════════
# 检查项 5：可读性
# ═════════════════════════════════════════════════════════════════

# [ ] 变量/函数名是否表意清晰？
# [ ] 是否避免了魔法数字？（使用命名常量）
# [ ] 注释是否解释了"为什么"而不是"是什么"？
# [ ] 缩进是否一致？（推荐 4 空格）
# [ ] 长命令是否使用反斜杠折行？

# ═════════════════════════════════════════════════════════════════
# 检查项 6：性能
# ═════════════════════════════════════════════════════════════════

# [ ] 循环体内是否调用了外部命令？
# ❌
while IFS= read -r line; do
    echo "$line" | grep "pattern"      # 每次迭代一个子进程
    basename "$line"                    # 又一个子进程
done < file.txt

# ✅
while IFS= read -r line; do
    [[ "$line" == *pattern* ]]          # 零 fork
    echo "${line##*/}"                  # 零 fork
done < file.txt

# [ ] 是否可以用 Shell 内置功能替代外部命令？
# [ ] 大文件是否逐行处理而非一次性读入内存？
# [ ] 是否可以合并多次管道操作为一个？

# ═════════════════════════════════════════════════════════════════
# 检查项 7：可维护性
# ═════════════════════════════════════════════════════════════════

# [ ] 函数是否单一职责？
# [ ] 脚本长度是否合理（< 500 行）？
# [ ] 是否有单元测试？
# [ ] 是否通过了 ShellCheck 检查？
# [ ] 接口（环境变量、参数、退出码）是否有文档？
```

## 11.3 审查中使用的工具命令

```bash
# 审查时必须运行的工具

# 1. ShellCheck - 静态分析
shellcheck -s bash script.sh
shellcheck -s sh script.sh        # POSIX 模式
shellcheck -Calways script.sh     # 显示所有级别的警告

# 2. 检查脚本中的硬编码敏感信息
grep -nE '(password|secret|token|api.?key|credential)\s*=' script.sh

# 3. 检测 bashisms（如果目标 Shell 是 POSIX sh）
# 使用 checkbashisms (来自 devscripts 包)
checkbashisms script.sh

# 4. 统计脚本复杂度
# 函数数量
grep -c '^[a-zA-Z_][a-zA-Z0-9_]*()' script.sh
# 行数
wc -l script.sh
# 外部命令调用次数
grep -cE '\$\(|``' script.sh

# 5. 检查是否缺少引号（启发式）
# 变量引用中缺少双引号的高风险模式
grep -nE '\$[a-zA-Z_][a-zA-Z0-9_]*[^"]' script.sh | grep -v '^ *#'
```

# 12. 常见反模式

## 12.1 反模式 1：无引号的变量展开

```bash
# ❌ 反模式：最危险的写法
file=$1
if [ -f $file ]; then              # 文件名含空格时出错
    cat $file | while read line; do
        echo $line | grep $pattern # 变量全未引用
    done
fi

# ✅ 正确写法
file="$1"
if [[ -f "$file" ]]; then
    while IFS= read -r line; do
        printf '%s\n' "$line" | grep "$pattern"
    done < "$file"
fi
```

## 12.2 反模式 2：ls 命令解析

```bash
# ❌ 反模式：尝试解析 ls 的输出
for file in $(ls *.txt); do        # 文件名含空格/换行时断裂
    echo "Processing $file"
done

files=$(ls /path/to/dir)            # 同样的错误
echo "$files" | while read f; do
    process "$f"
done

# ✅ 方法 1：直接通配符
for file in *.txt; do
    [[ -f "$file" ]] || continue
    echo "Processing $file"
done

# ✅ 方法 2：find -print0 + while read（处理大量文件）
while IFS= read -r -d '' file; do
    echo "Processing $file"
done < <(find . -name "*.txt" -print0)

# ✅ 方法 3：mapfile（bash 4+）
mapfile -t files < <(find . -name "*.txt" -type f)
for file in "${files[@]}"; do
    echo "Processing $file"
done
```

## 12.3 反模式 3：cat 的误用（UUOC）

```bash
# ❌ 反模式：无谓使用 cat (Useless Use of Cat)
cat file.txt | grep "pattern"          # cat 多余
cat file.txt | while IFS= read -r line; do

# ✅ cat 的合理用法
# 1. 拼接多个文件
cat file1.txt file2.txt | process

# 2. 需要免除 sudo 权限
sudo cat /var/log/syslog | grep "error"

# 3. 消除管道中 stdin 的歧义（某些命令区分文件和 stdin 行为）
cat "$file" | some_command -           # 显式传递 stdin

# ✅ 替代写法
grep "pattern" file.txt                # 直接读取文件
while IFS= read -r line; do ... done < file.txt   # 输入重定向到循环
< file.txt grep "pattern"              # 重定向到命令
```

## 12.4 反模式 4：eval 滥用

```bash
# ❌ 反模式：使用 eval 构造动态命令
command="docker run --name $container_name $image"
eval "$command"                        # 危险！

# ✅ 方法 1：直接用数组存储命令
args=(docker run --name "$container_name" "$image")
"${args[@]}"

# ✅ 方法 2：使用函数替代动态命令
run_container() {
    local name="$1"
    local image="$2"
    docker run --name "$name" "$image"
}

# ✅ 方法 3：间接变量引用（代替 eval 取值）
# ❌ eval echo "\$$var_name"
# ✅ bash 4.3+ nameref
declare -n ref="$var_name"
echo "$ref"

# ✅ bash 3.x 间接引用
echo "${!var_name}"
```

## 12.5 反模式 5：使用 `$?` 而非直接判断

```bash
# ❌ 反模式：冗余的 $? 检查
command
if [[ $? -eq 0 ]]; then
    echo "成功"
else
    echo "失败"
fi

# ✅ 直接判断退出码
if command; then
    echo "成功"
else
    echo "失败"
fi

# ❌ $? 在管道后失效
true | grep -q "pattern"
echo "$?"           # 这是上一个 echo 的退出码！不是 grep 的！

# ✅ pipefail + 检查
set -o pipefail
true | grep -q "pattern"
exit_code=$?
if (( exit_code != 0 )); then
    echo "管道失败 (exit code: $exit_code)"
fi
```

## 12.6 反模式 6：在循环中拼接字符串

```bash
# ❌ 反模式：循环内用 += 拼接长字符串（每次重新分配内存）
output=""
for i in {1..10000}; do
    output+="line $i"$'\n'             # 大循环时性能极差
done

# ✅ 方法 1：使用临时文件
tmp="$(mktemp)"
for i in {1..10000}; do
    echo "line $i"
done > "$tmp"
output="$(<"$tmp")"
rm -f "$tmp"

# ✅ 方法 2：直接重定向到目标
{
    for i in {1..10000}; do
        echo "line $i"
    done
} > output.txt

# ✅ 方法 3：使用 awk 或 join 等专用工具
printf 'line %d\n' {1..10000} > output.txt
```

## 12.7 反模式 7：不安全的临时文件

```bash
# ❌ 反模式：使用可预测的文件名
tmp="/tmp/myapp_$$"                   # 竞态条件：符号链接攻击 (CVE!)
echo "data" > "$tmp"

# ❌ 反模式：手动拼接临时文件名
tmp="/tmp/myapp.$(date +%s)"          # 仍然可预测

# ✅ 使用 mktemp（原子操作，安全）
tmp_file="$(mktemp)"                   # 在 /tmp 创建
tmp_dir="$(mktemp -d)"                 # 创建临时目录

# ✅ 使用 trap 确保清理
cleanup() {
    rm -f "$tmp_file"
    rm -rf "$tmp_dir"
}
trap cleanup EXIT
tmp_file="$(mktemp)"
tmp_dir="$(mktemp -d)"

# ✅ 更安全的临时文件（指定模板名）
tmp_file="$(mktemp -t "myapp.XXXXXX")"
```

## 12.8 反模式 8：for 循环读取命令输出

```bash
# ❌ 反模式：for 循环对命令输出进行单词分割
for word in $(cat file.txt); do       # 按空白分割，不是逐行
    echo "$word"
done

# 更可怕的：
for line in $(find . -name "*.txt"); do  # 文件名含空格就断裂
    process "$line"
done

# ✅ while read 逐行处理
while IFS= read -r line; do
    echo "$line"
done < file.txt

# ✅ 需要保留首尾空白时
while IFS= read -r line; do
    echo "$line"
done < file.txt

# ✅ 使用 find 时
find . -name "*.txt" -print0 | while IFS= read -r -d '' path; do
    process "$path"
done
```

## 12.9 反模式 9：硬编码路径和值

```bash
# ❌ 反模式：硬编码
LOG_FILE="/var/log/myapp/access.log"   # 在不存在的系统上失败
TIMEOUT=30                             # 魔法数字
API_URL="http://192.168.1.100:8080"    # 环境变化时需修改代码

# ✅ 配置外化
readonly DEFAULT_LOG_DIR="${HOME}/.local/share/myapp/logs"
readonly LOG_FILE="${MYAPP_LOG_FILE:-${DEFAULT_LOG_DIR}/access.log}"

readonly DEFAULT_TIMEOUT=30
TIMEOUT="${MYAPP_TIMEOUT:-$DEFAULT_TIMEOUT}"

readonly API_URL="${MYAPP_API_URL:?必须设置 MYAPP_API_URL 环境变量}"
```

## 12.10 反模式 10: echo 调试

```bash
# ❌ 反模式：用 echo 代码位置调试
echo "DEBUG: 到这儿了"
echo "DEBUG: 变量值: $var"

# ✅ 方法 1：使用 set -x / set +x 控制范围
set -x                      # 开启追踪
# ... 需要调试的代码 ...
set +x                      # 关闭追踪

# ✅ 方法 2：使用 DEBUG trap
trap 'echo "[DEBUG] 行 $LINENO: $BASH_COMMAND"' DEBUG
# ... 代码 ...
trap - DEBUG

# ✅ 方法 3：使用正式的日志函数
debug() {
    if [[ "${DEBUG:-false}" == "true" ]]; then
        echo "[DEBUG] $(date -u +%T) ${FUNCNAME[1]:-main}:$BASH_LINENO: $*" >&2
    fi
}
```

# 13. Shell 脚本的单元测试

## 13.1 BATS (Bash Automated Testing System)

BATS 是目前最流行的 Shell 测试框架，提供类似 Mocha/Jest 的 BDD 风格语法。

**安装：**

```bash
# macOS
brew install bats-core

# Linux (apt)
sudo apt install bats

# npm (跨平台)
npm install -g bats

# 验证
bats --version
```

**基础测试示例：**

```bash
#!/usr/bin/env bats
# tests/test_utils.bats - 测试 lib/utils.sh 模块

# 加载被测文件
setup() {
    load '../lib/utils.sh'
}

# ── 基础断言 ──
@test "trim_string 去除首尾空白" {
    result="$(trim_string "  hello world  ")"
    [[ "$result" == "hello world" ]]
}

@test "is_valid_email 正确识别合法邮箱" {
    is_valid_email "user@example.com"
    is_valid_email "a.b@c.d.co"
}

@test "is_valid_email 拒绝非法邮箱" {
    ! is_valid_email "not-an-email"
    ! is_valid_email "@no-local-part.com"
    ! is_valid_email ""
}

@test "absolute_path 返回绝对路径" {
    result="$(absolute_path "foo/bar")"
    [[ "$result" == /* ]]
}

# ── 使用 run 辅助函数测试退出码和输出 ──
@test "validate_port 拒绝超出范围的端口" {
    run validate_port "99999"
    (( status != 0 ))
    [[ "$output" == *"无效"* ]]
}

@test "validate_port 接受合法端口" {
    run validate_port "8080"
    (( status == 0 ))
}

# ── 参数化测试（循环） ──
@test "to_lowercase 转换" {
    declare -A cases=(
        ["HELLO"]="hello"
        ["HelloWorld"]="helloworld"
        ["ABC123"]="abc123"
    )
    for input in "${!cases[@]}"; do
        expected="${cases[$input]}"
        run to_lowercase "$input"
        [[ "$output" == "$expected" ]]
    done
}
```

**高级 BATS 技巧：**

```bash
#!/usr/bin/env bats

# ── setup / teardown ──
setup() {
    # 每个测试前执行
    TEST_DIR="$(mktemp -d)"
    cd "$TEST_DIR"
    export APP_CONFIG_DIR="$TEST_DIR/config"
    mkdir -p "$APP_CONFIG_DIR"
}

teardown() {
    # 每个测试后执行
    rm -rf "$TEST_DIR"
}

# ── setup_file / teardown_file（BATS 1.7+） ──
setup_file() {
    # 所有测试只执行一次
    export TEST_ROOT="$(mktemp -d)"
}

teardown_file() {
    rm -rf "$TEST_ROOT"
}

# ── 测试跳过 ──
@test "需要 docker 环境的测试" {
    if ! command -v docker >/dev/null; then
        skip "Docker 未安装"
    fi
    # ... 测试逻辑 ...
}

# ── 辅助函数 ──
generate_temp_file() {
    local content="$1"
    local file
    file="$(mktemp)"
    echo "$content" > "$file"
    echo "$file"
}

# ── 使用 bats 内置的 assert（加载 bats-assert） ──
# bats_require_minimum_version 1.5.0
# setup() { load '../lib/utils.sh'; load '/usr/lib/bats/bats-support/load'; load '/usr/lib/bats/bats-assert/load'; }

# @test "assert 增强" {
#     run some_function "arg"
#     assert_success
#     assert_output "expected output"
# }
```

## 13.2 shunit2

shunit2 是另一种流行的 Shell 测试框架，语法更接近 xUnit（JUnit 风格）。

**安装：**

```bash
# 下载单个文件即可
wget -O /usr/local/bin/shunit2 \
    https://raw.githubusercontent.com/kward/shunit2/master/shunit2
chmod +x /usr/local/bin/shunit2
```

**基础使用：**

```bash
#!/usr/bin/env bash
# tests/test_config_shunit2.sh

# 加载被测函数
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
source "${SCRIPT_DIR}/lib/config.sh"

# ── Mock 设置 ──
readonly TEST_CONFIG_FILE="$(mktemp)"

setUp() {
    # 每个测试前的初始化
    cat > "$TEST_CONFIG_FILE" <<EOF
DATABASE_HOST=localhost
DATABASE_PORT=5432
APP_NAME=myapp
LOG_LEVEL=INFO
EOF
}

tearDown() {
    # 每个测试后的清理
    rm -f "$TEST_CONFIG_FILE"
}

# ── 测试用例（函数名以 test 开头） ──

testGetConfigValue() {
    load_config "$TEST_CONFIG_FILE"
    local value
    value="$(get_config_value "DATABASE_HOST")"
    assertEquals "数据库主机配置" "localhost" "$value"
}

testGetConfigValueDefault() {
    load_config "$TEST_CONFIG_FILE"
    local value
    value="$(get_config_value "NON_EXISTENT" "default_val")"
    assertEquals "默认值" "default_val" "$value"
}

testConfigValueWithSpaces() {
    echo 'APP_DESCRIPTION=My Super App' >> "$TEST_CONFIG_FILE"
    load_config "$TEST_CONFIG_FILE"
    local value
    value="$(get_config_value "APP_DESCRIPTION")"
    assertEquals "含空格的值" "My Super App" "$value"
}

testEmptyConfigValue() {
    echo 'EMPTY_KEY=' >> "$TEST_CONFIG_FILE"
    load_config "$TEST_CONFIG_FILE"
    local value
    value="$(get_config_value "EMPTY_KEY" "fallback")"
    assertEquals "空值应使用默认" "fallback" "$value"
}

testConfigFileNotFound() {
    local result
    result="$(load_config "/nonexistent/path" 2>&1)"
    assertNotEquals "应返回错误" "0" "$?"
}

# ── 运行测试 ──
source /usr/local/bin/shunit2
```

**shunit2 断言速查：**

```bash
# 等价断言
assertEquals "描述" "期望值" "$实际值"
assertNotEquals "描述" "非期望值" "$实际值"
assertSame "描述" "期望值" "$实际值"       # 严格比较 (同 assertEquals)
assertNotSame "描述" "非期望值" "$实际值"

# 布尔断言
assertTrue "描述" "$condition_result"
assertFalse "描述" "$condition_result"

# 空/非空断言
assertNull "描述" "$value"                  # 期望值为空字符串
assertNotNull "描述" "$value"               # 期望值非空

# 包含断言
assertContains "描述" "$haystack" "$needle"
assertNotContains "描述" "$haystack" "$needle"

# 失败/成功
fail "强制失败的原因"
```

## 13.3 测试设计原则

```bash
# ── Shell 脚本测试的 5 个原则 ──

# 原则 1：可测试性设计
# 将核心逻辑与副作用（文件IO、网络、外部命令）分离
#
# ❌ 不可测试的代码
main() {
    tar -czf backup.tar.gz /var/data
    curl -F "file=@backup.tar.gz" "https://api.example.com/upload?token=${TOKEN}"
}
#
# ✅ 可测试的代码
create_archive() {
    local src="$1" dest="$2"
    tar -czf "$dest" "$src"
}
upload_file() {
    local file="$1" url="$2" token="$3"
    curl -F "file=@${file}" "${url}?token=${token}"
}
main() {
    create_archive "/var/data" "backup.tar.gz" || return $?
    upload_file "backup.tar.gz" "$API_URL" "$TOKEN" || return $?
}

# 原则 2：依赖注入
# 通过环境变量注入外部依赖，方便测试时替换
readonly API_BASE="${MYAPP_API_BASE:-https://api.example.com}"
readonly CURL_OPTS="${MYAPP_CURL_OPTS:--s -f --max-time 30}"

call_api() {
    local endpoint="$1"
    curl $CURL_OPTS "${API_BASE}/${endpoint}"
}

# 测试时可以重定向到 mock 服务器：
# MYAPP_API_BASE="http://localhost:9999/mock"

# 原则 3：Mock 外部命令
# 利用 PATH 或函数重载进行 mock

# 方法 A：重载函数
# 在测试中:
docker() {
    if [[ "$1" == "ps" ]]; then
        echo "container-id-12345"
        return 0
    fi
    command docker "$@"      # 其他调用委托给真实命令
}

# 方法 B：临时 PATH 注入
# 创建 mock 脚本在测试专用 PATH 中

# 原则 4：数据驱动测试
@test "urlencode 编码测试" {
    local -A test_cases=(
        ["hello world"]="hello%20world"
        ["a&b=c"]="a%26b%3Dc"
        ["foo/bar"]="foo%2Fbar"
        ["空格"]="%E7%A9%BA%E6%A0%BC"
    )
    for input in "${!test_cases[@]}"; do
        expected="${test_cases[$input]}"
        run urlencode "$input"
        assert_output "$expected"
    done
}

# 原则 5：覆盖率统计
# 使用 kcov 或手动统计哪些函数被测试覆盖
# 在测试运行后检查：
# 哪些函数没有对应的测试用例？
# 所有的 if 分支是否都被覆盖？
```

## 13.4 CI/CD 集成

```bash
#!/usr/bin/env bash
# scripts/run_tests.sh - 在 CI 中运行所有测试

set -euo pipefail

readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
readonly PROJECT_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"

# 颜色输出
GREEN='\e[32m'
RED='\e[31m'
NC='\e[0m'

run_shellcheck() {
    echo "=== ShellCheck 静态分析 ==="
    local -a targets=()
    while IFS= read -r -d '' file; do
        targets+=("$file")
    done < <(find "$PROJECT_DIR" -name "*.sh" -print0)

    shellcheck -s bash "${targets[@]}" && {
        echo -e "${GREEN}✓ ShellCheck 通过${NC}"
    } || {
        echo -e "${RED}✗ ShellCheck 发现警告${NC}"
        return 1
    }
}

run_bats() {
    echo "=== BATS 单元测试 ==="
    bats --recursive "$PROJECT_DIR/tests/" && {
        echo -e "${GREEN}✓ 所有测试通过${NC}"
    } || {
        echo -e "${RED}✗ 存在测试失败${NC}"
        return 1
    }
}

check_style() {
    echo "=== 代码风格检查 ==="
    local issues=0
    # 检查是否使用了 echo -e
    if grep -rn 'echo\s\+-e' "$PROJECT_DIR"/{lib,src,bin} --include="*.sh" 2>/dev/null; then
        echo -e "${RED}✗ 发现 echo -e（应使用 printf）${NC}"
        issues=1
    fi
    # 检查是否缺少 shebang
    for file in "$PROJECT_DIR"/{bin,src,lib}/*.sh; do
        [[ -f "$file" ]] || continue
        if ! head -1 "$file" | grep -q '^#!'; then
            echo -e "${RED}✗ $file 缺少 shebang${NC}"
            issues=1
        fi
    done
    return "$issues"
}

# ── 主流程 ──
main() {
    local failed=0
    run_shellcheck   || ((failed++))
    check_style      || ((failed++))
    run_bats         || ((failed++))

    echo ""
    if (( failed == 0 )); then
        echo -e "${GREEN}全部检查通过 ✓${NC}"
    else
        echo -e "${RED}${failed} 项检查失败 ✗${NC}"
        exit 1
    fi
}

main

# ── GitHub Actions 配置示例 ──
# .github/workflows/shell-test.yml:
#
# name: Shell Tests
# on: [push, pull_request]
# jobs:
#   test:
#     runs-on: ubuntu-latest
#     steps:
#       - uses: actions/checkout@v4
#       - name: Install dependencies
#         run: |
#           sudo apt-get update
#           sudo apt-get install -y shellcheck bats
#       - name: Run tests
#         run: bash scripts/run_tests.sh
```

## 13.5 BATS vs shunit2 对比

| 特性 | BATS | shunit2 |
|------|------|---------|
| 语法风格 | BDD (describe/it) | xUnit (testXxx) |
| 安装方式 | 包管理器 / npm | 单文件下载 |
| 依赖 | TAP 输出解析器 | 纯 Shell |
| setup/teardown | 每个测试 | 每个测试 |
| 标签过滤 | `--filter` | 无内置 |
| 并行执行 | 支持 (GNU Parallel) | 不支持 |
| 社区活跃度 | 高（2024 年活跃） | 中（维护模式） |
| 适用场景 | 新项目、团队协作 | 简单项目、少量测试 |
| 推荐度 | ★★★★★ | ★★★ |

**选型建议：**

- 新项目首选 **BATS**：社区活跃、语法现代、CI 集成友好
- 已有 shunit2 的项目不强制迁移，两者可以共存
- 简单验证脚本（< 5 个测试）两者皆可，shunit2 更轻量
