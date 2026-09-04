# page create

创建 JSX 自定义页面，并同时创建菜单入口。命令固定返回 `pageType=CUSTOM`。

该命令不接受 `--page-type`。工作台、看板、门户或复杂交互仍然属于 CUSTOM 页面；`rabetbase project` 管理独立子应用工程，不是页面类型或复杂页面的兜底入口。

```bash
rabetbase page create --page-pattern BLANK --name "客户看板" --appcode <appCode> --dry-run --format compress
rabetbase page create --page-pattern BLANK --name "客户看板" --parent-menu-id 100 --appcode <appCode>
rabetbase page create --page-pattern ONEPAGE --name "客户看板" --appcode <appCode> --dry-run --format compress
rabetbase page create --page-pattern DASHBOARD --name "业务数据看板" --appcode <appCode> --dry-run --format compress
rabetbase page create --name "客户看板" --page-dir ./customer-dashboard --appcode <appCode>
```

## 参数

| 参数 | 必填 | 说明 |
|---|---|---|
| `--name <name>` | 否 | 可选页面名称，1–100 个字符；省略时默认 `Custom_page_<timestamp>` |
| `--parent-menu-id <id>` | 否 | 父菜单 ID |
| `--page-pattern <pattern>` | 条件必填 | 页面模式；支持 `BLANK`、`ONEPAGE`、`DASHBOARD`，不能与 `--page-dir` 同时使用；模板详情见 [`page-templates.md`](../knowledge/custom-page/page-templates.md) |
| `--page-dir <dir>` | 条件必填 | 完整页面文件目录；不能与 `--page-pattern` 同时使用，目录必须包含文件；CLI 递归读取目录内的常规文件，将相对路径和 UTF-8 文件内容作为完整页面文件提交 |
| `--appcode <code>` | 是 | 目标应用 code |

## 页面内容来源

1. 使用内置模板时显式传入 `--page-pattern`
2. 使用完整页面文件时显式传入 `--page-dir`
3. 两者同时提供、均未提供或目录为空时，本地校验失败

## 执行步骤

1. 不使用页面模板时，若本地已有完整页面目录，直接将该目录传给 `--page-dir`；若本地没有已有内容，则先在当前工作目录创建临时专用页面目录并生成完整页面文件，再传 `--page-dir`。CLI 读取目录后负责转换为请求内容，调用方不需要也不得自行拼接 `page-content` JSON。
2. 使用内置模板时，按 [`page-templates.md`](../knowledge/custom-page/page-templates.md) 选择 `BLANK`、`ONEPAGE` 或 `DASHBOARD`，并显式传入对应的 `--page-pattern`。
3. 使用 `--dry-run` 确认页面名称、父菜单与最终页面文件。
4. 确认后使用相同参数执行创建。
5. 仅当步骤 1 创建了临时页面目录时，正式创建流程成功、失败、中断或取消后删除该目录，删除范围仅限本流程创建的临时目录；通过 `--page-dir` 传入的本地已有页面目录不得删除。
6. 创建成功时，从结构化输出的 `data.after.pageId` 确认新页面 ID，并核对 `data.pageType=CUSTOM`；模板创建返回对应的 `data.pagePattern`，目录创建返回 `data.pagePattern=null`。
7. 输出的 `pageUrl` 用于查看最新保存内容，包含尚未发布的修改；`editPageUrl` 用于打开页面编辑器，继续修改页面。

页面创建始终同时创建菜单。`BLANK`、`ONEPAGE` 与 `DASHBOARD` 分别读取内置的 `templates/custom-pages/blank.json`、`templates/custom-pages/onepage.json` 与 `templates/custom-pages/dashboard.json` 作为完整页面文件。未发布模式会在本地失败，不请求服务端。

三个模板的适用场景、内容边界、mock 数据替换、真实数据接入和交互设计统一见 [`page-templates.md`](../knowledge/custom-page/page-templates.md)。页面开发和 `client` 使用方式见 [`generation-standards.md`](../knowledge/custom-page/generation-standards.md)。
