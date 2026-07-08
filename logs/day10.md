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

## 今日完成情况

待完成。

## 遇到的问题

待记录。

## 今日复盘

待记录。

## 明日计划

- 如果 Day10 后处理完成，Day11 开始做推理性能计时和 benchmark。
- 记录 PyTorch / ONNX Runtime / 后续 TensorRT 的性能指标口径：延迟、FPS、warmup、平均值、P95。
- Jetson 不在身边时，先继续完成 PC 端部署链路；Jetson 在身边后再迁移。
