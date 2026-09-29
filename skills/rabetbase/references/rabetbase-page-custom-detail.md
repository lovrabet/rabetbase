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

地址说明（按当前环境、region 与显式 `appDomain`）：

- `pageUrl`：查看最新保存内容的页面地址，包含尚未发布的修改
- `runtimePageUrl`：完整运行态页面地址；host 为 `<appCode>.<appDomain host>`。该字段存在不代表页面已经发布，是否可绑定到流程必须以 `status` 为 `FORMAL` 为准
- `editPageUrl`：打开当前节点或独立部署工作台中的页面编辑器

为独立工作流配置 `flowJson.startPath`、`APPROVAL.path` 或 `END.path` 时，只能使用 `status === "FORMAL"` 页面返回的完整 `runtimePageUrl`。页面未发布时不得填写该地址，应先询问用户是否发布；发布成功并回读为 `FORMAL` 后再绑定。

`data.codeContent` 是当前完整页面文件。将其写入本地目录后完成新增、修改或删除，再以 `page custom-update --page-dir <dir>` 提交该目录中的完整页面内容。
