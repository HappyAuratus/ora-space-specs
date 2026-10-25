# 工作流整次成果与执行授权：评审草案

依据[整次成果提案](../../../decisions/cloud/workflow/20261025-run-owned-git-delivery-and-history.md)。
本文与 proposed ADR 一起用于审阅用户要求的验证范围，尚不是批准后的系统承诺；批准时必须按最终结论重新核对。
全部新增义务为 Missing。既有 Agent 交付测试不能证明 WorkflowRun 归属、多节点历史或任务混合接力。

## Workflow Revision Keeps Its Real Owner

- **风险：** 把 WorkflowRun 或节点 ID 塞进 Agent run 外键；重复回执登记多份成果；独立运行必须伪造任务。
- **前置状态：** 同租户有 Agent IssueRun、任务调用 WorkflowRun、独立 WorkflowRun，并各自持有 Workspace。
- **触发：** 分别交付，重放成功回执，提交跨运行、跨租户和错误 Workspace 的声明。
- **必须成立：** 正确归属各登记一次，工作流调用经 invocation 查询成果；独立工作流无需任务；错归属整笔拒绝。
- **禁止结果：** 多份逻辑 Revision、外键失效、工作流节点冒充 IssueRun、以调用方 UUID 自报决定归属。
- **验证义务与证据：** PostgreSQL 生产结算入口与约束测试，覆盖并发同结果、相反迟到结果和事务回滚。**Missing**。
- **决策依据：** D1，不变量 1、4。

## Only The Current Node Attempt Can Write Its Workspace

- **风险：** 旧尝试、取消后的会话或其他 Project 的执行仍可推送；目录暴露被误当授权。
- **前置状态：** 多节点工作流先后执行，至少一个节点失败并自动重试，另有同项目和异项目 Workspace。
- **触发：** 使用真实执行身份读 upstream/他人分支，推本工作区/他人分支/交付 ref；结束、取消、换代次后重试。
- **必须成立：** 合法会话可读本项目，写自身前缀仅快进；旧尝试、结束、取消、旧代次拒绝；交付有独立窄资格。
- **禁止结果：** 以 WorkflowRun 仍 running 代替节点尝试校验；会话创建/覆盖交付 ref；借目录进行跨租户访问。
- **验证义务与证据：** Git Server 真实读写及 Controller 授权接口，核对被拒 ref 完全不变；目录固定输入与历史元数据测试。**Missing**。
- **决策依据：** D2、D3，不变量 2、7。

## Run History Includes Every Established Session Attempt

- **风险：** 只归档最后一个节点，漏掉失败尝试；清单与对象不对应；伪造 not_started 掩盖丢历史。
- **前置状态：** 工作流有两个 Agent 节点，第一个有两次尝试，包含真实建立会话及未建立的启动失败。
- **触发：** 整次交付；注入缺历史、重复 execution ID、跨 checkout、摘要/大小不匹配和上传故障。
- **必须成立：** 每个建立的会话都有原样历史及唯一来源；未建立明确标记；所有对象和 ref 核验后才提交成果及回执。
- **禁止结果：** 用最后一段 JSONL 代表整次运行；跳过失败尝试；外部未知被写为成功；错误归档后删除工作区。
- **验证义务与证据：** Node 真实历史归档、对象存储核验、PG 结算和故障重放；旧 Agent JSONL 兼容读取。**Missing**。
- **决策依据：** D4，不变量 3、4。

## Restore Runs Once And Delivery Waits For All Writers

- **风险：** 后一节点重新恢复前序覆盖本次改动；最后节点结束但前一尝试进程还活着就保存。
- **前置状态：** 前序 Revision 有文件 A；工作流节点一新增 B，节点二读取 B 并新增 C，期间存在自动重试。
- **触发：** 恢复、节点推进、重启调度器/Node、恢复中取消、最后进入交付。
- **必须成立：** 进入第一个实际会话前恢复 A；后续节点仍看见 B；全部写入者停止后最终 Revision 含 A/B/C。
- **禁止结果：** 半恢复 checkout 运行 Agent；用最后 session ended 替代整次停止证据；后续重试恢复旧基线。
- **验证义务与证据：** 真实 Git、受控进程与节点故障；核对恢复次数、会话开始先后、最终提交和各工作区身份。**Missing**。
- **决策依据：** D2、D5，不变量 2、5。

## Execution Delivery And Release Remain Separate Facts

- **风险：** 图成功就释放；没有配置被当无改动；取消后丢部分成果；残留未知却允许后继启动。
- **前置状态：** 有后继请求的任务工作流，分别处于成功、失败、取消、交付失败和释放未知状态；另有无 Workspace 的纯计算图。
- **触发：** 注入 Git 拒绝、上传失败、Node 断线、迟到结果，重启后显式恢复或放弃未保存成果。
- **必须成立：** 三类结果分别可查，正常先保存后释放；故障保持屏障；纯计算记 not_required 并保留原代码基线。
- **禁止结果：** abandoned 等于 released、skipped 等于 saved、自动回退旧 Revision、无授权清理或隐式解除失败暂停。
- **验证义务与证据：** 真实服务故障矩阵，核对交付 ref、Revision、存储对象、进程停止和物理 Workspace 释放；无 Workspace 路径不得伪造历史。**Missing**。
- **决策依据：** D1、D5，不变量 4、6。
