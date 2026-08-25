# db diff

分页查看数据库表差异。只读。查看全部表统一使用 [`db tables`](rabetbase-db-tables.md)。

`db diff` 固定调用 `getDiffTableByPage`，只承担差异查询职责。`--all` 表示聚合全部差异分页，不表示查询全部表。

## 命令

```bash
rabetbase db diff --id 10157 --format compress
rabetbase db diff --id 10157 --all --format compress
```

## 参数

| Flag | 必填 | 说明 |
|---|---|---|
| `--id` | **是** | dblink id |
| `--table` | 否 | 表名模糊过滤 |
| `--page` / `--pagesize` | 否 | 默认 `1` / `20` |
| `--all`                 | 否     | 从第 1 页开始按 `pageSize=100` 自动聚合全部差异分页；优先级高于 `--page / --pagesize` |

## 刷新决策

- DBAgent 增量分析工作流默认先按 [`db diff-refresh-start/status`](rabetbase-db-diff-refresh.md) 完成一次异步差异刷新，再读取 `db diff`。
- 不根据 `tableCount` 的大小或缺失状态跳过刷新。
- 只有用户明确要求“不需要分析”“不刷新”或“只看现有结果”时才直接读取，并说明结果可能不是最新事实。
- 用户明确要求“实时刷新/强制刷新”时，同样执行默认异步刷新流程。

`db diff` 命令自身仍是只读命令，不会在实现内部隐式启动刷新任务；这里要求的是 Agent 在调用 `db diff` 前显式编排 `db diff-refresh-start/status`。

## 机器可读摘要

| 字段 | 语义 |
|---|---|
| `summaryByType` | 本次查询范围内四种 `diffType` 的计数 |
| `toAnalyzeTables` | 仅包含 `NEW_TABLE` 与 `MODIFIED_TABLE` 的表名，可传给 `analyze-start --tables` |
| `deletedTables` | 仅包含 `DELETED_TABLE`，供人工确认，不自动分析或清理 |
| `modifiedFieldDiffs` | 修改表的新增、删除、变更字段摘要 |

未传 `--all` 时，派生字段只代表当前页。差异接口未返回 `physicalTableCount`、`datasetTableCount` 或 `summary` 时，CLI 不以 `0` 伪造缺失值。输出只使用 `tables`，不保留旧 `tableList` 别名。

## `diffType` 语义

| `diffType` | 界面文案 | 含义 |
|---|---|---|
| `NEW_TABLE`        | 表新增   | 物理库中有该表，但尚未纳入或尚未完成上次智能分析。                     |
| `DELETED_TABLE`    | 表删除   | 相对上次已分析结果，该表在库侧已不存在或已被移除。                     |
| `MODIFIED_TABLE`   | 表修改   | 表仍在，但结构相对上次分析有变化。                                     |
| `ALREADY_ANALYZED` | 无差异   | 当前表结构与上次分析结果一致；完整全表事实由 `db tables` 返回。 |

服务端接口只返回差异表，因此 CLI 不再提供冗余的 `--changed-only`。`DELETED_TABLE` 不进入 `toAnalyzeTables`。自动分页在达到 `totalCount` 前遇到空页或没有新增唯一表时，命令失败，不把部分结果伪装成完整结果。

## 参考

- [差异刷新](rabetbase-db-diff-refresh.md)
- [database-connection-workflow.md](../guides/database-connection-workflow.md)
- [SKILL.md](../SKILL.md)
