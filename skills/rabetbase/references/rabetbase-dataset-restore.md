# dataset restore

恢复已删除数据集。命令按 Dataset id 定位唯一回收站删除日志并只恢复 Dataset；不会隐式恢复关联页面和菜单。

## 命令

```bash
rabetbase dataset restore --id 44791 --dry-run --format json
rabetbase dataset restore --code 058b8cb7aac0466eb1b3332fd6cd381d --dry-run --format json
rabetbase dataset restore --dbid 10282 --expected-count 3 --dry-run --format json
```

确认 dry-run 目标列表无误后再正式执行：

```bash
rabetbase dataset restore --dbid 10282 --expected-count 3 --confirm --yes --format json
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--appcode <code>` | 否 | 目标应用编码；未配置默认 app 时必填 |
| `--id <id>` | 与 `--code` / `--dbid` 三选一 | 数据集 ID |
| `--code <code>` | 与 `--id` / `--dbid` 三选一 | Dataset code |
| `--dbid <id>` | 与 `--id` / `--code` 三选一 | 恢复该数据库连接下所有已删除数据集 |
| `--expected-count <n>` | 否 | 命中数量保护；实际命中数不一致时中止 |
| `--confirm` | 正式执行必填 | 确认执行恢复 |
| `--dry-run` | 否 | 只预览目标列表，不调用恢复接口 |
| `--yes` | 正式执行建议 | 跳过高风险写入确认；非交互环境必填 |
| `--format <fmt>` | 否 | 输出格式，AI Agent 优先用 `compress` |

## 行为

- 命令只在已删除数据集中定位目标。
- dry-run 输出每个 Dataset 的 `relatedPageCount`，便于判断后续是否需要恢复页面。
- 每个 Dataset 必须恰好匹配一条删除记录；记录缺失或存在多条匹配记录时中止。
- 恢复结果固定标记 `pagesRestored=false`。关联页面需按需执行 `rabetbase page restore --id <pageId> --appcode <appCode>`；数据列表页恢复会恢复对应菜单。
- `--dbid` 批量恢复建议始终配合 `--expected-count`。
- 该命令不会创建新的物理表，也不会修改业务数据。

## 验证建议

恢复后可重新查看数据集详情：

```bash
rabetbase dataset detail --code <datasetCode> --format compress
rabetbase page restore --id <pageId> --appcode <appCode> --dry-run --format compress
```

## 参考

- [dataset delete](rabetbase-dataset-delete.md)
- [dataset list](rabetbase-dataset-list.md)
