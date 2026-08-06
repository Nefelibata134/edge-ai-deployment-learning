# Day 26：Jetson 设备基线、C++ 工具链与 GStreamer 兼容性

日期：2026-08-05 至 2026-08-06

状态：已完成

## 今日目标

- 通过 SSH 登录 Jetson Orin Nano，确认设备、系统和 NVIDIA 软件栈状态。
- 验证 C++17、CMake、pkg-config 和 GStreamer 开发环境能够支撑项目 2。
- 跑通不依赖摄像头的 GStreamer 测试源和本地 H.264 文件回放链路。
- 记录 JetPack、TensorRT、GStreamer、DeepStream 的兼容性结论。
- 亲自完成一个 C++ GStreamer 探测程序的小修改，达到“会运行、能解释、能修改”。

## 7 小时安排

| 时间 | 内容 | 产出 |
| ---: | --- | --- |
| 0.5 小时 | SSH 连接、创建 Day26 工作目录 | 稳定远程终端和目录 |
| 1.0 小时 | JetPack、L4T、CUDA、TensorRT、功耗模式复查 | 设备基线文本 |
| 1.0 小时 | C++17、CMake、pkg-config、GStreamer 工具链检查 | 工具链版本与缺项清单 |
| 1.5 小时 | GStreamer 测试源、硬件编码、硬件解码和文件回放 | 可复现的管线命令与结果 |
| 1.0 小时 | C++ GStreamer 探测程序：运行、讲解、修改 | 可编译的小程序 |
| 0.75 小时 | DeepStream 与 JetPack 兼容性核对 | 采用/暂缓结论及依据 |
| 0.75 小时 | CSIG 主赛规则、数据和 baseline 入口核对 | 下一次竞赛实验输入 |
| 0.5 小时 | 面试追问、日志、Git 状态和次日准备 | Day26 完整记录 |

## 必做 1：连接 Jetson

在 Windows 的 MobaXterm 中连接当前 Jetson Wi-Fi 地址；如果地址发生变化，先通过
COM5 串口执行 `ip -br addr` 查询。

登录后执行：

```bash
mkdir -p ~/model-deploy-day26/{reports,src,build,videos}
cd ~/model-deploy-day26
pwd
hostname
uname -m
```

预期：主机名为 Jetson 当前主机名，架构为 `aarch64`。

实测结果：SSH 连接稳定，主机名为 `nefelibata-desktop`，架构为 `aarch64`。

## 必做 2：采集设备与 NVIDIA 软件栈基线

```bash
cd ~/model-deploy-day26

{
  echo "## hostname"
  hostnamectl
  echo "## kernel"
  uname -a
  echo "## os"
  grep PRETTY_NAME /etc/os-release
  echo "## l4t"
  cat /etc/nv_tegra_release
  echo "## jetpack"
  dpkg -l | grep nvidia-jetpack
  echo "## tensorrt"
  dpkg -l | grep -E 'tensorrt|libnvinfer' | head -20
  echo "## cuda"
  /usr/local/cuda/bin/nvcc --version
  echo "## power mode"
  sudo nvpmodel -q
  echo "## memory"
  free -h
  echo "## storage"
  df -h /
} | tee reports/jetson-baseline.txt
```

另开一个终端短暂运行：

```bash
tegrastats --interval 1000
```

观察 5 行后按 `Ctrl+C`，记录空闲状态下的 RAM、温度和功耗。

实测基线：JetPack 6.2.1、L4T R36.4.3、CUDA 12.6、TensorRT 10.3，
25W 功耗模式；空闲整机功耗约 4.48 W，GPU 温度约 49 摄氏度。

## 必做 3：检查 C++ 与 GStreamer 工具链

```bash
{
  echo "## compiler"
  g++ --version | head -1
  echo "## cmake"
  cmake --version | head -1
  echo "## pkg-config"
  pkg-config --version
  echo "## gstreamer"
  gst-launch-1.0 --version
  echo "## gstreamer development package"
  pkg-config --modversion gstreamer-1.0
  echo "## required plugins"
  for plugin in videotestsrc appsink filesrc qtdemux h264parse nvv4l2decoder nvvidconv nvarguscamerasrc; do
    if gst-inspect-1.0 "$plugin" >/dev/null 2>&1; then
      echo "PASS $plugin"
    else
      echo "MISSING $plugin"
    fi
  done
} | tee reports/toolchain-baseline.txt
```

这里先只检查，不执行 `apt upgrade`，也不随意替换 Seeed 提供的 JetPack 软件包。

实测结果：GCC 11.4、CMake 3.22.1、GStreamer 1.20.3 可用；
`nvv4l2decoder`、`nvvidconv`、`nvarguscamerasrc` 等项目 2 所需插件存在。

## 必做 4：验证 GStreamer 数据流

### 4.1 测试源到空输出

```bash
gst-launch-1.0 -v \
  videotestsrc num-buffers=120 \
  ! video/x-raw,width=1280,height=720,framerate=30/1 \
  ! fakesink sync=false
```

理解数据流：

```text
videotestsrc -> caps filter -> fakesink
```

### 4.2 生成 H.264 测试文件

```bash
gst-launch-1.0 -e \
  videotestsrc num-buffers=300 pattern=ball \
  ! video/x-raw,width=1280,height=720,framerate=30/1 \
  ! nvvidconv \
  ! 'video/x-raw(memory:NVMM),format=NV12' \
  ! nvv4l2h264enc bitrate=4000000 \
  ! h264parse \
  ! qtmux \
  ! filesink location=videos/day26_test.mp4

ls -lh videos/day26_test.mp4
```

实测发现当前 Orin Nano 不提供 NVENC 硬件编码能力，系统中没有
`nvv4l2h264enc`。这不是安装损坏；项目 2 的关键路径是硬件解码和推理，
不依赖硬件编码。本次改用 `x264enc` 生成 H.264 测试文件。

### 4.3 使用 Jetson 硬件解码器回放到空输出

```bash
gst-launch-1.0 -v \
  filesrc location=videos/day26_test.mp4 \
  ! qtdemux \
  ! h264parse \
  ! nvv4l2decoder \
  ! nvvidconv \
  ! fakesink sync=false
```

摄像头还未到，不运行 `nvarguscamerasrc` 实时采集；今天只确认插件是否存在。

实测结果：`nvv4l2decoder` 成功将 H.264 文件解码为 NVMM 中的 NV12 帧，
随后由 `nvvidconv` 转换并送入 `fakesink`，完整管线正常收到 EOS。

## 必做 5：C++ GStreamer 探测程序

这一部分由协作过程逐段创建，不直接粘贴一个无法解释的完整项目。最终程序需要：

- 调用 `gst_init` 初始化 GStreamer。
- 使用 `gst_parse_launch` 创建测试管线。
- 进入 `PLAYING` 状态。
- 等待 `EOS` 或 `ERROR` 消息。
- 使用 RAII 或明确的清理路径释放 bus 和 pipeline。
- 使用 CMake 和 `pkg-config` 编译。

用户亲自完成的修改：把测试源的 `num-buffers` 从初始值改为另一个值，并解释
为什么程序仍会收到 EOS 后正常退出。

实测结果：C++17/CMake 程序成功完成配置、编译和链接。初始 120 帧运行约
4.01 秒；将 `num-buffers` 改为 60 后运行约 2.01 秒，均正常收到 EOS 并释放资源。

## 必做 6：DeepStream 兼容性门禁

需要回答：

1. 当前 JetPack/L4T 对应哪些官方 DeepStream 版本？
2. 当前 Seeed JetPack 6.2.1 镜像是否在该 DeepStream 版本的官方验证矩阵中？
3. 如果不在，继续使用原生 GStreamer + TensorRT C++ 是否能完成 v1？
4. DeepStream 是 v1 依赖、可选对照，还是明确暂缓？

今天只形成技术决策，不为安装 DeepStream 而破坏当前可用环境。

兼容性结论：NVIDIA DeepStream 7.1 的官方矩阵覆盖 Jetson Orin Nano、
Ubuntu 22.04、L4T 36.4、CUDA 12.6 和 TensorRT 10.3，与当前设备的核心运行栈匹配。
但当前 Seeed 镜像未安装 `deepstream-app`，也不存在 `nvstreammux`、`nvinfer`、
`nvtracker` 插件。项目 2 v1 因此采用原生 GStreamer + TensorRT C++ API +
ByteTrack；DeepStream 只保留为后续可选对照实验，不作为 v1 依赖。

官方依据：<https://docs.nvidia.com/metropolis/deepstream/7.1/text/DS_Installation.html#platform-and-os-compatibility>

## 竞赛并行任务

为 CSIG 工业异常检测主赛核对并记录：

- 是否允许个人参赛。
- 报名和提交截止时间。
- 数据下载入口与数据大小。
- 评价指标和提交格式。
- 官方或社区 baseline 是否可运行。

竞赛时间达到 45 分钟即停止，不挤占 Jetson 主线。

本项按用户决定延期到后续独立竞赛时段，不阻塞 Day26 的 Jetson 主线验收。

## 必须能解释

1. Jetson 为什么是 `aarch64`，而笔记本 WSL 通常是 `x86_64`？
2. 为什么 TensorRT engine 应在目标 Jetson 上重新构建？
3. GStreamer 中 source、element、caps 和 sink 分别是什么？
4. `nvv4l2decoder` 与纯 CPU 解码相比解决了什么问题？
5. 为什么实时视频系统在拥塞时不能无限积压帧？

## 完成标准

- [x] SSH 或串口连接稳定。
- [x] `reports/jetson-baseline.txt` 已生成。
- [x] `reports/toolchain-baseline.txt` 已生成，必需插件状态明确。
- [x] GStreamer 测试源管线成功。
- [x] H.264 测试文件生成并由硬件解码管线成功读取。
- [x] C++ GStreamer 小程序编译、运行并完成一次亲自修改。
- [x] DeepStream 兼容性结论有官方依据。
- [x] CSIG 主赛核对已明确延期到后续竞赛时段。
- [x] 学习仓库 README 进度、Day26 日志和 Git 状态已整理。

## 今日产出

- Jetson 系统、CUDA、TensorRT、功耗、内存和存储基线。
- GStreamer 1.20.3 插件与 C++17/CMake 工具链报告。
- H.264 软件编码、Jetson 硬件解码和 NVMM 数据通路验证。
- 可编译运行的 C++ GStreamer EOS/ERROR 探测程序。
- 120 帧与 60 帧运行时间对比，以及一次亲自完成的代码修正。
- DeepStream 7.1 兼容性门禁和项目 2 v1 技术决策。

## 复盘

Day26 已确认项目 2 的底层视频链路具备开发条件。Orin Nano 不提供 NVENC，
不影响以硬件解码、TensorRT 推理和实时事件分析为核心的 v1。后续不引入未安装的
DeepStream 依赖，先用原生 GStreamer 掌握帧来源、格式协商、EOS/ERROR 和资源释放。

## 明日预告

Day27 将建立 C++17/CMake 视频回放骨架，设计 `IFrameSource` 接口，并让同一套
代码可以在 PC 和 Jetson 上编译。Day28 再创建项目 2 的独立公开仓库。
