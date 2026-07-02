# Day 01 学习记录

日期：2026-07-02

## 今日目标

- 明确暑假主方向：边缘视觉部署。
- 建立 60 天学习路线。
- 明确两个就业项目。
- 准备 GitHub 记录结构。
- 整理 Jetson Orin Nano 采购清单。

## 今日结论

主方向选择：边缘视觉部署。

两个项目：

- 项目 1：PC 端 YOLO 模型 ONNX/TensorRT 部署与性能优化。
- 项目 2：Jetson Orin Nano 实时视觉感知系统，可扩展为视觉小车。

## 今日任务

- 阅读 `roadmap/personal-60-day-plan.md`，确认 60 天路线。
- 阅读 `hardware/jetson-buying-list.md`，确认 Jetson Orin Nano 采购和配件。
- 检查 GitHub 仓库是否已经可访问。
- 检查本机基础环境：Git、Python、VS Code、系统版本、CPU、内存、显卡。
- 判断是否需要安装 WSL2 + Ubuntu。
- 整理今天的环境检查结果。

## 今日 7 小时计划

### 第 1 段：确认路线和仓库（1 小时）

任务：

- 阅读个人版路线：`roadmap/personal-60-day-plan.md`。
- 阅读就业方向：`career/direction-and-jobs.md`。
- 确认 GitHub 仓库地址：`https://github.com/Nefelibata134/edge-ai-deployment-learning`。

产出：

- 能用自己的话说清主方向：边缘视觉部署。
- 能说清两个就业项目分别是什么。

### 第 2 段：电脑环境检查（1.5 小时）

任务：

- 查看 Windows 版本。
- 查看 CPU、内存、显卡。
- 检查 Git 是否可用。
- 检查 Python 是否可用。
- 检查 VS Code 是否已安装。

建议记录：

- 系统：
- CPU：
- 内存：
- 显卡：
- Python 版本：
- Git 版本：
- VS Code：

### 第 3 段：Git 和 GitHub 熟悉（1 小时）

任务：

- 理解本地仓库、远程仓库、commit、push 的关系。
- 查看当前仓库提交记录。
- 打开 GitHub 页面确认文件已经同步。

需要理解：

- `git status`：看当前文件状态。
- `git add`：把修改加入暂存区。
- `git commit`：保存一次本地版本。
- `git push`：推送到 GitHub。

### 第 4 段：安装和环境规划（1.5 小时）

任务：

- 确认是否已有 Python 3。
- 如果没有合适 Python，准备安装 Miniconda 或 Anaconda。
- 确认是否已有 VS Code。
- 了解 WSL2 + Ubuntu 是什么，判断是否需要安装。

今天不强求一次装完所有环境，先把现状查清楚。

### 第 5 段：Jetson 采购确认（1 小时）

任务：

- 阅读 `hardware/jetson-buying-list.md`。
- 确认开发板、SSD、摄像头、电源、散热、线材。
- 不急着买复杂小车套件，先保证 Jetson + 摄像头能跑通。

### 第 6 段：复盘和记录（1 小时）

任务：

- 把今天检查到的电脑配置发给 Codex。
- 记录已经完成和没完成的部分。
- 更新本文件的“动手记录”和“复盘”。
- 提交并推送到 GitHub。

## 动手记录

- 系统：Windows 11 家庭版 25H2，OS 内部版本 26200.8655。
- CPU：13th Gen Intel(R) Core(TM) i9-13950HX。
- 内存：15.65 GB。
- 显卡：NVIDIA GeForce RTX 4070 Laptop GPU，显存 8188 MiB。
- NVIDIA 驱动：NVIDIA-SMI 610.47，CUDA UMD Version 13.3。
- Python：
  - `py --version`：Python 3.13.9。
  - `py -0p` 显示已安装 Python 3.13 和 Python 3.10。
  - 已通过 `winget install -e --id Python.Python.3.10` 安装 Python 3.10.11。
- Git：git version 2.53.0.windows.2。
- VS Code：1.127.0，x64。
- WSL2 / Ubuntu：
  - 默认分发：Ubuntu-22.04。
  - 默认版本：WSL2。
  - 当前 Ubuntu-22.04 状态：Stopped。
- 浏览器工具：Kimi WebBridge 已修好，可连接 Chrome 并打开/读取网页。

## 今日完成情况

- 已完成本机基础环境检查。
- 已确认本机配置足够支持 PC 端模型部署学习。
- 已安装 Python 3.10.11，后续适合用于 PyTorch/ONNX/TensorRT 相关实验。
- 已确认 Git 和 VS Code 可用。
- 已确认 WSL2 + Ubuntu-22.04 已安装，后续可以作为 Linux 学习环境。
- 已确认 GitHub 学习仓库可用，并已经建立学习记录流程。
- 已更新 Jetson Orin Nano 采购清单。

## 遇到的问题

- 当前 `py --version` 默认指向 Python 3.13.9，深度学习生态对 Python 3.13 的支持可能不如 Python 3.10 稳定。
- 后续建议项目环境优先使用 Python 3.10。
- WSL2 已安装但当前未启动，明天学习 Linux 时需要进入 Ubuntu 环境实际练习。
- Windows 上 `wsl -l -v` 输出有乱码样式，不影响使用。

## 今日复盘

今天最重要的收获：

- 明确了主线是边缘视觉部署。
- 完成了学习仓库、GitHub、浏览器工具和本机环境检查。
- 电脑配置较强，RTX 4070 Laptop GPU 可以支撑 PC 端模型推理、ONNX 和 PyTorch 实验。
- Python 3.10 已安装，后续可以避免 Python 3.13 带来的兼容性问题。

还不清楚的点：

- WSL2 Ubuntu 环境内的 Linux 命令、包管理、文件路径和 Windows 目录映射还需要明天系统学习。
- 后续需要决定使用 venv、conda 还是 WSL2 环境来管理不同项目的 Python 依赖。

## 明日计划

- 学 Linux 基础命令：目录、文件、权限、进程、网络。
- 启动 Ubuntu-22.04，熟悉 WSL2。
- 练习 `pwd`、`ls`、`cd`、`mkdir`、`touch`、`cp`、`mv`、`rm`、`cat`、`less`、`grep`。
- 记录 Windows 路径和 Linux 路径的对应关系。
- 把 Day 02 学习记录提交到 GitHub。

## 需要确认

- GitHub 用户名。
- 仓库名。
- 是否公开仓库。
- 当前电脑配置：已确认。
