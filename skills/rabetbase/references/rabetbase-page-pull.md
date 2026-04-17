# page pull

拉取 DOV2 标准页的 formal schema 到本地 **`.rabetbase/page/<appCode>/`**，进入 `pull -> IDE 编辑 -> push` 工作流。

## 命令

```bash
rabetbase page pull --id 1020286 --format json
rabetbase page pull --datasetcode 097b7361b76c42bcb12b923fa5a08861 --format json
rabetbase page pull --datasetcode 097b7361b76c42bcb12b923fa5a08861 --version-tag a1b2c3d4 --format json
rabetbase page pull --id 1020286 --force --format json
rabetbase page pull --datasetcode 097b7361b76c42bcb12b923fa5a08861 --dry-run --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--id <pageId>` | number | 与 `--datasetcode` / `--alias` 互斥二选一 | — | 单页拉取 |
| `--datasetcode <code>` | string | 与 `--id` 互斥；和 `--alias` 二选一 | — | 按数据集批量拉取 |
| `--alias <name>` | string | 与 `--id` 互斥；和 `--datasetcode` 二选一 | — | 数据集别名 |
| `--version-tag <tag>` | string | 否 | — | 批量模式下指定页面组 |
| `--force` | boolean | 否 | `false` | 覆盖本地未推送修改 |
| `--dry-run` | boolean | 否 | `false` | 仅预览将被拉取/跳过的页面 |
| `--format <fmt>` | string | 否 | `compress` | 输出格式 |

## 本地文件

- 页面文件：`.rabetbase/page/<appCode>/<pageId>-<TYPE>.json`
- 基线锁文件：`.rabetbase/page.lock.json`

## 行为说明

- 单页模式：按 `pageId` 拉取单个页面并更新 lock
- 批量模式：按数据集筛选关联页面；若存在多个 `versionTag` 组且未指定 `--version-tag`，命令会 fail-fast
- 本地文件存在未推送修改且未传 `--force` 时，会跳过或报冲突

## 提示

- 本地编辑 formal schema 前，先执行一次 pull 建立基线
- 若要理解 schema 组件语义，再按需查看 `knowledge/page-schema/` 下的 PageSchema 知识；那是组件参考，不是 CLI 命令 reference

## 参考

- [SKILL.md](../SKILL.md)
- [rabetbase-page-push.md](rabetbase-page-push.md)
