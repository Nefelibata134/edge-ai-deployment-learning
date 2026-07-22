# Day 17 学习记录

日期：2026-07-20

主题：TensorRT Python 端到端推理、动态 shape、统一 benchmark 与 Jetson 接入预检

状态：已完成（2026-07-22 完成 Jetson 接入与传输验证）

## 今日目标

- 把 Day16 的 TensorRT engine 接成真实的 YOLO 图片检测 pipeline。
- 理解 TensorRT 10 中反序列化、Execution Context、动态 shape、显存分配和异步执行的关系。
- 使用同一份预处理和后处理代码，公平比较 ORT CUDA、TRT FP32 和 TRT FP16。
- 区分纯 GPU compute FPS、后端 roundtrip FPS 和端到端 pipeline FPS。
- 验证 FP16 engine 在 `320/640/960` 三个动态输入尺寸下都能运行。
- 用约 1 小时重新接入 Jetson，只做 SSH、环境快照和 ONNX 文件传输，不在板端构建 engine。
- 为 Day18 创建第一个独立项目仓库做好准备。

## 7 小时安排

| 时间 | 内容 | 产出 |
| ---: | --- | --- |
| 0.5 小时 | 复习 Day16 的 engine、纯计算 benchmark 和计时边界 | 能解释 828 FPS 的含义 |
| 1.5 小时 | 阅读 `TensorRTDetector`，理解显存、CUDA Stream 和 `execute_async_v3()` | TensorRT 可复用推理类 |
| 1 小时 | 运行真实图片检测并检查输出图 | `trt_fp16_bus.jpg` |
| 1.5 小时 | 统一 ORT CUDA、TRT FP32、TRT FP16 端到端 benchmark | 三份 JSON/TXT 报告和对比表 |
| 0.75 小时 | 动态 shape 320/640/960 实验与分析 | 分辨率、延迟和 FPS 表 |
| 1 小时 | Jetson SSH、版本、磁盘和 ONNX 传输预检 | Jetson 快照与 SHA256 校验 |
| 0.5 小时 | CSIG 主赛推进或规则/数据/baseline 记录 | 一条可核验竞赛记录 |
| 0.25 小时 | 检查题、日志和 Git 提交 | Day17 完成记录 |

## 今天和 Day16 的区别

Day16 测量的是 engine 的纯 GPU 计算段：

```text
预处理 -> H2D -> [CUDA Event: execute_async_v3] -> D2H -> NMS
                   ^ Day16 主要计时范围
```

Day17 测量的是部署 pipeline 的统一范围：

```text
已解码 BGR 图像
  -> letterbox / RGB / normalize / NCHW
  -> H2D + inference + D2H
  -> confidence filter + NMS
  -> 检测框
```

磁盘读图和画框不进入 benchmark。因为 ORT 和 TensorRT 都使用同一张已解码图像、同一份预处理和同一份后处理，所以三组数据才可以横向比较。

## 开始前：打开正确目录

在 VS Code 的 WSL 窗口中打开：

```bash
cd ~/model-deploy-day09
code .
```

TensorRT 任务使用：

```bash
conda activate trt310
```

ORT CUDA 任务使用：

```bash
conda activate ortgpu310
```

不要在 `base` 环境运行，也不要把两套环境重新混装。

## 必做 1：检查新增文件和已有产物

```bash
conda activate trt310
cd ~/model-deploy-day09

python --version
python -c "import tensorrt as trt; print(trt.__version__)"

ls -lh models/yolo11n.onnx
ls -lh models/yolo11n_fp32.engine models/yolo11n_fp16.engine
ls scripts/trt_detector.py scripts/trt_detect.py
ls scripts/benchmark_e2e.py scripts/compare_e2e_reports.py
```

预期关键版本：

```text
Python 3.10.20
TensorRT 10.9.0.34
```

## 必做 2：阅读可复用 TensorRT 推理类

文件：`~/model-deploy-day09/scripts/trt_detector.py`

完整代码：

```python
from pathlib import Path

import numpy as np
import tensorrt as trt
from cuda.bindings import runtime as cudart


def check_cuda(result, operation):
    error = result[0]
    if error != cudart.cudaError_t.cudaSuccess:
        raise RuntimeError(f"{operation} failed: {error}")

    values = result[1:]
    if len(values) == 1:
        return values[0]
    return values


class TensorRTDetector:
    """Minimal TensorRT 10 runner for one-input, one-output YOLO engines."""

    def __init__(self, engine_path, device_id=0):
        self.engine_path = Path(engine_path)
        if not self.engine_path.exists():
            raise FileNotFoundError(f"engine not found: {self.engine_path}")

        check_cuda(cudart.cudaSetDevice(device_id), "cudaSetDevice")
        self.stream = check_cuda(cudart.cudaStreamCreate(), "cudaStreamCreate")
        self.logger = trt.Logger(trt.Logger.ERROR)
        self.runtime = trt.Runtime(self.logger)
        self.engine = self.runtime.deserialize_cuda_engine(
            self.engine_path.read_bytes()
        )
        if self.engine is None:
            raise RuntimeError(f"failed to deserialize: {self.engine_path}")

        self.context = self.engine.create_execution_context()
        if self.context is None:
            raise RuntimeError("failed to create TensorRT execution context")

        self.input_names = self._tensor_names(trt.TensorIOMode.INPUT)
        self.output_names = self._tensor_names(trt.TensorIOMode.OUTPUT)
        if len(self.input_names) != 1 or len(self.output_names) != 1:
            raise RuntimeError(
                "expected one input and one output, got "
                f"inputs={self.input_names}, outputs={self.output_names}"
            )

        self.input_name = self.input_names[0]
        self.output_name = self.output_names[0]
        self.input_shape = None
        self.host_buffers = {}
        self.device_buffers = {}
        self.closed = False

    def _tensor_names(self, mode):
        names = []
        for index in range(self.engine.num_io_tensors):
            name = self.engine.get_tensor_name(index)
            if self.engine.get_tensor_mode(name) == mode:
                names.append(name)
        return names

    def _free_buffers(self):
        for device_buffer in self.device_buffers.values():
            check_cuda(cudart.cudaFree(device_buffer), "cudaFree")
        self.host_buffers.clear()
        self.device_buffers.clear()

    def configure(self, input_shape):
        input_shape = tuple(int(dim) for dim in input_shape)
        if input_shape == self.input_shape:
            return

        self._free_buffers()
        if not self.context.set_input_shape(self.input_name, input_shape):
            raise RuntimeError(f"failed to set input shape: {input_shape}")

        unresolved = self.context.infer_shapes()
        if unresolved:
            raise RuntimeError(f"unresolved tensor shapes: {unresolved}")

        for index in range(self.engine.num_io_tensors):
            name = self.engine.get_tensor_name(index)
            shape = tuple(int(dim) for dim in self.context.get_tensor_shape(name))
            dtype = np.dtype(trt.nptype(self.engine.get_tensor_dtype(name)))
            host_buffer = np.empty(shape, dtype=dtype)
            device_buffer = check_cuda(
                cudart.cudaMalloc(host_buffer.nbytes),
                f"cudaMalloc({name})",
            )
            self.host_buffers[name] = host_buffer
            self.device_buffers[name] = device_buffer

            if not self.context.set_tensor_address(name, int(device_buffer)):
                raise RuntimeError(f"failed to bind tensor address: {name}")

        self.input_shape = input_shape

    def infer(self, input_tensor):
        if self.closed:
            raise RuntimeError("TensorRTDetector is already closed")

        input_tensor = np.ascontiguousarray(input_tensor)
        self.configure(input_tensor.shape)
        input_buffer = self.host_buffers[self.input_name]
        input_buffer[...] = input_tensor.astype(input_buffer.dtype, copy=False)

        check_cuda(
            cudart.cudaMemcpyAsync(
                int(self.device_buffers[self.input_name]),
                input_buffer.ctypes.data,
                input_buffer.nbytes,
                cudart.cudaMemcpyKind.cudaMemcpyHostToDevice,
                self.stream,
            ),
            "cudaMemcpyAsync(H2D)",
        )

        if not self.context.execute_async_v3(int(self.stream)):
            raise RuntimeError("TensorRT execution failed")

        output_buffer = self.host_buffers[self.output_name]
        check_cuda(
            cudart.cudaMemcpyAsync(
                output_buffer.ctypes.data,
                int(self.device_buffers[self.output_name]),
                output_buffer.nbytes,
                cudart.cudaMemcpyKind.cudaMemcpyDeviceToHost,
                self.stream,
            ),
            "cudaMemcpyAsync(D2H)",
        )
        check_cuda(
            cudart.cudaStreamSynchronize(self.stream),
            "cudaStreamSynchronize(infer)",
        )
        return output_buffer

    def close(self):
        if self.closed:
            return
        self._free_buffers()
        check_cuda(cudart.cudaStreamDestroy(self.stream), "cudaStreamDestroy")
        self.closed = True

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        self.close()
```

阅读重点：

1. `engine` 是优化后的只读执行计划，`context` 保存某次执行的 shape 和地址状态。
2. 动态模型必须先执行 `context.set_input_shape()`，再查询输出 shape。
3. 输入尺寸改变后，输出张量大小也会改变，因此必须重新申请 Host/Device Buffer。
4. `set_tensor_address()` 把张量名和显存地址绑定起来。
5. `cudaMemcpyAsync(H2D)`、`execute_async_v3()`、`cudaMemcpyAsync(D2H)` 被提交到同一条 CUDA Stream。
6. `cudaStreamSynchronize()` 保证 CPU 在读取输出前，GPU 已经完成整条队列。
7. `close()` 必须释放显存和 Stream；长时间服务若持续泄漏显存，会最终崩溃。

## 必做 3：运行真实图片 TensorRT 检测

文件：`scripts/trt_detect.py`

完整代码：

```python
import argparse
from pathlib import Path

import cv2

from postprocess import postprocess
from preprocess import preprocess
from trt_detector import TensorRTDetector


DEMO_CLASS_NAMES = {0: "person", 5: "bus"}


def parse_args():
    parser = argparse.ArgumentParser(description="Run YOLO11 TensorRT detection.")
    parser.add_argument("--engine", type=Path, required=True)
    parser.add_argument("--image", type=Path, default=Path("images/bus.jpg"))
    parser.add_argument("--output", type=Path, required=True)
    parser.add_argument("--imgsz", type=int, default=640)
    parser.add_argument("--conf", type=float, default=0.25)
    parser.add_argument("--iou", type=float, default=0.45)
    return parser.parse_args()


def main():
    args = parse_args()
    image = cv2.imread(str(args.image))
    if image is None:
        raise FileNotFoundError(f"failed to read image: {args.image}")

    tensor, original_shape, scale, pad = preprocess(
        args.image,
        input_shape=(args.imgsz, args.imgsz),
    )

    with TensorRTDetector(args.engine) as detector:
        output = detector.infer(tensor)
        boxes, scores, class_ids = postprocess(
            output,
            original_shape,
            scale,
            pad,
            conf_threshold=args.conf,
            iou_threshold=args.iou,
        )

    print(f"engine: {args.engine}")
    print(f"input shape: {tensor.shape}")
    print(f"output shape: {output.shape}")
    print(f"detections: {len(boxes)}")

    for box, score, class_id in zip(boxes, scores, class_ids):
        x1, y1, x2, y2 = box.round().astype(int)
        class_id = int(class_id)
        label = DEMO_CLASS_NAMES.get(class_id, f"class_{class_id}")
        text = f"{label} {float(score):.2f}"
        cv2.rectangle(image, (x1, y1), (x2, y2), (0, 255, 0), 2)
        cv2.putText(
            image,
            text,
            (x1, max(20, y1 - 8)),
            cv2.FONT_HERSHEY_SIMPLEX,
            0.6,
            (0, 255, 0),
            2,
            cv2.LINE_AA,
        )
        print(
            f"{label:<8} score={float(score):.4f} "
            f"box={[round(float(value), 1) for value in box]}"
        )

    args.output.parent.mkdir(parents=True, exist_ok=True)
    if not cv2.imwrite(str(args.output), image):
        raise RuntimeError(f"failed to save image: {args.output}")
    print(f"saved: {args.output}")


if __name__ == "__main__":
    main()
```

运行：

```bash
conda activate trt310
cd ~/model-deploy-day09

python scripts/trt_detect.py \
  --engine models/yolo11n_fp16.engine \
  --image images/bus.jpg \
  --output outputs/trt_fp16_bus.jpg \
  --imgsz 640
```

预期：

```text
input shape: (1, 3, 640, 640)
output shape: (1, 84, 8400)
detections: 5
1 bus + 4 persons
saved: outputs/trt_fp16_bus.jpg
```

在 VS Code 中打开 `outputs/trt_fp16_bus.jpg`，确认框、类别和置信度位置正常。

## 必做 4：理解统一端到端 benchmark

文件：`scripts/benchmark_e2e.py`

核心执行与计时代码：

```python
def run_once(runner, image_bgr, imgsz):
    start = perf_counter()
    tensor, original_shape, scale, pad = preprocess_image(
        image_bgr,
        input_shape=(imgsz, imgsz),
    )
    after_preprocess = perf_counter()

    output = runner.infer(tensor)
    after_inference = perf_counter()

    boxes, scores, class_ids = postprocess(
        output,
        original_shape,
        scale,
        pad,
    )
    end = perf_counter()

    return {
        "preprocess_ms": (after_preprocess - start) * 1000.0,
        "backend_roundtrip_ms": (after_inference - after_preprocess) * 1000.0,
        "postprocess_ms": (end - after_inference) * 1000.0,
        "total_ms": (end - start) * 1000.0,
        "detections": int(len(boxes)),
        "class_ids": [int(value) for value in class_ids],
        "scores": [float(value) for value in scores],
    }
```

两种 Runner 的关键差异：

```python
class OrtCudaRunner:
    def __init__(self, model_path):
        import onnxruntime as ort

        if hasattr(ort, "preload_dlls"):
            ort.preload_dlls()
        self.session = ort.InferenceSession(
            str(model_path),
            providers=["CUDAExecutionProvider", "CPUExecutionProvider"],
        )
        self.providers = self.session.get_providers()
        if not self.providers or self.providers[0] != "CUDAExecutionProvider":
            raise RuntimeError(f"CUDA provider is not active: {self.providers}")
        self.input_name = self.session.get_inputs()[0].name
        self.output_name = self.session.get_outputs()[0].name

    def infer(self, tensor):
        return self.session.run(
            [self.output_name],
            {self.input_name: tensor},
        )[0]


class TrtRunner:
    def __init__(self, engine_path):
        from trt_detector import TensorRTDetector

        self.detector = TensorRTDetector(engine_path)

    def infer(self, tensor):
        return self.detector.infer(tensor)

    def close(self):
        self.detector.close()
```

完整脚本已经写入 `scripts/benchmark_e2e.py`。先分别在两套隔离环境运行，不要为了“一条命令”把 ORT GPU 和 TensorRT 重新装到同一个环境。

ORT CUDA：

```bash
conda activate ortgpu310
cd ~/model-deploy-day09

python scripts/benchmark_e2e.py \
  --backend ort-cuda \
  --model models/yolo11n.onnx \
  --label ort_cuda_fp32 \
  --image images/bus.jpg \
  --imgsz 640 \
  --warmup 20 \
  --runs 100
```

TensorRT FP32：

```bash
conda activate trt310

python scripts/benchmark_e2e.py \
  --backend trt \
  --model models/yolo11n_fp32.engine \
  --label trt_fp32 \
  --image images/bus.jpg \
  --imgsz 640 \
  --warmup 20 \
  --runs 100
```

TensorRT FP16：

```bash
python scripts/benchmark_e2e.py \
  --backend trt \
  --model models/yolo11n_fp16.engine \
  --label trt_fp16 \
  --image images/bus.jpg \
  --imgsz 640 \
  --warmup 20 \
  --runs 100
```

生成汇总：

```bash
python scripts/compare_e2e_reports.py \
  outputs/e2e_ort_cuda_fp32_640.json \
  outputs/e2e_trt_fp32_640.json \
  outputs/e2e_trt_fp16_640.json
```

## 本机已验证的 640 结果

| 后端 | 模型大小 | Backend roundtrip | Total mean | Total P95 | 端到端 FPS | 相对 ORT |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| ORT CUDA FP32 | 10.30 MiB | 6.780 ms | 9.449 ms | 11.854 ms | 105.84 | 1.00x |
| TRT FP32 | 16.54 MiB | 4.400 ms | 6.758 ms | 8.216 ms | 147.98 | 1.40x |
| TRT FP16 | 8.66 MiB | 3.472 ms | 5.848 ms | 7.203 ms | 171.01 | 1.62x |

三条路径都得到：

```text
detections: 5
class ids: [5, 0, 0, 0, 0]
```

注意：TRT FP16 的 171 FPS 小于 Day16 的 828 FPS，不是退化。171 FPS 包含 CPU 预处理、H2D、GPU 执行、D2H 和 NMS；828 FPS 只统计 engine 的纯 GPU 计算段。

## 必做 5：动态 shape 320/640/960

Day16 构建的 profile：

```text
min = 1x3x320x320
opt = 1x3x640x640
max = 1x3x960x960
```

运行 320：

```bash
python scripts/benchmark_e2e.py \
  --backend trt \
  --model models/yolo11n_fp16.engine \
  --label trt_fp16 \
  --image images/bus.jpg \
  --imgsz 320 \
  --warmup 20 \
  --runs 100
```

运行 960：

```bash
python scripts/benchmark_e2e.py \
  --backend trt \
  --model models/yolo11n_fp16.engine \
  --label trt_fp16 \
  --image images/bus.jpg \
  --imgsz 960 \
  --warmup 20 \
  --runs 100
```

本机已验证结果：

| 输入尺寸 | 检测数 | Total mean | Total P95 | 端到端 FPS |
| ---: | ---: | ---: | ---: | ---: |
| 320 | 5 | 2.814 ms | 3.368 ms | 355.36 |
| 640 | 5 | 5.848 ms | 7.203 ms | 171.01 |
| 960 | 5 | 14.864 ms | 18.809 ms | 67.28 |

结论：

- 输入尺寸增加时，像素数和候选框数量增加，预处理、GPU 推理和后处理都会变慢。
- 320 的速度最高，但不能据单张图片断言精度足够。
- 960 更慢，但对小目标可能更有利，需要完整验证集的 mAP 才能决定项目默认尺寸。
- 320/640/960 都保持 5 个检测，只说明这张测试图稳定，不代表整个数据集都不丢目标。

不要运行 1024 作为正式测试，因为它超过当前 profile 的 max=960；若需要 1024，应重新构建包含该上限的 engine。

## 必做 6：Jetson 重新接入与 ONNX 传输

Windows 已预检：

```text
172.20.10.13:22  reachable
192.168.55.1:22  reachable
```

Ping 不通不代表 SSH 不通；当前两个地址的 22 端口都可以看到。优先使用此前验证稳定的手机热点地址：

```bash
ssh nefelibata@172.20.10.13
```

进入 Jetson 后运行：

```bash
hostname
uname -m
cat /etc/os-release | grep PRETTY_NAME
cat /etc/nv_tegra_release
dpkg -l | grep nvidia-jetpack
dpkg -l | grep tensorrt
python3 --version
df -h /
free -h
sudo nvpmodel -q
mkdir -p ~/model-deploy-day17/models
exit
```

今天不要执行：

```bash
sudo apt upgrade
```

也不要在 Jetson 上安装 PC 的 Conda 环境或复制 RTX 4070 构建的 `.engine`。

回到笔记本 WSL 终端，传输 ONNX：

```bash
cd ~/model-deploy-day09
sha256sum models/yolo11n.onnx

scp models/yolo11n.onnx \
  nefelibata@172.20.10.13:~/model-deploy-day17/models/

ssh nefelibata@172.20.10.13 \
  'ls -lh ~/model-deploy-day17/models/yolo11n.onnx && sha256sum ~/model-deploy-day17/models/yolo11n.onnx'
```

笔记本和 Jetson 输出的 SHA256 必须一致。今天只验证模型能完整传到 Jetson；板端 engine 将在 Jetson 上使用它自己的 TensorRT、JetPack 和 GPU 构建。

如果热点 IP 变化：

- 用手机热点设备列表查看 Jetson 新地址；或
- 插 Type-C 后尝试 `192.168.55.1`；或
- 使用之前验证过的 COM 串口登录后运行 `hostname -I`。

### Jetson 实测结果（2026-07-22）

本次不依赖显示器完成了串口恢复、无线 SSH、SFTP 和 ONNX 传输：

```text
Type-C 串口：COM5，可登录 ttyGS0
无线接口：wlP1p1s0
无线连接：HUAWEI-F1A6MO_5G
Jetson Wi-Fi IP：192.168.3.121/24
笔记本 Wi-Fi IP：192.168.3.44/24
SSH：192.168.3.121:22，连接成功
远程工具：MobaXterm SSH + SFTP
```

Windows 能识别 Type-C 的串口和 `L4T-README` 盘，但 RNDIS/NCM 虚拟网卡未正常启动。因此本次使用 `COM5` 查询无线 IP，再切换到同一局域网内的 Wi-Fi SSH。该路径可在不关闭 `xiuxianyun4` 的情况下工作，因为 `192.168.3.0/24` 走本地 WLAN 路由。

Jetson 环境快照：

```text
架构：aarch64
Ubuntu：22.04.5 LTS
Jetson Linux：R36.4.3
JetPack：6.2.1+b38
TensorRT：10.3.0.30-1+cuda12.5
Python：3.10.12
根分区：233G，总可用约 206G
内存：7.4Gi，总可用约 5.6Gi
功耗模式：25W，Mode ID 1
```

ONNX 从 WSL 传到 Jetson：

```text
源文件：~/model-deploy-day09/models/yolo11n.onnx
目标：~/model-deploy-day17/models/yolo11n.onnx
大小：约 10 MiB
传输速度：4.4 MiB/s
耗时：约 2 秒
SHA256：6b110fcad67c342ad863124f1d1be243b068737f6890f89c46a68e184682b40e
```

WSL 与 Jetson 的 SHA256 完全一致，文件传输验证通过。RTX 4070 上生成的 `.engine` 没有复制到 Jetson；后续将在 Jetson 上使用板端 TensorRT 重新构建 engine。

## 个人竞赛 30 分钟

Kaggle Digit Recognizer 已完成有效提交，不再重复入门赛。今天继续主赛：

```text
第七届 CSIG 复杂工业场景异常检测算法挑战
```

完成下面任意一个可核验产出：

- 记录赛题任务究竟是分类、检测还是分割。
- 记录数据目录、标注格式、类别数和评价指标。
- 下载并阅读 baseline 的入口文件和依赖。
- 估算数据大小和本地/AutoDL 存储需求。
- 成功运行一次数据可视化或 baseline 的最小命令。

不要只写“看了比赛”，至少留下一个文件、命令、截图位置或明确结论。

## 检查题

1. `engine` 和 `execution context` 的职责分别是什么？
2. 为什么动态 shape 改变后要重新查询输出 shape 并重新分配 Buffer？
3. 为什么 H2D、推理和 D2H 要放在同一条 CUDA Stream？
4. Day16 的 828 FPS 和 Day17 的 171 FPS 分别包含什么？
5. 为什么统一 benchmark 要复用同一份预处理和后处理？
6. ORT CUDA FP32、TRT FP32、TRT FP16 的比较中，哪些变量仍然没有完全控制？
7. 为什么 320 上单张图片仍检测到 5 个目标，不能证明 320 与 960 精度相同？
8. 为什么不能把 RTX 4070 上的 TensorRT engine 复制到 Jetson 使用？
9. ONNX 文件传输后为什么要比较 SHA256？

## 常见问题

### 1. `ModuleNotFoundError: No module named 'tensorrt'`

当前不在 `trt310`：

```bash
conda activate trt310
which python
```

### 2. ORT 实际回退到 CPU

必须确认首个 Session Provider：

```text
['CUDAExecutionProvider', 'CPUExecutionProvider']
```

若只有 CPU，不要记录为 GPU benchmark。

### 3. `set_input_shape` 或 `infer_shapes` 失败

检查：

- 输入名是否是 `images`。
- shape 是否为 NCHW：`(1, 3, H, W)`。
- H/W 是否在 profile 的 320 到 960 范围内。
- 尺寸是否为 32 的倍数。

### 4. FPS 比 Day16 低很多

这是正确现象。今天把数据复制、预处理和 NMS 放回了 pipeline，口径更接近应用。

### 5. Jetson SSH 密码输入后没有字符

Linux 输入密码时默认不显示星号，正常输入后按 Enter。

### 6. `scp` 传得很慢

手机热点带宽和信号会影响速度。10.3 MiB ONNX 即使较慢也应可完成；不要中断后假设文件完整，最终以 SHA256 为准。

## 今日产出

WSL 练习目录：

```text
scripts/preprocess.py                  # 新增 preprocess_image，保持旧接口可用
scripts/trt_detector.py                # 可复用 TensorRT 10 推理类
scripts/trt_detect.py                  # 真实图片检测和画框
scripts/benchmark_e2e.py               # ORT/TRT 统一端到端 benchmark
scripts/compare_e2e_reports.py         # JSON 报告对比
outputs/trt_fp16_bus.jpg
outputs/e2e_ort_cuda_fp32_640.txt/json
outputs/e2e_trt_fp32_640.txt/json
outputs/e2e_trt_fp16_320.txt/json
outputs/e2e_trt_fp16_640.txt/json
outputs/e2e_trt_fp16_960.txt/json
```

学习仓库：

```text
logs/day17.md
README.md
```

## 今日完成清单

- [x] Codex 已生成并通过语法检查全部 Day17 脚本。
- [x] Codex 已验证 TRT FP16 真实图片检测并检查结果图。
- [x] Codex 已跑通三后端统一 benchmark 和动态 shape 预实验。
- [x] 用户阅读 `TensorRTDetector` 并能解释核心资源生命周期。
- [x] 用户在 VS Code 终端亲自运行真实图片检测。
- [x] 用户检查 `outputs/trt_fp16_bus.jpg`。
- [x] 用户亲自运行或复核统一 benchmark 输出。
- [x] 用户完成动态 shape 结果分析。
- [x] 用户完成 Jetson 环境快照和 ONNX SHA256 传输验证。
- [x] 用户完成检查题。
- [x] 用户完成一条可核验的 CSIG 推进记录。
- [x] 更新完成状态并提交 GitHub。

## 明日计划

- Day18 创建独立 GitHub 仓库 `industrial-defect-inference-service`。
- 确定工业缺陷数据集、类别、许可和评价指标。
- 只在学习仓库保留 proposal 和索引；正式代码进入独立项目仓库。
- 建立项目目录、环境说明、数据检查脚本和第一版 README。
