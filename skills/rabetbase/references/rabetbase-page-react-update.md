# page react-update

以完整页面文件 JSON 更新 React 自定义页面，并创建新的保存版本。未包含的文件会被删除。

```bash
rabetbase page react-update --id <pageId> --page-content '<json>' --dry-run --format compress
rabetbase page react-update --id <pageId> --page-content '<json>' --appcode <appCode>
```

## 参数

| 参数 | 必填 | 说明 |
|---|---|---|
| `--id <pageId>` | 是 | React 自定义页面 ID |
| `--page-content <json>` | 是 | 完整页面文件 JSON；对象的键为相对路径，值为文件内容 |
| `--appcode <code>` | 否 | 目标应用 code，省略时根据 `pageId` 查询 |

## 执行步骤

1. 先使用 `page react-detail` 读取 `data.codeContent` 中的完整页面文件。
2. 在本地完成所有文件新增、修改或删除后，使用 `--dry-run` 确认完整页面文件。
3. 确认后使用相同参数执行更新。
4. 从结构化输出的 `data.after.version` 确认已保存新版本。
5. 输出的 `pageUrl` 用于查看最新保存内容，包含尚未发布的修改；`editPageUrl` 用于打开页面编辑器，继续修改页面。
