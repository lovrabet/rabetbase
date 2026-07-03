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

`--format json` / `--format compress` 返回 CLI envelope，rename 业务结果固定在 `data` 下。Agent 校验 dry-run 或正式执行结果时，只读取 `data.after.name`、`data.request`、`data.submitted` 等内层稳定字段；不要按顶层 `changed` 判断，因为顶层属于框架 envelope。

```json
{
  "ok": true,
  "command": "rabetbase dataset rename",
  "risk": "write",
  "dryRun": true,
  "data": {
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
}
```

dry-run 输出会在同一个 `data` 对象内额外包含 `method`、`url`、`body`、`description`；这些字段只用于展示请求预览，不作为 Agent 的稳定校验入口。

当新旧名称相同时返回 `data.changed=false` 且 `data.submitted=false`，不会调用写接口。

## 连续重命名 SOP

当需要连续修改多个 Dataset 展示名时，逐条执行 `rabetbase dataset rename`，不要自行拼 HTTP 请求。

1. 先用 `rabetbase dataset list --format compress` 拉取线上最新 Dataset 事实；数据量大时用 `--jq` 缩小到 code/name/source 等必要字段。
2. 本地生成 rename plan，至少包含 `code`、`oldName`、`newName`、`reason`，并以线上事实填充 `oldName`。
3. 提交前做目标名唯一性和占用预检。该检查只是 best-effort：必须说明检查范围和过滤条件，不能把本地预检当作平台唯一性约束。
4. 每条先执行 `rabetbase dataset rename --code <code> --name <newName> --expect-name <oldName> --dry-run --format compress`。
5. dry-run 只读取 CLI envelope 内的 `data.after.name`、`data.request`、`data.submitted`、`data.changed` 等稳定字段；不要读取顶层 `changed`，不要依赖 stdout 文本片段。
6. dry-run 通过后，使用同一组 `code/name/expect-name` 去掉 `--dry-run` 正式执行，并逐条记录结果。
7. 全部执行后重新 `rabetbase dataset list`，按 code 回查线上名称，确认每条都与 plan 的 `newName` 一致。
8. 执行脚本包裹 CLI 时，不要假设 `stderr` 一定是 string；要按子进程库的真实类型处理 stdout/stderr/exit code。

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
