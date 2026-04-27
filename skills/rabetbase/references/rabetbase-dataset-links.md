# dataset links

获取应用数据集的链接关系图（数据集之间的 JOIN 关联关系）。

## 命令

```bash
rabetbase dataset links --format json
rabetbase dataset links --db ecommerce_db --format json
rabetbase dataset links --db 10157 --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--db <name|id>` | string | 否 | 自动枚举所有数据库 | 按数据库筛选，支持名称或数字 ID |
| `--verbose` | boolean | 否 | `false` | 返回 API 原始完整响应 |
| `--format <fmt>` | string | 否 | `pretty` | 输出格式 |

## 输出

返回每个数据库下所有数据集的字段结构（含 PK/FK 标记）和数据集之间的关联关系。写入 `link-*` 命令时，优先使用关系中的 `fromTable`、`fromColumn`、`toTable`、`toColumn` 作为精确 selector。

```json
{
  "db": "ecommerce_db",
  "dbId": 10157,
  "datasetCount": 15,
  "datasets": [{
    "name": "订单主表",
    "code": "...",
    "table": "order",
    "fields": [
      { "name": "user_id", "displayName": "用户ID", "type": "BIGINT UNSIGNED", "pk": false, "fk": true }
    ],
    "relations": [
      {
        "fromTable": "order",
        "fromColumn": "user_id",
        "toTable": "user",
        "toColumn": "id",
        "cardinality": "ONE_TO_MANY",
        "toDataset": "用户",
        "toCode": "..."
      }
    ]
  }]
}
```

## 提示

- AI 在编写跨数据集 SQL 或 BFF 前，用一次调用获取完整数据模型
- 无需多次调用 `dataset detail` 来了解表间关系
- 调用 `link-create` / `link-update` / `link-delete` 时，只把 `fromTable`、`fromColumn`、`toTable`、`toColumn` 映射为对应 flags；不要从展示名或 datasetCode 自行推断表名
- 只有 `cardinality` 可直接作为 `--cardinality` 的候选值；不要把兼容字段 `joinType` 的非基数值当作写入参数

## 参考

- [SKILL.md](../SKILL.md)
