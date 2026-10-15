# 流程设计态开发工作流

本工作流用于在 `rabetbase` 中创建、校验和管理表单审批流与独立工作流定义。它只覆盖设计态流程，不办理运行态待办。

执行前按当前步骤阅读：

- 命令参数与风险：[flow 命令参考](../references/rabetbase-flow-commands.md)
- 页面、人员、角色、脚本和通知配置发现：[审批流资源发现](../references/rabetbase-flow-resources.md)
- FlowConfig 字段与拓扑规则：[FlowConfig 契约](../knowledge/approval-flow/config-json-schema.md)
- 设计态与运行态边界：[审批流职责边界](../references/rabetbase-flow-runtime-boundary.md)
- 完整表单与独立流程示例：[FlowConfig 示例](flow-config-examples.md)，使用前替换为查询到的真实资源

## 流程类型判定

先确定外层 `flowType`，再设计节点；不要根据是否出现审批节点反推流程类型。

面向用户时按下列默认策略判定，不要先要求用户理解或选择 `flowType`：

- 用户只说“创建审批流”“创建工作流”“设计业务流程”或“配置多级审批”时，默认推荐 `INDEPENDENT_FLOW`，并用“独立工作流”与用户沟通。
- 需求包含填写、补充材料、执行业务、确认收货或其他人工办理步骤时，必须使用 `INDEPENDENT_FLOW`。
- 只有用户明确说明是“已有数据集表单提交后的纯审批”，且没有上述办理步骤时，才选择 `FORM_FLOW`。
- 业务信息不足时，可先说明推荐独立工作流及原因，再询问影响拓扑、办理人或数据绑定的必要业务信息。

| `flowType` | 产品语义 | 人工节点 | 页面与数据边界 |
|---|---|---|---|
| `FORM_FLOW` | 表单审批流 | 只允许 `taskMode: APPROVAL`，禁止 `HANDLE` | 绑定 `pageIds` 页面列表；表单填写发生在发起前，驳回修改走重新提交 |
| `INDEPENDENT_FLOW` | 独立工作流 | 可同时使用 `APPROVAL` 和 `HANDLE` | 不绑定 `pageIds` 页面列表；业务数据由工作流页面和 SDK 管理 |

独立工作流即使包含审批节点仍然是 `INDEPENDENT_FLOW`。需要填写、补充材料、执行采购或确认收货等人工办理步骤时，必须使用独立工作流，不能把 `HANDLE` 塞进表单审批流。

独立工作流的服务端语义包含两种 `flowJson.pageMode`：

- `CUSTOM_PAGE`：业务自定义页面；缺省时按此模式处理。
- `PLATFORM_FORM`：服务端平台表单页面语义；当前公开 Skill 和 CLI 不生成或提交该模式。

用户要求 `PLATFORM_FORM` 时，直接返回 `NEEDS_DSL_EXTENSION`，说明当前版本只支持业务自定义页面；不要继续生成 FlowConfig，也不要执行 create/update。

`pageMode` 不适用于 `FORM_FLOW`。自定义页面 Flow SDK 只查询和操作 `INDEPENDENT_FLOW + CUSTOM_PAGE`；页面运行态调用遵循 [`custom-page-flow-sdk.md`](custom-page-flow-sdk.md)。

`INDEPENDENT_FLOW + CUSTOM_PAGE` 可以选择性配置：

- `flowJson.startPath`：进入发起页面时使用。
- `APPROVAL.path`：待办、已办或任务详情进入当前节点页面时使用。
- `END.path`：流程到达该结果节点后进入对应终态页面时使用。

这些字段只是导航元数据，不是业务字段，不参与分支、权限、办理人或流程状态计算；不配置时由业务应用自行选择页面。

### 页面导航绑定门禁

导航字段的值必须是目标自定义页面的完整运行态地址 `runtimePageUrl`，例如 `https://app-demo.app.lovrabet.com/purchase/approve`；不得填写 `data.pages[].path`、`pageUrl`、`editPageUrl`，也不得根据 appCode、Domain 或菜单路径手工拼接。

1. 先执行 `rabetbase page custom-list --format compress`，根据 `pageId`、`label` 和 `runtimePageUrl` 选择目标页面。列表中的 `runtimePageUrl` 只是候选运行态地址，不证明页面已经发布。
2. 对每个准备绑定的页面执行 `rabetbase page custom-detail --id <pageId> --format compress`，以 `data.status` 判断当前保存内容是否已经发布。
3. 只有 `data.status === "FORMAL"` 时，才允许把该页面返回的完整 `runtimePageUrl` 填入 `flowJson.startPath`、`APPROVAL.path` 或 `END.path`。
4. `data.status` 不是 `FORMAL` 时，不得先填写候选 URL，也不得继续创建或更新包含该导航字段的 FlowConfig；返回 `NEEDS_USER_CONFIRMATION`，向用户说明页面尚未发布并询问是否先发布。
5. 用户同意发布后，先执行 `page custom-publish --id <pageId> --dry-run` 并展示预览，取得明确确认后正式发布；只有回读 `data.after.status === "FORMAL"` 后，才使用发布响应中的 `runtimePageUrl` 填写导航字段。用户不同意发布时保持对应字段缺失。

以上是 Skill 的远端事实门禁；本地 `flow validate` 只校验导航字段是否为非空字符串，不能替代页面发布状态检查。

## 标准流程

1. 明确 `flowName`、业务描述、业务状态、办理步骤和通过/拒绝路径，并按上表显式选择 `flowType`。
2. 按下方“表单流程的新建与绑定”或“独立工作流的页面接入”完成资源发现。`FORM_FLOW` 使用完整 `pageIds` 数组，旧 `datasetCode` 绑定保存时忽略；`INDEPENDENT_FLOW` 不配置外层数据集或页面绑定字段。编辑前回读定义、绑定和发布状态，曾发布的流程不能改 `flowType`。
3. 建立业务变量契约：逐项确认变量名、类型、初始来源、更新节点、消费节点以及是否用于待办/已办查询；区分顶层 `variables` 与 `formData`，禁止把 `approval_sys_*`、`approved` 当作业务变量。
4. 根据业务逻辑直接生成完整的 `nodes + edges` 拓扑，不使用 `templateName` 或 SmartCode 模板表。
5. 为每个人工节点确认 `taskMode`、`approvalMode`、`assignee.strategy` 和是否 `sequential`。决策用 `APPROVAL`；仅 `INDEPENDENT_FLOW` 可用 `HANDLE` 表达填写或确认数据后继续。办理完成后需要按业务变量分支时，按下方“HANDLE 后续路由”组合节点。
6. 逐个 `APPROVAL` 节点确认转交候选人 `transferCandidates`、超时动作 `timeout`、节点完成后默认抄送 `cc`；逐个 `END` 节点确认默认抄送 `cc`。只有 `INDEPENDENT_FLOW + CUSTOM_PAGE` 可配置这些节点的页面 `path`。页面导航只能取已发布页面的完整 `runtimePageUrl`；页面未发布时停止填写并询问是否先发布。用户未提出时保持不配置，不自行推断人员、页面或超时时长。
7. 固定办理人、转交候选人和默认抄送人使用 `flow runtime-user-search` 查询运行态 `userId`；角色审批使用 `flow runtime-role-list` 查询运行态 `roleId`，必要时用 `flow runtime-role-user-list` 核对成员。不要把 `role *` 查询出的配置端人员/角色写入 FlowConfig。
8. `SCRIPT` 节点和 `timeout.action: "EXECUTE_BFF"` 使用 `bff list --type ENDPOINT` 选择同应用、非个人的真实 `functionName`；同时确认同步/异步方式、重试配置和独立的 `resultVariable`。不要让多个脚本依赖默认的 `scriptResult`。SCRIPT 不配置审批人，执行身份由服务端处理，不传用户或系统标记。
9. `NOTIFICATION` 节点使用 `notification config-list --all` 选择真实 `configCode`，并确认 `failurePolicy`、模板内容、变量占位符和收件人来源。
10. 生成最小 FlowConfig。不要把服务端 id、`appCode`、部署状态、编译上下文、`templateName`、`formSchema/formDefinitions/formPermissions` 或节点 `form/formPermission` 写入文件。
11. 执行 `rabetbase flow validate --file <path> --format compress`。
12. 展示变量契约、拓扑、资源绑定、节点可选配置、warnings 和校验结果。只有 `valid=true` 才能进入远程写入。
13. 先执行 create/update 的 `--dry-run`，审阅实际目标和完整页面绑定变化，再按已授权的写入范围正式提交。未授权的远端写入才请求用户确认；已有授权不重复询问。保存响应中取真实流程 ID，不用示例 ID 或本地遗留 ID。
14. 完成下方“发布与运行验收”。publish 属于普通 `write`，先审阅 `--dry-run`，正式执行后回读定义，并交接运行态验收。

## 表单流程的新建与绑定

1. 新建时用 `flow page-list --name <keyword>` 查询 CREATE 标准页面，核对分页和真实页面 ID。未带流程 ID 的 `availabilityChecked=false` 表示候选尚未过滤其他流程占用；保存时服务端最终检查唯一性。已有流程用 `flow page-list --id <flow-id>`，排除其他未删除流程占用并保留自身绑定。
2. 将所有选中项写入外层 `pageIds`。一个流程可绑定不同数据集的页面；同一数据集的不同页面也可绑定不同流程。不能只取数组首项，不能通过 `datasetCode` 给同数据集所有页面隐式绑定。
3. 已知真实页面 ID 可以直接 create，无需先创建空草稿。只有需要先取得 ID 再筛可绑定页面时，才先保存未绑定的合法图，再查询并 update。空 `nodes/edges` 不能保存；START → END 虽可保存但会直接结束，不能交付为已配置人工审批。
4. 编辑先 `flow detail --id <flow-id>` 核对原绑定。省略页面字段保留原绑定；显式数组替换全部，`[]` 清空全部。先 dry-run 审阅 `pageBindings.before/after/added/removed`，有新增或移除页面时，正式 update 携带从该预览取得的 `--expected-page-ids '<原数组 JSON>'`。绑定集合未变、仅改节点时不要求该参数。不匹配时重新查询并审阅，不自行覆盖并发修改。
5. 在 `menu list` 中核对选中页面的有效入口。发布只标记明确绑定页面已有菜单为 `process`，不会创建页面或菜单，也不会补业务数据的审批字段。无真实 CREATE 页面时先按 [标准页面生成](../references/rabetbase-page-generate-start.md) 查询 `page data-list-status`：允许生成时预览，再按授权范围 `generate-start --apply` 并跟进原任务和完整页面组；`manualResolutionRequired` 或冲突时先处理事实，不盲目重生成。该入口生成包含 CREATE 的 CRUD 页面组，不接受 `--page-type CREATE`。完成后重新 `flow page-list` 取真实 CREATE ID，不用自定义页面导航地址代替 `pageIds`。
6. 交付可运行表单审批时必须至少绑定一个有效页面。仅保存未绑定草稿时明确交付范围，不能因为服务端允许发布就描述为可发起。
7. 生成或修改运行页面时，必须阅读 [表单创建与显式启动](form-flow-start.md)。仅 `FORM_FLOW` 页面先保存业务记录，再使用本次实际 `pageId/dataId` 发起；`datasetCode` 仅在启动接口无法从实际页面解析主标准表单时按已确认值传入，普通 create 仍必须指向实际目标模型且不自动发起审批。`INDEPENDENT_FLOW` / `CUSTOM_PAGE` 合同不变。

## 独立工作流的页面接入

- 使用 `INDEPENDENT_FLOW` 和显式或默认的 `pageMode: "CUSTOM_PAGE"`。用户要求 `PLATFORM_FORM` 时返回 `NEEDS_DSL_EXTENSION`，不生成不支持的表单 DSL。
- 业务自定义页面通过 SDK 发起或办理任务，按业务状态查询和更新业务数据；发起独立流程本身不要求先 create 数据集记录。
- 配置 `startPath` 或 `APPROVAL/END.path` 前完成页面导航绑定门禁；不需要导航时可省略。这些字段不决定流程分支或办理权限。
- 生成页面前阅读 [Flow SDK 指南](custom-page-flow-sdk.md)，检查宿主 SDK 确实提供无参数 `client.flow()`。项目所装 SDK 缺少能力时先解决依赖能力，不生成假方法或改用旧 Flow OpenAPI。

从零创建、还没有流程编码和页面时，先用完整业务拓扑创建不带导航字段的草稿，取得真实 `id/flowCode`；再按 [自定义页面开发](custom-page-workflow.md) 生成业务页面并按授权范围发布；回读已发布页面的 `runtimePageUrl`，导出流程文件补齐导航，用 update 保存并 publish，最后验收页面调用。生成期间发起按钮须等待流程已发布且可用。不要编造流程编码，也不要为了取得页面地址发布没有实际业务交互的占位页。已有页面或不需要导航时，可直接走常规 create/publish，无需重复建立资源。

## 发布与运行验收

1. 回读保存后的定义，核对 `flowType`、完整 `pageIds`、拓扑与资源。表单流程核对真实 CREATE 页面和菜单；固定审批人确认身份，角色审批确认运行态角色有有效成员；脚本与通知配置核对实际存在。创建者、发起者和模板不会自动替代节点 `assignee`。
2. 按授权范围 publish；成功后回读 `flow detail`，确认 `flowStatus=published`、真实 `flowCode` 和完整页面绑定。当前详情不公开 `enabled`，启用状态通过平台或实际运行入口核验，字段缺失不能当作已启用。发布不自动启用已停用流程；停用时在平台完成启用并再验收，不编造 CLI 启用命令。
3. 已发布流程 update 后回到草稿，需要重新发布。曾发布的流程禁止更改类型，即使目前草稿或停用；更换类型创建另一条定义，并明确页面和实例迁移范围。
4. publish 成功只表示设计部署成功，不表示运行发起成功。按职责边界交接 Runtime 页面验证：表单流程至少覆盖第二个绑定页面、同数据集不同页面的选流、先保存再发起及失败恢复；独立流程覆盖发起、人工节点、分支和终态，使用稳定幂等键处理发起重试。
5. 验收记录分别注明本地校验、保存、发布、启用状态和实际运行结果。未执行运行验证时直说尚未验证，不用设计态响应替代。

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

- `variables` 办理参数更新顶层流程变量；`formPatch` 按根级字段合并 `formData`，未传字段保留，传入字段替换，`null` 设为空值，嵌套对象和数组整字段替换。
- 对象和数组可以作为流程变量供页面、脚本或通知读取，但当前待办/已办变量查询只支持字符串、数字和布尔值等值条件；需要查询的业务状态应单独保存为顶层标量变量。
- `approval_sys_*`、`approved` 和平台维护的 `formData` 容器名属于保留范围，不作为业务变量名或 SCRIPT `resultVariable`。

## HANDLE 后续路由

- `HANDLE` 只表示“填写或确认数据后继续”，自身必须且只能保留一条 `ALWAYS` 出边，不承担业务分支选择。
- 自定义页面办理任务时，可以在办理请求中的 `variables` 更新业务变量，也可以用 `formPatch` 按根级字段合并更新 `formData`；任务完成后，下游节点读取更新后的值。
- 路由变量已由页面准备好时，生成 `HANDLE → CONDITION`，由条件网关表达多条业务路径。
- 需要 Backend Function 先计算、校验或归一化路由变量时，生成 `HANDLE → SCRIPT → CONDITION`；不需要计算时不要为了分支强制插入 `SCRIPT`。
- 不要给 `HANDLE` 直接配置多条出边，也不要让调用方传 `targetNodeId` 绕过流程图。办理节点存在多个业务结果不应判定为 `NEEDS_DSL_EXTENSION`；只有下游节点也无法表达需求时才进入 DSL 扩展判断。

## 候选确认

- 不编造 `userId`、`roleId`、`datasetCode`、`pageId`、页面 `path`、`runtimePageUrl`、`scriptName` 或 `configCode`。
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

新建路径（`<flow-id>` 取 create 响应 `data.after.id`）：

```bash
rabetbase flow create --file .rabetbase/flows/purchase.json --dry-run --format compress
rabetbase flow create --file .rabetbase/flows/purchase.json --format compress

rabetbase flow detail --id <flow-id> --format compress
rabetbase flow publish --id <flow-id> --dry-run --format compress
rabetbase flow publish --id <flow-id> --format compress
rabetbase flow detail --id <flow-id> --format compress
```

编辑路径与 create 二选一，不需要创建后无改动再 update。修改表单绑定时：

```bash
rabetbase flow detail --id <flow-id> --format compress
rabetbase flow update --id <flow-id> --file .rabetbase/flows/purchase.json --dry-run --format compress
# 用 dry-run 的 pageBindings.before 替换 <original-page-ids-json>
rabetbase flow update --id <flow-id> --file .rabetbase/flows/purchase.json --expected-page-ids '<original-page-ids-json>' --format compress
# 已发布定义更新后，需要再按新建路径的 publish/detail 步骤发布和核验
```

`flow create` 和 `flow publish` 成功后都会返回当前环境对应的 `data.flowUrl`，并在成功消息中附上同一地址。Agent 应直接使用该字段交付可打开的设计态流程页面，不自行猜测域名或路径；daily、production、国家/地区节点及独立部署由 CLI 当前路由配置决定。

不要根据本地文件中的遗留 `id` 或 `appCode` 决定提交目标。流程 ID 来自命令参数，应用编码来自当前工作区或显式 CLI app 选择。

表单审批流的运行页面按 [表单创建与显式启动](form-flow-start.md)处理：创建只保存数据，显式启动复用已有业务 ID；定义页面数组与实际发起页面必须区分。
