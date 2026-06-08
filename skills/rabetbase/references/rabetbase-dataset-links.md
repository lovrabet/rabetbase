# dataset links

获取应用的数据集关联关系清单。

## 命令

```bash
rabetbase dataset links --format json
rabetbase dataset links --db ecommerce_db --format json
rabetbase dataset links --db 10157 --format json
rabetbase dataset links --from-source METADATA --appcode app-xxx --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--from-source <type>` | string | 否 | `DB_TABLE` | 读取哪类数据集作为关系来源；`DB_TABLE` / `METADATA` |
| `--db <name|id>` | string | 否 | 自动枚举所有数据库 | 按数据库筛选，支持名称或数字 ID |
| `--verbose` | boolean | 否 | `false` | 返回 API 原始完整响应 |
| `--format <fmt>` | string | 否 | `pretty` | 输出格式 |

## 输出

DB_TABLE 是主流程。默认返回每个数据库下 DB_TABLE 数据集的字段结构（含 PK/FK 标记）和数据集之间的关联关系。写入 DB_TABLE `link-*` 命令时，优先使用关系中的 `fromTable`、`fromColumn`、`toTable`、`toColumn` 作为精确 selector。

METADATA 关系使用 `--from-source METADATA` 读取，写入 `link-update` 时使用 `fromCode`、`fromColumn`、`toCode`、`toColumn` 作为 selector。

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
        "fromCode": "...",
        "fromSource": "DB_TABLE",
        "fromColumn": "user_id",
        "toTable": "user",
        "toCode": "...",
        "toSource": "DB_TABLE",
        "toColumn": "id",
        "cardinality": "ONE_TO_MANY",
        "toDataset": "用户"
      }
    ]
  }]
}
```

## 提示

- AI 在编写跨数据集 SQL 或 BFF 前，用一次调用获取完整数据模型
- 无需多次调用 `dataset detail` 来了解表间关系
- `--from-source` 只允许 `DB_TABLE` 或 `METADATA`；拼写错误必须修正，不要让它回退到 DB_TABLE 默认路径
- 调用 DB_TABLE `link-create` / `link-update` / `link-delete` 时，只把 `fromTable`、`fromColumn`、`toTable`、`toColumn` 映射为对应 flags；不要从展示名或 datasetCode 自行推断表名
- 调用 METADATA `link-update` 时，用 `--from-source METADATA` 或 `--to-source METADATA`，映射 `fromCode/toCode` 到 `--from-code/--to-code`，并显式传入新的 `--to-label-column`
- 只有 `DB_TABLE -> DB_TABLE` 关系的 `cardinality` 可作为 `--cardinality` 的候选值；涉及 METADATA 类型数据集的关系不支持 `--cardinality`，服务端会将该类关系的 cardinality 规范化为 `ONE_TO_MANY`

## 参考

- [SKILL.md](../SKILL.md)
