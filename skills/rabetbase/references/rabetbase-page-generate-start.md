# page generate-start

异步提交或复用智能列表页（Smart List Page）生成任务。

## 命令

```bash
rabetbase page generate-start --datasetcode 097b7361b76c42bcb12b923fa5a08861 --format json
rabetbase page generate-start --alias order --format json
rabetbase page generate-start --datasetcode 097b7361b76c42bcb12b923fa5a08861 --dry-run --format json
```

## 行为说明

- 这是智能列表页生成的 **async-first 提交入口**。
- CLI 会先查 `standard-page-status` 做前置判定。
- 若允许生成，CLI 会自动生成 `clientOperationId`，并调用 Java 侧 `generate-standard-pages/start`。
- 返回结果里的 `operationId` 是服务端 job 主标识；`clientOperationId` 是调用方恢复 / 幂等锚点。
- `query.command` 默认给出 `operationId` 查询；调用方若只保留了 `clientOperationId`，也可以用它恢复到同一条任务查询。
- Agent / 自动化编排优先直接复用返回值里的 `query.command`，不要自己重新拼接恢复查询命令。

## 参考

- [rabetbase-page-generate-status.md](rabetbase-page-generate-status.md)
- [rabetbase-standard-page-status.md](rabetbase-standard-page-status.md)
