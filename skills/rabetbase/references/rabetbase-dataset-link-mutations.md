# dataset link-create / link-update / link-delete

管理应用的单条数据集关联关系。

DB_TABLE 类型数据集关联关系是主流程。默认不传来源类型参数时始终按 `DB_TABLE -> DB_TABLE` 处理；METADATA 只是辅助扩展，必须显式传 `--from-source METADATA` 或 `--to-source METADATA` 才能进入 METADATA 类型数据集分支。不要为了 METADATA 场景改变 DB_TABLE 命令心智。

## 命令

```bash
rabetbase dataset link-create --db 10157 --from-table order --from-column user_id --to-table user --to-column id --to-label-column name --cardinality MANY_TO_ONE --dry-run --format json
rabetbase dataset link-update --db 10157 --from-table order --from-column user_id --to-table user --to-column id --to-label-column display_name --cardinality MANY_TO_ONE --dry-run --format json
rabetbase dataset link-update --from-source METADATA --from-code open-record-code --from-column target_smart_cabinet --to-code cabinet-code --to-column id --expect-to-label-column device_code --to-label-column installation_location --dry-run --format json
rabetbase dataset link-update --to-source METADATA --db 10157 --from-table order --from-column smart_cabinet_ref --to-code cabinet-code --to-column id --to-label-column installation_location --dry-run --format json
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
| `--dry-run` | 可选 | 可选 | 可选 | 只预览请求，不实际写入；预览中包含 `operation`、`appCode`、`dbId`、`dbName` |
| `--yes` | - | - | 非交互执行必填 | `link-delete` 是高风险写操作 |
| `--format <fmt>` | 可选 | 可选 | 可选 | 输出格式 |

`link-update` 额外支持涉及 METADATA 类型数据集时使用的来源/目标数据集类型参数。默认来源/目标数据集类型仍是主流程 `DB_TABLE -> DB_TABLE`，DB_TABLE 更新必须显式传 `--db`、`--from-table`、`--to-table`、`--to-label-column` 和 `--cardinality`；不要让 Agent 依赖隐式数据库。

涉及 METADATA 类型数据集的关联关系只支持目标展示字段更新，因此 `--to-label-column` 必须显式传入；未传时 CLI 会拒绝请求，避免向后端提交 no-op update。该类关系的 `cardinality` 会被服务端规范化为 `ONE_TO_MANY`，CLI 会拒绝 `--cardinality` 参数，避免用户误以为传入值会被保留。

| Flag | update | 说明 |
|------|--------|------|
| `--from-source <type>` | 可选 | 来源数据集类型，默认 `DB_TABLE`；`METADATA` 来源数据集必须改用 `--from-code` |
| `--to-source <type>` | 可选 | 目标数据集类型，默认沿用 `--from-source`；单独传 `--to-source METADATA` 表示 `DB_TABLE -> METADATA` |
| `--db <name\|id>` | 条件必填 | `--from-source DB_TABLE` 时必填；`--from-source METADATA` 时禁止 |
| `--from-code <code>` | 条件必填 | `--from-source METADATA` 时必填 |
| `--to-code <code>` | 条件必填 | `--to-source METADATA` 时必填 |
| `--to-label-column <column>` | 条件必填 | 任一端为 METADATA 时必填，用于目标展示字段更新 |
| `--expect-to-label-column <column>` | 推荐 | 可选乐观并发检查；DB_TABLE 显式传入时回读当前目标展示字段，未配置目标展示字段时按服务端兜底语义使用 `to-column` 作为默认当前值 |
| `--cardinality` | 条件支持 | 仅 `DB_TABLE -> DB_TABLE` 支持；涉及 METADATA 类型数据集的关系由服务端规范化为 `ONE_TO_MANY`，传入会被 CLI 拒绝 |

## 输出

实际写入返回 `dataset-link.v1` 协议数据，包含变更前后状态和执行信息。纯 DB_TABLE `--dry-run` 保持旧请求预览对象，顶层包含 `operation`、`appCode`、`dbId`、`dbName` 和 `body`，用于确认写入落点。METADATA `link-update --dry-run` 返回 `dataset-link.v1` 审计结构，包含 `before/after/warnings/suggestedExpectFlags`，并同样包含 `body.appCode`、`body.dblinkId` 和 `body.relation` 供 AI agent 复核实际请求体。关系定位只使用结构化字段，不解析字符串 key。

```json
{
  "protocol": "dataset-link.v1",
  "operation": "create",
  "appCode": "app-xxx",
  "selector": {
    "fromSource": "DB_TABLE",
    "toSource": "DB_TABLE",
    "fromTable": "order",
    "fromColumn": "user_id",
    "toTable": "user",
    "toColumn": "id"
  },
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

METADATA `link-update` 的 `selector` 使用 `fromCode/toCode`。请求体对 METADATA 来源数据集传 `"dblinkId": null`，并在 `body.relation` 中显式带 `fromSourceType/toSourceType/fromDatasetCode/toDatasetCode`。未传 `--expect-to-label-column` 且当前 label 可读取时，输出 `warnings[].code = "no_drift_check_applied"` 和 `suggestedExpectFlags`。

## 提示

- 写入前先运行 `rabetbase dataset links --db <db> --format json`，用 `fromTable`、`fromColumn`、`toTable`、`toColumn` 确认真实 selector
- METADATA 关系先运行 `rabetbase dataset links --from-source METADATA --format json`，用 `fromCode`、`fromColumn`、`toCode`、`toColumn` 确认 selector
- 纯 DB_TABLE `link-update` 必须显式传 `--db`；共享自动化可传 `--expect-to-label-column` 做 label 漂移校验
- `--from-source` / `--to-source` 必须使用 `DB_TABLE` 或 `METADATA`；拼写错误不能让它回退到 DB_TABLE 默认路径
- 写命令只使用显式 selector flags；不要从 dataset 展示名、datasetCode 或字符串 key 自行推断表名
- `link-update` 只用于更新 `to-label-column` 与 `DB_TABLE -> DB_TABLE` 关系的 `cardinality`，它们是可编辑属性，不是关系定位字段
- `to-label-column` 不能从 `dataset links` 推断；必须来自用户明确指定或目标表字段清单，不确定时不要执行更新
- 涉及 METADATA 类型数据集的关系不支持 `--cardinality`；服务端会将该类关系的 cardinality 规范化为 `ONE_TO_MANY`，CLI 会直接拒绝该参数
- `link-update` 不修改源表、源字段、目标表或目标字段
- 不要在自动化流程中把删除旧关系再新增新关系视为原子编辑操作
- 如确需变更定位字段，先用 `dataset links` 核对当前关系，分别对删除和新增执行 `--dry-run`，并在用户明确确认后分步执行
- 新增、更新、删除单条关系分别使用 `link-create` / `link-update` / `link-delete`

## 参考

- [dataset links](rabetbase-dataset-links.md)
- [SKILL.md](../SKILL.md)
