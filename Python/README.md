本板块整理 Python 语言基础与 PyTorch 深度学习入门。前者面向"能写、能调试、能测试、能打包"的工程能力，后者面向"能训练、能评估、能保存、能推理"的最小闭环。

# 1. 模块定位

## 1.1 本模块解决什么问题

- Python 语法简单，但动态类型、作用域、可变对象和并发模型容易踩坑。
- 初学者会写脚本，却不会组织模块、测试和依赖。
- PyTorch 教程常常直接堆模型，缺少数据、训练循环和排错的完整链路。
- 训练不收敛、显存爆掉、指标异常时，不知道该从哪一层排查。

## 1.2 与其他板块的关系

| 板块 | 关联 |
|------|------|
| AI Agent | Python 是多数 Agent 框架和实验脚本的主要语言 |
| File | JSON/YAML 配置、数据读写 |
| C&CPP | 扩展模块、性能热点下沉 |
| Rust | 对比学习：所有权、并发与错误处理 |

# 2. 学习目标

## 2.1 知识目标

- 掌握 Python 类型、函数、模块、面向对象、异常和文件 IO。
- 理解 GIL、线程、进程、asyncio 的适用边界。
- 掌握 pytest、依赖管理和打包。
- 理解 Tensor、自动求导、模型、数据加载、训练循环和推理。

## 2.2 能力目标

- 能独立编写可维护的 Python 脚本和小型项目。
- 能为 IO 密集/CPU 密集任务选择并发方案。
- 能训练一个可复现的 PyTorch 模型并保存最佳权重。
- 能定位过拟合、数据泄漏、梯度异常和显存问题。

# 3. 文件清单与大纲

## 3.1 Introduction

| 文件 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_install_and_environment.md](Introduction/1_install_and_environment.md) | 安装与环境 | 解释器、虚拟环境、pip、项目结构 |
| [2_syntax_and_types.md](Introduction/2_syntax_and_types.md) | 语法与类型 | 动态类型、数字、字符串、None、转换 |
| [3_control_flow.md](Introduction/3_control_flow.md) | 控制流 | 条件、循环、range、match |
| [4_function.md](Introduction/4_function.md) | 函数 | 参数、作用域、闭包、lambda、注解 |
| [5_module_and_package.md](Introduction/5_module_and_package.md) | 模块与包 | import、包结构、搜索路径 |
| [6_oop.md](Introduction/6_oop.md) | 面向对象 | 类、继承、组合、魔术方法、dataclass |
| [7_exception.md](Introduction/7_exception.md) | 异常 | 异常体系、上下文管理、自定义异常 |
| [8_file_and_io.md](Introduction/8_file_and_io.md) | 文件与 IO | open、编码、pathlib、JSON/CSV |
| [9_standard_library.md](Introduction/9_standard_library.md) | 标准库 | collections、itertools、datetime、re |
| [10_concurrency.md](Introduction/10_concurrency.md) | 并发 | GIL、线程、进程、线程池、锁 |
| [11_async.md](Introduction/11_async.md) | 异步 | asyncio、协程、Task、超时与取消 |
| [12_testing.md](Introduction/12_testing.md) | 测试 | pytest、fixture、参数化、mock |
| [13_packaging.md](Introduction/13_packaging.md) | 打包 | pyproject.toml、构建、发布、版本 |
| [14_interview_and_summary.md](Introduction/14_interview_and_summary.md) | 自检与总结 | 高频问题、常见坑、学习路线 |

## 3.2 PyTorch

| 文件 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_overview_and_tensor.md](PyTorch/1_overview_and_tensor.md) | 概览与 Tensor | 创建、形状、dtype、device、运算 |
| [2_autograd.md](PyTorch/2_autograd.md) | 自动求导 | 计算图、backward、梯度累积、detach |
| [3_nn_module.md](PyTorch/3_nn_module.md) | nn.Module | 层、参数、forward、模型组合 |
| [4_dataset_and_dataloader.md](PyTorch/4_dataset_and_dataloader.md) | 数据加载 | Dataset、DataLoader、batch、collate |
| [5_training_loop.md](PyTorch/5_training_loop.md) | 训练循环 | 训练/验证、loss、反向、指标 |
| [6_optimizer_and_scheduler.md](PyTorch/6_optimizer_and_scheduler.md) | 优化器与调度 | SGD/Adam、学习率、权重衰减 |
| [7_gpu_and_mixed_precision.md](PyTorch/7_gpu_and_mixed_precision.md) | GPU 与混合精度 | device、显存、AMP、GradScaler |
| [8_save_and_load.md](PyTorch/8_save_and_load.md) | 保存与加载 | state_dict、checkpoint、恢复训练 |
| [9_visualization_and_deployment.md](PyTorch/9_visualization_and_deployment.md) | 可视化与部署 | TensorBoard、TorchScript、ONNX |
| [10_interview_and_summary.md](PyTorch/10_interview_and_summary.md) | 排错与总结 | 过拟合、数据泄漏、训练排错 |

# 4. 推荐阅读路径

```text
Python 初学者              PyTorch 初学者             工程化/面试
     │                          │                         │
     ▼                          ▼                         ▼
 1 → 2 → 3 → 4              Tensor → Autograd         测试 12
     │                          │                         │
     ▼                          ▼                         ▼
 5 → 6 → 7 → 8              nn.Module → 数据           打包 13
     │                          │                         │
     ▼                          ▼                         ▼
 9 → 10 → 11 → 12           训练循环 → 优化器           自检 14
```

- **快速上手**：Python 1 → 2 → 3 → 4 → 8，先能写脚本处理文件。
- **工程能力**：5 → 6 → 7 → 12 → 13，重点掌握模块、异常、测试和打包。
- **深度学习入门**：PyTorch 1 → 2 → 3 → 4 → 5，先跑通一次完整训练。
- **训练排错**：PyTorch 6 → 7 → 8 → 10，重点看指标、显存和保存恢复。

# 5. 阅读约定

| 标记 | 含义 |
|------|------|
| `★ **关键认知**` | 本节最核心的机制或结论 |
| `⚠` | 常见坑点或易误解处 |
| 代码块 | 均标注语言，示例尽量可直接运行 |
| 命令清单 | 可照抄执行，注意虚拟环境和依赖版本 |

# 6. 小结

| 主题 | 完成标准 |
|------|----------|
| Python 基础 | 能读写文件、组织模块、处理异常 |
| 并发 | 能根据 IO/CPU 选择线程、进程或 asyncio |
| 测试与打包 | 能写 pytest、管理依赖、构建分发包 |
| PyTorch | 能训练、评估、保存和加载模型 |
| 排错 | 能定位不收敛、过拟合、显存和数据问题 |

建议先完成 Python Introduction，再进入 PyTorch；如果已有 Python 基础，可直接从 PyTorch 1 开始，遇到语法问题再回查。
