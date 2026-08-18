# page custom-publish

发布自定义页面的当前保存内容。

```bash
rabetbase page custom-publish --id <pageId> --dry-run --format compress
rabetbase page custom-publish --id <pageId> --remark "发布说明"
```

## 参数

| 参数 | 必填 | 说明 |
|---|---|---|
| `--id <pageId>` | 是 | 自定义页面 ID |
| `--remark <text>` | 否 | 发布备注 |

## 执行步骤

1. 先使用 `page custom-detail` 检查 `status` 和页面版本
2. `status` 为 `FORMAL` 时表示当前内容已经发布，跳过发布，继续查看已发布页面或后续流程
3. `status` 不是 `FORMAL` 时，使用 `--dry-run` 确认目标页面和发布备注
4. 审阅预览后，使用相同参数移除 `--dry-run` 执行发布
5. 命令发布后回读确认 `data.after.status` 为 `FORMAL`；`data.runtimePageUrl` 查看最近一次已发布内容，`data.pageUrl` 查看最新保存内容（包含尚未发布的修改），`data.editPageUrl` 用于打开页面编辑器。
