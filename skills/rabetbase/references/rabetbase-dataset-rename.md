# dataset rename

安全修改 Dataset 展示名。命令只更新 Dataset 名称，不修改字段、关系、物理表映射或业务数据。

## 命令

```bash
rabetbase dataset rename \
  --code 1a90dbff5f094a9a89936fa99b10984c \
  --name "发票明细-销项" \
  --expect-name "发票明细" \
  --dry-run
```

确认 dry-run 结果无误后再去掉 `--dry-run` 执行：

```bash
rabetbase dataset rename \
  --code 1a90dbff5f094a9a89936fa99b10984c \
  --name "发票明细-销项" \
  --expect-name "发票明细" \
  --format json
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--appcode <code>` | 否 | 目标应用编码；未配置默认 app 时必填 |
| `--code <code>` | 是 | Dataset code，32 位 hex |
| `--name <newName>` | 是 | 新展示名 |
| `--expect-name <oldName>` | 是 | 当前展示名保护；不匹配即中止且不写入 |
| `--dry-run` | 否 | 只返回 before/after 与提交预览，不调用写接口 |
| `--format <fmt>` | 否 | 输出格式，AI Agent 优先用 `compress` |

## 输出

返回结构包含：

```json
{
  "protocol": "dataset-rename.v1",
  "dataset": {
    "id": 1010859,
    "code": "1a90dbff5f094a9a89936fa99b10984c"
  },
  "dryRun": true,
  "changed": true,
  "before": { "name": "发票明细" },
  "after": { "name": "发票明细-销项" },
  "request": {
    "modelId": 1010859,
    "modelCode": "1a90dbff5f094a9a89936fa99b10984c",
    "name": "发票明细-销项",
    "dataset": {
      "datasetId": 1010859,
      "datasetCode": "1a90dbff5f094a9a89936fa99b10984c",
      "datasetName": "发票明细-销项"
    }
  },
  "submitted": false
}
```

当新旧名称相同时返回 `changed=false`，不会调用写接口。

## 操作边界

- 命令先读取平台 `get-driven-data`，用返回的 Dataset id/code 构造最小重命名 payload。
- 写入时通过 `/smartapi/dataset/update-driven-data` 提交，仅携带 `modelId`、`modelCode`、顶层 `name` 与 `dataset.datasetName`。
- 命令不提交 `fields` / `properties`，避免用 CLI 本地快照覆盖字段元数据。
- `--expect-name` 是强制防漂移保护，目标 Dataset 当前名称不匹配时不写入。

## 验证建议

执行后用以下命令确认：

```bash
rabetbase dataset detail --code <datasetCode> --format compress --jq '.data | {id, code, name}'
rabetbase dataset list --code <datasetCode> --format compress --jq '.data.datasets[] | {id, code, name}'
```

## 参考

- [dataset detail](rabetbase-dataset-detail.md)
- [dataset list](rabetbase-dataset-list.md)
- [SKILL.md](../SKILL.md)
