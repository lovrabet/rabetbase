# page react-create

创建 React 自定义页面，初始化 JSX、样式和多语言词包，并同时创建菜单入口。

```bash
rabetbase page react-create --name "客户看板" --appcode <appCode> --dry-run --format compress
rabetbase page react-create --name "客户看板" --parent-menu-id 100 --appcode <appCode>
rabetbase page react-create --name "客户看板" --page-content '<json>' --appcode <appCode>
rabetbase page react-create --appcode <appCode> --dry-run --format compress
```

## 参数

| 参数 | 必填 | 说明 |
|---|---|---|
| `--name <name>` | 否 | 可选页面名称，1–100 个字符；省略时默认 `自定义页面_<timestamp>` |
| `--parent-menu-id <id>` | 否 | 父菜单 ID |
| `--page-content <json>` | 否 | 完整页面文件 JSON；对象的键为相对路径，值为文件内容，省略时生成默认 JSX、样式和词包 |
| `--appcode <code>` | 是 | 目标应用 code |

## 执行步骤

1. 可选传入 `--page-content` 作为完整页面文件 JSON；省略时 CLI 生成默认文件。
2. 使用 `--dry-run` 确认页面名称、父菜单与最终页面文件。
3. 确认后使用相同参数执行创建。
4. 从结构化输出的 `data.after.pageId` 确认新页面 ID，并核对页面名称和生成的文件。
5. 输出的 `pageUrl` 用于查看最新保存内容，包含尚未发布的修改；`editPageUrl` 用于打开页面编辑器，继续修改页面。

页面创建始终同时创建菜单。CLI 会按应用语言设置生成词包；读取设置失败时使用默认中英文词包。
