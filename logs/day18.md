# Day 18 学习记录

日期：2026-07-23

主题：项目 1 立项、独立仓库创建与生产级工程骨架

状态：已完成

## 今日目标

- 正式启动项目 1：工业钢材缺陷分割与 GPU 推理服务。
- 冻结任务、数据契约、系统架构、技术栈和 v1 验收边界。
- 创建独立公开 GitHub 仓库，不把正式项目代码混入学习仓库。
- 建立可运行、可测试、可持续演进的工程骨架。
- 保证公开仓库只展示工程开发过程，不出现 DayXX、学习记录或课程式描述。

## 项目定位

项目仓库：

```text
https://github.com/Nefelibata134/industrial-defect-inference-service
```

任务使用 Severstal: Steel Defect Detection 数据集：

```text
12,568 张训练图像
4 类匿名钢材表面缺陷
RLE 像素级掩码
```

项目目标不是只训练一个模型，而是完成以下生产部署链路：

```text
数据校验与固定划分
  -> PyTorch U-Net + ResNet18
  -> ONNX 导出与数值一致性
  -> TensorRT FP32/FP16
  -> Triton Inference Server
  -> FastAPI gateway
  -> Docker Compose
  -> 并发压测与 Prometheus 指标
```

## 今日工程决策

### 1. 项目与学习仓库分离

- 学习仓库只保存路线、日志、proposal、实验摘要和踩坑记录。
- 正式项目仓库保存源码、配置、测试、CI、ADR、benchmark、README 和发布记录。
- 项目仓库的 commit、README 和文档不使用 `DayXX` 或“学习任务”等措辞。

### 2. 数据规模与指标

12,568 张图像足以支撑完整的工业分割项目。项目重点不是盲目扩大数据量，而是保证：

- 数据完整性与 RLE 掩码正确性。
- 固定且可复现的训练、验证、测试划分。
- 每类 Dice、IoU、Precision、Recall。
- 图像级漏检率与失败样例分析。
- PyTorch、ONNX Runtime、TensorRT 的一致性和性能报告。

### 3. 服务端工程范围

v1 必须包含：

- PyTorch 训练与评估。
- ONNX 导出和 parity check。
- TensorRT FP16 engine 和 benchmark。
- Triton model repository 与动态 batching。
- FastAPI 网关、健康检查和错误处理。
- Docker Compose、负载测试和可观测指标。

INT8、gRPC、Kubernetes 和云端公开演示属于后续增强，不阻塞 v1。

## 创建的工程骨架

```text
.github/workflows/ci.yml
.editorconfig
.gitignore
CHANGELOG.md
README.md
configs/project.yaml
docs/adr/0001-dataset-and-task.md
docs/adr/0002-serving-architecture.md
docs/benchmark-protocol.md
docs/dataset-card.md
docs/system-design.md
pyproject.toml
scripts/check_environment.py
scripts/inspect_dataset.py
src/industrial_defect/__init__.py
src/industrial_defect/config.py
src/industrial_defect/dataset.py
tests/test_config.py
tests/test_dataset.py
```

其中：

- `dataset card` 记录数据来源、任务、类别、许可核对和禁止提交原始数据的规则。
- ADR 记录数据集选择与 Triton/FastAPI 服务架构决策。
- benchmark protocol 固定硬件、软件、warmup、重复次数和延迟统计口径。
- 数据校验器检查文件、图像尺寸、RLE、类别和配置。
- GitHub Actions 运行 lint 和 CPU 单元测试。

## 验证结果

项目环境：

```text
Conda environment: defect310
Python: 3.10.20
```

本地验证：

```text
ruff check .
All checks passed

pytest -q
4 passed
```

由于 Day19 才下载数据，当前运行数据检查器时报告数据集缺失属于预期行为，不能记录成“数据验证通过”。

## GitHub 结果

公开仓库已创建：

```text
Nefelibata134/industrial-defect-inference-service
visibility: public
default branch: main
```

首个提交：

```text
f649a82 chore: initialize production inference service
```

本地 `main` 已跟踪 `origin/main`，本地和远端完整提交 SHA 一致：

```text
f649a8292d913715a24ffed51587469ac8bf5517
```

GitHub 远端 README 已通过连接器读取验证。项目仓库保持干净，没有提交数据集、模型权重、TensorRT engine 或学习日志。

## 遇到的问题

WSL 首次通过 HTTPS 推送时没有可用的 Git 凭据，导致等待认证。随后为该项目仓库配置 Windows Git Credential Manager：

```text
/mnt/c/Program Files/Git/mingw64/bin/git-credential-manager.exe
```

包含空格的路径必须正确转义。修正后推送成功。该配置只影响项目仓库的 Git 凭据调用，不改变项目代码。

## 今日产出

- 独立公开项目仓库。
- 生产级 README、系统设计和两份 ADR。
- 数据集卡片和 benchmark 协议。
- Python package、配置加载和数据校验器。
- 4 个通过的 CPU 单元测试。
- GitHub Actions CI。
- 本地与远端一致的首个可追踪提交。

## 今日完成清单

- [x] 冻结 Severstal 分割任务和 v1 边界。
- [x] 明确项目 1 与项目 2 的能力差异。
- [x] 创建独立公开 GitHub 仓库。
- [x] 建立工程目录、配置、文档和测试。
- [x] 通过 Ruff 与 Pytest。
- [x] 提交并推送到 GitHub。
- [x] 验证远端 README 和默认分支。
- [x] 保持正式项目仓库无学习化措辞。

## 明日计划

Day19：

- 下载并核对 Severstal 数据集目录与文件数量。
- 运行数据校验器并修复真实数据问题。
- 实现和测试 RLE 编解码。
- 统计类别、正负样本和多标签分布。
- 创建固定随机种子的训练、验证、测试 manifest。
- 输出掩码可视化与数据报告。
- 预留约 1 小时补 C++ 基础。
