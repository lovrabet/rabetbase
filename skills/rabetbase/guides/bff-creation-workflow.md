# BFF 工作流规则

前置知识：`backend-function.md`、`data-api-guidelines.md`

## 核心原则

平台是唯一 source of truth。修改已有脚本时，从平台拉取最新内容。

## 工作流

```
速查公共函数 → 确认需求 → 校验字段 → [按需]查平台 → 编写本地脚本 → 自检 → status → dry-run → push/pull → [按需]运行态 smoke
```

### 0. 速查公共函数
执行 `rabetbase bff list --type COMMON --format json` 查看已有公共函数。如有可复用的工具函数，先用 `rabetbase bff detail --id <id> --format json` 确认入参、返回值和副作用，避免按名称猜测。

### 1. 确认需求
写码前必须明确：类型（ENDPOINT/HOOK/COMMON）、函数名、入参、返回结构，以及是否涉及数据集或消息通知。缺失则先问用户。通知型 BFF 还必须确认当前应用已有的 `configCode`、接收对象、标题/摘要和是否允许执行真实发送；不能用示例编码代替真实配置。

### 2. 校验依赖事实
BFF 涉及数据集时，执行 `rabetbase dataset detail --code <数据集编码> --format json`（或 `compress`）确认字段名、类型、必填字段、枚举值、关联关系。禁止凭经验猜字段名，禁止把 Demo 或历史案例里的字段、表名、枚举值复制到当前脚本。

不读写数据集的纯消息通知 ENDPOINT 可以不依赖数据集；业务明确要求在数据集操作执行前发送预通知或告警时使用 Before HOOK；作为数据集操作成功后副作用的通知，只有响应结果已包含通知所需字段时才使用 After HOOK。三者都必须按 [`backend-function.md`](backend-function.md) 的“消息通知扩展”核对 `configCode`、`audiences` 和 `message`。先执行以下只读命令获取当前应用的 EMAIL 配置：

```bash
rabetbase notification config-list --type EMAIL --format compress
```

从 `data.configs[]` 按 `configName` / `description` 选择配置，并使用同一项的 `configCode`。命令不会输出 `channelConfig`、`endpointUrl` 或通知凭据。没有结果或存在多个候选且业务目标不明确时，停下向用户确认；不得猜测。不要把 dataset 级通知通道的 `channelCode` 当成 BFF 所需的应用级 `configCode`。

BFF HOOK 可挂载 `DB_TABLE` 或 `METADATA` 数据集，具体 operation 以平台返回为准。`METADATA` 数据集不支持 SQL / aggregate 路径；脚本中应使用 `` context.client.models[`dataset_${datasetCode}`] `` 的标准操作能力。

常用字段投影：

```bash
rabetbase dataset detail --code <数据集编码> --format compress \
  --jq '.data.fields[] | {name, displayName, type, required, options}'
```

写入前必须确认：
* 业务必填字段：`data.fields[].required === true`，平台自动维护字段除外
* 枚举/选择字段：写入 `options[].value`，不要写展示用 `label`
* 外键字段：从 `data.relations[]` 或 `dataset relations` 确认真实关系

### 3. 查平台（按需）
* 新建 → 跳过
* 修改已有 → 执行 `rabetbase bff list --format json`（ENDPOINT 或 COMMON） + `rabetbase bff detail --id <id> --format json` 取 `id` 和最新内容
* 不确定 → 查一下

命中同名脚本时，停下问用户：修改还是另起新名。

### 4. 编写脚本（规范路径）
新建脚本应使用 **`rabetbase bff create`**，在 **`.rabetbase/bff/<appCode>/...`** 下生成脚手架后再编辑（路径与 `bff status` / `bff push` 一致）。**不要**在 `src/` 或仓库任意目录手写 BFF 再期望被 CLI 识别。
已有脚本仅在上述 BFF 树内修改；与 `backend-function.md` 中的目录约定一致。

通知需要在数据集 `create` / `update` / `delete` 执行前明确预告，并且通知失败应阻止本次操作时，选择 Before HOOK；通知由数据集操作成功触发，且响应结果已包含通知所需字段时，选择 After HOOK；响应结果不包含通知所需字段时，选择能在写入前读取并暂存必要字段、在成功后发送通知的受控 `ENDPOINT`。三者都使用 `await context.client.extension.execute("notification", "send", ...)`，并由 runtime 注入可信 `appCode` 和当前用户；不要把 `appCode`、渠道地址或密钥作为外部参数透传。

### 5. 自检
* 方法名正确
* 单条查询用 `getOne`
* BFF 模型键使用 `"dataset_" + 32 位数据集 code`
* METADATA 数据集的 BFF / HOOK 不走 SQL 或 aggregate；只使用平台返回的标准数据操作
* `filter()` 结果从 `.tableData` 读取，不是 `.list`
* `create()` 返回新记录 ID，不是完整对象；不要访问 `created.id`
* `batchCreate()` 返回新记录 ID 数组；入参直接使用非空对象数组，不使用 `{"items":[...]}` 包装
* 批量更新使用 `update({ id: [...] })`；不存在 `batchUpdate()`，也不传记录数组
* 枚举/选择字段写入 `options[].value`，不是展示 `label`
* BFF 中 `sql.execute` 返回数组，不是 `{ execSuccess, execResult }`
* 没有在 BFF 中使用前端 SDK 初始化能力，如 `createClient`、`registerModels`
* 参数校验、错误处理、脱敏
* 中文 JSDoc 已写清根请求参数、实际 `params.<字段名>`、返回值；显式抛出异常时包含 `@throws`
* 依赖数据集、调用 BF、执行 SQL、副作用维护项与实际代码一致，无依赖时明确填写“无”
* 通知型 BFF 只传 `configCode` / `audiences` / `message`，并且没有 `${...}` 模板表达式、旧 MANUAL 参数或渠道密钥
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
* 失败项进入 `failed`

### 9. 运行态 smoke（按需）
`rabetbase bff push` 只证明脚本配置已同步到平台管理侧。若本轮需求要求确认最终运行效果，应显式进入运行验证，例如在已安装并配置运行 CLI 的环境中执行：

```bash
lovrabet bff exec --appcode <appCode> --name <functionName> --params '<json>' --format compress
```

边界：
* 通知型 BFF 的 smoke 会真实发送外部消息；必须通过已确认上下文或显式 `--appcode` 锁定同一 app，执行前向用户展示 app、函数名、`configCode`、接收对象和消息摘要并取得明确确认
* `lovrabet bff detail` 只确认运行契约和版本，不返回脚本源码；通知参数必须来自本地已审查脚本或明确业务契约，不能按函数名猜
* 通知执行超时或客户端未拿到结果时，先按“状态未知”处理；不得自动重试，避免重复发送
* `lovrabet` CLI 不可用、未配置或无权限时，明确记录“运行态 smoke 未执行”，不要把它写成 `rabetbase` 验证已通过
* 管理态已同步但运行态仍返回旧版本时，优先按传播延迟 / 缓存延迟处理：等待后重试；必要时再对目标脚本执行一次精确 `bff push --force --type <type> --name <name>`
* 多次重试仍旧版本时，记录平台运行态缓存风险并上报；不要通过修改 Skill、配置文件或另建脚本来掩盖

### 10. 本地文件
脚本内容直接保存在本地文件中，纳入 Git 管理。路径遵循 `.rabetbase/bff/<appCode>/` 目录约定（详见 `backend-function.md`）：
* ENDPOINT → `.rabetbase/bff/<appCode>/ENDPOINT/<name>.js`
* HOOK → `.rabetbase/bff/<appCode>/HOOK/<alias>/<operationType>/<functionNode>/<name>.js`
* COMMON → `.rabetbase/bff/<appCode>/COMMON/<name>.js`

## 冲突处理

若返回 `failed`：
* 告知用户失败的 `lockKey` 和错误原因
* 不要假装成功
* 已成功推送的其他脚本不会自动回滚

## 本地文件

正常流程：先在本地创建/修改，再通过 `push` 同步远端，路径同 Step 10（`.rabetbase/bff/<appCode>/` 下）。
例外场景：
* 用户主动要求"同步平台最新到本地" → 从平台拉取 → 覆盖本地（`bff pull`）
* 用户要求删除脚本 → `bff delete --yes --target ...`

修改已有脚本时，从平台拉取最新内容。

## 修改已有脚本

`rabetbase bff list --format json` → `rabetbase bff detail --id <id> --format json` 拉最新内容 → 如需覆盖本地则 `bff pull` → 修改本地文件 → `bff status` → `bff push`

## BFF 语义差异

| 场景 | 前端 SDK | BFF (context.client) |
|------|---------|---------------------|
| SQL 返回值 | `{ execSuccess, execResult }` | 直接返回数组 |
| 模型键 | 可通过初始化/生成代码使用 alias | 使用 `"dataset_" + 32 位数据集 code` |
| `filter()` 返回 | `tableData` 为列表数据 | `tableData` 为列表数据，不是 `list` |
| `create()` 返回 | 以 SDK 文档/类型为准 | 新记录 ID，不是完整对象 |
| `batchCreate()` 返回 | 以 SDK 文档/类型为准 | 新记录 ID 数组；直接传非空对象数组 |
| 批量更新 | 以 SDK 文档/类型为准 | `update({ id: [...] })`；不存在 `batchUpdate()` |
| SDK 初始化能力 | `createClient` / `registerModels` | 不可用；`context.client` 由平台注入 |
| 前端调 BFF | `client.bff.execute({ scriptName, params })` 返回业务数据 | — |
| 发送应用级通知 | — | `context.client.extension.execute("notification", "send", { configCode, audiences, message })` |
