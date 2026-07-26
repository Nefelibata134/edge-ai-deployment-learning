# Day 22 学习记录

日期：2026-07-26

状态：已完成

## 今日目标

- 理解 PyTorch checkpoint 到 ONNX 模型的完整导出数据流。
- 导出固定空间尺寸、动态 batch 的四通道语义分割 ONNX 模型。
- 检查 ONNX 图结构、输入输出名称、形状和数据类型。
- 使用相同真实输入验证 PyTorch 与 ONNX Runtime 数值和掩码一致性。
- 为 Day23 TensorRT FP32/FP16 构建冻结模型契约和测试输入。

## 七小时时间分配

| 模块 | 时间 | 验收产出 |
| --- | ---: | --- |
| ONNX 导出原理 | 1 小时 | 能解释 checkpoint、`eval()`、dummy input、opset 和动态轴 |
| 实现导出器 | 1.5 小时 | `unet_resnet18.onnx` 与模型元数据 |
| ONNX 结构检查 | 1 小时 | checker 通过，确认 `images -> logits` |
| PyTorch/ORT 一致性 | 2 小时 | logits 误差、掩码不一致率和 parity report |
| 修改任务与面试题 | 0.5 小时 | 亲自完成一个参数修改并解释设计原因 |
| 竞赛或异常测试、文档 | 1 小时 | 低成本竞赛改进或正式项目边界测试 |

## 冻结的模型契约

```text
architecture: U-Net + ResNet18
checkpoint: models/class_aware_p075_e05_best_unet_resnet18.pt
checkpoint epoch: 2
checkpoint sha256:
a6a2a76857ba0a1a857a78204f77a82f58c09f26996f41ba51bac42479393ccc

input name: images
input dtype: float32
input shape: [batch, 3, 256, 1024]

output name: logits
output dtype: float32
output shape: [batch, 4, 256, 1024]

postprocess: sigmoid + threshold 0.80
```

空间尺寸固定为 `1024x256`，因为训练、验证和后续 benchmark 都使用该尺寸。
只把 batch 设为动态，允许 Triton 在不改变图像空间契约的情况下组成
`1/2/4/8` 批次。

## 导出数据流

```text
读取项目配置
-> 创建与训练时相同的 U-Net + ResNet18
-> 读取 checkpoint["model_state_dict"]
-> model.eval()
-> 创建 [1, 3, 256, 1024] float32 dummy input
-> torch.onnx.export()
-> onnx.checker.check_model()
-> 保存 checkpoint 哈希和模型契约
```

checkpoint 不是可独立推理的模型图，它主要保存参数和训练状态。导出时必须先
用同样的代码重建网络，再把 `model_state_dict` 加载进去。

`model.eval()` 会让 BatchNorm 和 Dropout 进入推理行为。即使当前网络没有
主动加入 Dropout，导出前显式调用仍是必要的推理契约。

dummy input 不只是“假数据”，它向导出器说明输入的 rank、通道数、空间尺寸
和数据类型，并触发一次前向计算以建立 ONNX 图。

## 今日验收标准

- ONNX checker 通过。
- 输入输出名称和形状符合冻结契约。
- ONNX Runtime 能加载并运行模型。
- PyTorch 和 ORT logits 的误差处于可接受范围。
- 阈值 `0.80` 后的二值掩码几乎完全一致。
- 正式项目静态检查和测试全部通过。

## 动手记录

1. 检查候选 checkpoint 内容，确认其包含参数、训练配置和恢复训练状态，但
   不包含可直接独立执行的网络结构。
2. 冻结输入输出契约，导出 U-Net ResNet18 ONNX 模型。
3. 使用 ONNX checker 和独立检查脚本验证图结构、opset、元数据和制品哈希。
4. 使用 8 张固定验证图片比较 PyTorch 与 ONNX Runtime 的 logits、二值掩码
   和宏平均 Dice。
5. 将验证批量由 `2` 改为 `4`，不重新导出模型，亲自确认动态 batch 可用。

```text
status: pass
batch: 4
max error: 9.72747802734375e-05
mean error: 1.9139116034239123e-06
mismatch: 0
```

动态 batch 的含义是第 0 维使用符号 `batch`，因此 `[2, 3, 256, 1024]`
和 `[4, 3, 256, 1024]` 都满足同一张 ONNX 图。通道数、高度和宽度仍固定，
改变这些维度不能直接复用当前契约。

## 遇到的问题

- 最初配置 opset 17，但当前 PyTorch 导出器在处理 `Resize` 时无法将实际图
  完整降级到 opset 17，生成图仍为 opset 18。
- 将项目配置明确改为 opset 18，并在导出后增加实际 opset 断言，避免配置
  与制品不一致。
- 小规模一致性回归集中的 Dice 只用于比较两个后端是否一致，不能代替
  1,885 张验证集上的正式模型质量指标。

## 今日产出

- ONNX 模型：`models/unet_resnet18_severstal.onnx`，opset 18。
- ONNX SHA256：
  `6622d3aada8f7fe0547c475f80a8697cb74f7e6e8c6b677a9dbaff45a4f03a0c`。
- 输入：`images`，float32，`[batch, 3, 256, 1024]`。
- 输出：`logits`，float32，`[batch, 4, 256, 1024]`。
- PyTorch/ORT 最大绝对误差 `9.73e-05`，平均绝对误差 `1.91e-06`。
- 阈值 `0.80` 后掩码不一致像素数为 `0`，宏平均 Dice 差值为 `0`。
- 正式项目新增可复现导出器、图检查器、一致性验证器、测试和工程报告。
- Ruff 全部通过，Pytest `52 passed`。

## 面试追问

### 为什么 checkpoint 不能直接交给 ONNX Runtime

checkpoint 中的 `model_state_dict` 是参数名到张量的映射，不包含完整的
ONNX 计算图。导出时要先用同一份模型代码重建网络、加载参数并切换到
`eval()`，再通过示例输入生成图。

### 为什么输出 logits 而不把 sigmoid 和阈值写进图

logits 保留连续分数，便于统一校准阈值、分析置信度和复用不同后处理策略。
部署端按照冻结契约执行 `sigmoid + 0.80`，既保持可调整性，也能验证各后端
后处理一致。

### 为什么 batch 改成 4 不需要重新导出

导出时只把输入和输出的第 0 维声明为动态符号 `batch`。批量大小是运行时
形状，不是固定图常量；只要通道和空间尺寸不变，ONNX Runtime 可以直接用
同一图执行不同批量。

## 明日计划

- 构建 TensorRT FP32 与 FP16 engine。
- 使用统一输入完成 PyTorch、ORT、TensorRT benchmark。
- 记录平均延迟、P50、P95、吞吐量、显存和模型大小。
