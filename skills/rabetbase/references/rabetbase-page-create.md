# page create

创建 JSX 自定义页面，并同时创建菜单入口。命令固定返回 `pageType=CUSTOM`。

该命令不接受 `--page-type`。工作台、看板、门户或复杂交互仍然属于 CUSTOM 页面；`rabetbase project` 管理独立子应用工程，不是页面类型或复杂页面的兜底入口。

```bash
rabetbase page create --page-pattern BLANK --name "客户看板" --appcode <appCode> --dry-run --format compress
rabetbase page create --page-pattern BLANK --name "客户看板" --parent-menu-id 100 --appcode <appCode>
rabetbase page create --name "客户看板" --page-content '<json>' --appcode <appCode>
```

## 参数

| 参数 | 必填 | 说明 |
|---|---|---|
| `--name <name>` | 否 | 可选页面名称，1–100 个字符；省略时默认 `自定义页面_<timestamp>` |
| `--parent-menu-id <id>` | 否 | 父菜单 ID |
| `--page-pattern <pattern>` | 条件必填 | 页面模式；当前只支持 `BLANK`，与 `--page-content` 必须且只能提供一个 |
| `--page-content <json>` | 条件必填 | 完整页面文件 JSON；对象的键为相对路径，值为文件内容，与 `--page-pattern` 必须且只能提供一个 |
| `--appcode <code>` | 是 | 目标应用 code |

## 执行步骤

1. 使用基础模板时传 `--page-pattern BLANK`；已有完整源码时只传 `--page-content`。
2. 使用 `--dry-run` 确认页面名称、父菜单与最终页面文件。
3. 确认后使用相同参数执行创建。
4. 从结构化输出的 `data.after.pageId` 确认新页面 ID，并核对 `data.pageType=CUSTOM`；模式创建返回 `data.pagePattern=BLANK`，完整源码创建返回 `null`。
5. 输出的 `pageUrl` 用于查看最新保存内容，包含尚未发布的修改；`editPageUrl` 用于打开页面编辑器，继续修改页面。

页面创建始终同时创建菜单。`BLANK` 模式会按应用语言设置生成词包；读取设置失败时使用默认中英文词包。未发布模式会在本地失败，不请求服务端。
