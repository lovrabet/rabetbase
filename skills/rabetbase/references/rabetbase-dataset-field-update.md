# dataset field-update

安全更新 Dataset 原始 `fields[]` 中的单个 field 对象。

本命令用于修正字段的业务配置元数据，例如把负责人字段的 `doType` 从 `TEXT` 改为 `USER`，或补齐 `SELECT` 字段的 `options`。它不是物理字段迁移工具，不修改字段身份、数据库类型或系统生命周期状态。

## 命令

```bash
rabetbase dataset field-update \
  --appcode app-64e32817 \
  --code 1a90dbff5f094a9a89936fa99b10984c \
  --field assignee_id \
  --patch-json '{"doType":"USER"}' \
  --expect-json '{"doType":"TEXT"}' \
  --dry-run \
  --format compress
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--appcode <code>` | 否 | 目标应用编码；未配置默认 app 时必填 |
| `--code <code>` | 是 | Dataset code，32 位 hex UUID |
| `--field <name>` | 是 | 精确匹配原始 `fields[].name` |
| `--patch-json <json>` | 是 | JSON 对象 patch；对象深度合并，数组/标量整体替换 |
| `--expect-json <json>` | 否 | 当前值保护；任一键不匹配即中止且不写入 |
| `--dry-run` | 否 | 只返回 before/after、changedPaths 和提交预览，不调用写接口 |
| `--format <fmt>` | 否 | 输出格式，AI Agent 优先用 `compress` |

## 可修改字段

第一版只允许 patch 已知可变业务配置属性：

- `doType`
- `displayName`
- `description`
- `options`
- `required`
- `datetimeFormat`
- `defaultAggregation`
- `chartRole`

## 禁止修改字段

`--patch-json` 禁止包含以下字段：

- 字段身份：`id`、`name`、`code`
- 物理库映射：`dbType`、`dbTypeLen`、`dbTypeScale`、`dbFieldName`、`tableName`
- 系统或生命周期属性：`deleted`、`deprecated`、`pkField`、`autoIncrement`、`systemRetain`

禁止规则只约束写入 patch。`--expect-json` 可以包含这些字段，用于确认当前 field 仍是预期对象。

## 输出

返回结构包含：

```json
{
  "protocol": "dataset-field-update.v1",
  "appCode": "app-64e32817",
  "selector": {
    "code": "1a90dbff5f094a9a89936fa99b10984c"
  },
  "dataset": {
    "id": 1010859,
    "code": "1a90dbff5f094a9a89936fa99b10984c",
    "name": "需求",
    "table": "requirements"
  },
  "field": {
    "id": 832007,
    "name": "assignee_id"
  },
  "dryRun": true,
  "changed": true,
  "changedPaths": ["doType"],
  "before": {
    "id": 832007,
    "name": "assignee_id",
    "displayName": "负责人",
    "doType": "TEXT"
  },
  "after": {
    "id": 832007,
    "name": "assignee_id",
    "displayName": "负责人",
    "doType": "USER"
  },
  "submitted": false
}
```

当 patch 后无变化时，返回 `changed=false`，不会调用写接口。

## 操作边界

- 命令读取平台 `get-driven-data` 原始结构，按 `fields[].name` 精确定位字段。
- 命令只使用 `--code` 定位 Dataset，保持 AI First 协议标识清晰且可复现。
- 非 `--dry-run` 且确有变化时，命令通过 `/smartapi/dataset/update-driven-data` 提交完整 driven data。
- V2 后端保存链路会触发关联页面更新；CLI 不额外调用 `rabetbase page sync`。
- 页面仍未生效时，再按页面状态单独处理 `page sync`、页面保存或发布。

## 提示

- 写入前必须先运行 `--dry-run`。
- 推荐用 `--expect-json` 保护当前值，例如 `--expect-json '{"doType":"TEXT"}'`。
- 不要修改 `dataset detail` 归一化输出里的 `type`；本命令 patch 的是平台原始 field 对象上的 `doType`。
- 不要用本命令修改字段名、数据库类型、主键、自增、删除状态等结构性或系统性属性。
- 不确定 Dataset code 时，可先用 `rabetbase dataset list --name <name> --format compress` 定位，再回到本命令显式传 `--code`。

## 参考

- [dataset detail](rabetbase-dataset-detail.md)
- [dataset list](rabetbase-dataset-list.md)
- [SKILL.md](../SKILL.md)
