# page custom-update

从本地目录读取完整页面文件更新 自定义页面，并创建新的保存版本。目录中未包含的文件会被删除。

```bash
rabetbase page custom-update --id <pageId> --page-dir <dir> --dry-run --format compress
rabetbase page custom-update --id <pageId> --page-dir <dir> --appcode <appCode>
```

## 参数

| 参数 | 必填 | 说明 |
|---|---|---|
| `--id <pageId>` | 是 | 自定义页面 ID |
| `--page-dir <dir>` | 是 | 包含完整页面文件的本地目录，CLI 递归读取其中的常规文件 |
| `--appcode <code>` | 否 | 目标应用 code，省略时根据 `pageId` 查询 |

## 执行步骤

1. 先使用 `page custom-detail` 读取 `data.codeContent` 中的完整页面文件，并写入专用本地目录。
2. 需要调整页面布局或交互时，以具体需求和最新页面文件为准。[`page-templates.md`](../knowledge/custom-page/page-templates.md) 中的 `BLANK`、`ONEPAGE` 和 `DASHBOARD` 仅作为可选参考；模板实现适合当前需求时，可将相关结构、样式和交互选择性合并到最新页面文件中，不要求使用模板。
3. 在该目录完成所有文件新增、修改或删除后，使用 `--page-dir` 与 `--dry-run` 确认完整页面文件。
4. 确认后使用相同参数执行更新。
5. 正式更新命令结束后，无论成功、失败或中断，删除步骤 1 创建的临时页面目录，删除范围仅限该目录。
6. 从结构化输出的 `data.after.version` 确认已保存新版本。
7. 输出的 `pageUrl` 用于查看最新保存内容，包含尚未发布的修改；`editPageUrl` 用于打开页面编辑器，继续修改页面。

`page custom-update` 不接受 `--page-pattern`。内置模板在更新场景中仅作为实现参考，详细规则见 [`page-templates.md`](../knowledge/custom-page/page-templates.md)；更新始终以 `page custom-detail` 返回的最新完整页面内容为基线，避免覆盖已有功能和尚未发布的修改。
