# Day 10 学习记录

日期：2026-07-08

## 今日目标

- 在 PC / WSL 环境中继续完成 YOLO ONNX 推理链路。
- 不依赖 Jetson，Jetson 不在身边不影响今天学习。
- 把 Day09 得到的 ONNX 原始输出 `(1, 84, 8400)` 解析成真实检测结果。
- 理解并实现 YOLO 后处理流程：置信度筛选、坐标转换、映射回原图、NMS、画框保存。
- 生成一张由自己代码完成后处理和画框的检测结果图。

## 今日 7 小时学习计划

| 时间 | 内容 | 目标产出 |
| --- | --- | --- |
| 第 1 小时 | 复习 Day09 ONNX 原始输出 | 理解 `84 = 4 个坐标 + 80 个类别分数`，`8400 = 候选框数量` |
| 第 2 小时 | 实现 `xywh -> xyxy` 坐标转换 | 写出坐标转换函数 |
| 第 3 小时 | 实现 letterbox 反变换 | 把 640 输入图上的坐标映射回原图 |
| 第 4 小时 | 实现置信度筛选和 NMS | 去掉低分框和重复框 |
| 第 5 小时 | OpenCV 画框和保存结果图 | 生成 `outputs/onnx_detect_bus.jpg` |
| 第 6 小时 | 对比 Ultralytics ONNX 推理结果 | 检查是否接近 `4 persons, 1 bus` |
| 第 7 小时 | 复盘和记录 | 更新日志，记录问题和明日计划 |

## 开始前准备

今天继续使用 Day09 的目录，不重新复制模型：

```bash
conda activate deploy310
cd ~/model-deploy-day09
code .
```

确认关键文件存在：

```bash
find . -maxdepth 2 -type f | sort
```

需要看到：

```text
images/bus.jpg
models/yolo11n.onnx
scripts/preprocess.py
scripts/onnx_infer.py
outputs/onnx_output.npy
```

## 必做 1：理解 YOLO ONNX 输出

Day09 的输出为：

```text
output shape: (1, 84, 8400)
```

含义：

- `1`：batch size。
- `84`：每个候选框的信息，前 4 个是坐标，后 80 个是 COCO 类别分数。
- `8400`：候选框数量。

后处理前需要先转换维度：

```python
pred = output[0]          # (84, 8400)
pred = np.transpose(pred) # (8400, 84)
```

## 必做 2：实现后处理函数

新建文件：

```text
scripts/postprocess.py
```

需要实现：

- `xywh_to_xyxy`
- 置信度筛选
- 坐标从 letterbox 输入图映射回原图
- OpenCV NMS

核心流程：

```text
raw output
-> transpose
-> split boxes and class scores
-> confidence filter
-> xywh to xyxy
-> remove padding and divide by scale
-> clip boxes to original image size
-> NMS
-> final boxes, scores, class_ids
```

写入代码：

```python
import cv2
import numpy as np


def xywh_to_xyxy(boxes):
    xyxy = np.zeros_like(boxes)
    xyxy[:, 0] = boxes[:, 0] - boxes[:, 2] / 2
    xyxy[:, 1] = boxes[:, 1] - boxes[:, 3] / 2
    xyxy[:, 2] = boxes[:, 0] + boxes[:, 2] / 2
    xyxy[:, 3] = boxes[:, 1] + boxes[:, 3] / 2
    return xyxy


def scale_boxes_to_original(boxes, original_shape, scale, pad):
    pad_left, pad_top = pad
    original_height, original_width = original_shape

    boxes[:, [0, 2]] -= pad_left
    boxes[:, [1, 3]] -= pad_top
    boxes[:, :4] /= scale

    boxes[:, [0, 2]] = np.clip(boxes[:, [0, 2]], 0, original_width)
    boxes[:, [1, 3]] = np.clip(boxes[:, [1, 3]], 0, original_height)
    return boxes


def postprocess(output, original_shape, scale, pad, conf_threshold=0.25, iou_threshold=0.45):
    pred = output[0]
    pred = np.transpose(pred)

    boxes_xywh = pred[:, :4]
    class_scores = pred[:, 4:]

    class_ids = np.argmax(class_scores, axis=1)
    scores = np.max(class_scores, axis=1)

    keep = scores >= conf_threshold
    boxes_xywh = boxes_xywh[keep]
    scores = scores[keep]
    class_ids = class_ids[keep]

    boxes_xyxy = xywh_to_xyxy(boxes_xywh)
    boxes_xyxy = scale_boxes_to_original(boxes_xyxy, original_shape, scale, pad)

    nms_boxes = []
    for box in boxes_xyxy:
        x1, y1, x2, y2 = box
        nms_boxes.append([float(x1), float(y1), float(x2 - x1), float(y2 - y1)])

    indices = cv2.dnn.NMSBoxes(
        bboxes=nms_boxes,
        scores=scores.tolist(),
        score_threshold=conf_threshold,
        nms_threshold=iou_threshold,
    )

    if len(indices) == 0:
        return np.empty((0, 4)), np.array([]), np.array([])

    indices = np.array(indices).reshape(-1)
    return boxes_xyxy[indices], scores[indices], class_ids[indices]
```

说明：

- ONNX 输出里的坐标是基于 `640 x 640` 输入图的。
- Day07/Day09 使用了 letterbox，所以映射回原图时必须先减掉 padding，再除以 scale。
- `cv2.dnn.NMSBoxes` 接收的框格式是 `[x, y, width, height]`，不是 `[x1, y1, x2, y2]`。

## 必做 3：完成 ONNX 检测脚本

新建文件：

```text
scripts/onnx_detect.py
```

目标：

- 加载 `models/yolo11n.onnx`
- 复用 `scripts/preprocess.py`
- 调用 `scripts/postprocess.py`
- 在原图上画框
- 保存结果到：

```text
outputs/onnx_detect_bus.jpg
```

写入代码：

```python
from pathlib import Path

import cv2
import onnxruntime as ort

from postprocess import postprocess
from preprocess import preprocess


COCO_NAMES = [
    "person", "bicycle", "car", "motorcycle", "airplane", "bus", "train", "truck", "boat",
    "traffic light", "fire hydrant", "stop sign", "parking meter", "bench", "bird", "cat",
    "dog", "horse", "sheep", "cow", "elephant", "bear", "zebra", "giraffe", "backpack",
    "umbrella", "handbag", "tie", "suitcase", "frisbee", "skis", "snowboard", "sports ball",
    "kite", "baseball bat", "baseball glove", "skateboard", "surfboard", "tennis racket",
    "bottle", "wine glass", "cup", "fork", "knife", "spoon", "bowl", "banana", "apple",
    "sandwich", "orange", "broccoli", "carrot", "hot dog", "pizza", "donut", "cake",
    "chair", "couch", "potted plant", "bed", "dining table", "toilet", "tv", "laptop",
    "mouse", "remote", "keyboard", "cell phone", "microwave", "oven", "toaster", "sink",
    "refrigerator", "book", "clock", "vase", "scissors", "teddy bear", "hair drier",
    "toothbrush",
]


def draw_detections(image, boxes, scores, class_ids):
    for box, score, class_id in zip(boxes, scores, class_ids):
        x1, y1, x2, y2 = box.astype(int)
        label = f"{COCO_NAMES[int(class_id)]} {score:.2f}"

        cv2.rectangle(image, (x1, y1), (x2, y2), (0, 255, 0), 2)
        cv2.putText(
            image,
            label,
            (x1, max(y1 - 8, 20)),
            cv2.FONT_HERSHEY_SIMPLEX,
            0.7,
            (0, 255, 0),
            2,
            cv2.LINE_AA,
        )

    return image


def main():
    model_path = Path("models/yolo11n.onnx")
    image_path = Path("images/bus.jpg")
    output_path = Path("outputs/onnx_detect_bus.jpg")

    output_path.parent.mkdir(parents=True, exist_ok=True)

    session = ort.InferenceSession(str(model_path), providers=["CPUExecutionProvider"])
    input_name = session.get_inputs()[0].name
    output_name = session.get_outputs()[0].name

    tensor, original_shape, scale, pad = preprocess(image_path)
    output = session.run([output_name], {input_name: tensor})[0]

    boxes, scores, class_ids = postprocess(output, original_shape, scale, pad)

    print("detections:", len(boxes))
    for box, score, class_id in zip(boxes, scores, class_ids):
        name = COCO_NAMES[int(class_id)]
        xyxy = [round(float(v), 1) for v in box]
        print(name, round(float(score), 4), xyxy)

    image = cv2.imread(str(image_path))
    image = draw_detections(image, boxes, scores, class_ids)
    cv2.imwrite(str(output_path), image)
    print("saved:", output_path)


if __name__ == "__main__":
    main()
```

运行：

```bash
python scripts/onnx_detect.py
```

预期现象：

```text
detections: 5
bus ...
person ...
person ...
person ...
person ...
saved: outputs/onnx_detect_bus.jpg
```

查看结果文件：

```bash
ls outputs
```

## 必做 4：对比结果

使用 Ultralytics 作为参考：

```bash
yolo predict model=models/yolo11n.onnx source=images/bus.jpg imgsz=640 conf=0.25
```

参考结果应接近：

```text
4 persons, 1 bus
```

自己写的后处理不要求框的位置和 Ultralytics 完全逐像素一致，但类别和数量应大体一致。

如果自己代码输出很多重复框，通常是 NMS 没生效或坐标格式传错。

如果输出框位置明显偏移，重点检查：

- `pad_left` 和 `pad_top` 是否减掉。
- 坐标是否除以 `scale`。
- `original_shape` 是否是 `(height, width)`。
- `cv2.dnn.NMSBoxes` 需要的是 `[x, y, width, height]`，不是 `[x1, y1, x2, y2]`。

## 今日完成情况

- 已完成 `scripts/postprocess.py`，实现 YOLO ONNX 输出后处理：
  - `xywh -> xyxy`
  - 置信度筛选
  - letterbox 坐标映射回原图
  - OpenCV NMS
- 已完成 `scripts/onnx_detect.py`，复用 Day09 的 `preprocess.py` 和 `yolo11n.onnx`，完成端到端检测。
- 已生成自己后处理和画框的结果图：

```text
outputs/onnx_detect_bus.jpg
```

- 自己代码的关键输出：

```text
detections: 5
bus 0.9392 [11.9, 228.4, 799.2, 735.2]
person 0.902 [48.6, 398.0, 243.2, 904.5]
person 0.8493 [670.6, 392.6, 810.0, 879.6]
person 0.8328 [223.1, 405.6, 345.2, 859.7]
person 0.3993 [0.0, 550.2, 66.0, 871.8]
saved: outputs/onnx_detect_bus.jpg
```

- 已使用 Ultralytics 的 ONNX 推理结果作为参考：

```bash
yolo predict model=models/yolo11n.onnx source=images/bus.jpg imgsz=640 conf=0.25
```

关键输出：

```text
Using ONNX Runtime 1.23.2 with CPUExecutionProvider
image 1/1 /home/nefelibata/model-deploy-day09/images/bus.jpg: 640x480 4 persons, 1 bus, 22.7ms
Speed: 10.6ms preprocess, 22.7ms inference, 20.1ms postprocess per image at shape (1, 3, 640, 480)
Results saved to /home/nefelibata/model-deploy-day09/runs/detect/predict-2
```

- 已对比两张结果图：
  - `outputs/onnx_detect_bus.jpg`
  - `runs/detect/predict-2/bus.jpg`
- 对比结论：两者都检测到 `1 bus + 4 persons`，框位置大体一致，Day10 后处理链路通过。

## 遇到的问题

- 第一次运行 `python scripts/onnx_detect.py` 时报错：

```text
numpy.exceptions.AxisError: axis 1 is out of bounds for array of dimension 1
```

原因：

```python
class_scores = pred[:,4]
```

这行只取了第 4 列，导致 `class_scores` 变成一维数组，后续执行：

```python
np.argmax(class_scores, axis=1)
```

时就会报错。

修复方式：

```python
class_scores = pred[:, 4:]
```

含义：

- `pred[:, 4]`：只取第 4 列，结果是一维。
- `pred[:, 4:]`：取第 4 列到最后，得到 80 个类别分数，结果是二维 `(8400, 80)`。

这个问题帮助我真正理解了 YOLO ONNX 输出中 `84 = 4 + 80` 的结构。

- Ultralytics 推理时提示 `onnxruntime-gpu` 不存在，并回退到 CPU：

```text
CUDA requested but CUDAExecutionProvider not available. Using CPU...
```

这不是今天的错误。Day10 的目标是跑通 ONNX 后处理，ONNX Runtime GPU 和 TensorRT 加速后面再处理。

## 今日复盘

- 今天完成了从 ONNX 原始输出到最终检测结果的关键一步。
- 现在已经不只是会调用 `yolo predict`，而是能自己完成 YOLO 检测后处理。
- 对 `(1, 84, 8400)` 的理解更清楚：
  - 前 4 维是候选框坐标。
  - 后 80 维是 COCO 类别分数。
  - 8400 是候选框数量。
- 后处理核心链路已经跑通：

```text
ONNX output
-> transpose
-> split boxes and class scores
-> confidence filter
-> xywh to xyxy
-> remove letterbox padding
-> scale back to original image
-> NMS
-> draw boxes
```

- 自己代码和 Ultralytics 都得到 `4 persons, 1 bus`，说明处理方向正确。
- 自己画的绿色框标签有轻微贴边问题，这是可视化细节，不影响检测逻辑。

## 明日计划

- Day11 开始做推理性能计时和 benchmark。
- 记录 ONNX Runtime 推理的 preprocess、inference、postprocess 总耗时。
- 学习 warmup、平均延迟、FPS、P95 等指标。
- Jetson 不在身边时，继续先完成 PC 端部署链路；Jetson 在身边后再迁移到边缘设备。
