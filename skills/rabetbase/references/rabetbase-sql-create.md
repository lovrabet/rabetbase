# sql create

先在远端创建一条新的自定义 SQL，再在本地同步目录中生成初始文件并写入 `.rabetbase/sql.lock.json`。

> **风险等级：high-risk-write** — 建议先 `--dry-run` 预览；正式执行需 `--yes` 或交互确认。

## 命令

```bash
rabetbase sql create --name getUserList --db-id 10001 --mode sql --format json
rabetbase sql create --name getUserList --db-id 10001 --mode sql --dry-run --format json
rabetbase sql create --name getUserMapper --db-id 10001 --mode mybatisXml --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--name <name>` | string | 交互补全 / 非交互必填 | — | SQL 展示名 |
| `--db-id <id>` | number | 交互补全 / 非交互必填 | — | 目标数据库 ID |
| `--mode <mode>` | string | 交互补全 / 非交互必填 | — | 本地文件模式：`sql` 或 `mybatisXml` |
| `--dry-run` | boolean | 否 | — | 预览目标路径与请求体 |
| `--yes` | boolean | 否 | — | 跳过高风险确认 |
| `--format <fmt>` | string | 否 | `compress` | 输出格式 |

## 本地输出

成功后会：

- 在远端创建 SQL，拿到服务端生成的 `sqlCode`
- 在 **`.rabetbase/sql/<appCode>/<dbName|db-<id>>/<sqlCode>_<sqlName>.sql|xml`** 生成初始文件
- 在 **`.rabetbase/sql.lock.json`** 写入 lock 条目（含 `remoteId`、`version`、`dbId`、`path`、`mode` 等）

生成文件会自动带上 `@lovrabet` 元注释头，后续 `sql push` 上传时会自动剥离，不会写回平台正文。

## 提示

- 团队长期维护 SQL 时，优先用 `sql create` 建立本地同步资产，而不是直接 `sql save`
- 生成后通常继续执行：编辑本地文件 → `sql validate --file ...` → `sql status` → `sql push --sqlcode ... --dry-run` → `sql push --sqlcode ... --yes`

## 参考

- [SKILL.md](../SKILL.md)
- [sql-creation-workflow.md](../guides/sql-creation-workflow.md)
