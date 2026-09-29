# Rabetbase FlowConfig 契约

本文件描述 `rabetbase flow validate/create/update/detail --output` 使用的可移植本地清单。服务端生成的 id、编码、状态和部署信息不属于本地文件。

## 外层结构

```json
{
  "flowName": "采购审批",
  "flowDesc": "采购申请的主管审批",
  "flowType": "INDEPENDENT_FLOW",
  "flowJson": {
    "pageMode": "CUSTOM_PAGE",
    "startPath": "https://app-demo.app.lovrabet.com/purchase/start",
    "nodes": [],
    "edges": []
  }
}
```

必填字段：

- `flowName`：非空流程名称。
- `flowType`：`FORM_FLOW` 或 `INDEPENDENT_FLOW`，不得根据节点或数据集绑定推断。
- `flowJson.nodes`：非空节点数组。
- `flowJson.edges`：非空连线数组。

可选字段：`flowDesc`、`datasetCode`、`pageId`。

`flowType` 是产品语义分类，不由节点形状推断：

- `FORM_FLOW`：表单审批流，可以配置 `datasetCode`、`pageId`；人工节点只允许 `taskMode: "APPROVAL"`，禁止 `HANDLE`。
- `INDEPENDENT_FLOW`：独立工作流，不能配置外层 `datasetCode` 或外层 `pageId`；可以同时使用 `APPROVAL` 和 `HANDLE`。

独立工作流中出现审批节点不会把它变成表单审批流。需要填写、补充材料、执行或确认等人工办理步骤时使用 `INDEPENDENT_FLOW + HANDLE`。

`flowJson.pageMode` 只适用于 `INDEPENDENT_FLOW`：

- `CUSTOM_PAGE`：业务自定义页面，缺省时按此值处理。
- `PLATFORM_FORM`：服务端的平台表单页面语义；当前 Skill 和 CLI 不支持生成或提交。

`FORM_FLOW` 不得配置 `pageMode`。自定义页面 SDK/OpenAPI 只返回 `INDEPENDENT_FLOW + CUSTOM_PAGE`。

`INDEPENDENT_FLOW + CUSTOM_PAGE` 可选配置非空字符串 `flowJson.startPath`，并在 `APPROVAL` 或 `END` 节点上可选配置非空字符串 `path`。Skill 必须填写 `page custom-detail` 确认为 `FORMAL` 的自定义页面所返回的完整 `runtimePageUrl`，不得填写菜单原始 `path`、最新保存内容地址 `pageUrl`、`editPageUrl` 或手工拼接地址；页面未发布时先保持字段缺失并询问用户是否发布。这些字段只用于页面跳转，不参与流程执行或业务判断；缺失时调用方自行选择页面。其他流程类型、`PLATFORM_FORM` 或其他节点类型不得配置这些导航字段。旧字段 `flowJson.startPageId` 和节点 `pageId` 不再支持。

本地 FlowConfig 必须显式填写 `flowType`。CLI 不再利用服务端的历史默认值推断为 `FORM_FLOW`，避免表单审批流和独立工作流混用。

Skill 根据业务逻辑直接生成完整 `nodes + edges` 拓扑，不读取或填写 SmartCode `templateName`。

`appCode` 来自当前 CLI 工作区，远程定义 `id` 来自 `--id`。二者都不落本地文件。

## 规范化

`normalizeFlowConfig` 不修改调用方对象，并对下列旧字段产生 warning 后从规范结果移除：

- 外层：`id`、`appCode`、`templateName`
- `flowJson`：`flowKey`、`flowCode`、`flowName`、`bindDatasetCode`、`datasetCode`、`appCode`、`pageId`、`templateName`
- 任意层级值为空的 `metadata`

`flowJson.schemaVersion` 按 SmartCode 规则单独处理：空值或默认值 `approval.simple-flow.v1` 会警告并移除；其他值保留在规范结果中，并由本地校验明确返回 `Unsupported schemaVersion`。这样不会把未知或未来版本静默降级为 v1。

`flow validate --write-normalized` 只写规范结果。校验器从外层元数据创建独立内存副本，临时注入编译上下文；这些值不会污染文件。

## 页面模式

使用 `INDEPENDENT_FLOW + CUSTOM_PAGE` 时：

- 采购申请等业务数据由自定义页面直接读写业务数据集，并按业务状态展示不同页面和操作。
- 流程只保存路由、办理人、脚本、通知等执行配置；页面通过 SDK 发起或办理任务，成功后自行更新业务状态。
- 禁止 `flowJson.formSchema`、`flowJson.formDefinitions`、`flowJson.formPermissions`、节点 `form`、节点 `formPermission` 和 `cc.readSources`。
- `validate/create/update/detail --output` 遇到上述字段会明确报错，不会静默删除或导出丢字段的文件。普通 `detail` 仍可只读查看远端原始定义。

`INDEPENDENT_FLOW + PLATFORM_FORM` 使用平台表单能力，不属于自定义页面 SDK 的返回范围。当前 CLI 只识别该服务端语义用于明确报错，`validate/create/update/detail --output` 均拒绝生成或提交此模式；Skill 必须返回 `NEEDS_DSL_EXTENSION`。普通 `detail` 仍可只读查看远端原始定义。

## 通用节点

```json
{
  "id": "managerApprove",
  "type": "APPROVAL",
  "name": "主管审批",
  "position": { "x": 300, "y": 180 }
}
```

- 支持 `START`、`END`、`APPROVAL`、`CONDITION`、`SCRIPT`、`NOTIFICATION`。
- 每个节点都必须有非空 `name`。
- 每个节点都必须显式提供有限且大于等于 0 的 `position.x/y`。
- 节点 ID 和边 ID 使用 `[A-Za-z_][A-Za-z0-9_]*`。
- 节点和边共用一个 BPMN ID 命名空间，任何元素都不能重名。

## 图结构

- 必须且只能有一个 `START`，至少有一个 `END`。
- `START` 无入边，`END` 无出边；其他节点必须同时满足必要的入边/出边要求。
- 所有节点必须从 `START` 可达，并且每个可达节点都必须存在一条通往某个 `END` 的路径。
- 支持自环、回退连线和多节点有向环；环路必须保留能够到达 `END` 的退出路径，不能形成无出口闭环。
- 相同 `source + target + conditionType` 只能出现一次，即使边 ID 或 expression 不同。

## APPROVAL

```json
{
  "id": "managerApprove",
  "type": "APPROVAL",
  "name": "主管审批",
  "path": "https://app-demo.app.lovrabet.com/customer-dashboard/approve",
  "position": { "x": 300, "y": 180 },
  "taskMode": "APPROVAL",
  "approvalMode": "SINGLE",
  "sequential": false,
  "assignee": {
    "strategy": "FIXED",
    "userIds": ["82"]
  },
  "transferCandidates": {
    "strategy": "FIXED",
    "userIds": ["15023", "16001"]
  }
}
```

- `taskMode` 支持 `APPROVAL`、`HANDLE`；省略时按 `APPROVAL`。`FORM_FLOW` 只能使用 `APPROVAL`，`HANDLE` 仅允许出现在 `INDEPENDENT_FLOW`。
- `path` 可省略；仅 `INDEPENDENT_FLOW + CUSTOM_PAGE` 的 `APPROVAL` 人工节点可配置非空字符串，用于待办、已办和任务详情跳转，不影响节点执行。
- `approvalMode` 支持 `SINGLE`、`ALL`、`ANY`；省略时按 `SINGLE`。`sequential` 控制多办理人是否顺序执行，默认 `false`。
- `FIXED` 需要非空有效 `userIds`；`SINGLE + FIXED` 只能有一个不同用户。
- `ROLE` 需要正整数 `roleId`。
- `transferCandidates` 可省略；存在时只支持 `FIXED`，且 `userIds` 必须非空，每项为非空字符串或正整数。
- `taskMode: "APPROVAL"` 必须且只能有一条 `APPROVED` 出边和一条 `REJECTED` 出边，两条边都不能是默认边。
- `taskMode: "HANDLE"` 必须且只能有一条 `ALWAYS` 出边，不能配置 `defaultFlow` 或 `expression`；办理完成后直接沿该边继续。
- 只要流程包含决策型 `APPROVAL` 任务，就必须同时存在 `result: "APPROVED"` 和 `result: "REJECTED"` 的 END 节点。只有 `HANDLE` 的流程不要求这两个结果节点。

### HANDLE 后续路由

- `HANDLE` 自身不选择业务分支。自定义页面可在办理请求中的 `variables` 更新业务变量，或通过 `formPatch` 按根级字段合并更新 `formData`；任务完成后，下游节点读取更新后的值。
- 路由变量已准备好时使用 `HANDLE → CONDITION`，由 CONDITION 的表达式和默认边决定后续路径。
- 需要 Backend Function 先计算、校验或归一化路由变量时使用 `HANDLE → SCRIPT → CONDITION`；不需要计算时可省略 SCRIPT。
- 不允许给 HANDLE 配置多条出边，也不允许通过 `targetNodeId` 让调用方直接选择目标节点。办理节点存在多个业务结果不应判定为 `NEEDS_DSL_EXTENSION`，因为分支职责属于下游 CONDITION。

### 审批超时

`APPROVAL.timeout.enabled=true` 时必须配置正整数 `duration.value` 和 `SECONDS/MINUTES/HOURS/DAYS` 单位：

- `AUTO_APPROVE`、`AUTO_REJECT` 必须中断当前任务，`interrupting` 省略时默认 `true`。
- `EXECUTE_BFF` 必须配置 `scriptName`，不能中断当前任务，`interrupting` 省略时默认 `false`；可用 `resultVariable` 保存结果。
- `HANDLE` 超时只支持 `EXECUTE_BFF`。

## END

```json
{
  "id": "approvedEnd",
  "type": "END",
  "name": "审批完成",
  "result": "APPROVED",
  "path": "https://app-demo.app.lovrabet.com/purchase/completed",
  "position": { "x": 560, "y": 100 }
}
```

- `path` 可省略；仅 `INDEPENDENT_FLOW + CUSTOM_PAGE` 的 `END` 节点可配置非空字符串，用于流程到达该结果节点后的终态页面导航，不影响流程结果。Runtime 会用实际到达的 `END.path` 追加 `processId`，生成已结束流程的 `detailUrl`。
- `result` 的流程语义保持不变；`path` 不是任务页面地址，不会作为待办或已办任务的当前节点路径返回。

## CONDITION

```json
[
  {
    "id": "edge_amount_finance",
    "source": "amountCheck",
    "target": "financeApprove",
    "conditionType": "EXPRESSION",
    "expression": "${formData.amount > 10000}"
  },
  {
    "id": "edge_amount_default",
    "source": "amountCheck",
    "target": "approvedEnd",
    "conditionType": "ALWAYS",
    "defaultFlow": true
  }
]
```

- CONDITION 至少有两条出边。
- 必须且只能有一条 `defaultFlow: true`。
- 默认边必须使用 `ALWAYS`，且不能定义 `expression`。
- 其余边必须使用 `EXPRESSION`，且必须有非空 `expression`。

表达式直接读取顶层流程变量；例如发起或办理请求中的 `variables.amount` 在表达式中写作 `${amount > 10000}`，`formData` 字段写作 `${formData.amount > 10000}`。生成表达式前先按开发工作流建立变量契约，不猜测变量名或类型。

## SCRIPT

```json
{
  "id": "checkPrice",
  "type": "SCRIPT",
  "name": "价格校验",
  "position": { "x": 520, "y": 180 },
  "scriptName": "checkProductPrice",
  "resultVariable": "priceCheckResult",
  "async": {
    "enabled": true,
    "retryCount": 3,
    "retryInterval": { "value": 30, "unit": "SECONDS" }
  }
}
```

- `scriptName` 必填。
- `resultVariable` 可省略；Runtime 省略时写入默认变量 `scriptResult`。存在时必须符合变量命名规则并在流程内唯一；多个 SCRIPT 或 `EXECUTE_BFF` 超时动作必须显式配置不同的结果变量，避免默认值互相覆盖。
- 禁止 `formData`、`approved` 和 `approval_sys_` 前缀。
- 表达式引用某个 `resultVariable` 时，产生该变量的 SCRIPT 必须位于引用边 source 的所有上游路径；SCRIPT 自己的出边可以读取刚写入的结果。
- `async.enabled=true` 时使用 Flowable 异步 Job；`retryCount` 默认 3、范围 1～10，`retryInterval` 默认 30 秒，也支持 `MINUTES/HOURS/DAYS`。
- SCRIPT 接收当前流程变量作为顶层 `params`，并额外收到 `params.approvalContext`；因此业务变量 `amount`、表单字段 `formData.amount` 分别通过 `params.amount`、`params.formData.amount` 读取，不使用 `params.variables` 包装。

## 节点默认抄送

APPROVAL 和 END 节点可配置 `cc.recipientUserIds`。APPROVAL 在整个节点办理结束后创建默认只读抄送记录（会签/或签不会按每个办理人重复抄送），END 在到达结果节点后创建。用户 ID 必须是 1～100 个不重复的非空字符串；`visible` 可限制可见表单字段。抄送详情的 `businessVariables` 从流程当前 variables 中读取，并过滤 `approval_sys_*`、`formData` 和 `approved`；当前页面模式不支持 `cc.readSources`。

## NOTIFICATION

```json
{
  "id": "notifyApproved",
  "type": "NOTIFICATION",
  "name": "发送审批结果通知",
  "position": { "x": 740, "y": 180 },
  "config": {
    "version": "v1",
    "configCode": "approval_email",
    "failurePolicy": "CONTINUE",
    "template": {
      "title": "审批结果通知",
      "summary": "${event.initiatorUsername} 发起的申请已经${event.approvalResultName}",
      "facts": [
        {
          "label": "申请金额",
          "value": "${variables.formData.amount}"
        }
      ],
      "detailMarkdown": "**审批节点：** ${context.nodeName}",
      "actions": [
        {
          "text": "审批详情页",
          "url": "${context.detailUrl}"
        }
      ]
    },
    "recipients": [
      {
        "type": "USER",
        "role": "TO",
        "value": "${event.initiatorUserId}"
      }
    ]
  }
}
```

配置规则：

- `version` 可省略或留空，默认 `v1`；当前只支持 `v1`。
- `configCode` 必须是从 `notification config-list` 选出的固定非空值，不能使用占位符、渠道名、URL、dataset code 或凭据。
- `failurePolicy` 支持 `CONTINUE`、`FAIL`，默认 `CONTINUE`。
- `template` 必填；不要输出 `theme` 或 `templateType`。
- `template.title`、`template.summary` 必须是非空字符串。
- `template.detailMarkdown` 可选，存在时必须是字符串。
- `template.facts` 可选，最多 8 项；每项需要非空 `label`、`value`。
- `template.actions` 可选，最多 2 项；每项需要非空 `text`、`url`。
- `actions[].url` 必须使用 `http://`、`https://`，或完整受支持占位符，例如 `${context.detailUrl}`。

占位符只允许以下命名空间，且根名称区分大小写：

```text
${event.xxx}
${context.xxx}
${variables.xxx}
```

不要使用 `{appCode}`、`${appCode}` 或其他未声明根命名空间。模板标题、摘要、facts、detailMarkdown、actions 和收件人 value 都执行该检查。

收件人规则：

- `recipients` 可省略或为空，此时使用渠道配置的默认目标。
- 显式项的 `type` 支持 `USER`、`ROLE`、`EMAIL`。
- `value` 必须是非空固定值或受支持占位符。
- `role` 可选，支持 `TO`、`CC`、`BCC`，默认 `TO`。
- 非空显式列表至少包含一个 `TO`。
- `CC`、`BCC` 只对 EMAIL 渠道有效。
- 显式收件人只用于 EMAIL 或飞书 APP 机器人；钉钉、企业微信、通用 Webhook 和飞书 Webhook 使用渠道默认目标。
- 飞书显式收件人应使用 `TO`，并符合渠道配置的 `receiveIdType`。

本地 validator 可确定并检查 version、模板必填字段、数量上限、action URL、占位符语法和 recipients 结构。`configCode` 是否存在、渠道是否支持显式收件人、模板变量解析后的真实值仍以服务端为准。

## 本地校验边界

当前本地校验统一由 `rabetbase flow validate` 调用内置 TS validator，覆盖本文明确列出的本地结构、规范化、节点配置、NOTIFICATION 可确定规则和图拓扑规则。Skill 不维护额外的校验脚本。

本地校验不能确认：

- userId、roleId、datasetCode、pageId、页面 path、runtimePageUrl、scriptName、configCode 是否在目标环境真实存在或页面是否已经发布。
- `configCode` 对应的当前通知渠道是否支持指定模板或显式收件人。
- 服务端保存、编译、部署和 Runtime 执行是否成功。

这些结果必须分别通过资源发现命令、服务端 flow 命令响应和 Runtime 验证确认。
