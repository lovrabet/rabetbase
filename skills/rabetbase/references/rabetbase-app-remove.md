# rabetbase app remove

从 `.rabetbase.json` 中移除一个应用配置（与 `app add` / `app use` 相同，仅操作 `.rabetbase.json`）。

## 命令

```bash
rabetbase app remove <name>
```

## 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `<name>` | string | **必填** — 要移除的应用名 |

## 输出

确认消息。如果移除的是 `defaultApp`，还会显示新的默认应用。

## 提示

- 如果移除的是 `defaultApp`，自动切换到下一个可用应用
- 如果移除后没有剩余应用，`apps` 和 `defaultApp` 字段会被清理，退回单应用模式
- 移除操作不可撤销，谨慎执行

## 参考

- [SKILL.md](../SKILL.md) — 总索引
- [rabetbase app list](rabetbase-app-list.md) — 查看所有应用
- [rabetbase app add](rabetbase-app-add.md) — 添加应用
