# 60 天模型部署学习路线

这是一版就业作品集导向路线。默认你每天可投入 2.5 到 4 小时，已有 Python 基础，深度学习和 Linux 处于可补强阶段。后续可以按你的实际水平调整。

## 阶段 1：部署基础与环境能力（Day 1-7）

| 天数 | 主题 | 当天任务 | 产出 |
| --- | --- | --- | --- |
| Day 1 | 目标拆解 | 明确目标岗位、学习时间、硬件预算；整理电脑环境 | `logs/day01.md` |
| Day 2 | Linux 基础 | 文件、进程、权限、网络、常用命令 | Linux 命令清单 |
| Day 3 | Python 工程化 | venv/conda、pip、requirements、argparse、logging | 可复现实验脚本 |
| Day 4 | Git/GitHub | commit、branch、README、issue、项目结构 | 本仓库首个提交 |
| Day 5 | 深度学习推理流程 | PyTorch 模型加载、预处理、后处理、batch | 本地推理 demo |
| Day 6 | 性能指标 | latency、throughput、FPS、显存、warmup、benchmark 方法 | benchmark 脚本 |
| Day 7 | 周复盘 | 整理本周知识图谱和问题清单 | 周报 1 |

## 阶段 2：ONNX 与推理引擎入门（Day 8-14）

| 天数 | 主题 | 当天任务 | 产出 |
| --- | --- | --- | --- |
| Day 8 | ONNX 基础 | 理解计算图、opset、动态 shape | ONNX 笔记 |
| Day 9 | PyTorch 导出 ONNX | 导出分类模型，检查输入输出 | `export_onnx.py` |
| Day 10 | ONNX Runtime | CPU/GPU 推理、provider、IOBinding 初步 | ORT 推理 demo |
| Day 11 | 模型可视化与排错 | Netron、shape mismatch、unsupported op | 排错记录 |
| Day 12 | 图像分类部署 | ResNet/MobileNet 完整部署流程 | 分类项目 demo |
| Day 13 | 性能对比 | PyTorch vs ONNX Runtime 延迟对比 | 对比表 |
| Day 14 | 周复盘 | 写一篇“模型从训练到部署”的 README | 周报 2 |

## 阶段 3：TensorRT 核心能力（Day 15-24）

| 天数 | 主题 | 当天任务 | 产出 |
| --- | --- | --- | --- |
| Day 15 | TensorRT 概念 | engine、builder、parser、profile | TRT 概念图 |
| Day 16 | trtexec | ONNX 转 engine，查看日志与性能 | trtexec 命令记录 |
| Day 17 | FP16 推理 | 开启 FP16，比较速度和精度 | FP32/FP16 对比 |
| Day 18 | INT8 量化概念 | calibration、scale、精度损失 | INT8 笔记 |
| Day 19 | TensorRT Python API | 加载 engine、绑定输入输出、执行推理 | TRT Python demo |
| Day 20 | 动态 shape | optimization profile、batch 变化 | 动态 batch demo |
| Day 21 | 常见错误 | unsupported op、plugin、workspace、版本兼容 | 错误手册 |
| Day 22 | YOLO 导出 | 选择 YOLOv8/YOLO11 小模型，导出 ONNX | 检测 ONNX |
| Day 23 | YOLO TensorRT 推理 | 前处理、NMS、后处理、可视化 | 检测 TRT demo |
| Day 24 | 周复盘 | 写“TensorRT 优化前后对比报告” | 周报 3 |

## 阶段 4：Jetson Orin Nano 与边缘部署（Day 25-36）

| 天数 | 主题 | 当天任务 | 产出 |
| --- | --- | --- | --- |
| Day 25 | Jetson 选型与采购 | 确认开发板、摄像头、电源、散热、存储卡/SSD | 采购清单 |
| Day 26 | JetPack 环境 | 刷机、CUDA/cuDNN/TensorRT、jtop | 环境截图与记录 |
| Day 27 | Jetson 基准测试 | CPU/GPU/内存/温度/功耗/FPS 测试 | 设备基线报告 |
| Day 28 | 摄像头输入 | USB/CSI 摄像头、OpenCV 采集 | 摄像头 demo |
| Day 29 | Jetson 上运行 ONNX | ONNX Runtime 或 TensorRT 测试 | 板端推理记录 |
| Day 30 | Jetson 上运行 YOLO | 实时目标检测 | 实时检测 demo |
| Day 31 | 性能调优 | nvpmodel、jetson_clocks、分辨率、batch | 调优报告 |
| Day 32 | DeepStream 入门 | pipeline、source、infer、sink | DeepStream demo |
| Day 33 | DeepStream 配置 | 修改模型、标签、输入源 | 自定义配置 |
| Day 34 | 视频流部署 | RTSP/本地视频/摄像头流 | 视频流 demo |
| Day 35 | 边缘端稳定性 | 自启动、日志、异常恢复 | 部署脚本 |
| Day 36 | 周复盘 | 整理 Jetson 踩坑手册 | 周报 4 |

## 阶段 5：模型服务化与工程能力（Day 37-46）

| 天数 | 主题 | 当天任务 | 产出 |
| --- | --- | --- | --- |
| Day 37 | FastAPI 推理服务 | REST API 封装模型推理 | API demo |
| Day 38 | Docker 基础 | Dockerfile、镜像、容器、挂载 | 推理镜像 |
| Day 39 | Docker 部署模型 | 将 ONNX/TensorRT 服务容器化 | 可运行容器 |
| Day 40 | Triton 概念 | model repository、config.pbtxt、backend | Triton 笔记 |
| Day 41 | Triton ONNX 模型 | 部署分类或检测模型 | Triton demo |
| Day 42 | 并发与压测 | wrk/ab/locust，统计 QPS 和 P95 | 压测报告 |
| Day 43 | 监控与日志 | 记录输入、耗时、错误、资源占用 | 日志方案 |
| Day 44 | 项目结构整理 | config、scripts、docs、tests | 标准项目骨架 |
| Day 45 | 简单测试 | 单元测试预处理/后处理/接口 | 测试用例 |
| Day 46 | 周复盘 | 写“模型服务化部署指南” | 周报 5 |

## 阶段 6：作品集项目冲刺（Day 47-56）

| 天数 | 主题 | 当天任务 | 产出 |
| --- | --- | --- | --- |
| Day 47 | 选题定稿 | 从 3 个项目里选 1 个主项目 | 项目 proposal |
| Day 48 | 数据与模型 | 准备数据、选择预训练模型 | baseline |
| Day 49 | PC 端推理 | 跑通 PyTorch/ONNX/TensorRT | 本地 demo |
| Day 50 | Jetson 端移植 | 移到开发板运行 | 板端 demo |
| Day 51 | 性能优化 | FP16/INT8、分辨率、pipeline 优化 | 优化报告 |
| Day 52 | 可视化展示 | 视频、截图、指标表、README | 展示材料 |
| Day 53 | 工程包装 | Docker/脚本/一键启动/配置文件 | 可复现项目 |
| Day 54 | 边界测试 | 异常输入、低光照、多人/多目标等 | 测试记录 |
| Day 55 | 简历化表达 | 写项目亮点、难点、指标、个人贡献 | 简历项目段落 |
| Day 56 | 周复盘 | 完成主项目 README | 周报 6 |

## 阶段 7：就业准备与竞赛准备（Day 57-60）

| 天数 | 主题 | 当天任务 | 产出 |
| --- | --- | --- | --- |
| Day 57 | 岗位画像 | 整理 30 个目标岗位 JD，提取高频技能 | 岗位关键词表 |
| Day 58 | 面试准备 | TensorRT/ONNX/Linux/C++/深度学习面试题 | 面试题库 |
| Day 59 | 简历与 GitHub | 完善简历、GitHub 首页、项目 README | 简历 v1 |
| Day 60 | 总复盘 | 复盘两个月成果，制定下一阶段投递计划 | 总结报告 |

## 推荐主项目方向

优先做一个能在 Jetson Orin Nano 上实时跑的项目：

1. 轻量级目标检测：YOLO + TensorRT + 摄像头实时检测。
2. 工业缺陷检测：分类/检测 + 性能优化报告。
3. 智能交通感知：车辆/行人检测 + 视频流部署。
4. 安全帽/工服检测：贴近工程落地，适合简历展示。
5. 手势识别或姿态估计：适合演示，但要控制模型复杂度。

项目 README 建议必须包含：

- 项目背景和应用场景。
- 模型结构、输入输出、部署流程。
- PC 与 Jetson 的性能对比。
- FP32/FP16/INT8 的速度和精度对比。
- 一键运行方式。
- 演示图片或视频。

