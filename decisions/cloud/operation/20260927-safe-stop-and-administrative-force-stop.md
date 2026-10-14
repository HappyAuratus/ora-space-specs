---
status: implemented
---

# 普通停止保护活动，管理员强停登记独立意图并等待目标真实终止

普通 stop/restart/delete 不打断活动或未知执行。强停是独立的管理员意图，关闭目标新准入，终止
目标全部工作并保留持久数据；不自动重启、接管或删卷。仅改变 Cloud 状态不能证明已停止。

本决策的后端状态机与 Controller/Sandbox Server 路径已实现。强停确认 UI、生命周期竞争和
未确认终止等完整故障演练由[后续 ADR](../execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md)
承接，核心证据见[停止用例](../../../test-cases/cloud/operation/safe-stop-and-administrative-force-stop.md)。
旧 `administrative_stop` 仍是有活动保护的停止，不静默改为强停。

## 继承与修改

继承[运行时生命周期](0-workspace-runtime-follows-desktop-node.md)、
[具体分支](20260926-require-concrete-default-branch.md)和 effect 先计划后派发的规则：独立 clone、
start 不重新 clone、确认终止后才换代或删除数据不变。普通生命周期保持项目串行；强停单独登记
责任，不靠删除在途 operation 腾出槽位。前端完成条件拆到后续 ADR。

## 决策

### D1：普通生命周期检查使用权限、当前会话和活动

创建者或管理员还需持有有效 session，目标无活动 ticket、未结束执行或未知责任。关闭准入后
验证 idle，不能在确认与 terminate 间接受新活动。restart 是一个持久 operation，确认停止后
启动新代次、挂载原数据，不由客户端拼接两请求。main 不单独删除；项目删除由管理员原子检查
全部运行时，无占用旁路或部分删除。

### D2：强停是独立授权和可审计责任

接口要求当前租户管理员、目标 version、幂等键、非空 reason 和 `impactConfirmed=true`。
不要求先抢普通操作权。事务记录发起者、目标代次、sandbox targets 与原执行关联，并关闭准入、
撤回普通资格。之后的终止是系统责任，不因发起人离开或浏览器关闭而丢失。

同键同输入返回原意图，同键不同输入冲突；新的并行意图在存在未完成强停时拒绝。历史 actor、
输入、执行身份及真实结果保留，服务授权不依赖 UI 弹窗。

### D3：在途 create/clone/plugin 也能登记强停

强停与普通领取分开。目标停止派发新普通副作用，但原 operation/execution/effect 不被删除或
伪装成功。固定目标包括尚在分配的 sandbox；Controller 使用当前许可终止这些目标，执行端
持久 tombstone 防止迟到 ensure 重建。其它运行时的普通操作不因本次强停被中断。

### D4：未确认终止保持责任，确认后只结算目标

持久状态为 registered、terminating、succeeded；响应丢失或不可达时保留未完成意图并对账，
不是一过期限就成功。每个目标须同时确认 terminated 与 lateEnsureFenced，全部目标确认后
才关闭对应 Node/sandbox 和执行责任，保留实际完成结果；被打断普通 operation 记录
`administrative_force_stop` 及意图引用。未确认前不得交接、启动新代次或删除数据。

### D5：进程与持久数据分别处理

PostgreSQL 保留业务及审计引用；Workspace 卷保存 Git、已保存文件和 Node 账本。普通停止和
强停移除执行实例但保留卷；重新启动使用原数据，显式 data delete 才删卷。内存、未保存编辑
和磁盘损坏后的数据不承诺无损恢复，跨宿主迁移/灾备另立决策。

## 与 approved 决策的偏离

approved 的四态“已请求、终止中、待对账、已停止”映射为持久三态 `registered`、
`terminating`、`succeeded`。`registered` 表示意图已保存，`terminating` 同时包含正在终止
及结果未知、继续对账；只有全部 sandbox targets 的真实终止及迟到 ensure 围栏得到确认
才进入 `succeeded`。待对账不是独立的第四个持久状态，也不由超时转成成功。

该表示用未完成意图及逐目标确认保存恢复责任，避免状态名与目标证据分别成为完成权威。
四态中的安全语义不变：未知时仍禁止交接、启动或删卷，UI 必须区分接受与终止完成。

## 不变量

1. 普通生命周期不能绕过当前会话和活动保护。
2. 强停按一个稳定意图终止明确目标，不改写其它运行时责任。
3. 在途动作保留身份，迟到结果不能恢复准入或旧 sandbox。
4. 未确认终止不能交接、换代或删卷。
5. 停止/强停保留已保存数据，显式删除才清理。
6. 已接受终止责任不因用户、权限或协调进程变化丢失。
