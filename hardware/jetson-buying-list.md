# Jetson Orin Nano 采购清单

## 推荐购买

- Jetson Orin Nano Developer Kit 或 Jetson Orin Nano Super Developer Kit。
- NVMe SSD：建议 500GB 起步。
- 稳定电源：按官方套件要求购买，不要随便用低功率电源。
- 摄像头：先买 USB 摄像头，后续再考虑 CSI 摄像头。
- 散热：确认套件自带散热；如果长时间推理，注意风扇和温度。
- 网线或稳定 Wi-Fi。
- HDMI/DP 显示器与线材，或准备远程连接方案。
- 读卡器和备用 microSD 卡。

## 可选购买

- 小车底盘。
- 电机驱动模块。
- 移动电源或电池组。
- 超声波/激光/IMU 传感器。
- 亚克力板、杜邦线、螺丝铜柱。

## 先不急买

- 昂贵的深度相机。
- 很复杂的机械结构。
- 多个传感器套件。

先把 Jetson + 摄像头 + 目标检测跑通，再升级小车部分。

## 到货后第一天检查

- 系统能否正常启动。
- JetPack 版本。
- CUDA、cuDNN、TensorRT 是否可用。
- 摄像头是否能被识别。
- `jtop` 能否查看温度、功耗和资源占用。
- 能否跑通一个官方 sample。

