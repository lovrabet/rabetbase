# rabetbase user-account

绑定当前已登录的 Lovrabet 平台用户与飞书或钉钉沙箱账号。只处理显式提供的账号 ID，不负责打开登录流程或自动获取 ID。

## 飞书绑定

先确认当前研发登录用户与飞书 union_id，再预演并显式执行：

```bash
rabetbase user-account feishu-bind --union-id on_example_001 --dry-run --format compress
rabetbase user-account feishu-bind --union-id on_example_001 --format compress
```

- 绑定对象由当前登录会话确定，不需要 appCode，不接受代其他平台用户绑定的 userId，也不需要 appId 或 secret。
- 使用与平台飞书应用相同开发者范围内的 union_id；不要传入 open_id 或 user_id。
- CLI 去除首尾空白后按 `^on_[A-Za-z0-9_-]+$` 校验。该绑定是用户声明，格式合法不代表已验证账号归属，也不保证账号可以收到消息。
- 普通 `write` 操作；dry-run 不发送绑定请求，`after.bound = true` 只是计划目标。
- 正式执行调用 `POST /smartapi/user-accounts/cli/feishu/bind`，请求体只含 `{unionId}`；仅服务端明确返回 `success: true, data: true` 才确认成功。
- 结果中 `selector.providerId = "feishu"`、`selector.unionId` 为去除首尾空白后的输入，`before: null` 表示没有绑定查询能力，不代表旧绑定不存在。
- 网络超时、响应丢失或成功响应无法解码时，结果未知，不自动重试。先通过平台核实当前绑定，再人工决定是否使用同一 union_id 重试。
- 不提供绑定查询或解绑命令；不承诺不同用户并发绑定同一 ID 的全局唯一性。

## 钉钉沙箱命令

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
- `data.selector.providerId = "dingtalk"`
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
