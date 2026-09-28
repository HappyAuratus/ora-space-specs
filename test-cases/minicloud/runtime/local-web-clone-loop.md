# minicloud 本机 Web 闭环验收

依据[根 ADR](../../../decisions/minicloud/runtime/0-local-web-client-embedded-controller.md)与
[入口迁移 ADR](../../../decisions/minicloud/runtime/20260921-controller-hosted-server.md)。本机形态已按
[minicloud 退役决策](../../../decisions/minicloud/runtime/20260928-retire-minicloud.md)删除：client、启动器、
Vite proxy 链路与 Controller 的过渡 JSON 接口都不再存在，原用例（本机 proxy 链路、HTTP 回复丢失、轮询展示、
server 重启、内嵌所有权、启动器关停）随被测对象一起移除。

仍然成立的下层承诺由各自的核心用例承担，不因入口删除而弱化：

- 持久接受、幂等、冲突与接管后确认：[持久协调双适配器](../../controller/persistence/coordination-store-adapters.md)。
- 组合校验、独占所有者、托管 Node 就绪、崩溃与关停顺序：
  [Controller 组合与托管 Node 生命周期](../../controller/lifecycle/executable-composition-and-single-node.md)。
- 可执行入口不监听、本机受理只经内嵌句柄：[Controller API 边界](../../controller/api-boundary/controller-hosted-client-api.md)。
- 开发与发布权限分离：`ora-utils` 的 `private_path`／`trusted_path` 测试，与 minicloud 无关，继续有效。
