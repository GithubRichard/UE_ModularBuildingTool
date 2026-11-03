# UE5 地编模式：技术分析与交互设计

> 目标：给地编同学一个"模块库独立窗口 + 场景内快速拼装"的编辑模式。
> 核心能力：组件快速筛选、磁吸、对齐、选择；独立窗口内 3D 渲染、拖拽进场景。
> 范围：只做编辑器工具，不改运行时逻辑；数据源复用 ModularBuilding 现有注册表。

---

## 1. 总体架构

```
CrossStreetLevelTools（Editor 模块，C++）
│
├── 模块库窗口 Palette（SDockTab，可停靠/独立浮动）
│   ├── 筛选条：搜索框、分类、标签、尺寸、收藏
│   ├── 卡片网格：3D 缩略图（Content Browser 同款管线）+ 中文名 + 尺寸
│   ├── 3D 大预览：SEditorViewport + FPreviewScene，选中模块自动旋转展示
│   └── 拖拽源：FModuleDragDropOp（继承 FAssetDragDropOp，走编辑器原生放置通道）
│
├── 拼装模式 EdMode（EM_CrossStreetPlace）
│   ├── 幽灵预览：半透明 ghost actor，实时跟随鼠标
│   ├── 磁吸：网格 / 表面 / 插槽（registry sockets）/ 角度
│   └── 工具条 Toolkit：磁吸开关、对齐按钮、放置选项
│
├── 放置工厂 UCrossStreetPlaceFactory（继承 UActorFactoryStaticMesh）
│   └── 强制 scale=1、pivot 补偿（读 geometry.json）、打 module=<id> 标签
│
└── 数据层 FCrossStreetRegistry（编辑器启动时加载）
    └── registry.json + geometry.json：分类/标签/中文名/尺寸/pivot/socket
```

**为什么这样拆**：窗口、模式、工厂是 UE 编辑器三件套里正交的三块，可以分阶段交付、独立测试。数据层直接吃你们现在的 JSON，不用再导一遍。

---

## 2. 技术选型分析

### 2.1 独立窗口：Slate C++（推荐）vs UMG 编辑器工具控件

| | Slate C++ | UMG Editor Utility Widget |
|---|---|---|
| 3D 实时渲染 | ✅ SEditorViewport + FPreviewScene（Mesh 编辑器同款） | ❌ UMG 没有原生 3D 视口，只能放预渲染图 |
| 拖拽到场景视口 | ✅ FDragDropOperation 全 Slate 通用，可被视口接收 | ⚠️ UMG 的拖拽载荷视口不认，要另写 C++ 接收端 |
| 缩略图 | ✅ FAssetThumbnailPool（Content Browser 同款） | ⚠️ 需自己画 |
| 开发量 | 大（要会 Slate） | 小，但能力天花板低 |

**结论**：直接上 Slate C++。UMG 方案看似快，但"3D 渲染 + 拖进场景"这两条核心需求它都满足不了，最后还是要补 C++，不如一次到位。

### 2.2 拼装交互：经典 FEdMode（推荐）vs Interactive Tools Framework

- **经典 FEdMode**：`FEdMode` 子类 + `FEditorModeRegistry` 注册。API 稳定十年，文档和参考多（UE 自带 Place 模式就是这个路子），和你们现有的 C++ 风格更合拍。
- **Interactive Tools Framework**（UE5 新）：`UBaseEdMode` + `UClickDragInputBehavior`，更现代，但概念多、调试链长。

**结论**：FEdMode。等模式跑顺了，个别复杂笔刷再考虑 Tools Framework。

### 2.3 拖拽：继承 FAssetDragDropOp（关键技巧）

不要自己发明拖拽协议。让 `FModuleDragDropOp : public FAssetDragDropOp`，把 StaticMesh 的 `FAssetData` 装进它父类的 `Assets` 数组，再额外带一个 `ModuleId` 字段。好处：

- 拖进关卡视口时，**编辑器原生放置通道直接认**（和从 Content Browser 拖进来行为一致：自动找 ActorFactory、自动处理放置位置）。
- 你们的 `UCrossStreetPlaceFactory` 在 `PostSpawnActor` 里做三件事：强制 scale=1、按 registry pivot 做位置补偿、打 `module=<id>` 标签。之后"按模块选中"就靠这个标签。

### 2.4 3D 预览与缩略图

- **卡片网格**：`FAssetThumbnailPool`（256 常驻）+ `FAssetThumbnail`，225 个模块无压力，和 Content Browser 同一渲染管线，效果一致。
- **大预览窗**：`SEditorViewport` 子类 + `FPreviewScene`，放选中模块，Tick 里缓慢旋转；支持滚轮缩放、右键旋转（直接复用 `FPreviewScene` 的默认输入）。
- 备选：如果缩略图管线调不通，你们已有截图管线（capture stage），可以预渲染 225 张 PNG 当缩略图——效果差一档，但零风险。

### 2.5 磁吸数据源

你们 mesh_meta 阶段已经采了"墙面 Socket"，这就是插槽磁吸的数据。不用再采。EdMode 读 `FCrossStreetRegistry` 里的 socket 列表即可。角度磁吸和网格磁吸是纯数学，不依赖数据。

---

## 3. 交互设计

### 3.1 模块库窗口布局

```
┌──────────────────────────────────────────────────────────┐
│ 模块库                                         [钉住][x] │
├──────────────────────────────────────────────────────────┤
│ 🔍 [ 搜索: 窗…          ] [分类: 全部 ▾] [标签 ▾] [☆收藏] │
├────────┬─────────────────────────────────────────────────┤
│ 分类   │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐             │
│ 墙体12 │  │ 3D   │ │ 3D   │ │ 3D   │ │ 3D   │  ← 卡片网格  │
│ 窗户9  │  │ 缩略 │ │      │ │      │ │      │    点击选中   │
│ 门9    │  └──────┘ └──────┘ └──────┘ └──────┘    拖拽进场景 │
│ 招牌27 │   方窗300x300      圆窗300x300  …               │
│ 雨棚4  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐             │
│ 植物15 │  │      │ │      │ │      │ │      │             │
│ …     │  └──────┘ └──────┘ └──────┘ └──────┘             │
│        ├─────────────────────────────────────────────────┤
│        │ 3D 预览 [旋转中的大模型]   400×14×300cm          │
│        │ pivot: 底面居中   socket: 4                     │
│        │ [拖到场景] [☆收藏] [在场景中选中全部实例]         │
└────────┴─────────────────────────────────────────────────┘
```

- **筛选**：搜索框匹配中文名/模块 ID/标签（拼音首字母可二期）；分类树左侧，数字是计数；多标签交集过滤。目标：225 个模块，3 次点击内定位。
- **卡片**：3D 缩略图 + 中文名 + 尺寸一行。悬停浮起尺寸/pivot/socket 摘要。
- **卡片右键菜单**：在场景中选中全部实例 / 隔离显示 / 聚焦 / 收藏。
- **双击卡片**：选中场景中全部该模块实例并聚焦（frame）。
- **拖拽**：从卡片或从 3D 大预览区拖出，`FModuleDragDropOp`，拖到关卡视口松手即放置。

### 3.2 核心流程

**流程 A：筛选 → 拖放（最常用）**
1. `窗口 > 模块库` 打开（或工具栏按钮，可停靠在 Content Browser 旁）。
2. 搜索"圆窗" → 网格剩 3 张卡 → 拖"圆窗300x300"到关卡视口。
3. 视口内出现半透明 ghost，跟随鼠标；开着插槽磁吸时靠近墙体 socket 自动吸附变绿。
4. 松手 → Actor 生成（scale 强制 1，已按 pivot 补偿，标签 `module=Window_300x300_02`）。
5. 连续放置模式下可继续点放；`Esc` / 右键退出。

**流程 B：磁吸微调**
1. 激活拼装模式（工具栏 Mode 下拉）。
2. Toolkit 开关：`[网格] [表面] [插槽] [角度]`，插槽半径滑杆，网格步长 400/200/100/50。
3. 拖动已有 Actor：ghost 实时显示吸附结果——绿色=插槽吸附，蓝色=网格吸附，无色=自由。
4. `R` 旋转（按角度步长 90/45/15），`Shift+R` 反向。

**流程 C：对齐**
1. 多选 Actor（最后一个选中的为基准）。
2. Toolkit 对齐区：X `[左|中|右]`，Y `[左|中|右]`，Z `[底|中|顶]`；另有 `[沿街面对齐]`（把选中 Actor 的 +Y 前脸对到同一平面——你们街景最高频的操作）。
3. 以包围盒为准，一键执行，可撤销（走标准事务 `GEditor->BeginTransaction`）。

**流程 D：按模块选择**
1. 卡片右键 → 在场景中选中全部实例；或 Toolkit 里按分类多选下拉框。
2. 配合隔离显示（`只显示选中`），快速检查某一类构件的拼装质量。

### 3.3 快捷键（建议）

| 按键 | 动作 |
|---|---|
| `Shift+Space` 或工具栏 | 打开/聚焦模块库 |
| 拼装模式内 `R` / `Shift+R` | 旋转 ±步长 |
| 拼装模式内 `Esc` / 右键 | 退出放置 |
| `Ctrl+Alt+G` | 开关网格磁吸（原生已支持，可沿用） |
| Toolkit 按钮 | 对齐六向 + 沿街面对齐 |

---

## 4. 磁吸与对齐：行为规格

### 4.1 磁吸优先级（同时命中时）

`插槽 > 表面 > 网格`。ghost 颜色即状态：绿/蓝/白。阈值：插槽 200cm 内最近 socket；表面沿法线偏移模块半厚。

### 4.2 插槽磁吸细节

- 数据源：`FCrossStreetRegistry`（mesh_meta 采的 socket：位置 + 朝向）。
- 悬停 Actor 时，200cm 内所有 socket 显示为小菱形标记；ghost 吸附到最近 socket，位置+朝向一次对齐。
- 你们的 plan.py 已经写过 pivot 补偿数学，C++ 侧照搬（`geometry.json` 的 pivot norm 全套用上）。

### 4.3 对齐细节

- 基准：最后选中的 Actor（UE 惯例）；单选时禁用多选对齐，只留沿街面对齐。
- 沿街面对齐：取每个 Actor 包围盒 +Y 最大面，对齐到基准的前脸平面——拼街墙时一键拉齐。
- 全部走 `FScopedTransaction`，支持 Ctrl+Z。

---

## 5. 分阶段交付与工作量

假设一位有 UE 编辑器开发经验的工程师：

| 阶段 | 内容 | 工时 | 交付价值 |
|---|---|---|---|
| P1 | 模块库窗口（筛选/卡片/缩略图/3D 大预览）+ 拖拽（FAssetDragDropOp）+ 放置工厂（scale 锁/pivot/标签）+ 按模块选择 | 1.5–2 周 | **筛选+拖放立即可用**；磁吸先用 UE 原生网格吸附顶着 |
| P2 | EdMode + ghost + 插槽/表面/角度磁吸 + 对齐工具条（含沿街面对齐） | 2–3 周 | 完整拼装体验 |
| P3（可选） | 批量排布（阵列/沿线/沿街自动排布）+ recipe 联动（把 plan.json 一键摆进场景，即 build_in_editor.py 的 UI 版） | 1–2 周 | 从手工拼装过渡到半自动生成 |

**建议**：P1 先行，地编同学第二周就能用上；P2 并行打磨。P3 和你们的大模型 PCG 路线汇合（配方驱动生成）。

---

## 6. 风险与注意事项

1. **Slate 学习曲线**：团队没写过 Slate 的话，P1 前 3 天基本在搭架子。备选是先用 UMG 做个只能看不能拖的"模块浏览器"过渡——但前面分析过，UMG 满足不了拖拽+3D，迟早要重写。
2. **UE 版本差异**：5.0–5.6 的 `FEdMode`/`SEditorViewport`/`FAssetThumbnail` API 基本稳定；Interactive Tools Framework 变化较大——这也是选经典 FEdMode 的原因之一。
3. **缩略图**：225 个模块常驻池没问题；如果个别模块缩略图渲染异常（你们有些是占位材质）， fallback 到预渲染 PNG。
4. **中文搜索**：一期做子串匹配（中文名/ID/标签）足够；拼音首字母二期再加。
5. **数据同步**：registry.json 更新后，编辑器重启自动重载（或加一个"刷新注册表"按钮，避免重启）。
6. **和现有管线不打架**：`build_in_editor.py` 的 Remote Execution 流程保持不动；地编模式是"手工快拼"通道，recipe 是"批量生成"通道，两者生成的 Actor 都用 `module=<id>` 标签，互操作。
7. **不要重做撤销**：所有修改场景的操作走 `FScopedTransaction`，白嫖编辑器撤销栈。
