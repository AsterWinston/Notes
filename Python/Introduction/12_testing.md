测试不是"写完代码再补"的附属品，而是**让你敢改代码的安全网**。Python 有内置的 `unittest` 和更简洁的 `pytest`，两者都能完成任务，但日常开发几乎默认 `pytest`。本篇讲清楚测试怎么写、fixture 怎么管资源、mock 的边界在哪、覆盖率该怎么看。

# 1. 测试基础

## 1.1 为什么写测试

| 收益 | 说明 |
|------|------|
| 回归保护 | 改代码后立刻知道有没有破坏旧功能 |
| 活文档 | 测试展示了函数的预期用法 |
| 设计压力 | 难测的代码往往耦合过重 |
| 重构底气 | 有测试才敢动结构 |

★ **关键认知**：测试的价值不在"证明代码正确"，而在**在你修改代码时快速告诉你哪里坏了**。所以测试应覆盖"容易出错、经常改动、业务关键"的部分，而不是追求 100% 覆盖率。

## 1.2 unittest

标准库自带，用类组织测试，断言靠 `assertXxx` 方法：

```python
# test_calc.py
import unittest

def add(a, b):
    return a + b

class TestAdd(unittest.TestCase):
    def test_positive(self):
        self.assertEqual(add(1, 2), 3)

    def test_type_error(self):
        with self.assertRaises(TypeError):
            add("1", 2)

if __name__ == "__main__":
    unittest.main()
```

```bash
python -m unittest test_calc.py
```

## 1.3 pytest

先安装：`pip install pytest`。它用普通函数 + 原生 `assert`，更简洁：

```python
# test_calc.py
import pytest

def add(a, b):
    return a + b

def test_add_positive():
    assert add(1, 2) == 3

def test_add_type_error():
    with pytest.raises(TypeError):
        add("1", 2)
```

常用命令：`pytest -v`（显示每个用例）、`pytest -k "add"`（只跑名字含 add 的）、`pytest -x`（第一个失败就停）。

★ **关键认知**：`pytest` 的核心优势是**用原生 `assert`**，失败时会自动展示两边的值并展开差异，而 `unittest` 需要记一堆 `assertXxx`。加上 fixture、参数化、插件生态，`pytest` 已成为事实标准。**新项目直接用 `pytest`**。

# 2. fixture

## 2.1 定义与注入

fixture 是"测试前准备、测试后清理"的可复用函数。测试函数**按参数名**声明需要哪个 fixture：

```python
import pytest

@pytest.fixture
def sample_user():
    return {"name": "张三", "age": 20}

def test_user_name(sample_user):
    assert sample_user["name"] == "张三"
```

## 2.2 作用域

```python
@pytest.fixture(scope="session")     # 整个测试会话一次
def db_connection():
    conn = create_connection()
    yield conn
    conn.close()

@pytest.fixture(scope="module")      # 每个模块一次
def config():
    ...
```

⚠ **作用域越大，测试间耦合风险越高**。共享可变对象可能被前一个测试改坏，导致"单独跑通过、一起跑失败"。默认用 `function`，确有性能需要再放大。

## 2.3 yield fixture

```python
import pytest

@pytest.fixture
def temp_file(tmp_path):
    path = tmp_path / "data.txt"
    path.write_text("hello", encoding="utf-8")
    yield path                  # 测试执行
    if path.exists():           # yield 之后是清理阶段
        path.unlink()
```

内置好用的 fixture：`tmp_path`（临时目录）、`capsys`（捕获输出）、`monkeypatch`（临时改属性/环境变量）、`caplog`（捕获日志）。

# 3. 参数化

同一逻辑测多组数据，用 `@pytest.mark.parametrize`：

```python
import pytest

def is_even(n: int) -> bool:
    return n % 2 == 0

@pytest.mark.parametrize("n, expected", [
    (0, True),
    (1, False),
    (2, True),
    (-3, False),
])
def test_is_even(n, expected):
    assert is_even(n) is expected
```

每个参数组合生成独立用例，失败时能精确定位是哪组数据。

```python
@pytest.mark.parametrize("text", ["abc", "", "3.14"])
def test_invalid_int(text):
    with pytest.raises(ValueError):
        int(text)
```

★ **关键认知**：参数化把"复制粘贴多个测试函数"变成"一张数据表"。**测试逻辑只写一次，数据变了只改表**。边界值、异常值、空值都应进这张表，这是提升覆盖率最划算的方式。

# 4. mock

## 4.1 为什么需要

测试不应依赖真实网络、数据库、时间。mock 用来**替换这些外部依赖**，让测试快速、稳定、可重复。

```python
# 被测代码
import requests

def get_username(user_id: int) -> str:
    resp = requests.get(f"https://api.example.com/users/{user_id}")
    return resp.json()["name"]
```

直接测试会真的发请求——慢、可能失败、依赖网络。用 mock 替换：

```python
from unittest.mock import patch

def test_get_username():
    fake_response = type("Resp", (), {
        "json": lambda self: {"name": "张三"}
    })()

    with patch("requests.get", return_value=fake_response):
        assert get_username(1) == "张三"
```

## 4.2 patch 的路径

```python
from unittest.mock import patch

with patch("module.func") as mock_func:
    mock_func.return_value = 42
    assert module.func() == 42

mock_func.assert_called_once()
mock_func.assert_called_with(1, 2)
```

⚠ **patch 的路径要写"被测代码使用它的地方"，不是"它定义的地方"**：

```python
# 被测模块 app.py 里写的是：from requests import get

patch("requests.get")   # 错误：app 已绑定自己的引用
patch("app.get")        # 正确：patch app 模块里的那个名字
```

这是 mock 最经典的坑，报错通常是"调用了真实函数"而不是"patch 没生效"。

## 4.3 monkeypatch

适合临时改环境变量、字典、对象属性：

```python
def test_env(monkeypatch):
    monkeypatch.setenv("MODE", "test")
    import os
    assert os.environ["MODE"] == "test"
```

它会在测试结束后自动还原，不用手写清理。

★ **关键认知**：**mock 的边界要贴着"外部依赖"**。只 mock 网络、数据库、时间、随机数、文件系统，**不要 mock 被测代码自己的内部函数**。后者会让测试变成"验证实现细节"，一重构就红，失去保护意义。

# 5. 覆盖率

```bash
pip install pytest-cov
pytest --cov=src --cov-report=term-missing
```

★ **关键认知**：覆盖率衡量的是"**代码被执行过**"，不是"**逻辑被验证过**"。只调用函数却不断言，也能刷高覆盖率。所以：**覆盖率看趋势和缺口，不要当 KPI**。行覆盖 70%～85% 通常是合理区间，重点补 `Missing` 里的分支和异常路径。

⚠ **不要为了数字写无意义测试**。`assert True`、只调用不断言、mock 掉所有逻辑，都能让覆盖率好看，但对质量毫无帮助。

# 6. 测试组织

## 6.1 目录结构

```text
project/
├─ src/
│  └─ myapp/
│     ├─ calc.py
│     └─ parser.py
├─ tests/
│  ├─ conftest.py          # 共享 fixture
│  ├─ test_calc.py
│  └─ test_parser.py
└─ pyproject.toml
```

`conftest.py` 里的 fixture 会被同目录及子目录的测试自动发现，不需要 import。

## 6.2 测试金字塔与命名

| 层级 | 范围 | 速度 | 数量 |
|------|------|------|------|
| 单元测试 | 单个函数/类 | 毫秒 | 最多 |
| 集成测试 | 模块间协作 | 秒 | 适量 |
| 端到端测试 | 整个系统 | 分钟 | 少量 |

```text
        ┌──────────┐
        │   E2E    │  少量，慢
        ├──────────┤
        │ 集成测试  │  适量
        ├──────────┤
        │ 单元测试  │  大量，快
        └──────────┘
```

★ **关键认知**：测试应该**分层且数量递减**。单元测试快、定位准，是主力；集成测试验证协作；E2E 验证关键链路。全部写成 E2E 会让测试慢到没人愿意跑，最终形同虚设。

命名建议：`test_<被测对象>_<场景>_<预期>`，如 `test_divide_by_zero_raises`。

## 6.3 CI
CI 里安装依赖后运行 `pytest --cov=src --cov-fail-under=80`，覆盖率低于阈值就让构建失败。

# 7. 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|---------|
| 依赖执行顺序 | 单独跑通过、一起跑失败 | 每个测试独立 |
| mock 路径写错 | 实际调了真实依赖 | patch "使用处" |
| mock 内部函数 | 重构就红 | 只 mock 外部依赖 |
| 只调用不断言 | 覆盖率虚高 | 每个测试至少一个断言 |

浮点比较用 `pytest.approx`，例如 `assert 0.1 + 0.2 == pytest.approx(0.3)`。

# 8. 练习

1. 给第 4 篇的 `safe_divide(a, b)` 写测试：覆盖正常除法、除零、非数字输入，用 `pytest.raises` 验证异常类型。
2. 用 `@pytest.mark.parametrize` 给"判断闰年"的函数写 8 组数据，覆盖世纪年、普通闰年、平年。
3. 用 `unittest.mock.patch` 替换一个"调用外部 API"的函数，写两个测试（成功、500 错误）；再用 `tmp_path` fixture 写一个文件读写集成测试。

# 9. 小结

| 概念 | 要点 |
|------|------|
| `pytest` | 原生 `assert`，首选框架 |
| fixture | 按参数名注入，`yield` 做清理 |
| 作用域 | 默认 `function`，越大越易污染 |
| 参数化 | 一张数据表覆盖多组输入 |
| mock | 只替换外部依赖，路径写"使用处" |
| 覆盖率 | 看缺口，不当 KPI |
| 测试金字塔 | 单元多、集成适量、E2E 少 |
| CI | `--cov-fail-under` 防下滑 |

下一篇进入打包与发布，讲清楚 `pyproject.toml` 的结构、依赖声明与版本约束、构建 wheel/sdist、发布到 PyPI，以及如何用锁文件保证环境可复现。
