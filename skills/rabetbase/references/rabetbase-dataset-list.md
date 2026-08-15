# dataset list

列出当前 App 下支持的 Dataset，来源包含 `DB_TABLE` 和 `METADATA`，支持按名称、code 和来源过滤。返回**精简列表**（每行含 `id`、`name`、`code`、`source`、`db`、`table` 等；若存在字段摘要，也会透出 `fields` 字段名列表）。单数据集的完整结构请用 **`dataset detail`**。

## 命令

```bash
rabetbase dataset list --format json
rabetbase dataset list --source METADATA --format json
rabetbase dataset list --source DB_TABLE --format json
rabetbase dataset list --name "用户" --format json
rabetbase dataset list --code "8f737f49c23e4506865b07ebe9dfa316" --format json
rabetbase dataset list --name "用户" --code "abc" --format json
rabetbase dataset list --format table
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--name <name>` | string | 否 | — | 按名称模糊过滤 |
| `--code <code>` | string | 否 | — | 按 dataset code 精确过滤 |
| `--source <source>` | string | 否 | — | 按来源过滤：`DB_TABLE` / `METADATA` |
| `--format <fmt>` | string | 否 | `pretty` | 输出格式：`json` / `pretty` / `table` |

## 输出

默认返回支持的数据集精简列表（`id`, `name`, `code`, `description`, `source`, `db`, `table`, `datasetKey`, `pk`，以及可用时的 `fields` 字段名字符串数组）。`fields` 为空是符合预期的轻量列表行为；METADATA 数据集没有物理表配置时，`db` / `table` / `datasetKey` / `pk` 为空是预期行为。需要完整字段定义、操作、关联页等请执行 `rabetbase dataset detail --code …`。

## 提示

- `--name`、`--code` 和 `--source` 可组合使用，同时传递时取交集
- 传 `--source` 时，结果只包含指定来源
- `--code` 为精确匹配，适合已知 datasetCode 的场景
- `--name` 为模糊匹配，适合搜索探索
- 要回答“有哪些 METADATA 类型数据集”时，优先使用 `rabetbase dataset list --source METADATA --format json`
- AI Agent 推荐始终使用 `--format json`
- 列表项较多时可用 `--jq` 只取需要的键，例如：`--jq '.data.datasets[] | {code, name}'`

## 参考

- [SKILL.md](../SKILL.md)
