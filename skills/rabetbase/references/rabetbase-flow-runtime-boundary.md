# 流程定义设计态与运行态边界

`rabetbase flow` 管理 SmartCode 中的表单审批流与独立工作流定义；运行态待办和任务办理不属于本仓 CLI。

## rabetbase 负责

- 查询、导出和校验 FlowConfig。
- 创建或更新设计态流程定义。
- 发布流程定义。
- 通过 `flow page-list` 发现表单绑定页面，通过 menu 核对入口，通过 dataset、BFF、notification 命令发现业务资源；通过 `flow runtime-user-search/runtime-role-list/runtime-role-user-list` 只读发现审批所需的运行态人员和角色。

## Runtime CLI 负责

- 查询当前用户待办和任务详情。
- 审批、拒绝和转交已有运行态任务。
- 通过实际待办、任务详情和办理结果验证当前用户的任务行为。完整实例状态、页面发起和业务数据验收使用对应 Runtime 页面或已核实的 API，不假设 CLI 有未提供的命令。

自定义页面代码可以通过页面注入的 `@lovrabet/sdk` client 查询和办理 `INDEPENDENT_FLOW + CUSTOM_PAGE`。这属于生成页面的运行时代码契约，不表示 `rabetbase flow` 命令本身可以办理任务；页面接入规则见 [`custom-page-flow-sdk.md`](../guides/custom-page-flow-sdk.md)。

`rabetbase` 只调用已核实的运行态人员/角色只读接口来生成 FlowConfig，不查询或办理任务。需要办理待办时，显式交接到 Runtime CLI 仓库及其公开 Skill。

`FORM_FLOW` 按 [表单发起指南](../guides/form-flow-start.md) 使用实际绑定页面的 Cookie 会话，先保存记录，再携带实际 `pageId/dataId` 显式发起；`datasetCode` 仅作为启动接口无法解析主标准表单时的已确认兜底参数。后端同步 `processInstanceId` 与业务记录的 PID 绑定，页面不得另用普通 update 写 PID；状态按 PID 查询真实流程，PID 存在不代表已完成。`INDEPENDENT_FLOW` / `CUSTOM_PAGE` 使用支持无参数 `client.flow()` 的页面 SDK，合同不变；不要求先创建数据集记录。项目 SDK 未提供该能力时先解决 SDK 能力缺失，不生成无法运行的调用。

## 完成口径

- `flow validate` 成功只表示本地结构和已实现的拓扑规则通过。
- `flow create/update` 成功只表示服务端保存了设计态定义。
- `flow publish` 是同步服务端操作；只有成功响应才表示发布完成，dry-run 不表示发布成功。
- 发布后回读 `flow detail` 核对 `flowStatus`、完整绑定及流程编码；当前详情不公开 `enabled`，启用状态由平台或实际运行入口核验，不把字段缺失当作已启用。发布不会自动启用已停用流程，也不保证绑定页面、菜单、审批人和脚本在运行时可用。
- 修改已发布流程会回到草稿，需要重新发布。曾经发布的流程不能修改 `flowType`，回到草稿或停用也不解除该限制。
- Runtime 最终行为仍需在 Runtime 环境中验证，不能用设计态查询代替。
