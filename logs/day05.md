# Day 05 学习记录

日期：2026-07-05

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

- 本次 Day 05 从 2026-07-04 开始练习，2026-07-05 复跑脚本并收尾记录。
- 使用环境：

```text
Python 3.10.20
/home/nefelibata/miniconda3/envs/deploy310/bin/python
```

- 创建练习目录：

```bash
mkdir -p ~/model-deploy-day05/images
cd ~/model-deploy-day05
```

- `python_basic.py` 运行结果：

```text
project: edge-ai
day: 5
topics:
- python
- opencv
- deployment
config: {'env': 'deploy310', 'python': '3.10', 'device': 'wsl2'}
env deploy310
```

- `file_io.py` 第一次运行报错：

```text
SyntaxError: expected ':'
```

原因是 `with open(...) as f` 行末少了冒号。修正后运行成功：

```text
Day 05 Python practice
Python scripts are used for model inference
OpenCV is used for image reading
```

- `day05_notes.txt` 内容：

```text
Day 05 Python practice
Python scripts are used for model inference
OpenCV is used for image reading
```

- `create_test_image.py` 运行结果：

```text
saved: images/test.jpg
```

- `read_image.py` 第一次读取图片时出现：

```text
FileNotFoundError: failed to read image: images/test.jpg
```

后续检查 `ls images` 发现 `test.jpg` 存在，说明问题来自脚本中的路径或保存内容不一致。修正后运行成功：

```text
image images/test.jpg
height 480
width: 640
channels: 3
saved: images/test_gray.jpg
```

- `images` 目录内容：

```text
test_copy.jpg
test_gray.jpg
test.jpg
```

- `batch_image_info.py` 运行结果：

```text
image count: 3
test.jpg: 640x480, channels=3
test_copy.jpg: 640x480, channels=3
test_gray.jpg: 640x480, channels=3
```

- 挑战任务中发现只有 `resize_images.py.save`，没有正式保存为 `resize_images.py`。收尾时补成正式脚本并复跑成功：

```text
saved: resized/test.jpg
saved: resized/test_copy.jpg
saved: resized/test_gray.jpg
```

- 最终目录结构：

```text
.
├── batch_image_info.py
├── create_test_image.py
├── day05_notes.txt
├── file_io.py
├── images
│   ├── test_copy.jpg
│   ├── test_gray.jpg
│   └── test.jpg
├── python_basic.py
├── read_image.py
├── resized
│   ├── test_copy.jpg
│   ├── test_gray.jpg
│   └── test.jpg
├── resize_images.py
└── resize_images.py.save

2 directories, 14 files
```

## 今日完成情况

- 已在 `deploy310` 环境中完成 Day 05 所有必做任务。
- 已复习 Python 变量、列表、字典、函数。
- 已完成 Python 文件写入和读取练习。
- 已使用 OpenCV 和 NumPy 生成测试图片 `images/test.jpg`。
- 已使用 OpenCV 读取单张图片，输出图片尺寸和通道数。
- 已将彩色图片转换为灰度图并保存为 `images/test_gray.jpg`。
- 已完成批量读取图片，并输出每张图片的宽、高和通道数。
- 已完成批量 resize，将图片保存到 `resized/` 目录。
- 已初步理解模型推理脚本中的常见流程：收集路径、读取图片、检查尺寸、处理图片、保存结果。

## 遇到的问题

- `file_io.py` 中 `with open(...) as f` 行末少了冒号，导致 `SyntaxError: expected ':'`。
- `read_image.py` 第一次运行时 `cv2.imread` 没读到图片，触发 `FileNotFoundError`；检查文件存在后修正脚本路径/内容，成功读取。
- `read_image.py` 中 `print("image", image_path)` 少了冒号，只影响输出格式，不影响功能。
- `if image is  None:` 中间多了一个空格，不影响运行，但建议写成 `if image is None:`。
- `resize_images.py` 没有正式保存，只留下 `resize_images.py.save`；收尾时补成正式脚本后成功运行。

## 今日复盘

今天最重要的收获：

- `deploy310` 环境已经可以支撑基础 Python + OpenCV 练习。
- Python 脚本报错时，优先看报错最后几行和具体文件行号。
- `cv2.imread` 读取失败会返回 `None`，在图像脚本里必须检查。
- `image.shape` 是理解图像尺寸的第一步，彩色图通常是 `(height, width, channels)`。
- 批量处理图片的基本流程是：用 `Path` 收集文件路径，循环读取，处理后保存。
- resize 是模型推理前的常见预处理动作，后续 YOLO 会进一步学习 letterbox。

还不清楚的点：

- OpenCV 的 BGR 和 RGB 通道区别还需要继续练习。
- 灰度图、彩色图、通道数之间的关系还需要通过更多例子理解。
- YOLO 输入尺寸不是简单 resize，后续需要学习 letterbox 和归一化。

## 明日计划

- 继续 OpenCV：颜色通道、裁剪、画框、文字标注。
- 写一个“模拟检测结果可视化”脚本，为 YOLO 画框做准备。
