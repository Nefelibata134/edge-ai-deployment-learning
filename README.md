# 模型部署学习计划

目标：用两个月建立“能找工作、能展示作品、能继续打竞赛”的模型部署能力。

主线：

1. 打牢部署基础：Linux、Python、C++、深度学习推理、ONNX。
2. 掌握推理优化：TensorRT、量化、性能分析、吞吐/延迟权衡。
3. 上手边缘设备：Jetson Orin Nano、JetPack、CUDA、DeepStream。
4. 做出作品集：至少 2 个可演示项目，1 个完整边缘 AI 项目。
5. 面向就业：沉淀简历项目、GitHub 记录、面试题、岗位投递方向。

建议节奏：

- 你的个人节奏：每天 7 小时，基础学习、动手实验、项目开发、性能测试和复盘同步推进。
- 每天都写学习日志，哪怕只有 10 行。
- 每周至少产出一个能运行的 demo、脚本、报告或性能对比图。

学习进度索引：

| 天数 | 日期 | 主题 | 记录 |
| --- | --- | --- | --- |
| Day 01 | 2026-07-02 | 明确方向、路线、GitHub 仓库和本机环境检查 | `logs/day01.md` |
| Day 02 | 2026-07-03 | WSL2 Ubuntu 与 Linux 基础命令 | `logs/day02.md` |
| Day 03 | 2026-07-03 | Linux 权限、包管理、文本编辑与 Python 环境准备 | `logs/day03.md` |
| Day 04 | 2026-07-04 | Conda、pip 与 Python 3.10 部署环境 | `logs/day04.md` |
| Day 05 | 2026-07-05 | Python 基础、文件读写与 OpenCV 读图 | `logs/day05.md` |
| Day 06 | 2026-07-05 | OpenCV 颜色通道、裁剪、画框与检测结果可视化 | `logs/day06.md` |
| Day 07 | 2026-07-05 | YOLO 预处理：letterbox、归一化与 NCHW 输入 | `logs/day07.md` |
| Day 08 | 2026-07-06 | YOLO 导出 ONNX 与模型结构检查 | `logs/day08.md` |
| Day 09 | 2026-07-06 | ONNX Runtime 加载 YOLO 并运行原始推理 | `logs/day09.md` |
| Day 10 | 2026-07-08 | YOLO ONNX 后处理：坐标转换、NMS 与画框 | `logs/day10.md` |
| Day 11 | 2026-07-10 | ONNX Runtime 性能测试：warmup、延迟、P50/P95 与 FPS（已完成） | `logs/day11.md` |
| Day 12 | 2026-07-11 | YOLO 动态输入尺寸：320/480/640 性能与检测结果对比（已完成） | `logs/day12.md` |
| Day 13 | 2026-07-12 | ONNX Runtime CUDA Provider、CPU/CUDA benchmark 与个人竞赛初筛（已完成） | `logs/day13.md` |

文档：

- `AGENTS.md`：上下文重置后的协作记忆、恢复顺序和仓库边界。
- `roadmap/60-day-plan.md`：当前唯一的 Day1-Day60 逐日执行表；发生计划冲突时以它为准。
- `roadmap/personal-60-day-plan.md`：个人目标、时间分配、项目和竞赛原则。
- `logs/template.md`：每日学习记录模板。
- `logs/day01.md`：第一天学习记录。
- `logs/day02.md`：第二天学习记录。
- `logs/day03.md`：第三天学习记录。
- `logs/day04.md`：第四天学习记录。
- `logs/day05.md`：第五天学习记录。
- `logs/day06.md`：第六天学习记录。
- `logs/day07.md`：第七天学习记录。
- `logs/day08.md`：第八天学习记录。
- `logs/day09.md`：第九天学习记录。
- `logs/day10.md`：第十天学习记录。
- `logs/day11.md`：第十一天学习记录。
- `logs/day12.md`：第十二天学习记录。
- `logs/day13.md`：第十三天学习记录。
- `projects/README.md`：作品集项目规划。
- `career/direction-and-jobs.md`：就业方向和岗位建议。
- `career/competition-plan.md`：竞赛准备建议。
- `career/competition-shortlist-2026-summer.md`：2026 暑假个人赛候选与规则核对表。
- `hardware/jetson-buying-list.md`：Jetson Orin Nano 采购清单。
- `hardware/jetson-arrival-check.md`：Jetson Orin Nano 到货检查记录。
- `github-workflow.md`：GitHub 记录和提交规范。
