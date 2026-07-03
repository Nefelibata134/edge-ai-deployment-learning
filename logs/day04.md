# Day 04 学习记录

日期：2026-07-03

## 今日目标

- 理解为什么不建议直接在 Conda `base` 环境中做模型部署实验。
- 明确 `conda`、`pip`、`python`、`python3`、环境变量之间的关系。
- 创建一个专门用于模型部署学习的 Python 3.10 环境：`deploy310`。
- 在新环境中安装基础工具包，并验证 Python、pip、NumPy、OpenCV 是否正常。
- 学会激活、退出、查看、删除 Conda 环境的基本命令。

## 今日 7 小时计划

### 第 1 段：复盘 Day 03 的环境发现（0.5 小时）

任务：

- 确认当前终端是否处于 Conda `base` 环境。
- 查看 Conda、Python、pip 的来源。

需要执行：

```bash
conda --version
conda info --envs
which conda
which python
python --version
which pip
pip --version
```

需要理解：

- 终端前面的 `(base)` 表示当前激活的是 Conda 的 base 环境。
- `which python` 能告诉你当前真正使用的是哪个 Python。
- base 环境适合管理 Conda，不适合乱装项目依赖。

### 第 2 段：理解 Conda 环境隔离（1 小时）

任务：

- 理解“每个项目一个环境”的意义。
- 知道为什么模型部署优先使用 Python 3.10。

需要记住：

- `base`：Conda 的默认环境，不建议装大量项目依赖。
- `deploy310`：我们给模型部署学习准备的新环境。
- Python 3.10：目前对 PyTorch、ONNX、OpenCV、部署工具更稳。
- Python 3.13：太新，很多深度学习库支持不够稳。

产出：

- 能说清楚为什么不在 `base` 里直接装 PyTorch/ONNX/TensorRT。

### 第 3 段：创建 Python 3.10 环境（1 小时）

任务：

- 创建 `deploy310` 环境。
- 激活环境并确认 Python 版本。

需要执行：

```bash
conda create -n deploy310 python=3.10 -y
conda activate deploy310
python --version
which python
pip --version
```

如果 `conda create` 下载较慢，先不要乱换命令，把报错或卡住的位置发给 Codex。

需要理解：

- `conda create -n deploy310 python=3.10` 创建一个独立环境。
- `conda activate deploy310` 切换到这个环境。
- 终端提示符应该从 `(base)` 变成 `(deploy310)`。

### 第 4 段：安装基础 Python 包（1 小时）

任务：

- 升级当前环境里的 pip。
- 安装基础科学计算和图像处理包。

需要执行：

```bash
python -m pip install --upgrade pip
pip install numpy matplotlib opencv-python-headless tqdm
```

需要理解：

- 在 Conda 环境里，优先用当前环境的 `python -m pip` 或 `pip` 安装 Python 包。
- `opencv-python-headless` 更适合服务器/WSL/Jetson 这类不依赖桌面窗口的环境。

### 第 5 段：写一个环境验证脚本（1 小时）

任务：

- 创建一个小脚本，验证 Python、NumPy、OpenCV 是否正常。

需要执行：

```bash
mkdir -p ~/model-deploy-day04
cd ~/model-deploy-day04
nano check_env.py
```

在 `nano` 中写入：

```python
import sys
import numpy as np
import cv2

print("python:", sys.version)
print("numpy:", np.__version__)
print("opencv:", cv2.__version__)
print("array:", np.array([1, 2, 3]) * 2)
```

保存后执行：

```bash
python check_env.py
```

产出：

- 能看到 Python、NumPy、OpenCV 版本输出。

### 第 6 段：学习环境切换和退出（1 小时）

任务：

- 练习退出环境、重新进入环境。
- 理解环境切换后 `python` 会变。

需要执行：

```bash
conda deactivate
python --version
which python
conda activate deploy310
python --version
which python
conda info --envs
```

需要理解：

- `conda deactivate` 会退出当前环境，通常回到 `base`。
- `conda activate deploy310` 会重新进入部署环境。
- 星号 `*` 表示当前激活的环境。

### 第 7 段：整理输出与复盘（1.5 小时）

任务：

- 把今天关键输出发给 Codex。
- Codex 更新本日志和 README。
- 提交并推送 GitHub。

需要整理：

- `conda info --envs` 输出。
- `python --version` 在 base 和 deploy310 中分别是多少。
- `which python` 在 base 和 deploy310 中分别指向哪里。
- `pip --version`。
- `python check_env.py` 输出。

## 动手记录

待完成后补充。

## 今日完成情况

待完成后补充。

## 遇到的问题

待完成后补充。

## 今日复盘

今天最重要的收获：

待完成后补充。

还不清楚的点：

待完成后补充。

## 明日计划

- 开始 Python 基础复习：变量、列表、字典、函数、文件读写。
- 用 Python 写一个简单的图片信息读取脚本。
- 为后续 OpenCV 和模型推理代码做准备。
