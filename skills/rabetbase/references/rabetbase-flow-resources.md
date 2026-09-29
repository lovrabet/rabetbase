# 流程定义资源发现

FlowConfig 中的真实标识必须来自 CLI 查询，不得由 Agent 猜测。

## 映射

| FlowConfig 字段 | 发现命令 | 选中值 |
|---|---|---|
| 外层 `datasetCode` | `dataset list`、`dataset detail` | Dataset `code` |
| 外层 `pageId` | `menu list` | 与目标页面对应的 `pageId` |
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

只把确认后的 Endpoint `functionName` 写入 `SCRIPT.scriptName` 或 `APPROVAL.timeout.scriptName`。COMMON、HOOK 或展示名称不能替代 Endpoint 函数名。

## 确认规则

- 无候选：返回 `NEEDS_USER_CONFIRMATION` 并请求更精确关键词。
- 多候选：展示稳定 ID、名称和必要上下文，请用户选择。
- 唯一候选：可以推荐，但身份、角色、脚本和通知配置仍需用户确认。
- 当前 DSL 无法表达需求：返回 `NEEDS_DSL_EXTENSION`，不要伪造配置。
