# rabetbase notification config-list

只读查询当前应用的应用级通知渠道配置，主要用于获取 BFF 消息通知扩展需要的 `configCode`。

## 命令

```bash
rabetbase notification config-list --format compress
rabetbase notification config-list --type EMAIL --format compress
rabetbase notification config-list --type EMAIL --appcode app-xxxxx --format json
```

`--type` 默认 `EMAIL`，支持：

* `EMAIL`
* `FEISHU`
* `DINGTALK`
* `WECOM`
* `WEBHOOK`

类型值使用大写。

## 输出

```json
{
  "ok": true,
  "data": {
    "appCode": "app-xxxxx",
    "channelType": "EMAIL",
    "count": 1,
    "configs": [
      {
        "id": 7,
        "configCode": "ncc_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "configName": "官方邮件",
        "channelType": "EMAIL",
        "description": "业务通知邮件",
        "createTime": "2026-07-01T00:00:00Z",
        "updateTime": "2026-07-02T00:00:00Z"
      }
    ]
  },
  "message": "Found 1 EMAIL notification config(s)"
}
```

CLI 只输出用于选择配置的安全字段，不输出：

* `channelConfig`
* `endpointUrl`
* SMTP 用户名、密码、token、secret
* 连接超时和创建人/更新人等身份审计字段

若服务端返回的任一记录缺少非空的 `configCode`、`configName` 或 `channelType`，CLI 会返回 `api_error`，不会用空字符串伪造可用配置，也不会输出可能不完整的成功列表。

## BFF 使用

先根据 `configName` 和 `description` 选择目标配置，再把同一项的 `configCode` 传给：

```javascript
await context.client.extension.execute("notification", "send", {
  configCode: "ncc_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  audiences,
  message,
});
```

边界：

* `configCode` 是应用级通知配置编码。
* dataset 级通知通道的 `channelCode` 不能替代 `configCode`。
* 没有查询结果时，不要编造配置编码。
* 多个候选无法判断时，向用户确认，不要默认取第一条。
* 该命令不会发送通知，也不会修改平台配置。

## 参考

* [BFF 创建工作流](../guides/bff-creation-workflow.md)
* [Backend Function 消息通知扩展](../guides/backend-function.md#消息通知扩展)
