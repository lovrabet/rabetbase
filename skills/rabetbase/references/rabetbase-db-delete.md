# db delete

逻辑删除一条 dblink 配置。**high-risk-write**：必须先 dry-run 检查依赖，正式执行显式传 `--confirm`；非交互环境同时传 **`--yes`**。

## 何时用

- 下线环境、误建连接需清理（确认无数据集仍依赖该连接）

## 命令

```bash
rabetbase db delete --id 10157 --dry-run --format compress
rabetbase db delete --id 10157 --expected-dataset-count 0 --confirm --yes --format compress
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--id` | **是** | dblink id |
| `--confirm` | 正式执行必填 | 确认删除当前无依赖连接 |
| `--expected-dataset-count` | 否 | 与执行前有效 DB_TABLE Dataset 数不一致时中止，可显式使用 `0` |
| `--yes` | CI/脚本必填 | 跳过高危确认 |
| `--appcode` | 否 | 覆盖 app |

## 安全边界

- dry-run 输出连接基本信息、`datasetDependency.hasDependencies`、`activeDatasetCount` 与 Dataset id/code/name/tableName。
- 正式删除前再次检查 Dataset 依赖；只要存在依赖就中止，不提供 `--force`，也不会自动删除 Dataset。

## 参考

- [database-connection-workflow.md](../guides/database-connection-workflow.md)
- [SKILL.md](../SKILL.md)「风险控制」
