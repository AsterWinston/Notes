`native` 是 Java 里唯一一个"承认自己不够用"的关键字——被它修饰的方法没有方法体，实现落在 C/C++ 里，靠 **JNI（Java Native Interface，Java 本地接口）** 这座桥与 JVM 通信。本篇讲清 **`native` 到底意味着什么、什么场景才值得用它、一套完整的 JNI 开发流程怎么走**，然后把 `JNIEnv`、类型映射、`System.load` 这些必须记住的细节一次讲透。理解它的价值不只在于"会写 JNI"，更在于知道**它有多危险、什么时候应该绕开它**。

# 1. native 关键字意味着什么

## 1.1 从语法看：没有方法体

一个 `native` 方法的声明**以分号结束**，后面没有 `{}`：

```java
public class NativeDemo {
    // 有修饰符、有返回值、有参数列表，唯独没有方法体
    public native int add(int a, int b);

    // native 方法同样可以是 static 的
    public static native long currentPid();

    // 编译错误：native 方法不能同时是 abstract
    // public abstract native void bad();
}
```

编译器对 `native` 方法的处理方式与普通方法有本质区别：

| 维度 | 普通方法 | `native` 方法 |
|------|----------|--------------|
| 方法体 | 必须有（或用 `abstract` 声明为抽象） | **必须没有**，以 `;` 结束 |
| 编译产物 | 生成字节码指令 | 只生成方法签名，`Code` 属性为空 |
| 方法标志位 | 无 | 常量池中带 `ACC_NATIVE` 标志 |
| 调用时的行为 | 解释执行 / JIT 编译成机器码 | JVM 查表跳转到**本地函数入口** |
| 能否 `abstract` | 能 | **不能**，二者语义冲突 |

> **注意**：`native` 与 `abstract` 不能共存。`abstract` 的意思是"我不管，交给子类实现"；`native` 的意思是"我不管，交给本地代码实现"。两者都在说"实现不在 Java 里"，但交付对象不同，编译器直接禁止这种组合。

## 1.2 从运行看：JVM 如何进入本地代码

调用一个 `native` 方法时，JVM 的执行流程如下：

```text
  Java 调用方
      │  demo.add(1, 2)
      ▼
┌─────────────────────────────┐
│ JVM 栈：新建一个 JNI 栈帧    │   ← 与 Java 栈帧不同，它记录的是"本地方法入口"
└──────────────┬──────────────┘
               │ 查找该 native 方法的注册表项
               ▼
┌─────────────────────────────┐
│ 从已加载的动态库中取函数地址 │   ← 地址来自 System.loadLibrary 时的符号解析
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│ 进入本地栈：执行 C/C++ 函数  │   ← 从此脱离 JVM 的监控，崩溃不再被 Java 捕获
└──────────────┬──────────────┘
               │ 函数返回
               ▼
┌─────────────────────────────┐
│ 回到 JVM：弹出 JNI 栈帧      │
└──────────────┬──────────────┘
               │
               ▼
          Java 继续执行
```

三个关键点：

1. **本地方法的入口地址是运行期绑定的**，来源是动态库导出符号表，而不是类加载时的静态链接；
2. **本地代码运行在独立于 Java 栈的本地栈上**，一旦越界访问内存，产生的是操作系统级别的 `SIGSEGV`，JVM 无法像捕获 `NullPointerException` 那样捕获它；
3. **每次调用都要跨越 Java 世界与本地世界的边界**，这个跨越本身有开销，这就是"JNI 不适合高频微操作"的原因（见 9.5）。

## 1.3 与反射调用的区别

初学者容易把 `native` 和"反射"混在一起，两者完全不同：

| 维度 | `native` 方法 | 反射调用 |
|------|--------------|----------|
| 实现位置 | JVM 之外的本地代码 | 仍然是 Java 字节码 |
| 是否脱离 JVM | 是，运行在本地栈 | 否，仍在 Java 栈 |
| 类型检查 | 编译期检查参数类型 | 运行期检查，写错才报错 |
| 是否安全 | 可能直接把 JVM 干崩 | 最坏是抛异常 |

反射的内容见第 19 篇，本篇不展开。

# 2. 为什么需要 JNI

## 2.1 四类典型场景

JNI 的存在，本质上是因为 **Java 不是万能的**。以下四类需求，纯 Java 做不到或者做不好：

| 场景 | 说明 | 举例 |
|------|------|------|
| **调用已有的 C/C++ 库** | 大量成熟的底层库只有 C/C++ 版本，重写成本极高 | OpenCV（图像处理）、FFmpeg（音视频）、OpenSSL（加密）、SQLite（嵌入式数据库） |
| **性能敏感的计算** | 需要 SIMD 指令、手写内存布局、无 GC 干扰的计算 | 数值计算、加解密、图像滤镜、音视频编解码 |
| **访问操作系统底层能力** | Java 标准库没有暴露的系统调用 | 获取进程 PID、修改文件权限、调用 `mmap`、设置 CPU 亲和性 |
| **对接硬件驱动** | 驱动程序只提供 C 接口 | 串口通信、USB 设备、工业采集卡、指纹仪 |

JDK 自己就重度使用 JNI。比如 `java.lang.Thread#start0()`、`java.io.FileInputStream#read0()`、`Object#hashCode()`、`System#currentTimeMillis()`，都是 `native` 方法——因为线程调度、文件读写、时钟读取这些操作最终必须落到操作系统上。

```java
// JDK 源码里的真实例子：Thread 类中的 start() 最终调用 native 的 start0()
public synchronized void start() {
    if (threadStatus != 0) throw new IllegalThreadStateException();
    group.add(this);
    boolean started = false;
    try {
        start0();          // ← native 方法，真正创建操作系统线程的是它
        started = true;
    } finally {
        // ...
    }
}
private native void start0();   // 没有方法体
```

## 2.2 为什么现代开发应尽量避免

既然 JNI 这么强大，为什么工程上通常说"**能不用就不用**"？因为它同时破坏了 Java 的四个核心优势：

| 代价 | 具体后果 |
|------|----------|
| **破坏跨平台性** | 必须为每个目标平台（Windows / Linux / macOS，x86_64 / ARM64）各编译一份动态库，还要塞进 JAR 里做平台判断。JDK 的"一次编写，到处运行"到此为止 |
| **内存安全失控** | 本地代码的数组越界、野指针、重复释放，产生的是 `SIGSEGV`，整个 JVM 直接挂掉并生成 `hs_err_pid<pid>.log`，Java 层的 `try-catch` 完全无效 |
| **调试极其困难** | 需要同时掌握 Java 调试器和 GDB / LLDB，跨语言栈帧断裂，Core Dump 分析门槛高 |
| **有性能损耗** | 每次调用都要跨越 JNI 边界、做参数装箱/转换、可能发生字符串编码转换与数组复制，调用频率一高，开销反而超过把 Java 逻辑用 JIT 优化后的效果 |

> **陷阱**：最常见的误判是"热点计算用 C 重写会更快"。JIT 编译器对纯 Java 循环的优化非常激进（逃逸分析、循环展开、向量化），而 JNI 的**跨界开销是固定的**。只有当单次调用里做的计算量足够大时，本地实现才划算——把 `add(int, int)` 这种一行代码写成 native，是纯粹的负优化。

> **最佳实践**：真要用本地代码，**把粒度做大**。一次 JNI 调用里完成一整批计算，而不是让 Java 循环里每轮都调一次 native 方法。跨界的次数决定开销，而不是本地代码的复杂度。

## 2.3 新的替代方向：java.lang.foreign

JDK 从 21 开始预览、在 **JDK 22 正式定稿**的 **`java.lang.foreign`（Foreign Function & Memory API，外部函数与内存 API，JEP 454）**，目标就是取代 JNI 的大部分使用场景。

它允许 Java 代码**直接调用本地函数并安全地操作堆外内存**，而不需要写一行 C 代码、不需要编译动态库：

| 维度 | 传统 JNI | `java.lang.foreign` |
|------|----------|---------------------|
| 是否需要写 C 胶水代码 | 需要 | **不需要** |
| 是否调用 `javac -h` 生成头文件 | 需要 | 不需要 |
| 跨平台 | 每个平台各编一份库 | 由 JDK 处理平台差异 |
| 内存安全 | 靠程序员自觉 | **`Arena` 管生命周期，越界访问受检** |
| 崩溃风险 | 本地崩溃 = JVM 崩溃 | 大幅降低，非法访问抛 `IndexOutOfBoundsException` 之类 |
| 成熟度 | 二十余年，生态完备 | 较新，生态仍在追赶 |

> **行业实践**：新项目如果只是要调用现成的 C 库、读写堆外内存，**优先评估 `java.lang.foreign`**；只有在需要深度复用已有 JNI 代码、或对接的库本身就是 JNI 封装时，才继续走传统 JNI。存量项目不必急着迁移，JNI 在可预见的将来仍被完整支持。

# 3. JNI 开发流程全景

## 3.1 完整流程图

```text
┌───────────────────────────────────────────────────────┐
│ ① 编写 Java 类，用 native 声明方法，静态块加载库        │
│    public native int add(int a, int b);               │
└──────────────────────────┬────────────────────────────┘
                           │ javac -h .
                           ▼
┌───────────────────────────────────────────────────────┐
│ ② 得到 NativeDemo.class + com_example_NativeDemo.h     │
│    头文件由 javac 自动生成，不要手写、不要手动修改       │
└──────────────────────────┬────────────────────────────┘
                           │ 照着 .h 里的函数声明去实现
                           ▼
┌───────────────────────────────────────────────────────┐
│ ③ 用 C/C++ 实现函数，文件名与函数名必须一字不差          │
│    JNIEXPORT jint JNICALL                             │
│    Java_com_example_NativeDemo_add(JNIEnv*, jobject,  │
│                                    jint, jint);       │
└──────────────────────────┬────────────────────────────┘
                           │ gcc / clang 编译成动态库
                           ▼
┌───────────────────────────────────────────────────────┐
│ ④ 得到动态链接库                                       │
│    Windows: native_lib.dll                            │
│    Linux:   libnative_lib.so                          │
│    macOS:   libnative_lib.dylib                       │
└──────────────────────────┬────────────────────────────┘
                           │ System.loadLibrary("native_lib")
                           ▼
┌───────────────────────────────────────────────────────┐
│ ⑤ 运行，JVM 解析动态库导出符号，把 native 方法绑定到函数 │
│    java -Djava.library.path=. com.example.NativeDemo  │
└───────────────────────────────────────────────────────┘
```

## 3.2 各平台动态库命名约定

`System.loadLibrary("native_lib")` 只给"库名"，**前缀和后缀由 JVM 按平台自动补全**，这是初学者最容易踩的坑之一：

| 平台 | 实际文件名 | `loadLibrary` 参数 | 编译命令关键参数 |
|------|-----------|-------------------|------------------|
| Windows | `native_lib.dll` | `"native_lib"` | `-shared` |
| Linux | `libnative_lib.so` | `"native_lib"` | `-shared -fPIC` |
| macOS | `libnative_lib.dylib` | `"native_lib"` | `-dynamiclib -fPIC` |

> **陷阱**：在 Linux 上把库编译成 `native_lib.so`（忘了 `lib` 前缀），`System.loadLibrary("native_lib")` 会去找 `libnative_lib.so`，找不到就抛 `UnsatisfiedLinkError`。用 `System.load("/绝对路径/native_lib.so")` 则不受命名约定约束。

# 4. 从零实现一个可运行的 JNI 示例

## 4.1 Java 端

```java
package com.example;

/**
 * 一个最小但完整的 JNI 示例：演示无参、带参、字符串、静态方法四种形态。
 * 编译与运行见 4.6 节。
 */
public class NativeDemo {
    // 加载动态库。写在 static 块里，保证类初始化时只执行一次
    static {
        System.loadLibrary("native_lib");   // 实际加载 libnative_lib.so / native_lib.dll
        // 或者用绝对路径：System.load("D:\\libs\\native_lib.dll");
    }

    // 无参数、无返回值
    public native void hello();

    // 基本类型参数与返回值
    public native int add(int a, int b);

    // 引用类型参数与返回值
    public native String concat(String a, String b);

    // native 方法同样是 static 也可以：C 侧第二个参数变成 jclass 而不是 jobject
    public static native long currentPid();

    public static void main(String[] args) {
        NativeDemo demo = new NativeDemo();
        demo.hello();
        System.out.println("1 + 2 = " + demo.add(1, 2));
        System.out.println(demo.concat("Hello ", "JNI!"));
        System.out.println("当前进程 pid = " + NativeDemo.currentPid());
    }
}
```

预期输出（pid 每次不同）：

```text
Hello from C!
1 + 2 = 3
Hello JNI!
当前进程 pid = 12884
```

## 4.2 用 javac -h 生成头文件

`javac -h <目录>` 是 JDK8 之后的标准做法，它会在编译 `.class` 的同时**扫描所有 `native` 方法并生成对应的 `.h` 头文件**。

```bash
# -h . 表示头文件生成到当前目录；点号不能省略
javac -h . src/com/example/NativeDemo.java
```

如果源码和输出目录要分开，可以配合 `-d`：

```bash
# .class 输出到 out/，.h 输出到 include/
javac -h include -d out src/com/example/NativeDemo.java
```

> **注意**：旧教程里常见的 `javah com.example.NativeDemo` 命令在 **JDK10 已被移除**，不要再用。现在必须用 `javac -h`。

## 4.3 生成的头文件长什么样

```c
/* DO NOT EDIT THIS FILE - it is machine generated */
#include <jni.h>
/* Header for class com_example_NativeDemo */

#ifndef _Included_com_example_NativeDemo
#define _Included_com_example_NativeDemo
#ifdef __cplusplus
extern "C" {          /* C++ 编译时必须用 extern "C"，否则符号名会被名称修饰破坏 */
#endif
/*
 * Class:     com_example_NativeDemo
 * Method:    hello
 * Signature: ()V
 */
JNIEXPORT void JNICALL Java_com_example_NativeDemo_hello
  (JNIEnv *, jobject);

JNIEXPORT jint JNICALL Java_com_example_NativeDemo_add
  (JNIEnv *, jobject, jint, jint);

JNIEXPORT jstring JNICALL Java_com_example_NativeDemo_concat
  (JNIEnv *, jobject, jstring, jstring);

/* static 方法：第二个参数是 jclass 而非 jobject */
JNIEXPORT jlong JNICALL Java_com_example_NativeDemo_currentPid
  (JNIEnv *, jclass);

#ifdef __cplusplus
}
#endif
#endif
```

头文件里的信息量很大，值得逐块读：

| 元素 | 含义 |
|------|------|
| 文件名 | `包名_类名.h`，包名的 `.` 换成 `_` |
| `#include <jni.h>` | 必须包含，`jint`、`jstring`、`JNIEnv` 都定义在这里 |
| `extern "C"` | 保证 C++ 下不启用名称修饰（name mangling），否则 JVM 找不到符号 |
| `JNIEXPORT` | 宏，在 Windows 上展开为 `__declspec(dllexport)`，声明该函数要导出 |
| `JNICALL` | 宏，指定调用约定（Windows 下是 `__stdcall`） |
| 第一个参数 `JNIEnv *` | JNI 环境指针，调用所有 JNI 函数都要靠它 |
| 第二个参数 | 实例方法为 `jobject`（调用者对象），静态方法为 `jclass`（类对象） |
| 注释里的 `Signature` | 方法签名，见 7.5 节 |

## 4.4 JNI 函数命名规则

JVM 靠**函数名的字符串匹配**来找本地实现，所以命名规则必须一字不差。

```text
                     Java_<包名>_<类名>_<方法名>
                            │        │        │
                            │        │        └─ 方法名（特殊字符要转义）
                            │        └─ 类名，内部类用 $ ，但在 C 里要写成 _00024
                            └─ 包名的 . 全部换成 _
```

三条转义规则：

| 原字符 | 转义写法 | 说明 |
|--------|----------|------|
| `.` | `_` | 包名、类名分隔符 |
| `_` | `_1` | 方法名里本身带下划线时 |
| `;` `[` `$` 等 | `_2` `_3` `_00024` | 非字母数字的 Unicode 码点，用 `_0` + 四位十六进制 |

```text
包名：com.example
类名：NativeDemo
方法名：hello
→ Java_com_example_NativeDemo_hello

方法名：get_name         （含下划线）
→ Java_com_example_NativeDemo_get_1name

内部类：com.example.Outer$Inner
→ Java_com_example_Outer_00024Inner
```

**重载方法的命名**：Java 允许同名不同参的方法，但 C 函数名不能重复。JNI 的做法是给方法名**追加参数类型缩写后缀**，用两个下划线分隔：

```text
方法名：add
  add(int, int)         →  Java_com_example_NativeDemo_add__II
  add(double, double)   →  Java_com_example_NativeDemo_add__DD
  add(String)           →  Java_com_example_NativeDemo_add__Ljava_lang_String_2
```

类型缩写的取值就是 7.5 节的签名表（`I` = int，`D` = double，`Ljava/lang/String;` = String，其中 `/` 变 `_`、`;` 变 `_2`）。

> **最佳实践**：**别手写这些函数名**。生成 `.h` 之后把声明直接复制到 `.c` 文件里，再把 `JNIEXPORT ... ;` 改成 `{ ... }`。手动拼写函数名是 `UnsatisfiedLinkError` 的头号原因。更稳妥的做法是用注册方式（`RegisterNatives`）动态绑定，可以彻底摆脱命名规则的束缚。

## 4.5 C/C++ 实现

```c
#include "com_example_NativeDemo.h"
#include <stdio.h>
#include <string.h>

/* 判断平台，用于取进程 ID —— 这正是"JNI 破坏跨平台性"的直观体现 */
#ifdef _WIN32
  #include <windows.h>
#else
  #include <unistd.h>
#endif

/* ① 无参无返回值：直接打印 */
JNIEXPORT void JNICALL Java_com_example_NativeDemo_hello(JNIEnv *env, jobject obj) {
    printf("Hello from C!\n");
}

/* ② 基本类型：JNI 的 jint 就是 32 位有符号整数，可以直接参与运算 */
JNIEXPORT jint JNICALL Java_com_example_NativeDemo_add
  (JNIEnv *env, jobject obj, jint a, jint b) {
    return a + b;
}

/* ③ 字符串：jstring 是不透明句柄，必须转成 C 字符串才能用，用完必须释放 */
JNIEXPORT jstring JNICALL Java_com_example_NativeDemo_concat
  (JNIEnv *env, jobject obj, jstring a, jstring b) {
    const char *strA = (*env)->GetStringUTFChars(env, a, NULL);   // 第 3 个参数传 NULL 表示不关心是否复制
    const char *strB = (*env)->GetStringUTFChars(env, b, NULL);

    char result[256];
    snprintf(result, sizeof(result), "%s%s", strA, strB);

    (*env)->ReleaseStringUTFChars(env, a, strA);   // 成对释放，否则内存泄漏
    (*env)->ReleaseStringUTFChars(env, b, strB);

    return (*env)->NewStringUTF(env, result);      // 从 C 字符串造一个 Java String 返回
}

/* ④ 静态方法：第二个参数是 jclass；不同平台取 pid 的 API 完全不同 */
JNIEXPORT jlong JNICALL Java_com_example_NativeDemo_currentPid
  (JNIEnv *env, jclass cls) {
#ifdef _WIN32
    return (jlong) GetCurrentProcessId();
#else
    return (jlong) getpid();
#endif
}
```

用 C++ 实现时，调用 JNI 函数的语法更简洁——**因为 C++ 中 `JNIEnv` 被定义成一个带成员函数的类**：

```cpp
#include "com_example_NativeDemo.h"
#include <iostream>
#include <string>

// C 写法：(*env)->GetStringUTFChars(env, a, NULL);
// C++ 写法：env->GetStringUTFChars(a, NULL);   —— 不用再传 env，也不用解引用
JNIEXPORT jstring JNICALL Java_com_example_NativeDemo_concat
  (JNIEnv *env, jobject obj, jstring a, jstring b) {
    const char *strA = env->GetStringUTFChars(a, nullptr);
    const char *strB = env->GetStringUTFChars(b, nullptr);

    std::string result = std::string(strA) + strB;

    env->ReleaseStringUTFChars(a, strA);
    env->ReleaseStringUTFChars(b, strB);

    return env->NewStringUTF(result.c_str());
}
```

> **注意**：C 和 C++ 的调用写法**不能混用**。C 里 `JNIEnv` 是 `const struct JNINativeInterface*` 的别名，所以得先解引用拿到函数表再传 `env`；C++ 里它是一个封装类，直接 `env->` 即可。看到两套写法不要以为是两种 API，它们是同一个函数表。

## 4.6 编译动态库

```bash
# Windows（MinGW-w64 的 gcc）
gcc -shared -I"%JAVA_HOME%/include" -I"%JAVA_HOME%/include/win32" \
    NativeDemo.c -o native_lib.dll -Wl,--add-stdcall-alias

# Linux
gcc -shared -fPIC -I"$JAVA_HOME/include" -I"$JAVA_HOME/include/linux" \
    NativeDemo.c -o libnative_lib.so

# macOS（Intel 或 Apple Silicon 均可，注意 JDK 架构要与之一致）
clang -dynamiclib -fPIC -I"$JAVA_HOME/include" -I"$JAVA_HOME/include/darwin" \
    NativeDemo.c -o libnative_lib.dylib
```

两个 include 路径缺一不可：`$JAVA_HOME/include` 下有通用的 `jni.h`，`$JAVA_HOME/include/<平台>` 下有平台相关的 `jni_md.h`（定义 `JNIEXPORT`、`JNICALL`、`jlong` 等）。

## 4.7 运行

```bash
# 把 .class 所在目录和库文件放到一起，然后指定 java.library.path
java -Djava.library.path=. com.example.NativeDemo
```

```text
Hello from C!
1 + 2 = 3
Hello JNI!
当前进程 pid = 12884
```

至此，一条完整的 JNI 链路就打通了。骨架记住即可：**Java 声明 → `javac -h` 出头文件 → 实现 C → 编译动态库 → 加载运行**。

# 5. JNI 数据类型映射

## 5.1 基本类型映射

Java 的基本类型在 JNI 里都有对应的 `j` 前缀类型。**它们的大小是跨平台固定的**——这正是 JNI 存在的意义之一，本地代码不用再操心 `int` 是 16 位还是 32 位。

| Java 类型 | JNI 类型 | 实际 C/C++ 类型 | 字节数 | 说明 |
|-----------|----------|-----------------|--------|------|
| `boolean` | `jboolean` | `unsigned char` | 1 | 只取 `JNI_TRUE`(1) / `JNI_FALSE`(0) |
| `byte` | `jbyte` | `signed char` | 1 | 有符号 |
| `char` | `jchar` | `unsigned short` | 2 | **UTF-16 码元**，不是 C 的 1 字节 `char` |
| `short` | `jshort` | `signed short` | 2 | |
| `int` | `jint` | `signed int` | 4 | |
| `long` | `jlong` | `signed long long` | 8 | 注意：C 的 `long` 在 Windows 上是 4 字节，**必须写 `long long`** |
| `float` | `jfloat` | `float` | 4 | |
| `double` | `jdouble` | `double` | 8 | |
| `void` | `void` | —— | —— | |

> **陷阱**：把 `jchar` 当成 C 的 `char` 用，会导致中文字符串完全乱码。Java 的 `char` 是 16 位 UTF-16，一个中文字符占一个 `jchar`；而 C 的 `char` 是 8 位的 UTF-8 字节，一个中文字符占 3 个字节。两者之间必须靠 `GetStringUTFChars` / `NewStringUTF` 转换，不能直接读内存。

## 5.2 引用类型映射

引用类型没有裸指针可用，全部用**不透明句柄**（opaque handle）表示——本地代码只能拿到一个"引用"，不能直接读写它指向的对象内存。

| Java 类型 | JNI 类型 | 说明 |
|-----------|----------|------|
| `java.lang.Object` | `jobject` | 所有引用类型的根 |
| `java.lang.Class` | `jclass` | 类对象 |
| `java.lang.String` | `jstring` | 字符串 |
| `java.lang.Throwable` | `jthrowable` | 异常对象 |
| `Object[]` | `jobjectArray` | 对象数组 |
| `boolean[]` | `jbooleanArray` | |
| `byte[]` | `jbyteArray` | 常用于传递二进制数据 |
| `char[]` | `jcharArray` | |
| `short[]` | `jshortArray` | |
| `int[]` | `jintArray` | |
| `long[]` | `jlongArray` | |
| `float[]` | `jfloatArray` | |
| `double[]` | `jdoubleArray` | |

> **注意**：JNI 的引用分为**局部引用**、**全局引用**、**弱全局引用**三种。局部引用只在一次本地方法调用内有效，方法返回后自动失效（且默认最多只保留 16 个，循环创建大量对象会溢出）。要把对象长期保存在 C 侧的静态变量里，必须用 `NewGlobalRef` 转成全局引用，用完再 `DeleteGlobalRef`。

# 6. JNIEnv 与 JavaVM：线程私有与进程唯一

这两个指针是 JNI 里最容易混淆的一对。

## 6.1 JNIEnv：线程私有

`JNIEnv` 是指向**函数表**的指针，它封装了所有 JNI 操作函数。关键性质是：**每个线程有自己独立的 `JNIEnv`，不能跨线程传递使用**。

原因在于 `JNIEnv` 内部承载了**当前线程的局部引用表**和**异常状态**——如果两个线程共用一个 `JNIEnv`，局部引用会互相覆盖，异常状态会互相干扰。

## 6.2 JavaVM：进程唯一

`JavaVM` 代表整个 JVM 实例，**一个进程里只有一个**。它只有少数几个函数，主要用途是：

```c
/* 在本地方法里取到 JavaVM */
JavaVM *vm;
(*env)->GetJavaVM(env, &vm);   // 把 JVM 指针存到全局变量，供后续线程使用
```

| 函数 | 作用 |
|------|------|
| `GetEnv(JNIEnv **env, jint version)` | 取当前线程的 `JNIEnv`；线程未附着时返回 `JNI_EDETACHED` |
| `AttachCurrentThread(JNIEnv **env, void *args)` | 把当前线程附着到 JVM，之后才能调用 JNI 函数 |
| `DetachCurrentThread()` | 解除附着，**必须在退出线程前调用**，否则 JVM 可能泄漏资源甚至崩溃 |
| `GetJavaVM(JavaVM **vm)` | 取 JVM 实例指针 |

## 6.3 对比表

| 维度 | `JNIEnv` | `JavaVM` |
|------|----------|----------|
| 作用域 | **线程私有**，每个线程一份 | **进程唯一**，全局一个 |
| 如何获取 | 作为本地方法的第 1 个参数直接传入 | `(*env)->GetJavaVM(env, &vm)` 或 `JNI_GetCreatedJavaVMs` |
| 能否跨线程 | **不能** | 能，可以存成全局变量 |
| 承载内容 | 函数表、局部引用表、异常状态 | JVM 实例，用于线程附着/脱离 |
| 典型用途 | 所有 JNI 操作函数的调用入口 | 在 C 创建的原生线程里附着到 JVM |

**原生线程回调 Java 的标准套路**（比如 C 侧起了一个接收数据包的线程，要回调 Java 的监听器）：

```c
static JavaVM *g_vm = NULL;              // 全局保存，供其他线程使用
static jobject g_listener = NULL;        // 全局引用，不能保存局部引用

JNIEXPORT void JNICALL Java_com_example_NativeDemo_register
  (JNIEnv *env, jobject obj, jobject listener) {
    (*env)->GetJavaVM(env, &g_vm);
    g_listener = (*env)->NewGlobalRef(env, listener);   // 必须转成全局引用，否则方法返回后失效
}

/* 这个函数由 C 自己的线程调用，它没有 JNIEnv 参数 */
void on_data_received(const char *data) {
    JNIEnv *env = NULL;
    jint r = (*g_vm)->GetEnv(g_vm, (void **) &env, JNI_VERSION_1_8);
    if (r == JNI_EDETACHED) {
        (*g_vm)->AttachCurrentThread(g_vm, (void **) &env, NULL);   // 先附着，才有 JNIEnv 可用
    }
    jclass cls = (*env)->GetObjectClass(env, g_listener);
    jmethodID mid = (*env)->GetMethodID(env, cls, "onData", "(Ljava/lang/String;)V");
    jstring s = (*env)->NewStringUTF(env, data);
    (*env)->CallVoidMethod(env, g_listener, mid, s);
    // 注意：长期存在的原生线程不需要每次 Detach，线程结束前调用一次即可
}
```

> **陷阱**：在原生线程里忘了 `AttachCurrentThread` 就直接调 JNI 函数，会拿到一个未初始化的 `JNIEnv` 并直接崩溃。这是 JNI 里最常见的段错误来源之一。

# 7. JNIEnv 常用函数

`JNIEnv` 的函数表有 200 多个函数，但日常用到的就下面几组。**记住一条总原则：凡是 Get 出来的资源，一定要有对应的 Release**。

## 7.1 字符串操作

| 函数 | 作用 |
|------|------|
| `const char* GetStringUTFChars(JNIEnv*, jstring, jboolean*)` | 把 `jstring` 转成 **UTF-8** 的 C 字符串；第 3 个参数传出"是否发生了内存复制"，传 `NULL` 表示不关心 |
| `void ReleaseStringUTFChars(JNIEnv*, jstring, const char*)` | **必须成对调用**，释放上面的资源 |
| `jstring NewStringUTF(JNIEnv*, const char*)` | 从 UTF-8 的 C 字符串创建 `jstring` |
| `jsize GetStringUTFLength(JNIEnv*, jstring)` | 返回 UTF-8 编码下的**字节数**（中文一个字算 3 字节） |
| `jsize GetStringLength(JNIEnv*, jstring)` | 返回 UTF-16 下的**字符数**（中文一个字算 1） |
| `const jchar* GetStringChars(JNIEnv*, jstring, jboolean*)` | 拿到 UTF-16 数据，适合需要按 `jchar` 处理的场景 |
| `void ReleaseStringChars(JNIEnv*, jstring, const jchar*)` | 配对释放 |

```c
/* 完整范式：获取 → 使用 → 释放，三步都不能少 */
const char *utf = (*env)->GetStringUTFChars(env, javaStr, NULL);
if (utf == NULL) {
    return;   /* 转换失败（通常是 OOM），此时 JVM 已挂起一个 OutOfMemoryError */
}
/* ... 使用 utf ... */
(*env)->ReleaseStringUTFChars(env, javaStr, utf);
```

> **陷阱**：`GetStringUTFChars` 返回的指针**不是 Java 字符串内部内存的直接映射**，JVM 可能为了转换编码而临时复制一块内存。不释放就会一直累积，在长生命周期的方法（比如每秒调用一次的循环）里立刻表现为内存泄漏。这条规则对 `GetStringChars`、`Get<Type>ArrayElements`、`GetPrimitiveArrayCritical` 全部适用。

## 7.2 数组操作

| 函数 | 作用 |
|------|------|
| `j<Type>* Get<Type>ArrayElements(JNIEnv*, jarray, jboolean*)` | 拿到数组元素指针，**可能返回一份拷贝**，例如 `GetIntArrayElements` |
| `void Release<Type>ArrayElements(JNIEnv*, jarray, j<Type>*, jint mode)` | 释放，`mode` 决定是否把修改写回 Java 数组 |
| `jsize GetArrayLength(JNIEnv*, jarray)` | 数组长度 |
| `j<Type>Array New<Type>Array(JNIEnv*, jsize length)` | 新建 Java 数组 |
| `void Get<Type>ArrayRegion(JNIEnv*, jarray, jsize start, jsize len, j<Type>* buf)` | 把数组的一段**复制**到 C 缓冲区，不需要 Release |
| `void Set<Type>ArrayRegion(JNIEnv*, jarray, jsize start, jsize len, const j<Type>* buf)` | 把 C 缓冲区写回 Java 数组 |

`Release<Type>ArrayElements` 的 `mode` 有三个取值，选错会导致数据丢失：

| mode | 行为 | 使用场景 |
|------|------|----------|
| `0` | **复制回 Java 数组**，并释放本地缓冲区 | 修改了数据、要写回（默认行为） |
| `JNI_COMMIT` | 复制回 Java 数组，**但不释放**本地缓冲区 | 需要多次提交、最后再释放 |
| `JNI_ABORT` | **放弃修改**，直接释放本地缓冲区 | 只读访问，省去一次复制 |

```c
/* 把 Java 传来的 int[] 每个元素加一，再写回 */
JNIEXPORT void JNICALL Java_com_example_NativeDemo_incAll
  (JNIEnv *env, jobject obj, jintArray arr) {
    jsize len = (*env)->GetArrayLength(env, arr);
    jint *body = (*env)->GetIntArrayElements(env, arr, NULL);   // 可能拷贝了一份

    for (jsize i = 0; i < len; i++) {
        body[i] += 1;
    }

    (*env)->ReleaseIntArrayElements(env, arr, body, 0);   // mode=0：写回并释放
}
```

> **最佳实践**：如果只是想**读**数组的一小段，用 `Get<Type>ArrayRegion` 更划算——它是一次明确的复制，不用记着释放，也避免了整块数组被钉住或复制。只有在需要遍历整个数组并原地修改时，才用 `Get<Type>ArrayElements`。

## 7.3 类、对象、方法、字段操作

| 函数 | 作用 |
|------|------|
| `jclass FindClass(JNIEnv*, const char* name)` | 按全限定名查找类，名字用 `/` 分隔，如 `"com/example/NativeDemo"` |
| `jmethodID GetMethodID(JNIEnv*, jclass, const char* name, const char* sig)` | 获取**实例方法** ID |
| `jmethodID GetStaticMethodID(JNIEnv*, jclass, const char* name, const char* sig)` | 获取**静态方法** ID |
| `jobject CallObjectMethod(JNIEnv*, jobject, jmethodID, ...)` | 调用实例方法，返回引用类型；`CallVoidMethod` / `CallIntMethod` / `CallBooleanMethod` 等对应其他返回类型 |
| `jobject CallStaticObjectMethod(JNIEnv*, jclass, jmethodID, ...)` | 调用静态方法 |
| `jfieldID GetFieldID(JNIEnv*, jclass, const char* name, const char* sig)` | 获取实例字段 ID |
| `jfieldID GetStaticFieldID(JNIEnv*, jclass, const char* name, const char* sig)` | 获取静态字段 ID |
| `j<Type> Get<Type>Field(JNIEnv*, jobject, jfieldID)` | 读实例字段值 |
| `void Set<Type>Field(JNIEnv*, jobject, jfieldID, j<Type> value)` | 写实例字段值 |

**一个完整例子：在 C 里构造 Java 对象并调用它的方法**

```c
JNIEXPORT jstring JNICALL Java_com_example_NativeDemo_greetFromC
  (JNIEnv *env, jobject obj, jstring name) {
    /* ① 找到类，注意用 / 而不是 . */
    jclass cls = (*env)->FindClass(env, "com/example/NativeDemo");
    if (cls == NULL) return NULL;   /* 类找不到，JVM 已挂起 NoClassDefFoundError */

    /* ② 取方法 ID。签名错一个字符就会失败并抛 NoSuchMethodError */
    jmethodID mid = (*env)->GetMethodID(env, cls, "formatGreeting", "(Ljava/lang/String;)Ljava/lang/String;");
    if (mid == NULL) return NULL;

    /* ③ 调用 */
    return (*env)->CallObjectMethod(env, obj, mid, name);
}
```

对应的 Java 侧方法：

```java
// 供 C 回调的方法，必须是 public（或至少对 JNI 可见）
public String formatGreeting(String name) {
    return "你好，" + name + "（这句由 Java 拼装）";
}
```

> **陷阱**：`GetMethodID` 和 `GetFieldID` 的签名参数写错时**不报编译错误**，只在运行期抛 `NoSuchMethodError` / `NoSuchFieldError`，排查时往往要盯半天。稳妥做法是先用 `javap -s -p NativeDemo.class` 打印出所有成员的准确签名，再复制到 C 代码里。

```bash
# 打印类里所有方法和字段的签名，这是写 JNI 签名时的"字典"
javap -s -p NativeDemo.class
```

## 7.4 异常处理

本地代码里调用 JNI 函数失败时，JVM **不会**自动中断执行，而是**挂起一个待处理异常**，然后继续往下跑。所以必须主动检查：

| 函数 | 作用 |
|------|------|
| `jthrowable ExceptionOccurred(JNIEnv*)` | 检查是否挂起了异常，有则返回该异常对象，无则返回 `NULL` |
| `jboolean ExceptionCheck(JNIEnv*)` | 同上，但只返回布尔值，更轻量 |
| `void ExceptionDescribe(JNIEnv*)` | 把异常堆栈打印到 `stderr`（调试用，等价于 `printStackTrace()`） |
| `void ExceptionClear(JNIEnv*)` | **清除**挂起的异常 |
| `void Throw(JNIEnv*, jthrowable)` | 抛出一个已有的异常对象 |
| `jint ThrowNew(JNIEnv*, jclass, const char* msg)` | 按类 + 消息**新建并抛出**异常 |

```c
JNIEXPORT jint JNICALL Java_com_example_NativeDemo_divide
  (JNIEnv *env, jobject obj, jint a, jint b) {
    if (b == 0) {
        /* 在 C 里抛一个 Java 异常，Java 调用方可以用 try-catch 接住 */
        jclass cls = (*env)->FindClass(env, "java/lang/ArithmeticException");
        (*env)->ThrowNew(env, cls, "/ by zero (来自 JNI)");
        return 0;   /* 抛完异常后必须立刻返回，不要继续执行 */
    }
    return a / b;
}
```

Java 侧：

```java
try {
    new NativeDemo().divide(1, 0);
} catch (ArithmeticException e) {
    System.out.println("捕获到：" + e.getMessage());   // 捕获到：/ by zero (来自 JNI)
}
```

> **注意**：在异常挂起状态下继续调用其他 JNI 函数，行为是**未定义**的（大多数函数会直接失败或返回垃圾值）。正确姿势是：`ExceptionCheck` 一旦为真，**立刻清理自己申请的资源然后返回**，把异常留给 Java 层处理；只有在确实想吞掉异常时才调 `ExceptionClear`。

## 7.5 方法签名格式

签名是 JNI 用来精确定位"哪个方法、哪个字段"的类型字符串，`GetMethodID` / `GetFieldID` 的最后一个参数就是它。

**基本类型签名**：

| Java 类型 | 签名 |
|-----------|------|
| `boolean` | `Z` |
| `byte` | `B` |
| `char` | `C` |
| `short` | `S` |
| `int` | `I` |
| `long` | `J` |
| `float` | `F` |
| `double` | `D` |
| `void` | `V` |

**引用类型与数组**：

| Java 类型 | 签名 | 规则 |
|-----------|------|------|
| `String` | `Ljava/lang/String;` | `L` + 全限定名（`.` 换成 `/`）+ `;` |
| `Object` | `Ljava/lang/Object;` | 同上 |
| `int[]` | `[I` | `[` + 元素签名 |
| `String[]` | `[Ljava/lang/String;` | 同上 |
| `int[][]` | `[[I` | 每一维加一个 `[` |

**方法签名 = `(参数签名列表)返回类型签名`**：

```text
void m(int, String)          →  (ILjava/lang/String;)V
int  add(int a, int b)       →  (II)I
String concat(String)        →  (Ljava/lang/String;)Ljava/lang/String;
void main(String[])          →  ([Ljava/lang/String;)V
long currentPid()            →  ()J      ← 无参数也要保留空括号
```

> **最佳实践**：手写签名极易出错。**用 `javap -s -p <类名>.class` 一次性打印全部签名**，把输出直接粘贴进 C 代码，比对着规则推导可靠得多。IDE 里也可以用 `javap` 插件。

# 8. System.load 与 System.loadLibrary

两者都能把动态库加载进 JVM 进程空间，区别只在**怎么找文件**：

| 维度 | `System.load(String filename)` | `System.loadLibrary(String libname)` |
|------|-------------------------------|-------------------------------------|
| 参数 | **库文件的完整路径**（绝对路径） | **库名**，不含前缀和后缀 |
| 查找方式 | 直接用给定路径，不做任何拼接 | 在 `java.library.path` 的每个目录里按平台命名规则查找 |
| 平台相关写法 | 需要自己写路径，Windows 用 `\` 或 `/` | 写一次，三个平台通用 |
| 示例 | `System.load("D:\\libs\\native_lib.dll")` | `System.loadLibrary("native_lib")` |
| 适用场景 | 库的位置不固定 / 在 JAR 内需要先解压到临时目录 | 库位置固定在 `java.library.path` 下 |

`java.library.path` 的默认值来自操作系统的库搜索路径：

| 平台 | 默认搜索路径来源 |
|------|------------------|
| Windows | `PATH` 环境变量中的目录 |
| Linux | `LD_LIBRARY_PATH` 环境变量中的目录（另含系统默认目录） |
| macOS | `DYLD_LIBRARY_PATH` 环境变量中的目录 |

```bash
# 通过 JVM 参数追加自己的库目录
java -Djava.library.path=D:\libs com.example.NativeDemo
```

```bash
# 在运行时打印当前生效的库搜索路径，排查 UnsatisfiedLinkError 的第一招
java -XshowSettings:properties -version 2>&1 | grep java.library.path
```

```java
// 也可以从代码里读出来
System.out.println(System.getProperty("java.library.path"));
```

> **注意**：`System.loadLibrary` 加载过一次后会被 JVM 缓存，**同一个库重复加载不会再次执行**（第二次加载同一路径会抛 `UnsatisfiedLinkError: Native Library ... already loaded in another classloader`）。所以库的加载必须放在 `static` 块里、且保证只被一个类加载器加载——在 Web 容器（Tomcat）里把 JNI 库放在应用 `WEB-INF/lib` 下，很容易因为类加载器隔离而踩这个坑，通常的做法是把库放到容器的公共目录并配置 `java.library.path`。

# 9. 常见问题与注意事项

## 9.1 本地代码崩溃 = 整个 JVM 崩溃

这是 JNI 最需要敬畏的一点。Java 层的异常是可恢复的；本地代码产生的 `SIGSEGV`（段错误）、`SIGBUS`、栈溢出则**无法被 Java 捕获**。

```text
# 崩溃时 JVM 会在工作目录下生成 hs_err_pid<pid>.log，里面能看到本地栈帧
# A fatal error has been detected by the Java Runtime Environment:
#
#  SIGSEGV (0xb) at pc=0x00007f8a4c1b2a33, pid=12884, tid=12885
#
# JRE version: OpenJDK Runtime Environment (17.0.9+9)
# Problematic frame:
# C  [native_lib.so+0x1233]  Java_com_example_NativeDemo_hello+0x13
#
# Java frames: (J=compiled Java code, j=interpreted, Vv=VM code)
# j  com.example.NativeDemo.hello()V+0
```

排查时看三处：`Problematic frame` 指出是哪个动态库的哪个函数崩的；`Java frames` 说明是从哪个 Java 调用进来的；`siginfo` 段落能看出是访问了非法地址还是栈溢出。

> **最佳实践**：给 JVM 加上 `-XX:+CreateCoredumpOnCrash` 并保留 `hs_err_pid*.log`，再把符号表（编译时加 `-g`）一起归档，崩溃才能定位到 C 代码的具体行号。

## 9.2 UnsatisfiedLinkError 的常见原因

`java.lang.UnsatisfiedLinkError` 是 JNI 报错频率最高的异常，按下面的顺序排查：

| 原因 | 表现 | 排查方式 |
|------|------|----------|
| **库文件找不到** | `no native_lib in java.library.path` | 检查 `java.library.path` 与文件名前缀后缀是否符合平台约定 |
| **函数名写错** | `com.example.NativeDemo.hello()V` 类似的符号名找不到 | 用 `nm -D` / `dumpbin /exports` 看动态库**实际导出了哪些符号**，与 JVM 期望的名字对比 |
| **架构不匹配** | `... is not a valid Win32 application` 或 `wrong ELF class` | 32 位的库不能给 64 位 JVM 用，ARM 的库不能给 x86 JVM 用；确认编译器目标架构一致 |
| **签名不匹配** | 报错信息里带着签名字符串 | 用 `javap -s` 核对签名 |
| **缺依赖库** | `libxxx.so: cannot open shared object file` | C 库依赖的其他 `.so` 也要在搜索路径上，用 `ldd` / `Dependencies.exe` 查依赖树 |
| **重复加载** | `Native Library ... already loaded` | 检查是不是被多个类加载器各加载了一次 |

```bash
# Linux：查看动态库导出的符号，验证函数名拼接是否正确
nm -D --defined-only libnative_lib.so | grep Java_

# Windows：用 dumpbin 看导出表
dumpbin /exports native_lib.dll

# macOS
nm -gU libnative_lib.dylib
```

## 9.3 内存管理

本地代码的内存**不受 GC 管理**，全部要手动释放：

| 资源 | 获取 | 释放 |
|------|------|------|
| 字符串 | `GetStringUTFChars` / `GetStringChars` | `ReleaseStringUTFChars` / `ReleaseStringChars` |
| 数组元素 | `Get<Type>ArrayElements` | `Release<Type>ArrayElements` |
| 全局引用 | `NewGlobalRef` | `DeleteGlobalRef` |
| C 侧 `malloc` 的内存 | `malloc` | `free` |
| 局部引用（循环内大量创建） | `FindClass` / `NewObject` 等 | `DeleteLocalRef` |

> **最佳实践**：**成对释放**是 JNI 铁律。除此之外还有一条：在长循环里用 `FindClass`、`NewStringUTF` 创建大量局部引用，会撞上局部引用表上限（默认 16 个），这时要在每轮循环末尾显式 `DeleteLocalRef`，或者用 `PushLocalFrame` / `PopLocalFrame` 成批管理。

## 9.4 线程安全

| 规则 | 说明 |
|------|------|
| `JNIEnv` 不能跨线程 | 每个线程的 `JNIEnv` 各管各的局部引用表和异常状态 |
| 全局引用可以跨线程 | `NewGlobalRef` 出来的对象可以在任意线程使用 |
| 原生线程必须先附着 | `AttachCurrentThread` 之后才能调 JNI 函数，退出前 `DetachCurrentThread` |
| 本地方法的并发保护 | JVM 不会为 `native` 方法自动加锁，共享的 C 静态变量要自己用互斥量保护 |

## 9.5 性能考虑

| 开销来源 | 说明 | 缓解手段 |
|----------|------|----------|
| **跨界调用** | 每次调用都要切换栈、做安全检查，是固定开销 | 减小调用次数，把粒度做大 |
| **数据复制** | `GetStringUTFChars` 可能复制字符串、`Get<Type>ArrayElements` 可能复制数组 | 只读时优先 `Get<Type>ArrayRegion` |
| **编码转换** | Java 的 UTF-16 与 C 的 UTF-8 之间要转换 | 避免在热路径里频繁来回转字符串 |
| **局部引用管理** | 循环里创建大量引用会触发 GC 压力 | 及时 `DeleteLocalRef` |
| **JIT 失效** | `native` 方法无法被 JIT 内联优化，也阻断了逃逸分析 | 只有计算量足够大时才值得 |

> **行业实践**：做 JNI 性能优化时，**先测量再优化**。用 `-XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly` 观察 JIT 行为，用 `perf` / `VTune` 看本地热点。很多"JNI 更慢"的案例，真正原因是跨界调用太频繁，而不是本地代码写得差。

## 9.6 打包与发布

| 问题 | 建议 |
|------|------|
| 多平台要打包多份库 | 按平台建目录（`lib/linux-x86_64`、`lib/win32-x86_64`），加载前用 `os.name` / `os.arch` 判断，从 JAR 中解压到临时目录再 `System.load` |
| 库文件必须能被写入 | JAR 内的库不能直接 `System.load`，要先落到磁盘临时文件 |
| 版本管理 | 动态库版本与 Java 侧接口要一起升级，接口变了必须同步重编并更新签名 |

```java
// 从 JAR 内解压动态库到临时文件再加载——打包分发时的标准做法
public static void loadFromJar(String resourcePath) throws IOException {
    try (InputStream in = NativeDemo.class.getResourceAsStream(resourcePath)) {
        if (in == null) throw new UnsatisfiedLinkError("JAR 内找不到：" + resourcePath);
        String suffix = System.getProperty("os.name").toLowerCase().contains("win") ? ".dll" : ".so";
        File tmp = File.createTempFile("native_lib", suffix);   // 必须有后缀，否则加载失败
        tmp.deleteOnExit();
        Files.copy(in, tmp.toPath(), StandardCopyOption.REPLACE_EXISTING);
        System.load(tmp.getAbsolutePath());   // 用绝对路径加载解压出来的文件
    }
}
```

# 10. 小结

| 主题 | 一句话要点 |
|------|-----------|
| `native` 关键字 | 方法**没有方法体**，以 `;` 结束；不能与 `abstract` 共存；调用时 JVM 经 JNI 栈帧跳进本地代码 |
| 为什么用 JNI | 复用 C/C++ 库、性能敏感计算、操作系统底层能力、硬件驱动；JDK 自身也大量使用 |
| 为什么尽量别用 | 破坏跨平台、内存错误直接崩 JVM、调试困难、有稳定跨界开销；新项目优先评估 `java.lang.foreign`（JDK22 正式） |
| 开发流程 | Java 声明 → `javac -h . Xxx.java` 生成 `.h` → 实现 C/C++ → 编译成 `.dll` / `.so` / `.dylib` → `System.loadLibrary` 加载运行 |
| 命名规则 | `Java_包名_类名_方法名`；`.` 换 `_`、`_` 换 `_1`；重载方法用 `__II`、`__DD` 这类签名后缀区分 |
| 类型映射 | 基本类型 `int → jint`、`long → jlong`（是 `long long`）；引用类型 `String → jstring`、`Object → jobject`、数组 `int[] → jintArray` |
| `JNIEnv` vs `JavaVM` | `JNIEnv` **线程私有**，不能跨线程传；`JavaVM` **进程唯一**，用于 `AttachCurrentThread` 附着原生线程 |
| `JNIEnv` 常用函数 | `GetStringUTFChars`/`ReleaseStringUTFChars`、`GetIntArrayElements`/`ReleaseIntArrayElements`、`FindClass`、`GetMethodID`、`CallObjectMethod`；**Get 与 Release 必须成对** |
| 加载库 | `System.load` 要绝对路径；`System.loadLibrary` 靠 `java.library.path` 查找，库名不带 `lib` 前缀与 `.so` / `.dll` 后缀 |
| 排错 | 本地崩溃生成 `hs_err_pid*.log`；`UnsatisfiedLinkError` 多为库找不到、函数名写错、架构不匹配三类 |

JNI 是 Java 通向"机器世界"的那扇门，但推门出去就不再受 JVM 保护。下一篇（第 22 篇）回到纯 Java 的地盘，讲 Java 语言的基础概念与运行机制；如果想看 JVM 如何在加载阶段解析 `ACC_NATIVE` 标志、把本地方法与动态库符号绑定，可以结合第 24 篇 `java.lang` 与第 19 篇反射一起理解。
