# db analyze-batch-plan / analyze-start / analyze-cancel / analyze-status

规划并管理 **schema 分析任务**。`analyze-batch-plan` 是生成本地批次方案的 **read** 命令；它不需要登录或 appcode，也不会创建、预留或返回服务端 `planId`。`analyze-start` 与 `analyze-cancel` 为 **write**；`analyze-status` 为 **read**（仅按服务端 plan 查询；命令能解析到 appcode 时会返回页面确认链接）。

## 何时用

- **analyze-batch-plan**：增量表清单较多时，先预览确定性安全批次；不创建服务端任务
- **analyze-start**：接入新库或表结构大变后，同步到 Lovrabet（一次调用只启动一个任务）
- **analyze-cancel**：任务卡住或误触，需要停掉当前分析
- **analyze-status**：轮询任务状态，并读取 CLI 派生的终态、重试和耗时字段

## trace / plan id 从哪来

见 [database-connection-workflow.md](../guides/database-connection-workflow.md) 第二节。摘要：

1. **`db list` / `db detail`** 中的 **`latestAnalysisTraceId`**
2. **`db analyze-start`** 返回的 **`data.planId`**

## 命令

```bash
# 预览本地增量表批次（不调用 API、不返回服务端 planId）
rabetbase db analyze-batch-plan --id 10157 --tables orders,order_item --format compress

# 全量或默认分析
rabetbase db analyze-start --id 10157 --format compress

# 仅分析指定表（逗号分隔）
rabetbase db analyze-start --id 10157 --tables orders,order_item --format compress

# 取消既有任务（必须使用要取消任务的原 planId）
rabetbase db analyze-cancel --id 10157 --plan <existingPlanId> --format compress

# 查状态（必须 --plan）
rabetbase db analyze-status --id 10157 --plan <traceId> --format compress
```

## 参数

| 子命令 | 主要 flags |
|--------|------------|
| **analyze-batch-plan** | `--id`（必填）、`--tables`（必填，增量表名列表） |
| **analyze-start** | `--id`（必填）、`--tables`（可选，增量表名列表） |
| **analyze-cancel** | `--id`（必填）、`--plan`（必填，必须是要取消的既有任务） |
| **analyze-status** | `--id`（必填）、`--plan`（必填） |

取消是独立分支：不得先执行 `analyze-start` 获取新 planId，再取消新任务。取消请求提交后，继续使用同一原 planId 查询状态；不要用可能变化的 `latestAnalysisTraceId` 替换已经明确的取消目标。

## `analyze-batch-plan` 输出与执行规则

`analyze-batch-plan` 输出的是本地 batch plan，不是服务端任务计划。它对表名执行 trim、过滤空项和按首次出现顺序去重，再按输入顺序贪心分批。每批最多 10 张表，紧凑 JSON UTF-8 字节数最多 960；单张表自身超过安全预算时返回 validation error。输出不包含 `planId`，只有后续 `analyze-start` 才会创建任务并返回真实 `planId`。规划结果还按每表 60 秒给出启发式耗时估算。

| 字段 | 语义 |
|------|------|
| `data.dbLinkId` | 本次规划的 dblink id |
| `data.tableCount` / `data.batchCount` | 归一化后的表数量与批次数 |
| `data.maxTablesPerBatch` | 每批最多表数，当前为 10 |
| `data.maxSerializedBytesPerBatch` | 规划安全预算，当前为 960 UTF-8 bytes |
| `data.serverInputParamsLimitBytes` | 单次启动的本地硬保护上限，当前为 1024 bytes |
| `data.planningEstimate.basis` | 估算依据，固定为 `heuristic` |
| `data.planningEstimate.assumedSecondsPerTable` | 规划假设，当前为每表 60 秒 |
| `data.planningEstimate.estimatedTotalSeconds` / `estimatedTotalMinutes` | 按归一化表数量计算的总耗时估算 |
| `data.batches[].index` | 从 1 开始的稳定批次序号 |
| `data.batches[].tableCount` | 当前批次表数量 |
| `data.batches[].inputParamsBytes` | `{dbLinkId, tableNames}` 紧凑 JSON 的 UTF-8 字节数 |
| `data.batches[].estimatedDurationSeconds` | 当前批次按每表 60 秒计算的耗时估算 |
| `data.batches[].tableNames` | 当前批次需要传给 `analyze-start --tables` 的有序表名 |

上述估算仅用于排期，不是服务端 SLA。排队、表复杂度、关系分析和自动重试都可能延长实际耗时；不得据此触发超时、取消、重试、自动重启或推断任务状态，任务事实和实际耗时以同一 `planId` 的 `analyze-status` 返回为准。

必须按 `index` 严格串行：每批独立调用一次 `analyze-start`，并保存一个独立 `planId`。当前批次为 `PENDING / RUNNING / RETRYING / CANCELING` 时继续查询同一 planId；`SUCCESS` 才直接进入下一批；`PARTIAL_SUCCESS` 记录失败事实后继续；`FAILED / CANCELLED` 停止整个批次编排；未知状态停止自动推进。

如果 `analyze-start` 没有明确返回 `data.planId`，停止自动推进且不得盲目重试，因为请求结果可能不确定。全部批次结束后重新执行完整 `db diff --all --changed-only`；只对仍在 `data.toAnalyzeTables` 的非删除表逐表串行恢复一次，最后再执行一次完整 diff。

显式单批 `analyze-start --tables` 的持久化参数超过 1024 UTF-8 bytes 时，CLI 会在调用 API 前返回 validation error 并提示先运行 `db analyze-batch-plan`。960 是规划安全预算；960 至 1024 bytes 的显式单批仍允许启动。未传 `--tables` 的全量分析不受此保护影响。

## `analyze-status` 输出

| 字段 | 语义 |
|------|------|
| `data.status` | 后端原始任务对象，保留 `status / jobContext / errorMsg` 等排障事实 |
| `data.parsedJobContext` | `jobContext` 为 JSON 对象时的完整解析结果；空值或非法值返回 `null` |
| `data.jobContextParseWarning` | 非法 JSON 或非对象 JSON 的解析警告；正常或空上下文为 `null` |
| `data.accumulatedErrorTypeCounts` | 原样读取 `parsedJobContext.accumulatedErrorTypeCounts` 的非空对象；缺失、空对象或非法类型返回 `null`，原因码大小写不做归一化 |
| `data.accumulatedErrorTypeCountsAdvisory` | 累计计数的数据质量提示；当前只作为参考信息展示，不作为自动决策输入 |
| `data.isTerminal` | 仅 `SUCCESS / PARTIAL_SUCCESS / FAILED / CANCELLED` 为 `true` |
| `data.isRetrying` | 仅原始状态为 `RETRYING` 时为 `true` |
| `data.elapsedMs` | 优先使用服务端 `costTimeMs`，否则由任务时间戳计算；无法计算时为 `null` |

`PENDING / RUNNING / RETRYING / CANCELING` 与未知状态都不是终态。不要根据 `failedTables`、`retryCount` 或错误文案推测状态。`PARTIAL_SUCCESS` 只表示任务已到终态；必须复跑 `db diff --all --changed-only` 验证目标差异收敛，才能得出本轮分析完成结论。

`analyze-status` 每次只查询一次。轮询时必须继续使用当前任务的同一 `planId`；查询超时、断网或临时 API 失败不等于分析失败，也不得触发新的 `analyze-start`。当前稳定任务状态中没有独立的 `TIMEOUT` 终态。

CLI 不通过固定格式或正则把 `errorMsg` 转换成机器协议；`data.status.errorMsg` 原文仍完整保留。Agent 可以语义理解该原文，并将其作为低置信度辅助证据，用于解释异常、提出排障建议和问题上报，但不得覆盖 `status` 的任务生命周期事实，不得替代最终完整 `db diff` 的收敛结论，也不得仅凭文案重新提交任务。服务端契约完成“顶层异常持久化、成功态保留、同表同轮去重”的回归验收前，`accumulatedErrorTypeCounts` 只作为参考信息展示，不驱动自动分支。

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
