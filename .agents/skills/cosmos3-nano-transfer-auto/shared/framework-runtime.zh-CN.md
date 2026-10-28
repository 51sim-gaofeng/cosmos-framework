<a id="environment-and-model-assets"></a>

# 环境与模型资产

[英文原文](framework-runtime.md) · [中文目录](../README.zh-CN.md)

> 本文为中文阅读版，命令和版本号保持原样。
> 请按实际硬件选择环境；译文不代表已在当前机器上验证所有训练配置。

应在包含 `docs/training_transfer_auto.md`、`docs/training_sim_auto.md`
和 `cosmos_framework/scripts/train_experiment.py` 的 Cosmos Framework 仓库中运行。
容器、系统软件包和 GPU 驱动的安装应遵循该仓库的 `docs/setup.md`。
以下命令使用仓库的 Python 3.13 训练环境。
先按照安装指南安装要求的 `uv` 版本（`0.11.29`）和系统软件包。
驱动兼容时选择 CUDA 13；对于较旧且没有受支持的前向兼容配置的驱动，
选择 CUDA 12.8 依赖组。

```bash
uv sync --locked --python 3.13 --group cu130-train --extra train --extra guardrail
source .venv/bin/activate
export COSMOS_INTERNAL=0
```

推理流程导入的部分工具也依赖 `train` 可选依赖组（extra）。
另一个可选依赖组 `guardrail` 提供默认推理所需的 `nltk` 等依赖；
仅使用 `--extra train` 不会安装它们。
保留仓库的锁文件：其中已经指定 FlashAttention-4。
不要随意升级 attention／CUTLASS 组件，更不能假定升级后的数值结果与原环境等价。
验证 PyTorch、TorchCodec／FFmpeg、CUDA 和注意力相关模块可以导入。
运行时兼容并不意味着显存能容纳特定训练或推理任务。
还应执行安装指南中的 GPU 资源分配检查和视频解码检查。
保留必需的 CUDA 兼容库路径；不要仅因为运行在容器里就重置 `LD_LIBRARY_PATH`。

训练需要多 GPU 资源、可工作的 NCCL，以及每个 rank 都能访问的数据集／检查点路径。
将 `IMAGINAIRE_OUTPUT_ROOT` 设置为训练输出目录并导出，
使工作进程可以读取它。需要时设置并导出缓存位置 `HF_HOME` 和 `UV_CACHE_DIR`。
机密凭据应通过专用凭据管理机制保存，绝不能写入清单文件（manifest）。

<a id="training-memory"></a>

## 训练显存与内存规划

训练与推理需要分别进行容量规划。除了模型权重，训练还需要梯度、
优化器状态和反向传播所需的激活；蒸馏阶段还可能同时加载教师网络和
用于估计生成分布得分的 fake-score 网络。
内存需求取决于训练阶段、相机数量、分辨率、片段长度及打包后的 token 预算，
而不仅是 GPU 数量。

上下文并行（CP）分配序列计算，FSDP 则分片模型和优化器状态。
增加 CP 不能替代 FSDP 的状态分片。
激活检查点（activation checkpointing）通过重计算减少激活内存占用；
CPU 激活卸载则以主机内存和传输开销换取 GPU 显存。
根据训练指南选择拓扑和内存设置，然后在预期负载下检查前向、反向、
检查点保存和重新加载。通过短训练检查，并不能证明资源足以容纳更长序列或更大的训练批次（batch）。

<a id="checkpoint-memory"></a>

## 检查点保存的内存要求

GPU 显存、主机 RAM 和 `/dev/shm` 容量是独立的约束。
在训练容器中检查 `free -h` 和 `df -h /dev/shm`。
异步 DCP 保存会在每个 rank 的页锁定共享 CPU 内存中暂存模型和优化器状态；
显存足够并不能保证保存时的内存足够。

进行短训练、保存和恢复检查时，使用 `checkpoint.dcp_async_mode_enabled=false`。
这能避开后台页锁定共享内存暂存路径，但仍需要主机 RAM 和常规数据加载器共享内存。
在较长任务中启用异步保存前，应为所有 rank 的检查点状态配足主机 RAM
和容器共享内存，然后完成一次保存和重新加载。
在允许的情况下，使用安装指南提供的 Docker IPC 选项。
不要把磁盘上出现了一个目录视为检查点已经保存完成。

<a id="model-snapshots"></a>

## 模型快照

| 工作流 | 模型仓库 | 本地变量 |
| --- | --- | --- |
| 分阶段训练的基础模型 | `nvidia/Cosmos3-Nano` | `NANO_HF` |
| Transfer 推理、微调及 Sim 基础模型 | `nvidia/Cosmos3-Nano-Transfer-Auto` | `TRANSFER_AUTO_HF` |
| 已发布 Sim 模型推理 | `nvidia/Cosmos3-Nano-Sim-Auto` | `SIM_AUTO_HF` |

使用已获授权的完整模型快照。从模型发布渠道获取不可变版本标识，
并在下载前设置对应的 `*_REVISION` 变量；
不要编造版本标识，也不要把 `main` 当成不可变版本。例如：

```bash
hf download nvidia/Cosmos3-Nano-Transfer-Auto \
  --revision "${TRANSFER_AUTO_REVISION:?set the chosen snapshot revision}" \
  --local-dir "${TRANSFER_AUTO_HF:?set a local snapshot directory}"
```

需要时，Nano 或 Sim-Auto 也使用相同方式。
接受适用的许可证，并通过 `hf auth login` 或 `HF_TOKEN` 认证，不要打印 token。
仓库 ID 不是本地检查点目录。配置、权重和 tokenizer 应来自同一快照。

<a id="training-components"></a>

## 训练组件

- 使用后训练指南将主模型的 HF 权重转换为 DCP。
  这是仅加载权重的初始化，不会恢复优化器历史。
- 使用匹配的 RGB VAE：训练配方要求 Wan2.2 VAE `.pth` 文件，
  该文件在 `Wan-AI/Wan2.2-TI2V-5B` 中发布。
- 将匹配 Auto 快照中的 `lidar_vae/config.json`
  和 `diffusion_pytorch_model.safetensors` 保存在一起。
  不要将这个 tokenizer 转换为 DCP。
- 构建训练数据时，使用匹配的本地文本 tokenizer 目录，
  通常是快照中的 `text_tokenizer/`。

VAE 是辅助组件，不是主模型的另一种训练起点。
检查实际快照目录结构，并明确绑定路径。
更换 LiDAR 传感器或投影方式时，仅保证 tokenizer 输入形状相同并不足以确保兼容。

<a id="execution-modes"></a>

## 执行模式

训练／配置解析时设置 `COSMOS_TRAINING=1`，推理时设置 `COSMOS_TRAINING=0`。
Auto 训练使用 `cosmos_framework.scripts.train_experiment`；
通用单视频 SFT 示例并未实现这些传感器训练配方。
两种 Auto 模型的推理都使用 `cosmos_framework.scripts.inference`。

在结果中保留 Framework 版本、锁文件和模型快照版本，
以便比较输出或恢复训练时复现环境。
