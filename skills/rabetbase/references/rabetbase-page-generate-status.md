# page generate-status

查询智能列表页（Smart List Page）生成任务状态。

## 命令

```bash
rabetbase page generate-status --datasetcode 097b7361b76c42bcb12b923fa5a08861 --operation-id op-001 --format json
rabetbase page generate-status --datasetcode 097b7361b76c42bcb12b923fa5a08861 --client-operation-id client-001 --format json
```

## 参数

- `--operation-id` 与 `--client-operation-id` 必须二选一
- `--operation-id` 优先用于异步主路径
- `--client-operation-id` 用于调用方恢复 / 幂等查询；通常只在拿不到或丢失 `operationId` 时使用

## 行为说明

- 命令查询的是**任务状态**，不是页面事实。
- 当任务进入成功终态时，CLI 会补查 `standard-page-status`，把页面事实一并附在结果里，方便你直接判断是否形成完整智能列表页组。
- 若任务仍在 `PENDING/RUNNING`，结果会保持机器可读的 `status / nextAction / query`。
- 提交生成后保存 `operationId` 与 `clientOperationId`；超时或网络失败时复用原标识查询，不重新执行 `generate-start --apply`。
- 成功终态时，页面组锚点位于 `data.standardPageStatus.pageSets[].versionTag`。存在多个完整组、残留页或冲突时交给用户确认，不猜测 `versionTag`。
- 确认页面组后，使用 `page pull --datasetcode <code> --version-tag <tag>` 精确拉取。

## 参考

- [rabetbase-page-generate-start.md](rabetbase-page-generate-start.md)
- [rabetbase-standard-page-status.md](rabetbase-standard-page-status.md)
