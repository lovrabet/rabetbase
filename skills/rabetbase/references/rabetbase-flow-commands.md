# rabetbase flow commands

设计态流程定义命令，覆盖表单审批流与独立工作流。除 `validate` 外均需要认证和当前应用；`appCode` 由 CLI 工作区上下文注入。

## validate

```bash
rabetbase flow validate --file <path> [--write-normalized] --format compress
```

- 纯本地命令，不要求登录或 App Code。
- 返回 `data.valid`、`errors`、`warnings` 和 `normalized`。
- `--write-normalized` 仅在配置有效时原子写回规范结果。
- 校验时注入的临时 `flowKey`、`flowName` 和数据集绑定只存在于内存。
- 空值或默认 `approval.simple-flow.v1` 会从 `flowJson.schemaVersion` 规范化移除；其他版本保留并报 `Unsupported schemaVersion`。

Skill 直接执行本机的 `rabetbase flow validate` 命令，不维护额外的校验脚本或第二套规则。

## list

```bash
rabetbase flow list \
  [--flow-code <code>] \
  [--flow-name <name>] \
  [--flow-status <draft|published>] \
  [--dataset-code <code>] \
  [--flow-type <type>] \
  [--page <n>] [--page-size <n>] \
  --format compress
```

调用 `POST /smartapi/flow/page`。输出 `data.flows` 和分页信息，保留后续 detail/update/publish 所需的服务端 `id`。

## detail

```bash
rabetbase flow detail --id <flow-id> --format compress
rabetbase flow detail --id <flow-id> --output .rabetbase/flows/purchase.json
rabetbase flow detail --id <flow-id> --output <path> --force
```

调用 `POST /smartapi/flow/detail`。控制台结果保留服务端定义；`--output` 会重新投影、校验并规范化为可移植 FlowConfig。包含当前不支持的自定义表单字段时只读 detail 仍成功，但 `--output` 明确失败，避免生成丢配置文件。目标已存在时必须显式 `--force`，写入使用原子替换。

## create

```bash
rabetbase flow create --file <path> --dry-run --format compress
rabetbase flow create --file <path> --format compress
```

调用 `POST /smartapi/flow/create`。提交前强制 normalize + validate。请求体只包含外层可写字段、当前工作区 `appCode` 和最小 `flowJson`；文件中的遗留 `id`、`appCode`、状态和部署字段不会提交。

`flowType` 必填，支持 `FORM_FLOW`、`INDEPENDENT_FLOW`。`FORM_FLOW` 是表单审批流，禁止 `taskMode: HANDLE`；`INDEPENDENT_FLOW` 是工作流，可以同时包含 `APPROVAL` 和 `HANDLE`，且不能同时配置外层 `datasetCode` 或 `pageId`。`flowJson.pageMode` 只用于独立工作流，当前 CLI 只允许 `CUSTOM_PAGE`，缺省也按 `CUSTOM_PAGE`；`PLATFORM_FORM` 返回不支持错误。`INDEPENDENT_FLOW + CUSTOM_PAGE` 可选配置 `flowJson.startPageId` 和人工节点 `pageId`，两者仅用于导航。

Skill 直接提交完整 `nodes + edges` 拓扑，不读取或发送 `templateName`。旧本地文件中的该字段会在规范化时移除并产生 warning。

当前公开 Skill 创建独立工作流时使用 `INDEPENDENT_FLOW + 业务自定义页面`；表单审批流仍使用 `FORM_FLOW` 及其外层数据集/页面绑定。CLI 不解析 `formSchema/formDefinitions/formPermissions` 或节点 `form/formPermission`，这些字段会在 validate/create/update 中报错。

## update

```bash
rabetbase flow update --id <flow-id> --file <path> --dry-run --format compress
rabetbase flow update --id <flow-id> --file <path> --format compress
```

调用 `POST /smartapi/flow/update`。目标 id 只取自 `--id`，不信任文件内字段。服务端对已发布流程的修改限制原样作为远程错误返回。

## publish

```bash
rabetbase flow publish --id <flow-id> --dry-run --format compress
rabetbase flow publish --id <flow-id> --yes --format compress
```

调用 `POST /smartapi/flow/publish`，风险为 `high-risk-write`。`--dry-run` 不发请求；正式执行必须确认或显式 `--yes`。SmartCode 同步返回发布结果，成功响应即为该命令的完成口径。

## 风险与完成口径

| 命令 | 风险 | 完成口径 |
|---|---|---|
| `validate` | `read` | 本地 `data.valid=true` |
| `list` / `detail` | `read` | 服务端查询成功；detail 写文件时还需本地原子写入成功 |
| `create` / `update` | `write` | 服务端返回创建或更新后的定义 |
| `publish` | `high-risk-write` | 服务端返回发布成功 |

本地 `READY` 不等于远程成功。服务端拒绝 create、update 或 publish 时返回 `REMOTE_VALIDATION_FAILED`，不得把 dry-run 或本地校验结果描述为已发布。
