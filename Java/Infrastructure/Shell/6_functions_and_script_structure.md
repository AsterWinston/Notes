
# 1. 函数定义

```bash
# 方式1（推荐，明确）
function_name() {
    命令
}

# 方式2（bash 专属，显式使用 function 关键字）
function function_name {
    命令
}

# 方式3（结合两者，可移植性最差）
function function_name() {
    命令
}
```

**方式1 是 POSIX 兼容且最清晰的写法。** 方式3 在某些非 Bash Shell 下会报错。

```bash
# 完整的函数示例
hello() {
    local name="${1:-World}"    # 第一个参数，默认值为 World
    echo "Hello, $name!"
}

hello "Alice"     # Hello, Alice!
hello              # Hello, World!
```

# 2. 函数参数

```bash
# 函数内部使用位置参数 $1 $2 ... 和 $@ $#
greet() {
    echo "参数个数: $#"
    echo "第一参数: $1"
    echo "所有参数: $@"
}

greet "Alice" 30 "New York"
# 参数个数: 3
# 第一参数: Alice
# 所有参数: Alice 30 New York
```

**重要：函数的 `$1` 和脚本的 `$1` 是独立的。** 函数调用时会创建一套新的位置参数，不会影响脚本层的 `$1`。

```bash
#!/bin/bash
# 脚本参数 vs 函数参数
print_args() {
    echo "函数内 \$1 = $1"
}

echo "脚本 \$1 = $1"      # 脚本的第一个参数
print_args "函数参数"       # 函数自己的参数
```

# 3. 返回值

## 3.1 退出状态码（0~255）

```bash
is_even() {
    if (( $1 % 2 == 0 )); then
        return 0    # 成功/真
    else
        return 1    # 失败/假
    fi
}

if is_even 10; then
    echo "10 是偶数"
fi
```

**`return` 只能返回 0~255。** 超过 255 的值会被截断（取模 256）。

## 3.2 返回字符串或数据

```bash
# 方法1：echo + 命令替换（最常用）
get_date() {
    date +%Y-%m-%d
}
today=$(get_date)

# 方法2：全局变量
result=""
get_uppercase() {
    result="${1^^}"
}
get_uppercase "hello"
echo "$result"    # HELLO

# 方法3：printf（比 echo 更精准）
get_file_size() {
    stat -c%s "$1" 2>/dev/null || printf "0"
}
size=$(get_file_size "file.txt")
```

## 3.3 返回值模式对比

| 方法 | 适合场景 | 缺点 |
|------|----------|------|
| `return N` | 成功/失败状态 | 只能返回 0~255 |
| `echo` / `printf` | 返回数据 | 输出混入调试信息时会出错 |
| 全局变量 | 返回复杂数据 | 耦合度高 |
| 文件描述符 | 返回大量数据 | 较复杂 |

# 4. 局部变量

```bash
func() {
    local a=10           # 局部变量（只能在函数内访问）
    local b c d          # 声明多个局部变量
    local name="$1"      # 用局部变量保存参数（推荐）
    
    echo "$a"            # 对外不可见
}

func
echo "${a:-未定义}"      # 未定义
```

**黄金法则：函数内变量必须加 `local`，否则就是全局变量。**

```bash
# 经典 bug 示例
count_up() {
    i=1       # bug! 没有 local，污染全局
    while [ $i -le 10 ]; do
        echo $i
        ((i++))
    done
}

for i in {1..5}; do
    count_up
    echo "loop: $i"     # 外层循环可能被破坏！
done
```

**`local` 的限制：** 只能在函数内使用，不能在脚本最外层使用。

# 5. 函数变量作用域

```bash
#!/bin/bash

global_var="我在全局"

outer() {
    local outer_var="我在外层函数"
    echo "outer: $global_var"
    echo "outer: $outer_var"

    inner() {
        local inner_var="我在内层函数"
        echo "inner: $global_var"    # 可以访问全局
        echo "inner: $outer_var"     # 可以访问外层局部（嵌套函数特性）
        echo "inner: $inner_var"     # 自己的局部
    }
    
    inner
    echo "还在 outer: ${inner_var:-不可访问}"  # 不可访问内层的局部
}

outer
echo "全局: ${outer_var:-不可访问}"    # 不可访问
```

**Bash 的动态作用域：** 嵌套函数可以访问外层函数的 `local` 变量（这点与大多数编程语言不同）。

# 6. 脚本参数解析

## 6.1 手动解析位置参数

```bash
#!/bin/bash
# 简单的位置参数解析
input_file="$1"
output_file="$2"

if [ $# -lt 2 ]; then
    echo "用法: $0 <输入文件> <输出文件>"
    exit 1
fi
```

## 6.2 getopts 标准参数解析

```bash
#!/bin/bash
# getopts 用于解析 -f -v 等单字母选项

usage() {
    echo "用法: $0 [-f file] [-v] [-h]"
    exit 1
}

verbose=0
file=""

while getopts "f:vh" opt; do
    case "$opt" in
        f) file="$OPTARG" ;;    # -f 需要参数（后面的 : 表示）
        v) verbose=1 ;;         # -v 是布尔标志
        h) usage ;;
        *) usage ;;
    esac
done

shift $((OPTIND - 1))       # 移除已解析的选项，$1 成为第一个非选项参数

echo "文件: $file"
echo "详细模式: $verbose"
echo "剩余参数: $@"
```

**getopts 速查：**
- `:` 后带 `:` 表示该选项需要参数（如 `f:`）
- `OPTARG`：当前选项的参数值
- `OPTIND`：下一个待处理参数的索引
- `shift $((OPTIND - 1))`：移除已处理的选项

## 6.3 长选项解析（手动实现）

```bash
#!/bin/bash
# 手动解析 --long-option

while [ $# -gt 0 ]; do
    case "$1" in
        --file=*)
            file="${1#*=}"
            ;;
        --file)
            file="$2"
            shift
            ;;
        --verbose)
            verbose=1
            ;;
        --help)
            usage
            ;;
        --)
            shift
            break
            ;;
        -*)
            echo "未知选项: $1"
            exit 1
            ;;
        *)
            break
            ;;
    esac
    shift
done
```

## 6.4 使用 getopt 支持长选项

```bash
#!/bin/bash
# getopt（非 getopts）支持长选项

OPTS=$(getopt -o f:vh -l file:,verbose,help -- "$@")
if [ $? -ne 0 ]; then
    usage
fi

eval set -- "$OPTS"

while true; do
    case "$1" in
        -f|--file)    file="$2"; shift 2 ;;
        -v|--verbose) verbose=1; shift ;;
        -h|--help)    usage; shift ;;
        --)           shift; break ;;
    esac
done
```

# 7. 脚本模块化

## 7.1 source 导入其他脚本

```bash
# main.sh
#!/bin/bash

# 方式1：直接 source
source ./lib/config.sh
. ./lib/logger.sh
. ./lib/database.sh

# 方式2：从脚本所在目录查找
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/lib/utils.sh"

init_app
log "启动完成"
```

## 7.2 防止重复加载

```bash
# lib/config.sh
# 防止同一个脚本被多次 source
if [ -n "${CONFIG_SH_LOADED:-}" ]; then
    return 0
fi
CONFIG_SH_LOADED=1

readonly APP_NAME="MyApp"
readonly APP_VERSION="1.0.0"
```

## 7.3 库函数设计原则

```bash
# 良好的库函数应该做到：
# 1. 不产生副作用（不修改全局状态，除非明确是目的）
# 2. 使用 local 变量
# 3. 错误时返回非零退出码
# 4. 用 || 而不是 && 处理错误

# 好的实践
parse_yaml() {
    local file="$1"
    local prefix="${2:-}"
    
    if [ ! -f "$file" ]; then
        echo "错误: 文件不存在: $file" >&2
        return 1
    fi
    
    # 解析逻辑...
}

# 调用
if ! parse_yaml "config.yml"; then
    exit 1
fi
```

# 8. 错误处理模式

```bash
#!/bin/bash
set -euo pipefail    # 严格模式（后文详述）

# 自定义错误处理
error_exit() {
    echo "错误: $1 (行号: ${BASH_LINENO[0]})" >&2
    exit 1
}

# 清理函数
cleanup() {
    echo "清理临时文件..."
    rm -rf /tmp/myapp-$$
}

# 注册清理函数（EXIT 信号捕获）
trap cleanup EXIT

# 执行关键操作
critical_operation() {
    if [ ! -f "important.conf" ]; then
        error_exit "配置文件缺失"
    fi
    
    # 操作代码
}

critical_operation
```

## 8.1 严格模式 `set -euo pipefail`

```bash
#!/bin/bash
set -e            # 任何命令失败（返回非0）立即退出
set -u            # 使用未定义变量时退出
set -o pipefail   # 管道中任一命令失败，整个管道返回失败

# 等价于
set -euo pipefail

# 临时关闭 set -e
set +e
risky_command
set -e
```

**特别注意 `set -e` 的行为：**

```bash
set -e

# 以下情况不会触发 -e 退出
if grep "pattern" file.txt; then ... fi    # 在 if 条件中
while ! ping -c 1 host; do ... done        # 在 while/until 条件中
command || true                              # 用 || 捕获
command && true                              # 用 && 连接
```

# 9. 脚本执行生命周期

```
脚本启动
  ↓
1. 读取 shebang，确定解释器
  ↓
2. 解析脚本参数（位置参数）
  ↓
3. 执行 BASH_ENV 环境文件（如果设置了且非交互）
  ↓
4. 逐行解析执行
   ├── 函数定义（记录函数体，不执行）
   ├── 变量赋值
   ├── 条件分支/循环
   └── 函数调用
  ↓
5. 遇到 exit 或脚本末尾
   ├── trap EXIT 触发
   └── 返回 exit 状态码
```

# 10. 脚本模板

```bash
#!/bin/bash
#===============================================================================
# 脚本名称: template.sh
# 描述:     脚本模板
# 用法:     ./template.sh [选项] <参数>
# 版本:     1.0.0
#===============================================================================

set -euo pipefail
IFS=$'\n\t'

# 全局变量
readonly SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# 函数定义
usage() {
    cat << EOF
用法: $SCRIPT_NAME [选项] <参数>

选项:
  -h, --help     显示帮助
  -v, --verbose  详细输出
  -f, --file    指定文件路径

示例:
  $SCRIPT_NAME -f config.conf
EOF
    exit 0
}

log() {
    local level="${1:-INFO}"
    local message="$2"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $message"
}

error() {
    log ERROR "$1" >&2
    exit 1
}

cleanup() {
    local exit_code=$?
    log INFO "脚本结束，退出码: $exit_code"
    [ -d "${TMPDIR:-}" ] && rm -rf "$TMPDIR"
    exit $exit_code
}

# 主逻辑
main() {
    trap cleanup EXIT
    
    TMPDIR="$(mktemp -d)"
    log INFO "临时目录: $TMPDIR"
    
    # 业务逻辑...
    
    log INFO "操作完成"
}

# 入口
main "$@"
```
