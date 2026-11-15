# 第三方归属与使用边界

本仓库的 `geotoponet/`、`scripts/`、`tests/`、`configs/`、`docs/` 均为自有实现，按 [MIT](LICENSE) 发布。**仓库不复制任何第三方项目源码、权重、数据或标注。**

## 运行时依赖

| 组件 | 用途 | 许可（以各项目官方声明为准） |
|---|---|---|
| PyTorch / torchvision | 模型与 ImageNet 预训练 ResNet34 骨干 | BSD-3-Clause |
| NumPy / SciPy / scikit-image | 数组、稀疏直接解、骨架化 | BSD-3-Clause |
| Pillow | 合成集 PNG 读写 | HPND |
| tifffile | 显微 CT 切片只读 | BSD-3-Clause |

`ResNet34_Weights.IMAGENET1K_V1` 权重由 torchvision 在首次使用时下载，本仓库不分发该权重；ImageNet 数据集本身另有使用条款，使用者需自行确认。离线环境可用 `--no-pretrained` 跳过下载，但那只是结构自检，**不是论文协议**。

## 外部资产（需自备，本仓库不发布）

- **源域 ResUNet34 权重**：正式协议从一份外部裂缝数据集上预训练的源域权重出发（`--source-ckpt`）。该权重不属于本仓库，使用前请自行核对其来源项目的许可与引用要求。
- **真实显微 CT 切片与 TWS 参考标注**：来自用户自有实验数据；标注由 Fiji/ImageJ 的 Trainable Weka Segmentation 软件导出，**不是人工金标准**。本仓库不含任何切片或标注文件，`--index` 指向的 `index.json` 需自行准备。
- **合成预训练集**：由同项目的另一份仓库（物理约束 3D+2D 合成数据生成器，MIT）产出，本仓库只按 `images/` + `masks/` 目录格式读取，不复制其源码，也不内置样本。

## 引用边界

- 论文正式成绩（三折 Dice 0.6582 / 0.6944 / 0.4153 等）记录在本仓库文档中仅供审计，**不随代码重新计算**；本仓库不发布产生这些数字所需的数据与权重。
- 本仓库只含一个模型版本，不提供对照方法的实现；对照结果请在论文的独立汇总文档中查阅。
