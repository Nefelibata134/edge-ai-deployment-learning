# Day 27：C++ 视频回放骨架与 IFrameSource 接口

日期：2026-08-06

状态：进行中

## 今日目标

- 在 Jetson 建立可复现的 C++17/CMake 视频回放工程骨架。
- 使用 `IFrameSource` 隔离文件、摄像头和后续 RTSP 等不同输入来源。
- 通过 GStreamer `appsink` 将解码帧交给 C++，读取宽高、PTS 和帧序号。
- 验证 EOS、错误路径、资源释放和有限缓冲策略。
- 让同一接口能够在 PC 与 Jetson 使用，平台差异只进入具体实现和管线配置。

今天仍在 `/home/nefelibata/model-deploy-day27` 中验证设计；Day28 再创建项目 2
独立仓库 `jetson-realtime-tracking-system`，不会把 Day 日志带入正式项目仓库。

## 7 小时安排

| 时间 | 内容 | 验收产出 |
| ---: | --- | --- |
| 0.5 小时 | 恢复 Jetson 连接、检查 `gstreamer-app` 开发依赖 | 环境检查结果 |
| 1.0 小时 | 设计 `Frame` 与 `IFrameSource` 契约 | 能解释接口边界 |
| 1.0 小时 | 建立 C++17/CMake 目录和编译目标 | 可重复构建的骨架 |
| 2.0 小时 | 实现 GStreamer 文件源和 `appsink` 拉帧 | 视频文件逐帧读取 |
| 0.75 小时 | 验证帧数、尺寸、PTS、EOS 和错误输入 | 回放报告 |
| 0.75 小时 | 有限缓冲、`drop-oldest` 设计与一次亲自修改 | 低延迟策略说明 |
| 0.5 小时 | PC/Jetson 差异、面试追问和 Day27 日志 | 当日复盘 |
| 0.5 小时 | 延期的竞赛任务或项目 2 架构整理 | 不阻塞主线的并行产出 |

## 必做 1：建立工作目录并检查依赖

在 Jetson 的 VS Code Remote SSH 终端执行：

```bash
mkdir -p ~/model-deploy-day27/{include,src,tests,build,reports,videos}
cd ~/model-deploy-day27

cp ~/model-deploy-day26/videos/day26_test.mp4 videos/replay.mp4

pkg-config --modversion gstreamer-1.0
pkg-config --modversion gstreamer-app-1.0
gst-inspect-1.0 appsink | head -20
ls -lh videos/replay.mp4
```

如果 `gstreamer-app-1.0` 缺失，只安装对应开发包，不执行系统整体升级。

## 必做 2：理解接口边界

`IFrameSource` 不关心帧来自文件、CSI 摄像头还是 RTSP。上层检测器只依赖统一的
`open/read/close` 契约：

```cpp
#pragma once

#include <cstdint>
#include <optional>
#include <string>
#include <vector>

struct Frame {
    std::vector<std::uint8_t> pixels;
    int width = 0;
    int height = 0;
    int channels = 0;
    std::int64_t pts_ns = 0;
    std::uint64_t sequence = 0;
};

class IFrameSource {
public:
    virtual ~IFrameSource() = default;

    virtual bool open() = 0;
    virtual std::optional<Frame> read() = 0;
    virtual void close() noexcept = 0;
    virtual std::string description() const = 0;
};
```

关键设计：

- 虚析构函数保证通过接口指针销毁具体实现时能完整释放 GStreamer 资源。
- `std::optional<Frame>` 用“有帧/无帧”表达 EOS，不使用特殊宽高或空指针暗号。
- `pts_ns` 保存展示时间戳，后续用于实时延迟、跟踪时间和事件持续时间计算。
- `sequence` 保存递增帧号，便于诊断丢帧和顺序错误。
- 今天先返回 Host 内存中的 BGR 帧；后续 TensorRT 接入时再评估 NVMM/CUDA
  零拷贝，不能提前让接口被某一种输入源绑死。

## 必做 3：CMake 骨架

Day27 需要的依赖是 GStreamer 核心库和 App Library：

```cmake
cmake_minimum_required(VERSION 3.16)
project(frame_source_replay LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

find_package(PkgConfig REQUIRED)
pkg_check_modules(
    GSTREAMER
    REQUIRED
    IMPORTED_TARGET
    gstreamer-1.0
    gstreamer-app-1.0
)

add_executable(
    frame_source_replay
    src/main.cpp
    src/gstreamer_file_source.cpp
)

target_include_directories(frame_source_replay PRIVATE include)
target_link_libraries(frame_source_replay PRIVATE PkgConfig::GSTREAMER)
target_compile_options(frame_source_replay PRIVATE -Wall -Wextra -Wpedantic)
```

## 必做 4：GStreamer 文件回放数据流

Jetson 使用硬件解码：

```text
filesrc
→ qtdemux
→ h264parse
→ nvv4l2decoder
→ nvvidconv
→ video/x-raw,format=BGRx
→ videoconvert
→ video/x-raw,format=BGR
→ appsink
```

`appsink` 是 GStreamer 和 C++ 业务代码之间的边界。需要设置：

```text
sync=false max-buffers=2 drop=true
```

- `sync=false`：本次基准不按视频原始时钟等待，尽快读取。
- `max-buffers=2`：限制内部缓冲，防止内存无限增长。
- `drop=true`：消费者变慢时优先保留更新的帧，满足实时系统低延迟目标。

PC 端可以将 `nvv4l2decoder` 替换为 `decodebin` 或系统可用的软件/硬件解码器，
`IFrameSource` 的调用方不需要改变。

## 必做 5：验证要求

回放程序至少输出：

```text
source: videos/replay.mp4
frame=1 size=1280x720 channels=3 pts_ms=0.000
...
eos=true
frames=<实际帧数>
elapsed_ms=<实际时间>
```

还要验证：

1. 正常视频能读取到 EOS。
2. 不存在的文件必须清晰失败，不能假装得到 0 帧后成功退出。
3. 帧序号严格递增。
4. 有效 PTS 不倒退。
5. `close()` 重复调用不会崩溃。

用户亲自修改：将最大读取帧数从 60 改为 30，重新编译运行，并解释为什么程序
可以在文件 EOS 之前主动、正常停止。

## 必须能解释

1. 为什么上层检测器应依赖 `IFrameSource`，而不是直接写死 `filesrc`？
2. `appsink` 在 GStreamer 与 C++ 之间承担什么作用？
3. `PTS` 与普通帧序号有什么区别？
4. 为什么实时输入要限制缓冲区长度？
5. 为什么 Jetson 管线和 PC 管线可以不同，但接口应保持相同？

## 完成标准

- [ ] `gstreamer-app-1.0` 开发依赖可用。
- [ ] C++17/CMake 骨架可重复配置和编译。
- [ ] `IFrameSource` 和 `Frame` 接口能够解释。
- [ ] Jetson 硬件解码文件源能够通过 `appsink` 读取帧。
- [ ] 帧尺寸、帧号、PTS、EOS 和错误文件路径均已验证。
- [ ] 完成一次最大读取帧数修改并重新验证。
- [ ] README 进度索引和 Day27 日志已更新。

## 明日预告

Day28 创建项目 2 的独立公开仓库，将经过验证的 C++17/CMake 骨架迁入正式仓库，
再把 ONNX 模型传到 Jetson 并在板端构建 FP16 TensorRT engine。
