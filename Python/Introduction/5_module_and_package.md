`import` 大概是 Python 里"天天用、却最少被理解"的语法。为什么有时 `ModuleNotFoundError`、为什么改了文件不生效、为什么相对导入一跑就报错、为什么两个模块互相 import 会炸——答案都在模块系统里。本篇从 `import` 的执行过程讲到包结构，把这些问题一次性解决。

# 1. 模块

## 1.1 模块与命名空间

一个 `.py` 文件就是一个模块，模块名是去掉后缀的文件名：

```text
project/
├─ main.py
└─ utils.py
```

```python
# utils.py
def add(a, b):
    return a + b

PI = 3.14159
```

```python
# main.py
import utils

print(utils.add(1, 2))   # 3
print(utils.PI)          # 3.14159
```

模块把代码组织成独立命名空间，避免不同文件的同名函数互相覆盖。

## 1.2 import 的写法

```python
import utils                     # 整个模块，用 utils.xxx 访问
import utils as u                # 起别名
from utils import add            # 只导入某个名字
from utils import add, PI        # 导入多个
from utils import add as plus    # 给名字起别名
from utils import *              # 导入所有公开名字（不推荐）
```

⚠ **`from module import *` 会污染命名空间**，正式代码一律避免。模块里可用 `__all__` 控制导出：

```python
__all__ = ["add", "PI"]
```

## 1.3 import 做了什么

```text
1. 在 sys.modules 缓存里查找 "utils"
   └─ 找到 → 直接返回，不再执行
2. 没找到 → 按 sys.path 顺序查找 utils.py
3. 找到文件 → 在新命名空间里从头执行整个文件
4. 把模块对象存入 sys.modules，并绑定到名字 "utils"
```

★ **关键认知**：**模块只在第一次 import 时执行一次**，之后都从 `sys.modules` 取。这解释了三个现象：顶层代码只跑一遍、循环导入可能拿到"执行到一半"的模块、改了文件必须重启进程才生效。

```python
import sys
print("utils" in sys.modules)   # 第一次导入后为 True
```

★ **关键认知**：模块顶层不要放有副作用的代码（打印、网络请求、读文件），真正的工作放进 `main()`：

```python
def main():
    print("作为脚本运行")

if __name__ == "__main__":
    main()
```

`__name__` 直接运行时是 `"__main__"`，被 import 时是模块名。这个判断是"既能导入、又能当脚本"的标准写法。

# 2. 包

## 2.1 包与 __init__.py

包是带 `__init__.py` 的目录（3.3+ 可省略，见 2.3）：

```text
myapp/
├─ __init__.py
├─ models/
│  ├─ __init__.py
│  ├─ user.py
│  └─ order.py
└─ utils.py
```

`__init__.py` 在包第一次导入时执行，常用来简化导入路径：

```python
# myapp/models/__init__.py
from .user import User
from .order import Order

__all__ = ["User", "Order"]
```

外部就能写 `from myapp.models import User`，不必知道 `User` 定义在 `user.py`。

## 2.2 子模块导入

```python
import myapp.models.user              # 完整路径
from myapp.models import user         # 导入模块
from myapp.models.user import User    # 导入类
from myapp.models import User         # 走 __init__.py 的再导出
```

★ **关键认知**：`from package import name` 先看包内有没有同名子模块，再看 `__init__.py` 有没有这个名字。所以 `from myapp.models import User` 能成功，前提是 `__init__.py` 里导入过它。

## 2.3 命名空间包

3.3+ 允许没有 `__init__.py` 的目录作为**命名空间包**，用于合并分散在不同位置的同名包。⚠ **普通项目不要依赖它**，保留 `__init__.py` 更清晰，也避免"目录被误当成包"。

# 3. 模块搜索路径

## 3.1 sys.path 的顺序

```python
import sys
for p in sys.path:
    print(p)
```

```text
1. 脚本所在目录（或 "" 表示当前目录）
2. PYTHONPATH 环境变量里的目录
3. 标准库目录
4. site-packages（第三方包）
```

★ **关键认知**：`sys.path[0]` 是**启动脚本所在目录**，不是当前工作目录。从别处运行 `python project/main.py` 时，`sys.path[0]` 是 `project/`，不是执行命令的目录。

## 3.2 同名遮蔽

因为脚本目录排最前，**本地同名文件会遮蔽标准库**：

```text
project/
├─ random.py     # 自己写的
└─ main.py
```

```python
import random   # 导入的是 project/random.py，不是标准库！
```

⚠ **绝对不要用标准库或常用库的名字命名文件**：`random.py`、`json.py`、`email.py`、`types.py`、`queue.py`、`socket.py`。这类问题报错往往指向标准库内部，极难排查。

## 3.3 修改路径

```python
import sys
sys.path.insert(0, "/path/to/libs")
import mylib
```

但**生产代码应避免手动改 `sys.path`**。优先级是：安装包（`pip install -e .`）→ `PYTHONPATH` → 调整运行方式 → 手动改路径（最后手段）。

# 4. 相对导入与绝对导入

## 4.1 绝对导入

```python
# myapp/models/order.py
from myapp.models.user import User
```

清晰、无歧义，PEP 8 推荐优先使用。缺点是包改名要全量替换。

## 4.2 相对导入

```python
# myapp/models/order.py
from .user import User        # 同级模块
from ..utils import helper    # 上一级包
from . import user            # 当前包的子模块
```

| 写法 | 含义 |
|------|------|
| `.` | 当前包 |
| `..` | 上一级包 |
| `...` | 上两级包 |

⚠ **相对导入只能在包内使用，且必须"作为包的一部分"运行**。直接运行 `python myapp/models/order.py` 会报：

```text
ImportError: attempted relative import with no known parent package
```

因为此时 `order.py` 的 `__name__` 是 `"__main__"`，解释器不知道父包是谁。正确方式是从项目根用模块启动：

```bash
python -m myapp.models.order
```

# 5. 循环导入

## 5.1 产生原因

```python
# a.py
from b import func_b

def func_a():
    return func_b()
```

```python
# b.py
from a import func_a

def func_b():
    return func_a()
```

```text
import a
  └─▶ 执行 a.py
        └─▶ import b
              └─▶ 执行 b.py
                    └─▶ from a import func_a
                          ✗ a 尚未执行完，func_a 不存在
```

★ **关键认知**：循环导入的根因不是"互相引用"，而是**在模块顶层、名字还没定义时就使用了它**。解决思路永远是"把引用推迟到真正需要的时候"。

## 5.2 解决方案

| 方案 | 做法 | 适用 |
|------|------|------|
| 延迟导入 | 函数内部 `import` | 最常用 |
| 只导入模块 | `import b` 而非 `from b import x` | 避开绑定时机 |
| 提取公共模块 | 抽到 `common.py` | 根治 |
| 类型检查导入 | `if TYPE_CHECKING:` | 仅注解使用 |

```python
# 延迟导入
def func_a():
    from b import func_b
    return func_b()
```

```python
# 仅为类型注解导入
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from b import BType
```

⚠ **循环导入往往是架构信号**：说明两个模块耦合过紧。先尝试提取公共依赖，而不是用延迟导入掩盖。

# 6. 常见错误

| 错误 | 现象 | 解决 |
|------|------|------|
| 文件与标准库同名 | 导入到自己的文件 | 重命名 |
| 改代码不生效 | 模块已缓存 | 重启进程 |
| 相对导入报错 | 直接运行包内文件 | `python -m package.module` |
| 循环导入 `ImportError` | 顶层互相引用 | 延迟导入 / 提取公共模块 |
| 顶层写副作用代码 | import 时执行 | 放进 `main()` |
| 手动改 `sys.path` | 换环境崩 | 安装包 / `PYTHONPATH` |

# 7. 练习

1. 创建 `shapes/` 包，内含 `circle.py` 和 `square.py`，各定义面积函数；在 `__init__.py` 里再导出，使外部可写 `from shapes import circle_area`。
2. 复现循环导入：写 `a.py`、`b.py` 互相 `from ... import ...`，观察报错；再用函数内延迟导入修复，并解释原因。
3. 建一个 `json.py`，运行 `import json; print(json.__file__)` 观察导入的是哪个文件；删除后对比 `sys.path[0]`，解释同名遮蔽。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 模块 | 一个 `.py` 文件，首次导入执行一次 |
| `sys.modules` | 模块缓存，改文件不自动重载 |
| 包 | 带 `__init__.py` 的目录 |
| `__name__` | 直接运行时为 `__main__` |
| `sys.path` | 脚本目录 → PYTHONPATH → 标准库 → 三方包 |
| 相对导入 | 只能在包内、用 `-m` 运行 |
| 循环导入 | 延迟绑定或提取公共模块 |

下一篇进入面向对象，讲清楚 `self` 的本质、类属性与实例属性的区别、继承与 MRO、魔术方法、`property`，以及 `dataclass` 如何省掉大量样板代码。
