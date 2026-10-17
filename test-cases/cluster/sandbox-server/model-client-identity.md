# 模型访问证书：用途和部署隔离

决策依据：[专用模型客户端身份](../../../decisions/cluster/sandbox-server/20261010-model-client-identity-replaces-shared-model-key-injection.md)。

## Model client identities must remain distinct from Node server identities

- 风险：扩大 WSS 证书用途，陈旧或错误实例取得模型访问权。
- 前置：Sandbox Server 已取得当前租户、Workspace、代次的启动许可。
- 触发：启用或禁用模型代理配置后创建 Node。
- 必须成立：仅启用时签发独立 clientAuth 证书和独立私钥；身份固定许可目标；入口脚本放入管理 tmpfs；无原始模型 Key 注入。
- 禁止结果：复用 WSS 私钥、客户端与服务端用途互换、材料进入恢复日志或 effect 结果。
- 证据与义务：`sandbox-server/internal/management/model_identity_test.go` 的真实证书解析及独立密钥/用途/范围断言；`sandbox-server/internal/lifecycle/model_environment_test.go` 的 `TestSecuredEnsurePreservesNodeEnvAndIsolatesModelIdentity` 断言只有启用模型代理时才注入 `ORA_MODEL_*`，模型身份与 Node 身份不同且范围精确，正常路径下材料不出现在 Node 配置、effect 响应（含重放）、状态目录与日志中；Desktop 与 model-gateway mTLS 拒绝矩阵。状态 `Partial`：签发、用途边界与正常路径不泄露有直接证据，ensure 失败路径下材料不进入 journal/effect 仍待直接测试。

## M4 deployment must preserve encryption identity and isolate acceptance resources

- 风险：重启更换主密钥导致历史凭据不可读，验收覆盖现有 Echo 栈。
- 前置：已有数据库、主密钥卷和 Echo 环境。
- 触发：重复启动、损坏或丢失主密钥、执行 M4 验收。
- 必须成立：复用加密身份；旧密文无法解密时启动失败；独立项目/端口/网络/标签；浏览器无真实密钥持久存储。
- 禁止结果：静默重置密钥、挂载 Gateway 签名私钥给模型服务、验收重建 Echo 的 `local` 镜像。
- 证据与义务：model-gateway 加密身份校验测试；bootstrap 脚本和隔离参数拒绝检查；[独立两协议实际链路证据](https://github.com/ora-space/cluster/pull/15#issuecomment-6107672919)复核同 Run 的容器/卷释放。状态 `Partial`，浏览器正常路径已有直接证据，主密钥卷灾备恢复演练和页面已有真实默认模型的供应商验证仍待补齐。
