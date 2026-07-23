# 作品集项目规划

两个项目共享同一条部署主线，但必须证明不同的工程能力：

```text
PyTorch -> ONNX -> TensorRT -> benchmark -> 可复现部署
```

- 项目 1 面向 x86 GPU 服务端，重点是语义分割、并发推理、服务接口和容器化。
- 项目 2 面向 Jetson 边缘端，重点是 C++ 流式处理、跟踪、事件规则、功耗和可靠性。
- 两个项目分别建立公开 GitHub 仓库，正式代码不放入本学习仓库。

## 项目 1：工业钢材缺陷分割与 GPU 推理服务

仓库名：`industrial-defect-inference-service`

### 项目定位

使用 Severstal Steel Defect Detection 数据集构建钢材表面缺陷语义分割系统，完成模型训练、ONNX/TensorRT 优化、Triton 服务化、FastAPI 网关和可复现压测。

数据集包含 12,568 张训练图像、4 类匿名钢材缺陷和 RLE 掩码。它足以支撑一个完整的工业分割项目，同时不会把暑假主要时间消耗在超大规模训练上。

### 核心架构

```text
Client
  -> FastAPI gateway
  -> Triton Inference Server
  -> TensorRT FP16 model
  -> mask decoding and response
  -> metrics and benchmark report
```

### 核心交付

- 数据检查、RLE 掩码解码、固定随机种子的训练/验证划分。
- U-Net + ResNet18 baseline，记录 Dice、IoU、Precision、Recall 和图像级漏检率。
- PyTorch、ONNX Runtime、TensorRT FP32/FP16 输出一致性和性能比较。
- Triton model repository、动态 batching 和 Prometheus 指标。
- FastAPI 图片上传、结果查询、错误处理和健康检查接口。
- Docker Compose 启动服务，完成并发压测和异常输入测试。
- 记录 mean/P50/P95/P99、QPS、错误率、GPU 显存、Triton queue/compute 时间。

### v1 边界

- 必做：Severstal baseline、ONNX、TensorRT FP16、Triton、FastAPI、Docker Compose、压测和完整 README。
- 选做：INT8、gRPC、Kubernetes、云端公开演示。
- 不把原始数据集、训练权重或大型 engine 提交到 Git。

## 项目 2：Jetson 实时跟踪与安全事件分析系统

仓库名：`jetson-realtime-tracking-system`

### 项目定位

在 Jetson Orin Nano 上实现一个长期运行的边缘视频分析进程，对人员或车辆执行实时检测、跨帧跟踪和安全事件判断。v1 聚焦三类可解释事件：

- 限制区域闯入。
- 指定方向越线。
- 区域停留超时。

这不是“摄像头跑一次 YOLO”的演示，而是包含视频输入、推理、跟踪、规则、证据留存、监控和故障恢复的完整边缘应用。

### 核心架构

```text
File / USB UVC / RTSP
  -> GStreamer capture and decode
  -> bounded frame queue with drop-oldest policy
  -> TensorRT C++ detector
  -> ByteTrack C++ tracker
  -> ROI / line-crossing / dwell rule engine
  -> JSONL event + snapshot + optional event clip
  -> tegrastats metrics + systemd watchdog
```

核心接口：

- `IFrameSource`：本地视频、USB 摄像头和 RTSP 使用同一输入接口。
- `IDetector`：TensorRT detector 与测试用 mock detector 可替换。
- `ITracker`：封装 ByteTrack，隔离检测与跟踪。
- `IEventSink`：输出 JSONL、截图和事件视频片段。

运行结构采用采集、推理/跟踪、事件写入三阶段线程和有界队列。输入拥塞时丢弃最旧帧并保留时间戳，避免实时系统因积压产生越来越大的延迟。

### 核心技术栈

- C++17、CMake、CTest。
- GStreamer 视频采集、解码和时间戳管理。
- TensorRT C++ API，使用 FP16 engine。
- YOLOX-Nano / YOLOX-Tiny 二选一，按 Jetson 实测精度与性能确定。
- ByteTrack C++ 多目标跟踪。
- YAML 配置、JSONL 事件协议、systemd 服务和日志轮转。
- GitHub Actions 运行不依赖 Jetson 的几何、事件状态机和配置解析测试。

选择 YOLOX 是为了获得 Apache-2.0 许可和成熟的 TensorRT C++ 路径；选择 ByteTrack 是为了使用 MIT 许可的跟踪实现。权重、数据集和第三方源码分别记录来源、版本、许可和校验值。

### 数据与评估

项目 2 是流式系统工程项目，不需要人为扩大训练数据集。数据设计分成三层：

1. MOT17 训练序列和标注：验证跟踪效果，输出 HOTA、IDF1、MOTA 和 ID switches。
2. 确定性的合成轨迹：覆盖越线、闯入、离开、停留、边界抖动和事件去重单元测试。
3. 8-12 段短事件视频：人工标注事件时间和类别，计算事件 Precision、Recall、重复事件率与时间戳误差。

文件回放是可复现基线，USB 摄像头和 RTSP 是真实输入验证。摄像头晚到不会阻塞 detector、tracker、事件规则和 benchmark 开发。

### 性能与可靠性验收

- 跟踪：HOTA、IDF1、MOTA、ID switches。
- 事件：Precision、Recall、重复事件率、触发时间误差。
- 实时性：effective FPS、mean/P50/P95 端到端延迟、丢帧率和队列深度。
- 设备：CPU/GPU/RAM、温度、功耗和 `nvpmodel` 模式。
- 稳定性：30-60 分钟 soak test、视频源断开重连、进程异常后 systemd 重启、磁盘与日志轮转。

性能目标在完成设备基线后锁定。初始工程目标是在选定的单路 720p 或 1080p 配置上达到不低于 25 FPS、稳定输入下丢帧率低于 1%，并通过 30 分钟无未恢复崩溃测试；最终 README 只报告真实测量结果。

### DeepStream 兼容性决策

当前设备是 JetPack 6.2.1 / L4T 36.4.3 / CUDA 12.6 / TensorRT 10.3。DeepStream 7.1 的官方验证组合是 JetPack 6.1，NVIDIA 论坛也提示在 JetPack 6.2 上可能遇到内存复制兼容问题。因此：

- v1 使用原生 GStreamer + TensorRT C++，不依赖 DeepStream。
- DeepStream 只作为兼容性门禁后的对照实验或 ADR。
- 如果验证不稳定，保留问题、版本和结论，不为追求关键词牺牲项目可靠性。

### v1 边界

- 必做：本地视频回放、TensorRT C++ 检测、ByteTrack、三类事件、JSONL/截图、设备指标、systemd、故障恢复和演示视频。
- 摄像头可用后补充 USB UVC 实时演示；RTSP 至少完成连接和断线恢复测试。
- 视觉小车、CUDA 自定义预处理、INT8、DeepStream 和多路流属于 v1.1/v2。

## 两个项目的区别

| 维度 | 项目 1 | 项目 2 |
| --- | --- | --- |
| 场景 | 钢材缺陷语义分割 | 实时安全事件分析 |
| 输入 | 图片请求、batch | 连续视频、USB、RTSP |
| 环境 | x86 RTX GPU / 云 GPU | Jetson Orin Nano ARM |
| 主要语言 | Python | C++17 |
| 模型 | 自训练 U-Net 分割模型 | 轻量检测器 + ByteTrack |
| 系统形态 | Triton/FastAPI 推理服务 | GStreamer 长驻进程 |
| 优化重点 | batch、并发、吞吐、服务延迟 | 实时性、背压、功耗、可靠性 |
| 主要指标 | Dice/IoU、QPS、P95/P99、显存 | HOTA/IDF1、事件准确率、FPS、功耗 |
| 工程关键词 | ONNX、TensorRT、Triton、Docker、FastAPI | C++、CMake、GStreamer、Jetson、systemd |

## 公开仓库展示规范

两个公开项目仓库只展示工程开发过程：

- README 首屏先展示演示图、架构图和关键指标，不写学习背景。
- 使用 release milestone、issue、ADR、`CHANGELOG.md` 和 benchmark report 记录演进。
- commit message 描述功能、修复、性能或文档变化，不使用 `DayXX` 或“完成学习任务”。
- 提供环境矩阵、复现命令、数据/权重获取脚本、许可和校验值。
- README 不承诺未测量的性能，不把第三方数据和权重直接提交到仓库。

## 云算力原则

- 本地 RTX 4070 用于开发、调试和小规模训练。
- 数据规模、显存或训练时长有明确需要时，可以使用 AutoDL、Google Cloud、AWS、Colab 等付费或免费算力。
- 租用 GPU 前先验证脚本和数据路径，记录 GPU、显存、训练时长、主要参数和大致成本。

## 时间节点

- Day18-Day25：完成项目 1 v1。
- Day26-Day27：Jetson C++/GStreamer/TensorRT 环境与设备基线。
- Day28：创建项目 2 独立仓库。
- Day29-Day40：完成项目 2 v1。
- Day41-Day53：两个项目性能、稳定性和可展示性增强。
- Day54-Day60：简历、GitHub、岗位和面试交付。

详细日程以 `roadmap/60-day-plan.md` 为唯一执行表。

## 主要技术资料

- NVIDIA DeepStream SDK：https://developer.nvidia.com/deepstream-sdk
- NVIDIA TensorRT：https://docs.nvidia.com/deeplearning/tensorrt/latest/index.html
- NVIDIA Jetson `tegrastats`：https://docs.nvidia.com/jetson/archives/r36.4/DeveloperGuide/AT/JetsonLinuxDevelopmentTools/TegrastatsUtility.html
- ByteTrack：https://github.com/FoundationVision/ByteTrack
- YOLOX：https://github.com/Megvii-BaseDetection/YOLOX
- MOTChallenge：https://motchallenge.net/
- TrackEval：https://github.com/JonathonLuiten/TrackEval
