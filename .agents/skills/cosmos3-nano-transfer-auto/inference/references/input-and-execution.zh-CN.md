[英文原文](input-and-execution.md) · [中文目录](../../README.zh-CN.md)

> 本文为中文阅读版。代码块、配置键和采样参数保持原样。

<a id="framework-requests-and-sampling"></a>
# Framework 请求与采样

使用 `cosmos_framework.scripts.inference` 和完整的本地模型快照。
准备一个 JSON 请求，或每行一个请求的 JSONL 文件。资产路径相对于该输入文件解析。
对于参考场景，使用 `$AUTO_SAMPLE_ROOT/input-transfer-auto.jsonl`。
保留原始文件，并在新目录中创建修正／裁剪后的变体。

<a id="camera-request"></a>
## 相机请求

下面用双相机请求演示如何引用自定义本地资产。对于已发布的七相机场景，
使用其提供的请求，不要重写路径。将描述文本替换为对已检查场景的描述。
可以直接提供初始图像；`condition_video_as_image` 从 RGB 视频中取第一帧。
所有相机使用一致的条件方式。

```json
{
  "model_mode": "image2video",
  "prompt": "Follow the per-camera controls and captions.",
  "resolution": "480",
  "fps": 30,
  "num_frames": 297,
  "guidance": 3.0,
  "shift": 10,
  "num_steps": 35,
  "seed": 0,
  "wsm": {},
  "multiview": {
    "load_caption_from_data": true,
    "condition_video_as_image": true,
    "use_first_chunk_only": false,
    "views": [
      {
        "camera_key": "camera_front_wide_120fov",
        "control_path": "wsm/camera_front_wide_120fov.mp4",
        "vision_path": "rgb/camera_front_wide_120fov.mp4",
        "caption_path": "captions/camera_front_wide_120fov.json"
      },
      {
        "camera_key": "camera_front_tele_30fov",
        "control_path": "wsm/camera_front_tele_30fov.mp4",
        "vision_path": "rgb/camera_front_tele_30fov.mp4",
        "caption_path": "captions/camera_front_tele_30fov.json"
      }
    ]
  }
}
```

对于 WSM 文生视频，将 `model_mode` 设为 `text2video`，
并省略 `vision_path` 字段和将视频用作图像的设置。
相机身份来自 `camera_key`，而不是顶层提示词。
所选相机保留[身份表](../../post-training/references/data-contract.zh-CN.md#camera-identities)中的物理 ID。
保留显式列表以维持自定义子集。

<a id="caption-sidecars"></a>
## 配套描述文件（sidecar）

每台相机的 `caption_path` 指向其独立文件：

```json
{
  "caption_structured": {
    "chunk_0_297": {
      "start_frame": 0,
      "end_frame": 297,
      "caption": "Camera-local description of this inspected interval."
    }
  }
}
```

边界基于 RGB 时钟，包含起始帧、不包含结束帧。
示例正文仅用于说明格式，不能代替对实际场景的描述。
由运行时统一添加相机、分辨率和 FPS 上下文信息，不要手动重复。
场景级 `prompt` 不能替代各视角文件。

选择其他源区间时，准备同步裁剪，并将其描述文本边界重新从零起算。
检查每台相机的覆盖范围。Transfer-Auto 的描述文本加载路径从描述文本块推导区间，
因此仅修改 `num_frames` 不够。300/400 帧的描述跨度可能在 RGB tokenizer 的
`1+4k` 网格上变成 297/397 帧。首次比较使用一个对齐的描述文本块，并检查实际输出时长。

<a id="joint-lidar"></a>
## 联合生成 LiDAR

提供的白天请求已经包含 LiDAR。对于自己的相机请求，添加以下顶层块。
不要添加 `joint_generation` 字段；Framework 请求格式不支持该字段：

```json
"lidar": {
  "control_path": "lidar/hdmap_rangemap.tar"
}
```

这是要插入对象中的片段，不是独立的 JSON 请求。
仅生成 RGB 时，省略整个 LiDAR 块。准备好的距离图归档保留源扫描 ID。
本地张量形式的控制还支持 FP32 safetensors，其 `frames` 张量形状为 `[3,T,H,W]`，
分别为以米计的距离、强度和有效性。不要用 MP4 渲染结果代替。
tokenizer 会自行执行填充／归一化。

可选的 `lidar.condition_path` 提供实测初始 LiDAR；
`lidar.num_conditional_sweeps` 设置目标前缀长度。这与 HD-map 控制分开。
使用前缀前，检查源 ID、距离投影和时钟。Sim-Auto 只允许其支持的初始前缀
（参见其输入指南）；不要直接套用双向模型中更宽松的前缀选项。

在 30 FPS RGB 和 10 Hz LiDAR 下，297 个推理帧对应 99 次扫描。
双向路径经取整后的数量与训练中包含端点的 61/21 和 397/133 窗口不同。
检查返回的 ID 和源覆盖范围，不要声称它们采用相同的端点约定。

<a id="sampling"></a>
## 采样

对于已发布的 **Transfer-Auto** 模型，从以下设置开始。
在请求中明确设置，然后在自己的场景上比较改动效果。

| 设置 | Transfer-Auto 起始值 |
| --- | --- |
| 分辨率／FPS | 参考示例使用 720p／30；480p 可降低显存使用 |
| 宽高比 | `"16,9"`：每台相机 832×480 或 1280×720 |
| 帧数 | 简短检查用 61；长示例用 297 |
| 步数／shift／随机种子 | 35 / 10 / 0 |
| Guidance | 3；1.5 用于较低 guidance 的比较 |
| Control guidance | 1；多视角纯 RGB 路径要求该值 |
| 各视角负向提示词 | 通过 `per_view_negative_prompt` 使用随附的[负向文本](../assets/negative-prompt.txt) |
| 分布式示例 | CP4 / CFGP1 |
| 可选编译／缓存 | 初始禁用；启用前先比较 |

<a id="gpu-memory-and-parallelism"></a>
### GPU 显存与并行

根据**所选模型和请求**规划资源分配：单 GPU 显存、GPU 数量和互联、
相机数量、分辨率、时长、仅 RGB 还是联合 LiDAR，以及采样设置都很重要。
示例使用四张 GPU，并不代表任意四 GPU 系统都能容纳所有请求。
Transfer-Auto 和 Sim-Auto 的执行与显存要求不同；不要把一个模型的显存或速度估计套用到另一个。

**上下文并行（CP）**将序列计算分布到多个 GPU。
增大 `--cp-size` 可以减少每 GPU 的激活显存并加速大型请求，
但会增加通信开销：更多 CP 或更多 GPU **不保证**更低延迟。
**FSDP** 对模型权重分片；它处理的是显存使用中的另一部分，
并不会消除激活、tokenizer 或输出缓冲区。
以 Auto 示例明确给出的 CP／CFGP 设置为起点；
Sim-Auto 要求 `--cfgp-size 1`。

首次低显存运行时，使用 480p、更少的所选相机或更短的对齐区间，
并提供匹配的控制和描述文本跨度。然后再检查预期的完整请求。
使用相同模型、输入、步数和编译／缓存选项比较并行设置。
同时测量模型生成耗时和包含输出保存的总耗时：去噪加速不一定带来同比例的端到端加速。

训练还要保存梯度、优化器状态和反向传播激活，蒸馏可能还保有额外网络。
因此推理资源分配并不等于训练显存建议。
请使用独立的[训练显存指导](../../shared/framework-runtime.zh-CN.md#training-memory)。

<a id="resolution-and-aspect-ratio"></a>
### 分辨率与宽高比

**Transfer-Auto 和 Sim-Auto 均支持 480p 和 720p**。
使用横向 `aspect_ratio: "16,9"` 预设（CLI 为 `--aspect-ratio 16,9`），
以尽量匹配训练时的图像几何，这是使用已发布模型时推荐的起点：

| 请求分辨率 | 每台相机的输出尺寸 | 与训练的匹配情况 |
| --- | --- | --- |
| `"480"` | 832×480 | 训练的 480p 尺寸；约 16:9 |
| `"720"` | 1280×720 | 训练的 720p 尺寸；恰为 16:9 |

480p 预设**不是** 854×480：使用运行时的 832×480 尺寸。
这些是每台相机的尺寸，而不是拼图尺寸。RGB 和 WSM 的空间变换必须匹配；
通过裁剪或拉伸改变宽高比也会改变有效相机几何。
准备输入时保留标定／对齐。对于 Auto，优先使用这些与训练匹配的横向尺寸。

720p 的**每台相机像素数约为 480p 的 2.3×**，因此需规划更大的显存与计算资源。
需要更多空间细节时使用 720p，需要更低资源消耗时使用 480p。
若交付物需要更大尺寸，在受支持尺寸下生成，再对独立输出副本进行放大。

<a id="generation-length-and-long-rollouts"></a>
### 生成长度与长时滚动生成

示例时长不是模型上限。在 **30 FPS** 下，约 300 帧覆盖 10 秒，900 帧覆盖 30 秒。
共享样例生成 **297 帧（9.9 秒）**。以实际保存帧数为准：
tokenizer 对齐、描述文本跨度和续接重叠可能改变最终时长。

**Transfer-Auto** 使用 30 FPS 下的 400 个源帧块训练，约 **13.3 秒**；
RGB tokenizer 将完整窗口对齐为 397 帧。
10 秒请求处于该时间尺度内，并不会仅仅因为更短就天然更差。
20 或 30 秒序列超出单个训练窗口。
对于较长 RGB 序列，使用分块续接，并检查物体身份、运动、几何及块间过渡。

对于**仅 RGB 的 Transfer-Auto**，Framework 支持描述文本分块续接：
在每台相机的描述文件中提供匹配的块边界、覆盖完整序列的控制，
并设置 `multiview.use_first_chunk_only: false`。
每个块进行双向去噪，以已生成的尾帧作为下一块条件。
修改顶层 `num_frames` 不会把长描述文本拆成多个块。
使块长度接近或低于训练窗口，检查实际输出时长和接缝。
**联合 RGB／LiDAR Transfer-Auto 当前每个请求生成一个描述文本块**。
提供一个对齐的描述区间，或设置 `multiview.use_first_chunk_only: true` 选择第一个区间。
联合生成中，禁用该选项时若包含多个描述文本块，请求会被拒绝。

**Sim-Auto** 以滚动更新的历史为条件，按时间顺序生成较小的潜变量块，
因此请求时长不必局限于一个 Transfer-Auto 训练窗口。
原文报告的定性测试显示，**持续数分钟的滚动生成没有明显的可见质量损失**；
这不代表所有自定义场景都能达到同样效果。
10 秒和 30 秒请求使用同样的滚动机制；
应为完整区间提供同步控制和描述文本，并检查后期画面、物体身份的持续一致性和跨视角一致性，而不只检查开头。
较长运行需要额外预算计算量和输入／输出内存。

<a id="source-timing-model-fps-and-playback-fps"></a>
### 源时间关系、模型 FPS 与播放 FPS

这三个概念不能混为一谈。已发布的 Auto 方案使用 **30 FPS 相机数据**。
将请求 FPS 与输入的有效采样率匹配。

| 量 | 含义 | 检查内容 |
| --- | --- | --- |
| 源采样率 | RGB／WSM 观测之间的实际时间间隔：10 FPS 时为 100 ms，30 FPS 时约为 33 ms | 时间戳、帧选择及同步；只有 MP4 头信息不够 |
| 请求 `fps`／CLI `--fps` | 模型假定的相机速率；影响时间位置／mRoPE 缩放和格式化后的 FPS／时长元数据 | 匹配有效源采样率，确保推理时间关系正确 |
| 保存视频的播放速率 | 生成帧展示的速度 | Framework 的 Auto 保存器当前使用请求 FPS；事后改变播放速率不改变生成过程 |

请求中不存在独立的 `multiview.output_fps` 字段。
若只比较播放效果，在生成后使用相同生成帧创建独立查看副本。
保留原始输出和请求。

修改 `--fps` 不会重采样多视角控制：加载器按索引选择解码帧。
不要将 10-FPS 录制数据重新标记为 30 FPS 并宣称兼容。
真正转换需要为所有受影响输入制定明确、同步的重采样策略；
复制／插值帧不能恢复缺失观测。
帧索引改变时，重新计算描述文本帧边界，保留事件时间，
并对照处理后的控制时间线检查所有提供的 RGB 条件。

数量与时长必须一致：297 个保存帧在 30 FPS 下播放 9.9 秒，
在 10 FPS 下则是 29.7 秒。
检查解析后的描述文本跨度、帧数和保存视频元数据；
仅修改顶层帧数不一定能选择新区间。

<a id="diagnose-a-quality-drop-with-10-fps-camera-data"></a>
#### 诊断 10-FPS 相机数据导致的质量下降

1. 确认实际源时间戳和加载器读取的帧。
   如果存在原始 30-FPS 录制数据，为同一物理场景区间保留输入条件匹配的 30-FPS 基线。
   同时改变源采样和模型 FPS 的比较无法隔离任何一个原因。
2. 对于**仅 RGB** 测试，使用对齐的 10-FPS RGB／WSM，并设置请求 `fps: 10`。
   保持检查点、相机、空间变换、描述文本、随机种子和采样设置不变。
   检查时长及每台相机的运动和几何。
3. 可选地，以请求 `fps: 30` 重复运行**相同的 10-FPS 帧序列**，
   保持相同帧数和输入。这会有意改变模型的时间假设及推导出的描述文本元数据。
   同时检查实际生效的描述文本；带时间戳文本此时可能与该时钟冲突。
   如有需要，以 10 FPS 查看独立副本。
   模型仍将观测到的位移解释为以**三倍速度**发生；减慢播放无法撤销这一点。
   该对照实验仅用于区分 FPS 条件的影响，不能作为质量修复方案。

每种情况使用新的输出目录，并记录源采样、请求 FPS、实际生成帧数、
描述文本时间关系和查看副本的播放速率。
领域适配请使用 [Transfer-Auto 后训练指南](../../post-training/README.zh-CN.md)，
并使用目标采样率的留出录制数据评估结果。
对于 Sim-Auto，应在因果蒸馏之前适配 Transfer-Auto。

**联合 RGB／LiDAR 需要单独检查时间关系。**
参考配置是 30-FPS 相机与 10-Hz LiDAR；10-Hz LiDAR 不意味着 10-FPS 相机。
相机 FPS 还决定扫描数量和源帧对齐。
当前联合 Sim-Auto 拒绝 30 以外的相机 FPS。
不要绕过该检查，也不要将重新标记实验用于联合输入；
保留受支持的传感器时钟，并验证距离控制 ID。

<a id="what-guidance-means"></a>
### guidance 的含义

`guidance` 是**无分类器引导（CFG）**的强度：
采样器比较正向描述文本条件下的预测与空描述或负向描述条件下的预测，并放大二者差异。
在 **1** 时，使用正向预测而不作额外放大；
模型仍读取描述文本和控制。大于 1 时，CFG 增加第二条预测分支和额外计算。

从 **3** 开始。如果结果显得过于刻意或不够自然，使用相同随机种子和输入比较 **1.5**。
更强 guidance 可以提高提示词遵循度，但并不全面提升质量，也可能放大伪影或提示词矛盾。
这两个值都不能修复错误视角 ID 或未对齐的 WSM。
`control_guidance` 单独控制布局条件的 CFG 强度，不等同于文本引导强度；
此基线保持为 **1**。在 1 时 WSM 条件仍有效。

<a id="what-shift-and-steps-mean"></a>
### shift 和步数的含义

`shift` 改变去噪时使用的**噪声水平调度**。
更大的 shift 将更多采样点分配到较高噪声水平，
改变早期结构形成和后期细化之间的平衡。
它不会在时间上移动视频、改变相机同步或添加帧。
初始保持 **shift 10**。

`num_steps` 控制去噪步数。**35** 是起始方案；
减少步数可减少计算。使用相同输入和随机种子比较步数。
首次比较 guidance 或描述文本时，固定 shift 和步数，以明确变化原因。

<a id="what-a-negative-prompt-means"></a>
### 负向提示词的含义

负向提示词描述不希望出现的结果，用于 CFG 比较分支。
它是偏好信号，不是硬性禁止或安全过滤器。
在各视角 Auto 路径中，一个 `per_view_negative_prompt` 字符串由所选相机共享。
省略时，该分支使用空描述。在 **guidance 1** 时，负向文本没有 CFG 效果。

阅读随附的[负向提示词](../assets/negative-prompt.txt)，
将其文本放入顶层 `per_view_negative_prompt` 字段，进行减少伪影的比较。
普通 `negative_prompt` 在各视角路径上并不等价。
先审阅文本：提供的示例包含恶劣天气遮蔽和镜头效果。
如果这些效果是所请求场景的一部分，应省略或修改冲突条款。
不存在数值上的“最佳负向提示词”；
在固定 guidance、随机种子和输入下，比较使用兼容负向提示词与不使用时的结果。

<a id="sim-auto-uses-a-distilled-schedule"></a>
### Sim-Auto 使用蒸馏后的固定采样调度

对于已发布的 **Sim-Auto** 学生模型，使用 **guidance 1**、
**control guidance 1** 和 **normalize_cfg false**。
引导行为已通过蒸馏融入其权重。
检查点提供固定的四步调度；不要移植 Transfer-Auto 的 35 步／shift-10 方案，
也不要指望负向提示词增加 CFG。
初始观测与滚动生成规则请遵循
[Sim-Auto 推理指南](../../../cosmos3-nano-sim-auto/inference/SKILL.md)。

<a id="compare-settings-on-a-fixed-scene"></a>
### 在固定场景上比较设置

首先验证相机映射、WSM 对齐和
[与训练一致的描述文本](../../shared/caption-policy.zh-CN.md#keep-training-and-inference-captions-aligned)。
然后比较 guidance 3 与 1.5，或兼容负向提示词与不使用负向提示词，
每次只改变一个因素。
固定检查点、随机种子、裁剪、时长和初始 RGB 条件；每个变体使用新的输出目录。
除了提示词遵循度，还要检查侧／后视角及时间／跨视角一致性。
解释比较结果前，检查 `sample_args.json` 中解析后的值。

HF 快照只包含导出时选定的权重版本。
推理时设置 EMA 标志，不能恢复模型产物中并未保存的另一套权重。
保留快照的注意力、tokenizer 和描述文本元数据。

<a id="inspect-outputs"></a>
## 检查输出

逐一检查各相机视频，以及 `sample_args.json`／`sample_outputs.json`。
存在保存的控制视频时，也要检查它们。
当前联合 Transfer 保存器不写入控制视频；
制作叠加图时，使用相应源 WSM，并采用相同的解析帧区间和空间变换。
联合输出包含数值形式的 `lidar.safetensors`，其中有距离、强度和帧 ID。
除了预览，还要检查数值数据，核对几何、运动和跨视角一致性。

更改设置后，在新的输出目录中重新运行。
每次比较一个因素，固定检查点、随机种子、输入几何和条件模式。
