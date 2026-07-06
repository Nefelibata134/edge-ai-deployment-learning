# Day 09 学习记录

日期：2026-07-06

## 今日目标

- 使用 VS Code 继续开发 ONNX 推理脚本。
- 理解 ONNX Runtime 的基本使用流程。
- 不依赖 Ultralytics `predict`，直接加载 `yolo11n.onnx` 跑推理。
- 复用 Day07 的预处理逻辑，生成 `(1, 3, 640, 640)` 输入张量。
- 得到 ONNX 原始输出 `(1, 84, 8400)`。
- 初步理解 YOLO 原始输出如何变成检测框。

## 今日 7 小时学习计划

| 时间 | 内容 | 目标产出 |
| --- | --- | --- |
| 第 1 小时 | 复习 Day08 ONNX 输入输出 | 记住输入 `images`，输出 `output0` |
| 第 2 小时 | ONNX Runtime 基础 | 写出 `ort_check.py`，查看 providers 和输入输出 |
| 第 3 小时 | 复用预处理 | 写出 `preprocess.py`，输出 `(1, 3, 640, 640)` |
| 第 4 小时 | ONNX 单图推理 | 写出 `onnx_infer.py`，得到 `(1, 84, 8400)` |
| 第 5 小时 | 输出张量初步解析 | 打印最高置信度候选框和类别 |
| 第 6 小时 | 对比 Ultralytics 结果 | 理解 ONNX 输出还没经过完整后处理 |
| 第 7 小时 | 复盘记录 | 总结 ONNX Runtime 推理链路 |

今天先不要求完整复刻 Ultralytics 的最终检测框，重点是跑通 ONNX Runtime 和理解原始输出。

## 开始前准备

如果 Netron 还在运行，可以在启动 Netron 的终端按：

```text
Ctrl + C
```

停止它。浏览器里的 `http://localhost:8081` 页面可以先关掉。

在 VS Code / WSL 终端执行：

```bash
conda activate deploy310
mkdir -p ~/model-deploy-day09/models ~/model-deploy-day09/images ~/model-deploy-day09/scripts ~/model-deploy-day09/outputs
cd ~/model-deploy-day09
cp ~/model-deploy-day08/models/yolo11n.onnx models/yolo11n.onnx
cp ~/model-deploy-day08/models/yolo11n.pt models/yolo11n.pt
cp ~/model-deploy-day07/images/bus.jpg images/bus.jpg
python --version
which python
python -c "import onnxruntime as ort; print(ort.__version__)"
```

需要看到：

```text
Python 3.10.x
/home/nefelibata/miniconda3/envs/deploy310/bin/python
1.23.2
```

检查目录：

```bash
find . -maxdepth 2 -type f | sort
```

## 必做 1：查看 ONNX Runtime Provider

在 VS Code 中新建：

```text
scripts/ort_check.py
```

写入：

```python
import onnxruntime as ort

print("onnxruntime:", ort.__version__)
print("available providers:")
for provider in ort.get_available_providers():
    print("-", provider)
```

运行：

```bash
python scripts/ort_check.py
```

需要理解：

- `CPUExecutionProvider` 表示 ONNX Runtime 可以用 CPU 跑。
- 如果只有 CPU provider，不代表失败。今天重点是跑通 ONNX Runtime。
- 后面 TensorRT/Jetson 会再关注 GPU/TensorRT provider。

## 必做 2：查看 ONNX 模型输入输出

在 VS Code 中新建：

```text
scripts/ort_model_info.py
```

写入：

```python
from pathlib import Path

import onnxruntime as ort

model_path = Path("models/yolo11n.onnx")
session = ort.InferenceSession(str(model_path), providers=["CPUExecutionProvider"])

print("model:", model_path)
print("providers:", session.get_providers())

print("\ninputs:")
for item in session.get_inputs():
    print("name:", item.name)
    print("shape:", item.shape)
    print("type:", item.type)

print("\noutputs:")
for item in session.get_outputs():
    print("name:", item.name)
    print("shape:", item.shape)
    print("type:", item.type)
```

运行：

```bash
python scripts/ort_model_info.py
```

需要重点记录：

- 输入名是否是 `images`。
- 输出名是否是 `output0`。
- 输入 shape 是否能接受 `(1, 3, 640, 640)`。

## 必做 3：写预处理函数

在 VS Code 中新建：

```text
scripts/preprocess.py
```

写入：

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
    image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
    image = image.astype(np.float32) / 255.0
    image = np.transpose(image, (2, 0, 1))
    image = np.expand_dims(image, axis=0)

    return image, original_shape, scale, pad


if __name__ == "__main__":
    tensor, original_shape, scale, pad = preprocess(Path("images/bus.jpg"))
    print("tensor shape:", tensor.shape)
    print("tensor dtype:", tensor.dtype)
    print("original shape:", original_shape)
    print("scale:", scale)
    print("pad:", pad)
```

运行：

```bash
python scripts/preprocess.py
```

预期看到：

```text
tensor shape: (1, 3, 640, 640)
tensor dtype: float32
```

## 必做 4：ONNX Runtime 单图推理

在 VS Code 中新建：

```text
scripts/onnx_infer.py
```

写入：

```python
from pathlib import Path

import numpy as np
import onnxruntime as ort

from preprocess import preprocess


model_path = Path("models/yolo11n.onnx")
image_path = Path("images/bus.jpg")

session = ort.InferenceSession(str(model_path), providers=["CPUExecutionProvider"])
input_name = session.get_inputs()[0].name
output_name = session.get_outputs()[0].name

tensor, original_shape, scale, pad = preprocess(image_path)

outputs = session.run([output_name], {input_name: tensor})
output = outputs[0]

print("input name:", input_name)
print("output name:", output_name)
print("input shape:", tensor.shape)
print("output shape:", output.shape)
print("output dtype:", output.dtype)
print("output min/max:", float(output.min()), float(output.max()))
print("original shape:", original_shape)
print("scale:", scale)
print("pad:", pad)

np.save("outputs/onnx_output.npy", output)
print("saved: outputs/onnx_output.npy")
```

运行：

```bash
python scripts/onnx_infer.py
```

预期看到：

```text
input name: images
output name: output0
input shape: (1, 3, 640, 640)
output shape: (1, 84, 8400)
```

这一步完成，说明 ONNX Runtime 推理已经跑通。

## 进阶：初步解析最高置信度候选框

在 VS Code 中新建：

```text
scripts/inspect_output.py
```

写入：

```python
from pathlib import Path

import numpy as np


output_path = Path("outputs/onnx_output.npy")
output = np.load(output_path)

print("raw output shape:", output.shape)

pred = output[0]              # (84, 8400)
pred = np.transpose(pred)     # (8400, 84)

boxes = pred[:, :4]
class_scores = pred[:, 4:]

best_class_ids = np.argmax(class_scores, axis=1)
best_scores = np.max(class_scores, axis=1)

top_indices = np.argsort(best_scores)[-10:][::-1]

print("top candidates:")
for idx in top_indices:
    box = boxes[idx]
    score = best_scores[idx]
    cls_id = best_class_ids[idx]
    print(
        "idx=", int(idx),
        "cls=", int(cls_id),
        "score=", round(float(score), 4),
        "box_xywh=", [round(float(v), 1) for v in box],
    )
```

运行：

```bash
python scripts/inspect_output.py
```

需要理解：

- 这还不是最终检测结果。
- 这里没有做置信度阈值筛选。
- 这里没有做 NMS。
- `box_xywh` 也还没有映射回原图。
- 但你已经能看到 ONNX 原始输出里确实有候选框和类别分数。

## 挑战：对比 Ultralytics 推理

运行：

```bash
yolo predict model=models/yolo11n.onnx source=images/bus.jpg imgsz=640 conf=0.25
```

观察：

- Ultralytics 可以直接用 ONNX 做预测。
- 它内部帮你做了预处理、后处理、NMS 和画框。
- 我们今天自己写的 ONNX Runtime 脚本只做到“原始推理输出”，后处理明天继续。

## 今日完成情况

- 已创建 Day09 工作目录：

```text
~/model-deploy-day09
├── images
├── models
├── outputs
└── scripts
```

- 已准备文件：

```text
images/bus.jpg
models/yolo11n.onnx
models/yolo11n.pt
```

- 已确认 ONNX Runtime 版本和可用 provider：

```text
onnxruntime: 1.23.2
available providers:
- AzureExecutionProvider
- CPUExecutionProvider
```

- 已使用 `ort_model_info.py` 查看 ONNX 模型输入输出：

```text
model: models/yolo11n.onnx
providers: ['CPUExecutionProvider']

inputs:
name: images
shape: ['batch', 3, 'height', 'width']
type: tensor(float)

outputs:
name: output0
shape: ['batch', 84, 'anchors']
type: tensor(float)
```

- 已复用 Day07 预处理逻辑，得到模型输入张量：

```text
tensor shape: (1, 3, 640, 640)
tensor dtype: float32
original shape: (1080, 810)
scale: 0.5925925925925926
pad: (80, 0)
```

- 已完成 ONNX Runtime 单图推理：

```text
input name: images
output name: output0
input shape: (1, 3, 640, 640)
output shape: (1, 84, 8400)
output dtype: float32
output min/max: 0.0 637.1781005859375
original shape: (1080, 810)
scale: 0.5925925925925926
pad: (80, 0)
saved: outputs/onnx_output.npy
```

- 已初步解析 ONNX 原始输出中的最高分候选框：

```text
raw output shape: (1, 84, 8400)
top candidates:
idx= 8169 cls= 5 score= 0.9392 box_xywh= [320.3, 285.5, 466.6, 300.3]
idx= 8129 cls= 5 score= 0.9371 box_xywh= [323.7, 287.4, 462.6, 300.2]
idx= 8130 cls= 5 score= 0.9311 box_xywh= [323.5, 286.9, 464.3, 300.5]
idx= 8170 cls= 5 score= 0.9223 box_xywh= [322.7, 285.8, 469.5, 301.0]
idx= 8131 cls= 5 score= 0.9204 box_xywh= [324.4, 286.1, 462.1, 299.4]
idx= 8265 cls= 0 score= 0.902 box_xywh= [166.5, 385.9, 115.3, 300.2]
idx= 8150 cls= 5 score= 0.8991 box_xywh= [323.2, 286.1, 467.9, 299.8]
idx= 8149 cls= 5 score= 0.8931 box_xywh= [323.0, 286.1, 462.1, 298.7]
idx= 8264 cls= 0 score= 0.8857 box_xywh= [166.2, 385.6, 115.3, 299.7]
idx= 8244 cls= 0 score= 0.8817 box_xywh= [165.7, 385.4, 115.4, 301.4]
```

- 已使用 Ultralytics 加载 ONNX 进行对比推理：

```bash
yolo predict model=models/yolo11n.onnx source=images/bus.jpg imgsz=640 conf=0.25
```

关键输出：

```text
Loading models/yolo11n.onnx for ONNX Runtime inference...
CUDA requested but CUDAExecutionProvider not available. Using CPU...
Using ONNX Runtime 1.23.2 with CPUExecutionProvider

image 1/1 /home/nefelibata/model-deploy-day09/images/bus.jpg: 640x480 4 persons, 1 bus, 35.3ms
Results saved to /home/nefelibata/model-deploy-day09/runs/detect/predict
```

- 今日产出文件：

```text
scripts/ort_check.py
scripts/ort_model_info.py
scripts/preprocess.py
scripts/onnx_infer.py
scripts/inspect_output.py
outputs/onnx_output.npy
runs/detect/predict/bus.jpg
```

## 遇到的问题

- 当前安装的是 CPU 版 ONNX Runtime，因此可用 provider 只有 `AzureExecutionProvider` 和 `CPUExecutionProvider`，没有 `CUDAExecutionProvider`。
- Ultralytics 用 ONNX 推理时提示缺少 `onnxruntime-gpu`，并回退到 CPU：

```text
CUDA requested but CUDAExecutionProvider not available. Using CPU...
```

这不是错误。今天重点是 ONNX Runtime 推理流程，GPU 加速后面通过 `onnxruntime-gpu` 或 TensorRT 专门处理。

- ONNX 模型在 ORT 中显示动态维度：

```text
['batch', 3, 'height', 'width']
```

实际输入 `(1, 3, 640, 640)` 能正常运行，说明动态维度可以接收固定大小输入。

- 原始输出中同一个目标会出现多个高分候选框，例如 bus 出现了多个相近候选框。后续需要 NMS 去重。

## 今日复盘

今天最重要的收获：

- 已经不依赖 Ultralytics `predict`，成功用 ONNX Runtime 加载并运行 `yolo11n.onnx`。
- ONNX Runtime 的核心流程是：创建 `InferenceSession`，读取输入输出名，准备输入张量，调用 `session.run()`。
- Day07 的预处理函数可以直接服务 ONNX Runtime 推理，说明预处理链路是正确的。
- ONNX 原始输出 `(1, 84, 8400)` 里包含候选框坐标和类别分数，但还不是最终检测结果。
- `cls=5` 对应 bus，`cls=0` 对应 person，和 Ultralytics 的 `4 persons, 1 bus` 能对上。
- ONNX Runtime CPU 推理已经跑通，后续可以继续做后处理，再进入 GPU/TensorRT 加速。

还不清楚的点：

- `box_xywh` 如何转换成 `xyxy`。
- letterbox 后的坐标如何映射回原图。
- 多个重复候选框如何通过 NMS 合并成最终检测结果。
- ONNX Runtime GPU 和 TensorRT 的性能差异还没有开始比较。

## 明日计划

- Day10：YOLO ONNX 后处理。
- 将 `output0` 解析为检测框。
- 实现置信度过滤、坐标转换和 NMS。
- 用 OpenCV 把 ONNX Runtime 的检测结果画回原图。
