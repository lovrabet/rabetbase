# rabetbase app use

切换默认应用（持久写入 `.rabetbase.json` 的 `defaultApp` 字段）。

## 命令

```bash
rabetbase app use <name>
```

## 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `<name>` | string | **必填** — 已配置的应用名（如 `order`、`product`） |

## 输出

确认消息，显示切换后的默认应用名及 appcode。

## 提示

- 应用名必须已通过 `rabetbase app add` 配置
- 切换后所有不带 `--app` flag 的命令都会使用新的默认应用
- 如果只需临时切换，使用 `--app <name>` flag 而非 `app use`

## 参考

- [SKILL.md](../SKILL.md) — 总索引
- [rabetbase app list](rabetbase-app-list.md) — 查看所有应用
- [rabetbase app add](rabetbase-app-add.md) — 添加应用
