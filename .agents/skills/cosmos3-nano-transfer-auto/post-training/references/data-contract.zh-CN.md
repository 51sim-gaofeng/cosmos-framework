<a id="prepare-custom-training-data"></a>
# 准备自定义训练数据

[英文原文](data-contract.md) · [中文目录](../../README.zh-CN.md)

> 本文为中文阅读版。代码块、配置键和数据格式保持原样。

使用自己的同步驾驶采集数据和渲染控制数据。数据约定关注内容、
时序和传感器含义，不要求特定的存储服务或采集系统。

<a id="source-files"></a>
## 源文件

本地组织方式示例如下：

```text
dataset/
  train/scene-0001/
    rgb/<camera_key>.mp4
    wsm/<camera_key>.mp4
    captions/<camera_key>.json
    lidar/target/
    lidar/control/
    timestamps.json
    calibration.json
  validation/scene-0002/...
```

这些文件夹用于组织源文件；下文的索引才是原生构建工具的输入。
Framework 构建工具打包已有媒体和 tokenizer 资源。
它不渲染 WSM、不生成 caption，也不合成实测 LiDAR。

如需根据自己的场景渲染控制数据，请遵循
[WSM README](../../../cookbooks/cosmos3/simulation/auto/wsm/README.md) 和
[场景转换指南](../../../cookbooks/cosmos3/simulation/auto/wsm/skills/scenario-to-rds-hq/SKILL.md)，并执行
[渲染器检查](../../inference/references/wsm-conventions.zh-CN.md#renderer-evidence-and-geometry)。
你需要地图几何、物体轨迹、自车位姿、信号灯状态，以及经过标定的传感器投影/时序。
仅有 RGB 和实测 LiDAR 采集数据并不能提供这些标注。
按匹配的 LiDAR 射线单独准备 HD-map 距离图控制数据；
不要将相机控制 MP4 作为数值型 LiDAR 控制数据使用。

<a id="local-training-index"></a>
## 本地训练索引

创建一个 JSONL 文件，每行对应一个场景区间。资产路径相对于该 JSONL 文件解析。
以下为单相机示例；为每个选定相机添加 video/WSM/caption 条目，
并保留其物理键名：

```json
{"id":"drive1-clip1","stratum":"480","assets":{"video_camera_front_wide_120fov":"train/scene-0001/rgb/camera_front_wide_120fov.mp4","wsm_camera_front_wide_120fov":"train/scene-0001/wsm/camera_front_wide_120fov.mp4","caption_camera_front_wide_120fov":"train/scene-0001/captions/camera_front_wide_120fov.json","lidar/lidar_rangemap":"train/scene-0001/lidar/target/lidar_rangemap.tar","lidar/hdmap_rangemap":"train/scene-0001/lidar/control/hdmap_rangemap.tar"}}
```

保持 `id` 与距离图归档内部的片段键一致。`stratum` 为 `480` 或 `720`；
默认训练配方会采样两者，因此需要同时提供，或明确禁用缺失的数据流。

每个 caption 文件包含：

```json
{"caption_structured":{"chunk_0_400":{"start_frame":0,"end_frame":400,"caption":"Description grounded in this camera's recorded RGB and recording metadata."}}}
```

帧范围包含起始帧、不包含结束帧。将示例区间和文本替换为源数据的真实覆盖范围及描述。
保留完整文本；阶段格式化器会选择适当的窗口或首段。
即使前广角相机不在选定的 RGB 子集中，也要包含其 caption 元数据，
供仅 LiDAR 训练使用。

距离图 `.tar` 文件使用
`cosmos_framework.model.generator.tokenizers.lidar.preprocessing.load_prepared_lidar_video_3ch`
读取的稀疏格式。每个扫描帧应包含以下归档成员：

```text
<clip-id>.<source-frame-id>.lidar_row.npz
<clip-id>.<source-frame-id>.lidar_col.npz
<clip-id>.<source-frame-id>.lidar_range.npz
<clip-id>.<source-frame-id>.lidar_intensity.npz
```

每个 NPZ 在 `arr_0` 下存储一个一维数组；各数组长度相等。
行索引是 [0,128) 内的整数光束编号，列索引是 [0,3600) 内的整数，距离单位为米。
打包前，对重复射线保留最近回波。实测强度为 [0,1] 浮点数或 [0,255] 整数。
HD-map 控制省略 intensity 成员；预处理时，已占用射线的强度设为 1。
使用与索引匹配的一致片段 ID，以及实际相机时钟对应的源 ID
（对于对齐的规则 30/10 Hz 数据为 0,3,6,...）。
不要通过重新编号掩盖扫描帧缺失。
使用下文所述的投影，而不是任意图像索引。
[参考场景](../../shared/reference-scene.zh-CN.md) 展示了具体文件名和原生索引，
并提供 HF 下载及校验和说明。

按照 [后训练指南](../README.zh-CN.md#prepare-a-local-training-bundle) 运行
`cosmos_framework.data.generator.local_simulation.auto_bundle`。
它会将 `manifest.json`、`samples.json`、`assets/` 和 `resources/tokenizer/`
写入新目录。打包后，检查同步的 RGB/WSM 叠加图以及解码后的训练 batch。

| 来源 | 内容 | 作用 |
| --- | --- | --- |
| 每个相机的 RGB | 选定场景区间的真实视频 | 目标；可选用未加训练噪声的初始帧作为条件 |
| 每个相机的 WSM | 按该相机的有效投影、裁剪和时间渲染的布局 | 完整且不加训练噪声的控制，不是外观真值 |
| 逐视角 caption | 逐相机描述正文，具有匹配的样本内局部时间跨度 | 每个相机使用独立文本序列 |
| HD-map 距离图控制 | 投影到预期 LiDAR 射线上的几何，已占用射线的强度为 1 | 完整且不加训练噪声的 LiDAR 控制 |
| 实测 LiDAR 距离目标 | 已知扫描帧 ID 对应的实测米制距离、归一化强度和有效性 | 目标；可选用未加训练噪声的初始扫描帧作为条件 |

本文所述 `auto_bundle` 构建工具要求**完整的配对行**：
包括两个距离图条目、每个选定相机的 RGB／WSM／caption，以及前广角 caption，
即使没有选中该相机也必须提供。分传感器训练从这些配对行中选择
仅 RGB 或仅 LiDAR 任务；打包时不允许缺少成员。
在数据集中保留许可与授权记录，以及按场景／行驶记录划分训练集和验证集的标识。
绝不能把同一次驾驶中相邻且重叠的片段拆分到训练集和验证集中。

使用 WSM v4.0.1 约定（渲染器选择器 `version=v4`），
包括信号灯状态/正面/背面、车道标线和投影。
使用相同的裁剪和缩放检查同步的控制/RGB 叠加图。
[输入检查指南](../../inference/references/input-preparation.zh-CN.md)
涵盖训练和推理所需的相机几何及渲染器检查。

<a id="training-captions"></a>
## 训练描述文本（caption）

优先使用带时间戳的描述，说明每个相机的场景及其演变。
这是原文所述模型训练数据使用的格式；与之保持一致有助于减小后训练数据与原训练数据之间的格式差异。
单段 caption 同样可用。
[参考场景](../../shared/reference-scene.zh-CN.md#choose-a-caption-style)
同时提供 `captions/` 和 `captions_with_timestamp/`，
并说明如何先修改训练索引以选择后者，再构建数据包。

将带时间戳的段落保留在现有 `caption` 字符串中，
并保留外层 `start_frame` / 不包含结束帧的 `end_frame` 边界。
使用真实的片段内局部时间，在有意义的变化处分段，而不是强制等分。
样本中带时间戳的段落属于一个外层 chunk，不是独立的训练样本。
保留完整源文本；选定的训练配方会应用完整 caption 或首段策略。

训练文本必须描述**实际录制的场景**，而不是你希望模型之后生成的外观。
对于包含 RGB 的样本，按以下顺序使用信息来源：

1. 检查选定相机和区间内同步的目标 RGB，确认可见外观、天气、
   光照、路面和运动。对于像素无法确定的事实（如地理环境），使用采集元数据。
2. 将 WSM 与这些目标进行比较，核对相机身份、几何、参与者和时序。
   控制与目标不一致时需要调查；不要改写 caption 来掩盖问题，
   也不要虚构缺失的控制物体。
3. 保留有依据的已有描述；撰写逐相机正文，并在跨视角和时间维度上
   保持身份、外观和物理运动一致。同一参与者的颜色/类型和轨迹必须
   在共同时钟上保持一致；对于朝向不同的相机，屏幕运动方向相反或
   可见时间区间不同可能是正确的。不要为未改变的目标赋予新的车漆、
   服装、背景或天气。如果某属性无法观察且缺少元数据，
   应保持未指定并报告限制，而不是猜测。
4. 在开始/结束时刻及描述的变化处，将实际使用的 caption 与目标 RGB 核对；
   对模糊事件还应检查额外帧。只有完成此目标内容核对后，
   才应用该阶段的完整正文或首段策略。

例如，白天干燥道路上的真实蓝色汽车，不能仅因为请求了某种数据增强，
就被标注为白色汽车和雨夜场景。新的天气或外观条件需要相应的目标数据，
而不只是编辑标签。

保持 [caption 风格与推理一致](../../shared/caption-policy.zh-CN.md#keep-training-and-inference-captions-aligned)：
优先使用带时间戳的逐相机描述，并在不同视角间保持外观/运动一致。
上述基于真实内容的规则仍适用于每个训练目标。

随附的 `av-per-view-caption` 指南用于**编写推理提示词**：
在未来 RGB 将由模型生成时，它可以根据 WSM 赋予合理的外观。
不要将这种仅基于 WSM 的流程作为训练标签生成器。
如果无法检查所需的 RGB 目标，应保留已有 caption 并标记为未验证；
不要宣称新 caption 适合作为监督信号。

**分传感器阶段中的仅 LiDAR 任务：** 格式化器使用对齐的前向相机 caption
元数据，而不为该任务加载相机视频。数据包仍包含所需的相机成员。
对照源采集数据和距离图配对数据，核验 caption 来源、场景身份及区间；
应用该阶段的首段策略。LiDAR 本身无法验证车漆、光照或天气。
应使用配对的采集数据/元数据，而不是编造这些属性。

<a id="camera-identities"></a>
## 相机身份

适配自己的相机组（rig）时，保留实际标定对应的视点，并渲染与采集 RGB 对齐的 WSM，
包括相同的裁剪和投影。准备这些训练配对数据时，不要将控制重定向到参考相机组：
它会改变控制视点，却不变换 RGB 目标。参见
[参考相机组与自定义相机组工作流](../../inference/references/input-preparation.zh-CN.md#choose-reference-rig-or-custom-rig-generation)。

对于自己的相机组，在构建训练索引前，使用
[Camera Rig Explorer](../../../cookbooks/cosmos3/simulation/auto/wsm/docs/camera-rig/guide.md)
和 [最接近的相机身份映射流程](../../inference/references/input-preparation.zh-CN.md#use-camera-rig-explorer-for-your-own-rig)。
保持此映射与推理一致。源文件名可以保留你自己的命名；
在索引中将其路径与选定的受支持相机键关联。
原生数据包构建工具接受受支持且唯一的键；
它不会为任意传感器名称创建新的视角嵌入。

即使删除或重排视角，也要保留这些已学习的身份：

| ID | 相机键 |
| --- | --- |
| 0 | `camera_front_wide_120fov` |
| 1 | `camera_cross_right_120fov` |
| 2 | `camera_rear_right_70fov` |
| 3 | `camera_rear_tele_30fov` |
| 4 | `camera_rear_left_70fov` |
| 5 | `camera_cross_left_120fov` |
| 6 | `camera_front_tele_30fov` |
| 7 | `camera_front_fisheye_200fov` |
| 8 | `camera_left_fisheye_200fov` |
| 9 | `camera_right_fisheye_200fov` |
| 10 | `camera_rear_fisheye_200fov` |

LiDAR 使用传感器身份 11。选定子集中的索引不是已学习的传感器 ID。
例如，前广角／左后／前长焦仍保留 ID 0/4/6。
相机名称中的标称信息不等同于完整标定。分配自定义相机前，
比较朝向、FOV 和镜头投影。使用与采集 RGB 相同的标定渲染控制。

<a id="time-and-spatial-alignment"></a>
## 时间与空间对齐

- 参考训练配方使用标称 30 FPS 相机和 10 Hz LiDAR。
- 使用真实时间戳/源 ID，在相机、控制、caption 和 LiDAR 之间
  选择同一区间，而不只是让文件时长相同。
- 对 RGB 和 WSM 使用相同的选帧与缩放/中心裁剪进行解码。
  渲染/裁剪时一致地应用标定变化。
- 为适配 tokenizer，将 RGB 裁剪为 `1 + 4k` 帧。
  短序列联合阶段使用 61 帧 RGB / 包含端点的 21 个扫描帧；
  完整的长序列阶段窗口在解码最多 400 帧 RGB 后，
  达到 397 帧 RGB / 133 个扫描帧。
- 显式处理缺失/不规则扫描帧。规则张量索引不会保留被省略的源时间戳。
  拒绝数据不足/未同步的样本，或采用有文档说明且经过验证的对齐策略。

使用片段内局部 caption 时间。30 FPS 下的 61 帧片段播放时长为
61/30 秒，而最后一帧的时间戳为 60/30；它们是不同的量。
存储足够信息，以便恢复选定的源区间。

<a id="model-facing-batch-contract"></a>
## 模型输入批次（batch）约定

以下是原生加载器的输出要求，不是用于替代数据集文件结构的规范。
将新数据集投入训练前，应在实际 batch 中验证这些要求。

1. **视觉：**`video=[WSM_control, RGB_target]`，每项为 `[3,V*T,H,W]`，
   在模型归一化前为 uint8。按相机优先顺序展平：
   先排列一个相机的所有时刻，再排列下一个相机。
   选定视角 ID 和逐视角文本应保持相同顺序。
2. **LiDAR：**`lidar=[HDmap_range_control, LiDAR_target]`，
   每项为 `[3,T_lidar,128,1808]`。通道分别为米制距离、
   [0,1] 内的强度及二值有效性。距离投影将 3600 个原生水平分箱
   池化为 1800 条实际投影射线，再循环填充到 1808。填充部分不代表额外的物理射线。
   使用提供的 Pandar128 光束仰角标定、从 +180° 到 −180° 且不包含终点的
   方位角、x 向前/y 向左/z 向上的坐标，以及最近回波选择。
   tokenizer 的仿射归一化使用固定的 5–100 m 范围；
   无效距离为零，并配有独立的有效性掩码。
   不要用任意 128 行图像或不同距离尺度替代。
   其他传感器需要经过验证的投影适配器，而非仅改变形状的缩放；
   参考投影不支持垂直重采样。保留 tokenizer 的过滤/有效性规则。
3. **视频编码：**将 RGB/WSM 归一化到 [-1,1]，各相机独立编码；
   时间/空间压缩为 4/16，latent 通道数为 48，模型 patch 大小为 2。
   **LiDAR 编码：**无时间下采样，空间压缩为 16，
   latent 通道数为 128，模型 patch 大小为 1。使用提供的 tokenizer。
4. **文本：**最终阶段保留选定相机的完整逐相机正文。
   由运行时统一添加相机／相机组及片段时长／分辨率的上下文前缀，不要重复添加。
   按样本整理 caption 列表，保留独立的文本边界。
   短序列联合阶段的 caption 缩短策略见训练配方。
5. **条件：** 控制信号始终保持干净，即不加入训练噪声；目标前缀仅在 I2V 中保持干净。
   在 30/10 Hz 下，一个作为条件的相机 latent 对应一个初始实测 LiDAR 扫描帧。
   *整段* LiDAR 条件概率为零并不禁用这一时间前缀。
6. **Dropout/损失：**联合训练中，样本级和逐视角 caption dropout
   各使用 0.1 的概率；两个长序列联合阶段均在 10% 的样本上
   同时丢弃两个 LiDAR 条目。这不是仅丢弃前缀。
   其他情况下控制信号保持不变。干净元素的预测误差为零，
   但仍计入配置的局部条目均值分母；
   不要改为仅对参与预测的元素计算均值。

检查已安装的加载器如何处理数据损坏、短片段及缺失 caption。
统计被拒绝/重试的样本，并检查预期的数据源混合比例是否仍能实现。
对于一个真实样本，保留解码帧、选定 ID/时间、caption、
距离单位/掩码、格式化后的形状、条件掩码和打包长度。
分别验证保留 LiDAR 的情况和丢弃 LiDAR 的情况。

在开始训练前，使用这些检查验证新数据集。
