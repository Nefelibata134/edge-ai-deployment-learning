# Day 36：统一运行指标与 Jetson 设备遥测

日期：2026-08-20

状态：已完成

## 今日目标

- 将终端中的流水线统计变成版本化、机器可读的 JSON 契约。
- 使用后台 `tegrastats` 采样 CPU、GPU、RAM、温度和输入功耗。
- 保证设备采样不进入采集或 TensorRT 推理线程。
- 同时保留启动预热和稳态运行的丢帧口径。
- 在 IMX219 + TensorRT + ByteTrack 真机链路中验证指标准确性。

## 7 小时计划

| 时间 | 内容 | 验收标准 |
| --- | --- | --- |
| 1.0 小时 | 审计现有队列、推理、跟踪、事件和 `tegrastats` 指标 | 明确已有统计与缺失契约 |
| 1.0 小时 | 设计版本化 JSON schema | 字段、单位和不可用语义固定 |
| 1.5 小时 | 实现后台 `tegrastats` 采样器 | 不阻塞实时线程，可停止和汇总 |
| 1.0 小时 | 汇总吞吐、丢帧、重连和分阶段 P50/P95 | 一份报告覆盖 pipeline 和 device |
| 1.0 小时 | 原子写出和失败降级 | 不发布半个 JSON，遥测失败不终止推理 |
| 1.0 小时 | 本机与 Jetson 回归 | 单元测试、CSI/TensorRT 真机均通过 |
| 0.5 小时 | 用户参数修改、文档和提交 | 完成会运行、能解释、能修改 |

## 为什么需要统一指标

之前流水线已经在进程结束时打印队列、丢帧、推理和端到端延迟，Jetson benchmark 也能单独记录 `tegrastats`，但二者没有统一的机器可读契约。监控或服务管理若依赖终端文本，会面临字段位置变化、单位不明确和设备日志与本次运行无法对应的问题。

新数据流为：

```text
capture / TensorRT / ByteTrack / events
  -> in-process counters and latency samples

tegrastats child process
  -> pipe
  -> background parser thread
  -> device samples

run finished
  -> aggregate pipeline + device summaries
  -> temporary JSON
  -> atomic rename
  -> versioned runtime metrics document
```

设备采样只在传入 `--metrics-json` 时启动。`tegrastats` 缺失或失败时，流水线指标仍写出，设备部分使用 `available=false` 和错误信息表达不可用，不能用数值 0 冒充真实设备状态。

## 三类指标

### Counter

`produced_frames`、`dropped_frames`、`total_detections` 和 `restart_attempts` 是一次运行中持续累计的计数器。

### Gauge

GPU 利用率、温度、功耗和 RAM 是某个采样时刻的状态值，运行结束后汇总 mean、P95 和 max。

### Window summary

推理和端到端延迟来自本次测量窗口，输出 mean、P50、P95 和 max。P95 表示 95% 的样本延迟不超过该值，比只有平均值更能暴露尾延迟。

## 丢帧口径

第一次 500 ms 采样实验得到：

```text
warmup_dropped=7
dropped=0
dropped_total=7
drop_rate_percent=0.0
```

7 帧全部在 TensorRT 第一次执行的预热阶段被最新帧队列丢弃。稳态丢帧率只评价正式测量窗口：

```text
100 * dropped / (measured_frames + dropped)
```

因此该次结果是 `0 / (300 + 0) = 0%`。`dropped_total` 仍被保留，用于暴露启动阶段压力。如果正式处理 300 帧并丢弃 3 帧，小数比例约为 `0.0099`，百分比为 `0.99%`。

## 自动化验证

`tegrastats` 解析和汇总：

```text
parser=true
summary=true
invalid_line=true
status=PASS
```

JSON schema 与原子发布：

```text
schema=true
pipeline=true
latency=true
device=true
replace=true
atomic=true
status=PASS
```

完整本机回归：

- C++：13/13 tests passed。
- Python：10/10 tests passed。

## Jetson 真机结果

固定配置为 IMX219 1280x720/60 FPS 采集、30 FPS 应用输出、YOLOX-Nano TensorRT FP16、30 帧预热、300 帧测量和容量 2 的最新帧队列。

| 遥测间隔 | 设备样本 | 推理 P95 | 端到端 P95 | 有效 FPS | 稳态丢帧率 |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 500 ms | 22 | 12.830 ms | 13.342 ms | 30.037 | 0.00% |
| 1000 ms | 11 | 12.821 ms | 13.389 ms | 30.093 | 0.00% |

500 ms 实验还记录：平均输入功耗 7.164 W、最大输入功耗 7.315 W、最大 GPU 利用率 43%、最大 GPU 温度 52.187 C、最大 RAM 使用 2166 MiB。1000 ms 实验平均输入功耗为 7.230 W。

采样间隔翻倍后样本数正好减半，推理 P95 只变化 0.009 ms，端到端 P95 只变化 0.047 ms，属于正常运行波动。这验证了设备遥测没有进入实时关键路径。

## 真机发现并修复的问题

第一次 JSON 输出中，`power_mean_w` 是 `[7.1635]`，错误地成为单元素数组。原因是 `nlohmann::json{value}` 使用 initializer-list 语义。改用 `nlohmann::json(value)` 后恢复为数字标量，并新增 `is_number()` 回归断言。

修复后的真机输出：

```text
power_mean_w: 7.229545454545454
power_mean_type: float
scalar_fixed: True
```

这说明 schema 测试不能只检查字段存在，还必须检查字段类型和单位。

## 用户修改验证

用户将 `--tegrastats-interval-ms` 从 500 修改为 1000，并提前预测设备样本约从 22 减少到 11，而推理 P95 基本不变。真机结果与预测一致。

500 ms 是当前默认值，因为在无可测推理开销的前提下能更快发现短时功耗、利用率和温度变化；1000 ms 适合只需要粗粒度设备状态的运行场景。

## 会运行、能解释、能修改

### 会运行

- 构建并运行 `tegrastats` 解析与 JSON schema 检查。
- 在 IMX219、TensorRT 和 ByteTrack 同时运行时生成统一指标 JSON。
- 使用 Python 验证 schema、字段类型、延迟和设备统计。

### 能解释

- 区分 counter、gauge 和窗口延迟汇总。
- 解释总丢帧与稳态丢帧率为什么不同。
- 解释设备遥测失败为何不应导致推理失败。
- 解释为什么后台采样间隔不改变模型检测间隔。

### 能修改

- 修改遥测采样间隔并预测样本数变化。
- 使用真机数据确认采样不影响推理 P95。
- 根据真实 JSON 输出识别并验证标量类型修复。

## 今日产出

- `JetsonTelemetrySampler` 后台进程采样与线程安全汇总。
- 版本 1 runtime metrics JSON 契约。
- 原子 JSON 发布和不可用设备状态语义。
- 队列、丢帧、恢复、检测、跟踪、事件、延迟与设备指标统一报告。
- 500 ms / 1000 ms 真机采样开销对照报告。
- 正式项目提交：
  - `72e3b57 feat: publish unified runtime telemetry`
  - `e134d92 fix: encode telemetry summaries as scalars`
  - `9032e1a docs: record runtime telemetry overhead`

## 复盘

- 监控指标首先是数据契约，字段类型和单位与数值本身同样重要。
- 启动阶段与稳态阶段必须分开，否则一次 CUDA 首次执行会污染长期性能判断。
- 设备采样应与实时关键路径隔离，并允许独立失败。
- 对照实验需要一次只改变一个参数；本次只改变采样间隔，因此可以判断性能差异来自采样频率还是普通波动。

## 下一步

- Day37：加入 systemd、watchdog、SIGTERM 优雅退出、离线缓存和日志轮转，形成可长期无头运行的服务。
