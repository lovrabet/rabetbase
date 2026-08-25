# db tables

列出指定 dblink 下的全部表对照结果及分析状态。命令使用全表分页接口并自动聚合所有分页，包含 `ALREADY_ANALYZED`、`NEW_TABLE`、`MODIFIED_TABLE` 和 `DELETED_TABLE`。只读。

## 何时用

- 查看全部表，包括无差异表
- 写自定义 SQL / 对表前确认真实表名
- 主动选择需要重新分析的既有表

## 命令

```bash
rabetbase db tables --id 10157 --format compress
rabetbase db tables --id 10157 --table order --format compress
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--id` | **是** | dblink id（`db list`） |
| `--table` | 否 | 表名模糊过滤；仍自动聚合匹配结果的全部分页 |

## 机器可读输出

- `totalCount`：全表对照结果总数
- `physicalTableCount` / `datasetTableCount`：服务端提供时原样返回
- `summaryByType`：四种 `diffType` 的完整结果计数
- `tables`：唯一表结果数组；不输出旧 `tagList` 或 `tableList` 别名

`DELETED_TABLE` 是全表对照事实，可能不属于当前物理表，但仍保留在结果中，避免客户端过滤后破坏总数语义。

## 参考

- [database-connection-workflow.md](../guides/database-connection-workflow.md)
- [SKILL.md](../SKILL.md)
