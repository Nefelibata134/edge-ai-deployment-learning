# Day 23 学习记录

日期：2026-07-27

状态：已完成

## 今日目标

- 理解 ONNX、TensorRT network、optimization profile、serialized engine
  和 execution context 之间的关系。
- 使用冻结的 ONNX 模型分别构建严格 FP32 和 FP16 TensorRT engine。
- 使用相同真实输入验证 ONNX Runtime、TensorRT FP32 和 TensorRT FP16
  的 logits 与二值掩码一致性。
- 使用统一 warmup、测试次数和 batch 进行延迟、吞吐量、显存及制品大小对比。
- 为 Day24 Triton model repository 冻结 TensorRT engine 契约和版本信息。

## 七小时时间分配

| 模块 | 时间 | 验收产出 |
| --- | ---: | --- |
| TensorRT 概念与环境 | 1 小时 | 能解释 parser、builder、profile、engine、context |
| FP32/FP16 engine 构建 | 1.5 小时 | 两个 plan 文件、构建配置与哈希 |
| TensorRT 推理数据流 | 1.5 小时 | GPU buffer、CUDA stream、动态 batch 推理 |
| 正确性与性能验证 | 1.5 小时 | ORT/TRT parity 和统一 benchmark 报告 |
| 修改任务与面试追问 | 0.5 小时 | 亲自改变 batch 或 profile 并解释影响 |
| 文档与竞赛并行任务 | 1 小时 | 正式 benchmark 文档与一次低成本竞赛推进 |

## 当前环境基线

```text
host: WSL2 Ubuntu 22.04
GPU: NVIDIA GeForce RTX 4070 Laptop GPU
GPU memory: 8188 MiB
TensorRT environment: trt310
Python: 3.10
TensorRT: 10.9.0.34, CUDA 12 variant
CUDA Runtime: 12.9
ONNX opset: 18
```

当前 TensorRT 来自 NVIDIA 官方 pip Full Runtime，包含 builder、runtime 和
Python API，但不包含 C++ headers 与 `trtexec`。项目 1 是 Python 服务端项目，
今天使用 TensorRT Python API完成可复现构建和推理；`trtexec` 后续通过匹配
版本的 Debian/Tar 或 Triton 容器补充对照，不混装不匹配的系统库。

## 冻结的构建输入

```text
ONNX:
models/unet_resnet18_severstal.onnx

ONNX SHA256:
6622d3aada8f7fe0547c475f80a8697cb74f7e6e8c6b677a9dbaff45a4f03a0c

input:
images, float32, [batch, 3, 256, 1024]

output:
logits, float32, [batch, 4, 256, 1024]

optimization profile:
min batch = 1
opt batch = 4
max batch = 8
```

FP32 engine 将显式关闭 TF32，以获得严格 FP32 对照。FP16 engine 启用
`BuilderFlag.FP16`，不支持 FP16 的层允许回退到 FP32。

## 核心数据流

```text
ONNX bytes
-> TensorRT ONNX parser
-> TensorRT network definition
-> builder config + optimization profile + precision
-> tactic profiling and kernel selection
-> serialized engine / plan
-> runtime.deserialize_cuda_engine()
-> execution context
-> set_input_shape() + set_tensor_address()
-> host-to-device copy
-> execute_async_v3()
-> device-to-host copy
-> logits
```

## 今日验收标准

- FP32 和 FP16 engine 均成功构建并能反序列化。
- 两个 engine 都接受 batch `1/2/4/8`，拒绝 profile 范围外的 batch。
- TensorRT 输出形状符合 `[batch, 4, 256, 1024]`。
- FP32 与 ONNX Runtime 数值误差满足严格阈值。
- FP16 阈值后二值掩码差异保持在可接受范围。
- benchmark 区分 GPU 推理延迟和端到端传输延迟。
- 记录 warmup、测试次数、精度、batch、GPU、TensorRT/CUDA 版本和制品哈希。

## 动手记录

### 1. 构建 TensorRT engine

- 使用 NVIDIA 官方 TensorRT 10.9 Python API 解析冻结 ONNX。
- 构建严格 FP32 engine：关闭 TF32。
- 构建 FP16 engine：启用 `BuilderFlag.FP16`。
- 动态 batch profile 固定为：

```text
min = [1, 3, 256, 1024]
opt = [4, 3, 256, 1024]
max = [8, 3, 256, 1024]
```

构建结果：

| 精度 | 构建时间 | engine 大小 | execution context 显存 |
| --- | ---: | ---: | ---: |
| FP32 | 12.55 s | 94.40 MiB | 524 MiB |
| FP16 | 29.25 s | 30.04 MiB | 262 MiB |

### 2. 实现 TensorRT 动态 batch 推理

运行时完成了以下步骤：

1. 反序列化 engine。
2. 创建 execution context 和 CUDA stream。
3. 根据实际 batch 调用 `set_input_shape()`。
4. 为输入与输出申请 GPU buffer。
5. 绑定 tensor address。
6. 执行异步 H2D、`execute_async_v3()` 和 D2H。
7. 同步 stream 并返回 CPU float32 logits。

亲自修改和验证：

```text
batch=2: PASS, output=(2, 4, 256, 1024)
batch=9: REJECTED
valid profile: batch 1..8
```

结论：若需要支持 batch 16，必须修改 max profile 后重新构建 engine，
不能只修改推理代码。

### 3. 正确性验证

模型输出是 logits，而项目阈值 `0.8` 是概率阈值，因此先换算为：

```text
logit threshold = log(0.8 / 0.2) = 1.386294
```

验证结果：

- TensorRT FP32 对 ONNX Runtime：二值掩码 mismatch 为 `0`。
- TensorRT FP32 最大绝对误差不超过 `3.81e-5`。
- TensorRT FP16 二值掩码 mismatch 约为 `1.1e-5` 到 `1.3e-5`。
- FP16 最大绝对误差不超过 `1.31`，平均绝对误差约 `0.021` 到 `0.023`。
- FP32、FP16 均通过项目声明的验收阈值。

### 4. 统一 benchmark

统一口径：

- 输入边界：CPU float32 tensor。
- 输出边界：CPU float32 logits。
- 不包含图片解码、resize、归一化和后处理。
- warmup `50` 次，正式测量 `500` 次。
- batch：`1/4/8`。

| Backend | B1 mean / P95 | B4 mean / P95 | B8 mean / P95 | B8 throughput |
| --- | ---: | ---: | ---: | ---: |
| PyTorch CUDA | 7.02 / 7.48 ms | 33.16 / 36.87 ms | 68.13 / 74.47 ms | 117.4 img/s |
| ONNX Runtime CUDA | 8.23 / 9.13 ms | 33.89 / 36.66 ms | 70.96 / 76.11 ms | 112.7 img/s |
| TensorRT FP32 | 5.18 / 5.63 ms | 20.58 / 22.12 ms | 44.88 / 49.51 ms | 178.2 img/s |
| TensorRT FP16 | 3.22 / 3.45 ms | 10.31 / 11.41 ms | 24.84 / 29.93 ms | 322.1 img/s |

在本机 batch 8 下，TensorRT FP16 吞吐量约为 PyTorch CUDA 的
`2.74` 倍、TensorRT FP32 的 `1.81` 倍。

PyTorch 使用 CUDA 13，ONNX Runtime 与 TensorRT 使用独立 CUDA 12
环境，因此跨框架数字是工程对照，不是完全控制变量实验；TensorRT
FP32/FP16 才是同一运行时下的直接精度对照。

### 5. 竞赛并行推进

- 核实 Kaggle Digit Recognizer 基线提交状态为 `COMPLETE`。
- 基线公榜分数：`0.98607`。
- 基线为两层 CNN、训练 5 轮。
- 保留基线不动，创建只增加训练期随机平移增强的私有 v2 notebook，
  仍训练 5 轮，用于单变量对照。
- v2 验证准确率：`0.9875`。
- v2 提交文件：`28000` 行、无空值、标签范围 `0-9`，格式检查通过。
- v2 公榜分数：`0.98635`，相对基线提升 `0.00028`。

## 遇到的问题

1. `profile.set_shape()` 成功时返回 `None`，不能把返回值直接当成成功布尔值；
   应先调用，再检查 `bool(profile)`。
2. 最初错误地用概率阈值 `0.8` 直接比较 logits，后改为 logit 阈值
   `1.386294`。
3. ONNX Runtime CUDA 环境需要先执行 `onnxruntime.preload_dlls()`，
   才能稳定加载独立 CUDA 12 运行库。
4. WSL 下 `nvidia-smi` 无法可靠提供当前 Linux 进程显存，PyTorch/ORT
   改用 `cudaMemGetInfo` 前后差值；TensorRT 报告 context 与 I/O buffer
   容量下界，并明确不包含权重、CUDA context 和缓存。
5. 单次推理存在冷启动和系统抖动，正式 benchmark 必须 warmup；首次加载
   延迟单独记录，不能混入稳态平均值。
6. RTX 4070 上生成的 plan 不能直接复制到 Jetson 使用。Jetson 需要从
   ONNX 在目标板环境重新构建自己的 engine。

## 今日产出

- 可复现的 FP32/FP16 TensorRT engine 构建脚本。
- 支持动态 batch、CUDA stream 和资源释放的 TensorRT Python runtime。
- PyTorch CUDA、ONNX Runtime CUDA、TensorRT FP32/FP16 统一 benchmark。
- engine 构建报告、正确性报告、性能报告和制品 SHA-256。
- TensorRT benchmark 工程文档。
- `70 passed` 的项目测试结果，Ruff 与 Git diff 检查通过。
- Kaggle Digit Recognizer 公榜基线记录与数据增强 v2 实验。

## 明日计划

- 建立 Triton model repository。
- 配置 TensorRT backend、模型版本和动态 batching。
- 实现 FastAPI 网关、健康检查和基础接口测试。
- 准备 Docker Compose 与 Prometheus 指标验证。
