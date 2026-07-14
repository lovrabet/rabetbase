# db analyze-start / analyze-cancel / analyze-status

管理 **schema 分析任务**：启动全量/增量分析、取消、查询状态。`analyze-start` 与 `analyze-cancel` 为 **write**；`analyze-status` 为 **read**（仅按 plan 查询；命令能解析到 appcode 时会返回页面确认链接）。

## 何时用

- **analyze-start**：接入新库或表结构大变后，同步到 Lovrabet（生成/更新数据集元数据的前置）
- **analyze-cancel**：任务卡住或误触，需要停掉当前分析
- **analyze-status**：轮询任务状态，并读取 CLI 派生的终态、重试和耗时字段

## trace / plan id 从哪来

见 [database-connection-workflow.md](../guides/database-connection-workflow.md) 第二节。摘要：

1. **`db list` / `db detail`** 中的 **`latestAnalysisTraceId`**
2. **`db analyze-start`** 返回的 **`data.planId`**

## 命令

```bash
# 全量或默认分析
rabetbase db analyze-start --id 10157 --format compress

# 仅分析指定表（逗号分隔）
rabetbase db analyze-start --id 10157 --tables orders,order_item --format compress

# 取消（可省略 --plan，CLI 会取该连接 latestAnalysisTraceId）
rabetbase db analyze-cancel --id 10157 --format compress

# 查状态（必须 --plan）
rabetbase db analyze-status --id 10157 --plan <traceId> --format compress
```

## 参数

| 子命令 | 主要 flags |
|--------|------------|
| **analyze-start** | `--id`（必填）、`--tables`（可选，增量表名列表） |
| **analyze-cancel** | `--id`（必填）、`--plan`（可选，缺省用 latestAnalysisTraceId） |
| **analyze-status** | `--id`（必填）、`--plan`（必填） |

## `analyze-status` 输出

| 字段 | 语义 |
|------|------|
| `data.status` | 后端原始任务对象，保留 `status / jobContext / errorMsg` 等排障事实 |
| `data.parsedJobContext` | `jobContext` 为 JSON 对象时的完整解析结果；空值或非法值返回 `null` |
| `data.jobContextParseWarning` | 非法 JSON 或非对象 JSON 的解析警告；正常或空上下文为 `null` |
| `data.isTerminal` | 仅 `SUCCESS / PARTIAL_SUCCESS / FAILED / CANCELLED` 为 `true` |
| `data.isRetrying` | 仅原始状态为 `RETRYING` 时为 `true` |
| `data.elapsedMs` | 优先使用服务端 `costTimeMs`，否则由任务时间戳计算；无法计算时为 `null` |

`PENDING / RUNNING / RETRYING / CANCELING` 与未知状态都不是终态。不要根据 `failedTables`、`retryCount` 或错误文案推测状态。`PARTIAL_SUCCESS` 只表示任务已到终态；必须复跑 `db diff --all --changed-only` 验证目标差异收敛，才能得出本轮分析完成结论。

`RETRYING` 状态下的服务端 `costTimeMs` 可能只代表当前或最近一轮执行耗时，因此此时的 `elapsedMs` 不应解释为整个任务的累计总耗时。

## 输出链接

`analyze-start` 和 `analyze-status` 在配置或 `--appcode` 能提供 appcode 时，会返回：

| 字段 | 用途 |
|------|------|
| `data.links.erPage` | 分析后引导用户打开数据集关系页面确认数据集关系 |
| `data.links.databasePage` | 引导用户打开数据库连接页 |

当 `analyze-status` 返回的原始任务状态为 `SUCCESS` 或 `PARTIAL_SUCCESS` 时，应检查 `data.links.erPage`，并把该链接返回给用户，用于确认 ER 关系与数据集同步结果。若没有 `data.links`，通常是当前命令上下文没有解析到 appcode；重新传 `--appcode` 或在已配置 app 的项目中查询即可生成 ER 页面链接。

链接会按当前环境生成：

```text
daily      https://daily.lovrabet.com/web-app/app/<appCode>/data/er?dbId=<id>
daily      https://daily.lovrabet.com/web-app/app/<appCode>/data/database
production https://app.lovrabet.com/app/<appCode>/data/er?dbId=<id>
production https://app.lovrabet.com/app/<appCode>/data/database
```

## 参考

- [database-connection-workflow.md](../guides/database-connection-workflow.md)
- [SKILL.md](../SKILL.md)
