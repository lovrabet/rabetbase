# dataset user-deleted-field-list

列出指定 Dataset 中由用户明确删除的字段。只读命令不会修改 Dataset、物理表或关联页面。

## 命令

```bash
rabetbase dataset user-deleted-field-list \
  --code 1a90dbff5f094a9a89936fa99b10984c \
  --format compress
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--appcode <code>` | 否 | 目标应用编码；未配置默认 app 时必填 |
| `--code <code>` | 是 | Dataset code |
| `--format <fmt>` | 否 | 输出格式，AI Agent 优先用 `compress` |

## 输出与语义

- 输出 `operation`、Dataset 摘要、`count` 和排序稳定的 `fields`。
- 每个字段只包含 `id`、`code`、`displayName`、`deleted=true`、`deletionSource=user`。
- 不透传服务端原始 `extend` 或 `_userModifiedKeys`。
- 空列表正常返回 `ok=true` 和 `count=0`。
- 这里只包含用户删除墓碑字段，不包含没有用户墓碑的普通 AI 软删字段。

DB 分析 `SUCCESS` 表示任务按当前规则完成；用户删除墓碑会阻止同名物理列自动重新生成。因此排查“物理列存在但 Dataset 字段缺失”时，先运行本命令。

## 后续恢复

需要恢复时，将返回的精确 `fields[].code` 传给 `field-restore --field`。不要传展示名或字段 ID：

```bash
rabetbase dataset field-restore \
  --code 1a90dbff5f094a9a89936fa99b10984c \
  --field space_type_code \
  --dry-run \
  --format compress
```

## 参考

- [dataset field-restore](rabetbase-dataset-field-restore.md)
- [dataset detail](rabetbase-dataset-detail.md)
- [SKILL.md](../SKILL.md)
