# sql save

已废弃。

> **迁移说明：**
> - 新建 SQL：改用 `rabetbase sql create --name <name> --db-id <id> --mode sql|mybatisXml`
> - 修改 SQL：编辑 `.rabetbase/sql/<appCode>/<dbName|db-<id>>/<sqlCode>_<sqlName>.sql|xml` 后改用 `rabetbase sql push --sqlcode <code> --yes`

当前 CLI 保留该命令只是为了给出迁移提示；不再执行实际保存。

## 旧命令示例（仅用于识别）

```bash
rabetbase sql save --file ./tmp/getUserList.sql --sqlname getUserList --format json
```

执行后应收到“已废弃”的迁移提示，而不是实际保存。

## 替代工作流

### 新建 SQL

```bash
rabetbase sql create --name getUserList --db-id 10001 --mode sql --format json
```

### 修改 SQL

```bash
rabetbase sql validate --file ./.rabetbase/sql/app-xxxxxxxx/sample_db/2305f915-dd48cd4c_getUserList.sql --format json
rabetbase sql push --sqlcode 2305f915-dd48cd4c --dry-run --format json
rabetbase sql push --sqlcode 2305f915-dd48cd4c --yes --format json
```

## 说明

- `sql validate` 仍然保留，作为 SQL 内容校验命令
- 历史上由 `sql save` 触发的 `blocked` 冲突处理说明，见 [`guides/conflict-detection.md`](../guides/conflict-detection.md)

## 参考

- [SKILL.md](../SKILL.md)
- [sql-creation-workflow.md](../guides/sql-creation-workflow.md)
- [conflict-detection.md](../guides/conflict-detection.md)
