<a id="four-phase-transfer-auto-curriculum"></a>
# Transfer-Auto 四阶段训练流程

[英文原文](canonical-recipe.md) · [中文目录](../../README.zh-CN.md)

> 本文为中文阅读版。代码块、配置键和训练数值保持原样。

从 **Cosmos3-Nano** 开始，使用自己的同步数据，依次进行分传感器训练、
短序列联合训练、长序列联合训练，以及长序列联合训练的延续阶段。

最初的分传感器阶段使用 30 FPS、最长 397 帧的单相机 RGB 片段，
以及独立的 10 Hz、21 个扫描帧的 LiDAR 片段。RGB 加载器请求 400 个源帧，
然后裁剪为 tokenizer 兼容的 `1 + 4k` 长度。这些独立任务不需要匹配时长。
整个 70k（70,000）次更新阶段均使用此设置；61 帧 RGB／21 个扫描帧
是后续短序列联合阶段使用的对齐窗口。

<a id="phase-to-config-mapping"></a>
## 阶段与配置的对应关系

四个阶段均使用同一份自有配对数据包。

| 阶段 | Framework 选择器 | 任务／时间跨度 | 迭代终点 |
| --- | --- | --- | --- |
| 1. 分传感器 | `transfer_auto_phase1` | 30% 单相机 WSM→RGB，最长 397 帧；70% HD-map→LiDAR，21 个扫描帧；使用独立样本 | 70k |
| 2. 短序列联合 | `transfer_auto_phase2` | 配对的多视角 RGB/LiDAR；61 帧 / 21 个扫描帧 | 新增 10k |
| 3. 长序列联合 | `transfer_auto_phase3` | 最长 397 帧 / 133 个扫描帧 | 联合训练计数 22k |
| 4. 长序列联合延续 | `transfer_auto_phase4` | 与阶段 3 相同的任务、注意力机制及 10% 整段 LiDAR dropout | 联合训练计数 50k |

所有联合阶段均采用已发布的去重注意力机制，跨视角窗口覆盖此前
0.4 秒并包含当前时刻。阶段 1 在单视角内使用双向注意力。
阶段 4 保留阶段 3 的训练状态并延续同一调度，不引入新的注意力策略。

<a id="configure-the-phases"></a>
## 配置各阶段

使用 [后训练指南](../README.zh-CN.md) 中的入口和启动器。
设置 `COSMOS_TRANSFER_AUTO_LOCAL_BUNDLE`、匹配的 RGB/LiDAR VAE 路径
以及新的输出标识。从不可变的 Cosmos3-Nano 快照开始：

```bash
python -m cosmos_framework.scripts.convert_model_to_dcp \
  --checkpoint-path "${NANO_HF:?set the Cosmos3-Nano snapshot}" \
  -o "${NANO_DCP:?set a new conversion directory}"
export COSMOS_TRANSFER_AUTO_NANO_DCP="$NANO_DCP"
```

这会创建仅含权重的初始化产物。阶段 1 加载兼容的已有权重，仅对新增的 LiDAR
投影层和传感器嵌入表进行新初始化，同时新建优化器、调度器和 EMA。

每个阶段均需在 dry run 和正式启动前设置 `settings`：

```bash
# Phase 1: retain 400 requested RGB frames (up to 397 usable) and 21 LiDAR sweeps.
settings=(experiment=transfer_auto_phase1 "job.name=$AUTO_RUN_NAME")

# Phase 2: load the actual phase-1 full checkpoint directly.
export COSMOS_TRANSFER_AUTO_PHASE1_CHECKPOINT="$SEPARATE_SENSOR_CHECKPOINT"
settings=(
  experiment=transfer_auto_phase2
  "job.name=$AUTO_RUN_NAME"
  "model.config.ema.iteration_shift=$SEPARATE_SENSOR_UPDATES"
)

# Phase 3: continue the joint counter, EMA and optimizer.
export COSMOS_TRANSFER_AUTO_PHASE2_CHECKPOINT="$SHORT_JOINT_CHECKPOINT"
settings=(
  experiment=transfer_auto_phase3
  "job.name=$AUTO_RUN_NAME"
  "model.config.ema.iteration_shift=$SEPARATE_SENSOR_UPDATES"
)

# Phase 4: continue from joint iteration 22000.
export COSMOS_TRANSFER_AUTO_PHASE3_CHECKPOINT="$LONG_JOINT_CHECKPOINT"
settings=(
  experiment=transfer_auto_phase4
  "job.name=$AUTO_RUN_NAME"
  "model.config.ema.iteration_shift=$SEPARATE_SENSOR_UPDATES"
)
```

在前一阶段完成后，分别执行相应代码段。为每个阶段指定不同的 `AUTO_RUN_NAME`。
`SEPARATE_SENSOR_UPDATES` 是从阶段 1 继承的实际 EMA 历史步数
（按上述预算以及配置工厂默认值为 70,000）。
缩短阶段 1 时，应在**所有**联合阶段中覆盖此偏移量。
在完整训练前，通过训练/保存/重新加载检查验证阶段衔接。

| 转换 | 恢复 | 重置 |
| --- | --- | --- |
| Cosmos3-Nano → 分传感器 | 兼容的普通模型权重 | 优化器、调度、计数器和 EMA；初始化新增传感器模块 |
| 分传感器 → 短序列联合 | 模型、EMA 和 Adam 的一阶／二阶矩状态 | 数据、调度器及训练器计数器；保留实际 EMA 历史步数 |
| 短序列 → 长序列联合 | 模型、EMA、Adam、调度及计数器 | 仅数据迭代器 |
| 阶段 3 → 阶段 4 | 同样的完整状态 | 仅数据迭代器 |
| 同阶段恢复 | 完整检查点，包括受支持的数据加载器状态 | 不执行阶段转换重置 |

<a id="scientific-settings"></a>
## 实验设置

| 设置 | 分传感器（阶段 1） | 短序列联合（阶段 2） | 长序列联合（阶段 3 和 4） |
| --- | --- | --- | --- |
| 分辨率采样占比 | RGB 15% 480p + 15% 720p；LiDAR 70% | 60% 480p / 40% 720p | 相同 |
| 描述文本（caption）策略 | RGB 使用完整正文；仅 LiDAR 时使用首段 | 每个相机的首段 | 每个相机的完整正文 |
| 打包上限 | 240k | 300k | 1.5M |
| RGB / LiDAR 损失缩放 | 1 / 1 | 1 / 2 | 1 / 2 |
| LiDAR 流式上下文 / 分块 | 21 / 21 | 21 / 21 | 21 / 20 |
| 样本级 / 逐视角 caption dropout | 0.1 / 0 | 0.1 / 0.1 | 0.1 / 0.1 |
| 整段 LiDAR dropout | 0 | 0 | 0.1 |

这些比例针对逻辑样本，而非相同数量的像素、token 或计算量。
保留各相机独立的 caption、物理 ID、时钟，并遵循 [数据约定](data-contract.zh-CN.md)。
对于完整的十一相机数据包，480p 下参考相机组的采样分布为
60% 七相机／30% 十一相机／10% 其他符合条件的子集；
720p 下使用 90% 七选六 / 10% 其他符合条件的子集。
相机数量较少时，会在符合条件的组合间重新归一化采样概率；这是数据适配的一部分。

没有独立的整段 RGB 模态随机丢弃（dropout）：相机由上述相机组／子集策略选择。
逐视角 caption dropout 删除描述正文，但保留相机头部文本；
样本级 CFG dropout 同时清空所有视角的 caption。任务系统提示词保留不变。
视角补全和控制 dropout 均禁用。

使用融合 AdamW，基础学习率为 `1e-4`，betas 为 `(0.9,0.99)`，
epsilon 为 `1e-8`，权重衰减为 `0.05`，梯度裁剪为 `1.0`。
训练生成器、时间／视频投影层、LiDAR 投影层及传感器嵌入表；
其余参数（包括参考文本部分的参数）保持冻结。模块选择为 `moe_gen`、`time_embedder`、
`vae2llm`、`llm2vae`、`lidar2llm`、`llm2lidar`、`rig_view_embed`。

调度器使用 **LambdaLinear**，不是余弦调度：

| 调度 | 起始学习率 | 峰值 | 下限 | 预热 | 周期 |
| --- | --- | --- | --- | --- | --- |
| 分传感器 | 1e-10 | 8e-5 | 1e-5 | 2000 | 70000 |
| 共用联合调度 | 1e-5 | 4e-5 | 1e-5 | 1000 | 50000 |

预热和衰减均为线性；周期包含预热。各联合阶段共用保存的调度器计数器。
最终的 50k 表示联合训练累计达到 50,000 次更新，不是额外再更新 50,000 次。
缩短冒烟测试只改变停止/保存间隔，不改变实验调度。

EMA 使用幂律曲线 `rate=0.1`，不是固定衰减 0.1。
对于速率 `s`，令 `g` 为方程
`g^3+7g^2+(16-s^-2)g+12-s^-2=0` 的最大实根。
当 `i=iteration+iteration_shift` 时，i<1 时 beta 为 0，
否则为 `(1-1/(i+1))^(g+1)`。加载 EMA 时保留实际历史步数；
新的直接微调任务重新建立 EMA 历史。进入长序列阶段时，不要再次叠加偏移量。

使用 BF16 计算/归约、FP32 主权重及选择性激活检查点
（联合阶段保存标记的运算）。参考上下文/FSDP 并行度为 32/32。
在 512 个 rank、梯度累积为 1 时，world-size / CP 得到 16 个数据副本；
FSDP 不是另一个除数。在适配后的硬件上验证每次更新实际打包的片段数。

<a id="objective-and-conditioning"></a>
## 目标函数与条件设置

使用整流流（rectified flow）的速度预测目标、`waver` 视频时间采样及均匀损失加权。
训练 shift 在 480p 下为 5，在 720p 下为 10；仅 LiDAR 时为 5。
Transfer 样本以 50/50 的比例混合文生视频（T2V）与图生视频（I2V）任务，控制信号始终存在。
联合 I2V 以一个初始相机潜变量（latent）和对齐的实测 LiDAR 前缀作为条件。
整段 LiDAR 条件和视角补全关闭；控制和前缀 dropout 为零。
整段 LiDAR dropout 同时移除目标与控制。

保持 `local_item_mean` 和 `normalize_loss_by_active=False`：
作为条件的元素误差为零，但仍计入均值分母。
正式采用适配后的训练配方前，检查实际 batch、掩码、损失和恢复后的状态。

使用 [参数指南](parameter-guide.zh-CN.md) 进行适配，
并参考 [后训练指南](../README.zh-CN.md#export-and-check-inference) 完成最终导出。
