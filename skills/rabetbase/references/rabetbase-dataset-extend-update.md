# dataset extend-update

安全更新 Dataset 顶层 `extend` 对象。

本命令用于修正表元数据级别的业务配置，例如修改 `yt_table_metadata.extend.businessGroup`。它不是字段更新工具，不修改 `fields[]` 或 `properties[]` 中的字段对象。

## 命令

```bash
rabetbase dataset extend-update \
  --appcode app-64e32817 \
  --code 1a90dbff5f094a9a89936fa99b10984c \
  --patch-json '{"businessGroup":"mt_2"}' \
  --expect-json '{"businessGroup":"mt_1"}' \
  --dry-run \
  --format compress
```

确认 dry-run 结果无误后再去掉 `--dry-run` 执行。

## 参数

| Flag                   | 必填                       | 说明                                                       |
| ---------------------- | -------------------------- | ---------------------------------------------------------- |
| `--appcode <code>`     | 否                         | 目标应用编码；未配置默认 app 时必填                        |
| `--code <code>`        | 是                         | Dataset code，32 位 hex UUID                               |
| `--patch-json <json>`  | 是                         | JSON 对象 patch；第一版只允许 `businessGroup`              |
| `--expect-json <json>` | 否                         | 当前 `extend` 值保护；任一键不匹配即中止且不写入           |
| `--dry-run`            | 否                         | 只返回 before/after、changedPaths 和提交预览，不调用写接口 |
| `--format <fmt>`       | 否                         | 输出格式，AI Agent 优先用 `compress`                       |

## 可修改字段

第一版只允许 patch 已知可变业务配置属性：

- `businessGroup`

## 输出

返回结构包含：

```json
{
  "protocol": "dataset-extend-update.v1",
  "appCode": "app-64e32817",
  "selector": {
    "code": "1a90dbff5f094a9a89936fa99b10984c"
  },
  "dataset": {
    "id": 1010859,
    "code": "1a90dbff5f094a9a89936fa99b10984c",
    "name": "保单条款",
    "table": "mt_policy_term",
    "db": 10293
  },
  "dryRun": true,
  "changed": true,
  "changedPaths": ["extend.businessGroup"],
  "before": {
    "extend": {
      "businessGroup": "mt_1",
      "tableLabel": "保单条款"
    }
  },
  "after": {
    "extend": {
      "businessGroup": "mt_2",
      "tableLabel": "保单条款"
    }
  },
  "submitted": false
}
```

当 patch 后无变化时，返回 `changed=false`，不会调用写接口。

## 操作边界

- 命令读取平台 `get-driven-data` 原始结构，解析顶层 `detail.extend`。
- 命令只使用 `--code` 定位 Dataset，保持 AI First 协议标识清晰且可复现。
- 非 `--dry-run` 且确有变化时，命令通过 `/smartapi/dataset/update-driven-data` 提交完整 driven data。
- 若平台返回的 `extend` 是 JSON 字符串，写回时保持字符串形态；若是对象，写回时保持对象形态。
- 命令不修改字段对象。字段级业务配置继续使用 `rabetbase dataset field-update`。

## 提示

- 写入前必须先运行 `--dry-run`。
- 推荐用 `--expect-json` 保护当前值，例如 `--expect-json '{"businessGroup":"mt_1"}'`。
- 不确定 Dataset code 时，先用 `rabetbase dataset list --name <name> --format compress` 或 `rabetbase dataset detail` 相关上下文定位，再回到本命令显式传 `--code`。

## 参考

- [dataset detail](rabetbase-dataset-detail.md)
- [dataset field-update](rabetbase-dataset-field-update.md)
- [dataset list](rabetbase-dataset-list.md)
- [SKILL.md](../SKILL.md)
