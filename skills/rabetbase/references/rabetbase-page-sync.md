# page sync

将数据集字段变更同步到已存在的 DOV2 标准页。

## 命令

```bash
rabetbase page sync --datasetcode 097b7361b76c42bcb12b923fa5a08861 --format json
rabetbase page sync --alias order --format json
rabetbase page sync --datasetcode 097b7361b76c42bcb12b923fa5a08861 --dry-run --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--datasetcode <code>` | string | 与 `--alias` 二选一 | — | 数据集 code（32 位 hex） |
| `--alias <name>` | string | 与 `--datasetcode` 二选一 | — | `api.ts` 中的数据集别名 |
| `--dry-run` | boolean | 否 | `false` | 仅预览请求与前置校验，不实际同步页面 |
| `--format <fmt>` | string | 否 | `compress` | 输出格式 |

## 行为说明

- 该命令面向**已有标准页**的增量同步场景。
- CLI 会先检查数据集下是否存在未删除的关联标准页。
- 若当前没有可同步页面，命令会 fail-fast，并提示改走 `rabetbase page generate-start --datasetcode <code>` 这条异步生成路径；具体轮询与回查流程见 [page-development-workflow.md](../guides/page-development-workflow.md)。

## 提示

- 这是平台侧页面编排命令，不会把页面 schema 拉到本地
- 若要进入本地 schema 编辑工作流，使用 [`rabetbase page pull`](rabetbase-page-pull.md)

## 参考

- [SKILL.md](../SKILL.md)
- [rabetbase-page-generate-start.md](rabetbase-page-generate-start.md)
