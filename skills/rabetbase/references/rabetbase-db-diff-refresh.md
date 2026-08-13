# db diff-refresh-start / diff-refresh-status

显式刷新服务端数据库表差异结果。`diff-refresh-start` 为 **write**，只提交一次异步任务；`diff-refresh-status` 为 **read**，只查询一次指定 `traceId`。

## 何时刷新

- `db detail` 返回 `tableCount > 200`，读取差异前建议刷新。
- 用户明确要求“实时刷新”或“强制刷新”，无论表数量多少都刷新。
- `tableCount <= 200` 时服务端通常实时计算差异，默认不刷新。

刷新只影响默认 `db diff` 差异视角；`db diff --view all` 本身就是实时全表分页，不需要先刷新差异快照。

## 命令

```bash
rabetbase db diff-refresh-start --id 10157 --format compress
rabetbase db diff-refresh-status --id 10157 --plan <traceId> --format compress
```

## 执行规则

1. `diff-refresh-start` 只执行一次并保存 `data.traceId`。
2. 若响应没有明确 `traceId`，结果未知，停止且不得自动重提。
3. `PENDING / RUNNING / RETRYING` 时继续执行 `data.query.command`，始终查询同一个 traceId。
4. `SUCCESS` 时执行 `data.lookup.command`，重新读取刷新后的差异结果。
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
- `data.lookup`：仅成功时返回的 `db diff --all --changed-only` 命令

差异刷新任务与 schema 分析任务不是同一业务动作。不要用 `diff-refresh-start` 代替 `analyze-start`，也不要把刷新成功描述为数据集分析已经完成。

## 参考

- [db diff](rabetbase-db-diff.md)
- [database-connection-workflow.md](../guides/database-connection-workflow.md)
- [SKILL.md](../SKILL.md)
