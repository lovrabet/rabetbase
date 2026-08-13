# rabetbase user-account

绑定当前已登录的 Lovrabet 平台用户与钉钉沙箱用户账号。该能力只处理显式提供的钉钉沙箱用户 ID，不负责打开登录流程或自动获取 ID。

## 命令

先预览：

```bash
rabetbase user-account dingding-sandbox-bind \
  --ding-talk-user-id <id> \
  --dry-run \
  --format compress
```

确认 ID 后正式绑定：

```bash
rabetbase user-account dingding-sandbox-bind \
  --ding-talk-user-id <id> \
  --format compress
```

该命令不需要 `appCode`，但必须使用当前有效的登录 Cookie。不要传入平台 `userId`、`providerId` 或应用编码。

## 输出确认

重点检查以下字段：

- `data.operation = "bind"`
- `data.selector.providerId = "dingding-sandbox"`
- `data.selector.dingTalkUserId` 与输入一致
- 正式执行成功时 `data.after.bound = true`
- `data.dryRun` 与本次执行模式一致

`before: null` 表示服务端当前没有提供旧绑定查询能力，不代表旧绑定不存在。只有接口明确返回 `data: true` 时，才可将正式绑定视为成功。

## 失败与恢复

- `--ding-talk-user-id` 缺失或只包含空白字符时，CLI 在发起请求前拒绝执行。
- 登录态无效时，先执行 `rabetbase auth login`，再重新确认目标钉钉用户 ID。
- 网络超时或响应丢失时，结果可能不确定，不得自动重试。
- 人工确认登录态和目标 ID 后，只能使用相同的 `--ding-talk-user-id` 显式重试，避免误绑到其他账号。
- 当前命令不提供绑定查询或解绑能力；需要确认既有绑定时，应通过平台现有查询渠道核实。
