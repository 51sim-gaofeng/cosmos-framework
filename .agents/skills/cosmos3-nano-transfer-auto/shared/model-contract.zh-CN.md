<a id="cosmos3-nano-transfer-auto"></a>

# Cosmos3-Nano-Transfer-Auto 模型约定

[英文原文](model-contract.md) · [中文目录](../README.zh-CN.md)

> 本文为中文阅读版。

模型根据同步的场景布局视频（WSM 控制）和逐相机描述文本（caption）生成驾驶视频。
如需以已有外观作为生成起点，可提供初始 RGB 图像。
加入 HD-map 距离图控制后，还可以同时生成相机视频和 LiDAR 数据。

Transfer-Auto 使用双向生成，即同一生成窗口内的不同时间位置可以相互参考。
若需要依据历史按时间顺序持续生成的因果流式工作流，
请使用 [Sim-Auto](../../cosmos3-nano-sim-auto/inference/SKILL.md)。

<a id="inputs-that-must-stay-consistent"></a>

## 必须保持一致的输入

- 每份控制数据、caption 和初始图像都必须与其物理相机匹配。
  相机键（`camera_key`）选择的是模型已学习的视角身份；
  模型不会根据文件名或图像内容自动纠正错误的身份分配。
- 两种 Auto 模型均使用 WSM v4.0.1 控制；渲染器选择项为 `version=v4`。
- 控制和目标使用相同场景区间、相机投影及裁切。
  检查相机／LiDAR 对齐时，要包含实际时间戳。
- 推理时，从用户或场景元数据获取目标时段、天气、路面状况和地理环境。
  训练时，这些描述必须与配对的真实目标及采集元数据一致；遵循
  [训练 caption 指南](../post-training/references/data-contract.zh-CN.md#training-captions)。
  WSM 提供交通布局和运动，而不是这些外观与环境属性。
- 逐相机 caption 应分别保存。每个相机读取自己的文本；
  LiDAR 可以读取全部相机 caption。
- 使用选定权重随附的模型配置和 tokenizer。
  记录模型产物版本，并注明使用普通权重还是移动平均权重（EMA）。
  不要仅因检查点名称相似就选择另一个检查点。

<a id="model-settings-not-user-setup-steps"></a>

## 应随模型保留的设置

已发布模型使用学习得到的传感器身份，每个相机内部采用双向注意力。
额外的跨相机／传感器注意力连接覆盖当前时刻及此前 0.4 秒，
并对重叠的注意力键（key）去重。控制数据仍使用配置中指定的同视角连接。
微调、加载或导出时，应保留随附模型配置中的这些设置；
不要根据产品名称字符串重新构造它们。

模型支持 11 个相机身份和一个 LiDAR 身份。
相机标定用于验证／渲染控制数据；
本训练配方不会通过位姿位置编码输入相机位姿张量。

<a id="choosing-a-workflow"></a>

## 选择工作流

- **使用已发布模型：** 准备控制数据／caption 并运行
  [推理](../inference/README.zh-CN.md)，无需原始训练任务的历史状态。
- **适配自己的数据：** 从已发布权重及其最终模型配置开始，
  准备配对的目标／控制数据，并按照
  [后训练指南](../post-training/README.zh-CN.md) 定义新的微调任务。
- **复现分阶段训练：** 按照
  [训练配方](../post-training/references/canonical-recipe.zh-CN.md) 的四个阶段，
  使用自有配对数据从 Cosmos3-Nano 开始。

根据选定工作流，使用 [后训练](../post-training/README.zh-CN.md)、
[推理](../inference/README.zh-CN.md) 或
[故障排查](../troubleshooting/README.zh-CN.md) 指南。
保留 Transfer-Auto、Sim-Auto 文档及其共享资源的目录结构，避免链接失效。

<a id="execution-prerequisites"></a>

## 执行前提

使用 [Framework 环境与模型资产](framework-runtime.zh-CN.md)、
原生 [推理请求](../inference/references/input-and-execution.zh-CN.md)
和 [原生训练索引](../post-training/references/data-contract.zh-CN.md#local-training-index)。
命令应在匹配的 Cosmos Framework 仓库中运行。

[共享示例场景](reference-scene.zh-CN.md) 提供 HF 下载方式、
校验和说明、本地目录结构及共用训练索引。
该数据版本与下述所需兼容模型快照的版本相互独立。

| 任务 | 必需组件 |
| --- | --- |
| 生成 | 完整模型快照／版本、匹配的配置／tokenizer、准备好的请求及同步控制／caption |
| 直接微调 | 已发布 Transfer-Auto 权重、原生本地数据包中的用户配对数据、全新的训练状态和选定调度策略 |
| 完整分阶段训练 | Cosmos3-Nano 初始化模型、匹配的 tokenizer、覆盖各阶段片段长度的用户数据包，以及四阶段衔接 |

如果某项资产缺失或与选定模型冲突，应报告具体缺口。
不要用通用视频数据集替代，也不要仅凭文件名或张量形状相同就推断
相机／LiDAR 兼容。

<a id="handoffs"></a>

## 工作流交接

| 从 → 到 | 交接内容 | 检查项 |
| --- | --- | --- |
| 数据／caption → 训练或推理 | 相机／控制／目标映射、同步区间、渲染器设置、场景属性和独立 caption | 检查像素、实际时间戳及加载器实际读取的文本 |
| 训练 → 推理 | 权重、匹配的模型配置／tokenizer、权重版本和已解析训练设置 | 检查加载是否完整且兼容，以及所有请求相机和可选 LiDAR 的输出 |
| 推理 → 故障排查 | 输入请求、已解析设置、输出，以及一个明确失败的相机／时间点 | 检查相机映射、控制、caption 及受影响的输出区间 |

准备变体时保留原始输入。除非用户明确请求外部工作流，否则机密媒体应保留在本地。
