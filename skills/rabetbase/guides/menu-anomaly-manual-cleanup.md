# 菜单异常审计与安全修复

用于重复 `path`、根级菜单重名、疑似菜单同步重复创建等场景。默认通过 `menu list` 审计；CLI 只按精确 ID 生成可审阅计划，页面菜单或目录树必须完整暴露影响后再执行。

## 执行原则

- 菜单事实只从 `rabetbase menu list --verbose` 获取。
- 不绕过 `rabetbase menu list --verbose` 读取菜单配置。
- 删除只能使用精确菜单 ID，不得根据 label/path 自动猜测目标。
- 单删允许 `resources=[]` 的空 folder 或任意非 folder 叶子菜单；非空 folder 必须显式 `--recursive`。
- 页面菜单删除会触发平台既有 Page/PageSchema 级联；同一 pageId 被计划外菜单引用时必须阻断。
- 删除先用 `--id` 和 `--plan-out` 生成计划，确认后只使用 `--plan <file> --yes` 执行；层级调整仍交由平台处理。

## 平台入口

先按当前 `rabetbase` 环境计算 `<appBaseUrl>`：

```text
production https://app.lovrabet.com/app/<appCode>
daily      https://daily.lovrabet.com/web-app/app/<appCode>
```

总入口：

```text
<appBaseUrl>/pages
```

精细入口：

```text
<appBaseUrl>/pages/<menuCode>
```

当菜单 `path` 可作为页面标识使用时，可生成：

```text
<appBaseUrl>/pages/<encodeURIComponent(menu.path)>
```

重复 `path` 场景下，多条异常菜单的精细入口可能相同；这只能帮助用户快速进入页面配置，不能替代按 `id / parentId / label` 做删除确认。

## 审计步骤

1. 读取菜单事实：

```bash
rabetbase menu list --appcode <appCode> --verbose --format json
```

`menu list` 返回 DFS 扁平数组，并直接提供 `childrenIds`、`childrenCount`、`pageId` 和 `pageType`。

2. 按规则分类异常：

必须修复：
- 非空 `path` 重复。
- `parentId` 指向不存在的菜单。
- 菜单树出现循环引用。
- 同一父级下重复关键业务菜单。

需要人工确认：
- 根级 `label` 重复。
- `visible=false` 但仍有 `path` 或 `resources`。
- 多个菜单共享同一 `pageId`。
- `procode` 菜单缺少 resources，但同组重复菜单存在 resources。

不默认视为异常：
- 空 `path` 的 folder。
- 不同父级下的同名 `label`。
- folder 没有 resources。

3. 输出修复清单：

```text
平台总入口: <appBaseUrl>/pages
精细入口: <appBaseUrl>/pages/<menuCode>

异常类型: duplicate_path
path: <path>
建议保留:
- id: <id>, label: <label>, parentId: <parentId>, resources: <count>
建议删除:
- id: <id>, label: <label>, type: <type>, parentId: <parentId>, childrenCount: <count>, pageId: <pageId|null>, resources: <count>
停止条件:
- 平台无法确认该 id
- 目标不是空 folder、非 folder 叶子或已完整计划的 folder 子树
- pageId 被计划外菜单引用
- folder resources 不为空
- 平台显示与清单不一致
```

4. 对确认后的精确目标先预演。空 folder 或非 folder 叶子使用单目标计划：

```bash
rabetbase menu delete \
  --appcode <appCode> \
  --id <menuId> \
  --expect-parent-id <id|root> \
  --expect-path '<path>' \
  --expected-children-count 0 \
  --dry-run \
  --plan-out ./menu-delete-<menuId>.plan.json \
  --format compress
```

非空 folder 使用递归计划，并完整审阅 `targets`、`deleteOrder`、`cascade.pages` 和 `warnings`：

```bash
rabetbase menu delete \
  --appcode <appCode> \
  --id <folderId> \
  --recursive \
  --dry-run \
  --plan-out ./menu-tree-delete-<folderId>.plan.json \
  --format compress
```

确认 dry-run 的 `before`、`snapshotHash`、完整目标集合和叶子优先顺序后，在 30 分钟内执行：

```bash
rabetbase menu delete \
  --plan <reviewed-plan-file> \
  --yes \
  --format compress
```

计划绑定 `appCode`、菜单快照、完整目标事实和删除顺序；任一事实漂移、计划缺项、时间字段异常或计划过期都应重新生成计划。发生部分失败时以 `remainingIds` 为事实重新生成计划，不复用旧计划。需要管理 folder 的 label 或 sort 时使用 `group-update`；调整归属仍转平台处理，并在完成后用 `menu list` 回查。

5. 再读取菜单事实并复查：

```bash
rabetbase menu list --appcode <appCode> --verbose --format json
```

复查至少确认重复 `path` 清零、目标保留菜单仍存在、resources 符合预期。

CLI 返回 blocker 时不要绕过；把清单交给用户核实计划外引用或平台事实，处理后仍用 `menu list` 回查。
