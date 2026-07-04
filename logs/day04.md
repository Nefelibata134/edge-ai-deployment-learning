# Day 04 学习记录

日期：2026-07-04

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

- WSL2 Ubuntu 启动时间：2026-07-04 09:10 CST。
- 当前 WSL2 基础信息：

```text
Ubuntu 22.04.5 LTS
Linux 6.6.87.2-microsoft-standard-WSL2 x86_64
Memory usage: 5%
IPv4 address for eth1: 192.168.3.44
```

- Conda 基础信息：

```text
conda --version -> conda 26.1.1
which conda -> /home/nefelibata/miniconda3/bin/conda
```

- 创建环境前已有环境：

```text
# conda environments:
#
# * -> active
# + -> frozen
base                 *   /home/nefelibata/miniconda3
auto                     /home/nefelibata/miniconda3/envs/auto
```

- `base` 环境中的 Python 和 pip：

```text
which python -> /home/nefelibata/miniconda3/bin/python
python --version -> Python 3.13.12
which pip -> /home/nefelibata/miniconda3/bin/pip
pip --version -> pip 26.0.1 from /home/nefelibata/miniconda3/lib/python3.13/site-packages/pip (python 3.13)
```

- 创建 Python 3.10 环境：

```bash
conda create -n deploy310 python=3.10 -y
```

创建成功后环境位置：

```text
/home/nefelibata/miniconda3/envs/deploy310
```

- 激活 `deploy310` 后的 Python 和 pip：

```text
python --version -> Python 3.10.20
which python -> /home/nefelibata/miniconda3/envs/deploy310/bin/python
pip --version -> pip 26.1.2 from /home/nefelibata/miniconda3/envs/deploy310/lib/python3.10/site-packages/pip (python 3.10)
```

- 升级 pip：

```text
python -m pip install --upgrade pip
Requirement already satisfied: pip in ./miniconda3/envs/deploy310/lib/python3.10/site-packages (26.1.2)
```

- 安装基础包时第一次把 `tqdm` 误写为 `tqam`，安装失败；随后单独安装 `tqdm` 成功：

```text
Successfully installed tqdm-4.68.3
```

- 安装 NumPy、Matplotlib、OpenCV：

```text
Successfully installed contourpy-1.3.2 cycler-0.12.1 fonttools-4.63.0 kiwisolver-1.5.0 matplotlib-3.10.9 numpy-2.2.6 opencv-python-headless-5.0.0.93 pillow-12.3.0 pyparsing-3.3.2 python-dateutil-2.9.0.post0 six-1.17.0
```

- 创建 Day 04 练习目录：

```bash
mkdir -p ~/model-deploy-day04
cd ~/model-deploy-day04
```

- `check_env.py` 第一次内容中出现两个问题：

```text
import numpy as numpy
print("numpy:", np._version_)
```

问题：

- `import numpy as numpy` 后不能用 `np`。
- 版本属性应写成 `__version__`，不是 `_version_`。

- 修正后脚本运行成功：

```text
python: 3.10.20 (main, Jun 11 2026, 15:17:37) [GCC 14.3.0]
numpy: 2.2.6
opencv: 5.0.0
array: [2 4 6]
```

- 环境切换验证：

```text
conda deactivate
python --version -> Python 3.13.12
which python -> /home/nefelibata/miniconda3/bin/python

conda activate deploy310
python --version -> Python 3.10.20
which python -> /home/nefelibata/miniconda3/envs/deploy310/bin/python
```

- 最终 Conda 环境列表：

```text
# conda environments:
#
# * -> active
# + -> frozen
base                     /home/nefelibata/miniconda3
auto                     /home/nefelibata/miniconda3/envs/auto
deploy310            *   /home/nefelibata/miniconda3/envs/deploy310
```

## 今日完成情况

- 已确认 WSL2 中 Conda 版本为 26.1.1。
- 已确认 `base` 环境使用 Python 3.13.12，不适合直接作为模型部署实验环境。
- 已成功创建 `deploy310` 环境。
- 已成功激活 `deploy310`，并确认其 Python 版本为 3.10.20。
- 已安装 `tqdm`、`numpy`、`matplotlib`、`opencv-python-headless` 等基础包。
- 已创建 `~/model-deploy-day04/check_env.py` 环境验证脚本。
- 已成功验证 Python、NumPy、OpenCV 和数组计算。
- 已练习 `conda deactivate` 和 `conda activate deploy310`，并观察到不同环境下 `python` 路径和版本不同。
- 已用 `conda info --envs` 查看环境列表，并确认 `deploy310` 为当前激活环境。

## 遇到的问题

- `python --versiom` 拼写错误，正确命令是 `python --version`。
- `python -m pip insatll --upgrade pip` 拼写错误，正确命令是 `python -m pip install --upgrade pip`。
- `pip install numpy matplotlib opencv-python-headless tqam` 中 `tqdm` 写成了 `tqam`，导致找不到包；随后用 `pip install tqdm` 成功安装。
- 第一次写 `check_env.py` 时 `import numpy as numpy` 却使用 `np`，导致 `NameError: name 'np' is not defined`。
- 第二次写版本号时使用了 `np._version_` 和 `cv2._version_`，正确写法是 `np.__version__` 和 `cv2.__version__`。
- `conda infp --envs` 拼写错误，正确命令是 `conda info --envs`。

## 今日复盘

今天最重要的收获：

- Conda 环境的核心价值是隔离项目依赖；以后不要把模型部署依赖直接装进 `base`。
- 同一台机器上可以有多个 Python，当前使用哪个 Python 由激活的环境和 `$PATH` 决定。
- `base` 中的 Python 是 3.13.12，`deploy310` 中的 Python 是 3.10.20。
- `which python` 比单看终端提示符更可靠，它能直接告诉你当前 Python 的真实路径。
- Python 报错要从最后一行读起，例如 `NameError` 和 `AttributeError` 都能直接指出问题位置。
- 版本属性前后是两个下划线：`__version__`。

还不清楚的点：

- `pip install` 和 `conda install` 的选择规则还需要继续学习。
- 后续需要理解 PyTorch、ONNX、OpenCV 这些包是否应该固定版本。
- 还需要学习如何用 `requirements.txt` 固化项目依赖，保证环境可复现。

## 明日计划

- 开始 Python 基础复习：变量、列表、字典、函数、文件读写。
- 用 Python 写一个简单的图片信息读取脚本。
- 为后续 OpenCV 和模型推理代码做准备。
