
# 1. 日志轮转脚本

## 1.1 按大小轮转

```bash
#!/bin/bash
#
# log_rotate_by_size.sh - 按文件大小轮转日志
#
# 用法:
#   ./log_rotate_by_size.sh <日志文件> [最大大小MB] [保留份数]
#
# 示例:
#   ./log_rotate_by_size.sh /var/log/app.log 100 7

set -euo pipefail

LOG_FILE="${1:?用法: $0 <日志文件> [最大大小MB] [保留份数]}"
MAX_SIZE_MB="${2:-100}"
KEEP_COUNT="${3:-7}"
MAX_SIZE_BYTES=$((MAX_SIZE_MB * 1024 * 1024))

readonly LOG_FILE MAX_SIZE_MB KEEP_COUNT MAX_SIZE_BYTES

rotate_log() {
    local log_file="$1"

    if [[ ! -f "$log_file" ]]; then
        echo "日志文件不存在: $log_file"
        return 1
    fi

    local file_size
    file_size=$(stat -c%s "$log_file" 2>/dev/null || stat -f%z "$log_file" 2>/dev/null)

    if (( file_size < MAX_SIZE_BYTES )); then
        return 0
    fi

    echo "[$(date '+%Y-%m-%d %H:%M:%S')] 日志大小 ${file_size} 字节，超过阈值 ${MAX_SIZE_BYTES} 字节，开始轮转..."

    # 删除最旧的备份（超出保留份数）
    local old_log
    for old_log in $(ls -1t "${log_file}".* 2>/dev/null | tail -n +"$KEEP_COUNT"); do
        echo "  删除旧备份: $old_log"
        rm -f "$old_log"
    done

    # 轮转：将 .1 改为 .2，依此类推
    local i
    for (( i = KEEP_COUNT - 1; i >= 1; i-- )); do
        if [[ -f "${log_file}.${i}" ]]; then
            mv "${log_file}.${i}" "${log_file}.$((i + 1))"
        fi
    done

    # 将当前日志移动为 .1
    cp "$log_file" "${log_file}.1"
    : > "$log_file"   # 清空当前日志

    echo "  轮转完成。已将当前日志移动至 ${log_file}.1"

    # 可选：发送信号通知应用重新打开日志
    # pkill -USR1 -f "your_app" 2>/dev/null || true
}

rotate_log "$LOG_FILE"
```

## 1.2 按时间轮转 + 压缩归档

```bash
#!/bin/bash
#
# log_rotate_by_time.sh - 按时间周期轮转日志并压缩归档
#
# 用法:
#   ./log_rotate_by_time.sh [--keep-days 30] [--pattern "*.log"] <日志目录>

set -euo pipefail

LOG_DIR=""
KEEP_DAYS=30
PATTERN="*.log"

usage() {
    cat <<EOF
用法: $0 [选项] <日志目录>

选项:
    --keep-days N   保留最近 N 天的归档（默认: 30）
    --pattern PAT   匹配日志文件的 glob 模式（默认: *.log）
    -h, --help      显示帮助

示例:
    $0 --keep-days 7 /var/log/myapp
    $0 --keep-days 14 --pattern "access*.log" /var/log/nginx
EOF
    exit "${1:-0}"
}

while (( $# > 0 )); do
    case "$1" in
        --keep-days) KEEP_DAYS="$2"; shift 2 ;;
        --pattern)   PATTERN="$2";   shift 2 ;;
        -h|--help)   usage 0 ;;
        -*)          echo "未知选项: $1"; usage 1 ;;
        *)           LOG_DIR="$1"; shift ;;
    esac
done

if [[ -z "$LOG_DIR" || ! -d "$LOG_DIR" ]]; then
    echo "错误: 请指定有效的日志目录"
    usage 1
fi

readonly LOG_DIR KEEP_DAYS PATTERN

TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
ARCHIVE_NAME="logs_${TIMESTAMP}.tar.gz"

echo "===== 日志轮转开始 $(date) ====="

# 1. 打包并压缩当前日志
cd "$LOG_DIR"
shopt -s nullglob
log_files=( $PATTERN )
if (( ${#log_files[@]} > 0 )); then
    echo "打包 ${#log_files[@]} 个日志文件..."
    tar -czf "/tmp/${ARCHIVE_NAME}" "${log_files[@]}"
    echo "归档文件: /tmp/${ARCHIVE_NAME} (大小: $(du -h "/tmp/${ARCHIVE_NAME}" | cut -f1))"

    # 清空原始日志
    for f in "${log_files[@]}"; do
        : > "$f"
    done
else
    echo "没有匹配的日志文件，跳过打包。"
fi

# 2. 转移归档到日志目录下的 archive 子目录
mkdir -p "${LOG_DIR}/archive"
mv "/tmp/${ARCHIVE_NAME}" "${LOG_DIR}/archive/"

# 3. 删除超过保留天数的归档
echo "清理 ${KEEP_DAYS} 天前的归档..."
find "${LOG_DIR}/archive" -name "logs_*.tar.gz" -type f -mtime "+${KEEP_DAYS}" -print -delete

echo "===== 日志轮转结束 $(date) ====="

# 4. 可选：压缩单独的大日志文件（超过 50MB）
find "$LOG_DIR" -maxdepth 1 -name "$PATTERN" -size +50M | while IFS= read -r big_log; do
    echo "大文件压缩: $big_log"
    gzip -c "$big_log" > "${big_log}.$(date '+%Y%m%d').gz"
    : > "$big_log"
done
```

## 1.3 保留最近 N 天日志（简化版）

```bash
#!/bin/bash
# clean_old_logs.sh - 删除指定目录下 N 天前的日志文件

LOG_DIR="${1:-/var/log/myapp}"
KEEP_DAYS="${2:-30}"

if [[ ! -d "$LOG_DIR" ]]; then
    echo "错误: 目录不存在: $LOG_DIR"
    exit 1
fi

echo "清理 ${LOG_DIR} 下 ${KEEP_DAYS} 天前的日志..."

# 删除 N 天前的 .log 文件
find "$LOG_DIR" -name "*.log" -type f -mtime "+${KEEP_DAYS}" -print -delete

# 删除空目录
find "$LOG_DIR" -type d -empty -delete 2>/dev/null

echo "清理结束。"
```

# 2. 备份脚本

## 2.1 全量增量备份（基于 rsync + 硬链接快照）

```bash
#!/bin/bash
#
# snapshot_backup.sh - 基于 rsync + 硬链接的类 Time Machine 快照备份
#
# 原理：用 --link-dest 指向上一次备份，未变化的文件通过硬链接共享，
#       只占用变化文件的空间，实现"增量"效果。
#
# 用法:
#   ./snapshot_backup.sh <源目录> <备份根目录> [保留快照数]

set -euo pipefail

SRC="${1:?用法: $0 <源目录> <备份根目录> [保留快照数]}"
BACKUP_ROOT="${2:?缺少备份根目录}"
KEEP="${3:-30}"

readonly SRC BACKUP_ROOT KEEP

# 格式化源路径为合法目录名（去掉 / 替换为 _）
SRC_NAME="${SRC#/}"
SRC_NAME="${SRC_NAME//\//_}"

BACKUP_DIR="${BACKUP_ROOT}/${SRC_NAME}"
SNAPSHOT_DIR="${BACKUP_DIR}/snapshots"
TIMESTAMP=$(date '+%Y-%m-%d_%H%M%S')
LAST_BACKUP=$(ls -1d "${SNAPSHOT_DIR}"/*/ 2>/dev/null | tail -1 || true)

mkdir -p "$SNAPSHOT_DIR"

echo "===== 快照备份 ====="
echo "源:      $SRC"
echo "目标:    ${SNAPSHOT_DIR}/${TIMESTAMP}"
echo "上次备份: ${LAST_BACKUP:-无（全量模式）}"

RSYNC_OPTS=(
    -aAXvh               # 归档模式 + 扩展属性
    --delete             # 删除目标中源已不存在的文件
    --exclude='/tmp/*'
    --exclude='/.cache'
    --exclude='*.swp'
    --exclude='*.swo'
    --exclude='*.tmp'
)

# --link-dest：未变化的文件通过硬链接共享，不占用额外磁盘
if [[ -d "$LAST_BACKUP" ]]; then
    echo "使用增量模式（link-dest=${LAST_BACKUP}）"
    rsync "${RSYNC_OPTS[@]}" --link-dest="$LAST_BACKUP" "$SRC/" "${SNAPSHOT_DIR}/${TIMESTAMP}/"
else
    echo "全量备份..."
    rsync "${RSYNC_OPTS[@]}" "$SRC/" "${SNAPSHOT_DIR}/${TIMESTAMP}/"
fi

# 记录备份
echo "$TIMESTAMP - 主机: $(hostname)" >> "${BACKUP_DIR}/backup.log"

# 删除旧快照
echo "清理过期快照（保留最近 ${KEEP} 个）..."
ls -1dt "${SNAPSHOT_DIR}"/*/ 2>/dev/null | tail -n +"$((KEEP + 1))" | while IFS= read -r old; do
    echo "  删除: $old"
    rm -rf "$old"
done

# 统计
snapshot_count=$(ls -1d "${SNAPSHOT_DIR}"/*/ 2>/dev/null | wc -l)
disk_usage=$(du -sh "$BACKUP_DIR" 2>/dev/null | cut -f1)
echo "快照数: $snapshot_count，磁盘占用: $disk_usage"
echo "===== 备份完成 ====="
```

## 2.2 MySQL 备份

```bash
#!/bin/bash
#
# mysql_backup.sh - MySQL 多数据库备份（支持单库/全库、压缩、远程传输）
#
# 用法: ./mysql_backup.sh [--db DBNAME] [--all] [--gzip]
#
# 环境变量:
#   MYSQL_HOST     - 数据库主机（默认: localhost）
#   MYSQL_PORT     - 数据库端口（默认: 3306）
#   MYSQL_USER     - 数据库用户（默认: root）
#   MYSQL_PASSWORD - 数据库密码（或通过 ~/.my.cnf 配置）

set -euo pipefail

MYSQL_HOST="${MYSQL_HOST:-localhost}"
MYSQL_PORT="${MYSQL_PORT:-3306}"
MYSQL_USER="${MYSQL_USER:-root}"
BACKUP_DIR="${BACKUP_DIR:-/data/backup/mysql}"
KEEP_DAYS="${KEEP_DAYS:-14}"

DATE=$(date '+%Y%m%d_%H%M%S')
SINGLE_DB=""
BACKUP_ALL=false
USE_GZIP=false

usage() {
    cat <<EOF
用法: $0 [选项]

选项:
    --db DB       备份指定数据库
    --all         备份所有数据库
    --gzip        使用 gzip 压缩
    -h, --help    显示帮助

示例:
    $0 --db myapp --gzip
    $0 --all
EOF
    exit "${1:-0}"
}

while (( $# > 0 )); do
    case "$1" in
        --db)   SINGLE_DB="$2"; shift 2 ;;
        --all)  BACKUP_ALL=true; shift ;;
        --gzip) USE_GZIP=true;   shift ;;
        -h|--help) usage 0 ;;
        *)      echo "未知选项: $1"; usage 1 ;;
    esac
done

mkdir -p "$BACKUP_DIR"

# 构造 MySQL 连接参数
MYSQL_OPTS=(-h"$MYSQL_HOST" -P"$MYSQL_PORT" -u"$MYSQL_USER")
if [[ -n "${MYSQL_PASSWORD:-}" ]]; then
    MYSQL_OPTS+=(-p"$MYSQL_PASSWORD")
fi

backup_db() {
    local db="$1"
    local outfile="${BACKUP_DIR}/${db}_${DATE}.sql"
    echo "备份数据库: $db -> ${outfile}"
    mysqldump "${MYSQL_OPTS[@]}" --single-transaction --routines --triggers --events "$db" > "$outfile"

    if [[ "$USE_GZIP" == true ]]; then
        gzip -f "$outfile"
        outfile="${outfile}.gz"
    fi

    echo "  完成: $outfile ($(du -h "$outfile" | cut -f1))"
}

if [[ "$BACKUP_ALL" == true ]]; then
    echo "=== 全库备份模式 ==="
    databases=$(mysql "${MYSQL_OPTS[@]}" -N -e "SHOW DATABASES" | grep -v -E '^(information_schema|performance_schema|sys|mysql)$')
    for db in $databases; do
        backup_db "$db"
    done
elif [[ -n "$SINGLE_DB" ]]; then
    echo "=== 单库备份: $SINGLE_DB ==="
    backup_db "$SINGLE_DB"
else
    echo "错误: 请指定 --db 或 --all"
    usage 1
fi

# 清理过期备份
echo "清理 ${KEEP_DAYS} 天前的备份..."
find "$BACKUP_DIR" -name "*.sql*" -type f -mtime "+${KEEP_DAYS}" -print -delete

echo "=== 备份完成 $(date) ==="
```

## 2.3 rsync 远程同步

```bash
#!/bin/bash
#
# rsync_remote.sh - rsync 远程同步脚本（支持 SSH + 排除规则 + 带宽限制）
#
# 用法:
#   推送: ./rsync_remote.sh push /local/path user@host:/remote/path
#   拉取: ./rsync_remote.sh pull user@host:/remote/path /local/path

set -euo pipefail

DIRECTION="${1:?用法: $0 push|pull <源> <目标>}"
SRC="${2:?}"
DEST="${3:?}"

readonly DIRECTION SRC DEST

# ── 配置 ──
SSH_PORT="${SSH_PORT:-22}"
BANDWIDTH_LIMIT="${BANDWIDTH_LIMIT:-0}"       # KB/s, 0 表示不限速
EXCLUDE_FILE="${EXCLUDE_FILE:-}"
DELETE_FLAG="${DELETE_FLAG:---delete}"         # 同步删除

RSYNC_OPTS=(
    -avz
    --progress
    -e "ssh -p ${SSH_PORT} -o StrictHostKeyChecking=no"
    --exclude='.git'
    --exclude='node_modules'
    --exclude='*.swp'
    --exclude='*.tmp'
    --exclude='.DS_Store'
)

# 带宽限制
if (( BANDWIDTH_LIMIT > 0 )); then
    RSYNC_OPTS+=(--bwlimit="$BANDWIDTH_LIMIT")
fi

# 排除文件列表
if [[ -n "$EXCLUDE_FILE" && -f "$EXCLUDE_FILE" ]]; then
    RSYNC_OPTS+=(--exclude-from="$EXCLUDE_FILE")
fi

# 同步删除
RSYNC_OPTS+=("$DELETE_FLAG")

echo "===== Rsync 同步 ====="
echo "方向: $DIRECTION"
echo "源:   $SRC"
echo "目标: $DEST"

case "$DIRECTION" in
    push)
        # 本地推到远程
        echo "推送模式..."
        rsync "${RSYNC_OPTS[@]}" "$SRC" "$DEST"
        ;;
    pull)
        # 远程拉到本地（数据来自远程）
        echo "拉取模式..."
        rsync "${RSYNC_OPTS[@]}" "$SRC" "$DEST"
        ;;
    *)
        echo "错误: 方向必须是 push 或 pull"
        exit 1
        ;;
esac

echo "===== 同步完成 $(date) ====="
```

# 3. 进程监控与自动重启

## 3.1 监控特定进程并自动重启

```bash
#!/bin/bash
#
# process_watchdog.sh - 进程监控守护脚本
#
# 监控指定进程，发现不存活则自动重启，支持告警。
#
# 用法:
#   ./process_watchdog.sh --name "nginx" --cmd "systemctl start nginx" [--interval 10]

set -euo pipefail

PROCESS_NAME=""
START_CMD=""
CHECK_INTERVAL="${CHECK_INTERVAL:-10}"       # 检查间隔（秒）
MAX_RETRY="${MAX_RETRY:-3}"                   # 连续重启失败阈值
RESTART_COOLDOWN="${RESTART_COOLDOWN:-60}"    # 冷却时间（秒），避免反复重启

usage() {
    cat <<EOF
用法: $0 --name <进程名> --cmd <启动命令> [选项]

选项:
    --name NAME        进程名称（用于 pgrep 匹配）
    --cmd  COMMAND     启动命令
    --interval SEC     检查间隔秒数（默认: 10）
    --max-retry N      最大连续重试次数（默认: 3）
    --cooldown SEC     重启冷却时间秒数（默认: 60）
    -h, --help         显示帮助

示例:
    $0 --name nginx --cmd "systemctl start nginx"
    $0 --name myapp --cmd "nohup /opt/myapp/bin/start.sh &> /dev/null &" --interval 5
EOF
    exit "${1:-0}"
}

while (( $# > 0 )); do
    case "$1" in
        --name)           PROCESS_NAME="$2";    shift 2 ;;
        --cmd)            START_CMD="$2";       shift 2 ;;
        --interval)       CHECK_INTERVAL="$2";  shift 2 ;;
        --max-retry)      MAX_RETRY="$2";       shift 2 ;;
        --cooldown)       RESTART_COOLDOWN="$2"; shift 2 ;;
        -h|--help)        usage 0 ;;
        *)                echo "未知选项: $1"; usage 1 ;;
    esac
done

if [[ -z "$PROCESS_NAME" || -z "$START_CMD" ]]; then
    echo "错误: 必须指定 --name 和 --cmd"
    usage 1
fi

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [WATCHDOG] $*"
}

# 检查进程是否存在
is_alive() {
    pgrep -x "$PROCESS_NAME" >/dev/null 2>&1
}

# 发送告警（可替换为实际告警通道，见 1.9 节）
send_alert() {
    echo "[ALERT] $*" >&2
}

last_restart_time=0
consecutive_failures=0

log "看门狗启动，监控进程: $PROCESS_NAME，检查间隔: ${CHECK_INTERVAL}s"

while true; do
    if is_alive; then
        consecutive_failures=0
    else
        log "进程 $PROCESS_NAME 不存活！"

        # 检查冷却时间
        now=$(date +%s)
        if (( now - last_restart_time < RESTART_COOLDOWN )); then
            log "在冷却时间内（${RESTART_COOLDOWN}s），跳过重启。"
            sleep "$CHECK_INTERVAL"
            continue
        fi

        # 超过最大重试次数
        if (( consecutive_failures >= MAX_RETRY )); then
            msg="进程 $PROCESS_NAME 连续重启失败 ${consecutive_failures} 次，放弃自动重启！"
            log "$msg"
            send_alert "$msg"
            sleep "$CHECK_INTERVAL"
            continue
        fi

        log "尝试重启（第 $((consecutive_failures + 1)) 次）..."
        if eval "$START_CMD"; then
            log "重启命令执行成功。"
            ((consecutive_failures++))
            last_restart_time=$(date +%s)
            # 等几秒再检查是否真正启动
            sleep 3
            if is_alive; then
                log "进程确认已启动。"
                consecutive_failures=0
            else
                log "进程启动命令执行了但进程未出现！"
                send_alert "进程 $PROCESS_NAME 启动失败"
            fi
        else
            log "启动命令执行失败！"
            ((consecutive_failures++))
            send_alert "进程 $PROCESS_NAME 启动命令执行失败"
        fi
    fi

    sleep "$CHECK_INTERVAL"
done
```

## 3.2 死锁 / 僵尸进程检测

```bash
#!/bin/bash
#
# check_zombie_and_deadlock.sh - 僵尸进程检测 + Java 线程死锁检测
#
# 用法: ./check_zombie_and_deadlock.sh [--jstack] [--alert]

set -euo pipefail

DO_JSTACK=false
DO_ALERT=false

while (( $# > 0 )); do
    case "$1" in
        --jstack) DO_JSTACK=true; shift ;;
        --alert)  DO_ALERT=true;  shift ;;
        *)        shift ;;
    esac
done

log() { echo "[$(date '+%H:%M:%S')] $*"; }

# ── 1. 僵尸进程检测 ──
zombie_count=$(ps -eo stat | grep -c '^Z' 2>/dev/null || true)
if (( zombie_count > 0 )); then
    log "检测到 ${zombie_count} 个僵尸进程 <defunct>"
    ps -eo pid,ppid,stat,comm | awk 'NR==1 || $3 ~ /^Z/'
    if [[ "$DO_ALERT" == true ]]; then
        log "[ALERT] 僵尸进程数: $zombie_count"
    fi
else
    log "无僵尸进程。"
fi

# ── 2. D 状态（不可中断睡眠，可能 I/O 阻塞）进程检测 ──
d_count=$(ps -eo stat | grep -c '^D' 2>/dev/null || true)
if (( d_count > 0 )); then
    log "检测到 ${d_count} 个 D 状态进程（不可中断睡眠）"
    ps -eo pid,ppid,stat,wchan,comm | awk 'NR==1 || $3 ~ /^D/'
fi

# ── 3. CPU 使用率异常的进程 ──
log "CPU 使用率 Top 5："
ps -eo pid,pcpu,pmem,comm --sort=-pcpu | head -6

# ── 4. Java 线程死锁检测（如果启用） ──
if [[ "$DO_JSTACK" == true ]]; then
    log "Java 死锁检测..."
    j_pids=$(pgrep -f 'java' 2>/dev/null || true)
    if [[ -z "$j_pids" ]]; then
        log "没有运行中的 Java 进程。"
    else
        for pid in $j_pids; do
            log "检查 PID: $pid"
            # 死锁信息（jstack 输出到 stderr 最后几行通常包含死锁）
            jstack_output=$(jstack "$pid" 2>&1 || true)
            if echo "$jstack_output" | grep -q "deadlock"; then
                log "  !! 检测到死锁 !! PID=$pid"
                echo "$jstack_output" | grep -A 20 "deadlock"
                if [[ "$DO_ALERT" == true ]]; then
                    log "[ALERT] Java PID=$pid 发生死锁"
                fi
            else
                log "  无死锁"
            fi
        done
    fi
fi

# ── 5. 文件句柄使用情况 ──
log "文件句柄（open files）使用 Top 5："
for pid in $(ps -eo pid --sort=-pid | tail -n +2 | head -5); do
    fds=$(ls /proc/"$pid"/fd 2>/dev/null | wc -l || echo 0)
    comm=$(ps -p "$pid" -o comm= 2>/dev/null || echo "?")
    echo "  PID=$pid ($comm): ${fds} 个文件描述符"
done

log "检测完成。"
```

# 4. 批量文件处理

## 4.1 批量重命名

```bash
#!/bin/bash
#
# bulk_rename.sh - 批量重命名文件
#
# 支持模式：
#   --replace OLD NEW    将文件名中的 OLD 替换为 NEW
#   --prefix PRE         添加前缀
#   --suffix SUF         添加后缀（在扩展名之前）
#   --lowercase          转为小写
#   --uppercase          转为大写
#   --number             按数字编号重命名（1.txt, 2.txt...）
#   --dry-run            仅预览，不实际执行
#
# 用法: ./bulk_rename.sh --replace ' ' '_' *.txt

set -euo pipefail

DRY_RUN=false
MODE=""
ARG1=""
ARG2=""

usage() {
    cat <<EOF
用法: $0 [选项] <文件...>

选项:
    --replace OLD NEW   替换文件名中的字符串
    --prefix STR        添加前缀
    --suffix STR        添加后缀（扩展名之前）
    --lowercase         全部转为小写
    --uppercase         全部转为大写
    --number            按序号重命名（保留扩展名）
    --dry-run           仅预览，不执行
    -h, --help          帮助

示例:
    $0 --replace 'photo' 'pic' --dry-run *.jpg
    $0 --prefix 'backup_' --dry-run *.conf
    $0 --number *.log
    $0 --uppercase *.txt
EOF
    exit "${1:-0}"
}

while (( $# > 0 )); do
    case "$1" in
        --replace)   MODE="replace";   ARG1="$2"; ARG2="$3"; shift 3 ;;
        --prefix)    MODE="prefix";    ARG1="$2"; shift 2 ;;
        --suffix)    MODE="suffix";    ARG1="$2"; shift 2 ;;
        --lowercase) MODE="lowercase"; shift ;;
        --uppercase) MODE="uppercase"; shift ;;
        --number)    MODE="number";    shift ;;
        --dry-run)   DRY_RUN=true;     shift ;;
        -h|--help)   usage 0 ;;
        -*)          echo "未知选项: $1"; usage 1 ;;
        *)           break ;;
    esac
done

if [[ -z "$MODE" ]]; then
    echo "错误: 请指定操作模式"
    usage 1
fi

files=("$@")
if (( ${#files[@]} == 0 )); then
    echo "错误: 请指定要处理的文件"
    usage 1
fi

rename_file() {
    local src="$1"
    local dst="$2"

    if [[ "$src" == "$dst" ]]; then
        return
    fi

    if [[ -e "$dst" ]]; then
        echo "  跳过: 目标文件已存在: $dst"
        return
    fi

    if [[ "$DRY_RUN" == true ]]; then
        echo "  [DRY-RUN] $src  ->  $dst"
    else
        mv -v "$src" "$dst"
    fi
}

# ── 各种模式实现 ──
case "$MODE" in
    replace)
        for f in "${files[@]}"; do
            dir=$(dirname "$f")
            base=$(basename "$f")
            newbase="${base//$ARG1/$ARG2}"
            rename_file "$f" "${dir}/${newbase}"
        done
        ;;

    prefix)
        for f in "${files[@]}"; do
            dir=$(dirname "$f")
            base=$(basename "$f")
            rename_file "$f" "${dir}/${ARG1}${base}"
        done
        ;;

    suffix)
        for f in "${files[@]}"; do
            dir=$(dirname "$f")
            base=$(basename "$f")
            name="${base%.*}"
            ext="${base##*.}"
            if [[ "$name" == "$base" ]]; then
                ext=""  # 无扩展名
            else
                ext=".$ext"
            fi
            rename_file "$f" "${dir}/${name}${ARG1}${ext}"
        done
        ;;

    lowercase)
        for f in "${files[@]}"; do
            dir=$(dirname "$f")
            base=$(basename "$f")
            lower=$(echo "$base" | tr '[:upper:]' '[:lower:]')
            rename_file "$f" "${dir}/${lower}"
        done
        ;;

    uppercase)
        for f in "${files[@]}"; do
            dir=$(dirname "$f")
            base=$(basename "$f")
            upper=$(echo "$base" | tr '[:lower:]' '[:upper:]')
            rename_file "$f" "${dir}/${upper}"
        done
        ;;

    number)
        # 先排序
        mapfile -t sorted < <(printf '%s\n' "${files[@]}" | sort)
        index=1
        for f in "${sorted[@]}"; do
            dir=$(dirname "$f")
            ext=".${f##*.}"
            if [[ "$ext" == ".$f" ]]; then ext=""; fi  # 无扩展名
            newname=$(printf "%03d" $index)
            rename_file "$f" "${dir}/${newname}${ext}"
            ((index++))
        done
        ;;
esac

echo "处理完成。共 ${#files[@]} 个文件。"
```

## 4.2 图片压缩并行处理

```bash
#!/bin/bash
#
# batch_image_compress.sh - 批量压缩图片（支持并行处理）
#
# 依赖: ImageMagick (convert) 或 optipng / jpegoptim
#
# 用法: ./batch_image_compress.sh [--quality 75] [--max-size 900] [--parallel 4] <目录>

set -euo pipefail

QUALITY=75
MAX_SIZE_PX=1200
PARALLEL=4
TARGET_DIR=""
DRY_RUN=false

usage() {
    cat <<EOF
用法: $0 [选项] <目录>

选项:
    --quality N      JPEG/WebP 质量 (1-100, 默认: 75)
    --max-size PX    最大宽/高像素 (默认: 1200)
    --parallel N     并行任务数 (默认: 4)
    --dry-run        仅扫描不压缩
    -h, --help       帮助
EOF
    exit "${1:-0}"
}

while (( $# > 0 )); do
    case "$1" in
        --quality)   QUALITY="$2";     shift 2 ;;
        --max-size)  MAX_SIZE_PX="$2"; shift 2 ;;
        --parallel)  PARALLEL="$2";    shift 2 ;;
        --dry-run)   DRY_RUN=true;     shift ;;
        -h|--help)   usage 0 ;;
        -*)          echo "未知: $1"; usage 1 ;;
        *)           TARGET_DIR="$1";  shift ;;
    esac
done

if [[ -z "$TARGET_DIR" || ! -d "$TARGET_DIR" ]]; then
    echo "错误: 请指定有效目录"
    usage 1
fi

# 检查依赖
check_dep() {
    if ! command -v "$1" &>/dev/null; then
        echo "警告: 未找到 $1，部分格式将无法处理"
        return 1
    fi
    return 0
}

HAS_MAGICK=false
HAS_JPEGOPTIM=false
HAS_OPTIPNG=false
check_dep "convert"    && HAS_MAGICK=true
check_dep "jpegoptim"  && HAS_JPEGOPTIM=true
check_dep "optipng"    && HAS_OPTIPNG=true

echo "目标目录: $TARGET_DIR"
echo "JPEG 质量: $QUALITY | 最大尺寸: ${MAX_SIZE_PX}px | 并行数: $PARALLEL"
echo "ImageMagick: $HAS_MAGICK | jpegoptim: $HAS_JPEGOPTIM | optipng: $HAS_OPTIPNG"
echo ""

# 压缩单张图片的函数
compress_image() {
    local file="$1"
    local ext="${file##*.}"
    ext=$(echo "$ext" | tr '[:upper:]' '[:lower:]')
    local original_size
    original_size=$(stat -c%s "$file" 2>/dev/null || stat -f%z "$file" 2>/dev/null)

    case "$ext" in
        jpg|jpeg)
            if [[ "$HAS_JPEGOPTIM" == true ]]; then
                jpegoptim --max="$QUALITY" --strip-all --all-progressive "$file" &>/dev/null
            elif [[ "$HAS_MAGICK" == true ]]; then
                convert "$file" -resize "${MAX_SIZE_PX}x${MAX_SIZE_PX}>" -quality "$QUALITY" "$file"
            fi
            ;;
        png)
            if [[ "$HAS_OPTIPNG" == true ]]; then
                optipng -o2 -quiet "$file" &>/dev/null
            elif [[ "$HAS_MAGICK" == true ]]; then
                convert "$file" -resize "${MAX_SIZE_PX}x${MAX_SIZE_PX}>" PNG8:"$file"
            fi
            ;;
        gif|bmp|tiff)
            if [[ "$HAS_MAGICK" == true ]]; then
                convert "$file" -resize "${MAX_SIZE_PX}x${MAX_SIZE_PX}>" "$file"
            fi
            ;;
    esac

    local new_size
    new_size=$(stat -c%s "$file" 2>/dev/null || stat -f%z "$file" 2>/dev/null)
    local saved=$((original_size - new_size))
    local pct=0
    (( original_size > 0 )) && pct=$((saved * 100 / original_size))
    echo "  $file: ${original_size}B -> ${new_size}B (节省 ${saved}B, ${pct}%)"
}

export -f compress_image
export QUALITY MAX_SIZE_PX HAS_MAGICK HAS_JPEGOPTIM HAS_OPTIPNG

# 收集文件列表
echo "扫描图片文件..."
mapfile -t img_files < <(find "$TARGET_DIR" -type f \( \
    -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" \
    -o -iname "*.gif" -o -iname "*.bmp" -o -iname "*.tiff" \) 2>/dev/null)

total=${#img_files[@]}
echo "共找到 ${total} 个图片文件"
echo ""

if [[ "$DRY_RUN" == true ]]; then
    echo "[DRY-RUN 模式] 以下文件将被处理:"
    printf '%s\n' "${img_files[@]}"
    exit 0
fi

if (( total == 0 )); then
    echo "没有可处理的图片。"
    exit 0
fi

echo "开始压缩（并行数=$PARALLEL）..."

export COMPRESS_TOTAL="$total"
export COMPRESS_DONE=0

printf '%s\n' "${img_files[@]}" | xargs -P "$PARALLEL" -I {} bash -c 'compress_image "$1"' _ {}

echo ""
echo "===== 压缩完成 ====="
```

# 5. 配置文件解析

## 5.1 INI 文件解析

```bash
#!/bin/bash
#
# ini_parser.sh - INI 配置文件解析库
#
# INI 格式示例:
#   [section1]
#   key1 = value1
#   key2 = "value with spaces"
#
# 用法:
#   source ini_parser.sh
#   ini_parse "config.ini"
#   echo "${ini__section1__key1}"

set -euo pipefail

# 解析 INI 文件，生成变量 ini__<section>__<key>
ini_parse() {
    local ini_file="$1"
    local section=""
    local key="" value=""

    if [[ ! -f "$ini_file" ]]; then
        echo "INI 文件不存在: $ini_file" >&2
        return 1
    fi

    # 清除之前的解析结果
    unset $(compgen -v ini__ 2>/dev/null || true)

    while IFS= read -r line || [[ -n "$line" ]]; do
        # 去除首尾空白
        line=$(echo "$line" | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//')

        # 跳过空行和注释
        [[ -z "$line" || "$line" == \;* || "$line" == \#* ]] && continue

        # 匹配 [section]
        if [[ "$line" =~ ^\[(.+)\]$ ]]; then
            section="${BASH_REMATCH[1]}"
            # 将 section 名中的非法字符替换为 _
            section=$(echo "$section" | tr -c '[:alnum:]_' '_')
            continue
        fi

        # 匹配 key = value
        if [[ "$line" =~ ^([^=]+)=(.*)$ ]]; then
            key=$(echo "${BASH_REMATCH[1]}" | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//')
            value="${BASH_REMATCH[2]}"
            # 去除 value 首尾空白和引号
            value=$(echo "$value" | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//' -e 's/^"//' -e 's/"$//' -e "s/^'//" -e "s/'$//")

            key=$(echo "$key" | tr -c '[:alnum:]_' '_')

            if [[ -n "$section" ]]; then
                printf -v "ini__${section}__${key}" '%s' "$value"
            else
                printf -v "ini__${key}" '%s' "$value"
            fi
        fi
    done < "$ini_file"

    return 0
}

# 获取 INI 值
ini_get() {
    local section="$1"
    local key="$2"
    local default="${3:-}"
    local var_name

    if [[ -n "$section" ]]; then
        section=$(echo "$section" | tr -c '[:alnum:]_' '_')
        var_name="ini__${section}__${key}"
    else
        var_name="ini__${key}"
    fi

    echo "${!var_name:-$default}"
}

# 列出解析后的所有变量
ini_list() {
    compgen -v ini__ | while read -r var; do
        echo "$var = ${!var}"
    done
}

# ── 使用示例 ──
# ini_parse "config.ini"
# db_host=$(ini_get "database" "host" "localhost")
# db_port=$(ini_get "database" "port" "3306")
```

## 5.2 YAML 简易解析（无外部依赖）

```bash
#!/bin/bash
#
# yaml_simple_parser.sh - 纯 Bash 解析扁平 YAML（不支持嵌套/列表）
#
# 适用场景：配置项均为 key: value 的简单 YAML 文件
#
# 注意：对于复杂 YAML，推荐使用 yq（https://github.com/mikefarah/yq）

set -euo pipefail

yaml_parse() {
    local yaml_file="$1"
    local indent_level=0
    local key="" value=""
    local prefix=""

    if [[ ! -f "$yaml_file" ]]; then
        echo "YAML 文件不存在: $yaml_file" >&2
        return 1
    fi

    unset $(compgen -v yaml__ 2>/dev/null || true)

    while IFS= read -r line || [[ -n "$line" ]]; do
        # 跳过空行和注释
        [[ -z "$line" || "$line" =~ ^[[:space:]]*# ]] && continue

        # 跳过列表项（-- 开头）
        [[ "$line" =~ ^[[:space:]]*-[[:space:]] ]] && continue

        # 匹配 key: value（允许冒号后有空格）
        if [[ "$line" =~ ^[[:space:]]*([a-zA-Z_][a-zA-Z0-9_-]*):[[:space:]]*(.*) ]]; then
            key="${BASH_REMATCH[1]}"
            value="${BASH_REMATCH[2]}"
            # 去除尾部空白
            value=$(echo "$value" | sed -e 's/[[:space:]]*$//')
            # 去除引号
            value=$(echo "$value" | sed -e 's/^"//' -e 's/"$//' -e "s/^'//" -e "s/'$//")

            # 计算缩进层级
            indent_level=$(echo "$line" | sed 's/[^ ].*//' | wc -c)
            indent_level=$((indent_level / 2))

            if (( indent_level == 0 )); then
                prefix=""
            fi

            if [[ -z "$value" ]]; then
                # 可能是嵌套的父 key
                prefix="${key}__"
            else
                printf -v "yaml__${prefix}${key}" '%s' "$value"
            fi
        fi
    done < "$yaml_file"

    return 0
}

yaml_get() {
    local key="$1"
    local default="${2:-}"
    local var_name="yaml__${key}"
    echo "${!var_name:-$default}"
}

yaml_list() {
    compgen -v yaml__ | while read -r var; do
        echo "$var = ${!var}"
    done
}
```

## 5.3 环境变量替换模板

```bash
#!/bin/bash
#
# envsubst_template.sh - 将模板文件中的 ${VAR} 替换为环境变量值
#
# 类似 envsubst 命令，但提供更多功能：默认值、必填检查
#
# 用法:
#   VAR1=hello VAR2=world ./envsubst_template.sh template.txt > output.txt
#
# 模板语法:
#   ${VAR_NAME}            - 环境变量值（不存在则报错）
#   ${VAR_NAME:-default}   - 有默认值
#   ${VAR_NAME:?error msg} - 必须存在否则报错

set -euo pipefail

TEMPLATE="${1:?用法: $0 <模板文件>}"

if [[ ! -f "$TEMPLATE" ]]; then
    echo "模板文件不存在: $TEMPLATE" >&2
    exit 1
fi

# 自定义 envsubst（不依赖外部 gettext）
envsubst_custom() {
    local content="$1"
    local missing_vars=()

    while IFS= read -r line; do
        # 匹配 ${VAR} 或 ${VAR:-default} 或 ${VAR:?error}
        while [[ "$line" =~ \$\{([a-zA-Z_][a-zA-Z0-9_]*)(:-([^}]*))?(\?\?([^}]*))?(:([^}]*))?\} ]]; do
            full_match="${BASH_REMATCH[0]}"
            var_name="${BASH_REMATCH[1]}"
            default_val="${BASH_REMATCH[4]:-}"
            required_msg="${BASH_REMATCH[6]:-}"

            if [[ -n "${!var_name+x}" ]]; then
                # 变量已设置
                replacement="${!var_name}"
            elif [[ "$full_match" == *":-"* ]]; then
                # 使用默认值
                replacement="$default_val"
            elif [[ "$full_match" == *":?"* ]]; then
                # 必填且缺失
                echo "错误: ${var_name} 未设置. ${required_msg}" >&2
                exit 1
            elif [[ -z "${!var_name+x}" ]]; then
                # 普通变量缺失
                missing_vars+=("$var_name")
                replacement=""
            else
                replacement=""
            fi

            line="${line//${full_match}/${replacement}}"
        done
        echo "$line"
    done <<< "$content"

    if (( ${#missing_vars[@]} > 0 )); then
        echo "警告: 以下变量未定义: ${missing_vars[*]}" >&2
    fi
}

content=$(cat "$TEMPLATE")
envsubst_custom "$content"
```

# 6. 系统健康检查

## 6.1 一键巡检脚本

```bash
#!/bin/bash
#
# health_check.sh - 系统健康巡检（CPU / 内存 / 磁盘 / 网络 / 进程）
#
# 用法:
#   ./health_check.sh                    # 终端输出
#   ./health_check.sh --json             # JSON 格式输出
#   ./health_check.sh --html > report.html

set -euo pipefail

OUTPUT_MODE="text"     # text | json | html

usage() {
    cat <<EOF
用法: $0 [选项]

选项:
    --json         JSON 格式输出
    --html         HTML 格式输出
    -h, --help     帮助
EOF
    exit "${1:-0}"
}

while (( $# > 0 )); do
    case "$1" in
        --json)  OUTPUT_MODE="json";  shift ;;
        --html)  OUTPUT_MODE="html";  shift ;;
        -h|--help) usage 0 ;;
        *)       shift ;;
    esac
done

TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
HOSTNAME=$(hostname)
KERNEL=$(uname -r)
UPTIME_STR=$(uptime)

# ── 采集函数 ──

get_cpu_info() {
    local cpu_usage
    cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print 100 - $8}' 2>/dev/null || \
                cat /proc/stat | awk '/^cpu / { idle=$5; total=0; for(i=2;i<=NF;i++) total+=$i; printf "%.1f", (1-idle/total)*100 }')
    local load
    load=$(uptime | awk -F'load average:' '{print $2}' | sed 's/^[[:space:]]*//')

    echo "cpu_usage=${cpu_usage}"
    echo "cpu_load=${load}"
    echo "cpu_cores=$(nproc)"
}

get_mem_info() {
    # Linux
    if [[ -f /proc/meminfo ]]; then
        local total=$(awk '/^MemTotal:/ {print $2}' /proc/meminfo)
        local avail=$(awk '/^MemAvailable:/ {print $2}' /proc/meminfo)
        if [[ -z "$avail" ]]; then
            local free=$(awk '/^MemFree:/ {print $2}' /proc/meminfo)
            local buffers=$(awk '/^Buffers:/ {print $2}' /proc/meminfo)
            local cached=$(awk '/^Cached:/ {print $2}' /proc/meminfo)
            avail=$((free + buffers + cached))
        fi
        local usage_pct=$(( (total - avail) * 100 / total ))
        echo "mem_total=$((total / 1024))MB"
        echo "mem_avail=$((avail / 1024))MB"
        echo "mem_usage_pct=${usage_pct}%"
    else
        echo "mem_total=N/A"
        echo "mem_avail=N/A"
        echo "mem_usage_pct=N/A"
    fi
}

get_disk_info() {
    echo "disk_mounts=$(df -h | grep -vE '^(Filesystem|tmpfs|devtmpfs|none)' | awk 'NF>1 {printf "%s(%s) ", $6, $5}')"
    # 超过 80% 的分区
    local critical
    critical=$(df -h | awk 'NR>1 && +$5 > 80 {print $6"="$5}' | paste -sd ',' -)
    echo "disk_critical=${critical:-无}"
    # inode 使用
    echo "disk_inode_warn=$(df -i | awk 'NR>1 && +$5 > 80 {print $6"="$5}' | paste -sd ',' -)"
}

get_net_info() {
    # 默认网关连通性
    local gw
    gw=$(ip route show default 2>/dev/null | awk '/default/ {print $3}' | head -1)
    if [[ -z "$gw" ]]; then
        gw=$(route -n 2>/dev/null | awk '/^0.0.0.0/ {print $2}' | head -1)
    fi
    if [[ -n "$gw" ]]; then
        if ping -c 2 -W 2 "$gw" &>/dev/null; then
            echo "net_gateway=${gw}(通)"
        else
            echo "net_gateway=${gw}(不通!)"
        fi
    else
        echo "net_gateway=未找到默认网关"
    fi

    # 监听端口
    local listen_ports
    listen_ports=$(ss -tlnp 2>/dev/null | awk 'NR>1 {print $4}' | grep -oP ':\K\d+' | sort -n | uniq | paste -sd ',' -)
    echo "net_listen_ports=${listen_ports:-无}"

    # TCP 连接状态统计
    echo "net_conn_stats=$(ss -s 2>/dev/null | head -1)"
}

get_process_info() {
    # zombie
    local zombie
    zombie=$(ps -eo stat 2>/dev/null | grep -c '^Z' || echo 0)
    echo "proc_zombie=$zombie"

    # 总进程数
    local total_proc
    total_proc=$(ps aux 2>/dev/null | wc -l)
    echo "proc_total=$total_proc"

    # CPU Top 3
    echo "proc_top_cpu=$(ps -eo pid,pcpu,comm --sort=-pcpu 2>/dev/null | head -4 | tail -3 | awk '{printf "%s(%.1f%%) ", $3, $2}')"

    # MEM Top 3
    echo "proc_top_mem=$(ps -eo pid,pmem,comm --sort=-pmem 2>/dev/null | head -4 | tail -3 | awk '{printf "%s(%.1f%%) ", $3, $2}')"
}

# ── 收集所有信息 ──
declare -A HEALTH_DATA

while IFS='=' read -r key value; do
    HEALTH_DATA["$key"]="$value"
done < <(
    get_cpu_info
    get_mem_info
    get_disk_info
    get_net_info
    get_process_info
)

# ── 输出 ──

print_text() {
    cat <<EOF
╔══════════════════════════════════════════════════════╗
║             系统健康检查报告                         ║
╠══════════════════════════════════════════════════════╣
║ 主机: ${HOSTNAME}
║ 时间: ${TIMESTAMP}
║ 内核: ${KERNEL}
║ 运行时间: ${UPTIME_STR}
╠══════════════════════════════════════════════════════╣
║ 【CPU】
║   使用率:   ${HEALTH_DATA[cpu_usage]:-N/A}%
║   负载:     ${HEALTH_DATA[cpu_load]:-N/A}
║   核心数:   ${HEALTH_DATA[cpu_cores]:-N/A}
╠══════════════════════════════════════════════════════╣
║ 【内存】
║   总量:     ${HEALTH_DATA[mem_total]:-N/A}
║   可用:     ${HEALTH_DATA[mem_avail]:-N/A}
║   使用率:   ${HEALTH_DATA[mem_usage_pct]:-N/A}
╠══════════════════════════════════════════════════════╣
║ 【磁盘】
║   各挂载点: ${HEALTH_DATA[disk_mounts]:-N/A}
║   使用>80%: ${HEALTH_DATA[disk_critical]:-无}
╠══════════════════════════════════════════════════════╣
║ 【网络】
║   网关连通: ${HEALTH_DATA[net_gateway]:-N/A}
║   监听端口: ${HEALTH_DATA[net_listen_ports]:-N/A}
╠══════════════════════════════════════════════════════╣
║ 【进程】
║   僵尸进程: ${HEALTH_DATA[proc_zombie]:-N/A}
║   总进程数: ${HEALTH_DATA[proc_total]:-N/A}
║   CPU Top:  ${HEALTH_DATA[proc_top_cpu]:-N/A}
║   MEM Top:  ${HEALTH_DATA[proc_top_mem]:-N/A}
╚══════════════════════════════════════════════════════╝
EOF
}

print_json() {
    local json="{"
    json+="\"hostname\":\"${HOSTNAME}\","
    json+="\"timestamp\":\"${TIMESTAMP}\","
    json+="\"kernel\":\"${KERNEL}\","
    for key in "${!HEALTH_DATA[@]}"; do
        json+="\"${key}\":\"${HEALTH_DATA[$key]}\","
    done
    json="${json%,}"
    json+="}"
    echo "$json"
}

print_html() {
    cat <<EOF
<!DOCTYPE html>
<html><head><meta charset="UTF-8"><title>健康检查报告</title>
<style>body{font-family:monospace;max-width:900px;margin:0 auto;padding:20px}
table{border-collapse:collapse;width:100%}th,td{border:1px solid #ddd;padding:8px;text-align:left}
th{background:#4CAF50;color:white}tr:nth-child(even){background:#f2f2f2}
.warn{color:orange;font-weight:bold}.critical{color:red;font-weight:bold}</style></head>
<body>
<h1>系统健康检查报告</h1>
<p>主机: ${HOSTNAME} | 时间: ${TIMESTAMP} | 内核: ${KERNEL}</p>
<table>
EOF
    for key in "${!HEALTH_DATA[@]}"; do
        local cls=""
        [[ "$key" == *critical* && "${HEALTH_DATA[$key]}" != "无" ]] && cls=" class='critical'"
        echo "<tr><td>${key}</td><td${cls}>${HEALTH_DATA[$key]}</td></tr>"
    done
    echo "</table></body></html>"
}

case "$OUTPUT_MODE" in
    text) print_text ;;
    json) print_json ;;
    html) print_html ;;
esac
```

# 7. 部署脚本

## 7.1 Git pull + 构建 + 重启服务

```bash
#!/bin/bash
#
# deploy.sh - 应用部署脚本（Git 拉取 → 构建 → 平滑重启）
#
# 用法:
#   ./deploy.sh --env production --branch main
#
# 配置项可通过环境变量覆盖:
#   DEPLOY_DIR    - 部署目录（默认: /opt/myapp）
#   BRANCH        - Git 分支（默认: main）
#   BUILD_CMD     - 构建命令（默认: npm run build）
#   RESTART_CMD   - 重启命令（默认: systemctl restart myapp）
#   HEALTH_URL    - 健康检查 URL（默认: http://localhost:8080/health）
#   MAX_WAIT      - 健康检查最大等待秒数（默认: 60）

set -euo pipefail

DEPLOY_DIR="${DEPLOY_DIR:-/opt/myapp}"
BRANCH="${BRANCH:-main}"
BUILD_CMD="${BUILD_CMD:-npm run build}"
RESTART_CMD="${RESTART_CMD:-systemctl restart myapp}"
HEALTH_URL="${HEALTH_URL:-http://localhost:8080/health}"
MAX_WAIT="${MAX_WAIT:-60}"
BACKUP_ENABLED="${BACKUP_ENABLED:-true}"
KEEP_RELEASES="${KEEP_RELEASES:-5}"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log_info()  { echo -e "${GREEN}[INFO]${NC}  $(date '+%H:%M:%S') $*"; }
log_warn()  { echo -e "${YELLOW}[WARN]${NC}  $(date '+%H:%M:%S') $*"; }
log_error() { echo -e "${RED}[ERROR]${NC} $(date '+%H:%M:%S') $*" >&2; }

cleanup() {
    log_warn "部署中断，执行清理..."
    # 在这里恢复备份等
}
trap cleanup EXIT

# ── 参数解析 ──
while (( $# > 0 )); do
    case "$1" in
        --env)    ENV="$2"; shift 2 ;;
        --branch) BRANCH="$2"; shift 2 ;;
        --dir)    DEPLOY_DIR="$2"; shift 2 ;;
        -h|--help)
            echo "用法: $0 [--env prod] [--branch main] [--dir /opt/app]"
            exit 0 ;;
        *) shift ;;
    esac
done

log_info "===== 部署开始 ====="
log_info "目录: $DEPLOY_DIR | 分支: $BRANCH"

if [[ ! -d "$DEPLOY_DIR" ]]; then
    log_error "部署目录不存在: $DEPLOY_DIR"
    exit 1
fi

cd "$DEPLOY_DIR"

# ── 1. 部署前备份 ──
if [[ "$BACKUP_ENABLED" == true ]]; then
    BACKUP_NAME="backup_$(date '+%Y%m%d_%H%M%S').tar.gz"
    log_info "创建备份: ${BACKUP_NAME}"
    tar -czf "/tmp/${BACKUP_NAME}" --exclude='node_modules' --exclude='.git' \
        -C "$DEPLOY_DIR" . 2>/dev/null || true
fi

# ── 2. Git 拉取 ──
log_info "Git fetch + checkout ${BRANCH}..."
git fetch origin "$BRANCH"
git reset --hard "origin/${BRANCH}"
git clean -fd

CURRENT_COMMIT=$(git rev-parse --short HEAD)
log_info "当前 commit: $CURRENT_COMMIT"

# ── 3. 构建（如果定义了构建命令） ──
if [[ -n "$BUILD_CMD" ]]; then
    log_info "执行构建: $BUILD_CMD"
    eval "$BUILD_CMD"
    log_info "构建完成。"
fi

# ── 4. 重启服务 ──
log_info "重启服务: $RESTART_CMD"
eval "$RESTART_CMD"

# ── 5. 健康检查 ──
if [[ -n "$HEALTH_URL" ]]; then
    log_info "等待服务健康（最多 ${MAX_WAIT}s）..."
    waited=0
    while (( waited < MAX_WAIT )); do
        if curl -sf -o /dev/null "$HEALTH_URL"; then
            log_info "服务健康检查通过！"
            break
        fi
        sleep 2
        ((waited+=2))
        echo -n "."
    done
    echo ""
    if (( waited >= MAX_WAIT )); then
        log_error "健康检查超时！服务可能异常。"
        # 这里可以触发自动回滚
        exit 1
    fi
fi

# ── 6. 清理旧备份 ──
if [[ "$BACKUP_ENABLED" == true ]]; then
    BACKUP_DIR="/tmp"
    ls -1t "${BACKUP_DIR}"/backup_*.tar.gz 2>/dev/null | tail -n +"$((KEEP_RELEASES + 1))" | xargs rm -f 2>/dev/null || true
fi

trap - EXIT

log_info "===== 部署成功 ====="
echo "  Commit:  $CURRENT_COMMIT"
echo "  分支:    $BRANCH"
echo "  目录:    $DEPLOY_DIR"
```

## 7.2 蓝绿部署简化版

```bash
#!/bin/bash
#
# blue_green_deploy.sh - 蓝绿部署简化版
#
# 原理：维护两套环境（blue / green），部署到非活跃环境，
#       验证通过后切换流量。
#
# 用法:
#   ./blue_green_deploy.sh --nginx-conf /etc/nginx/conf.d/myapp.conf

set -euo pipefail

APP_PORT_BLUE="${APP_PORT_BLUE:-8081}"
APP_PORT_GREEN="${APP_PORT_GREEN:-8082}"
NGINX_CONF="${NGINX_CONF:-/etc/nginx/conf.d/myapp.conf}"
HEALTH_URL_BASE="${HEALTH_URL_BASE:-http://localhost}"
MAX_WAIT="${MAX_WAIT:-30}"
APP_START_CMD="${APP_START_CMD:-}"

ACTIVE_ENV_FILE="/tmp/.blue_green_active"

log_info()  { echo "[$(date '+%H:%M:%S')] [INFO]  $*"; }
log_error() { echo "[$(date '+%H:%M:%S')] [ERROR] $*" >&2; }

# 读取当前活跃环境
if [[ -f "$ACTIVE_ENV_FILE" ]]; then
    ACTIVE_ENV=$(cat "$ACTIVE_ENV_FILE")
else
    ACTIVE_ENV="blue"   # 默认首次从 blue 开始
fi

# 确定非活跃环境
if [[ "$ACTIVE_ENV" == "blue" ]]; then
    INACTIVE_ENV="green"
    INACTIVE_PORT="$APP_PORT_GREEN"
    ACTIVE_PORT="$APP_PORT_BLUE"
else
    INACTIVE_ENV="blue"
    INACTIVE_PORT="$APP_PORT_BLUE"
    ACTIVE_PORT="$APP_PORT_GREEN"
fi

log_info "当前活跃环境: ${ACTIVE_ENV} (端口: ${ACTIVE_PORT})"
log_info "目标部署环境: ${INACTIVE_ENV} (端口: ${INACTIVE_PORT})"

# ── 1. 部署到非活跃环境 ──
log_info "部署到 ${INACTIVE_ENV} 环境..."
# 假设启动脚本通过环境变量区分端口
# 实际中可能是 docker compose up、kubectl apply 等
if [[ -n "$APP_START_CMD" ]]; then
    PORT="$INACTIVE_PORT" ENV="$INACTIVE_ENV" eval "$APP_START_CMD"
fi

# ── 2. 健康检查 ──
log_info "对 ${INACTIVE_ENV} 环境进行健康检查..."
waited=0
while (( waited < MAX_WAIT )); do
    if curl -sf -o /dev/null "${HEALTH_URL_BASE}:${INACTIVE_PORT}/health"; then
        log_info "${INACTIVE_ENV} 环境健康检查通过！"
        break
    fi
    sleep 2
    ((waited+=2))
done

if (( waited >= MAX_WAIT )); then
    log_error "${INACTIVE_ENV} 健康检查超时，部署中止！"
    exit 1
fi

# ── 3. 切换流量 ──
log_info "切换 Nginx 流量到 ${INACTIVE_ENV} (端口 ${INACTIVE_PORT})..."

if [[ -f "$NGINX_CONF" ]]; then
    # 备份原配置
    cp "$NGINX_CONF" "${NGINX_CONF}.bak.$(date '+%Y%m%d_%H%M%S')"

    # 替换 upstream 指向的端口
    sed -i "s/server 127.0.0.1:${ACTIVE_PORT}/server 127.0.0.1:${INACTIVE_PORT}/g" "$NGINX_CONF"

    # 重载 Nginx
    if nginx -t 2>/dev/null; then
        nginx -s reload 2>/dev/null || systemctl reload nginx
        log_info "Nginx 重载成功，流量已切换到 ${INACTIVE_ENV}。"
    else
        log_error "Nginx 配置检查失败，回滚配置！"
        cp "${NGINX_CONF}.bak."* "$NGINX_CONF" 2>/dev/null
        exit 1
    fi
else
    log_error "Nginx 配置文件不存在: $NGINX_CONF"
    exit 1
fi

# ── 4. 更新活跃环境标记 ──
echo "$INACTIVE_ENV" > "$ACTIVE_ENV_FILE"

log_info "===== 蓝绿部署完成 ====="
log_info "新活跃环境: ${INACTIVE_ENV}"
log_info "旧环境 (${ACTIVE_ENV}) 仍在运行，确认无问题后可关闭。"
```

# 8. 数据管道（ETL 简化版）

## 8.1 日志提取 + 转换 + 入库

```bash
#!/bin/bash
#
# etl_simple.sh - 简化 ETL 管道：Nginx 日志 → 结构化提取 → SQLite 入库
#
# 用法:
#   ./etl_simple.sh /var/log/nginx/access.log
#
# 日志格式（combined）:
#   192.168.1.1 - - [10/May/2026:14:30:00 +0800] "GET /api/users HTTP/1.1" 200 1234 "-" "curl/7.0"

set -euo pipefail

LOG_FILE="${1:?用法: $0 <access.log>}"
DB_FILE="${DB_FILE:-./logs.db}"
BATCH_SIZE="${BATCH_SIZE:-1000}"

log() { echo "[$(date '+%H:%M:%S')] $*"; }

# ── 1. Extract（提取） ──
log "阶段1: Extract - 解析日志..."

TMP_EXTRACT=$(mktemp)

# Nginx combined log 正则提取
# 提取: IP, 时间, 方法, URL, 状态码, 响应大小, User-Agent
awk '
match($0, /^([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+) .* \[([^\]]+)\] "([A-Z]+) ([^"]*) HTTP\/[0-9.]+" ([0-9]+) ([0-9]+) .* "([^"]*)"$/, arr) {
    ip     = arr[1]
    time   = arr[2]
    method = arr[3]
    url    = arr[4]
    status = arr[5]
    size   = arr[6]
    ua     = arr[7]

    # 格式化时间: 10/May/2026:14:30:00 +0800 → 2026-05-10 14:30:00
    gsub(/\//, " ", time)
    gsub(/:/, " ", time)
    split(time, t, " ")
    month_map["Jan"]="01"; month_map["Feb"]="02"; month_map["Mar"]="03"
    month_map["Apr"]="04"; month_map["May"]="05"; month_map["Jun"]="06"
    month_map["Jul"]="07"; month_map["Aug"]="08"; month_map["Sep"]="09"
    month_map["Oct"]="10"; month_map["Nov"]="11"; month_map["Dec"]="12"
    mon = month_map[t[2]]
    ts = t[3] "-" mon "-" t[1] " " t[4] ":" t[5] ":" t[6]

    printf "%s|%s|%s|%s|%s|%s|%s\n", ts, ip, method, url, status, size, ua
}
' "$LOG_FILE" > "$TMP_EXTRACT"

line_count=$(wc -l < "$TMP_EXTRACT")
log "提取完成，共 ${line_count} 行"

# ── 2. Transform（转换） ──
log "阶段2: Transform - 数据清洗/增强..."

TMP_TRANSFORM=$(mktemp)

# 示例转换：
#   - 截断过长的 URL（>500 字符）
#   - 状态码标准化
#   - 添加请求类别（api/static/page）
while IFS='|' read -r ts ip method url status size ua; do
    # URL 截断
    if (( ${#url} > 500 )); then
        url="${url:0:500}..."
    fi

    # 请求类别
    if [[ "$url" == /api/* ]]; then category="api"
    elif [[ "$url" =~ \.(css|js|png|jpg|gif|svg|ico)$ ]]; then category="static"
    else category="page"
    fi

    # 状态码分类
    if   (( status >= 200 && status < 300 )); then status_class="2xx"
    elif (( status >= 300 && status < 400 )); then status_class="3xx"
    elif (( status >= 400 && status < 500 )); then status_class="4xx"
    elif (( status >= 500               )); then status_class="5xx"
    else status_class="other"
    fi

    echo "${ts}|${ip}|${method}|${url}|${status}|${status_class}|${size}|${category}|${ua:0:200}"
done < "$TMP_EXTRACT" > "$TMP_TRANSFORM"

log "转换完成。"

# ── 3. Load（入库） ──
log "阶段3: Load - SQLite 入库..."

# 创建表
sqlite3 "$DB_FILE" <<'DDL'
CREATE TABLE IF NOT EXISTS access_logs (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    ts           TEXT,
    ip           TEXT,
    method       TEXT,
    url          TEXT,
    status       INTEGER,
    status_class TEXT,
    size         INTEGER,
    category     TEXT,
    user_agent   TEXT
);
CREATE INDEX IF NOT EXISTS idx_ts ON access_logs(ts);
CREATE INDEX IF NOT EXISTS idx_ip ON access_logs(ip);
CREATE INDEX IF NOT EXISTS idx_status ON access_logs(status);
DDL

# 导入数据（使用临时文件拼接 SQL）
IMPORT_SQL=$(mktemp)
echo "BEGIN TRANSACTION;" > "$IMPORT_SQL"

while IFS='|' read -r ts ip method url status status_class size category ua; do
    # 转义单引号
    url_esc="${url//\'/\'\'}"
    ua_esc="${ua//\'/\'\'}"
    echo "INSERT INTO access_logs(ts,ip,method,url,status,status_class,size,category,user_agent) VALUES('$ts','$ip','$method','$url_esc',$status,'$status_class',$size,'$category','$ua_esc');" >> "$IMPORT_SQL"
done < "$TMP_TRANSFORM"

echo "COMMIT;" >> "$IMPORT_SQL"

sqlite3 "$DB_FILE" < "$IMPORT_SQL"

log "入库完成。数据库: $DB_FILE"

# ── 4. 简单验证 ──
log "验证 - 数据统计:"
sqlite3 "$DB_FILE" <<'SQL'
.mode column
.headers on
SELECT '总记录数', COUNT(*) FROM access_logs
UNION ALL
SELECT '唯一IP', COUNT(DISTINCT ip) FROM access_logs
UNION ALL
SELECT '错误率(5xx)', printf('%.2f%%', 100.0*SUM(CASE WHEN status_class='5xx' THEN 1 ELSE 0 END)/COUNT(*)) FROM access_logs;
SQL

# ── 5. 清理 ──
rm -f "$TMP_EXTRACT" "$TMP_TRANSFORM" "$IMPORT_SQL"

log "===== ETL 完成 ====="
```

# 9. 告警通知

## 9.1 钉钉 / 企业微信 Webhook

```bash
#!/bin/bash
#
# webhook_notify.sh - Webhook 告警通知（支持钉钉、企业微信、飞书）
#
# 用法:
#   source webhook_notify.sh
#   dingtalk_notify "告警标题" "告警内容"
#   wecom_notify   "告警标题" "告警内容"
#   feishu_notify  "告警标题" "告警内容"
#
# 环境变量:
#   DINGTALK_WEBHOOK_URL   - 钉钉机器人 Webhook URL
#   WECOM_WEBHOOK_URL      - 企业微信机器人 Webhook URL
#   FEISHU_WEBHOOK_URL     - 飞书机器人 Webhook URL

set -euo pipefail

# ── 钉钉通知 ──
dingtalk_notify() {
    local title="${1:-告警}"
    local content="${2:-}"
    local url="${DINGTALK_WEBHOOK_URL:-}"
    local level="${3:-INFO}"

    if [[ -z "$url" ]]; then
        echo "[DingTalk] 未配置 DINGTALK_WEBHOOK_URL，跳过通知。" >&2
        return 0
    fi

    local level_color
    case "$level" in
        ERROR|error) level_color="#FF0000" ;;
        WARN|warn)   level_color="#FFA500" ;;
        *)           level_color="#008000" ;;
    esac

    local hostname="${HOSTNAME:-$(hostname)}"
    local timestamp="$(date '+%Y-%m-%d %H:%M:%S')"

    local payload=$(cat <<EOF
{
    "msgtype": "markdown",
    "markdown": {
        "title": "[${level}] ${title}",
        "text": "## ${title}\n\n> 级别: <font color=\"${level_color}\">${level}</font>\n> 主机: ${hostname}\n> 时间: ${timestamp}\n\n${content}"
    }
}
EOF
)

    curl -sf -X POST -H "Content-Type: application/json" -d "$payload" "$url" >/dev/null 2>&1 || {
        echo "[DingTalk] 发送失败" >&2
        return 1
    }
    echo "[DingTalk] 通知已发送: ${title}"
}

# ── 企业微信通知 ──
wecom_notify() {
    local title="${1:-告警}"
    local content="${2:-}"
    local url="${WECOM_WEBHOOK_URL:-}"

    if [[ -z "$url" ]]; then
        echo "[WeCom] 未配置 WECOM_WEBHOOK_URL，跳过通知。" >&2
        return 0
    fi

    local hostname="${HOSTNAME:-$(hostname)}"
    local timestamp="$(date '+%Y-%m-%d %H:%M:%S')"

    local payload=$(cat <<EOF
{
    "msgtype": "markdown",
    "markdown": {
        "content": "## ${title}\n> 主机: <font color=\"comment\">${hostname}</font>\n> 时间: ${timestamp}\n${content}"
    }
}
EOF
)

    curl -sf -X POST -H "Content-Type: application/json" -d "$payload" "$url" >/dev/null 2>&1 || {
        echo "[WeCom] 发送失败" >&2
        return 1
    }
    echo "[WeCom] 通知已发送: ${title}"
}

# ── 飞书通知 ──
feishu_notify() {
    local title="${1:-告警}"
    local content="${2:-}"
    local url="${FEISHU_WEBHOOK_URL:-}"
    local level="${3:-INFO}"

    if [[ -z "$url" ]]; then
        echo "[Feishu] 未配置 FEISHU_WEBHOOK_URL，跳过通知。" >&2
        return 0
    fi

    local level_color
    case "$level" in
        ERROR|error) level_color="red" ;;
        WARN|warn)   level_color="orange" ;;
        *)           level_color="green" ;;
    esac

    local hostname="${HOSTNAME:-$(hostname)}"
    local timestamp="$(date '+%Y-%m-%d %H:%M:%S')"

    local payload=$(cat <<EOF
{
    "msg_type": "interactive",
    "card": {
        "header": {
            "title": {
                "tag": "plain_text",
                "content": "[${level}] ${title}"
            },
            "template": "${level_color}"
        },
        "elements": [
            {
                "tag": "markdown",
                "content": "**主机:** ${hostname}\n**时间:** ${timestamp}\n\n${content}"
            }
        ]
    }
}
EOF
)

    curl -sf -X POST -H "Content-Type: application/json" -d "$payload" "$url" >/dev/null 2>&1 || {
        echo "[Feishu] 发送失败" >&2
        return 1
    }
    echo "[Feishu] 通知已发送: ${title}"
}

# ── 统一通知接口 ──
notify_all() {
    local title="$1"
    local content="$2"
    local level="${3:-INFO}"

    dingtalk_notify "$title" "$content" "$level" &
    wecom_notify   "$title" "$content" &
    feishu_notify  "$title" "$content" "$level" &
    wait
}
```

## 9.2 邮件发送

```bash
#!/bin/bash
#
# mail_notify.sh - 邮件发送脚本（支持 sendmail / msmtp / SMTP）
#
# 用法:
#   ./mail_notify.sh --to admin@example.com --subject "告警" --body "内容"
#
# 方式 1: 系统 sendmail
# 方式 2: msmtp（需配置 ~/.msmtprc）
# 方式 3: curl 直连 SMTP（仅支持 587 STARTTLS）

set -euo pipefail

TO=""
SUBJECT=""
BODY=""
CC=""
ATTACHMENTS=()
MAIL_METHOD="${MAIL_METHOD:-sendmail}"      # sendmail | msmtp | smtp

usage() {
    cat <<EOF
用法: $0 --to 收件人 --subject 主题 --body 正文 [选项]

选项:
    --to EMAIL        收件人地址（必填）
    --cc EMAIL        抄送地址
    --subject TEXT    邮件主题（必填）
    --body TEXT       邮件正文
    --body-file FILE  从文件读取正文
    --attach FILE     附件（可多次指定）
    --method METHOD   发送方式: sendmail/msmtp/smtp（默认: sendmail）
    -h, --help        帮助
EOF
    exit "${1:-0}"
}

while (( $# > 0 )); do
    case "$1" in
        --to)        TO="$2";        shift 2 ;;
        --cc)        CC="$2";        shift 2 ;;
        --subject)   SUBJECT="$2";   shift 2 ;;
        --body)      BODY="$2";      shift 2 ;;
        --body-file) BODY=$(cat "$2"); shift 2 ;;
        --attach)    ATTACHMENTS+=("$2"); shift 2 ;;
        --method)    MAIL_METHOD="$2"; shift 2 ;;
        -h|--help)   usage 0 ;;
        *)           shift ;;
    esac
done

if [[ -z "$TO" || -z "$SUBJECT" ]]; then
    echo "错误: --to 和 --subject 是必填项"
    usage 1
fi

BODY="${BODY:-无内容}"

# ── 方式 1: sendmail ──
send_with_sendmail() {
    if ! command -v sendmail &>/dev/null; then
        echo "错误: sendmail 不可用" >&2
        return 1
    fi

    {
        echo "To: $TO"
        [[ -n "$CC" ]] && echo "Cc: $CC"
        echo "Subject: $SUBJECT"
        echo "Content-Type: text/plain; charset=UTF-8"
        echo ""
        echo "$BODY"
    } | sendmail -t
}

# ── 方式 2: msmtp ──
send_with_msmtp() {
    if ! command -v msmtp &>/dev/null; then
        echo "错误: msmtp 不可用" >&2
        return 1
    fi

    {
        echo "To: $TO"
        [[ -n "$CC" ]] && echo "Cc: $CC"
        echo "Subject: $SUBJECT"
        echo ""
        echo "$BODY"
    } | msmtp --read-recipients "$TO"
}

# ── 方式 3: curl + SMTP ──
send_with_smtp() {
    local smtp_host="${SMTP_HOST:-}"
    local smtp_port="${SMTP_PORT:-587}"
    local smtp_user="${SMTP_USER:-}"
    local smtp_pass="${SMTP_PASS:-}"

    if [[ -z "$smtp_host" || -z "$smtp_user" || -z "$smtp_pass" ]]; then
        echo "错误: 请设置 SMTP_HOST / SMTP_USER / SMTP_PASS 环境变量" >&2
        return 1
    fi

    curl -sf --url "smtps://${smtp_host}:${smtp_port}" \
        --ssl-reqd \
        --mail-from "$smtp_user" \
        --mail-rcpt "$TO" \
        --user "${smtp_user}:${smtp_pass}" \
        -T <(cat <<MAIL
From: ${smtp_user}
To: ${TO}
Subject: ${SUBJECT}
Content-Type: text/plain; charset=UTF-8

${BODY}
MAIL
        )
}

# ── 发送 ──
case "$MAIL_METHOD" in
    sendmail) send_with_sendmail ;;
    msmtp)    send_with_msmtp ;;
    smtp)     send_with_smtp ;;
    *)
        echo "错误: 未知发送方式: $MAIL_METHOD" >&2
        exit 1
        ;;
esac

echo "邮件已发送至: $TO"
```

# 10. 参数解析框架

## 10.1 getopt 长选项 + 子命令模式

```bash
#!/bin/bash
#
# argparser.sh - Shell 参数解析框架（getopt 长选项 + 子命令模式）
#
# 设计目标：像 git / docker 一样的子命令风格 CLI
#
# 用法:
#   ./myapp.sh [全局选项] <子命令> [子命令选项]
#
# 示例:
#   ./myapp.sh --verbose deploy --env production --branch main
#   ./myapp.sh backup --type full --compress
#
# 添加子命令: 定义函数 cmd_<name>() 并在 COMMANDS 数组中注册

set -euo pipefail

SCRIPT_NAME="$(basename "$0")"

# ── 全局变量 ──
GLOBAL_VERBOSE=false
GLOBAL_CONFIG=""
GLOBAL_DRY_RUN=false

# ── 已注册的子命令列表 ──
COMMANDS=("deploy" "backup" "status" "logs")

# ── 帮助 ──
global_usage() {
    cat <<EOF
用法: ${SCRIPT_NAME} [全局选项] <命令> [命令选项]

命令:
    deploy    部署应用
    backup    数据备份
    status    查看状态
    logs      查看日志

全局选项:
    -v, --verbose     详细输出
    -c, --config FILE 配置文件路径
    -n, --dry-run     空运行模式
    -h, --help        显示帮助

运行 '${SCRIPT_NAME} <命令> --help' 查看命令帮助。
EOF
    exit 0
}

# ── 子命令注册 ──
register() {
    local cmd="$1"
    COMMANDS+=("$cmd")
}

# ── 解析全局选项（不影响子命令参数） ──
parse_global_opts() {
    local args=()
    while (( $# > 0 )); do
        case "$1" in
            -v|--verbose)  GLOBAL_VERBOSE=true; shift ;;
            -c|--config)   GLOBAL_CONFIG="$2";  shift 2 ;;
            -n|--dry-run)  GLOBAL_DRY_RUN=true; shift ;;
            -h|--help)     global_usage ;;
            --)            shift; args+=("$@"); break ;;
            -*)            echo "未知全局选项: $1" >&2; global_usage ;;
            *)             args+=("$1"); shift ;;
        esac
    done
    set -- "${args[@]}"

    # 剩下的第一个参数必须是子命令
    if (( $# == 0 )); then
        echo "错误: 缺少子命令" >&2
        echo "可用命令: ${COMMANDS[*]}" >&2
        global_usage
    fi

    SUBCOMMAND="$1"
    shift
    SUBCOMMAND_ARGS=("$@")
}

# ── 通用子命令选项解析函数 ──
# 使用 getopt 解析长选项
parse_cmd_opts() {
    local opts="$1"
    shift

    # 将剩余参数传给 getopt
    local parsed
    parsed=$(getopt -o "$opts" -- "$@" 2>/dev/null) || {
        echo "选项解析失败" >&2
        return 1
    }
    eval set -- "$parsed"

    # 调用者需要通过 $@ 读取
    echo "$@"
}

# ==================== 子命令实现 ====================

cmd_deploy_usage() {
    cat <<EOF
用法: ${SCRIPT_NAME} deploy [选项]

选项:
    -e, --env ENV        部署环境（dev/staging/prod）
    -b, --branch BRANCH  Git 分支（默认: main）
    -t, --tag TAG        Git 标签
    --no-build           跳过构建步骤
    --no-restart         跳过重启步骤
    -h, --help           帮助

示例:
    ${SCRIPT_NAME} deploy --env production --branch main
    ${SCRIPT_NAME} -v deploy -e staging -t v2.1.0
EOF
    exit 0
}

cmd_deploy() {
    local env="dev"
    local branch="main"
    local tag=""
    local no_build=false
    local no_restart=false

    OPTS=$(getopt -o e:b:t:h -l env:,branch:,tag:,no-build,no-restart,help \
        -n "${SCRIPT_NAME} deploy" -- "$@") || exit 1
    eval set -- "$OPTS"

    while true; do
        case "$1" in
            -e|--env)    env="$2";          shift 2 ;;
            -b|--branch) branch="$2";       shift 2 ;;
            -t|--tag)    tag="$2";          shift 2 ;;
            --no-build)  no_build=true;     shift ;;
            --no-restart)no_restart=true;   shift ;;
            -h|--help)   cmd_deploy_usage ;;
            --)          shift; break ;;
            *)           echo "内部错误"; exit 1 ;;
        esac
    done

    echo "=== 部署命令 ==="
    echo "  环境:   $env"
    echo "  分支:   $branch"
    [[ -n "$tag" ]]               && echo "  标签:   $tag"
    $GLOBAL_VERBOSE               && echo "  [verbose 模式]"
    $GLOBAL_DRY_RUN               && echo "  [dry-run 模式]"
    $no_build                     && echo "  [跳过构建]"
    $no_restart                   && echo "  [跳过重启]"

    # 实际部署逻辑
    if [[ "$GLOBAL_DRY_RUN" == true ]]; then
        echo "  (空运行，未实际执行)"
    else
        echo "  正在部署..."
        # git checkout, build, restart...
        echo "  部署完成。"
    fi
}

cmd_backup_usage() {
    cat <<EOF
用法: ${SCRIPT_NAME} backup [选项]

选项:
    -t, --type TYPE       备份类型: full|incremental（默认: full）
    --compress            压缩备份
    -o, --output DIR      备份输出目录（默认: /data/backup）
    -r, --retain N        保留最近 N 个备份（默认: 30）
    -h, --help            帮助

示例:
    ${SCRIPT_NAME} backup --type full --compress
    ${SCRIPT_NAME} backup -t incremental -o /backup/app -r 7
EOF
    exit 0
}

cmd_backup() {
    local type="full"
    local compress=false
    local output="/data/backup"
    local retain=30

    OPTS=$(getopt -o t:o:r:h -l type:,compress,output:,retain:,help \
        -n "${SCRIPT_NAME} backup" -- "$@") || exit 1
    eval set -- "$OPTS"

    while true; do
        case "$1" in
            -t|--type)     type="$2";       shift 2 ;;
            --compress)    compress=true;   shift ;;
            -o|--output)   output="$2";     shift 2 ;;
            -r|--retain)   retain="$2";     shift 2 ;;
            -h|--help)     cmd_backup_usage ;;
            --)            shift; break ;;
            *)             echo "内部错误"; exit 1 ;;
        esac
    done

    echo "=== 备份命令 ==="
    echo "  类型:   $type"
    echo "  压缩:   $compress"
    echo "  输出:   $output"
    echo "  保留:   $retain 个"
    $GLOBAL_VERBOSE && echo "  [verbose 模式]"
    $GLOBAL_DRY_RUN && echo "  [dry-run 模式]"

    if [[ "$GLOBAL_DRY_RUN" == false ]]; then
        echo "  正在备份..."
        # 实际备份逻辑
        echo "  备份完成。"
    fi
}

cmd_status_usage() {
    cat <<EOF
用法: ${SCRIPT_NAME} status [选项]

选项:
    -f, --format FMT   输出格式: text|json（默认: text）
    --watch            持续监控模式
    -h, --help         帮助
EOF
    exit 0
}

cmd_status() {
    local format="text"
    local watch=false

    OPTS=$(getopt -o f:h -l format:,watch,help -n "${SCRIPT_NAME} status" -- "$@") || exit 1
    eval set -- "$OPTS"

    while true; do
        case "$1" in
            -f|--format) format="$2";    shift 2 ;;
            --watch)     watch=true;     shift ;;
            -h|--help)   cmd_status_usage ;;
            --)          shift; break ;;
            *)           echo "内部错误"; exit 1 ;;
        esac
    done

    echo "=== 状态 ==="
    case "$format" in
        json) echo '{"status":"running","version":"1.0.0","uptime":"3d 4h"}' | python3 -m json.tool 2>/dev/null || echo '{"status":"running"}' ;;
        *)    echo "  状态: running | 版本: 1.0.0 | 运行: 3d 4h" ;;
    esac
}

cmd_logs_usage() {
    cat <<EOF
用法: ${SCRIPT_NAME} logs [选项]

选项:
    -n, --lines N       显示最近 N 行（默认: 100）
    -f, --follow        跟随模式
    --level LEVEL       日志级别: debug|info|warn|error（默认: info）
    -h, --help          帮助
EOF
    exit 0
}

cmd_logs() {
    local lines=100
    local follow=false
    local level="info"

    OPTS=$(getopt -o n:l:fh -l lines:,level:,follow,help -n "${SCRIPT_NAME} logs" -- "$@") || exit 1
    eval set -- "$OPTS"

    while true; do
        case "$1" in
            -n|--lines)  lines="$2";     shift 2 ;;
            -f|--follow) follow=true;    shift ;;
            -l|--level)  level="$2";     shift 2 ;;
            -h|--help)   cmd_logs_usage ;;
            --)          shift; break ;;
            *)           echo "内部错误"; exit 1 ;;
        esac
    done

    echo "=== 日志 ==="
    echo "  行数:  $lines | 跟随: $follow | 级别: $level"
    # 模拟日志输出
    for i in $(seq 1 ${lines}); do
        echo "[2026-05-$(printf '%02d' $((RANDOM % 30 + 1))) 10:$(printf '%02d' $((RANDOM % 60))):$(printf '%02d' $((RANDOM % 60)))] [$level] 日志行 $i - 这是示例日志内容"
    done | tail -n "$lines"
}

# ==================== 主入口 ====================

main() {
    parse_global_opts "$@"

    # 查找并执行子命令
    local cmd_func="cmd_${SUBCOMMAND}"

    # 检查函数是否存在
    if ! declare -f "$cmd_func" >/dev/null 2>&1; then
        echo "错误: 未知子命令 '$SUBCOMMAND'" >&2
        echo "可用命令: ${COMMANDS[*]}" >&2
        exit 1
    fi

    # 执行子命令
    "$cmd_func" "${SUBCOMMAND_ARGS[@]}"
}

main "$@"
```

## 10.2 子命令分发框架（简化版）

```bash
#!/bin/bash
#
# subcommand_framework.sh - 子命令分发的最简实现
#
# 适用场景：脚本功能模块化，每个操作作为独立子命令
#
# 使用方式:
#   1. 定义函数: cmd_<名称>() { ... }
#   2. 调用: main "$@"

set -euo pipefail

SCRIPT_NAME="$(basename "$0")"

# ── 帮助函数 ──
print_help() {
    cat <<EOF
用法: ${SCRIPT_NAME} <命令> [选项]

命令:
EOF
    # 自动发现所有 cmd_ 前缀的函数
    declare -F | grep -oP 'cmd_\K\w+' | while read -r cmd; do
        printf "    %-15s %s\n" "$cmd" "$(declare -f "cmd_${cmd}_desc" &>/dev/null && "cmd_${cmd}_desc" || echo "（无描述）")"
    done
    exit 0
}

# ── 子命令注册宏 ──
# 用法: register "start" "启动服务" cmd_start
register() {
    :
}

# ── 自动分发 ──
dispatch() {
    local cmd="${1:-}"

    if [[ -z "$cmd" || "$cmd" == "-h" || "$cmd" == "--help" ]]; then
        print_help
    fi

    local func="cmd_${cmd}"
    if declare -f "$func" >/dev/null 2>&1; then
        shift
        "$func" "$@"
    else
        echo "错误: 未知命令 '$cmd'" >&2
        echo "运行 '$SCRIPT_NAME --help' 查看可用命令。" >&2
        exit 1
    fi
}

# ==================== 示例子命令 ====================

cmd_start_desc() { echo "启动服务"; }
cmd_start() {
    echo "服务已启动。PID=$$"
}

cmd_stop_desc() { echo "停止服务"; }
cmd_stop() {
    echo "服务已停止。"
}

cmd_restart_desc() { echo "重启服务"; }
cmd_restart() {
    echo "重启中..."
    cmd_stop
    sleep 1
    cmd_start
}

cmd_info_desc() { echo "显示系统信息"; }
cmd_info() {
    cat <<EOF
主机名:   $(hostname)
内核版本: $(uname -r)
运行时间: $(uptime -p 2>/dev/null || uptime | awk -F',' '{print $1}')
当前用户: $(whoami)
EOF
}

# ── 主入口 ──
dispatch "$@"
```
