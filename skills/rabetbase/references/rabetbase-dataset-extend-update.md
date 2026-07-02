# dataset extend-update

Dataset 顶层 `extend` 更新命令。

本命令保留为 Dataset 顶层 `extend` 的专用入口，但当前可写 key 白名单为空。它不是字段更新工具，不修改 `fields[]` 或 `properties[]` 中的字段对象，也不作为业务模型分组更新路径；业务模型分组请使用 `dataset business-group-update`。

## 命令

```bash
rabetbase dataset extend-update \
  --appcode app-64e32817 \
  --code 1a90dbff5f094a9a89936fa99b10984c \
  --patch-json '{"someExtendKey":"value"}' \
  --dry-run \
  --format compress
```

当前该命令会因白名单为空拒绝 patch key。若目标是修改业务模型分组，不要使用本命令。

## 参数

| Flag                   | 必填                       | 说明                                                       |
| ---------------------- | -------------------------- | ---------------------------------------------------------- |
| `--appcode <code>`     | 否                         | 目标应用编码；未配置默认 app 时必填                        |
| `--code <code>`        | 是                         | Dataset code，32 位 hex UUID                               |
| `--patch-json <json>`  | 是                         | JSON 对象 patch；当前白名单为空                            |
| `--expect-json <json>` | 否                         | 当前 `extend` 值保护；任一键不匹配即中止且不写入           |
| `--dry-run`            | 否                         | 预览模式；当前仍会因白名单为空被拒绝                       |
| `--format <fmt>`       | 否                         | 输出格式，AI Agent 优先用 `compress`                       |

## 可修改字段

当前白名单为空，没有开放可修改字段。

`businessGroup` 会被明确拒绝，并提示改用 `dataset business-group-update`。

## 输出

当前传入 `businessGroup` 时会返回校验错误，错误语义类似：

```json
{
  "ok": false,
  "error": {
    "message": "dataset extend-update does not support businessGroup; use dataset business-group-update"
  }
}
```

## 操作边界

- 命令只使用 `--code` 定位 Dataset，保持目标清晰且可复现。
- 当前白名单为空；任何 `--patch-json` key 都会被拒绝。
- `businessGroup` 不属于本命令职责。
- 命令不修改字段对象。字段级业务配置继续使用 `rabetbase dataset field-update`。
- 业务模型分组更新请使用 `rabetbase dataset business-group-update`。

## 提示

- 不要用 `extend-update` 修改业务模型分组。
- 修改业务模型分组时，使用 `rabetbase dataset business-group-update`，并先执行 `--dry-run`。
- 不确定 Dataset code 时，先用 `rabetbase dataset list --name <name> --format compress` 或 `rabetbase dataset detail` 相关上下文定位，再回到本命令显式传 `--code`。

## 参考

- [dataset detail](rabetbase-dataset-detail.md)
- [dataset business-group-update](rabetbase-dataset-business-group-update.md)
- [dataset field-update](rabetbase-dataset-field-update.md)
- [dataset list](rabetbase-dataset-list.md)
- [SKILL.md](../SKILL.md)
