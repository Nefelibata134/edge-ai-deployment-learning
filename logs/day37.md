# Day 37：systemd 无头服务、watchdog 与持久化运维

日期：2026-08-20 至 2026-08-21

状态：已完成

## 今日目标

- 将实时视觉程序从手动终端命令变成可长期无头运行的 systemd 服务。
- 使用 `SIGTERM` 完成推理、采集、遥测和证据写入组件的优雅退出。
- 让 systemd watchdog 依据真实视频帧进度判断服务健康，而不是只检查进程是否存在。
- 为事件证据、运行指标、模型缓存和日志建立持久化目录与容量边界。
- 验证服务异常后的自动重启、定时清理和不中断运行的日志轮转。

## 7 小时计划

| 时间 | 内容 | 验收标准 |
| --- | --- | --- |
| 1.0 小时 | 审计实时程序生命周期和后台线程 | 明确启动、运行、停止与失败路径 |
| 1.5 小时 | 连续运行模式与信号退出 | `SIGTERM` 后资源有序释放且退出码为 0 |
| 1.0 小时 | systemd notify 与 watchdog | READY、WATCHDOG、STOPPING 通知均可验证 |
| 1.0 小时 | 服务安装、用户权限与环境配置 | 普通服务用户可访问摄像头、GPU 和持久化目录 |
| 0.75 小时 | 离线缓存清理与日志轮转 | 磁盘使用有边界，轮转不停止服务 |
| 1.25 小时 | 本机回归与 Jetson 故障注入 | 自动化测试和真机恢复均通过 |
| 0.5 小时 | 参数修改、记录与提交 | 完成会运行、能解释、能修改 |

## 为什么需要无头服务

此前程序需要在 SSH 终端中手动启动，终端断开、进程崩溃或设备重启后都无法自行恢复。边缘视觉设备通常长期放在现场运行，因此必须把模型推理程序纳入操作系统的进程管理，而不是依赖某个终端窗口。

新的生命周期为：

```text
systemd starts service
  -> launcher loads /etc/edge-vision/edge-vision.env
  -> create a per-process spool directory
  -> start capture, TensorRT, tracking, events and telemetry
  -> receive real frames and notify systemd watchdog
  -> publish metrics and evidence to persistent storage

SIGTERM
  -> signal handler only records the stop request
  -> main thread closes queues and stops workers
  -> flush writers and metrics
  -> notify STOPPING and exit 0

no real frame progress
  -> watchdog notification stops
  -> systemd terminates the unhealthy process
  -> Restart=on-failure starts a fresh process and session
```

## 关键设计

### 信号处理只设置标志

异步信号可能在程序持有互斥锁、执行 GStreamer 操作或写文件时到达。信号处理函数如果直接释放资源或调用复杂库函数，可能发生死锁或破坏内部状态。因此处理函数只写入 `volatile sig_atomic_t`，真正的清理工作由主线程执行。

### watchdog 必须绑定真实帧进度

进程仍存在不代表服务健康。摄像头、RTSP 或解码器可能已经停止出帧，而主循环和线程仍未退出。如果程序仍无条件发送 watchdog，systemd 会误认为服务正常。

本次实现只在最近收到并处理过真实帧时发送 watchdog 通知。进程卡住或输入源长期无帧后，通知停止，systemd 才能检测并恢复故障。

### 连续模式仍然保持内存有界

长期运行不能无限保存延迟样本。`--metrics-window-frames` 使用固定容量窗口，只保留最近一段运行数据；事件录像仍使用有界队列和受限的前后帧缓冲，磁盘证据由定时清理服务按时间和总字节数回收。

### 持久化目录与会话隔离

运行数据统一放在 `/var/lib/edge-vision`：

- `models/` 保存设备本地 TensorRT engine。
- `metrics/latest.json` 原子发布最新运行指标。
- `spool/<session>/` 保存本次运行的 JSONL、截图和事件片段。
- `current` 符号链接指向当前会话。
- `.cache/` 和 `.nv/ComputeCache/` 保存 GStreamer 与 CUDA 缓存。

每次进程启动使用新的会话目录，避免异常退出后继续写入旧会话，也便于清理程序保护当前目录。

### 日志轮转不能中断写入

systemd 以 root 创建重定向日志，所以 logrotate 策略必须使用 `su root root`。服务继续持有原文件描述符，因此采用 `copytruncate`：先复制当前日志，再原地截断，进程不需要重启。

C++ 标准输出重定向到文件后会由行缓冲变成块缓冲，第一次轮转后短时间看不到新数据。启动脚本最终使用：

```text
exec stdbuf -oL -eL "$binary" "${args[@]}"
```

将 stdout 和 stderr 改为行缓冲，既保持日志及时可见，又不侵入推理程序业务代码。

## 自动化验证

Jetson 完整构建后共执行 15 项 C++ 测试，全部通过，其中服务运行时检查结果为：

```text
signal_requested=true
ready_notification=true
watchdog_notification=true
stopping_notification=true
status=PASS
```

本机还通过 11 项 Python 测试和部署脚本语法检查，覆盖安装配置、会话清理、指标状态和边界行为。

## Jetson 持续运行与优雅退出

正式服务使用 IMX219、YOLOX-Nano TensorRT FP16、ByteTrack 和 30 FPS 处理链路。持续运行后发送 `SIGTERM`，结果为：

| 指标 | 结果 |
| --- | ---: |
| 测量帧数 | 9683 |
| 有效 FPS | 约 30.00 |
| 稳态丢帧率 | 0.00% |
| 延迟窗口占用 | 4096 / 4096 |
| `shutdown_requested` | `true` |
| `shutdown_signal` | 15 |
| systemd `Result` | `success` |
| 主进程退出状态 | 0 |

Argus 正常输出清理日志，服务退出后没有遗留推理或 `tegrastats` 子进程。这证明退出不是强制杀进程，而是完成了资源释放和最终指标写出。

## watchdog 故障注入

使用 `SIGSTOP` 暂停服务主进程，模拟“进程仍存在但完全停止工作”的故障：

1. systemd 在约 30 秒未收到 watchdog 后判定超时。
2. 先发送 `SIGABRT` 请求终止。
3. 旧进程在 `TimeoutStopSec=20` 内无法响应，随后被 `SIGKILL` 清理。
4. `RestartSec=5` 后自动启动新进程。
5. 新进程获得新的 PID 和会话目录，`NRestarts=1`，服务恢复为 `active/running`。

从故障注入到恢复约 43 秒。该实验验证的是整条 systemd 恢复链，而不是仅验证一个布尔状态。

## 离线缓存与日志验证

定时清理服务执行成功：

```text
removed_sessions=0
retained_bytes=108340875
```

当前数据没有超过策略边界，因此没有误删；清理 timer 已配置为每小时运行。

日志轮转过程中依次发现并修复了三个真实问题：

1. Jetson 未安装 `logrotate`，安装脚本补充了依赖检查和明确提示。
2. `runtime.log` 属于 root，原来的服务用户无权轮转，策略改为 `su root root`。
3. 轮转成功后新日志短时间保持 0 字节，原因是 stdout 块缓冲，启动器加入 `stdbuf` 行缓冲。

最终验证结果：

```text
force_exit=0
size_before=0
size_after=298
copytruncate_continuity=true
ActiveState=active
SubState=running
```

轮转产生 `runtime.log.1` 和压缩的历史日志，同时当前日志继续增长，服务未重启。

## Jetson 兼容性问题

最初服务使用 systemd 的私有临时目录隔离，但 `nvarguscamerasrc` 依赖宿主机 `/tmp/argus_socket` 与 `nvargus-daemon` 通信。隔离 `/tmp` 后进程虽然能启动，摄像头链路却异常。CSI 主服务因此明确保留宿主机临时目录访问，并通过 `video`、`render` 用户组和 Argus socket 权限验证设备访问。

这说明服务加固不能机械开启所有 sandbox 选项，必须理解硬件运行时实际使用的 IPC 和设备节点。

## 用户修改验证

用户将 `/etc/edge-vision/edge-vision.env` 中的日志间隔从 300 帧改为 120 帧，并判断：

- 不需要重新编译，因为这是运行参数而不是源代码。
- `daemon-reload` 不会替换正在运行的进程，因为环境文件只在服务启动时读取。
- 需要重启服务，新的主进程参数中才会出现 `--log-interval 120`。

重启后的进程命令行确认新值已经生效。

## 会运行、能解释、能修改

### 会运行

- 安装、启动、停止、重启并检查 `edge-vision.service`。
- 查看 systemd 状态、运行日志、最新指标和当前事件会话。
- 手动运行清理服务和日志轮转验证。

### 能解释

- 解释信号处理函数为什么不能直接执行复杂清理。
- 解释进程存在为什么不能证明摄像头服务健康。
- 解释 watchdog 为什么必须依赖真实帧进度。
- 解释 `copytruncate`、日志所有权和输出缓冲之间的关系。

### 能修改

- 修改服务环境文件中的运行参数并判断是否需要编译、reload 或 restart。
- 使用 systemd 属性和进程命令行验证参数真正进入新进程。
- 通过日志文件大小变化验证轮转后服务仍在持续写入。

## 今日产出

- systemd `Type=notify` 无头服务和配置化启动脚本。
- 真实帧进度驱动的 watchdog 与故障自动恢复。
- `SIGTERM` 优雅退出和最终指标发布。
- 独立会话 spool、容量/时间清理服务与定时器。
- root 日志轮转和行缓冲持续写入。
- 服务运行时、部署配置和清理策略自动化测试。
- 正式项目提交：
  - `34e26c8 feat: add supervised headless runtime`
  - `57f80c7 fix: preserve Argus camera service access`
  - `cf9b5e9 fix: validate service deployment dependencies`
  - `1a1bfe6 fix: rotate systemd-owned runtime log`
  - `4b28d4c fix: line-buffer supervised runtime logs`

## 复盘

- 无头运行不是把命令放进 systemd 文件，而是要完整处理健康判定、退出、重启、权限、持久化和磁盘边界。
- watchdog 的核心是选择正确的健康信号；对视频系统而言，真实帧进度比线程或进程存活更可信。
- 部署问题经常发生在代码之外：IPC 隔离、文件所有权、软件包依赖和 stdio 缓冲都会影响服务行为。
- 运维功能必须经过故障注入和实际文件变化验证，不能只看配置文件写得是否合理。

## 下一步

- Day38：拆分检测、跟踪、事件分析和证据写入的阶段开销，并在完整流水线下复测 720p/1080p、功耗和温度，形成端到端性能报告。
