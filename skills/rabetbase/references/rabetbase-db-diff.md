# db diff

分页查看数据库表差异或全部数据表。只读。

## 两种视角

| 视角           | 命令                 | 服务端语义                                                                                              |
|---|---|---|
| 差异表（默认） | `db diff`            | 调用 `getDiffTableByPage`。表数量不大于 200 时服务端通常实时计算；大于 200 时可能读取最近一次差异快照。 |
| 全部表         | `db diff --view all` | 调用实时分页接口，包含 `ALREADY_ANALYZED`，用于查看无差异表或手动选择重新分析。                         |

`--all` 始终表示“聚合当前视角的全部分页”，不表示“全部表”。需要全部表必须显式传 `--view all`。

## 命令

```bash
# 默认差异表视角
rabetbase db diff --id 10157 --format compress
rabetbase db diff --id 10157 --all --changed-only --format compress

# 全部表实时视角
rabetbase db diff --id 10157 --view all --page 1 --pagesize 20 --format compress
rabetbase db diff --id 10157 --view all --all --format compress
```

## 参数

| Flag | 必填 | 说明 |
|---|---|---|
| `--id` | **是** | dblink id |
| `--view`                | 否     | `diff`（默认）或 `all`                                                                    |
| `--table` | 否 | 表名模糊过滤 |
| `--page` / `--pagesize` | 否 | 默认 `1` / `20` |
| `--all`                 | 否     | 从第 1 页开始按 `pageSize=100` 自动聚合当前视角全部分页；优先级高于 `--page / --pagesize` |
| `--changed-only`        | 否     | 从输出行中排除 `ALREADY_ANALYZED`；保留 `DELETED_TABLE`                                   |

## 刷新决策

先执行 `db detail --id <id>` 读取 `tableCount`：

- `tableCount <= 200`：直接读取 `db diff`；除非用户明确要求，否则不要刷新。
- `tableCount > 200`：读取前建议按 [`db diff-refresh-start/status`](rabetbase-db-diff-refresh.md) 刷新差异快照。
- 用户明确要求“实时刷新/强制刷新”：不考虑表数量，执行刷新。
- 用户明确要求“不刷新/只看现有结果”：直接读取；大库需提示结果可能滞后。
- `tableCount` 缺失：直接读取并说明新鲜度未知，不要因为未知而隐式发起写任务。

`db diff` 是只读命令，绝不隐式启动刷新任务。

## 机器可读摘要

| 字段 | 语义 |
|---|---|
| `view`               | 当前视角：`diff` 或 `all`                                                      |
| `summaryByType` | 本次查询范围内四种 `diffType` 的计数 |
| `toAnalyzeTables` | 仅包含 `NEW_TABLE` 与 `MODIFIED_TABLE` 的表名，可传给 `analyze-start --tables` |
| `deletedTables` | 仅包含 `DELETED_TABLE`，供人工确认，不自动分析或清理 |
| `modifiedFieldDiffs` | 修改表的新增、删除、变更字段摘要 |

未传 `--all` 时，派生字段只代表当前页。默认差异视角通常不会返回 `physicalTableCount`、`datasetTableCount` 或 `summary`；CLI 不以 `0` 伪造缺失值。全表实时视角在服务端提供这些字段时会原样返回。

## `diffType` 语义

| `diffType` | 界面文案 | 含义 |
|---|---|---|
| `NEW_TABLE`        | 表新增   | 物理库中有该表，但尚未纳入或尚未完成上次智能分析。                     |
| `DELETED_TABLE`    | 表删除   | 相对上次已分析结果，该表在库侧已不存在或已被移除。                     |
| `MODIFIED_TABLE`   | 表修改   | 表仍在，但结构相对上次分析有变化。                                     |
| `ALREADY_ANALYZED` | 无差异   | 当前表结构与上次分析结果一致；在 `--view all` 中可供用户主动重新分析。 |

`DELETED_TABLE` 不进入 `toAnalyzeTables`。自动分页在达到 `totalCount` 前遇到空页或没有新增唯一表时，命令失败，不把部分结果伪装成完整结果。

## 参考

- [差异刷新](rabetbase-db-diff-refresh.md)
- [database-connection-workflow.md](../guides/database-connection-workflow.md)
- [SKILL.md](../SKILL.md)
