# dataset link-create / link-update / link-delete

管理应用数据集 ER 图中的单条关联关系。

## 命令

```bash
rabetbase dataset link-create --db 10157 --from-table order --from-column user_id --to-table user --to-column id --to-label-column name --cardinality MANY_TO_ONE --dry-run --format json
rabetbase dataset link-update --db 10157 --from-table order --from-column user_id --to-table user --to-column id --to-label-column display_name --cardinality MANY_TO_ONE --dry-run --format json
rabetbase dataset link-delete --db 10157 --from-table order --from-column user_id --to-table user --to-column id --dry-run --format json
```

## 参数

| Flag | create | update | delete | 说明 |
|------|--------|--------|--------|------|
| `--db <name|id>` | 必填 | 必填 | 必填 | 数据库名称或 dblink ID |
| `--from-table <table>` | 必填 | 必填 | 必填 | 源表名 |
| `--from-column <column>` | 必填 | 必填 | 必填 | 源字段名 |
| `--to-table <table>` | 必填 | 必填 | 必填 | 目标表名 |
| `--to-column <column>` | 必填 | 必填 | 必填 | 目标字段名 |
| `--to-label-column <column>` | 必填 | 必填 | 不支持 | 目标展示字段名，可编辑属性，非定位字段 |
| `--cardinality <value>` | 必填 | 必填 | 不支持 | 关系基数，可编辑属性；`ONE_TO_ONE` / `ONE_TO_MANY` / `MANY_TO_ONE` / `MANY_TO_MANY` |
| `--dry-run` | 可选 | 可选 | 可选 | 只预览请求，不实际写入 |
| `--yes` | - | - | 非交互执行必填 | `link-delete` 是高风险写操作 |
| `--format <fmt>` | 可选 | 可选 | 可选 | 输出格式 |

## 输出

写命令返回 `dataset-link.v1` 协议数据，包含变更前后状态和执行信息。关系定位只使用结构化字段，不解析字符串 key。

```json
{
  "protocol": "dataset-link.v1",
  "operation": "create",
  "appCode": "app-xxx",
  "db": { "id": 10157, "name": "ecommerce_db" },
  "before": null,
  "after": {
    "from": { "table": "order", "column": "user_id" },
    "to": { "table": "user", "column": "id", "labelColumn": "name" },
    "cardinality": "MANY_TO_ONE"
  },
  "dryRun": false,
  "backend": {
    "method": "POST",
    "path": "/smartapi/question/er-config/erCreate"
  },
  "warnings": []
}
```

`before` / `after` 只包含 ER 读取结果或本次输入中可稳定确认的信息；定位关系时使用 `from.table`、`from.column`、`to.table`、`to.column`。

## 提示

- 写入前先运行 `rabetbase dataset links --db <db> --format json`，用 `fromTable`、`fromColumn`、`toTable`、`toColumn` 确认真实 selector
- 写命令只使用显式 selector flags；不要从 dataset 展示名、datasetCode 或字符串 key 自行推断表名
- `link-update` 只用于更新 `to-label-column` 与 `cardinality`，它们是可编辑属性，不是关系定位字段
- `to-label-column` 不能从 `dataset links` 推断；必须来自用户明确指定或目标表字段真源，不确定时不要执行更新
- `link-update` 不修改源表、源字段、目标表或目标字段
- 不要在自动化流程中把删除旧关系再新增新关系视为原子编辑操作
- 如确需变更定位字段，先用 `dataset links` 核对当前关系，分别对删除和新增执行 `--dry-run`，并在用户明确确认后分步执行
- 新增、更新、删除单条关系分别使用 `link-create` / `link-update` / `link-delete`

## 参考

- [dataset links](rabetbase-dataset-links.md)
- [SKILL.md](../SKILL.md)
