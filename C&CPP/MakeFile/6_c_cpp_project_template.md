前五篇讲的是零件：规则、变量、模式、依赖、伪目标与并行。这一篇把它们组装成一个可以直接用的 C/C++ 项目模板。目标不是"能编译"，而是**换配置不改规则、加文件不改结构、调试和发布互不污染**。

# 1. 目标与目录结构

## 1.1 一个可持续的项目布局

```text
project/
├── Makefile
├── include/
│   └── util.h
├── src/
│   ├── main.cpp
│   └── util.cpp
├── tests/
│   └── test_util.cpp
├── lib/                  # 第三方或自建的 .a/.so
├── build/                # 构建产物，不提交
│   ├── obj/
│   │   ├── debug/
│   │   └── release/
│   └── bin/
└── .gitignore
```

对应的 `.gitignore` 至少应包含 `build/`、`*.o`、`*.d`、`*.a`、`*.so` 和最终可执行文件。

★ **关键认知**：**源码目录与构建目录必须分离**。所有 `.o`、`.d`、可执行文件都进 `build/`，`make clean` 只需要删一个目录，既不会误删源码，也不会把中间产物提交进 Git。

## 1.2 目录职责

| 目录 | 放什么 | 是否提交 |
|------|--------|----------|
| `src/` | `.cpp` 实现 | 是 |
| `include/` | 对外头文件 | 是 |
| `tests/` | 测试代码 | 是 |
| `lib/` | 预编译库或子模块产物 | 视情况 |
| `build/` | 全部中间产物与可执行文件 | 否 |

# 2. 编译选项

## 2.1 四类变量各司其职

GNU Make 的约定俗成是：

| 变量 | 用途 | 例子 |
|------|------|------|
| `CPPFLAGS` | 预处理选项，主要是 `-I`、`-D` | `-Iinclude -DDEBUG` |
| `CXXFLAGS` | C++ 编译选项 | `-std=c++17 -Wall -O2` |
| `LDFLAGS` | 链接器选项 | `-Llib -Wl,-rpath,...` |
| `LDLIBS` | 要链接的库 | `-lutil -lpthread` |

把它们分开，是为了让外部可以只覆盖其中一类，例如 `make LDLIBS=-lfoo`。

## 2.2 警告与标准

```makefile
CXXFLAGS := -std=c++17 -Wall -Wextra -Wpedantic
```

| 选项 | 作用 |
|------|------|
| `-std=c++17` | 指定语言标准 |
| `-Wall -Wextra` | 打开常用警告 |
| `-Wpedantic` | 严格遵循标准，拒绝扩展 |
| `-Werror` | 警告视为错误（CI 常用） |

⚠ **关键认知**：不要一开始就开 `-Werror`。它会让"某个编译器版本新增的警告"直接变成构建失败。建议本地开发不开，CI 里单独开一个严格构建任务。

## 2.3 调试与发布

```makefile
MODE ?= debug

ifeq ($(MODE),debug)
CXXFLAGS += -g -O0 -DDEBUG
else ifeq ($(MODE),release)
CXXFLAGS += -O2 -DNDEBUG
else
$(error MODE 只能是 debug 或 release，当前为 $(MODE))
endif
```

| 配置 | 优化 | 调试信息 | 宏 | 用途 |
|------|------|----------|-----|------|
| debug | `-O0` | `-g` | `DEBUG` | 断点、单步 |
| release | `-O2` | 无 | `NDEBUG` | 发布、性能测试 |

两种配置使用不同的 `OBJDIR`，避免调试和发布的 `.o` 混在一起：

```makefile
OBJDIR := $(BUILD)/obj/$(MODE)
```

★ **关键认知**：调试与发布**必须使用不同的目标文件目录**。Make 不跟踪编译选项变化，如果共用目录，从 debug 切到 release 时不会重编，最终链接出来的可能是半调试半优化的混合体。

# 3. 完整模板

## 3.1 变量与文件发现

```makefile
APP      := app
BUILD    ?= build
MODE     ?= debug

SRCDIR   := src
INCDIR   := include
BINDIR   := $(BUILD)/bin
OBJDIR   := $(BUILD)/obj/$(MODE)

SRCS     := $(wildcard $(SRCDIR)/*.cpp)
OBJS     := $(patsubst $(SRCDIR)/%.cpp,$(OBJDIR)/%.o,$(SRCS))
DEPS     := $(OBJS:.o=.d)
TARGET   := $(BINDIR)/$(APP)
```

`wildcard` 负责发现源文件，`patsubst` 把 `src/foo.cpp` 映射成 `build/obj/debug/foo.o`。新增 `.cpp` 不需要改 Makefile。

## 3.2 规则

```makefile
CXX      ?= g++
CPPFLAGS := -I$(INCDIR)
CXXFLAGS := -std=c++17 -Wall -Wextra -MMD -MP
LDFLAGS  :=
LDLIBS   :=

.PHONY: all clean run

all: $(TARGET)

$(TARGET): $(OBJS) | $(BINDIR)
	$(CXX) $(LDFLAGS) $^ $(LDLIBS) -o $@

$(OBJDIR)/%.o: $(SRCDIR)/%.cpp | $(OBJDIR)
	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -c $< -o $@

$(BINDIR) $(OBJDIR):
	mkdir -p $@

run: $(TARGET)
	./$(TARGET)

clean:
	rm -rf $(BUILD)

-include $(DEPS)
```

几个容易忽略的点：

| 写法 | 原因 |
|------|------|
| `$^ $(LDLIBS)` 顺序 | 库必须放在目标文件之后，否则链接器可能丢弃符号 |
| `\| $(BINDIR)` | 目录只需存在，不应触发重建 |
| `mkdir -p $@` | 目录规则用 `$@`，一次定义服务两个目录 |
| `-include $(DEPS)` | 首次构建 `.d` 不存在也不报错 |

## 3.3 使用方式

```bash
make                  # 默认 debug
make MODE=release     # 发布构建，输出到 build/obj/release 和 build/bin
make -j8 -O           # 并行构建并同步输出
make run              # 构建后运行
make clean            # 删除整个 build/
```

```text
build/
├── obj/
│   ├── debug/
│   │   ├── main.o
│   │   ├── main.d
│   │   ├── util.o
│   │   └── util.d
│   └── release/
└── bin/
    └── app
```

# 4. 静态库与动态库

## 4.1 静态库

```makefile
AR       ?= ar
ARFLAGS  := rcs
LIBDIR   := lib
LIB      := $(LIBDIR)/libutil.a
LIB_SRCS := src/util.cpp
LIB_OBJS := $(patsubst $(SRCDIR)/%.cpp,$(OBJDIR)/%.o,$(LIB_SRCS))

$(LIB): $(LIB_OBJS) | $(LIBDIR)
	$(AR) $(ARFLAGS) $@ $^

$(LIBDIR):
	mkdir -p $@
```

`ar rcs` 的四个字母分别是：替换、插入、创建、生成索引。静态库本质是 `.o` 的归档，链接时被复制进可执行文件。

## 4.2 动态库

```makefile
PIC_OBJDIR := $(BUILD)/obj/$(MODE)/pic
PIC_OBJS   := $(patsubst $(SRCDIR)/%.cpp,$(PIC_OBJDIR)/%.o,$(LIB_SRCS))

$(LIBDIR)/libutil.so: $(PIC_OBJS) | $(LIBDIR)
	$(CXX) -shared $^ -o $@

$(PIC_OBJDIR)/%.o: $(SRCDIR)/%.cpp | $(PIC_OBJDIR)
	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fPIC -c $< -o $@
```

动态库的目标文件必须用 `-fPIC` 编译，否则在某些平台上链接会失败。因此 `.o` 和 `.so` 不能共用同一批目标文件。

## 4.3 链接与 rpath

```makefile
LDFLAGS += -L$(LIBDIR)
LDLIBS  += -lutil
```

运行时会找不到 `.so`，需要告诉加载器去哪里找：

```makefile
# $ORIGIN 表示可执行文件所在目录，$$ 是 Make 对 $ 的转义
LDFLAGS += -Wl,-rpath,'$$ORIGIN/../lib'
```

| 方案 | 写法 | 适用场景 |
|------|------|----------|
| 绝对路径 rpath | `-Wl,-rpath,$(abspath $(LIBDIR))` | 本机调试 |
| 相对路径 rpath | `-Wl,-rpath,'$$ORIGIN/../lib'` | 可搬迁的发布包 |
| 环境变量 | `LD_LIBRARY_PATH=lib ./app` | 临时验证 |
| 系统目录 | 安装到 `/usr/local/lib` 并 `ldconfig` | 系统级安装 |

★ **关键认知**：静态库在链接期解决，动态库在**运行期**解决。链接成功不等于运行成功，`ldd ./app`（Linux）或 `otool -L`（macOS）才能看出运行时依赖是否真的能找到。

# 5. 第三方库与 pkg-config

手写 `-I` 和 `-l` 容易漏掉依赖链。`pkg-config` 把库的头文件路径、库路径和依赖库统一输出：

```bash
pkg-config --cflags libpng
pkg-config --libs libpng
```

接入 Makefile：

```makefile
PKG_CONFIG ?= pkg-config
PKG        := libpng

ifeq ($(shell $(PKG_CONFIG) --exists $(PKG) && echo yes),yes)
CPPFLAGS += $(shell $(PKG_CONFIG) --cflags $(PKG))
LDLIBS   += $(shell $(PKG_CONFIG) --libs $(PKG))
else
$(error 找不到 $(PKG)，请先安装对应的开发包)
endif
```

⚠ **关键认知**：`$(shell pkg-config ...)` 在**读取阶段**执行，所以 `make clean` 时也会调用它。如果库可能不存在，把检测放在生成目标的条件里，或者允许 `make clean` 跳过这段逻辑，否则清理都会被依赖检查卡住。

# 6. 常见错误

| 现象 | 原因 | 修复 |
|------|------|------|
| 链接报 undefined reference | 库写在目标文件前面 | 调整成 `$^ $(LDLIBS)` |
| 运行时找不到 `.so` | 没有 rpath 或 `LD_LIBRARY_PATH` | 加 `-Wl,-rpath` |
| debug/release 产物混用 | 共用 `OBJDIR` | 按 `MODE` 分目录 |
| 新增 `.cpp` 不参与编译 | 源文件列表是手写的 | 用 `wildcard` |
| `mkdir -p` 报错 | Windows `cmd.exe` 不支持 | 用 MSYS2/MinGW 的 sh，或换 `mkdir` |
| `pkg-config` 找不到包 | 只装了运行时库，没装开发包 | 安装 `-dev` / `-devel` 包 |
| 修改 `MODE` 后没重编 | Make 不跟踪变量变化 | 不同模式不同目录，或先 clean |

# 7. 动手练习

1. 按本篇结构搭一个 `src/`、`include/` 的小项目，要求 `make`、`make MODE=release`、`make clean` 三条命令都能正常工作，并对比两种模式的产物路径。
2. 把 `util.cpp` 抽成静态库 `lib/libutil.a`，让 `app` 链接它；再用 `nm` 或 `ar t` 检查库内容，确认链接顺序正确。
3. 用 `pkg-config` 接入一个系统库（例如 `libpng` 或 `zlib`），故意把包名写错，观察 `$(error ...)` 的报错位置和信息。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 目录结构 | 源码与构建分离，`clean` 只删 `build/` |
| 选项变量 | `CPPFLAGS`、`CXXFLAGS`、`LDFLAGS`、`LDLIBS` 分工 |
| 模式切换 | debug/release 用不同 `OBJDIR` |
| 依赖生成 | `-MMD -MP` + `-include $(DEPS)` |
| 库 | 静态库 `ar rcs`，动态库 `-fPIC -shared` |
| 链接顺序 | 目标文件在前，库在后 |
| 运行时 | rpath 或 `LD_LIBRARY_PATH` |
| 第三方 | `pkg-config` 统一提供 flags |

下一篇进入调试与排错，讲清 `make -n`、`make -p`、`--debug` 各自能回答什么问题，如何打印变量，以及 tab 错误、循环依赖、目标不执行等高频故障的定位路径。
