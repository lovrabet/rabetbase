---
name: rabetbase
version: 2.0.4-beta.4
description: "Lovrabet 开发工作流 CLI — 通过 rabetbase 命令管理数据集、SQL 查询、BFF 脚本、菜单同步、代码生成。触发词：数据集、数据表、自定义 SQL、sql.execute、bff.execute、get_dataset_detail、validate_sql_content、save_or_update_custom_sql、@lovrabet/sdk、lovrabet 开发、rabetbase、filter、codegen、init、menu sync、menu update、project create、project upgrade。"
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
> **输出格式：** AI Agent 必须使用 `--format json` 获取结构化输出。

## 前置条件

1. **认证**：`rabetbase auth` 通过浏览器完成 OAuth 登录
2. **AppCode**：确保 `.rabetbase.json` 中设置了 `appcode`（单应用）或 `apps`（多应用），或通过 `--appcode <code>` / `--app <name>` 传入（旧名 `.lovrabet.json` 仍可读）
3. **配置文件**：`rabetbase init` 初始化 `.rabetbase.json`（旧名仍兼容读取）。完整字段说明见 [`.rabetbase.json` 配置参考](references/rabetbase-config.md)
4. **多应用场景**：一个项目有多个应用时，先 `rabetbase app add` 配置各应用，再用 `--app <name>` 或 `rabetbase app use <name>` 切换

## Agent 快速执行顺序

1. **判断需求类型**
   - 标准数据 CRUD → SDK filter/getOne/create/update/delete
   - 简单聚合 → SDK aggregate
   - 复杂 JOIN / 数据库函数 → 自定义 SQL
   - 外部系统调用 / 跨表事务 / 复杂业务编排 → BFF
2. **先拿元数据，再写代码**
   - 至少先查 `rabetbase dataset detail --code xxx --format json` 获取表结构
   - 跨表场景还需查目标表的结构
   - 需要了解数据模型关系时用 `rabetbase dataset links --format json`
3. **SQL 工作流严格分步**
   - 查现有 → 确认字段 → 编写 → 校验(`sql validate`) → 保存(`sql save`) → 测试(`sql exec`)
4. **BFF 工作流严格分步**
   - 查现有 → 确认字段 → 查公共函数 → 本地创建(`bff new`) → 检查状态(`bff status`) → 先预览(`--dry-run`) → 再拉取/推送/删除

## Agent 禁止行为

- **不要猜字段名** — 必须从 `dataset detail` 返回值获取真实字段名、类型、枚举值
- **不要跳过 validate** — SQL 保存前必须通过 `sql validate` 或 `sql save --dry-run` 校验
- **不要手动拼 API URL** — 所有操作通过 CLI 命令完成，不要直接调 HTTP 接口
- **不要臆测 sqlCode / id** — 从 `sql list` 或 `bff list` 获取真实标识
- **不要含糊处理失败** — `sql save` 返回 `blocked` 或 `bff push` / `bff delete` 返回 `failed` 时必须明确告知用户
- **不要在不确认表结构时就写 SQL** — 先 `dataset detail`，后写 SQL
- **不要循环单条查询** — 用 SDK `filter + $in` 批量查询，不要 N+1
- **不要把 MCP 工具名当 CLI 命令** — 使用 `rabetbase sql list`，不是 `list_sql_queries`

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
| 初始化项目 | [`rabetbase init`](references/rabetbase-init.md) | 智能检测旧配置，支持 `--appcode` 非交互 |
| 创建新项目 | [`rabetbase project create`](references/rabetbase-project-create.md) | 支持 `--name` + `--appcode` 非交互 |
| 从 lovrabet-cli 迁移 | [`rabetbase project upgrade`](references/rabetbase-project-upgrade.md) | 6 步自动迁移，`--yes` 跳过确认 |
| 运行 package.json 脚本 | [`rabetbase run <script>`](references/rabetbase-run.md) | 自动检测包管理器，`start`/`dev` 前做版本检查 |
| 安装 skill 包 | [`rabetbase skill install`](references/rabetbase-skill-install.md) | 全局安装 rabetbase skill |
| 退出登录 | [`rabetbase auth logout`](references/rabetbase-auth-logout.md) | 删除本地认证 cookie |
| 诊断配置问题 | [`rabetbase doctor`](references/rabetbase-doctor.md) | 打印合并配置、域名、认证状态 |
| 更新 CLI 版本 | [`rabetbase update`](references/rabetbase-update.md) | 自动检测最新版本并升级 |
| 修改配置文件 | [`rabetbase config set <key> <value>`](references/rabetbase-config.md) | 支持 `--global` 写全局配置 |
| 列出配置 | [`rabetbase config list`](references/rabetbase-config.md) | 查看当前生效的配置 |
| 同步菜单到平台 | [`rabetbase menu sync`](references/rabetbase-menu-sync.md) | 本地页面 → 平台菜单，支持交互/静默 |
| 更新菜单 CDN URL | [`rabetbase menu update`](references/rabetbase-menu-update.md) | 批量更新线上菜单资源地址 |
| 查找数据集 | [`rabetbase dataset list --name "xxx"`](references/rabetbase-dataset-list.md) | 服务端模糊匹配；也可 `--code` 精确查 |
| 查看表结构和字段 | [`rabetbase dataset detail --code xxx`](references/rabetbase-dataset-detail.md) | 含字段定义和操作列表 |
| 查看 Dataset 操作定义 | [`rabetbase dataset operations --code xxx`](references/rabetbase-dataset-operations.md) | 获取 filter/getOne/create 等参数定义 |
| 查看数据模型关系 | [`rabetbase dataset links`](references/rabetbase-dataset-links.md) | 跨表 JOIN 关系图 |
| 生成 API 客户端代码 | [`rabetbase api pull`](references/rabetbase-api-pull.md) | 拉取数据集并生成 `src/api/` TypeScript 代码 |
| 查看生成的 API 模型 | [`rabetbase api list`](references/rabetbase-api-list.md) | 列出已生成的数据模型 |
| 查看现有 SQL | [`rabetbase sql list --name "xxx"`](references/rabetbase-sql-list.md) | 分页，按名称过滤，支持 `--app` 限定应用 |
| 查看 SQL 详情 | [`rabetbase sql detail --sqlcode xxx`](references/rabetbase-sql-detail.md) | 含完整 SQL 内容和参数定义 |
| 校验 SQL 内容 | [`rabetbase sql validate --file xxx`](references/rabetbase-sql-validate.md) | 类型检测、危险语句检查、参数提取 |
| 保存/更新 SQL | [`rabetbase sql save --file xxx`](references/rabetbase-sql-save.md) | 内置校验，不可跳过 |
| 执行 SQL 查询 | [`rabetbase sql exec --sqlcode xxx`](references/rabetbase-sql-exec.md) | 支持 `--params` JSON 参数 |
| 查看现有 BFF | [`rabetbase bff list`](references/rabetbase-bff-list.md) | 按类型和名称过滤，支持 `--app` 限定应用 |
| 查看 BFF 详情 | [`rabetbase bff detail --id n`](references/rabetbase-bff-detail.md) | 含完整脚本内容 |
| 创建本地 BFF | [`rabetbase bff new --type ENDPOINT --name xxx`](references/rabetbase-bff-new.md) | 在 `.rabetbase/bff/<appCode>/...` 下创建脚手架 |
| 查看 BFF 本地状态 | [`rabetbase bff status`](references/rabetbase-bff-status.md) | 检查 added / modified / unchanged / remoteOnly |
| 拉取远端 BFF | [`rabetbase bff pull`](references/rabetbase-bff-pull.md) | 从远端同步到本地 |
| 推送本地 BFF | [`rabetbase bff push --yes`](references/rabetbase-bff-push.md) | high-risk-write，需 `--yes` |
| 删除 BFF | [`rabetbase bff delete --yes --target xxx`](references/rabetbase-bff-delete.md) | high-risk-write，删远端并清理本地 |
| 生成 SDK 代码 | [`rabetbase codegen sdk --code xxx`](references/rabetbase-codegen-sdk.md) | 按操作生成 TypeScript |
| 生成 SQL 调用代码 | [`rabetbase codegen sql --sqlcode xxx`](references/rabetbase-codegen-sql.md) | sdk/bff 两种 target |
| 列出已配置应用 | [`rabetbase app list`](references/rabetbase-app-list.md) | 多应用模式 |
| 切换默认应用 | [`rabetbase app use <name>`](references/rabetbase-app-use.md) | 持久修改 defaultApp |
| 添加应用 | [`rabetbase app add <name> --appcode <code>`](references/rabetbase-app-add.md) | 首个自动设为 default |
| 移除应用 | [`rabetbase app remove <name>`](references/rabetbase-app-remove.md) | 移除后自动切换 default |
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
| Diagnostics | [`doctor`](references/rabetbase-doctor.md) |
| Configuration | [`config set`](references/rabetbase-config.md) / [`config get`](references/rabetbase-config.md) / [`config list`](references/rabetbase-config.md) |
| Menu | [`sync`](references/rabetbase-menu-sync.md) / [`update`](references/rabetbase-menu-update.md) |
| app commands | [`list`](references/rabetbase-app-list.md) / [`use`](references/rabetbase-app-use.md) / [`add`](references/rabetbase-app-add.md) / [`remove`](references/rabetbase-app-remove.md) |
| dataset commands | [`list`](references/rabetbase-dataset-list.md) / [`detail`](references/rabetbase-dataset-detail.md) / [`operations`](references/rabetbase-dataset-operations.md) / [`links`](references/rabetbase-dataset-links.md) |
| api commands | [`pull`](references/rabetbase-api-pull.md) / [`list`](references/rabetbase-api-list.md) |
| sql commands | [`list`](references/rabetbase-sql-list.md) / [`detail`](references/rabetbase-sql-detail.md) / [`validate`](references/rabetbase-sql-validate.md) / [`save`](references/rabetbase-sql-save.md) / [`exec`](references/rabetbase-sql-exec.md) |
| bff commands | [`list`](references/rabetbase-bff-list.md) / [`detail`](references/rabetbase-bff-detail.md) / [`new`](references/rabetbase-bff-new.md) / [`status`](references/rabetbase-bff-status.md) / [`pull`](references/rabetbase-bff-pull.md) / [`push`](references/rabetbase-bff-push.md) / [`delete`](references/rabetbase-bff-delete.md) |
| codegen commands | [`sdk`](references/rabetbase-codegen-sdk.md) / [`sql`](references/rabetbase-codegen-sql.md) |
| Skills | [`install`](references/rabetbase-skill-install.md) |

## 风险控制

所有声明式命令有 **risk level**：

| 级别 | 含义 | 使用方式 |
|------|------|---------|
| `read` | 只读查询，随时可执行 | 直接执行 |
| `write` | 修改数据，如 `sql save` | 先 `--dry-run` 预览，再正式执行 |
| `high-risk-write` | 影响运行时行为，如 `bff push` / `bff delete` | 必须 `--yes` 确认或交互确认；CI 模式强制 `--yes` |

`sql save` 内置 SQL 校验（与 `sql validate` 共用核心），DELETE / DDL 语句被自动阻止。

配置 `riskLevel` 可限制允许执行的最高风险等级（详见 [配置参考](references/rabetbase-config.md)）。

## 输出格式

| 格式 | 用途 | 示例 |
|------|------|------|
| `--format json` | AI Agent / 程序解析 | `{ "ok": true, "command": "...", "risk": "read", "data": {...} }` |
| `--format pretty` | 人类阅读（默认） | 彩色文本输出 |
| `--format table` | 列表数据展示 | 表格输出 |

## 常见错误速查

| 错误类型 | 含义 | 解决方案 |
|---------|------|---------|
| `auth_required` | 未登录 | 执行 `rabetbase auth` |
| `config_missing` | 未配置 appcode | `rabetbase project init` 或传 `--appcode` |
| `flag_missing` | 缺少必填参数 | 检查 reference 文档确认必填 flags |
| `validation_error` | 输入校验失败（含 SQL 类型阻止） | 检查 SQL 内容或参数格式 |
| `api_error` | 后端 API 错误 | 检查 appcode、网络、权限 |
| `cancelled` | 用户取消高风险操作 | 用 `--yes` 跳过确认，或修改 riskLevel |
| `blocked` (sql save) | 平台冲突检测 | 告知用户手动在平台操作，写本地草稿 |

## 冲突处理

`sql save` 返回 `blocked: true` 时：
- 告知用户手动在平台操作
- 将内容写入本地草稿文件（`.draft.sql` / `.draft.js`）
- 禁止重试、禁止绕过
- 未保存成功时必须明确告知用户，禁止含糊带过

## 前端页面规则

- CLI 生成的页面顶部注释必须保留，追加说明用 `@modified`
- 页面组件设置 `displayName`
- 写页面前必须 `dataset detail` 确认字段和关系
- 禁止 emoji、AI 味文案、感叹号、花哨颜色；文案简洁专业
- 图标用 `@ant-design/icons`，颜色用 AntD token

## 深入指南

以下 guide 文件提供各主题的详细说明、完整示例和边界情况：

| 主题 | Guide |
|------|-------|
| SDK 完整参数与返回值 | [`typescript-sdk.md`](guides/typescript-sdk.md) |
| SQL MyBatis 语法与动态 SQL | [`sql-mybatis.md`](guides/sql-mybatis.md) |
| 前端页面开发约束 | [`frontend-development.md`](guides/frontend-development.md) |
| 故障诊断手册 | [`troubleshooting.md`](guides/troubleshooting.md) |
| BFF 脚本编写规范 | [`backend-function.md`](guides/backend-function.md) |
| 数据接口访问规范 | [`data-api-guidelines.md`](guides/data-api-guidelines.md) |
| SQL 创建工作流细则 | [`sql-creation-workflow.md`](guides/sql-creation-workflow.md) |
| BFF 创建工作流细则 | [`bff-creation-workflow.md`](guides/bff-creation-workflow.md) |
| 冲突检测与处理 | [`conflict-detection.md`](guides/conflict-detection.md) |
| 开发质量与最佳实践 | [`best-practices.md`](guides/best-practices.md) |
