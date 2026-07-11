# Day 12 学习记录

日期：2026-07-11

## 今日目标

- 理解 YOLO 输入尺寸对计算量、推理延迟和检测效果的影响。
- 确认当前动态 ONNX 模型可以接收 `320 / 480 / 640` 三种输入尺寸。
- 使用与 Day11 相同的 `10` 次 warmup 和 `100` 次正式测试进行公平对比。
- 自动生成输入尺寸性能对比表 `outputs/benchmark_imgsz_cpu.csv`。
- 保存三种尺寸的检测结果图，观察框、类别和置信度变化。
- 学会区分“单张图检测数量”和“数据集准确率”。

今天仍然只使用 PC / WSL，不需要 Jetson。

## 今日 7 小时学习计划

| 时间 | 内容 | 目标产出 |
| --- | --- | --- |
| 第 1 小时 | 复习动态输入和特征图 | 理解输入尺寸、像素数和候选框数量的关系 |
| 第 2 小时 | 检查模型动态 shape | 确认模型支持 `height / width` 动态变化 |
| 第 3 小时 | 编写多尺寸 benchmark | 自动测试 `320 / 480 / 640` |
| 第 4 小时 | 保存 CSV 和检测结果图 | 得到性能表和三张可视化图片 |
| 第 5 小时 | 分析速度变化 | 比较 inference、total、P95 和 FPS |
| 第 6 小时 | 分析检测结果变化 | 比较检测数量、类别、框和置信度 |
| 第 7 小时 | 复盘与 GitHub 记录 | 写结论、排错记录和 Day13 计划 |

## 开始前准备

在 WSL 中执行：

```bash
conda activate deploy310
cd ~/model-deploy-day09
code .
```

确认终端提示符是 `(deploy310)`，然后检查环境：

```bash
which python
python --version
python scripts/ort_model_info.py
```

当前模型应显示：

```text
inputs:
name: images
shape: ['batch', 3, 'height', 'width']
type: tensor(float)

outputs:
name: output0
shape: ['batch', 84, 'anchors']
type: tensor(float)
```

这说明当前 ONNX 模型支持动态输入尺寸。`height`、`width` 和输出 `anchors` 会随输入尺寸变化。

确认 Day10 和 Day11 文件存在：

```bash
ls scripts/preprocess.py
ls scripts/postprocess.py
ls scripts/onnx_detect.py
ls models/yolo11n.onnx
ls images/bus.jpg
```

## 必做 1：理解输入尺寸为什么影响速度

YOLO 输入尺寸越大，需要处理的像素越多：

| 输入尺寸 | 像素数量 | 相对 320 的像素量 |
| --- | ---: | ---: |
| 320 x 320 | 102,400 | 1.00 倍 |
| 480 x 480 | 230,400 | 2.25 倍 |
| 640 x 640 | 409,600 | 4.00 倍 |

计算量通常会随像素数明显增长，但实际延迟不一定严格按 `1 : 2.25 : 4` 增长，因为还会受到：

- CPU 并行效率。
- 算子实现。
- 内存访问。
- ONNX Runtime 线程调度。
- 系统负载与缓存。

YOLO11 检测头使用多个尺度的特征图。对于正方形输入，候选框数量可近似理解为：

```text
anchors = (imgsz / 8)^2 + (imgsz / 16)^2 + (imgsz / 32)^2
```

因此三种输入尺寸对应：

| 输入尺寸 | 候选框数量 |
| --- | ---: |
| 320 | 2,100 |
| 480 | 4,725 |
| 640 | 8,400 |

输入尺寸越大，模型能保留更多图像细节，对小目标通常更友好，但推理速度会下降。

今天选用的三个尺寸都是 `32` 的整数倍，这与模型多次下采样的结构匹配。

## 必做 2：编写多尺寸性能测试脚本

在 VS Code 新建：

```text
scripts/benchmark_imgsz.py
```

写入完整代码：

```python
import csv
from pathlib import Path
from time import perf_counter

import cv2
import numpy as np
import onnxruntime as ort

from onnx_detect import draw_detections
from postprocess import postprocess
from preprocess import preprocess


MODEL_PATH = Path("models/yolo11n.onnx")
IMAGE_PATH = Path("images/bus.jpg")
CSV_PATH = Path("outputs/benchmark_imgsz_cpu.csv")
IMAGE_OUTPUT_DIR = Path("outputs/imgsz")

INPUT_SIZES = [320, 480, 640]
WARMUP_RUNS = 10
TEST_RUNS = 100


def summarize(latencies_ms):
    values = np.asarray(latencies_ms, dtype=np.float64)
    mean_ms = float(np.mean(values))

    return {
        "mean_ms": mean_ms,
        "min_ms": float(np.min(values)),
        "max_ms": float(np.max(values)),
        "p50_ms": float(np.percentile(values, 50)),
        "p95_ms": float(np.percentile(values, 95)),
        "fps": 1000.0 / mean_ms,
    }


def benchmark_size(session, input_name, output_name, input_size):
    input_shape = (input_size, input_size)

    warmup_tensor, _, _, _ = preprocess(
        IMAGE_PATH,
        input_shape=input_shape,
    )

    for _ in range(WARMUP_RUNS):
        session.run([output_name], {input_name: warmup_tensor})

    preprocess_times = []
    inference_times = []
    postprocess_times = []
    total_times = []

    last_output = None
    last_result = None

    for _ in range(TEST_RUNS):
        total_start = perf_counter()

        stage_start = perf_counter()
        tensor, original_shape, scale, pad = preprocess(
            IMAGE_PATH,
            input_shape=input_shape,
        )
        preprocess_times.append((perf_counter() - stage_start) * 1000.0)

        stage_start = perf_counter()
        output = session.run([output_name], {input_name: tensor})[0]
        inference_times.append((perf_counter() - stage_start) * 1000.0)

        stage_start = perf_counter()
        boxes, scores, class_ids = postprocess(
            output,
            original_shape,
            scale,
            pad,
        )
        postprocess_times.append((perf_counter() - stage_start) * 1000.0)

        total_times.append((perf_counter() - total_start) * 1000.0)

        last_output = output
        last_result = (boxes, scores, class_ids)

    boxes, scores, class_ids = last_result

    original_image = cv2.imread(str(IMAGE_PATH))
    if original_image is None:
        raise FileNotFoundError(f"failed to read image: {IMAGE_PATH}")

    result_image = draw_detections(
        original_image.copy(),
        boxes,
        scores,
        class_ids,
    )
    result_path = IMAGE_OUTPUT_DIR / f"bus_{input_size}.jpg"
    cv2.imwrite(str(result_path), result_image)

    preprocess_stats = summarize(preprocess_times)
    inference_stats = summarize(inference_times)
    postprocess_stats = summarize(postprocess_times)
    total_stats = summarize(total_times)

    return {
        "input_size": input_size,
        "pixels": input_size * input_size,
        "anchors": int(last_output.shape[-1]),
        "detections": len(boxes),
        "preprocess_mean_ms": preprocess_stats["mean_ms"],
        "inference_mean_ms": inference_stats["mean_ms"],
        "inference_p95_ms": inference_stats["p95_ms"],
        "postprocess_mean_ms": postprocess_stats["mean_ms"],
        "total_mean_ms": total_stats["mean_ms"],
        "total_p50_ms": total_stats["p50_ms"],
        "total_p95_ms": total_stats["p95_ms"],
        "total_fps": total_stats["fps"],
        "result_image": str(result_path),
    }


def print_result(result):
    print(
        f"imgsz={result['input_size']:>3}  "
        f"anchors={result['anchors']:>4}  "
        f"detections={result['detections']:>2}  "
        f"infer={result['inference_mean_ms']:>8.3f} ms  "
        f"total={result['total_mean_ms']:>8.3f} ms  "
        f"P95={result['total_p95_ms']:>8.3f} ms  "
        f"FPS={result['total_fps']:>7.2f}"
    )


def save_csv(results):
    fieldnames = list(results[0].keys())

    with CSV_PATH.open("w", newline="", encoding="utf-8") as file:
        writer = csv.DictWriter(file, fieldnames=fieldnames)
        writer.writeheader()

        for result in results:
            formatted = result.copy()

            for key, value in formatted.items():
                if isinstance(value, float):
                    formatted[key] = f"{value:.3f}"

            writer.writerow(formatted)


def main():
    CSV_PATH.parent.mkdir(parents=True, exist_ok=True)
    IMAGE_OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

    session = ort.InferenceSession(
        str(MODEL_PATH),
        providers=["CPUExecutionProvider"],
    )
    input_name = session.get_inputs()[0].name
    output_name = session.get_outputs()[0].name

    print("model:", MODEL_PATH)
    print("providers:", session.get_providers())
    print("model input shape:", session.get_inputs()[0].shape)
    print("warmup runs per size:", WARMUP_RUNS)
    print("test runs per size:", TEST_RUNS)
    print()

    results = []

    for input_size in INPUT_SIZES:
        print(f"benchmarking imgsz={input_size} ...")
        result = benchmark_size(
            session,
            input_name,
            output_name,
            input_size,
        )
        results.append(result)
        print_result(result)
        print("saved:", result["result_image"])
        print()

    save_csv(results)

    print("summary:")
    for result in results:
        print_result(result)

    print(f"\nsaved: {CSV_PATH}")


if __name__ == "__main__":
    main()
```

## 必做 3：运行多尺寸 benchmark

确认当前目录：

```bash
pwd
```

应为：

```text
/home/nefelibata/model-deploy-day09
```

运行：

```bash
python scripts/benchmark_imgsz.py
```

程序会依次执行：

```text
320: 10 次 warmup + 100 次正式测试
480: 10 次 warmup + 100 次正式测试
640: 10 次 warmup + 100 次正式测试
```

预期输出结构：

```text
model: models/yolo11n.onnx
providers: ['CPUExecutionProvider']
model input shape: ['batch', 3, 'height', 'width']
warmup runs per size: 10
test runs per size: 100

benchmarking imgsz=320 ...
imgsz=320  anchors=2100  detections=...  infer=... ms  total=... ms  P95=... ms  FPS=...
saved: outputs/imgsz/bus_320.jpg

benchmarking imgsz=480 ...
imgsz=480  anchors=4725  detections=...  infer=... ms  total=... ms  P95=... ms  FPS=...
saved: outputs/imgsz/bus_480.jpg

benchmarking imgsz=640 ...
imgsz=640  anchors=8400  detections=...  infer=... ms  total=... ms  P95=... ms  FPS=...
saved: outputs/imgsz/bus_640.jpg

saved: outputs/benchmark_imgsz_cpu.csv
```

不同电脑的延迟和 FPS 会不同，不要照抄预期数值。

## 必做 4：检查 CSV 和结果图

查看 CSV：

```bash
column -s, -t outputs/benchmark_imgsz_cpu.csv
```

如果 `column` 命令不可用，直接执行：

```bash
cat outputs/benchmark_imgsz_cpu.csv
```

检查输出文件：

```bash
ls -lh outputs/benchmark_imgsz_cpu.csv outputs/imgsz/*.jpg
```

在 VS Code 文件栏依次打开：

```text
outputs/imgsz/bus_320.jpg
outputs/imgsz/bus_480.jpg
outputs/imgsz/bus_640.jpg
```

观察：

- 三种尺寸分别检测出多少个目标。
- 是否都检测到了 bus。
- 4 个 person 是否都保留。
- 边缘位置的人是否更容易在小尺寸下丢失。
- 同一目标的置信度是否变化。
- 框的位置是否明显偏移。

## 必做 5：整理性能对比表

把实际结果填写到下表：

| imgsz | anchors | detections | inference mean | total mean | total P95 | total FPS |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 320 | 待填写 | 待填写 | 待填写 | 待填写 | 待填写 | 待填写 |
| 480 | 待填写 | 待填写 | 待填写 | 待填写 | 待填写 | 待填写 |
| 640 | 待填写 | 待填写 | 待填写 | 待填写 | 待填写 | 待填写 |

然后回答：

1. 从 320 增加到 640，像素数量变为多少倍？
2. inference mean 增长了多少倍？是否与像素数量严格一致？
3. 哪个输入尺寸的 total FPS 最高？
4. 哪个输入尺寸的检测结果最完整？
5. 如果目标是实时摄像头检测，你会优先选择哪个尺寸？为什么？

## 重要概念：检测数量不等于准确率

今天只用一张 `bus.jpg`，所以只能做功能和现象对比，不能声称：

```text
640 的准确率是 ...
320 的准确率是 ...
```

原因是准确率需要带真实标签的数据集，并计算：

- Precision。
- Recall。
- mAP50。
- mAP50-95。

今天可以得出的结论是：

```text
在这张测试图和当前阈值下，不同输入尺寸的速度与检测输出存在什么差异。
```

后续正式项目会使用验证集评估 mAP，再讨论真正的速度与精度权衡。

## 选做加餐 1：计算加速比例

假设 640 的完整流程平均延迟为基线：

```text
speedup_320 = total_mean_640 / total_mean_320
speedup_480 = total_mean_640 / total_mean_480
```

例如结果为 `2.50`，表示 320 输入的完整流程约比 640 快 `2.50` 倍。

不要使用 FPS 直接相除后又称为“延迟降低倍数”，描述时要明确比较的是延迟还是 FPS。

## 选做加餐 2：再运行一轮

为了检查稳定性，再运行一次：

```bash
python scripts/benchmark_imgsz.py
```

注意：第二次运行会覆盖第一次的 CSV 和结果图。运行前先把第一轮终端结果保留在日志中。

如果两轮差异明显，记录：

- 是否接通电源。
- Windows 是否处于最佳性能模式。
- 是否有其他程序占用 CPU。
- 机器温度是否升高。

## 常见错误

### 1. 在 `(base)` 环境运行时报缺少依赖

先执行：

```bash
conda activate deploy310
```

确认提示符变为 `(deploy310)` 后再运行。

### 2. 模型提示输入尺寸必须是 640

先检查：

```bash
python scripts/ort_model_info.py
```

当前模型应显示动态输入：

```text
['batch', 3, 'height', 'width']
```

如果显示 `[1, 3, 640, 640]`，说明使用了其他静态 ONNX 模型，不能直接执行今天的三尺寸测试。

### 3. `ImportError` 或找不到 `onnx_detect`

从项目根目录运行：

```bash
cd ~/model-deploy-day09
python scripts/benchmark_imgsz.py
```

### 4. 检测框位置错误

今天继续复用 Day10 已验证的 `postprocess.py`。如果只在某个尺寸下发生偏移，检查调用 `preprocess()` 时是否传入了：

```python
input_shape=(input_size, input_size)
```

并确认 `scale` 和 `pad` 是该轮预处理返回的值。

### 5. 小尺寸检测数量减少

这不一定是代码错误。缩小输入会损失细节，置信度也可能跌到 `0.25` 以下。先打开结果图，比较类别、分数和框，再判断问题。

## 今日产出清单

完成后应有：

```text
scripts/benchmark_imgsz.py
outputs/benchmark_imgsz_cpu.csv
outputs/imgsz/bus_320.jpg
outputs/imgsz/bus_480.jpg
outputs/imgsz/bus_640.jpg
```

## 今日完成情况

- [ ] 确认 ONNX 模型支持动态输入尺寸。
- [ ] 理解输入像素数和 anchors 数量的变化。
- [ ] 完成 `scripts/benchmark_imgsz.py`。
- [ ] 完成三个输入尺寸的公平性能测试。
- [ ] 生成 CSV 性能表。
- [ ] 生成并查看三张检测结果图。
- [ ] 分析速度、波动和检测输出的差异。

## 实际测试结果

完成后填写。

## 遇到的问题

完成后填写。

## 今日复盘

完成后填写。

## 明日计划

- Day13 开始 ONNX Runtime GPU 环境评估与 CUDA Execution Provider 学习。
- 先核对 CUDA、cuDNN、ONNX Runtime 版本兼容关系，再决定是否安装，避免破坏当前可用 CPU 环境。
- 保留 Day11 和 Day12 的 CPU 基线，后续与 GPU、TensorRT 使用同一套测试方法比较。
