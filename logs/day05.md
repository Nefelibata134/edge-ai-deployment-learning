# Day 05 学习记录

日期：2026-07-04

## 今日目标

- 在 `deploy310` 环境中复习 Python 基础语法。
- 理解变量、列表、字典、函数、文件读写这些后续推理脚本常用内容。
- 用 OpenCV 读取图片、查看尺寸、保存处理后的图片。
- 写出第一个接近模型部署项目风格的小脚本：批量读取图片并输出信息。
- 形成“必做 + 进阶 + 挑战”的学习节奏，做得快就继续加速。

## 今日任务结构

今天不再硬拆成固定 7 个一小时段，而是按三层推进：

- 必做：完成 Python 基础与单张图片读取。
- 进阶：完成多张图片批量读取。
- 挑战：完成图片 resize 后保存到新目录。

如果必做很快完成，直接进入进阶和挑战。

## 开始前准备

进入 WSL2 Ubuntu 后执行：

```bash
conda activate deploy310
python --version
which python
mkdir -p ~/model-deploy-day05/images
cd ~/model-deploy-day05
```

需要确认：

- 终端前缀是 `(deploy310)`。
- `python --version` 是 Python 3.10.x。
- `which python` 指向 `/home/nefelibata/miniconda3/envs/deploy310/bin/python`。

## 必做 1：Python 基础复习

创建脚本：

```bash
nano python_basic.py
```

写入：

```python
name = "edge-ai"
day = 5
topics = ["python", "opencv", "deployment"]
config = {
    "env": "deploy310",
    "python": "3.10",
    "device": "wsl2"
}

def print_plan(name, day, topics):
    print("project:", name)
    print("day:", day)
    print("topics:")
    for topic in topics:
        print("-", topic)

print_plan(name, day, topics)
print("config:", config)
print("env:", config["env"])
```

运行：

```bash
python python_basic.py
```

需要理解：

- 列表 `topics` 适合保存一组同类内容。
- 字典 `config` 适合保存有名字的配置项。
- 函数 `print_plan` 把一组动作封装起来，后面推理脚本会大量用函数。

## 必做 2：文件读写

创建脚本：

```bash
nano file_io.py
```

写入：

```python
lines = [
    "Day 05 Python practice",
    "Python scripts are used for model inference",
    "OpenCV is used for image reading"
]

with open("day05_notes.txt", "w", encoding="utf-8") as f:
    for line in lines:
        f.write(line + "\n")

with open("day05_notes.txt", "r", encoding="utf-8") as f:
    content = f.read()

print(content)
```

运行：

```bash
python file_io.py
cat day05_notes.txt
```

需要理解：

- `with open(...)` 是 Python 里安全读写文件的常见写法。
- `"w"` 是写入，会覆盖旧内容。
- `"r"` 是读取。
- `encoding="utf-8"` 可以减少中文乱码问题。

## 必做 3：准备一张测试图片

今天先用 Python 生成一张测试图，避免到处找图片。

创建脚本：

```bash
nano create_test_image.py
```

写入：

```python
import cv2
import numpy as np

image = np.zeros((480, 640, 3), dtype=np.uint8)
image[:] = (40, 80, 120)

cv2.rectangle(image, (160, 120), (480, 360), (0, 255, 0), 3)
cv2.putText(image, "Day05 OpenCV", (170, 250), cv2.FONT_HERSHEY_SIMPLEX, 1.2, (255, 255, 255), 2)

cv2.imwrite("images/test.jpg", image)
print("saved: images/test.jpg")
```

运行：

```bash
python create_test_image.py
ls images
```

需要理解：

- OpenCV 默认颜色顺序是 BGR，不是 RGB。
- `np.zeros((480, 640, 3))` 表示高 480、宽 640、3 通道彩色图片。
- `cv2.imwrite` 保存图片。

## 必做 4：读取单张图片并输出信息

创建脚本：

```bash
nano read_image.py
```

写入：

```python
import cv2

image_path = "images/test.jpg"
image = cv2.imread(image_path)

if image is None:
    raise FileNotFoundError(f"failed to read image: {image_path}")

height, width, channels = image.shape

print("image:", image_path)
print("height:", height)
print("width:", width)
print("channels:", channels)

gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
cv2.imwrite("images/test_gray.jpg", gray)
print("saved: images/test_gray.jpg")
```

运行：

```bash
python read_image.py
ls images
```

需要理解：

- `cv2.imread` 读取失败时会返回 `None`。
- `image.shape` 对彩色图通常是 `(height, width, channels)`。
- YOLO 推理前也要读取图片、检查尺寸、做预处理。

## 进阶：批量读取图片

创建第二张测试图：

```bash
cp images/test.jpg images/test_copy.jpg
```

创建脚本：

```bash
nano batch_image_info.py
```

写入：

```python
from pathlib import Path
import cv2

image_dir = Path("images")
image_paths = sorted(list(image_dir.glob("*.jpg")) + list(image_dir.glob("*.png")))

print("image count:", len(image_paths))

for image_path in image_paths:
    image = cv2.imread(str(image_path))
    if image is None:
        print("failed:", image_path)
        continue

    height, width, channels = image.shape
    print(f"{image_path.name}: {width}x{height}, channels={channels}")
```

运行：

```bash
python batch_image_info.py
```

需要理解：

- `pathlib.Path` 比手写字符串路径更适合管理文件路径。
- 批量推理图片时，也会先收集图片路径，再逐张读取。

## 挑战：批量 resize 并保存

创建脚本：

```bash
nano resize_images.py
```

写入：

```python
from pathlib import Path
import cv2

input_dir = Path("images")
output_dir = Path("resized")
output_dir.mkdir(exist_ok=True)

for image_path in sorted(input_dir.glob("*.jpg")):
    image = cv2.imread(str(image_path))
    if image is None:
        print("failed:", image_path)
        continue

    resized = cv2.resize(image, (320, 240))
    output_path = output_dir / image_path.name
    cv2.imwrite(str(output_path), resized)
    print("saved:", output_path)
```

运行：

```bash
python resize_images.py
tree .
```

需要理解：

- 模型推理前常常需要 resize 到固定输入尺寸。
- 今天先练普通 resize，后面 YOLO 会讲 letterbox。

## 今日需要发给 Codex 的输出

把以下输出发给 Codex：

```bash
python --version
which python
python python_basic.py
python file_io.py
python create_test_image.py
python read_image.py
python batch_image_info.py
python resize_images.py
tree ~/model-deploy-day05
```

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

- 继续 OpenCV：颜色通道、裁剪、画框、文字标注。
- 写一个“模拟检测结果可视化”脚本，为 YOLO 画框做准备。
