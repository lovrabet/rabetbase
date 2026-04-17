# page generate-status

查询标准页生成任务状态。

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
- 当任务进入成功终态时，CLI 会补查 `standard-page-status`，把页面事实一并附在结果里，方便你直接判断是否形成完整标准页组。
- 若任务仍在 `PENDING/RUNNING`，结果会保持机器可读的 `status / nextAction / query`。

## 参考

- [rabetbase-page-generate-start.md](rabetbase-page-generate-start.md)
- [rabetbase-standard-page-status.md](rabetbase-standard-page-status.md)
