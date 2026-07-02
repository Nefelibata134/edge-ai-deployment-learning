# 作品集项目规划

## 项目 1：ONNX/TensorRT 图像分类部署

目标：完成从 PyTorch 到 ONNX 到 TensorRT 的完整部署链路。

展示重点：

- 模型导出和推理脚本。
- PyTorch、ONNX Runtime、TensorRT 性能对比。
- FP32/FP16 指标对比。

## 项目 2：Jetson 实时目标检测

目标：在 Jetson Orin Nano 上运行摄像头实时检测。

展示重点：

- 摄像头实时输入。
- YOLO ONNX/TensorRT 部署。
- FPS、延迟、温度、功耗记录。
- 优化前后对比。

## 项目 3：模型推理服务化

目标：把模型封装为可访问的服务。

展示重点：

- FastAPI 或 Triton 推理服务。
- Docker 部署。
- 压测结果。
- 日志和错误处理。

