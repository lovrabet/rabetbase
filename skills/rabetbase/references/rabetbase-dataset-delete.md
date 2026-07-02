# dataset delete

废弃数据集。命令调用服务端删除接口，将平台数据集置为已删除，并由服务端同步删除关联页面和菜单；不会删除物理数据库或物理表。

## 命令

```bash
rabetbase dataset delete --id 44964 --dry-run --format json
rabetbase dataset delete --code 86f8efc75370461f95e1e9526d8d9386 --dry-run --format json
rabetbase dataset delete --dbid 10282 --expected-count 3 --dry-run --format json
```

确认 dry-run 目标列表无误后再正式执行：

```bash
rabetbase dataset delete --dbid 10282 --expected-count 3 --confirm --yes --format json
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--appcode <code>` | 否 | 目标应用编码；未配置默认 app 时必填 |
| `--id <id>` | 与 `--code` / `--dbid` 三选一 | 数据集 ID |
| `--code <code>` | 与 `--id` / `--dbid` 三选一 | Dataset code |
| `--dbid <id>` | 与 `--id` / `--code` 三选一 | 废弃该数据库连接下所有未删除数据集 |
| `--expected-count <n>` | 否 | 命中数量保护；实际命中数不一致时中止 |
| `--confirm` | 正式执行必填 | 确认执行废弃 |
| `--dry-run` | 否 | 只预览目标列表，不调用删除接口 |
| `--yes` | 正式执行建议 | 跳过高风险写入确认；非交互环境必填 |
| `--format <fmt>` | 否 | 输出格式，AI Agent 优先用 `compress` |

## 行为

- 命令只在未删除数据集中定位目标。
- 正式执行调用 `/smartapi/dataset/delete-dataset`。
- 服务端会软删除数据集，并软删除该数据集关联页面和菜单。
- `--dbid` 批量废弃建议始终配合 `--expected-count`。
- 该命令不会删除物理数据库、数据库连接或物理表。

## 验证建议

废弃后可在平台已删除数据集视图确认。需要恢复时使用：

```bash
rabetbase dataset restore --id <datasetId> --dry-run --format json
```

## 参考

- [dataset restore](rabetbase-dataset-restore.md)
- [dataset list](rabetbase-dataset-list.md)
