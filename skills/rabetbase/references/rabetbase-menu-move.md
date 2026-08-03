# rabetbase menu move

## Use when

把当前 App 中一个已有的非 folder 叶子菜单移动到已有 folder 或根级，或者在当前父级内调整该叶子的 sort。

> **风险等级：high-risk-write** — 必须先 dry-run，正式执行必须显式 `--yes`。

## Read before writing

```bash
rabetbase menu list --format compress
```

从返回事实中读取源菜单 `id`、当前 `parentId`、`type`、`childrenCount`、`sort`，以及目标 folder 的 `id` 和 `type`。源必须是 `childrenCount=0` 的非 folder 菜单，数字形式的目标父级必须是 folder；根级使用 `root`。

## Preview

```bash
rabetbase menu move --id "<menu-id>" --parent-id "<folder-id|root>" --expect-parent-id "<current-folder-id|root>" --dry-run --format compress
```

检查 `data.before.parentId`、`data.after.parentId`、`data.after.sort` 和 `data.plannedActions[].body`。跨父级且未传 `--sort` 时默认追加到目标同级末尾；同一父级未传 `--sort` 时返回 no-op。

## Execute

```bash
rabetbase menu move --id "<menu-id>" --parent-id "<folder-id|root>" --expect-parent-id "<current-folder-id|root>" --yes --format compress
```

每次只执行一个源菜单。前一项只有在 `data.verification.matchesRequestedState=true` 后才处理下一项；首个失败立即停止。

## Stable result fields

- `data.operation="move"`
- `data.selector.id`、`data.selector.parentId`
- `data.before.parentId`、`data.before.sort`
- `data.after.parentId`、`data.after.sort`
- `data.succeeded[].updated`、`data.succeeded[].skipped`
- `data.verification.matchesRequestedState`
- `data.snapshotHash` 仅用于观察和排障，不作为再次执行的输入

## Failure handling

- `menu_state_drift`：当前父级与 `--expect-parent-id` 不一致；重新读取菜单并重新 dry-run。
- `menu_move_source_not_leaf`：源不是非 folder 叶子；选择正确的源 ID。
- `menu_parent_not_folder`：数字目标不是 folder；重新读取目标事实。
- `menu_verification_failed`：写请求可能已经生效，但最终事实未确认；先执行 `menu list`，不要直接重试写入。
- 其他写入失败同样先读取最新菜单事实，再决定是否生成新的 dry-run。

## Boundaries

- 命令只修改源菜单的 `parentId` 和 `sort`，不修改 label、path、url、visible、resources 或 extend。
- `--expect-parent-id` 是 CLI 写前检查，不是服务端原子 compare-and-set。
- 其他菜单的名称、资源或审计字段变化不阻断当前移动。
- 新建目录并归组继续使用 `menu regroup-start`；修改 folder 名称或 sort 使用 `menu group-update`。
