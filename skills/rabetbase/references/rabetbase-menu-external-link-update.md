# menu external-link-update

按精确 ID 原地更新一个既有 `iframe` 或 `link` 叶子菜单的 URL。只修改 URL，不删除重建菜单。

> **风险等级：write** — 会改变线上菜单实际打开地址；建议先 dry-run，正式执行复用相同参数并移除 `--dry-run`。

## 命令

```bash
rabetbase menu external-link-update \
  --id "<menu-id>" \
  --url "https://admin.example.com/new-page?hideShell=true" \
  --expect-url "https://admin.example.com/old-page?hideShell=true" \
  --expect-parent-id "<folder-id|root>" \
  --dry-run \
  --format compress

rabetbase menu external-link-update \
  --id "<menu-id>" \
  --url "https://admin.example.com/new-page?hideShell=true" \
  --expect-url "https://admin.example.com/old-page?hideShell=true" \
  --expect-parent-id "<folder-id|root>" \
  --format compress
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--id <id>` | number | 是 | — | 既有菜单的精确正整数 ID |
| `--url <https-url>` | string | 是 | — | 新的绝对 HTTPS URL，不允许 URL userinfo |
| `--expect-url <current-url>` | string | 是 | — | 真正需要写入时精确断言当前 URL；允许断言历史非 HTTPS 地址 |
| `--expect-parent-id <id\|root>` | string | 是 | — | 始终断言当前父级 |
| `--appcode <code>` | string | 否 | 配置解析 | 显式指定 App Code |
| `--dry-run` | boolean | 否 | `false` | 输出精确 URL patch，不写入 |

## 目标边界

- 目标类型只能是 `iframe` 或 `link`。
- 目标必须没有子菜单，且平台 path 非空。
- 命令一次只处理一个 ID，不支持 label/path 模糊定位或批量 URL patch。
- 服务端请求体严格为 `{ id, appCode, url }`，不发送 label、type、path、parentId、sort、visible、extend 或权限字段。

## 并发与幂等

1. `--expect-parent-id` 始终校验，包括 no-op。
2. 当当前 URL 不等于目标 URL、真正需要写入时，当前 URL 必须匹配 `--expect-url`。
3. 当当前 URL 已等于目标 URL 时，返回成功 no-op；此时允许 `--expect-url` 仍是旧 URL。
4. 写入前重新读取完整菜单快照；任一菜单事实漂移都会阻断写入。
5. 写后回读并确认 URL 已生效，同时 ID、label、type、path、parentId、sort、visible 保持不变。

## 响应丢失

更新请求永不自动重试。发生超时、连接重置等响应丢失时：

- 回读确认目标 URL 和全部不变量后，返回成功并标记 `recovered=true`。
- 回读仍是旧 URL、目标缺失或回读失败时，返回 `menu_update_recovery_required`，并提供 `verification.lookupCommand`。
- 回读发现目标 URL 已生效但其他菜单字段改变时，返回验证失败，要求人工核对。

## 常见错误

- 用 `menu asset-update` 修改 iframe URL；该命令只管理微前端 JS/CSS resources。
- 用 `external-link-create` 新建后删除旧菜单；这会改变 ID、path 及其权限关联。
- response loss 后直接重发写请求；应先执行错误中的 lookup 命令。
- 省略 `--expect-url` 或 `--expect-parent-id`，失去并发保护。

## 参考

- [menu list](rabetbase-menu-list.md)
- [menu external-link-create](rabetbase-menu-external-link-create.md)
- [menu asset-update](rabetbase-menu-asset-update.md)
- [SKILL.md](../SKILL.md)
