# db diff-refresh-start / diff-refresh-status

显式刷新服务端数据库表差异结果。`diff-refresh-start` 为 **write**，只提交一次异步任务；`diff-refresh-status` 为 **read**，只查询一次指定 `traceId`。

## 何时刷新

- DBAgent 增量分析工作流默认刷新，无论 `tableCount` 多少或是否缺失。
- 用户明确要求“实时刷新”或“强制刷新”时执行刷新。
- 只有用户明确要求“不需要分析”“不刷新”或“只看现有结果”时才跳过，并说明现有差异可能不是最新事实。

刷新用于形成一致、可跟踪的最新差异事实。刷新成功后，按意图执行 `db diff` 查询差异表，或执行 `db tables` 查询包含无差异表在内的全部表。

## 命令

```bash
rabetbase db diff-refresh-start --id 10157 --format compress
rabetbase db diff-refresh-status --id 10157 --plan <traceId> --format compress
```

## 执行规则

1. `diff-refresh-start` 只执行一次并保存 `data.traceId`。
2. 若响应没有明确 `traceId`，结果未知，停止且不得自动重提。
3. `PENDING / RUNNING / RETRYING` 时继续执行 `data.query.command`，始终查询同一个 traceId。
4. `SUCCESS` 时按原始意图执行 `db diff --all`（差异表）或 `db tables`（全部表）。状态命令不替调用者选择后续查询。
5. `FAILED / CANCELLED` 为失败终态；报告 `data.status.errorMsg`，不得自动重启。
6. 未知状态不是终态，停止自动推进或继续只读查询，不猜测成功。

## 输出

`diff-refresh-start`：

- `data.dbLinkId`
- `data.traceId`
- `data.status`
- `data.query.command`

`diff-refresh-status`：

- `data.status`：服务端原始任务事实
- `data.knownStatus`：是否属于已知运行态或终态；未知时为 `false` 且不提供自动查询命令
- `data.isTerminal`
- `data.isSuccessful`
- `data.query`：非终态时的同 traceId 查询命令

`diff-refresh-status` 只表达任务状态，不输出结果查询 `lookup`，避免在“全部表”和“差异表”两种意图之间替 Agent 作错误选择。

差异刷新任务与 schema 分析任务不是同一业务动作。不要用 `diff-refresh-start` 代替 `analyze-start`，也不要把刷新成功描述为数据集分析已经完成。

## 参考

- [db diff](rabetbase-db-diff.md)
- [database-connection-workflow.md](../guides/database-connection-workflow.md)
- [SKILL.md](../SKILL.md)
