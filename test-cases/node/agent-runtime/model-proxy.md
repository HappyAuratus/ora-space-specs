# Node 个人模型会话：临时授权与终止

决策依据：[运行模型代理](../../../decisions/node/agent-runtime/20261010-run-scoped-model-proxy.md)。

## Model sessions must acquire scoped access without persisting tokens

- 风险：真实 Key 或临时令牌进入控制账本、checkout、JSONL 或成果。
- 前置：带模型绑定的执行，Node 已验证专用模型客户端证书。
- 触发：启动、续期、结束、Node 重启。
- 必须成立：授权目标固定；续期沿用令牌；配置只在会话运行目录和进程环境；结束撤销并清理；重启按既有中断规则收尾。
- 禁止结果：缺模型代理能力退回直连、WSS 证书替代客户端身份、后台续期脱离会话生命周期。
- 证据与义务：Desktop `model_proxy/tests` 的真实 mTLS、范围、续期/撤销及配置/清理测试；Node 协议兼容与能力拒绝测试。两协议的真实浏览器及 API 正常链路补充生产 CLI、无检测到的凭据材料的 Revision 和容器/卷释放，见 [M4 验收记录](../../../implementation/opencode-m4.md)。状态 `Partial`，全栈崩溃恢复仍缺直接证据；受控模型服务不替代 Bluezone 新 Key 验收。

## Model ending must preserve the durable end reason

- 风险：平台已经记录结束，但撤销先于结束命令到达，使 Node 错记为错误并丢弃成果。
- 前置：目标身份和当前绑定仍有效，会话已进入 ending。
- 触发：首轮授权或续期返回专用 `model_session_ending`。
- 必须成立：停止模型调用，在有限时间内等待已有持久结束命令，保留其结束原因并完成成果封装；未开始会话也可封装空记录。
- 禁止结果：把身份伪造/错误代次/其他撤销识别成正常 ending，或无限等待。
- 证据与义务：Core 的 ending 错误分类矩阵，Desktop `session/model` ending 竞态及未启动会话空 JSONL 封存测试。状态 `Partial`：分类与 Node 收尾有直接证据；初始授权已 ending 时的空记录 Revision 上传、登记及释放仍缺全栈直接证据。
