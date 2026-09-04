# rabetbase deployment sync-all

研发资源保存后的增量同步由服务端自动处理。`deployment sync` 已移除，`bff push` 与 `sql push` 也不再追加客户端同步请求。

`sync-all` 是管理员补偿工具，仅用于以下场景：

- 独立部署环境首次初始化。
- 服务端自动同步启用前的历史数据回填。
- 已确认自动同步遗漏后的故障恢复。

它不属于日常研发发布步骤，也不能证明目标环境与源环境完全一致。服务端对源端现存记录执行 UPSERT，不删除目标端额外存在的历史记录。

## 使用前确认

- 已完成 rabetbase 认证，并明确目标应用的 `appCode`。
- 已确认属于初始化、回填或恢复场景，而不是普通资源保存或发布。
- 应用已停止编辑，当前处于安静窗口，避免全量任务与服务端自动增量同步并行。
- 已准备在任务结束后执行独立的业务验收。

## 先预览

dry-run 只生成本地计划，不调用服务端，也不验证部署目标：

```sh
rabetbase deployment sync-all \
  --appcode <appCode> \
  --dry-run \
  --format compress
```

## 正式执行

经用户确认后复用同一 appCode；CI 或非交互环境需要全局 `--yes`：

```sh
rabetbase deployment sync-all \
  --appcode <appCode> \
  --confirm \
  --yes \
  --format compress
```

- `outcome=not_needed`：服务端判定当前应用无需创建任务，属于正常完成。
- `outcome=submitted`：保存 `jobId`，再查询同一任务。
- 提交结果未知：不得再次执行 `sync-all`；先用 `sync-jobs` 恢复任务事实。

## 查询任务

```sh
rabetbase deployment sync-status --job-id <jobId> --format compress
rabetbase deployment sync-jobs --appcode <appCode> --format compress
```

只根据 `isTerminal` 判断是否为已知终态；若为 `null`，保留服务端状态并交给用户判断。逐项资源类型和 action 是服务端开放字符串，不建立枚举闭集，也不因新值自动重提。

任务成功只表示服务端完成了当前支持范围内的 UPSERT。目标端额外记录不会被删除，且任务明细可能受服务端保留上限影响，因此完成后仍需按业务事实验收。
