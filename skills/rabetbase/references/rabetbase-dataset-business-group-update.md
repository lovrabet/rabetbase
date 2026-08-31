# dataset business-group-update

安全更新业务场景分组（`businessGroup`）。

使用 Dataset code 修改业务场景分组。DB_TABLE 与 METADATA 可设置非空分组，METADATA 还可清空分组。

## 命令

```bash
rabetbase dataset business-group-update \
  --appcode app-64e32817 \
  --code 1a90dbff5f094a9a89936fa99b10984c \
  --business-group "数据智能---AI任务与平台配置" \
  --expect-business-group "旧分组" \
  --dry-run \
  --format compress
```

确认 dry-run 结果无误后再去掉 `--dry-run` 执行。

清空 METADATA Dataset 的当前业务场景分组：

```bash
rabetbase dataset business-group-update \
  --code 1a90dbff5f094a9a89936fa99b10984c \
  --business-group ungrouped \
  --expect-business-group "旧分组" \
  --dry-run \
  --format compress
```

## 参数

| Flag | 必填 | 说明 |
| --- | --- | --- |
| `--appcode <code>` | 否 | 目标应用编码；未配置默认 app 时必填 |
| `--code <code>` | 是 | Dataset code，32 位 hex UUID |
| `--business-group <name>` | 是 | 目标业务场景分组；METADATA 可用保留值 `ungrouped` 清空，DB_TABLE 不支持清空 |
| `--expect-business-group <name>` | 否 | 当前业务场景分组保护；使用 `ungrouped` 表示期望当前未分组；不匹配即中止且不写入 |
| `--dry-run` | 否 | 只返回 before/after 预览，不执行写入 |
| `--format <fmt>` | 否 | 输出格式，AI Agent 优先用 `compress` |

## 行为

- 命令用 `--code` 定位 Dataset。
- DB_TABLE 与 METADATA 都可设置非空业务场景分组。
- METADATA 使用 `--business-group ungrouped` 清空分组；DB_TABLE 使用该值时会失败且不写入。
- 当前分组为空、`null` 或字段缺失时统一显示为 `ungrouped`。
- `--expect-business-group` 用于保护当前值；当前未分组时传 `ungrouped`，不匹配时中止且不写入。
- `--dry-run` 只预览 before/after，不执行写入。
- 正式写入后会校验目标值已生效。
- 当目标值和当前值一致时，命令返回 `changed=false`，不会执行写入。

## 输出

返回结构包含：

```json
{
  "operation": "update",
  "appCode": "app-64e32817",
  "selector": {
    "code": "1a90dbff5f094a9a89936fa99b10984c"
  },
  "dataset": {
    "code": "1a90dbff5f094a9a89936fa99b10984c",
    "name": "保单条款"
  },
  "before": {
    "businessGroup": "旧分组"
  },
  "after": {
    "businessGroup": "数据智能---AI任务与平台配置"
  },
  "changed": true,
  "dryRun": true,
  "submitted": false
}
```

## 提示

- 写入前必须先运行 `--dry-run`。
- `businessGroup` 用于按业务场景组织 Dataset，而不是按数据库、表类型或服务名等技术结构分组。优先复用当前应用已有分组；需要跨领域消歧时可使用 `<业务领域>---<业务场景>`，否则使用 `<业务场景>`。建议最多两级，层级不得为空或带首尾空白。
- `ungrouped` 表示未分组；可用于 METADATA 清空和任意 Dataset 的当前值保护，不作为真实分组名称。
- 推荐用 `--expect-business-group` 保护当前值。
- 不确定 Dataset code 时，先用 `rabetbase dataset list --name <name> --format compress` 定位。
- 业务场景分组只使用本命令更新。

## 参考

- [dataset detail](rabetbase-dataset-detail.md)
- [dataset list](rabetbase-dataset-list.md)
- [dataset extend-update](rabetbase-dataset-extend-update.md)
- [SKILL.md](../SKILL.md)
