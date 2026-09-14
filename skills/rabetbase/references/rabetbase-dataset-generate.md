# dataset generate-start / generate-status

从自然语言需求生成可审阅的单数据集或多数据集批次 design 快照，再基于快照提交异步创建任务，最后查询任务状态拿到完整结果。

推荐顺序：

1. `dataset generate-start` preview：生成并写出本地 design JSON。
2. `dataset generate-start --apply`：读取已审阅的 design JSON，提交异步任务。
3. `dataset generate-status`：查询任务状态；成功后使用 `createdDataset.code`。

## 适用范围

当前能力面向服务端支持的 `METADATA` 数据集生成链路。单数据集保持原协议；批次使用 `kind=dataset-batch`、`schemaVersion=1`，一次创建 1–20 个数据集及其 lookup 关系。

## Batch：一次生成多个关联数据集

```bash
rabetbase dataset generate-start \
  --batch \
  --description "创建客户和订单数据集，订单通过 customer_id 关联客户 id" \
  --design-output .rabetbase/dataset-designs/customer-orders.json \
  --format compress
```

批次 Preview 不需要且不接受 `--name`。审阅文件时确认：`localKey` 唯一且为小写标识；每项 `design.relations` 为空；所有关系只出现在顶层 `relations`；`unresolvedRelations` 仅是待确认提示，不会落库。

```bash
rabetbase dataset generate-start \
  --apply \
  --design-file .rabetbase/dataset-designs/customer-orders.json \
  --format compress
```

Apply 会按 `kind` 自动识别批次。`--batch` 可以显式要求输入必须是批次文件。首次提交默认生成 `dataset-batch-...` 恢复 ID；明确恢复或重试同一批时可传 `--client-operation-id <原ID>`。提交超时、响应损坏或未确认批次模式时，错误 `dataset_batch_submit_outcome_unknown` 会保留原 `clientOperationId`；只运行 `data.query.command` 或文本提示中的 status 查询，不要再次 POST。

批次 start 响应必须确认 `mode=batch`。Status 只有同时满足以下条件才返回 `generated_datasets_created`：

1. `expectedLocalKeys` 与 `createdDatasets[].localKey` 精确一致，每项都有真实 code。
2. `expectedRelations` 与 `createdRelations` 按来源数据集、来源字段、目标数据集、目标字段、relationType、cardinality 六元组精确一致；字段名比较忽略大小写。
3. 集合中不存在缺失、重复或多余项，真实关系携带的数据集 code 与创建结果一致。

否则返回 `unknown_reconcile_failed`。批次输出使用复数 `createdDatasets` / `createdRelations`，不输出单数 `createdDataset`。

## Preview：生成 design 快照

```bash
rabetbase dataset generate-start \
  --name "客户档案" \
  --description "记录客户姓名、手机号、归属销售和跟进状态" \
  --design-output .rabetbase/dataset-designs/customer-profile.json \
  --format compress
```

Preview 阶段必填：

| Flag | 说明 |
|------|------|
| `--name <name>` | 目标 Dataset 展示名 |
| `--description <text>` | 自然语言需求描述 |
| `--design-output <path>` | 写出的本地 design JSON 文件 |
| `--appcode <code>` | 可选；未配置默认 app 时必填 |
| `--format compress` | Agent 优先使用 |

Preview 阶段禁止传 `--design-file`。Preview 只产生 design 文件，不创建 Dataset。

## Start：提交异步创建任务

确认 design 文件后再执行：

```bash
rabetbase dataset generate-start \
  --apply \
  --design-file .rabetbase/dataset-designs/customer-profile.json \
  --format compress
```

Start 阶段必填：

| Flag | 说明 |
|------|------|
| `--apply` | 提交异步创建任务 |
| `--design-file <path>` | 已审阅的 design JSON 文件 |
| `--appcode <code>` | 可选；未配置默认 app 时必填 |
| `--format compress` | Agent 优先使用 |

Start 阶段禁止同时传 `--name`、`--description`、`--design-output`。Start 不会重新根据描述生成 design，也不代表 Dataset 已创建完成。

## Status：查询任务状态

优先使用 start 输出里的 `query.command`：

```bash
rabetbase dataset generate-status --task-id 2da9d1d7-8e34-4ce5-97af-72d2df9870aa --format compress
```

如果旧响应没有 `taskId`，可回退 `operationId`；如果两者都没有，再使用 `clientOperationId`：

```bash
rabetbase dataset generate-status --operation-id op-xxx --format compress
rabetbase dataset generate-status --client-operation-id dataset-generate-xxx --format compress
```

`--task-id`、`--operation-id` 与 `--client-operation-id` 必须且只能传一个。taskId 是首选恢复标识；CLI 会把它映射为后端 Dataset status 的 `operationId` query，服务端接口本身没有原生 taskId query。只有 status 输出 `status=generated_dataset_created` 且包含非空 `createdDataset.code` 时，后续才可执行 `dataset detail`、`page generate-start` 或 `api pull`。

## 生成耗时与失败阶段

Preview、Start、Status 的 `data.metrics` 透传服务端实际提供的指标，`data.failedStage` 表示服务端报告的失败阶段。旧响应或没有可用指标时返回 `null`，不代表耗时为零。指标与 design 分离，不能把指标写回设计快照或用于重新创建任务。

可选指标包括：`schemaVersion`、`attemptId`、`attempt`、`datasetCount`、`fieldCount`、`previewDurationMs`、`queueWaitMs`、`executionDurationMs`、`resultReadDurationMs`、`stageDurationsMs`、`modelCallCount`、`retryCount`。耗时单位为毫秒；只使用返回中存在的测量值，不补造缺失阶段。重复 Status 查询不会累计服务端执行时间，`resultReadDurationMs` 仅表示本次结果读取。阶段可能嵌套，不能把阶段耗时相加当作端到端耗时。

采集基线时：

1. 固定输入、模型、环境、数据集数和字段数，分别记录简单结构、复杂枚举、已有数据集引用样例；保留每次原始输出和任务 ID。
2. 预热后每个样例至少重复 10 次，分开记录 preview、排队、执行、结果读取及客户端观察到的完成耗时；排除人工审阅 design 的时间。10 次仅供初步比较，不足以稳定估计 P95。
3. 优化前后保持相同条件，比较中位耗时和长尾，同时回读完整字段、枚举、主键及关联，不能以漏保存换速度。
4. 失败、超时或响应丢失时恢复原任务，不重提；缺指标样本应标注不可用，不能当作零耗时样本。

以上步骤只适用于有授权的测试环境。批次指标仍与 design 分离；指标出现只说明可观测性可用，不能独自证明性能已改善或批次结果完整。

## dry-run

三条路径都支持 `--dry-run`：

```bash
rabetbase dataset generate-start \
  --name "客户档案" \
  --description "记录客户姓名和手机号" \
  --design-output .rabetbase/dataset-designs/customer-profile.json \
  --dry-run \
  --format compress

rabetbase dataset generate-start \
  --apply \
  --design-file .rabetbase/dataset-designs/customer-profile.json \
  --dry-run \
  --format compress

rabetbase dataset generate-status --task-id 2da9d1d7-8e34-4ce5-97af-72d2df9870aa --dry-run --format compress
```

Preview dry-run 只返回将调用的 preview endpoint 和 body，不写文件。Start dry-run 会读取本地 design 文件，并返回将调用的 start endpoint 和 body，但不提交任务。Status dry-run 只返回将查询的 status endpoint。

## 输出

Preview 输出的 `data` 包含：

```json
{
  "operation": "dataset.generate.preview",
  "input": {
    "appCode": "app-xxx",
    "datasetName": "客户档案",
    "requirementDescription": "记录客户姓名和手机号"
  },
  "design": {},
  "designFile": ".rabetbase/dataset-designs/customer-profile.json",
  "warnings": [],
  "nextAction": {
    "command": "rabetbase dataset generate-start --apply --design-file .rabetbase/dataset-designs/customer-profile.json --format compress"
  }
}
```

Start 输出的 `data` 包含：

```json
{
  "operation": "dataset.generate.start",
  "appCode": "app-xxx",
  "taskId": "2da9d1d7-8e34-4ce5-97af-72d2df9870aa",
  "operationId": "2da9d1d7-8e34-4ce5-97af-72d2df9870aa",
  "clientOperationId": "dataset-generate-xxx",
  "jobStatus": "PENDING",
  "progressRate": 0,
  "currentStep": "queued",
  "reused": false,
  "status": "operation_pending",
  "nextAction": "query_operation_status",
  "query": {
    "command": "rabetbase dataset generate-status --task-id 2da9d1d7-8e34-4ce5-97af-72d2df9870aa --format compress"
  }
}
```

Status 成功后的 `data` 包含：

```json
{
  "operation": "dataset.generate.status",
  "appCode": "app-xxx",
  "taskId": "2da9d1d7-8e34-4ce5-97af-72d2df9870aa",
  "operationId": "2da9d1d7-8e34-4ce5-97af-72d2df9870aa",
  "clientOperationId": "dataset-generate-xxx",
  "jobStatus": "SUCCESS",
  "progressRate": 100,
  "currentStep": "done",
  "errorMsg": null,
  "status": "generated_dataset_created",
  "createdDataset": {
    "id": 123,
    "code": "1a90dbff5f094a9a89936fa99b10984c",
    "name": "客户档案",
    "tableName": "meta_customer_profile",
    "fieldCount": 4,
    "relationCount": 0
  },
  "nextAction": null,
  "query": {
    "command": "rabetbase dataset generate-status --task-id 2da9d1d7-8e34-4ce5-97af-72d2df9870aa --format compress"
  }
}
```

`status` 常见值：

| status | 含义 |
|--------|------|
| `operation_pending` | 任务等待执行 |
| `operation_running` | 任务执行中；服务端 `PROCESSING` / `RUNNING` 等非终态均继续查询同一个 taskId |
| `generated_dataset_created` | Dataset 已创建，可使用 `createdDataset.code` |
| `operation_failed` | 任务失败，查看 `errorMsg` |
| `unknown_reconcile_failed` | 任务状态无法与 Dataset 结果对应，继续用 `query.command` 查询或人工确认 |

## Agent 执行要求

- 先执行 preview，审阅 `--design-output` 文件，再执行 start。
- 不要把 preview 输出视为已创建 Dataset。
- 不要把 start 输出视为已创建 Dataset；必须用 `generate-status` 查询到 `generated_dataset_created`。
- `PENDING` / `PROCESSING` 都不是完成；始终继续查询同一个 taskId。
- 超时、未知状态或 start 响应丢失时只做状态恢复，不得自动重提生成请求。
- 批次响应丢失时保留并复用原 `clientOperationId`；同一 design 自动生成新 ID 会成为另一批操作。
- 批次成功后逐项使用 `createdDatasets[].code`，并核对 `createdRelations`；不要读取单数 `createdDataset`。
- 成功后用 `rabetbase dataset detail --code <createdDataset.code> --format compress` 确认结构。
- 如需生成数据列表页，再执行 `rabetbase page generate-start --datasetcode <createdDataset.code> --format compress`。
- 真实业务行数据验证交接到 `lovrabet data filter|getOne`。

## 常见失败

| 失败 | 处理 |
|------|------|
| Preview 缺 `--name` / `--description` / `--design-output` | 补齐必填参数 |
| Preview 传了 `--design-file` | 去掉 `--design-file`，或改用 `--apply` |
| Start 缺 `--design-file` | 指向已审阅的 design JSON 文件 |
| Start 同时传了 `--name` / `--description` / `--design-output` | 删除这些 preview 参数 |
| Batch Preview 传了 `--name` | 删除 `--name`；批次由 description 描述整体需求 |
| `--batch --apply` 读取单设计 | 使用 `kind=dataset-batch`、`schemaVersion=1` 的批次文件 |
| Batch Start 响应未确认 `mode=batch` | 停止，不要自动重提；用原 `clientOperationId` 查询状态 |
| Batch SUCCESS 缺少完整 expected/created 集合 | 按 `unknown_reconcile_failed` 处理，继续恢复查询或人工核验 |
| Status 缺任务标识 | 首选传 `--task-id`；旧响应才回退 `--operation-id` 或 `--client-operation-id` |
| design 文件不是 JSON object | 修正为 JSON object 后重试 |

## 参考

- [dataset detail](rabetbase-dataset-detail.md)
- [page generate-start](rabetbase-page-generate-start.md)
- [api pull](rabetbase-api-pull.md)
