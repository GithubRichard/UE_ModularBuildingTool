# UE_ModularBuildingTool

UE 5.7 地编（关卡编辑）辅助工具：为模块化建筑 Kit（CrossStreet，225 模块）提供
**模块库独立窗口 + 场景内快速拼装模式**——组件快速筛选、磁吸、对齐、选择，
独立 3D 窗口拖拽进场景。

> 当前阶段：方案待选，三个设计方案并列，均未在引擎中验证。代码实现尚未开始。

## 设计方案

- **方案 1：拼装模式** —
  [技术分析与交互设计](docs/ue5-level-editor-mode-design.md)，
  包含总体架构、技术选型（Slate C++、编辑器模式基类、拖拽通道方案）、
  交互流程、磁吸与对齐行为规格、数据契约、分阶段交付计划、风险提示。
- **方案 2：模块库 + 免切换放置** —
  [设计方案](docs/design-option-2-modeless-placement.md)
  （[表格版](docs/design-option-2-plan.xlsx)），
  不用切换模式，吸附优先靠模型包围盒计算，插槽只作可选增强。
- **方案 3：窗口会话式地编** —
  [设计方案](docs/design-option-3-window-session.md)，
  会话寄生在模块库窗口「地编」Tab 上，UE 模式全程不变；
  视口 overlay 只接管左键与快捷键；状态可见性三件套；
  吸附与数据策略沿用方案 2，拖拽通道与 P0 验证沿用方案 1 v2。

## 其他文档

- [交互效果图（已按方案 3 重画）](docs/ue5-editor-mode-mockup.html) —
  两张高保真 mockup（用浏览器打开）：图 1 模块库「模块库 / 地编」双 Tab 窗口，
  图 2 「地编」Tab 会话激活状态（视口 overlay 底部状态条，工具栏保持「选择」模式）。

## 可操作交互原型

- [地编组件预览窗口 · 交互原型](docs/ue5-modular-preview-window-interactive.html)
  （下载 HTML 后用现代浏览器打开）。提供侧栏停靠与浮动窗口两种布局，可操作搜索、
  拖放、独立模型预览、已有模块移动、混合选择、安装点保持换款、撤销及辅助暂停。
- 本原型采用“窗口打开即启用辅助，吸附开关独立恢复”的交互方式，供与方案 3 的
  Tab 会话方式对比。PIE／不兼容模式暂停辅助，关闭窗口取消未提交预览。
- 模型、场景和 UE 状态均为浏览器内示意，不连接 Unreal Editor，不代表插件已实现。

## 数据来源

方案 2 的字段要求见其“注册数据”一节。方案 1 复用 `ModularBuilding` 现有注册表：
`data/kits/crossstreet/registry.json`（分类/标签/中文名/尺寸）、
`geometry.json`（包围盒/pivot）、mesh_meta 阶段采集的 socket（插槽磁吸）。
字段要求和待确认项见方案 1 第 5、9 节。
