# dataset relations

读取应用的数据集关联关系清单，默认读取 D0/DO V2 关系事实。这里的关联关系不是数据库外键：它可以是 V2 数据集之间的字段映射、自关联父子关系，也可以是 `DB_TABLE` 字段消费 `METADATA` 字典数据集。默认输出只包含稳定领域字段，Agent 不消费服务端原始 relation。

## 命令

```bash
rabetbase dataset relations --format compress
rabetbase dataset relations --datasetcode <datasetCode> --format compress
rabetbase dataset relations --from-source DB_TABLE --to-source METADATA --format compress
```

## 参数

| Flag | 类型 | 必填 | 说明 |
|------|------|------|------|
| `--datasetcode <code>` | string | 否 | 只读取指定来源数据集的关系 |
| `--from-source <type>` | string | 否 | 来源数据集类型：`DB_TABLE` / `METADATA` |
| `--to-source <type>` | string | 否 | 目标数据集类型：`DB_TABLE` / `METADATA` |
| `--appcode <code>` | string | 否 | 覆盖当前配置中的 AppCode |

暂不支持 `METADATA -> DB_TABLE`，命令会在校验阶段拒绝该方向。

## 输出

返回 `dataset-relations.v1` 协议。主定位字段是 `datasetCode + field`；`table` 仅作为 `DB_TABLE` 的物理表辅助信息，`METADATA` 侧不会模拟物理表。目标展示字段使用 `to.labelField`，关系基数使用 `relation.cardinality`。

```json
{
  "protocol": "dataset-relations.v1",
  "appCode": "app-xxx",
  "total": 1,
  "relations": [{
    "from": {
      "source": "DB_TABLE",
      "datasetCode": "from-dataset-code",
      "datasetName": "来源数据集",
      "table": "from_table",
      "field": "from_field"
    },
    "to": {
      "source": "METADATA",
      "datasetCode": "to-dataset-code",
      "datasetName": "目标数据集",
      "field": "to_field",
      "labelField": "to_label_field"
    },
    "relation": {
      "cardinality": "ONE_TO_MANY"
    }
  }]
}
```

## Agent 使用规则

- 页面关系审计、标准页面绑定、字典/选项源判断，直接使用 `dataset relations --format compress`；命令底层默认读取 D0/DO V2 关系事实
- 写入前用本命令确认来源/目标 `datasetCode + field`、`to.labelField` 和 `relation.cardinality`
- `DB_TABLE -> METADATA` 表示业务表字段消费 METADATA 字典，不要把它理解为数据库外键
- `METADATA -> METADATA` 可以是自关联父子关系；是否自关联由 `from.datasetCode === to.datasetCode` 自行判断，不在协议里增加派生字段
- 读取协议只表达关系事实，不输出派生能力字段
- 物理库连接、测连、表清单和结构差异属于 `rabetbase db ...` 命令职责；数据集关系读取统一使用本命令

## 参考

- [dataset relation mutations](rabetbase-dataset-relation-mutations.md)
- [SKILL.md](../SKILL.md)
