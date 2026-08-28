# 项目 3：Jetson + STM32 采购清单

记录日期：2026-08-28

对应项目：`projects/project3-jetson-stm32-vision-safety-controller.md`

采购原则：分两批购买。第一批只打通 STM32、SocketCAN 和真实 CAN 总线；
CAN 双向通信与心跳超时通过后，再购买电机和机械件。

## 已有设备

- Jetson Orin Nano Super Developer Kit 8GB。
- 256GB NVMe SSD、官方供电和网络环境。
- IMX219 CSI 摄像头。
- 项目 2 的 C++ 视觉检测、跟踪、事件与 systemd 运行链路。

不需要再次购买 Jetson、摄像头、GPU 或另一块 SoC。

## 第一批：CAN 与 MCU 必买

| 物品 | 数量 | 用途 | 购买要求 |
| --- | ---: | --- | --- |
| `NUCLEO-G474RE` 原装开发板 | 1 | STM32G474RE、FreeRTOS、PWM、编码器、FDCAN | 必须是原装 Nucleo；板载 ST-LINK，不另买下载器 |
| 微雪 `SN65HVD230 CAN Board` | 2 | Jetson 和 STM32 各自连接 CAN 物理层 | 3.3V 供电、带 ESD 保护 |
| 120 ohm 终端电阻 | 2 | CAN 总线两端终端 | 如果模块自带可切换 120 ohm，避免重复并联 |
| Micro-B USB 数据线 | 1 | Nucleo 供电、烧录和调试 | 必须能传数据，不是充电专用线 |
| 双绞线 | 1-2 米 | CANH/CANL | 两端短支线，避免松散长线 |
| 杜邦线 | 1 套 | 3.3V、GND、TX、RX、GPIO | 需要公对母和母对母 |
| 面包板、220 ohm 电阻、LED、按键、主动蜂鸣器 | 1 套 | 无电机阶段验证状态机 | 优先购买已焊针脚模块 |
| 数字万用表 | 1 | 电压、通断和接地检查 | 已有则不买 |

说明：Jetson Orin Nano 和 STM32G474RE 都有 CAN 控制器，但控制器不能直接接
CANH/CANL，所以两个节点仍各需要一个外部收发器。第一版使用经典 CAN
500 kbit/s。SN65HVD230 支持 3.3V 和最高 1 Mbit/s，不把它用于 CAN FD 高速数据段。

2026-08-28 参考价格：

- `NUCLEO-G474RE`：DigiKey 含税单价约 166.47 元，不含可能的运费。
- 微雪 `SN65HVD230 CAN Board`：国内元器件渠道约 21.28 元/块，库存和运费需下单时复核。
- 第一批已有万用表时预计约 250-400 元；没有万用表时增加约 50-150 元。

## 第二批：物理执行机构

| 物品 | 数量 | 用途 | 购买要求 |
| --- | ---: | --- | --- |
| 6V 带 AB 相霍尔编码器减速电机 | 1 | 演示正常、减速和停车并反馈实际转速 | 选择低速桌面电机；必须确认堵转电流 |
| `TB6612FNG` 电机驱动模块 | 1 | PWM、方向、制动和 `STBY` 硬件使能 | 不买裸芯片；优先资料完整的成品模块 |
| 6V 2A 稳压电源 | 1 | 电机独立供电 | 与所选电机额定电压匹配，不从 Jetson/Nucleo 取电 |
| 蘑菇头自锁急停开关 | 1 | 硬件禁止电机和 MCU 急停输入 | 同时作用于驱动 `STBY/EN` 与 GPIO |
| 瞬时复位按钮 | 1 | 故障解除后的人工重新许可 | 恢复通信不能自动运行 |
| 红、黄、绿 LED 与主动蜂鸣器 | 1 套 | 展示 `FAULT/STOP/WARNING/NORMAL` | 配限流电阻 |
| 保险丝、接线端子、开关 | 1 套 | 低压电机供电保护和可靠接线 | 规格按电机工作与堵转电流确定 |
| 电机固定架或可见旋转盘 | 1 | 直观看到转速变化 | 第一版不购买完整小车底盘 |

TB6612FNG 芯片规格为最大 15V、平均 1.2A、峰值 3.2A。选购电机时不能只看
额定电流，还必须核对堵转电流；不满足余量时应更换更高电流驱动器。

第二批预计约 150-300 元。完整第一版原型预计约 400-700 元，价格不含已有
Jetson、IMX219 和可能已有的工具。

## 可选调试工具

| 物品 | 何时购买 | 作用 |
| --- | --- | --- |
| 8 通道 USB 逻辑分析仪 | 遇到 PWM、编码器或中断时序问题后 | 观察数字信号时序 |
| `gs_usb` / CANable 兼容 USB-CAN | 需要 PC 作为第三个独立节点时 | 抓包、注入和总线故障测试 |
| 洞洞板、焊台和热缩管 | 面包板验证完成后 | 固化演示接线 |

这些工具不是第一版启动条件。Jetson 上的 `candump`、`cansend` 和 SocketCAN
已经能够完成最初的双节点抓包与测试。

## 暂时不要买

- MCP2515 SPI CAN 模块：Jetson 和 STM32 已有 CAN 控制器，额外控制器只增加驱动与电平问题。
- 完整小车底盘、锂电池组、IMU、激光雷达和超声波套件。
- 第二块 Jetson、PLC、昂贵示波器或工业 CAN 分析仪。
- CAN FD 高速收发器：第一版经典 CAN 带宽足够。
- BLDC 电机、FOC 驱动器和三相功率板。
- 自制 PCB 和隔离 CAN 模块：桌面短线验证通过后再按需求评估。

## 下单前确认

- Nucleo 商品型号完整写明 `NUCLEO-G474RE`，并包含板载 ST-LINK。
- 两块 CAN 模块都是 3.3V SN65HVD230，不是常见的 5V MCP2515 模块。
- 确认 CAN 模块是否自带 120 ohm 终端，确保总线上最终只有两处终端。
- 电机额定电压、工作电流和堵转电流与电源、驱动模块匹配。
- 电机电源独立于 Jetson 和 Nucleo，联调时建立正确公共参考地。
- 急停首先切断或禁止电机驱动，再把状态通知 MCU；不能只实现软件按钮。

## 技术资料

- NVIDIA Jetson CAN：<https://docs.nvidia.com/jetson/archives/r36.5/DeveloperGuide/HR/ControllerAreaNetworkCan.html>
- STM32G474RE：<https://www.st.com/en/microcontrollers-microprocessors/stm32g474re.html>
- NUCLEO-G474RE：<https://www.st.com/en/evaluation-tools/nucleo-g474re.html>
- Linux SocketCAN：<https://docs.kernel.org/networking/can.html>
- SN65HVD230：<https://www.ti.com/lit/ds/symlink/sn65hvd230.pdf>
- TB6612FNG：<https://toshiba.semicon-storage.com/info/datasheet_en_20141001.pdf?did=10660>
- FreeRTOS queue：<https://www.freertos.org/Documentation/02-Kernel/02-Kernel-features/02-Queues-mutexes-and-semaphores/01-Queues>
