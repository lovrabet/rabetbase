# db diff

分页查看**线上库相对上次分析**的 schema 差异（新增表、删除表、改字段等）。只读。

## 何时用

- 表结构变更后，决定是跑全量分析还是 `analyze-start --tables …` 增量
- 排查「平台数据集与真实库不一致」

## 命令

```bash
rabetbase db diff --id 10157 --format compress
rabetbase db diff --id 10157 --table order --page 1 --pagesize 20 --format compress
rabetbase db diff --id 10157 --all --changed-only --format compress
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--id` | **是** | dblink id |
| `--table` | 否 | 表名模糊过滤 |
| `--page` / `--pagesize` | 否 | 默认 `1` / `20` |
| `--all` | 否 | 从第 1 页开始按 `pageSize=100` 自动聚合全部分页；优先级高于 `--page / --pagesize` |
| `--changed-only` | 否 | 只从 `tables / tableList` 排除 `ALREADY_ANALYZED`；保留 `DELETED_TABLE` |

## 机器可读摘要

| 字段 | 语义 |
|------|------|
| `summaryByType` | 本次查询范围内四种 `diffType` 的计数 |
| `toAnalyzeTables` | 仅包含 `NEW_TABLE` 与 `MODIFIED_TABLE` 的表名，可传给 `analyze-start --tables` |
| `deletedTables` | 仅包含 `DELETED_TABLE`，供人工确认，不自动分析或清理 |
| `modifiedFieldDiffs` | 修改表的新增、删除、变更字段摘要 |

未传 `--all` 时，上述字段只代表当前页；需要完整范围时使用 `--all`。派生字段基于过滤前的查询范围计算，因此使用 `--changed-only` 时，`summaryByType.ALREADY_ANALYZED` 仍可反映原始计数。

## `diffType` 语义（与 yuntoo-web-app 增量分析抽屉一致）

接口返回 `IDbTableDiffInfo.diffType`，工作台「数据库连接 → 增量分析」里对每行表的展示文案如下，可与 CLI 输出对照：

| `diffType` | 界面文案 | 含义 |
|------------|----------|------|
| `NEW_TABLE` | 表新增 | 物理库中**有**该表，但**尚未纳入**（或尚未完成）上次智能分析；适合勾选后走 `db analyze-start --tables …` 做增量分析。 |
| `DELETED_TABLE` | 表删除 | 相对上次已分析快照，该表在库侧**已不存在**（或已被移除）；界面上勾选增量分析时**不可选**该行。 |
| `MODIFIED_TABLE` | 表修改 | 表仍在，但**结构相对上次分析有变**；同条记录里可能有 `fieldDiff`（`addedColumns` / `deletedColumns` / `modifiedColumns`）。 |
| `ALREADY_ANALYZED` | 无差异 | 当前表结构与上次分析结果**一致**，无需因本表再跑增量（界面用灰色「无差异」标签）。 |

`DELETED_TABLE` 不会进入 `toAnalyzeTables`。如果自动分页在达到 `totalCount` 前遇到空页或不再产生新的唯一表，命令会失败，不把部分结果当作完整结果返回。

`--all` 聚合的是服务端实时分页结果，不是事务一致性快照。数据库结构在翻页期间持续变化时，结果可能来自不同查询时刻；检测到分页无进展会直接失败。此时应等待库表结构稳定后重试，并在分析结束后再次执行 `db diff --all --changed-only` 验证差异收敛。

## 参考

- [database-connection-workflow.md](../guides/database-connection-workflow.md)
- [SKILL.md](../SKILL.md)
