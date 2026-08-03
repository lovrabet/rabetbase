# menu delete

按线上菜单 ID 精确删除空 folder、非 folder 叶子菜单，或一个 folder 子树。

> **风险等级：high-risk-write** — 必须先生成删除计划，正式执行必须消费该计划并显式 `--yes`。

## 命令

```bash
rabetbase menu delete --id "<folder-id>" --expected-children-count 0 --dry-run --plan-out ./menu-delete.plan.json --format compress
rabetbase menu delete --plan ./menu-delete.plan.json --yes --format compress

rabetbase menu delete --id "<folder-id>" --recursive --dry-run --plan-out ./menu-tree-delete.plan.json --format compress
rabetbase menu delete --plan ./menu-tree-delete.plan.json --yes --format compress
```

## Flags

| Flag | 类型 | 必填 | 说明 |
|---|---|---|---|
| `--id <id>` | number | 计划阶段 | 精确菜单 ID；不接受 label/path 定位 |
| `--recursive` | boolean | 否 | 计划阶段包含精确 folder ID 下的全部后代，叶子优先 |
| `--expect-parent-id <id\|root>` | string | 否 | 计划阶段断言当前父级 |
| `--expect-path <path>` | string | 否 | 计划阶段断言当前 path |
| `--expected-children-count <n>` | number | 否 | 计划阶段断言直接子节点数量，删除 folder 时推荐为 `0` |
| `--plan-out <file>` | string | 计划阶段 | 保存可审阅的删除计划 |
| `--dry-run` | boolean | 计划阶段 | 返回目标快照和影响计划，不写入 |
| `--plan <file>` | string | 正式执行 | 读取 30 分钟内生成的删除计划 |
| `--yes` | boolean | 正式执行 | 显式授权删除 |

## 安全规则

- 单目标必须是 `resources=[]` 的空 folder，或任意非 folder 叶子菜单；无 `pageId` 的 link/iframe 等叶子不会产生页面级联。
- 非空目标必须是 folder，并显式传 `--recursive`；计划必须覆盖完整子树且保持叶子优先、根目录最后。
- folder 的 resources 必须为空；非 folder 叶子可以保留其资源配置，具有 `pageId` 时页面与 PageSchema 删除由平台现有删除语义处理。
- 任一计划内 pageId 被计划外菜单引用时返回 `menu_page_shared`，避免平台页面级联影响计划外菜单。
- expect 条件不一致时返回 `menu_state_drift`，不写入。
- dry-run 必须同时提供 `--id` 和 `--plan-out`；正式执行以 `--plan` 为唯一目标来源，不再接受 ID、`--recursive` 或 expect 条件。
- 计划创建后 30 分钟过期，并绑定 `appCode`、`snapshotHash`、完整目标集合和删除顺序；时间字段异常、计划不完整、顺序被修改或任一事实漂移时拒绝写入。
- dry-run 的 `cascade.pages` 只列出菜单直接绑定的 pageId；平台可能按既有规则继续处理关联 Page/PageSchema，不要把该列表解释为服务端全部级联影响。
- dry-run 的 `plannedActions` 必须只有一个批量删除动作，`body.idList` 与叶子优先的 `deleteOrder` 完全一致。
- 正式执行只消费计划一次；批量请求不提供整体事务，错误时可能已有部分 ID 被删除。
- 请求错误或返回计数与计划数量不一致时不自动重试；结构化结果通过 `absentIds` 和 `remainingIds` 表达回读事实。
- 成功计数匹配后仍要重新读取 menu list；只有所有计划 ID 均已消失时才报告成功。
- 出现部分结果或状态未知时，先读取最新菜单事实并生成新计划，禁止再次消费原计划。

## 输出

dry-run 的 `data` 包含 `operation / selector / before / targets / deleteOrder / plannedActions / cascade / snapshotHash / warnings`，计划文件同时记录 `appCode`、有效期、完整目标事实和指纹。`plannedActions` 中的单个批量请求以 `idList` 携带完整 `deleteOrder`。正式执行返回批量确认与写后回查结果；计划过期、快照漂移、计划缺项、计数不一致或其他 blocker 均以结构化失败返回，不会按当前事实重新规划。

## 相关命令

- [menu list](rabetbase-menu-list.md)
- [菜单异常治理](../guides/menu-anomaly-manual-cleanup.md)
