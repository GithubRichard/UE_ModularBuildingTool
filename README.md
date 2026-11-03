# UE_ModularBuildingTool

UE5 地编（关卡编辑）辅助工具：为模块化建筑 Kit（CrossStreet，225 模块）提供
**模块库独立窗口 + 场景内快速拼装模式**——组件快速筛选、磁吸、对齐、选择，
独立 3D 窗口拖拽进场景。

> 当前阶段：设计评审。代码实现尚未开始。

## 文档

- [技术分析与交互设计](docs/ue5-level-editor-mode-design.md) —
  总体架构、技术选型（Slate C++ / 经典 FEdMode / FAssetDragDropOp 拖拽通道）、
  交互流程、磁吸与对齐行为规格、分阶段交付计划、风险提示。
- [交互效果图](docs/ue5-editor-mode-mockup.html) —
  两张高保真 mockup（用浏览器打开）：图 1 模块库独立窗口，
  图 2 拼装模式视口 + 磁吸/对齐工具条。

## 数据来源

设计复用 `ModularBuilding` 现有注册表：
`data/kits/crossstreet/registry.json`（分类/标签/中文名/尺寸）、
`geometry.json`（包围盒/pivot）、mesh_meta 阶段采集的 socket（插槽磁吸）。
