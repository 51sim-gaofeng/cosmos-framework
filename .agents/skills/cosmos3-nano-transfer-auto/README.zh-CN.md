# Cosmos3-Nano-Transfer-Auto 中文文档目录

本目录收录 `cosmos3-nano-transfer-auto/` 下全部 15 份英文 Markdown 指南的中文阅读版，
涵盖模型与数据约定、推理、后训练和故障排查。加上本索引，共 16 份中文文档；
英文原文保持不变。

- 普通文档使用同目录的 `*.zh-CN.md`；原 `SKILL.md` 的中文阅读版为同目录
  `README.zh-CN.md`，不额外注册或修改 agent skill。
- 命令、代码块、配置键、接口名称及数值保持原样。代码块内的英文注释也保留，
  便于逐字核对命令和示例。
- 文档范围内的引用优先跳转到中文版，并保留原英文标题锚点以支持章节链接。
  指向其他目录（例如 Sim-Auto、Framework 安装指南和渲染器）的链接仍为原文；
  这些外部文档不属于本次翻译范围。
- 每份译文顶部都提供英文原文链接。源码或英文文档更新后，
  应同步核对译文。文中涉及的效果、支持范围和经验结论均来自对应原文，
  不代表译者已在当前硬件上验证所有配置，也不构成对自有数据效果的保证。

## 术语速查

以下说明用于辅助阅读；配置键、接口名称和命令参数仍使用原文写法。

| 术语 | 本文档中的含义 |
| --- | --- |
| caption | 描述文本：训练时描述真实目标，推理时描述期望生成的场景；不是叠加在视频上的字幕 |
| rig | 相机组或传感器配置，包括安装位置、朝向、镜头和标定；不等同于车型名称 |
| camera key | 相机键（`camera_key`），用于选择模型已学习的视角身份；不是完整的数值标定 |
| WSM 控制 | 按相机视角渲染的场景布局与运动条件，不代表真实车漆、天气或光照 |
| HD-map 距离图 | 将地图几何投影到 LiDAR 射线得到的控制数据，不是实测点云 |
| sidecar | 与媒体配套保存的独立描述或元数据文件 |
| clip／sweep | clip 指一个时间片段；sweep 指一次 LiDAR 扫描，文中称“扫描帧” |
| 配方（recipe） | 一组相互配套的模型、数据、训练目标和调度设置，不只是单条启动命令 |
| 后训练／微调 | 后训练是在已有模型上继续训练的阶段；微调是其中常见的一种方法 |
| 蒸馏／因果生成 | 蒸馏让学生模型学习教师模型；因果生成按时间顺序使用历史生成后续，二者不是同一概念 |
| 热启动／恢复训练 | 热启动以已有权重开始新任务；恢复训练还需还原优化器、调度器、迭代计数等状态 |
| 干净数据（clean） | 在扩散／流训练中未加入训练噪声的数据，不表示没有真实传感器噪声 |

## 从哪里开始

| 需求 | 阅读入口 |
| --- | --- |
| 了解模型输入输出、相机身份及工作流 | [模型约定](shared/model-contract.zh-CN.md) |
| 准备环境、模型权重和 tokenizer | [环境与模型资产](shared/framework-runtime.zh-CN.md) |
| 跑通官方白天示例 | [共用参考场景](shared/reference-scene.zh-CN.md) |
| 输入 WSM，生成相机视频／LiDAR | [推理指南](inference/README.zh-CN.md) |
| 适配自己的配对数据 | [后训练指南](post-training/README.zh-CN.md) |
| 排查视角变形、缺少 LiDAR 或运行失败 | [故障排查](troubleshooting/README.zh-CN.md) |

## 共享指南（4 份）

| 中文文档 | 英文原文 |
| --- | --- |
| [模型约定](shared/model-contract.zh-CN.md) | [model-contract.md](shared/model-contract.md) |
| [环境与模型资产](shared/framework-runtime.zh-CN.md) | [framework-runtime.md](shared/framework-runtime.md) |
| [两种 Auto 模型共用的白天场景](shared/reference-scene.zh-CN.md) | [reference-scene.md](shared/reference-scene.md) |
| [描述文本准备与自动前缀](shared/caption-policy.zh-CN.md) | [caption-policy.md](shared/caption-policy.md) |

## 推理（5 份）

| 中文文档 | 英文原文 |
| --- | --- |
| [推理指南](inference/README.zh-CN.md) | [SKILL.md](inference/SKILL.md) |
| [输入准备与相机映射](inference/references/input-preparation.zh-CN.md) | [input-preparation.md](inference/references/input-preparation.md) |
| [请求格式与执行参数](inference/references/input-and-execution.zh-CN.md) | [input-and-execution.md](inference/references/input-and-execution.md) |
| [逐视角描述文本指南](inference/references/caption-guide.zh-CN.md) | [caption-guide.md](inference/references/caption-guide.md) |
| [WSM 控制检查与颜色约定](inference/references/wsm-conventions.zh-CN.md) | [wsm-conventions.md](inference/references/wsm-conventions.md) |

## 后训练（4 份）

| 中文文档 | 英文原文 |
| --- | --- |
| [后训练指南](post-training/README.zh-CN.md) | [SKILL.md](post-training/SKILL.md) |
| [训练数据约定与相机身份](post-training/references/data-contract.zh-CN.md) | [data-contract.md](post-training/references/data-contract.md) |
| [四阶段训练配方与检查点衔接](post-training/references/canonical-recipe.zh-CN.md) | [canonical-recipe.md](post-training/references/canonical-recipe.md) |
| [训练参数指南](post-training/references/parameter-guide.zh-CN.md) | [parameter-guide.md](post-training/references/parameter-guide.md) |

## 故障排查（2 份）

| 中文文档 | 英文原文 |
| --- | --- |
| [故障排查指南](troubleshooting/README.zh-CN.md) | [SKILL.md](troubleshooting/SKILL.md) |
| [常见症状与检查项](troubleshooting/references/symptom-guide.zh-CN.md) | [symptom-guide.md](troubleshooting/references/symptom-guide.md) |
