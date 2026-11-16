
# 1. cut — 按分隔符切取字段

```bash
# 按字符位置切取
cut -c 1-5 file.txt          # 每行第 1~5 个字符
cut -c 1,3,5 file.txt        # 第 1、3、5 个字符
cut -c -10 file.txt           # 前 10 个字符
cut -c 11- file.txt           # 第 11 个字符到末尾

# 按分隔符切取
cut -d: -f1,7 /etc/passwd    # 以冒号为分隔符，取第 1 和第 7 字段
cut -d, -f1-3 data.csv       # 取 1~3 字段
cut -d, -f1 --complement data.csv  # 取除第 1 字段外的所有字段

# 实用场景
echo "192.168.1.100" | cut -d. -f4          # 100
echo "user@example.com" | cut -d@ -f2       # example.com
ps aux | tr -s ' ' | cut -d' ' -f2,11       # PID 和命令
```

**局限性：** cut 无法处理连续分隔符（如多个空格），需要先 `tr -s` 压缩。

# 2. sort — 排序

```bash
# 基本排序
sort file.txt                       # 按首字符排序
sort -n file.txt                    # 按数值排序
sort -r file.txt                    # 逆序
sort -u file.txt                    # 排序 + 去重

# 按指定列排序
sort -t: -k2 -n /etc/passwd         # 以冒号为分隔符，按第 2 列数值排序
sort -t, -k3,3nr -k1,1 data.csv     # 先按第 3 列逆序数值，再按第 1 列

# 关键选项
sort -h file.txt                    # 人类可读格式（1K < 2M < 1G）
sort -V file.txt                    # 版本号排序（v1.9 < v1.10）
sort -M file.txt                    # 月份排序（Jan < Feb < ...）
sort -R file.txt                    # 随机排序
sort -t $'\t' -k2 data.txt          # Tab 分隔，用 bash 的 $'\t'

# 排序稳定性
sort --stable -k1 file.txt          # 稳定排序
sort -s -k1 file.txt                # 关闭"最后的比较"（性能优化）

# 检查文件是否已排序
sort -c file.txt                    # 乱序时报告
sort -C file.txt                    # 静默检查（退出码判断）
```

## 2.1 排序实战

```bash
# 找出 10 个最大的文件
du -h | sort -hr | head -10

# 按 IP 出现频率排序
awk '{print $1}' access.log | sort | uniq -c | sort -rn

# 日志按时间排序
sort -t' ' -k4,4n -k5,5n access.log

# 按列组合排序（第 2 列升序，第 1 列降序）
sort -k2,2 -k1,1r data.txt
```

# 3. uniq — 去重与统计

```bash
# uniq 只能去掉相邻的重复行（所以通常先 sort）

sort file.txt | uniq                   # 去重
sort file.txt | uniq -c                # 统计出现次数
sort file.txt | uniq -d                # 只显示重复行
sort file.txt | uniq -u                # 只显示不重复的行
sort file.txt | uniq -c | sort -rn     # 按频率降序排列

# 跳过前 N 个字段比较
uniq -f 1 file.txt                     # 跳过第 1 个字段
uniq -s 4 file.txt                     # 跳过前 4 个字符

# 按指定字符数前缀去重
uniq -w 5 file.txt                     # 只比较前 5 个字符
```

## 3.1 实战

```bash
# 统计最常出现的 10 个单词
cat file.txt | tr -s ' ' '\n' | sort | uniq -c | sort -rn | head -10

# 找出重复文件（按 MD5）
md5sum *.txt | sort | uniq -d -w 32

# 统计 shell 使用情况
awk -F: '{print $NF}' /etc/passwd | sort | uniq -c | sort -rn
```

# 4. tr — 字符替换与删除

```bash
# 字符级转换
echo "hello" | tr 'a-z' 'A-Z'             # HELLO
echo "hello" | tr '[:lower:]' '[:upper:]'  # HELLO（字符类）

# 删除字符
echo "a,b,c" | tr -d ','                   # abc
echo "hello   world" | tr -s ' '           # hello world（压缩重复）

# 字符补集
echo "abc123" | tr -cd '0-9'               # 123（只保留数字）
echo "abc123" | tr -d -c '0-9\n'           # 同上（注意 \n）

# 转义/转换特定字符
echo "line1\nline2" | tr '\\' '/'          # 反斜杠转斜杠

# 实用模式
tr '\r' '\n' < dosfile.txt                 # CRLF → LF
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 16  # 随机密码
```

**tr vs sed：**
| | tr | sed |
|--|----|-----|
| 粒度 | 字符级 | 行级 |
| 性能 | 极快 | 快 |
| 正则 | 不支持 | 支持 |
| 多字符替换 | 仅单字符映射 | 任意替换 |

# 5. wc — 计数

```bash
wc -l file.txt          # 行数
wc -w file.txt          # 单词数
wc -c file.txt          # 字节数（注意不是字符数）
wc -m file.txt          # 字符数
wc -L file.txt          # 最长行的长度

# 多个文件
wc -l *.txt             # 每个文件行数 + 总计

# 管道统计
find . -name "*.java" | wc -l      # 文件数量
grep ERROR app.log | wc -l         # 错误行数
ps aux | wc -l                      # 进程数（含头部）

# 空行计数
grep -c '^$' file.txt
grep -cv '^$' file.txt             # 非空行数
```

# 6. paste — 按列合并文件

```bash
# 并排合并（默认 Tab 分隔）
paste file1.txt file2.txt

# 指定分隔符
paste -d: file1.txt file2.txt

# 序列化（将多行合并为一行，Tab 分隔）
paste -s file.txt

# 将多行用逗号连接
paste -s -d, file.txt
```

# 7. join — 关系连接（类似 SQL JOIN）

```bash
# 内连接（两个文件必须按连接字段排序）
join file1.txt file2.txt

# 指定字段分隔符
join -t: file1 file2

# 指定连接字段
join -1 2 -2 1 file1 file2        # file1 的第 2 字段 = file2 的第 1 字段

# 外连接
join -a1 file1 file2             # 左外连接
join -a2 file1 file2             # 右外连接
join -a1 -a2 file1 file2        # 全外连接
```

# 8. diff / patch — 文件比较与补丁

```bash
# 比较两个文件
diff file1.txt file2.txt
diff -u file1.txt file2.txt       # unified diff（最常用格式）
diff -y file1.txt file2.txt       # 并排显示
diff -r dir1/ dir2/               # 递归比较目录

# 忽略空白
diff -w file1 file2
diff -b file1 file2

# 生成补丁
diff -u old version > changes.patch

# 应用补丁
patch < changes.patch
patch -p1 < changes.patch         # 跳过 1 级目录
```

# 9. column — 格式化列输出

```bash
# 将文本格式化为表格
mount | column -t

# 指定分隔符
cat /etc/passwd | column -t -s:

# 多列输出（先填满每列再换列）
ls | column -c 80

# JSON/CSV 对齐（配合其他工具）
paste <(seq 5) <(shuf -i 1-100 -n 5) | column -t -s $'\t'
```

# 10. expand / unexpand — Tab 和空格转换

```bash
expand file.txt                 # Tab → 空格（默认 8）
expand -t 4 file.txt            # Tab → 4 空格
unexpand -t 4 file.txt          # 4 空格 → Tab
unexpand --first-only file.txt  # 只转换行首空格
```

# 11. find + xargs — 文件查找与批量处理

## 11.1 find 核心用法

```bash
# 按名称查找
find . -name "*.log"                        # 精确匹配
find . -iname "*.log"                       # 忽略大小写
find . -name "*.sh" -o -name "*.py"         # 或条件

# 按类型
find . -type f                              # 普通文件
find . -type d                              # 目录
find . -type l                              # 符号链接
find . -type f -empty                       # 空文件

# 按时间
find . -mtime -7                            # 7 天内修改
find . -mtime +30                           # 30 天前修改
find . -mmin -60                            # 60 分钟内修改
find . -newer reference.txt                 # 比 reference.txt 新

# 按大小
find . -size +100M                          # > 100MB
find . -size -1k                            # < 1KB
find . -size +10M -size -50M               # 10~50MB

# 按权限
find . -perm 644
find . -perm -u+x                           # 用户可执行

# 按用户/组
find . -user alice
find . -group dev

# 按深度
find . -maxdepth 2 -name "*.conf"           # 最多 2 层
find . -mindepth 2 -name "*.conf"           # 至少 2 层

# 修剪目录（排除）
find . -path ./node_modules -prune -o -name "*.js" -print
```

## 11.2 find -exec 与 xargs

```bash
# -exec（每个文件调用一次命令）
find . -name "*.bak" -exec rm {} \;
find . -name "*.txt" -exec grep "pattern" {} +

# -exec + 模式（批量传参，比 \; 快）
find . -name "*.java" -exec grep -l "TODO" {} +

# xargs（外部命令，更灵活）
find . -name "*.log" | xargs rm
find . -name "*.txt" -print0 | xargs -0 rm         # 安全：处理含空格/换行的文件名
find . -name "*.txt" | xargs -I {} cp {} /backup/ # 占位符

# xargs 并行处理
find . -name "*.log" -print0 | xargs -0 -P 4 gzip  # 4 个并行任务

# find + xargs 性能对比
# -exec \;   最慢（每个文件一次 fork）
# -exec +    较快（批量传参）
# xargs      快（批量传参 + 并行）
```

# 12. tee — 分流输出

```bash
# 输出到文件和终端
command | tee output.log

# 追加模式
command | tee -a output.log

# 同时写入多个文件
command | tee file1.log file2.log

# 与 sudo 配合（> 不受 sudo 控制）
echo "config" | sudo tee /etc/app/conf > /dev/null

# 分流到多个命令
echo "test data" | tee >(grep "test") >(wc -c) > /dev/null
```

# 13. comm — 比较两个已排序的文件

```bash
# 三列输出：仅在 file1 / 仅在 file2 / 共有的行
comm file1 file2

# 只显示特定列
comm -12 file1 file2      # 只输出共有的（第 3 列）
comm -23 file1 file2      # 只输出仅在 file1 的（第 2 列）

# 比较目录中的文件列表
comm -12 <(ls dir1 | sort) <(ls dir2 | sort)
```

# 14. xxd / hexdump — 十六进制查看

```bash
# 查看文件的十六进制内容
xxd file.bin
xxd -l 100 file.bin              # 只看前 100 字节
xxd -s 0x100 file.bin            # 从偏移 0x100 开始

# 十六进制编辑
xxd file.bin > file.hex          # 导出
# 编辑 file.hex
xxd -r file.hex > file_new.bin   # 还原

# 查看字符串的十六进制
echo -n "hello" | xxd -p         # 68656c6c6f
```

# 15. 组合实战

```bash
# 统计所有 Java 文件的总行数
find . -name "*.java" -exec cat {} + | wc -l

# 找出并删除 30 天前的日志
find /var/log -name "*.log" -mtime +30 -delete

# 查找所有符号链接
find . -type l -ls

# 统计项目中各文件类型的数量
find . -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn

# 压缩超过 100MB 的日志文件
find /var/log -type f -name "*.log" -size +100M -exec gzip {} \;

# 批量重命名（将所有 .jpeg 改为 .jpg）
find . -name "*.jpeg" -exec bash -c 'mv "$0" "${0%.jpeg}.jpg"' {} \;

# 统计所有代码文件中 TODO 的数量
grep -r "TODO" --include="*.{java,py,js,ts}" . | wc -l
```
