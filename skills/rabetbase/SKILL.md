---
name: rabetbase
version: 2.3.5
description: "Lovrabet 开发工作流 CLI — 通过 rabetbase 命令管理数据集、数据库连接（dblink）、智能列表页（Smart List Page）、SQL 查询、BFF 脚本、菜单事实读取与同步、代码生成，以及平台问题上报。触发词：数据集、数据表、dataset relation-audit、dataset delete、dataset restore、废弃数据集、恢复数据集、dataset rename、dataset field-update、dataset extend-update、dataset business-group-update、businessGroup、字段对象更新、doType、options、智能列表页、Smart List Page、page generate-start、page generate-status、page relation-audit、page sync、page pull、page push、dblink、数据库连接、schema 分析、db list、db detail、db test、db tables、db diff、db diff --table、db analyze-start、analyze-cancel、analyze-status、traceId、自定义 SQL、sql.execute、bff.execute、get_dataset_detail、validate_sql_content、save_or_update_custom_sql、@lovrabet/sdk、lovrabet 开发、rabetbase、filter、codegen、init、appcode、app list、workspace、workspace init、workspace use、workspace add、workspace remove、多应用、默认应用、menu list、菜单异常审计、菜单手动删除清单、menu sync、menu update、角色、用户组、权限、role、permit、role list、role create、role user-add、role user-remove、user-resolve、销售组、加入开发者、page-set、page-get、row-roles、SELF、ALL、行级权限、role-menus-set、role-apis-set、菜单权限、API 权限、DEV、CUSTOM、project create、project upgrade、schema、jq、compress、issue report、平台问题、platform issue、问题上报。"
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

1. **认证**：`rabetbase auth login` 通过浏览器完成 OAuth 登录；非交互 Agent 使用 `rabetbase auth login --yes`，把终端打印的授权链接发给用户打开，链接有效期 10 分钟
2. **AppCode**：确保 `.rabetbase.json` 中设置了 `apps + defaultApp`（推荐）或兼容读取的顶层 `appcode`，或通过 `--appcode <code>` / `--app <name>` 传入（旧名 `.lovrabet.json` 仍可读）
3. **配置文件**：`rabetbase init` 做**首次安装后的全局引导**（可写 `.rabetbase.json`；旧名仍兼容读取）。完整字段说明见 [`.rabetbase.json` 配置参考](references/rabetbase-config.md)。目录绑定请用下方 `workspace init`，二者不要混用
4. **工作目录应用绑定**：当前目录要固定使用某个应用时，使用 `rabetbase workspace init --appcode <code>` 或 `rabetbase workspace use --app <name>`（选型见「app vs workspace 职责边界」）
5. **多应用场景**：一个项目有多个应用时，先 `rabetbase workspace add <name> --appcode …` 登记各应用，再用 `--app <name>` 临时切换，或 `workspace use --app <name>` 修改当前工作目录默认应用
6. **平台发现**：当你不知道当前登录账号能访问哪些应用时，先 `rabetbase app list --remote`；它查询平台目录，不修改本地配置
7. **本地配置视图**：当你要确认当前项目或全局已经登记了哪些应用、默认应用是谁时，用 `rabetbase app list`。`rabetbase app` 本身只显示帮助，不等价于 `app list`
8. **本地 SQL / BFF 目录**：新建或长期维护的源文件应落在 CLI 与 `bff status` / `sql pull` 一致的路径，避免写在 `src/`、`queries/` 等随意目录后再迁移。
   - **SQL**：项目根 **`.rabetbase/sql/<appCode>/<dbName|db-<id>>/<sqlCode>_<sqlName>.sql|xml`**；草稿或人工兜底可放 `.draft.sql`。`sql create` / `sql pull` / `sql push` / `sql status` 默认围绕这套目录与 `.rabetbase/sql.lock.json` 工作。`sql save` 已废弃，不再作为推荐路径。
   - **BFF**：**`.rabetbase/bff/<appCode>/...`**（由 `bff create` 创建或 `bff pull` 同步；与 `bff status` / `bff push` 扫描范围一致）。详见 [`guides/sql-creation-workflow.md`](guides/sql-creation-workflow.md)、[`guides/bff-creation-workflow.md`](guides/bff-creation-workflow.md)。
9. **可选：`lovrabet` CLI 查数** — 若本机已安装 **`@lovrabet/lovrabet-cli`**（命令 **`lovrabet`**）且 **版本 ≥ 2.0**（`lovrabet --version`），可用 **`lovrabet data filter` / `lovrabet data getOne`** 按 SDK 语义在终端查看真实行数据与 JSON，便于调试；**非人人安装**，未安装或版本低于 2.0 时仅用 `rabetbase` 即可。详见 [`guides/data-api-guidelines.md`](guides/data-api-guidelines.md) 中「可选：`lovrabet` CLI 查数」。

## Skill Freshness

- **本地 repo 与 CLI 契约优先** — 若已安装的全局 skill 描述与当前仓库 `skills/rabetbase/`、`rabetbase --help`、`rabetbase schema` 不一致，以**当前仓库内容和 CLI 实际输出**为准。
- **发现 skill 过期时应主动刷新** — 若不一致已经影响当前任务判断，且用户没有禁止网络/环境变更，应主动执行 `rabetbase cli-skill install`；等价的显式非交互命令是 `npx skills add lovrabet/rabetbase -g -y`。
- **刷新后必须重新读取** — 刷新完成后，重新打开当前 `SKILL.md` 与所需 reference，再继续执行任务，避免沿用旧记忆。
- **刷新失败也不能回退到旧结论** — 若安装失败，明确告诉用户“本地全局 skill 可能过期”，并继续以仓库内 `skills/rabetbase/` 和 `rabetbase schema` 作为 source of truth。

## Agent 快速执行顺序

1. **判断需求类型**
   - Instant API 标准数据记录操作 → SDK filter/getOne/create/update/delete
   - 简单聚合且数据集是 DB_TABLE → SDK aggregate；聚合列用 `aggregate[].column`，不要用旧别名 `field`
   - 复杂 JOIN / 数据库函数且数据集是 DB_TABLE → 自定义 SQL
   - 外部系统调用 / 跨表事务 / 复杂业务编排 → BFF；BFF HOOK 可挂 DB_TABLE 或 METADATA，具体 operation 以后端返回为准
2. **先拿元数据，再写代码**
   - 至少先查 `rabetbase dataset detail --code xxx --format compress`（或 `json`）获取表结构
   - 跨表场景还需查目标表的结构
   - 写入前用 `data.fields[]` 确认真实字段、必填字段、枚举选项；枚举/选择字段写入 `options[].value`，不是展示 `label`
   - 需要了解数据集关联关系时用 `rabetbase dataset relations --format compress`（或 `json`）；需要审计关系事实错误和人工复核项时用 `rabetbase dataset relation-audit --format compress`
   - 需要从文本需求创建新的 `METADATA` 数据集时，先读 [`dataset generate-start/status`](references/rabetbase-dataset-generate.md)，执行 preview 写出 design 文件，审阅后用 `generate-start --apply --design-file` 提交任务，再用 `generate-status` 查询到成功
   - **管理物理库连接 / 测连 / 同步表结构分析**时用 `rabetbase db …`（先 [`db list`](references/rabetbase-db-list.md)，trace/plan id 见 [`database-connection-workflow.md`](guides/database-connection-workflow.md)）
   - DB 增量分析只有在 `analyze-status.data.isTerminal === true` 且复跑 `db diff --all --changed-only` 后目标差异收敛时才算完成；不要从运行中的 `failedTables` 推测最终结果
   - 输出很大且只需子集时，在 `compress`/`json` 上加 `--jq '.data…'` 缩小结果
   - **（可选）** 若环境已安装 **`lovrabet` ≥ 2.0**，可用 `lovrabet data filter` / `data getOne` 对数据集做与 SDK 同语义的终端查数；**不要**假设用户已安装或版本够新，见 [`guides/data-api-guidelines.md`](guides/data-api-guidelines.md)
3. **SQL 工作流严格分步**
   - 推荐路径：查现有(`sql list/detail`) 或新建(`sql create`) → 拉/落本地(`sql pull` / `sql create`) → 编辑同步目录文件 → 可选校验(`sql validate`) → 检查状态(`sql status`) → 先预览(`sql push --dry-run` / `sql delete --dry-run`) → 再 `sql push` / `sql delete` → `sql detail` / `sql exec` 验证
   - `sql save` 已废弃；用户若提到它，直接引导迁移到 `sql create` / `sql push`
4. **BFF 工作流严格分步**
   - 查现有 → 确认字段 → 查公共函数 → 本地创建(`bff create`) → 检查状态(`bff status`) → 先预览(`--dry-run`) → 再拉取/推送/删除
   - 推送完成只代表脚本配置已同步到平台；若用户要确认最终运行效果，显式交接到运行验证（如可用的 `lovrabet bff exec`），不要把运行验证伪装成 `rabetbase` 已完成
5. **智能列表页（Smart List Page，page）工作流**
   - 工作流判断先看 [`guides/page-development-workflow.md`](guides/page-development-workflow.md)
   - `page` 命令职责分离：[`page generate-start`](references/rabetbase-page-generate-start.md) 负责提交或复用任务，[`page generate-status`](references/rabetbase-page-generate-status.md) 负责查询任务状态，[`page standard-page-status`](references/rabetbase-standard-page-status.md) 负责查询智能列表页事实
   - 数据集字段变更后同步已有智能列表页：[`page sync`](references/rabetbase-page-sync.md)
   - 页面关系绑定审计：先读 [`rabetbase-page-relation-binding.md`](references/rabetbase-page-relation-binding.md)，再执行 [`page relation-audit`](references/rabetbase-page-relation-binding.md)
   - 遇到“关联数据带不出 / 标准页面没有生成关联关系 / 页面关联绑定异常”类问题，固定顺序是：`dataset relations` 只读确认关系事实 → `dataset relation-audit` 审计关系事实风险 → `page standard-page-status` 判断页面是否存在 → `page relation-audit` 审计页面绑定；若关系事实本身不对，先按 [`dataset relation-create/update/delete`](references/rabetbase-dataset-relation-mutations.md) 做 `--dry-run` 方案并等用户确认；已有页面再 `page sync --dry-run`，无页面走 `page generate-start --dry-run`
   - `page sync` 只负责同步已有智能列表页，不是数据集关系修复命令，也不能承诺自动修复所有 `wrong_code` / `wrong_label` / `missing`
   - 本地 schema 开发：[`page pull`](references/rabetbase-page-pull.md) → IDE 编辑 → [`page push`](references/rabetbase-page-push.md)
   - 需要理解 formal schema 组件语义时，再按需阅读 `knowledge/page-schema/` 下的 PageSchema 组件资料；不要把它与 `rabetbase page` 命令 reference 混淆
6. **Legacy modernization / Application Blueprint 工作流**
   - 当用户要把无人交接老项目、外包接手项目或遗留系统翻新到 Lovrabet 体系时，先阅读 [`guides/legacy-application-blueprint-workflow.md`](guides/legacy-application-blueprint-workflow.md)
   - 主产物是 `.rabetbase/blueprint/<appCode>/application-blueprint.md`，不是直接生成或推送 SQL/BFF
   - 必须把老代码 Logic Graph 与 dbagent 已生成的 Dataset / Relations 绑定，再判断 Instant API、Custom SQL、BFF、Hook、COMMON 或 Java Service 落点

## 应用决议指引

1. **已明确 appcode**
   - 用户直接给了 `app-xxxx`
   - 或当前命令已经显式传了 `--appcode`
   - 直接执行目标命令，不需要先查 `rabetbase app list --remote`

2. **已明确本地应用名**
   - 当前项目 `.rabetbase.json` 已有 `apps + defaultApp`
   - 或用户给了 `--app <name>`
   - 先用 `rabetbase app list` 验证本地配置，再执行目标命令

3. **只知道业务名称，不知道能访问哪个应用**
   - 先 `rabetbase app list --remote --format compress`
   - 从平台目录里确认当前登录账号可访问的应用
   - 再决定是否需要 `rabetbase workspace add` / `rabetbase workspace use`，或临时传 `--appcode`

4. **不要混淆两种视图**
   - `rabetbase app list`：本地配置视图，回答“当前项目/全局登记了哪些应用”
   - `rabetbase app list --remote`：平台目录视图，回答“当前登录账号能访问哪些应用”

5. **当两边不一致时**
   - 平台上有，本地没有：说明还没登记到本地配置，可用 `rabetbase workspace add`
   - 本地有，平台上查不到：说明当前账号可能无权限，或本地配置已过时，应先核对权限与 appcode

## 数据库连接（`db`）

与 **`dataset`（数据集模型）** 的分工：**`db *`** 管平台登记的 **物理库连接（dblink）**、**测连**、**schema 分析任务**、**物理表清单与差异**；表字段、枚举以 **`dataset detail`** 为准，数据集关联关系优先以 **`dataset relations`** 为准，关系事实风险用 **`dataset relation-audit`**。

- **入口**：先 [`db list`](references/rabetbase-db-list.md) 取 `id` 与 `latestAnalysisTraceId`（如有）。
- **只读常用**：[`db detail`](references/rabetbase-db-detail.md)、[`db test`](references/rabetbase-db-test.md)、[`db tables`](references/rabetbase-db-tables.md)、[`db diff`](references/rabetbase-db-diff.md)（`--table` 为表名**子串**过滤，分页用 `--page` / `--pagesize`）。
- **分析任务**：`db analyze-start` / `analyze-cancel` / `analyze-status`（plan/trace 来源见下）。
- **ER 图引导**：`db analyze-status` 终态成功后，检查并转述 `data.links.erPage`；若没有 `data.links`，补 `--appcode` 或进入已配置 app 的项目上下文后重查。
- **写入**：[`db create`](references/rabetbase-db-create.md) / [`db update`](references/rabetbase-db-update.md) / [`db delete`](references/rabetbase-db-delete.md)（高危删连需确认）；`create`/`update` 支持 **`--dry-run`** 预览。`db create` 成功后返回连接信息与数据库连接页链接。
- **横切流程与 trace/plan**：[`database-connection-workflow.md`](guides/database-connection-workflow.md)。**单命令细则**见 `references/rabetbase-db-*.md`（与仓库 `src/commands/db/` 一一对应）。

## 用户组与权限（`role` / `permit`）

先分清两个层面再动手：**成员归属**（谁在组里，用 `role user-*`）与**能力授权**（组能看什么/做什么，用 `permit *`）。「某人不能看财务」通常是收紧其所在 `CUSTOM` 组的能力，而不是把人移出组；内置 `DEV`/`ADMIN` 的权限矩阵不可改。

- **横切工作流与决策树**（问题 → 命令、`role-menus-set` vs `page-set --menu-roles` 的区别、5 个剧本）先读 [`guides/role-permit-workflow.md`](guides/role-permit-workflow.md)。
- **角色类型**：`ADMIN`/`DEV`/`USER`（内置）与 `CUSTOM`（用户创建）；`update`/`delete` 仅 `CUSTOM`。
- **加人/移人**：先 [`role user-resolve`](references/rabetbase-role-user-resolve.md) 拿 userId，再 [`role user-add`/`user-remove`](references/rabetbase-role-user-add.md)（`high-risk-write`，内部 read-merge-write，只改目标组）。
- **建/改/删组**：[`role list/detail/create/update/delete`](references/rabetbase-role-list.md)。
- **收紧菜单/接口（按组）**：[`role-menus-set`](references/rabetbase-permit-role-menus-set.md) / [`role-apis-set`](references/rabetbase-permit-role-apis-set.md)，必须显式 `--menus`/`--datasets`。
- **单页权限（含行级只看自己）**：[`page-get`/`page-set`](references/rabetbase-permit-page-set.md)；行级 `SELF` 只用于 `--row-roles`。
- **铁律**：写前先 `page-get`/`role-menus` 拿真实标识 → `--dry-run` 看 `data.before/after` → `--yes`；行级/接口收紧的运行态效果交给 `lovrabet data filter` 验证。

## 配置作用域原则（`--global`）

- **写操作默认当前项目**（`init`、`workspace add`、`config set`、`project create` 等），除非用户**显式**传 `--global`。
- **不要**在用户未要求时给命令加 `--global`** — 默认行为已是「项目优先」；只有用户明确要改全局配置或不在项目内且意图写全局时才使用。
- **`config set`**：在**没有**项目配置文件（当前目录未解析到 `.rabetbase.json`）且**未**传 `--global` 时，CLI **拒绝执行**并提示使用 `--global` 或先 `rabetbase init`，**不会**静默写入全局。
- **`project create`** 生成的新项目 `.rabetbase.json` **只继承**少量全局偏好（如 `cookie` / `locale` / `format` / `riskLevel` 等），**不会**把全局 `apps` / `defaultApp` 带入新项目文件。
- **`api pull` / `api list`**：默认仅针对**项目** `apps`；需要合并全局已登记应用时加 `--global`（见各命令 reference）。
- **`app list`**：默认展示**合并**后的全量；`--global` 仅全局、`--project` 仅项目（见 [`rabetbase app list`](references/rabetbase-app-list.md)）。

## app vs workspace 职责边界

划分轴 = **「这是关于 app 的客观事实，还是关于我当前工作环境的配置？」**

- **`app` = 应用的客观事实查询**：回答「我有/能访问哪些 app」，与在哪写代码无关。**只读**，不承载增删。
  - `app list`（本地登记视图）/ `app list --remote`（平台可访问目录）。
- **`workspace` = 配置当前工作环境用哪些 app**：一切「改我的配置」的写操作都在这里。
  - `workspace init`（首次绑定当前目录）/ `workspace use --app <name>`（切换当前目录默认应用）。
  - `workspace add <name> --appcode <code>`（登记应用 profile）/ `workspace remove <name>`（移除，high-risk-write）。
  - 默认写当前项目 `.rabetbase.json`；`--global` 写全局（全局视作一个大工作空间）。`init`/`use` 不从全局复制 cookie/accessKey。

**三入口选型（不要混用）**：

| 场景 | 命令 |
|------|------|
| 首次安装后的全局引导 | `rabetbase init` |
| 当前目录还没绑定默认 app | `rabetbase workspace init --appcode <code>`（**始终**写 `defaultApp`） |
| 目录已有配置，只再登记一个 profile | `rabetbase workspace add <name> --appcode <code>`（仅当尚无 default 时才顺带设默认；`--global` 写全局） |
| 切换当前目录默认应用 | `rabetbase workspace use --app <name>` |

**入口判断**：
- 「看有哪些 app / 平台上能访问哪些」→ `app list`（`--remote` 查平台）。
- 「登记/删除应用 profile」→ `workspace add` / `workspace remove`（要动全局清单时加 `--global`）。
- 「让当前目录用某应用 / 切换默认应用」→ `workspace init` / `workspace use`。

## Agent 禁止行为

- **不要猜字段名** — 必须从 `dataset detail` 返回值获取真实字段名、类型、枚举值
- **不要跳过 validate 与 dry-run** — 修改已有 SQL 并准备 `sql push` 前，必须先跑 `sql validate`；执行高风险同步前至少确认过 `sql create --dry-run`、`sql push --dry-run` 或 `sql delete --dry-run` 的预览
- **不要手动拼 API URL** — 所有操作通过 CLI 命令完成，不要直接调 HTTP 接口
- **不要臆测 sqlCode / id** — 从 `sql list` 或 `bff list` 获取真实标识
- **不要臆测 dblink id 或分析 trace/plan id** — 从 `db list` / `db detail` / `db analyze-start` 的返回字段获取（见 [database-connection-workflow.md](guides/database-connection-workflow.md)）
- **只使用可见命令** — 以本 skill、`rabetbase schema`、`rabetbase --help` 中出现的命令为准；不要凭训练记忆调用未出现的旧别名
- **不要含糊处理失败** — `sql create` / `sql push` / `sql delete` / `bff push` / `bff delete` 返回失败时，必须明确告知用户是哪条资源失败、为什么失败
- **不要在不确认表结构时就写 SQL** — 先 `dataset detail`，后写 SQL、Backend Function、页面……
- **不要把案例字段当通用规则** — 任何 Demo、历史项目、示例里的字段名、枚举值、表名都不能照搬；必须以当前 `dataset detail` 为准
- **不要循环单条查询** — 用 SDK `filter + $in` 批量查询，不要 N+1
- **不要依赖 CLI 输出文本片段判断写入结果** — 对 `dataset rename --dry-run` / 正式执行，只读取结构化 envelope 的 `data.*` 稳定字段
- **不要跳过 Dataset rename 最终线上回查** — 连续重命名后必须按 code 重新查询线上名称，不能只相信本地 plan 或 dry-run
- **不要把 MCP 工具名当 CLI 命令** — 使用 `rabetbase sql list`，不是 `list_sql_queries`
- **不要擅自加 `--global`** — 见上文「配置作用域原则」；默认写项目、读合并；仅在用户明确要求或文档说明的场景使用 `--global`。

## 接口选型优先级

遇到新需求时按优先级选择实现方式：

1. **标准 SDK 接口**（filter/getOne/create 等）— 能用就不写 SQL
2. **aggregate 聚合接口** — DB_TABLE 的简单分组汇总；METADATA 不默认支持 aggregate；聚合定义使用 `column` 指定列，`field` 仅作为历史兼容别名
3. **自定义 SQL** — DB_TABLE 的复杂 JOIN、数据库函数、跨表统计；METADATA 不支持 SQL 路径
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
const data = await client.sql.execute<MyRow>({
  sqlCode: "xxx",
  params: { key: "val" },
});
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

|                 | 前端 SDK                                     | BFF (context.client)                 |
| --------------- | -------------------------------------------- | ------------------------------------ |
| SQL 返回值      | `{ execSuccess, execResult }`                | 直接返回数组 `T[]`                   |
| 单条查询        | `getOne({ id })`                             | `getOne({ id })`                     |
| 模型键          | 可通过初始化/生成代码使用 alias              | 使用 `"dataset_" + 32 位数据集 code` |
| `filter()` 返回 | `tableData` 为列表数据                       | `tableData` 为列表数据，不是 `list`  |
| `create()` 返回 | 以 SDK 文档/类型为准                         | 新记录 ID，不是完整对象              |
| SDK 初始化能力  | `createClient` / `registerModels`            | 不可用；`context.client` 由平台注入  |
| 调 BFF          | `client.bff.execute({ scriptName, params })` | —                                    |

## 意图 → 命令索引

| 意图 | 推荐命令 | 备注 |
|------|---------|------|
| 初始化项目 | [`rabetbase init`](references/rabetbase-init.md) | 智能检测旧配置，支持 `--appcode` 非交互；**不加 `--global`** |
| 创建新项目 | [`rabetbase project create`](references/rabetbase-project-create.md) | 支持 `--name` + `--appcode` 非交互 |
| 从 lovrabet-cli 迁移 | [`rabetbase project upgrade`](references/rabetbase-project-upgrade.md) | 6 步自动迁移，`--yes` 跳过确认 |
| 老项目翻新蓝图 / Legacy Application Blueprint | [`guides/legacy-application-blueprint-workflow.md`](guides/legacy-application-blueprint-workflow.md) | 先输出 `.rabetbase/blueprint/<appCode>/application-blueprint.md`，把老代码逻辑与 Dataset / Relations 绑定后再生成迁移 Backlog |
| 运行 package.json 脚本 | [`rabetbase run <script>`](references/rabetbase-run.md) | 自动检测包管理器，`start`/`dev` 前做版本检查 |
| 安装 / 重装 / 刷新 CLI Built-in Skill | [`rabetbase cli-skill install`](references/rabetbase-cli-skill-install.md) | 全局安装或刷新 rabetbase CLI Built-in Skill；发现本地 skill 过期时优先执行 |
| 退出登录 | [`rabetbase auth logout`](references/rabetbase-auth-logout.md) | 删除本地认证 cookie |
| 诊断配置问题 | [`rabetbase doctor`](references/rabetbase-doctor.md) | 合并配置、各侧 JSON 语法、域名、认证状态 |
| 上报平台问题 | [`rabetbase issue report`](references/rabetbase-issue-report.md) | 由 Skill 先组织 Markdown 上下文，再写入平台 Issue 采集链路 |
| 导出命令契约（flags/risk 等） | [`rabetbase schema`](references/rabetbase-schema.md) | 与 `--help` 同源；**无需登录**；大结果用 `--format compress` |
| 更新 CLI 版本 | [`rabetbase update`](references/rabetbase-update.md) | 自动检测最新版本并升级 |
| 初始化/切换当前工作目录应用 | [`rabetbase workspace`](references/rabetbase-workspace.md) | 写当前目录 `.rabetbase.json`；不从全局复制 cookie/accessKey |
| 修改配置文件 | [`rabetbase config set <key> <value>`](references/rabetbase-config.md) | 默认写项目；无项目配置且未 `--global` 会拒绝；`--global` 写 `~/.rabetbase.json` |
| 列出配置 | [`rabetbase config list`](references/rabetbase-config.md) | 查看当前生效的配置 |
| 管理运行态 app-config | [`rabetbase app-config list/get/set/delete`](references/rabetbase-app-config.md) | 运行态 app-config 管理面；默认不输出明文 value，`set/delete` 先 `--dry-run` |
| 查看线上菜单事实 / 菜单异常审计 | [`rabetbase menu list`](references/rabetbase-menu-list.md) | 只读返回当前 App 菜单事实；过滤用 `--jq`；人工修复重复菜单先读 [`menu-anomaly-manual-cleanup`](guides/menu-anomaly-manual-cleanup.md) |
| 同步菜单到平台 | [`rabetbase menu sync`](references/rabetbase-menu-sync.md) | 本地页面 → 平台菜单，支持交互/静默 |
| 修改菜单资源 URL / 更新菜单 CDN URL | [`rabetbase menu update`](references/rabetbase-menu-update.md) | 高频高风险写入；先用 `menu list` 确认资源现状，再 `--dry-run` 看 diff，默认用 `--mode patch`，最后复用同参数加 `--yes` |
| 用户组与权限总览（先读工作流/决策树） | [`guides/role-permit-workflow.md`](guides/role-permit-workflow.md) | 先分清「成员归属」vs「能力授权」，再定位到具体命令；含 `role-menus-set` 与 `page-set` 的区别与 5 个剧本 |
| 查看/创建/管理角色（用户组） | [`rabetbase role list/detail/create/update/delete`](references/rabetbase-role-list.md) | 仅 CUSTOM 可改/删；DEV/ADMIN 拒绝；写操作先 `--dry-run` 再 `--yes` |
| 解析昵称/用户名到 userId | [`rabetbase role user-resolve`](references/rabetbase-role-user-resolve.md) | 基于租户成员目录；重名报错要求 `--user <id>` 消歧 |
| 把员工加入/移出角色 | [`rabetbase role user-add / user-remove`](references/rabetbase-role-user-add.md) | 内部全量 read-merge-write，只改目标角色；`high-risk-write`，先 `--dry-run` |
| 配置页面权限 / 行级只看自己 | [`rabetbase permit page-get / page-set`](references/rabetbase-permit-page-set.md) | 行级 SELF 用 `page-set --row-roles SELF`；只改传入槽位，其余原样回写 |
| 收紧角色菜单/接口访问（如销售组不看财务） | [`rabetbase permit role-menus-set / role-apis-set`](references/rabetbase-permit-role-menus-set.md) | 先 `dataset list`/`menu list` 定位，再显式 `--menus`/`--datasets`；一期不自动 suggest |
| 查找数据集 | [`rabetbase dataset list --name "xxx"`](references/rabetbase-dataset-list.md) | 默认返回全部 DO V2 数据集；查指定来源用 `--source DB_TABLE` / `--source METADATA`；也可 `--code` 精确查 |
| 查看表结构和字段 | [`rabetbase dataset detail --code xxx`](references/rabetbase-dataset-detail.md) | 含字段定义和操作列表 |
| 废弃数据集 | [`rabetbase dataset delete`](references/rabetbase-dataset-delete.md) | high-risk-write；只从未删除数据集中定位，必须先 `--dry-run`，正式执行加 `--confirm --yes`，批量废弃推荐 `--expected-count` |
| 恢复已删除数据集 | [`rabetbase dataset restore`](references/rabetbase-dataset-restore.md) | high-risk-write；只从已删除数据集中定位，必须先 `--dry-run`，正式执行加 `--confirm --yes`，批量恢复推荐 `--expected-count` |
| 修改 Dataset 展示名 | [`rabetbase dataset rename`](references/rabetbase-dataset-rename.md) | 只更新 Dataset 展示名；必须先 `--dry-run`，用 `--expect-name` 防漂移；连续重命名先读对应章节 |
| 从文本生成新 METADATA 数据集 | [`rabetbase dataset generate-start`](references/rabetbase-dataset-generate.md) / [`generate-status`](references/rabetbase-dataset-generate.md) | 三步：preview 生成本地 design 快照；审阅后 `--apply --design-file` 提交异步任务；查询成功后使用 `createdDataset.code` |
| 安全更新 Dataset 原始字段对象 | [`rabetbase dataset field-update`](references/rabetbase-dataset-field-update.md) | 使用 `--code` 定位 Dataset，只允许 patch 已知可变业务配置字段；必须先 `--dry-run`，用 `--expect-json` 防漂移 |
| Dataset 顶层 extend 更新命令 | [`rabetbase dataset extend-update`](references/rabetbase-dataset-extend-update.md) | 当前无可写字段；不得用于 `businessGroup`。业务模型分组只能使用 `rabetbase dataset business-group-update` |
| 更新业务模型分组 | [`rabetbase dataset business-group-update`](references/rabetbase-dataset-business-group-update.md) | 唯一入口；使用 `--code` 定位 Dataset；必须先 `--dry-run`，推荐用 `--expect-business-group` 防漂移 |
| 查看 Dataset 操作定义 | [`rabetbase dataset operations --code xxx`](references/rabetbase-dataset-operations.md) | 获取 filter/getOne/create 等参数定义 |
| 查看数据集关联关系 | [`rabetbase dataset relations`](references/rabetbase-dataset-relations.md) | 标准只读入口，输出 `datasetCode + field` 关系事实；支持 `DB_TABLE -> DB_TABLE`、`DB_TABLE -> METADATA`、`METADATA -> METADATA` |
| 审计数据集关联关系 | [`rabetbase dataset relation-audit`](references/rabetbase-dataset-relation-audit.md) | 只读审计关系事实结构错误、风险和人工复核项 |
| 管理单条数据集关联关系 | [`rabetbase dataset relation-create/update/delete`](references/rabetbase-dataset-relation-mutations.md) | 单条关系写入；写入前用 `relations` 确认 `datasetCode + field` 关系事实，DB_TABLE 写入所需表名来自显式参数或物理表事实 |
| 首次生成智能列表页 | [`rabetbase page generate-start --datasetcode <code>`](references/rabetbase-page-generate-start.md) | 提交或复用服务端异步任务 |
| 查询智能列表页生成任务状态 | [`rabetbase page generate-status --datasetcode <code> --operation-id <id>`](references/rabetbase-page-generate-status.md) | 查询 job 状态，支持 `operationId` / `clientOperationId` |
| 查询智能列表页事实快照 | [`rabetbase page standard-page-status --datasetcode <code>`](references/rabetbase-standard-page-status.md) | 查询智能列表页四件套、残留页与菜单事实 |
| 审计页面关系绑定 | [`rabetbase page relation-audit --datasetcode <code>`](references/rabetbase-page-relation-binding.md) | 只读检查页面 options 绑定是否匹配 `dataset relations` |
| 同步已有智能列表页 | [`rabetbase page sync --datasetcode <code>`](references/rabetbase-page-sync.md) | 数据集字段变更后同步到关联智能列表页 |
| 拉取页面 schema 到本地 | [`rabetbase page pull --id <pageId>`](references/rabetbase-page-pull.md) | 写入 `.rabetbase/page/<appCode>/`，进入本地编辑工作流 |
| 推送本地页面 schema | [`rabetbase page push --id <pageId>`](references/rabetbase-page-push.md) | 推送后自动回拉 canonical schema 覆盖本地 |
| 数据库连接（dblink）/ 测连 / 结构分析 | [`rabetbase db list`](references/rabetbase-db-list.md) 起 | **`id`**、**trace/plan id** 与“终态 + 复跑 diff”完成口径见 [database-connection-workflow.md](guides/database-connection-workflow.md)；各子命令见 `references/rabetbase-db-*.md` |
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
| 发现平台可访问应用 | `rabetbase app list --remote` | 查询当前登录账号在平台上的应用目录，不修改本地配置 |
| 查看 app 服务帮助 | `rabetbase app` | 只显示 `app` 子命令帮助，不等价于 `app list` |
| 绑定当前目录到应用 | [`rabetbase workspace init --appcode <code>`](references/rabetbase-workspace.md) | 首次绑定；**始终**写 `defaultApp`；与 `rabetbase init`（全局引导）不同 |
| 切换当前工作目录默认应用 | [`rabetbase workspace use --app <name>`](references/rabetbase-workspace.md) | 持久修改当前目录 defaultApp |
| 登记应用 | [`rabetbase workspace add <name> --appcode <code>`](references/rabetbase-workspace.md) | 仅当尚无 default 时顺带设默认；`--global` 写全局 |
| 移除应用 | [`rabetbase workspace remove <name>`](references/rabetbase-workspace.md) | high-risk-write；移除后自动切换 default |
| 临时切换应用执行 | 任何命令加 `--app <name>` 或 `--appcode <code>` | 不修改配置文件 |
| 查看配置文件格式 | [`.rabetbase.json` 配置参考](references/rabetbase-config.md) | 完整字段、优先级、环境变量 |

## 命令分组

> **执行前必做：** 从下表定位到命令后，务必先阅读对应命令的 reference 文档，再调用命令。

| 命令分组 | 说明 |
|----------|------|
| Quick Start | [`init`](references/rabetbase-init.md) |
| Project | [`create`](references/rabetbase-project-create.md) / [`upgrade`](references/rabetbase-project-upgrade.md) |
| Workspace | [`workspace init` / `workspace use` / `workspace add` / `workspace remove`](references/rabetbase-workspace.md) |
| Run Scripts | [`run`](references/rabetbase-run.md) |
| Authentication | [`auth login`](references/rabetbase-auth-login.md) / [`auth logout`](references/rabetbase-auth-logout.md) |
| Self Update | [`update`](references/rabetbase-update.md) |
| Schema | [`schema` / `schema export`](references/rabetbase-schema.md) |
| Diagnostics | [`doctor`](references/rabetbase-doctor.md) |
| Platform Issue | [`report`](references/rabetbase-issue-report.md) |
| Configuration | [`config set`](references/rabetbase-config.md) / [`config get`](references/rabetbase-config.md) / [`config list`](references/rabetbase-config.md) |
| Runtime App Config Management | [`app-config list/get/set/delete`](references/rabetbase-app-config.md) |
| Menu | [`list`](references/rabetbase-menu-list.md) / [`sync`](references/rabetbase-menu-sync.md) / [`update`](references/rabetbase-menu-update.md) |
| Roles | [`list`](references/rabetbase-role-list.md) / `detail` / `create` / `update` / `delete` / [`user-resolve`](references/rabetbase-role-user-resolve.md) / [`user-add` / `user-remove`](references/rabetbase-role-user-add.md) |
| Permissions | [`page-get` / `page-set`](references/rabetbase-permit-page-set.md) / `role-menus` / [`role-menus-set`](references/rabetbase-permit-role-menus-set.md) / [`role-apis-set`](references/rabetbase-permit-role-apis-set.md) |
| app commands | [`list`](references/rabetbase-app-list.md)（只读事实；平台目录用 `list --remote`）。登记/移除应用改用 `workspace add` / `workspace remove` |
| dataset commands | [`list`](references/rabetbase-dataset-list.md) / [`detail`](references/rabetbase-dataset-detail.md) / [`delete`](references/rabetbase-dataset-delete.md) / [`restore`](references/rabetbase-dataset-restore.md) / [`generate-start/status`](references/rabetbase-dataset-generate.md) / [`rename`](references/rabetbase-dataset-rename.md) / [`field-update`](references/rabetbase-dataset-field-update.md) / [`extend-update`](references/rabetbase-dataset-extend-update.md) / [`business-group-update`](references/rabetbase-dataset-business-group-update.md) / [`operations`](references/rabetbase-dataset-operations.md) / [`relations`](references/rabetbase-dataset-relations.md) / [`relation-audit`](references/rabetbase-dataset-relation-audit.md) / [`relation-create/update/delete`](references/rabetbase-dataset-relation-mutations.md) |
| page commands | [`generate-start`](references/rabetbase-page-generate-start.md) / [`generate-status`](references/rabetbase-page-generate-status.md) / [`standard-page-status`](references/rabetbase-standard-page-status.md) / [`relation-audit`](references/rabetbase-page-relation-binding.md) / [`sync`](references/rabetbase-page-sync.md) / [`pull`](references/rabetbase-page-pull.md) / [`push`](references/rabetbase-page-push.md) |
| Database Connections (`db`) | [`list`](references/rabetbase-db-list.md) / [`detail`](references/rabetbase-db-detail.md) / [`create`](references/rabetbase-db-create.md) / [`update`](references/rabetbase-db-update.md) / [`delete`](references/rabetbase-db-delete.md) / [`test`](references/rabetbase-db-test.md) / [`analyze`](references/rabetbase-db-analyze.md) / [`tables`](references/rabetbase-db-tables.md) / [`diff`](references/rabetbase-db-diff.md) |
| api commands | [`pull`](references/rabetbase-api-pull.md) / [`list`](references/rabetbase-api-list.md) |
| sql commands | [`list`](references/rabetbase-sql-list.md) / [`detail`](references/rabetbase-sql-detail.md) / [`create`](references/rabetbase-sql-create.md) / [`status`](references/rabetbase-sql-status.md) / [`pull`](references/rabetbase-sql-pull.md) / [`push`](references/rabetbase-sql-push.md) / [`delete`](references/rabetbase-sql-delete.md) / [`validate`](references/rabetbase-sql-validate.md) / [`save`（deprecated）](references/rabetbase-sql-save.md) / [`exec`](references/rabetbase-sql-exec.md) |
| bff commands | [`list`](references/rabetbase-bff-list.md) / [`detail`](references/rabetbase-bff-detail.md) / [`create`](references/rabetbase-bff-create.md) / [`status`](references/rabetbase-bff-status.md) / [`pull`](references/rabetbase-bff-pull.md) / [`push`](references/rabetbase-bff-push.md) / [`delete`](references/rabetbase-bff-delete.md) |
| codegen commands | [`sdk`](references/rabetbase-codegen-sdk.md) / [`sql`](references/rabetbase-codegen-sql.md) |
| CLI Built-in Skill | [`install`](references/rabetbase-cli-skill-install.md) |

## 风险控制

所有声明式命令有 **risk level**：

| 级别              | 含义                                                                                                    | 使用方式                                         |
| ----------------- | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| `read`            | 只读查询，随时可执行                                                                                    | 直接执行                                         |
| `write`           | 修改数据，如 `sql pull`                                                                                 | 先 `--dry-run` 预览，再正式执行                  |
| `high-risk-write` | 影响运行时行为，如 `menu update` / `sql create` / `sql push` / `sql delete` / `bff push` / `bff delete` | 必须 `--yes` 确认或交互确认；CI 模式强制 `--yes` |

`sql validate` 仍是 SQL 内容校验入口；`sql save` 已废弃，不再作为推荐写入路径。

配置 `riskLevel` 可限制允许执行的最高风险等级（详见 [配置参考](references/rabetbase-config.md)）。

## 输出格式

声明式命令统一走 **标准信封**：`ok`、`command`、`risk`、可选 `data`（失败时可有 `error` 等，以实际输出为准）。

| 格式                    | 用途                                   | 说明                                                           |
| ----------------------- | -------------------------------------- | -------------------------------------------------------------- |
| **`--format compress`** | **AI / 脚本优先**                      | 与 `json` **语义相同**，**单行紧凑**、无缩进换行，显著省 token |
| `--format json`         | 程序解析、调试                         | 同上信封，**缩进**，便于人工阅读                               |
| `--format pretty`       | 人类阅读（未指定 format 时的常见默认） | 非 JSON 的彩色文本                                             |
| `--format table`        | 列表数据                               | 表格                                                           |

**`--jq '<expr>'`**（全局）：仅配合 **`json` 或 `compress`**；对**最终打印的整段 JSON**执行 jq（表达式作用在信封上，例如取 `data` 内字段用 `.data.xxx`）。**jq 可执行文件**按 **`JQ_PATH` → CLI 内置 sidecar jq → `PATH` 上的 jq** 的顺序解析。若显式设置了 **`JQ_PATH`**，它必须指向一个真实可执行文件；否则命令会直接报错，不会静默回退。

**不确定子命令、flags、risk、是否要 appcode**：执行 **`rabetbase schema`**（即 `schema export`，**无需认证**），输出与 `rabetbase --help` 同源的机器可读元数据；详见 [`references/rabetbase-schema.md`](references/rabetbase-schema.md) 与仓库 **`docs/user-guide/12-schema命令.md`**。

## 常见错误速查

| 错误类型           | 含义                                                       | 解决方案                                                   |
| ------------------ | ---------------------------------------------------------- | ---------------------------------------------------------- |
| `auth_required`    | 未登录                                                     | 执行 `rabetbase auth`                                      |
| `config_missing`   | 未配置 appcode                                             | `rabetbase workspace init --appcode <code>` 或传 `--appcode` |
| `flag_missing`     | 缺少必填参数                                               | 检查 reference 文档确认必填 flags                          |
| `validation_error` | 输入校验失败（含 SQL 类型阻止）                            | 检查 SQL 内容或参数格式                                    |
| `api_error`        | 后端 API 错误                                              | 检查 appcode、网络、权限                                   |
| `cancelled`        | 用户取消高风险操作                                         | 用 `--yes` 跳过确认，或修改 riskLevel                      |
| `blocked`          | 平台冲突检测或资源状态冲突（常见于旧保存链路或平台侧限制） | 明确说明未完成远端写入，引导用户去平台处理或改走同步工作流 |

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

| 主题                   | Use when                                                                                                              | Guide                                                                       |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| SDK 完整参数与返回值   | 初始化 client、filter/create、sql.execute、bff.execute、错误处理                                                      | [`typescript-sdk.md`](guides/typescript-sdk.md)                             |
| SQL MyBatis 与动态 SQL | 写自定义 SQL、`<if>`/`<foreach>`、参数绑定                                                                            | [`sql-mybatis.md`](guides/sql-mybatis.md)                                   |
| 前端页面开发约束       | React 页、表单、列表、与数据集绑定                                                                                    | [`frontend-development.md`](guides/frontend-development.md)                 |
| 故障诊断               | CLI/登录/数据集/保存失败排障                                                                                          | [`troubleshooting.md`](guides/troubleshooting.md)                           |
| BFF 脚本规范           | HOOK/ENDPOINT/COMMON、`context.client`、目录与注释模板                                                                | [`backend-function.md`](guides/backend-function.md)                         |
| 数据接口访问           | 先 detail 再编码、外键/枚举、禁止 N+1、批量与关联查询；可选 `lovrabet data` 查数（**`lovrabet` CLI ≥ 2.0** 且已安装） | [`data-api-guidelines.md`](guides/data-api-guidelines.md)                   |
| SQL 创建工作流         | list/detail → pull/new → edit → validate → status → push/delete → exec 全链路                                         | [`sql-creation-workflow.md`](guides/sql-creation-workflow.md)               |
| BFF 创建工作流         | new → status → dry-run → pull/push                                                                                    | [`bff-creation-workflow.md`](guides/bff-creation-workflow.md)               |
| 冲突检测与保存         | `blocked`、未保存时的用户沟通、响应结构                                                                               | [`conflict-detection.md`](guides/conflict-detection.md)                     |
| 质量与最佳实践         | 审查 SQL/BFF、命名、高危边界、描述字段                                                                                | [`best-practices.md`](guides/best-practices.md)                             |
| 数据库连接与分析       | 接入/改连/测连、`traceId` 来源、`db analyze-*` 与 dataset 分工；子命令速查见上文 **「数据库连接（db）」**             | [`database-connection-workflow.md`](guides/database-connection-workflow.md) |
| 菜单异常审计与人工修复 | 重复 path、根级菜单重名、需生成平台手动删除清单                                                                       | [`menu-anomaly-manual-cleanup.md`](guides/menu-anomaly-manual-cleanup.md)   |
| 用户组与权限配置       | 建组/加人、按组收紧菜单与接口、单页权限与行级只看自己；先分清成员归属 vs 能力授权                                       | [`role-permit-workflow.md`](guides/role-permit-workflow.md)                 |
