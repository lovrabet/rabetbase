# 数据库连接（db）工作流

前置知识：`auth` 已登录，并已通过 `.rabetbase.json` 或 `--appcode` 提供 **appcode**。详见 [SKILL.md](../SKILL.md)「前置条件」。

## 解决什么问题

| 场景 | 说明 |
|------|------|
| 接入已有 MySQL/PostgreSQL 等 | 在平台登记 **dblink**，供后续同步表结构、生成数据集 |
| 改连接串 / 账号 | 更新已有连接，避免在界面点来点去 |
| 验证网络与白名单 | **测连**确认研发环境能打到库 |
| 同步结构到 Lovrabet | **分析任务**把表结构同步到平台；可看进度、取消 |
| 看物理表与差异 | **tables / diff** 在写 SQL、对表前确认真实库状态 |

**与 `dataset` 的区别**：`dataset detail` 里的 `dbId` 来自数据集绑定；**`db list` 的 `id` 是 dblink 主键**，二者数值上通常一致，但 **CLI 管连接用 `db *`，管模型用 `dataset *`**。

## 前置参数从哪拿

```text
appcode     → 配置 / rabetbase app list / --appcode app-xxxx
dblink id   → rabetbase db list 返回 connections[].id（或工作台「数据库连接」列表上的 id）
trace/plan  → 见下一节（分析任务专用）
```

## traceId / planId（`db analyze-*` 用）

三者等价说法：**分析任务 id、plan id、trace id**（平台字段名多为 `latestAnalysisTraceId`）。

| 来源 | 命令 | 字段 |
|------|------|------|
| 列表快照 | `db list` | 每条连接 `latestAnalysisTraceId`、`latestAnalysisStatus` |
| 单条详情 | `db detail --id <n>` | 同上 |
| 刚启动的任务 | `db analyze-start --id <n>` | 响应 `data.planId` |

**`db analyze-status`**：必须传 **`--plan <id>`**，值来自上表之一。

**`db analyze-cancel`**：必须显式传入要取消的既有任务 **`--plan <id>`**。优先使用当前正在跟踪的原 `planId`；`latestAnalysisTraceId` 只是连接快照，不得覆盖已知目标。

## 推荐流程（Agent）

### A. 取消已有任务

```text
已知需要取消的原 <planId>：
  db analyze-cancel --id <id> --plan <原 planId>
    → 只取消该 planId 对应的既有任务，不调用 analyze-start
  db analyze-status --id <id> --plan <原 planId>
    → 继续查询同一 planId；取消请求已提交不等于已到达 CANCELLED
```

- 取消分支不得调用 `analyze-start`，不得为了获得 planId 新建全量或单表任务。
- 只取消该 `planId` 对应的既有任务，不得用连接级最新快照替换明确目标。
- `CANCELING` 不是终态；只有状态真实变为 `CANCELLED` 或其它既有终态后，才按后续业务意图决定是否启动新分析。

### B. 只读排查（已有连接）

```text
db list --format compress
  → 记下目标 connections[].id
db test --id <id>
db tables --id <id>
db diff --id <id>        # 可选：看与上次分析的 schema 差异
```

### C. 新建连接并跑分析

```text
db create --dbname … --dbtype MYSQL --dburl host:port --username … --password … [--autostart]
  → 记下返回的 data.connection.id；可把 data.links.databasePage 返回给用户按需查看
db test --id <id>        # 可选
db analyze-start --id <id>
  → 记下 data.planId；把 data.links.erPage 返回给用户确认 ER 图
db analyze-status --id <id> --plan <planId>   # 轮询直到终态
  → data.isTerminal=true 后复跑 db diff，确认目标差异收敛
  → 检查并返回 data.links.erPage，引导用户确认 ER 图与数据集同步结果
```

### D. 既有连接增量分析

```text
db detail --id <id>
db diff --id <id> --all --changed-only
  → 只取 data.toAnalyzeTables；用户要求跳过的表先从列表中剔除
db analyze-batch-plan --id <id> --tables <toAnalyzeTables> --format compress
  → 保存 data.batches；这是本地 batch plan，不是服务端任务 plan
  → 命令不创建或预留任务，输出不包含 `planId`
  → data.planningEstimate 与 batches[].estimatedDurationSeconds 按每表 60 秒估算，仅用于排期

按 data.batches[].index 顺序处理，每批严格串行：
  db analyze-start --id <id> --tables <当前 batch.tableNames>
    → 只有此时才创建服务端任务；每批只调用一次并独立保存返回的 data.planId
    → 如果没有明确返回 `data.planId`，立即停止；请求结果不确定时不得盲目重试
  db analyze-status --id <id> --plan <当前批次 planId>
    → 始终查询当前批次的同一 planId
    → PENDING / RUNNING / RETRYING / CANCELING：继续等待，不启动下一批
    → SUCCESS：进入下一批
    → PARTIAL_SUCCESS：记录失败事实后进入下一批
    → FAILED / CANCELLED：停止整个批次编排，不再启动后续批次
    → 未知状态：停止自动推进并报告，不猜测为终态

全部批次处理完毕：
db diff --id <id> --all --changed-only
  → 重新读取完整事实；目标差异收敛后才算本轮分析完成
  → 仍未收敛则只对剩余非删除表进入下面的逐表恢复
```

全部批次终态后，如果新的 `data.toAnalyzeTables` 仍非空，按返回顺序逐表处理，每张表在本轮只恢复一次：

```text
对每个 <tableName>（每次只传一张表）：
  db analyze-start --id <id> --tables <tableName>
    → 保存本次返回的 data.planId
  db analyze-status --id <id> --plan <本次 planId>
    → 使用本次 planId 重复查询，直到 data.isTerminal=true
    → 到达终态后才处理下一张表

全部单表任务结束后：
  db diff --id <id> --all --changed-only
    → 以最终 `db diff --all --changed-only` 的 data.toAnalyzeTables 判断是否收敛
    → 已尝试的剩余表只恢复一次，不再次循环提交
```

规划估算的 `basis` 固定为 `heuristic`。排队、表复杂度、关系分析和自动重试可能延长实际耗时，因此估算不是服务端 SLA，也不得驱动超时、取消、重试、自动重启或状态判断；任务事实与实际耗时始终以同一 `planId` 的 `db analyze-status` 返回为准。

- 规划出的每个批次对应一个真实且独立的 `planId`；不存在聚合父 planId，也不得把多个 planId 合成为一个虚假标识。
- 状态查询超时、断网或临时 API 失败时，继续使用同一 `planId` 恢复查询，不重新执行 `analyze-start`。
- 同一个 dblink 不并行启动多个单表任务。单表任务进入 `PARTIAL_SUCCESS`、`FAILED` 或 `CANCELLED` 后记录结果并继续下一张，避免一张异常表阻断其它表。
- 同一张表到达终态后，本轮编排不再无限提交；最终仍未收敛时返回表名、单表任务 `planId`、原始终态及可用排障信息。
- `data.accumulatedErrorTypeCounts` 只原样展示并注明可能不完整；字段缺失、为空或原因码大小写不一致均不得改变编排。
- Agent 可以语义理解 `data.status.errorMsg`，用于解释、建议和问题上报，但要标明它是低置信度辅助证据；不得仅凭文案判断终态、计算失败表数或重新提交任务。

- `PARTIAL_SUCCESS` 是任务终态，不等于目标差异已经收敛。
- `data.isRetrying=true` 只对应后端 `RETRYING`；运行中的 `failedTables` 不是最终失败结论。
- `DELETED_TABLE` 只进入 `data.deletedTables` 供人工确认，不传给 `analyze-start`，也不自动清理。
- 未知状态的 `data.isTerminal` 为 `false`，继续查询或人工排查，不猜测结果。

### E. 改连接再测

```text
db detail --id <id>
db update --id <id> --dburl … [--username …] [--password …]   # 只传要改的字段
db test --id <id>
```

### F. 删除连接（高危）

```text
db delete --id <id> --yes    # high-risk-write，非交互必须 --yes
```

## 与其它命令的配合

- **写 SQL 前要 `dbId`**：`dataset detail --code …` 里 `db.id`；或 `db list` 的 `id` 与数据集侧 `dbId` 对应。
- **看模型关系**：用 `dataset relations`；物理库连接和表清单仍用 `db list` / `db tables` 对照。
- **给用户确认页面**：DB 更新或分析完成后返回 `data.links.erPage`；新建连接后返回 `data.links.databasePage`。
- **不确定 flags**：`rabetbase schema`（无需登录）查契约。

## 输出格式

db 子命令默认 **`--format compress`**（省 token）。需要人工读时用 `--format json`。

## 参考

- 各子命令单页：`../references/rabetbase-db-*.md`
- [SKILL.md](../SKILL.md)
