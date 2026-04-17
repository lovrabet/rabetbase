# sql delete

删除一条远端 SQL，并把本地同步目录中的对应文件移入 `.rabetbase/sql-trash/`。

> **风险等级：high-risk-write** — 建议先 `--dry-run` 预览；正式执行需 `--yes` 或交互确认。

## 命令

```bash
rabetbase sql delete --sqlcode 2305f915-dd48cd4c --dry-run --format json
rabetbase sql delete --sqlcode 2305f915-dd48cd4c --yes --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--sqlcode <code>` | string | 是 | — | 要删除的 SQL code |
| `--dry-run` | boolean | 否 | — | 预览远端 id 与本地文件路径 |
| `--yes` | boolean | 否 | — | 跳过高风险确认 |
| `--format <fmt>` | string | 否 | `compress` | 输出格式 |

## 行为说明

- 依赖 `.rabetbase/sql.lock.json` 找到远端 `id` 与本地文件
- 远端删除成功后，会移除 lock 条目
- 若本地文件存在，会移动到 **`.rabetbase/sql-trash/<timestamp>/...`**
- 会自动清理同步目录中因此变空的中间目录

## 提示

- 若提示 `SQL lock file not found` 或 `No SQL lock entry found`，先执行 `sql pull` 或确认这条 SQL 是否由当前项目跟踪

## 参考

- [SKILL.md](../SKILL.md)
- [sql-creation-workflow.md](../guides/sql-creation-workflow.md)
