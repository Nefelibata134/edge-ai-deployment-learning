# Day 08 学习记录

日期：2026-07-06

## 今日目标

- 使用 VS Code 继续开发，不再用 `nano` 写主要脚本。
- 理解 ONNX 是什么，以及为什么模型部署要从 `.pt` 走向 `.onnx`。
- 将 `yolo11n.pt` 导出为 ONNX 模型。
- 用命令行和 Python 检查 ONNX 文件是否生成成功。
- 使用 Netron 查看 ONNX 模型的输入输出结构。
- 为 Day09 ONNX Runtime 推理做准备。

## 今日 7 小时学习计划

| 时间 | 内容 | 目标产出 |
| --- | --- | --- |
| 第 1 小时 | 复习 Day07 预处理 | 确认 `(1, 3, 640, 640)` 是模型输入形状 |
| 第 2 小时 | 理解 ONNX | 知道 `.pt`、`.onnx`、TensorRT engine 的关系 |
| 第 3 小时 | 导出 YOLO ONNX | 生成 `yolo11n.onnx` |
| 第 4 小时 | 检查 ONNX 文件 | 用 Python 打印模型输入输出信息 |
| 第 5 小时 | Netron 查看结构 | 看到输入名、输出名、输入尺寸、输出尺寸 |
| 第 6 小时 | 整理导出记录 | 写 `inspect_onnx.py`，记录导出命令和结果 |
| 第 7 小时 | 复盘 | 总结 ONNX 在部署链路中的位置 |

如果 Jetson Orin Nano 今天到货，可以穿插做开箱检查，但主线仍然先完成 ONNX 导出。

## VS Code 工作流

从今天开始，默认：

- VS Code 负责编辑脚本。
- VS Code 终端负责运行命令。
- 右下角 Python 解释器选择：

```text
/home/nefelibata/miniconda3/envs/deploy310/bin/python
```

如果自动补全消失，优先检查解释器是否是 `deploy310`。

## 开始前准备

在 WSL2 Ubuntu / VS Code 终端执行：

```bash
conda activate deploy310
mkdir -p ~/model-deploy-day08/models ~/model-deploy-day08/scripts ~/model-deploy-day08/outputs
cd ~/model-deploy-day08
python --version
which python
yolo version
```

需要看到：

```text
Python 3.10.x
/home/nefelibata/miniconda3/envs/deploy310/bin/python
8.4.87
```

复制 Day06 已下载的权重：

```bash
cp ~/model-deploy-day06/yolo11n.pt models/yolo11n.pt
ls -lh models
```

如果复制失败，说明 Day06 目录里没有权重文件，可以后面导出时让 Ultralytics 自动下载。

## 必做 1：理解 ONNX 的位置

今天先记住这条部署链：

```text
PyTorch / Ultralytics .pt
-> ONNX .onnx
-> ONNX Runtime 验证
-> TensorRT engine
-> PC / Jetson 部署
```

简单理解：

- `.pt`：训练框架里的模型，适合训练和快速验证。
- `.onnx`：更通用的模型交换格式，适合跨框架部署。
- TensorRT engine：NVIDIA GPU 上进一步优化后的推理文件，适合追求速度。

今天只做 `.pt -> .onnx`，先不进入 TensorRT。

## 必做 2：导出 YOLO ONNX

在 `~/model-deploy-day08` 目录执行：

```bash
yolo export model=models/yolo11n.pt format=onnx imgsz=640 opset=12 simplify=True
```

如果 `models/yolo11n.pt` 不存在，就执行：

```bash
yolo export model=yolo11n.pt format=onnx imgsz=640 opset=12 simplify=True
```

导出完成后检查：

```bash
find . -name "*.onnx" -type f -ls
```

预期能看到类似：

```text
models/yolo11n.onnx
```

说明：

- `format=onnx`：导出 ONNX。
- `imgsz=640`：输入尺寸 640。
- `opset=12`：ONNX 算子版本，先用稳定值。
- `simplify=True`：尝试简化 ONNX 图，方便后续部署。

## 必做 3：安装 ONNX 检查工具

执行：

```bash
python -m pip install onnx netron
```

检查：

```bash
python -c "import onnx; print(onnx.__version__)"
python -c "import netron; print(netron.__version__)"
```

## 必做 4：用 Python 检查 ONNX 输入输出

在 VS Code 中新建：

```text
scripts/inspect_onnx.py
```

写入：

```python
from pathlib import Path

import onnx


def dim_to_str(dim):
    if dim.dim_value:
        return str(dim.dim_value)
    if dim.dim_param:
        return dim.dim_param
    return "?"


model_path = Path("models/yolo11n.onnx")

if not model_path.exists():
    raise FileNotFoundError(f"ONNX model not found: {model_path}")

model = onnx.load(str(model_path))
onnx.checker.check_model(model)

print("model:", model_path)
print("ir_version:", model.ir_version)
print("opset:", [opset.version for opset in model.opset_import])

print("\ninputs:")
for input_info in model.graph.input:
    shape = [
        dim_to_str(dim)
        for dim in input_info.type.tensor_type.shape.dim
    ]
    elem_type = input_info.type.tensor_type.elem_type
    print("-", input_info.name, "shape=", shape, "elem_type=", elem_type)

print("\noutputs:")
for output_info in model.graph.output:
    shape = [
        dim_to_str(dim)
        for dim in output_info.type.tensor_type.shape.dim
    ]
    elem_type = output_info.type.tensor_type.elem_type
    print("-", output_info.name, "shape=", shape, "elem_type=", elem_type)
```

运行：

```bash
python scripts/inspect_onnx.py
```

你需要重点看：

- input name 是什么。
- input shape 是不是类似 `[1, 3, 640, 640]`。
- output shape 是什么。

## 必做 5：用 Netron 查看 ONNX

启动 Netron：

```bash
netron models/yolo11n.onnx --host 0.0.0.0 --port 8081
```

然后在浏览器打开：

```text
http://localhost:8081
```

需要记录：

- 模型输入名。
- 模型输入尺寸。
- 模型输出名。
- 模型输出尺寸。
- 第一层大概是什么，最后一层大概是什么。

看完后在终端按：

```text
Ctrl + C
```

停止 Netron。

## 进阶：导出动态 batch ONNX

如果前面完成很快，可以再导出一个动态版本：

```bash
yolo export model=models/yolo11n.pt format=onnx imgsz=640 dynamic=True opset=12 simplify=True
```

然后再运行：

```bash
python scripts/inspect_onnx.py
```

注意：如果它覆盖了原来的 `models/yolo11n.onnx`，可以先复制保存：

```bash
cp models/yolo11n.onnx models/yolo11n_static.onnx
```

今天不要求完全理解动态维度，只需要知道：

- 固定输入：部署简单，形状更明确。
- 动态输入：更灵活，但部署和优化会更复杂。

## Jetson 到货检查

如果 Jetson 今天到货，先做低风险检查：

- 拍照保存外包装、主板、接口、SSD、配件。
- 确认是不是官方载板。
- 确认是否有 SD 卡槽。
- 确认 256G SSD 是否已经装好。
- 不急着折腾 TensorRT，先确认硬件无误。

## 今日完成情况

待完成后补充。

## 遇到的问题

待完成后补充。

## 今日复盘

今天最重要的收获：

待完成后补充。

还不清楚的点：

待完成后补充。

## 明日计划

- Day09：使用 ONNX Runtime 跑 YOLO ONNX 推理。
- 对比 `.pt` 推理和 `.onnx` 推理的输入输出差别。
- 开始理解后处理：输出张量、置信度过滤、NMS。
