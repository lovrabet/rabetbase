# menu group-create

创建一个空菜单分组。group 是命令领域名称，线上持久化类型为 `folder`。

> **风险等级：high-risk-write** — 必须先 dry-run，正式执行必须显式 `--yes`。

## 命令

```bash
rabetbase menu group-create --label "用户管理" --parent-id "<parent-folder-id>" --sort 10 --dry-run --format compress
rabetbase menu group-create --label "用户管理" --parent-id "<parent-folder-id>" --sort 10 --yes --format compress
```

## Flags

| Flag | 类型 | 必填 | 说明 |
|---|---|---|---|
| `--label <label>` | string | 是 | folder 名称 |
| `--parent-id <id\|root>` | string | 否 | 父 folder；默认 root |
| `--sort <n>` | number | 否 | 非负整数；默认目标父级同级末尾 |
| `--dry-run` | boolean | 预览时 | 返回创建计划，不写入 |
| `--yes` | boolean | 正式执行时 | 显式授权创建 |

## 事实约束

- 指定父级时，父级必须存在且 `type=folder`。
- 创建结果必须为 `type=folder`，并保留平台生成的非空 `path`。
- `resources` 必须为空且 `childrenCount=0`；命令通过 `menu list` 回查这些最终事实。
- dry-run 只展示一次 folder 创建请求。
- 创建请求成功但后续回查失败时，错误输出会返回已创建的菜单 ID 和 path。该分组可能仍在线上；先用 `menu list` 按 ID 核对，不要直接重试创建。

## 相关命令

- [menu group-update](rabetbase-menu-group-update.md)
- [menu regroup-start](rabetbase-menu-regroup-start.md)
- [menu list](rabetbase-menu-list.md)
