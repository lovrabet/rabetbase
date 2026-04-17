# sql push

将本地同步目录中的 SQL 文件上传到远端，并刷新 `.rabetbase/sql.lock.json`。

> **风险等级：high-risk-write** — 建议先 `--dry-run` 预览；正式执行需 `--yes` 或交互确认。

## 命令

```bash
rabetbase sql push --format json
rabetbase sql push --sqlcode 2305f915-dd48cd4c --dry-run --format json
rabetbase sql push --sqlcode 2305f915-dd48cd4c --yes --format json
rabetbase sql push --sqlcode 2305f915-dd48cd4c --force --yes --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--sqlcode <code>` | string | 否 | — | 仅推指定 SQL；不传则扫描当前应用同步目录下的全部 SQL |
| `--force` | boolean | 否 | — | 即使本地 hash 与 lock 一致也强制推送 |
| `--dry-run` | boolean | 否 | — | 仅预览会推哪些 SQL |
| `--yes` | boolean | 否 | — | 跳过高风险确认 |
| `--format <fmt>` | string | 否 | `compress` | 输出格式 |

## 行为说明

- 以 `.rabetbase/sql.lock.json` 为准，读取本地同步目录中的当前文件
- 上传前会自动剥离 `@lovrabet` 头注释，只把 SQL/XML 正文发到平台
- 若本地文件名中的 `sqlName` 变化，推送时会同步更新远端 `sqlName`
- 若本地文件被移动到新的数据库目录，推送时会按目录名重新解析目标 `dbId`
- 推送成功后会刷新 lock 中的 `hash / path / sqlName / dbId / version`

## 常见失败

- `missing remote version; run rabetbase sql pull to refresh version info first`
  - 说明 lock 中缺失乐观锁版本号，先执行 `sql pull`
- `local SQL is not tracked by sql.lock.json`
  - 说明这不是已同步资产；先执行 `sql create` 或 `sql pull`
- `Multiple local SQL files found for sqlCode ...`
  - 说明同一 `sqlCode` 存在多个本地文件，先清理到只剩一份

## 参考

- [SKILL.md](../SKILL.md)
- [sql-creation-workflow.md](../guides/sql-creation-workflow.md)
