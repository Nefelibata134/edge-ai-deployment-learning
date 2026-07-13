# Day 14 学习记录

日期：2026-07-13

主题：ONNX Runtime I/O Binding 与 GPU 数据传输

## 今日目标

- 理解普通 `session.run()` 隐含的 CPU/GPU 数据复制。
- 区分 CPU NumPy、CUDA OrtValue 和设备端输出。
- 使用 I/O Binding 完成一次正确的 YOLO11n CUDA 推理。
- 公平比较三条路径：普通 NumPy、I/O Binding CPU 往返、I/O Binding 设备端驻留。
- 验证三条路径输出一致，并记录 mean、P50、P95 和 FPS。
- 用 30 分钟准备 Kaggle Digit Recognizer baseline，不在今天重投入竞赛。

今天不需要 Jetson，继续使用 RTX 4070 + WSL2。

## 7 小时安排

| 时间 | 内容 | 产出 |
| ---: | --- | --- |
| 0.5 小时 | 环境检查与 Day13 复习 | Provider 和 Python 路径确认 |
| 1 小时 | 理解普通 ORT 与 I/O Binding 数据路径 | 三条数据路径图 |
| 1.5 小时 | 完成 `io_binding_check.py` | CUDA 输入输出验证 |
| 2 小时 | 完成 `benchmark_iobinding.py` | 三路径统一 benchmark |
| 1 小时 | 多轮运行、计算加速比、分析波动 | 性能对比表 |
| 0.5 小时 | Kaggle Digit Recognizer baseline 准备 | 数据与提交格式记录 |
| 0.5 小时 | 复盘、日志和 Git 提交 | Day14 完成记录 |

## 必做 1：环境检查

在 VS Code 的 WSL 终端中执行：

```bash
conda activate ortgpu310
cd ~/model-deploy-day09

which python
python --version
python -c "import onnxruntime as ort; print(ort.__version__); print(ort.get_available_providers())"
ls models images scripts outputs
```

预期关键结果：

```text
/home/nefelibata/miniconda3/envs/ortgpu310/bin/python
Python 3.10.20
1.23.2
['TensorrtExecutionProvider', 'CUDAExecutionProvider', 'CPUExecutionProvider']
```

如果终端前面不是 `(ortgpu310)`，不要继续运行脚本。

## 必做 2：理解三条数据路径

### 路径 A：普通 `session.run()`

```text
NumPy tensor in CPU RAM
        |
        | hidden Host -> Device copy
        v
CUDA inference
        |
        | hidden Device -> Host copy
        v
NumPy output in CPU RAM
```

它最方便，适合单次 Python 推理，但输入输出默认都是 NumPy，ORT 需要在内部完成数据复制。

### 路径 B：I/O Binding，但仍然 CPU 往返

```text
bind_cpu_input
        |
        v
CUDA inference
        |
copy_outputs_to_cpu
        v
NumPy output
```

这条路径显式控制绑定，但仍包含 Host/Device 往返，因此不保证比 `session.run()` 更快。

### 路径 C：输入输出都留在 GPU

```text
CUDA OrtValue input
        |
        v
CUDA inference
        |
        v
CUDA OrtValue output
```

输入上传和输出下载不放在循环中，适合前后处理也在 GPU、模型串联或视频 pipeline。今天用它观察纯设备端路径。

重要：路径 C 的时间不能直接冒充完整应用端到端延迟，因为它没有计算图片预处理、首次 H2D 上传和最终 D2H 下载。

## 必做 3：完成 I/O Binding 正确性检查

在 VS Code 中新建：

```text
scripts/io_binding_check.py
```

写入完整代码：

```python
from pathlib import Path

import numpy as np
import onnxruntime as ort

from postprocess import postprocess
from preprocess import preprocess


MODEL_PATH = Path("models/yolo11n.onnx")
IMAGE_PATH = Path("images/bus.jpg")


def main():
    ort.preload_dlls(directory="")

    available_providers = ort.get_available_providers()
    if "CUDAExecutionProvider" not in available_providers:
        raise RuntimeError(
            f"CUDAExecutionProvider is unavailable: {available_providers}"
        )

    session = ort.InferenceSession(
        str(MODEL_PATH),
        providers=[
            (
                "CUDAExecutionProvider",
                {
                    "device_id": 0,
                    "cudnn_conv_algo_search": "EXHAUSTIVE",
                },
            ),
            "CPUExecutionProvider",
        ],
    )

    input_name = session.get_inputs()[0].name
    output_name = session.get_outputs()[0].name
    tensor, original_shape, scale, pad = preprocess(IMAGE_PATH)

    reference_output = session.run(
        [output_name],
        {input_name: tensor},
    )[0]

    input_gpu = ort.OrtValue.ortvalue_from_numpy(
        tensor,
        "cuda",
        0,
    )

    io_binding = session.io_binding()
    io_binding.bind_ortvalue_input(input_name, input_gpu)
    io_binding.bind_output(output_name, "cuda", 0)

    session.run_with_iobinding(io_binding)
    io_binding.synchronize_outputs()

    output_ortvalues = io_binding.get_outputs()
    output_gpu = output_ortvalues[0]
    output_cpu = io_binding.copy_outputs_to_cpu()[0]

    np.testing.assert_allclose(
        reference_output,
        output_cpu,
        rtol=1e-4,
        atol=1e-4,
    )

    reference_boxes, _, _ = postprocess(
        reference_output,
        original_shape,
        scale,
        pad,
    )
    bound_boxes, _, _ = postprocess(
        output_cpu,
        original_shape,
        scale,
        pad,
    )

    print("onnxruntime:", ort.__version__)
    print("available providers:", available_providers)
    print("session providers:", session.get_providers())
    print("input name:", input_name)
    print("output name:", output_name)
    print("input device:", input_gpu.device_name())
    print("output device:", output_gpu.device_name())
    print("output shape:", output_gpu.shape())
    print("reference detections:", len(reference_boxes))
    print("iobinding detections:", len(bound_boxes))
    print("max absolute error:", float(np.max(np.abs(reference_output - output_cpu))))
    print("I/O Binding validation: PASS")


if __name__ == "__main__":
    main()
```

运行：

```bash
python scripts/io_binding_check.py
```

必须看到：

```text
input device: cuda
output device: cuda
reference detections: 5
iobinding detections: 5
I/O Binding validation: PASS
```

解释：

- `OrtValue.ortvalue_from_numpy(..., "cuda", 0)`：创建位于 0 号 CUDA 设备的 OrtValue。
- `bind_ortvalue_input`：把已经在 GPU 上的输入绑定给模型。
- `bind_output(..., "cuda", 0)`：要求 ORT 在 GPU 上分配输出。
- `run_with_iobinding`：按照绑定关系执行推理。
- `get_outputs()`：取得设备端 OrtValue，不自动变成 NumPy。
- `copy_outputs_to_cpu()`：需要 Python/NumPy 后处理时，再显式下载输出。

## 必做 4：完成三路径 benchmark

新建：

```text
scripts/benchmark_iobinding.py
```

写入完整代码：

```python
import argparse
from pathlib import Path
from time import perf_counter

import numpy as np
import onnxruntime as ort

from postprocess import postprocess
from preprocess import preprocess


MODEL_PATH = Path("models/yolo11n.onnx")
IMAGE_PATH = Path("images/bus.jpg")


def parse_args():
    parser = argparse.ArgumentParser(
        description="Compare ONNX Runtime CUDA I/O paths."
    )
    parser.add_argument("--imgsz", type=int, default=640)
    parser.add_argument("--warmup", type=int, default=10)
    parser.add_argument("--runs", type=int, default=100)
    return parser.parse_args()


def summarize(values_ms):
    values = np.asarray(values_ms, dtype=np.float64)
    mean_ms = float(values.mean())

    return {
        "mean_ms": mean_ms,
        "p50_ms": float(np.percentile(values, 50)),
        "p95_ms": float(np.percentile(values, 95)),
        "min_ms": float(values.min()),
        "max_ms": float(values.max()),
        "fps": 1000.0 / mean_ms,
    }


def benchmark(name, run_once, warmup, runs):
    for _ in range(warmup):
        run_once()

    latencies_ms = []
    for _ in range(runs):
        start = perf_counter()
        run_once()
        latencies_ms.append((perf_counter() - start) * 1000.0)

    return name, summarize(latencies_ms)


def format_stats(name, stats):
    return (
        f"{name:<23} "
        f"mean={stats['mean_ms']:8.3f} ms  "
        f"P50={stats['p50_ms']:8.3f} ms  "
        f"P95={stats['p95_ms']:8.3f} ms  "
        f"min={stats['min_ms']:8.3f} ms  "
        f"max={stats['max_ms']:8.3f} ms  "
        f"FPS={stats['fps']:7.2f}"
    )


def main():
    args = parse_args()

    if args.imgsz <= 0 or args.imgsz % 32 != 0:
        raise ValueError("imgsz must be a positive multiple of 32")
    if args.warmup < 0 or args.runs <= 0:
        raise ValueError("warmup must be >= 0 and runs must be > 0")

    ort.preload_dlls(directory="")
    if "CUDAExecutionProvider" not in ort.get_available_providers():
        raise RuntimeError("CUDAExecutionProvider is unavailable")

    session = ort.InferenceSession(
        str(MODEL_PATH),
        providers=[
            (
                "CUDAExecutionProvider",
                {
                    "device_id": 0,
                    "cudnn_conv_algo_search": "EXHAUSTIVE",
                },
            ),
            "CPUExecutionProvider",
        ],
    )

    input_name = session.get_inputs()[0].name
    output_name = session.get_outputs()[0].name
    tensor, original_shape, scale, pad = preprocess(
        IMAGE_PATH,
        input_shape=(args.imgsz, args.imgsz),
    )

    reference_output = session.run(
        [output_name],
        {input_name: tensor},
    )[0]

    # Path B: bind CPU input, run on CUDA, then copy output to CPU.
    cpu_io = session.io_binding()
    cpu_io.bind_cpu_input(input_name, tensor)
    cpu_io.bind_output(output_name, "cuda", 0)

    def run_iobinding_cpu_roundtrip():
        session.run_with_iobinding(cpu_io)
        return cpu_io.copy_outputs_to_cpu()[0]

    # Path C: upload input once, preallocate output once, keep both on CUDA.
    input_gpu = ort.OrtValue.ortvalue_from_numpy(
        tensor,
        "cuda",
        0,
    )
    output_gpu = ort.OrtValue.ortvalue_from_shape_and_type(
        reference_output.shape,
        np.float32,
        "cuda",
        0,
    )

    device_io = session.io_binding()
    device_io.bind_ortvalue_input(input_name, input_gpu)
    device_io.bind_ortvalue_output(output_name, output_gpu)

    def run_iobinding_device_only():
        session.run_with_iobinding(device_io)
        device_io.synchronize_outputs()

    results = []
    results.append(
        benchmark(
            "session.run numpy",
            lambda: session.run([output_name], {input_name: tensor})[0],
            args.warmup,
            args.runs,
        )
    )
    results.append(
        benchmark(
            "iobinding cpu roundtrip",
            run_iobinding_cpu_roundtrip,
            args.warmup,
            args.runs,
        )
    )
    results.append(
        benchmark(
            "iobinding device only",
            run_iobinding_device_only,
            args.warmup,
            args.runs,
        )
    )

    cpu_bound_output = run_iobinding_cpu_roundtrip()
    run_iobinding_device_only()
    device_output = output_gpu.numpy()

    np.testing.assert_allclose(
        reference_output,
        cpu_bound_output,
        rtol=1e-4,
        atol=1e-4,
    )
    np.testing.assert_allclose(
        reference_output,
        device_output,
        rtol=1e-4,
        atol=1e-4,
    )

    detection_counts = []
    for output in (reference_output, cpu_bound_output, device_output):
        boxes, _, _ = postprocess(
            output,
            original_shape,
            scale,
            pad,
        )
        detection_counts.append(len(boxes))

    stats_by_name = dict(results)
    normal_mean = stats_by_name["session.run numpy"]["mean_ms"]
    device_mean = stats_by_name["iobinding device only"]["mean_ms"]
    device_speedup = normal_mean / device_mean

    lines = [
        "YOLO11n ONNX Runtime CUDA I/O Binding Benchmark",
        f"onnxruntime: {ort.__version__}",
        f"providers: {session.get_providers()}",
        f"model: {MODEL_PATH}",
        f"image: {IMAGE_PATH}",
        f"imgsz: {args.imgsz}",
        f"warmup runs: {args.warmup}",
        f"test runs: {args.runs}",
        f"input device: {input_gpu.device_name()}",
        f"output device: {output_gpu.device_name()}",
        f"detections: {detection_counts}",
        "",
    ]

    for name, stats in results:
        lines.append(format_stats(name, stats))

    lines.extend(
        [
            "",
            f"device-only speedup vs session.run: {device_speedup:.2f}x",
            "Output validation: PASS",
            "",
            "Note: device-only excludes the initial H2D upload and final D2H download.",
        ]
    )

    report = "\n".join(lines)
    print(report)

    output_path = Path(
        f"outputs/benchmark_iobinding_{args.imgsz}.txt"
    )
    output_path.parent.mkdir(parents=True, exist_ok=True)
    output_path.write_text(report + "\n", encoding="utf-8")
    print(f"saved: {output_path}")


if __name__ == "__main__":
    main()
```

运行正式测试：

```bash
python scripts/benchmark_iobinding.py --imgsz 640 --warmup 10 --runs 100
cat outputs/benchmark_iobinding_640.txt
```

建议连续运行三次：

```bash
python scripts/benchmark_iobinding.py --imgsz 640 --warmup 10 --runs 100
python scripts/benchmark_iobinding.py --imgsz 640 --warmup 10 --runs 100
python scripts/benchmark_iobinding.py --imgsz 640 --warmup 10 --runs 100
```

不要只挑最快的一次。记录三次结果的范围，并说明系统后台负载会造成波动。

## 必做 5：填写性能对比与结论

| 路径 | mean | P50 | P95 | FPS | 包含 H2D | 包含 D2H |
| --- | ---: | ---: | ---: | ---: | --- | --- |
| `session.run numpy` | 待填写 | 待填写 | 待填写 | 待填写 | 是 | 是 |
| `iobinding cpu roundtrip` | 待填写 | 待填写 | 待填写 | 待填写 | 是 | 是 |
| `iobinding device only` | 待填写 | 待填写 | 待填写 | 待填写 | 否，循环外上传 | 否，输出留在 GPU |

回答以下问题：

1. 为什么 `iobinding cpu roundtrip` 不一定比普通 `session.run()` 快？
2. `bind_cpu_input` 和 `bind_ortvalue_input` 的数据位置有什么区别？
3. 为什么设备端路径更适合“GPU 前处理 -> 模型 1 -> 模型 2 -> GPU 后处理”的 pipeline？
4. 今天的 device-only FPS 能否直接写成摄像头系统 FPS？为什么？

预期结论：

- I/O Binding 的核心价值是控制数据位置、减少不必要的数据复制，而不是调用一个新 API 就必然加速。
- 小模型的计算时间很短，数据复制和 Python 调度占比会更加明显。
- 如果每帧最终都立即下载到 CPU 做 NumPy 后处理，I/O Binding 的收益可能很小。
- TensorRT Python API 也会涉及设备内存、输入输出绑定和同步，今天是 Day15-Day17 的直接前置。

## 必做 6：Kaggle baseline 准备 30 分钟

打开 `Digit Recognizer` 页面，只完成信息确认，不在今天展开长时间训练：

```text
任务：MNIST 0-9 图像多分类
训练数据：train.csv
测试数据：test.csv
提交模板：sample_submission.csv
评价指标：classification accuracy
第一次有效提交目标：Day22-Day25 前
```

在 `career/competition-shortlist-2026-summer.md` 中保留该计划即可。Day14 的主线仍然是 I/O Binding。

## 常见错误

### 1. `input device` 或 `output device` 不是 cuda

确认创建 OrtValue 和绑定输出时都写了：

```python
"cuda", 0
```

并确认：

```text
session providers: ['CUDAExecutionProvider', 'CPUExecutionProvider']
```

### 2. `CUDAExecutionProvider is unavailable`

确认环境：

```bash
conda activate ortgpu310
which python
python -m pip show onnxruntime-gpu
```

不要在 `base` 或 `deploy310` 中重复安装。

### 3. 输出结果不一致

先检查三条路径是否使用相同的：

- 模型文件。
- 输入 tensor。
- 输入尺寸。
- Provider 顺序。
- 输出名称。

不要删除 `np.testing.assert_allclose` 来掩盖错误。

### 4. device-only 快得离谱

它没有包含首次输入上传、最终输出下载、OpenCV 预处理和 NMS。它代表设备驻留 pipeline 中的推理段，不代表完整应用。

### 5. I/O Binding 反而更慢

可能原因：

- 每轮都重新创建 OrtValue 或 binding。
- 每轮仍调用 `copy_outputs_to_cpu()`。
- 模型太小，API 调度和同步占比明显。
- WSL、GPU 调度或后台程序造成波动。
- warmup 不充分。

## 今日产出清单

```text
scripts/io_binding_check.py
scripts/benchmark_iobinding.py
outputs/benchmark_iobinding_640.txt
Day14 三路径性能对比表
Digit Recognizer baseline 信息记录
```

## 今日完成情况

- [ ] 环境与 CUDA Provider 检查通过。
- [ ] I/O Binding 输入输出设备检查通过。
- [ ] 普通输出和绑定输出数值一致。
- [ ] 完成三路径 benchmark。
- [ ] 连续运行三次并分析波动。
- [ ] 回答四个分析问题。
- [ ] 完成 Kaggle baseline 的 30 分钟准备。

## 实际测试结果

完成后填写。

## 遇到的问题

完成后填写。

## 今日复盘

完成后填写。

## 明日计划

- Day15 进入 TensorRT 架构、engine、builder、parser 和 optimization profile。
- 检查 PC/WSL 中的 TensorRT 与 `trtexec` 环境。
- 继续沿用 Day11-Day14 的统一 benchmark 方法。
