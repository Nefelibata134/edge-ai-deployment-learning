# Day 32：MOT17 跟踪评估与最终模型选型

日期：2026-08-14 至 2026-08-15

状态：已完成

## 今日目标

- 建立符合 MOTChallenge 规范的离线序列推理与结果输出流程。
- 使用固定版本 TrackEval 计算 HOTA、IDF1、MOTA 和 ID switches。
- 将公开训练序列预先划分为 calibration 与 holdout，隔离调参与最终评估。
- 在相同后处理和 ByteTrack 参数下比较 YOLOX-Nano 与 YOLOX-Tiny。
- 冻结最终配置，只运行一次 holdout，形成可复现的跟踪质量报告。

## 7 小时计划

| 时间 | 内容 | 验收标准 |
| --- | --- | --- |
| 1.0 小时 | 理解跟踪指标与 MOTChallenge 十列格式 | 能区分帧数、轨迹观测数和唯一 ID 数 |
| 1.0 小时 | 建立结果写入器和确定性测试 | 行数、字段数、帧号和坐标约定正确 |
| 1.0 小时 | 准备 MOT17 与固定 TrackEval | 数据校验通过，评估器版本可追溯 |
| 1.5 小时 | 运行 Nano calibration 基线与阈值对照 | 获得受控参数实验结果 |
| 1.0 小时 | 同参数比较 Nano/Tiny | 只改变模型容量，完成最终模型选择 |
| 1.0 小时 | 一次性运行 holdout | 输出最终泛化指标与性能统计 |
| 0.5 小时 | 整理报告和复盘 | 公开协议、结果表和限制说明完整 |

## 评估数据流

```text
MOT17 image sequence
    -> sequential image decode
    -> YOLOX TensorRT FP16 detector
    -> person detections
    -> ByteTrack
    -> MOTChallenge ten-column rows
    -> pinned TrackEval
    -> HOTA / IDF1 / MOTA / IDSW
```

离线评估不使用实时采集队列，也不丢帧。每张图按原始顺序处理，保证结果帧号与标注严格对应。只使用 FRCNN 命名序列，避免 MOT17 中同一个物理视频的 DPM、FRCNN、SDP 三份副本被重复统计。

## 数据划分

| 分区 | 序列 | 用途 |
| --- | --- | --- |
| Calibration | 02、04、05、10-FRCNN | 比较阈值与模型，选择配置 |
| Holdout | 09、11、13-FRCNN | 配置冻结后的唯一一次最终评估 |

MOT17 官方测试集不公开标注，因此使用公开训练序列建立固定 calibration/holdout。holdout 结果不能继续用于修改阈值或模型，否则会发生测试集信息泄漏，holdout 也会在事实上变成新的验证集。

## MOTChallenge 输出契约

每条轨迹观测写成十列：

```text
frame, track_id, x, y, width, height, confidence, -1, -1, -1
```

- 一行表示某个目标在某一帧的一次轨迹观测，不等于一条完整轨迹。
- 同一个 `track_id` 可以跨多帧产生多行。
- 帧号和边界框起点按 MOTChallenge 要求转换为从 1 开始。
- 确定性测试验证了输出字段数、行数、跳过无效轨迹和统计守恒。

## Calibration 参数对照

Nano 基线配置为 `track=0.50`、`new=0.60`，得到：

| HOTA | IDF1 | MOTA | IDSW |
| ---: | ---: | ---: | ---: |
| 27.51 | 30.76 | 21.49 | 144 |

降低跟踪阈值后，漏检逐步减少，但误检、碎片和 ID switches 增加：

| 配置 | HOTA | IDF1 | MOTA | IDSW | TP | FN | FP |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Nano `t040/n050` | 28.91 | 33.24 | 23.17 | 175 | 23,636 | 62,258 | 3,561 |
| Nano `t030/n040` | 29.19 | 34.50 | 24.38 | 227 | 25,997 | 59,897 | 4,830 |

`t030/n040` 的 MOTA 继续提高，是因为相比上一组减少的 2,361 次漏检，大于新增误检和 ID 切换的总增量。继续降低阈值已经明显损害关联质量，因此停止阈值搜索。

## Nano 与 Tiny 受控比较

只更换检测模型，其余参数保持一致：

```text
score=0.10
nms=0.45
track=0.30
new_track=0.40
match=0.80
track_buffer=30
```

| 模型 | HOTA | IDF1 | MOTA | IDSW | Recall | Precision |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| YOLOX-Nano FP16 | 29.19 | 34.50 | 24.38 | 227 | 30.27% | 84.33% |
| YOLOX-Tiny FP16 | **33.31** | **39.80** | **29.65** | 232 | **34.14%** | **89.00%** |

Tiny 同时提高召回率、精确率和关联质量，HOTA 提高 4.12，IDF1 提高 5.30，MOTA 提高 5.27。虽然多出 5 次 ID 切换，但整体收益更大。其序列推理 P95 约 14 ms，仍满足本项目的 30 FPS 目标，因此最终选择 YOLOX-Tiny FP16。

## 最终 Holdout 结果

最终配置冻结后处理 3 个序列、共 2,175 帧：

| HOTA | DetA | AssA | IDF1 | MOTA | Recall | Precision | IDSW |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| **38.89** | 35.26 | 43.36 | **46.75** | **39.19** | 46.00% | 87.93% | 132 |

| TP | FN | FP | GT detections | Output detections |
| ---: | ---: | ---: | ---: | ---: |
| 12,146 | 14,257 | 1,667 | 26,403 | 13,813 |

性能结果：

- TensorRT 推理 P95：13.98 至 14.07 ms。
- ByteTrack P95：0.28 至 0.32 ms。
- 有效吞吐量：30.54 至 33.00 FPS。

holdout 分数高于 calibration，主要因为检测召回率从 34.14% 提高到 46.00%，说明这些序列对当前检测器相对容易。它不代表可以继续根据 holdout 调参，也不能说明系统达到 MOT17 最优水平。

## 会运行、能解释、能修改

### 会运行

- 校验固定 MOT17 序列的图片、标注、帧数和编号。
- 生成标准 MOTChallenge 跟踪结果。
- 运行固定版本 TrackEval 并生成 JSON、Markdown 指标摘要。

### 能解释

- 为什么一行是轨迹观测，而不是一条完整轨迹。
- HOTA、IDF1、MOTA 和 IDSW 分别观察什么问题。
- 为什么降低阈值会同时减少漏检并增加误检与 ID 切换。
- 为什么 calibration 可以调参，而 holdout 只能用于一次最终评估。
- 为什么 holdout 分数更高不等于发生了数据泄漏。

### 能修改

- 修改跟踪阈值，观察 TP、FN、FP、IDSW 和综合指标的变化。
- 在参数完全一致时将 Nano 替换为 Tiny，完成只改变单一变量的受控比较。
- 修正报告标题，避免将 calibration 结果误标为 holdout。

## 遇到的问题

- Jetson 无法直接访问 MOTChallenge 下载地址，改为在笔记本获取压缩包后传入 Jetson。
- Jetson 访问 GitHub 偶发 GnuTLS 中断，使用重试和 Git bundle 保持代码同步。
- 固定 TrackEval 版本要求兼容旧 NumPy 别名；使用 Ubuntu 22.04 系统 NumPy，并修正版本门禁判断。
- TrackEval 提示缺少 `pycocotools`，该依赖属于未使用的可选 BURST 数据集，不影响 MOT17 评估。
- 初版摘要标题固定写成 Holdout，导致 calibration 报告标注错误；增加可配置标题和测试后修复。

## 今日产出

- MOTChallenge 十列输出器及测试。
- MOT17 数据完整性校验、序列推理和 TrackEval 包装脚本。
- 固定 calibration/holdout 协议。
- Nano 阈值对照、Nano/Tiny 同参数比较和最终 holdout 指标。
- 正式项目中的 MOT17 评估协议、结果文档和机器可读 CSV。
- 正式项目提交：`7620630 docs: publish MOT17 tracking evaluation`。

## 复盘

- 实时 FPS 只能证明系统跑得动，MOT17 与 TrackEval 补上了跟踪质量证据。
- 跟踪结果受检测器上限明显影响，Tiny 的检测改进同时抬高 DetA、AssA、HOTA 和 IDF1。
- 阈值优化必须观察多个指标，不能只追求 MOTA 或只减少漏检。
- calibration/holdout 隔离比多跑几组参数更重要，它决定最终指标是否可信。
- 当前系统具有实时性和可复现评估，但漏检与身份切换仍是明确限制，报告中应如实保留。

## 下一步

- Day33：实现 ROI 闯入、方向越线和停留状态机，并使用合成轨迹完成确定性事件测试。
