
# 1. 索引数组

## 1.1 基本操作

```bash
# 定义数组
arr=(a b c d e)          # 空格分隔
files=(*.txt)            # 通配符展开
empty=()                 # 空数组

# 赋值
arr[0]="first"
arr[1]="second"
arr+=(f g)               # 追加元素

# 读取
echo "${arr[0]}"         # 第一个元素（索引从 0 开始）
echo "${arr[-1]}"        # 最后一个元素（负索引，bash 4.2+）
echo "${arr[@]}"         # 所有元素（独立单词）— 推荐
echo "${arr[*]}"         # 所有元素（合并为一个字符串）

# 长度
echo "${#arr[@]}"        # 元素数量
echo "${#arr[0]}"        # 第 0 个元素的字符串长度

# 切片（bash 4.0+）
echo "${arr[@]:1:3}"     # 从索引 1 开始取 3 个元素
echo "${arr[@]:2}"       # 从索引 2 到末尾

# 遍历
for item in "${arr[@]}"; do
    echo "[$item]"
done

# 按索引遍历
for i in "${!arr[@]}"; do
    echo "索引 $i: ${arr[$i]}"
done
```

## 1.2 数组复制与拼接

```bash
a=(1 2 3)
b=(4 5 6)

# 拼接
c=("${a[@]}" "${b[@]}")     # (1 2 3 4 5 6)

# 复制（必须用 [@] 加引号）
copy=("${arr[@]}")

# 错误复制（去掉了引号）
wrong=(${arr[@]})            # 会按 IFS 重新分割
```

## 1.3 常用操作模式

```bash
# 删除元素（将对应位置设为空，用 unset 移除）
unset arr[2]

# 删除整个数组
unset arr

# 检查是否包含某个值
contains() {
    local item
    for item in "${@:2}"; do
        [[ "$item" == "$1" ]] && return 0
    done
    return 1
}

# 数组去重
arr=(a b a c b d)
unique=()
for item in "${arr[@]}"; do
    [[ " ${unique[*]} " == *" $item "* ]] || unique+=("$item")
done
# unique = (a b c d)
```

## 1.4 命令输出转数组

```bash
# 按行转数组（mapfile / readarray，bash 4.0+）
mapfile -t lines < file.txt
# 或
readarray -t lines < file.txt

# 旧版 bash 兼容写法
lines=()
while IFS= read -r line; do
    lines+=("$line")
done < file.txt

# 按单词转数组（利用 IFS）
IFS=':' read -ra parts <<< "path:to:file"
echo "${parts[0]}"     # path
echo "${parts[1]}"     # to
echo "${parts[2]}"     # file
```

# 2. 关联数组（Bash 4.0+）

```bash
# 声明关联数组（必须先 declare -A）
declare -A map

# 赋值
map["name"]="Alice"
map["age"]=30
map["city"]="Beijing"

# 读取
echo "${map["name"]}"

# 遍历键值对
for key in "${!map[@]}"; do
    echo "$key => ${map[$key]}"
done

# 所有键
echo "${!map[@]}"

# 所有值
echo "${map[@]}"

# 键的数量
echo "${#map[@]}"

# 检查键是否存在
if [[ -v map["name"] ]]; then
    echo "键 name 存在"
fi
```

## 2.1 关联数组实战

```bash
# 配置文件解析为关联数组
declare -A config
while IFS='=' read -r key value; do
    [[ "$key" =~ ^# ]] && continue    # 跳过注释
    [[ -z "$key" ]] && continue        # 跳过空行
    config["$key"]="$value"
done < config.ini

# 统计单词频率
declare -A freq
for word in $(cat text.txt); do
    ((freq["$word"]++))
done
for word in "${!freq[@]}"; do
    printf "%-20s %d\n" "$word" "${freq[$word]}"
done | sort -rn -k2

# 简单的 JSON 提取
declare -A user
user[name]="Bob"
user[age]=25
user[city]="Shanghai"

json="{"
for key in "${!user[@]}"; do
    json+="\"$key\":\"${user[$key]}\","
done
json="${json%,}}"     # 移除最后的逗号
echo "$json"
```

# 3. 参数展开中的字符串操作

Bash 的 `${}` 展开提供了丰富的内置字符串操作，无需调用外部工具。

## 3.1 长度

```bash
str="Hello World"
echo ${#str}           # 11
```

## 3.2 子串提取

```bash
str="abcdefghij"
echo ${str:2}          # cdefghij（从索引 2 到末尾）
echo ${str:2:3}        # cde（从索引 2 取 3 个字符）
echo ${str: -3}        # hij（从末尾取 3 个，注意空格）
echo ${str: -5:2}      # fg（从末尾第 5 个起取 2 个）
```

## 3.3 模式删除

```bash
path="/home/user/data.txt"

# ${var#pattern}  从开头删除最短匹配
echo ${path#*/}        # home/user/data.txt

# ${var##pattern} 从开头删除最长匹配
echo ${path##*/}       # data.txt

# ${var%pattern}  从末尾删除最短匹配
echo ${path%.*}        # /home/user/data

# ${var%%pattern} 从末尾删除最长匹配
echo ${path%%.*}       # /home/user/data（同上，因为是第一个 . 之前）
```

## 3.4 查找替换

```bash
str="hello world, hello universe"

# ${var/pattern/replacement}   替换第一个匹配
echo ${str/hello/hi}           # hi world, hello universe

# ${var//pattern/replacement}  替换所有匹配
echo ${str//hello/hi}          # hi world, hi universe

# ${var/#pattern/replacement}  从开头匹配替换
echo ${str/#hello/HI}          # HI world, hello universe

# ${var/%pattern/replacement}  从结尾匹配替换
echo ${str/%universe/mars}     # hello world, hello mars
```

## 3.5 大小写转换

```bash
str="hello WORLD"

echo ${str^}        # Hello WORLD（首字符大写）
echo ${str^^}       # HELLO WORLD（所有大写）
echo ${str,}        # hello WORLD（首字符小写）
echo ${str,,}       # hello world（所有小写）
echo ${str~~}       # HELLO world（反转大小写）
```

## 3.6 位置查找（间接获取）

```bash
str="hello world"

# 第 0 个字符
echo ${str:0:1}     # h
# 字符串中的位置（通过子串判断）
if [[ "$str" == *"wor"* ]]; then echo "包含 wor"; fi
```

# 4. 字符串拼接与分割

```bash
# 拼接
a="Hello"
b="World"
combined="$a $b"               # Hello World
combined+="!"                  # Hello World!

# 重复字符
separator="="
printf -v line "%*s" 20 ""     # 创建 20 个空格
echo "${line// /=}"            # ====================

# 分割（用 IFS）
str="a:b:c:d"
IFS=':' read -ra parts <<< "$str"
echo "${parts[0]}"             # a
echo "${parts[2]}"             # c

# 逐字符处理（用 while + 子串）
str="abcdef"
for ((i=0; i<${#str}; i++)); do
    echo "${str:$i:1}"
done
```

# 5. 字符串比较

```bash
# 相等性
[[ "$a" == "$b" ]]     # 字符串相等
[[ "$a" != "$b" ]]     # 不等
[[ -z "$str" ]]        # 是否为空
[[ -n "$str" ]]        # 是否非空

# 字典序（只在 [[ ]] 中）
[[ "$a" < "$b" ]]     # a 在 b 之前
[[ "$a" > "$b" ]]     # a 在 b 之后

# 模式匹配
[[ "$filename" == *.txt ]]     # 以 .txt 结尾
[[ "$url" == http* ]]          # 以 http 开头

# 正则匹配
[[ "$str" =~ ^[a-z]+$ ]]
```

# 6. 使用外部工具处理字符串

当 Bash 内置能力不够时，常用以下工具：

| 工具 | 适用场景 |
|------|----------|
| `tr` | 字符级替换、删除、大小写转换 |
| `sed` | 复杂替换、多行处理 |
| `awk` | 字段分割、结构化处理 |
| `cut` | 固定格式切分 |
| `grep` | 查找/过滤包含特定模式的行 |
| `sort` | 排序 |

```bash
# tr
echo "HELLO" | tr 'A-Z' 'a-z'           # hello
echo "a,b,c" | tr ',' ':'               # a:b:c

# cut
echo "user:pass:uid" | cut -d: -f1,3     # user:uid

# awk 取列
echo "a b c" | awk '{print $2}'          # b

# sed 替换
echo "v1.0.0" | sed 's/v//'              # 1.0.0
```

# 7. 多行字符串与缩进处理

```bash
# Here Document 多行字符串赋值
text=$(cat <<'EOF'
line 1
line 2
line 3
EOF
)

# 处理缩进（去除每行的公共前导空白）
# 手动方式
while IFS= read -r line; do
    echo "${line#"${line%%[![:space:]]*}"}"   # 去除前导空白
done <<< "$text"

# 注意：<<- 只能去除 Tab（不能去除空格）
cat <<-EOF
	这一行的前导 Tab 会被去除
	这一行也是
EOF
```

# 8. 编码与特殊字符

```bash
# 移除回车符（\r，处理 Windows 文件）
line=$(echo "$line" | tr -d '\r')
# 或
line="${line%$'\r'}"

# 去除首尾空白
trim() {
    local var="$*"
    var="${var#"${var%%[![:space:]]*}"}"   # 去首部空白
    var="${var%"${var##*[![:space:]]}"}"   # 去尾部空白
    echo -n "$var"
}

# URL 编码（简易版）
urlencode() {
    local string="$1"
    local length="${#string}"
    for ((i=0; i<length; i++)); do
        local c="${string:$i:1}"
        case $c in
            [a-zA-Z0-9.~_-]) printf '%s' "$c" ;;
            *) printf '%%%02X' "'$c" ;;
        esac
    done
}

# Base64 编解码
echo "hello" | base64              # aGVsbG8K
echo "aGVsbG8K" | base64 -d       # hello
```
