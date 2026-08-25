# 查询自定义页面 ID

## 命令

```bash
rabetbase page custom-list --format compress
rabetbase page custom-list --appcode <appCode> --format compress
```

## 适用场景

需要查找 自定义页面的 `pageId`，再执行 `page custom-detail`、`page custom-update` 或 `page custom-publish`。

## 参数

- `--appcode <appCode>`：可选；不传时从当前工作区 `defaultApp` 或唯一应用自动决议，传入时覆盖工作区选择

工作区无可决议应用且未传 `--appcode` 时，命令返回结构化的应用配置缺失错误，不会请求菜单接口。

## 输出

`data.pages` 是当前应用中可操作的 自定义页面列表

- `label`：页面名称
- `pageUrl`：查看最新保存内容的页面地址，包含尚未发布的修改；基于当前 region 或显式 `appDomain`
- `editPageUrl`：打开当前节点或独立部署工作台中的页面编辑器
