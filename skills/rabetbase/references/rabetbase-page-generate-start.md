# page generate-start

异步提交或复用智能列表页（Smart List Page）生成任务。

## 命令

```bash
# 默认行为：仅预览，不调用平台 API
rabetbase page generate-start --datasetcode 097b7361b76c42bcb12b923fa5a08861 --format json
rabetbase page generate-start --alias order --format json
rabetbase page generate-start --datasetcode 097b7361b76c42bcb12b923fa5a08861 --dry-run --format json

# 真正提交任务
rabetbase page generate-start --datasetcode 097b7361b76c42bcb12b923fa5a08861 --apply --format json
rabetbase page generate-start --alias order --apply --format json
```

## 行为说明

- 这是智能列表页生成的 **async-first 提交入口**。
- **默认仅返回 dry-run 预览，不会调用 `generate-standard-pages/start`**；需要真正提交任务必须显式加 `--apply`。`--dry-run` 与不传 `--apply` 行为等价。
- CLI 会先查 `standard-page-status` 做前置判定。
- 若允许生成且传入 `--apply`，CLI 会自动生成 `clientOperationId`，并调用 Java 侧 `generate-standard-pages/start`。
- 返回结果里的 `operationId` 是服务端 job 主标识；`clientOperationId` 是调用方恢复 / 幂等锚点。
- `query.command` 默认给出 `operationId` 查询；调用方若只保留了 `clientOperationId`，也可以用它恢复到同一条任务查询。
- Agent / 自动化编排优先直接复用返回值里的 `query.command`，不要自己重新拼接恢复查询命令。
- Agent 自动化场景下，编排器应先跑一次默认预览校验候选数据集，再带 `--apply` 提交；不应直接对所有数据集跑 `--apply`。

## 参考

- [rabetbase-page-generate-status.md](rabetbase-page-generate-status.md)
- [rabetbase-standard-page-status.md](rabetbase-standard-page-status.md)
