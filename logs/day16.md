# Day 16 学习记录

日期：2026-07-19

主题：构建 TensorRT FP32/FP16 Engine 并进行公平性能比较

## 今日目标

- 理解 `trtexec` 与 TensorRT Python Builder API 的关系。
- 使用同一份动态 ONNX 构建 PC 端 FP32 和 FP16 engine。
- 记录构建时间、engine 大小、动态 Profile 和 workspace。
- 使用 CUDA Event 测量纯 GPU 计算延迟，并同时观察主机侧同步延迟。
- 使用同一张真实图片验证 FP32/FP16 的检测级结果一致性。
- 正确解释“原始输出不完全 allclose”和“最终检测结果一致”为什么可以同时成立。
- 为 Day17 的 TensorRT Python 完整推理 pipeline 打好基础。

今天主线仍使用 RTX 4070 + WSL2，不需要 Jetson。Jetson 将在 Day17 用约 1 小时做重新接入预检。

## 7 小时安排

| 时间 | 内容 | 产出 |
| ---: | --- | --- |
| 0.5 小时 | 环境检查、复习 Day15 构建组件 | TensorRT/Python/GPU 信息 |
| 1 小时 | 理解 `trtexec` 与 Builder API、构建和运行的区别 | 命令/API 对照表 |
| 1.5 小时 | 阅读构建脚本，生成 FP32/FP16 engine | 两个 engine 和构建报告 |
| 1.5 小时 | 阅读 benchmark 脚本，完成 20 次 warmup + 100 次测量 | 两份性能报告 |
| 1 小时 | 保存真实图片输出，做 raw output 与检测级一致性验证 | 精度差异记录 |
| 0.5 小时 | 计算加速比、体积缩减并回答检查题 | 结果对比表 |
| 0.5 小时 | CSIG 主赛规则/数据/baseline 推进一步 | 一条竞赛记录 |
| 0.5 小时 | 复盘、日志和 Git 提交 | Day16 完成记录 |

## 开始前：路线选择

当前 `trt310` 环境通过 pip 安装了 TensorRT 10.9 Python API 和运行库，但没有 `trtexec`、Docker 或系统级 TensorRT SDK。

NVIDIA 官方说明中：

- `trtexec` 用于从 ONNX 构建 engine 和进行性能测试。
- pip 安装适合 Python 开发，但完整 CLI/C++ 工具通常来自 tar、deb/rpm 或容器。
- TensorRT 10.x 的标准 Python 构建接口是 `build_serialized_network()`。
- TensorRT 10.x 的标准命名张量执行接口是 `execute_async_v3()`。

因此今天使用与 `trtexec` 等价的官方 Python API 完成实验，避免重复下载数 GB SDK。后续工程需要 C++ 或 CLI 自动化时，再安装完整发行包。

参考：

- [NVIDIA TensorRT 安装方式](https://docs.nvidia.com/deeplearning/tensorrt/latest/installing-tensorrt/installing.html)
- [NVIDIA TensorRT Python API](https://docs.nvidia.com/deeplearning/tensorrt/10.x.x/inference-library/python-api-docs.html)
- [NVIDIA trtexec 性能测试](https://docs.nvidia.com/deeplearning/tensorrt/10.x.x/performance/benchmarking.html)

## 必做 1：确认环境和已有产物

在 VS Code 的 WSL 终端执行：

```bash
conda activate trt310
cd ~/model-deploy-day09

which python
python --version
python -c "import tensorrt as trt; print('TensorRT:', trt.__version__)"
python -c "from cuda.bindings import runtime; print('cuda-python: PASS')"
python -m pip check

ls -lh models/yolo11n.onnx
ls -lh models/yolo11n_fp32.engine models/yolo11n_fp16.engine
cat outputs/trt_build_report.txt
```

预期关键环境：

```text
Python 3.10.20
TensorRT: 10.9.0.34
cuda-python: PASS
```

Day16 已完成一次经过验证的构建，所以你不必为了“看它再等五分钟”而重复构建。应先阅读代码和现有报告；需要练习时可以指定新的输出目录。

## 必做 2：理解 trtexec 与 Python API 的对应关系

如果完整 SDK 中存在 `trtexec`，等价构建命令大致为：

```bash
# FP32
trtexec \
  --onnx=models/yolo11n.onnx \
  --saveEngine=models/yolo11n_fp32.engine \
  --minShapes=images:1x3x320x320 \
  --optShapes=images:1x3x640x640 \
  --maxShapes=images:1x3x960x960 \
  --memPoolSize=workspace:1024 \
  --skipInference

# FP16
trtexec \
  --onnx=models/yolo11n.onnx \
  --saveEngine=models/yolo11n_fp16.engine \
  --minShapes=images:1x3x320x320 \
  --optShapes=images:1x3x640x640 \
  --maxShapes=images:1x3x960x960 \
  --memPoolSize=workspace:1024 \
  --fp16 \
  --skipInference
```

今天本机没有 `trtexec`，上面只用于建立参数映射，不要直接执行。Python 脚本中的对应关系是：

| trtexec 参数 | Python API |
| --- | --- |
| `--onnx` | `OnnxParser.parse()` |
| `--saveEngine` | `build_serialized_network()` + `write_bytes()` |
| `--minShapes/optShapes/maxShapes` | `OptimizationProfile.set_shape()` |
| `--memPoolSize` | `set_memory_pool_limit()` |
| `--fp16` | `config.set_flag(BuilderFlag.FP16)` |
| `--loadEngine` | `Runtime.deserialize_cuda_engine()` |
| 实际推理 | `ExecutionContext.execute_async_v3()` |

## 必做 3：阅读完整构建脚本

文件：

```text
~/model-deploy-day09/scripts/build_trt_engines.py
```

完整代码：

```python
import argparse
from pathlib import Path
from time import perf_counter

import tensorrt as trt


DEFAULT_MODEL = Path("models/yolo11n.onnx")
DEFAULT_OUTPUT_DIR = Path("models")


def parse_args():
    parser = argparse.ArgumentParser(
        description="Build FP32 and FP16 TensorRT engines from ONNX."
    )
    parser.add_argument("--model", type=Path, default=DEFAULT_MODEL)
    parser.add_argument("--output-dir", type=Path, default=DEFAULT_OUTPUT_DIR)
    parser.add_argument(
        "--precision",
        choices=("fp32", "fp16", "both"),
        default="both",
    )
    parser.add_argument("--min-size", type=int, default=320)
    parser.add_argument("--opt-size", type=int, default=640)
    parser.add_argument("--max-size", type=int, default=960)
    parser.add_argument("--workspace-gib", type=float, default=1.0)
    return parser.parse_args()


def shape_tuple(tensor: trt.ITensor) -> tuple[int, ...]:
    return tuple(int(dim) for dim in tensor.shape)


def parse_onnx(model_path: Path, logger: trt.Logger):
    builder = trt.Builder(logger)
    network = builder.create_network(0)
    parser = trt.OnnxParser(network, logger)

    if not parser.parse(model_path.read_bytes()):
        errors = [
            str(parser.get_error(index))
            for index in range(parser.num_errors)
        ]
        raise RuntimeError("ONNX parse failed:\n" + "\n".join(errors))

    return builder, network, parser


def add_dynamic_profile(
    builder: trt.Builder,
    network: trt.INetworkDefinition,
    config: trt.IBuilderConfig,
    min_size: int,
    opt_size: int,
    max_size: int,
) -> bool:
    profile = builder.create_optimization_profile()
    has_dynamic_input = False

    for index in range(network.num_inputs):
        tensor = network.get_input(index)
        if -1 not in shape_tuple(tensor):
            continue

        has_dynamic_input = True
        profile.set_shape(
            tensor.name,
            min=(1, 3, min_size, min_size),
            opt=(1, 3, opt_size, opt_size),
            max=(1, 3, max_size, max_size),
        )

    if has_dynamic_input:
        if not profile:
            raise RuntimeError("optimization profile is invalid")
        profile_index = config.add_optimization_profile(profile)
        if profile_index < 0:
            raise RuntimeError("failed to add optimization profile")

    return has_dynamic_input


def build_one(
    model_path: Path,
    output_path: Path,
    precision: str,
    min_size: int,
    opt_size: int,
    max_size: int,
    workspace_bytes: int,
) -> dict:
    logger = trt.Logger(trt.Logger.ERROR)
    builder, network, parser = parse_onnx(model_path, logger)
    config = builder.create_builder_config()
    config.set_memory_pool_limit(
        trt.MemoryPoolType.WORKSPACE,
        workspace_bytes,
    )

    if precision == "fp16":
        if not builder.platform_has_fast_fp16:
            raise RuntimeError("this GPU does not report fast FP16 support")
        config.set_flag(trt.BuilderFlag.FP16)

    has_profile = add_dynamic_profile(
        builder,
        network,
        config,
        min_size,
        opt_size,
        max_size,
    )

    start = perf_counter()
    serialized_engine = builder.build_serialized_network(network, config)
    build_seconds = perf_counter() - start

    if serialized_engine is None:
        raise RuntimeError(f"failed to build {precision} engine")

    output_path.parent.mkdir(parents=True, exist_ok=True)
    output_path.write_bytes(bytes(serialized_engine))

    result = {
        "precision": precision.upper(),
        "path": str(output_path),
        "size_bytes": output_path.stat().st_size,
        "build_seconds": build_seconds,
        "layers_before_build": network.num_layers,
        "dynamic_profile": has_profile,
    }

    del serialized_engine
    del parser
    del network
    del builder
    return result


def format_result(result: dict) -> str:
    size_mib = result["size_bytes"] / (1024 * 1024)
    return (
        f"{result['precision']:<4} "
        f"build={result['build_seconds']:.3f} s  "
        f"size={size_mib:.2f} MiB  "
        f"layers={result['layers_before_build']}  "
        f"profile={result['dynamic_profile']}  "
        f"path={result['path']}"
    )


def main():
    args = parse_args()

    if not args.model.exists():
        raise FileNotFoundError(f"model not found: {args.model}")
    if not (0 < args.min_size <= args.opt_size <= args.max_size):
        raise ValueError("require 0 < min-size <= opt-size <= max-size")
    if any(size % 32 != 0 for size in (args.min_size, args.opt_size, args.max_size)):
        raise ValueError("all image sizes must be multiples of 32")
    if args.workspace_gib <= 0:
        raise ValueError("workspace-gib must be positive")

    workspace_bytes = int(args.workspace_gib * (1 << 30))
    precisions = (
        ("fp32", "fp16")
        if args.precision == "both"
        else (args.precision,)
    )

    print("TensorRT:", trt.__version__)
    print("model:", args.model)
    print(
        "profile:",
        f"min=1x3x{args.min_size}x{args.min_size}",
        f"opt=1x3x{args.opt_size}x{args.opt_size}",
        f"max=1x3x{args.max_size}x{args.max_size}",
    )
    print("workspace bytes:", workspace_bytes)

    results = []
    for precision in precisions:
        output_path = args.output_dir / f"yolo11n_{precision}.engine"
        print(f"\nbuilding {precision.upper()} engine...")
        result = build_one(
            args.model,
            output_path,
            precision,
            args.min_size,
            args.opt_size,
            args.max_size,
            workspace_bytes,
        )
        results.append(result)
        print(format_result(result))

    report_lines = [
        "YOLO11n TensorRT Engine Build Report",
        f"TensorRT: {trt.__version__}",
        f"model: {args.model}",
        (
            "profile: "
            f"min=1x3x{args.min_size}x{args.min_size}, "
            f"opt=1x3x{args.opt_size}x{args.opt_size}, "
            f"max=1x3x{args.max_size}x{args.max_size}"
        ),
        f"workspace bytes: {workspace_bytes}",
        "",
        *(format_result(result) for result in results),
    ]
    report = "\n".join(report_lines) + "\n"
    report_path = Path("outputs/trt_build_report.txt")
    report_path.parent.mkdir(parents=True, exist_ok=True)
    report_path.write_text(report, encoding="utf-8")
    print(f"\nsaved report: {report_path}")


if __name__ == "__main__":
    main()
```

第一次构建命令：

```bash
python scripts/build_trt_engines.py --precision both
```

如果想练习但不覆盖现有 engine：

```bash
python scripts/build_trt_engines.py \
  --precision both \
  --output-dir models/day16_rebuild
```

注意：

- 动态 Profile 为 `320/640/960`。
- workspace 为 1 GiB。
- FP32 和 FP16 分别构建，不能只给文件改名。
- 构建会进行 tactic 搜索，耗时明显长于反序列化和推理。
- engine 文件只信任自己构建或可信来源提供的产物。

## 必做 4：阅读完整 benchmark 脚本

文件：

```text
~/model-deploy-day09/scripts/benchmark_trt_engine.py
```

完整代码：

```python
import argparse
from pathlib import Path
from time import perf_counter

import numpy as np
import tensorrt as trt
from cuda.bindings import runtime as cudart


def parse_args():
    parser = argparse.ArgumentParser(
        description="Benchmark a TensorRT engine with CUDA events."
    )
    parser.add_argument("--engine", type=Path, required=True)
    parser.add_argument("--label", required=True)
    parser.add_argument("--imgsz", type=int, default=640)
    parser.add_argument("--warmup", type=int, default=20)
    parser.add_argument("--runs", type=int, default=100)
    parser.add_argument("--save-output", type=Path)
    parser.add_argument("--image", type=Path)
    return parser.parse_args()


def check_cuda(result, operation):
    error = result[0]
    if error != cudart.cudaError_t.cudaSuccess:
        raise RuntimeError(f"{operation} failed: {error}")
    values = result[1:]
    if len(values) == 1:
        return values[0]
    return values


def summarize(values_ms):
    values = np.asarray(values_ms, dtype=np.float64)
    mean_ms = float(values.mean())
    return {
        "mean": mean_ms,
        "p50": float(np.percentile(values, 50)),
        "p95": float(np.percentile(values, 95)),
        "min": float(values.min()),
        "max": float(values.max()),
        "fps": 1000.0 / mean_ms,
    }


def format_stats(name, stats):
    return (
        f"{name:<16} "
        f"mean={stats['mean']:8.3f} ms  "
        f"P50={stats['p50']:8.3f} ms  "
        f"P95={stats['p95']:8.3f} ms  "
        f"min={stats['min']:8.3f} ms  "
        f"max={stats['max']:8.3f} ms  "
        f"FPS={stats['fps']:8.2f}"
    )


def tensor_names(engine, mode):
    names = []
    for index in range(engine.num_io_tensors):
        name = engine.get_tensor_name(index)
        if engine.get_tensor_mode(name) == mode:
            names.append(name)
    return names


def main():
    args = parse_args()

    if not args.engine.exists():
        raise FileNotFoundError(f"engine not found: {args.engine}")
    if args.imgsz <= 0 or args.imgsz % 32 != 0:
        raise ValueError("imgsz must be a positive multiple of 32")
    if args.warmup < 0 or args.runs <= 0:
        raise ValueError("warmup must be >= 0 and runs must be > 0")

    logger = trt.Logger(trt.Logger.ERROR)
    runtime = trt.Runtime(logger)
    engine = runtime.deserialize_cuda_engine(args.engine.read_bytes())
    if engine is None:
        raise RuntimeError(f"failed to deserialize: {args.engine}")

    context = engine.create_execution_context()
    if context is None:
        raise RuntimeError("failed to create execution context")

    input_names = tensor_names(engine, trt.TensorIOMode.INPUT)
    output_names = tensor_names(engine, trt.TensorIOMode.OUTPUT)
    if len(input_names) != 1:
        raise RuntimeError(f"expected one input, got {input_names}")

    input_name = input_names[0]
    input_shape = (1, 3, args.imgsz, args.imgsz)
    if not context.set_input_shape(input_name, input_shape):
        raise RuntimeError(f"failed to set input shape: {input_shape}")

    unresolved = context.infer_shapes()
    if unresolved:
        raise RuntimeError(f"unresolved tensor shapes: {unresolved}")

    check_cuda(cudart.cudaSetDevice(0), "cudaSetDevice")
    stream = check_cuda(cudart.cudaStreamCreate(), "cudaStreamCreate")
    start_event = check_cuda(cudart.cudaEventCreate(), "cudaEventCreate(start)")
    end_event = check_cuda(cudart.cudaEventCreate(), "cudaEventCreate(end)")

    device_buffers = {}
    host_buffers = {}

    try:
        for index in range(engine.num_io_tensors):
            name = engine.get_tensor_name(index)
            shape = tuple(int(dim) for dim in context.get_tensor_shape(name))
            dtype = np.dtype(trt.nptype(engine.get_tensor_dtype(name)))
            host = np.empty(shape, dtype=dtype)
            device = check_cuda(
                cudart.cudaMalloc(host.nbytes),
                f"cudaMalloc({name})",
            )
            host_buffers[name] = host
            device_buffers[name] = device

            if not context.set_tensor_address(name, int(device)):
                raise RuntimeError(f"failed to bind tensor address: {name}")

        if args.image is None:
            rng = np.random.default_rng(0)
            input_tensor = rng.random(input_shape, dtype=np.float32)
            input_source = "seeded random tensor"
        else:
            from preprocess import preprocess

            input_tensor, _, _, _ = preprocess(
                args.image,
                input_shape=(args.imgsz, args.imgsz),
            )
            input_source = str(args.image)

        host_buffers[input_name][...] = input_tensor.astype(
            host_buffers[input_name].dtype,
            copy=False,
        )

        check_cuda(
            cudart.cudaMemcpyAsync(
                int(device_buffers[input_name]),
                host_buffers[input_name].ctypes.data,
                host_buffers[input_name].nbytes,
                cudart.cudaMemcpyKind.cudaMemcpyHostToDevice,
                stream,
            ),
            "cudaMemcpyAsync(H2D)",
        )
        check_cuda(
            cudart.cudaStreamSynchronize(stream),
            "cudaStreamSynchronize(H2D)",
        )

        for _ in range(args.warmup):
            if not context.execute_async_v3(int(stream)):
                raise RuntimeError("TensorRT warmup execution failed")
        check_cuda(
            cudart.cudaStreamSynchronize(stream),
            "cudaStreamSynchronize(warmup)",
        )

        gpu_ms = []
        host_ms = []
        for _ in range(args.runs):
            host_start = perf_counter()
            check_cuda(
                cudart.cudaEventRecord(start_event, stream),
                "cudaEventRecord(start)",
            )
            if not context.execute_async_v3(int(stream)):
                raise RuntimeError("TensorRT execution failed")
            check_cuda(
                cudart.cudaEventRecord(end_event, stream),
                "cudaEventRecord(end)",
            )
            check_cuda(
                cudart.cudaEventSynchronize(end_event),
                "cudaEventSynchronize(end)",
            )
            host_ms.append((perf_counter() - host_start) * 1000.0)
            gpu_ms.append(
                float(
                    check_cuda(
                        cudart.cudaEventElapsedTime(start_event, end_event),
                        "cudaEventElapsedTime",
                    )
                )
            )

        for name in output_names:
            check_cuda(
                cudart.cudaMemcpyAsync(
                    host_buffers[name].ctypes.data,
                    int(device_buffers[name]),
                    host_buffers[name].nbytes,
                    cudart.cudaMemcpyKind.cudaMemcpyDeviceToHost,
                    stream,
                ),
                f"cudaMemcpyAsync(D2H:{name})",
            )
        check_cuda(
            cudart.cudaStreamSynchronize(stream),
            "cudaStreamSynchronize(D2H)",
        )

        gpu_stats = summarize(gpu_ms)
        host_stats = summarize(host_ms)
        output_summary = {
            name: {
                "shape": host_buffers[name].shape,
                "min": float(host_buffers[name].min()),
                "max": float(host_buffers[name].max()),
            }
            for name in output_names
        }

        lines = [
            "YOLO11n TensorRT Engine Benchmark",
            f"label: {args.label}",
            f"TensorRT: {trt.__version__}",
            f"engine: {args.engine}",
            f"engine size: {args.engine.stat().st_size / (1024 * 1024):.2f} MiB",
            f"input: {input_name} {input_shape}",
            f"input source: {input_source}",
            f"outputs: {output_summary}",
            f"warmup runs: {args.warmup}",
            f"test runs: {args.runs}",
            "",
            format_stats("GPU compute", gpu_stats),
            format_stats("host observed", host_stats),
            "",
            "Note: input H2D and output D2H are outside the timed loop.",
        ]
        report = "\n".join(lines) + "\n"
        print(report, end="")

        report_path = Path(
            f"outputs/benchmark_trt_{args.label}_{args.imgsz}.txt"
        )
        report_path.parent.mkdir(parents=True, exist_ok=True)
        report_path.write_text(report, encoding="utf-8")
        print(f"saved: {report_path}")

        if args.save_output is not None:
            if len(output_names) != 1:
                raise RuntimeError(
                    "--save-output currently requires exactly one output"
                )
            args.save_output.parent.mkdir(parents=True, exist_ok=True)
            np.save(args.save_output, host_buffers[output_names[0]])
            print(f"saved output: {args.save_output}")

    finally:
        for device in device_buffers.values():
            check_cuda(cudart.cudaFree(device), "cudaFree")
        check_cuda(cudart.cudaEventDestroy(start_event), "cudaEventDestroy(start)")
        check_cuda(cudart.cudaEventDestroy(end_event), "cudaEventDestroy(end)")
        check_cuda(cudart.cudaStreamDestroy(stream), "cudaStreamDestroy")


if __name__ == "__main__":
    main()
```

正式测试命令：

```bash
python scripts/benchmark_trt_engine.py \
  --engine models/yolo11n_fp32.engine \
  --label fp32 \
  --image images/bus.jpg \
  --imgsz 640 \
  --warmup 20 \
  --runs 100 \
  --save-output outputs/trt_fp32_output.npy

python scripts/benchmark_trt_engine.py \
  --engine models/yolo11n_fp16.engine \
  --label fp16 \
  --image images/bus.jpg \
  --imgsz 640 \
  --warmup 20 \
  --runs 100 \
  --save-output outputs/trt_fp16_output.npy
```

计时边界：

```text
OpenCV preprocess -> H2D copy -> [CUDA Event timed execute_async_v3] -> D2H copy -> NMS
                                   ^ only this part is GPU compute
```

- `GPU compute`：CUDA Event 记录的设备流执行时间。
- `host observed`：主机发起执行并等待 Event 完成的时间，包含 Python/CUDA 调度和同步开销。
- H2D、D2H、OpenCV 预处理、NMS 都不在今天的计时循环里。
- 因此今天的 FPS 是 engine 计算段理论能力，不是摄像头端到端 FPS。

## 必做 5：检测级一致性验证

原始输出比较：

```bash
python scripts/compare_trt_outputs.py
```

检测结果比较：

```bash
python scripts/compare_trt_detections.py
```

完整检测比较代码：

```python
import argparse
from pathlib import Path

import numpy as np

from postprocess import postprocess
from preprocess import preprocess


def parse_args():
    parser = argparse.ArgumentParser(
        description="Compare FP32 and FP16 TensorRT detections."
    )
    parser.add_argument(
        "--fp32",
        type=Path,
        default=Path("outputs/trt_fp32_output.npy"),
    )
    parser.add_argument(
        "--fp16",
        type=Path,
        default=Path("outputs/trt_fp16_output.npy"),
    )
    parser.add_argument(
        "--image",
        type=Path,
        default=Path("images/bus.jpg"),
    )
    parser.add_argument("--imgsz", type=int, default=640)
    return parser.parse_args()


def sort_detections(boxes, scores, class_ids):
    order = np.argsort(scores)[::-1]
    return boxes[order], scores[order], class_ids[order]


def print_detections(label, boxes, scores, class_ids):
    print(f"{label} detections: {len(boxes)}")
    for box, score, class_id in zip(boxes, scores, class_ids):
        rounded_box = [round(float(value), 2) for value in box]
        print(
            f"- class={int(class_id):2d} "
            f"score={float(score):.4f} "
            f"box={rounded_box}"
        )


def main():
    args = parse_args()
    fp32_output = np.load(args.fp32)
    fp16_output = np.load(args.fp16)

    _, original_shape, scale, pad = preprocess(
        args.image,
        input_shape=(args.imgsz, args.imgsz),
    )

    fp32 = sort_detections(
        *postprocess(fp32_output, original_shape, scale, pad)
    )
    fp16 = sort_detections(
        *postprocess(fp16_output, original_shape, scale, pad)
    )

    print("image:", args.image)
    print_detections("FP32", *fp32)
    print_detections("FP16", *fp16)

    fp32_boxes, fp32_scores, fp32_classes = fp32
    fp16_boxes, fp16_scores, fp16_classes = fp16

    same_count = len(fp32_boxes) == len(fp16_boxes)
    same_classes = same_count and np.array_equal(
        fp32_classes,
        fp16_classes,
    )

    print("same detection count:", same_count)
    print("same class sequence:", same_classes)

    if same_count and same_classes and len(fp32_boxes) > 0:
        box_difference = np.abs(fp32_boxes - fp16_boxes)
        score_difference = np.abs(fp32_scores - fp16_scores)
        print("max box coordinate difference:", float(box_difference.max()))
        print("mean box coordinate difference:", float(box_difference.mean()))
        print("max score difference:", float(score_difference.max()))
        print("mean score difference:", float(score_difference.mean()))

    if not same_count or not same_classes:
        raise RuntimeError("FP32 and FP16 detection results do not match")

    print("detection-level validation: PASS")


if __name__ == "__main__":
    main()
```

为什么 raw output 的 `allclose=False` 不等于 FP16 失败：

- FP16 改变浮点精度和部分 tactic，不能期望与 FP32 逐元素完全相同。
- YOLO 原始输出含大量低置信度候选框，其中少量坐标差异可能使严格全数组比较失败。
- 更关键的是经过相同置信度阈值、坐标还原和 NMS 后，最终业务结果是否稳定。
- 今天真实图片上 FP32/FP16 都得到 5 个目标、类别顺序一致，检测级验证通过。
- 正式项目还要在完整验证集上比较 mAP，单张图片一致不能代替精度评估。

## 本机实际结果

### 构建结果

```text
FP32 build=78.234 s   size=16.54 MiB
FP16 build=215.718 s  size=8.66 MiB
```

FP16：

- engine 体积减少约 `47.6%`。
- 构建耗时更长，因为搜索了额外的 FP16 tactic。
- 构建慢不代表推理慢，二者是不同阶段。

### 100 次正式 benchmark

| 精度 | Engine | GPU mean | GPU P50 | GPU P95 | 理论 FPS | Host mean |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| FP32 | 16.54 MiB | 2.040 ms | 2.037 ms | 2.213 ms | 490.09 | 2.136 ms |
| FP16 | 8.66 MiB | 1.207 ms | 1.189 ms | 1.276 ms | 828.32 | 1.335 ms |

结论：

```text
FP16 GPU compute speedup = 2.040 / 1.207 = 1.69x
FP16 engine size ratio  = 8.66 / 16.54 = 52.4%
FP16 engine size reduction = 47.6%
```

### 真实图片检测一致性

```text
FP32 detections: 5
FP16 detections: 5
same detection count: True
same class sequence: True
max box coordinate difference: 0.2047 px
mean box coordinate difference: 0.0745 px
max score difference: 0.00273
detection-level validation: PASS
```

Raw output：

```text
close ratio: 0.9996216
allclose: False
```

应同时记录两者，不能删除 `allclose=False` 来美化结果，也不能忽略检测级验证而武断地说 FP16 不可用。

## 必做 6：回答检查题

1. 为什么 TensorRT 构建 engine 比加载 ONNX 或执行单次推理慢很多？
2. FP16 构建时间比 FP32 更长，是否说明 FP16 推理更慢？
3. `GPU compute` 和 `host observed` 分别包含什么？
4. 今天的 828 FPS 为什么不能写成摄像头项目 FPS？
5. 为什么 H2D 和 D2H 被放在计时循环外？
6. 为什么 raw output `allclose=False`，但检测级验证仍可 PASS？
7. 单张图片检测一致是否足以证明 FP16 没有精度损失？
8. 为什么 RTX 4070 的 engine 不能复制到 Jetson 使用？

## 个人竞赛 30 分钟

Kaggle Digit Recognizer 已完成首次有效提交。今天不重复做入门提交，转向主赛候选：

```text
第七届 CSIG 复杂工业场景异常检测算法挑战
```

完成以下任意一项：

- 下载并查看赛题规则。
- 确认是否允许个人参赛及队伍要求。
- 记录数据目录、标注格式和评价指标。
- 找到官方 baseline，并记录运行环境和第一条启动命令。
- 如果数据量较大，只完成下载方案和存储空间评估，不在今天训练。

## 常见问题

### 1. `profile.set_shape()` 被判断为失败

当前 TensorRT 10.9 Python binding 中，`set_shape()` 可能返回 `None`，不能直接用 `if not profile.set_shape(...)` 判断。正确做法是先设置，再检查整个 Profile：

```python
profile.set_shape(...)
if not profile:
    raise RuntimeError("optimization profile is invalid")
```

这是今天实际遇到并修复的问题。

### 2. 构建超过终端等待时间

构建可能仍在 WSL 后台进行。不要重复启动，先检查：

```bash
pgrep -af build_trt_engines.py
nvidia-smi
ls -lh models/*.engine
```

### 3. `trtexec: command not found`

pip 版 TensorRT 不包含完整 CLI。今天使用 Python API 是计划内路线，不需要为了一个命令重新安装整个 SDK。

### 4. engine 反序列化失败

确认：

- Engine 是在当前 RTX 4070、当前 TensorRT 环境中构建。
- 没有把 Jetson engine 拿到 PC 使用。
- 文件构建过程没有中断。
- 使用的是 `trt310` 环境。

### 5. 输出 FPS 高得离谱

今天仅测 engine 计算段。正式应用还包括采集、解码、预处理、H2D、D2H、NMS、画框、网络和队列。

## 今日产出

```text
models/yolo11n_fp32.engine
models/yolo11n_fp16.engine
scripts/build_trt_engines.py
scripts/benchmark_trt_engine.py
scripts/compare_trt_outputs.py
scripts/compare_trt_detections.py
outputs/trt_build_report.txt
outputs/benchmark_trt_fp32_640.txt
outputs/benchmark_trt_fp16_640.txt
outputs/trt_fp32_output.npy
outputs/trt_fp16_output.npy
```

## 今日完成情况

- [x] TensorRT 10.9、CUDA Python 和模型环境检查通过。
- [x] FP32/FP16 engine 已成功构建并验证可反序列化。
- [x] 构建报告、engine 大小和构建时间已生成。
- [x] 两种精度已完成 20 次 warmup + 100 次 CUDA Event 测量。
- [x] 真实图片 FP32/FP16 检测级一致性验证通过。
- [ ] 用户阅读并解释两个核心脚本。
- [ ] 用户完成八道检查题。
- [ ] 完成 30 分钟 CSIG 主赛推进。
- [ ] 完成 Day16 复盘。

## 明日计划

- Day17 使用 TensorRT Python API 组织完整预处理、H2D、推理、D2H、后处理与画框。
- 与 ONNX Runtime CUDA 的统一 pipeline 指标比较，避免混用计时边界。
- 用约 1 小时完成 Jetson SSH、版本快照、磁盘检查和 ONNX 文件传输预检。
- 项目 2 暂不提前建仓，Day18 仍按计划创建项目 1 独立仓库。

