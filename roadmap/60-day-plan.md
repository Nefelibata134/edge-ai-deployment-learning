# 60 天模型部署学习路线（历史进度与候选任务）

最后统一日期：2026-08-28

Day1-Day41 保留为历史学习索引，Day42-Day60 保留为可选任务池。自
2026-08-28 起不再要求按 Day 顺序执行：用户的当次明确需求优先，可以跳过、
替换或重新排序任何未完成项。上下文恢复后不得从本文件自动推断下一任务；只有
用户明确说“开始 DayXX”时，才按对应 Day 组织学习计划和日志。

## 总目标

- 完成两个独立 GitHub 项目，而不是等到最后两周才开始项目。
- 项目 1：工业钢材缺陷分割与 GPU 推理服务。
- 项目 2：Jetson C++ 实时跟踪与安全事件分析系统。
- 暑假结束前形成简历、项目 README、演示视频、性能报告和面试准备材料。

## 参考并行节奏

| 主线 | 每周投入 | 执行规则 |
| --- | ---: | --- |
| 部署学习与两个项目 | 31-34 小时 | 根据用户当前需求选择项目加固项，不强制按 Day 顺序 |
| C++ / Linux / 系统工程 | 4-6 小时 | 结合正式项目补语言、并发、进程、网络和故障处理 |
| 就业准备 | 2-3 小时 | Day41 起逐步沉淀技术讲解、岗位关键词和项目材料 |
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
| Day13 | ORT CUDA Provider 与 CPU/CUDA benchmark | CUDA 4.14x 推理加速与 Provider 验证 | 已完成 |

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
| Day18 | 创建项目仓库，冻结 Severstal 分割任务、ADR、目录和数据契约 | 工程骨架、dataset card、数据校验器 | 数据路径与仓库边界核对 |
| Day19 | 下载并检查 12,568 张图像，解码 RLE 掩码，固定训练/验证划分 | 数据报告、掩码可视化和 split 文件 | C++ 基础 1 小时 |
| Day20 | 训练 U-Net + ResNet18 baseline | checkpoint、loss 曲线和训练配置 | 训练日志与实验命名 |
| Day21 | 验证集评估和错误分析 | Dice、IoU、Precision、Recall、漏检与失败样例 | Linux/Git 复习 |
| Day22 | 导出 ONNX，验证 PyTorch/ORT 数值与掩码一致性 | ONNX 模型、parity report | ONNX 导出与结构检查 |
| Day23 | 构建 TensorRT FP32/FP16 engine，完成统一 benchmark | PyTorch/ORT/TRT 延迟与显存表 | benchmark 结果归档 |
| Day24 | Triton model repository、动态 batching、FastAPI 网关和 Docker Compose | 可调用服务、Prometheus 指标和接口测试 | C++ 基础 1 小时 |
| Day25 | 并发压测、异常输入、README、结果图和演示视频；冻结项目 1 v1 | QPS/P50/P95/P99 报告和 v1 release | Jetson 当晚前准备到位 |

项目 1 的 Triton 是 v1 核心；INT8、gRPC、Kubernetes 和云端公开演示是 v2 增强，不阻塞 Day25 v1。

## Day26-Day30：Jetson 基线与项目 2 启动

Jetson 已购买并完成首次初始化，因此不再安排“选型和采购”，直接进入环境复查和板端实验。

| 天数 | 主任务 | 当天产出 |
| --- | --- | --- |
| Day26 | 复查 JetPack/TensorRT/CMake/GStreamer，建立 DeepStream 兼容性 ADR | Jetson C++ 工具链与兼容性报告 |
| Day27 | 建立 C++17/CMake 骨架、视频文件回放和 `IFrameSource` 接口 | PC/Jetson 可编译的 replay demo |
| Day28 | 创建项目 2 仓库；将 ONNX 移到 Jetson 并构建板端 FP16 engine；IMX219 到货后做 CSI 冒烟测试 | 独立仓库、板端 engine、detector 接口、摄像头能力快照 |
| Day29 | GStreamer 文件/CSI 输入、时间戳和有界帧队列 | 采集线程、drop-oldest 策略和队列测试 |
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
| Day35 | IMX219 CSI/RTSP 输入与断线重连；保留文件回放，USB UVC 作为可选适配器 | 多输入适配器和重连测试 |
| Day36 | 接入 `tegrastats`、队列深度、丢帧率和端到端延迟指标 | 设备与 pipeline metrics |
| Day37 | systemd、watchdog、优雅退出、离线缓存和日志轮转 | headless 服务与恢复脚本 |
| Day38 | 对比检测、跟踪、事件分析和证据写入各阶段开销，并复测 720p/1080p 与功耗模式 | 完整流水线延迟、功耗和温度报告 |
| Day39 | 60 分钟 soak test、视频源故障和进程崩溃注入 | 稳定性与恢复报告 |
| Day40 | README 首屏演示、架构图、复现命令、指标表；冻结项目 2 v1 | 项目 2 v1 release |

项目 2 的 v1 必须使用 C++17、CMake、GStreamer、TensorRT C++ API 和 ByteTrack。视觉小车、DeepStream、CUDA 自定义预处理和 INT8 是增强项，不是 v1 完成条件。

## Day41-Day53：两个项目 v2 工程加固

| 天数 | 主任务 | 当天产出 |
| --- | --- | --- |
| Day41 | 项目 1 质量验收：类别阈值、错误切片、失败样例和固定测试集复核 | 模型质量与限制报告 |
| Day42 | Triton 并发、动态 batching、QPS 和 P95/P99 瓶颈分析 | 服务压测与容量报告 |
| Day43 | 网关超时、请求大小限制、背压、异常输入和过载行为 | 服务可靠性测试 |
| Day44 | Docker Compose、依赖锁定、模型版本、健康检查和指标监控 | 可复现运行与可观测性检查 |
| Day45 | 服务中断、Triton 不可用和模型加载失败注入，验证恢复与错误协议 | 故障注入报告 |
| Day46 | 在干净环境执行项目 1 全量回归、README 与 release checklist | 项目 1 v2 release |
| Day47 | 项目 2 C++ 代码质量：RAII、线程退出、静态检查、CTest 和 CI | C++ 质量报告 |
| Day48 | profile 预处理、内存复制、TensorRT buffer 与后处理，按证据优化热点 | 关键路径 profile 与前后对比 |
| Day49 | 建立真实视频事件回归集，核对漏报、误报、证据文件与事件协议 | 事件验收报告 |
| Day50 | 启动配置、engine 元数据、输入契约和不兼容 artifact 的 fail-fast 检查 | 部署契约与错误矩阵 |
| Day51 | 受控评估 INT8；只有精度、延迟和工程复杂度通过门禁才保留 | FP16/INT8 质量性能对比 |
| Day52 | 最终配置进行 2 小时稳定性与断流/重启恢复测试 | v2 长跑与恢复报告 |
| Day53 | 两台环境复现、演示素材、架构与限制文档；冻结项目 2 v2 | 项目 2 v2 release |

## Day54-Day60：就业交付

| 天数 | 主任务 | 当天产出 |
| --- | --- | --- |
| Day54 | GitHub 首页、置顶仓库、项目导航和公开仓库内容审计 | GitHub 项目导航 |
| Day55 | 两个项目的指标化简历描述，并逐项核对可证明证据 | 简历项目段落 |
| Day56 | 整理 30 个岗位 JD，建立技能覆盖、缺口和投递优先级 | 岗位关键词与差距表 |
| Day57 | 从入口到输出讲清两个项目的数据流、线程模型、错误处理和性能指标 | 项目技术讲解稿与架构图 |
| Day58 | ONNX/TensorRT/CUDA、C++/Linux、网络/Docker 高频追问与现场排错 | 面试题库和排错记录 |
| Day59 | 两轮模拟面试、简历修改和首批岗位投递 | 简历 v1、模拟面试记录和投递表 |
| Day60 | 60 天总复盘、两个项目最终验收和下一阶段 30 天计划 | 总结报告与后续执行表 |

## 不变的验收标准

- 项目必须独立仓库、可运行、可复现，并有环境、命令、指标和演示材料。
- 项目 1 证明语义分割训练、TensorRT/Triton GPU 推理服务和并发压测能力。
- 项目 2 证明 C++、GStreamer、Jetson TensorRT、持续视频流、跟踪、事件规则和设备可靠性能力。
- 两个公开项目仓库不得出现 `DayXX`、学习打卡或教程式措辞，使用 release、ADR、CHANGELOG 和 benchmark 记录开发过程。
- 每日学习结束更新 `logs/dayXX.md`，README 同步进度索引。
