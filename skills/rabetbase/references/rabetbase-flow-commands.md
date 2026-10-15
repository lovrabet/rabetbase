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

## page-list

```bash
rabetbase flow page-list --id <existing-form-flow-id> --name <keyword> --page 1 --page-size 20 --format compress
rabetbase flow page-list --format compress
```

查询当前应用的 CREATE 标准页面，返回 `data.pages`、`paging` 和 `availabilityChecked`。`--id` 是当前 FORM_FLOW 的记录 ID，不是 flowCode；传入时过滤其他未删除表单流程的占用并保留自身绑定。新建尚无 ID 时省略，`availabilityChecked=false`，结果是未校验占用的候选；保存时仍做唯一性校验。无效、跨应用或独立流程 ID 由服务端拒绝。分页和名称筛选由服务端执行。

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

创建成功后返回 `data.flowUrl`，成功消息也会附上该地址。地址由当前 CLI 环境和 App Domain 自动生成，例如 daily 环境为 `https://daily.lovrabet.com/web-app/app/<appCode>/data/workflow?workflowId=<id>`；不要写死 daily 或 production 域名。

`flowType` 必填，支持 `FORM_FLOW`、`INDEPENDENT_FLOW`。`FORM_FLOW` 是表单审批流，禁止 `taskMode: HANDLE`；`INDEPENDENT_FLOW` 是工作流，可以同时包含 `APPROVAL` 和 `HANDLE`，且不能绑定外层 `datasetCode` 或 `pageId/pageIds`。`flowJson.pageMode` 只用于独立工作流，当前 CLI 只允许 `CUSTOM_PAGE`，缺省也按 `CUSTOM_PAGE`；`PLATFORM_FORM` 返回不支持错误。`INDEPENDENT_FLOW + CUSTOM_PAGE` 可选配置 `flowJson.startPath` 以及 `APPROVAL` / `END` 节点 `path`，这些字段仅用于导航。Skill 填写这些字段时必须使用 `page custom-detail` 确认为 `FORMAL` 的自定义页面所返回的完整 `runtimePageUrl`；未发布页面需先询问用户是否发布，不得直接填写 `custom-list` 中的候选地址。

Skill 直接提交完整 `nodes + edges` 拓扑，不读取或发送 `templateName`。旧本地文件中的该字段会在规范化时移除并产生 warning。

当前公开 Skill 创建独立工作流时使用 `INDEPENDENT_FLOW + 业务自定义页面`；表单审批流使用 `FORM_FLOW` 和外层 `pageIds` 页面数组。旧 `pageId` 单值兼容，旧 `datasetCode` 保存时忽略。CLI 不解析 `formSchema/formDefinitions/formPermissions` 或节点 `form/formPermission`，这些字段会在 validate/create/update 中报错。

页面绑定使用 `pageIds: [101, 102]`，详情输出保留完整数组；响应 `pageId` 是 CSV 字符串，不用于写入。更新时省略两个页面字段保留原绑定，显式 `pageIds: []` 清空绑定。每个页面只能属于一个未删除 FORM_FLOW，草稿和停用也占用。仅 `FORM_FLOW` 的运行态表单数据需先 create，再以实际 `pageId` 和已有 `dataId` 显式启动；`datasetCode` 是启动接口可选的解析兜底参数，不是所有接口的可选字段，普通 create 仍需实际目标模型。见[表单发起](../guides/form-flow-start.md)。

## update

```bash
rabetbase flow update --id <flow-id> --file <path> --dry-run --format compress
rabetbase flow update --id <flow-id> --file <path> --format compress
# 显式 pageIds（含 []）改变绑定集合时，正式提交须断言原绑定
rabetbase flow update --id <flow-id> --file <path> --expected-page-ids '[101,102]' --format compress
```

调用 `POST /smartapi/flow/update`。目标 id 只取自 `--id`，不信任文件内字段。服务端对已发布流程的修改限制原样作为远程错误返回。

显式绑定更新先查询详情。dry-run 返回真实 `before` 和 `pageBindings.before/after/added/removed`，可不传预期数组；有新增或移除页面时正式执行须 `--expected-page-ids '<JSON数组>'` 与刚读取的原绑定按集合相符，`[]` 表示预期原来无绑定。缺失、非法或不匹配时不发更新请求，重新读取并审阅变化。该参数仅用于 CLI 提交前检查，不发送给后端，也不提供服务端原子 CAS；读写间仍可能并发变化。省略绑定字段的 update 保留原绑定；显式数组与原集合一致、仅修改节点时也不要求该参数。

更新已发布定义后回到草稿，需要重新发布。曾发布的定义禁止修改 `flowType`，草稿或停用不解除限制。

## publish

```bash
rabetbase flow publish --id <flow-id> --dry-run --format compress
rabetbase flow publish --id <flow-id> --format compress
```

调用 `POST /smartapi/flow/publish`，风险为普通 `write`。`--dry-run` 不发请求，建议正式执行前先审阅预演结果。SmartCode 同步返回发布结果，成功响应即为该命令的完成口径。

发布成功后同样返回 `data.flowUrl` 并写入成功消息。Agent 应优先使用该结构化字段向用户提供流程页面地址。

发布后回读 detail 核对 `flowStatus=published` 和完整页面绑定；当前详情不公开 enabled，启用状态通过平台或实际运行入口核验。publish 不自动启用停用流程，不创建页面/菜单/业务审批字段，也不检查角色成员和脚本在运行时必定可用；交付可发起的表单流程应先确认至少一个真实 CREATE 页面及有效菜单，再按创建工作流完成运行验收。

## 风险与完成口径

| 命令 | 风险 | 完成口径 |
|---|---|---|
| `validate` | `read` | 本地 `data.valid=true` |
| `list` / `detail` | `read` | 服务端查询成功；detail 写文件时还需本地原子写入成功 |
| `create` / `update` | `write` | 服务端返回创建或更新后的定义 |
| `publish` | `write` | 服务端返回发布成功 |

本地 `READY` 不等于远程成功。服务端拒绝 create、update 或 publish 时返回 `REMOTE_VALIDATION_FAILED`，不得把 dry-run 或本地校验结果描述为已发布。
