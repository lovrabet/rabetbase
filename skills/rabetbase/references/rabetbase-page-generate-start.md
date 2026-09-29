# page generate-start

异步提交或复用数据列表页（Data List Page）生成任务。

生成流程会在生成页面后自动创建菜单入口，无需额外绑定子应用 appName 或执行 `menu sync`。提交成功仅表示任务已受理；跟进原任务，并核对完整页面组及对应菜单事实后，再报告页面和菜单已就绪。

## 命令

```bash
# 默认行为：仅预览，会读取前置事实，不提交生成任务
rabetbase page generate-start --datasetcode 097b7361b76c42bcb12b923fa5a08861 --format json
rabetbase page generate-start --alias order --format json
rabetbase page generate-start --datasetcode 097b7361b76c42bcb12b923fa5a08861 --dry-run --format json

# 真正提交任务
rabetbase page generate-start --datasetcode 097b7361b76c42bcb12b923fa5a08861 --apply --format json
rabetbase page generate-start --alias order --apply --format json
```

## 行为说明

- 这是数据列表页生成的 **async-first 提交入口**。
- **默认仅返回 dry-run 预览，不会调用 `generate-standard-pages/start`**；需要真正提交任务必须显式加 `--apply`。`--dry-run` 与不传 `--apply` 行为等价。
- CLI 会先执行 `page data-list-status` 做前置判定；底层仍调用既有服务端 `standard-page-status` 接口。
- 若允许生成且传入 `--apply`，CLI 会自动生成 `clientOperationId`，并调用 Java 侧 `generate-standard-pages/start`。
- 提交结果固定返回 `pageType=DATA_LIST` 与 `pagePattern=CRUD_PAGE_SET`；命令不接受 `--page-type` 或 `--page-pattern`。
- 返回结果里的 `taskId` 是首选查询标识；`operationId` 用于兼容旧响应，`clientOperationId` 是调用方恢复 / 幂等锚点。
- `query.command` 优先使用 `taskId`，缺失时使用 `operationId`，再回退到 `clientOperationId`，查询时只携带其中一个标识。
- Agent / 自动化编排优先直接复用返回值里的 `query.command`，不要自己重新拼接恢复查询命令。
- Agent 自动化场景下，编排器应先跑一次默认预览校验候选数据集，再带 `--apply` 提交；不应直接对所有数据集跑 `--apply`。

## 参考

- [rabetbase-page-generate-status.md](rabetbase-page-generate-status.md)
- [rabetbase-data-list-status.md](rabetbase-data-list-status.md)
