# sql status

检查本地 SQL 同步目录与 `.rabetbase/sql.lock.json` 的状态。

## 命令

```bash
rabetbase sql status --format json
rabetbase sql status --remote --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--remote` | boolean | 否 | — | 额外检查远端存在但本地/lock 都没有的 SQL |
| `--format <fmt>` | string | 否 | `compress` | 输出格式 |

## 输出分类

| 字段 | 含义 |
|------|------|
| `added` | 本地同步目录中存在，但 `sql.lock.json` 尚未跟踪 |
| `modified` | 本地文件 hash、文件名（`sqlName`）、或相对路径与 lock 不一致 |
| `missing` | lock 中有条目，但本地文件缺失 |
| `unchanged` | 本地文件与 lock 一致 |
| `remoteOnly` | 仅在加 `--remote` 时返回：远端存在，但本地和 lock 都没有 |

## 提示

- 修改文件名会被视为 `sqlName` 变更；`sql push` 时会把新文件名回写远端
- 把文件移动到新的数据库目录，会在 `sql push` 时尝试按目录名重新绑定 `dbId`
- `status` 只看本地文件与 lock；若要核对远端是否漂移，可结合 `sql pull --dry-run` 或 `sql detail`

## 参考

- [SKILL.md](../SKILL.md)
- [sql-creation-workflow.md](../guides/sql-creation-workflow.md)

