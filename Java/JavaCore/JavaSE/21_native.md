# 1. 概述

```
JNI（Java Native Interface，Java 本地接口）是 Java 平台的一个重要特性，
允许 Java 代码调用 C/C++ 等本地语言编写的代码，反之亦然。
native 关键字用于声明一个方法由本地语言实现。

使用场景：
  - 调用操作系统底层 API
  - 复用已有的 C/C++ 库
  - 对性能要求极高的代码（数学计算、图形处理等）
  - Java 不具备的功能（如获取进程 PID）
```

# 2. JNI 开发流程

```
2.1 编写 Java 类，用 native 声明本地方法
2.2 javac 编译 Java 源文件为 .class
2.3 javac -h <dir> <.java> 生成 JNI 头文件（.h）
2.4 使用 C/C++ 实现头文件中声明的函数
2.5 编译 C/C++ 代码为动态链接库
    - Windows：编译为 .dll 文件
    - Linux：编译为 .so 文件
    - macOS：编译为 .dylib 或 .jnilib 文件
2.6 Java 中使用 System.loadLibrary() 或 System.load() 加载库
```

# 3. 自定义 native 方法

## 3.1 Java 端

```java
public class NativeDemo {
    // 声明本地方法
    public native void hello();
    public native int add(int a, int b);
    public native String concat(String a, String b);

    // 加载动态库
    static {
        System.loadLibrary("native_lib");  // 库文件名为 native_lib.dll / libnative_lib.so
        // 或者：System.load("D:\\libs\\native_lib.dll");
    }

    public static void main(String[] args) {
        NativeDemo demo = new NativeDemo();
        demo.hello();
        System.out.println("1 + 2 = " + demo.add(1, 2));
        System.out.println(demo.concat("Hello ", "JNI!"));
    }
}
```

## 3.2 编译命令

```
javac NativeDemo.java  // 编译 Java 源文件
javac -h . NativeDemo.java  // 生成 JNI 头文件（在当前目录）
```

## 3.3 JNI 函数命名规则

```
JNI 函数名的格式：
  Java_<包名>_<类名>_<方法名>

  包名中的 . 替换为 _
  方法名中如果包含 _，则在 _ 后加上 _1（即 __ 替代 _）

示例：
  包名：com.example
  类名：NativeDemo
  方法名：hello
  → JNI 函数名：Java_com_example_NativeDemo_hello

完整函数签名：
  JNIEXPORT void JNICALL Java_com_example_NativeDemo_hello(JNIEnv *env, jobject obj)
  
  参数说明：
    - JNIEnv *env：JNI 环境指针，用于调用 JNI 函数
    - jobject obj：调用该方法的 Java 对象引用（如果是 static 方法，则为 jclass cls）

重载方法命名：
  JNI 会使用参数类型缩写后缀来区分重载方法。
  例如 add(int, int) → add__II，add(double, double) → add__DD
```

## 3.4 C/C++ 实现

```c
#include "com_example_NativeDemo.h"
#include <stdio.h>
#include <string.h>

// 实现 hello 方法
JNIEXPORT void JNICALL Java_com_example_NativeDemo_hello(JNIEnv *env, jobject obj) {
    printf("Hello from C!\n");
}

// 实现 add 方法
JNIEXPORT jint JNICALL Java_com_example_NativeDemo_add(JNIEnv *env, jobject obj, jint a, jint b) {
    return a + b;
}

// 实现 concat 方法
JNIEXPORT jstring JNICALL Java_com_example_NativeDemo_concat(JNIEnv *env, jobject obj, jstring a, jstring b) {
    const char *strA = (*env)->GetStringUTFChars(env, a, NULL);
    const char *strB = (*env)->GetStringUTFChars(env, b, NULL);
    
    char result[256];
    snprintf(result, sizeof(result), "%s%s", strA, strB);
    
    (*env)->ReleaseStringUTFChars(env, a, strA);
    (*env)->ReleaseStringUTFChars(env, b, strB);
    
    return (*env)->NewStringUTF(env, result);
}
```

# 4. JNI 数据类型映射

```
4.1 基本类型映射

  Java 类型     JNI 类型       C/C++ 类型        字节数
  boolean      jboolean       unsigned char     8位（JNI定义为8位，实际0/1判断）
  byte         jbyte          signed char       1
  char         jchar          unsigned short    2（UTF-16）
  short        jshort         signed short      2
  int          jint           signed int        4
  long         jlong          signed long long  8
  float        jfloat         float             4
  double       jdouble        double            8
  void         void           —                 —

4.2 引用类型映射

  Java 类型        JNI 类型
  java.lang.Object   jobject
  java.lang.Class    jclass
  java.lang.String   jstring
  java.lang.Throwable jthrowable
  Object[]           jobjectArray
  boolean[]          jbooleanArray
  byte[]             jbyteArray
  char[]             jcharArray
  short[]            jshortArray
  int[]              jintArray
  long[]             jlongArray
  float[]            jfloatArray
  double[]           jdoubleArray
```

# 5. JNIEnv 常用函数

```
JNIEnv 是 JNI 的核心接口，提供了大量函数用于 Java 与本地代码交互。

5.1 字符串操作
  const char* GetStringUTFChars(JNIEnv*, jstring, jboolean*)
    将 jstring 转换为 C 字符串（UTF-8 编码）。

  void ReleaseStringUTFChars(JNIEnv*, jstring, const char*)
    释放 GetStringUTFChars 获得的字符串资源。

  jstring NewStringUTF(JNIEnv*, const char*)
    从 C 字符串创建 jstring（UTF-8 编码）。

  jsize GetStringUTFLength(JNIEnv*, jstring)
    获取 UTF-8 字符串的字节长度。

5.2 数组操作
  j<class_type> Get<Type>ArrayElements(JNIEnv*, jarray, jboolean*)
    获取数组元素指针（可能复制）。例如 GetIntArrayElements。

  void Release<Type>ArrayElements(JNIEnv*, jarray, j<class_type>*, jint mode)
    释放数组元素指针。mode: 0(复制回), JNI_COMMIT(提交但不释放), JNI_ABORT(放弃修改)

  jsize GetArrayLength(JNIEnv*, jarray)
    获取数组长度。

  j<class_type>Array New<Type>Array(JNIEnv*, jsize)
    创建新的 Java 类型数组。

5.3 类/对象/方法操作
  jclass FindClass(JNIEnv*, const char* name)
    查找 Java 类。

  jmethodID GetMethodID(JNIEnv*, jclass, const char* name, const char* sig)
    获取实例方法 ID。sig 是方法签名。

  jmethodID GetStaticMethodID(JNIEnv*, jclass, const char* name, const char* sig)
    获取静态方法 ID。

  jobject CallObjectMethod(JNIEnv*, jobject, jmethodID, ...)
    Call<TYPE>Method 系列函数：调用 Java 对象的实例方法。

  jobject CallStaticObjectMethod(JNIEnv*, jclass, jmethodID, ...)
    CallStatic<TYPE>Method 系列函数：调用 Java 类的静态方法。

  jfieldID GetFieldID(JNIEnv*, jclass, const char* name, const char* sig)
    获取实例字段 ID。

  jfieldID GetStaticFieldID(JNIEnv*, jclass, const char* name, const char* sig)
    获取静态字段 ID。

  获取/设置字段值的系列函数：
    Get<TYPE>Field(JNIEnv*, jobject, jfieldID)
    Set<TYPE>Field(JNIEnv*, jobject, jfieldID, value)

5.4 异常处理
  jthrowable ExceptionOccurred(JNIEnv*)
    检查是否有异常发生。

  void ExceptionClear(JNIEnv*)
    清除当前异常。

  void ExceptionDescribe(JNIEnv*)
    打印异常堆栈到 stderr。

5.5 方法签名格式
  基本类型签名：
    Z(boolean)  B(byte)  C(char)  S(short)  I(int)
    J(long)     F(float) D(double)  V(void)

  引用类型签名：L<全限定类名>;  例如：Ljava/lang/String;
  数组类型签名：[<元素类型>    例如：[I (int[]), [Ljava/lang/String; (String[])
  方法签名格式：(参数类型)返回类型  例如：(ILjava/lang/String;)V 表示 void m(int, String)
```

# 6. System.load vs System.loadLibrary

```
System.load(String filename)         // 加载指定完整路径的动态库文件
  System.load("D:\\libs\\native_lib.dll");

System.loadLibrary(String libname)   // 从 java.library.path 路径加载指定名称的库
  System.loadLibrary("native_lib");  // 查找 native_lib.dll (Windows) 或 libnative_lib.so (Linux)

java.library.path 默认路径：
  - Windows：PATH 环境变量中的目录
  - Linux：LD_LIBRARY_PATH 环境变量中的目录
  - 可通过 JVM 参数指定：-Djava.library.path=D:\\libs
```

# 7. 注意事项

```
7.1 内存管理
  - 本地代码分配的内存需要手动释放
  - GetStringUTFChars 获取的字符串需要用 ReleaseStringUTFChars 释放
  - Get<Type>ArrayElements 获取的数组需要用 Release<Type>ArrayElements 释放

7.2 线程安全
  - JNIEnv 指针是线程相关的，不能跨线程使用
  - 可以使用 JNI 函数 AttachCurrentThread 和 DetachCurrentThread

7.3 性能考虑
  - JNI 调用本身有开销，不适合高频次的微操作
  - 尽量减少 Java 和 C/C++ 之间的数据复制

7.4 跨平台
  - 动态库的文件名和编译方式在不同平台不同
  - C/C++ 代码可能需要注意平台兼容性
```
