
# 1. GCC（GNU Compiler Collection）

## 1.1 基本用法

```bash
# 安装
sudo apt install build-essential    # 含 gcc、g++、make 等
sudo apt install gcc
sudo apt install g++

# 编译 C 程序
gcc -o hello hello.c                # 编译并链接
gcc -c hello.c                      # 仅编译为 .o 文件
gcc -E hello.c                      # 仅预处理
gcc -S hello.c                      # 生成汇编文件 .s

# 编译 C++ 程序
g++ -o hello hello.cpp

# 多源文件
gcc -o program file1.c file2.c file3.c
```

## 1.2 常用编译选项

| 选项 | 作用 |
|------|------|
| `-o <file>` | 指定输出文件名 |
| `-c` | 仅编译（不链接）|
| `-g` | 生成调试符号（供 GDB 使用）|
| `-O0 -O1 -O2 -O3 -Os` | 优化级别 |
| `-Wall -Wextra -Werror` | 警告控制 |
| `-I<path>` | 头文件搜索路径 |
| `-L<path>` | 库文件搜索路径 |
| `-l<name>` | 链接库（如 -lm 链接数学库）|
| `-std=c11 -std=c++17` | 指定语言标准 |
| `-D<macro>` | 定义预处理器宏 |
| `-fPIC` | 生成位置无关代码（用于共享库）|
| `-static` | 静态链接 |

## 1.3 编译优化级别

```bash
# 各优化级别的效果
-O0: 不优化（编译最快，适合调试）
-O1: 基本优化
-O2: 较高优化（推荐，平衡编译时间和执行性能）
-O3: 最高优化（可能增加代码体积，极少数情况反而变慢）
-Os: 针对体积优化
-Ofast: 突破标准合规性的激进优化（不推荐生产使用）
```

## 1.4 查看编译过程

```bash
# 查看 gcc 子命令
gcc -v -o hello hello.c

# 查看链接的库
gcc -### hello.c

# 查看默认搜索路径
gcc -print-search-dirs
gcc -E -x c /dev/null -v 2>&1 | grep "^ "
```

# 2. make — 自动化构建

## 2.1 基本原理

make 根据 **Makefile** 文件的规则，判断文件的依赖关系，只重新编译变更的部分。

## 2.2 Makefile 基本语法

```makefile
# Makefile
target: dependencies
	command
```

```makefile
# 最简单的 Makefile
hello: hello.c
	gcc -o hello hello.c

clean:
	rm -f hello
```

## 2.3 进阶 Makefile 示例

```makefile
# 编译器和标志
CC = gcc
CFLAGS = -Wall -Wextra -O2 -g
LDFLAGS = -lm

# 源文件
SRCS = main.c util.c network.c
OBJS = $(SRCS:.c=.o)
TARGET = myapp

# 默认目标
all: $(TARGET)

# 链接
$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)

# 编译规则
%.o: %.c
	$(CC) $(CFLAGS) -c -o $@ $<

# 头文件依赖
main.o: main.h
util.o: util.h util_internal.h
network.o: network.h

# 清理
clean:
	rm -f $(OBJS) $(TARGET)

# 安装
install: $(TARGET)
	cp $(TARGET) /usr/local/bin/

# 伪目标
.PHONY: all clean install
```

## 2.4 自动变量

| 变量 | 含义 |
|------|------|
| `$@` | 目标文件名 |
| `$<` | 第一个依赖文件 |
| `$^` | 所有依赖文件（去重）|
| `$?` | 比目标更新的依赖文件 |
| `$*` | 目标文件的主文件名（不含后缀）|

## 2.5 常用 make 命令

```bash
make                          # 执行第一个目标（通常是 all）
make -j$(nproc)               # 并行编译（使用所有 CPU 核心）
make -j4                      # 4 个并行任务
make clean                    # 清理
make install                  # 安装

make -n                       # 只显示命令，不执行（dry run）
make -B                       # 强制重新编译所有
make -d                       # 调试模式（显示依赖分析）
```

# 3. CMake

## 3.1 基本概念

CMake 是跨平台的构建系统生成器，它不直接编译代码，而是生成 Makefile 或其他构建文件。

```bash
# 典型用法
mkdir build && cd build
cmake ..
make
```

## 3.2 基本 CMakeLists.txt

```cmake
# CMakeLists.txt（最小版本要求 3.10）
cmake_minimum_required(VERSION 3.10)
project(MyApp VERSION 1.0.0 LANGUAGES C CXX)

# 设置 C 标准
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# 添加可执行文件
add_executable(myapp main.c util.c network.c)

# 添加库
add_library(mylib STATIC lib.c)
# STATIC / SHARED / MODULE

# 链接库
target_link_libraries(myapp PRIVATE mylib)

# 头文件路径
target_include_directories(myapp PRIVATE include)

# 编译选项
target_compile_options(myapp PRIVATE -Wall -Wextra)

# 安装
install(TARGETS myapp DESTINATION bin)
install(TARGETS mylib DESTINATION lib)
```

## 3.3 外部依赖

```cmake
# 查找系统包
find_package(OpenSSL REQUIRED)
target_link_libraries(myapp PRIVATE OpenSSL::SSL OpenSSL::Crypto)

# FetchContent（自动下载）
include(FetchContent)
FetchContent_Declare(
    json
    GIT_REPOSITORY https://github.com/nlohmann/json.git
    GIT_TAG v3.11.2
)
FetchContent_MakeAvailable(json)
```

## 3.4 CMake 构建流程

```bash
# 推荐的 out-of-source 构建
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE=Release     # 配置
cmake --build . -j$(nproc)              # 编译
cmake --build . --target install        # 安装（可指定 DESTDIR）

# Debug 构建
cmake .. -DCMAKE_BUILD_TYPE=Debug

# 查看可用的配置选项
cmake -LA ..
```

# 4. 库管理

## 4.1 静态库 vs 动态库

| 特性 | 静态库 (.a) | 动态库 (.so) |
|------|-------------|--------------|
| 链接方式 | 编译时包含进可执行文件 | 运行时加载 |
| 可执行文件大小 | 大 | 小 |
| 内存占用 | 每个进程有独立副本 | 内存共享 |
| 更新 | 需重新编译 | 替换 .so 文件即可 |
| 启动速度 | 快 | 略慢（需加载库）|
| 兼容性管理 | 简单 | 需处理 SONAME |

## 4.2 创建和使用静态库

```bash
# 创建目标文件
gcc -c -o libfoo.o foo.c

# 打包为静态库
ar rcs libfoo.a libfoo.o

# 使用静态库
gcc -o myapp main.c -L. -lfoo
# 或直接指定路径
gcc -o myapp main.c libfoo.a
```

## 4.3 创建和使用动态库

```bash
# 创建位置无关的目标文件
gcc -fPIC -c -o libfoo.o foo.c

# 创建动态库
gcc -shared -o libfoo.so.1.0.0 libfoo.o

# 创建符号链接（SONAME 管理）
ln -s libfoo.so.1.0.0 libfoo.so.1
ln -s libfoo.so.1 libfoo.so

# 链接程序
gcc -o myapp main.c -L. -lfoo

# 运行前设置库路径
LD_LIBRARY_PATH=. ./myapp
```

## 4.4 ldconfig — 动态库缓存

```bash
# 添加库路径
sudo cp libfoo.so.1.0.0 /usr/local/lib/
sudo ldconfig                     # 更新缓存

# 查看已缓存的库
ldconfig -p | grep foo

# 列出未定义的符号
ldd /path/to/myapp
```

## 4.5 pkg-config

管理库的编译和链接标志：

```bash
# 查看库信息
pkg-config --cflags openssl        # 头文件路径
pkg-config --libs openssl          # 链接标志
pkg-config --cflags --libs sdl2    # 同时查看

# 在 Makefile 中使用
CFLAGS += $(shell pkg-config --cflags openssl)
LDLIBS += $(shell pkg-config --libs openssl)

# 列出所有已知的 .pc 文件
pkg-config --list-all
```

# 5. 编译安装的典型模式

```bash
# autoconf/automake 项目
./configure --prefix=/usr/local
make -j$(nproc)
sudo make install

# CMake 项目
mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/usr/local
make -j$(nproc)
sudo make install

# Python 项目（setuptools/poetry）
python3 setup.py build
sudo python3 setup.py install
# 或 pip 安装
pip install .

# 卸载（编译安装没有自动卸载，需要手动处理）
# 使用 checkinstall 替代 make install
./configure && make && sudo checkinstall
```
