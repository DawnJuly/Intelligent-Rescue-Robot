# Intelligent-Rescue-Robot

智能救援机器人竞赛项目仓库,收录两阶段的完整方案:

- **2024 初赛方案**——基于 OpenCV 颜色识别的视觉方案;
- **2026 新方案**——基于 YOLOv8 训练 + CanMV-K230 边缘端部署的深度学习方案。

机器人通过视觉识别场地上不同颜色的球体、安全区与家区,经串口向运动机构发送指令,完成抓取与归位任务。

## 仓库结构

```
Intelligent_Rescue_Robot/
├── 24.3.29智能救援初赛/            # 2024 初赛方案(OpenCV 颜色识别)
│   ├── 12.4改/                     # 模块化重构版
│   └── 初赛/                       # 比赛版本与第0~10版迭代记录
└── 26.6.15recuse/                  # 2026 新方案(YOLOv8 + K230)
    ├── ultralytics-main/           # YOLOv8 训练工程(内置数据集与权重)
    ├── model/                      # nncase 转换产物(.kmodel)
    ├── python/                     # K230 端 MicroPython 脚本(01~16 迭代)
    ├── to_ONNX.py / to_kmodel.py   # 模型导出与转换脚本
    ├── take_photo.py               # 数据集采集
    └── resize.py / rename.py       # 数据整理
```

## 检测类别

模型共 8 类:红/蓝/黄/黑四种颜色的球(Red、Blue、Yellow、Black),红/蓝安全区(RedSafe、BlueSafe),红/蓝家区(RedHome、BlueHome)。

## 2026 方案:YOLOv8 + CanMV K230

完整链路:**训练 → 导出 ONNX → nncase 转 KModel → K230 端部署**。

1. **训练**:`ultralytics-main/` 内置 YOLOv8 框架、数据集与训练好的权重(`best800.pt`);
2. **导出 ONNX**:`to_ONNX.py` 将训练权重导出为 ONNX——opset 11、固定输入尺寸、batch=1、FP32 并用 onnxsim 简化算子,以匹配 K230 KPU 工具链约束;
3. **转换 KModel**:`to_kmodel.py` 使用 nncase 2.11.0(对应 Windows x64 的 whl 已随仓库提供)将 ONNX 转为 KPU 可运行的 `.kmodel`,产物在 `model/` 下;
4. **端侧部署**:`python/16K230.py` 为最终版脚本——在 CanMV MicroPython 上加载 `/sdcard/lastnew.kmodel`,运行 YOLOv8 检测(置信度阈值 0.75,NMS 阈值 0.75),检测结果经 UART1(波特率 115200,TX=3 / RX=4)发送给机器人主控。

`python/` 下 01~16 号脚本保留了从串口调试、单双球判断、安全区判断到 K230 集成的完整开发迭代过程。

## 2024 方案:OpenCV 颜色识别

- `color_detector.py`:Lab 颜色空间阈值分割,识别红/蓝/黄/黑四色目标,并处理矩形嵌套关系;
- `ball_processor.py`:实现 `BlueProcessor` / `RedProcessor` 两个处理器,对应蓝方/红方比赛策略,利用距离系数估算目标远近,按设定间隔循环检测;
- `main.py`:程序入口,负责初始化与异常处理;
- `camera_config.py`:摄像头参数配置;`serial_commmunicator.py`:串口收发;
- `初赛/测试/`:保留第0~10版完整迭代记录(多球判断、亮度调整、安全区逻辑、轻量化等)。

## 运行环境

| 阶段 | 环境 |
| --- | --- |
| 2024 视觉端 | Python + OpenCV,Lab 颜色空间阈值识别 |
| 2026 训练/转换 | PyTorch(Ultralytics)、ONNX、onnxsim、nncase 2.11.0 |
| 2026 部署端 | CanMV-K230(MicroPython),模型需拷贝至设备 `/sdcard/lastnew.kmodel` |

## 其他

- 2024 方案的颜色阈值基于 Lab 空间(L 亮度、a 红↔绿、b 黄↔蓝),现场光照变化时需重新标定;
- K230 端推理框架为 CanMV 官方 `libs.PipeLine` / `libs.YOLO`,模型输入大小与标签顺序需和训练配置保持一致。
