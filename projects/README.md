# 作品集项目规划

两个项目必须共享一条清晰的技术主线，同时证明不同层面的工程能力。

共同主线：

```text
PyTorch -> ONNX -> TensorRT -> benchmark -> 可复现部署
```

区别要求：

- 不能只是把同一个模型从 PC 搬到 Jetson。
- 应在应用场景、输入形态、系统架构、性能指标和工程难点上形成明显差异。
- 每个项目单独建立 GitHub 仓库，学习计划和每日记录仍保留在本仓库。

## 项目 1：工业视觉缺陷检测与 GPU 推理服务

建议英文名：`industrial-defect-inference-service`

### 项目定位

在本地 RTX 4070 或云 GPU 上完成工业缺陷检测模型的训练、评估、推理加速和服务化部署。

这个项目重点证明：

- 能处理数据集并训练一个面向具体场景的视觉模型。
- 能从 PyTorch 导出 ONNX 和 TensorRT。
- 能把模型封装成可调用、可压测、可复现的 GPU 推理服务。
- 能用准确率和服务性能两组指标评价系统。

### 核心功能

- 选择公开工业缺陷数据集，完成数据检查、划分和可视化。
- 训练轻量目标检测模型，优先使用 YOLO11n 或 YOLO11s。
- 输出 Precision、Recall、mAP50 和 mAP50-95。
- 导出 ONNX，并生成 TensorRT FP32 / FP16 engine。
- 对比 PyTorch、ONNX Runtime、TensorRT 的延迟和吞吐。
- 使用 FastAPI 提供图片上传和 JSON 检测接口。
- 返回类别、置信度、检测框和可视化结果图。
- 使用 Docker 固化运行环境。
- 使用 Locust、wrk 或同类工具测试 QPS、P50 和 P95。
- 记录 GPU 型号、显存占用、batch、输入尺寸和测试方法。

### 可选增强

- TensorRT INT8 校准与精度对比。
- Triton Inference Server 和动态 batching。
- gRPC 接口。
- 云端部署和公开演示地址。
- 将比赛数据集或比赛 baseline 复用为项目数据来源。

### 核心指标

- 模型指标：Precision、Recall、mAP50、mAP50-95。
- 单请求指标：平均延迟、P50、P95。
- 服务指标：QPS、并发数、batch、错误率。
- 资源指标：GPU 显存、CPU 占用、镜像大小。

### 最小可交付版本

- 一个训练得到的缺陷检测模型。
- PyTorch / ONNX Runtime / TensorRT FP16 三种推理方式。
- 一个 FastAPI 推理接口。
- 一份性能对比表。
- Dockerfile、requirements、运行命令和结果截图。
- 完整 README 和演示视频。

## 项目 2：Jetson 实时多目标跟踪与事件检测系统

建议英文名：`jetson-realtime-tracking-system`

### 项目定位

在 Jetson Orin Nano 上处理摄像头或视频流，完成实时目标检测、跨帧跟踪和事件判断。

这个项目重点证明：

- 能在 ARM 边缘设备上部署视觉模型。
- 能处理持续视频流，而不只是单张图片请求。
- 能把检测结果组合成跟踪、计数和事件逻辑。
- 能在功耗、温度和实时性约束下完成系统优化。

### 核心功能

- 支持 USB 摄像头和本地视频输入。
- 使用 YOLO11n TensorRT FP16 完成目标检测。
- 使用 ByteTrack、DeepStream tracker 或同类方法进行多目标跟踪。
- 支持至少两类事件逻辑：
  - 越线计数。
  - 指定区域进入、离开或停留检测。
- 在画面中显示目标 ID、类别、置信度、FPS 和计数。
- 保存事件截图和 CSV / JSON 日志。
- 对比不同输入尺寸和 Jetson 功耗模式。
- 记录端到端延迟、FPS、温度、功耗和内存占用。
- 支持无显示器 SSH 运行、日志查看和正常退出。
- 提供启动脚本，条件允许时配置开机自启动。

### 可选增强

- 使用 DeepStream / GStreamer 重构视频 pipeline。
- 接入 RTSP 网络视频流。
- 增加 Web 状态页或事件查询接口。
- 用 C++ 重写摄像头采集、后处理或关键推理路径。
- 接入视觉小车，实现目标跟随或简单避障。

### 核心指标

- 实时指标：平均 FPS、P50 / P95 端到端延迟、丢帧情况。
- 设备指标：温度、功耗、CPU / GPU / 内存占用。
- 稳定性指标：持续运行时间、异常输入恢复、摄像头断开处理。
- 业务指标：计数结果、事件触发结果、目标 ID 连续性。

### 最小可交付版本

- Jetson TensorRT FP16 实时检测与跟踪。
- 越线计数和区域事件检测。
- 事件日志与结果视频。
- 输入尺寸、功耗模式、FPS 和温度对比表。
- 一键启动脚本、环境说明和完整 README。
- 一段完整演示视频。

## 两个项目的区别

| 维度 | 项目 1 | 项目 2 |
| --- | --- | --- |
| 场景 | 工业缺陷检测 | 实时视频感知与事件检测 |
| 输入 | 图片请求、batch | 摄像头或连续视频流 |
| 环境 | x86 RTX GPU / 云 GPU | Jetson Orin Nano ARM 边缘设备 |
| 模型来源 | 自己训练的领域模型 | 预训练或轻量微调模型 + tracker |
| 系统形态 | HTTP/gRPC 推理服务 | 长时间运行的视频 pipeline |
| 主要优化 | 并发、batch、吞吐、服务延迟 | 实时性、功耗、温度、稳定性 |
| 主要指标 | mAP、QPS、P50/P95、显存 | FPS、端到端延迟、功耗、温度 |
| 工程关键词 | FastAPI、Docker、Triton、压测 | Jetson、DeepStream、GStreamer、跟踪 |

两个项目共享模型部署基础，但分别覆盖服务端推理工程和边缘实时视觉工程，放在同一份简历中不会互相重复。

## 云算力原则

- 本地 RTX 4070 用于开发、调试和小规模训练。
- 当数据集、显存或训练时长有明确需要时，可以使用付费云算力。
- AutoDL、Google Cloud、AWS、Google Colab 或其他合规平台都可以按任务选择。
- 云 GPU 不限制为免费资源，也不人为限制在 8GB 显存以内。
- 使用付费资源前先确定训练脚本、数据路径、预估时长和保存策略，避免把租用时间浪费在环境排错上。
- 项目 README 记录云 GPU 型号、显存、训练时长、主要参数和大致成本，使实验可复现。

## 项目时间节点

- Day13-Day17：GPU / TensorRT 基础，继续完善项目 1 所需能力。
- Day18：创建项目 1 独立仓库并确定数据集、指标和目录结构。
- Day19-Day25：完成项目 1 训练、ONNX/TensorRT、FastAPI、Docker、benchmark 和 v1 README。
- Day26-Day27：项目 1 缓冲与修复，同时完成 Jetson 环境、摄像头和设备基线。
- Day28：创建项目 2 独立仓库，并在 Jetson 上生成板端 TensorRT engine。
- Day29-Day30：跑通 Jetson 实时检测和功耗/分辨率调优。
- Day31-Day40：完成项目 2 检测、跟踪、事件逻辑、性能测试、README 和 v1 演示。
- Day41-Day53：两个项目 v2、竞赛改进、C++/DeepStream/稳定性增强。
- Day54-Day60：简历、GitHub、岗位、面试和投递材料。

详细日程以 `roadmap/60-day-plan.md` 为唯一执行表。
