# 2026 暑假个人竞赛候选表

最后核对日期：2026-07-14

## 筛选目标

- 优先允许个人参赛。
- 优先计算机视觉任务。
- 能在 Day22-Day25 前完成第一次有效提交。
- 可以使用 RTX 4070 或合理付费云算力。
- 竞赛成果能沉淀为训练、评估、实验记录或项目功能。
- 最多保留一个主赛和一个低成本练习赛，不同时重投入多个比赛。

## 初始候选

| 候选 | 类型 | 与主线匹配 | 单人资格 | 当前状态 | 初步结论 |
| --- | --- | --- | --- | --- | --- |
| 天池海藻目标检测练习赛 | YOLO 目标检测 | 高 | 待登录确认 | 页面可读取，需确认提交入口 | 第一优先，适合完成首次有效提交 |
| 第七届 CSIG 复杂工业场景异常检测算法挑战 | 工业异常检测 | 很高 | 已以个人账号报名，团队细则后续复核 | 2026-07-12 报名成功，进行中 | 正式主赛候选，与项目 1 高度相关 |
| CCL2026-Eval 跨境电商图像文本翻译大赛 | 图像文本翻译 | 低 | 已以个人账号报名 | 2026-07-12 报名成功，进行中 | 保持报名，暂不作为暑期主投入 |
| 天池地表建筑物识别练习赛 | 航拍语义分割 | 中 | 待登录确认 | 练习赛页面可读取 | 目标检测赛不可用时的备选 |
| Kaggle Digit Recognizer | MNIST 图像多分类 | 中 | 可个人完成 | 2026-07-13 完成 baseline 与平台接受的首次提交 | 已跑通 Kaggle 全流程；后续只做低成本优化，不挤占项目主线 |
| Kaggle Biohub Cell Tracking | 3D 检测、跟踪与谱系重建 | 高 | 可单人成队，队伍上限 5 人 | 报名截止 2026-09-22，提交截止 2026-09-29 | 难度过高，仅保留为后续高难度备选 |

## 官方页面

- [天池海藻目标检测练习赛](https://tianchi.aliyun.com/competition/entrance/532171/information)
- [第七届 CSIG 复杂工业场景异常检测算法挑战](https://tianchi.aliyun.com/competition/entrance/532482)
- [天池地表建筑物识别练习赛](https://tianchi.aliyun.com/competition/entrance/531872/information)
- [CCL2026-Eval 跨境电商图像文本翻译大赛](https://tianchi.aliyun.com/competition/entrance/532463)
- [Kaggle Competitions](https://www.kaggle.com/competitions)
- [Kaggle Digit Recognizer](https://www.kaggle.com/competitions/digit-recognizer)
- [Kaggle Biohub - Cell Tracking During Development](https://www.kaggle.com/competitions/biohub-cell-tracking-during-development)

## Day13-Day15 核对表

登录平台后逐项填写：

| 项目 | 目标检测练习赛 | CSIG 工业异常检测赛 | 地表建筑物识别 | Kaggle Digit Recognizer |
| --- | --- | --- | --- | --- |
| 可否报名 | 待填写 | 已报名 | 待填写 | 可参与并提交 |
| 可否单人参赛 | 待填写 | 已以个人账号报名，细则待复核 | 待填写 | 可以 |
| 截止时间 | 待填写 | 待页面复核 | 待填写 | 长期开放 |
| 数据大小 | 待填写 | 待填写 | 待填写 | 待填写 |
| 评价指标 | mAP50-95 | 待填写 | 语义分割指标，待填写 | 分类准确率 |
| 提交格式 | JSON，待实际下载模板确认 | 待填写 | 待填写 | CSV 标签预测 |
| baseline 难度 | 低 | 高 | 中 | 低 |
| 是否保留 | 待决定 | 主赛候选 | 待决定 | 已完成首次提交，保留为低成本练习赛 |

## 最终选择

Day13 初步选择，最迟 Day15 复核：

```text
主赛：第七届 CSIG 复杂工业场景异常检测算法挑战
练习赛：Kaggle Digit Recognizer
选择理由：主赛与工业视觉项目高度相关；练习赛用于低成本跑通 Kaggle 全流程
第一次有效提交：已于 Day14 提前完成 Kaggle Digit Recognizer 平台有效提交
每周投入时间：4-5 小时
使用算力：RTX 4070、Kaggle 免费 Notebook；必要时使用合理付费云算力
```

## 重要说明

- 页面可访问不等于当前仍可报名或提交，必须登录后核对按钮和规则。
- “允许团队最多 5 人”不自动等于本科生一定有资格，仍需阅读参赛对象条款。
- 练习赛可以用于完成竞赛全流程，但简历中要如实写为练习赛或学习赛。
- 如果正式比赛与两个就业项目严重冲突，优先保证项目交付，只保留练习赛有效提交。

## 已完成里程碑

- 2026-07-13：完成 Kaggle `Digit Recognizer` baseline，生成提交文件并获得平台接受，首次有效提交目标提前完成。
- 当前没有记录具体 Public Score 和排行榜位置，待用户提供后再补充，不能凭印象填写。
- 下一阶段竞赛重点转向 CSIG 工业异常检测赛的规则、数据和 baseline，并保留一次 Digit Recognizer 低成本优化作为可选任务。
