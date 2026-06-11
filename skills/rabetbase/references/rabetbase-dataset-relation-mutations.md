# dataset relation-create / relation-update / relation-delete

管理应用的单条数据集关联关系。这里的“关联关系”不是传统外键：它可以是 V2 数据集之间的字段映射、自关联父子关系，也可以是 `DB_TABLE` 字段消费 `METADATA` 字典数据集；目标展示字段 `refTableLabelName` / `to-label-column` 是关系的一等属性。

读取数据集关联关系事实使用 `rabetbase dataset relations`。新增、更新、删除单条关联关系分别使用 `rabetbase dataset relation-create` / `relation-update` / `relation-delete`。

未显式传来源/目标类型参数时，来源/目标数据集类型为 `DB_TABLE -> DB_TABLE`；涉及 `METADATA` 时必须显式传 `--from-source METADATA` 或 `--to-source METADATA`。暂不支持 `METADATA -> DB_TABLE`。

## 命令

```bash
rabetbase dataset relation-create --db <dbNameOrId> --from-table <fromTable> --from-column <fromField> --to-table <toTable> --to-column <toField> --to-label-column <toLabelField> --cardinality MANY_TO_ONE --dry-run --format json
rabetbase dataset relation-update --db <dbNameOrId> --from-table <fromTable> --from-column <fromField> --to-table <toTable> --to-column <toField> --to-label-column <nextLabelField> --cardinality MANY_TO_ONE --dry-run --format json
rabetbase dataset relation-update --from-source METADATA --from-code <fromDatasetCode> --from-column <fromField> --to-code <toDatasetCode> --to-column <toField> --expect-to-label-column <currentLabelField> --to-label-column <nextLabelField> --dry-run --format json
rabetbase dataset relation-update --to-source METADATA --db <dbNameOrId> --from-table <fromTable> --from-column <fromField> --to-code <toDatasetCode> --to-column <toField> --to-label-column <toLabelField> --dry-run --format json
rabetbase dataset relation-delete --db <dbNameOrId> --from-table <fromTable> --from-column <fromField> --to-table <toTable> --to-column <toField> --dry-run --format json
```

## 参数

| Flag | create | update | delete | 说明 |
|------|--------|--------|--------|------|
| `--db <name|id>` | 必填 | 必填 | 必填 | 数据库名称或 dblink ID |
| `--from-table <table>` | 必填 | 必填 | 必填 | 源表名 |
| `--from-column <column>` | 必填 | 必填 | 必填 | 源字段名 |
| `--to-table <table>` | 必填 | 必填 | 必填 | 目标表名 |
| `--to-column <column>` | 必填 | 必填 | 必填 | 目标字段名 |
| `--to-label-column <column>` | 必填 | 必填 | 不支持 | 目标展示字段名，决定关联记录展示值 |
| `--cardinality <value>` | 必填 | 必填 | 不支持 | 关系基数；`ONE_TO_ONE` / `ONE_TO_MANY` / `MANY_TO_ONE` / `MANY_TO_MANY` |
| `--dry-run` | 可选 | 可选 | 可选 | 只预览请求，不实际写入；预览中包含 `operation`、`appCode`、`dbId`、`dbName` |
| `--yes` | - | - | 非交互执行必填 | `relation-delete` 是高风险写操作 |
| `--format <fmt>` | 可选 | 可选 | 可选 | 输出格式 |

`relation-update` 支持涉及 METADATA 类型数据集时使用的来源/目标数据集类型参数。默认来源/目标数据集类型为 `DB_TABLE -> DB_TABLE`，DB_TABLE 更新必须显式传 `--db`、`--from-table`、`--to-table`、`--to-label-column` 和 `--cardinality`；不要让 Agent 依赖隐式数据库。

涉及 METADATA 类型数据集的关联关系只支持目标展示字段更新，因此 `--to-label-column` 必须显式传入；未传时 CLI 会拒绝请求，避免向后端提交 no-op update。该类关系的 `cardinality` 会被服务端规范化为 `ONE_TO_MANY`，CLI 会拒绝 `--cardinality` 参数，避免用户误以为传入值会被保留。

支持矩阵：

| From | To | 支持 | 说明 |
|------|----|:---:|------|
| `DB_TABLE` | `DB_TABLE` | 是 | V2 数据集之间关系，底层可映射到表关系 |
| `DB_TABLE` | `METADATA` | 是 | `DB_TABLE` 字段消费 `METADATA` 字典/选项源 |
| `METADATA` | `METADATA` | 是 | V2 标准页面/配置数据集关系 |
| `METADATA` | `DB_TABLE` | 否 | CLI 校验阶段拒绝 |

| Flag | update | 说明 |
|------|--------|------|
| `--from-source <type>` | 可选 | 来源数据集类型，默认 `DB_TABLE`；`METADATA` 来源数据集必须改用 `--from-code` |
| `--to-source <type>` | 可选 | 目标数据集类型，默认沿用 `--from-source`；单独传 `--to-source METADATA` 表示 `DB_TABLE -> METADATA` 字典关系 |
| `--db <name\|id>` | 条件必填 | `--from-source DB_TABLE` 时必填；`--from-source METADATA` 时禁止 |
| `--from-code <code>` | 条件必填 | `--from-source METADATA` 时必填 |
| `--to-code <code>` | 条件必填 | `--to-source METADATA` 时必填 |
| `--to-label-column <column>` | 条件必填 | 任一端为 METADATA 时必填，用于目标展示字段更新 |
| `--expect-to-label-column <column>` | 推荐 | 可选乐观并发检查；DB_TABLE 显式传入时回读当前目标展示字段，未配置目标展示字段时按服务端兜底语义使用 `to-column` 作为默认当前值 |
| `--cardinality` | 条件支持 | 仅 `DB_TABLE -> DB_TABLE` 支持；涉及 METADATA 类型数据集的关系由服务端规范化为 `ONE_TO_MANY`，传入会被 CLI 拒绝 |

## 输出

实际写入返回 `dataset-relation-mutation.v1` 协议数据，包含变更前后状态和执行信息。纯 DB_TABLE `--dry-run` 返回请求预览对象，顶层包含 `operation`、`appCode`、`dbId`、`dbName` 和 `body`，用于确认写入落点。METADATA `relation-update --dry-run` 返回 `dataset-relation-mutation.v1` 审计结构，包含 `before/after/warnings/suggestedExpectFlags`，并同样包含 `body.appCode`、`body.dblinkId` 和 `body.relation` 供复核实际请求体。关系定位只使用结构化字段，不解析字符串 key。

读取关系事实使用 `dataset-relations.v1`，主定位字段为 `datasetCode + field`；单条关系写入命令返回 `dataset-relation-mutation.v1` 执行结果。

```json
{
  "protocol": "dataset-relation-mutation.v1",
  "operation": "create",
  "appCode": "app-xxx",
  "selector": {
    "fromSource": "DB_TABLE",
    "toSource": "DB_TABLE",
    "fromTable": "from_table",
    "fromColumn": "from_field",
    "toTable": "to_table",
    "toColumn": "to_field"
  },
  "db": { "id": 10157, "name": "db_name" },
  "before": null,
  "after": {
    "from": { "table": "from_table", "column": "from_field" },
    "to": { "table": "to_table", "column": "to_field", "labelColumn": "to_label_field" },
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

METADATA `relation-update` 的关系定位使用 `fromCode/toCode`。请求体对 METADATA 来源数据集传 `"dblinkId": null`，并在 `body.relation` 中显式带 `fromSourceType/toSourceType/fromDatasetCode/toDatasetCode`。未传 `--expect-to-label-column` 且当前 label 可读取时，输出 `warnings[].code = "no_drift_check_applied"` 和 `suggestedExpectFlags`。

`DB_TABLE -> METADATA` 场景下，`--to-label-column` 表示消费 METADATA 数据集时使用的目标展示字段。例如来源字段保存目标 `toField`，页面或运行时展示 `toLabelField`。

## 提示

- 写入前如需读取现状，统一运行 `rabetbase dataset relations --format compress` 或 `--format json`，确认来源/目标 `datasetCode + field`、`to.labelField` 和 `relation.cardinality`
- DB_TABLE 写命令需要的 `--db`、`--from-table`、`--to-table` 来自用户明确输入或 `db tables` / `dataset detail` 的物理表事实；不要自行兜底推断
- 纯 DB_TABLE `relation-update` 必须显式传 `--db`；共享自动化可传 `--expect-to-label-column` 做 label 漂移校验
- `--from-source` / `--to-source` 必须使用 `DB_TABLE` 或 `METADATA`；拼写错误不能让它回退到 DB_TABLE 默认路径
- 写命令只使用显式定位 flags；不要从 dataset 展示名、datasetCode 或字符串 key 自行推断表名
- `relation-update` 只用于更新 `to-label-column` 与 `DB_TABLE -> DB_TABLE` 关系的 `cardinality`，不修改来源或目标端字段
- `to-label-column` 必须来自用户明确指定、目标字段清单，或当前关系事实 `to.labelField`；不确定时不要执行更新
- 涉及 METADATA 类型数据集的关系不支持 `--cardinality`；服务端会将该类关系的 cardinality 规范化为 `ONE_TO_MANY`，CLI 会直接拒绝该参数
- 暂不支持 `METADATA -> DB_TABLE`
- `relation-update` 不修改源表、源字段、目标表或目标字段
- 不要在自动化流程中把删除旧关系再新增新关系视为原子编辑操作
- 如确需变更定位字段，先用 `dataset relations` 核对当前关系，分别对删除和新增执行 `--dry-run`，并在用户明确确认后分步执行
- 新增、更新、删除单条关系分别使用 `relation-create` / `relation-update` / `relation-delete`

## 参考

- [dataset relations](rabetbase-dataset-relations.md)
- [SKILL.md](../SKILL.md)
