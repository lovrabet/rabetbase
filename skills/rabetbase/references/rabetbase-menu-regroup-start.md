# menu regroup-start

提交一个异步任务：新建根级 folder，并将一组精确的非 folder 叶子菜单放入其中。命令返回重新分组任务的 `taskId/status`。

> **风险等级：high-risk-write** — 必须先 dry-run，正式执行必须显式 `--yes`。

## 命令

```bash
rabetbase menu regroup-start --label "业务管理" --menu-ids <menuId1>,<menuId2> --dry-run --format compress
rabetbase menu regroup-start --label "业务管理" --menu-ids <menuId1>,<menuId2> --yes --format compress
rabetbase task status --task-id <taskId> --format compress
```

## Flags

| Flag | 类型 | 必填 | 说明 |
|---|---|---|---|
| `--label <label>` | string | 是 | 新建根 folder 名称；不能与现有根 folder 同名 |
| `--menu-ids <ids>` | string | 是 | 逗号分隔的精确叶子菜单 ID，最多 100 个 |
| `--dry-run` | boolean | 预览时 | 返回当前快照、before/after 和请求体，不写入 |
| `--yes` | boolean | 正式执行时 | 显式授权重新分组 |

## 状态流

1. 命令校验精确 ID、叶子类型、重复 ID、根 folder 同名冲突和快照漂移。
2. 服务端接受请求并返回异步重新分组任务。
3. 命令收到有效 `taskId` 即视为提交成功；随后尝试回读 `menu list`。分组暂不可见或回读失败时返回 `groupingApplied="unconfirmed"`，不将任务误判为失败。
4. `PENDING / PROCESSING` 继续查询同一 taskId；`SUCCESS / FAILED` 是终态。

> **Agent 完成口径：** 有效 `taskId + PENDING/PROCESSING` 表示任务提交成功，但不表示重新分组完成。必须执行返回的 `data.query.command`，查询同一个 taskId 到 `SUCCESS / FAILED`。保留初始响应的 `data.verification.lookupCommand`；无论初始响应已是 `SUCCESS`，还是后续查询到 `SUCCESS`，都必须执行该只读命令并确认最终菜单事实。回读仍未确认时只重试只读查询，不得重提 `regroup-start`。

## 安全规则

- 请求非幂等：每次被接受都会新建一个根 folder。
- 提交失败、超时或任务长期非终态时不自动重提；有 taskId 时只查询原任务，没有 taskId 时先用 `menu list` 检查是否已经分组。
- 只允许非 folder 叶子菜单；既有 folder 层级调整仍由平台处理。
- 命令不暴露通用 `menuGroups` JSON 透传，每次只编排一个新分组。

## 相关命令

- [task status](rabetbase-task-status.md)
- [menu list](rabetbase-menu-list.md)
- [menu group-create](rabetbase-menu-group-create.md)
- [menu group-update](rabetbase-menu-group-update.md)
