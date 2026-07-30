# menu delete

按线上菜单 ID 精确删除一个空 folder。

> **风险等级：high-risk-write** — 必须先生成删除计划，正式执行必须消费该计划并显式 `--yes`。

## 命令

```bash
rabetbase menu delete --id <folderId> --expected-children-count 0 --dry-run --plan-out ./menu-delete.plan.json --format compress
rabetbase menu delete --plan ./menu-delete.plan.json --yes --format compress
```

## Flags

| Flag | 类型 | 必填 | 说明 |
|---|---|---|---|
| `--id <id>` | number | 计划阶段 | 精确空 folder ID；不接受 label/path 定位 |
| `--expect-parent-id <id\|root>` | string | 否 | 计划阶段断言当前父级 |
| `--expect-path <path>` | string | 否 | 计划阶段断言当前 path |
| `--expected-children-count <n>` | number | 否 | 计划阶段断言直接子节点数量，删除 folder 时推荐为 `0` |
| `--plan-out <file>` | string | 计划阶段 | 保存可审阅的删除计划 |
| `--dry-run` | boolean | 计划阶段 | 返回目标快照和影响计划，不写入 |
| `--plan <file>` | string | 正式执行 | 读取 30 分钟内生成的删除计划 |
| `--yes` | boolean | 正式执行 | 显式授权删除 |

## 安全规则

- 目标必须存在于当前 `menu list` 线上事实中，且 `type=folder`。
- 目标必须同时满足 `childrenCount=0`、`pageId=null`、`resources=[]`；普通页面菜单和含资源的 folder 交由平台处理。
- expect 条件不一致时返回 `menu_state_drift`，不写入。
- dry-run 必须同时提供 `--id` 和 `--plan-out`；正式执行以 `--plan` 为唯一目标来源，不再接受 ID 或 expect 条件。
- 计划创建后 30 分钟过期，并绑定 `appCode`、`snapshotHash` 和完整目标事实；时间字段被延长、计划来自未来或任一绑定事实漂移时拒绝写入。
- 允许执行的计划不包含页面或 PageSchema 级联。
- 写入后自动重新读取 menu list，确认目标 ID 已消失。

## 输出

dry-run 的 `data` 直接包含 `operation / selector / before / after / dryRun / backend / snapshotHash / warnings`，计划文件同时记录 `appCode`、有效期和目标事实。正式执行会返回删除与写后回查结果；计划过期、快照漂移或 blocker 均以结构化失败返回，不会按当前事实重新规划。

## 相关命令

- [menu list](rabetbase-menu-list.md)
- [菜单异常治理](../guides/menu-anomaly-manual-cleanup.md)
