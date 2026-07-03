# dataset relation-audit

只读审计数据集关系事实是否结构完整，并列出明确错误、结构风险和需要人工复核的关系。它消费 `dataset relations` 同一套关系事实，不修改数据集关系，也不审计页面字段选项绑定。

## 命令

```bash
rabetbase dataset relation-audit --format compress
rabetbase dataset relation-audit --datasetcode <datasetCode> --format compress
rabetbase dataset relation-audit --from-source DB_TABLE --to-source METADATA --format compress
```

## 参数

| Flag | 类型 | 必填 | 说明 |
|------|------|------|------|
| `--datasetcode <code>` | string | 否 | 只审计指定来源数据集的关系 |
| `--from-source <type>` | string | 否 | 来源数据集类型：`DB_TABLE` / `METADATA` |
| `--to-source <type>` | string | 否 | 目标数据集类型：`DB_TABLE` / `METADATA` |
| `--appcode <code>` | string | 否 | 覆盖当前配置中的 AppCode |

## 输出

输出包含 `summary` 与按风险排序的 `items`。`status` 取值为 `ok`、`warning`、`error`、`manual_review`；`manual_review` 表示 CLI 不能仅凭元数据证明业务语义正确，需要人工确认。

```json
{
  "appCode": "app-xxx",
  "total": 1,
  "summary": {
    "ok": 0,
    "warning": 0,
    "error": 0,
    "manualReview": 1
  },
  "items": [{
    "status": "manual_review",
    "severity": "medium",
    "manualReviewRequired": true,
    "from": { "source": "DB_TABLE", "datasetCode": "order", "field": "customer_id" },
    "to": { "source": "DB_TABLE", "datasetCode": "customer", "field": "id" },
    "relation": { "cardinality": "MANY_TO_ONE" },
    "reasons": [{
      "code": "missing_label_field",
      "message": "目标关系缺少展示字段，需要人工确认业务展示口径"
    }]
  }]
}
```

## Agent 使用规则

- DB 增量变更排查顺序：先 `db diff` 看物理表与数据集差异，再 `dataset relation-audit` 看关系事实风险。
- 页面展示异常排查顺序：先 `dataset relations` 确认关系事实，再 `dataset relation-audit` 判断关系事实是否需要修复，最后用 `page relation-audit` 审计页面绑定。
- `error` 表示结构上能确定的问题，例如目标数据集不存在、字段不存在或方向不支持。
- `manual_review` 不是失败结论，表示缺展示字段、关系基数不明或物理约束证据不足，需要人工确认业务口径。
- 本命令只读；修复关系必须先走 `dataset relation-create/update/delete --dry-run`，并等用户确认。

## 参考

- [dataset relations](rabetbase-dataset-relations.md)
- [dataset relation mutations](rabetbase-dataset-relation-mutations.md)
- [page relation binding](rabetbase-page-relation-binding.md)
