# rabetbase role user-resolve

把昵称/用户名解析为 `userId`。事实源是当前租户的成员目录（`getByTenantCode`，按当前会话 tenant 解析）。用于 `user-add` / `user-remove` 前先拿到稳定 `userId`。

## 命令

```bash
rabetbase role user-resolve --name 小张 --format compress
```

## 行为

- 精确匹配 `nickname` 或 `username`（大小写不敏感）。
- 命中 0 个：报 validation error，提示核对或改用 `--user <id>`。
- 命中多个：报歧义并列出候选 `nickname(userId)`，要求用 id 消歧。
- 输出 `data.user`：`userId`、`username`、`nickname`、`matchedBy`（`id` / `name`）。

## 边界

- 该接口未列入 Apifox 6396450，依赖 live 环境；未登录会报 auth required。
- 只做只读解析，不写任何角色成员；实际加入用 [`role user-add`](rabetbase-role-user-add.md)。

## 参考

- [role user-add / user-remove](rabetbase-role-user-add.md)
- [role list](rabetbase-role-list.md)
- [SKILL.md](../SKILL.md)
