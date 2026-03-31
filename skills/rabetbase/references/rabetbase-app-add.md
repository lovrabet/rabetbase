# rabetbase app add

添加一个新应用到 `.rabetbase.json`。如果是第一个应用，自动设为 `defaultApp`。其余 CLI 配置仍兼容读取 `.lovrabet.json`；`app` 子命令始终写入 `.rabetbase.json`。

## 命令

```bash
rabetbase app add <name> --appcode <code> [--env <env>] [--apiDir <dir>] [--cookie ...] [--riskLevel ...]
```

## 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `<name>` | string | **必填** — 应用名（自定义标识，如 `order`、`product`） |
| `--appcode` | string | **必填** — 应用的 App Code |
| `--env` | string | 目标环境（`daily` / `production`） |
| `--apiDir` | string | API 目录路径 |
| `--cookie` | string | 该应用专用 Cookie |
| `--accessKey` | string | 该应用专用 Access Key |
| `--format` | string | 输出格式 |
| `--pageSize` | number | 分页条数 |
| `--riskLevel` | string | 风险等级 |
| `--locale` | string | 地区设置 |

> 除 `--appcode` 必填外，其余均可选。未指定的字段将继承顶层共享值。

## 输出

确认消息，显示添加/更新的应用名和 appcode。

## 提示

- 若应用名已存在，执行更新操作
- 第一个添加的应用自动成为 `defaultApp`
- 添加后可通过 `rabetbase app list` 查看所有应用
- 典型的多应用初始化流程：

```bash
rabetbase app add order --appcode app-order-xxx --env production --apiDir ./apps/order/src/api
rabetbase app add product --appcode app-product-yyy --env daily --apiDir ./apps/product/src/api
rabetbase app use order
```

## 参考

- [SKILL.md](../SKILL.md) — 总索引
- [rabetbase app list](rabetbase-app-list.md) — 查看所有应用
- [rabetbase app use](rabetbase-app-use.md) — 切换默认应用
- [rabetbase app remove](rabetbase-app-remove.md) — 移除应用
