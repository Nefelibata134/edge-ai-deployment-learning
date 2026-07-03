# Jetson Orin Nano 采购清单

## 购买时间建议

建议本周内购买开发板和必备配件。

原因：

- 前两周主要在 PC 上补 Linux、Python、ONNX 和 TensorRT，但开发板提前到手可以同步熟悉 Jetson 环境。
- Jetson 刷机、网络、摄像头、依赖安装可能会遇到问题，早点买能给排错留时间。
- 不建议等到第 5 周再买，否则项目 2 的时间会被物流和环境问题压缩。

## 现在就买

### 1. 开发板

首选：

- NVIDIA Jetson Orin Nano Super Developer Kit。

确认点：

- 8GB 版本。
- 官方开发套件，不是单独的核心板 module。
- 标称 67 INT8 TOPS。
- 盒内应包含 19V 电源。

不建议：

- Jetson Nano 老款。
- Jetson Orin Nano 4GB。
- 只买 Orin Nano module 但没有载板。
- 来路不明的二手板。

### 2. 存储

建议同时买：

- NVMe SSD：500GB，M.2 2280，PCIe 3.0/4.0 都可。
- microSD 卡：64GB 或 128GB，UHS-I，品牌卡。
- U 盘：64GB 或更大，用于制作 Jetson ISO 启动盘。

建议优先级：

- 必买：NVMe SSD。
- 必买：U 盘。
- 可买：microSD 卡，作为备用启动/恢复介质。

理由：

- 官方文档说明开发板支持 microSD 和外接 NVMe。
- M.2 Key-M Type 2280 插槽支持 PCIe 3.0 x4，适合装 2280 NVMe SSD。
- 模型、Docker 镜像、数据集、TensorRT engine 会占空间，长期只靠 microSD 不舒服。

### 3. 摄像头

第一阶段建议买：

- USB 摄像头，1080p，UVC 免驱，优先罗技/奥尼/海康威视/普通免驱摄像头。

第二阶段可选：

- Jetson 兼容 CSI 摄像头，优先 IMX219 或 IMX477，并确认带 22-pin 转接线。

选择建议：

- 先买 USB 摄像头，最省心，适合 OpenCV、YOLO 实时检测。
- CSI 摄像头后面再买，用于更贴近嵌入式和机器人项目。
- 不要一开始买深度相机，成本高、环境复杂，容易偏离模型部署主线。

### 4. 显示、键鼠和网络

按你的现有设备决定：

- DisplayPort 显示器，或 DisplayPort 转 HDMI 转接线/转接头。
- USB 键盘和鼠标。
- 网线。

注意：

- Jetson Orin Nano Developer Kit 的 USB-C 不支持显示输出。
- 如果显示器只有 HDMI，需要买 DisplayPort 转 HDMI。
- 初次配置建议接显示器、键盘、鼠标，后面再用 SSH 远程。

### 5. 散热和供电

- 官方套件自带电源，优先使用盒内 19V 电源。
- 散热先用套件自带方案。
- 暂时不用额外买大散热器；后面如果长时间满载再考虑。

## 到货后再买

- 小车底盘。
- 电机驱动模块。
- 移动电源或电池组。
- IMU。
- 超声波或激光测距模块。
- 亚克力板、铜柱、杜邦线。

理由：

- 先做 Jetson 视觉部署项目，再把视觉结果接到小车控制。
- 过早买底盘容易变成硬件调车，分散模型部署主线。

## 先不要买

- 昂贵的深度相机。
- 很复杂的机械结构。
- 多个传感器套件。
- Jetson AGX Orin 或 Jetson Thor 这类高价开发套件。
- 非官方载板套装，除非明确知道兼容性和资料完整。

先把 Jetson + 摄像头 + 目标检测跑通，再升级小车部分。

## 推荐下单组合

### 标准组合

- Jetson Orin Nano Super Developer Kit。
- 500GB NVMe SSD，M.2 2280。
- 64GB 或 128GB U 盘。
- 1080p USB 摄像头。
- DisplayPort 转 HDMI 转接线或转接头。
- 网线。

### 稍微完整一点

在标准组合基础上增加：

- 128GB microSD 卡。
- CSI 摄像头 IMX219 或 IMX477。
- USB 集线器。
- 小螺丝刀和 M.2 SSD 固定螺丝包。

## 购买渠道建议

优先顺序：

1. NVIDIA 官方页面跳转的授权经销商。
2. 国内电商平台的品牌官方店或高信誉开发板店铺。
3. Waveshare、Seeed、亚博智能等有资料沉淀的店铺。

下单前确认：

- 商品名是否明确写 Jetson Orin Nano Super Developer Kit。
- 是否是 8GB。
- 是否含官方载板和 19V 电源。
- 是否开发套件而不是单独 module。
- 是否支持开票和售后。
- 是否现货。

价格判断：

- NVIDIA 官方标价是 249 美元，但国内实际价格会受汇率、渠道、税费和现货影响。
- 如果价格明显低于正常渠道，要警惕二手、拆机、单核心板或不含配件。

## 2026-07-03 电商初筛记录

淘宝搜索关键词：

- `Jetson Orin Nano Super Developer Kit`
- `Jetson Orin Nano Super Developer Kit 945-13766-0000-000`

当前看到的候选：

| 优先级 | 候选 | 页面价格信息 | 判断 |
| --- | --- | --- | --- |
| 推荐 | 微雪旗舰店：微雪 Jetson Orin Nano Super NVIDIA 英伟达原装 AI 开发套件 ROS | 约 2869 元，页面显示 24 人付款 | 优先考虑。店铺和资料沉淀较好，适合学习和排错。 |
| 可考虑 | AI硬件之家：英伟达 NVIDIA Jetson Orin Nano Super Developer Kit，型号 945-13766-0000-000 | 约 2100 元，页面显示 0 人付款 | 价格有吸引力，但下单前必须确认是否为官方 8GB 开发套件、是否含官方载板和 19V 电源。 |
| 可考虑 | 耿武电子 / 边缘存算一体机等店铺的 8G 开发者套件 | 约 2600-2890 元 | 需要逐项确认配件和售后。 |
| 暂不推荐 | 标价 493、450、1469、1809 元等明显低价或描述含糊的商品 | 价格低但描述可能是配件、载板、旧款 Jetson Nano B01、单 module 或兼容套件 | 不作为首次购买选择。 |

下单前必须向客服确认：

- 是否为 `NVIDIA Jetson Orin Nano Super Developer Kit`。
- 是否为 8GB。
- 型号是否为 `945-13766-0000-000`。
- 是否包含官方载板。
- 是否包含 19V 官方/原配电源。
- 是否可以开票。
- 是否支持 7 天无理由或质量问题退换。
- 是否现货。

闲鱼判断：

- 当前网页搜索结果加载不稳定，且二手风险高。
- 不建议作为第一次购买渠道。
- 如果一定看闲鱼，只考虑全新未拆、可验货、卖家能提供购买凭证和完整配件的商品。

## 到货后第一天检查

- 系统能否正常启动。
- JetPack 版本。
- CUDA、cuDNN、TensorRT 是否可用。
- 摄像头是否能被识别。
- `jtop` 能否查看温度、功耗和资源占用。
- 能否跑通一个官方 sample。

## 参考链接

- NVIDIA Jetson Orin Nano Super Developer Kit: https://www.nvidia.com/en-us/autonomous-machines/embedded-systems/jetson-orin/nano-super-developer-kit/
- NVIDIA Jetson Orin Nano Developer Kit User Guide: https://docs.nvidia.com/jetson/orin-nano-devkit/user-guide/latest/
- Hardware Layout: https://docs.nvidia.com/jetson/orin-nano-devkit/user-guide/latest/hardware_layout.html
- Supported Hardware: https://docs.nvidia.com/jetson/orin-nano-devkit/user-guide/latest/supported_hardware.html
