<a id="troubleshoot-cosmos3-nano-transfer-auto"></a>

# Cosmos3-Nano-Transfer-Auto 故障排查

[英文原文](SKILL.md) · [中文目录](../README.zh-CN.md)

> 本文为中文阅读版，不是独立注册的 agent skill。
> 原文名称为 `cosmos3-nano-transfer-auto-troubleshooting`。
> 原文描述：诊断模型故障，包括侧／后视角变形、WSM 对齐、caption 不匹配、
> 帧数异常、LiDAR 缺失、内存不足，以及检查点或恢复失败。
> 用于定向检查与恢复，不用于自动重映射输入、重新训练、
> 因果 Sim-Auto 调试或质量认证。

Cosmos3-Nano-Transfer-Auto 根据布局控制和逐相机描述文本（caption）
生成驾驶视频及可选 LiDAR。
从一个失败示例开始，在 [症状指南](references/symptom-guide.zh-CN.md) 中选择对应部分。
兼容性检查使用 [共享模型约定](../shared/model-contract.zh-CN.md)；
该模型不使用因果 Sim-Auto 推理工作流。

<a id="collect-one-failing-example"></a>

## 收集一个可定位问题的失败示例

收集实际使用的请求／命令、已解析配置、检查点及权重版本、
实际读取的逐视角文本、相机键 → 控制数据映射、
选定时间区间、一份失败输出及其控制数据，以及错误／日志片段。
共享诊断信息时，不要包含凭据和机密源文件位置。

如果问题涉及视频时序或尺寸，先对比实际文件：

```bash
: "${TRANSFER_AUTO_CONTROL:?Set an existing local WSM video path}"
: "${TRANSFER_AUTO_RGB:?Set its corresponding local generated RGB video path}"
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,r_frame_rate,avg_frame_rate,nb_frames:format=duration \
  -of json "$TRANSFER_AUTO_CONTROL"
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,r_frame_rate,avg_frame_rate,nb_frames:format=duration \
  -of json "$TRANSFER_AUTO_RGB"
```

输入是两个实际文件，输出是元数据报告。
检查通过需要满足：两个文件都能读取元数据，且按文档说明完成预处理后，
其时序和尺寸均与已解析请求一致。
如果缺少 `nb_frames`，应统计解码帧数。
检查同步帧以核对视角身份和事件对齐。

<a id="diagnose-correct-verify"></a>

## 诊断、修正、验证

使用症状指南检查可能原因，提出最小修正，
随后验证受影响的输入和输出。
保留原始 manifest、caption 和输出。

对于侧／后视角故障，先验证相机映射、控制数据和实际 caption，
再对比参考输入与失败场景，每次只改变一个可疑因素。

报告中应分别列出观测事实、待验证假设、已执行检查、修正建议和剩余不确定性。
将修正后的请求交给 [推理指南](../inference/README.zh-CN.md)，
或将状态／拓扑问题交给 [后训练指南](../post-training/README.zh-CN.md)。

<a id="references"></a>

## 参考资料

- [症状指南与检查项](references/symptom-guide.zh-CN.md)
- [模型约定](../shared/model-contract.zh-CN.md)
