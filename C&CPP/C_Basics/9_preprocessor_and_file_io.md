预处理器在真正的编译之前运行，负责展开头文件、替换宏、裁剪条件代码；文件 I/O 则让程序能读写磁盘。这两块看似无关，却共享一个特征：错误往往在运行时才暴露，而且报错信息不直观。本篇把宏的展开规则、头文件保护和文件读写的错误处理讲清楚。

# 1. 预处理器

## 1.1 预处理做了什么

```text
源文件 + 头文件
      │
      ▼
┌─────────────────────────────┐
│ 预处理                       │
│  · 展开 #include             │
│  · 替换 #define              │
│  · 处理 #if / #ifdef         │
│  · 删除注释                   │
└──────────────┬──────────────┘
               ▼
          纯 C 代码（.i）
```

★ **关键认知**：预处理器只做**文本替换**，不理解类型、作用域和语法。宏的所有坑，本质上都来自"它不知道自己在替换什么"。

## 1.2 宏定义

```c
#include <stdio.h>

#define PI 3.14159
#define SQUARE(x) ((x) * (x))
#define MAX(a, b) ((a) > (b) ? (a) : (b))

int main(void) {
    printf("%f\n", PI);           // 3.141590
    printf("%d\n", SQUARE(5));    // 25
    printf("%d\n", MAX(3, 7));    // 7
    return 0;
}
```

对象式宏直接替换标识符，函数式宏先替换参数再展开。

## 1.3 宏的陷阱

**缺少括号**是最经典的问题：

```c
#define SQUARE(x) x * x

SQUARE(1 + 2)   // 展开为 1 + 2 * 1 + 2 = 5，不是 9
```

正确写法是给参数和整体都加括号：

```c
#define SQUARE(x) ((x) * (x))
```

**副作用**是第二个坑，参数被求值多次：

```c
#define MAX(a, b) ((a) > (b) ? (a) : (b))

int i = 1;
int m = MAX(i++, 10);   // i++ 可能执行两次
```

★ **关键认知**：宏参数每在宏体里出现一次，就会被求值一次。带副作用的表达式（`i++`、函数调用）传给宏，行为不可控。能用 `static inline` 函数替代时，优先用函数。

**多语句宏**要用 `do { ... } while (0)` 包裹：

```c
#define LOG(msg) do {          \
    printf("[log] %s\n", msg); \
} while (0)

if (ok)
    LOG("done");   // 不会破坏 if-else 结构
else
    LOG("failed");
```

⚠ **关键认知**：`#define` 没有作用域，一旦定义就影响之后的所有代码，直到 `#undef`。头文件里定义宏会污染所有包含它的文件，应尽量避免。

## 1.4 条件编译

```c
#define DEBUG 1

#if DEBUG
    printf("调试信息\n");
#endif

#ifdef _WIN32
    // Windows 专用代码
#elif defined(__linux__)
    // Linux 专用代码
#else
    // 其他平台
#endif
```

| 指令 | 作用 |
|------|------|
| `#if` / `#elif` / `#else` / `#endif` | 条件包含 |
| `#ifdef` / `#ifndef` | 是否定义 |
| `defined(X)` | 是否定义，可用于 `#if` 表达式 |

★ **关键认知**：条件编译在**预处理阶段**裁剪代码，被裁掉的分支根本不参与编译。这与运行时 `if` 有本质区别：后者两边都会编译，只是不执行。

## 1.5 头文件保护

同一个头文件被重复包含，会导致重复定义。标准做法是**包含守卫**：

```c
#ifndef MY_HEADER_H
#define MY_HEADER_H

// 头文件内容

#endif
```

或者用非标准但广泛支持的 `#pragma once`：

```c
#pragma once

// 头文件内容
```

| 方式 | 可移植性 | 说明 |
|------|----------|------|
| 包含守卫 | 标准，所有编译器支持 | 宏名必须唯一 |
| `#pragma once` | 事实标准 | 写法简单，几乎所有编译器支持 |

⚠ **关键认知**：包含守卫的宏名要全局唯一，通常用"项目名_文件名_H"的形式。两个头文件用了同一个守卫名，会导致其中一个被静默跳过。

## 1.6 include 的两种形式

```c
#include <stdio.h>      // 尖括号：系统头文件，从系统路径查找
#include "my_header.h"  // 双引号：先查当前目录，再查系统路径
```

★ **关键认知**：尖括号与双引号的区别是**搜索顺序**，不是"能不能找到"。用 `-I` 添加的路径对两者都生效，所以真正的纪律是：系统库用 `< >`，自己的头文件用 `" "`。

# 2. 文件 I/O

## 2.1 FILE 与打开模式

C 用 `FILE *` 表示文件流：

```c
#include <stdio.h>

FILE *fp = fopen("data.txt", "r");
if (fp == NULL) {
    perror("fopen");   // 打印失败原因
    return 1;
}
// 使用文件
fclose(fp);
```

| 模式 | 含义 | 文件不存在时 |
|------|------|-------------|
| `"r"` | 只读 | 失败 |
| `"w"` | 只写，清空内容 | 创建 |
| `"a"` | 追加 | 创建 |
| `"r+"` | 读写 | 失败 |
| `"w+"` | 读写，清空 | 创建 |
| `"a+"` | 读与追加 | 创建 |

加 `b` 表示二进制模式，例如 `"rb"`、`"wb"`。在 Windows 上文本模式会做换行转换，二进制数据必须用 `b`。

★ **关键认知**：`fopen` 失败时返回 `NULL`，并在 `errno` 里留下错误码。**每一次 `fopen` 都必须检查**，否则后续 `fread`、`fwrite` 会解引用空指针。

## 2.2 文本读写

```c
#include <stdio.h>

int main(void) {
    FILE *fp = fopen("out.txt", "w");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    fprintf(fp, "name=%s\n", "alice");  // 格式化写入
    fclose(fp);
    char buf[64];
    fp = fopen("out.txt", "r");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    while (fgets(buf, sizeof(buf), fp) != NULL) { // 逐行读取
        printf("%s", buf);
    }
    fclose(fp);
    return 0;
}
```

| 函数 | 用途 |
|------|------|
| `fprintf` / `fscanf` | 格式化读写 |
| `fgets` | 读一行，保留换行，安全 |
| `fputs` | 写字符串 |
| `fgetc` / `fputc` | 读写单个字符 |

⚠ **关键认知**：不要用 `gets`，它无法限制长度，已被 C11 移除。读取一行用 `fgets`，并注意目标缓冲区要足够大，且换行符会被保留。

## 2.3 二进制读写

```c
#include <stdio.h>

struct record { int id; double value; };

int main(void) {
    struct record r = {1, 3.14}, out;
    FILE *fp = fopen("data.bin", "wb");
    if (fp == NULL) { perror("fopen"); return 1; }
    size_t n = fwrite(&r, sizeof(r), 1, fp); // 写一个结构体
    fclose(fp);

    fp = fopen("data.bin", "rb");
    if (fp == NULL) { perror("fopen"); return 1; }
    n = fread(&out, sizeof(out), 1, fp);
    if (n != 1) { /* 读取失败或文件不完整 */ }
    fclose(fp);
    return 0;
}
```

★ **关键认知**：`fread` 和 `fwrite` 返回的是**成功读写的元素个数**，不是字节数。判断是否读满要比较返回值与请求的元素数量。

⚠ **关键认知**：直接 `fwrite` 结构体会把填充字节和本机字节序一并写入，跨平台不可读。需要跨平台交换时，应逐字段按固定宽度和字节序序列化。

# 3. 错误处理

## 3.1 errno 与 perror

```c
#include <errno.h>
#include <stdio.h>
#include <string.h>

FILE *fp = fopen("missing.txt", "r");
if (fp == NULL) {
    printf("errno = %d, reason: %s\n", errno, strerror(errno));
    perror("fopen"); // 直接打印到 stderr
}
```

| 工具 | 作用 |
|------|------|
| `errno` | 全局错误码，失败时由库函数设置 |
| `strerror` | 把错误码转成描述字符串 |
| `perror` | 输出"前缀 + 错误描述" |

★ **关键认知**：`errno` 只有在函数**失败后**才有意义，成功时不会被清零。判断错误要看函数的返回值，不能只看 `errno`。写完文件记得 `fclose`，它会刷新缓冲区并关闭；程序异常退出时，缓冲区里未写入的数据可能丢失。

# 4. 常见错误

| 错误 | 表现 | 修复 |
|------|------|------|
| 宏参数不加括号 | 运算结果错误 | 参数与整体都加括号 |
| 忘记头文件保护 | 重复定义 | 包含守卫或 `#pragma once` |
| `fopen` 不判空 | 空指针崩溃 | 立即检查 `NULL` |
| 用 `gets` | 缓冲区溢出 | 改用 `fgets` |
| 文本模式读写二进制 | 数据被换行转换破坏 | 用 `"rb"` / `"wb"` |
| 只检查 `errno` | 误判 | 以返回值为准 |

⚠ **关键认知**：文件操作的每个函数都可能失败，但只有 `fopen` 失败最常见、后果最直接。把"检查返回值"当成肌肉记忆，是从能跑升级到可靠的必经一步。

# 5. 动手练习

1. 写一个宏 `ARRAY_LEN(a)` 求数组元素个数，并说明为什么要用 `sizeof(a) / sizeof((a)[0])`。
2. 实现一个配置文件读取器，逐行解析 `key=value`，忽略空行和以 `#` 开头的注释。
3. 写一个程序把结构体数组写入二进制文件再读回，验证字段完整，并讨论跨平台时需要注意什么。

# 6. 小结

| 概念 | 要点 |
|------|------|
| 宏 | 纯文本替换，参数与整体都要加括号 |
| 副作用 | 宏参数可能被多次求值，优先用内联函数 |
| 条件编译 | 预处理阶段裁剪代码 |
| 头文件保护 | `#ifndef` 守卫或 `#pragma once` |
| `include` | 尖括号查系统路径，双引号先查当前目录 |
| 文件打开 | `fopen` 必须判空，二进制用 `b` |
| 读写 | `fgets` 安全，`fread` 返回元素个数 |
| 错误处理 | 以返回值为准，`errno` 与 `strerror` 辅助 |

下一篇进入最佳实践与总结，把编码规范、防御式编程、调试工具、常见坑清单串成一张知识地图，并给出继续深入的学习路线。
