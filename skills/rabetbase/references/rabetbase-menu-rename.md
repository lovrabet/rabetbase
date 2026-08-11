# menu rename

按精确菜单 ID 修改菜单名称。该能力适用于所有菜单类型，不根据页面类型、打开方式或是否为分组选择不同命令。

## 命令

```bash
rabetbase menu rename --id "<menu-id>" --label "新名称" --expect-label "原名称" --dry-run --format compress
rabetbase menu rename --id "<menu-id>" --label "新名称" --expect-label "原名称" --format compress
```

## Flags

| Flag | 类型 | 说明 |
|---|---|---|
| `--id <id>` | number | 必填，`menu list` 返回的精确菜单 ID |
| `--label <label>` | string | 必填，新菜单名称；首尾空白会被移除，不能是空字符串 |
| `--expect-label <label>` | string | 必填，断言当前菜单名称，防止覆盖他人的并发修改 |
| `--dry-run` | boolean | 输出完整 before/after 和写入计划，不修改线上菜单 |

## 执行顺序

1. 先运行 `rabetbase menu list --format compress`，确认目标菜单的 `id` 和当前 `label`。
2. 使用当前名称作为 `--expect-label` 执行 dry-run。
3. 确认 `before/after` 仅有 `label` 发生预期变化。
4. 复用相同参数并移除 `--dry-run`，执行正式写入。
5. 检查结果中的 `verification.matchesRequestedState=true`。

新名称与当前名称相同时返回成功 no-op，不发送写请求。即使是 no-op，当前名称仍必须匹配 `--expect-label`。

## 安全规则

- 不按 `folder`、`iframe`、`link`、`procode` 或其他类型分支。
- 只修改 `label`，保留菜单 ID、父级、排序、path、URL、页面绑定、资源、可见性和权限关联。
- dry-run 后线上菜单事实发生变化时，正式执行会拒绝写入，需重新获取菜单事实并预览。
- 响应丢失时不自动重试写入。CLI 回读确认新名称与其他菜单事实后才返回 `recovered=true`；无法确认时按返回的 `verification.lookupCommand` 检查线上状态。

## 常见错误

- `menu_state_drift`：当前名称或菜单快照已变化。重新执行 `menu list` 和 dry-run，并更新 `--expect-label`。
- `menu_update_recovery_required`：写入结果无法安全确认。先执行错误结果中的只读 lookup 命令，不要直接重试。

## 相关命令

- [menu list](rabetbase-menu-list.md)
- [menu group-update](rabetbase-menu-group-update.md)
- [menu external-link-update](rabetbase-menu-external-link-update.md)
- [menu asset-update](rabetbase-menu-asset-update.md)
