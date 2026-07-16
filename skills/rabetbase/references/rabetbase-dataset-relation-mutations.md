# dataset relation-create / relation-update / relation-delete

管理应用的单条数据集关联关系。这里的“关联关系”不是传统外键：它可以是 V2 数据集之间的字段映射、自关联父子关系，也可以是 `DB_TABLE` 字段消费 `METADATA` 字典数据集；目标展示字段 `refTableLabelName` / `to-label-column` 是关系的一等属性。

读取数据集关联关系事实使用 `rabetbase dataset relations`。新增、更新、删除单条关联关系分别使用 `rabetbase dataset relation-create` / `relation-update` / `relation-delete`。

create 使用来源和目标 datasetCode 描述关系两端。relation-update 只要求 `appCode + relationId` 定位待更新关系，其余关系参数都是可选覆盖项；CLI 先读取完整关系，再合并显式参数并提交完整对象。CLI 会读取合并后的数据集类型并做方向校验。支持 `DB_TABLE -> DB_TABLE`、`DB_TABLE -> METADATA`、`METADATA -> METADATA`；暂不支持 `METADATA -> DB_TABLE`。

## Agent 安全边界

- 面向用户答疑时，只给出“先只读确认、再生成 dry-run 方案、用户确认后执行”的路径，不直接让用户执行正式写入。
- 关系缺失时使用 `relation-create --dry-run` 预览；展示字段或支持的关系属性不符合预期时使用 `relation-update --dry-run` 预览。
- 正式写入后 CLI 会自动回读关系事实验证结果；如果后端调用返回但关系未创建、未更新或未删除，命令会失败，不要把这种失败当作已落库。
- `relation-update` 可覆盖关系两端字段，但执行前必须通过 `--dry-run` 核对合并后的完整关系对象，避免把关系移动到错误的数据集或字段。
- 不要把关系写入、页面审计和 `page sync` 混成“一键修复”。关系事实修正后，再重新执行 `page relation-audit`，页面仍需同步时再走 `page sync --dry-run`。

## 命令

```bash
rabetbase dataset relation-create --from-datasetcode <fromDatasetCode> --from-column <fromField> --to-datasetcode <toDatasetCode> --to-column <toField> --to-label-column <toLabelField> --cardinality MANY_TO_ONE --dry-run --format json
rabetbase dataset relation-update --appcode <appCode> --relation-id <relationId> --to-label-column <nextLabelField> --dry-run --format json
rabetbase dataset relation-update --appcode <appCode> --relation-id <relationId> --expect-to-label-column <currentLabelField> --to-label-column <nextLabelField> --dry-run --format json
rabetbase dataset relation-delete --appcode <appCode> --relation-id <relationId> --dry-run --format json
```

## 参数

| Flag | create | update | delete | 说明 |
|------|--------|--------|--------|------|
| `--appcode <appCode>` | 可选 | 必填 | 必填 | 目标应用 code；update/delete 必须显式传入 |
| `--from-datasetcode <code>` | 必填 | 可选覆盖 | 不支持 | 来源数据集 code；update 省略时沿用当前值 |
| `--from-column <column>` | 必填 | 可选覆盖 | 不支持 | 源字段名；update 省略时沿用当前值 |
| `--to-datasetcode <code>` | 必填 | 可选覆盖 | 不支持 | 目标数据集 code；update 省略时沿用当前值 |
| `--to-column <column>` | 必填 | 可选覆盖 | 不支持 | 目标字段名；update 省略时沿用当前值 |
| `--to-label-column <column>` | 必填 | 可选覆盖 | 不支持 | 目标展示字段名；update 省略时沿用当前值 |
| `--cardinality <value>` | 必填 | 可选覆盖 | 不支持 | create 的所有支持方向均必填；create/update 均使用显式合法值，update 省略时沿用已有值，缺失时默认 `ONE_TO_MANY` |
| `--biz-relation-type <type>` | 可选 | 可选覆盖 | 不支持 | 业务关系类型；update 的所有支持方向均可设置为 `normal` / `main_sub` |
| `--relation-id <id>` | 不支持 | 必填 | 必填 | 关系记录 ID；update/delete 使用该 ID 定位关系 |
| `--dry-run` | 可选 | 可选 | 可选 | 只预览请求，不实际写入；预览中包含应用、关系定位和待写入字段 |
| `--yes` | - | - | 非交互执行必填 | `relation-delete` 是高风险写操作 |
| `--format <fmt>` | 可选 | 可选 | 可选 | 输出格式 |

`relation-update` 先按 `appCode + relationId` 读取完整关系，再用显式参数覆盖同名字段。最终请求始终包含 relationId、两端 datasetCode/字段、目标展示字段、关系基数和业务关系类型。所有支持方向都可显式传入 `--cardinality` 与 `--biz-relation-type`，不区分 DB_TABLE/METADATA；cardinality 使用显式合法值，省略时沿用已有值，已有值缺失时默认 `ONE_TO_MANY`。

支持矩阵：

| From | To | 支持 | 说明 |
|------|----|:---:|------|
| `DB_TABLE` | `DB_TABLE` | 是 | V2 数据集之间关系，底层可映射到表关系 |
| `DB_TABLE` | `METADATA` | 是 | `DB_TABLE` 字段消费 `METADATA` 字典/选项源 |
| `METADATA` | `METADATA` | 是 | V2 标准页面/配置数据集关系 |
| `METADATA` | `DB_TABLE` | 否 | CLI 校验阶段拒绝 |

| Flag | update | 说明 |
|------|--------|------|
| `--appcode <appCode>` | 必填 | 目标应用 code |
| `--relation-id <id>` | 必填 | 待更新关系的记录 ID |
| `--from-datasetcode <code>` | 可选 | 覆盖来源数据集 code |
| `--from-column <column>` | 可选 | 覆盖来源字段 |
| `--to-datasetcode <code>` | 可选 | 覆盖目标数据集 code |
| `--to-column <column>` | 可选 | 覆盖目标字段 |
| `--to-label-column <column>` | 可选 | 覆盖目标展示字段 |
| `--expect-to-label-column <column>` | 推荐 | 可选乐观并发检查；DB_TABLE 显式传入时回读当前目标展示字段，未配置目标展示字段时按服务端兜底语义使用 `to-column` 作为默认当前值 |
| `--cardinality` | 可选 | 所有支持方向均使用显式合法值；省略时沿用已有值，缺失时默认 `ONE_TO_MANY` |
| `--biz-relation-type <type>` | 可选 | 所有支持方向均可设置为 `normal` / `main_sub`；省略时沿用当前值 |

## 输出

实际写入返回变更前后状态、写后验证结果和执行信息。`relation-update --dry-run` 返回完整请求体、`before` 和 `warnings`，并在适用时返回 `suggestedExpectFlags`；create/delete 的 dry-run 返回各自的请求预览和定位信息。update/delete 使用 `appCode + relationId` 定位，不解析字符串 key。

正式写入后，CLI 会自动回读关系事实：

- `relation-create`：确认关系已存在，且 `to-label-column` 与请求一致；请求包含 `cardinality` 时一并确认回读值一致；显式传入 `biz-relation-type` 时校验业务关系类型。
- `relation-update`：确认关系仍存在，且 `to-label-column`、合并后的 `cardinality` 和业务关系类型与请求一致。
- `relation-delete`：确认关系已不存在。

如果后端写接口返回但回读不匹配，命令以 `relation_write_not_verified` 失败，不返回成功结果。

关系事实字段主定位字段为 `datasetCode + field`；单条关系写入命令返回结构化执行结果。

```json
{
  "operation": "create",
  "appCode": "app-xxx",
  "selector": {
    "fromDatasetCode": "from_dataset_code",
    "fromColumn": "from_field",
    "toDatasetCode": "to_dataset_code",
    "toColumn": "to_field"
  },
  "before": null,
  "after": {
    "from": { "datasetCode": "from_dataset_code", "column": "from_field" },
    "to": { "datasetCode": "to_dataset_code", "column": "to_field", "labelColumn": "to_label_field" },
    "cardinality": "MANY_TO_ONE"
  },
  "dryRun": false,
  "backend": {
    "method": "POST",
    "path": "/smartapi/question/er-config/cli/erCreate"
  },
  "verification": {
    "verified": true,
    "status": "matched",
    "afterReadback": {
      "from": { "datasetCode": "from_dataset_code", "column": "from_field" },
      "to": { "datasetCode": "to_dataset_code", "column": "to_field", "labelColumn": "to_label_field" },
      "cardinality": "MANY_TO_ONE",
      "condition": null
    }
  },
  "warnings": []
}
```

`before` / `after` / `verification.afterReadback` 表示写入前后可稳定确认的关系事实。`selector` 使用 `fromDatasetCode`、`fromColumn`、`toDatasetCode`、`toColumn` 表达关系两端。

未传 `--expect-to-label-column` 且当前 label 可读取时，METADATA `relation-update` 输出 `warnings[].code = "no_drift_check_applied"` 和 `suggestedExpectFlags`。

`DB_TABLE -> METADATA` 场景下，`--to-label-column` 表示消费 METADATA 数据集时使用的目标展示字段。例如来源字段保存目标 `toField`，页面或运行时展示 `toLabelField`。

## 提示

- 写入前如需读取现状，统一运行 `rabetbase dataset relations --format compress` 或 `--format json`；更新或删除前确认 `relationId`，并确认来源/目标 `datasetCode + field`、`to.labelField`、`relation.cardinality` 和 `relation.bizRelationType`
- 正式写入返回成功表示 CLI 已完成写后回读验证；若返回 `relation_write_not_verified`，先重新执行 `dataset relations` 查看当前事实，不要继续批量写入。
- create 使用 `--from-datasetcode` 和 `--to-datasetcode` 描述关系两端；update/delete 使用 `--appcode + --relation-id` 定位当前关系，update 的其他参数只负责覆盖现值
- 共享自动化可传 `--expect-to-label-column` 做 label 漂移校验
- 写命令只使用结构化定位字段；不要从 dataset 展示名或字符串 key 自行推断关系
- `relation-update` 可覆盖来源/目标 datasetCode、字段、展示字段、关系基数和业务关系类型；未传字段保持当前值
- `to-label-column` 必须来自用户明确指定、目标字段清单，或当前关系事实 `to.labelField`；不确定时不要执行更新
- `relation-create --cardinality` 不区分 DB_TABLE/METADATA，所有支持方向均必填并按显式合法值提交；update 省略时沿用已有值，缺失时默认 `ONE_TO_MANY`
- 暂不支持 `METADATA -> DB_TABLE`
- 修改关系两端字段时必须先检查 `--dry-run` 中的完整合并结果，并确认新方向仍在支持矩阵内
- 不要在自动化流程中把删除旧关系再新增新关系视为原子编辑操作
- 如确需删除旧关系再新增新关系，先用 `dataset relations` 核对当前关系，分别对删除和新增执行 `--dry-run`，并在用户明确确认后分步执行
- 新增、更新、删除单条关系分别使用 `relation-create` / `relation-update` / `relation-delete`

## 参考

- [dataset relations](rabetbase-dataset-relations.md)
- [SKILL.md](../SKILL.md)
