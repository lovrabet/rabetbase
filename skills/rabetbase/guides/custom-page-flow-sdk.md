# 自定义页面 Flow SDK

本指南约束 rabetbase 生成或修改的自定义页面如何通过页面运行时注入的 `@lovrabet/sdk` client，发起、查询和办理独立自定义页面工作流。

## 适用范围

- 流程定义必须是 `INDEPENDENT_FLOW + CUSTOM_PAGE`。
- 页面承担流程发起、当前用户待办/已办/已发起查询、详情、审批、驳回、办理、转交、撤回、重提、退回、作废或抄送交互。
- 不适用于 `FORM_FLOW`、`PLATFORM_FORM`，也不用于页面直接设计或发布 FlowConfig。

流程定义和发布仍由 `rabetbase flow validate/create/update/publish` 完成。本指南只描述生成到页面中的运行时调用。

## 身份、应用与权限边界

自定义页面从页面上下文获得已经初始化的 SDK client：

```jsx
import { useSdkClient } from "@/context/app-context";

const client = useSdkClient();
const flow = client.flow();
```

遵守以下约束：

1. 页面使用当前 Runtime 登录用户的 Cookie 会话。
2. `appCode` 来自页面注入的 client。页面业务方法不接收 `appCode`，不要把它放进 `variables` 或 `formData`。
3. `client.flow()` 不传 `operatorUserId`。不要在页面中保存或传递 AccessKey、Cookie、OpenAPI Token 或时间戳。
4. SDK 自动把 `appCode` 放到普通 Flow POST 和自定义页面 POST 的 JSON body；`getReturnTargets()`、`markCcRead()` 所需的 query 参数也由 SDK 处理。
5. Runtime 最终校验当前用户、租户、应用边界、任务办理人、流程状态和应用管理员权限。页面不能通过自定义参数扩大权限。
6. `canHandle`、`canCancel`、`canWithdraw`、`canResubmit`、`canVoid`、`canReturn` 只用于控制页面交互，不能代替服务端校验。

如果项目安装的 `@lovrabet/sdk` 没有无参数 `client.flow()` 或相关类型，停止生成调用并说明需要升级 SDK；不要回退到旧的 `client.flow(operatorUserId)` 或 Flow OpenAPI 示例。

## 开发前确认

1. 使用 `rabetbase flow list` 找到目标流程，以 `flow detail` 确认 `flowType=INDEPENDENT_FLOW`、`pageMode=CUSTOM_PAGE`、流程编码和发布状态。
2. 明确当前页面是发起页、办理页、详情页、终态页还是工作台；导航用的 `startPath` / `APPROVAL.path` / `END.path` 必须来自已发布页面的完整 `runtimePageUrl`，但不参与权限和流程状态计算。
3. 建立业务变量契约：变量名、类型、初始来源、更新节点、消费节点和是否用于列表查询。
4. 对动态表单确认 `formDataVersion` 的来源和刷新时机。
5. 对应用范围列表确认当前用户确实需要且具备应用管理员权限。

不要编造 `flowCode`、`taskId`、`processInstanceId`、节点 ID 或用户 ID。流程和页面资源从当前 CLI 查询结果取得，运行时 ID 从 SDK 返回值取得。

## 数据位置

| 字段 | 用途 | 示例 |
| --- | --- | --- |
| `formData` | 发起时提交完整业务表单 | 采购主题、分类、数量、预算、供应商、原因 |
| `formPatch` | 办理或重提时按顶层字段合并修改 `formData` | 补充供应商、最终金额、附件 |
| `variables` | 流程分支、精确查询和业务状态同步 | `businessId`、`departmentCode`、`purchaseStatus` |
| `variableKeys` | 限制查询响应中的 `businessVariables` | `["businessId", "purchaseStatus"]` |

- 待办和已办变量查询只使用普通业务标量做等值条件。
- 对象和数组可以作为流程变量传递，但不作为待办/已办精确查询条件。
- 不写入、删除或依赖 `approval_sys_*`、`approved` 等系统变量。
- 当前任务完成后，下一任务可能属于其他用户。不要用当前用户的 `listTodo()` 推断下一办理人。

`approve()`、`reject()`、`complete()` 和 `resubmit()` 都可以通过 `formPatch` 修改普通非动态自定义页面的表单数据。合并只发生在 `formData` 根级：未传字段保持原值，传入字段替换原值，`null` 明确设为空值，嵌套对象和数组整字段替换，不做深合并。普通表单不要求动态 schema，也不强制传 `formDataVersion`；动态表单遵守已发布 schema 的字段权限和版本校验，办理请求只提交 Runtime 返回的最新 `formDataVersion`，不传 schema，也不用客户端自造的业务版本代替。

`approve()`、`reject()`、`complete()` 的兼容调用可以只通过 `variables.formData` 提交完整表单数据；新页面优先使用 `formPatch` 表达局部修改。这三个方法的同一次请求不能同时传 `formPatch` 和 `variables.formData`，Runtime 会拒绝这种无法确定覆盖顺序的请求。`resubmit()` 不兼容 `variables.formData`，重提表单只能使用 `formPatch`。顶层 `variables` 中除 `formData` 外的字段仍按业务变量处理。

## 方法选择

### 查询

| 页面意图 | SDK 方法 | 参数 | 返回值 |
| --- | --- | --- | --- |
| 当前用户待办 | `listTodo(query?)` | `FlowVariableQuery` | `FlowPage<FlowCustomPageTaskItem>` |
| 应用范围待办 | `listAppTodo(query?)` | `FlowVariableQuery` | `FlowPage<FlowCustomPageTaskItem>` |
| 当前用户已办 | `listDone(query?)` | `FlowVariableQuery` | `FlowPage<FlowCustomPageTaskItem>` |
| 应用范围已办 | `listAppDone(query?)` | `FlowVariableQuery` | `FlowPage<FlowCustomPageTaskItem>` |
| 当前用户已发起 | `listSubmitted(query?)` | `FlowSubmittedQuery` | `FlowPage<FlowCustomPageSubmittedProcess>` |
| 应用范围已发起 | `listAppSubmitted(query?)` | `FlowSubmittedQuery` | `FlowPage<FlowCustomPageSubmittedProcess>` |
| 可发起流程 | `listIndependentFlows(query?)` | `FlowDefinitionQuery` | `FlowPage<FlowCustomPageDefinition>` |
| 任务详情 | `getTaskDetail(taskId, variableKeys?)` | `string, string[]?` | `FlowCustomPageTaskDetail` |
| 流程/当前任务详情 | `getProcessDetail(processInstanceId, variableKeys?)` | `string, string[]?` | `FlowCustomPageTaskDetail` |
| 当前任务别名 | `getCurrentTask(processInstanceId, variableKeys?)` | `string, string[]?` | `FlowCustomPageTaskDetail` |
| 时间线 | `getTimeline(processInstanceId)` | `string` | `FlowCustomPageTimeline` |
| 审批记录 | `getRecords(processInstanceId)` | `string` | `FlowRecords` |
| 可退回节点 | `getReturnTargets(taskId)` | `string` | `FlowReturnTarget[]` |
| 抄送列表 | `listCc(query?)` | `FlowCcQuery` | `FlowPage<FlowCcRecord>` |
| 抄送详情 | `getCcDetail(ccRecordId)` | `number \| string` | `FlowCustomPageCcDetail` |

`listAppTodo()`、`listAppDone()`、`listAppSubmitted()` 只用于应用管理员工作台。普通用户页面使用非 `App` 方法。

待办、已办和任务详情中的 `path` 是流程定义中当前 `APPROVAL` 节点配置的完整运行态页面地址，类型为 `string`；页面按该地址导航。不要再从这些自定义页面任务响应中读取节点 `pageId`。可发起流程定义中的 `startPath` 是发起页的完整运行态地址；`END.path` 是流程定义中的终态导航元数据，不是任务路径。设计态 Skill 只允许使用 `page custom-detail` 确认为 `FORMAL` 的页面所返回的 `runtimePageUrl`。

任务、任务详情和已发起列表中的 `detailUrl` 由 Runtime 使用相应节点 `path` 与流程实例 ID 生成，查询参数名固定为 `processId`。当 `path` 不含 query 时追加 `?processId=...`，已有 `?` 时追加 `&processId=...`；流程运行中使用当前 `APPROVAL.path`，流程正常结束后使用实际到达的 `END.path`。`path` 或流程实例 ID 缺失时 `detailUrl` 为空。

## 返回对象字段语义与页面展示

本节是页面生成 Agent 必须使用的固定响应契约，字段来自 Runtime Java DTO、查询投影 Mapper 和状态枚举。不能只根据 TypeScript 字段名自行猜测含义，也不能把整个响应对象直接 `JSON.stringify` 后当成正式页面。

通用规则：

- 所有 `*Time` 都是毫秒时间戳；为空表示事件尚未发生或没有可展示时间。页面格式化为本地日期时间，不直接展示原始数字。
- `taskId`、`processInstanceId`、`flowCode`、`nodeKey` 等 ID 用于查询、导航、React `key` 和提交动作；默认不作为面向业务用户的主标题。
- 名称字段优先展示；名称为空时可回退到对应 ID，但要使用弱化样式，不能编造名称。
- `path` 是节点配置的运行态页面基地址；`detailUrl` 是 Runtime 已追加 `processId` 的可导航详情地址。存在 `detailUrl` 时直接使用，不要再次拼接 query。
- `businessVariables` 只包含调用时通过 `variableKeys` 请求的业务变量；未请求、变量不存在或被系统字段过滤时返回空对象。空对象不表示流程没有业务变量。
- `formData` 是流程保存的业务表单数据，应按当前业务字段契约渲染；不要把未知对象直接展开成面向终端用户的原始 JSON。
- `can*` 字段是当前登录用户在当前服务端状态下的按钮权限事实。按钮显隐和禁用以它们为准，不能在前端根据用户 ID、角色或状态字符串重新推导权限。
- 列表、详情、时间线和记录可能因权限或历史数据缺少可选字段。页面必须对 `null`、空字符串、空对象和空数组提供稳定空态。

### 通用分页 `FlowPage<T>`

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `records` | 当前页记录；没有数据时为空数组 | 表格或卡片列表的数据源 |
| `currentPage` | 当前页码，从 1 开始 | 分页器当前页 |
| `pageSize` | 当前每页条数 | 分页器 page size |
| `totalCount` | 满足条件的总记录数 | 分页器 total 和统计文案 |
| `totalPages` | 总页数，由 `totalCount / pageSize` 向上取整 | 判断是否还有下一页；不要用当前记录数代替 |

### 可发起流程 `FlowCustomPageDefinition`

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `flowCode` | 服务端生成的稳定流程编码 | 调用 `start()` 的必填值，不把 `flowName` 当编码 |
| `flowName` | 流程展示名称 | 发起入口卡片标题、下拉选项 label |
| `flowDesc` | 流程说明 | 卡片说明；为空时不渲染占位段落 |
| `version` | 当前可发起的已发布流程版本 | 可作为辅助版本信息，不参与前端选路 |
| `pageMode` | 页面模式；本入口固定面向 `CUSTOM_PAGE` | 通常不展示，用于诊断契约是否匹配 |
| `startPath` | 已配置的完整发起页运行态地址 | “发起”按钮导航目标；为空时不能猜 URL，应使用当前页面内置发起表单或显示未配置 |

### 发起结果 `FlowCustomPageStartResponse`

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `flowCode` | 实际发起的流程编码 | 成功结果核对或埋点 |
| `processInstanceId` | 新流程实例 ID；幂等重放时是原实例 ID | 后续详情、时间线、记录和流程级动作的主键 |
| `idempotentReplay` | 本次请求是否命中了相同 `idempotencyKey` 的既有结果 | `true` 表示复用了原实例，不是新建了第二个实例，也不是失败 |

发起成功后用 `processInstanceId` 读取服务端详情。不要从 `idempotentReplay=false` 推导流程已经到达哪个节点。

### 待办/已办列表项 `FlowCustomPageTaskItem`

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `taskId` | 当前或历史人工任务 ID | 打开任务详情、同意、拒绝、办理、转交和退回时使用 |
| `taskName` | 人工任务展示名称 | 列表主标题或“当前环节”列 |
| `taskDefinitionKey` | 流程定义中的节点 ID | 稳定定位节点；不代替 `taskName` 展示 |
| `taskMode` | `APPROVAL` 或 `HANDLE` | `APPROVAL` 显示同意/拒绝；`HANDLE` 显示办理完成，不混用动作 |
| `path` | 当前 `APPROVAL` 节点配置的完整运行态页面地址 | 仅作页面基地址；打开具体实例优先使用 `detailUrl` |
| `detailUrl` | Runtime 基于 `path` 和 `processInstanceId` 生成的详情地址 | 行点击或“查看详情”导航；为空时留在当前页面并按 ID 查询详情 |
| `assignee` | 实际办理人用户 ID；候选任务可能为空 | 动作参数和诊断用途，默认不直接展示 |
| `assigneeName` | 实际办理人名称；用户解析失败时可为空 | “办理人”列；为空可回退 `assignee` |
| `handleMode` | 当前用户办理授权来源：`ASSIGNEE`、`CANDIDATE` 或 `APP_ADMIN`；已办或不可办理时可为空 | 可作为管理员代办提示；不能代替服务端权限校验 |
| `transferCandidates` | 当前节点允许转交的候选用户 | 只在数组非空且任务可办理时展示转交入口 |
| `processInstanceId` | 所属流程实例 ID | 读取流程详情、时间线和记录 |
| `flowCode` | 所属流程编码 | 筛选、辅助信息和埋点 |
| `flowName` | 所属流程名称 | 列表中的流程名称 |
| `initiatorUserId` | 发起人用户 ID | 稳定标识或筛选，不优先直接展示 |
| `initiatorUsername` | 发起人展示名称 | “发起人”列；为空时回退 `initiatorUserId` |
| `taskStatus` | 当前任务生命周期状态 | 状态标签；不能与 `processStatus` 混为一个状态 |
| `processStatus` | 整个流程实例状态 | 流程状态标签；拒绝结论需结合时间线的 `approvalResult` 判断 |
| `createTime` | 任务创建时间 | 待办到达时间或已办开始时间 |
| `endTime` | 任务结束时间；待办通常为空 | 已办完成时间；为空不显示“已完成” |
| `processStartTime` | 流程发起时间 | 列表的“发起时间” |
| `businessVariables` | 按 `variableKeys` 投影出的业务变量 | 展示业务单号、金额、部门等稳定摘要字段 |

`transferCandidates` 的元素字段固定为：

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `userId` | 候选用户 ID | `transfer()` / `batchTransfer()` 的 `targetUserId` |
| `userName` | 候选用户名称；解析失败时为空 | 转交选择器 label；为空时回退 `userId` |

### 已发起列表项 `FlowCustomPageSubmittedProcess`

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `processInstanceId` | 流程实例 ID | 详情、时间线、记录、撤回、重提、撤销和作废的主键 |
| `flowCode` | 流程编码 | 筛选或辅助信息 |
| `flowName` | 流程名称 | 列表主标题 |
| `startTime` | 流程发起时间 | “发起时间”列 |
| `endTime` | 流程结束时间；运行中为空 | “完成时间”列 |
| `status` | 流程实例状态，语义同 `processStatus` | 状态标签 |
| `cancelReason` | 已结束且被撤销时的删除/撤销原因；其他情况通常为空 | 仅在 `CANCELLED` 等终止态按需展示 |
| `detailUrl` | 当前审批节点或实际到达 `END` 节点的详情地址 | “查看详情”导航；为空时按 `processInstanceId` 在当前页面读取详情 |
| `currentNodeNames` | 当前运行节点名称的去重快捷列表 | 简洁“当前环节”文案；详细展示使用 `currentNodes` |
| `currentNodes` | 当前真实运行节点摘要 | 并行节点、当前审批人和节点类型展示 |
| `nextNodes` | 根据已发布流程配置推演的下一节点摘要 | 只能作为“预计下一步”；条件分支或运行变量变化时不保证最终一定到达 |
| `businessVariables` | 按 `variableKeys` 投影出的业务变量 | 列表业务摘要 |

`currentNodes` / `nextNodes` 的元素字段固定为：

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `nodeKey` | 流程节点 ID | React key、诊断和节点定位 |
| `nodeName` | 节点展示名称 | 环节名称 |
| `nodeType` | 节点类型，例如 `APPROVAL`、`SCRIPT`、`NOTIFICATION`、`END` | 图标或节点类别标签 |
| `status` | 当前节点固定为 `RUNNING`，预计下一节点固定为 `PENDING` | 节点状态标签 |
| `approvers` | 审批节点的实际或预计审批人；非审批节点为空数组 | 审批人头像/名称组；预计值不能表述为已分配事实 |

### 任务/流程详情 `FlowCustomPageTaskDetail`

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `taskId` | 当前或历史任务 ID；已结束流程按实例查询时可能为空 | 任务级动作只在非空且 `canHandle=true` 时使用 |
| `taskName` | 当前或历史任务名称 | 详情页当前环节标题 |
| `taskDefinitionKey` | 当前任务对应的流程节点 ID | 节点定位，不代替展示名称 |
| `taskMode` | `APPROVAL` 或 `HANDLE` | 决定展示同意/拒绝还是办理完成 |
| `path` | 当前任务节点的完整运行态页面基地址 | 导航元数据；优先使用 `detailUrl` |
| `detailUrl` | 已带 `processId` 的完整详情地址 | 分享当前应用内导航或从工作台跳转，不重复拼接 |
| `assignee` | 当前或历史实际办理人 ID | 动作和诊断用途 |
| `assigneeName` | 当前或历史实际办理人名称 | 办理人展示 |
| `processInstanceId` | 流程实例 ID | 所有流程级查询和动作的主键 |
| `flowCode` | 流程编码 | 辅助信息 |
| `flowName` | 流程名称 | 页面标题 |
| `initiatorUserId` | 发起人 ID | 稳定标识 |
| `initiatorUsername` | 发起人名称 | 发起人展示 |
| `taskCreateTime` | 当前/历史任务创建时间 | 当前环节开始时间 |
| `taskEndTime` | 当前/历史任务结束时间；运行中为空 | 当前环节完成时间 |
| `processStartTime` | 流程开始时间 | 发起时间 |
| `processEndTime` | 流程结束时间；运行中为空 | 流程完成时间 |
| `taskStatus` | 任务状态 | 当前环节状态标签 |
| `processStatus` | 流程实例状态 | 页面总状态标签 |
| `returnReason` | 撤回或退回发起人的原因；其他状态为空 | 非空时按 `processStatus` 显示“撤回原因”或“退回原因”，不能用 `cancelReason` 代替 |
| `canHandle` | 当前用户是否可办理当前任务 | 所有任务动作的总开关 |
| `handleMode` | 当前用户以办理人、候选人或应用管理员身份办理 | 管理员代办提示；为空表示当前不可办理 |
| `canCancel` | 当前用户是否可撤销并结束运行中流程 | 控制“撤销”按钮；与“撤回”不同 |
| `canWithdraw` | 当前用户是否可撤回流程并保留后续重提能力 | 控制“撤回”按钮 |
| `canResubmit` | 当前用户是否可重新提交已撤回或退回的流程 | 控制“重新提交”按钮 |
| `canVoid` | 当前用户是否可作废流程 | 控制高风险“作废”按钮 |
| `canReturn` | 当前任务是否可退回历史审批节点 | 控制“退回”按钮，并且必须使用 `returnTargets` |
| `returnTargets` | 当前允许退回的历史节点 | 退回选择器；不能传列表外的节点 ID |
| `approvalRound` | 当前审批轮次，从 1 开始；撤回/退回后重提可能增加 | 可展示“第 N 轮”，不要当流程版本 |
| `transferCandidates` | 当前节点允许转交的人 | 转交选择器 |
| `formData` | 当前流程保存的完整业务表单数据 | 详情主体或办理表单初始值；按业务 schema 展示 |
| `businessVariables` | 按 `variableKeys` 投影出的业务变量 | 业务状态、单号和分支上下文摘要 |
| `timeline` | 与 `getTimeline()` 相同结构的流程轨迹 | 时间线或流程图区域；为空时显示独立空态 |

`returnTargets` 的元素只有 `nodeId` 和 `nodeName`：页面显示 `nodeName`，提交 `returnTask()` 时使用 `nodeId`。自定义页面详情当前不返回 `formDataVersion`；如果动态表单办理要求乐观锁版本，必须通过相应 Runtime/SDK 契约读取最新版本，不能把客户端业务版本、`approvalRound`、流程版本或时间戳冒充为 `formDataVersion`。流程挂起时 `canHandle=false`；`canResubmit` 和 `canVoid` 仍分别按当前用户及流程状态的服务端权限返回，页面不要因为 `canHandle=false` 自行覆盖这些流程级权限。

### 时间线 `FlowCustomPageTimeline`

生成时间线 UI 时，必须继续阅读 [时间线展示指南](custom-page-flow-timeline-display.md)，按“节点 → 办理人状态 → 操作记录”的纵向结构生成页面。中文页面必须显示“提交人 / 处理人 / 处理结果 / 处理说明 / 处理时间”等字段标题，不能只把姓名、状态和时间串起来。下表解释字段，展示指南规定标签、排序、文案、颜色、转签、多次执行和展开方式。

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `processInstanceId` | 流程实例 ID | 时间线归属和后续刷新 |
| `flowCode` | 流程编码 | 辅助信息 |
| `flowName` | 流程名称 | 时间线标题 |
| `startUserId` | 发起人 ID | 稳定标识 |
| `startUserName` | 发起人名称；解析失败时可为空 | 发起节点展示 |
| `startTime` | 流程开始时间 | 发起节点时间 |
| `endTime` | 流程结束时间；运行中为空 | 终态时间 |
| `status` | 流程实例状态 | 总状态标签 |
| `cancelReason` | 撤销/终止原因；非撤销场景通常为空 | 终止说明 |
| `returnReason` | 撤回或退回发起人的原因；其他状态为空 | 非空时按 `status` 显示“撤回原因”或“退回原因”，不能用 `cancelReason` 代替 |
| `canCancel` | 当前用户是否可撤销流程 | 控制撤销按钮 |
| `canWithdraw` | 当前用户是否可撤回流程 | 控制撤回按钮 |
| `canResubmit` | 当前用户是否可重新提交 | 控制重提按钮 |
| `canVoid` | 当前用户是否可作废流程 | 控制作废按钮 |
| `approvalRound` | 当前审批轮次，从 1 开始 | 时间线轮次提示 |
| `steps` | 按流程顺序组织的节点轨迹 | 纵向时间线；同一节点重复执行时按 `occurrence` 区分 |
| `flowDiagram` | 发布版本流程图及本实例执行状态 | 可视化流程图；为空时回退到 `steps`，不要自行从时间线猜连线 |

`steps[]` 字段：

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `order` | 当前轨迹项顺序 | 排序；不要用数组原始位置替代持久语义 |
| `nodeKey` | BPMN/FlowConfig 节点 ID | 节点定位 |
| `nodeName` | 节点名称 | 时间线标题 |
| `occurrence` | 同一节点在本实例中的第几次执行，从 1 开始 | 区分重复执行；`nodeName` 已可能带“（第二次）”，直接使用原名，不重复追加次数；会签/或签的多个任务仍属于同一次执行 |
| `nodeType` | `START`、`APPROVAL`、`CONDITION`、`SCRIPT`、`NOTIFICATION`、虚拟 `RESUBMIT` 或 `END` | 选择图标和展示模板；`RESUBMIT` 不属于原 BPMN 图 |
| `approvalMode` | 审批模式，例如单人、或签、会签的服务端值；非审批节点可为空 | 审批节点辅助说明 |
| `sequential` | 多人审批是否串行；不适用时为空 | 会签/或签展示方式，不用于前端计算流程结果 |
| `startTime` | 节点最早开始时间；未执行为空 | 节点开始时间 |
| `endTime` | 节点完成时间；运行中或未执行为空 | 节点结束时间 |
| `status` | 节点执行状态 | 节点状态标签 |
| `approvalResult` | 节点业务结论；未得出结论时为空 | 审批/结束结论标签；判断拒绝结果应优先看实际到达 `END` 的该字段 |
| `tasks` | 此节点内的实际任务、历史任务或未来计划办理人 | 展开办理人和意见；非人工节点可为空数组 |

`steps[].tasks[]` 字段：

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `taskId` | Flowable 任务 ID；未来计划任务尚未创建时为空 | 仅真实当前任务可作为任务动作 ID；不能对计划任务提交动作 |
| `assignee` | 实际办理人 ID；未来节点可来自配置预解析 | 稳定标识 |
| `assigneeName` | 办理人名称；解析失败时为空 | 办理人展示 |
| `startTime` | 任务开始时间；计划任务为空 | 时间线时间 |
| `endTime` | 任务结束时间；运行中或计划任务为空 | 完成时间 |
| `status` | 任务状态 | 任务状态标签 |
| `approvalResult` | 任务结论；运行中、未创建或未得出结论时为空；自动跳过任务可继承节点结论 | 先判断任务 `status`，`SKIPPED + APPROVED` 仍显示“已跳过”，不能显示该人员已同意 |
| `cancelReason` | 流程撤销原因；仅撤销事件任务使用 | 撤销说明 |
| `comments` | 该任务的审批意见、系统动作和表单变更 | 意见列表，按 `time` 展示 |

`comments[]` 字段：

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `userId` | 操作人 ID | 稳定标识 |
| `name` | 操作人名称 | 操作人展示 |
| `type` | 固定动作码 | 图标/颜色映射和诊断；展示文案优先用 `typeName` |
| `typeName` | Runtime 给出的可展示动作名称 | 直接作为“通过”“转签”“重新提交”等动作标题 |
| `fullMessage` | 审批意见或系统动作说明 | 意见正文；为空时不渲染空气泡 |
| `time` | 动作时间 | 意见时间 |
| `fieldChanges` | `FORM_UPDATE` 对应的字段变更；其他动作通常为空 | 字段修改 diff，不替代 `fullMessage` |
| `nodeId` | 表单修改来源节点 ID | 审计定位，默认不作为正文 |
| `formRef` | 表单定义引用 | 审计/调试元数据 |
| `dataPath` | 表单数据命名空间 | 审计/调试元数据 |
| `formDataVersion` | 该次表单修改对应的数据版本 | 审计展示；不能作为当前办理请求的最新版本 |

`fieldChanges[]` 字段固定为 `fieldKey`、`fieldLabel`、`oldValue`、`newValue`。页面优先显示 `fieldLabel`，为空时回退 `fieldKey`；旧值和新值要按业务字段类型格式化，不能一律转成 `[object Object]`。

`flowDiagram` 字段：

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `schemaVersion` | 流程图配置结构版本 | 诊断信息，通常不展示 |
| `flowKey` | 流程图业务键 | 图实例标识 |
| `flowName` | 流程图名称 | 图标题 |
| `version` | 本实例绑定的历史发布版本 | 可展示版本；旧实例不能改用最新版本的图 |
| `nodes` | 全量流程节点及本实例状态 | 流程图节点数据源 |
| `edges` | 全量流程连线及是否实际经过 | 流程图边数据源 |

`flowDiagram.nodes[]` 包含 `id`、`name`、`type`、`result`、`status`、`approvalResult`、`startTime`、`endTime`、`x`、`y`。其中 `result` 是 `END` 节点配置的预期结果，`approvalResult` 是本实例实际轨迹结论；`x/y` 是设计态坐标，缺失时由页面布局算法兜底。

`flowDiagram.edges[]` 包含 `id`、`source`、`target`、`name`、`conditionType`、`expression`、`defaultFlow`、`executed`。`executed=true` 表示本实例已经走过该连线；条件表达式只作流程图辅助信息，页面不能在前端重新执行表达式决定流程状态。

### 平铺审批记录 `FlowRecords`

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `processInstanceId` | 流程实例 ID | 记录归属 |
| `flowName` | 流程名称 | 记录区标题 |
| `status` | 流程实例状态 | 总状态标签 |
| `canCancel` | 当前用户是否可撤销流程 | 控制撤销按钮 |
| `records` | 按节点顺序展开的审批记录 | 审批记录表格；适合表格，不等同于完整流程图 |

`records[]` 字段：

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `nodeName` | 节点名称 | 表格“环节”列 |
| `occurrence` | 同一节点第几次执行，从 1 开始 | 重复流转轮次 |
| `taskId` | 任务 ID；提交、流程结束等系统记录可为空 | 当前运行任务动作定位 |
| `assignees` | 此记录关联的办理人列表 | 办理人名称组 |
| `status` | 记录状态；已发生动作为 `COMPLETED`，待办可为 `PENDING` / `RUNNING` | 状态标签 |
| `canHandle` | 当前登录用户是否可办理这条运行中任务 | 行级办理按钮总开关 |
| `transferCandidates` | 当前节点允许转交的人 | 行级转交选择器 |
| `userId` | 实际评论/操作人 ID | 稳定标识 |
| `name` | 实际评论/操作人名称 | 操作人展示 |
| `type` | 动作码；系统记录可为 `SUBMIT` / `PROCESS_END` | 图标或颜色映射 |
| `typeName` | 可展示动作名称 | 动作列文案 |
| `fullMessage` | 评论或系统记录正文 | 意见列 |
| `fieldChanges` | 表单字段变更 | 修改详情 |
| `time` | 操作时间 | 时间列 |

`assignees[]` 只有 `userId` 和 `userName`。运行中的会签/或签可能包含多个办理人，不要只取第一项。时间线适合展示节点全貌和未来节点，平铺记录适合展示已经发生的操作；同一详情页可以二选一，也可以分为“流程进度”和“审批记录”两个区域，不能把两者混成一条无层级数组。

### 抄送 `FlowCcRecord` 与 `FlowCustomPageCcDetail`

`FlowCcRecord` 字段：

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `id` | 抄送记录 ID | `getCcDetail()` 和 `markCcRead()` 的参数 |
| `requestId` | 本次手动抄送请求的幂等标识 | 审计/诊断，默认不展示 |
| `appCode` | 所属应用编码 | 应用边界核对，通常不展示 |
| `flowType` | 流程类型；自定义页面入口固定为独立流程 | 诊断字段 |
| `flowCode` | 流程编码 | 辅助信息 |
| `flowName` | 流程名称 | 抄送列表主标题 |
| `flowVersion` | 抄送发生时的流程版本 | 审计辅助信息 |
| `processInstanceId` | 流程实例 ID | 详情与轨迹定位 |
| `sourceNodeId` | 产生抄送的来源节点 ID | “抄送于某环节”的节点定位 |
| `senderUserId` | 抄送发送人 ID | 稳定标识 |
| `senderUserName` | 抄送发送人名称 | “抄送人”展示 |
| `recipientUserId` | 接收人 ID | 当前记录归属，通常不重复展示 |
| `recipientUserName` | 接收人名称 | 管理或审计页面展示 |
| `comment` | 抄送备注 | 列表摘要或详情说明 |
| `readStatus` | `UNREAD` 或 `READ` | 未读点、筛选和状态标签 |
| `readTime` | 首次标记已读时间；未读时为空 | 已读时间 |
| `createTime` | 抄送产生时间 | 列表时间，列表默认按其倒序 |

`FlowCustomPageCcDetail` 固定包含：

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `ccRecord` | 当前抄送记录 | 抄送来源、发送人、备注和已读状态 |
| `processDetail` | 抄送发生时可见的流程详情与表单快照 | 只读详情；不是流程当前最新可编辑表单 |
| `approvalRecords` | 截止抄送发生时间可见的审批记录 | 只读历史；抄送发生后的记录不会出现在该快照中 |

打开抄送详情后可调用 `markCcRead(id)`，成功后刷新列表或本地把该记录标记为 `READ`。抄送详情不能显示办理按钮，不能用快照 `formData` 覆盖当前业务数据。

### 批量操作 `FlowBatchOperationResult`

| 字段 | 语义 | 页面使用 |
| --- | --- | --- |
| `totalCount` | 本次请求的任务总数 | 批量结果摘要 |
| `successCount` | 成功任务数 | 成功提示 |
| `failureCount` | 失败任务数 | 失败提示；大于 0 时不能提示“全部成功” |
| `results` | 每个任务的独立结果 | 逐条反馈和失败重试选择 |

`results[]` 包含 `taskId`、`success`、`errorCode`、`errorMsg`。部分成功是正常返回形态；只对失败项显示原始错误，并刷新整个列表确认最新状态，不能盲目重试已经成功的任务。

`approve()`、`reject()`、`complete()`、`transfer()`、`cancel()`、`withdraw()`、`resubmit()`、`returnTask()`、`voidProcess()`、`updateVariables()` 和 `markCcRead()` 成功时返回 `void`。无返回对象不等于没有变化；任务动作后以 `taskId` 回读任务详情，并以已有的 `processInstanceId` 回读流程详情和时间线；流程动作后回读流程详情和时间线，列表页面同时刷新相应列表。不要从请求参数推断最终任务、流程状态或下一办理人。

### 状态和动作码展示映射

流程实例状态：

| 值 | 建议文案 | 说明 |
| --- | --- | --- |
| `RUNNING` | 进行中 | 流程正在流转 |
| `SUSPENDED` | 已挂起 | 底层保留状态；出现时只读展示，不擅自提供恢复按钮 |
| `WITHDRAWN` | 已撤回 | 等待发起人重新提交 |
| `RETURNED` | 已退回发起人 | 等待修改后重新提交 |
| `COMPLETED` | 已完成 | 只表示实例正常结束，不等于一定审批通过 |
| `CANCELLED` | 已撤销 | 流程已结束 |
| `VOIDED` | 已作废 | 流程已结束且不可继续办理 |

审批拒绝不能仅凭 `processStatus` / `status` 判断。正常走到拒绝 `END` 时流程实例仍可能是 `COMPLETED`；页面要从实际到达的 `END` 步骤或图节点的 `approvalResult=REJECTED` 展示“已拒绝”。

撤回或退回发起人后，页面必须对原 `processInstanceId` 调用 `resubmit()`，不能再次调用 `start()`。重提在同一流程实例中开启新的 `approvalRound`，从 `START` 的后继节点重新流转；页面刷新详情和时间线后再展示新一轮状态。退回发起人目前只支持 `FORM_FLOW`，本指南范围内的 `INDEPENDENT_FLOW + CUSTOM_PAGE` 不提供该动作；`returnTask()` 只能选择 `getReturnTargets()` 返回的历史审批节点。

任务/节点状态可出现 `PENDING`、`RUNNING`、`PARTIALLY_COMPLETED`、`COMPLETED`、`SKIPPED`、`RETURNED`、`RETURNED_TO_STARTER`、`WITHDRAWN`、`VOIDED`、`CANCELLED`。`PARTIALLY_COMPLETED` 表示非串行多人节点已有部分任务完成但节点仍在运行；`SKIPPED` 表示未执行或因多实例结果被系统跳过，不能展示成失败。

`approvalResult` 可出现 `APPROVED`、`REJECTED`、`RESUBMITTED`、`RETURNED`、`RETURNED_TO_STARTER`、`WITHDRAWN`、`VOIDED`、`CANCELLED`。状态表示生命周期，`approvalResult` 表示业务结论，两者要分别展示。

评论动作 `type` 的固定展示语义：

| 值 | 默认文案 |
| --- | --- |
| `APPROVE` | 通过 |
| `AUTO_SKIP` | 自动跳过 |
| `REJECT` | 拒绝 |
| `TIMEOUT_APPROVE` | 超时自动同意 |
| `TIMEOUT_REJECT` | 超时自动拒绝 |
| `COMPLETE` | 办理 |
| `COMMENT` | 评论 |
| `FORM_UPDATE` | 修改表单 |
| `TRANSFER` | 转签 |
| `WITHDRAW` | 撤回 |
| `RESUBMIT` | 重新提交 |
| `RETURN` | 退回 |
| `RETURN_TO_STARTER` | 退回发起人 |
| `VOID` | 作废 |
| `CANCEL` | 撤销 |
| `SUBMIT` | 提交 |
| `PROCESS_END` | 流程结束 |

Runtime 已返回 `typeName` 时优先展示 `typeName`；上表用于图标、颜色和旧数据兜底，不覆盖服务端文案。

### 页面组合建议

- 工作台：待办、已办、我发起的、抄送我的分别使用对应列表方法；列表业务摘要来自明确请求的 `businessVariables`，不能从 `formData` 猜列。
- 详情页头部：使用 `flowName`、`processStatus`、发起人、`processStartTime`；业务主体使用 `formData`；当前环节使用任务字段；进度使用 `timeline.steps` 或 `flowDiagram`；审计记录使用 `getRecords()`。
- 时间线默认按 [时间线展示指南](custom-page-flow-timeline-display.md) 生成纵向节点、办理人状态和操作记录三层视图；流程图作为独立查看入口，不能把转签操作人的记录误作当前办理人状态。
- 操作区：先检查 `canHandle`，再按 `taskMode` 决定动作；转交同时要求候选列表非空；退回只能从 `returnTargets` 选择；流程级按钮直接使用各 `can*` 字段。
- 终态页：不要假设 `COMPLETED` 等于通过；从实际 `END.approvalResult` 区分通过和拒绝，并展示 `processEndTime`。
- 抄送详情：使用只读快照，不展示任务办理按钮；标记已读后刷新 `readStatus`。

### 写入和办理

| 页面意图 | SDK 方法 | 参数 | 返回值 |
| --- | --- | --- | --- |
| 发起 | `start(request)` | `FlowStartRequest` | `FlowCustomPageStartResponse` |
| 修改业务变量 | `updateVariables(request)` | `FlowUpdateVariablesRequest` | `void` |
| 同意 | `approve(request)` | `FlowTaskActionRequest` | `void` |
| 驳回 | `reject(request)` | `FlowTaskActionRequest` | `void` |
| 完成办理 | `complete(request)` | `FlowTaskActionRequest` | `void` |
| 批量同意/驳回 | `batchApprove` / `batchReject` | `FlowBatchTaskActionRequest` | `FlowBatchOperationResult` |
| 转交 | `transfer(request)` | `FlowTransferRequest` | `void` |
| 批量转交 | `batchTransfer(request)` | `FlowBatchTransferRequest` | `FlowBatchOperationResult` |
| 取消/撤回/作废 | `cancel` / `withdraw` / `voidProcess` | `FlowProcessReasonRequest` | `void` |
| 重新提交 | `resubmit(request)` | `FlowResubmitRequest` | `void` |
| 退回指定节点 | `returnTask(request)` | `FlowReturnRequest` | `void` |
| 手动抄送 | `ccProcess(request)` | `FlowCcRequest` | `FlowCcRecord[]` |
| 标记抄送已读 | `markCcRead(ccRecordId)` | `number \| string` | `void` |

## 请求类型

```typescript
interface FlowVariableQuery {
  currentPage?: number; // 默认 1
  pageSize?: number; // 默认 20
  variables?: Record<string, any>;
  variableKeys?: string[];
}

interface FlowSubmittedQuery {
  currentPage?: number;
  pageSize?: number;
  variableKeys?: string[];
  status?: string;
}

interface FlowDefinitionQuery {
  currentPage?: number;
  pageSize?: number;
  flowName?: string;
}

interface FlowCcQuery {
  currentPage?: number;
  pageSize?: number;
  readStatus?: string;
}

interface FlowStartRequest {
  flowCode: string;
  formData?: Record<string, any>;
  variables?: Record<string, any>;
  idempotencyKey?: string;
}

interface FlowUpdateVariablesRequest {
  processInstanceId: string;
  set?: Record<string, any>;
  remove?: string[];
}

interface FlowTaskActionRequest {
  taskId: string;
  formPatch?: Record<string, any>;
  formDataVersion?: number;
  variables?: Record<string, any>;
  comment?: string;
}

interface FlowBatchTaskActionRequest {
  taskIds: string[];
  comment?: string;
  variables?: Record<string, any>;
}

interface FlowTransferRequest {
  taskId: string;
  targetUserId: string;
  comment?: string;
}

interface FlowBatchTransferRequest {
  taskIds: string[];
  targetUserId: string;
  comment?: string;
}

interface FlowProcessReasonRequest {
  processInstanceId: string;
  reason?: string;
}

interface FlowResubmitRequest {
  processInstanceId: string;
  formPatch?: Record<string, any>;
  variables?: Record<string, any>;
  formDataVersion?: number;
  comment?: string;
}

interface FlowReturnRequest {
  taskId: string;
  targetNodeId: string;
  reason?: string;
}

interface FlowCcRequest {
  processInstanceId: string;
  taskId?: string;
  recipientUserIds: string[];
  comment?: string;
  requestId?: string;
}
```

必填字符串不能为空；`taskIds`、`recipientUserIds` 至少包含一个有效 ID。普通非动态自定义页面使用 `formPatch` 时不需要构造 schema 或 `formDataVersion`。动态表单要求版本时，办理和重提必须使用 Runtime 返回的最新 `formDataVersion`，不要提交 schema，也不要自行递增、缓存猜测或使用客户端业务版本。`approve()`、`reject()`、`complete()` 不能同时传 `formPatch` 与 `variables.formData`；`resubmit()` 始终不能传 `variables.formData`。

## 采购申请示例

页面不初始化 SDK，只使用 `useSdkClient()`：

```jsx
import { useSdkClient } from "@/context/app-context";

const client = useSdkClient();
const flow = client.flow();

const started = await flow.start({
  flowCode: "purchase_request",
  formData: {
    subject: "研发设备采购",
    category: "信息技术设备",
    quantity: 1,
    budgetAmount: 12000,
    supplier: "示例供应商",
    reason: "项目开发使用",
  },
  variables: {
    businessId: "PUR-20260916-001",
    departmentCode: "RD",
    budgetAmount: 12000,
  },
  idempotencyKey: "purchase-PUR-20260916-001",
});

const todoPage = await flow.listTodo({
  variables: { businessId: "PUR-20260916-001" },
  variableKeys: ["businessId", "departmentCode", "budgetAmount"],
});

const task = todoPage.records[0];
if (task) {
  await flow.approve({
    taskId: task.taskId,
    comment: "预算和供应商信息已确认",
    variables: { purchaseStatus: "APPROVED" },
  });
}

console.log(started.processInstanceId);
```

发起时为同一业务单生成稳定的 `idempotencyKey`，网络重试复用该值。手动抄送同一次操作重试时复用 `requestId`。

## 页面状态与错误处理

- 列表、详情和写入分别维护加载、空态、失败和成功状态。
- 提交按钮在请求进行中禁用，避免重复点击；幂等键仍必须稳定。
- 写入成功后重新读取相关列表或详情，不根据本地猜测下一任务或流程状态。
- 捕获 `LovrabetError`，使用 `status`、`code`、`message`、`description` 和 `response` 形成用户可理解的错误；不要记录 Cookie 或凭据。
- 遇到应用边界、办理人、流程状态或管理员权限错误时，保留 Runtime 原始语义；不要删除 `appCode`、伪造身份或改用低层 HTTP 绕过 SDK。

## 页面自检

- [ ] 目标流程是已发布的 `INDEPENDENT_FLOW + CUSTOM_PAGE`。
- [ ] 页面使用 `useSdkClient()` 和无参数 `client.flow()`。
- [ ] 页面没有手动传 `appCode`、`operatorUserId`、Cookie、AccessKey 或 Token。
- [ ] `formData`、`formPatch`、`variables` 和 `variableKeys` 没有混用。
- [ ] 普通表单的 `formPatch` 按根级字段合并，嵌套对象和数组没有被误认为深合并。
- [ ] `approve()`、`reject()`、`complete()` 没有同时传 `formPatch` 和 `variables.formData`；`resubmit()` 没有传 `variables.formData`。
- [ ] 列表查询使用稳定业务变量；没有查询系统变量。
- [ ] 动态表单办理使用最新 `formDataVersion`。
- [ ] 退回目标来自 `getReturnTargets()`。
- [ ] 应用范围方法只出现在管理员页面。
- [ ] 写入有重复提交保护，成功后按服务端事实刷新。
- [ ] 页面完整处理 Runtime 权限、状态和业务错误。
