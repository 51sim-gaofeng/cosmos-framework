[英文原文](wsm-conventions.md) · [中文目录](../../README.zh-CN.md)

> 本文为中文译文。配置键、类别标识、RGB 数值和命令保持原样，便于与代码核对。

<a id="wsm-control-checks"></a>
# WSM 控制检查说明

两种 Auto 模型均使用 **WSM v4.0.1** 布局控制。渲染器的配置选择项仍为 `version=v4`；不要改成 `v4.0.1`，也不要根据样例目录名推断渲染器版本。检查时应使用下方图例，并结合匹配的相机标定和经过验证的渲染器配置。WSM3 控制尚未被验证为可替代方案。不能仅凭颜色确认渲染器构建版本符合要求。

**WSM 颜色表示类别和信号状态，不表示真实车漆、光照、天气或地理环境。** 有损视频压缩会改变像素的精确数值。

| 特征 | 预期含义 |
| --- | --- |
| 动态物体 | 填充长方体，边缘为浅灰色，类别颜色沿物体前后方向渐变；不以明暗表示距离 |
| 交通标志 | 按类别着色的平面，不包含标志文字或真实轮廓；已识别标志的背面为深灰色 |
| 交通信号灯 | 正面表达状态；侧面和背面保持深灰色；方向三角形及行人、骑行者符号具有实际含义 |
| 未知状态／无激活信号 | 分别为灰色／近黑色；两者均不能解释为已知红灯或绿灯 |
| 路面箭头 | 支持的方向形状沿标注轮廓绘制；不支持或不适用的轮廓可能变成实心填充 |
| STOP／KEEP CLEAR 区域 | 分别为红色／绿色填充区域，不含文字 |
| 边界 | 颜色区分隔离设施／围栏、路缘、接缝、导向柱、墙及其他类型；子类型名称包含“虚线”也不意味着实际按虚线绘制 |
| 车道线 | 使用输入指定的颜色、样式和几何，与道路边界分开处理 |
| 人行横道／停止线／立杆 | 分别使用填充多边形／线／线；颜色见下方地图要素表 |

<a id="objects"></a>
## 物体

物体类别的参考 RGB，按正面到背面排列：

| 类别 | 正面 | 背面 |
| --- | --- | --- |
| 汽车（Car） | (0,46,136) | (126,206,255) |
| 卡车（Truck） | (204,55,0) | (255,192,64) |
| 行人（Pedestrian） | (148,0,62) | (255,124,171) |
| 骑行者（Cyclist） | (0,80,66) | (102,208,198) |
| 其他（Other） | (53,26,20) | (166,136,125) |

长方体边缘使用 (200,200,200)。颜色渐变表示物体朝向，不表示车漆或距离。投影必须遵循相机镜头模型，F-theta 畸变不可忽略。参考渲染器会对静止物体轨迹进行稳定处理，并在片段范围内对其尺寸取平均。包含相机中心的包围盒不会显示。粗粒度类别不能确定精确车型子类或持续一致的物体身份。

<a id="signs"></a>
## 交通标志

标志以平面矩形绘制，不呈现真实标志轮廓或印刷文字。

| 类别 | RGB |
| --- | --- |
| 停车让行（Stop） | (220,20,20) |
| 减速让行（Yield） | (255,140,0) |
| 禁止驶入（Do not enter） | (200,0,100) |
| 单行道，方向未指定 | (40,110,220) |
| 向左单行 | (0,180,230) |
| 向右单行 | (30,60,200) |
| 人行横道 | (255,220,0) |
| 已识别标志的背面 | (40,40,40) |
| 其他／未识别 | (8,2,255) |

已识别但不属于上述命名类别的标志，例如限速标志，使用通用正面颜色和深色背面。未识别标志的两面均使用通用颜色。不要从通用着色面推断具体限速数值或其他文字。

<a id="signals"></a>
## 交通信号灯

| 状态或表面 | RGB |
| --- | --- |
| 红灯 | (240,50,50) |
| 黄灯 | (255,200,0) |
| 绿灯 | (130,240,132) |
| 红黄灯 | (208,136,72) |
| 白色行人通行信号（`white_vru`） | (255,255,255) |
| 未知状态 | (150,150,150) |
| 已观测但无激活信号（`stateless`） | (20,20,20) |
| 正面外壳 | (105,105,105) |
| 侧面／背面外壳 | (60,60,60) |

三角形表示左转、直行或右转。行人／骑行者的红灯状态使用 X 形符号；绿灯和黄灯使用对应颜色的圆形。白色行人通行信号使用白色圆形。未知状态不等于熄灭、红灯或绿灯。画面中不同信号灯可能控制不同驶入方向；出现在图像中的信号灯不一定适用于自车。

<a id="road-markings-and-boundaries"></a>
## 路面标记与边界

七种方向箭头类别使用近白色 (244,244,244) 绘制：`forward`、`left`、`right`、`left_forward`、`right_forward`、`merge_left`、`merge_right`。它们需要受支持的类别和四角轮廓。不支持的类别或不适用的轮廓（例如五角轮廓）会变成实心填充；退化区域会被跳过。缺少箭头细节不能证明路面标记缺失。

| 路面涂绘区域 | RGB | 含义 |
| --- | --- | --- |
| STOP | (255,0,0) | 填充多边形，无文字 |
| KEEP CLEAR | (0,255,32) | 填充多边形，无文字 |
| TEXT PAINT／未识别标记 | (20,254,185) | 通用填充，不编码文字内容 |

KEEP CLEAR 和 TEXT PAINT 多边形会在投影前细分，以适应镜头畸变。道路边界使用**连续实线折线**绘制，即使输入子类型的名称包含点线或虚线也是如此：

| 边界分组 | 包含类型 | RGB |
| --- | --- | --- |
| 隔离设施／围栏 | `barrier`、`fence` | (253,1,232) |
| 路缘 | `curb`、`low_curb`、`tall_curb` | (207,255,191) |
| 道路边界／接缝 | `road_boundary` | (0,159,96) |
| 障碍杆／导向柱 | `obstacle_bars` | (255,207,159) |
| 墙 | `wall` | (0,128,159) |
| 其他 | 其余、缺失或未知类型 | (239,191,255) |

<a id="lane-lines-and-other-map-features"></a>
## 车道线与其他地图要素

车道线不同于道路边界。输入提供的颜色／样式组合决定绘制线型。白色为 (255,255,255)，黄色为 (255,255,0)，无法识别的组合为 (181,164,71)。参考映射如下：

| 颜色／样式组合 | 线型 | 相对粗细 |
| --- | --- | --- |
| `WHITE SOLID_SINGLE` | `solid` | 1 |
| `WHITE LONG_DASHED_SINGLE` | `long_dashed` | 1 |
| `WHITE SHORT_DASHED_SINGLE` | `short_dashed` | 1 |
| `WHITE DOT_DASHED_SINGLE` | `dot_dashed` | 1 |
| `WHITE OTHER` | `solid` | 3 |
| `WHITE SOLID_GROUP` | `solid`, `solid` | 1 |
| `YELLOW DOT_SOLID_SINGLE` | `dotted_1_9` | 1 |
| `YELLOW DOT_SOLID_GROUP` | `dotted_1_9`, `dotted_1_9` | 1 |
| `YELLOW DASHED_SOLID` | `solid`, `long_dashed` | 1 |
| `YELLOW SOLID_DASHED` | `long_dashed`, `solid` | 1 |
| `YELLOW SOLID_SINGLE` | `solid` | 1 |
| `YELLOW SOLID_GROUP` | `solid`, `solid` | 1 |
| `YELLOW LONG_DASHED_SINGLE` | `long_dashed` | 1 |
| `YELLOW OTHER` | `solid` | 3 |
| `OTHER` | `solid` | 1 |

同时列出两个线型时，表示按渲染器配置顺序组成双线；不要仅根据类别名称推断其在图像中的左／右位置。

| 地图要素 | 几何形式 | RGB |
| --- | --- | --- |
| 立杆（Pole） | 折线 | (183,69,177) |
| 停止线（Waitline） | 折线；标记为隐式时省略 | (108,179,59) |
| 人行横道（Crosswalk） | 填充多边形 | (139,93,255) |

<a id="renderer-evidence-and-geometry"></a>
## 渲染器依据与几何检查

Cosmos Framework 在 `.agents/skills/cookbooks/cosmos3/simulation/auto/wsm/` 下提供了可独立安装的渲染器。
请按照其 [README](../../../cookbooks/cosmos3/simulation/auto/wsm/README.md) 配置独立 Python 环境、GPU／EGL 和 FFmpeg 依赖，并了解许可证要求。需要转换场景时，请遵循[场景转换指南](../../../cookbooks/cosmos3/simulation/auto/wsm/skills/scenario-to-rds-hq/SKILL.md)。本文提供的是描述文本编写与输入检查所需的图例，不能替代渲染器，也不提供对文档未说明的传感器格式的转换能力。

请检查以下相对于渲染器包的资源：

- `docs/wsm4.md`：WSM v4.0.1 行为及渲染说明。
- `docs/rds-hq.md`：类别转换输入、坐标、标定和时钟。
- `cosmos_auto/wm/color_scheme/config_color_*.json`：调色板；车道线几何配置位于 `config_color_geometry_laneline.json`。
- `cosmos_auto/wm/conf/`：所选版本、投影、相机和输出配置。

在已安装渲染器的 `wsm/` 目录中，可以使用以下命令检查本地配置：

```bash
cosmos-auto-wm \
  input_root="${RDS_HQ_ROOT:?set your converted local scenario}" \
  output_root="${WSM_OUTPUT:?set a new output directory}" \
  version=v4 \
  output_frame_spec.target_render_fps=30 \
  output_frame_spec.target_chunk_frame=400 \
  output_frame_spec.overlap_frame=0 output_frame_spec.max_chunk=1 \
  --cfg job --resolve
```

渲染前，在配置中选择匹配的相机组、裁剪和分辨率。检查完成后，移除 `--cfg job --resolve`，将结果渲染到新目录。这里请求 400 帧是为了覆盖训练范围；仅准备推理控制时，应使用所选推理时间区间。渲染器附带的场景包含标注和标定，但没有录制的 RGB／LiDAR，因此不能替代配对的[白天参考样例](../../shared/reference-scene.zh-CN.md)或你自己的训练目标数据。

记录渲染器构建版本及解析后的配置，包括 `version=v4`。对于 RDS-HQ 输入，WSM v4.0.1 读取的是 `3d_traffic_lights_lightnet_status`，而不是旧的 `3d_traffic_lights_status`；需要检查状态覆盖范围和时间关系，不能只检查是否存在包围盒。参考渲染器默认输出为 10 FPS，而不是模型 RGB 的 30 FPS。应配置并验证所需的采样间隔；仅修改 MP4 的 FPS 元数据并不能同步场景内容或对其重新采样。

逐一验证各相机的投影类型、光轴方向、视场角（FOV）、裁剪和时间戳。F-theta 与针孔投影控制不能通过重命名文件互换。如果有源 RGB，应将控制与其对照检查，包括片段起止和存在歧义的事件。检查上游类别是否正确转换：粗粒度的卡车标签可能原本来自工程机械。

如果渲染器来源或所需类别的含义未知，应明确报告具体不确定之处，避免作出受影响的结论。未经授权且没有证据，不要修补控制、重新着色调色板，或根据文件名推断版本。
