# Day 24 学习记录

日期：2026-07-27 至 2026-07-28

状态：已完成

## 今日主题

将 Day23 生成的 TensorRT FP16 engine 从“本地脚本可运行”升级为一个可部署、
可调用、可监控的 GPU 推理服务。

今天的完整链路是：

```text
客户端上传钢材图片
-> FastAPI 网关
-> 图片校验与预处理
-> Triton Inference Server
-> TensorRT FP16 engine
-> RTX 4070 GPU 推理
-> sigmoid、阈值、连通域与 RLE 后处理
-> JSON 响应
-> Prometheus 采集网关和 Triton 指标
```

## 为什么今天重要

Day23 解决的是“如何把模型编译成更快的 TensorRT engine”，Day24 解决的是
“如何让其他程序稳定地调用这个模型”。

对应企业技术栈：

- TensorRT：模型编译和 GPU 推理优化。
- Triton Inference Server：模型加载、版本管理、动态 batching 和推理协议。
- FastAPI：业务接口、输入校验、错误处理和结果封装。
- Docker Compose：固定运行环境并编排多个服务。
- Prometheus：采集请求量、延迟、GPU 和 Triton 推理指标。
- pytest：验证接口成功路径和失败路径。

## 七小时时间分配

| 模块 | 时间 | 验收产出 |
| --- | ---: | --- |
| 服务架构与版本兼容 | 0.5 小时 | 能解释 engine 与 Triton/TensorRT 版本关系 |
| Triton model repository | 1.5 小时 | 模型版本目录、`config.pbtxt`、动态 batching |
| FastAPI 网关 | 1.5 小时 | 健康检查、图片上传、推理和 JSON 响应 |
| Docker Compose 与 Prometheus | 1 小时 | 三服务编排和两个监控采集目标 |
| 端到端与失败路径测试 | 1 小时 | 真实图片、415、422、503 和服务指标 |
| C++ 基础 | 1 小时 | 与项目 2 相关的 C++17 类和资源管理练习 |
| 复盘与文档 | 0.5 小时 | 日志、README 索引、提交与推送 |

## 当前兼容性基线

```text
host: WSL2 Ubuntu 22.04
GPU: NVIDIA GeForce RTX 4070 Laptop GPU
driver: 610.47
Triton image: nvcr.io/nvidia/tritonserver:25.03-py3
Triton Server: 2.56.0
CUDA in Triton image: 12.8.1
TensorRT in Triton image: 10.9.0.34
engine precision: FP16
engine profile: batch 1..8
```

Day23 的 FP16 engine 使用 TensorRT `10.9.0.34` 构建。今天选择包含相同
TensorRT 版本的 Triton 25.03，避免直接加载 plan 时发生反序列化兼容错误。

engine 信息：

```text
model:
steel_defect_segmentation

artifact:
model_repository/steel_defect_segmentation/1/model.plan

size:
31,494,324 bytes

SHA256:
0e040ac97a207baf476a07716c16e4c9c024d85d286ac1977e7be8b127ce201d
```

## 模块 1：Triton model repository

目录结构：

```text
model_repository/
└── steel_defect_segmentation/
    ├── config.pbtxt
    └── 1/
        └── model.plan
```

核心配置：

```text
platform: tensorrt_plan
max_batch_size: 8
input:  images, FP32, [3, 256, 1024]
output: logits, FP32, [4, 256, 1024]
preferred_batch_size: [4, 8]
max_queue_delay_microseconds: 2000
instance_group: one GPU instance
```

理解检查：

- `max_batch_size: 8`：单个请求不能超过 8，动态调度合并后的一次执行总
  batch 也不能超过 8。
- `preferred_batch_size: [4, 8]`：是偏好值，不是只允许 batch 4 和 8。
- 最大等待 2 ms：低流量时可能增加等待延迟；高流量时有机会合并请求，
  提高 GPU 利用率和吞吐量。

## 模块 2：FastAPI 推理网关

已实现接口：

| 方法 | 路径 | 作用 |
| --- | --- | --- |
| GET | `/health/live` | 确认网关进程存活 |
| GET | `/health/ready` | 确认 Triton 和模型已就绪 |
| POST | `/v1/segment` | 上传钢材图片并执行缺陷分割 |
| GET | `/metrics` | 暴露 Prometheus 指标 |

`/v1/segment` 数据流：

```text
UploadFile
-> MIME、文件大小和 1600x256 尺寸校验
-> resize 到 1024x256
-> HWC/BGR 转 CHW/RGB float32
-> 增加 batch 维度，得到 [1, 3, 256, 1024]
-> Triton HTTP 推理
-> logits [1, 4, 256, 1024]
-> sigmoid 和阈值
-> resize 回原图尺寸
-> RLE、连通域和像素数
-> JSON
```

错误契约：

| 状态码 | 场景 |
| ---: | --- |
| 413 | 上传文件超过大小限制 |
| 415 | 文件媒体类型不受支持 |
| 422 | 图片无法解码、尺寸或通道不符合契约 |
| 503 | Triton 不可用或模型推理失败 |

## 模块 3：Docker Compose 与可观测性

Compose 管理三个服务：

```text
triton:
  8000 HTTP
  8001 gRPC
  8002 metrics

api:
  8080 FastAPI

prometheus:
  9090 Prometheus
```

启动后状态：

```text
triton: healthy
api: healthy
prometheus: running
steel_defect_segmentation version 1: READY
```

Prometheus 采集状态：

```text
inference-gateway -> up
triton -> up
```

网关指标能够区分：

- HTTP 请求数量和状态码。
- 端到端请求延迟。
- 网关观察到的 Triton 调用延迟。

Triton 指标能够区分：

- 成功和失败请求数。
- inference count 与 execution count。
- queue、compute input、compute infer 和 compute output 时间。
- GPU 利用率、显存和功耗。

实际观察：

```text
gateway POST /v1/segment 200: 2
gateway inference observations: 2

Triton successful requests: 2
Triton inference count: 2
Triton execution count: 2
queue duration total: 4601 us
compute infer duration total: 25339 us
```

由此计算：

- 平均排队约 `2.30 ms`，与动态 batching 的最大等待 `2 ms` 接近。
- 平均 TensorRT compute infer 约 `12.67 ms`。
- `inference_count=2` 且 `execution_count=2`，说明两个低流量请求分别执行，
  没有被动态合并。
- 若两个单样本请求被合并成一次执行，应观察到 inference count 为 `2`、
  execution count 为 `1`。

## 已完成的真实验证

### 1. GPU 与模型加载

- Docker 容器成功识别 RTX 4070。
- Triton 成功反序列化 FP16 engine。
- 模型 `steel_defect_segmentation:1` 状态为 `READY`。

### 2. 真实钢材图片

输入：

```text
data/raw/severstal/train_images/0002cc93b.jpg
threshold: 0.80
```

返回结果：

```text
HTTP 200
detected classes: defect_1, defect_3
```

首次请求和并发请求包含 CUDA、服务与传输开销，不能直接当作稳态 GPU kernel
延迟。Triton 指标已将排队、输入、计算和输出时间拆开；Day25 再通过固定
warmup 和并发压测生成正式 P50/P95/P99 报告。

### 3. 失败路径

```text
Markdown 文件:
HTTP 415 unsupported media type

1449x601 截图:
HTTP 422 image shape mismatch
```

### 4. 阈值修改验证

对同一张钢材图片只改变概率阈值：

| 类别 | threshold 0.80 | threshold 0.95 |
| --- | ---: | ---: |
| defect_1 | 1979 px，5 个连通域 | 1481 px，4 个连通域 |
| defect_2 | 0 px | 0 px |
| defect_3 | 3212 px，7 个连通域 | 211 px，1 个连通域 |
| defect_4 | 0 px | 0 px |

结论：

- 提高阈值不会重新训练模型，只会改变 logits 经 sigmoid 后的二值判定。
- 较低阈值通常检出更多区域、Recall 更高，但可能增加误报。
- 较高阈值通常预测更谨慎、Precision 可能更高，但可能增加漏检。
- 工业质检若漏检代价更高，不能仅为减少误报而盲目使用 `0.95`。

### 5. 自动化检查

```text
pytest: 80 passed
Ruff: all checks passed
git diff --check: passed
```

## C++17：RAII 与 CMake

在独立练习目录 `~/model-deploy-day24-cpp` 中实现 `ScopedTimer`：

```text
构造函数:
记录开始时间

析构函数:
对象离开作用域时自动计算并输出耗时

复制构造与复制赋值:
= delete
```

这对应 TensorRT C++、CUDA 和 GStreamer 中常见的 RAII 资源管理方式：
对象创建时获取资源，对象销毁时自动释放资源，避免遗漏清理或重复释放。

CMake 构建结果：

```text
[100%] Built target scoped_timer
```

第一次运行：

```text
TensorRT inference started
TensorRT inference finished: 20.102 ms
program finished
```

删除额外作用域后，`timer` 一直存活到 `main()` 结束，因此析构顺序变为：

```text
TensorRT inference started
program finished
TensorRT inference finished: 20.134 ms
```

亲自将模拟时间从 `20` 修改为 `35` 毫秒并重新构建：

```text
TensorRT inference started
program finished
TensorRT inference finished: 35.2266 ms
```

同时理解了：仅修改 `.cpp` 时执行 `cmake --build build` 即可，CMake 会重新
编译变化的源文件；只有 CMake 配置发生变化时才需要重新配置生成阶段。

## 当前进度

已经完成：

- Docker Desktop、WSL 集成和容器 GPU 访问验证。
- Triton 版本匹配和官方镜像固定。
- Triton model repository 与动态 batching。
- FastAPI 推理网关和输入错误契约。
- Docker Compose 三服务编排。
- Prometheus 双目标采集。
- 真实图片、错误媒体类型和错误尺寸验证。
- FastAPI 状态码、batch 维度和阈值修改验证。
- Prometheus 网关/Triton 指标读取和动态 batching 判断。
- C++17 RAII、对象生命周期与 CMake 增量构建。
- 80 项项目测试与代码检查。

## 今日复盘

- TensorRT engine 只是运行时制品；Triton 负责模型加载、版本和调度，
  FastAPI 负责业务输入输出契约。
- 动态 batching 的 preferred batch 是偏好而非强制，低流量下可能付出
  queue delay，但高并发时可减少 execution count 并提高吞吐。
- 网关延迟不等于 GPU kernel 延迟，必须结合 Triton 分阶段指标分析。
- 概率阈值提高会减少预测缺陷区域，通常在 Precision 与 Recall 之间形成
  权衡；工业质检不能只为降低误报而忽略漏检成本。
- C++ RAII 用对象生命周期管理资源，是项目 2 中封装 TensorRT engine、
  CUDA buffer、stream 和视频管线的重要基础。

## 明日计划

- 对项目 1 执行并发压测，记录 QPS、P50、P95 和 P99。
- 覆盖异常输入、Triton 不可用和服务恢复场景。
- 完成 README 结果区、性能报告和项目 1 v1 release。
- 检查 Jetson，为项目 2 的 C++ 实时视频管线做准备。
