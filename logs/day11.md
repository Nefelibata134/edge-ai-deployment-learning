# Day 11 学习记录

日期：2026-07-10

## 今日目标

- 在 PC / WSL 环境中为 YOLO11n ONNX 推理建立性能基线。
- 理解 `warmup`、单次延迟、平均延迟、P50、P95、FPS 的含义。
- 分别测量预处理、ONNX Runtime 推理、后处理和端到端总耗时。
- 避免把模型加载、首次初始化和图片读取误算进稳定推理延迟。
- 生成可复用的 benchmark 脚本和文本性能报告。

今天不需要 Jetson，继续使用 Day09/Day10 的 PC / WSL 目录：

```text
/home/nefelibata/model-deploy-day09
```

## 今日 7 小时学习计划

| 时间 | 内容 | 目标产出 |
| --- | --- | --- |
| 第 1 小时 | 复习完整推理链路和性能指标 | 能区分 preprocess、inference、postprocess、end-to-end |
| 第 2 小时 | 学习可靠计时方法 | 理解 `time.perf_counter()`、warmup 和多次重复 |
| 第 3 小时 | 编写核心推理 benchmark | 得到 ONNX Runtime inference 的 mean、P50、P95、FPS |
| 第 4 小时 | 编写端到端 benchmark | 分阶段记录预处理、推理、后处理和 total |
| 第 5 小时 | 运行 100 次并保存报告 | 生成 `outputs/benchmark_cpu.txt` |
| 第 6 小时 | 分析数据和常见误区 | 判断瓶颈在哪，理解 FPS 的适用范围 |
| 第 7 小时 | 复盘和 GitHub 记录 | 填写实际结果、问题和 Day12 计划 |

## 开始前准备

在 WSL 终端执行：

```bash
conda activate deploy310
cd ~/model-deploy-day09
code .
```

确认解释器和依赖：

```bash
which python
python --version
python -c "import onnxruntime as ort; print(ort.__version__); print(ort.get_available_providers())"
```

预期看到：

```text
/home/nefelibata/miniconda3/envs/deploy310/bin/python
Python 3.10.20
1.23.2
['AzureExecutionProvider', 'CPUExecutionProvider']
```

确认文件存在：

```bash
find models images scripts -maxdepth 2 -type f | sort
```

至少需要：

```text
images/bus.jpg
models/yolo11n.onnx
scripts/preprocess.py
scripts/postprocess.py
```

## 必做 1：理解今天的性能指标

### 1. 延迟 latency

一张图片完成某个阶段所需的时间，通常用毫秒 `ms` 表示。

```text
latency_ms = elapsed_seconds * 1000
```

### 2. 平均延迟 mean

多次运行耗时的算术平均值。它能描述总体速度，但容易受到偶发慢请求影响。

### 3. P50

中位数。50% 的运行耗时不超过该值。它比平均值更能代表常见体验。

### 4. P95

95% 的运行耗时不超过该值。P95 可以观察推理是否偶尔出现明显抖动。

### 5. FPS

在 batch size 为 1、串行处理图片时，可用平均延迟估算：

```text
FPS = 1000 / mean_latency_ms
```

注意：只用 inference 延迟计算的是理论核心推理 FPS；用 total 延迟计算的才更接近完整程序 FPS。

### 6. warmup

正式计时前先运行若干次推理。第一次运行可能包含内存分配、算子初始化和缓存建立，直接计入会让结果失真。

今天使用：

```text
warmup = 10
runs = 100
```

## 必做 2：编写核心推理 benchmark

在 VS Code 新建：

```text
scripts/benchmark_inference.py
```

写入完整代码：

```python
from pathlib import Path
from time import perf_counter

import numpy as np
import onnxruntime as ort

from preprocess import preprocess


MODEL_PATH = Path("models/yolo11n.onnx")
IMAGE_PATH = Path("images/bus.jpg")
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


def main():
    session = ort.InferenceSession(
        str(MODEL_PATH),
        providers=["CPUExecutionProvider"],
    )
    input_name = session.get_inputs()[0].name
    output_name = session.get_outputs()[0].name

    tensor, _, _, _ = preprocess(IMAGE_PATH)

    print("provider:", session.get_providers())
    print("input shape:", tensor.shape)
    print("warmup runs:", WARMUP_RUNS)
    print("test runs:", TEST_RUNS)

    for _ in range(WARMUP_RUNS):
        session.run([output_name], {input_name: tensor})

    latencies_ms = []

    for _ in range(TEST_RUNS):
        start = perf_counter()
        session.run([output_name], {input_name: tensor})
        elapsed_ms = (perf_counter() - start) * 1000.0
        latencies_ms.append(elapsed_ms)

    stats = summarize(latencies_ms)

    print("\ninference benchmark:")
    print(f"mean: {stats['mean_ms']:.3f} ms")
    print(f"min:  {stats['min_ms']:.3f} ms")
    print(f"max:  {stats['max_ms']:.3f} ms")
    print(f"P50:  {stats['p50_ms']:.3f} ms")
    print(f"P95:  {stats['p95_ms']:.3f} ms")
    print(f"FPS:  {stats['fps']:.2f}")


if __name__ == "__main__":
    main()
```

运行：

```bash
python scripts/benchmark_inference.py
```

你会看到类似结构，具体数值以你的电脑为准：

```text
provider: ['CPUExecutionProvider']
input shape: (1, 3, 640, 640)
warmup runs: 10
test runs: 100

inference benchmark:
mean: xx.xxx ms
min:  xx.xxx ms
max:  xx.xxx ms
P50:  xx.xxx ms
P95:  xx.xxx ms
FPS:  xx.xx
```

## 必做 3：编写分阶段端到端 benchmark

上一个脚本只测：

```text
session.run(...)
```

真实检测程序还包括预处理和后处理。新建：

```text
scripts/benchmark_pipeline.py
```

写入完整代码：

```python
from pathlib import Path
from time import perf_counter

import numpy as np
import onnxruntime as ort

from postprocess import postprocess
from preprocess import preprocess


MODEL_PATH = Path("models/yolo11n.onnx")
IMAGE_PATH = Path("images/bus.jpg")
OUTPUT_PATH = Path("outputs/benchmark_cpu.txt")
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


def format_stats(name, stats):
    return (
        f"{name:<12} "
        f"mean={stats['mean_ms']:8.3f} ms  "
        f"P50={stats['p50_ms']:8.3f} ms  "
        f"P95={stats['p95_ms']:8.3f} ms  "
        f"min={stats['min_ms']:8.3f} ms  "
        f"max={stats['max_ms']:8.3f} ms  "
        f"FPS={stats['fps']:7.2f}"
    )


def main():
    OUTPUT_PATH.parent.mkdir(parents=True, exist_ok=True)

    session = ort.InferenceSession(
        str(MODEL_PATH),
        providers=["CPUExecutionProvider"],
    )
    input_name = session.get_inputs()[0].name
    output_name = session.get_outputs()[0].name

    warmup_tensor, _, _, _ = preprocess(IMAGE_PATH)

    for _ in range(WARMUP_RUNS):
        session.run([output_name], {input_name: warmup_tensor})

    preprocess_times = []
    inference_times = []
    postprocess_times = []
    total_times = []
    detection_counts = []

    for _ in range(TEST_RUNS):
        total_start = perf_counter()

        stage_start = perf_counter()
        tensor, original_shape, scale, pad = preprocess(IMAGE_PATH)
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
        detection_counts.append(len(boxes))

    all_stats = {
        "preprocess": summarize(preprocess_times),
        "inference": summarize(inference_times),
        "postprocess": summarize(postprocess_times),
        "total": summarize(total_times),
    }

    lines = [
        "YOLO11n ONNX Runtime CPU Benchmark",
        f"model: {MODEL_PATH}",
        f"image: {IMAGE_PATH}",
        f"providers: {session.get_providers()}",
        f"warmup runs: {WARMUP_RUNS}",
        f"test runs: {TEST_RUNS}",
        f"detections: {detection_counts[-1]}",
        "",
    ]

    for name in ("preprocess", "inference", "postprocess", "total"):
        lines.append(format_stats(name, all_stats[name]))

    report = "\n".join(lines)
    print(report)
    OUTPUT_PATH.write_text(report + "\n", encoding="utf-8")
    print(f"\nsaved: {OUTPUT_PATH}")


if __name__ == "__main__":
    main()
```

运行：

```bash
python scripts/benchmark_pipeline.py
```

查看保存的报告：

```bash
cat outputs/benchmark_cpu.txt
```

预期结构：

```text
YOLO11n ONNX Runtime CPU Benchmark
model: models/yolo11n.onnx
image: images/bus.jpg
providers: ['CPUExecutionProvider']
warmup runs: 10
test runs: 100
detections: 5

preprocess   mean= ... ms  P50= ... ms  P95= ... ms  min= ... ms  max= ... ms  FPS= ...
inference    mean= ... ms  P50= ... ms  P95= ... ms  min= ... ms  max= ... ms  FPS= ...
postprocess  mean= ... ms  P50= ... ms  P95= ... ms  min= ... ms  max= ... ms  FPS= ...
total        mean= ... ms  P50= ... ms  P95= ... ms  min= ... ms  max= ... ms  FPS= ...

saved: outputs/benchmark_cpu.txt
```

## 必做 4：检查计时是否可信

连续运行两次：

```bash
python scripts/benchmark_pipeline.py
python scripts/benchmark_pipeline.py
```

两次结果不需要完全相同，但平均延迟不应相差数倍。

测试时注意：

- 接通笔记本电源，避免省电模式主动降频。
- 暂停大型下载、游戏和其他高占用程序。
- 不要只运行 1 次就得出结论。
- 不要把 `InferenceSession` 的创建时间算进每次推理。
- `preprocess()` 每轮都会从磁盘读取图片，所以这里的预处理时间包含图片读取；需要在报告里如实说明。
- 今天使用的是 `CPUExecutionProvider`，RTX 4070 没有参与 ONNX Runtime 推理。

## 必做 5：分析你的结果

运行结束后回答下面 5 个问题，并写到“今日复盘”：

1. `inference mean` 是多少毫秒？
2. `total mean` 是多少毫秒？对应完整流程 FPS 是多少？
3. 四个阶段中哪个平均耗时最大？
4. P95 比 P50 高多少？波动是否明显？
5. 为什么 Ultralytics 输出的 Speed 和自己的 benchmark 不一定完全一致？

第 5 题参考方向：

- warmup 次数不同。
- 计时边界不同。
- Ultralytics 和自己的预处理/后处理实现细节不同。
- 线程设置、系统负载和缓存状态不同。
- Ultralytics 展示的各阶段平均方式可能与自己的循环不同。

## 选做加餐：观察 CPU 占用

另开一个 WSL 终端：

```bash
htop
```

如果没有安装：

```bash
sudo apt update
sudo apt install -y htop
```

然后在原终端运行 benchmark，观察是否有多个 CPU 核心参与。

也可以查看 ONNX Runtime session 的线程设置：

```python
options = ort.SessionOptions()
print("intra_op_num_threads:", options.intra_op_num_threads)
print("inter_op_num_threads:", options.inter_op_num_threads)
```

默认显示 `0` 表示由 ONNX Runtime 自动决定，不是只使用 0 个线程。

## 常见错误

### 1. `ModuleNotFoundError: No module named 'preprocess'`

请从项目根目录运行：

```bash
cd ~/model-deploy-day09
python scripts/benchmark_pipeline.py
```

不要进入 `scripts` 目录后再随意改变运行方式。

### 2. `No such file or directory: models/yolo11n.onnx`

说明当前目录不对，执行：

```bash
pwd
ls models/yolo11n.onnx
```

正确工作目录应为：

```text
/home/nefelibata/model-deploy-day09
```

### 3. P95 或 max 偶尔很高

先再运行一次，不要立刻认定代码有问题。系统调度、后台程序、温度和电源模式都会带来抖动。我们记录 P95，就是为了看见这种现象。

### 4. 结果仍显示 CPU

这是今天的预期：

```text
providers: ['CPUExecutionProvider']
```

今天先完成 CPU 基线，不在同一天混入 `onnxruntime-gpu` 的 CUDA/cuDNN 兼容问题。

## 今日产出清单

完成后应有：

```text
scripts/benchmark_inference.py
scripts/benchmark_pipeline.py
outputs/benchmark_cpu.txt
```

检查：

```bash
ls -lh scripts/benchmark_inference.py scripts/benchmark_pipeline.py outputs/benchmark_cpu.txt
```

## 今日完成情况

- [x] 理解 mean、P50、P95、FPS 和 warmup。
- [x] 完成 `scripts/benchmark_inference.py`。
- [x] 完成 `scripts/benchmark_pipeline.py`。
- [x] 完成 10 次 warmup + 100 次正式测试。
- [x] 生成 `outputs/benchmark_cpu.txt`。
- [x] 分析性能瓶颈并记录实际数据。

## 实际测试结果

### 核心推理 benchmark

```text
provider: ['CPUExecutionProvider']
input shape: (1, 3, 640, 640)
warmup runs: 10
test runs: 100
mean: 26.977 ms
min: 23.011 ms
max: 44.184 ms
P50: 25.872 ms
P95: 41.242 ms
FPS: 37.07
```

### 最终端到端 CPU 基线

```text
model: models/yolo11n.onnx
image: images/bus.jpg
providers: ['CPUExecutionProvider']
warmup runs: 10
test runs: 100
detections: 5

preprocess   mean=   3.929 ms  P50=   3.864 ms  P95=   4.480 ms  min=   3.400 ms  max=   7.069 ms  FPS= 254.54
inference    mean=  26.069 ms  P50=  26.249 ms  P95=  28.165 ms  min=  23.340 ms  max=  33.873 ms  FPS=  38.36
postprocess  mean=   0.999 ms  P50=   0.987 ms  P95=   1.158 ms  min=   0.850 ms  max=   1.459 ms  FPS=1000.68
total        mean=  31.001 ms  P50=  31.272 ms  P95=  33.505 ms  min=  27.680 ms  max=  38.661 ms  FPS=  32.26
```

最终 CPU 基线结论：

- 单张图片完整检测平均耗时为 `31.001 ms`，约 `32.26 FPS`。
- 核心 ONNX 推理平均耗时为 `26.069 ms`，约占总耗时的 `84.1%`，是当前主要瓶颈。
- 预处理平均耗时为 `3.929 ms`，约占总耗时的 `12.7%`。
- 后处理平均耗时为 `0.999 ms`，约占总耗时的 `3.2%`。
- 最终一轮 total P95 为 `33.505 ms`，比 P50 高 `2.233 ms`，这一轮波动较小。
- 检测数量始终为 5，与 Day10 的 `1 bus + 4 persons` 一致。

### 多轮重复测试

为了观察波动，连续运行了多次端到端 benchmark：

| 测试 | inference mean | total mean | total P50 | total P95 | total FPS |
| --- | ---: | ---: | ---: | ---: | ---: |
| 第 1 轮 | 28.821 ms | 34.278 ms | 33.569 ms | 40.020 ms | 29.17 |
| 第 2 轮 | 34.181 ms | 38.794 ms | 41.986 ms | 44.342 ms | 25.78 |
| 第 3 轮 | 26.457 ms | 31.119 ms | 29.444 ms | 41.966 ms | 32.14 |
| 第 4 轮 | 26.069 ms | 31.001 ms | 31.272 ms | 33.505 ms | 32.26 |

多轮结果说明：

- 完整流程性能大致在 `25.78-32.26 FPS` 之间。
- 推理阶段受 CPU 调度、后台程序、缓存状态和电源策略影响，会发生波动。
- 第 2 轮明显偏慢，证明只测一次不能代表稳定性能。
- 第 4 轮的 P50 和 P95 最接近，适合作为当前稳定基线，但性能报告仍应说明测试次数和运行环境。

## 遇到的问题

### 1. 在 base 环境运行时缺少 NumPy

第一次运行：

```bash
(base) python scripts/benchmark_inference.py
```

报错：

```text
ModuleNotFoundError: No module named 'numpy'
```

原因不是 benchmark 代码错误，而是当前处于 Miniconda 的 `base` 环境，Day04 安装的 NumPy、ONNX Runtime 等依赖位于 `deploy310` 环境。

修复：

```bash
conda activate deploy310
python scripts/benchmark_inference.py
```

以后运行项目前先观察终端提示符是否为：

```text
(deploy310)
```

### 2. 终端出现 `^[[A`

第二轮输出开头出现：

```text
^[[A
```

这是终端把方向键上键的控制字符显示了出来，不是 Python 或 ONNX Runtime 报错，对测试结果没有影响。


## 今日复盘

- 今天第一次建立了可重复的模型部署性能测试方法，不再只看一次 `yolo predict` 的速度输出。
- 学会了正式计时前做 warmup，并使用 100 次运行计算 mean、P50、P95、min 和 max。
- 核心推理 FPS 和端到端 FPS 含义不同：
  - 核心推理约 `38.36 FPS`。
  - 包含图片读取、预处理和后处理的完整流程约 `32.26 FPS`。
- 当前性能瓶颈是 ONNX Runtime CPU 推理，预处理和后处理占比较小。
- P95 能反映偶发慢请求。多轮测试中 P95 波动明显，因此后续比较 ONNX GPU 和 TensorRT 时必须使用相同图片、输入尺寸、warmup 次数和测试次数。
- Ultralytics 的 Speed 与自己的 benchmark 不一定完全一致，因为计时边界、warmup、预处理实现、后处理实现、线程设置和系统负载可能不同。
- `deploy310` 是本项目的依赖环境；终端处于 `(base)` 时不能假定项目依赖已经安装。

## 明日计划

- Day12 对比 `320 / 480 / 640` 不同输入尺寸对速度和检测结果的影响。
- 生成输入尺寸、平均延迟、P95、FPS、检测数量的对比表。
- 固定 benchmark 规则，为后续 ONNX Runtime GPU 和 TensorRT 测试建立统一标准。
- Jetson 不在身边时继续完成 PC 端部署链路。
