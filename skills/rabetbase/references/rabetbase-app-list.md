# rabetbase app list

列出 `.lovrabet.json` 中所有已配置的应用。

## 命令

```bash
rabetbase app list
rabetbase app          # 等价，默认子命令
```

## 参数

无额外参数。

## 输出

- 多应用模式：每个应用的名称、appcode、env、apiDir，标记默认应用和当前应用
- 单应用模式：显示当前 appcode 并提示迁移到多应用模式的方法

## 提示

- 无 `apps` 配置时显示单应用信息
- `*` 标记表示当前选中的应用（通过 `--app` flag 临时选中）
- `(default)` 标记表示 `defaultApp`

## 参考

- [SKILL.md](../SKILL.md) — 总索引
- [rabetbase app add](rabetbase-app-add.md) — 添加应用
- [rabetbase app use](rabetbase-app-use.md) — 切换默认应用
