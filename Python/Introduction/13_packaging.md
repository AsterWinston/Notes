写脚本和做项目是两件事。一旦代码要被别人安装、被 CI 构建、发布到 PyPI，"依赖写在哪、版本怎么约束、产物长什么样"就变成必须回答的问题。现代 Python 打包围绕 `pyproject.toml` 展开，本篇从元数据讲到发布与可复现环境。

# 1. 项目元数据

## 1.1 从 setup.py 到 pyproject.toml

```text
旧方案                        新方案
setup.py（命令式）      →     pyproject.toml（声明式）
setup.cfg（配置）       →
requirements.txt       →     [project.dependencies]
```

★ **关键认知**：`pyproject.toml` 不是"又一个配置文件"，而是**构建系统的标准契约**。它声明"项目是什么、依赖什么、用什么后端构建"，任何符合 PEP 517 的工具（pip、build、uv）都能读懂。**新项目不要再写 `setup.py`**。

## 1.2 基本结构

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "myapp"
version = "0.1.0"
description = "一个示例项目"
readme = "README.md"
requires-python = ">=3.11"
license = { text = "MIT" }
dependencies = [
    "requests>=2.31,<3.0",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0", "pytest-cov>=5.0", "ruff>=0.5"]

[project.scripts]
myapp = "myapp.cli:main"
```

| 字段 | 作用 |
|------|------|
| `name` | 包名，PyPI 唯一标识 |
| `version` | 版本号，遵循 PEP 440 |
| `requires-python` | 支持的 Python 版本范围 |
| `dependencies` | 运行时依赖 |
| `optional-dependencies` | 可选依赖组 |
| `scripts` | 安装后生成的命令行入口 |

## 1.3 构建后端

| 后端 | 特点 | 适用 |
|------|------|------|
| `hatchling` | 现代、零配置、速度快 | 新项目推荐 |
| `setuptools` | 生态最广、兼容旧项目 | 迁移项目 |
| `flit` | 极简，适合纯 Python 包 | 小型库 |
| `poetry-core` | 与 Poetry 配套 | 用 Poetry 时 |

```toml
# setuptools 版本
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"
```

★ **关键认知**：`build-backend` 决定构建行为，但**对使用者透明**。`pip install` 时会自动创建隔离环境、安装 `requires` 里的构建依赖、调用后端。切换后端通常不需要改使用方式，这是 PEP 517 隔离设计的意义。

# 2. 依赖管理

## 2.1 声明依赖

```toml
[project]
dependencies = [
    "requests>=2.31,<3.0",                    # 带版本范围
    "click",                                   # 不约束
    "tomli; python_version < '3.11'",          # 环境标记
    "uvloop>=0.19; sys_platform != 'win32'",   # 仅非 Windows
]
```

环境标记让依赖按平台/版本生效，常见写法：`"colorama; sys_platform == 'win32'"`。

## 2.2 版本约束

| 写法 | 含义 | 建议 |
|------|------|------|
| `==2.31.0` | 精确锁定 | 只在锁文件里用 |
| `>=2.31` | 最低版本 | 太宽松 |
| `>=2.31,<3.0` | 兼容范围 | **最推荐** |
| `~=2.31` | 等价 `>=2.31,<3.0` | 语义化版本简写 |
| `!=2.32.0` | 排除问题版本 | 临时规避 bug |

★ **关键认知**：**库项目声明"兼容范围"，应用项目锁定"精确版本"**。库要被别人安装，范围太窄会与下游冲突；应用要可复现，范围太宽会导致不同机器装出不同版本。所以库写 `>=2.31,<3.0`，应用用锁文件钉死到 `2.31.0`。

⚠ **不要滥用 `==`**。库的 `dependencies` 里写死精确版本，会造成依赖冲突——别的包需要 `2.30` 时无法共存。精确版本属于锁文件，不属于包的元数据。

## 2.3 可选依赖与锁文件

```toml
[project.optional-dependencies]
dev = ["pytest>=8.0", "ruff>=0.5"]
docs = ["sphinx>=7.0"]
```

```bash
pip install -e ".[dev]"          # 主依赖 + dev 组
pip install -e ".[dev,docs]"     # 多个组
```

`optional-dependencies` 面向**使用者**（`pip install "myapp[postgres]"`）；PEP 735 的 `[dependency-groups]` 面向**开发者**（只在本仓库用）。

| 工具 | 声明文件 | 锁文件 |
|------|---------|--------|
| pip-tools | `requirements.in` | `requirements.txt` |
| Poetry | `pyproject.toml` | `poetry.lock` |
| uv | `pyproject.toml` | `uv.lock` |
| PDM | `pyproject.toml` | `pdm.lock` |

★ **关键认知**：锁文件必须**提交到仓库**（应用项目），它记录所有直接与间接依赖的精确版本和哈希，是"换台机器装出完全一样环境"的唯一保证。`pyproject.toml` 可以漂移，锁文件不能。

# 3. 构建

## 3.1 wheel 与 sdist

| 产物 | 后缀 | 内容 | 用途 |
|------|------|------|------|
| wheel | `.whl` | 预构建分发包 | 直接安装，快 |
| sdist | `.tar.gz` | 源码包 | 现场构建 |

```text
源码
 ├─ 构建 ──▶ dist/myapp-0.1.0-py3-none-any.whl
 └─ 构建 ──▶ dist/myapp-0.1.0.tar.gz
```

`py3-none-any` 表示"Python 3、无 ABI 依赖、任意平台"。纯 Python 包都是这个标签；含 C 扩展的会带平台标签，如 `cp312-cp312-win_amd64`。

## 3.2 构建命令

```bash
pip install build

python -m build              # wheel + sdist，输出到 dist/
python -m build --wheel      # 只构建 wheel
```

构建前先清理旧的 `dist/`、`build/`、`*.egg-info/`，避免把旧产物打进去。

## 3.3 布局与打包配置

`src/` 布局需要在构建后端里指定包位置：

```toml
# hatchling
[tool.hatch.build.targets.wheel]
packages = ["src/myapp"]
```

```toml
# setuptools
[tool.setuptools.packages.find]
where = ["src"]
```

⚠ **忘记配置会导致"装上了但 import 不到"**。构建成功不代表包被正确收录——用 `python -m zipfile -l dist/*.whl` 检查 wheel 里到底有没有你的模块。

★ **关键认知**：构建和安装是两回事。`pip install -e .` 是**开发模式**，把源码目录挂到环境里，改代码立即生效；`python -m build` 产出的是**分发包**，用于发布和部署。发布前一定要在干净环境里 `pip install dist/*.whl` 验证。

# 4. 发布

## 4.1 先发 TestPyPI

```bash
python -m build
python -m twine upload --repository testpypi dist/*
pip install --index-url https://test.pypi.org/simple/ myapp
```

## 4.2 发布到 PyPI

```bash
pip install twine
python -m twine check dist/*    # 校验元数据
python -m twine upload dist/*   # 需要 PyPI API token
```

⚠ **版本号一旦上传就不可覆盖**。PyPI 不允许重复上传同名同版本，改代码后必须**递增版本号**再传。发布前务必在本地和 TestPyPI 验证充分。

## 4.3 版本号规范

```text
主版本.次版本.修订号
  │      │      └─ 修 bug，向后兼容
  │      └──────── 加功能，向后兼容
  └─────────────── 破坏性变更
```

| 变更 | 版本 |
|------|------|
| 修 bug | `1.2.3 → 1.2.4` |
| 加功能 | `1.2.3 → 1.3.0` |
| 破坏兼容 | `1.2.3 → 2.0.0` |
| 预发布 | `2.0.0a1`、`2.0.0rc1` |

用 `hatch-vcs` 或 `setuptools-scm` 可从 Git tag 自动生成版本号，避免手改遗漏。

# 5. 可复现环境

"在我机器上能跑"通常有三个来源：

| 来源 | 表现 | 解决 |
|------|------|------|
| Python 版本不同 | 语法/标准库行为差异 | `requires-python` + CI 矩阵 |
| 依赖版本漂移 | 间接依赖升级导致行为变化 | 锁文件 |
| 系统库差异 | C 扩展、编译环境不同 | 容器 / 预编译 wheel |

| 工具 | 速度 | 锁文件 | 学习成本 |
|------|------|--------|---------|
| pip + venv | 慢 | 需 pip-tools | 低 |
| Poetry | 中 | 有 | 中 |
| uv | 极快 | 有 | 低 |
| PDM | 中 | 有 | 中 |

```bash
# uv 示例
uv venv
uv sync                # 按 uv.lock 安装，严格复现
uv add requests
uv add --dev pytest
```

★ **关键认知**：可复现的核心不是"用哪个工具"，而是**两件事**：锁定 Python 版本、锁定依赖版本。工具只是让这件事更省力。哪怕只用 `pip`，把 `pip freeze` 的结果提交、在 CI 里按它安装，也能做到基本可复现。

# 6. 常见错误

| 错误 | 现象 | 正确做法 |
|------|------|---------|
| 库依赖写 `==` | 下游冲突 | 写兼容范围 |
| 锁文件不提交 | 环境不可复现 | 提交 `uv.lock` 等 |
| 忘配 `packages` | 装上了 import 不到 | 检查 wheel 内容 |
| 版本号重复上传 | PyPI 拒绝 | 递增版本 |
| 发布前不验证 | 安装失败 | 干净环境装 wheel |
| 只测源码不测产物 | 打包遗漏文件 | `pip install dist/*.whl` |
| 新项目写 `setup.py` | 过时、难维护 | 用 `pyproject.toml` |

# 7. 练习

1. 为一个已有脚本创建 `pyproject.toml`（用 hatchling），声明 `requests`、`click` 依赖、`dev` 可选组、一个 `myapp` 命令行入口；用 `pip install -e ".[dev]"` 验证。
2. 用 `python -m build` 构建 wheel 和 sdist，用 `python -m zipfile -l dist/*.whl` 检查产物里包含哪些文件，确认包和入口点都在。
3. 给项目加一个有破坏性的改动，按 SemVer 决定新版本号；说明为什么不能沿用旧版本号重新上传，以及应先在 TestPyPI 做什么验证。

# 8. 小结

| 概念 | 要点 |
|------|------|
| `pyproject.toml` | 唯一标准入口，声明元数据与依赖 |
| 构建后端 | hatchling / setuptools，对使用者透明 |
| 版本约束 | 库写范围，应用写锁文件 |
| wheel / sdist | 预构建分发包 / 源码包 |
| `python -m build` | 产出 `dist/` 下的包 |
| PyPI | 版本不可覆盖，先发 TestPyPI |
| 版本号 | SemVer，破坏性变更升主版本 |
| 可复现 | 锁 Python + 锁依赖 |

下一篇是模块总结，用知识地图串起 14 篇的主线，给出高频自检题、常见坑清单和继续深入的学习路线。
