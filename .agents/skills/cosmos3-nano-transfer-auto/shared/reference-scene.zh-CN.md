<a id="one-daytime-scene-for-both-auto-models"></a>

# 两种 Auto 模型共用的白天场景

[英文原文](reference-scene.md) · [中文目录](../README.zh-CN.md)

> 本文为中文阅读版；代码块及其注释保持原样。

Transfer-Auto 和 Sim-Auto 使用同一个七相机场景。
它包含真实 RGB／LiDAR 目标、共用的 WSM v4.0.1 控制及逐相机描述文本（caption）。
模型权重、VAE 和文本 tokenizer 需要另行准备。

场景及原始 `captions/` 文件来自已发布的
[NVIDIA PhysicalAI Autonomous Vehicles 数据集](https://huggingface.co/datasets/nvidia/PhysicalAI-Autonomous-Vehicles)。

<a id="get-the-scene"></a>

## 获取场景

从 Transfer-Auto **模型仓库**下载
[白天示例](https://huggingface.co/nvidia/Cosmos3-Nano-Transfer-Auto/tree/main/assets/sample_data)。
两个模型使用相同资产：

- 仓库：`nvidia/Cosmos3-Nano-Transfer-Auto`
- 仓库类型：`model`
- 子目录：`assets/sample_data`
- 示例版本：`main`

确保 Framework 环境中可以使用 Hugging Face CLI，
并在仓库外选择一个本地下载目录。
如果访问仓库需要授权，先接受模型访问条款并使用 `hf auth login` 认证。

```bash
export AUTO_SAMPLE_DOWNLOAD_DIR=/path/to/your/auto-example
export AUTO_SAMPLE_REVISION=main
hf download nvidia/Cosmos3-Nano-Transfer-Auto \
  --repo-type model --revision "$AUTO_SAMPLE_REVISION" \
  --include 'assets/sample_data/*' \
  --local-dir "$AUTO_SAMPLE_DOWNLOAD_DIR"
export AUTO_SAMPLE_ROOT="$AUTO_SAMPLE_DOWNLOAD_DIR/assets/sample_data"
(cd "$AUTO_SAMPLE_ROOT" && sha256sum -c SHA256SUMS)
```

保留相对路径结构不变。此命令只下载示例，不下载模型权重或 tokenizer。
上述版本选择针对**数据**；兼容的模型快照需按照
[环境与模型资产](framework-runtime.zh-CN.md) 单独获取。
校验和列表覆盖 36 个内容文件；目录中另有 `SHA256SUMS` 和 `FILES.tsv`。

<a id="files-and-meanings"></a>

## 文件及含义

```text
sample_data/
  README.md
  training-index.jsonl
  input-transfer-auto.jsonl
  input-sim-auto.jsonl
  SHA256SUMS
  FILES.tsv
  daytime_scene/
    rgb/<camera_key>.mp4       # seven real target videos
    wsm/<camera_key>.mp4       # seven shared control videos
    captions/<camera_key>.json # seven paragraph-style captions
    captions/negative_prompt.txt # per-view negative prompt
    captions_with_timestamp/
      <camera_key>.json        # seven timestamped alternatives
      README.md
    lidar/control.tar         # HD-map range controls
    lidar/target.tar          # recorded LiDAR targets
```

相机分别为前广角、右侧交叉视角、右后视角、后长焦、左后视角、
左侧交叉视角和前长焦，使用完整的
[物理名称与已学习 ID](../post-training/references/data-contract.zh-CN.md#camera-identities)。
文件不会重命名为 c0／c1。两份推理请求都通过 `camera_key` 明确映射各视角。

RGB 和 WSM 均为 300 帧、30 FPS、1280×720。
描述文本使用 `caption_structured.chunk_0_300`，帧范围为左闭右开的 `[0,300)`。
每个 LiDAR 归档含有 100 个扫描帧，其相机时钟帧 ID 为 0,3,...,297；
归档成员的 clip key 为 `daytime_scene`。
这些指南将控制格式称为 **WSM v4.0.1**；渲染器选项仍是 `version=v4`。

示例提供的是已经渲染好的控制数据，不包含原始标定和独立时间戳文件（sidecar），
也不包含可用于重新渲染的完整原始场景。
准备自己的数据时，应保留这些源记录，以便验证标定和同步。

<a id="choose-a-caption-style"></a>

## 选择描述文本风格

示例为同一场景提供了两种 caption：

- `captions/`：每个相机一段描述性文本，模型同样支持这种形式。
- `captions_with_timestamp/`：带时间戳的描述，说明各相机画面如何随时间变化。
  这与原文所述训练数据的格式一致；原文报告这种格式的生成效果更好。
  后训练优先使用这种格式，以减小自有数据与模型训练分布之间的 caption 格式差异。

两种版本使用相同 JSON schema 和相机文件名。
带时间戳版本在单个 `caption_structured.chunk_0_300.caption` 字符串中包含多个段落，
外层帧范围仍为 `[0,300)`。
`[0.0s-3.2s]:` 等文本区间使用共享的片段局部时钟；
各视角的段落边界可以不同。这些时间戳本身不会创建独立的数据加载块或生成块。

随附请求和训练索引仍选择 `captions/`。
若要使用带时间戳版本，应在**原文件旁边**复制相关 JSONL，
并将全部七个 caption 路径中的 `daytime_scene/captions/`
改为 `daytime_scene/captions_with_timestamp/`：

- **推理：** 更新每个 `multiview.views[].caption_path`，
  并将 `AUTO_REQUEST` 指向副本。
  在 `multiview` 下保留 `load_caption_from_data=true`
  和 `first_caption_paragraph_only=false`，以保留全部段落。
- **训练：** 更新 `assets.caption_<camera_key>` 路径，
  将 `AUTO_INDEX` 指向副本，并构建新数据包。
  已经打包的数据包不会随之改变。
  保留选定配方采用的完整 caption 或开头段落策略。

媒体、camera key 及其他设置保持不变。
按照 [推理检查](../inference/references/input-preparation.zh-CN.md#4-delegate-caption-authoring)
或 [基于真实目标的训练指南](../post-training/references/data-contract.zh-CN.md#training-captions)，
联合检查各视角的物体外观、物理运动及事件时间是否一致。

<a id="run-inference"></a>

## 运行推理

遵循对应的推理指南。使用原始段落 caption 时，绑定以下随附请求；
如果已经选择带时间戳版本，则继续让 `AUTO_REQUEST` 指向修改后的副本。

```bash
# Transfer-Auto:
export AUTO_REQUEST="${AUTO_SAMPLE_ROOT:?set the local scene directory}/input-transfer-auto.jsonl"
# Or Sim-Auto:
export AUTO_REQUEST="${AUTO_SAMPLE_ROOT:?set the local scene directory}/input-sim-auto.jsonl"
```

[Transfer-Auto 推理](../inference/README.zh-CN.md#run)
从请求读取 guidance 3 和 shift 10，并通过 CLI 设置 720p、35 步
和随附的逐视角负向提示词，与原生 notebook 一致。
下载的 JSONL 保持不变。
[Sim-Auto 推理](../../cosmos3-nano-sim-auto/inference/SKILL.md)
使用 guidance 1 和学生模型导出的四步采样计划。
两个请求均选择七个相机、首帧 RGB 条件及 HD-map LiDAR 控制；
都不提供初始实测 LiDAR 扫描帧。

请求使用一个覆盖全部 300 个源帧的 caption 区间。
`use_first_chunk_only: true` 选择整个区间。
预期输出是 30 FPS 的 297 个相机帧和 10 Hz 的 99 个 LiDAR 扫描帧。
不要在 JSONL 中添加 `joint_generation`：
Framework 根据相机／WSM 和 LiDAR 输入选择联合输出。
路径相对于 JSONL 文件解析。

<a id="prepare-the-training-bundle"></a>

## 准备训练数据包

原始段落 caption 使用下面的随附索引。
带时间戳 caption 则继续让 `AUTO_INDEX` 指向上述修改后的副本。

```bash
export AUTO_INDEX="${AUTO_SAMPLE_ROOT:?set the local scene directory}/training-index.jsonl"
```

使用七个 camera key 和匹配的文本 tokenizer，运行
[Framework 原生数据包构建工具](../post-training/README.zh-CN.md#prepare-a-local-training-bundle)。
它会在新目录中生成 `manifest.json`、`samples.json`、
`assets/` 和 `resources/tokenizer/`。
两个模型的配置均使用这一数据包格式。
这里不使用示例专属的数据集类，也不需要新增适配器。

这些文件各有不同用途：

| 文件 | 读取方 | 用途 |
| --- | --- | --- |
| `input-*-auto.jsonl` | 原生推理入口 | 控制、caption、初始 RGB 及采样设置 |
| `training-index.jsonl` | 原生 `auto_bundle` 构建工具 | 将现有真实目标与控制配对 |
| 生成的 `manifest.json`／`samples.json` | 原生训练加载器 | 打包后的成员、tokenizer 及变换设置 |

<a id="short-training-check"></a>

## 短训练验证

本示例包含 **720p 视频和 300 个源帧**。
以下短测试使用 21 个对齐的 RGB 帧，以及原生相机组采样器选出的相机子集，
检查数据加载、参数更新及检查点保存；
模型质量应在自己的数据集上完成完整训练后评估。
完整分阶段训练需要两种分辨率数据分层，
以及足以覆盖 400 个源帧窗口的采集数据。

在一台配备四张 H200 的节点上进行小规模 **Transfer-Auto 直接微调**验证时，
从 CP=2、FSDP=2 开始。CP 分配序列计算，FSDP 分片模型和优化器状态。
CP=4／FSDP=1 会使这些状态不分片，并不是等价的省显存配置。
这些并行度描述的是相互重叠的设备网格（mesh），
不是乘积必须始终等于 GPU 数量的两个维度。

构建微调 `settings` 数组后，追加：

```bash
sample_stream=dataloader_train.dataloaders.joint_720
settings+=(
  '~dataloader_train.dataloaders.joint_480'
  "${sample_stream}.ratio=1.0"
  "${sample_stream}.dataloader.dataset.transform_overrides.num_video_frames=24"
  "${sample_stream}.dataloader.dataset.transform_overrides.joint_aligned_num_video_frames=21"
  trainer.max_iter=2
  checkpoint.save_iter=2
  checkpoint.dcp_async_mode_enabled=false
  model_parallel.context_parallel_size=2
  model.config.parallelism.context_parallel_shard_degree=2
  model.config.parallelism.data_parallel_shard_degree=2
)
python -m cosmos_framework.scripts.train_experiment \
  "${entry[@]}" --dryrun -- "${settings[@]}"
torchrun --standalone --nproc-per-node=4 \
  -m cosmos_framework.scripts.train_experiment \
  "${entry[@]}" -- "${settings[@]}"
```

这是使用数据包相机子集采样器进行的短 720p 检查，
不保证完整长度或固定七相机工作负载能够运行。
应在实际资源上验证 GPU 显存峰值。
这里采用同步保存；启用异步保存前，先参阅
[检查点内存](framework-runtime.zh-CN.md#checkpoint-memory)。
只有 loss 为有限值、检查点已完整保存、且同一任务成功重新加载后，
才能将检查视为通过。

对于 **Sim-Auto Stage 1 TF**，先应用
[数据包相机映射覆盖设置](../../cosmos3-nano-sim-auto/shared/data-contract.md#preserve-your-bundles-camera-mappings)。
在 [阶段启动命令](../../cosmos3-nano-sim-auto/post-training/references/runtime.md)
中的该数组**之后**，追加以下示例覆盖设置：

```bash
sample_stream=dataloader_train.dataloaders.cam_lidar_joint_720p_separate_view_text
sample_overrides=(
  '~dataloader_train.dataloaders.cam_lidar_joint_480p_separate_view_text'
  "${sample_stream}.ratio=1.0"
  "${sample_stream}.dataloader.dataset.transform_overrides.num_video_frames=24"
  "${sample_stream}.dataloader.dataset.transform_overrides.joint_aligned_num_video_frames=21"
  trainer.max_iter=2
  checkpoint.save_iter=2
  checkpoint.dcp_async_mode_enabled=false
)
settings+=("${sample_overrides[@]}")
```

在构建 `settings` 后、第一次 dry run 前应用这些设置。
这些覆盖项仅用于缩小本次测试规模，不应据此改写已发布的训练配方。
Sim-Auto 的资源分配及全部三个阶段，应遵循
[受限规模训练／保存／恢复指南](../../cosmos3-nano-sim-auto/post-training/references/smoke.md)。
不要覆盖原始完整运行配置。
仅完成两次更新的冒烟测试，不能证明模型收敛、恢复流程正确或生成质量达标。
完整训练流程需要额外的更长采集数据，并将验证行驶记录与训练行驶记录分开。

<a id="use-your-own-data-and-verify-results"></a>

## 使用自己的数据并验证结果

机密源文件应保存在仓库之外。
按照 [训练数据约定](../post-training/references/data-contract.zh-CN.md)
生成等价的配对资产和索引行；
不要用生成输出伪造真实目标，也不要为目标标注虚构天气。

1. 下载后验证提供的校验和。
2. 检查同步 RGB／WSM 叠加、物理相机映射及 caption 时间范围。
3. 检查解码后的训练数据：控制／目标角色、相机 ID、时间选择、
   距离单位、强度及有效性。
4. 运行两个模型，逐一检查各相机的生成视频及 LiDAR 数值输出。
   记录模型版本、示例版本及已解析请求。

单个场景只用于说明接口；训练应使用更大且多样化的数据集，
并用独立留出的采集数据评估泛化。
