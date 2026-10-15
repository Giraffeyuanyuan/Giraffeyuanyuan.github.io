---
title: LOD 代码、数据与参考资料交接说明
date: 2026-10-15 20:00:00
updated: 2026-10-15 20:00:00
tags:
  - LoD
  - 三维重建
  - CityGML
  - CityJSON
  - 3D Gaussian Splatting
categories:
  - 学习笔记
  - 三维重建
description: 整理 OSGB / HeightMask 与 Gaussian / LoD2 两条建筑重建路线，涵盖代码地图、实验数据、运行方法、质量复核和单栋最小复现实验，并附四张建筑语义示意图。
toc: true
---

本文整理 LOD 建模项目的代码、数据与参考资料，供接手项目和复现实验时查阅。

> 阅读说明：文中标注的“交接包内”路径，以及代码、配置和实验数据路径，均指原交接包或原工作区中的文件，需在对应本地环境中查看。博客收录说明正文和四张语义图；代码、数据、模型权重及本地参考文献未随文章上传。

<!-- more -->

这份说明给刚接手 LOD 建模工作的同学使用。建议先读本文件，再按“最小可运行路线”查看代码和数据，最后阅读论文与规范资料。

## 一、项目目标

将高斯点云、OSGB 或 LAS 等三维数据转换为可检查、可融合的建筑 LoD 模型，重点关注 LoD1.3、LoD2.2，以及 CityJSON/CityGML 建筑语义组织。

当前主要有两条路线：

| 路线 | 输入 | 核心方法 | 结果 |
|---|---|---|---|
| OSGB / HeightMask | OSGB 倾斜摄影网格 | 高程栅格、建筑组件分类、足迹规整、参数化重建 | LoD1.3，主要是平顶体量 |
| Gaussian / LoD2 | 3D Gaussian Splatting PLY | 俯视渲染、SAM3 实例分割、RoofMapNet 屋顶线、footprint 规整、Roofer 重建 | LoD2.2，包含屋顶形状 |

一句话理解：OSGB 路线偏“几何测量与规则判断”，Gaussian 路线偏“多视角渲染 + AI 分割 + 几何重建”。

## 二、优先阅读顺序

1. 工作区总览（交接包内：`参考资料/00-index.md`）：工作区总索引。
2. 流水线全景解读（交接包内：`参考资料/PIPELINE_GUIDE.md`）：两条流水线的通俗全景图。
3. 代码架构说明（交接包内：`参考资料/01-pipeline-architecture.md`）：代码模块和入口。
4. 输入数据说明（交接包内：`参考资料/03-input-data.md`）：输入数据格式、大小和字段。
5. 配置与环境说明（交接包内：`参考资料/05-configs-and-envs.md`）：YAML 配置和运行环境。
6. 实验档案（交接包内：`参考资料/02-experiments.md`）：历史实验和结果状态。

7. 本文末尾的四张语义图，以及原交接包的 `README.md`：用于理解 `BuildingPart`、`BuildingInstallation`、`OuterFloorSurface` 等概念。

## 三、代码地图

原交接包中的代码总目录：`代码/`

### 1. LoD2-automation：主调度器

路径：`代码/LoD2-automation/`

- `lod2_automation\__main__.py`：模块入口。
- `cli.py`：`run`、`batch`、`validate`、`doctor`、`list-stages` 等命令。
- `runner.py`：阶段调度、断点续跑、逐栋失败隔离、最终报告。
- `config.py`：YAML/JSON Schema 配置和默认值合并。
- `stages\preflight.py`：输入 PLY 体检。
- `stages\segment_instances.py`：全场景实例发现和逐栋点云切分。
- `stages\build_roofprint.py`：DOM、RoofMapNet、SAM3、足迹规整和 GPKG。
- `stages\run_roofer.py`：调用 Roofer 生成单栋 LoD2。
- `stages\assemble_scene.py`：合并 CityJSON/OBJ。
- `stages\fix_base_z.py`：底面高程修正。
- `stages\evaluate.py`：点云覆盖率、平面度、LoD2 误差等评估。

### 2. RoofMapNet：屋顶线与足迹约束

路径：`代码/RoofMapNet/`

- `tools\render_gaussian_ply_dom.py`：高斯点云转 DOM。
- `tools\run_single_ply_dom_roofline.py`：DOM 屋顶线推理入口。
- `infer_dom.py`、`dom_inference.py`：模型推理和 NPZ 输出。
- `line_postprocess.py`、`line_analysis.py`：线段过滤、拓扑检查和调试。
- 预训练权重：`models\roofmapnet.pth`。

### 3. 3d_seg：全场景分块和实例分割

路径：`代码/3d_seg/`

- `scripts\topdown_red_blue.py`：俯视渲染、多视角融合和实例发现。
- `scripts\sam3_instance_from_images.py`：二维实例结果映射回三维。
- `semantic_instance_pipeline.py`：语义投票和实例合并。
- `scene\gaussian_model.py`、`gaussian_renderer\`：3DGS 数据和渲染。

### 4. SAM3：建筑掩膜

路径：`代码/sam3/`

- `demo_batch.py`：批量 mask 生成。
- 权重：`models\sam3\sam3.pt`。

### 5. Roofer：屋顶几何重建

路径：`代码/roofer/`

- `rooferpy\`：Python/C++ 绑定。
- `src\core\reconstruction\`：平面检测、边界构造和重建。
- `src\extra\io\`：CityJSON、LAS、GDAL/OGR 输出。
- `scripts\roofprint_regularization\build_regularized_roofprint.py`：04/05 两条路线共用的足迹规整脚本。

### 6. 单栋调试入口

single_building_lod_cityjson.py（交接包内：`代码/单栋工具/single_building_lod_cityjson.py`） 是一个单栋 LAS → LoD1.3/LoD2.2 CityJSON 的实验入口。它可以：

- 读取单栋 LAS；
- 使用已有 footprint，或谨慎地从单栋点云推断凸包 footprint；
- 生成分类 LAS（ground=2、building=6）；
- 调用 Roofer；
- 分离 LoD1.3、LoD2.2 CityJSON；
- 输出 `run_report.json`、输入 SHA256 和几何 QA。

该脚本适合单栋验证，不适合直接替代场景级实例分割。

## 四、示例数据和结果

### 主 Gaussian 数据（原工作区数据，不随代码包复制）

- 主输入：原工作区 `D:\merge_output\merge_ply.ply`；数据位置和字段说明见 03-input-data.md（交接包内：`参考资料/03-input-data.md`）。
- 约 1.6 GB，约 29,776,564 个高斯点。
- 字段包括位置、尺度、旋转、透明度和颜色特征；本身没有建筑语义标签。
- 当前主场景是公安场景，约 90–100 栋建筑，已有交付结果为 84 栋。

### 单块 Gaussian 示例

- 原工作区 `D:\program\data_yuyi\yicheng\merged_by_prefix\42.ply`
- 宜城场景块示例，可配合 DOM、mask、GPKG 和 Roofer 结果学习完整中间流程。

### 主实验

原交接包内配置和报告：`示例数据/主实验_公安05/`；完整大体量输出仍保留在原工作区。

- 配置：`gongan_merge_ply.local.yaml`
- 结果：`output\底部修复\gongan_merge_ply_05\fusion\scene_regularized_lod22.city.json`
- OBJ：`output\fusion\scene_lod2_roof.obj`
- 评估：`output\evaluation\evaluation_summary.md`
- 失败/复核清单：`output\failed_or_review_required.csv`
- 运行记录：`output\workflow_report.md`、`run_report.json`、`run_snapshot.json`

当前记录：实例发现 105 个，Roofer 成功输出 84 栋；部分阶段为 `review_required`，不能把所有 review 都理解为几何失败。

### OSGB / LoD1.3 示例

- HeightMask 源码位于 `代码/LoD2-automation/lod2_automation/osgb_lod13/`。
- 典型中间产物：`raw_full_mesh.ply`、`height_components.json`、`roofprint_report.json`、`elevation_report.json`、`tile_manifest.json`。
- 宜城成果说明：宜城LOD结果与中间文件说明.md（交接包内：`参考资料/宜城LOD结果与中间文件说明.md`）。

### Golden reference

Golden reference 原始位置见 代码与数据清单（交接包内：`代码/README.md`），大文件未复制到原交接包。

- `HAV\building_0049\`：单栋黄金样本。
- `OSGB\lod13_heightmask_canary20\`：OSGB LoD1.3 20 瓦片样本。
- `OSGB\real_osgb_lod2_test2_dense\`、`real_osgb_lod2_test3_dense\`：LoD2 测试样本。

## 五、运行方法（先做小规模验证）

在 WSL/对应运行环境中：

```bash
cd <原工作区>/experiments/05_merge_ply_gongan
source env_gongan.sh
python -m lod2_automation doctor
python -m lod2_automation validate gongan_merge_ply.local.yaml
python -m lod2_automation inspect gongan_merge_ply.local.yaml
python -m lod2_automation run gongan_merge_ply.local.yaml
```

建议先运行 `doctor`、`validate` 和单栋/小样本，再启动完整场景。RTX 4070 Ti SUPER 16 GB 环境建议 `workers: 1`。

配置重点：

- `segmentation.prompts: [building, roof]`；
- `single_building.dom_size: 4096`；
- `single_building.regularization_mode: moderate`；
- 无 LiDAR 时 Roofer 使用 `complete_gaussian`；
- `resume: true`，`continue_on_building_failure: true`。

## 六、已知风险

1. `code/` 整棵代码树没有完整 Git 基线，修改前必须复制文件或保存实验目录。
2. 3d_seg 多视角渲染在 16 GB 显存上可能卡死，应注意 `torch.no_grad()`、释放显存和单进程分块。
3. DOM 尺寸和像素到世界坐标的换算强耦合，不能随意重采样 4096×4096 DOM。
4. RoofMapNet 的线目前主要服务于 footprint 质量改进，不能简单等同于最终屋顶几何。
5. `review_required` 是质量复核门，不一定等于失败；必须查看对应 manifest、日志和 QA 图。
6. 当前主实验的 `reference_mode: own_input` 属于自参考评估，不能直接当作外部真实精度。
7. 本文末尾四张语义图是教学示意图，不是点云真值，也不能直接转换成 CityGML 几何。

## 七、论文、算法与规范参考

### 本地资料

- 项目代码深挖笔记.md（交接包内：`参考资料/项目代码深挖笔记.md`）：HeightMask、RoofMapNet、OSGB+LAS 试点以及 LoD3 方向的代码分析。
- TECHNICAL_ALGORITHMS_FOR_PAPER_CN.md（交接包内：`参考资料/TECHNICAL_ALGORITHMS_FOR_PAPER_CN.md`）：OpenGaussian 语义/实例分割的公式、投票、聚类和消融设计。
- LOD0-LOD4_分级定义与项目判定指南_图解版.pdf（交接包内：`参考资料/LOD0-LOD4_分级定义与项目判定指南_图解版.pdf`）：项目内部的 LoD 分级图解。
- Roofprint到LoD3建筑结构图解.pdf（交接包内：`参考资料/Roofprint到LoD3建筑结构图解.pdf`）：从 footprint 到 LoD3 的结构表达示意。
- ISPRS Annals 论文 PDF（交接包内：`参考资料/isprs-annals-X-4-W6-2025-225-2025.pdf`）：与屋顶结构/建筑重建方向相关的本地论文资料，阅读时以论文首页作者、题名和实验设置为准，不要只根据文件名引用。

### 项目笔记中提到的外部研究方向

- RoofMapNet：渐进式节点提取、遮挡感知和屋顶结构线预测。
- LOD3-Model：利用斜摄影和实例/窗口检测，将 LoD2.2 向门窗细节扩展。
- LOD3_buildings：基于 SfM、CNN 和 FreeCAD 的 LoD3 建筑重建。
- 3DGS / OpenGaussian：高斯表示、多视角渲染、语义投票和实例分割。

### 官方规范

- [PLATEAU 建筑物 LoD2 总览](https://www.mlit.go.jp/plateaudocument/toc4/toc4_02/toc4_02_01/toc4_02_01_03/_f89b6454-422a-5298-ffed-3c750c3d4835)
- [PLATEAU OuterFloorSurface 定义](https://www.mlit.go.jp/plateaudocument/toc4/toc4_02/toc4_02_03/toc4_02_03_01/_8b7b6076-e598-4db5-3051-1e0eae71574f)

## 八、建议师弟的第一周任务

1. 阅读 `00-index.md`、`PIPELINE_GUIDE.md` 和本说明。
2. 用一个小 PLY 跑通 `doctor → validate → inspect`。
3. 检查一个单栋实例的 DOM、mask、footprint 和 CityJSON。
4. 对照四张语义图，手工判断阳台、屋顶露台、雨棚和楼梯的语义类别。
5. 阅读 `single_building_lod_cityjson.py`，理解 LAS 分类、footprint、Roofer 调用和 QA。
6. 最后再尝试完整 05 路线，不要一开始修改主配置或覆盖主实验结果。

## 九、推荐的最小复现实验

先做 最小复现实验_单栋LOD（交接包内：`最小复现实验_单栋LOD/README.md`）。它会生成 4,939 个点的确定性两坡屋顶 LAS 和一个 GeoJSON footprint，再调用单栋重建脚本，输出 LoD1.3、LoD2.2 CityJSON 和 QA 报告。这个实验不需要复制主场景的几十 GB 数据，是熟悉“输入—分类—Roofer—CityJSON—QA”链路的第一步。

## 十、建筑语义图解

以下四张图来自原交接包，生成于 2026-10-14，是 AI 辅助的概念教学示意，不是点云测量成果，也不能直接作为 CityGML 几何数据导入。

### 1. 悬挑阳台：语义剖面

![1. 悬挑阳台：语义剖面](/img/lod-handover/cantilever-balcony-section.png)

用剖面说明悬挑阳台。蓝色表示作为 `BuildingInstallation` 的附属构件；仅可通行的顶部表面用橙色表示 `OuterFloorSurface`。图中保留了窗、门等剖面表达，因此不要把它当作“纯 LoD2 几何规范图”。

### 2. 悬挑阳台与屋顶露台

![2. 悬挑阳台与屋顶露台](/img/lod-handover/balcony-and-roof-terrace.png)

左侧是悬挑阳台，整体附着构件属于 `BuildingInstallation`；右侧是建筑体量顶部的可通行屋顶露台，地面属于 `OuterFloorSurface`，周边低层体量仍是 `BuildingPart`，普通屋面为 `RoofSurface`。

### 3. LoD2 层级递进

![3. LoD2 层级递进](/img/lod-handover/lod2-levels.png)

从 LoD2.0 到 LoD2.2 的概念递进。LoD2.0 简化外形；LoD2.1 增加主要外部安装；LoD2.2 增加较小的外部安装。窗、门在这张图中只作为平面立面标记，不表达窗洞、窗框或室内空间。

### 4. 建筑语义分解

![4. 建筑语义分解](/img/lod-handover/building-semantics.png)

将同一建筑拆解为两个主要 `BuildingPart`、边界表面和少量外部安装，并用虚线表示对应关系。它适合讲解组织方式；若用于生产数据，仍需依据真实点云、拓扑闭合性和项目采集门槛重新建模。

### 颜色约定

- 紫色：`RoofSurface`
- 蓝灰色：`WallSurface`
- 灰色体量：`Building` / `BuildingPart`
- 蓝色：`BuildingInstallation`
- 橙色：`OuterFloorSurface`；图中接地板也使用橙色作视觉区分，实际语义应按数据来源判定 `GroundSurface`
- 深灰矩形：平面窗/门立面标记，不代表三维开口

颜色不是 CityGML 编码的一部分，正式数据应以语义类、几何关系和 XML/数据库属性为准。
