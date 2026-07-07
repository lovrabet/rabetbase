# dataset generate-start / generate-status

从自然语言需求生成可审阅的数据集 design 快照，再基于快照提交异步创建任务，最后查询任务状态拿到新 Dataset。

推荐顺序：

1. `dataset generate-start` preview：生成并写出本地 design JSON。
2. `dataset generate-start --apply`：读取已审阅的 design JSON，提交异步任务。
3. `dataset generate-status`：查询任务状态；成功后使用 `createdDataset.code`。

## 适用范围

当前能力面向服务端支持的 `METADATA` 数据集生成链路。design 文件结构由服务端返回并由 CLI 原样提交，调用方不需要手写或选择内部 DO 版本。

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
rabetbase dataset generate-status --operation-id op-xxx --format compress
```

如果没有 `operationId`，使用 `clientOperationId`：

```bash
rabetbase dataset generate-status --client-operation-id dataset-generate-xxx --format compress
```

`--operation-id` 与 `--client-operation-id` 二选一。只有 status 输出 `status=generated_dataset_created` 且包含 `createdDataset.code` 时，后续才可执行 `dataset detail`、`page generate-start` 或 `api pull`。

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

rabetbase dataset generate-status --operation-id op-xxx --dry-run --format compress
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
  "operationId": "op-xxx",
  "clientOperationId": "dataset-generate-xxx",
  "jobStatus": "PENDING",
  "progressRate": 0,
  "currentStep": "queued",
  "reused": false,
  "status": "operation_pending",
  "nextAction": "query_operation_status",
  "query": {
    "command": "rabetbase dataset generate-status --operation-id op-xxx --format compress"
  }
}
```

Status 成功后的 `data` 包含：

```json
{
  "operation": "dataset.generate.status",
  "appCode": "app-xxx",
  "operationId": "op-xxx",
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
    "command": "rabetbase dataset generate-status --operation-id op-xxx --format compress"
  }
}
```

`status` 常见值：

| status | 含义 |
|--------|------|
| `operation_pending` | 任务等待执行 |
| `operation_running` | 任务执行中 |
| `generated_dataset_created` | Dataset 已创建，可使用 `createdDataset.code` |
| `operation_failed` | 任务失败，查看 `errorMsg` |
| `unknown_reconcile_failed` | 任务状态无法与 Dataset 结果对应，继续用 `query.command` 查询或人工确认 |

## Agent 执行要求

- 先执行 preview，审阅 `--design-output` 文件，再执行 start。
- 不要把 preview 输出视为已创建 Dataset。
- 不要把 start 输出视为已创建 Dataset；必须用 `generate-status` 查询到 `generated_dataset_created`。
- 成功后用 `rabetbase dataset detail --code <createdDataset.code> --format compress` 确认结构。
- 如需生成智能列表页，再执行 `rabetbase page generate-start --datasetcode <createdDataset.code> --format compress`。
- 真实业务行数据验证交接到 `lovrabet data filter|getOne`。

## 常见失败

| 失败 | 处理 |
|------|------|
| Preview 缺 `--name` / `--description` / `--design-output` | 补齐必填参数 |
| Preview 传了 `--design-file` | 去掉 `--design-file`，或改用 `--apply` |
| Start 缺 `--design-file` | 指向已审阅的 design JSON 文件 |
| Start 同时传了 `--name` / `--description` / `--design-output` | 删除这些 preview 参数 |
| Status 缺任务标识 | 传 `--operation-id` 或 `--client-operation-id` |
| design 文件不是 JSON object | 修正为 JSON object 后重试 |

## 参考

- [dataset detail](rabetbase-dataset-detail.md)
- [page generate-start](rabetbase-page-generate-start.md)
- [api pull](rabetbase-api-pull.md)
