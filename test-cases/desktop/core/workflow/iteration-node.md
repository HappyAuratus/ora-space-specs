# 迭代复合节点

迭代节点把同一冻结图中的一个 `parentId` 区域作为 foreach body。风险既包括运行时按轮持久化，
也包括编辑器不能让视觉几何悄悄改写执行结构。本文件登记跨编辑、快照与 Desktop 真实执行的核心义务。

## Iteration Membership Is Authored Only Through Region Entry Points

### 风险

若拖拽几何可以写入或移除 `parentId`，普通排版会改变运行语义；若三类新增入口分别维护连线，
Condition 分支 handle、历史边界或自动保存可能出现不一致。

### 前置状态

- 画布含一个迭代容器。
- Agent、Condition 声明支持 `iteration`，其他已开放节点只支持 `workflow`。

### 触发

作者从内部起点新增、在内部边插入、从普通输出或 Condition 分支追加节点；或尝试把外层节点拖入区域。

### 必须成立

- entry 创建带 `iteration-entry` source handle 的容器到成员边。
- edge 插入保留原 `sourceHandle`，并把一条边替换为 `source → new → target`。
- output 从指定节点/分支追加新成员；新 Agent 的 `interactive` 为 false。
- 拖拽只改变位置，不改变 `parentId`；越界外层节点回到起始位置。
- 一次插入只形成一次历史操作与一次自动保存脏标记。

### 禁止结果

- 内部起点持久化为运行时节点。
- 目录拖放在区域内生成没有 `parentId` 的重叠外层节点。
- Condition 插入后丢失分支 handle。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 能力声明只允许 Agent、Condition 进入区域 | `packages/workflow-mock/tests/capabilities.test.ts`；`workflow-flow/iteration-actions.test.tsx` | `Covered` |
| 三类插入统一维护成员、连线与分支 handle | `workflow-iteration-graph.test.ts` | `Covered` |
| 拖拽不改变身份，外层落入区域回退 | `workflow-iteration-containment.test.ts` | `Covered` |
| UI 新增只提交一次历史/自动保存 | 纯变换与 `WorkflowEditor.updateWorkflow` 单入口有单元覆盖，尚无调用次数渲染断言 | `Partial` |

### 决策依据

[迭代复合运行时决策 D7](../../../../decisions/desktop/core/workflow/20260915-iteration-composite-runtime.md#d7编辑器与运行视图的呈现承诺)。

## Iteration Frames Preserve Fitted Geometry Across Editing And Reload

### 风险

固定容器会裁切内部 DAG；把测量态当持久化事实会让折叠覆盖展开尺寸；把父子节点混排会造成外层重叠。

### 前置状态

- 区域包含一个或多个成员，旧快照可能没有 `initialWidth` / `initialHeight`。

### 触发

新增、插入、移动、删除或自动整理成员，折叠后展开，保存并重新加载快照。

### 必须成立

- 展开尺寸至少为 560×340；新增/移动只扩展，删除/整理紧凑计算。
- 折叠不覆盖持久化展开尺寸，重新展开恢复原尺寸。
- 每个内部 DAG 独立排布；外层按拟合后容器尺寸留位。
- 缺少尺寸的旧快照按最小尺寸加载，`parentId` 与 `iteration-entry` handle 保留。

### 禁止结果

- 持久化派生的 React Flow `extent`。
- 内外节点进入同一个拓扑布局队列。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 扩展、紧凑与折叠尺寸恢复 | `workflow-iteration-graph.test.ts` | `Covered` |
| 内外 DAG 分层布局并按容器真实宽度留位 | `workflow-flow/layout.test.ts` | `Covered` |
| 旧快照缺尺寸仍保留成员和 entry handle | `packages/workflow-runtime/src/graph-codec.test.ts` | `Covered` |

### 决策依据

同上决策 D7 的尺寸与布局规则。

## Deleting An Iteration Region Is Atomic And Repairable

### 风险

只删除容器会留下悬空成员和边；删除收集目标若静默改选会改变输出语义；多个底层删除事件若拆成多步，
撤销无法恢复完整区域。

### 前置状态

- 非空区域含成员、入口边与内部边；`collectSelector` 可指向其中一个成员。

### 触发

作者确认删除容器，或单独删除收集目标成员。

### 必须成立

- 确认文案显示成员数量。
- 容器、成员与相关边级联删除，并作为一个历史步骤完整撤销/重做。
- 删除收集目标时清空 `collectSelector` 并提示重新配置，不自动选择替代目标。

### 禁止结果

- 孤儿成员、悬空边或指向不存在节点的 selector 留在草稿中。
- 一次级联删除拆成多个用户可见历史步骤。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| selector 清理与容器紧凑 | `workflow-iteration-graph.test.ts` | `Covered` |
| 级联结果可由一次 undo 完整恢复 | `workflow-history.test.ts` | `Covered` |
| 成员数量确认与 React Flow 级联集合 | `workflow-editor.tsx` 行为实现，尚无完整对话框渲染断言 | `Partial` |

### 决策依据

同上决策 D7 的删除与引用修复规则。

## Editor Geometry And Entry Handles Do Not Change Runtime Semantics

### 风险

编辑器保存的尺寸字段或装饰性 entry handle 若被后端误当作新图契约，发布可能失败或轮次调度发生变化。

### 前置状态

- 图含持久化容器尺寸、`parentId`、`iteration-entry` source handle，以及 Condition → Agent 的多节点区域。
- Start 输入为三项数组。

### 触发

通过 Desktop 生产接口创建、发布并运行该图。

### 必须成立

- 图成功发布并执行三轮，Condition 每轮选择分支，Agent 每轮各获一个会话。
- 终端输出包含三项聚合结果；编辑器几何字段不参与调度。

### 禁止结果

- 把装饰性 entry handle 解析成额外运行时节点。
- 因存在 `initialWidth` / `initialHeight` 拒绝图或改变区域边界。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 带编辑器几何和 entry handle 的多节点区域发布、执行三轮并聚合 | `e2e/tests/desktop/core/session/tests/iteration.rs::editor_authored_multi_node_region_executes_three_rounds` | `Covered` |

### 决策依据

同上决策 D7；运行时区域仍只由 `parentId` 与普通边表达。
