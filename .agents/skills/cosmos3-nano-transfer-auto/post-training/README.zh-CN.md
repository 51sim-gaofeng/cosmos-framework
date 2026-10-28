<a id="post-train-cosmos3-nano-transfer-auto"></a>
# Cosmos3-Nano-Transfer-Auto 后训练指南

[英文原文](SKILL.md) · [中文目录](../README.zh-CN.md)

> 本文为中文阅读版，不是独立注册的 agent skill。
> 命令、参数名和环境变量保持原样；翻译范围内的链接指向中文版，范围外的链接保留原文。
> 译文不代表已在当前机器上验证训练流程。
>
> 原文元数据：
> - 名称：`cosmos3-nano-transfer-auto-post-training`
> - 描述：在 Cosmos Framework 中，使用配对驾驶数据微调已发布的
>   Cosmos3-Nano-Transfer-Auto 模型，或从 Cosmos3-Nano 开始执行四阶段训练流程。

使用自己的配对采集数据微调已发布的 Transfer-Auto 模型，使其根据相机 WSM
和 HD-map 控制信号生成与之匹配的相机视频和 LiDAR 数据。
快速入门从已发布权重开始；如果要从 Cosmos3-Nano 开始训练，
请按照下文的四阶段训练流程进行。

本指南使用 **Cosmos Framework**。你需要提供自己的采集数据和控制数据；
不需要访问模型最初使用的训练数据集。

<a id="choose-your-starting-point"></a>
## 选择起点

| 目标 | 起始模型 | 对应指南 |
| --- | --- | --- |
| 适配自己的数据 | 已发布的 Transfer-Auto 检查点 | [微调快速入门](#quickstart-fine-tune-the-released-model) |
| 复现各训练阶段 | Cosmos3-Nano 检查点 | [四阶段训练流程](#reproduce-the-four-training-phases) |
| 生成驾驶场景 | 已发布的 Transfer-Auto 检查点 | [推理指南](../inference/README.zh-CN.md) |
| 构建流式模拟器 | 自己的 Transfer-Auto 检查点 | [Sim-Auto 蒸馏](../../cosmos3-nano-sim-auto/post-training/SKILL.md) |

<a id="set-up-the-environment"></a>
## 配置环境

按照 [Framework 安装指南](../../../../docs/setup.md)，在选定的仓库副本中完成安装，
再按 [Auto 运行要求](../shared/framework-runtime.zh-CN.md) 补齐相关配置。
复用已经能够正常工作的 Framework 环境。

以下命令使用本仓库中的原生
[四阶段训练配方](../../../../cosmos_framework/configs/simulation/common/transfer_auto_recipes.py)
和 [实验注册](../../../../cosmos_framework/configs/simulation/experiment/auto/transfer_auto.py)：
`transfer_auto_phase1` 至 `transfer_auto_phase4`。

<a id="finetuning-quickstart"></a>

<a id="quickstart-fine-tune-the-released-model"></a>
## 快速入门：微调已发布模型

<a id="activate-the-framework-environment"></a>
### 激活 Framework 环境

在 Framework 仓库根目录中，使用已经准备好的环境运行。
四阶段训练流程也使用同一个入口：

```bash
cd "${FRAMEWORK_DIR:?set the absolute Framework checkout root}"
source .venv/bin/activate
export COSMOS_INTERNAL=0 COSMOS_TRAINING=1
entry=(
  --config cosmos_framework/configs/simulation/config.py
  --experiment-module cosmos_framework.configs.simulation.experiment.auto.transfer_auto
)
```

阅读 [数据约定](references/data-contract.zh-CN.md)，了解各类传感器数据的含义；
更改训练设置前，先阅读 [参数指南](references/parameter-guide.zh-CN.md)。
用户资产应存放在仓库之外。
[参考场景](../shared/reference-scene.zh-CN.md) 提供 Hugging Face 下载方式、
`daytime_scene` 资产和共用的原生训练索引。以它作为格式参考，
再准备具有足够多样性的自有采集数据集。

<a id="prepare-the-released-model"></a>
### 准备已发布模型

将 `TRANSFER_AUTO_HF` 设置为本地模型快照目录，
将 `TRANSFER_AUTO_REVISION` 设置为选定的不可变版本标识，
将 `TRANSFER_AUTO_DCP` 设置为新的转换输出目录。
根据 [模型资产指南](../shared/framework-runtime.zh-CN.md#model-snapshots)，
获取匹配的 RGB VAE 和文本 tokenizer。
将 `VIDEO_VAE` 设置为匹配的 Wan2.2 VAE 文件路径，
将 `LOCAL_QWEN_TOKENIZER` 设置为匹配的本地文本 tokenizer 目录，
供数据准备使用。

```bash
hf download nvidia/Cosmos3-Nano-Transfer-Auto \
  --revision "${TRANSFER_AUTO_REVISION:?set the chosen snapshot revision}" \
  --local-dir "${TRANSFER_AUTO_HF:?set a local snapshot directory}"
python -m cosmos_framework.scripts.prepare_sim_auto_checkpoint \
  --checkpoint-path "$TRANSFER_AUTO_HF" \
  --output-path "${TRANSFER_AUTO_DCP:?set a new weights-only DCP directory}"
export COSMOS_TRANSFER_AUTO_VIDEO_VAE="${VIDEO_VAE:?set the matched RGB VAE file}"
export COSMOS_TRANSFER_AUTO_LIDAR_VAE="$TRANSFER_AUTO_HF/lidar_vae/diffusion_pytorch_model.safetensors"
```

虽然这个准备工具的名称包含 `sim_auto`，但这里它转换的是双向 Auto 模型权重，
并检查已经训练好的 LiDAR 头和传感器表。**它不会执行模型蒸馏。**

<a id="prepare-a-local-training-bundle"></a>
### 准备本地训练数据包

描述文本（caption）文件只保存场景描述，不要手动添加模型上下文前缀。
原生训练流程会在格式化和分词时，自动添加相机组（rig）／当前视角的朝向与 FOV
描述，以及任务对应的控制指令。两者都不需要手动添加，详见
[自动前缀](../shared/caption-policy.zh-CN.md#caption-text-and-automatic-prefixes)。

1. 准备时间同步的 RGB 和 WSM、基于真实目标内容的逐相机 caption、
   实测 LiDAR，以及 HD-map 距离图控制数据。优先使用
   [带时间戳的 caption](references/data-contract.zh-CN.md#training-captions)，
   以匹配训练格式；段落式 caption 仍受支持。
2. 根据 [输入准备指南](../inference/references/input-preparation.zh-CN.md)
   检查相机身份和数据对齐。不要把推理时虚构的目标外观描述作为训练标签。
3. 创建 [JSONL 训练索引](references/data-contract.zh-CN.md#local-training-index)，
   或使用参考场景中的 `$AUTO_SAMPLE_ROOT/training-index.jsonl`。
   该索引默认选择段落式 caption；若要使用带时间戳的版本，
   请在打包前按照 [caption 选择说明](../shared/reference-scene.zh-CN.md#choose-a-caption-style)
   进行切换。设置 `AUTO_INDEX`、`LOCAL_QWEN_TOKENIZER`，
   并指定一个新的 `AUTO_BUNDLE` 目录。
4. 将选定的物理相机数据打包。以下列表与七相机示例一致：

```bash
python -m cosmos_framework.data.generator.local_simulation.auto_bundle \
  --index "${AUTO_INDEX:?set your training index}" \
  --tokenizer "${LOCAL_QWEN_TOKENIZER:?set the local text tokenizer}" \
  --output "${AUTO_BUNDLE:?set a new bundle directory}" \
  --camera-keys \
    camera_front_wide_120fov camera_cross_right_120fov \
    camera_rear_right_70fov camera_rear_tele_30fov \
    camera_rear_left_70fov camera_cross_left_120fov camera_front_tele_30fov
export COSMOS_TRANSFER_AUTO_LOCAL_BUNDLE="$AUTO_BUNDLE"
```

构建工具会复制媒体文件和 tokenizer 资源，并根据实际提供的相机确定可用子集；
相比十一相机数据集，这会改变训练中各相机组合的采样分布。
需要提供训练配方要求的两类分辨率数据，或者在构建数据加载器前，
移除缺失的数据流。对于这个 720p／300 帧示例，请使用
[限制运行规模的示例设置](../shared/reference-scene.zh-CN.md#short-training-check)。
将数据源用于正式训练前，应检查解码后的样本和格式化后的实际训练批次（batch）。

<a id="configure-the-fine-tuning-run"></a>
### 配置微调任务

选择唯一的 `AUTO_RUN_NAME`，并导出共用的 `IMAGINAIRE_OUTPUT_ROOT`。
使用最终模型训练配方，以仅加载权重的方式初始化新任务，并解析配置：

```bash
settings=(
  experiment=transfer_auto_phase4
  "job.name=${AUTO_RUN_NAME:?set a fresh run name}"
  "checkpoint.load_path=$TRANSFER_AUTO_DCP"
  checkpoint.load_training_state=False
  'checkpoint.keys_not_to_resume=[]'
  'checkpoint.keys_to_skip_loading=[net_ema.]'
  model.config.ema.iteration_shift=0
)
python -m cosmos_framework.scripts.train_experiment \
  "${entry[@]}" --dryrun -- "${settings[@]}"
```

此配置默认进行 50,000 步联合训练。调整学习率、调度策略、
dropout 和训练时长时，请参阅 [参数指南](references/parameter-guide.zh-CN.md)。
这会从模型权重启动一个新任务，**不是恢复原始训练任务**。
普通模型权重以严格模式加载；新任务重新建立移动平均权重（EMA）及其历史状态。

<a id="launch-and-monitor"></a>
### 启动与监控

无论采用哪条训练路径，都需要在选定的 `settings` 后追加与实际资源分配匹配的
并行拓扑参数。根据分配的资源设置 `NNODES`、`GPUS_PER_NODE`、`NODE_RANK`、
`MASTER_ADDR`、`CONTEXT_PARALLEL` 和 `FSDP_SHARDS`：

```bash
torchrun --nnodes="$NNODES" --nproc-per-node="$GPUS_PER_NODE" \
  --node-rank="$NODE_RANK" --rdzv-backend=c10d \
  --rdzv-endpoint="$MASTER_ADDR:29500" \
  -m cosmos_framework.scripts.train_experiment "${entry[@]}" -- \
  "${settings[@]}" \
  model_parallel.context_parallel_size="$CONTEXT_PARALLEL" \
  model.config.parallelism.context_parallel_shard_degree="$CONTEXT_PARALLEL" \
  model.config.parallelism.data_parallel_shard_degree="$FSDP_SHARDS"
```

启动前，使用相同的拓扑覆盖参数再次解析配置。
参考 CP／FSDP 并行度为 **32／32**；较小的资源配置需要验证显存和 batch 设置，
而不是仅仅减少启动进程数。
先单独运行一次小规模冒烟测试，再验证训练、保存和重新加载流程。
对于四张 H200 上的参考场景，请从明确给出的
[短测试拓扑和同步保存设置](../shared/reference-scene.zh-CN.md#short-training-check) 开始。
启用异步保存前，先阅读 [检查点内存要求](../shared/framework-runtime.zh-CN.md#checkpoint-memory)。

检查实际采样的任务与分辨率分布、RGB／LiDAR 损失（loss）是否为有限值、
学习率、实际更新的参数，以及检查点是否完整。
缩短 `trainer.max_iter` 并不要求同时缩短学习率调度器的周期。

在同一阶段恢复训练时，应恢复完整的模型／EMA、优化器、调度器、迭代计数，
以及受支持的数据加载器状态。复用该任务的输出标识，
不要再次应用用于新任务热启动的状态重置设置。
CLI 中指定的停止迭代数是**累计迭代终点**，不是本次还要执行的更新次数。

<a id="export-and-check-inference"></a>
## 导出并检查推理

使用**训练时保存的已解析配置**，选择要导出的权重版本，
并配合匹配的 tokenizer 进行导出：

```bash
python -m cosmos_framework.scripts.export_model \
  --checkpoint-path "${AUTO_CHECKPOINT:?set a completed iteration root}" \
  --config-file "${AUTO_CONFIG:?set the saved run config.yaml}" \
  --use-ema-weights --no-vit \
  --student-only-checkpoint-metadata --bundle-lidar-vae \
  --per-view-captions on \
  --per-view-caption-task av_joint_camera_lidar_transfer \
  -o "${AUTO_EXPORT:?set a new export directory}"
```

这里导出的是联合训练的最终模型，不是分传感器训练的中间模型。
`--no-vit` 表示不包含 ViT，导出的模型产物仅用于生成任务。
保持 tokenizer 资产完整；分享前，移除导出元数据中与具体机器相关的路径。
使用 [推理指南](../inference/README.zh-CN.md)，在固定的留出场景上进行检查。

<a id="four-phase-curriculum"></a>

<a id="reproduce-the-four-training-phases"></a>
## 复现四个训练阶段

按照 [训练流程与阶段衔接说明](references/canonical-recipe.zh-CN.md) 执行：

1. **Cosmos3-Nano → RGB／LiDAR 分传感器训练：**
   进行 70k（70,000）次更新，使用最长 397 帧的单相机 RGB 片段
   （请求帧数为 400），以及独立的、包含 21 个扫描帧的 LiDAR 片段。
2. **分传感器检查点 → 短序列 RGB／LiDAR 联合训练：**
   61 帧 RGB 与 21 个 LiDAR 扫描帧对齐。
3. **短序列联合检查点 → 长序列 RGB／LiDAR 联合训练：**
   最长 397 帧 RGB 与 133 个 LiDAR 扫描帧对齐，
   继续训练至联合训练迭代数 22,000。
4. **继续长序列联合训练：**
   保持相同的模型、注意力机制、dropout 和调度策略，
   继续至联合训练迭代数 50,000。

分传感器任务使用不同的时长，因为它们是彼此独立的样本。
只有联合训练阶段才要求 RGB／LiDAR 时间窗口匹配。
阶段 2–4 在包含当前时刻的前 0.4 秒时间窗口内使用去重的跨视角注意力。
阶段 3 和阶段 4 均以 10% 的概率执行整段 LiDAR 随机丢弃（dropout）。

<a id="references"></a>
## 参考资料

- [Framework 环境与模型资产](../shared/framework-runtime.zh-CN.md)
- [训练数据约定与数据包布局](references/data-contract.zh-CN.md)
- [参数指南](references/parameter-guide.zh-CN.md)
- [共用 Auto 参考场景](../shared/reference-scene.zh-CN.md)
- [相机与 WSM 输入检查](../inference/references/input-preparation.zh-CN.md)
- [四阶段训练配方与检查点衔接](references/canonical-recipe.zh-CN.md)
