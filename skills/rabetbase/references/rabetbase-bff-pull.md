# bff pull

把远端 Backend Function 脚本同步到本地 `.rabetbase/bff/<appCode>/...`。

## 命令

```bash
rabetbase bff pull --format json
rabetbase bff pull --type ENDPOINT --format json
rabetbase bff pull --dry-run --format json
rabetbase bff pull --type HOOK --force --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--type <type>` | string | 否 | — | 仅拉指定类型：`COMMON` / `ENDPOINT` / `HOOK` |
| `--force` | boolean | 否 | — | 强制覆盖本地未同步改动 |
| `--dry-run` | boolean | 否 | — | 仅预览将要拉取的远端脚本与目标本地路径 |
| `--format <fmt>` | string | 否 | `pretty` | 输出格式 |

## 输出

返回：
- `pulled`
- `skipped`
- `conflicts`
- `failed`

`--dry-run` 时返回预览对象，包含远端脚本、目标 `filePath`、`alias`、`datasetCode`、Hook operation/node 和预期状态（如 `would_pull` / `conflict`）。有已跟踪的旧 HOOK 目录需要迁移时，还会返回 `hookDirectoryMigrations`。

远端脚本列表会自动聚合服务端全部分页；不会只同步第一页。

## 提示

- 默认会保护本地未同步改动，冲突时进入 `conflicts`，包含受审阅约束的 `nextAction`
- `conflicts[].code=BFF_LOCAL_UNSYNCED` 表示本地脚本自上次 pull 后已修改；先审阅，再在本地版本应成为事实来源时执行 `nextAction.command` 精确推送更新远端。不要把它报告为失败或用 `--force` 覆盖内容
- `--dry-run` 不会改本地文件，也不会写 lock
- HOOK 一级目录优先使用 `api.ts` 模型中的 alias；没有 alias 时使用 Dataset code
- 挂载关系中有明确 Dataset code、但当前 SDK 模型没有 alias 时，仍按原始 Dataset code 下载，不阻断其他脚本；push 更新已有脚本时仍须回读确认脚本身份、版本与挂载关系
- pull 会将 lock 跟踪的旧表名或过期 alias 目录迁到当前 SDK alias；当前没有 alias 时迁到 Dataset code。若目标目录已存在、旧目录已被另一 Dataset alias 复用且含未跟踪脚本，或本地文件系统无法安全迁移，命令会停止且不覆盖本地脚本
- 拉取后建议立刻跑一次 `bff status`

## 参考

- [SKILL.md](../SKILL.md)
- [conflict-detection.md](../guides/conflict-detection.md)
