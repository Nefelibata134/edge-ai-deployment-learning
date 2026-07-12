# Day 13 学习记录

日期：2026-07-12

## 今日目标

- 理解 ONNX Runtime Execution Provider 的作用和 CPU / CUDA fallback 机制。
- 在不破坏 `deploy310` 的前提下创建独立 GPU 环境 `ortgpu310`。
- 安装与当前驱动兼容的稳定版 ONNX Runtime GPU、CUDA 12 和 cuDNN 运行库。
- 验证 RTX 4070 能被 `CUDAExecutionProvider` 使用。
- 使用同一脚本公平对比 ONNX Runtime CPU 与 CUDA 的端到端性能。
- 开始 Day13-Day15 的个人视觉竞赛筛选，核对可单人参赛、截止时间和提交入口。

今天不需要 Jetson。

## 今日 7 小时学习计划

| 时间 | 内容 | 目标产出 |
| --- | --- | --- |
| 第 1 小时 | 理解 CUDA Execution Provider 和版本兼容 | 看懂驱动、CUDA runtime、cuDNN、ORT 的关系 |
| 第 2 小时 | 创建隔离 GPU 环境 | 建立 `ortgpu310`，保留 `deploy310` 不动 |
| 第 3 小时 | 安装并验证 ORT GPU | 出现 `CUDAExecutionProvider` |
| 第 4 小时 | 编写统一 provider benchmark | 同一个脚本支持 CPU / CUDA |
| 第 5 小时 | 运行 CPU 与 CUDA 对比 | 得到延迟、P50、P95、FPS 和检测数 |
| 第 6 小时 | 个人竞赛初筛 | 核对候选赛题规则、截止时间和单人资格 |
| 第 7 小时 | 排错、复盘与记录 | 保存报告并更新 Day13 日志 |

## 当前环境审计

已经确认：

```text
GPU: NVIDIA GeForce RTX 4070 Laptop GPU, 8188 MiB
Windows / WSL driver: 610.43.02 / 610.47
driver reported CUDA UMD: 13.3
deploy310 Python: 3.10.20
PyTorch: 2.12.1+cu130
PyTorch CUDA: 13.0
PyTorch CUDA available: True
PyTorch cuDNN package: cuDNN 9.x for CUDA 13
ONNX Runtime CPU: 1.23.2
ONNX Runtime GPU: not installed
```

`nvidia-smi` 显示的 CUDA 13.3 表示当前驱动最高支持的 CUDA 版本，不等于每个 Python 环境都必须安装 CUDA 13.3。

ONNX Runtime 官方说明：

- PyPI 的稳定版 `onnxruntime-gpu` 默认使用 CUDA 12.x。
- CUDA 13.x 当前使用官方 nightly 源。
- ONNX Runtime 与运行库需要匹配 CUDA 和 cuDNN 的主版本。
- `onnxruntime-gpu[cuda,cudnn]` 可以把所需 CUDA / cuDNN 运行库作为 Python 依赖安装。

官方资料：

- [ONNX Runtime 安装说明](https://onnxruntime.ai/docs/install/)
- [CUDA Execution Provider](https://onnxruntime.ai/docs/execution-providers/CUDA-ExecutionProvider.html)

今天选择稳定 CUDA 12 方案，不使用 CUDA 13 nightly：

```text
Windows NVIDIA driver
  -> 向下兼容 CUDA 12 应用
ortgpu310
  -> onnxruntime-gpu 1.23.2
  -> CUDA 12 runtime from Python packages
  -> cuDNN 9 from Python packages
```

这样不会修改 `deploy310` 中现有的 PyTorch CUDA 13 环境。

## 必做 1：确认原 CPU 环境仍然可用

在 WSL 中执行：

```bash
conda activate deploy310
cd ~/model-deploy-day09
python scripts/ort_check.py
```

应继续看到：

```text
onnxruntime: 1.23.2
available providers:
- AzureExecutionProvider
- CPUExecutionProvider
```

不要在 `deploy310` 中执行 `pip uninstall onnxruntime` 或安装 `onnxruntime-gpu`。

## 必做 2：创建独立 GPU 环境

先退出 VS Code 中可能正在运行的 Python 程序，然后在 WSL 终端执行：

```bash
conda deactivate
conda create -n ortgpu310 python=3.10 pip -y
conda activate ortgpu310
```

确认：

```bash
which python
python --version
python -m pip --version
```

预期路径：

```text
/home/nefelibata/miniconda3/envs/ortgpu310/bin/python
```

安装项目最小依赖：

```bash
python -m pip install --upgrade pip
python -m pip install numpy==2.2.6 opencv-python-headless==5.0.0.93
python -m pip install "onnxruntime-gpu[cuda,cudnn]==1.23.2"
```

最后一条会下载较大的 NVIDIA 运行库，等待过程中不要关闭终端。

安装完成后检查：

```bash
python -m pip show onnxruntime onnxruntime-gpu
python -m pip list | grep -E "onnxruntime|nvidia|numpy|opencv"
```

预期：

- `onnxruntime-gpu` 存在。
- `onnxruntime` CPU 独立包不存在。
- 列表中包含 CUDA 12 / cuDNN 相关 NVIDIA Python 包。

## 必做 3：编写 CUDA Provider 检查脚本

在 VS Code 中打开原练习目录：

```bash
cd ~/model-deploy-day09
code .
```

选择解释器：

```text
/home/nefelibata/miniconda3/envs/ortgpu310/bin/python
```

新建：

```text
scripts/ort_gpu_check.py
```

写入完整代码：

```python
from pathlib import Path

import onnxruntime as ort

from preprocess import preprocess


MODEL_PATH = Path("models/yolo11n.onnx")
IMAGE_PATH = Path("images/bus.jpg")


def main():
    ort.preload_dlls(directory="")

    available_providers = ort.get_available_providers()

    print("onnxruntime:", ort.__version__)
    print("available providers:")
    for provider in available_providers:
        print("-", provider)

    if "CUDAExecutionProvider" not in available_providers:
        raise RuntimeError(
            "CUDAExecutionProvider is unavailable. "
            "Check the GPU package and CUDA/cuDNN runtime libraries."
        )

    providers = [
        (
            "CUDAExecutionProvider",
            {
                "device_id": 0,
                "cudnn_conv_algo_search": "EXHAUSTIVE",
            },
        ),
        "CPUExecutionProvider",
    ]

    session = ort.InferenceSession(
        str(MODEL_PATH),
        providers=providers,
    )

    input_name = session.get_inputs()[0].name
    output_name = session.get_outputs()[0].name
    tensor, _, _, _ = preprocess(IMAGE_PATH)

    output = session.run([output_name], {input_name: tensor})[0]

    print("\nsession providers:", session.get_providers())
    print("input:", input_name, tensor.shape, tensor.dtype)
    print("output:", output_name, output.shape, output.dtype)
    print("output min/max:", float(output.min()), float(output.max()))
    print("CUDA provider check: PASS")


if __name__ == "__main__":
    main()
```

运行：

```bash
conda activate ortgpu310
cd ~/model-deploy-day09
python scripts/ort_gpu_check.py
```

预期关键输出：

```text
available providers:
- CUDAExecutionProvider
- CPUExecutionProvider

session providers: ['CUDAExecutionProvider', 'CPUExecutionProvider']
input: images (1, 3, 640, 640) float32
output: output0 (1, 84, 8400) float32
CUDA provider check: PASS
```

`CPUExecutionProvider` 仍然出现在第二位是正常的，它负责 CUDA Provider 不支持的算子 fallback。第一位必须是 `CUDAExecutionProvider`。

## 必做 4：编写统一 CPU / CUDA benchmark

新建：

```text
scripts/benchmark_provider.py
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
        description="Benchmark YOLO11n with ONNX Runtime providers."
    )
    parser.add_argument(
        "--provider",
        choices=["cpu", "cuda"],
        required=True,
    )
    parser.add_argument("--imgsz", type=int, default=640)
    parser.add_argument("--warmup", type=int, default=10)
    parser.add_argument("--runs", type=int, default=100)
    return parser.parse_args()


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


def create_session(provider_name):
    if provider_name == "cuda":
        ort.preload_dlls(directory="")

        if "CUDAExecutionProvider" not in ort.get_available_providers():
            raise RuntimeError("CUDAExecutionProvider is unavailable")

        providers = [
            (
                "CUDAExecutionProvider",
                {
                    "device_id": 0,
                    "cudnn_conv_algo_search": "EXHAUSTIVE",
                },
            ),
            "CPUExecutionProvider",
        ]
    else:
        providers = ["CPUExecutionProvider"]

    return ort.InferenceSession(
        str(MODEL_PATH),
        providers=providers,
    )


def main():
    args = parse_args()

    if args.imgsz <= 0 or args.imgsz % 32 != 0:
        raise ValueError("imgsz must be a positive multiple of 32")

    if args.warmup < 0 or args.runs <= 0:
        raise ValueError("warmup must be >= 0 and runs must be > 0")

    output_path = Path(
        f"outputs/benchmark_{args.provider}_{args.imgsz}.txt"
    )
    output_path.parent.mkdir(parents=True, exist_ok=True)

    session = create_session(args.provider)
    input_name = session.get_inputs()[0].name
    output_name = session.get_outputs()[0].name
    input_shape = (args.imgsz, args.imgsz)

    warmup_tensor, _, _, _ = preprocess(
        IMAGE_PATH,
        input_shape=input_shape,
    )

    for _ in range(args.warmup):
        session.run([output_name], {input_name: warmup_tensor})

    preprocess_times = []
    inference_times = []
    postprocess_times = []
    total_times = []
    detection_counts = []

    for _ in range(args.runs):
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
        detection_counts.append(len(boxes))

    all_stats = {
        "preprocess": summarize(preprocess_times),
        "inference": summarize(inference_times),
        "postprocess": summarize(postprocess_times),
        "total": summarize(total_times),
    }

    lines = [
        "YOLO11n ONNX Runtime Provider Benchmark",
        f"requested provider: {args.provider}",
        f"session providers: {session.get_providers()}",
        f"onnxruntime: {ort.__version__}",
        f"model: {MODEL_PATH}",
        f"image: {IMAGE_PATH}",
        f"imgsz: {args.imgsz}",
        f"warmup runs: {args.warmup}",
        f"test runs: {args.runs}",
        f"detections: {detection_counts[-1]}",
        "",
    ]

    for name in ("preprocess", "inference", "postprocess", "total"):
        lines.append(format_stats(name, all_stats[name]))

    report = "\n".join(lines)
    print(report)
    output_path.write_text(report + "\n", encoding="utf-8")
    print(f"\nsaved: {output_path}")


if __name__ == "__main__":
    main()
```

## 必做 5：公平比较 CPU 与 CUDA

确保处于同一个 GPU 环境：

```bash
conda activate ortgpu310
cd ~/model-deploy-day09
```

先在 `onnxruntime-gpu` 包自带的 CPU Provider 上测试：

```bash
python scripts/benchmark_provider.py --provider cpu --imgsz 640 --warmup 10 --runs 100
```

再测试 CUDA：

```bash
python scripts/benchmark_provider.py --provider cuda --imgsz 640 --warmup 10 --runs 100
```

查看报告：

```bash
cat outputs/benchmark_cpu_640.txt
cat outputs/benchmark_cuda_640.txt
```

今天的默认 CUDA benchmark 使用 NumPy 输入和 NumPy 输出，因此 inference 时间包含：

```text
CPU tensor
-> Host to Device copy
-> GPU computation
-> Device to Host copy
-> NumPy output
```

这是接近当前 Python 程序的实际延迟。后面学习 I/O Binding 时，会尝试减少 CPU 与 GPU 之间的数据复制。

填写对比表：

| Provider | inference mean | inference P95 | total mean | total P95 | total FPS | detections |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| CPU | 待填写 | 待填写 | 待填写 | 待填写 | 待填写 | 待填写 |
| CUDA | 待填写 | 待填写 | 待填写 | 待填写 | 待填写 | 待填写 |

计算：

```text
inference speedup = CPU inference mean / CUDA inference mean
total speedup = CPU total mean / CUDA total mean
```

如果 CUDA 没有比 CPU 快很多，不要立刻认定 GPU 无效。需要检查：

- 模型是否很小。
- 输入尺寸是否较小。
- Host / Device 数据复制占比。
- warmup 是否充分。
- 是否真的使用 `CUDAExecutionProvider`。
- 是否有算子 fallback 到 CPU。

## 必做 6：开始个人竞赛筛选

打开：

```text
career/competition-shortlist-2026-summer.md
```

今天只做规则核对，不开始重型训练。需要完成：

1. 登录天池和 Kaggle。
2. 检查候选页面是否仍可报名或提交。
3. 确认是否允许个人参赛。
4. 记录截止时间、数据量、评价指标和提交格式。
5. 从候选中保留最多两个，不同时开赛。

优先完成“目标检测练习赛”的规则核对，因为它最容易在 Day22-Day25 前形成第一次有效提交。

## 常见错误

### 1. 安装后只有 CPU Provider

先确认环境：

```bash
conda activate ortgpu310
which python
python -m pip show onnxruntime-gpu
```

再运行：

```bash
python -c "import onnxruntime as ort; ort.preload_dlls(directory=''); print(ort.get_available_providers())"
```

如果仍没有 CUDA，把完整报错发给我，不要在 `deploy310` 里重复安装。

### 2. `libcudnn.so`、`libcublas.so` 或 `libcudart.so` 找不到

确认使用了带 extras 的安装命令：

```bash
python -m pip install "onnxruntime-gpu[cuda,cudnn]==1.23.2"
```

并在创建 session 前执行：

```python
ort.preload_dlls(directory="")
```

### 3. VS Code 又选择了 deploy310

观察右下角 Python 解释器，重新选择：

```text
/home/nefelibata/miniconda3/envs/ortgpu310/bin/python
```

终端中也要确认：

```bash
which python
```

### 4. CUDA session 创建失败后自动回退 CPU

脚本已经检查 `CUDAExecutionProvider`。不要只看程序能否运行，还要看：

```text
session providers: ['CUDAExecutionProvider', 'CPUExecutionProvider']
```

如果 CUDA 不在第一位，不能把结果当作 GPU benchmark。

### 5. GPU 首次运行特别慢

CUDA context、显存分配和 cuDNN 算法搜索会增加首次运行时间，因此必须 warmup。首次 session 创建时间不计入循环 benchmark。

## 今日产出清单

完成后应有：

```text
conda environment: ortgpu310
scripts/ort_gpu_check.py
scripts/benchmark_provider.py
outputs/benchmark_cpu_640.txt
outputs/benchmark_cuda_640.txt
career/competition-shortlist-2026-summer.md 中的规则核对记录
```

## 今日完成情况

- [ ] 确认原 `deploy310` CPU 环境仍然可用。
- [ ] 创建独立 `ortgpu310` 环境。
- [ ] 安装稳定版 ONNX Runtime GPU 及 CUDA 12 / cuDNN 运行库。
- [ ] `CUDAExecutionProvider` 检查通过。
- [ ] 完成统一 CPU / CUDA benchmark。
- [ ] 计算 inference 和 total speedup。
- [ ] 核对个人竞赛候选的报名、单人资格和截止时间。

## 实际测试结果

完成后填写。

## 遇到的问题

完成后填写。

## 今日复盘

完成后填写。

## 明日计划

- Day14 学习 GPU 数据传输与 ONNX Runtime I/O Binding。
- 对比普通 `session.run()` 与设备端绑定后的推理延迟。
- 继续核对个人赛候选，最迟 Day15 确定一个赛题。
- 为 Day15-Day17 的 TensorRT 学习准备统一 benchmark 接口。
