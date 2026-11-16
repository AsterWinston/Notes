
# 1. for 循环

## 1.1 列表迭代

```bash
# 基本形式
for var in item1 item2 item3; do
    echo "$var"
done

# 空格分隔的列表
for fruit in apple banana orange; do
    echo "水果: $fruit"
done

# 范围迭代（使用大括号扩展）
for i in {1..5}; do
    echo "第 $i 次"
done

# 步长大括号扩展
for i in {0..20..5}; do
    echo "$i"          # 0 5 10 15 20
done

# 通配符迭代
for file in *.txt; do
    [ -f "$file" ] || continue   # 安全保护
    echo "处理: $file"
done

# 省略 in 列表：等价 for var in "$@"
for arg; do
    echo "$arg"
done
```

## 1.2 命令替换列表

```bash
# 遍历命令输出（默认按 IFS 分割）
for user in $(cut -d: -f1 /etc/passwd); do
    echo "$user"
done

# 遍历文件中的每一行（逐行处理）
while IFS= read -r line; do
    echo "$line"
done < file.txt

# 为什么不用 for：for 会按 IFS 逐词分割，行内空格会导致问题
# 正确方式是用 while read
```

## 1.3 C 风格 for

```bash
# Bash 特有的 C 风格 for（非 POSIX）
for (( i=0; i<10; i++ )); do
    echo "$i"
done

# 多个变量
for (( i=0, j=10; i<j; i++, j-- )); do
    echo "i=$i, j=$j"
done

# 无限循环
for (( ; ; )); do
    echo "永远不会停..."
    sleep 1
    (( count++ ))
    (( count >= 10 )) && break
done
```

# 2. while 循环

```bash
while 条件; do
    命令
done
```

## 2.1 基本模式

```bash
# 计数器模式
count=1
while [ $count -le 5 ]; do
    echo "计数: $count"
    ((count++))
done

# 逐行读取文件（最常用模式）
while IFS= read -r line; do
    echo "[$line]"
done < input.txt

# 读取命令输出（管道避免子shell问题）
while IFS= read -r line; do
    echo "$line"
done < <(ls -l /tmp)

# 无限循环
while true; do
    date
    sleep 1
done

# 条件监控
while ping -c 1 -W 1 8.8.8.8 > /dev/null 2>&1; do
    echo "网络正常: $(date)"
    sleep 5
done
echo "网络断开: $(date)"
```

## 2.2 多列/多变量读取

```bash
# 读取 /etc/passwd（按冒号分割）
while IFS=: read -r user pass uid gid desc home shell; do
    echo "用户: $user, UID: $uid, Shell: $shell"
done < /etc/passwd

# 按空格读取多列
while read -r col1 col2 col3 rest; do
    echo "[$col1] [$col2] [$col3] [$rest]"
done << 'EOF'
name age city extra info here
Alice 30 Beijing more stuff
Bob 25 Shanghai other data
EOF
```

## 2.3 进程替换 < <(cmd) 替代管道

这是一个关键的 Bash 技巧：

```bash
# 问题：管道创建子 shell，循环内修改变量不生效
sum=0
seq 1 10 | while read -r n; do
    ((sum += n))
done
echo "sum=$sum"       # sum=0（子 shell 中的修改在父 shell 看不到！）

# 解决方案：进程替换
sum=0
while read -r n; do
    ((sum += n))
done < <(seq 1 10)
echo "sum=$sum"       # sum=55（正确）

# 或使用 Here String 替代
sum=0
while read -r n; do
    ((sum += n))
done <<< "$(seq 1 10)"
echo "sum=$sum"       # sum=55
```

# 3. until 循环

`until` 与 `while` 相反：当条件为**假**时继续，条件为**真**时退出。

```bash
until 条件; do
    命令
done

# 等待文件出现
echo "等待 .ready 文件..."
until [ -f /tmp/.ready ]; do
    sleep 1
done
echo "检测到文件！"

# 计数器等价于 while
count=1
until [ $count -gt 5 ]; do
    echo "$count"
    ((count++))
done

# 等待服务就绪
until curl -s http://localhost:8080/health > /dev/null 2>&1; do
    echo "等待服务启动..."
    sleep 2
done
echo "服务已就绪"
```

# 4. select 菜单

`select` 自动生成带编号的菜单列表，配合 `PS3` 提示符使用。

```bash
PS3="请选择: "
select option in 启动 停止 重启 退出; do
    case "$option" in
        启动)
            echo "服务启动中..."
            ;;
        停止)
            echo "服务停止中..."
            ;;
        重启)
            echo "服务重启中..."
            ;;
        退出)
            echo "再见"
            break
            ;;
        *)
            echo "无效选项"
            ;;
    esac
done
```

# 5. break 与 continue

```bash
# break：跳出循环
for i in {1..10}; do
    if [ "$i" -eq 5 ]; then
        break
    fi
    echo "$i"
done
# 输出 1 2 3 4

# break N：跳出 N 层嵌套循环
for i in {1..3}; do
    for j in {1..3}; do
        if [ "$j" -eq 2 ]; then
            break 2    # 跳出两层循环
        fi
        echo "i=$i j=$j"
    done
done

# continue：跳过本轮
for i in {1..5}; do
    if [ "$i" -eq 3 ]; then
        continue
    fi
    echo "$i"
done
# 输出 1 2 4 5

# continue N：跳到第 N 层循环的下一次迭代
for i in {1..3}; do
    for j in {1..3}; do
        if [ "$j" -eq 2 ]; then
            continue 2    # 跳到外层 for 的下一次迭代
        fi
        echo "i=$i j=$j"
    done
done
```

# 6. 循环性能优化

## 6.1 避免循环内 fork 外部命令

```bash
# 慢：每次迭代都 fork 一个 sed 进程
for file in *.txt; do
    sed -i 's/old/new/g' "$file"
done

# 快：一次性处理所有文件
sed -i 's/old/new/g' *.txt

# 慢：循环内调用 date
for i in {1..1000}; do
    echo "$(date): line $i"
done

# 快：只调用一次 date，用 printf 格式化
timestamp=$(date)
for i in {1..1000}; do
    echo "$timestamp: line $i"
done
```

## 6.2 大括号展开 vs 外部命令

```bash
# 快：大括号展开（bash 内置）
for i in {1..1000}; do :; done

# 慢：外部命令 seq
for i in $(seq 1 1000); do :; done

# 更慢：seq 加命令替换
for i in $(seq 1 1000); do echo "$i"; done
```

## 6.3 有效的文件遍历

```bash
# 能用 glob 就不要 ls（ls 有外部调用开销，且文件名含空格会出问题）
for file in *.log; do
    echo "处理: $file"
done

# 并行处理（利用后台进程）
for file in *.log; do
    process_file "$file" &
done
wait    # 等待所有后台任务完成
```

# 7. 循环实战模式

## 7.1 批量重命名

```bash
# 模式 1：借助参数展开
for file in *.jpeg; do
    mv "$file" "${file%.jpeg}.jpg"
done

# 模式 2：带序号
count=1
for file in *; do
    ext="${file##*.}"
    mv "$file" "file_$(printf '%03d' $count).$ext"
    ((count++))
done
```

## 7.2 批量执行远程命令

```bash
# 遍历服务器列表执行命令
servers=(
    "server1.example.com"
    "server2.example.com"
    "server3.example.com"
)

for server in "${servers[@]}"; do
    echo "=== $server ==="
    ssh "$server" "uptime; df -h /"
done
```

## 7.3 进度条模拟

```bash
#!/bin/bash
total=50
for ((i=1; i<=total; i++)); do
    percent=$((i * 100 / total))
    filled=$((i * 50 / total))
    bar=$(printf '=%.0s' $(seq 1 $filled))
    printf "\r[%-50s] %3d%%" "$bar" "$percent"
    sleep 0.1
done
echo    # 完成后的换行
```

## 7.4 轮询等待

```bash
max_attempts=30
attempt=0

while [ $attempt -lt $max_attempts ]; do
    if curl -s http://localhost:8080/health > /dev/null; then
        echo "服务就绪"
        break
    fi
    attempt=$((attempt + 1))
    echo "等待中... ($attempt/$max_attempts)"
    sleep 2
done

if [ $attempt -eq $max_attempts ]; then
    echo "服务启动超时"
    exit 1
fi
```

## 7.5 并行任务处理

```bash
#!/bin/bash
# 并行处理多个任务，控制并发数
max_jobs=4
running=0

for task in tasks/*; do
    # 等待直到有空闲的槽位
    while [ $running -ge $max_jobs ]; do
        wait -n       # 等待任意一个子进程完成
        ((running--))
    done

    process "$task" &
    ((running++))
done

wait  # 等待所有任务完成
echo "全部完成"
```
