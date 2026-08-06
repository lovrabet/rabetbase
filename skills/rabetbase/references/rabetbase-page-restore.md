# page restore

恢复当前应用下一条已删除的 STANDARD 或 JSX 页面。

## 命令

```bash
rabetbase page restore --id 1023895 --dry-run --format json
rabetbase page restore --id 1023895 --format json
rabetbase page restore --id 1450 --page-type JSX --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--id <pageId>` | number | 是 | — | 已删除页面 ID，必须为正整数 |
| `--page-type <type>` | string | 否 | — | `STANDARD` 或 `JSX`；用于缩小查询范围或消除跨类型同 ID 歧义 |
| `--dry-run` | boolean | 否 | `false` | 定位页面并预览恢复请求，不执行恢复 |
| `--format <fmt>` | string | 否 | `compress` | 输出格式 |

## 行为说明

- 未传 `--page-type` 时，CLI 查询当前应用下 STANDARD 与 JSX 两类已删除页面，并按 `--id` 唯一定位。
- 同一个数字 ID 在两类页面中同时存在时，CLI 不猜测目标；需显式传 `--page-type STANDARD` 或 `--page-type JSX`。
- STANDARD 页面恢复请求只携带页面 ID；JSX 页面恢复请求同时携带当前 `appCode` 和页面 ID。
- 自动识别时任一类型查询失败都会终止恢复，避免在无法排除同 ID 歧义时写错目标。
- JSX 恢复返回未更新记录时，按状态已变化或页面已恢复处理，不报告成功。

## 输出

结构化结果的稳定字段包括：

- `data.operation = "page.restore"`
- `data.appCode`
- `data.selector.id`
- `data.selector.pageType`
- `data.page.id/name/pageType/type/datasetIds`
- `data.restored`
- `data.dryRun`
- `data.backend.method/path`

## 参考

- [SKILL.md](../SKILL.md)
