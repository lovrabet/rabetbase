# 流程设计态开发工作流

本工作流用于在 `rabetbase` 中创建、校验和管理表单审批流与独立工作流定义。它只覆盖设计态流程，不办理运行态待办。

执行前按当前步骤阅读：

- 命令参数与风险：[flow 命令参考](../references/rabetbase-flow-commands.md)
- 人员、角色、脚本和通知配置发现：[审批流资源发现](../references/rabetbase-flow-resources.md)
- FlowConfig 字段与拓扑规则：[FlowConfig 契约](../knowledge/approval-flow/config-json-schema.md)
- 设计态与运行态边界：[审批流职责边界](../references/rabetbase-flow-runtime-boundary.md)

## 流程类型判定

先确定外层 `flowType`，再设计节点；不要根据是否出现审批节点反推流程类型。

| `flowType` | 产品语义 | 人工节点 | 页面与数据边界 |
|---|---|---|---|
| `FORM_FLOW` | 表单审批流 | 只允许 `taskMode: APPROVAL`，禁止 `HANDLE` | 绑定 `datasetCode/pageId`；表单填写发生在发起前，驳回修改走重新提交 |
| `INDEPENDENT_FLOW` | 独立工作流 | 可同时使用 `APPROVAL` 和 `HANDLE` | 不绑定 `datasetCode/pageId`；业务数据由工作流页面和 SDK 管理 |

独立工作流即使包含审批节点仍然是 `INDEPENDENT_FLOW`。需要填写、补充材料、执行采购或确认收货等人工办理步骤时，必须使用独立工作流，不能把 `HANDLE` 塞进表单审批流。

独立工作流的服务端语义包含两种 `flowJson.pageMode`：

- `CUSTOM_PAGE`：业务自定义页面；缺省时按此模式处理。
- `PLATFORM_FORM`：服务端平台表单页面语义；当前公开 Skill 和 CLI 不生成或提交该模式。

用户要求 `PLATFORM_FORM` 时，直接返回 `NEEDS_DSL_EXTENSION`，说明当前版本只支持业务自定义页面；不要继续生成 FlowConfig，也不要执行 create/update。

`pageMode` 不适用于 `FORM_FLOW`。自定义页面 SDK/OpenAPI 只查询和操作 `INDEPENDENT_FLOW + CUSTOM_PAGE`。

`INDEPENDENT_FLOW + CUSTOM_PAGE` 可以选择性配置：

- `flowJson.startPageId`：进入发起页面时使用。
- 人工节点 `pageId`：待办、已办或任务详情进入当前节点页面时使用。

这两个字段只是导航元数据，不是业务字段，不参与分支、权限、办理人或流程状态计算；不配置时由业务应用自行选择页面。

## 标准流程

1. 明确 `flowName`、业务描述、业务状态、办理步骤和通过/拒绝路径，并按上表显式选择 `flowType`。
2. `FORM_FLOW` 配置外层 `datasetCode/pageId`；`INDEPENDENT_FLOW` 不配置这两个表单绑定字段，并使用显式或默认的 `pageMode: "CUSTOM_PAGE"`。用户要求 `PLATFORM_FORM` 时返回 `NEEDS_DSL_EXTENSION`。自定义页面可按需配置 `flowJson.startPageId` 和人工节点 `pageId`，自行查询和修改业务数据集，并按业务状态决定展示内容。
3. 建立业务变量契约：逐项确认变量名、类型、初始来源、更新节点、消费节点以及是否用于待办/已办查询；区分顶层 `variables` 与 `formData`，禁止把 `approval_sys_*`、`approved` 当作业务变量。
4. 根据业务逻辑直接生成完整的 `nodes + edges` 拓扑，不使用 `templateName` 或 SmartCode 模板表。
5. 为每个人工节点确认 `taskMode`、`approvalMode`、`assignee.strategy` 和是否 `sequential`。决策用 `APPROVAL`；仅 `INDEPENDENT_FLOW` 可用 `HANDLE` 表达填写或确认数据后继续。办理完成后需要按业务变量分支时，按下方“HANDLE 后续路由”组合节点。
6. 逐个人工节点确认可选配置：自定义页面 `pageId`、转交候选人 `transferCandidates`、超时动作 `timeout`、节点完成后默认抄送 `cc`；用户未提出时保持不配置，不自行推断人员、页面或超时时长。
7. 固定办理人、转交候选人和默认抄送人使用 `flow runtime-user-search` 查询运行态 `userId`；角色审批使用 `flow runtime-role-list` 查询运行态 `roleId`，必要时用 `flow runtime-role-user-list` 核对成员。不要把 `role *` 查询出的配置端人员/角色写入 FlowConfig。
8. `SCRIPT` 节点和 `timeout.action: "EXECUTE_BFF"` 使用 `bff list --type ENDPOINT` 选择真实 `functionName`；同时确认同步/异步方式、重试配置和独立的 `resultVariable`。不要让多个脚本依赖默认的 `scriptResult`。
9. `NOTIFICATION` 节点使用 `notification config-list --all` 选择真实 `configCode`，并确认 `failurePolicy`、模板内容、变量占位符和收件人来源。
10. 生成最小 FlowConfig。不要把服务端 id、`appCode`、部署状态、编译上下文、`templateName`、`formSchema/formDefinitions/formPermissions` 或节点 `form/formPermission` 写入文件。
11. 执行 `rabetbase flow validate --file <path> --format compress`。
12. 展示变量契约、拓扑、资源绑定、节点可选配置、warnings 和校验结果。只有 `valid=true` 才能进入远程写入。
13. 用户确认后先执行 create/update 的 `--dry-run`，再正式提交。
14. publish 属于 `high-risk-write`，先 `--dry-run`，正式执行必须显式 `--yes`。

## 业务变量契约

流程定义不保存发起请求的实际业务数据，但 CONDITION、SCRIPT、NOTIFICATION 和运行态查询都依赖稳定的变量名称。生成拓扑前至少整理下列信息：

| 项目 | 说明 |
|---|---|
| 名称与类型 | 例如 `amount: number`、`businessStatus: string`、`purchase: object` |
| 初始来源 | 发起请求的顶层 `variables`，或 `formData` 内字段 |
| 更新者 | 发起页面、某个 HANDLE/APPROVAL 办理请求、SCRIPT 结果 |
| 消费者 | CONDITION 表达式、SCRIPT、NOTIFICATION 或业务页面 |
| 查询用途 | 用于待办/已办变量查询时，只能使用非系统的字符串、数字或布尔值做等值条件 |

同一份数据在不同位置的访问方式不同：

| 数据来源 | CONDITION | SCRIPT 入参 | NOTIFICATION |
|---|---|---|---|
| 顶层 `variables.amount` | `${amount > 10000}` | `params.amount` | `${variables.amount}` |
| `formData.amount` | `${formData.amount > 10000}` | `params.formData.amount` | `${variables.formData.amount}` |

- `variables` 办理参数更新顶层流程变量；`formPatch` 增量增加、修改或删除 `formData` 字段。
- 对象和数组可以作为流程变量供页面、脚本或通知读取，但当前待办/已办变量查询只支持字符串、数字和布尔值等值条件；需要查询的业务状态应单独保存为顶层标量变量。
- `approval_sys_*`、`approved` 和平台维护的 `formData` 容器名属于保留范围，不作为业务变量名或 SCRIPT `resultVariable`。

## HANDLE 后续路由

- `HANDLE` 只表示“填写或确认数据后继续”，自身必须且只能保留一条 `ALWAYS` 出边，不承担业务分支选择。
- 自定义页面办理任务时，可以在办理请求中的 `variables` 更新业务变量，也可以用 `formPatch` 增量更新 `formData`；任务完成后，下游节点读取更新后的值。
- 路由变量已由页面准备好时，生成 `HANDLE → CONDITION`，由条件网关表达多条业务路径。
- 需要 Backend Function 先计算、校验或归一化路由变量时，生成 `HANDLE → SCRIPT → CONDITION`；不需要计算时不要为了分支强制插入 `SCRIPT`。
- 不要给 `HANDLE` 直接配置多条出边，也不要让调用方传 `targetNodeId` 绕过流程图。办理节点存在多个业务结果不应判定为 `NEEDS_DSL_EXTENSION`；只有下游节点也无法表达需求时才进入 DSL 扩展判断。

## 候选确认

- 不编造 `userId`、`roleId`、`datasetCode`、`pageId`、`scriptName` 或 `configCode`。
- 多个候选必须展示稳定标识并请用户选择。
- 唯一命中可作为推荐项，但人员身份、角色、拓扑和通知渠道仍需用户确认。
- 无命中时继续询问关键词，不自动替换为相似资源。
- 当前 DSL 支持有向环路；无法表达并行网关、子流程、加签、委派、动态审批人或流程配置式自定义表单时，停止生成可提交配置。

## 状态

| 状态 | 含义 |
|---|---|
| `READY` | FlowConfig 完整且本地校验通过，可以等待用户决定是否提交 |
| `NEEDS_USER_CONFIRMATION` | 资源候选、身份、拓扑或写操作尚未确认 |
| `NEEDS_DSL_EXTENSION` | 需求超出当前简单审批 DSL |
| `INVALID_CONFIG` | 本地结构或拓扑校验失败 |
| `REMOTE_VALIDATION_FAILED` | 服务端拒绝 create、update 或 publish |

`READY` 只表示本地配置可提交，不表示已经创建、发布或启用。远程结果必须以对应命令的服务端响应为准。

## 本地文件与提交

推荐目录：

```text
.rabetbase/flows/<flow-name>.json
```

旧文件先执行：

```bash
rabetbase flow validate --file .rabetbase/flows/purchase.json
rabetbase flow validate --file .rabetbase/flows/purchase.json --write-normalized
```

`--write-normalized` 会原子替换原文件。warnings 中列出的旧字段会被删除；为服务端编译临时注入的上下文只存在于内存，不会写回文件。

远程提交顺序：

```bash
rabetbase flow create --file .rabetbase/flows/purchase.json --dry-run --format compress
rabetbase flow create --file .rabetbase/flows/purchase.json --format compress

rabetbase flow update --id 42 --file .rabetbase/flows/purchase.json --dry-run --format compress
rabetbase flow update --id 42 --file .rabetbase/flows/purchase.json --format compress

rabetbase flow publish --id 42 --dry-run --format compress
rabetbase flow publish --id 42 --yes --format compress
```

不要根据本地文件中的遗留 `id` 或 `appCode` 决定提交目标。流程 ID 来自命令参数，应用编码来自当前工作区或显式 CLI app 选择。
