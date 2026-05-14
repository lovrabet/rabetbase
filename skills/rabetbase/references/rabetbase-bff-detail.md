# bff detail

根据 ID 获取 BFF 脚本详情（含完整脚本内容）。

## 命令

```bash
rabetbase bff detail --id 42 --format json
rabetbase bff detail --id 42 --verbose --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--id <id>` | number | 是 | — | BFF 脚本 ID（从 `bff list` 获取） |
| `--verbose` | boolean | 否 | — | 返回完整原始对象 |
| `--format <fmt>` | string | 否 | `pretty` | 输出格式 |

## 输出

返回 BFF 脚本的完整信息：id、functionName、scriptContent（完整脚本源码）、scriptType、description。

## 使用边界

`bff detail` 用于确认远端事实和取得最新源码。长期维护仍应在 `.rabetbase/bff/<appCode>/...` 同步目录中进行；不要把 `scriptContent` 复制到任意目录后再期望 `bff status` / `bff push` 识别。

## 提示

- 修改已有 BFF 前必须先用此命令拉取最新内容（平台是 source of truth）
- scriptContent 是完整的 JavaScript 源码
- 需要落本地维护时，优先用 `rabetbase bff pull --format json` 同步到规范目录

## 参考

- [SKILL.md](../SKILL.md)
- [bff-creation-workflow.md](../guides/bff-creation-workflow.md)
- [backend-function.md](../guides/backend-function.md)
