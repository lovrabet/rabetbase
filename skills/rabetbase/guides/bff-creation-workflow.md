# Backend Function 工作流规则

前置知识：`backend-function.md`、`data-api-guidelines.md`

## 核心原则

同一平台、同一应用的同一 BFF，以平台数据库中最新成功保存的内容及版本为远端基线。Git 分支不隔离该资源，其他开发者从不同分支执行 `bff push` 也可能更新它；`git pull` 不能替代获取平台最新基线。

每轮修改已有脚本前，先获取平台最新源码及版本，与本地改动比较、必要时合并，不能直接覆盖未提交修改。正式 push 继续使用版本冲突保护，防止覆盖获取基线后其他开发者或分支提交到平台的更新。

同一应用、连接和资源下，已确认且仍有效的需求与依赖事实可以复用；源码基线仅复用本轮修改开始前已从平台取得的最新内容，不能用上一轮或历史会话中的源码代替。本轮编辑和测试期间不因每次本地修改重复拉取；出现远端变化证据、真实冲突或写入结果未知时，按对应规则重新核对。按任务影响选择步骤，不按改动行数或文件数量省略必要验证。遵守项目工作目录约定，不通过更换目录或配置绕过冲突。

## 工作流

```
确认需求与已有事实 → [按需]发现资源及校验依赖 → 确认平台基线 → 编写本地脚本 → 自检与相关测试 → status → dry-run → push → [按需]运行态 smoke
```

### 0. 速查公共函数（按需）
新建能力或需要寻找可复用依赖时，执行 `rabetbase bff list --type COMMON --format json`。对拟复用但契约尚未确认的函数，用 `rabetbase bff detail --id <id> --format json` 确认入参、返回值和副作用；已有有效依赖事实且本次不改变依赖时不重复发现。

### 1. 确认需求
写码前必须明确：类型（ENDPOINT/HOOK/COMMON）、函数名、入参、返回结构，以及是否涉及数据集或消息通知。先复用已确认事实，无法从现有上下文确认的业务决策再问用户。通知型 Backend Function 还必须确认当前应用已有的 `configCode`、接收对象、标题/摘要和是否允许执行真实发送；不能用示例编码代替真实配置。

### 2. 校验依赖事实
Backend Function 涉及数据集时，核对本次依赖的字段名、类型、必填字段、枚举值、关联关系。事实缺失、依赖变化或已有证据不再有效时，执行 `rabetbase dataset detail --code <数据集编码> --format json`（或 `compress`）；已确认且未变化的事实直接复用。禁止凭经验猜字段名，禁止把 Demo 或历史案例里的字段、表名、枚举值复制到当前脚本。

不读写数据集的纯消息通知 ENDPOINT 可以不依赖数据集；业务明确要求在数据集操作执行前发送预通知或告警时使用 Before HOOK；作为数据集操作成功后副作用的通知，只有响应结果已包含通知所需字段时才使用 After HOOK。三者都必须按 [`backend-function.md`](backend-function.md) 的“消息通知扩展”核对 `configCode`、`audiences` 和 `message`。先执行以下只读命令获取当前应用的 EMAIL 配置：

```bash
rabetbase notification config-list --type EMAIL --format compress
```

从 `data.configs[]` 按 `configName` / `description` 选择配置，并使用同一项的 `configCode`。命令不会输出 `channelConfig`、`endpointUrl` 或通知凭据。没有结果或存在多个候选且业务目标不明确时，停下向用户确认；不得猜测。不要把 dataset 级通知通道的 `channelCode` 当成 Backend Function 所需的应用级 `configCode`。

Backend Function HOOK 可挂载 `DB_TABLE` 或 `METADATA` 数据集，具体 operation 以平台返回为准。`DB_TABLE` 使用 `context.client.models.byTable("<物理表名>")`；同名物理表来自多个 dblink 时，必须从 `rabetbase db list` 取得真实 ID 后传入 `{ dblinkId }`，不要让运行时任选。`METADATA` 没有物理表，且不支持 SQL / aggregate 路径，使用 `` context.client.models[`dataset_${datasetCode}`] `` 的标准操作能力。

常用字段投影：

```bash
rabetbase dataset detail --code <数据集编码> --format compress \
  --jq '.data.fields[] | {name, displayName, type, required, options}'
```

写入前必须确认：
* 业务必填字段：`data.fields[].required === true`，平台自动维护字段除外
* 枚举/选择字段：写入 `options[].value`，不要写展示用 `label`
* 外键字段：同库关系从 `data.relations[]` 或 `dataset relations` 确认；跨库关系使用 `dataset cross-relation-list`，不能用同库关系列表代替

涉及跨连接读取或拼接时，先读 [跨库 BFF 查询与拼接](cross-database-bff.md)，确认完整键、基数、目标字段用途、各端授权和读取预算。业务关系说明与平台登记事实分别核对；冲突时报告差异，不能自动采用平台关系。复合键不能拆成独立的单字段 Relation。

### 3. 获取本轮平台基线
* 新建 → 跳过
* 修改已有 → 已知 ID 时执行 `rabetbase bff detail --id <id> --format json` 获取最新源码及版本；ID 未知时先 list 定位。本轮已通过 detail 或 pull 取得同一目标最新基线时不重复读取。需要更新本地副本时按冲突规则 pull 或合并，不覆盖未审阅的本地改动
* 目标不确定 → 先 list 定位，再读取所选资源详情

新建时命中同名脚本且用户意图不明确，再确认修改还是另起新名；用户已明确要求修改目标函数时直接继续。

### 4. 编写脚本（规范路径）
新建脚本应使用 **`rabetbase bff create`**，在 **`.rabetbase/bff/<appCode>/...`** 下生成脚手架后再编辑（路径与 `bff status` / `bff push` 一致）。**不要**在 `src/` 或仓库任意目录手写 Backend Function 再期望被 CLI 识别。
已有脚本仅在上述 Backend Function 树内修改；与 `backend-function.md` 中的目录约定一致。

通知需要在数据集 `create` / `update` / `delete` 执行前明确预告，并且通知失败应阻止本次操作时，选择 Before HOOK；通知由数据集操作成功触发，且响应结果已包含通知所需字段时，选择 After HOOK；响应结果不包含通知所需字段时，选择能在写入前读取并暂存必要字段、在成功后发送通知的受控 `ENDPOINT`。三者都使用 `await context.client.extension.execute("notification", "send", ...)`，并由 runtime 注入可信 `appCode` 和当前用户；不要把 `appCode`、渠道地址或密钥作为外部参数透传。

### 5. 自检
按变更影响运行相关行为测试，覆盖本次目标及受影响的既有约束；涉及权限、写入或外部副作用时验证相关入口。测试证据必须对应最终待推送内容及依赖，合并后内容变化须重跑受影响测试。

* 方法名正确
* 单条查询用 `getOne`
* `DB_TABLE` 优先使用 `context.client.models.byTable("<物理表名>")`；同名表存在多个 dblink 时补 `{ dblinkId: <已确认 ID> }`
* `METADATA` 使用 `"dataset_" + 数据集 code`；`DB_TABLE` 仅在兼容调用时使用该形式
* METADATA 数据集的 Backend Function / HOOK 不走 SQL 或 aggregate；只使用平台返回的标准数据操作
* `filter()` 结果从 `.tableData` 读取，不是 `.list`
* `create()` 返回新记录 ID，不是完整对象；不要访问 `created.id`
* `batchCreate()` 返回新记录 ID 数组；入参直接使用非空对象数组，不使用 `{"items":[...]}` 包装
* 批量更新使用 `update({ id: [...] })`；不存在 `batchUpdate()`，也不传记录数组
* 枚举/选择字段写入 `options[].value`，不是展示 `label`
* Backend Function 中 `sql.execute` 返回数组，不是 `{ execSuccess, execResult }`
* Backend Function 中 Custom SQL 默认使用 `context.client.sql.byName("<唯一 SQL 名>").execute({ params })`；名称不唯一时先处理 `SQL_NAME_AMBIGUOUS`，不要回退任意 `sqlCode`
* 没有在 Backend Function 中使用前端 SDK 初始化能力，如 `createClient`、`registerModels`
* 参数校验、错误处理、脱敏
* 中文 JSDoc 已写清根请求参数、实际 `params.<字段名>`、返回值；显式抛出异常时包含 `@throws`
* 依赖数据集、调用 BF、执行 SQL、副作用维护项与实际代码一致，无依赖时明确填写“无”
* 通知型 Backend Function 只传 `configCode` / `audiences` / `message`，并且没有 `${...}` 模板表达式、旧 MANUAL 参数或渠道密钥
* 通知型 ENDPOINT 限制调用者可传的字段、`configCode` 和接收对象范围，不形成任意通知转发器
* 通知型 Before HOOK 使用“即将执行”或“准备执行”的消息语义，`await` 发送后返回原始 `params`；不直接返回通知扩展结果，并明确接受“通知已发送但后续业务仍可能失败”
* 通知型 After HOOK 只使用业务接口响应结果或固定可信规则派生接收对象与消息；若通知失败或超时，按业务操作可能已完成、通知状态未知处理，不得自动重试原业务请求

### 6. 检查本地状态
执行 `rabetbase bff status --format json`：
* 确认新增脚本进入 `added`
* 确认修改脚本进入 `modified`
* 若状态异常，先不要推送

### 7. 预览推送
执行 `rabetbase bff push --type <type> --name <name> --dry-run --format json`：
* 查看本次是 `create` 还是 `update`
* 检查目标 `lockKey`、`filePath` 和预期状态
* dry-run 不会上传远端，也不会改 lock

### 8. 推送到平台
执行 `rabetbase bff push --type <type> --name <name> --format json`：
* 成功项进入 `uploaded`
* 未变更项进入 `skipped: unchanged`
* 同步分歧进入 `conflicts`；逐项审阅 `lockKey`、`code` 和 `nextAction`，不要将它们说成失败
* 失败项进入 `failed`

push 失败或响应丢失时，按[写入结果与恢复动作](conflict-detection.md#写入结果与恢复动作)处理。

### 9. 运行态 smoke（按需）
`rabetbase bff push` 的成功保存结果只证明对应脚本配置已写入平台。若需求要求确认运行效果，核对目标应用后执行验证；非独立部署应用沿用平台内的普通验证流程。已确认独立部署，或出现平台到业务环境的同步疑点时，读取[独立部署指南](independent-deployment.md)。

通过目标浏览器请求验证，或在运行 CLI 已配置到对应业务环境、认证与权限适用、函数契约及参数已确认且副作用获准后执行：

```bash
lovrabet bff exec --appcode <appCode> --name <functionName> --params '<json>' --format compress
```

边界：
* 通知型 Backend Function 的 smoke 会真实发送外部消息；必须通过已确认上下文或显式 `--appcode` 锁定同一 app，执行前向用户展示 app、函数名、`configCode`、接收对象和消息摘要并取得明确确认
* `lovrabet bff detail` 只确认运行契约和版本，不返回脚本源码；通知参数必须来自本地已审查脚本或明确业务契约，不能按函数名猜
* 通知执行超时或客户端未拿到结果时，先按“状态未知”处理；不得自动重试，避免重复发送
* `lovrabet` CLI 不可用、未配置或无权限时，明确记录“运行态 smoke 未执行”，不要把它写成 `rabetbase` 验证已通过

### 10. 本地文件
脚本内容直接保存在本地文件中，纳入 Git 管理。路径遵循 `.rabetbase/bff/<appCode>/` 目录约定（详见 `backend-function.md`）：
* ENDPOINT → `.rabetbase/bff/<appCode>/ENDPOINT/<name>.js`
* HOOK → `.rabetbase/bff/<appCode>/HOOK/<alias-or-datasetCode>/<operationType>/<functionNode>/<name>.js`
* COMMON → `.rabetbase/bff/<appCode>/COMMON/<name>.js`

`bff create` 使用 Dataset alias；没有可用 alias（例如已删除 Dataset 遗留的 HOOK）时，CLI 使用 Dataset code 目录。`bff pull` 会将 lock 跟踪的旧表名或过期 alias 目录安全迁移到当前 SDK alias；当前没有 alias 时迁移到 Dataset code。目标目录冲突、映射歧义或无法安全迁移时，命令会失败而不会覆盖本地脚本；先用 `--dry-run` 查看迁移计划。

## 冲突处理

若返回 `conflicts`：
* 告知用户冲突的 `lockKey`、`code` 和 `nextAction`
* `BFF_LOCAL_UNSYNCED`：审阅本地脚本；确认本地版本应生效后，执行该项精确 `bff push` 命令更新远端
* `BFF_REMOTE_VERSION_CHANGED` / `BFF_REMOTE_VERSION_MISSING`：保留本地脚本，先执行该项 `bff detail` 命令读取远端源码，合并后再重试 push
* 不要自动使用 `--force`、盲目重试整批脚本，或将同步分歧报告为失败

若返回 `failed`：
* 告知用户失败的 `lockKey` 和错误原因
* 不要假装成功
* 已成功推送的其他脚本不会自动回滚

## 验证与收尾

优先消费 CLI 返回的保存及锁更新结果。最终内容通过相关测试、平台保存结果明确、本地同步状态正确且本次要求的运行验证完成后，报告结果并结束。无法完成的环节明确标为未验证，不把保存成功当成运行验收成功。

仅在内容、依赖、配置或目标变化，出现冲突、失败、未知结果或新风险时，追加能解决该问题的检查。没有新证据时不重复同内容测试、远端回读或 push；这不限制对真实失败的继续调查。

## Backend Function 语义差异

| 场景 | 前端 SDK | Backend Function (context.client) |
|------|---------|---------------------|
| SQL 调用 / 返回值 | `sql.execute({ sqlCode, params })`，返回 `{ execSuccess, execResult }` | 默认使用 `sql.byName(sqlName).execute({ params })`，直接返回数组；`sql.execute({ sqlCode, params })` 为兼容调用方式 |
| 数据集访问 | 可通过初始化/生成代码使用 alias | `DB_TABLE` 默认使用 `models.byTable(tableName, { dblinkId? })`；`METADATA` 使用 `"dataset_" + 数据集 code`，`DB_TABLE` 也支持该兼容调用方式 |
| `filter()` 返回 | `tableData` 为列表数据 | `tableData` 为列表数据，不是 `list` |
| `create()` 返回 | 以 SDK 文档/类型为准 | 新记录 ID，不是完整对象 |
| `batchCreate()` 返回 | 以 SDK 文档/类型为准 | 新记录 ID 数组；直接传非空对象数组 |
| 批量更新 | 以 SDK 文档/类型为准 | `update({ id: [...] })`；不存在 `batchUpdate()` |
| SDK 初始化能力 | `createClient` / `registerModels` | 不可用；`context.client` 由平台注入 |
| 前端调 Backend Function | `client.bff.execute({ scriptName, params })` 返回业务数据 | — |
| 发送应用级通知 | — | `context.client.extension.execute("notification", "send", { configCode, audiences, message })` |
