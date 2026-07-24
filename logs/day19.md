# Day 19 学习记录

日期：2026-07-24

主题：Severstal 数据验证、RLE、确定性划分、掩码可视化与 C++ 基础

状态：已完成

## 今日目标

- 下载并验证 Severstal Steel Defect Detection 完整数据集。
- 理解并验证 RLE 掩码编解码。
- 生成可复现的训练、验证、测试 manifest。
- 可视化四类钢材缺陷及多类别掩码。
- 按“会运行、能解释、能修改”完成核心模块学习。
- 补充 1 小时 C++17 基础，为项目 2 做准备。

## 数据集验证

实际数据统计：

```text
图像总数：12568
正缺陷图像：6666
正常图像：5902
正掩码总数：7095
类别 1/2/3/4：897 / 247 / 5150 / 801
图像尺寸：1600 x 256
缺失、额外、不可读图像：0
```

数据检查器会验证：

- 数据目录、CSV 和图像目录是否存在。
- CSV 列、类别编号和 RLE 结构是否合法。
- 标注引用的图片是否真实存在。
- 图像是否可解码、数量和尺寸是否符合配置。
- 输入异常时返回非零退出码，避免错误数据进入训练。

面试表述：

> 训练前的数据检查能够尽早发现损坏文件、标注错位和尺寸异常，避免训练数小时后才失败，也能防止静默错误导致模型性能下降但难以定位。

## RLE 编解码

Severstal RLE 使用：

- `start length` 成对表示连续正像素。
- 起始位置从 1 开始编号。
- 按列优先顺序展开和还原。
- 解码形状必须使用 `(height, width)`。

验证命令：

```bash
pytest -q tests/test_rle.py

python -c "from industrial_defect.rle import decode_rle, encode_rle; \
mask=decode_rle('1 2 8 2', (3, 4)); \
print(mask); print('encoded:', encode_rle(mask))"
```

结果：

```text
10 passed
[[1 0 0 0]
 [1 0 1 0]
 [0 0 1 0]]
encoded: 1 2 8 2
```

排查 RLE 错位时依次确认数据集展开顺序、`(height, width)`、是否从 1 编号，以及解码时是否正确减 1。

## 确定性 Manifest

正式划分：

```text
train：8798
val：1885
test：1885
seed：42
```

主要设计：

- 先为磁盘上的全部图片创建四维标签，补回未出现在稀疏 CSV 中的正常图片。
- 使用完整标签组合进行 label-powerset 分组。
- 优先分配稀有标签组合。
- 使用最大余数法保证 split 数量之和严格等于 12568。
- 使用 `SHA256(seed:image_id)` 生成稳定顺序，不依赖 CSV 行顺序。
- 同一源图像只进入一个集合，避免数据泄漏。

临时将比例改为 `80%/10%/10%`，输出到 `/tmp`，验证结果：

```text
train：10054
val：1257
test：1257
total：12568
```

原始标注哈希不变，manifest 哈希变化，说明数据源没有改变，但数据划分版本发生了变化。

## 掩码可视化

运行：

```bash
python scripts/visualize_masks.py \
  --count 4 \
  --alpha 0.45
```

结果覆盖四类缺陷，其中 `002fc4e19.jpg` 同时包含类别 1 和类别 2。脚本完成：

```text
CSV 正掩码读取
  -> 按图片和类别保存 RLE
  -> 每类选择代表样本
  -> RLE 解码为二维布尔掩码
  -> 统计缺陷像素面积
  -> 按 BGR 类别颜色进行 alpha blending
```

透明度对照：

- `alpha=0.15` 更容易观察原始钢材纹理。
- `alpha=0.85` 更容易迅速定位缺陷区域。

本地 overlay 位于被 `.gitignore` 忽略的 `outputs/` 中，不上传原始数据衍生图片。

## C++17 基础

工具链：

```text
g++ 11.4.0
CMake 3.22.1
```

练习内容：

- `struct` 组织图片名和四维缺陷标签。
- `std::array<int, 4>` 表达固定类别数量。
- `std::vector<DefectSample>` 保存数量可变的样本。
- `const Type&` 只读访问对象并避免复制。
- `const` 成员函数保证不修改当前对象。
- 使用严格警告编译 C++17 程序。

亲自增加的成员函数：

```cpp
int defect_count() const {
    int count = 0;

    for (int label : labels) {
        if (label == 1) {
            ++count;
        }
    }

    return count;
}
```

编译与运行：

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic \
  main.cpp -o defect_summary

./defect_summary
```

输出：

```text
steel_001.jpg has_defect=1 defect_classes=2
steel_002.jpg has_defect=0 defect_classes=0
steel_003.jpg has_defect=1 defect_classes=1
positive: 2/3
```

## 遇到的问题

### Manifest 行尾

`csv.writer` 默认写入 CRLF，导致 `git diff --check` 把 manifest 每行识别为尾随空白。将 writer 固定为 LF 后重新生成 manifest 和摘要，检查通过。

### C++ 花括号

最初把 `return count` 放在 `for` 循环内部。通过逐层标记函数、循环和条件的花括号，修正为循环结束后再返回。

### 编辑器未保存

VS Code 中修改后未保存，编译器仍读取磁盘上的旧文件。通过文件修改时间和 `grep` 定位问题，保存后重新编译。

### 可执行文件名

编译时使用 `-o defect_summary`，但首次误运行 `./defect`。通过 `ls -l` 确认真实文件名后正确执行。

## 工程验证与 GitHub

正式项目完整检查：

```text
pytest：23 passed
Ruff：All checks passed
git diff --check：passed
```

项目提交：

```text
e714340 feat: add deterministic Severstal data pipeline
```

已推送到：

```text
https://github.com/Nefelibata134/industrial-defect-inference-service
```

公开仓库没有提交 Kaggle token、原始图像、本地掩码图、模型权重或学习日志。

## 今日产出

- 完整数据检查报告。
- RLE 编解码模块和 10 个专项测试。
- 12,568 张图像的固定 split manifest。
- 数据划分统计和输入/划分哈希。
- 四类缺陷的本地掩码可视化。
- 数据下载、检查、划分与可视化的可复现命令。
- C++17 数据结构、只读引用和成员函数练习。
- 正式项目第二个工程提交并推送。

## 明日计划

Day20：

- 构建 PyTorch Dataset 和 DataLoader。
- 训练 U-Net + ResNet18 baseline。
- 记录训练配置、loss 曲线、Dice/IoU 和 checkpoint。
- 预留竞赛 baseline 时间。
