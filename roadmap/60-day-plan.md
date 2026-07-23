# 60 天模型部署学习路线（当前执行版）

最后统一日期：2026-07-23

这份文件是 Day 级别的唯一执行表。若它与早期聊天、周框架或旧计划冲突，以本文件和当天 `logs/dayXX.md` 为准。

## 总目标

- 完成两个独立 GitHub 项目，而不是等到最后两周才开始项目。
- 项目 1：工业钢材缺陷分割与 GPU 推理服务。
- 项目 2：Jetson C++ 实时跟踪与安全事件分析系统。
- 至少完成一次天池或 Kaggle 有效提交，并对一个视觉赛题留下可复现实验记录。
- 暑假结束前形成简历、项目 README、演示视频、性能报告和面试准备材料。

## 固定并行节奏

| 主线 | 每周投入 | 执行规则 |
| --- | ---: | --- |
| 部署学习与两个项目 | 27-30 小时 | 从 Day18 起以项目驱动学习，不再连续堆教程 |
| 个人竞赛 | 4-5 小时 | 最多一个主赛加一个练习赛，先完成有效提交 |
| C++ / Linux / 就业准备 | 3-4 小时 | 每周至少两次短练习，补部署岗位基础 |
| 日志与 GitHub | 每天约 1 小时 | 更新日志、结果、问题和 commit |

## Day1-Day13：已完成的基础链路

| 天数 | 已完成主题 | 关键产出 | 状态 |
| --- | --- | --- | --- |
| Day1 | 方向、仓库、本机环境与采购规划 | 路线、GitHub、硬件清单 | 已完成 |
| Day2 | WSL2 与 Linux 基础命令 | 文件、进程、网络练习 | 已完成 |
| Day3 | Linux 权限、包管理与环境变量 | 权限脚本、排错记录 | 已完成 |
| Day4 | Conda、pip 与 Python 3.10 | `deploy310` 环境 | 已完成 |
| Day5 | Python、文件读写与 OpenCV | 图像读写脚本 | 已完成 |
| Day6 | OpenCV 可视化与 YOLO 入门 | 真实图片检测和批量推理 | 已完成 |
| Day7 | letterbox、归一化与 NCHW | YOLO 预处理脚本 | 已完成 |
| Day8 | YOLO 导出 ONNX | `yolo11n.onnx` 与结构检查 | 已完成 |
| Day9 | ONNX Runtime 原始推理 | ORT 推理和输出解析 | 已完成 |
| Day10 | 坐标转换、NMS 与画框 | 完整 ONNX 检测脚本 | 已完成 |
| Day11 | warmup、P50/P95、FPS | CPU benchmark | 已完成 |
| Day12 | 动态输入尺寸比较 | 320/480/640 对比 | 已完成 |
| Day13 | ORT CUDA Provider 与竞赛初筛 | CUDA 4.14x 推理加速、天池报名、Kaggle 候选 | 已完成 |

## Day14-Day17：TensorRT 前置与核心入门

| 天数 | 主任务 | 当天产出 | Jetson |
| --- | --- | --- | --- |
| Day14 | ONNX Runtime I/O Binding，理解 Host/Device 数据复制 | 普通 `session.run()` 与 I/O Binding 对比 | 不需要 |
| Day15 | TensorRT 架构、engine、builder、parser、profile；确认 PC 环境 | TensorRT 概念图与环境检查 | 不需要 |
| Day16 | 使用 `trtexec` 或等价 Builder API 构建 PC 端 FP32/FP16 engine | 构建命令、日志、engine 文件和延迟表 | 不需要，保持 PC 主线 |
| Day17 | TensorRT Python API、动态 shape、统一 benchmark；Jetson 重新接入预检 | TRT Python 推理 demo、错误清单、Jetson SSH/环境快照/ONNX 传输验证 | 使用约 1 小时，不在板端建 engine |

Jetson 从 Day16 起已可使用，但不提前并行启动项目 2。Day17 只做低风险接入和文件传输预检；Day18-Day25 仍集中完成项目 1 v1，Day26 起再正式进入 Jetson 基线和项目 2。这样可以利用设备，同时避免两个项目同时展开造成交付失控。

## Day18-Day25：项目 1 v1 冲刺

项目仓库：`industrial-defect-inference-service`，Day18 单独创建。

| 天数 | 主任务 | 当天产出 | 并行任务 |
| --- | --- | --- | --- |
| Day18 | 创建项目仓库，冻结 Severstal 分割任务、ADR、目录和数据契约 | 工程骨架、dataset card、数据校验器 | 竞赛规则/数据核对 |
| Day19 | 下载并检查 12,568 张图像，解码 RLE 掩码，固定训练/验证划分 | 数据报告、掩码可视化和 split 文件 | C++ 基础 1 小时 |
| Day20 | 训练 U-Net + ResNet18 baseline | checkpoint、loss 曲线和训练配置 | 竞赛 baseline |
| Day21 | 验证集评估和错误分析 | Dice、IoU、Precision、Recall、漏检与失败样例 | Linux/Git 复习 |
| Day22 | 导出 ONNX，验证 PyTorch/ORT 数值与掩码一致性 | ONNX 模型、parity report | 竞赛低成本优化 |
| Day23 | 构建 TensorRT FP32/FP16 engine，完成统一 benchmark | PyTorch/ORT/TRT 延迟与显存表 | 竞赛有效提交/改进 |
| Day24 | Triton model repository、动态 batching、FastAPI 网关和 Docker Compose | 可调用服务、Prometheus 指标和接口测试 | C++ 基础 1 小时 |
| Day25 | 并发压测、异常输入、README、结果图和演示视频；冻结项目 1 v1 | QPS/P50/P95/P99 报告和 v1 release | Jetson 当晚前准备到位 |

项目 1 的 Triton 是 v1 核心；INT8、gRPC、Kubernetes 和云端公开演示是 v2 增强，不阻塞 Day25 v1。

## Day26-Day30：Jetson 基线与项目 2 启动

Jetson 已购买并完成首次初始化，因此不再安排“选型和采购”，直接进入环境复查和板端实验。

| 天数 | 主任务 | 当天产出 |
| --- | --- | --- |
| Day26 | 复查 JetPack/TensorRT/CMake/GStreamer，建立 DeepStream 兼容性 ADR | Jetson C++ 工具链与兼容性报告 |
| Day27 | 建立 C++17/CMake 骨架、视频文件回放和 `IFrameSource` 接口 | PC/Jetson 可编译的 replay demo |
| Day28 | 创建项目 2 仓库；将 ONNX 移到 Jetson 并构建板端 FP16 engine | 独立仓库、板端 engine、detector 接口 |
| Day29 | GStreamer 文件输入、时间戳和有界帧队列 | 采集线程、drop-oldest 策略和队列测试 |
| Day30 | TensorRT C++ detector；对比 YOLOX-Nano/Tiny、720p/1080p 和功耗模式 | 第一版检测性能与设备基线表 |

TensorRT engine 在 PC 和 Jetson 上分别构建，不把 x86 engine 直接复制到 Jetson 使用。

## Day31-Day40：项目 2 v1 冲刺

项目仓库：`jetson-realtime-tracking-system`。

| 天数 | 主任务 | 当天产出 |
| --- | --- | --- |
| Day31 | 接入 ByteTrack C++，打通检测到稳定目标 ID | 检测/跟踪 pipeline 和结果视频 |
| Day32 | 使用 MOT17 + TrackEval 评估跟踪参数 | HOTA、IDF1、MOTA、ID switches 报告 |
| Day33 | 实现 ROI 闯入、方向越线、停留状态机和合成轨迹测试 | 事件规则模块和 CTest |
| Day34 | JSONL 事件协议、事件去重、截图和可选前后事件视频 | event schema、样例证据与 I/O 测试 |
| Day35 | USB UVC/RTSP 输入与断线重连；摄像头缺席时保留文件回放 | 多输入适配器和重连测试 |
| Day36 | 接入 `tegrastats`、队列深度、丢帧率和端到端延迟指标 | 设备与 pipeline metrics |
| Day37 | systemd、watchdog、优雅退出、离线缓存和日志轮转 | headless 服务与恢复脚本 |
| Day38 | 720p/1080p、Nano/Tiny、FP16 和功耗模式综合 benchmark | 性能/功耗/温度矩阵 |
| Day39 | 30 分钟 soak test、视频源故障和进程崩溃注入 | 稳定性与恢复报告 |
| Day40 | README 首屏演示、架构图、复现命令、指标表；冻结项目 2 v1 | 项目 2 v1 release |

项目 2 的 v1 必须使用 C++17、CMake、GStreamer、TensorRT C++ API 和 ByteTrack。视觉小车、DeepStream、CUDA 自定义预处理和 INT8 是增强项，不是 v1 完成条件。

## Day41-Day53：两个项目 v2、竞赛和工程补强

| 天数 | 主任务 | 当天产出 |
| --- | --- | --- |
| Day41 | 项目 1 数据增强、损失函数和错误分析优化 | 新旧 Dice/IoU 对比 |
| Day42 | Triton 并发、动态 batching、QPS 和 P95/P99 调优 | 服务压测报告 |
| Day43 | Docker Compose、依赖锁定、模型版本与指标监控 | 一键运行环境 |
| Day44 | 项目 1 单元/集成测试、异常输入和 README 完善 | 项目 1 v2 |
| Day45 | 项目 2 多视频场景、事件阈值和跟踪参数调优 | 跟踪/事件对比报告 |
| Day46 | C++ 代码质量、RAII、线程退出、CTest 和 CI 补强 | C++ 测试与静态检查 |
| Day47 | CUDA 预处理、零拷贝或 TensorRT buffer 复用优化 | 关键路径 profile 与对比 |
| Day48 | Jetson 60 分钟稳定性、资源监控和日志轮转 | 长跑测试报告 |
| Day49 | 选做 INT8 或更轻模型，不影响主交付 | 可选量化报告 |
| Day50 | 通过兼容性门禁后做 DeepStream 对照，或完成 RTSP/视觉小车扩展 | 一个明确增强功能或兼容性 ADR |
| Day51 | 主竞赛一次有记录的改进 | 实验表与新分数 |
| Day52 | 两个项目演示视频、截图、架构图和指标图 | 展示素材 |
| Day53 | 冻结两个项目 v2，核对从零复现流程 | 两个可投递项目 |

## Day54-Day60：就业交付

| 天数 | 主任务 | 当天产出 |
| --- | --- | --- |
| Day54 | GitHub 首页、置顶仓库和项目导航 | GitHub 作品集首页 |
| Day55 | 两个项目的简历 STAR/指标化描述 | 简历项目段落 |
| Day56 | 整理 30 个岗位 JD 和高频技能 | 岗位关键词表 |
| Day57 | ONNX、TensorRT、CUDA、模型优化面试题 | 部署面试题库 |
| Day58 | Linux、C++、网络、Docker 和深度学习基础面试题 | 基础面试题库 |
| Day59 | 模拟面试、简历修改和第一批投递清单 | 简历 v1、投递表 |
| Day60 | 总复盘、竞赛复盘和下一阶段计划 | 60 天总结报告 |

## 竞赛并行里程碑

| 时间 | 竞赛安排 | 完成标准 |
| --- | --- | --- |
| Day13-Day15 | 已报名两项天池；确定 CSIG 主赛候选和 Kaggle Digit Recognizer 练习赛 | 规则与候选表 |
| Day16-Day21 | 跑通 Digit Recognizer 或选定赛题 baseline | 本地/Notebook baseline |
| Day22-Day25 | 完成第一次平台接受的有效提交 | 排行榜分数和截图 |
| Day26-Day35 | CSIG 或最终主赛完成至少两次有记录实验 | 参数、指标和结论 |
| Day36-Day45 | 根据项目进度继续一次改进或停止冲榜 | 竞赛 README/复盘 |
| Day46-Day60 | 整理比赛产出，关注华为 ICT、嵌入式和机器人赛事 | 可写入简历的真实记录 |

实际进度：Kaggle `Digit Recognizer` 的第一次平台有效提交已于 Day14 提前完成。Day16-Day25 原有竞赛时间用于一次低成本优化或 CSIG 工业异常检测赛 baseline，不重复追求“第一次提交”。

## 不变的验收标准

- 项目必须独立仓库、可运行、可复现，并有环境、命令、指标和演示材料。
- 项目 1 证明语义分割训练、TensorRT/Triton GPU 推理服务和并发压测能力。
- 项目 2 证明 C++、GStreamer、Jetson TensorRT、持续视频流、跟踪、事件规则和设备可靠性能力。
- 两个公开项目仓库不得出现 `DayXX`、学习打卡或教程式措辞，使用 release、ADR、CHANGELOG 和 benchmark 记录开发过程。
- 竞赛不能只有报名，至少需要一次有效提交。
- 每日学习结束更新 `logs/dayXX.md`，README 同步进度索引。
