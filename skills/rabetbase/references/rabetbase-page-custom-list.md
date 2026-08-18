# 查询自定义页面 ID

## 命令

```bash
rabetbase page custom-list --appcode <appCode> --format json
```

## 适用场景

需要查找 自定义页面的 `pageId`，再执行 `page custom-detail`、`page custom-update` 或 `page custom-publish`。

## 参数

- `--appcode <appCode>`：必填，目标应用 code

## 输出

`data.pages` 是当前应用中可操作的 自定义页面列表

- `label`：页面名称
- `pageUrl`：查看最新保存内容的页面地址，包含尚未发布的修改；日常为 `https://daily.lovrabet.com/web-app/app/<appCode>/pages/<path>`，线上为 `https://app.lovrabet.com/app/<appCode>/pages/<path>`
- `editPageUrl`：打开页面编辑器的地址，可继续修改页面；线上为 `https://app.lovrabet.com/app/<appCode>/ide/<pageId>`，日常为 `https://daily.lovrabet.com/web-app/app/<appCode>/ide/<pageId>`
