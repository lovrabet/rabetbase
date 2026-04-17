# sql pull

将远端自定义 SQL 同步到本地 **`<项目根>/.rabetbase/sql/<appCode>/<dbName|db-<id>>/`**，并刷新 `.rabetbase/sql.lock.json`。

## 命令

```bash
rabetbase sql pull --format json
rabetbase sql pull --sqlcode <xxxxxxxx-xxxxxxxx> --format json
rabetbase sql pull --name <过滤名> --format json
rabetbase sql pull --dry-run --format json
rabetbase sql pull --force --format json
```

## 参数

| Flag | 说明 |
|------|------|
| `--sqlcode` | 仅拉取指定 SQL（格式 `xxxxxxxx-xxxxxxxx`） |
| `--name` | 按展示名过滤（与 list API 一致） |
| `--force` | 覆盖与远端不一致的本地文件 |
| `--dry-run` | 预览目标路径与状态（不写文件） |

## 输出文件

每条 SQL 会写入：

```text
.rabetbase/sql/<appCode>/<dbName|db-<id>>/<sqlCode>_<sqlName>.sql|xml
```

文件头部会自动补齐：

- `-- @lovrabet.sqlCode`
- `-- @lovrabet.sqlName`
- `-- @lovrabet.dbId`
- `-- @lovrabet.dbName`
- `-- @lovrabet.mode`
- `-- @lovrabet.syncedAt`
- `-- @lovrabet.description`

正文为远端 `sqlContent`；同时会更新 `.rabetbase/sql.lock.json` 中的 `path / hash / remoteId / version / dbId / sqlName / mode`。

## 行为说明

- 本地文件与远端生成内容完全一致：计入 `skipped: unchanged`
- 本地存在且与远端不同、未加 `--force`：跳过并提示 `local differs from remote`
- `--force` 且存在需覆盖的本地文件时：交互模式下会先确认（非交互需显式 `--force`）
- 即使本地 `status` 为 `unchanged`，若远端后来被平台侧改动，`pull --dry-run` 仍可能提示冲突；这说明是“远端漂移”，不是 CLI 入口故障

## 参考

- [sql-creation-workflow.md](../guides/sql-creation-workflow.md)
- [SKILL.md](../SKILL.md)
