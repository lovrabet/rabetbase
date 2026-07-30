# menu group-update

按 ID 修改现有 folder 的名称或 sort，不调整父级。

> **风险等级：high-risk-write** — 必须先 dry-run，正式执行必须显式 `--yes`。

## 命令

```bash
rabetbase menu group-update --id <folderId> --label "客户管理" --sort 20 --dry-run --format compress
rabetbase menu group-update --id <folderId> --label "客户管理" --sort 20 --yes --format compress
```

至少提供 `--label`、`--sort` 之一。

## Flags

| Flag | 类型 | 说明 |
|---|---|---|
| `--id <id>` | number | 精确 folder ID |
| `--label <label>` | string | 新名称，不能是空字符串 |
| `--sort <n>` | number | 新 sort，非负整数 |
| `--expect-parent-id <id\|root>` | string | 断言当前父级 |
| `--expect-path <path>` | string | 断言当前 path |
| `--expected-children-count <n>` | number | 断言直接子节点数 |
| `--dry-run` | boolean | 输出 before/after，不写入 |
| `--yes` | boolean | 正式执行必填 |

## 安全规则

- 目标必须为 `type=folder`。
- 命令只修改 label 和 sort，不修改 parentId 或 path。
- expect 条件漂移时不写入。
- 写入后自动 menu list 回查 label 和 sort。

## 相关命令

- [menu group-create](rabetbase-menu-group-create.md)
- [menu regroup-start](rabetbase-menu-regroup-start.md)
- [menu list](rabetbase-menu-list.md)
