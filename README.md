# UE_ModularBuildingTool

UE 5.7 地编（关卡编辑）辅助工具：为模块化建筑 Kit（CrossStreet，225 模块）提供
**模块库独立窗口 + 场景内快速拼装模式**——组件快速筛选、磁吸、对齐、选择，
独立 3D 窗口拖拽进场景。

> 当前阶段：方案待选，两个设计方案并列，均未在引擎中验证。代码实现尚未开始。

## 设计方案

- **方案 1：拼装模式** —
  [技术分析与交互设计](docs/ue5-level-editor-mode-design.md)，
  包含总体架构、技术选型（Slate C++、编辑器模式基类、拖拽通道方案）、
  交互流程、磁吸与对齐行为规格、数据契约、分阶段交付计划、风险提示。
- **方案 2：模块库 + 免切换放置** —
  [设计方案](docs/design-option-2-modeless-placement.md)
  （[表格版](docs/design-option-2-plan.xlsx)），
  不用切换模式，吸附优先靠模型包围盒计算，插槽只作可选增强。

## 其他文档

- [方案 1 交互效果图](docs/ue5-editor-mode-mockup.html) —
  两张高保真 mockup（用浏览器打开）：图 1 模块库独立窗口，
  图 2 拼装模式视口 + 磁吸/对齐工具条。待修改项见方案 1 第 8 节。

## 数据来源

方案 2 的字段要求见其“注册数据”一节。方案 1 复用 `ModularBuilding` 现有注册表：
`data/kits/crossstreet/registry.json`（分类/标签/中文名/尺寸）、
`geometry.json`（包围盒/pivot）、mesh_meta 阶段采集的 socket（插槽磁吸）。
字段要求和待确认项见方案 1 第 5、9 节。
