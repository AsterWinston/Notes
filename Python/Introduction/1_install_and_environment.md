搭环境是学 Python 的第一道坎：`python` 命令指向哪个解释器、第三方包装到了哪里、为什么换台机器就跑不起来。这些问题几乎都源于同一个原因——**没有把"解释器、虚拟环境、依赖"三者的关系理清**。本篇从安装讲到项目结构，把这条链路一次打通。

# 1. 安装与解释器

## 1.1 安装要点

Windows 到 [python.org](https://www.python.org/downloads/) 下载 3.11/3.12 安装包，安装时勾选 **Add python.exe to PATH** 和 **Install launcher for all users**。macOS 可用 `brew install python@3.12`，Linux 建议用 `pyenv` 或发行版新仓库。

★ **关键认知**：安装 Python 装的是**解释器**和**标准库**；第三方包不在安装目录，而在 `site-packages` 里，位置由"当前用的是哪个解释器"决定。后面所有"包装哪去了"的困惑都从这里来。

## 1.2 python、python3 与 py

| 平台 | 命令 | 说明 |
|------|------|------|
| Windows | `py -3.12` | 官方启动器，最可靠 |
| Windows | `python` | 取决于 PATH，可能指向任意版本 |
| macOS/Linux | `python3` | 系统默认 Python 3 |
| macOS/Linux | `python` | 可能不存在或指向 Python 2 |

```bash
# 查看命令真实指向哪个解释器
python -c "import sys; print(sys.executable)"

# Windows 精确指定版本
py -3.12 --version
```

★ **关键认知**：`python --version` 不代表你正在用的解释器，**`sys.executable` 才是唯一可信答案**。排查环境问题第一步永远是打印它。

## 1.3 解释器的执行链路

```text
源码 app.py
   │
   ▼
编译为字节码（.pyc）
   │
   ▼
CPython 虚拟机逐条执行
   │
   ▼
调用 C 实现完成系统操作
```

`__pycache__/` 里的 `.pyc` 只是启动加速缓存，**不能脱离解释器运行，也不能当分发包**。

★ **关键认知**：日常说的 "Python" 通常指 **CPython**——用 C 写的官方实现。此外还有 PyPy（JIT）、Jython、MicroPython，语法相近但底层行为不同。初学只认 CPython。

# 2. 虚拟环境

## 2.1 为什么需要

项目 A 要 `requests==2.28`，项目 B 要 `requests==2.31`。全局只装一份必然冲突，虚拟环境为每个项目复制独立的 `site-packages`：

```text
全局解释器 C:\Python312\
├─ python.exe
└─ Lib\site-packages\          ← 全局包，所有项目共享

项目 A\.venv\Lib\site-packages\  ← 只有 A 的依赖
项目 B\.venv\Lib\site-packages\  ← 只有 B 的依赖
```

★ **关键认知**：venv **不复制整个 Python**，只创建一个轻量目录，里面有指向基础解释器的 `python.exe` 和空的 `site-packages`。所以创建很快，占用空间主要是后来装的包。

## 2.2 创建与激活

```bash
# 创建
python -m venv .venv

# Windows PowerShell
.\.venv\Scripts\Activate.ps1

# Windows CMD
.venv\Scripts\activate.bat

# macOS / Linux
source .venv/bin/activate

# 退出
deactivate
```

激活后提示符前出现 `(.venv)`。⚠ PowerShell 报"禁止运行脚本"是执行策略问题，用 `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` 放开当前用户即可。

## 2.3 本质是 PATH 优先级

```text
激活前：PATH = C:\Python312\; C:\Windows\system32\; ...
激活后：PATH = D:\demo\.venv\Scripts\; C:\Python312\; C:\Windows\system32\; ...
```

★ **关键认知**：虚拟环境不是魔法开关，本质就是 **PATH 前置 + 独立 site-packages**。关掉终端即失效，所以每开新窗口都要重新激活。理解这点就能解释：为什么 `deactivate` 只是恢复 PATH、为什么 IDE 要单独配解释器、为什么未激活时 `pip install` 会装到全局。

# 3. pip 与依赖

## 3.1 基本命令

```bash
pip install requests                 # 安装
pip install "requests==2.31.0"       # 指定版本
pip install --upgrade requests       # 升级
pip uninstall requests               # 卸载
pip list                             # 已安装
pip show requests                    # 详情与依赖
```

★ **关键认知**：**pip 属于某一个解释器**。用哪个 `pip` 取决于 PATH，而 PATH 取决于是否激活环境。最稳妥的写法是 `python -m pip install ...`，强制"用当前 python 对应的 pip"。

## 3.2 镜像与 requirements

```bash
# 临时换清华源
pip install requests -i https://pypi.tuna.tsinghua.edu.cn/simple

# 永久配置（写入用户级配置）
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

```text
# requirements.txt
requests==2.31.0
flask>=3.0,<4.0
```

```bash
pip install -r requirements.txt
pip freeze > requirements.txt      # 导出全部依赖（含间接依赖）
```

⚠ **镜像地址不要写进项目仓库**，它属于个人网络环境。永久配置放用户级 `pip.ini` / `~/.pip/pip.conf`。

# 4. PATH 与常见错误

## 4.1 PATH 原理

系统执行命令时按 PATH 顺序逐个目录查找同名可执行文件：

```text
输入 python：
PATH = D:\demo\.venv\Scripts; C:\Python312; C:\Windows
                    │
                    ▼
   命中第一个：D:\demo\.venv\Scripts\python.exe
```

★ **关键认知**：PATH 是"**先命中先用**"，后面的同名命令被遮蔽。这既是虚拟环境生效的机制，也是"装了新版却还是旧版"的根因——旧路径排在前面。

## 4.2 排查表

| 现象 | 原因 | 解决 |
|------|------|------|
| `python 不是内部或外部命令` | 安装没勾 PATH | 重装勾选或手动加目录 |
| 装了包却 `ModuleNotFoundError` | 装到了另一个解释器 | `python -m pip` 安装，核对 `sys.executable` |
| 激活后仍用全局包 | 终端未重开或激活失败 | 看提示符是否有 `(.venv)` |
| 换机器跑不起来 | 依赖没记录 | 提交依赖声明与锁文件 |

⚠ **系统 PATH 只该有一个默认解释器**，项目解释器交给虚拟环境按需激活。两个都写进去，冲突极难排查。

# 5. 项目结构

```text
myproject/
├─ .venv/                 # 虚拟环境，不提交
├─ src/
│  └─ myproject/
│     ├─ __init__.py
│     └─ main.py
├─ tests/
│  └─ test_main.py
├─ pyproject.toml         # 元数据与依赖
└─ .gitignore
```

```text
# .gitignore
.venv/
__pycache__/
*.pyc
dist/
build/
*.egg-info/
```

★ **关键认知**：`src/` 布局的核心是**隔离**——让项目源码和当前目录分开，测试时 `import myproject` 只能导入真正安装的包，不会因为"当前目录恰好有同名文件夹"而误导入。

# 6. 练习

1. 创建并激活虚拟环境，分别运行 `python -c "import sys; print(sys.executable)"` 和 `pip -V`，确认两者指向同一环境。
2. 建两套虚拟环境，分别安装 `requests==2.28.0` 和 `requests==2.31.0`，用 `pip show` 验证互不影响。
3. 写一个 `.gitignore`，确保 `.venv/`、`__pycache__/`、`*.pyc` 不被提交，并用 `git status` 验证。

# 7. 小结

| 概念 | 要点 |
|------|------|
| 解释器 | 用 `sys.executable` 确认真实路径 |
| 虚拟环境 | 独立 site-packages + PATH 前置 |
| 激活 | 只改当前终端 PATH，关窗即失效 |
| pip | 属于某个解释器，优先 `python -m pip` |
| PATH | 顺序查找，先命中先用 |
| 项目结构 | `src/` 隔离源码，`.venv` 不入库 |

下一篇进入语法与类型，讲清楚变量其实是"名字绑定"、动态类型的运行机制、数字与字符串的常见陷阱，以及 `None` 和类型转换的正确姿势。
