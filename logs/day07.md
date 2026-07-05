# Day 07 学习记录

日期：2026-07-05

## 今日目标

- 使用 VS Code 作为主要编辑器，终端只负责运行命令。
- 理解 YOLO 推理前为什么要做图像预处理。
- 掌握 `resize`、`letterbox`、BGR/RGB、归一化、`HWC -> NCHW`。
- 写出一个接近 ONNX/TensorRT 输入格式的预处理脚本。
- 为 Day08 导出 ONNX 和 Day09 ONNX Runtime 推理打基础。

## 今日 7 小时学习计划

| 时间 | 内容 | 目标产出 |
| --- | --- | --- |
| 第 1 小时 | 复习 Day06 YOLO 输出 | 理解 `xyxy`、`conf`、`cls` 和原图尺寸关系 |
| 第 2 小时 | 普通 resize | 写出 `resize_demo.py`，观察图片被拉伸的问题 |
| 第 3 小时 | letterbox 原理 | 理解等比例缩放和 padding，写出 `letterbox_demo.py` |
| 第 4 小时 | BGR/RGB 和归一化 | 写出 `normalize_demo.py`，理解像素从 `0-255` 到 `0-1` |
| 第 5 小时 | HWC/NCHW | 写出 `tensor_shape_demo.py`，得到 `(1, 3, 640, 640)` |
| 第 6 小时 | 封装 YOLO 预处理函数 | 写出 `preprocess_yolo.py`，输出模型输入张量 |
| 第 7 小时 | 复盘和记录 | 整理命令、输出、问题，为 ONNX 做准备 |

如果今天完成很快，可以加做：把预处理后的张量保存为 `.npy`，后面 ONNX Runtime 直接读取做对比。

## VS Code 工作流

从 Day07 开始，默认使用 VS Code，不再用 `nano` 写主要脚本。

推荐打开方式：

1. Windows 打开 VS Code。
2. 左下角选择连接 WSL。
3. 打开文件夹：

```text
/home/nefelibata/model-deploy-day07
```

VS Code 左下角需要显示类似：

```text
WSL: Ubuntu-22.04
```

终端里执行：

```bash
conda activate deploy310
cd ~/model-deploy-day07
```

## 开始前准备

在 WSL2 Ubuntu 终端执行：

```bash
conda activate deploy310
mkdir -p ~/model-deploy-day07/images ~/model-deploy-day07/outputs
cd ~/model-deploy-day07
cp ~/model-deploy-day06/images/real/bus.jpg images/bus.jpg
cp ~/model-deploy-day06/images/real/zidane.jpg images/zidane.jpg
python --version
which python
```

需要确认：

- 终端前缀是 `(deploy310)`。
- `python --version` 是 Python 3.10.x。
- `which python` 指向 `/home/nefelibata/miniconda3/envs/deploy310/bin/python`。
- `images/` 里有 `bus.jpg` 和 `zidane.jpg`。

检查图片：

```bash
ls images
```

## 必做 1：普通 resize

在 VS Code 中新建 `resize_demo.py`：

```python
from pathlib import Path

import cv2

image_path = Path("images/bus.jpg")
image = cv2.imread(str(image_path))

if image is None:
    raise FileNotFoundError(f"failed to read image: {image_path}")

resized = cv2.resize(image, (640, 640))

print("original shape:", image.shape)
print("resized shape:", resized.shape)

cv2.imwrite("outputs/bus_resize_640.jpg", resized)
print("saved: outputs/bus_resize_640.jpg")
```

运行：

```bash
python resize_demo.py
```

需要理解：

- `cv2.resize(image, (640, 640))` 的参数顺序是 `(width, height)`。
- 普通 resize 会强行拉伸图片，可能改变目标形状。
- 目标检测通常不希望直接粗暴拉伸，所以 YOLO 常用 letterbox。

## 必做 2：letterbox 等比例缩放

在 VS Code 中新建 `letterbox_demo.py`：

```python
from pathlib import Path

import cv2
import numpy as np


def letterbox(image, new_shape=(640, 640), color=(114, 114, 114)):
    height, width = image.shape[:2]
    new_height, new_width = new_shape

    scale = min(new_width / width, new_height / height)
    resized_width = int(round(width * scale))
    resized_height = int(round(height * scale))

    pad_width = new_width - resized_width
    pad_height = new_height - resized_height
    pad_left = pad_width // 2
    pad_right = pad_width - pad_left
    pad_top = pad_height // 2
    pad_bottom = pad_height - pad_top

    resized = cv2.resize(image, (resized_width, resized_height))
    padded = cv2.copyMakeBorder(
        resized,
        pad_top,
        pad_bottom,
        pad_left,
        pad_right,
        cv2.BORDER_CONSTANT,
        value=color,
    )

    return padded, scale, (pad_left, pad_top)


image_path = Path("images/bus.jpg")
image = cv2.imread(str(image_path))

if image is None:
    raise FileNotFoundError(f"failed to read image: {image_path}")

letterboxed, scale, pad = letterbox(image)

print("original shape:", image.shape)
print("letterbox shape:", letterboxed.shape)
print("scale:", scale)
print("pad left/top:", pad)

cv2.imwrite("outputs/bus_letterbox_640.jpg", letterboxed)
print("saved: outputs/bus_letterbox_640.jpg")
```

运行：

```bash
python letterbox_demo.py
```

需要理解：

- letterbox = 等比例缩放 + 灰边填充。
- 它能保持目标不变形。
- 后处理时如果要把检测框映射回原图，需要用到 `scale` 和 `pad`。

## 必做 3：BGR 转 RGB 与归一化

在 VS Code 中新建 `normalize_demo.py`：

```python
from pathlib import Path

import cv2

image_path = Path("images/bus.jpg")
image_bgr = cv2.imread(str(image_path))

if image_bgr is None:
    raise FileNotFoundError(f"failed to read image: {image_path}")

image_rgb = cv2.cvtColor(image_bgr, cv2.COLOR_BGR2RGB)
image_float = image_rgb.astype("float32") / 255.0

print("bgr shape:", image_bgr.shape, "dtype:", image_bgr.dtype)
print("rgb shape:", image_rgb.shape, "dtype:", image_rgb.dtype)
print("float shape:", image_float.shape, "dtype:", image_float.dtype)
print("min:", image_float.min())
print("max:", image_float.max())
```

运行：

```bash
python normalize_demo.py
```

需要理解：

- OpenCV 读图是 BGR。
- 很多模型训练时使用 RGB。
- 归一化后像素范围从 `0-255` 变成 `0-1`。
- 模型输入通常需要 `float32`。

## 必做 4：HWC 转 NCHW

在 VS Code 中新建 `tensor_shape_demo.py`：

```python
from pathlib import Path

import cv2
import numpy as np

image_path = Path("images/bus.jpg")
image = cv2.imread(str(image_path))

if image is None:
    raise FileNotFoundError(f"failed to read image: {image_path}")

image = cv2.resize(image, (640, 640))
image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
image = image.astype(np.float32) / 255.0

print("HWC:", image.shape)

image = np.transpose(image, (2, 0, 1))
print("CHW:", image.shape)

image = np.expand_dims(image, axis=0)
print("NCHW:", image.shape)
print("dtype:", image.dtype)
```

运行：

```bash
python tensor_shape_demo.py
```

需要理解：

- OpenCV/NumPy 图片默认是 `HWC`：高、宽、通道。
- PyTorch/ONNX/TensorRT 常见输入是 `NCHW`：批次、通道、高、宽。
- 单张图片也要加 batch 维度，所以是 `(1, 3, 640, 640)`。

## 进阶：封装 YOLO 预处理函数

在 VS Code 中新建 `preprocess_yolo.py`：

```python
from pathlib import Path

import cv2
import numpy as np


def letterbox(image, new_shape=(640, 640), color=(114, 114, 114)):
    height, width = image.shape[:2]
    new_height, new_width = new_shape

    scale = min(new_width / width, new_height / height)
    resized_width = int(round(width * scale))
    resized_height = int(round(height * scale))

    pad_width = new_width - resized_width
    pad_height = new_height - resized_height
    pad_left = pad_width // 2
    pad_right = pad_width - pad_left
    pad_top = pad_height // 2
    pad_bottom = pad_height - pad_top

    resized = cv2.resize(image, (resized_width, resized_height))
    padded = cv2.copyMakeBorder(
        resized,
        pad_top,
        pad_bottom,
        pad_left,
        pad_right,
        cv2.BORDER_CONSTANT,
        value=color,
    )

    return padded, scale, (pad_left, pad_top)


def preprocess(image_path, input_shape=(640, 640)):
    image_bgr = cv2.imread(str(image_path))

    if image_bgr is None:
        raise FileNotFoundError(f"failed to read image: {image_path}")

    original_shape = image_bgr.shape[:2]
    image, scale, pad = letterbox(image_bgr, new_shape=input_shape)
    cv2.imwrite("outputs/preprocessed_letterbox.jpg", image)

    image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
    image = image.astype(np.float32) / 255.0
    image = np.transpose(image, (2, 0, 1))
    image = np.expand_dims(image, axis=0)

    return image, original_shape, scale, pad


image_path = Path("images/bus.jpg")
tensor, original_shape, scale, pad = preprocess(image_path)

print("input:", image_path)
print("original shape:", original_shape)
print("tensor shape:", tensor.shape)
print("tensor dtype:", tensor.dtype)
print("tensor min/max:", tensor.min(), tensor.max())
print("scale:", scale)
print("pad:", pad)

np.save("outputs/bus_preprocessed.npy", tensor)
print("saved: outputs/bus_preprocessed.npy")
print("saved: outputs/preprocessed_letterbox.jpg")
```

运行：

```bash
python preprocess_yolo.py
```

预期输出重点：

```text
tensor shape: (1, 3, 640, 640)
tensor dtype: float32
```

这就是后续 ONNX Runtime / TensorRT 推理会接收的典型输入格式。

## 挑战：批量预处理图片

如果进阶完成很快，继续写 `batch_preprocess.py`：

- 遍历 `images/*.jpg`。
- 对每张图做 letterbox + RGB + 归一化 + NCHW。
- 每张图保存一个 `.npy`。
- 打印每张图的原始尺寸、scale、pad。

目标输出示例：

```text
images/bus.jpg -> outputs/bus.npy, original=(1080, 810), tensor=(1, 3, 640, 640)
images/zidane.jpg -> outputs/zidane.npy, original=(720, 1280), tensor=(1, 3, 640, 640)
```

## Jetson 到货检查预告

如果 Jetson Orin Nano 明天到货，先不要急着装复杂环境。优先做：

- 拍照记录外包装、主板、供电、SSD、配件。
- 确认是否为官方载板，是否有 SD 卡槽。
- 确认 256G SSD 是否已经安装。
- 第一次开机后记录 JetPack、Ubuntu、CUDA、TensorRT、Python 版本。
- 先跑基础检查，再考虑 YOLO。

## 今日完成情况

- 已切换到 VS Code 作为主要编辑器，并解决 Python 自动补全问题。
- 已确认 Day07 环境：

```text
Python 3.10.20
/home/nefelibata/miniconda3/envs/deploy310/bin/python
```

- 已准备测试图片：

```text
images/bus.jpg
images/zidane.jpg
```

- 已完成普通 resize，输出：

```text
original shape: (1080, 810, 3)
resized shape: (640, 640, 3)
saved: outputs/bus_resize_640.jpg
```

- 已完成 letterbox 等比例缩放和灰边填充，输出：

```text
original shape: (1080, 810, 3)
letterbox shape: (640, 640, 3)
scale: 0.5925925925925926
pad left/top: (80, 0)
saved: outputs/bus_letterbox_640.jpg
```

- 已完成 BGR 转 RGB 和归一化，输出：

```text
bgr shape: (1080, 810, 3) dtype: uint8
rgb shape: (1080, 810, 3) dtype: uint8
float shape: (1080, 810, 3) dtype: float32
min: 0.0
max: 1.0
```

- 已完成 `HWC -> CHW -> NCHW` 维度转换，输出：

```text
HWC: (640, 640, 3)
CHW: (3, 640, 640)
NCHW: (1, 3, 640, 640)
dtype: float32
```

- 已完成封装版 YOLO 预处理脚本 `preprocess_yolo.py`，输出：

```text
input: images/bus.jpg
original shape: (1080, 810)
tensor shape: (1, 3, 640, 640)
tensor dtype: float32
tensor min/max: 0.0 1.0
scale: 0.5925925925925926
pad: (80, 0)
saved: outputs/bus_preprocessed.npy
saved: outputs/preprocessed_letterbox.jpg
```

- 今日产出文件：

```text
resize_demo.py
letterbox_demo.py
normalize_demo.py
tensor_shape_demo.py
preprocess_yolo.py
outputs/bus_resize_640.jpg
outputs/bus_letterbox_640.jpg
outputs/preprocessed_letterbox.jpg
outputs/bus_preprocessed.npy
```

## 遇到的问题

- VS Code 自动补全消失。原因是没有选择当前项目使用的 Python 解释器；选择 `/home/nefelibata/miniconda3/envs/deploy310/bin/python` 后恢复。
- `cv2.resize(image, (640, 640))` 的参数顺序容易误解，它是 `(width, height)`，不是 `(height, width)`。
- 普通 resize 和 letterbox 都能得到 `(640, 640, 3)`，但普通 resize 会拉伸图像，letterbox 会保持比例并加灰边。

## 今日复盘

今天最重要的收获：

- YOLO/ONNX/TensorRT 推理前，不是直接把原图喂给模型，而是要先做预处理。
- 普通 resize 会改变图像比例，可能影响检测效果；letterbox 通过等比例缩放和 padding 保持目标形状。
- BGR/RGB 是通道顺序问题，OpenCV 读图默认 BGR，但很多模型训练和推理习惯使用 RGB。
- 归一化会把像素从 `0-255` 转成 `0-1`，并把数据类型变成 `float32`。
- `HWC -> NCHW` 是从图像格式转成模型输入格式的关键一步。
- `(1, 3, 640, 640)` 里的 `1` 是 batch 维度，后面的 `3, 640, 640` 是通道、高、宽。
- `scale` 和 `pad` 不只是中间值，后面把模型检测框映射回原图时会用到。

还不清楚的点：

- letterbox 后，如何把模型输出框从 640x640 坐标映射回原图坐标，还需要结合后处理继续练习。
- ONNX 模型的输入名、输出名、动态尺寸还没有开始学。
- TensorRT 的 engine、FP16、workspace、batch 等概念还没开始学。

## 明日计划

- 开始 Day08：YOLO `.pt -> .onnx` 导出。
- 使用 Netron 初步查看 ONNX 输入输出结构。
- 如果 Jetson 到货，穿插完成 Jetson 开箱和系统检查。
