# rabetbase notification config-create / config-update / config-delete

管理当前应用的应用级通知配置。它们使用应用级 `configCode`；数据集级通知通道的 `channelCode` 不能作为这些命令的输入，也不能替代 `configCode`。

## 命令与风险

```bash
rabetbase notification config-create --file notification.json --dry-run --format compress
rabetbase notification config-create --file notification.json --format compress

rabetbase notification config-update --id 88 --from-env NOTIFICATION_CONFIG_JSON --dry-run --format compress
rabetbase notification config-update --id 88 --from-env NOTIFICATION_CONFIG_JSON --yes --format compress

rabetbase notification config-delete --id 88 --dry-run --format compress
rabetbase notification config-delete --id 88 --yes --format compress
```

* `config-create` 是 `write`：支持 `--dry-run`，预览后可执行。
* `config-update --id <id>` 和 `config-delete --id <id>` 是 `high-risk-write`：强烈建议先运行 `--dry-run`，正式执行必须显式添加 `--yes`。
* 删除应用级配置可能使引用它的数据集级通道不可用；CLI 不会验证引用关系。先确认影响范围再删除。

## 安全 JSON 输入

create 和 update 必须在下列来源中恰好一个：`--file <path>`、`--stdin` 或 `--from-env <NAME>`。不得同时使用多个来源，也不要把敏感 JSON 放进命令行参数。

输入 JSON 是 CLI 文档，不是服务端原始 DTO。允许的顶层字段只有：

```json
{
  "configName": "Technical alerts",
  "channelType": "FEISHU",
  "endpointUrl": "<sensitive endpoint>",
  "channelConfig": { "credential": "<sensitive value>" },
  "description": "Technical alert bot",
  "connectTimeout": 5000,
  "readTimeout": 10000
}
```

`channelConfig` 必须是 JSON 对象，CLI 会将其序列化为服务端所需格式。`config-create` 必须提供非空 `configName` 和 `channelType`；`config-update` 至少提供一个允许字段。支持类型：`EMAIL`、`FEISHU`、`DINGTALK`、`WECOM`、`WEBHOOK`。

禁止提交 `id`、`appCode`、`configCode`、`channelCode` 或任何未知顶层字段。服务端负责渠道专有的 endpoint 与凭据规则。显式 `null` 可清空可选字符串；`connectTimeout` 或 `readTimeout` 为 `null` 会恢复服务端默认超时：`connectTimeout` 为 `5000`、`readTimeout` 为 `10000`。

## 输出与结果确认

预览和执行均返回 `operation`、`selector`、`before`、`after`、`dryRun`、`backend`、`warnings`、`verification`。输出不包含真实 `endpointUrl`、webhook token、SMTP 凭据、`channelConfig`、环境变量值、stdin 内容或文件内容；已设置的敏感字段显示为 `<redacted>`。

update 会读取精确 ID 并合并补丁，省略的字段及服务端已脱敏的敏感字段都会保留。create 和 update 按精确 ID 回读确认；delete 按删除前的 `channelType` 列表确认 ID 不存在。

请求结果未知、网络中断或回读无法证明结果时，不得自动重提。只执行只读查询：

```bash
rabetbase notification config-list --appcode app-xxxxx --type FEISHU --format compress
```

根据返回的 `configName`、`description`、`configCode` 与操作前记录人工确认后再决定下一步。
