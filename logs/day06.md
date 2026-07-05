# Day 06 学习记录

日期：2026-07-05

## 今日目标

- 继续使用 `deploy310` 环境，把 OpenCV 从“能读图”推进到“能画检测结果”。
- 理解 OpenCV 默认使用 BGR 通道，不是 RGB。
- 学会裁剪 ROI、画矩形框、写文字、保存结果图。
- 写出一个接近 YOLO 推理后处理风格的小脚本：读取图片，模拟检测框，画框和标签。
- 为后续 YOLO 检测结果可视化、摄像头实时显示打基础。

## 今日 7 小时学习计划

| 时间 | 内容 | 目标产出 |
| --- | --- | --- |
| 第 1 小时 | 复习 Day05，检查环境和图片目录 | 确认 `deploy310`、OpenCV、测试图片都可用 |
| 第 2 小时 | 学习 BGR/RGB 与颜色通道 | 生成红、绿、蓝色块图，理解 OpenCV 通道顺序 |
| 第 3 小时 | 图像裁剪和 ROI | 从图片中裁剪一块区域并保存 |
| 第 4 小时 | OpenCV 画框和写文字 | 在图片上画矩形框、类别名和置信度 |
| 第 5 小时 | 模拟 YOLO 检测结果 | 用列表/字典保存检测框，并批量画到图上 |
| 第 6 小时 | 整理成可复用脚本 | 写出 `draw_detections.py`，形成函数结构 |
| 第 7 小时 | 复盘和记录 | 整理命令、报错、产出图和明日计划 |

如果前面完成很快，可以直接进入挑战任务：批量给多张图片画框并保存到 `outputs/`。

## 开始前准备

进入 WSL2 Ubuntu 后执行：

```bash
conda activate deploy310
python --version
which python
mkdir -p ~/model-deploy-day06/images ~/model-deploy-day06/outputs
cd ~/model-deploy-day06
```

需要确认：

- 终端前缀是 `(deploy310)`。
- `python --version` 是 Python 3.10.x。
- `which python` 指向 `/home/nefelibata/miniconda3/envs/deploy310/bin/python`。

如果 Day05 的测试图片还在，可以复制一份过来：

```bash
cp ~/model-deploy-day05/images/test.jpg ~/model-deploy-day06/images/test.jpg
```

如果复制失败，就重新生成一张测试图片。

## 必做 1：检查 OpenCV 和测试图片

创建脚本：

```bash
nano check_image.py
```

写入：

```python
from pathlib import Path

import cv2

image_path = Path("images/test.jpg")
image = cv2.imread(str(image_path))

if image is None:
    raise FileNotFoundError(f"failed to read image: {image_path}")

height, width, channels = image.shape
print("image:", image_path)
print("height:", height)
print("width:", width)
print("channels:", channels)
```

运行：

```bash
python check_image.py
```

你需要看到图片的高、宽和通道数。

## 必做 2：理解 BGR 颜色通道

创建脚本：

```bash
nano color_blocks.py
```

写入：

```python
import cv2
import numpy as np

height, width = 200, 600
image = np.zeros((height, width, 3), dtype=np.uint8)

image[:, 0:200] = (255, 0, 0)     # Blue in BGR
image[:, 200:400] = (0, 255, 0)   # Green in BGR
image[:, 400:600] = (0, 0, 255)   # Red in BGR

cv2.imwrite("outputs/color_blocks.jpg", image)
print("saved: outputs/color_blocks.jpg")
```

运行：

```bash
python color_blocks.py
```

需要理解：

- OpenCV 的颜色顺序是 BGR。
- `(255, 0, 0)` 在 OpenCV 里是蓝色，不是红色。
- 后续模型输出图片可视化时，画框颜色也遵循 BGR。

## 必做 3：裁剪图片 ROI

创建脚本：

```bash
nano crop_roi.py
```

写入：

```python
from pathlib import Path

import cv2

image_path = Path("images/test.jpg")
image = cv2.imread(str(image_path))

if image is None:
    raise FileNotFoundError(f"failed to read image: {image_path}")

height, width, _ = image.shape

x1 = width // 4
y1 = height // 4
x2 = width * 3 // 4
y2 = height * 3 // 4

roi = image[y1:y2, x1:x2]
cv2.imwrite("outputs/test_roi.jpg", roi)

print("original:", image.shape)
print("roi:", roi.shape)
print("saved: outputs/test_roi.jpg")
```

运行：

```bash
python crop_roi.py
```

需要理解：

- OpenCV/NumPy 裁剪顺序是 `image[y1:y2, x1:x2]`。
- 坐标通常写成 `(x1, y1, x2, y2)`，但数组索引先写 y 再写 x。
- 这点后面处理 YOLO 检测框时非常重要。

## 必做 4：画矩形框和文字

创建脚本：

```bash
nano draw_box.py
```

写入：

```python
from pathlib import Path

import cv2

image_path = Path("images/test.jpg")
image = cv2.imread(str(image_path))

if image is None:
    raise FileNotFoundError(f"failed to read image: {image_path}")

x1, y1, x2, y2 = 160, 120, 480, 360
label = "object 0.95"

cv2.rectangle(image, (x1, y1), (x2, y2), (0, 255, 0), 2)
cv2.putText(
    image,
    label,
    (x1, y1 - 10),
    cv2.FONT_HERSHEY_SIMPLEX,
    0.8,
    (0, 255, 0),
    2,
)

cv2.imwrite("outputs/test_box.jpg", image)
print("saved: outputs/test_box.jpg")
```

运行：

```bash
python draw_box.py
```

需要理解：

- `cv2.rectangle` 用两个点确定矩形。
- `cv2.putText` 可以把类别名和置信度写到图上。
- 检测模型真正输出后，常见流程就是“模型输出框坐标 -> 在原图画框 -> 保存或显示”。

## 进阶：模拟 YOLO 检测结果

创建脚本：

```bash
nano draw_detections.py
```

写入：

```python
from pathlib import Path

import cv2


def draw_detection(image, detection):
    x1, y1, x2, y2 = detection["box"]
    label = detection["label"]
    score = detection["score"]
    color = detection["color"]

    text = f"{label} {score:.2f}"
    cv2.rectangle(image, (x1, y1), (x2, y2), color, 2)
    cv2.putText(
        image,
        text,
        (x1, max(y1 - 10, 20)),
        cv2.FONT_HERSHEY_SIMPLEX,
        0.7,
        color,
        2,
    )


image_path = Path("images/test.jpg")
image = cv2.imread(str(image_path))

if image is None:
    raise FileNotError(f"failed to read image: {image_path}")

detections = [
    {"label": "person", "score": 0.92, "box": (80, 80, 260, 360), "color": (0, 255, 0)},
    {"label": "car", "score": 0.86, "box": (300, 150, 560, 340), "color": (255, 0, 0)},
]

for detection in detections:
    draw_detection(image, detection)

cv2.imwrite("outputs/test_detections.jpg", image)
print("saved: outputs/test_detections.jpg")
```

运行：

```bash
python draw_detections.py
```

注意：上面代码里故意留了一个小错误，运行后你需要根据报错修正。

提示：

- 重点看报错最后几行。
- 错误和 `FileNotError` 有关。
- 修正后再运行一次。

## 挑战：批量画框

如果上面任务完成很快，继续做挑战：

- 把 Day05 的 `test.jpg`、`test_copy.jpg`、`test_gray.jpg` 复制到 Day06 的 `images/`。
- 写 `batch_draw.py`。
- 遍历 `images/*.jpg`。
- 给每张图片画同一个模拟检测框。
- 保存到 `outputs/`。

参考结构：

```python
from pathlib import Path

import cv2

input_dir = Path("images")
output_dir = Path("outputs")
output_dir.mkdir(exist_ok=True)

for image_path in input_dir.glob("*.jpg"):
    image = cv2.imread(str(image_path))
    if image is None:
        print("skip:", image_path)
        continue

    height, width = image.shape[:2]
    x1 = width // 4
    y1 = height // 4
    x2 = width * 3 // 4
    y2 = height * 3 // 4

    cv2.rectangle(image, (x1, y1), (x2, y2), (0, 255, 255), 2)
    cv2.putText(image, "demo 0.90", (x1, max(y1 - 10, 20)), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 255), 2)

    output_path = output_dir / image_path.name
    cv2.imwrite(str(output_path), image)
    print("saved:", output_path)
```

## YOLO 进阶加餐：预训练模型图片推理

完成 OpenCV 画框后，可以进入 YOLO 推理入门。今天只做“使用预训练模型推理”，不训练模型。

目标：

- 安装 Ultralytics YOLO。
- 用命令行跑一张图片。
- 用 Python 脚本跑一张图片。
- 理解模型输出里的 `boxes`、类别、置信度和坐标。
- 把今天 OpenCV 学到的画框知识和 YOLO 输出联系起来。

### 1. 安装 YOLO

在 WSL2 Ubuntu 中确认当前环境：

```bash
conda activate deploy310
python --version
which python
cd ~/model-deploy-day06
```

WSL/服务器环境优先安装 headless 版本：

```bash
python -m pip install ultralytics-opencv-headless
```

安装完成后检查：

```bash
yolo version
python -c "import ultralytics; print(ultralytics.__version__)"
```

### 2. 准备测试图片

如果还没有真实图片，先用 Day05 的测试图：

```bash
cp ~/model-deploy-day05/images/test.jpg ~/model-deploy-day06/images/test.jpg
```

也可以后续换成真实街景、人物、车辆图片，YOLO 的效果会更明显。

### 3. 命令行推理

先用官方预训练小模型跑图片：

```bash
yolo predict model=yolo26n.pt source=images/test.jpg imgsz=640 conf=0.25
```

注意：

- Ultralytics 命令参数写法是 `key=value`，不要写成 `--model`。
- 第一次运行会自动下载模型权重。
- 结果通常保存在 `runs/detect/predict/` 目录。

查看输出目录：

```bash
find runs -maxdepth 3 -type f
```

### 4. Python 推理脚本

创建脚本：

```bash
nano yolo_predict.py
```

写入：

```python
from pathlib import Path

from ultralytics import YOLO

model = YOLO("yolo26n.pt")
image_path = Path("images/test.jpg")

results = model.predict(source=str(image_path), imgsz=640, conf=0.25)

for result in results:
    print("image:", result.path)
    print("boxes:", len(result.boxes))

    for box in result.boxes:
        cls_id = int(box.cls[0])
        conf = float(box.conf[0])
        xyxy = box.xyxy[0].tolist()
        name = result.names[cls_id]
        print(name, f"{conf:.2f}", [round(v, 1) for v in xyxy])

    result.save(filename="outputs/yolo_predict.jpg")
    print("saved: outputs/yolo_predict.jpg")
```

运行：

```bash
python yolo_predict.py
```

需要理解：

- `result.boxes` 是检测框集合。
- `box.xyxy` 是 `(x1, y1, x2, y2)` 格式。
- `box.conf` 是置信度。
- `box.cls` 是类别编号。
- `result.names[cls_id]` 可以把类别编号转换成类别名。

### 5. 如果没有检测结果

如果输出 `boxes: 0`，不一定是你错了，可能是测试图太简单。换一张真实图片再跑：

```bash
mkdir -p images/real
```

把真实图片放进 `images/real/`，然后运行：

```bash
yolo predict model=yolo26n.pt source=images/real imgsz=640 conf=0.25
```

## 今日完成情况

- 已完成 OpenCV 基础可视化练习：
  - 检查图片读取和尺寸信息。
  - 生成 BGR 颜色块图片。
  - 裁剪 ROI。
  - 使用 `cv2.rectangle` 画矩形框。
  - 使用 `cv2.putText` 写类别和置信度。
  - 使用字典模拟 YOLO 检测结果，并批量画框保存。
- 已安装 Ultralytics YOLO：

```text
ultralytics: 8.4.87
```

- 已完成命令行 YOLO 推理：

```bash
yolo predict model=yolo11n.pt source=images/real imgsz=640 conf=0.25
```

关键输出：

```text
Ultralytics 8.4.87
Python-3.10.20
torch-2.12.1+cu130
CUDA:0 (NVIDIA GeForce RTX 4070 Laptop GPU, 8188MiB)

bus.jpg: 640x480 4 persons, 1 bus, 68.5ms
zidane.jpg: 384x640 2 persons, 1 tie, 58.4ms
Results saved to /home/nefelibata/model-deploy-day06/runs/detect/predict
```

- 已完成 Python 单图 YOLO 推理脚本 `yolo_predict.py`，读取并打印检测框：

```text
image: /home/nefelibata/model-deploy-day06/images/real/bus.jpg
boxes: 5
bus 0.94 [3.8, 229.4, 796.2, 728.4]
person 0.89 [671.0, 394.8, 809.8, 878.7]
person 0.88 [47.4, 399.6, 239.3, 904.2]
person 0.86 [223.1, 408.7, 344.5, 860.4]
person 0.62 [0.0, 556.1, 68.8, 872.4]
saved: outputs/yolo_predict_bus.jpg
```

- 已完成 Python 批量 YOLO 推理脚本 `yolo_batch_predict.py`：

```text
image count: 2

image: images/real/bus.jpg
boxes: 5
bus 0.94 [3.8, 229.4, 796.2, 728.4]
person 0.89 [671.0, 394.8, 809.8, 878.7]
person 0.88 [47.4, 399.6, 239.3, 904.2]
person 0.86 [223.1, 408.7, 344.5, 860.4]
person 0.62 [0.0, 556.1, 68.8, 872.4]
saved: outputs/yolo_batch/bus.jpg

image: images/real/zidane.jpg
boxes: 3
person 0.84 [748.5, 41.8, 1148.1, 711.1]
person 0.78 [148.5, 203.1, 1125.4, 715.0]
tie 0.45 [361.4, 437.7, 524.7, 717.3]
saved: outputs/yolo_batch/zidane.jpg
```

## 遇到的问题

- VS Code 中编辑脚本后没有保存，导致 `color_blocks.py` 实际是空文件，运行后没有生成图片。解决方式：运行前 `Ctrl + S` 保存，并确认终端运行的是同一个目录下的文件。
- `draw_box.py` 中把 `cv2.rectangle` 拼成了 `cv2.retangle`，触发 `AttributeError`。解决方式：根据报错检查函数名拼写。
- `cv2.putText` 的文字坐标写成了 `(x1, y1, -10)`，触发 `Expected sequence length 2, got 3`。解决方式：坐标必须是 `(x, y)`，应写成 `(x1, y1 - 10)`。
- WSL 中运行 `code .` 出现 `Code.exe: Exec format error`，说明 WSL 直接启动 Windows 版 VS Code 的互通有问题。解决方式：从 Windows VS Code 连接 WSL 并打开 `/home/nefelibata/model-deploy-day06`。
- 安装 YOLO 时 VS Code 弹出“是否创建虚拟环境”，因为已经使用 Conda 的 `deploy310`，不需要再创建新的虚拟环境。

## 今日复盘

今天最重要的收获：

- OpenCV 默认颜色顺序是 BGR，不是 RGB。
- OpenCV 画框需要 `(x1, y1)` 和 `(x2, y2)` 两个点，文字位置也是 `(x, y)` 二维坐标。
- YOLO 检测结果中的 `box.xyxy` 正好就是 `(x1, y1, x2, y2)`，可以直接对接 OpenCV 画框。
- `box.conf` 是置信度，`box.cls` 是类别编号，`result.names[cls_id]` 可以得到类别名。
- 命令行推理适合快速验证模型是否能跑；Python 推理适合后续写项目代码和部署脚本。
- 本机 WSL2 能调用 RTX 4070 跑 YOLO，输出中出现 `CUDA:0`，说明 GPU 推理已经打通。

还不清楚的点：

- YOLO 推理前的 resize、letterbox、归一化细节还需要学习。
- `preprocess`、`inference`、`postprocess` 三段耗时分别代表什么，还需要结合部署优化继续理解。
- 目前使用的是 `.pt` 模型，后续需要学习导出 ONNX，再进一步进入 TensorRT。

## 明日计划

- 继续 OpenCV 图像预处理。
- 学习 resize、归一化、通道转换、`NCHW` 和 `NHWC`。
- 为后续 ONNX / YOLO 输入预处理做准备。
