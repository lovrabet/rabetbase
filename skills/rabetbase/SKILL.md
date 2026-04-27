---
name: rabetbase
version: 2.1.2
description: "Lovrabet 开发工作流 CLI — 通过 rabetbase 命令管理数据集、数据库连接（dblink）、智能列表页（Smart List Page）、SQL 查询、BFF 脚本、菜单同步、代码生成，以及平台问题上报。触发词：数据集、数据表、智能列表页、Smart List Page、page generate-start、page generate-status、page sync、page pull、page push、dblink、数据库连接、schema 分析、db list、db detail、db test、db tables、db diff、db diff --table、db analyze-start、analyze-cancel、analyze-status、traceId、自定义 SQL、sql.execute、bff.execute、get_dataset_detail、validate_sql_content、save_or_update_custom_sql、@lovrabet/sdk、lovrabet 开发、rabetbase、filter、codegen、init、menu sync、menu update、project create、project upgrade、schema、jq、compress、issue report、平台问题、platform issue、问题上报。"
metadata:
  requires:
    bins: ["rabetbase"]
    node: ">=20"
  cliHelp: "rabetbase --help"
---

# Rabetbase CLI

> **前置条件：** 使用任何 API 命令前，需完成认证和配置（见下方）。
> **执行前必做：** 执行任何命令前，必须先阅读对应命令的 reference 文档，再调用命令。
> **命名约定：** 统一使用 `rabetbase <service> <command> [flags]` 格式。
> **输出格式：** AI Agent 需要结构化输出时，**优先 `--format compress`**（与 `json` 相同信封，单行紧凑、省 token）；需要人类可读缩进时用 **`--format json`**。可在 **`json` / `compress`** 上叠加 **`--jq '<expr>'`**（对最终打印的整条 JSON 做 jq；二进制查找顺序为 **`JQ_PATH` → CLI 内置 sidecar jq → `PATH` 上的 jq**，通常无需单独安装系统 jq）。不确定当前 CLI 有哪些子命令或 flags 时，先跑 **`rabetbase schema`**（与 `--help` 同源的机器可读契约，无需登录）。

## 前置条件

1. **认证**：`rabetbase auth` 通过浏览器完成 OAuth 登录
2. **AppCode**：确保 `.rabetbase.json` 中设置了 `apps + defaultApp`（推荐）或兼容读取的顶层 `appcode`，或通过 `--appcode <code>` / `--app <name>` 传入（旧名 `.lovrabet.json` 仍可读）
3. **配置文件**：`rabetbase init` 初始化 `.rabetbase.json`（旧名仍兼容读取）。完整字段说明见 [`.rabetbase.json` 配置参考](references/rabetbase-config.md)
4. **多应用场景**：一个项目有多个应用时，先 `rabetbase app add <name> --appcode …` 配置各应用，再用 `--app <name>` 或 `rabetbase app use <name>` 切换
5. **平台发现**：当你不知道当前登录账号能访问哪些应用时，先 `rabetbase app remote`；它查询平台目录，不修改本地配置
6. **本地配置视图**：当你要确认当前项目或全局已经登记了哪些应用、默认应用是谁时，用 `rabetbase app list`。`rabetbase app` 本身只显示帮助，不等价于 `app list`
7. **本地 SQL / BFF 目录（团队约定）**：新建或长期维护的源文件应落在 CLI 与 `bff status` / `sql pull` 一致的路径，避免写在 `src/`、`queries/` 等随意目录后再迁移。
   - **SQL**：项目根 **`.rabetbase/sql/<appCode>/<dbName|db-<id>>/<sqlCode>_<sqlName>.sql|xml`**；草稿或人工兜底可放 `.draft.sql`。`sql create` / `sql pull` / `sql push` / `sql status` 默认围绕这套目录与 `.rabetbase/sql.lock.json` 工作。`sql save` 已废弃，不再作为推荐路径。
   - **BFF**：**`.rabetbase/bff/<appCode>/...`**（由 `bff create` 创建或 `bff pull` 同步；与 `bff status` / `bff push` 扫描范围一致）。详见 [`guides/sql-creation-workflow.md`](guides/sql-creation-workflow.md)、[`guides/bff-creation-workflow.md`](guides/bff-creation-workflow.md)。
8. **可选：`lovrabet` CLI 查数** — 若本机已安装 **`@lovrabet/lovrabet-cli`**（命令 **`lovrabet`**）且 **版本 ≥ 2.0**（`lovrabet --version`），可用 **`lovrabet data filter` / `lovrabet data getOne`** 按 SDK 语义在终端查看真实行数据与 JSON，便于调试；**非人人安装**，未安装或版本低于 2.0 时仅用 `rabetbase` 即可。详见 [`guides/data-api-guidelines.md`](guides/data-api-guidelines.md) 中「可选：`lovrabet` CLI 查数」。

## Skill Freshness

- **本地 repo 与 CLI 契约优先** — 若已安装的全局 skill 描述与当前仓库 `skills/rabetbase/`、`rabetbase --help`、`rabetbase schema` 不一致，以**当前仓库内容和 CLI 实际输出**为准。
- **发现 skill 过期时应主动刷新** — 若不一致已经影响当前任务判断，且用户没有禁止网络/环境变更，应主动执行 `rabetbase skill install`；等价的显式非交互命令是 `npx skills add lovrabet/rabetbase -g -y`。
- **刷新后必须重新读取** — 刷新完成后，重新打开当前 `SKILL.md` 与所需 reference，再继续执行任务，避免沿用旧记忆。
- **刷新失败也不能回退到旧结论** — 若安装失败，明确告诉用户“本地全局 skill 可能过期”，并继续以仓库内 `skills/rabetbase/` 和 `rabetbase schema` 作为 source of truth。

## Agent 快速执行顺序

1. **判断需求类型**
   - Instant API 标准数据记录操作 → SDK filter/getOne/create/update/delete
   - 简单聚合 → SDK aggregate
   - 复杂 JOIN / 数据库函数 → 自定义 SQL
   - 外部系统调用 / 跨表事务 / 复杂业务编排 → BFF
2. **先拿元数据，再写代码**
   - 至少先查 `rabetbase dataset detail --code xxx --format compress`（或 `json`）获取表结构
   - 跨表场景还需查目标表的结构
   - 需要了解数据模型关系时用 `rabetbase dataset links --format compress`（或 `json`）
   - **管理物理库连接 / 测连 / 同步表结构分析**时用 `rabetbase db …`（先 [`db list`](references/rabetbase-db-list.md)，trace/plan id 见 [`database-connection-workflow.md`](guides/database-connection-workflow.md)）
   - 输出很大且只需子集时，在 `compress`/`json` 上加 `--jq '.data…'` 缩小结果
   - **（可选）** 若环境已安装 **`lovrabet` ≥ 2.0**，可用 `lovrabet data filter` / `data getOne` 对数据集做与 SDK 同语义的终端查数；**不要**假设用户已安装或版本够新，见 [`guides/data-api-guidelines.md`](guides/data-api-guidelines.md)
3. **SQL 工作流严格分步**
   - 团队主路径：查现有(`sql list/detail`) 或新建(`sql create`) → 拉/落本地(`sql pull` / `sql create`) → 编辑同步目录文件 → 可选校验(`sql validate`) → 检查状态(`sql status`) → 先预览(`sql push --dry-run` / `sql delete --dry-run`) → 再 `sql push` / `sql delete` → `sql detail` / `sql exec` 验证
   - `sql save` 已废弃；用户若提到它，直接引导迁移到 `sql create` / `sql push`
4. **BFF 工作流严格分步**
   - 查现有 → 确认字段 → 查公共函数 → 本地创建(`bff create`) → 检查状态(`bff status`) → 先预览(`--dry-run`) → 再拉取/推送/删除
   - 推送完成只代表研发态脚本已同步；若用户要确认运行态效果，显式交接到运行态验证（如可用的 `lovrabet bff exec`），不要把运行态 smoke 伪装成 `rabetbase` 已完成
5. **智能列表页（Smart List Page，page）工作流**
   - 工作流判断先看 [`guides/page-development-workflow.md`](guides/page-development-workflow.md)
   - `page` 命令职责分离：[`page generate-start`](references/rabetbase-page-generate-start.md) 负责提交或复用任务，[`page generate-status`](references/rabetbase-page-generate-status.md) 负责查询任务状态，[`page standard-page-status`](references/rabetbase-standard-page-status.md) 负责查询智能列表页事实
   - 数据集字段变更后同步已有智能列表页：[`page sync`](references/rabetbase-page-sync.md)
   - 本地 schema 开发：[`page pull`](references/rabetbase-page-pull.md) → IDE 编辑 → [`page push`](references/rabetbase-page-push.md)
   - 需要理解 formal schema 组件语义时，再按需阅读 `knowledge/page-schema/` 下的 PageSchema 组件资料；不要把它与 `rabetbase page` 命令 reference 混淆

## 应用决议指引

1. **已明确 appcode**
   - 用户直接给了 `app-xxxx`
   - 或当前命令已经显式传了 `--appcode`
   - 直接执行目标命令，不需要先查 `rabetbase app remote`

2. **已明确本地应用名**
   - 当前项目 `.rabetbase.json` 已有 `apps + defaultApp`
   - 或用户给了 `--app <name>`
   - 先用 `rabetbase app list` 验证本地配置，再执行目标命令

3. **只知道业务名称，不知道能访问哪个应用**
   - 先 `rabetbase app remote --format compress`
   - 从平台目录里确认当前登录账号可访问的应用
   - 再决定是否需要 `rabetbase app add` / `rabetbase app use`，或临时传 `--appcode`

4. **不要混淆两种视图**
   - `rabetbase app list`：本地配置视图，回答“当前项目/全局登记了哪些应用”
   - `rabetbase app remote`：平台目录视图，回答“当前登录账号能访问哪些应用”

5. **当两边不一致时**
   - 平台上有，本地没有：说明还没登记到本地配置，可用 `rabetbase app add`
   - 本地有，平台上查不到：说明当前账号可能无权限，或本地配置已过时，应先核对权限与 appcode

## 数据库连接（`db`）

与 **`dataset`（数据集模型）** 的分工：**`db *`** 管平台登记的 **物理库连接（dblink）**、**测连**、**schema 分析任务**、**物理表清单与差异**；表字段、枚举、模型关系仍以 **`dataset detail` / `dataset links`** 为准。

- **入口**：先 [`db list`](references/rabetbase-db-list.md) 取 `id` 与 `latestAnalysisTraceId`（如有）。
- **只读常用**：[`db detail`](references/rabetbase-db-detail.md)、[`db test`](references/rabetbase-db-test.md)、[`db tables`](references/rabetbase-db-tables.md)、[`db diff`](references/rabetbase-db-diff.md)（`--table` 为表名**子串**过滤，分页用 `--page` / `--pagesize`）。
- **分析任务**：`db analyze-start` / `analyze-cancel` / `analyze-status`（plan/trace 来源见下）。
- **ER 图引导**：`db analyze-status` 终态成功后，检查并转述 `data.links.erPage`；若没有 `data.links`，补 `--appcode` 或进入已配置 app 的项目上下文后重查。
- **写入**：[`db create`](references/rabetbase-db-create.md) / [`db update`](references/rabetbase-db-update.md) / [`db delete`](references/rabetbase-db-delete.md)（高危删连需确认）；`create`/`update` 支持 **`--dry-run`** 预览。`db create` 成功后返回连接信息与数据库连接页链接。
- **横切流程与 trace/plan**：[`database-connection-workflow.md`](guides/database-connection-workflow.md)。**单命令细则**见 `references/rabetbase-db-*.md`（与仓库 `src/commands/db/` 一一对应）。

## 配置作用域原则（`--global`）

- **写操作默认当前项目**（`init`、`app add`、`config set`、`project create` 等），除非用户**显式**传 `--global`。
- **不要**在用户未要求时给命令加 `--global`** — 默认行为已是「项目优先」；只有用户明确要改全局配置或不在项目内且意图写全局时才使用。
- **`config set`**：在**没有**项目配置文件（当前目录未解析到 `.rabetbase.json`）且**未**传 `--global` 时，CLI **拒绝执行**并提示使用 `--global` 或先 `rabetbase init`，**不会**静默写入全局。
- **`project create`** 生成的新项目 `.rabetbase.json` **只继承**少量全局偏好（如 `cookie` / `locale` / `format` / `riskLevel` 等），**不会**把全局 `apps` / `defaultApp` 带入新项目文件。
- **`api pull` / `api list`**：默认仅针对**项目** `apps`；需要合并全局已登记应用时加 `--global`（见各命令 reference）。
- **`app list`**：默认展示**合并**后的全量；`--global` 仅全局、`--project` 仅项目（见 [`rabetbase app list`](references/rabetbase-app-list.md)）。

## Agent 禁止行为

- **不要猜字段名** — 必须从 `dataset detail` 返回值获取真实字段名、类型、枚举值
- **不要跳过 validate 与 dry-run** — 修改已有 SQL 并准备 `sql push` 前，必须先跑 `sql validate`；执行高风险同步前至少确认过 `sql create --dry-run`、`sql push --dry-run` 或 `sql delete --dry-run` 的预览
- **不要手动拼 API URL** — 所有操作通过 CLI 命令完成，不要直接调 HTTP 接口
- **不要臆测 sqlCode / id** — 从 `sql list` 或 `bff list` 获取真实标识
- **不要臆测 dblink id 或分析 trace/plan id** — 从 `db list` / `db detail` / `db analyze-start` 的返回字段获取（见 [database-connection-workflow.md](guides/database-connection-workflow.md)）
- **不要把废弃命令当主路径** — `sql save` 仅用于返回迁移提示；`bff save` 已退出主工作流。除了解释兼容/迁移，不要再基于它们设计步骤
- **不要含糊处理失败** — `sql create` / `sql push` / `sql delete` / `bff push` / `bff delete` 返回失败时，必须明确告知用户是哪条资源失败、为什么失败
- **不要在不确认表结构时就写 SQL** — 先 `dataset detail`，后写 SQL
- **不要循环单条查询** — 用 SDK `filter + $in` 批量查询，不要 N+1
- **不要把 MCP 工具名当 CLI 命令** — 使用 `rabetbase sql list`，不是 `list_sql_queries`
- **不要擅自加 `--global`** — 见上文「配置作用域原则」；默认写项目、读合并；仅在用户明确要求或文档说明的场景使用 `--global`。

## 接口选型优先级

遇到新需求时按优先级选择实现方式：

1. **标准 SDK 接口**（filter/getOne/create 等）— 能用就不写 SQL
2. **aggregate 聚合接口** — 简单分组汇总
3. **自定义 SQL** — 复杂 JOIN、数据库函数、跨表统计
4. **BFF** — 外部系统调用、跨表事务、复杂业务编排

## SDK 核心规则

### 初始化

```typescript
import { createClient } from "@lovrabet/sdk";
const client = createClient({
  appCode: "your-app-code",
  accessKey: process.env.RABETBASE_ACCESS_KEY,
  models: [{ tableName: "users", datasetCode: "abc123", alias: "users" }],
});
```

### filter 查询（最常用）

```typescript
const result = await client.models.users.filter({
  where: { status: { $eq: "active" } },
  select: ["id", "name"],
  orderBy: [{ createTime: "desc" }],
  currentPage: 1,
  pageSize: 20,
});
```

参数名强制约束：`select`（非 fields）、`orderBy`（非 sort）、`currentPage`/`pageSize`（非 page/limit）

where 条件强制使用操作符：`$eq` `$ne` `$gte` `$lte` `$gt` `$lt` `$in` `$contain` `$startWith` `$endWith`

### SQL 调用

```typescript
const data = await client.sql.execute<MyRow>({ sqlCode: "xxx", params: { key: "val" } });
if (data.execSuccess && data.execResult) {
  console.log(data.execResult);
}
```

### BFF 调用

```typescript
const result = await client.bff.execute<DashboardData>({
  scriptName: "getUserDashboard",
  params: { userId: "123" },
});
```

### 前端 vs BFF 关键差异

| | 前端 SDK | BFF (context.client) |
|---|---------|---------------------|
| SQL 返回值 | `{ execSuccess, execResult }` | 直接返回数组 `T[]` |
| 单条查询 | `getOne({ id })` | `getOne({ id })` |
| 调 BFF | `client.bff.execute({ scriptName, params })` | — |

## 意图 → 命令索引

| 意图 | 推荐命令 | 备注 |
|------|---------|------|
| 初始化项目 | [`rabetbase init`](references/rabetbase-init.md) | 智能检测旧配置，支持 `--appcode` 非交互；**不加 `--global`** |
| 创建新项目 | [`rabetbase project create`](references/rabetbase-project-create.md) | 支持 `--name` + `--appcode` 非交互 |
| 从 lovrabet-cli 迁移 | [`rabetbase project upgrade`](references/rabetbase-project-upgrade.md) | 6 步自动迁移，`--yes` 跳过确认 |
| 运行 package.json 脚本 | [`rabetbase run <script>`](references/rabetbase-run.md) | 自动检测包管理器，`start`/`dev` 前做版本检查 |
| 安装 / 重装 / 刷新 skill 包 | [`rabetbase skill install`](references/rabetbase-skill-install.md) | 全局安装或刷新 rabetbase skill；发现本地 skill 过期时优先执行 |
| 退出登录 | [`rabetbase auth logout`](references/rabetbase-auth-logout.md) | 删除本地认证 cookie |
| 诊断配置问题 | [`rabetbase doctor`](references/rabetbase-doctor.md) | 合并配置、各侧 JSON 语法、域名、认证状态 |
| 上报平台问题 | [`rabetbase issue report`](references/rabetbase-issue-report.md) | 由 Skill 先组织 Markdown 上下文，再写入平台 Issue 采集链路 |
| 导出命令契约（flags/risk 等） | [`rabetbase schema`](references/rabetbase-schema.md) | 与 `--help` 同源；**无需登录**；大结果用 `--format compress` |
| 更新 CLI 版本 | [`rabetbase update`](references/rabetbase-update.md) | 自动检测最新版本并升级 |
| 修改配置文件 | [`rabetbase config set <key> <value>`](references/rabetbase-config.md) | 默认写项目；无项目配置且未 `--global` 会拒绝；`--global` 写 `~/.rabetbase.json` |
| 列出配置 | [`rabetbase config list`](references/rabetbase-config.md) | 查看当前生效的配置 |
| 同步菜单到平台 | [`rabetbase menu sync`](references/rabetbase-menu-sync.md) | 本地页面 → 平台菜单，支持交互/静默 |
| 更新菜单 CDN URL | [`rabetbase menu update`](references/rabetbase-menu-update.md) | 批量更新线上菜单资源地址 |
| 查找数据集 | [`rabetbase dataset list --name "xxx"`](references/rabetbase-dataset-list.md) | 服务端模糊匹配；也可 `--code` 精确查 |
| 查看表结构和字段 | [`rabetbase dataset detail --code xxx`](references/rabetbase-dataset-detail.md) | 含字段定义和操作列表 |
| 查看 Dataset 操作定义 | [`rabetbase dataset operations --code xxx`](references/rabetbase-dataset-operations.md) | 获取 filter/getOne/create 等参数定义 |
| 查看数据模型关系 | [`rabetbase dataset links`](references/rabetbase-dataset-links.md) | 跨表 JOIN 关系图 |
| 管理单条数据模型关系 | [`rabetbase dataset link-create/update/delete`](references/rabetbase-dataset-link-mutations.md) | 单条 ER 关系写入；先 `links` 确认 `fromTable/fromColumn/toTable/toColumn`，不要解析字符串 key |
| 首次生成智能列表页 | [`rabetbase page generate-start --datasetcode <code>`](references/rabetbase-page-generate-start.md) | 提交或复用服务端异步任务 |
| 查询智能列表页生成任务状态 | [`rabetbase page generate-status --datasetcode <code> --operation-id <id>`](references/rabetbase-page-generate-status.md) | 查询 job 状态，支持 `operationId` / `clientOperationId` |
| 查询智能列表页事实快照 | [`rabetbase page standard-page-status --datasetcode <code>`](references/rabetbase-standard-page-status.md) | 查询智能列表页四件套、残留页与菜单事实 |
| 同步已有智能列表页 | [`rabetbase page sync --datasetcode <code>`](references/rabetbase-page-sync.md) | 数据集字段变更后同步到关联智能列表页 |
| 拉取页面 schema 到本地 | [`rabetbase page pull --id <pageId>`](references/rabetbase-page-pull.md) | 写入 `.rabetbase/page/<appCode>/`，进入本地编辑工作流 |
| 推送本地页面 schema | [`rabetbase page push --id <pageId>`](references/rabetbase-page-push.md) | 推送后自动回拉 canonical schema 覆盖本地 |
| 数据库连接（dblink）/ 测连 / 结构分析 | [`rabetbase db list`](references/rabetbase-db-list.md) 起 | **`id`**、**trace/plan id** 怎么拿见 [database-connection-workflow.md](guides/database-connection-workflow.md)；各子命令见 `references/rabetbase-db-*.md` |
| 生成 API 客户端代码 | [`rabetbase api pull`](references/rabetbase-api-pull.md) | 拉取数据集并生成 `src/api/` TypeScript 代码 |
| 查看生成的 API 模型 | [`rabetbase api list`](references/rabetbase-api-list.md) | 列出已生成的数据模型 |
| 查看现有 SQL | [`rabetbase sql list --name "xxx"`](references/rabetbase-sql-list.md) | 分页，按名称过滤；默认查当前决议到的单个应用 |
| 查看 SQL 详情 | [`rabetbase sql detail --sqlcode xxx`](references/rabetbase-sql-detail.md) | 含完整 SQL 内容和参数定义 |
| 新建本地同步 SQL | [`rabetbase sql create --name xxx --db-id 10001 --mode sql`](references/rabetbase-sql-create.md) | 先在远端创建，再落本地文件与 `sql.lock.json` |
| 查看本地同步状态 | [`rabetbase sql status`](references/rabetbase-sql-status.md) | 检查 `added / modified / missing / unchanged / remoteOnly` |
| 拉取远端 SQL 到本地 | [`rabetbase sql pull`](references/rabetbase-sql-pull.md) | 写入 `.rabetbase/sql/<appCode>/<dbName|db-<id>>/`，含 `@lovrabet` 头注释 |
| 推送本地 SQL 到远端 | [`rabetbase sql push --sqlcode xxx --yes`](references/rabetbase-sql-push.md) | 以 `sql.lock.json` 为基准上传同步目录中的本地文件 |
| 删除 SQL | [`rabetbase sql delete --sqlcode xxx --yes`](references/rabetbase-sql-delete.md) | 删除远端并将本地文件移入 `.rabetbase/sql-trash/` |
| 校验 SQL 内容 | [`rabetbase sql validate --file xxx`](references/rabetbase-sql-validate.md) | 类型检测、危险语句检查、参数提取 |
| `sql save` 废弃说明 | [`rabetbase sql save`](references/rabetbase-sql-save.md) | 已废弃；新建改用 `sql create`，修改改用编辑本地同步文件后 `sql push` |
| 执行 SQL 查询 | [`rabetbase sql exec --sqlcode xxx`](references/rabetbase-sql-exec.md) | 支持 `--params` JSON 参数 |
| 查看现有 BFF | [`rabetbase bff list`](references/rabetbase-bff-list.md) | 按类型和名称过滤，支持 `--app` 限定应用 |
| 查看 BFF 详情 | [`rabetbase bff detail --id n`](references/rabetbase-bff-detail.md) | 含完整脚本内容 |
| 创建本地 BFF | [`rabetbase bff create --type ENDPOINT --name xxx`](references/rabetbase-bff-create.md) | 在 `.rabetbase/bff/<appCode>/...` 下创建脚手架 |
| 查看 BFF 本地状态 | [`rabetbase bff status`](references/rabetbase-bff-status.md) | 检查 added / modified / unchanged / remoteOnly |
| 拉取远端 BFF | [`rabetbase bff pull`](references/rabetbase-bff-pull.md) | 从远端同步到本地 |
| 推送本地 BFF | [`rabetbase bff push --yes`](references/rabetbase-bff-push.md) | high-risk-write，需 `--yes` |
| 删除 BFF | [`rabetbase bff delete --yes --target xxx`](references/rabetbase-bff-delete.md) | high-risk-write，删远端并清理本地 |
| 生成 SDK 代码 | [`rabetbase codegen sdk --code xxx`](references/rabetbase-codegen-sdk.md) | 按操作生成 TypeScript |
| 生成 SQL 调用代码 | [`rabetbase codegen sql --sqlcode xxx`](references/rabetbase-codegen-sql.md) | sdk/bff 两种 target |
| 列出已配置应用 | [`rabetbase app list`](references/rabetbase-app-list.md) | 默认合并视图；`--global` / `--project` 限定单层；`items[].named`、`meta` 见 reference |
| 发现平台可访问应用 | `rabetbase app remote` | 查询当前登录账号在平台上的应用目录，不修改本地配置 |
| 查看 app 服务帮助 | `rabetbase app` | 只显示 `app` 子命令帮助，不等价于 `app list` |
| 切换默认应用 | [`rabetbase app use <name>`](references/rabetbase-app-use.md) | 持久修改 defaultApp |
| 添加应用 | [`rabetbase app add <name> --appcode <code>`](references/rabetbase-app-add.md) | 首个自动设为 default |
| 移除应用 | [`rabetbase app remove <name>`](references/rabetbase-app-remove.md) | high-risk-write；移除后自动切换 default |
| 临时切换应用执行 | 任何命令加 `--app <name>` 或 `--appcode <code>` | 不修改配置文件 |
| 查看配置文件格式 | [`.rabetbase.json` 配置参考](references/rabetbase-config.md) | 完整字段、优先级、环境变量 |

## 命令分组

> **执行前必做：** 从下表定位到命令后，务必先阅读对应命令的 reference 文档，再调用命令。

| 命令分组 | 说明 |
|----------|------|
| Quick Start | [`init`](references/rabetbase-init.md) |
| Project | [`create`](references/rabetbase-project-create.md) / [`upgrade`](references/rabetbase-project-upgrade.md) |
| Run Scripts | [`run`](references/rabetbase-run.md) |
| Authentication | [`auth login`](references/rabetbase-config.md) / [`auth logout`](references/rabetbase-auth-logout.md) |
| Self Update | [`update`](references/rabetbase-update.md) |
| Schema | [`schema` / `schema export`](references/rabetbase-schema.md) |
| Diagnostics | [`doctor`](references/rabetbase-doctor.md) |
| Platform Issue | [`report`](references/rabetbase-issue-report.md) |
| Configuration | [`config set`](references/rabetbase-config.md) / [`config get`](references/rabetbase-config.md) / [`config list`](references/rabetbase-config.md) |
| Menu | [`sync`](references/rabetbase-menu-sync.md) / [`update`](references/rabetbase-menu-update.md) |
| app commands | [`list`](references/rabetbase-app-list.md) / `remote` / [`use`](references/rabetbase-app-use.md) / [`add`](references/rabetbase-app-add.md) / [`remove`](references/rabetbase-app-remove.md) |
| dataset commands | [`list`](references/rabetbase-dataset-list.md) / [`detail`](references/rabetbase-dataset-detail.md) / [`operations`](references/rabetbase-dataset-operations.md) / [`links`](references/rabetbase-dataset-links.md) / [`link-create/update/delete`](references/rabetbase-dataset-link-mutations.md) |
| page commands | [`generate-start`](references/rabetbase-page-generate-start.md) / [`generate-status`](references/rabetbase-page-generate-status.md) / [`standard-page-status`](references/rabetbase-standard-page-status.md) / [`sync`](references/rabetbase-page-sync.md) / [`pull`](references/rabetbase-page-pull.md) / [`push`](references/rabetbase-page-push.md) |
| Database Connections (`db`) | [`list`](references/rabetbase-db-list.md) / [`detail`](references/rabetbase-db-detail.md) / [`create`](references/rabetbase-db-create.md) / [`update`](references/rabetbase-db-update.md) / [`delete`](references/rabetbase-db-delete.md) / [`test`](references/rabetbase-db-test.md) / [`analyze`](references/rabetbase-db-analyze.md) / [`tables`](references/rabetbase-db-tables.md) / [`diff`](references/rabetbase-db-diff.md) |
| api commands | [`pull`](references/rabetbase-api-pull.md) / [`list`](references/rabetbase-api-list.md) |
| sql commands | [`list`](references/rabetbase-sql-list.md) / [`detail`](references/rabetbase-sql-detail.md) / [`create`](references/rabetbase-sql-create.md) / [`status`](references/rabetbase-sql-status.md) / [`pull`](references/rabetbase-sql-pull.md) / [`push`](references/rabetbase-sql-push.md) / [`delete`](references/rabetbase-sql-delete.md) / [`validate`](references/rabetbase-sql-validate.md) / [`save`（deprecated）](references/rabetbase-sql-save.md) / [`exec`](references/rabetbase-sql-exec.md) |
| bff commands | [`list`](references/rabetbase-bff-list.md) / [`detail`](references/rabetbase-bff-detail.md) / [`create`](references/rabetbase-bff-create.md) / [`status`](references/rabetbase-bff-status.md) / [`pull`](references/rabetbase-bff-pull.md) / [`push`](references/rabetbase-bff-push.md) / [`delete`](references/rabetbase-bff-delete.md) |
| codegen commands | [`sdk`](references/rabetbase-codegen-sdk.md) / [`sql`](references/rabetbase-codegen-sql.md) |
| Skills | [`install`](references/rabetbase-skill-install.md) |

## 风险控制

所有声明式命令有 **risk level**：

| 级别 | 含义 | 使用方式 |
|------|------|---------|
| `read` | 只读查询，随时可执行 | 直接执行 |
| `write` | 修改数据，如 `sql pull` | 先 `--dry-run` 预览，再正式执行 |
| `high-risk-write` | 影响运行时行为，如 `sql create` / `sql push` / `sql delete` / `bff push` / `bff delete` | 必须 `--yes` 确认或交互确认；CI 模式强制 `--yes` |

`sql validate` 仍是 SQL 内容校验入口；`sql save` 已废弃，不再作为推荐写入路径。

配置 `riskLevel` 可限制允许执行的最高风险等级（详见 [配置参考](references/rabetbase-config.md)）。

## 输出格式

声明式命令统一走 **标准信封**：`ok`、`command`、`risk`、可选 `data`（失败时可有 `error` 等，以实际输出为准）。

| 格式 | 用途 | 说明 |
|------|------|------|
| **`--format compress`** | **AI / 脚本优先** | 与 `json` **语义相同**，**单行紧凑**、无缩进换行，显著省 token |
| `--format json` | 程序解析、调试 | 同上信封，**缩进**，便于人工阅读 |
| `--format pretty` | 人类阅读（未指定 format 时的常见默认） | 非 JSON 的彩色文本 |
| `--format table` | 列表数据 | 表格 |

**`--jq '<expr>'`**（全局）：仅配合 **`json` 或 `compress`**；对**最终打印的整段 JSON**执行 jq（表达式作用在信封上，例如取 `data` 内字段用 `.data.xxx`）。**jq 可执行文件**按 **`JQ_PATH` → CLI 内置 sidecar jq → `PATH` 上的 jq** 的顺序解析。若显式设置了 **`JQ_PATH`**，它必须指向一个真实可执行文件；否则命令会直接报错，不会静默回退。

**不确定子命令、flags、risk、是否要 appcode**：执行 **`rabetbase schema`**（即 `schema export`，**无需认证**），输出与 `rabetbase --help` 同源的机器可读元数据；详见 [`references/rabetbase-schema.md`](references/rabetbase-schema.md) 与仓库 **`docs/user-guide/12-schema命令.md`**。

## 常见错误速查

| 错误类型 | 含义 | 解决方案 |
|---------|------|---------|
| `auth_required` | 未登录 | 执行 `rabetbase auth` |
| `config_missing` | 未配置 appcode | `rabetbase project init` 或传 `--appcode` |
| `flag_missing` | 缺少必填参数 | 检查 reference 文档确认必填 flags |
| `validation_error` | 输入校验失败（含 SQL 类型阻止） | 检查 SQL 内容或参数格式 |
| `api_error` | 后端 API 错误 | 检查 appcode、网络、权限 |
| `cancelled` | 用户取消高风险操作 | 用 `--yes` 跳过确认，或修改 riskLevel |
| `blocked` | 平台冲突检测或资源状态冲突（常见于旧保存链路或平台侧限制） | 明确说明未完成远端写入，引导用户去平台处理或改走同步工作流 |

## 冲突处理

凡是出现 `blocked: true`、`action: "blocked"`、或其他明确表示**远端未写入成功**的返回：都必须直接说明“平台未保存/未同步成功”，必要时引导用户去平台处理或回到 `create + 本地编辑 + push` 工作流；**禁止重试绕过、禁止粉饰为已保存**。完整分支表、响应 JSON 样例与沟通要求见 [`guides/conflict-detection.md`](guides/conflict-detection.md)。

## 前端页面规则

- CLI 生成的页面顶部注释必须保留，追加说明用 `@modified`
- 页面组件设置 `displayName`
- 写页面前必须 `dataset detail` 确认字段和关系
- 禁止 emoji、AI 味文案、感叹号、花哨颜色；文案简洁专业
- 图标用 `@ant-design/icons`，颜色用 AntD token

## 深入指南

**文档分层**：`references/` 按 **单条 CLI 命令**（执行前先读对应页）；`guides/` 按 **横切主题**（工作流、数据访问、前后端规范）。二者配合 SKILL 正文渐进披露。

以下 guide 提供详细说明、示例与边界情况：

| 主题 | Use when | Guide |
|------|----------|-------|
| SDK 完整参数与返回值 | 初始化 client、filter/create、sql.execute、bff.execute、错误处理 | [`typescript-sdk.md`](guides/typescript-sdk.md) |
| SQL MyBatis 与动态 SQL | 写自定义 SQL、`<if>`/`<foreach>`、参数绑定 | [`sql-mybatis.md`](guides/sql-mybatis.md) |
| 前端页面开发约束 | React 页、表单、列表、与数据集绑定 | [`frontend-development.md`](guides/frontend-development.md) |
| 故障诊断 | CLI/登录/数据集/保存失败排障 | [`troubleshooting.md`](guides/troubleshooting.md) |
| BFF 脚本规范 | HOOK/ENDPOINT/COMMON、`context.client`、目录与注释模板 | [`backend-function.md`](guides/backend-function.md) |
| 数据接口访问 | 先 detail 再编码、外键/枚举、禁止 N+1、批量与关联查询；可选 `lovrabet data` 查数（**`lovrabet` CLI ≥ 2.0** 且已安装） | [`data-api-guidelines.md`](guides/data-api-guidelines.md) |
| SQL 创建工作流 | list/detail → pull/new → edit → validate → status → push/delete → exec 全链路 | [`sql-creation-workflow.md`](guides/sql-creation-workflow.md) |
| BFF 创建工作流 | new → status → dry-run → pull/push | [`bff-creation-workflow.md`](guides/bff-creation-workflow.md) |
| 冲突检测与保存 | `blocked`、未保存时的用户沟通、响应结构 | [`conflict-detection.md`](guides/conflict-detection.md) |
| 质量与最佳实践 | 审查 SQL/BFF、命名、高危边界、描述字段 | [`best-practices.md`](guides/best-practices.md) |
| 数据库连接与分析 | 接入/改连/测连、`traceId` 来源、`db analyze-*` 与 dataset 分工；子命令速查见上文 **「数据库连接（db）」** | [`database-connection-workflow.md`](guides/database-connection-workflow.md) |
