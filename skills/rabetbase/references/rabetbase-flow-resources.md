# 流程定义资源发现

FlowConfig 中的真实标识必须来自 CLI 查询，不得由 Agent 猜测。

## 映射

| FlowConfig 字段 | 发现命令 | 选中值 |
|---|---|---|
| `FORM_FLOW` 外层 `pageIds` | `flow page-list`；已有流程传 `--id <flow-id>` | CREATE 页面的 `id`，保留所有选中项 |
| 表单页面菜单核对 | `menu list` | 与选中页面对应的 `pageId`、有效 `path` 和菜单类型 |
| 本次表单发起的 `datasetCode` | 实际页面配置、`dataset list/detail` | 当前业务记录的数据集 `code`，不写成流程定义绑定 |
| `flowJson.startPath` | `page custom-list` + `page custom-detail` | 已发布自定义页面的完整 `runtimePageUrl` |
| `APPROVAL.path` / `END.path` | `page custom-list` + `page custom-detail` | 已发布自定义页面的完整 `runtimePageUrl` |
| `APPROVAL.assignee.userIds` | `flow runtime-user-search` | 运行态 `users[].userId` |
| `APPROVAL.assignee.roleId` | `flow runtime-role-list` | 运行态角色 `id` |
| 角色成员核对 | `flow runtime-role-user-list` | 运行态只读成员清单 |
| `transferCandidates.userIds` | `flow runtime-user-search` | 运行态 `users[].userId` |
| `APPROVAL.cc.recipientUserIds` / `END.cc.recipientUserIds` | `flow runtime-user-search` | 运行态 `users[].userId` |
| `SCRIPT.scriptName` | `bff list --type ENDPOINT` | Endpoint `functionName` |
| `APPROVAL.timeout.scriptName` | `bff list --type ENDPOINT` | Endpoint `functionName` |
| `NOTIFICATION.config.configCode` | `notification config-list --all` | `configs[].configCode` |
| `NOTIFICATION.config.recipients[].value`（固定 USER） | `flow runtime-user-search` | 运行态 `users[].userId` |
| `NOTIFICATION.config.recipients[].value`（固定 ROLE） | `flow runtime-role-list` | 运行态角色 `id` |

`page custom-list` 只用于筛选候选页面；填写 `startPath` 和节点 `path` 前必须再执行 `page custom-detail --id <pageId>`。只有 `data.status` 为 `FORMAL` 时，才使用详情返回的完整 `runtimePageUrl`；页面未发布时保持字段缺失并询问用户是否先发布。不得使用数值 `pageId`、菜单名称、菜单原始 `path`、`pageUrl`、`editPageUrl` 或手工拼接地址代替。占位符形式的通知收件人来自运行态变量，不需要资源查询；固定 EMAIL 收件人由用户明确提供，不推断邮箱地址。

## 表单流程页面发现

```bash
# 新建前查候选；结果尚未过滤其他流程占用
rabetbase flow page-list --name <keyword> --page 1 --page-size 20 --format compress
# 编辑已有流程；ID 来自 flow create/list/detail，不是 flowCode
rabetbase flow page-list --id <flow-id> --name <keyword> --page 1 --page-size 20 --format compress
rabetbase menu list --format compress
```

- 使用 `data.pages[].id` 写入完整 `pageIds` 数组；HTTP 数字字符串先按十进制转换并确认是正的 JavaScript 安全整数，再写为 JSON 数字，不截断、不四舍五入。检查 `paging`，不能把一页结果当成全部页面。
- 新建查询的 `availabilityChecked=false`，页面是否占用由保存时最终校验；草稿和停用的未删除 FORM_FLOW 仍占用页面。
- 有 ID 时查询排除其他流程占用，保留自身绑定。仍需核对页面确实存在、类型为 CREATE 及菜单路由有效；筛选成功不表示运行态已经可发起。
- 已知真实页面 ID 可直接 create。需要先取得 ID 再筛可用页面时，先保存有合法拓扑的未绑定草稿，再查询并 update；不得用空图占位。
- 一个流程可以绑定不同数据集的 CREATE 页面。`datasetCode` 不决定定义绑定，也不能从数组首项代替本次实际页面。
- 不用 `menu list` 代替绑定候选查询；菜单用于核对页面入口。发布只标记明确绑定页面已有菜单为 `process`，不会自动给同数据集其他页面绑定流程或创建菜单。

## 搜索运行态应用用户

```bash
rabetbase flow runtime-user-search --keyword <text> [--page <n>] [--page-size <n>] --format compress
```

- 只查询当前运行态应用已关联的用户，对 userId、nickname、username、displayName、email 做不区分大小写的包含匹配。
- 输出 `scope: "runtime"`、安全字段 `userId`、`username`、`nickname`、`displayName`、`email`、`status` 和分页信息。
- 不输出手机号、访问密钥或无关身份字段。
- 多个候选时必须由用户确认；不要默认取第一项。

不要用 `role`、`app members-list` 或 `tenant members-list` 的人员结果填写审批 FlowConfig。

## 查询运行态角色

```bash
rabetbase flow runtime-role-list [--keyword <role-name>] [--page <n>] [--page-size <n>] --format compress
```

- 查询当前应用运行态 `env=RUNTIME` 角色，`--keyword` 由服务端按角色名模糊过滤。
- 输出 `scope: "runtime"`、角色 `id/name/type`、备注和人数/权限数安全摘要，不输出权限矩阵。
- `APPROVAL.assignee.roleId` 只能使用这里返回的运行态角色 ID；不要使用 `role list/detail` 的配置端角色 ID。

## 查看运行态角色成员

```bash
rabetbase flow runtime-role-user-list --role <role-id-or-name> [--page <n>] [--page-size <n>] --format compress
```

- 在当前应用运行态角色成员分组中按 ID 或精确名称解析角色。
- 角色名称不唯一时要求改用角色 ID。
- 输出 `scope: "runtime"`；只读，不修改成员关系。

## 查询通知配置

```bash
rabetbase notification config-list --all --format compress
rabetbase notification config-list --type EMAIL --format compress
```

- `--all` 查询 `EMAIL`、`FEISHU`、`DINGTALK`、`WECOM`、`WEBHOOK` 的安全摘要。
- `--all` 与 `--type` 互斥；未指定时继续默认查询 `EMAIL`。
- 只使用同一条记录的 `configCode`，不要用渠道名、URL、dataset code 或凭据代替。

## 查询脚本

```bash
rabetbase bff list --type ENDPOINT --name <keyword> --format compress
```

只把确认后的同应用、非个人 Endpoint `functionName` 写入 `SCRIPT.scriptName` 或 `APPROVAL.timeout.scriptName`。COMMON、HOOK、WORKFLOW 脚本类型或展示名称不能替代 Endpoint 函数名。SCRIPT 节点不配置审批人；执行身份由服务端处理，不向脚本参数注入用户或系统标记。

## 确认规则

- 无候选：返回 `NEEDS_USER_CONFIRMATION` 并请求更精确关键词。
- 多候选：展示稳定 ID、名称和必要上下文，请用户选择。
- 唯一候选：可以推荐，但身份、角色、脚本和通知配置仍需用户确认。
- 当前 DSL 无法表达需求：返回 `NEEDS_DSL_EXTENSION`，不要伪造配置。
