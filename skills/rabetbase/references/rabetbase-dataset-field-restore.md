# dataset field-restore

恢复一个由用户明确删除的 Dataset 字段。命令只恢复 Dataset 字段元数据，不恢复或校验物理列，也不自动同步、保存或发布关联页面。

## 安全流程

```bash
rabetbase dataset user-deleted-field-list \
  --code 1a90dbff5f094a9a89936fa99b10984c \
  --format compress

rabetbase dataset field-restore \
  --code 1a90dbff5f094a9a89936fa99b10984c \
  --field space_type_code \
  --dry-run \
  --format compress

rabetbase dataset field-restore \
  --code 1a90dbff5f094a9a89936fa99b10984c \
  --field space_type_code \
  --format compress
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--appcode <code>` | 否 | 目标应用编码；未配置默认 app 时必填 |
| `--code <code>` | 是 | Dataset code |
| `--field <code>` | 是 | 用户删除列表中的精确字段 code |
| `--dry-run` | 否 | 只预览唯一字段和写请求，不调用恢复接口 |
| `--format <fmt>` | 否 | 输出格式，AI Agent 优先用 `compress` |

`--field` 对应 `dataset detail` 归一化输出的 `data.fields[].name`，不是 `displayName`。命令不支持逗号批量、模糊匹配或交互选择；传入 `--all`、`--fields`、`--confirm` 或 `--yes` 会被明确拒绝。

## 完成口径

正式执行只提交一个 column ID。以下条件全部满足才返回 `ok=true`：

1. 服务端实际恢复数量等于 `1`。
2. 写后用户删除字段列表中已不存在目标 column ID。
3. 写后 Dataset 活动字段集合中存在相同 ID 和 code。

dry-run 输出包含 `operation`、`selector`、`dataset`、`before/after`、`dryRun`、`backend`、`verification`、`warnings`，并以 `body` 精确展示 `{ datasetId, columnIds: [id] }`。正式执行的 `after` 来自真实回读，找不到活动字段时为 `null`。服务端返回 `0`、字段仍在删除列表或活动字段未出现时返回结构化失败，Agent 不得自动重试写请求。

## 失败路径

- 找不到精确 field code：先重新运行 `user-deleted-field-list`，不要改用展示名猜测。
- 同一 field code 出现多条：视为服务端元数据歧义，命令在写入前中止。
- 写请求传输结果不确定：CLI 不会再次 POST，而会尝试双回读并返回 `ok=false`；只重新查询列表和详情，不要直接重试。
- 写后回读失败：保留已提交请求的 backend 上下文并返回 `ok=false`；只重新查询列表和详情，不要直接重试。
- 恢复成功但页面仍无字段：这是预期边界。单独评估页面状态，确有需要时显式执行 `rabetbase page sync`；字段恢复命令本身不调用页面同步。

## 参考

- [dataset user-deleted-field-list](rabetbase-dataset-user-deleted-field-list.md)
- [dataset detail](rabetbase-dataset-detail.md)
- [page sync](rabetbase-page-sync.md)
- [SKILL.md](../SKILL.md)
