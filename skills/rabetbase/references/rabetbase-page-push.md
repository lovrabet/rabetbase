# page push

将本地页面 schema 推送回平台；推送成功后会自动回拉 canonical schema 覆盖本地文件与 lock。

## 命令

```bash
rabetbase page push --id 1020286 --format json
rabetbase page push --datasetcode 097b7361b76c42bcb12b923fa5a08861 --format json
rabetbase page push --datasetcode 097b7361b76c42bcb12b923fa5a08861 --version-tag a1b2c3d4 --format json
rabetbase page push --id 1020286 --dry-run --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--id <pageId>` | number | 与 `--datasetcode` / `--alias` 互斥二选一 | — | 单页推送 |
| `--datasetcode <code>` | string | 与 `--id` 互斥；和 `--alias` 二选一 | — | 按数据集批量推送 |
| `--alias <name>` | string | 与 `--id` 互斥；和 `--datasetcode` 二选一 | — | 数据集别名 |
| `--version-tag <tag>` | string | 否 | — | 批量模式下指定页面组 |
| `--dry-run` | boolean | 否 | `false` | 做本地校验与远端版本检查，不实际推送 |
| `--format <fmt>` | string | 否 | `compress` | 输出格式 |

## 前置条件

- 本地已通过 [`rabetbase page pull`](rabetbase-page-pull.md) 建立基线
- 本地页面文件位于 `.rabetbase/page/<appCode>/`
- lock 文件 `.rabetbase/page.lock.json` 存在对应页面记录

## 行为说明

- 单页模式：校验本地文件、读取 lock、比对远端版本，再调用更新接口
- 批量模式：只推送本地 hash 有变更的页面；无基线时会 fail-fast，提示先执行 `page pull`
- 推送后会再次拉取远端 canonical schema，并覆盖本地文件和 lock，避免本地保留非 canonical 版本

## 提示

- 若远端页面已发生变化，命令会要求先重新 pull，而不是盲推覆盖
- 批量模式下若存在多个 `versionTag` 组，需显式传 `--version-tag`

## 参考

- [SKILL.md](../SKILL.md)
- [rabetbase-page-pull.md](rabetbase-page-pull.md)
