# 个人版 60 天模型部署学习路线

最后统一日期：2026-07-13

本文件说明目标、资源分配和阶段验收；具体每天做什么，以 [`roadmap/60-day-plan.md`](./60-day-plan.md) 为唯一执行表。

## 个人背景

- 大三，电子信息工程。
- 每天可投入约 7 小时。
- Python 入门，C++ 较弱但有 C 语言基础，Linux 和深度学习处于入门阶段。
- 不考研，暑假结束后准备实习、秋招和项目投递。
- 已有 RTX 4070 Laptop、WSL2 和 Jetson Orin Nano 8GB。
- Jetson 已完成首次初始化、JetPack/TensorRT 检查和 SSH/串口连接。

## 主方向

主方向：边缘视觉部署。

该方向同时覆盖电子信息背景和就业关键词：Linux、Python、C++、OpenCV、ONNX、TensorRT、CUDA、Jetson、DeepStream、摄像头、性能优化和视频流工程。

暂不把 CUDA 算子开发、驱动开发或纯底层嵌入式作为暑假主线，因为当前两个月更需要可展示项目和岗位匹配度。

## 暑假必须完成

1. 完成两个独立 GitHub 项目。
2. 至少完成一次天池或 Kaggle 有效提交，并对一个视觉赛题做有记录的优化。
3. 完成 TensorRT FP16、PC/Jetson benchmark 和设备端部署。
4. 完成简历、GitHub 首页、项目演示和岗位关键词表。
5. 每周持续补 C++、Linux 和部署面试基础。

## 三条并行主线

从 Day13 起不再采用“先学 45 天、最后才做项目”的方式。

| 主线 | 每周投入 | 暑假验收 |
| --- | ---: | --- |
| 部署学习与两个项目 | 27-30 小时 | 两个独立、可复现项目 |
| 个人竞赛 | 4-5 小时 | 至少一次有效提交和一次优化 |
| C++ / Linux / 就业准备 | 3-4 小时 | 小练习、面试题和岗位材料 |

当天主任务提前完成时，剩余时间优先投入项目或竞赛，不继续堆叠低价值教程。

## 每日 7 小时参考节奏

| 时间 | 内容 |
| ---: | --- |
| 1-1.5 小时 | 新概念与官方资料 |
| 2-3 小时 | 当天部署实验或项目功能 |
| 1-1.5 小时 | benchmark、排错和结果分析 |
| 0.5-1 小时 | C++ / Linux / 竞赛轮换 |
| 0.5-1 小时 | 日志、README 和 Git 提交 |

进入项目冲刺后，可以把基础学习时间转给项目开发，但每天仍需留下可见产出。

## 两个就业项目

### 项目 1：工业钢材缺陷分割与 GPU 推理服务

仓库建议：`industrial-defect-inference-service`

核心交付：

- Severstal 钢材缺陷数据集、RLE 掩码和 U-Net + ResNet18 baseline。
- Dice、IoU、Precision、Recall 和图像级漏检率评估。
- PyTorch、ONNX Runtime、TensorRT FP32/FP16 推理对比。
- Triton 动态 batching、FastAPI 网关和 Docker Compose。
- QPS、P50/P95/P99、显存、错误率和完整演示材料。

项目 1 证明领域模型训练、GPU 优化和服务端工程能力。

### 项目 2：Jetson 实时跟踪与安全事件分析系统

仓库建议：`jetson-realtime-tracking-system`

核心交付：

- C++17、CMake、GStreamer 和 TensorRT C++ FP16 实时检测。
- YOLOX-Nano/Tiny 性能选型和 ByteTrack C++ 多目标跟踪。
- 限制区域闯入、方向越线和停留超时事件状态机。
- JSONL 事件、截图、事件去重和可选前后事件视频。
- MOT17/TrackEval 跟踪指标和人工事件视频评估。
- FPS、P50/P95、丢帧率、队列深度、温度、功耗和稳定性报告。
- systemd、watchdog、断线重连、日志轮转和无显示器运行。

RTSP 输入属于 v1，USB 摄像头晚到时可先用文件回放。视觉小车、DeepStream、CUDA 自定义预处理和 INT8 属于增强项；DeepStream 只有通过 JetPack 兼容性验证后才进入主分支。

两个项目分别证明服务端 GPU 推理工程和边缘实时视觉工程，不是同一个项目换设备重复一次。

## 当前统一时间节点

| 时间 | 主目标 | 硬节点 |
| --- | --- | --- |
| Day1-Day13 | Linux、OpenCV、YOLO、ONNX、ORT CPU/GPU | 已完成 |
| Day14-Day17 | I/O Binding 与 TensorRT 核心流程 | TRT Python demo |
| Day18-Day25 | 项目 1 冲刺 | Day18 建仓，Day25 冻结 v1 |
| Day26-Day30 | Jetson C++ 工具链与项目 2 启动 | 板端 TRT C++、GStreamer replay、项目 2 建仓 |
| Day31-Day40 | 项目 2 跟踪、事件和可靠性冲刺 | Day40 冻结 v1 |
| Day41-Day53 | 两项目 v2、竞赛、性能与稳定性 | Day53 两项目可投递 |
| Day54-Day60 | 简历、GitHub、岗位和面试 | 完成第一批投递材料 |

这与之前的协作承诺一致：项目 1 不拖到 Day47 才开始，项目 2 不拖到最后一周，竞赛也不放到 Day57 后才处理。

## Jetson 使用节点

- Day14-Day24：以 RTX 4070 + WSL 为主，不要求 Jetson 在身边。
- Day25 晚上前：Jetson、电源和 Type-C 线准备到位；摄像头未到时用本地视频和 RTSP 替代。
- Day26-Day40：Jetson 是主设备，必须在身边。
- Day41-Day53：按项目增强和稳定性测试间歇使用 Jetson。

TensorRT engine 通常在目标平台构建，PC 与 Jetson 各自生成，不直接跨平台复用 engine。

## 竞赛并行计划

| 时间 | 任务 | 验收 |
| --- | --- | --- |
| Day13-Day15 | 两项天池报名、Kaggle 候选确认 | 候选与规则表 |
| Day16-Day21 | Digit Recognizer 或最终练习赛 baseline | 可运行 Notebook |
| Day22-Day25 | 第一次有效提交 | 排行榜分数/截图 |
| Day26-Day35 | CSIG 或最终主赛至少两次实验 | 实验对比表 |
| Day36-Day45 | 决定继续冲榜或停止，整理复盘 | 竞赛 README |
| Day46-Day60 | 关注华为 ICT、嵌入式和机器人赛事 | 开学后赛事清单 |

当前优先级：

- 正式主赛候选：第七届 CSIG 复杂工业场景异常检测算法挑战。
- 低成本练习赛：Kaggle Digit Recognizer。
- CCL2026-Eval 已报名，但暂不重投入。

## 阶段检查点

| 截止点 | 必须完成 |
| --- | --- |
| Day17 | TensorRT FP16 和 Python 推理流程跑通 |
| Day25 | 项目 1 v1 + 第一次竞赛有效提交 |
| Day30 | Jetson GStreamer 回放、TensorRT C++ 检测和设备基线跑通 |
| Day40 | 项目 2 跟踪、三类事件、可靠性和演示视频完成 |
| Day53 | 两个项目 v2、README 和复现命令完成 |
| Day60 | 简历、GitHub、岗位表、面试题和总复盘完成 |

## 调整原则

- Jetson 暂时不在身边时，继续 PC 端 TensorRT 和项目 1，不等待设备。
- 必要时可以使用 AutoDL、Google Cloud、AWS、Colab 等云算力。
- 竞赛不能挤掉项目硬节点，同时只重投入一个主赛。
- 项目 1 的 Triton 和项目 2 的 C++/GStreamer 是 v1 核心；INT8、DeepStream、CUDA 自定义预处理或视觉小车是增强项。
- 每天结束更新 `logs/dayXX.md` 和 README 进度索引。
