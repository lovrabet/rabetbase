# dataset business-group-update

安全更新业务模型分组（`businessGroup`）。

本命令用于修改业务模型分组。它不同于 `dataset extend-update`：`extend-update` 当前可写 key 白名单为空，且不支持修改 `businessGroup`。

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

## 参数

| Flag | 必填 | 说明 |
| --- | --- | --- |
| `--appcode <code>` | 否 | 目标应用编码；未配置默认 app 时必填 |
| `--code <code>` | 是 | Dataset code，32 位 hex UUID |
| `--business-group <name>` | 是 | 目标业务模型分组；空字符串表示清空 |
| `--expect-business-group <name>` | 否 | 当前业务模型分组保护；不匹配即中止且不写入 |
| `--dry-run` | 否 | 只返回 before/after 预览，不执行写入 |
| `--format <fmt>` | 否 | 输出格式，AI Agent 优先用 `compress` |

## 行为

- 命令用 `--code` 定位 Dataset。
- 当前值和写后校验都按 Dataset code 读取业务模型分组；不需要 `dblinkId`、`tableName` 或 `sourceType`。
- `--business-group` 为空字符串时表示清空业务模型分组。
- 回读到 `businessGroup` 为空字符串时表示未分组，是合法当前值。
- `--expect-business-group` 用于保护当前值；不匹配时中止且不写入。
- `--dry-run` 只预览 before/after，不执行写入。
- 正式写入后会校验目标值已生效。
- 当目标值和当前值一致时，命令返回 `changed=false`，不会执行写入。

## 输出

返回结构包含：

```json
{
  "protocol": "dataset-business-group-update.v1",
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
- 推荐用 `--expect-business-group` 保护当前值。
- 不确定 Dataset code 时，先用 `rabetbase dataset list --name <name> --format compress` 定位。
- 业务模型分组只能使用本命令更新，不要通过 `dataset extend-update` 或 `/smartapi/dataset/update-driven-data` 修改。
- `dataset extend-update` 当前可写 key 白名单为空；未来如开放其他 Dataset 顶层 `extend` 字段，以 `dataset extend-update` reference 为准。

## 参考

- [dataset detail](rabetbase-dataset-detail.md)
- [dataset list](rabetbase-dataset-list.md)
- [dataset extend-update](rabetbase-dataset-extend-update.md)
- [SKILL.md](../SKILL.md)
