# 交互节点自动重试的交付边界

## Only The Final Attempt Can Authorize Successors

### 风险

旧发送的输出与完成请求在自动重试后仍能让后继使用无效输入。

### 前置状态与触发

交互式 Agent 节点有一个仅在它成功后启动的后继。分别在两条非首轮人工回合路径中触发静默取消，原请求确认取消后，自动重试仅返回 `EndTurn`。

- 普通完成路径使用 `completionSignal: end_turn` 且未声明输出契约。原发送和取消排空期间都产生文本。停靠后从旧页面重复提交普通完成请求，再独立验证二次确认的强制完成。
- 自动接受路径使用 `completionSignal: agent` 并开启 `autoAcceptAgentCompletion`。确认服务端已接受原尝试的完成请求。最后一次发送不发信号，等待宿主处理自动接受，再检查后继启动次数。
- 恢复路径从同一会话提交新的人工回合，按冻结契约交付有效结果。
- 兼容路径加载升级前没有重发边界的历史，分别读取已停靠和已完成的节点，对比原 turn key、最终文本、状态及审计。

### 必须成立

- 旧页面重复请求每次均被后端拒绝，自动接受也无法推进，后继的实际启动次数为零。卡片和详情均显示节点运行失败，原因与原会话重试入口在刷新后仍可见。
- 当前尝试的有效纯工具交付按冻结契约验收。新旧尝试的工具或消息 ID 相同时仍按各自窗口取值。
- 只有再次确认的 force 可覆盖门禁，审计保存空重试原因，提交输出只来自最终尝试。后续回合正常恢复时保留原失败记录。
- 旧历史按原回合窗口取最终文本。打开或重放旧记录后，已完成节点保持原终态，已有审计逐项一致。

### 禁止结果

取消前或取消排空期间的文本被提交、旧 token 授权新回合、Pending 工具被当作真实完成、根据耗时或聊天文案直接判失败。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 持久化边界、复用 ID 与真实工具状态 | [历史组装测试](https://github.com/ora-space/desktop/blob/8d67184e1625dc2e711af06c2dbed004d81ed479/crates/history/src/assembler_tests.rs#L560-L673) 验证重发边界、同 ID 增量路由和未完成状态。[完成快照测试](https://github.com/ora-space/desktop/blob/8d67184e1625dc2e711af06c2dbed004d81ed479/crates/backend/src/workflow/run/completion_snapshot.rs#L635-L712) 验证最终尝试内真实工具结果可用，并排除有归属歧义的完成状态。 | Covered |
| 后端拒绝普通及自动推进，后继未启动，成功恢复保留审计 | [工作流端到端测试](https://github.com/ora-space/desktop/blob/8d67184e1625dc2e711af06c2dbed004d81ed479/e2e/tests/desktop/core/session/tests/workflow_agent_completion.rs#L720-L802) 覆盖普通完成与 agent 自动接受，断言旧信号失效、后继未启动、原会话交付有效结果后成功且原失败记录保留。 | Covered |
| 旧页面重复完成被拒绝且后继启动次数为零 | [工作流端到端测试的重复完成入口](https://github.com/ora-space/desktop/blob/8d67184e1625dc2e711af06c2dbed004d81ed479/e2e/tests/desktop/core/session/tests/workflow_agent_completion.rs#L761-L780) 两次提交均返回 `WorkflowNodeIncomplete`，并检查后继节点没有启动时间。 | Covered |
| 无重发边界的旧历史沿用原窗口，已完成节点及审计保持原值 | 缺少升级前历史夹具，尚未直接验证读回原 turn key、最终文本、已完成状态及审计。 | Missing |
| 强制采用空重试的产出与独立原因 | [强制完成端到端测试](https://github.com/ora-space/desktop/blob/8d67184e1625dc2e711af06c2dbed004d81ed479/e2e/tests/desktop/core/session/tests/workflow_agent_completion.rs#L804-L848) 验证完成后输出为空，审计保存 `retry_without_result`。 | Covered |
| 刷新或重启后展示失败和恢复入口 | [运行卡片测试](https://github.com/ora-space/desktop/blob/8d67184e1625dc2e711af06c2dbed004d81ed479/packages/app-shell/src/features/workflow-run/run-theater-act-card.test.tsx#L560-L590) 和[会话面板测试](https://github.com/ora-space/desktop/blob/8d67184e1625dc2e711af06c2dbed004d81ed479/packages/app-shell/src/features/workflow-run/run-node-session-chat.test.tsx#L1274-L1305) 验证失败文案和重试入口。[后端重启测试](https://github.com/ora-space/desktop/blob/8d67184e1625dc2e711af06c2dbed004d81ed479/e2e/tests/desktop/core/session/tests/workflow_agent_completion.rs#L823-L833) 验证异常停靠和 blocker 仍可读。缺少刷新或重启后从界面验证原因与恢复入口的端到端证据。 | Partial |

### 决策依据

[交互式节点按最终尝试的有效产出恢复](../../../../decisions/desktop/core/workflow/20261118-interactive-retry-output.md)
