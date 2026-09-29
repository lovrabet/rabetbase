# 流程定义设计态与运行态边界

`rabetbase flow` 管理 SmartCode 中的表单审批流与独立工作流定义；运行态待办和任务办理不属于本仓 CLI。

## rabetbase 负责

- 查询、导出和校验 FlowConfig。
- 创建或更新设计态流程定义。
- 发布流程定义。
- 通过 dataset、menu、BFF、notification 命令发现设计资源；通过 `flow runtime-user-search/runtime-role-list/runtime-role-user-list` 只读发现审批所需的运行态人员和角色。

## Runtime CLI 负责

- 查询当前用户待办和任务详情。
- 审批、拒绝和转交已有运行态任务。
- 验证流程实例、业务状态和任务权限。

自定义页面代码可以通过页面注入的 `@lovrabet/sdk` client 查询和办理 `INDEPENDENT_FLOW + CUSTOM_PAGE`。这属于生成页面的运行时代码契约，不表示 `rabetbase flow` 命令本身可以办理任务；页面接入规则见 [`custom-page-flow-sdk.md`](../guides/custom-page-flow-sdk.md)。

`rabetbase` 只调用已核实的运行态人员/角色只读接口来生成 FlowConfig，不查询或办理任务。需要办理待办时，显式交接到 Runtime CLI 仓库及其公开 Skill。

## 完成口径

- `flow validate` 成功只表示本地结构和已实现的拓扑规则通过。
- `flow create/update` 成功只表示服务端保存了设计态定义。
- `flow publish` 是同步服务端操作；只有成功响应才表示发布完成，dry-run 不表示发布成功。
- Runtime 最终行为仍需在 Runtime 环境中验证，不能用设计态查询代替。
