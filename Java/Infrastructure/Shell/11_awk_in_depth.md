
# 1. awk 概述

**awk** 是一种**编程语言**，专门用于文本处理和数据分析。它一次读取一行，按字段分割，执行模式-动作规则。

awk 的名字来源于三位创始人姓氏的首字母：**A**lfred **A**ho、Peter **W**einberger、Brian **K**ernighan。

```bash
awk '模式 { 动作 }' 文件
awk -F: '{print $1}' /etc/passwd
```

**核心执行模型：**

```
BEGIN { 动作 }                     ← 处理开始前
模式 { 动作 }                      ← 每行匹配时
模式 { 动作 }                      ← 可多个规则
END { 动作 }                       ← 所有行处理完后
```

# 2. 字段与记录

```bash
# awk 自动按 FS（字段分隔符，默认空白）分割每行
awk '{print $1, $3}' data.txt     # 第 1 和 第 3 个字段
awk '{print $NF}' data.txt        # 最后一个字段
awk '{print $(NF-1)}' data.txt    # 倒数第二个字段
awk '{print NF}' data.txt         # 字段数量
awk '{print NR, $0}' data.txt     # 行号 + 整行（$0 = 整行）

# 设置分隔符
awk -F: '{print $1}' /etc/passwd              # 冒号分隔
awk -F'[,;]' '{print $1}' file.txt             # 逗号或分号分隔
awk 'BEGIN{FS=":"; OFS="\t"} {print $1,$3}'    # 输入冒号，输出 Tab
```

**内置变量速查：**

| 变量 | 含义 |
|------|------|
| `$0` | 整行内容 |
| `$1, $2, ...` | 第 N 个字段 |
| `NF` | 当前行的字段数量 |
| `NR` | 当前处理的总行数 |
| `FNR` | 当前文件的行号（多文件时各文件独立计数） |
| `FS` | 输入字段分隔符（默认 `[ \t\n]+`） |
| `OFS` | 输出字段分隔符（默认空格） |
| `RS` | 输入记录分隔符（默认换行） |
| `ORS` | 输出记录分隔符（默认换行） |
| `FILENAME` | 当前处理的文件名 |
| `ARGC` | 命令行参数个数 |
| `ARGV` | 命令行参数数组 |

# 3. 模式匹配

```bash
# 正则匹配
awk '/error/' log.txt                          # 包含 error 的行
awk '/error/ || /warn/' log.txt                # error 或 warn
awk '!/comment/' file.txt                      # 不包含 comment 的行
awk '$2 ~ /^[0-9]+$/' file.txt                 # 第 2 字段全为数字

# 行号范围
awk 'NR==1,NR==5' file.txt                     # 第 1~5 行
awk 'NR>=10 && NR<=20' file.txt                # 第 10~20 行

# 字段条件
awk '$3 > 100' file.txt                        # 第 3 字段 > 100
awk '$1 == "Alice"' file.txt                   # 第 1 字段等于 Alice
awk 'length($0) > 80' file.txt                 # 行长度 > 80
awk '$NF ~ /\.conf$/' file.txt                 # 最后字段以 .conf 结尾

# BEGIN / END
awk 'BEGIN {print "开始处理"} {print} END {print "处理结束共", NR, "行"}' file.txt
```

# 4. 变量与运算

```bash
# awk 支持数字和字符串运算，自动类型转换
awk '{sum += $1} END {print sum}' numbers.txt                    # 求和
awk '{sum += $1; count++} END {print sum/count}' numbers.txt     # 求平均
awk '{if ($1 > max) max=$1} END {print max}' numbers.txt         # 求最大值

# 字符串操作
awk '{print toupper($1)}' file.txt             # 转大写
awk '{print tolower($1)}' file.txt             # 转小写
awk '{print length($1)}' file.txt              # 字符串长度
awk '{print substr($1, 1, 3)}' file.txt        # 子串
awk '{print index($0, "error")}' file.txt      # 查找位置
awk '{print match($0, /[0-9]+/)}' file.txt     # 正则匹配位置
awk '{gsub(/old/, "new"); print}' file.txt     # 全局替换
awk '{sub(/old/, "new"); print}' file.txt      # 仅替换第一个
```

# 5. 数组（关联数组）

```bash
# awk 的数组本质是关联数组（键值对）
# 统计单词频率
awk '{for(i=1;i<=NF;i++) count[$i]++} END {for(w in count) print w, count[w]}' file.txt

# 去重
awk '!seen[$0]++' file.txt

# 按值排序（需要借助外部 sort）
awk '{count[$1]++} END {for(k in count) print count[k], k}' file.txt | sort -rn

# 多维数组（用 SUBSEP 组合键）
awk '{matrix[$1, $2] = $3} END {print matrix["A", "B"]}' file.txt

# 判断键是否存在
awk '{if ("key" in arr) print "exists"}'
```

# 6. 流程控制

```bash
# if-else
awk '{
    if ($3 > 80) {print $1, "优秀"}
    else if ($3 > 60) {print $1, "及格"}
    else {print $1, "不及格"}
}' scores.txt

# for 循环
awk '{for(i=1; i<=NF; i++) if($i ~ /^[0-9]+$/) sum+=$i} END {print sum}' file.txt

# while 循环
awk '{i=1; while(i<=NF) {print $i; i++}}' file.txt

# next（跳过当前行）
awk '/^#/ {next} {print}' file.txt    # 跳过注释行

# exit（终止）
awk '{print; if(NR>=10) exit}' file.txt   # 只处理前 10 行
```

# 7. 内置函数

```bash
# 数学函数
awk '{print int($1)}'          # 取整
awk '{print rand()}'           # 随机数 [0,1)
awk '{print sqrt($1)}'         # 平方根
awk '{print sin($1)}'          # 正弦（弧度）
awk '{print log($1)}'          # 自然对数
awk '{print exp($1)}'          # e 的幂

# 字符串函数
awk '{print sprintf("%05d", $1)}'     # 格式化
awk '{$1=""; print substr($0,2)}'     # 删除第一列
```

# 8. 格式化输出

```bash
# print（自动换行，空格分割字段）
awk '{print $1, $3}' file.txt

# printf（不自动换行，手动指定格式）
awk '{printf "%-20s %5d\n", $1, $2}' file.txt

# 常用 printf 格式
# %s  字符串    %d  整数    %f  浮点数
# %-20s  左对齐，宽度 20
# %05d   补齐 0，宽度 5
# %.2f   两位小数
```

# 9. 多文件处理

```bash
# 依次处理多个文件
awk '{print FILENAME, $0}' file1.txt file2.txt

# FNR 区分不同文件
awk 'FNR==1 {print "=== " FILENAME " ==="} {print}' file1.txt file2.txt

# 比较两个文件（内连接）
awk 'NR==FNR {arr[$1]=$2; next} {if($1 in arr) print $0, arr[$1]}' file1 file2

# 找出只在 file1 中的行（左差集）
awk 'NR==FNR {arr[$0]; next} !($0 in arr)' file2 file1
```

# 10. 实战案例

## 10.1 数据统计

```bash
# 求和、平均、最大、最小
awk '{sum+=$1; if(NR==1 || $1>max) max=$1; if(NR==1 || $1<min) min=$1}
     END {print "SUM:", sum, "AVG:", sum/NR, "MAX:", max, "MIN:", min}' data.txt

# 分组统计（类似 SQL GROUP BY）
awk '{sum[$1]+=$2; count[$1]++} END {for(k in sum) print k, sum[k], sum[k]/count[k]}' data.txt

# 计算百分位（简易版）
sort -n data.txt | awk '{all[NR]=$1} END{print all[int(NR*0.95)]}'

# 统计访问日志中每个小时的请求数
awk '{split($4,a,":"); print a[2]}' access.log | sort | uniq -c
```

## 10.2 日志分析

```bash
# 提取状态码大于等于 400 的请求
awk '$9>=400' access.log

# 统计每个 URL 的 P99 延迟
awk '{latency[$7][length(latency[$7])+1]=$10}
     END {for(url in latency) {
         n=length(latency[url]); 
         # 需要排序，简化处理
     }}' access.log

# Nginx 访问日志：统计 UV（独立 IP 数）
awk '{ips[$1]++} END {print length(ips)}' access.log

# PV（总请求数）
awk 'END {print NR}' access.log

# Top 10 IP
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
```

## 10.3 文本格式化

```bash
# 行列转置
awk '
{ for(i=1; i<=NF; i++) { a[NR,i] = $i } }
NF>p { p = NF }
END {
    for(j=1; j<=p; j++) {
        str=a[1,j];
        for(i=2; i<=NR; i++) str=str" "a[i,j];
        print str
    }
}' file.txt

# CSV 转 JSON（简化版）
awk -F, '
BEGIN { print "[" }
NR>1 {
    printf "  {\"name\":\"%s\",\"age\":%d,\"city\":\"%s\"}", $1, $2, $3
    if(NR<total) print ","
}
END { print "\n]" }
' total=$(wc -l < file.csv) data.csv

# 提取指定列（灵活的列选择）
awk '{print $NF, $1}' file.txt             # 最后一列 + 第一列
awk '{for(i=1;i<=NF;i+=2) printf "%s ", $i; print ""}' file.txt  # 奇数列
```

## 10.4 系统管理

```bash
# 解析 ps 输出
ps aux | awk 'NR>1 {sum[$11]+=$6} END {for(p in sum) print sum[p], p}' | sort -rn

# 磁盘使用百分比过滤
df -h | awk 'NR==1 || $5+0>80'

# 解析 /proc/meminfo
awk '/MemTotal|MemFree|Cached/{printf "%-15s %10.2f GB\n", $1, $2/1024/1024}' /proc/meminfo

# 合并两个文件（类似 SQL JOIN）
awk 'NR==FNR {map[$1]=$2; next} {print $0, map[$1]}' file1 file2
```

## 10.5 多分隔符与复杂解析

```bash
# 单条记录跨多行（修改 RS）
awk 'BEGIN{RS="\n\n"; FS="\n"} {print $1}'  # 段落模式

# 空格和逗号同时作为分隔符
awk -F'[ ,]+' '{print $1, $3}'

# 字段中有引号包裹（CSV 引号处理）
awk -F, '{gsub(/^"|"$/,"",$2); print $2}' data.csv
```

# 11. awk 与其他工具配合

```bash
# awk + sort + uniq 组合
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head

# awk 内调用外部命令（不推荐，每行都 fork）
awk '{cmd="date -d @"$1; cmd | getline d; close(cmd); print d,$0}' file.txt

# 更好的做法：只计算一次
awk 'BEGIN{cmd="date"; cmd|getline d; close(cmd); print d}'
```

# 12. awk 性能

- awk 是 C 语言实现的，处理速度远快于纯 Shell 循环
- 大文件的字段处理和模式匹配交给 awk，不要用 Shell while read
- 避免在 awk 内频繁调用外部命令（`system()` / `getline`）
- 复杂数据处理优先考虑 awk 而非 sed

# 13. GAWK 扩展特性（GNU Awk）

多数 Linux 发行版默认使用 GNU Awk (gawk)，它有额外的扩展：

```bash
# 时间函数
gawk 'BEGIN {print strftime("%Y-%m-%d %H:%M:%S")}'

# 位运算
gawk 'BEGIN {print and(5, 3), or(5, 3), xor(5, 3)}'

# 排序（PROCINFO["sorted_in"]）
gawk 'BEGIN {
    PROCINFO["sorted_in"]="@val_num_desc"
    a["x"]=5; a["y"]=2; a["z"]=9
    for(k in a) print k, a[k]
}'

# 网络编程（双向管道）
gawk 'BEGIN {
    service = "/inet/tcp/0/localhost/80"
    print "GET / HTTP/1.1\r\nHost: localhost\r\n\r\n" |& service
    while ((service |& getline) > 0) print $0
    close(service)
}'
```

# 14. 对比：何时用 awk，何时用其他工具

| 场景 | 推荐工具 |
|------|----------|
| 简单字符串替换 | sed |
| 按列提取/计算 | awk |
| 模式搜索 | grep |
| 复杂文本处理逻辑 | awk |
| 结构化数据（JSON/XML） | jq / xmllint |
| 大规模数据分析 | Python / SQL |
