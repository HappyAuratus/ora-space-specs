# 模型访问证书：用途和部署隔离

决策依据：[专用模型客户端身份](../../../decisions/cluster/sandbox-server/20261010-model-client-identity-replaces-shared-model-key-injection.md)。

## Model client identities must remain distinct from Node server identities

- 风险：扩大 WSS 证书用途，陈旧或错误实例取得模型访问权。
- 前置：Sandbox Server 已取得当前租户、Workspace、代次的启动许可。
- 触发：启用或禁用模型代理配置后创建 Node。
- 必须成立：仅启用时签发独立 clientAuth 证书和独立私钥；身份固定许可目标；入口脚本放入管理 tmpfs；无原始模型 Key 注入。
- 禁止结果：复用 WSS 私钥、客户端与服务端用途互换、材料进入恢复日志或 effect 结果。
- 证据与义务：`sandbox-server/internal/management/model_identity_test.go` 的真实证书解析及独立密钥/用途/范围断言；Desktop 与 model-gateway mTLS 拒绝矩阵。状态 `Partial`：签发及用途边界有直接证据，材料不进入 journal/effect 的单独故障路径仍待直接测试。

## M4 deployment must preserve encryption identity and isolate acceptance resources

- 风险：重启更换主密钥导致历史凭据不可读，验收覆盖现有 Echo 栈。
- 前置：已有数据库、主密钥卷和 Echo 环境。
- 触发：重复启动、损坏或丢失主密钥、执行 M4 验收。
- 必须成立：复用加密身份；旧密文无法解密时启动失败；独立项目/端口/网络/标签；浏览器无真实密钥持久存储。
- 禁止结果：静默重置密钥、挂载 Gateway 签名私钥给模型服务、验收重建 Echo 的 `local` 镜像。
- 证据与义务：model-gateway 加密身份校验测试；bootstrap 脚本和隔离参数拒绝检查；独立 WSL 栈真实 CLI 的两协议验收。状态 `Partial`，主密钥卷灾备恢复演练和真实 Bluezone/完整浏览器证据仍待补齐。
