# GeoTopoNet-V5

火山岩显微 CT 裂缝分割的**唯一正式模型**：ResNet34 U 形骨架 + 地质方位先验 + 双极性脊响应 + 多尺度细线融合 + 可微 Darcy 灵敏度门控，训练时叠加全程可微等效导纳比一致性损失。

> The single released model for physics-constrained fracture segmentation in volcanic-rock micro-CT. One architecture, one frozen configuration, no variant registry.

本仓库是从完整论文项目中**单独抽出的可复现代码**。只保留 V5 一个版本：没有其他代际变体、没有逐模块消融开关、没有对照骨架、没有多版本 `build(name=...)` 注册表。论文里的对照方法与消融结果不在本仓库内。

---

## 1. 唯一交付的模型

| 项目 | 值 |
|---|---|
| 模型类 | `geotoponet.GeoTopoNetV5` |
| 输出 | 1 通道裂缝前景 logits |
| 参数量 | **29,702,947**（实算，相对纯 ResUNet34 增加 467,042，+1.60%） |
| 输入 | 3 通道 ImageNet 归一化灰度（任意尺寸 pad 到 32 倍数） |
| 正式配置 | `configs/v5_loco.json`（留一岩心三折，冻结） |
| 评测 | Dice / clDice / E_log 三级指标 |

结构顺序：`PolarityRidgeStem → ResNet34 编码器 → 瓶颈 → GOP 方位先验 → U 形解码 → MSF 多尺度融合 → MultiGridDarcy 门控 → 1×1 输出头`。全部新增分支在加载源域权重后零初始化，因此起点与纯 ResUNet34 严格一致，所有增益可归因到物理约束而不是初始化差异。详见 [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)。

## 2. 安装

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
pip install -r requirements-dev.txt                     # 仅跑测试时需要
```

Python 3.9+，需要 torch ≥ 2.3 / torchvision ≥ 0.18。首次构造模型会下载 ImageNet 预训练 ResNet34 权重；离线自检请加 `--no-pretrained`（不下载，但**不是**论文协议）。

## 3. 快速开始

```bash
# 无需任何数据/权重的安装自检（CPU，约 1 分钟）
python scripts/check_model.py
python scripts/verify_metrics.py
python -m geotoponet.train --smoke

# 完整训练：留出 CS1，用另外三块岩心的 train 段 + 240 张合成图
python -m geotoponet.train --config configs/v5_loco.json \
  --synth-root data/synthetic_volc --index data/index.json \
  --fold cs1_micro --seed 0 --source-ckpt /path/to/source_resunet34.pt \
  --out runs/cs1_s0

# 留出折一次性评测（阈值由调用方显式给出，脚本不扫描）
python -m geotoponet.evaluate --config configs/v5_loco.json \
  --index data/index.json --fold cs1_micro \
  --weights runs/cs1_s0/weights.pt --threshold 0.55 --out eval_cs1.json

# 单张切片推理
python -m geotoponet.infer --image slice.tif \
  --weights runs/cs1_s0/weights.pt --out prob.png
```

`data/index.json` 格式见 [docs/TRAINING.md](docs/TRAINING.md)；合成集目录格式（`train/images/*.png` + `train/masks/*.png`）与同项目的 3D+2D 生成器仓库一致。

## 4. 正式结果（冻结，不在本仓库重算）

留一岩心三折，种子筛查与阈值定标**只用训练岩心的验证段**，留出岩心标签只做一次性评测：

| 留出岩心 | 切片数 | 冻结配置 | Dice | clDice | E_log |
|---|---:|---|---:|---:|---:|
| CS1 | 724 | mean @ 0.55 | **0.6582** | 0.5869 | 0.0146 |
| CS105 | 800 | s0 @ 0.70 | **0.6944** | 0.6830 | 0.0435 |
| DS9 | 800 | s0 @ 0.90 | **0.4153** | 0.3673 | 0.2420 |
| **三折均值** | **2324** | — | **0.5893** | **0.5457** | **0.1001** |

对照：单岩心六种子基线 Dice 0.3210 / clDice 0.2815 / E_log 0.3097。W21 只作开发、选择兼训练样品，**不作为第四折独立岩心写入正式结论**。DS9 低分归因为 TWS 参考标注的语义天花板，六组零接触干预全部为负结果；容差匹配、事后最优阈值与 top-q 召回都不是模型成绩。详见 [docs/REPRODUCE.md](docs/REPRODUCE.md)。

## 5. 像素—骨架—导流三级指标

| 级别 | 指标 | 管什么 |
|---|---|---|
| 像素 | Dice / 前景率 / 精度 / 召回 | 裂缝像素画得准不准 |
| 骨架 | clDice | 线状连通结构连得对不对 |
| 导流 | E_log | 相对等效导纳算得对不对（越低越好） |

`E_log = mean_axis |log(Q_pred/Q_ref)|`，掩膜粗化到 100×100 网格、κ=1000、两方向五点有限体积稳态单相 Darcy 直接解。`python scripts/verify_metrics.py` 用构造掩膜实算：

- 等量删/补 8 个像素（前景率不变）：Dice 仅从 1.0000 变到 0.9911；
- 只切断主缝 24 px：`E_log = 0.7950`，随机删同样 24 px：`E_log = 0.0216`，相差 **36.9 倍**，而两者 Dice 完全相同（0.9865）；
- 开度缩放 s=1.7 时 Darcy 流量比 4.913 = s³，相对误差 0。

定义与边界见 [docs/METRICS.md](docs/METRICS.md)，受控实验细节见 `scripts/verify_metrics.py`。

## 6. 复现性与自检（已实测）

| 检查 | 结果 |
|---|---|
| 参数量 | 29,702,947，与论文正式稿一致 |
| 前向 / 损失 / 反传 | `(1,1,128,128)` logits、`(1,1,32,32)` 灵敏度、loss 有限、梯度有限 |
| 零初始化 | ridge / GOP / MSF / AGA 四个新增分支恒等误差均为 0 |
| 确定性 | 同 seed 两次实例化，eval 前向逐位一致（差 0） |
| 单版本门禁 | `geotoponet/*.py` 中除 V5 外无任何代际版本标记 |
| 全程导纳 | 微调阶段损失含导纳项，预训练阶段不含，机器可判定 |
| 单元测试 | `pytest -q` 31 项通过 |

```bash
python -m pytest -q
python scripts/check_model.py
python scripts/verify_metrics.py
```

## 7. 边界（不要误读）

- `E_log` 是**相对等效导纳**的对数误差，不是绝对渗透率，也不是岩心驱替实测；基质导纳归一为 1。
- 真实域标签是 Fiji/ImageJ TWS 软件导出的**参考分割，不是人工金标准**；所有精度数字都是「对 TWS 参考分割的复现程度」。
- 本仓库**不发布**真实 CT 切片、TWS 标注、合成数据集、源域预训练权重与论文权重，也不随代码重算正式三折数字。仅靠本仓库无法比特级复现论文成绩，只能复现模型、损失、训练协议与评测口径。
- `train.py` / `evaluate.py` **刻意不做**阈值扫描、种子池筛选、早停或模型比较；配置里的冻结阈值与种子只作审计记录，不会被自动套用。留出折标签只被读取一次用于打分。
- 本仓库只含 V5 一个版本；对照方法（其他骨架、其他预训练源）属于论文里的独立实验，不在这里实现，绝对数值也不可与第 4 节相减。

## 8. 目录

```
geotoponet/
  model.py       唯一模型类 GeoTopoNetV5 及其全部内部模块
  flow.py        训练端可微等效导纳比算子
  losses.py      冻结的正式损失（BCE+Dice+拓扑+全程导纳）
  metrics.py     像素/骨架/导流三级评测（Darcy 直接解）
  data.py        灰度归一化、增强、批构造、切片索引
  train.py       合成预训练 + 留一岩心微调
  infer.py       单张切片推理
  evaluate.py    整栈一次性评测
configs/         唯一正式配置
scripts/         模型自检、三级指标受控实验
tests/           单元测试
docs/            结构、训练协议、指标、复现说明
```

## 9. 发布到 GitHub

仓库已初始化（`main` 分支）且源码已暂存；本机 Git 未配置提交身份，因此没有代做提交：

```bash
git config user.name "你的名字"
git config user.email "你的 GitHub 邮箱"
git commit -m "Initial release: GeoTopoNet-V5 (single model)"
git remote add origin https://github.com/<用户名>/GeoTopoNet-V5.git
git push -u origin main
```

推送前可再跑一次 `python -m pytest -q`。权重、数据、训练产物已在 `.gitignore` 中，不会被提交。

## 10. 许可

本仓库自有代码采用 [MIT](LICENSE)。第三方依赖与外部资产的归属、获取方式和限制见 [NOTICE.md](NOTICE.md)。
