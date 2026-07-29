# Day 25 学习记录

日期：2026-07-28 至 2026-07-29

状态：已完成

## 今日主题

完成工业钢材缺陷分割 GPU 推理服务的并发压测、故障恢复、结果材料和
v1 质量门禁，完成项目 1 v1 实现。

今天从“单次请求能够成功”推进到“能够用可复现方法回答服务性能、容量、
失败行为和恢复能力”。

## 七小时时间分配

| 模块 | 时间 | 验收产出 |
| --- | ---: | --- |
| 压测指标与实验方法 | 0.75 小时 | 能解释 QPS、P50/P95/P99、并发和吞吐 |
| 并发压测工具 | 1.5 小时 | 固定 warmup、并发 1/2/4/8、JSON/Markdown 报告 |
| GPU 服务实测 | 1.25 小时 | 延迟、QPS、错误率和 Triton 指标 |
| 故障与恢复 | 1 小时 | 413/415/422/503、Triton 停止和恢复验证 |
| 结果报告与 README | 1 小时 | 性能表、限制、复现命令和容量建议 |
| v1 质量门禁 | 0.75 小时 | 测试、Ruff、Compose、措辞和大文件检查 |
| 版本整理与日志 | 0.75 小时 | v1.0.0 版本记录、Day25 日志和提交推送 |

## 今日入口状态

项目 1 已具备：

- U-Net + ResNet18 钢材缺陷分割模型。
- PyTorch、ONNX Runtime 和 TensorRT FP32/FP16 验证与 benchmark。
- TensorRT FP16 engine 和动态 batch profile `1..8`。
- Triton model repository 和动态 batching。
- FastAPI 图片校验、推理接口、健康检查和 Prometheus 指标。
- Docker Compose 编排 Triton、API 和 Prometheus。
- 90 项自动化测试。

Day24 单请求验证：

```text
image: 0002cc93b.jpg
HTTP: 200
detected classes: defect_1, defect_3
```

## 模块 1：压测指标与实验方法

### 延迟

单个请求从客户端发出到收到完整响应所经过的时间。

- `P50`：一半请求不超过该延迟，表示典型体验。
- `P95`：95% 请求不超过该延迟，观察慢请求。
- `P99`：99% 请求不超过该延迟，观察尾延迟。
- `mean`：平均值，容易被少量特别慢的请求影响，不能单独使用。

### 吞吐

- `QPS`：每秒成功处理的请求数。
- 并发数不是 QPS；并发表示同一时间允许多少请求处于执行或等待状态。
- 提高并发可能提高 QPS，也可能使排队和尾延迟增加。

### 错误率

```text
error_rate = failed_requests / total_requests
```

性能结果必须同时报告错误率。丢弃失败请求后得到的高 QPS 没有意义。

### 实验控制

本次压测固定：

- 相同图片、模型、threshold 和输入尺寸。
- 同一 Triton/TensorRT/Docker 版本。
- 每个并发档位先 warmup，再记录正式请求。
- 并发档位为 `1/2/4/8`。
- 分别记录客户端端到端延迟和 Triton 服务端阶段指标。

## 模块 2：并发压测工具

正式项目新增了可复现的服务负载测试工具：

- 使用 `httpx` 向 FastAPI `/v1/segment` 发送真实图片请求。
- 每个并发档位先 warmup，再执行固定数量的测量请求。
- 记录成功数、错误率、QPS、mean、P50、P95、P99。
- 在测量前后读取 Triton Prometheus counter，计算请求数、执行次数、
  平均 batch 和 queue 时间。
- 客户端设置 `trust_env=False`，避免普通代理环境变量进入本地压测。
- 对不可信的 Triton 阶段计数进行一致性检查，异常值记录为 `n/a`，
  不伪造结果。

新增文件：

```text
scripts/load_test_service.py
src/industrial_defect/load_testing.py
tests/test_load_testing.py
```

## 模块 3：最终服务负载测试

固定条件：

```text
image: 0002cc93b.jpg
threshold: 0.8
warmup: 5 requests / concurrency
measured: 100 requests / concurrency
concurrency: 1, 2, 4, 8
```

最终结果：

| 并发 | 成功 | 错误 | QPS | P50 | P95 | P99 |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 1 | 100 | 0 | 3.16 | 314.22 ms | 338.93 ms | 349.25 ms |
| 2 | 100 | 0 | 3.56 | 553.04 ms | 594.27 ms | 603.99 ms |
| 4 | 100 | 0 | 3.64 | 1080.18 ms | 1194.73 ms | 1220.48 ms |
| 8 | 100 | 0 | 3.79 | 2095.35 ms | 2154.96 ms | 2198.43 ms |

400 个测量请求全部返回 HTTP 200。

结论：

- 并发从 1 增加到 8，QPS 只提高约 19.9%。
- P95 从 338.93 ms 增加到 2154.96 ms。
- 单实例服务在约 3.8 QPS 附近饱和，继续提高并发主要增加排队和尾延迟。
- 延迟敏感场景推荐并发 1；并发 2 可换取有限吞吐提升；并发 4 和 8
  属于当前响应协议下的过载区。
- 单张图片的模型输出为 `4 x 256 x 1024` 个 FP32 logits，约 4 MiB。
  TensorRT engine 本身只需数毫秒，稠密输出的 Triton HTTP 传输和并发
  排队是当前主要容量边界。

## 模块 4：动态 batching A/B

只改变 Triton 动态 batching 配置，在并发 4 下比较：

| 配置 | QPS | P95 |
| --- | ---: | ---: |
| 动态 batching，preferred 4/8 | 3.56 | 1144.13 ms |
| 关闭动态 batching | 3.43 | 1199.69 ms |

动态 batching 带来小幅吞吐和尾延迟改善，因此保留：

```protobuf
dynamic_batching {
  preferred_batch_size: [4, 8]
  max_queue_delay_microseconds: 2000
}
```

这次实验说明动态 batching 不是无条件加速，必须在相同负载下做 A/B。

## 模块 5：故障与恢复

第一次停止 Triton 后，API 能正确返回 503，但故障请求等待接近一分钟。
检查代码发现：

- `configs/project.yaml` 声明了 `request_timeout_seconds: 5`。
- 运行时没有读取和使用该值。
- Triton HTTP 客户端因此沿用了更长的传输等待。

修复内容：

- `ServiceSettings` 新增 `request_timeout_seconds`。
- Compose 新增 `MODEL_REQUEST_TIMEOUT_SECONDS=5`。
- readiness 和 inference 使用 `asyncio.wait_for` 设置应用级期限。
- 新增慢模型契约测试，确保超时返回 503。

修复后验证：

| 检查 | Triton 停止 | Triton 恢复后 |
| --- | --- | --- |
| `/health/ready` | 约 1.18 秒返回 503 | 返回 200 |
| `/v1/segment` | 约 1.18 秒返回 503 | 返回 200 |

恢复后不需要重启 API，同一图片再次检测到 `defect_1` 和 `defect_3`。

## 模块 6：v1 质量门禁

正式项目完成：

- `90 passed`。
- `ruff check .` 通过。
- 本次修改文件 `ruff format --check` 通过。
- `git diff --check` 通过。
- 正式仓库禁用措辞扫描通过，没有 `DayXX`、学习打卡、教程练习、
  招聘展示或简历用途描述。
- README 新增服务负载结果和容量建议。
- 新增 `docs/service-load-test.md`、机器可读 JSON 和 `CHANGELOG.md`。
- Python 包和 FastAPI OpenAPI 版本更新为 `1.0.0`。

项目代码、数据集、模型权重和 TensorRT engine 的边界保持正确：

- 原始数据、权重和 engine 不进入 Git。
- 正式项目仓库只描述系统、接口、指标、限制和部署方法。
- 学习过程与问答只保留在本学习仓库。

## 面试追问

### 为什么 QPS 提高但 P95 变差

系统接近饱和后，更多并发请求会排队。少量吞吐提升是以更长等待时间换来的，
因此容量评估必须同时观察 QPS、错误率和尾延迟。

### 为什么不能只报告 TensorRT engine 延迟

用户实际等待还包含图片上传、校验、预处理、Triton 调度、输出传输、
后处理和 JSON 响应。模型延迟和服务延迟回答的是不同问题。

### 为什么故障测试很重要

单次正常推理只能证明正常路径可用。服务还必须在依赖不可用时快速返回明确
状态，并在依赖恢复后自动恢复，否则会占满 worker 或需要人工重启。

## 今日产出

正式项目仓库：

- 并发负载测试模块、CLI 和单元测试。
- 400 请求正式负载报告及机器可读结果。
- 动态 batching A/B 结论。
- 模型调用超时修复和恢复验证。
- README、服务文档、benchmark 文档和 changelog。
- v1.0.0 代码与文档质量门禁。

学习仓库：

- Day25 完整记录。
- 项目 1 v1 状态更新。

## 当前进度

已完成：

- 并发压测工具和单元测试。
- GPU 服务正式压测。
- 动态 batching A/B。
- 故障、超时和自动恢复验证。
- 服务报告、README 和 v1.0.0 变更记录。
- 全仓库 90 项测试、Ruff 和措辞检查。

发布状态：

- 项目自有源码和文档采用 MIT License。
- 第三方数据、权重和 NVIDIA 组件继续遵循各自条款。
- annotated tag `v1.0.0` 指向包含许可证的提交。
- GitHub Release `v1.0.0` 已公开发布。

## 明日计划

进入 Day26：

- Jetson Orin Nano 设备基线与 C++ 工具链检查。
- GStreamer 基础管线和本地视频输入验证。
- 为项目 2 的视频流接口和 CMake 工程做准备。
