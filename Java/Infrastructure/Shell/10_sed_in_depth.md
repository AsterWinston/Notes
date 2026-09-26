
# 1. sed 概述

**sed（Stream Editor）** 是 Linux 下最强大的**流式文本编辑器**，按行读取输入、执行编辑命令、输出结果。

```bash
sed [选项] '命令' 输入文件
sed [选项] -f 脚本文件 输入文件
```

**核心工作模型：** 模式空间（Pattern Space）与保持空间（Hold Space）

```
输入行 → [模式空间] → 执行命令 → 输出
              ↕
         [保持空间]（临时存储区）
```

# 2. 基本语法

## 2.1 命令格式

```bash
sed 's/old/new/' file.txt            # 替换
sed '3d' file.txt                     # 删除第 3 行
sed -n '5,10p' file.txt               # 打印第 5~10 行
sed '/pattern/d' file.txt             # 删除匹配行
sed -i 's/old/new/g' file.txt         # 直接修改文件
```

## 2.2 常用选项

| 选项 | 含义 |
|------|------|
| `-n` | 不自动打印模式空间（需显式用 p 命令打印） |
| `-i` | 直接修改文件（`-i.bak` 备份原文件） |
| `-e` | 指定多个命令 |
| `-f` | 从文件读取 sed 脚本 |
| `-E` / `-r` | 使用扩展正则（ERE） |
| `-z` | 以 NUL 字符分隔行（处理包含换行的数据） |

# 3. 地址寻址

```bash
# 行号寻址
sed -n '1p' file.txt            # 第 1 行
sed -n '1,5p' file.txt          # 第 1~5 行
sed -n '1,+3p' file.txt         # 第 1 行 + 后续 3 行（共 4 行）
sed -n '1~2p' file.txt          # 奇数行（1,3,5...）
sed -n '2~2p' file.txt          # 偶数行（2,4,6...）
sed -n '$p' file.txt            # 最后一行
sed -n '5,$p' file.txt          # 第 5 行到末尾

# 正则寻址
sed -n '/pattern/p' file.txt    # 匹配 pattern 的行
sed -n '/start/,/end/p' file.txt  # 从匹配 start 到匹配 end

# 混合寻址
sed -n '5,/pattern/p' file.txt  # 从第 5 行到匹配 pattern

# 取反（!）
sed -n '/pattern/!p' file.txt   # 不匹配 pattern 的行
```

# 4. 核心命令

## 4.1 s — 替换（最常用）

```bash
# 基本语法：s/正则/替换/标志
sed 's/hello/world/' file.txt          # 每行第一个匹配
sed 's/hello/world/g' file.txt         # 每行所有匹配
sed 's/hello/world/2' file.txt         # 每行第 2 个匹配
sed 's/hello/world/gi' file.txt        # 全局 + 忽略大小写
sed 's|/usr/local|/opt|g' file.txt     # 使用其他分隔符（避免转义）

# 使用 & 引用匹配到的内容
echo "hello world" | sed 's/\w\+/"&"/g'       # "hello" "world"

# 使用 \1 \2 ... 引用捕获组（使用 ERE 或转义括号）
echo "2025-05-17" | sed -E 's/([0-9]+)-([0-9]+)-([0-9]+)/\1年\2月\3日/'
# 2025年05月17日

# 大小写转换（GNU sed）
echo "hello" | sed 's/.*/\U&/'               # HELLO
echo "hello" | sed 's/.*/\u&/'               # Hello（首字母大写）
echo "HELLO" | sed 's/.*/\L&/'               # hello

# 在匹配行前后添加内容
sed 's/^/>> /' file.txt                   # 每行前加 >> 
sed 's/$/ <</' file.txt                   # 每行后加 <<
```

## 4.2 d — 删除

```bash
sed '3d' file.txt                # 删除第 3 行
sed '3,5d' file.txt              # 删除 3~5 行
sed '/^#/d' file.txt             # 删除注释行
sed '/^$/d' file.txt             # 删除空行
sed '/start/,/end/d' file.txt    # 删除 start~end 之间的所有行
sed '/^$/{N;/^\n$/d}' file.txt   # 删除连续空行（保留一个）
```

## 4.3 p — 打印（常配合 -n）

```bash
sed -n '1,5p' file.txt           # 打印 1~5 行
sed -n '/error/p' log.txt        # 打印含 error 的行
sed -n '10,20p' file.txt         # 相当于 head -20 | tail -11
```

## 4.4 a / i / c — 追加 / 插入 / 替换行

```bash
# a：在匹配行之后追加
sed '/chapter 2/a\--- new section ---' file.txt

# i：在匹配行之前插入
sed '1i\File Header\n===========' file.txt

# c：替换整行
sed '/old config/c\new config = 1' file.conf

# 多行内容（使用时换行，行尾加 \ 续行）
sed '3a\
line one\
line two' file.txt
```

## 4.5 y — 字符级转换（类似 tr）

```bash
echo "hello" | sed 'y/abcdefghijklmnopqrstuvwxyz/ABCDEFGHIJKLMNOPQRSTUVWXYZ/'
# HELLO

echo "ABC" | sed 'y/A-Z/a-z/'
# abc
```

## 4.6 w / r — 写文件 / 读文件

```bash
# w：将匹配行写入文件
sed -n '/error/w errors.log' app.log

# r：读取文件内容插入到匹配行之后
sed '/<body>/r header.html' template.html
```

# 5. 模式空间与保持空间

两个空间的交互是 sed 最强大也最难懂的部分。

| 命令 | 功能 |
|------|------|
| `h` | 把模式空间复制到保持空间（覆盖） |
| `H` | 把模式空间**追加**到保持空间 |
| `g` | 把保持空间复制到模式空间（覆盖） |
| `G` | 把保持空间**追加**到模式空间 |
| `x` | 交换模式空间和保持空间 |
| `n` | 读取下一行到模式空间（覆盖） |
| `N` | 读取下一行并**追加**到模式空间 |
| `D` | 删除模式空间中到第一个换行符的内容 |

## 5.1 经典案例

```bash
# 交换相邻两行
sed '1!G;h;$!d' file.txt
# 逻辑：非第一行先把保持空间追加回来 → 保存当前行 → 

# 反转文件（类似 tac）
sed '1!G;h;$!d' file.txt

# 匹配行后插入一空行
sed '/pattern/G' file.txt

# 删除连续空行（留一行）
sed '/^$/N;/\n$/D' file.txt

# 打印匹配行和下一行（类似 grep -A1）
sed -n '/pattern/{N;p}' file.txt

# 多行模式匹配（例如 XML/HTML 跨行标签）
sed -n '/<div>/,/<\/div>/p' page.html
```

# 6. 分支与条件

```bash
# sed 也支持条件跳转（标签 + 分支）

# :label   定义标签
# b label  无条件跳转
# t label  最近一次 s 替换成功时跳转

# 示例：删除 HTML 标签
sed ':start s/<[^>]*>//g; /</b start' file.html

# 示例：用冒号补齐四位数字
echo "1:23:456" | sed ':loop; s/\(^\|:\)\([0-9]\)\b/\10\2/; t loop'
```

# 7. 多命令执行

```bash
# 用 -e 指定多个命令
sed -e 's/foo/bar/' -e 's/hello/world/' file.txt

# 用分号分隔
sed 's/foo/bar/; s/hello/world/' file.txt

# 用 {} 分组（仅在特定行范围执行多命令）
sed '/pattern/{
    s/old/new/g
    s/hello/world/g
}' file.txt
```

# 8. 实战案例集

## 8.1 文本转换

```bash
# CSV 逗号分隔符换为 Tab
sed 's/,/\t/g' data.csv

# Windows 行尾转 Unix （\r\n → \n）
sed -i 's/\r$//' file.txt

# 删除行首行尾空白
sed 's/^[[:space:]]*//; s/[[:space:]]*$//' file.txt

# 每行前添加行号（类似 cat -n）
sed = file.txt | sed 'N; s/\n/\t/'

# 提取 XML/HTML 中标签的内容
grep -oP '(?<=<title>).*?(?=</title>)' page.html
# 纯 sed 版（复杂）
sed -n 's/.*<title>\(.*\)<\/title>.*/\1/ip' page.html
```

## 8.2 配置修改

```bash
# 修改配置文件中的值
sed -i 's/^PORT=.*/PORT=9090/' config.conf

# 取消注释某行
sed -i 's/^# *\(PermitRootLogin\)/\1/' /etc/ssh/sshd_config

# 注释某行
sed -i 's/^\(PermitRootLogin\)/# \1/' /etc/ssh/sshd_config

# 在指定行后插入
sed -i '/^\[mysqld\]/a\sql_mode=STRICT_TRANS_TABLES' /etc/my.cnf

# 替换包含 / 的路径（用 | 或 # 作分隔符）
sed -i 's|/var/www/old|/var/www/new|g' nginx.conf
```

## 8.3 日志处理

```bash
# 查看第 100~200 行
sed -n '100,200p' app.log

# 查看某个时间段的日志
sed -n '/2025-05-17 14:00/,/2025-05-17 15:00/p' app.log

# 提取所有 ERROR 及其后 2 行
sed -n '/ERROR/{N;N;p}' app.log

# 删除 ANSI 颜色代码
sed 's/\x1b\[[0-9;]*m//g' colored.log

# 统计每个 IP 的访问次数
sed -n 's/^\([0-9.]*\).*/\1/p' access.log | sort | uniq -c | sort -rn
```

## 8.4 代码处理

```bash
# 删除单行注释
sed '/^[[:space:]]*\/\//d' file.java

# 删除多行注释（简单版）
sed '/\/\*/,/\*\//d' file.java

# 统一缩进（4 空格替换 Tab）
sed -i 's/\t/    /g' *.py

# 删除行尾空格
sed -i 's/[[:space:]]*$//' *.sh

# 文件末尾添加缺失的换行符
sed -i -e '$a\' file.txt
```

# 9. sed 脚本文件

复杂操作建议写入独立脚本文件：

```bash
# transform.sed
#!/bin/sed -f
# 这个文件是 sed 脚本

# 删除注释和空行
/^#/d
/^$/d

# 替换
s/foo/bar/g
s/hello/world/g

# 执行
sed -f transform.sed input.txt
```

# 10. sed 性能注意事项

- **小文件随意**，大文件（>100MB）优先考虑其他工具
- 避免不必要的命令串联，合并多个 `s` 到一个 `sed` 调用
- `sed -i` 本质是写临时文件 + 替换，没有额外的内存开销
- 对于超大文件，`sed` 是流式的，内存可控
