[英文原文](SKILL.md) · [中文目录](../README.zh-CN.md)

> 本文为中文阅读版，不是独立注册的 agent skill。命令、配置键和代码示例保持原样。

> 原文元数据：名称为 `cosmos3-nano-transfer-auto-inference`。用途：在 Cosmos Framework 中，使用相机 WSM 控制和各视角描述文本准备或运行 Cosmos3-Nano-Transfer-Auto，可选初始 RGB 和联合 LiDAR。适用于相机映射、WSM／渲染器与同步检查、已有描述文本检查、`av-per-view-caption` 生成或扩写、面向特定 KPI 的描述文本、原生请求、采样设置及输出检查。仅检查输入或仅处理描述文本的请求，在完成所请求的工作流后即停止。

<a id="run-inference-with-cosmos3-nano-transfer-auto"></a>
# 使用 Cosmos3-Nano-Transfer-Auto 运行推理

根据每台相机的 WSM 控制和描述文本（caption），为各个所选相机生成视频。
添加 HD-map 距离图控制后，可联合生成 RGB／LiDAR。提供初始 RGB 图像时采用图生视频模式；
省略初始图像时采用文生视频模式，但仍以 WSM 作为布局与运动条件。

已发布模型针对参考相机组进行了优化。可以尝试自定义相机组，但随着相机位置、
观察方向、FOV 和镜头投影与参考配置的差异增大，应预期生成质量下降。
请先选择[使用参考相机组视点，还是保留自己的相机组](references/input-preparation.zh-CN.md#choose-reference-rig-or-custom-rig-generation)；
后者可能需要[后训练](../post-training/README.zh-CN.md#quickstart-fine-tune-the-released-model)。

<a id="start-here"></a>
## 从这里开始

1. **先尝试参考场景。** 使用完整的参考请求和已发布模型建立基线，然后再替换为自己的输入。
2. **选择并匹配相机。** 遵循[相机组工作流](references/input-preparation.zh-CN.md#choose-reference-rig-or-custom-rig-generation)，
   然后使用 [Camera Rig Explorer](../../cookbooks/cosmos3/simulation/auto/wsm/docs/camera-rig/guide.md) 和
   [映射指南](references/input-preparation.zh-CN.md#2-map-physical-cameras-to-learned-identities)，
   为自己的相机组选择最接近的受支持身份。每台相机的 RGB、WSM 和描述文本必须保持对应；
   `camera_key` 用于选择其学习到的视角嵌入。
3. **为每个相机视角准备描述文本。** 按照[与训练一致的描述风格](../shared/caption-policy.zh-CN.md#keep-training-and-inference-captions-aligned)
   描述所请求的场景，然后检查各视角之间的同步与一致性。
4. **生成、检查，再调优。** 参考方案使用 **720p**、guidance **3**、shift **10**、
   **35 步**，以及随附的[负向提示词](assets/negative-prompt.txt)。
   修改之前先阅读[这些设置的作用](references/input-and-execution.zh-CN.md#sampling)；
   检查每台相机，而不仅仅是前视相机。

<a id="choose-the-task"></a>
## 选择任务

- **检查输入或已有描述文本：** 阅读[输入准备](references/input-preparation.zh-CN.md)，仅完成所请求的检查。
  检查已有描述文本时，不调用编写流程，也不重写原文。
- **生成或扩写描述文本（`av-per-view-caption`）：** 阅读[描述文本指南](references/caption-guide.zh-CN.md)，
  包括其中的描述文本策略。遵循其选项、默认值和 WSM 检查要求；完成所请求的描述文本交付物后即结束。
- **解释选项或默认值：** 阅读相应指南并作出解释，不要求提供媒体，也不启动工作。
- **运行生成：** 按照下文的“准备”“运行”和“采样与输出检查”执行。

模型／配置问题请阅读[模型约定](../shared/model-contract.zh-CN.md)。
请求解释并不代表授权启动作业、上传文件或修改源资产。

<a id="prepare"></a>
## 准备

1. 遵循[环境与快照说明](../shared/framework-runtime.zh-CN.md)。
   将 `TRANSFER_AUTO_HF` 设置为完整的本地 Transfer-Auto 快照路径。
2. 下载并校验[共享参考场景](../shared/reference-scene.zh-CN.md#get-the-scene)的校验和，或准备自己的场景。
3. 执行[输入准备](references/input-preparation.zh-CN.md)，该流程会在请求时调用[各视角描述文本编写](references/caption-guide.zh-CN.md)。
4. 对于参考场景，设置 `AUTO_REQUEST="$AUTO_SAMPLE_ROOT/input-transfer-auto.jsonl"`。
   默认选择段落式描述文本；若要使用推荐的带时间戳方案，请遵循
   [选择描述文本风格](../shared/reference-scene.zh-CN.md#choose-a-caption-style)。
   对于自己的数据，按照[输入与采样指南](references/input-and-execution.zh-CN.md)准备 Framework JSON／JSONL 请求。
   与提供的样例和 notebook 一样，在请求中设置 `"guidance": 3` 和 `"shift": 10`；
   推理模式从 JSON 读取这些设置。将 `AUTO_OUTPUT` 设置为一个新的输出目录。

准备完成后，应得到推理请求、逐相机配套描述文件（sidecar），以及同步的控制数据和可选初始观测。
仅有“相机键到描述正文”的字典，还不能直接运行推理。

只提供场景描述正文。Framework 会自动添加所选相机组和当前视角的方向／镜头／FOV 描述，
并将 WSM 控制任务指令作为系统提示词加入。不要在描述文件中重复这两部分。
参见[自动前缀](../shared/caption-policy.zh-CN.md#caption-text-and-automatic-prefixes)。

<a id="run"></a>
## 运行

在已分配的 GPU 环境中，从 Framework 根目录执行：

```bash
export COSMOS_INTERNAL=0 COSMOS_TRAINING=0
AUTO_NEGATIVE_PROMPT="$(cat .agents/skills/cosmos3-nano-transfer-auto/inference/assets/negative-prompt.txt)"
torchrun --standalone --nproc-per-node=4 \
  -m cosmos_framework.scripts.inference \
  -i "${AUTO_REQUEST:?set a prepared request}" \
  -o "${AUTO_OUTPUT:?set a fresh output directory}" \
  --checkpoint-path "${TRANSFER_AUTO_HF:?set the local snapshot}" \
  --resolution 720 --num-steps 35 \
  --per-view-negative-prompt "${AUTO_NEGATIVE_PROMPT:?Run from the Framework root and check the bundled negative-prompt.txt}" \
  --cp-size 4 --cfgp-size 1 \
  --no-use-torch-compile --no-diffusion-cache
```

请求提供 guidance 3 和 shift 10；命令设置 720p、35 步，以及与原生 Framework notebook
相同的随附负向提示词。这些 CLI 选项在运行时覆盖请求中的相应值，不会修改下载的 JSONL。
相机、描述文本、控制、初始图像、FPS、帧数和随机种子也来自请求。

在请求中明确列出物理视角；保留自定义子集时，不要再添加用于选择预设视角数量的覆盖参数。
模型快照包含自身配置；该命令不需要训练实验选择器。
此分布式示例需要四张满足要求的 GPU；显存是否足够及实际运行速度取决于请求和硬件。
更改分配方案前，阅读 [GPU 显存与并行](references/input-and-execution.zh-CN.md#gpu-memory-and-parallelism)。
保留已安装运行时的防护策略；如果为基准测试而禁用某项防护，必须明确记录。

<a id="sampling-and-output-checks"></a>
## 采样与输出检查

参考请求以 720p／30 FPS 生成 297 帧。首次进行简短检查时，将命令改为 `--resolution 480`，
并准备一个 61 帧区间。描述文本跨度与控制覆盖范围必须匹配所选区间：
由描述文本驱动的 Transfer-Auto 输出长度可能与顶层覆盖值不同。

480p 和 720p 均受支持。使用 `aspect_ratio: "16,9"` 获得与训练匹配的 832×480 或 1280×720 尺寸；
参见[分辨率与宽高比](references/input-and-execution.zh-CN.md#resolution-and-aspect-ratio)。
Transfer-Auto 在约 13.3 秒的块上训练，并通过分块续接支持更长的 RGB 序列。
扩展请求前请阅读[时长指南](references/input-and-execution.zh-CN.md#generation-length-and-long-rollouts)，
尤其是在联合生成 RGB／LiDAR 时。

分别检查[源时间关系、模型 FPS 和播放 FPS](references/input-and-execution.zh-CN.md#source-timing-model-fps-and-playback-fps)。
参考相机速率为 30 FPS；修改 `--fps` 不会对控制进行重采样，
将 10-FPS 帧声明为 30 FPS 只是一项诊断实验，不是质量修复方法。

使用独立的各视角描述文本时，通过 `per_view_negative_prompt` 而非 `negative_prompt`
传递所提供的[负向文本](assets/negative-prompt.txt)。比较时保留用户明确指定的设置。
[采样指南](references/input-and-execution.zh-CN.md#sampling)解释了 guidance 选项与联合 LiDAR 时间关系。

检查 `sample_args.json`、`sample_outputs.json`、所有相机／控制视频，
以及联合输出中的 `lidar.safetensors`。检查视角身份、实际时长、距离／强度数值和帧 ID。
使用匹配的裁剪和时间保存 50/50 WSM／RGB 叠加图。
叠加图对齐只能说明几何匹配，不能证明物理相机与模型相机键的映射正确。

更改输入或设置时，使用新的输出目录。

<a id="references"></a>
## 参考资料

- [相机、WSM 与描述文本输入检查](references/input-preparation.zh-CN.md)
- [各视角描述文本生成与策略](references/caption-guide.zh-CN.md)
- [模型约定](../shared/model-contract.zh-CN.md)
- [Framework 环境与模型资产](../shared/framework-runtime.zh-CN.md)
- [共享 Auto 参考场景](../shared/reference-scene.zh-CN.md)
- [请求字段、采样与输出](references/input-and-execution.zh-CN.md)
