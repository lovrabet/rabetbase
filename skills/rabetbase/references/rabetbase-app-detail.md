# rabetbase app detail

按需读取所选应用的独立部署数据同步能力标识 `vipDeploy`。风险等级为 `read`，需要当前开发者凭据及应用访问权限。

```bash
rabetbase app detail --format compress
rabetbase app detail --app <name> --format compress
rabetbase app detail --appcode <code> --format compress
```

不传选择器时使用当前应用；`--app` 临时选择已配置 profile，`--appcode` 指定 AppCode。多 profile 对应同一 AppCode 时使用 `--app` 消除歧义。沿用所选应用的既有平台连接及开发者凭据，不根据业务域名猜测平台地址。

## 输出

`data` 只包含 `vipDeploy`：

| 值 | 含义 |
|---|---|
| `true` | 平台允许进入该应用的显式独立部署数据同步流程，仍需满足对应权限和参数要求 |
| `false` | 平台未开启该能力，显式同步接口跳过执行；不推荐使用同步恢复功能 |
| `null` | 请求成功，但返回字段缺失或不是可识别的布尔值，尚未确认 |

例如：`"data":{"vipDeploy":true}`。

平台应用详情是唯一事实来源。每次执行读取一次，不透出整个扩展对象，也不返回其他部署配置。不会读取本地 `vipDeploy`，不会写 `.rabetbase.json`、改变默认应用或刷新本地缓存。手工配置的同名字段不参与判断。

网络、权限或认证失败正常返回错误，不以本地值兜底，也不转换成 `false` 或 `null`。

## 使用场景

按需查询的触发条件、结果处理及部署排查统一见[独立部署指南](../guides/independent-deployment.md)。
