# menu visibility-update

按精确菜单 ID 或 path 批量显示、隐藏线上菜单。该命令只更新 `visible`，不删除页面，也不修改 path、URL、父级、排序或资源。

## 隐藏菜单

```bash
rabetbase menu list --format compress
rabetbase menu visibility-update --menu-ids "10,11" --visible=false --expect-visible=true --expected-count 2 --dry-run --format compress
rabetbase menu visibility-update --menu-ids "10,11" --visible=false --expect-visible=true --expected-count 2 --yes --format compress
rabetbase menu list --format json --jq '.data.menus[] | select(.id == 10 or .id == 11) | {id, path, visible}'
```

## 恢复显示

```bash
rabetbase menu visibility-update --menu-ids "10,11" --visible=true --expect-visible=false --expected-count 2 --dry-run --format compress
rabetbase menu visibility-update --menu-ids "10,11" --visible=true --expect-visible=false --expected-count 2 --yes --format compress
```

## Flags

| Flag | 类型 | 说明 |
|---|---|---|
| `--menu-ids <csv>` | string | 精确菜单 ID；只接受正安全整数，可与 paths 组合并去重 |
| `--paths <csv>` | string | 精确菜单 path；同一路径命中多条时必须改用 ID |
| `--visible <true\|false>` | boolean | 必填，目标可见性；显式 `false` 不会被忽略 |
| `--expect-visible <true\|false>` | boolean | 可选，断言所有目标当前可见性，防止覆盖并发变化 |
| `--expected-count <n>` | number | 可选，断言去重后的精确目标数量 |
| `--dry-run` | boolean | 输出目标、before/after、snapshotHash 和最小请求，不写入 |
| `--yes` | boolean | 正式执行必填；该命令是 `high-risk-write` |

`--menu-ids` 与 `--paths` 至少提供一个。任一 selector 未命中会整批失败，不会静默更新子集；ID 与 path 选中同一菜单时自动去重。

## 执行检查

1. 先用 `menu list` 取得精确 ID、path 和当前 `visible`。
2. dry-run 时同时传 `--expect-visible` 与 `--expected-count`。
3. 检查 `targets[].before/after`，确认只有 `visible` 改变。
4. 复用相同 selector、目标值和断言，添加 `--yes` 正式执行。
5. 检查 `succeeded/failed` 与 `verification.matchesRequestedState`，再用 `menu list` 回查。

目标已经等于期望值时返回 `skipped=true`，不发送更新请求。批量请求出现部分失败时不会自动重试；先按 `succeeded/failed` 和 `menu list` 核对实际状态，再为仍未生效的精确 ID 重新 dry-run。

## 常见错误

- `menu_selector_not_found`：ID 或 path 不存在；重新执行 `menu list` 获取当前事实。
- `menu_selector_ambiguous`：一个 path 命中多条菜单；按错误提示改用 `--menu-ids`。
- `menu_expected_count_mismatch`：去重后的目标数量与断言不一致；重新检查 selector。
- `menu_state_drift`：当前可见性或菜单快照发生变化；重新读取并 dry-run。
- `menu_verification_failed`：写后 visible 或未修改字段无法确认；先执行 `menu list`，不要直接重试整批。

隐藏菜单只影响菜单展示，不等于禁用页面或修改权限。页面能否通过直达地址访问，仍由平台现有权限事实决定。

## 相关命令

- [menu list](rabetbase-menu-list.md)
- [menu move](rabetbase-menu-move.md)
- [menu asset-update](rabetbase-menu-asset-update.md)
