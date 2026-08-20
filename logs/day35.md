# Day 35：RTSP 输入、无帧超时与断流恢复

日期：2026-08-20

状态：已完成

## 今日目标

- 在文件回放和 IMX219 CSI 之外增加 H.264 RTSP 输入。
- 区分连接超时、连接存在但无帧和媒体流 EOS。
- 对 CSI/RTSP 使用统一的有限重连策略和流代次语义。
- 验证重连后 ByteTrack 和事件状态不会继承旧流状态。
- 完成“会运行、能解释、能修改”三层验收。

## 7 小时计划

| 时间 | 内容 | 验收标准 |
| --- | --- | --- |
| 1.0 小时 | 复核输入接口、采集线程和重连边界 | 能画出源到推理的完整数据流 |
| 1.5 小时 | 实现 H.264 RTSP GStreamer 适配器 | TCP/UDP、延迟和超时参数可配置 |
| 1.0 小时 | 实现无帧超时和有限重连 | 空连接不能无限重连 |
| 1.0 小时 | 补充流代次、重连预算和回归测试 | 文件、CSI 和现有模块无回归 |
| 1.5 小时 | Jetson 真实 RTSP、EOS 和进程断流实验 | 能恢复并继续检测跟踪 |
| 0.5 小时 | 用户参数修改与理解检查 | 能解释超时参数的可靠性权衡 |
| 0.5 小时 | 文档、提交与复盘 | 正式仓库和学习记录同步 |

## RTSP 数据流

```text
RTSP server
  -> rtspsrc
  -> rtph264depay
  -> h264parse
  -> nvv4l2decoder
  -> nvvidconv
  -> BGR appsink
  -> FrameCaptureWorker
  -> bounded latest-frame queue
  -> TensorRT detector
  -> ByteTrack
  -> event analytics and evidence
```

RTSP 适配器复用现有 `IFrameSource`，没有让网络输入细节进入检测器、跟踪器或事件模块。CLI 支持 TCP/UDP、接收延迟、无帧超时、输出尺寸和输出 FPS。TCP 作为默认传输用于可靠交付；UDP 可以在受控网络中降低传输等待，但会接受丢包风险。

## 三层故障检测

```text
TCP/RTSP 连接失败
  -> rtspsrc tcp-timeout

连接存在但没有可解码帧
  -> application read timeout

EOS、错误或读帧超时
  -> close pipeline
  -> bounded reopen attempts
  -> first valid frame confirms recovery
  -> increment stream generation
  -> reset tracker and event state
```

只依赖 TCP 超时不够，因为套接字仍连接时，摄像头或服务端编码器仍可能停止产出视频。应用层必须为“最后一次收到有效帧”建立独立活性判断。

重连预算只在收到真实帧后重置。GStreamer 的异步 `open()` 被接受不等于视频恢复；如果每次 `open()` 都重置预算，空连接会形成无限重连循环。

## 自动化验证

RTSP 管线检查：

```text
tcp_pipeline=true
udp_pipeline=true
invalid_config=true
status=PASS
```

采集与恢复状态机检查：

```text
drop_oldest=true
close_unblocks=true
capture_worker=true
capture_recovery=true
recovery_budget_per_outage=true
empty_reopen_is_not_recovery=true
status=PASS
```

完整本机回归：

- C++：12/12 tests passed。
- Python：10/10 tests passed。

## Jetson 真实验证

### RTSP 采集基线

使用 H.264 MP4 启动本地 RTSP 服务，C++ 客户端读取 120 帧：

| 指标 | 结果 |
| --- | ---: |
| 目标帧 | 120/120 |
| 有效 FPS | 29.362 |
| 丢帧 | 0 |
| 坏帧 | 0 |
| PTS 单调 | true |
| 流代次 | 0 |

### EOS 自动恢复

原始回放只有约 300 帧，客户端请求 450 帧，迫使管线在 EOS 后重新打开：

```text
target_reached=true
restart_attempts=1
restart_successes=1
stream_generation=1
recovery_exhausted=false
pts_monotonic=true
exit_code=0
```

帧源保留连续 sequence，并在新媒体时间线从零开始时增加 PTS 偏移，使应用层时间仍保持单调；`stream_generation` 单独表达逻辑流已经改变。

### 视频容器与编码不匹配

事件审计视频扩展名为 MP4，但内部由 OpenCV `mp4v` 编码，不能直接进入要求 H.264 的 RTSP 回放管线。第一次连接因此产生零帧并在 3 次重试后正确失败：

```text
produced=0
restart_attempts=3
recovery_exhausted=true
exit_code=1
```

MP4 是容器，不等于 H.264 编码。转码时，Jetson `decodebin` 选择硬件解码器并输出 NVMM 帧，直接连接 CPU `videoconvert` 出现 `not-negotiated`；改用 `nvvidconv` 将 NVMM 转为普通 I420 后，得到 4.0 MB 的 H.264 文件并通过 `h264parse` 校验。

### RTSP 检测与跟踪基线

转码后打通 RTSP、硬件解码、TensorRT 和 ByteTrack：

| 指标 | 结果 |
| --- | ---: |
| 测量帧 | 180 |
| 检测帧 | 180 |
| 跟踪帧 | 180 |
| 轨迹观测 | 218 |
| 唯一轨迹 ID | 3 |
| 推理 P95 | 13.304 ms |
| 端到端 P95 | 14.177 ms |
| 有效 FPS | 30.100 |
| 稳态丢帧 | 0 |

### 受控服务中断

将 `--rtsp-timeout-ms` 从 3000 修改为 1000，运行中停止 RTSP 服务约 4 秒再重启。系统达到 450 个目标帧，恢复后稳态端到端 P95 为 14.220 ms，`recovery_exhausted=false`，旧轨迹状态没有跨流继承。

实验还暴露了旧统计口径：5 次异步管线打开都被计为成功，但只有 2 个流真正产出帧，因而只有 2 次 tracker reset。修正后，成功次数和流代次都只在重连后的第一张有效帧到达时增加。

最终自动 EOS 验收结果：

```text
target_reached=true
restart_attempts=1
restart_successes=1
stream_generation=1
tracker_resets=1
recovery_exhausted=false
invalid_frames=0
inference_p95_ms=13.327
end_to_end_p95_ms=14.208
effective_fps=28.464
exit_code=0
```

## 用户修改验证

用户将无帧超时从 3000 ms 改为 1000 ms，并完成真实服务停止与重启实验。较短超时能更快识别无帧故障，但网络抖动或关键帧间隔较长时更容易误判，因此该参数需要结合网络条件和码流特征配置，不能盲目压低。

## 会运行、能解释、能修改

### 会运行

- 构建并运行 RTSP 管线和重连状态机检查。
- 启动本地 RTSP 回放服务并使用 C++ 客户端读取。
- 运行 RTSP 到 TensorRT、ByteTrack 的完整实时链路。
- 注入 EOS 和服务进程中断并验证恢复。

### 能解释

- 为什么 TCP 连接存在不代表视频源健康。
- 为什么 MP4 容器和 H.264 编码不能混为一谈。
- 为什么 `open()` 成功不能直接计作流恢复。
- 为什么重连后必须清空跟踪和事件状态。
- 为什么超时越短并不一定越可靠。

### 能修改

- 修改 RTSP 无帧超时并解释故障检测速度与误重连风险。
- 根据 Jetson NVMM 内存类型修正转码管线。
- 使用真实统计验证尝试次数、成功恢复、流代次和 tracker reset 的一致性。

## 今日产出

- H.264 RTSP GStreamer 输入适配器。
- TCP/UDP、网络延迟和无帧超时 CLI。
- CSI/RTSP 共用的有限重连状态机。
- 以首张有效帧确认恢复的统计语义。
- RTSP 回放测试服务器和确定性回归测试。
- Jetson RTSP 采集、检测、跟踪、EOS 和断流恢复报告。
- 正式项目提交：
  - `ccb3d8f feat: add resilient RTSP ingestion`
  - `8531c81 fix: confirm stream recovery on first frame`

## 复盘

- 网络视频系统需要分别监控连接活性和帧活性，不能把传输层存活当成业务健康。
- 故障恢复统计必须以可观察结果为准；管线对象创建成功只是中间状态，第一张有效帧才是恢复完成。
- `stream_generation` 将重连事实显式传给跟踪和事件模块，比依赖时间戳或 ID 猜测更可靠。
- 故障注入不仅验证恢复路径，也能发现正常基线测试看不到的监控语义缺陷。

## 下一步

- Day36：接入 `tegrastats`、队列深度、丢帧率、端到端延迟和设备状态指标，形成统一 pipeline metrics。
