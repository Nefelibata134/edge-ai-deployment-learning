# Day 40：项目 2 v1 发布冻结与完整仓库审计

日期：2026-08-23

状态：已完成

## 今日目标

- 用真实 Jetson 结果确认项目 2 v1 候选版本可以发布。
- 从代码、测试、文档、隐私、许可证、依赖和可复现性多个角度审计正式仓库。
- 修正审计发现的发布说明问题，并等待最终 GitHub CI 通过。
- 让 `v1.0.0` 标签准确指向最终提交，创建正式 GitHub Release。
- 明确 v1 已完成的系统边界和仍然存在的限制，避免把“能运行”写成“没有风险”。

## 7 小时计划

| 时间 | 内容 | 验收标准 |
| --- | --- | --- |
| 1.0 小时 | 真机发布候选复核 | IMX219 真实帧、TensorRT、跟踪、指标和 systemd 状态均正常 |
| 1.5 小时 | 仓库边界、隐私和许可证审计 | 无密钥、个人路径、模型、视频和原始报告误提交；第三方依赖声明完整 |
| 1.5 小时 | 干净构建、测试和动态检查 | fresh build、C++、Python、release validator、ASan/UBSan 全部通过 |
| 1.0 小时 | 核心运行时与运维边界复核 | 有界队列、恢复状态、线程回收、watchdog 和持久化边界可解释 |
| 1.0 小时 | README、CHANGELOG 和 release notes 修正 | 架构、命令、指标、同步/异步 I/O 和限制描述准确 |
| 1.0 小时 | 提交、CI、标签和 GitHub Release | 最终 CI 成功，标签与提交一致，公开正式 Release 可访问 |

## 发布候选真机复核

在 Jetson Orin Nano 8GB 上用 IMX219 CSI、YOLOX Nano FP16 TensorRT 和完整实时程序验证了最终候选版本：

| 指标 | 结果 |
| --- | ---: |
| 测量帧 | 120 |
| `target_reached` | true |
| 无效帧 | 0 |
| 测量阶段丢帧 | 0 |
| 有效 FPS | 30.194 |
| TensorRT P95 | 8.321 ms |
| 端到端 P95 | 13.386 ms |
| systemd 服务 | active / running / ready |

测试画面中没有目标时 `detections=0`，只能说明本次画面没有达到检测阈值，不能说明推理没有运行。判断链路健康要同时看真实帧推进、目标帧是否达到、无效帧、恢复状态和延迟样本。

## 完整仓库审计

### 1. 公开边界与隐私

- 检查全部 133 个跟踪文件，没有发现 API key、token、私钥、密码或邮箱等敏感信息。
- 没有 Windows 用户路径、Jetson 用户目录、局域网设备地址或个人身份信息。
- 没有跟踪 `.plan`、`.onnx`、图片、视频、日志、原始报告、MOT17 数据或构建目录。
- 最大跟踪文件约 62.5 KiB，没有异常二进制或大文件。
- `.gitignore` 覆盖模型、引擎、运行输出、报告、数据、外部工具和构建目录。

### 2. 许可证与依赖

- 项目自身采用 MIT License。
- ByteTrack 上游 MIT 许可证保留在 `third_party/bytetrack/LICENSE`。
- 固定记录 ByteTrack 和 TrackEval 上游 commit，避免依赖来源漂移。
- YOLOX 下载脚本同时固定来源、SHA256 和模型输入输出契约。
- `THIRD_PARTY_NOTICES.md` 补全 OpenCV、Eigen 和 CUDA Toolkit，并保留 GStreamer、TensorRT、YOLOX、ByteTrack、TrackEval 和 nlohmann JSON 的许可说明。

### 3. 构建、测试与代码检查

| 检查 | 结果 |
| --- | --- |
| 干净目录配置与构建 | PASS |
| C++ / CTest | 15/15 PASS |
| Python tests | 23/23 PASS |
| Release validator | PASS |
| shell 语法 | PASS |
| `git diff --check` | PASS |
| ASan + UBSan | 15/15 PASS |
| cppcheck | 无阻断问题，仅有非阻断风格提示 |

本地结果之外，最终提交对应的 GitHub Actions `CI #55` 也成功完成，避免只依赖开发机环境。

### 4. 核心运行时边界

- 采集队列和视频输出队列均有容量上限，过载时优先保留新帧。
- RTSP 只有在收到首个真实解码帧后才计为恢复成功。
- 新视频流代次会重置跟踪与事件状态，避免继承旧 ID、停留时间和越线状态。
- 事件状态会清理长期不可见轨迹；后台线程在退出时均被回收。
- systemd 使用 ready、真实帧 watchdog、24 小时代次和本地保留策略监督服务。
- `realtime_detect.cpp` 仍然较大，是后续拆分维护的技术债，但不影响 v1 正确性。

## 审计发现与修正

| 发现 | 修正 |
| --- | --- |
| README 曾把全部证据写入描述成不阻塞 | 明确截图和 JSONL 是可测量的同步 I/O，事件片段和标注视频是有界异步编码 |
| Quick Start 缺少克隆仓库和部分依赖步骤 | 加入 `apt-get update`、`git`、`curl`、`git clone` 和进入目录命令 |
| RTSP URI 中的凭据风险没有披露 | 说明 URI 作为进程参数时可能被本机进程检查工具看到 |
| 稳定性报告仍写 Python 22/22 | 更新为当前 23/23 |
| 第三方清单漏列 OpenCV、Eigen 和 CUDA | 按官方许可来源补全 |
| Release 页面复制相对文档链接会失效 | 发布正文使用固定到 `v1.0.0` 标签的绝对链接 |
| 旧标签指向审计前提交 | 在 Release 创建前重建注释标签，使其指向最终提交 `02bd0ef` |

修正提交为：

```text
02bd0ef docs: harden v1 release documentation
```

## v1 已知限制

- COCO 预训练 YOLOX 不是针对具体安全场景训练的模型，真实业务精度仍需要场景数据验证。
- 1080p OpenCV MP4V 软件编码无法在当前设备保持所有 30 FPS 输出帧，但有界输出队列不会拖停检测主链路。
- 截图和 JSONL 为保持证据顺序仍在处理线程同步写入，慢存储会增加事件帧延迟。
- RTSP 凭据如果直接放入 URI，可能被本机进程检查看到。
- 事件证据只保存在本地，尚未实现远程上传、认证和设备集群管理。
- 稳定性证据覆盖一台设备的一小时测试，不代表多设备、多日耐久结果。
- x86 GitHub CI 不编译 Jetson TensorRT 路径，必须与 Jetson 真机候选验证共同使用。

这些限制不是隐藏的失败，而是 v1 对外承诺的明确边界。

## 正式发布

- 最终提交：`02bd0efa0124463534cc54757760a305ac604cfd`
- GitHub CI：`CI #55`，成功。
- 注释标签：`v1.0.0`，准确指向最终提交。
- Release 标题：`Jetson Realtime Tracking System v1.0.0`。
- Release 类型：正式版本，并标记为 Latest。
- Release 地址：<https://github.com/Nefelibata134/jetson-realtime-tracking-system/releases/tag/v1.0.0>
- 附件：仅 GitHub 自动生成的源码归档，没有模型、TensorRT engine、视频或个人画面。

## 会运行、能解释、能修改

### 会运行

- 在 Jetson 上重新构建 TensorRT engine 和完整 C++ 运行时。
- 用 IMX219、文件和 RTSP 三种输入验证实时链路。
- 运行 CTest、Python tests、性能测试、稳定性测试和故障注入。
- 安装 systemd 服务并检查 ready、watchdog、真实帧和最终指标。

### 能解释

- 解释为什么 `detections=0` 不等于推理没有运行。
- 解释 tag、commit 和 GitHub Release 的区别，以及标签为什么必须指向最终审计提交。
- 解释同步证据 I/O 与异步视频编码对关键路径的不同影响。
- 解释公开仓库为什么不能提交模型引擎、数据集、运行视频和私人画面。
- 解释 v1 的稳定性声明为什么只能覆盖单设备一小时。

### 能修改

- 根据目标设备重新生成 TensorRT engine，而不是复制其他设备的 plan。
- 修改输入源、阈值、队列容量、事件规则和 systemd 环境配置。
- 扩展测试或发布校验时同步更新稳定性报告和 release notes。
- 在后续版本中拆分大型运行入口，并加入场景模型、远程证据传输或更长耐久测试。

## 今日产出

- 完整公开仓库审计和问题修正。
- 最终 fresh build、15 项 C++、23 项 Python、sanitizer 与 release validator 结果。
- 准确指向最终提交的 `v1.0.0` 标签。
- 项目 2 首个正式 GitHub Release。
- 可公开引用的架构、性能、稳定性、兼容性和已知限制说明。

## 复盘

- 发布不是把代码打一个标签，而是冻结可复现的系统边界、证据和限制。
- 文档中的一句“不会阻塞”也必须能由代码路径和指标支持，不能为了好看过度承诺。
- CI、真机和 sanitizer 解决的是不同风险，任何单项通过都不能替代完整发布门禁。
- 相对链接在仓库文档中正确，但复制到 GitHub Release 后语境改变，需要重新验证。
- 已知限制写清楚不会削弱项目，反而能证明对工程边界和证据强度有判断。

## 下一步

- Day41：回到项目 1，固定测试集，复核类别阈值、错误切片、失败样例和质量限制，形成模型质量报告。
