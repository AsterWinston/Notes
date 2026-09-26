
# 1. 三种正则表达式标准

| 标准 | 全称 | Shell 使用场景 |
|------|------|---------------|
| **BRE** | Basic Regular Expression | `grep`、`sed`（默认模式） |
| **ERE** | Extended Regular Expression | `grep -E`、`egrep`、`sed -E`、`awk` |
| **PCRE** | Perl Compatible Regular Expression | `grep -P`、现代编程语言（Python/JS/Java） |

## 1.1 元字符对比

| 功能 | BRE | ERE | PCRE |
|------|-----|-----|------|
| 任意字符 | `.` | `.` | `.` |
| 零或多 | `*` | `*` | `*` |
| 一或多 | `\+` | `+` | `+` |
| 零或一 | `\?` | `?` | `?` |
| 或 | `\|` | `\|` | `\|` |
| 分组 | `\( \)` | `( )` | `( )` |
| 量词 | `\{n,m\}` | `{n,m}` | `{n,m}` |
| 词边界 | `\b` | `\b` | `\b` |
| 非捕获组 | 不支持 | 不支持 | `(?: )` |
| 环视 | 不支持 | 不支持 | `(?= )` `(?<= )` |
| 反向引用 | `\1` | `\1` | `\1` |

**记忆技巧：** 在 BRE 中，`( ) { } + ? |` 这些字符的字面量不需要转义，要赋予特殊含义反而需要转义。ERE 则相反。

# 2. 元字符详解

## 2.1 锚点

```bash
^          # 行首
$          # 行尾
\<         # 词首（BRE）
\>         # 词尾（BRE）
\b         # 词边界（PCRE）
\B         # 非词边界（PCRE）
\A         # 字符串首（PCRE）
\Z         # 字符串尾（PCRE）
```

```bash
# 实战
grep '^From:' emails.txt           # 行首为 From:
grep 'bash$' /etc/passwd           # 行尾为 bash
grep '\<root\>' /etc/passwd       # 精确匹配单词 root
```

## 2.2 字符类

```bash
[abc]            # a、b 或 c
[^abc]           # 非 a、b、c
[a-z]            # 小写字母
[A-Z]            # 大写字母
[0-9]            # 数字
[a-zA-Z0-9_]     # 字母数字下划线
```

**POSIX 字符类：**

| 类 | 等价 | 说明 |
|----|------|------|
| `[:alnum:]` | `[a-zA-Z0-9]` | 字母数字 |
| `[:alpha:]` | `[a-zA-Z]` | 字母 |
| `[:digit:]` | `[0-9]` | 数字 |
| `[:lower:]` | `[a-z]` | 小写字母 |
| `[:upper:]` | `[A-Z]` | 大写字母 |
| `[:space:]` | `[ \t\n\r\f\v]` | 空白字符 |
| `[:blank:]` | `[ \t]` | 空格和 Tab |
| `[:punct:]` | 标点符号 | `!"#$%&'()*+,-./:;<=>?@[\]^_`{|}~` |
| `[:xdigit:]` | `[0-9a-fA-F]` | 十六进制数字 |
| `[:print:]` | 可打印字符 |
| `[:graph:]` | 可见字符（不含空格） |

```bash
grep '[[:space:]]' file.txt        # 包含空白字符的行
grep '[^[:alnum:]]' file.txt       # 包含非字母数字的行
grep -E '[[:digit:]]+' file.txt    # 包含一个或多个数字
```

## 2.3 量词

```bash
*         # 零次或多次
+         # 一次或多次（ERE）
?         # 零次或一次（ERE）
{n}       # 恰好 n 次
{n,}      # 至少 n 次
{,m}      # 最多 m 次
{n,m}     # n~m 次
```

**贪婪 vs 非贪婪：** ERE 总是贪婪的（尽可能多匹配）。BRE/ERE 不支持非贪婪。PCRE 用 `*?` `+?` `??` 启用非贪婪。

```bash
# 贪婪（默认）：匹配尽可能多
echo '<p>first</p><p>second</p>' | grep -oP '<p>.*</p>'
# <p>first</p><p>second</p>

# 非贪婪（PCRE）：匹配尽可能少
echo '<p>first</p><p>second</p>' | grep -oP '<p>.*?</p>'
# <p>first</p>
# <p>second</p>
```

## 2.4 分组与反向引用

```bash
# 分组（ERE）
grep -E '(error|warn|fatal)' log.txt

# 反向引用（匹配与捕获组相同的内容）
grep -E '<([a-z]+)>.*</\1>' html.txt     # 匹配配对的标签

# 非捕获组（PCRE）
grep -P '(?:http|https)://' urls.txt      # 不创建捕获组
```

**`BASH_REMATCH` 与捕获组：**

```bash
if [[ "2025-05-17" =~ ^([0-9]{4})-([0-9]{2})-([0-9]{2})$ ]]; then
    echo "${BASH_REMATCH[0]}"   # 完整匹配: 2025-05-17
    echo "${BASH_REMATCH[1]}"   # 年: 2025
    echo "${BASH_REMATCH[2]}"   # 月: 05
    echo "${BASH_REMATCH[3]}"   # 日: 17
fi
```

# 3. ERE 高级元字符

## 3.1 交替（或）

```bash
grep -E 'error|warning|fatal' log.txt
grep -E '(GET|POST) /api/' access.log
```

## 3.2 边界

```bash
grep -E '\<root\>' /etc/passwd        # 词边界（ERE 也支持 \< \>）
grep -P '\broot\b' /etc/passwd         # PCRE 词边界
```

# 4. PCRE 高级特性

仅在 `grep -P`、`[[ $str =~ $regex ]]`（bash）、以及编程语言中可用。

## 4.1 环视（Lookaround）

```bash
# 正向前瞻（?= 后面跟着...）
grep -oP '\w+(?=@)' emails.txt      # 提取 @ 前的用户名

# 负向前瞻（?! 后面不跟着...）
grep -P '\d+(?!\.\d)' numbers.txt   # 不被小数点跟着的数字

# 正向后顾（?<= 前面是...）
grep -oP '(?<=host:)\S+' log.txt    # host: 后的值

# 负向后顾（?<! 前面不是...）
grep -P '(?<!www\.)example\.com'    # 不以 www. 开头的域名
```

## 4.2 其他 PCRE 特性

```bash
# 命名捕获组
grep -oP '(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})'

# 修饰符开关
(?i)pattern      # 忽略大小写
(?i:pattern)     # 组内忽略大小写

# 非捕获组
(?:pattern)      # 分组但不捕获

# 独立性（原子组）
(?>pattern)      # 不回溯

# Unicode 属性
\p{L}           # 任意字母（包括中文）
\p{N}           # 任意数字
\p{Han}         # 汉字
```

# 5. 常用正则模式库

```bash
# ==== 数字 ====
[0-9]+                          # 正整数
-?[0-9]+                        # 整数（含负数）
[0-9]+\.[0-9]+                  # 正浮点数
-?[0-9]+\.[0-9]+                # 浮点数
0[xX][0-9a-fA-F]+               # 十六进制数

# ==== 字符串 ====
^\s*$                           # 空行（或只有空白）
^\S+$                           # 无空白字符行
^[A-Za-z0-9_-]{3,16}$           # 用户名（3-16 字符）

# ==== 网络 ====
([0-9]{1,3}\.){3}[0-9]{1,3}     # IPv4 地址（宽松）
[0-9a-fA-F:]+                   # IPv6 地址（宽松）
[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}  # 邮箱

# ==== 日期时间 ====
[12]\d{3}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])  # YYYY-MM-DD
([01]\d|2[0-3]):[0-5]\d:[0-5]\d                   # HH:MM:SS

# ==== 路径 ====
^/([^/]+/?)*$                   # Unix 绝对路径
\./[^/]+(/[^/]+)*$              # 相对路径

# ==== 编程 ====
^[a-zA-Z_][a-zA-Z0-9_]*$        # 变量/函数命名
//.*$                           # 单行注释（类 C）
/\*[\s\S]*?\*/                  # 多行注释（类 C）

# ==== 中文 ====
[\x{4e00}-\x{9fff}]+            # 中文字符（PCRE）
[一-龥]                         # 中文字符（部分实现支持）
\p{Han}+                        # 汉字（PCRE Unicode）

# ==== 日志 ====
^\d{4}-\d{2}-\d{2}\s+\d{2}:\d{2}:\d{2}  # 时间戳
\[(ERROR|WARN|INFO|DEBUG)\]              # 日志级别
```

# 6. Shell 中正则的使用场景

## 6.1 `[[ $str =~ regex ]]`（Bash 内置）

```bash
# Bash 3.2+ 支持，使用 ERE 语法
url="https://api.example.com/v1/users"

if [[ "$url" =~ ^https://([^/]+)(/.*)?$ ]]; then
    host="${BASH_REMATCH[1]}"
    path="${BASH_REMATCH[2]}"
    echo "Host: $host"
    echo "Path: $path"
fi
```

## 6.2 `expr` 命令（古老方式）

```bash
expr "abc123" : '\([a-z]*\)[0-9]*'     # abc
expr "abc123" : '.*\([0-9]\{3\}\)'     # 123
```

## 6.3 文件名通配符 vs 正则

| | 通配符 (Glob) | 正则表达式 |
|--|--------------|-----------|
| 用途 | 匹配文件名 | 匹配文本内容 |
| `*` | 任意字符序列 | 前一字符重复 0 次以上 |
| `?` | 单个任意字符 | 前一字符出现 0 或 1 次 |
| `[...]` | 字符集 | 字符集 |
| Shell 上下文 | `ls *.txt`、`case $var in pat)` | `[[ =~ ]]`、`grep`、`sed` |

```bash
# 通配符
ls *.txt               # 匹配所有 .txt 文件
echo {a,b,c}           # 展开为 a b c

# 正则（完全不同）
grep 'a.*c' file.txt   # 匹配以 a 开头、c 结尾的行
```

## 6.4 `case` 中的模式匹配

```bash
# case 使用通配符模式，不是正则
case "$filename" in
    *.txt)  echo "文本文件" ;;
    *.jpg|*.png) echo "图片" ;;
    *log*.*) echo "日志文件" ;;
    [A-Z]*) echo "大写开头" ;;
esac
```

# 7. 正则性能优化

```bash
# 避免灾难性回溯
# 坏：嵌套量词
^(a+)+$             # 匹配 aaaaaaX 时会指数级回溯

# 好：清晰的结构
^a+$

# 坏：无限制的 .* 
.*pattern           # 先匹配整行，再逐步回退

# 好：更精确的模式
^[^#]*pattern       # 或使用原子组（PCRE）
```

**性能建议：**
1. 使用字符类代替 `.`（更精确，减少回溯）
2. 使用锚点（`^` `$`）缩小匹配范围
3. 大文件搜索使用 `LC_ALL=C` 跳过 Unicode
4. 能用固定字符串就不用正则（`grep -F`）
5. 优先使用 `ripgrep` 或 `ag` 替代 `grep`

# 8. 测试与调试正则

```bash
# 在线工具（浏览器）
# regex101.com — 支持 PCRE/Python/JS/Go
# regexr.com

# 命令行快速测试
echo "test string" | grep -oP 'regex'

# 交互测试
while true; do
    read -r -p "正则: " pattern
    read -r -p "文本: " text
    if [[ "$text" =~ $pattern ]]; then
        echo "匹配！"
        for i in "${!BASH_REMATCH[@]}"; do
            echo "  组 $i: ${BASH_REMATCH[$i]}"
        done
    else
        echo "不匹配"
    fi
done
```
