# bff push

把本地 Backend Function 脚本上传到远端。

> **风险等级：write** — 建议先使用 `--dry-run` 预览。

## 命令

```bash
rabetbase bff push --format json
rabetbase bff push --type ENDPOINT --name getUserList --format json
rabetbase bff push --type ENDPOINT --name getUserList --dry-run --format json
rabetbase bff push --type HOOK --name beforeFilter --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--type <type>` | string | 否 | — | 仅推指定类型：`COMMON` / `ENDPOINT` / `HOOK` |
| `--name <name>` | string | 否 | — | 仅推指定函数名；传时必须搭配 `--type` |
| `--force` | boolean | 否 | — | 忽略 hash 保护强制推送 |
| `--dry-run` | boolean | 否 | — | 仅预览将要推送的本地脚本 |
| `--format <fmt>` | string | 否 | `pretty` | 输出格式 |

## 输出

返回：
- `uploaded`
- `skipped`
- `conflicts`
- `failed`

常见 `skipped` 原因：
- `unchanged`

`--dry-run` 时返回预览对象，包含本地 `filePath`、`alias`、`datasetCode`、Hook operation/node、warning、目标模式（`create` / `update`）和预期状态（如 `unchanged` / `would_push`）。

`conflicts[].code=BFF_REMOTE_VERSION_CHANGED` 或 `BFF_REMOTE_VERSION_MISSING` 表示本地 lock 不能安全更新远端脚本。保留本地脚本，先执行 `nextAction.command` 的 `bff detail` 取得远端源码，审阅并合并后再重试；只在明确放弃本地修改时才考虑 `--force`。这类同步分歧不进入 `failed`，其 `nextAction` 仅供审阅后执行。

## 推送前检查

`bff push` 负责同步脚本，不等价于业务逻辑验证。推送前按 [`backend-function.md`](../guides/backend-function.md) 自检，尤其确认：

- `DB_TABLE` 数据集优先通过 `context.client.models.byTable("<物理表名>")` 访问；同名表跨 dblink 时传入已确认的 `{ dblinkId }`
- `METADATA` 数据集使用 `"dataset_" + 数据集 code`；`DB_TABLE` 使用该形式时属于兼容路径
- Backend Function 的 Custom SQL 默认使用 `context.client.sql.byName("<唯一 SQL 名>").execute({ params })`；名称不存在或重名时先处理 `SQL_NAME_NOT_FOUND` / `SQL_NAME_AMBIGUOUS`
- 写入字段、必填字段、枚举 `options[].value` 来自当前 `dataset detail`
- `filter()` 结果从 `.tableData` 读取
- `create()` 返回新记录 ID，不访问 `created.id`
- Backend Function 中没有使用前端 SDK 初始化能力，如 `createClient`、`registerModels`
- 通知型 Backend Function 只使用已确认的 `configCode` / `audiences` / `message`，没有渠道地址、密钥、旧 MANUAL 参数或 `${...}` 模板表达式

## 提示

- 推送前先跑 `bff status`
- `--dry-run` 不会上传远端，也不会改 lock
- 精确推单个函数时用 `--type + --name`
- HOOK 优先从 SDK alias 目录解析 Dataset；没有 alias 时 Dataset code 目录保持可读
- 正式上传仍使用平台真实 Dataset ID，目录名不会替代远端 Dataset 身份
- 已有 HOOK 的 Dataset 元数据缺失时，CLI 回读脚本 ID、应用、名称、类型、版本及唯一挂载关系；全部与本地 lock 匹配才允许按脚本 ID 更新且不重绑，返回 `EXISTING_HOOK_BINDING_PRESERVED` warning；新建 HOOK 不走此路径
- 无法解析或查不到目标 Dataset 的 HOOK 单独进入 `failed`，其余可确认目标的脚本继续上传；部分失败时命令返回 `ok: false`，必须检查 `uploaded` 与 `failed`，不要整批盲目重试
- 预演中无法确认目标的脚本显示 `status: failed` 与 `error`，不会上传或更改 lock
- alias 与 Dataset code 命名空间碰撞、Dataset 映射漂移或同一 Dataset 出现多个本地目录会在首次远端写入前阻断
- HOOK 可挂载 `DB_TABLE` 或 `METADATA` 数据集；METADATA 不支持 SQL / aggregate 路径，脚本中使用平台返回的标准数据操作
- 推送成功只代表脚本配置已同步到平台；需要运行验证时，按[创建流程的运行验证步骤](../guides/bff-creation-workflow.md#9-运行态-smoke按需)确认目标、权限及函数副作用后执行
- 已确认独立部署并需验证，或出现平台到业务环境的同步疑点时，读取[独立部署指南](../guides/independent-deployment.md)；普通 push 不额外查询部署标识

## 参考

- [SKILL.md](../SKILL.md)
- [bff-creation-workflow.md](../guides/bff-creation-workflow.md)
- [backend-function.md](../guides/backend-function.md)
- [conflict-detection.md](../guides/conflict-detection.md)
