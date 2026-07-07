# Jetson Orin Nano 到货检查记录

日期：2026-07-06

## 目标

- 先确认硬件和配件无误。
- 先记录版本和卖家描述，不急着刷机。
- 避免一上来改系统、拆装或装复杂环境导致排错困难。

## 重要提醒

- 如果商家已经预装系统，第一次开机前先拍照和记录，不要急着重刷。
- 如果使用 JetPack 7.2 或更新版本，根据 NVIDIA 官方文档，流程可能不再是直接把镜像写入 microSD，而是制作 USB 安装盘，再安装到 microSD 或 NVMe SSD。
- 如果是 JetPack 6.x，仍可能遇到 SD 卡镜像、SDK Manager、NVMe 安装等不同流程，后续按实际版本决定。

## 第 1 步：外观和配件拍照

建议拍这些照片：

- 外包装正面和型号标签。
- 开发板整体正面。
- 开发板背面。
- 电源适配器标签。
- SSD 位置和 SSD 标签。
- 摄像头、线材、转接头等配件。
- 商品清单或订单页面截图。

## 第 2 步：确认是不是官方开发套件

重点看：

- 是否是 Jetson Orin Nano 8GB。
- 是否是官方开发套件，而不是单独核心模组。
- 是否有官方载板。
- 是否有 microSD 卡槽。
- 是否有 M.2 NVMe SSD 插槽。
- 是否已经安装 256G SSD。
- 是否带官方电源。

记录：

```text
型号：
是否 8GB：
是否官方载板：
是否有 microSD 卡槽：
SSD 容量：
是否预装系统：
电源规格：
配件：
```

## 第 3 步：先不要做的事

暂时不要：

- 不要先刷机。
- 不要先拆散热器。
- 不要先拔 SSD。
- 不要先安装大量 Python 包。
- 不要先装 ROS、Docker 大环境。
- 不要先接小车底盘或外设模块。

先确认硬件和系统状态。

## 第 4 步：首次开机前准备

建议准备：

- 原装电源。
- 显示器。
- DisplayPort 线，或 DP 转 HDMI 转接头。
- USB 键盘鼠标。
- 网线。
- 备用 U 盘。
- Windows 电脑，用于查资料和后续可能制作安装盘。

注意：

- Jetson Orin Nano Developer Kit 的 USB-C 通常不是显示输出口，不要把它当 HDMI/DP 使用。
- 第一次配置建议先接显示器、键盘、鼠标，后续再配置 SSH。

## 第 5 步：如果能进入系统

进入系统后先记录这些命令输出：

```bash
uname -a
cat /etc/os-release
df -h
free -h
lsblk
python3 --version
nvcc --version
dpkg -l | grep nvidia-jetpack
dpkg -l | grep tensorrt
```

如果 `nvcc` 或 TensorRT 查不到，不要急着修，先记录现象。

## 第 6 步：明天之后的目标

短期目标：

- 确认 JetPack / Ubuntu / CUDA / TensorRT 版本。
- 配置网络和 SSH。
- 拷贝一张图片到 Jetson。
- 跑最小 OpenCV 读图测试。
- 再考虑 YOLO 推理。

中期目标：

- 把 PC 上的 ONNX 推理流程迁移到 Jetson。
- 生成 TensorRT engine。
- 对比 PC、Jetson、ONNX Runtime、TensorRT 的推理速度。

## 待记录

```text
到货时间：2026-07-06
购买版本：Seeed Studio reComputer J401 Nano Bundle / J401 carrier board bundle kit with Jetson Orin Nano Module
商家名称：待补充
是否官方套件：不是 NVIDIA 原版 Developer Kit 载板；是 Seeed Studio J401 第三方载板套件，盒子标有 NVIDIA Elite Partner
是否 256G SSD：是，照片可见 FORESEE XP1000F256G-C4H1400 256G SSD
是否预装系统：是，已成功完成首次系统配置
初次开机结果：先卡在 OEM end-user configuration，断电重启后成功进入系统配置界面并进入桌面
问题：需按 Seeed J401 文档确认刷机/系统流程，不直接套用 NVIDIA 官方原版开发套件 SD 卡流程；不要执行系统 upgrade
```

## 2026-07-06 照片观察

已确认：

- 包装标识为 `J401 carrier board bundle kit with Jetson Orin Nano Module`。
- 板卡侧面标签为 `seeed studio reComputer J401 Nano Bundle`。
- 盒内清单显示：
  - Carrier Board x1
  - Jetson Orin Nano x1
  - 256G SSD x1
  - Wi-Fi Module x1
  - Heat Sink with Fan x1
  - Power Adapter x1
  - Wi-Fi Antenna Kit x2
- 背面照片可见：
  - FORESEE 256G SSD 已安装。
  - Wi-Fi 模块已安装。
  - 40-pin GPIO。
  - RTC 电池座。
- 接口照片可见：
  - DC 圆口电源。
  - HDMI。
  - 4 个 USB 3.x Type-A。
  - 千兆网口。
  - USB-C。
  - 2 个 CSI 摄像头接口。

初步判断：

- 这是一套 Seeed Studio J401 载板 + Jetson Orin Nano 模组的生态套件，不是 NVIDIA 官方原版 Orin Nano Developer Kit 载板。
- 对学习边缘 AI / YOLO / ONNX / TensorRT 部署是可用的。
- 后续系统、刷机、接口说明应优先参考 Seeed J401 文档。

## 2026-07-06 客服资料记录

客服提供的资料要点：

- 官方 Jetson Orin Nano 开发者套件入门教程，含刷机指南：
  - 飞书链接：`https://icnhodrfplht.feishu.cn/wiki/KOoOWuBRfiL4oKkKRX5cXMapndb`
  - 密码：`seeedstudio2026`
- reComputer 自研载板系列教程，含硬件接口使用、参数、原理图：
  - `https://wiki.seeedstudio.com/NVIDIA_Jetson/`
- 全系列通用刷机指南：
  - `https://wiki.seeedstudio.com/flash/jetpack_to_selected_product/`
- reComputer 常见问题：
  - `https://wiki.seeedstudio.com/Jetson_FAQ/`
- Jetson 核心模组资料下载：
  - `https://developer.nvidia.com/embedded/downloads`
- QQ 技术群：
  - `1063528275`

重要注意：

- 不要直接执行系统升级类命令：
  - `sudo apt upgrade`
  - `sudo apt full-upgrade`
  - `sudo apt dist-upgrade`
  - `do-release-upgrade`
- 原因：Seeed reComputer / J401 属于第三方 Jetson 载板，内核、驱动、设备树和 BSP 可能是定制版本。盲目升级可能替换关键包，导致兼容性问题。
- `sudo apt update` 只是刷新软件源索引，通常可以执行；真正需要谨慎的是 `upgrade`。
- 后续安装软件时，如果终端提示要升级大量 `nvidia-*`、`linux-*`、`kernel`、`l4t` 相关包，应先停止并确认。

## 2026-07-07 初始化结果

系统信息：

```text
Host: nefelibata-desktop
User: nefelibata
Ubuntu: 22.04.5 LTS
Kernel: 5.15.148-tegra
Architecture: aarch64
JetPack: 6.2.1
Jetson Linux: R36.4.3
Python: 3.10.12
CUDA Toolkit: 12.6
TensorRT: 10.3.0.30 + CUDA 12.5
Power Mode: 25W
Memory: about 7.4 GiB
SSD: NVMe 238.5G, root partition about 233G
```

CUDA 检查：

```text
/usr/local/cuda/bin/nvcc --version
CUDA compilation tools, release 12.6, V12.6.68
```

说明：

- `nvcc` 已存在于 `/usr/local/cuda/bin/nvcc`。
- 若直接运行 `nvcc` 提示 command not found，需要把 `/usr/local/cuda/bin` 加入 `PATH`。

`tegrastats` 检查：

```text
RAM about 1397/7620MB
SWAP 0/3810MB
GR3D_FREQ 0%
CPU/GPU/SOC temperature around 47-49C
VDD_IN about 4.5W
```

说明：

- 空闲状态温度和功耗正常。
- GPU 状态可通过 `tegrastats` 监控。

远程连接：

```text
SSH service: active (running)
Jetson IP on phone hotspot: 172.20.10.13
MobaXterm SSH: connected successfully
Remote prompt: nefelibata@nefelibata-desktop:~$
SFTP sidebar: /home/nefelibata/
```

说明：

- 笔记本已经可以通过 MobaXterm 远程连接 Jetson。
- 后续可以不接显示器，通过 SSH 管理 Jetson。
- `/usr/bin/xauth: file /home/nefelibata/.Xauthority does not exist` 是 X11 转发提示，不影响 SSH 命令行使用。

Type-C USB 救援通道：

```text
Windows USB network adapter: 以太网 4 / Remote NDIS Compatible Device
Windows USB IP: 192.168.55.100/24
Jetson USB IP: 192.168.55.1
SSH port 22: reachable
```

说明：

- 当手机热点 IP 变化、无法通过 Wi-Fi 找到 Jetson 时，可以用 Type-C 数据线连接笔记本和 Jetson。
- 在 MobaXterm 或 PowerShell 中使用 `ssh nefelibata@192.168.55.1` 作为找回入口。
- `L4T-README` 盘是 Jetson USB device mode 暴露给 Windows 的只读说明盘，不是 Jetson 系统盘，不要格式化或删除。
