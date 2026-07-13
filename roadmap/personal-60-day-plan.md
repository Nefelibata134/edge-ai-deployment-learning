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

### 项目 1：工业视觉缺陷检测与 GPU 推理服务

仓库建议：`industrial-defect-inference-service`

核心交付：

- 公开工业缺陷数据集训练和 mAP 评估。
- PyTorch、ONNX Runtime、TensorRT FP16 推理。
- FastAPI 图片推理接口和 Docker 环境。
- QPS、P50/P95、显存和准确率对比。
- 完整 README、结果图、API 示例和演示视频。

项目 1 证明领域模型训练、GPU 优化和服务端工程能力。

### 项目 2：Jetson 实时多目标跟踪与事件检测系统

仓库建议：`jetson-realtime-tracking-system`

核心交付：

- Jetson TensorRT FP16 实时检测。
- ByteTrack 或等价多目标跟踪。
- 越线计数、区域进入/离开或停留事件。
- CSV/JSON 日志、事件截图和结果视频。
- FPS、P50/P95、温度、功耗和稳定性报告。
- 无显示器 SSH 运行和启动脚本。

视觉小车、DeepStream、RTSP 和 C++ 重写属于增强项，不阻塞项目 v1。

两个项目分别证明服务端 GPU 推理工程和边缘实时视觉工程，不是同一个项目换设备重复一次。

## 当前统一时间节点

| 时间 | 主目标 | 硬节点 |
| --- | --- | --- |
| Day1-Day13 | Linux、OpenCV、YOLO、ONNX、ORT CPU/GPU | 已完成 |
| Day14-Day17 | I/O Binding 与 TensorRT 核心流程 | TRT Python demo |
| Day18-Day25 | 项目 1 冲刺 | Day18 建仓，Day25 冻结 v1 |
| Day26-Day30 | Jetson 基线与项目 2 启动 | 板端 TRT、摄像头、项目 2 建仓 |
| Day31-Day40 | 项目 2 冲刺 | Day40 冻结 v1 |
| Day41-Day53 | 两项目 v2、竞赛、C++ 与稳定性 | Day53 两项目可投递 |
| Day54-Day60 | 简历、GitHub、岗位和面试 | 完成第一批投递材料 |

这与之前的协作承诺一致：项目 1 不拖到 Day47 才开始，项目 2 不拖到最后一周，竞赛也不放到 Day57 后才处理。

## Jetson 使用节点

- Day14-Day24：以 RTX 4070 + WSL 为主，不要求 Jetson 在身边。
- Day25 晚上前：Jetson、摄像头、电源和 Type-C 线准备到位。
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
| Day30 | Jetson 环境、摄像头和实时检测跑通 |
| Day40 | 项目 2 v1 跑通并有演示视频 |
| Day53 | 两个项目 v2、README 和复现命令完成 |
| Day60 | 简历、GitHub、岗位表、面试题和总复盘完成 |

## 调整原则

- Jetson 暂时不在身边时，继续 PC 端 TensorRT 和项目 1，不等待设备。
- 必要时可以使用 AutoDL、Google Cloud、AWS、Colab 等云算力。
- 竞赛不能挤掉项目硬节点，同时只重投入一个主赛。
- 项目功能先完成 v1，再做 INT8、Triton、DeepStream 或视觉小车等增强项。
- 每天结束更新 `logs/dayXX.md` 和 README 进度索引。
