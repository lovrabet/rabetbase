# page custom-detail

查询自定义页面详情。

```bash
rabetbase page custom-detail --id <pageId> --format compress
```

## 参数

| 参数 | 必填 | 说明 |
|---|---|---|
| `--id <pageId>` | 是 | 自定义页面 ID |
| `--appcode <code>` | 否 | 目标应用 code，省略时根据 `pageId` 查询 |

## 输出

返回页面详情和当前完整页面文件。`createTime` 和 `updateTime` 以 `YYYY-MM-DD HH:mm:ss` 展示。

地址说明（按当前环境）：

- `pageUrl`：查看最新保存内容的页面地址，包含尚未发布的修改
- `runtimePageUrl`：查看已发布内容的页面地址，仅在有已发布内容时返回；格式为 `https://<appCode>.app.lovrabet.com/<pageCode>`（线上）或 `https://<appCode>.daily.lovrabet.com/<pageCode>`（日常）
- `editPageUrl`：打开页面编辑器的地址，可继续修改页面；线上为 `https://app.lovrabet.com/app/<appCode>/ide/<pageId>`，日常为 `https://daily.lovrabet.com/web-app/app/<appCode>/ide/<pageId>`

`data.codeContent` 是当前完整页面文件，可在本地完成新增、修改或删除后，作为 `page custom-update --page-content <json>` 的完整页面内容提交。
